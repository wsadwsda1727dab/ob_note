# DSH 插件市场与插件更新的失败排查与解决

> 文档名称：DSH 插件市场与插件更新的失败排查与解决
> 适用范围：DSH Desktop 在升级宿主版本后，插件市场与插件更新报错的诊断与处理
> 适用对象：本库作者，以及协助排查 DSH 插件问题的 AI
> 实测宿主：DSH Desktop `0.2.0-rc.2` → `0.2.1-rc.2`
> DSH_HOME：`%USERPROFILE%\.dsh`
> 当前 profile：`desktop`
> 最近更新：2026-10-19
> 关联文档：[[00note总览]]、[[07DSH插件安装]]、[[09DSH工作区写入权限]]、[[13DSH插件市场自更新限制]]

**目录**

- [[#1. 快速分诊]]
- [[#2. 背景与结论]]
- [[#3. 现象：三类失败与完整时间线]]
- [[#4. 根因一：already-installed 守卫]]
- [[#5. 根因二：pnpm 策略违规与白名单只认第一条]]
- [[#6. 处置方案]]
- [[#7. 操作步骤]]
- [[#8. 验证]]
- [[#9. 遗留与注意事项]]
- [[#10. 变更记录]]

## 1. 快速分诊

先看三处材料，再决定改什么。

| 材料 | 路径 |
|---|---|
| 市场操作日志 | `%USERPROFILE%\.dsh\profiles\desktop\.dsh-market\log.ndjson` |
| 单次 pnpm 输出 | `%USERPROFILE%\.dsh\profiles\desktop\.plugin-manager\logs\operation-*\pnpm.log` |
| HMR 覆盖层 | `%USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml` |

判据与去向：

| 看到的现象 | 判据 | 去看 |
|---|---|---|
| `{"code":"already-installed"}` | `.plugin-manager\logs\` 下**没有**对应的 `pnpm.log`，说明守卫在 pnpm 启动前就抛错了 | §4、§7 |
| `ERR_PNPM_RESOLUTION_POLICY_VIOLATIONS_UNHANDLED` | **有** `pnpm.log`，错误出在 pnpm 内部 | §5、§7.1 |
| `uninstall-blocked`、`refused while agents are running` | 有会话或代理在跑，市场拒绝卸载 | §9 第 4 条 |
| 插件从列表消失，但 `package.json` 里还留着依赖 | 卸载先把 bundles 摘了，pnpm 那步失败 | §5.3、§7.5 |

一句话判据：**失败发生在 pnpm 之前还是之内，决定了是改 HMR 还是改白名单。**

---

## 2. 背景与结论

把 DSH Desktop 从 `0.2.0-rc.2` 升级到 `0.2.1-rc.2` 之后，在插件市场里更新任何插件都会失败，报错如下：

```text
更新失败: dshmarket — dshmarket 更新失败，且更新前的构建未能验证恢复（{"code":"already-installed"}）；
请先检查该 profile，再重新启动。
```

结论先行：

- 这不是 profile 损坏，也不是插件本身坏了，**profile 的 `package.json` 与 `node_modules` 当时都是完整可用的**。
- 失败由两条独立链路叠成：更新被宿主的 **`already-installed` 守卫**拦下；卸载则撞上 **pnpm 的策略处理器缺失**。
- 处置办法是**临时关闭 HMR** 绕开守卫，并把 `billion-context` 的多条白名单规则**合并成一条**，完成更新后再还原 HMR。
- 最终三个插件全部更新成功，HMR 已恢复。

---

## 3. 现象：三类失败与完整时间线

### 3.1 时间线

时间取自市场日志，均为 UTC。

| 时间 | 操作 | 结果 |
|---|---|---|
| 10-17T11:36Z | 宿主 `0.2.0-rc.2` 下更新 `dsh-context`、`dsh-mnemon` | 成功 |
| 10-17T17:07Z 之后 | 宿主换成 `0.2.1-rc.2`，再更新任何插件 | 全部 `already-installed`，回滚同样失败 |
| 10-18T05:10:54Z | 卸载 `dsh-mnemon`：先摘掉 bundles 与对应 patch 行 | 卸载前置步骤完成 |
| 10-18T05:12:02Z | `pnpm remove dsh-mnemon` | `exit=1`，策略违规错误 |
| 10-18T05:13:27Z | 尝试重新启用 | `no loader entry matched` |
| 10-18T05:13:51Z | 改为更新 `dsh-mnemon` | `already-installed` |
| 10-18T05:17:33Z | 更新 `dsh-workbuddy-connect` 到 `0.7.8` | `already-installed` |
| 10-18T05:23:35Z | 卸载 `dsh-workbuddy-connect` | 被拒：`refused while agents are running` |
| 10-18T05:24:30Z、05:25:00Z | 再次卸载 `dsh-workbuddy-connect` | `exit=1`，策略违规错误 |
| 10-18T05:34:50Z | 市场合并 `billion-context` 的多条白名单规则 | 合并完成 |
| 10-18T05:35:48Z | 更新 `dshmarket` | 成功，升到 `1.66.37` |
| 10-18T05:36:34Z | 更新 `dsh-workbuddy-connect` | 成功（带兼容性警告） |
| 10-18T05:36:54Z | 更新 `dsh-mnemon` | 成功，升到 `0.5.32` |

### 3.2 两个关键观察

1. **`already-installed` 的失败没有 `pnpm.log`**。市场每次真正调用 pnpm 都会留一份输出，而这类失败一份都没有，说明守卫在 pnpm 启动之前就已经抛错。
2. **卸载失败有 `pnpm.log`，且错误发生在 pnpm 内部**。它和更新失败是两条不同的链路，只解决其中一个仍会卡住。

---

## 4. 根因一：already-installed 守卫

### 4.1 守卫代码

宿主 `app.asar` 里的 `installBundle` 在调用 pnpm 之前做了如下判断（`0.2.1-rc.2` 可读版约在 1678699 行；行号随版本变化，仅作定位线索）：

```js
const before = readProfileManifest('dsh', this.profile.dir).dependencies ?? {};
const known = specPackageName(spec, before, this.profile.dir);
if (this.ownerContext.get('hmr') !== undefined && known !== undefined && Object.hasOwn(before, known)) {
    throw new ManagementFailure('already-installed');
}
```

只要同时满足三个条件就直接拒绝：

- 当前 profile 加载了 HMR 插件（`ownerContext.get('hmr') !== undefined`）；
- 能从安装规格里解析出包名；
- 该包**已经在 `dependencies` 里**。

`manageError()` 把 `ManagementFailure` 的 `code` 原样输出，这就是界面上 `{"code":"already-installed"}` 的来源。

### 4.2 为什么桌面版必然命中

`desktop` profile 一定带 `profileContext`，而宿主基础组合包里 HMR 的启用条件是：

```yaml
- id: hmr
  name: '@deepseek-ai/dsh-hmr'
  disabled: !!js "!ctx.get('profileContext')"
```

`profileContext` 恒存在，所以 **HMR 在桌面版恒开，守卫恒生效**。升级前后的行为差异与这条守卫的加入时间吻合（旧版是否完全没有该判断，未逐版本验证，属推测）。

### 4.3 为什么「未验证恢复」也失败

市场更新失败后会尝试回滚，回滚仍是往 `dependencies` 里写同一个包，同样撞上守卫。日志里每次都成对出现两条记录：

```text
update-rollback: failed update command and restoration of the previous build could not be verified — {"code":"already-installed"}
update: ... exit=1 err={"code":"already-installed"}
```

这就是提示里那句「更新前的构建未能验证恢复」的含义。

---

## 5. 根因二：pnpm 策略违规与白名单只认第一条

### 5.1 触发条件

卸载走的是 `pnpm remove`，需要重新解析依赖树，解析时命中供应链冷静期策略。DSH 调用 pnpm 时传入了冷静期参数（具体是用 `--config.minimum-release-age` 传入，未抓取实际命令行，属推测）。

放行靠 `pnpm-workspace.yaml` 里的白名单：

```yaml
minimumReleaseAgeExclude:
  - dshmarket@1.66.6 || 1.66.7 || ... || 1.66.28 || 1.66.37
  - billion-context@0.1.189 || 0.1.195 || 0.1.196 || 0.1.197
```

（首行为节选，省略号表示中间还有若干版本。）

**关键约束：pnpm 对同一个包名只认第一条规则。** 市场日志在合并前记录过这一点：

```text
minimumReleaseAgeExclude held several rules for billion-context,
and pnpm honours only the FIRST per name (#732) — merged each package's rules into one, before running
```

所以 `billion-context` 在列表里出现多行时，只有第一行生效；把 `0.1.199` 另起一行追加，**等于没加**。本次排查中我最初就是这样追加的，真正生效的是市场在 05:34:50 自己把多条规则合并成了一条。

### 5.2 为什么报「没有处理器」

pnpm 的错误码是 `ERR_PNPM_RESOLUTION_POLICY_VIOLATIONS_UNHANDLED`，提示原文：

```text
Wire setupPolicyHandlers (in @pnpm/installing.commands) or supply a callback directly.
```

意思是「没有人接住策略违规这个事件」。更新与安装路径上，市场会在调用 pnpm 前自己处理策略（合并规则、必要时放行）；**卸载路径没有接这个处理器**，pnpm 于是直接抛错。这属于调用方接线不全，与插件本身无关。

### 5.3 顺带造成的半完成状态

卸载流程是两步：先把插件从 `dsh.profile.bundles` 摘掉、禁用对应 patch 行（05:10:54），再执行 `pnpm remove`。第二步失败后，插件就停在中间态：

- `package.json` 的 `dependencies` 里还在；
- `dsh.profile.bundles` 里已经没了；
- loader 里也找不到（`no loader entry matched`）。

所以「插件从列表消失但仍占着依赖」不是配置损坏，而是卸载流程走到一半的正常中间态，按 §7.5 重新装入即可恢复。

---

## 6. 处置方案

### 6.1 三条路

| 方案 | 做法 | 代价 |
|---|---|---|
| A. 临时关闭 HMR | 在 `cordis.patch.yml` 里禁用 `hmr`，重启后更新，再还原 | 三次重启；期间没有热加载 |
| B. 市场内先卸载再装 | 每次更新前卸载目标插件 | 会撞上 §5.1 的卸载崩溃；有会话在跑时还会被拒 |
| C. 内置 CLI 直连 pnpm | 完全退出 DSH 后执行 `dsh plugin --profile desktop add <包名>` | 绕过市场守卫，但仍走 profile 的兼容性预检；没有回滚；版本要自己核对 |

### 6.2 为什么选 A

- B 需要每次卸载，正好踩在没修好的那条链路上。
- C 绕过了守卫，但兼容性预检仍在，出错时市场监管与回滚都不存在，只适合 A 不可用时兜底。
- A 只改一行配置，更新走的是 `pnpm add`，不碰卸载路径。

前提是先把白名单理顺：**同一包名只能有一条规则**，多行合并成一行。

### 6.3 最终动作顺序

1. 合并白名单里 `billion-context` 的多条规则。
2. 备份并临时禁用 HMR。
3. 重启 DSH。
4. 在市场里依次更新插件，恢复 `dsh-mnemon` 的启用状态。
5. 重启，确认新版本生效。
6. 删除 HMR 覆盖，重启，恢复热加载。

---

## 7. 操作步骤

以下路径以本次实测环境为准，`%USERPROFILE%` 展开为实际用户目录。

### 7.1 合并 billion-context 的白名单规则

编辑 `%USERPROFILE%\.dsh\profiles\desktop\pnpm-workspace.yaml`，把 `billion-context` 的所有规则并成**一条**：

```yaml
minimumReleaseAgeExclude:
  - billion-context@0.1.189 || 0.1.195 || 0.1.196 || 0.1.197 || 0.1.199
```

- 不要新开一行追加，见 §5.1：pnpm 只认第一条。
- pnpm 11 在安装新版本时也会自己往列表里补条目，格式被写乱后越来越难读，手工改时保持这种范围写法。
- 该版本建议长期保留：`0.1.199` 是已装并在用的版本，保留它能避免以后重复触发同一条策略。

### 7.2 确认当前 HMR 状态

打开 `%USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml`，搜索 `id: hmr`：

- 搜不到：没有被覆盖，按组合包默认**启用**。
- 搜到三行 `- id: hmr` / `disabled: true`：当前处于**禁用**状态，需要按 7.6 还原。

### 7.3 临时禁用 HMR

先备份，再在 `cordis.patch.yml` 末尾追加：

```yaml
- id: hmr
  disabled: true
```

原文件 79 行，追加后为 82 行。

### 7.4 完全退出并重启 DSH

必须**完全退出**（不是关窗口），否则 profile 仍被占用。

- 卸载类操作要求没有会话在跑（否则报 `refused while agents are running`）。
- 更新期间不要用终端里的 `dsh plugin` 命令，那是另一条通道，会绕过市场的界面与回滚。

### 7.5 在市场里更新插件

按顺序，每次等一个走完再点下一个：

1. `dshmarket` 自身。
2. `dsh-workbuddy-connect` 到 `0.7.8`。
3. `dsh-mnemon` 到 `0.5.32`，并确认它重新出现在已启用列表里。

若第 1 步提示「无法自我卸载」，本节给出的两条界面路径**均不可行**：桌面端「设置 → 插件」对 `dshmarket` 不渲染更新入口。先跳过市场、把其余插件更新完，市场改用完全退出 DSH 后的手动 `add`，见 [[13DSH插件市场自更新限制]] 第 6 章。

半完成状态（§5.3）就在第 3 步恢复：重新装回即可，`dependencies` 里残留的条目不碍事。

### 7.6 还原 HMR

删掉 7.3 追加的三行，确认文件回到 79 行，再重启一次。

---

## 8. 验证

### 8.1 版本

```text
dshmarket 1.66.37
dsh-workbuddy-connect 0.7.8
dsh-mnemon 0.5.32
```

### 8.2 启用状态

打开 `%USERPROFILE%\.dsh\profiles\desktop\package.json`，确认 `dsh.profile.bundles` 里三个插件都在（半完成状态的特征是依赖在、bundles 不在）。

同时看 `%USERPROFILE%\.dsh\profiles\desktop\.dsh-market\state.json`：`disabled` 为空、无 `brokenPlugins`。

### 8.3 HMR 已恢复

`cordis.patch.yml` 里搜不到 `id: hmr` 覆盖行，且文件回到 79 行。

### 8.4 运行证据

- `%USERPROFILE%\.dsh\.workbuddy-host-heartbeat.json` 的修改时间晚于重启时间，说明 workbuddy 实际加载并运行。
- 市场日志 `log.ndjson` 在重启之后没有新增 `error` 级事件。

---

## 9. 遗留与注意事项

1. **兼容性警告不等于失败**：`dsh-workbuddy-connect@0.7.8` 声明依赖 `@deepseek-ai/schemastery@^3.18.5`，本机宿主是 `3.18.4`。市场只记一条 `update-compat` 警告，插件仍正常加载。若日后出现与该包相关的异常，根因在此。
2. **更新市场自身仍走特殊路径**：市场拒绝自我卸载；桌面端「设置 → 插件」对 `dshmarket` 也**不渲染更新入口**（它不在 `OFFICIAL_ON_DEMAND_CATALOG` 里，`installTarget` 为 `undefined`，见 #909）。唯一可行的是完全退出 DSH 后手动 `add`，命令、`--save-exact` 与 peer 重解析等注意事项见 [[13DSH插件市场自更新限制]] 第 6 章。
3. **关闭 HMR 期间新装插件不受影响**：守卫只拦 `dependencies` 里已有的包，新插件不在其中，能正常安装；受影响的是热加载，改配置要重启才生效。
4. **卸载前先停会话**：有会话或代理在跑时市场会拒绝卸载，报 `refused while agents are running`，这不是配置问题，停掉会话再试即可。
5. **白名单只认第一条**：同一包名多条规则时只有第一条生效（`#732`），追加必须并到同一行，否则静默失效。
6. **未复验项**：白名单合并后没有再执行一次卸载，因此「卸载崩溃是否随之消失」尚未验证；下一轮做卸载时留意日志即可确认。
7. **文中行号与版本号会过期**：`app.asar` 行号、pnpm 与各插件版本都只对本文记录的时间点成立。

---

## 10. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-18 | 1.0.0 | 新建。记录 DSH 从 `0.2.0-rc.2` 升到 `0.2.1-rc.2` 后插件市场与插件更新失败的排查与解决全过程。 |
| 2026-10-18 | 1.0.1 | 按市场日志补齐时间线；新增「快速分诊」「确认 HMR 状态」「卸载被拒」等内容；更正重启次数、新装插件是否受守卫影响、白名单只认第一条（`#732`）三处表述；对未验证的推断加了标注。 |
| 2026-10-19 | 1.0.2 | 订正第 9 章第 2 条与第 7.5 节的分支：原写「需走设置 → 插件页面」，实测该页面对 `dshmarket` 不渲染更新入口（不在 `OFFICIAL_ON_DEMAND_CATALOG` 内，`installTarget` 为 `undefined`，见 #909），改为指向 [[13DSH插件市场自更新限制]] 第 6 章的手动 `add` 路径。 |
| 2026-10-19 | 1.0.3 | 按 [[05Markdown规范]] 第 5.1 节把正文 7 行、共 14 处中文半角引号改为 `「 」`（含第 4.3、5.2 两个标题），内容未变。 |

---
