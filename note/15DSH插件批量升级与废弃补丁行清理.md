# DSH 插件批量升级与废弃补丁行清理

> 文档名称：DSH 插件批量升级与废弃补丁行清理
> 适用范围：DSH Desktop 的 `desktop` profile 下 9 个第三方插件的版本核对、批量升级、发布等待期豁免、`dsh-ops` 新旧组件行迁移与升级后核验
> 适用对象：本库作者，以及协助维护本机 DSH 插件环境的 AI
> 实测宿主：DSH Desktop 内置 `@deepseek-ai/dsh` = `0.2.1-rc.3`
> DSH_HOME：`%USERPROFILE%\.dsh`
> 当前 profile：`desktop`
> 实测日期：2026-10-29
> 最近更新：2026-10-29
> 关联文档：[[00note总览]]、[[07DSH插件安装]]、[[12DSH插件更新问题与解决]]、[[13DSH插件市场自更新限制]]

**目录**

- [[#1. 结论速览]]
- [[#2. 升级范围与版本对照]]
- [[#3. 五个包的版本内容要点]]
- [[#4. 升级前的四个阻塞项与处置]]
- [[#5. 操作步骤]]
- [[#6. 实际改动清单]]
- [[#7. 验证结果]]
- [[#8. 未验证项]]
- [[#9. 回滚]]
- [[#10. 值得留的经验]]
- [[#11. 变更记录]]

## 1. 结论速览

| 问题 | 答案 |
|---|---|
| 本次动了什么 | 把 5 个落后版本升到最新，另 4 个本已最新未动；删除 profile 补丁层里 2 条指向旧组件名的废弃覆盖行 |
| 升级是否需要装 pnpm | 不需要。`dsh.cmd` 自带 carrier，`dsh.cmd plugin --profile desktop --version` 输出 `11.28.5` |
| 为什么要动等待期白名单 | 2 个新版本仍在 `minimumReleaseAge` 默认 1440 分钟内，写进 lockfile 后会让 pnpm 校验拦下全部插件安装 |
| 旧补丁行有没有副作用 | 没有挂载副作用，只有启动时 2 条 `patch: entry ... not found` 警告；删除前后组合出的 entry 树完全一致 |
| 怎么确认升级真的生效 | 看重启后请求头的工具集：官方 `read`/`glob`/`grep`/`pwsh` 被隐藏、`ops_bash` 存在，说明 `dsh-ops` 的 standard 模式已在跑 |
| 还有什么没解决 | 三个纯 UI 插件拿不到宿主侧运行时证据；插件市场的目录抓取仍走腾讯云镜像并超时 |

一句话判据：**本轮升级的验证结论靠「请求头工具集变化 + 组件定义状态版本 + 心跳文件」三类运行时痕迹互证，不依赖插件市场界面。**

---

## 2. 升级范围与版本对照

### 2.1 版本对照

升级前从 `%USERPROFILE%\.dsh\profiles\desktop\package.json` 与 npm registry 的 `latest` 逐项对比，9 个依赖中 5 个落后、4 个已最新。

| 包 | 升级前 | 升级后 | 处置 |
|---|---|---|---|
| `@michengai/dsh-archive-manager` | `1.1.16` | `1.1.18` | 升级 |
| `dsh-better-sidebar` | `0.28.1` | `0.28.2` | 升级 |
| `dsh-context` | `0.77.0` | `0.78.0` | 升级 |
| `dsh-ops` | `0.4.1` | `0.4.4` | 升级 |
| `dshmarket` | `1.70.2` | `1.73.3` | 升级 |
| `billion-context` | `0.1.209` | `0.1.209` | 已最新，未动 |
| `dsh-connect-workbuddy` | `5.13.1` | `5.13.1` | 已最新，未动 |
| `dsh-find-plugins` | `0.3.0` | `0.3.0` | 已最新，未动 |
| `dsh-mnemon` | `0.5.36` | `0.5.36` | 已最新，未动 |

### 2.2 宿主兼容性

三个带 `engines.dsh` 声明的包都覆盖本机宿主 `0.2.1-rc.3`：

| 包 | `engines.dsh` 声明 | 是否覆盖 `0.2.1-rc.3` |
|---|---|---|
| `@michengai/dsh-archive-manager` | `>=0.2.0-rc.1 <0.3.0-0` | 覆盖 |
| `dsh-context` | `>=0.2.0-rc.2 \|\| >=0.2.1-rc.2` | 覆盖 |
| `dsh-ops` | `>=0.2.0-rc.2 <0.2.2-0` | 覆盖 |
| `dsh-better-sidebar` | 无 `engines.dsh` | 不适用 |
| `dshmarket` | 无 `engines` | 不适用 |

`dsh-better-sidebar` 的兼容性改由 peer 依赖表达：17 项 peer 全部是 `@deepseek-ai/dsh-*`、`@deepseek-ai/cordis ^4.0.5`、`@huanlin/dsh-plugin-better-locale` 与 `react`/`react-dom`，宿主件版本要求为 `^0.2.1-rc.1`。

---

## 3. 五个包的版本内容要点

以下内容取自各仓库的 CHANGELOG 与 GitHub Releases；`dsh-better-sidebar` 未取到发布说明，只记元数据。

### 3.1 `@michengai/dsh-archive-manager` 1.1.17、1.1.18

- **1.1.17**：侧栏工作区行与会话行支持整行右键菜单；收藏星标改为复用宿主前导插槽的 16×16 图标并绝对定位，消除标题抖动，状态点只在悬停或已收藏时淡出；归档设置页做视觉一体化（工作区卡片改抽屉式，会话行高 48px → 40px，复选框回归宿主原生 Checkbox 加 `accent-color`）；弹窗统一用 `portalToBody` 挂载，修大弹窗偏左与文字透视重叠，关闭按钮补中英俄三语无障碍标签；设置页 section ID 改为 `archive-manager`，消除与第三方同类插件的注册冲突。
- **1.1.18**：修收藏星与宿主会话状态点互相遮挡。会话运行中或等待审批时宿主状态点保持完全可见，已收藏会话在活跃态下把星标降为右下角 10px 金色角标并加轮廓阴影，空闲后恢复 16px 居中。

### 3.2 `dsh-context` 0.78.0

- **Token Stats 环按内容种类重建**：System Prompt、Tool Schemas、User Messages、Injected Context、Skill Injections、Tool Calls、Reasoning、Answers 八类，废弃旧的 Output 切片；Reasoning 独立成片，Tool Calls 合并调用、参数与工具返回。
- **完整执行历史**：command 卡与统计命令栏读取整个会话日志，不再只看最近 400 次；向后分页加载带进度，失败时保留部分列表可重试。
- **跳转与表面统一**：整个 Context Events 磁贴可悬停并跳转，turn/step 数字跳到 Context Trend 卡；Fleet 的 Agent Network、成员卡与任务卡共享表面、色条与悬停处理，聚焦某成员时淡化无关卡片。
- **性能（同机 headless Chrome 实测）**：Context 页 12k 事件 / 987 步从 414ms 降到 218–225ms（−47%）；长任务 133ms → 91–99ms（−28%）；堆内存 54.9MB → 50.1MB；CLS 从 0.071 降到 0；单会话行推导 12.7ms → 0.3–0.4ms；日聚合 13.4ms → 约 0；Fleet 请求并发由 9 降到最多 4。
- **状态版本**：`contextTimeline` 的 `stateVersion` 为 34，低于宿主要求时走降级 stub；本机宿主满足门槛，走正常注册路径。

### 3.3 `dsh-ops` 0.4.2、0.4.3、0.4.4

- **0.4.2**：组件模型改为预设模式管理，新增 Minimal / Standard / PTC / Cordis 四个模式组件与兼容模式开关，替换旧的 `shell`、`file`、`ptc`、`hide-official` 四行。Standard 与 Cordis 模式自动用 `ops_*` 替换官方读取与搜索工具，Windows 上还用 `ops_bash` 替换官方 shell。另新增 macOS 13+ Intel 原生运行时、`grep` 的 `exactTotal: false` 有界前缀查询、`dsh-ops review snapshot` 命令，Bash 输出支持宿主终端卡片，并修复 PTC 呈现、Windows 多受限 Bash 根进程访问、Linux bwrap 私有 `/tmp` 中 `ref://` 分页等问题。许可证统一 MIT。
- **0.4.3**：把宿主适配收敛到自有 `adapter/dsh` 边界，宿主兼容声明扩到 `>=0.2.0-rc.2 <0.2.2-0`，修 scope 动态加载在服务卸载/重载场景的生命周期竞态。
- **0.4.4**：修 0.4.3 在 npm 上误带旧版 Linux x64 与 macOS worker 的打包问题，并补 Linux/macOS worker 的可执行权限。**对 Windows 无实质行为变化，属打包补丁。**
- **升级注意**：0.4.3 的 CHANGELOG 明确要求删除 profile 里指向 `dsh-ops-shell`、`dsh-ops-file`、`dsh-ops-ptc`、`dsh-ops-hide-official` 的旧覆盖行；旧模块仍作为 legacy 导出保留，但已不在 bundle 内，留着只会每次启动打一条找不到行的警告。

### 3.4 `dshmarket` 1.70.3 至 1.73.3

| 版本 | 主要变更 |
|---|---|
| `1.70.3` | 支持 npm 上已改名的插件迁移；撤掉悬停预览、收回两列卡片密度；修 hot-mount YAML 中撇号转义 |
| `1.71.0` | 桌面端更新到最高的成熟 npm 版本 |
| `1.72.0` | 新增日语 locale；补缺失的状态键；修 HTTP 接受 Chrome 151+ 无端口 loopback Origin |
| `1.73.0` | 修仅有 GitHub 来源的插件仍显示宿主版本要求 |
| `1.73.1` | 修失败更新丢失旁观者 bundle 行；迁移事务补身份证据，npm 改名两侧注册行不相交时回滚并返回 409 |
| `1.73.2` | 不再给 `!!js` 条件行写字面量覆盖，条件行交回表达式裁决；最近收录改为一行横滑条带 |
| `1.73.3` | 无变更说明（空发布，仅附完整变更日志链接） |

### 3.5 `dsh-better-sidebar` 0.28.2

未取到该版本的发布说明：GitHub Releases 接口返回的内容不属于该版本范围，npm 元数据里也没有 changelog 字段。可确认的只有元数据——`engines.node >= 20`，peer 依赖全部对齐到 `^0.2.1-rc.1` 线，包解压后 19,302,404 字节、477 个文件。因此本版本只能判定为跟随 0.2.1 线的小幅补丁，**具体修了什么未验证**。

---

## 4. 升级前的四个阻塞项与处置

### 4.1 市场目录源不可用

插件市场的区域设置文件 `%USERPROFILE%\.dsh\profiles\desktop\.dsh-market\state.json` 为 `{"disabled":[],"groups":{},"groupOrder":[],"region":"china","regionAuto":true}`，即走国内区，目录源是腾讯云镜像上的 `dsh-plugin-catalog` 包。日志尾部连续多条 `catalog fetch failed`、`HTTP 522`/`525`、`no new data for 15s while downloading the catalog`。

处置：**不改市场设置，也不改任何 registry 配置**。本轮升级只用 `dsh.cmd` 显式指定版本安装，`%USERPROFILE%\.npmrc` 与 profile 下的 `.npmrc` 均不存在，pnpm 默认走 `registry.npmjs.org`，绕开了市场的目录抓取链路。市场本身能否恢复与本次升级无关，见第 8 章。

### 4.2 本机无独立 pnpm

本机没有可用的独立 `pnpm`：`which pnpm` 无结果，`%LOCALAPPDATA%\pnpm` 下只有 `store` 目录，`corepack pnpm --version` 因目标目录不存在报 `EPERM`。

处置：改用 DSH 自带的 CLI carrier。在 profile 目录下执行，已实测输出 `11.28.5`：

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
dsh.cmd plugin --profile desktop --version
```

### 4.3 CLI 侧锁文件被沙箱拦

`dsh.cmd plugin --profile desktop ls` 报 `EPERM: operation not permitted, open '...\package.json.lock'`，栈顶是 `@deepseek-ai/dsh-atomic-write` 的 `withFileLock`；`dsh.cmd --profile desktop --dump-config` 另被拒为 `profile "desktop" is managed exclusively by the Electron application`。

处置：`plugin add` 本条命令执行成功（exit code 0），但 `plugin ls` 在会话沙箱策略下仍报同一个 `EPERM`。因此**升级后的核验不再依赖 CLI**，改走文件、组合模拟与运行时痕迹三类证据，见第 7 章。

### 4.4 新版本仍在发布等待期

profile 的 `pnpm-workspace.yaml` 未设置 `minimumReleaseAge`，按 pnpm 默认取 1440 分钟。按 npm registry 的 `time` 字段核对发布时间（UTC）：

| 包 | 版本 | 发布时间 | 距升级时刻 |
|---|---|---|---|
| `@michengai/dsh-archive-manager` | `1.1.18` | 2026-10-27T07:38:39Z | 已过窗口 |
| `dsh-better-sidebar` | `0.28.2` | 2026-10-27T15:14:31Z | 已过窗口 |
| `dsh-ops` | `0.4.4` | 2026-10-28T02:44:21Z | 刚过窗口 |
| `dsh-context` | `0.78.0` | 2026-10-28T15:23:19Z | 未过窗口 |
| `dshmarket` | `1.73.3` | 2026-10-28T20:39:01Z | 未过窗口 |

未过窗口的版本一旦写进 lockfile，pnpm 校验整个 lockfile 会连带拦下所有插件的安装与更新。处置：按用户明确选择，在 `minimumReleaseAgeExclude` 里为这 5 个新版本全部加豁免（其中只有 2 个当时确实未过窗口，另外 3 个也已接近窗口边界，一并豁免以免后续单独升级时再踩），**放弃这一轮的等待期供应链保护**。写入时注意 pnpm 只读同名条目的第一条，必须追加到已有行的行尾而不是新增重复键。

---

## 5. 操作步骤

按顺序执行。每步都给了判据：判据不成立就先停下来，不要跳到下一步。全部命令在 Windows 下执行，路径一律写成 `%USERPROFILE%` 占位符。

### 5.1 取证：确认宿主版本与落后版本

先确定三件事：宿主 `@deepseek-ai/dsh` 的版本、当前 profile 有哪几个依赖、各自与 npm 上的 `latest` 差多少。

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
call dsh.cmd --version
type package.json
```

把 `dependencies` 的每一项版本与 npm registry 的 `latest` 逐项对比，列出「落后」与「已最新」两组，落成表格再动手。

判据：宿主版本要落在各插件 `engines.dsh` 声明的区间内（如 `dsh-ops` 是 `>=0.2.0-rc.2 <0.2.2-0`）；区间外不要升。

### 5.2 备份整个声明文件集

不要只备份 `package.json`。真正难撤的是 `pnpm-workspace.yaml` 的等待期白名单和 `cordis.patch.yml` 的组件行。

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
set "BK=.plugin-manager\backup-20261029-114932"
mkdir "%BK%"
copy /y package.json "%BK%" >nul
copy /y pnpm-lock.yaml "%BK%" >nul
copy /y pnpm-workspace.yaml "%BK%" >nul
copy /y cordis.patch.yml "%BK%" >nul
copy /y compatibility.json "%BK%" >nul
xcopy /y /e /i ".dsh-market" "%BK%\.dsh-market" >nul
```

把 `BK` 的值换成自己的时间戳。这组命令不能直接粘进 `cmd //c "..."` 这种一行形态——`cmd //c` 会把整串交给一次解析，`set` 设的变量在同一个 `&&` 链里取不到值；要么逐行粘进交互式 cmd 窗口，要么写成 `.cmd` 文件再调用（本轮验证用的就是后者）。

判据：备份目录里的文件大小与源文件一致。

### 5.3 确认 CLI 可用，不要先装 pnpm

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
call dsh.cmd plugin --profile desktop --version
```

输出 pnpm 版本号（本轮为 `11.28.5`）就说明 carrier 可用，**本机没有独立 pnpm 也不需要装**。

判据：有版本号输出。若报错再考虑装 pnpm，不要跳过这一步直接装。

在 `%USERPROFILE%\.dsh\profiles\desktop` 目录下执行时，报错可能是 `EPERM: operation not permitted, open '...\package.json.lock'`（`@deepseek-ai/dsh-atomic-write` 的 `withFileLock`）。这不是 CLI 坏了：`plugin add` 在本轮同一环境下能成功，只有读锁的 `plugin ls` / 读配置的命令被拦。换到不受限的普通 cmd 窗口即可；受限环境下跳过 CLI 核验，直接走第 5.6 节的三层证据。

### 5.4 处理发布等待期

先查每个目标版本的发布时间（npm registry 的 `time` 字段），与当前时间比较：

- 距发布已超过 `minimumReleaseAge`（未配置时为默认 1440 分钟）→ 无需处理。
- 未超过 → 必须写豁免，否则该版本进 lockfile 后 pnpm 校验会连带拦下**全部**插件安装与更新。

豁免写进 `pnpm-workspace.yaml` 的 `minimumReleaseAgeExclude`：

```yaml
minimumReleaseAgeExclude:
  - 已有条目... || 新版本号
  - '新包名@新版本号'
```

两条硬约束：**追加到同名条目的行尾**，不要新增重复键（pnpm 只读第一条，重复键会被静默忽略）；包里带 `@` 作用域时整条要用单引号包住。

判据：`grep` 该文件，每个待升级的包只有一个同名条目。

### 5.5 执行升级

一条命令装完所有指定版本。这是本轮实际执行的形态（bash 转调 cmd，后台 job，退出码 0）：

```bash
cd "C:/Users/<用户名>/.dsh/profiles/desktop" && cmd //c "dsh.cmd plugin --profile desktop add @michengai/dsh-archive-manager@1.1.18 dsh-better-sidebar@0.28.2 dsh-context@0.78.0 dsh-ops@0.4.4 dshmarket@1.73.3" 2>&1 | tail -60
```

`cmd //c "..."` 内的那段才是交给 cmd 的完整命令，外层是 bash 侧的目录切换与输出截断。原生 cmd 等价写法：

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
call dsh.cmd plugin --profile desktop add @michengai/dsh-archive-manager@1.1.18 dsh-better-sidebar@0.28.2 dsh-context@0.78.0 dsh-ops@0.4.4 dshmarket@1.73.3
```

版本写成显式 `@版本` 即精确写入 `dependencies`，不需要 `--save-exact`。

判据：退出码 0，且输出里有 `Lockfile passes supply-chain policies` 一行。本轮该行是 `✓ Lockfile passes supply-chain policies (278 entries in 9.5s)`，另有 `Packages: +10 -2`、`Done in 26.7s using pnpm v11.28.5`。

**先完全退出 DSH 再执行更稳**（不是关窗口，是退出进程）。本轮是在运行中直接升级的：实例 11:27:55 启动、升级 11:49 完成，靠之后重启加载新版，结果正确但属于事后生效；运行中替换 `node_modules` 会让已加载模块与磁盘不一致，对卸载类操作还会直接报 `refused while agents are running`。口径与理由见 [[12DSH插件更新问题与解决]] 第 7.4 节。

注意 `dsh.cmd` 是批处理，参数里的 `&` 会被当作命令分隔符执行、`%` 会被当变量展开；遇到这类参数改用环境变量传递，换写 `dsh` 或 `dsh.cmd` 都躲不过。

### 5.6 核验（不要用 `plugin ls`）

`plugin ls` 要写 `package.json.lock`，在受限沙箱下必被拦（`EPERM ... withFileLock`）；`--dump-config` 对 Electron 管理的 profile 直接拒绝。改用三层证据：

1. **文件层**：`package.json` 的 `dependencies` 版本、`node_modules/<包>/package.json` 的 `version`、`node_modules/.modules.yaml` 的 `hoistedLocations`，三处应一致。
2. **组合层**：按第 7.3 节的方法模拟补丁组合，确认新组件行确实进了 entry 树。
3. **运行时层**：**重启 DSH** 后看请求头的工具集。这是唯一能证明「插件真的在跑」的硬证据。

### 5.7 `dsh-ops` 类组件的行迁移

`dsh-ops` 从 0.4.2 起把组件改成预设模式，旧行会失效。步骤：

1. 读 `node_modules/dsh-ops/cordis.patch.yml`，列出它现在插入的所有 id。
2. 读 profile 的 `cordis.patch.yml`，找出不在新列表里的旧 id（本轮是 `dsh-ops-shell`、`dsh-ops-file`）。
3. 对照 `node_modules/dsh-ops/docs/CHANGELOG.md` 与 `docs/component-reference.md` 确认这些 id 已被标记为 legacy。
4. 删除前先做组合模拟，比较删除前后组合出的完整 entry 树——**等价才删**。
5. 删除，备份原文件（本轮备份为 `cordis.patch.yml.bak-<日期>`）。

判据：删除前后 entry 树 `JSON.stringify` 完全相同，且警告条数从 N 降到 0。宿主对补丁的容错是「警告并跳过」，所以旧行不会让启动失败，它只是每次启动都打一条 `patch: entry ... not found`。

### 5.8 重启确认

重启 DSH，检查两件事：启动不再有补丁警告；请求头的工具集与预期一致（本轮 `dsh-ops` standard 模式把官方 `read`/`glob`/`grep`/`pwsh` 隐藏、换成 `ops_*`，工具总数 59）。

纯 UI 插件（`dsh-better-sidebar`、`dsh-archive-manager`、`dshmarket`）没有宿主侧痕迹，只能打开界面肉眼确认，见第 8 章第 1 条。

---

## 6. 实际改动清单

改动全部集中在 profile 目录，共三个文件、一个备份目录。

### 6.1 `pnpm-workspace.yaml`

五处改动，前四处追加到已有行的行尾，第五处是文件第 27 行新增的一条：

```yaml
minimumReleaseAgeExclude:
  # 第 7 行行尾追加
  - dshmarket@1.66.6 || ... || 1.70.2 || 1.73.3
  # 第 12 行行尾追加
  - dsh-context@0.66.1 || ... || 0.78.0
  # 第 21 行
  - dsh-ops@0.3.0 || 0.4.0 || 0.4.1 || 0.4.4
  # 第 26 行
  - '@michengai/dsh-archive-manager@1.1.16 || 1.1.18'
  # 第 27 行新增
  - 'dsh-better-sidebar@0.28.2'
```

改后全文 30 行，`minimumReleaseAgeExclude` 共 21 条，另保留 `allowBuilds` 的 `'@google/genai'` 与 `protobufjs` 两项。

### 6.2 `package.json`

`dependencies` 五项版本更新为 `1.1.18`、`0.28.2`、`0.78.0`、`0.4.4`、`1.73.3`，全部写成精确版本（`--save-exact` 语义）。`dsh.profile.bundles` 的 11 项顺序未变。

### 6.3 `cordis.patch.yml`

删除两条指向旧组件名的覆盖行，共 14 行：

| 位置 | 删掉的 id | 原 `name` | 原 config |
|---|---|---|---|
| 第 98–105 行 | `dsh-ops-shell` | `dsh-ops/shell` | `promptPolicy: true`、`publishBashTool: true`、`shellPolicy: advise`、`allowSystemShellFallback: false`、`configUpdateVersion: 2` |
| 第 106–111 行 | `dsh-ops-file` | `dsh-ops/file` | `promptPolicy: true`、`required: false`、`configUpdateVersion: 2` |

- 文件从 709 行 / 24147 字节变为 695 行 / 23839 字节（差 308 字节即删除的 14 行）。
- 顶层补丁块从 12 条减为 10 条，存活条目内容与顺序逐一不变。
- 编码保持无 BOM、LF、末尾有换行。
- 这 5 项显式配置与 0.4.4 的默认值逐项相同（`shellPolicy` 的默认就是 `advise`），因此删除不改变任何有效行为。

### 6.4 备份

升级前的声明文件整目录备份到 `%USERPROFILE%\.dsh\profiles\desktop\.plugin-manager\backup-20261029-114932\`，含 `package.json`、`pnpm-lock.yaml`、`pnpm-workspace.yaml`、`cordis.patch.yml`、`compatibility.json` 与 `dsh-market/` 整个目录。

改 `cordis.patch.yml` 前另存单文件备份 `cordis.patch.yml.bak-20261029-1215`（24147 字节，SHA-256 与原件一致）。

---

## 7. 验证结果

### 7.1 安装与锁定

升级命令结束后 pnpm 报告 `✓ Lockfile passes supply-chain policies (278 entries in 9.5s)`、`Packages: +10 -2`、`Done in 26.7s using pnpm v11.28.5`，退出码 0。有 `Issues with peer dependencies found` 警告但未展开，见第 8 章。

`package.json` 的 9 项依赖与 `node_modules/<包>/package.json` 的实际版本逐一核对一致，`node_modules/.modules.yaml` 的 `hoistedLocations` 也已记录为新版本。

### 7.2 运行时构成

重启后（进程启动时间 12:17:06）以三类痕迹确认：

| 插件 | 运行时证据 | 结论 |
|---|---|---|
| `dsh-ops` 0.4.4 | 请求头工具集为 59 个，`ops_bash` 存在而官方 `read`/`pwsh` 已消失 | 生效，且走的是 standard 模式 |
| `dsh-context` 0.78.0 | 会话投影 `contextTimeline` 的 `ver` 为 34，`contextBreakdown` 为 6 | 生效，走正常注册而非降级 stub |
| `dsh-connect-workbuddy` 5.13.1 | `%USERPROFILE%\.dsh\.workbuddy-host-heartbeat.json` 记录 `pluginVersion: 5.13.1`，`registeredAt` 在重启后 | 生效 |
| `dsh-find-plugins` 0.3.0 | `find_dsh_plugins` 在工具集内 | 生效 |
| `billion-context` 0.1.209 | `compress`、`decompress`、`search_context`、`acp_status`、`acp_cache` 均在工具集内 | 生效 |
| `dsh-mnemon` 0.5.36 | 22 个 `mnemon_*` 工具在工具集内 | 生效 |
| `dsh-better-sidebar` 0.28.2 | 无宿主侧痕迹 | 未验证 |
| `@michengai/dsh-archive-manager` 1.1.18 | 无宿主侧痕迹 | 未验证 |
| `dshmarket` 1.73.3 | 无宿主侧痕迹 | 未验证 |

`dsh-ops` 那条是**升级后行为变化的唯一硬证据**：升级前四次请求头都是 53 个或 59 个工具且 `read`/`pwsh` 为 true，重启后变为 59 个工具、`read`/`pwsh` 为 false。这与 standard 模式的替换表（隐藏官方 `read`、`glob`、`grep`、`pwsh`、`pwsh_persistent`）完全吻合，说明新组件行确实在发布 `ops_*` 工具，旧行删除也没有把 shell 一起弄丢。

### 7.3 补丁行删除等价性

宿主不把补丁警告落盘，只能由组合模拟证明。做法是从 `app.asar` 中提取宿主的 `applyEntryPatches` 并逐行复刻，按 `dsh.profile.bundles` 顺序叠加 11 个 bundle 的补丁层，再叠加 profile 用户层，比较删除前后两版：

| 项 | 删除前 | 删除后 |
|---|---|---|
| 用户层补丁块 | 12 | 10 |
| 组合后顶层 entry | 208 | 208 |
| 警告条数 | 2 | 0 |
| 完整 entry 树 | — | 与删除前 `JSON.stringify` 完全相同 |

模拟告警原文为 `patch: entry %C not found` 加 id，与宿主的警告模板一致。宿主的应用器在找不到 id 时只警告并跳过，不会中断启动，所以旧行本来就是「必然落空的无害补丁」；删除只去掉这两条落空动作，不改变任何挂载。

复刻时最容易写错的两处（漏掉就会得到假的 ID 未找到告警，把结论带偏）：

1. **id 索引要递归进组**。宿主的 `buildMap` 在遍历时对 `entry.group && Array.isArray(entry.config)` 的条目继续递归，所以组内条目的 id 也能被补丁命中；只索引顶层会误判一批 id 不存在。
2. **`insert` 带 `id` 时改的是 `target.config`，不是 `target.insert`**；且目标必须是组（`target.group`），否则报 `patch insert: entry %C is not a group` 并跳过。插入后还要对 `insert` 再跑一次 `buildMap`，同一补丁列表里后续的补丁才能命中刚插入的行。

另一个坑在解析层：`!!js` 标签要用 `yaml.JSON_SCHEMA.extend(new yaml.Type('tag:yaml.org,2002:js', {...}))`，用 `DEFAULT_SCHEMA` 会因为布尔/日期等隐式类型规则不同而改变解析结果。

---

## 8. 未验证项

1. **三个纯 UI 插件没有运行时直接证据。** `dsh-better-sidebar`、`@michengai/dsh-archive-manager`、`dshmarket` 的界面逻辑都在浏览器端 bundle 里，宿主侧只留 `dsh.client` 声明、不产生工具。尝试过的事实是：Web GUI 地址全站返回 401（`dsh web authentication required`），`/client/bundle.js` 等路径 404，`.dsh` 与临时目录下没有 client bundle 的构建产物，Electron 存储里也搜不到插件名。**能确认的只是它们的 patch 行在组合模拟中已正确进入 208 条 entry；界面是否真的加载需要打开应用肉眼确认。**
2. **peer 依赖警告未展开。** 升级输出的 `Issues with peer dependencies found` 没有逐条查看，`pnpm peers check` 未运行。
3. **`dsh-better-sidebar` 0.28.2 的实际改动未知。** 发布说明未取到，只能凭 peer 依赖版本推断为跟随 0.2.1 线的补丁。
4. **市场目录源仍然不可用。** 腾讯云镜像上的 `dsh-plugin-catalog` 持续超时，市场内的浏览与搜索功能受影响；本轮未处理，也没有修改 market 的区域设置。
5. **回滚的第二步未实测。** 第 9 章给出拷回声明文件的命令，但「拷回后重装依赖让 `node_modules` 回到旧版本」这一步本轮没有跑过。
6. **文中所有 `app.asar` 偏移、行号与宿主任意实现细节只对 `@deepseek-ai/dsh` = `0.2.1-rc.3` 成立**，宿主升级后会失效。
7. **组合模拟脚本放在临时目录，已被清理。** 需要复现时按第 7.3 节的方法重写，并注意该节列出的递归索引、`insert` 语义与 `!!js` schema 三个易错点。

---

## 9. 回滚

分两种情况，都在 Windows `cmd` 下执行。回滚前建议先确认当前没有正在运行的 DSH 会话。

### 9.1 只撤销补丁行删除

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
copy /y "cordis.patch.yml.bak-20261029-1215" "cordis.patch.yml"
```

恢复后文件回到 709 行，启动警告也会回来；不影响功能。

### 9.2 整体退回升级前状态

```cmd
cd /d "%USERPROFILE%\.dsh\profiles\desktop"
xcopy /y /e /i ".plugin-manager\backup-20261029-114932\*" "."
```

拷回的是 `package.json`、`pnpm-lock.yaml`、`pnpm-workspace.yaml`、`cordis.patch.yml`、`compatibility.json` 与 `dsh-market/`。声明文件恢复不等于依赖恢复，还需要按锁文件重装依赖；**这一步本轮未实测**，写法与升级时同形，把版本换成旧值即可。

风险提示：`xcopy` 会覆盖同名文件，执行前先确认 `backup-20261029-114932` 目录存在且完整。

---

## 10. 值得留的经验

- **升级前先备份整个 profile 的声明文件，而不只是 `package.json`。** 本次真正难撤的改动在 `pnpm-workspace.yaml` 的等待期白名单和 `cordis.patch.yml` 的组件行，单备份 `package.json` 回不去。
- **`dsh.cmd` 自带 pnpm carrier，不必先解决本机 pnpm 缺失。** 先跑 `dsh.cmd plugin --profile desktop --version` 确认版本，再决定要不要装 pnpm。
- **等待期豁免必须追加到同名首行。** pnpm 只读 `minimumReleaseAgeExclude` 里的第一条同名条目，新增重复键的部分会被静默忽略。
- **不要用 `plugin ls` 做升级后核验。** 它要写 `package.json.lock`，在受限沙箱里必被拦；`--dump-config` 又对 Electron 管理的 profile 直接拒绝。可用的替代证据是请求头工具集、组件定义状态版本和心跳文件。
- **宿主对补丁的容错是「警告并跳过」。** 所以旧组件行不会让启动失败，只会在每次启动打一条警告；判断该不该删，看的是组合后的 entry 树是否等价，而不是警告有没有出现。
- **纯 UI 插件只能靠界面验证。** 宿主侧没有它们的运行痕迹，任何「已生效」的结论都要么来自组合模拟（只能证明 patch 行进了 entry 树），要么来自肉眼确认。
- **改受限目录里的脚本不要用 `sed -i`。** 临时目录下的原地编辑会被沙箱拒绝并给出 `Permission denied`，改用编辑工具或整文件重写。

---

## 11. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-29 | 1.0.0 | 新建。记录 5 个插件的批量升级、5 条等待期豁免、`dsh-ops` 0.4.x 新旧组件行迁移、`cordis.patch.yml` 14 行删除与三类运行时核验，附回滚与未验证项。素材来自 2026-10-29 的 DSH 插件升级会话。 |
| 2026-10-29 | 1.0.1 | 订正两处：等待期实际只有 2 个版本未过窗口（原写 3 个）；`dsh-ops-shell` 块为第 98–105 行（原写 98–104 行）。§5 补上 `dsh.cmd plugin add` 升级命令、执行环境与执行目录，明确升级走 DSH 自带 CLI 而非系统 pnpm。 |
| 2026-10-29 | 1.0.2 | 补 §5 的实际执行形态：给出会话里真正跑过的整条 bash 命令（`cd ... && cmd //c "dsh.cmd plugin ... add ..."`），并说明 `cmd //c` 内那段才是交给 cmd 的完整命令，原生 cmd 写法另列。用户名按 `note/05Markdown规范.md` 第 9 章改写占位符。 |
| 2026-10-29 | 1.1.0 | 新增第 5 章「操作步骤」：按取证 → 备份 → 确认 CLI → 处理等待期 → 执行升级 → 核验 → 组件行迁移 → 重启确认的顺序给出 8 个可照做的步骤，每步带命令与判据，并补 `dsh` 与 `dsh.cmd` 的调用差异与批处理参数陷阱。原第 5–9 章顺延为第 6–10 章，章内交叉引用同步更新。 |
| 2026-10-29 | 1.1.1 | 第 5.3 节补受限环境下的 `EPERM ... package.json.lock` 说明；第 7.3 节补复刻宿主补丁应用器的三个易错点（id 索引必须递归进组、`insert` 改 `target.config` 且目标须为组、`!!js` 用 `JSON_SCHEMA.extend`）。组合模拟已重跑，新版 0 条告警、旧版 2 条、entry 树完全相同。 |
| 2026-10-29 | 1.1.2 | 第 5.5 节补一句「先完全退出 DSH 再执行更稳」，对齐 [[12DSH插件更新问题与解决]] 第 7.4 节的口径，并注明本轮实为运行中升级、靠重启生效。 |

---
