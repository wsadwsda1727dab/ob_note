# DSH 插件市场无法自我更新的起始版本、原因与出路

> 文档名称：DSH 插件市场无法自我更新的起始版本、原因与出路
> 适用范围：DSH Desktop 的市场插件（`dshmarket`）在自身「更新」入口报 #900 文案时的判定、原因追溯与手动更新路径
> 适用对象：本库作者，以及协助排查 DSH 插件市场更新问题的 AI
> 实测宿主：DSH Desktop 内置 `@deepseek-ai/dsh` = `0.2.1-rc.2`（`dsh.cmd --version` 实测）
> DSH_HOME：`%USERPROFILE%\.dsh`
> 当前 profile：`desktop`
> 实测版本：市场 `1.66.37`（本机已装）／`1.66.42`（npm `latest`，2026-10-18T20:43:54Z 发布）
> 最近更新：2026-10-25
> 关联文档：[[00note总览]]、[[07DSH插件安装]]、[[12DSH插件更新问题与解决]]

**目录**

- [[#1. 结论速览]]
- [[#2. 起始版本：两层限制]]
- [[#3. 原因：机制链]]
- [[#4. 为什么桌面端「设置 → 插件」也救不了]]
- [[#5. 现状]]
- [[#6. 手动更新的两条路径]]
- [[#7. 未验证项与注意事项]]
- [[#8. 变更记录]]

## 1. 结论速览

| 问题 | 答案 |
|---|---|
| 从什么时候开始「装不上」 | 宿主 `@deepseek-ai/dsh` **0.2.1-rc.1** 起，`installBundle` 对已在 `dependencies` 里的包抛 `already-installed` |
| 从什么时候开始「明确报 #900」 | 市场 **1.66.36** 起加 `SELF_NAMES` 守卫；在此之前的版本会真的把市场自己卸掉 |
| 为什么 | 桌面端唯一的升级方式是「先卸载、再安装」；市场一旦被卸载，执行第二步的代码已经不在运行组合里 |
| 现在是什么状态 | 本机 `1.66.37`，npm `latest` 是 `1.66.42`；官方 #900 仍 open，客户端侧指引未实现 |
| 怎么更新 | 完全退出 DSH Desktop 后，用桌面端自带的 CLI carrier 或捆绑的 pnpm 手动 `add` |

一句话判据：**看到 #900 那段中英双语文案，说明守卫按设计拒绝了请求，不是安装损坏，profile 不需要修复。**

---

## 2. 起始版本：两层限制

这里有两个不同的起点，混在一起会得出错误结论：一层是宿主让「重新安装同一包」这件事本身失败，另一层是市场为避免自我销毁而加的显式拒绝。

### 2.1 第 1 层：能力上无法自更新——宿主 0.2.1-rc.1

宿主 `@deepseek-ai/dsh-plugin-manager/lib/index.js:1991`：

```js
if (this.ownerContext.get("hmr") !== void 0 && known !== void 0 && Object.hasOwn(before, known))
    throw new ManagementFailure("already-installed");
```

`before` 取自 `readProfileManifest("dsh", this.profile.dir).dependencies`（`:1989`）。只要包名已经在依赖里，`add` 就会被拒绝；而「更新」在语义上恰恰就是「对同一个包再 `add` 一次」。

注意第一个条件 `this.ownerContext.get("hmr") !== void 0`：**守卫只在 HMR 启用时成立**。[[12DSH插件更新问题与解决]] 第 7.3 节的「临时禁用 HMR」之所以能绕开它，原因就在这里。

### 2.2 第 2 层：市场侧加护栏——市场 1.66.36

市场先要绕过上一条限制：`lib/official-desktop.js` 的 `installOrReinstall` 遇到 `already-installed`，就回退成「先 `removeBundle`、再 `installBundle`」，与桌面端 UI 的两步更新一致。但这样市场自己就成了被卸载的对象，于是 1.66.36 加了护栏。

逐版本抓取 npm 上的 `lib/official-desktop.js`，检查其中是否出现 `#900`、`market-self-update` 字符串，结果如下。表中的长度是文件字符数，不是字节数；本机 `1.66.37` 实测为 15397 字符 / 16067 字节，可作口径参照：

| 版本 | 文件字符数 | `SELF_NAMES` 守卫 |
|---|---|---|
| 1.66.29 – 1.66.32 | 9179 | 无 |
| 1.66.33 – 1.66.34 | 11395 | 无（#900 报告人在这两版实测到市场被卸掉） |
| 1.66.35 | 13486 | 无 |
| **1.66.36** | **15397** | **首次出现** |
| 1.66.37 – 1.66.39 | 15397 | 有 |
| 1.66.40 | 16622 | 有（文案改写，见 §4.2） |
| 1.66.41 – 1.66.42 | 18757 | 有 |

GitHub 上 `v1.66.35...v1.66.36` 的第一个 commit 就是 `17b448b fix(desktop): never remove the market itself to update it (#900)`。

### 2.3 1.66.36 前后的行为差异

- **1.66.35 及更早**：自更新会真的把市场从 `dsh.profile.bundles` 里摘掉，然后卡在第二步。结果是市场从列表消失、`dependencies` 里还留着依赖，profile 处于半损坏状态，且日志里只有裸的 `{"code":"already-installed"}`，没有任何说明。
- **1.66.36 起**：进入 `installOrReinstall` 后先检查 `SELF_NAMES`，命中就直接返回 `market-self-update` 并附上 #900 文案。静默损坏变成了显式拒绝。

宿主侧这条守卫的来历，市场源码注释引的是 `#896`，本次排查第 2 轮核对到的是 `#897`（closed）；两者与守卫的对应关系未逐条复核，见 §7。

---

## 3. 原因：机制链

### 3.1 桌面端没有「原地升级」

客户端 `@deepseek-ai/dsh-client-ui-plugin-manager/lib/client.js:1162` 的 `update` 只有一条路。真正的动作在 `:1170` 起的 `run` 回调里：

```js
this.run(packageName, { packageName, action: "uninstall" }, async () => {
    const removed = await this.ctx.remote.pluginManager.removeBundle(packageName);
    if (removed.ok && removed.value.error?.code === "stop-profile") {
        this.openCatalogInstall(pkg, target, pkg.enabled);
        return;
    }
    this.applied(removed, packageName);
    this.openCatalogInstall(pkg, target, pkg.enabled);
});
```

无论走哪个分支，都是先 `removeBundle`、再 `openCatalogInstall`，没有「原地升级」的写法。

### 3.2 卸载对市场是致命的

`@deepseek-ai/dsh-plugin-manager/lib/index.js:2145` 的 `removeBundle` 在真正跑 `pnpm remove` 之前，先做的是：

```js
if (bundle.enabled) {
    await this.selectBundle(name, false);
    Object.assign(result, await this.reload());
}
```

也就是把该 bundle 从运行组合里摘掉并触发热重载。对其他插件这只是普通卸载；对市场，被摘掉的正是「接下来要去安装新版本」的那段代码。

注意 `:2142` 还有一道前置闸：HMR 未启用时（`ownerContext.get("hmr") === void 0`），若该 bundle 已在启动列表里或正被加载，`removeBundle` 直接抛 `stop-profile` 而不是动手移除。这解释了客户端 `update` 里为何单独有一个 `stop-profile` 分支——HMR 关闭时市场连「被卸掉」都轮不到。

### 3.3 第二步于是永远不会执行

`openCatalogInstall` 是一个「再也不会被消费的 UI 意图」。坏结局是 `dependencies` 里还留着 `dshmarket`，`dsh.profile.bundles` 里那一行却没了，下次启动组合直接失败——这正是 #900 描述的现象。

### 3.4 市场自己加护栏

`lib/official-desktop.js:70`：

```js
const SELF_NAMES = new Set(['dshmarket', 'dsh-market']);
```

`installOrReinstall` 内，`:129-131`：

```js
if (SELF_NAMES.has(name)) {
    return { application: 'failed', error: { code: 'market-self-update', message: selfUpdateMessage(spec) } };
}
```

`selfUpdateMessage`（`lib/official-desktop.js:81`，本机 `1.66.37` 共 6 行）返回的就是中英双语的 #900 文案。所以 1.66.36 起，市场是「知情地拒绝」，而不是「更新被弄坏」。

---

## 4. 为什么桌面端「设置 → 插件」也救不了

这是最常见的一处误解，值得单独说明。

### 4.1 该页面不渲染市场的更新入口

宿主 `listBundles()`（`@deepseek-ai/dsh-plugin-manager/lib/index.js:1688`、`:1707`）只对预置目录里命中的包填 `installTarget`：

```js
const catalog = new Map(OFFICIAL_ON_DEMAND_CATALOG.map((entry) => [entry.packageName, entry]));
// ...
if (offered !== void 0) target = { installTarget: officialBundleInstallTarget(...) };
```

实测反解 `app.asar`，`OFFICIAL_ON_DEMAND_CATALOG` 只有两项：`@deepseek-ai/dsh-subagent-claude-code`、`@deepseek-ai/dsh-subagent-codex`。

客户端 `updateAvailable`（`client.js:2614`）取到的 `target` 就是 `installTarget`，第 2616 行据此短路：

```js
const target = pkg.installTarget;                                    // :2615
if (!pkg.installed || !pkg.removable || pkg.availability === "installation" || target === void 0) return false;   // :2616
```

`dshmarket` 不在目录里，`installTarget` 为 `undefined`，所以**市场那一行根本不会出现更新按钮**。

### 4.2 你看到的文案是旧版

`selfUpdateMessage` 的返回值按版本对比。

**`1.66.37` – `1.66.39`**：

> 在桌面端的「设置 → 插件」里更新市场，或完全退出 DSH 后在终端执行 `dsh plugin --profile desktop add …`

两条路都想不通：前者在该页面没有按钮（§4.1）；后者在普通终端会被宿主以 `profile "desktop" is managed exclusively by the Electron application` 拒绝。

**`1.66.40` – `1.66.42`**：

> 应用内的「设置 → 插件」没有更新入口（#909）。要更新：完全退出 DSH Desktop，在 profile 目录执行 `pnpm add dshmarket@<ver> --config.minimum-release-age=0`

这条可行，见 §6 方法 B。

本机 `1.66.37` 对应前一段，所以那段文案给出的两条路在本机都走不通。

### 4.3 错误码尚未在客户端落地

`1.66.42` 的 `lib/routes.js` 已把 `market-self-update` 作为 `failureCode` 回传，但 `client/client.js` 中检索 `market-self-update`、`failureCode` 均为 **0 处命中**。即便升到 `1.66.42`，点「更新」也只会弹错误，不会弹出指引。

---

## 5. 现状

### 5.1 本机

| 项 | 值 |
|---|---|
| profile | `%USERPROFILE%\.dsh\profiles\desktop` |
| 已装市场 | **`1.66.37`**（`package.json` 与 `node_modules\dshmarket\package.json` 一致） |
| `dsh.profile.bundles` | 8 项，含 `dshmarket` |
| profile 是否损坏 | **否**：依赖在、bundle 行在、市场日志中无针对 `dshmarket` 的 `update-rollback` |
| 宿主运行时 | `@deepseek-ai/dsh` = `0.2.1-rc.2`；捆绑 pnpm `11.28.5` |
| `dsh` 是否在 PATH | **不在**（用户级与机器级均已确认），必须用完整路径 |

### 5.2 日志里两类失败形态不同

日志文件：`%USERPROFILE%\.dsh\profiles\desktop\.dsh-market\log.ndjson`。

| 时间（UTC） | 当时市场版本 | 日志内容 | 含义 |
|---|---|---|---|
| 2026-10-17T12:01 – 12:06 | `1.66.35` | 裸 `{"code":"already-installed"}` | 当时还没有守卫，走到一半被宿主挡住 |
| 2026-10-19T04:47:42 / 04:48:07 / 04:48:44 / 04:48:46 | `1.66.37` | 四次同一段 #900 文案，`exit=1` | 守卫按设计拒绝 |

同一段日志里 `2026-10-19T04:46:53Z` 有一条 `update-rollback`，那是 `billion-context` 的，与市场无关。

### 5.3 上游

| 项 | 值 |
|---|---|
| npm `latest` | **1.66.42**（2026-10-18T20:43:54Z） |
| 与 latest 的差 | `1.66.37` → `1.66.42`，中间 5 个版本 |
| 1.66.42 改了什么 | #906（载体被误判 broken 后立刻卸载）、#925（更新内容弹窗显示旧版说明），均与自更新无关 |
| `official-desktop.js` 有无变动 | `1.66.41` → `1.66.42` 哈希未变，说明守卫未动 |
| #900（市场不能自我卸载） | **open** |
| #909（错误码未在客户端实现） | **open** |
| #906、#925 | open |
| #897（`already-installed` 必然失败） | closed |
| 宿主版本通道 | npm `latest` 是 `0.2.0-rc.2`，`next` 才是 `0.2.1-rc.2`；本机按 `next` 打的包，后续升级宿主要留意通道 |

---

## 6. 手动更新的两条路径

前提：**完全退出 DSH Desktop**（当前会话会中断）。`withFileLock` 只在管理器操作期间持有、等待上限 120 秒，但运行中的应用不会重新加载改过的 `node_modules`，且 Windows 上覆盖已被加载的模块目录有失败风险。

执行环境均为 Windows `cmd`；两条命令都用绝对路径，因此可在任意工作目录执行。

动手前先备份两个能被覆盖的文件，出错时用于手工回滚（方法 A 自带 `restore()`，方法 B 没有）：

```cmd
copy "%USERPROFILE%\.dsh\profiles\desktop\package.json" "%USERPROFILE%\.dsh\profiles\desktop\package.json.bak"
copy "%USERPROFILE%\.dsh\profiles\desktop\pnpm-lock.yaml" "%USERPROFILE%\.dsh\profiles\desktop\pnpm-lock.yaml.bak"
```

两条路径都会改写 profile 的 `dependencies` 与 `node_modules`，属覆盖性操作；回滚方式是把这两个 `.bak` 拷回原名后重跑一次 `pnpm install --frozen-lockfile`。

### 6.1 方法 A：走桌面端自带的 CLI carrier（推荐）

```cmd
"D:\Software\sf1\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd" plugin --profile desktop add dshmarket@1.66.42 --save-exact
```

它能用而 PATH 上的普通 CLI 不能用的原因：`dsh.cmd` 设 `ELECTRON_RUN_AS_NODE=1` 后调 `dsh-desktop-host/lib/cli.js`，其 `runDesktopCli` 传 `manageDesktopProfile: true`，`bin.js` 里 `rejectElectronProfile` 那条检查因此被绕过。

好处是走 `runProfilePnpm`：它在 pnpm 之前先对将要安装的包做兼容性预检（`evaluatePluginCompatibility`），不通过时以「nothing was installed」拒绝、不改动 profile；安装后若发现不兼容则执行 `restore()` 还原 `package.json`、`pnpm-lock.yaml` 与各直接依赖的 `node_modules\<name>\package.json`，再按 `--frozen-lockfile` 重装；诊断写入 `%USERPROFILE%\.dsh\profiles\desktop\.plugin-manager\logs\`。

已验证：`dsh.cmd --version` 返回 `0.2.1-rc.2`。

### 6.2 方法 B：直接用捆绑的 node + pnpm

即 `1.66.40+` 文案指的路子：

```cmd
"D:\Software\sf1\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\node\bin\node.exe" "D:\Software\sf1\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\pnpm\bin\pnpm.mjs" --dir "%USERPROFILE%\.dsh\profiles\desktop" add dshmarket@1.66.42 --save-exact --config.minimum-release-age=0
```

已验证：`node.exe` 与 `pnpm.mjs` 均在，`pnpm --version` 返回 `11.28.5`。该方法**不经过**宿主的兼容性预检与回滚，出错时只能靠事先备份的 `package.json`、`pnpm-lock.yaml` 手工恢复，属兜底路径。

### 6.3 三条必须注意的

1. **写裸包名必须加 `--save-exact`；命令里已显式写 `@版本` 时可省。** 捆绑的 pnpm 11.28.5 在 `saveExact` 与 `savePrefix` 都未设置时走 `getRangeSpecStyle` 的 `default: return "major"`，裸包名（`add dshmarket`）会被写成 `^1.66.42`；本机 profile 里全是精确版本（如 `"dshmarket": "1.66.37"`），所以裸名要加 `--save-exact`。但显式写了 `@版本` 时，`add` 的写回值由 `calcVersionRange` 取 `inferRangeSpecStyle(bareSpecifier)` 决定，`1.66.42` 被判为精确写法而直接沿用，`--save-exact` 不参与决策。2026-10-25 用捆绑 pnpm 在临时目录以 `--lockfile-only --offline` 实测四组：裸名写回 `^3.18.5`、裸名加 `--save-exact` 写回 `3.18.5`、显式 `@3.18.5` 写回 `3.18.5`、已有 `^3.18.4` 时升到 `@3.18.5` 也写回 `3.18.5`。§6.1、§6.2 的命令都显式带了版本，`--save-exact` 在这里属于冗余保险，留着无害；若改成裸名则不能省。宿主自己也只是把 `--save-exact` 作为可选参数透传，市场调用 `installBundle` 时没有传。
2. **手动 `add` 会连带重解析 peer 实例。** #909 评论中有人实测从 `1.66.27` 升到 `1.66.42` 时，`pnpm-lock.yaml` 里 `dshmarket@1.66.42` 的 peer 从 `@deepseek-ai/schemastery@3.18.4` 变成 `3.18.5`。本机 `1.66.37` 声明的是 `@deepseek-ai/schemastery: ^3.18.1`（optional peer），升级后同样可能被重解析。这是已知副作用，不是失败。
3. **`--config.minimum-release-age=0` 在本机是多余的，但写上无害。** `pnpm-workspace.yaml` 里没有 `minimumReleaseAge` 键，pnpm 11.28.5 的默认值是 `opts3.minimumReleaseAge ?? 0`。

### 6.4 升级后自查

```cmd
type "%USERPROFILE%\.dsh\profiles\desktop\package.json"
```

确认两点：`"dshmarket": "1.66.42"`（不是 `^1.66.42`），且 `dsh.profile.bundles` 里 `dshmarket` 那一行还在。

---

## 7. 未验证项与注意事项

1. **「守卫只与 HMR 启用有关」推论未实测。** §2.1 的源码条件表明，HMR 关闭时 `installBundle` 不会抛 `already-installed`，`installOrReinstall` 也就走不到 `SELF_NAMES` 分支，理论上市场可以在 HMR 关闭期间完成自我更新。此推论**未经实测**，不要当成确定结论；[[12DSH插件更新问题与解决]] 第 7.3 节的 HMR 方案只针对普通插件验证过。
2. **#900 是否已修未复核。** 截至本次排查，GitHub 上 #900 仍为 open；`1.66.42` 的 `official-desktop.js` 哈希与 `1.66.41` 相同，说明至少到该版本守卫仍在。
3. **`1.66.29` 之前的版本未逐版探测。** 更早的版本是否也带这条保护逻辑未知。
4. **升级后的 `minimumReleaseAgeExclude` 未验证。** 本机该文件里 `dshmarket` 那条目前只列到 `1.66.37`；市场自身有 `#732` / `#883` 的合并逻辑，升级后是否会正确追加，与本次手工执行是否冲突，均未验证。
5. **issue 编号对应关系未逐条复核。** 市场源码注释引 `#896`，本次排查第 2 轮记录为 `#897`，两者是 issue 与 PR 的关系还是号码笔误，未进一步确认。
6. **`app.asar` 行号会过期。** 本篇所有行号只对 `@deepseek-ai/dsh` = `0.2.1-rc.2`、市场 `1.66.37` 这一时间点成立。
7. **本篇记录的是「不能自更新」的机制与出路，普通插件更新失败另有两条根因。** 宿主 `already-installed` 守卫与 pnpm 策略违规的排查、HMR 临时关闭与白名单合并的完整操作，见 [[12DSH插件更新问题与解决]]。

---

## 8. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-19 | 1.0.0 | 新建。记录市场无法自我更新的两层起始版本、机制链、「设置 → 插件」为何无入口、本机与上游现状、两条手动更新路径与未验证项。素材来自 2026-10-19 的 DSH 排查会话（三轮）。 |
| 2026-10-19 | 1.0.1 | 格式自查：§4.2 的版本文案表单元格过长且「」同形套嵌，改为逐版本引用块；§5.3 拆开一行超长单元格；§6 补备份与回滚步骤（两条命令都覆盖 profile）。 |
| 2026-10-25 | 1.0.2 | 订正 §6.3 第 1 条：原写「必须加 `--save-exact`」，实际只在不写版本时必要，命令显式写 `@版本` 时 pnpm 的 `calcVersionRange` 会沿用精确写法。补 2026-10-25 四组 `--lockfile-only` 实测结果。 |

---
