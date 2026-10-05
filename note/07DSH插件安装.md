# DSH 插件安装方法与步骤

> 文档名称：DeepSeek Harness（DSH）插件安装方法与步骤
> 操作系统：Windows
> 适用范围：DSH Desktop 图形界面安装、DSH 命令行安装、插件兼容性排查与回滚
> 实测宿主：DSH Desktop，内置运行时 `@deepseek-ai/dsh` = `0.2.0-rc.2`
> DSH 安装目录：`D:\Software\sf1\DeepSeek Harness`
> DSH_HOME：`%USERPROFILE%\.dsh`
> 当前 profile：`desktop`
> 内置 CLI：`D:\Software\sf1\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd`
> 内置 pnpm：`D:\Software\sf1\DeepSeek Harness\resources\runtime\pnpm\bin\pnpm.cjs`
> 环境提示：本机 `dsh` 与 `pnpm` 都不在系统 PATH 上，二者都由 DSH Desktop 自带，命令行安装必须用完整路径调用
> 最近更新：2026-10-03

**目录**

- [[#1. 先理解一个概念：profile]]
- [[#2. 方法一：DSH Desktop 图形界面安装（桌面用户首选）]]
- [[#3. 方法二：命令行安装]]
- [[#4. 兼容性闸门：决定插件能不能被加载]]
- [[#5. 常见错误速查]]
- [[#6. 安装前后的安全操作]]
- [[#7. 一次完整实测记录（可复现）]]
- [[#8. 一页速查]]
- [[#9. 变更记录]]

## 1. 先理解一个概念：profile

DSH 用 **profile** 来组合一套运行界面。插件是装进某个 profile 里的，装错 profile 等于没装。

- profile 目录：`$DSH_HOME\profiles\<profile 名>`
- 随附模板：`web`、`headless`、`acp`、`sdk`、`sdk-minimal`
- **`desktop` 是 Electron 保留的 profile**：由 `loadProfileDirectory` 加载，不参与常规 CLI profile 查找；对它执行 `dsh plugin` 前必须先**完全退出 Desktop**
- 一个 profile 的两处关键声明（`package.json`）：

```json
{
  "name": "dsh-profile-desktop",
  "private": true,
  "dependencies": { "dsh-cache-billing": "^0.1.4" },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "@deepseek-ai/dsh-web-app",
        "dsh-cache-billing"
      ]
    }
  }
}
```

- `dependencies`：实际参与安装的包
- `dsh.profile.bundles`：启动时按顺序叠加的层，**插件必须同时出现在这里才会被加载**
- `cordis.patch.yml`：你的覆盖层，用来改配置/禁用/插配置项

---

## 2. 方法一：DSH Desktop 图形界面安装（桌面用户首选）

这是最不容易出错的方式，也不用退出 Desktop。

1. 打开 **设置 → 插件（Plugins） → 插件管理／安装**
2. 在「**包名或地址**」输入框里**只填包名**，例如：

```text
dshmarket@1.66.6
```

3. 按提示安装，必要时重启 DSH
4. 回到界面确认：插件出现在已装列表，版本号与预期一致

### 2.1 输入框到底接受什么

DSH 用同一个校验器判断这个输入，**只接受这几种形式**：

| 形式 | 例子 |
| --- | --- |
| 包名 | `dshmarket` |
| 包名@版本 | `dshmarket@1.66.6` |
| scoped 包名 | `@scope/pkg` 或 `@scope/pkg@1.0.0` |
| git 仓库地址 | `git+https://github.com/用户/仓库.git` |
| 压缩包地址 | `https://.../pkg.tgz` |
| 本地绝对路径 | `D:\code\my-plugin`（必须是绝对路径） |

包名必须匹配（最长 214 字符）：

```text
/^(?:@[a-z0-9][a-z0-9._~-]*\/)?[a-z0-9][a-z0-9._~-]*$/
```

### 2.2 最容易踩的坑：把整条命令行贴进输入框

如果在输入框里填成：

```text
dsh plugin --profile web add dshmarket
```

就会报：

```text
无法识别这个包名或地址：not a package name the registry accepts
```

原因：输入里含空格，不匹配上面的包名正则，直接落到「not a package name the registry accepts」这一支。同一校验器的其它拒绝原因也可以帮你判断：

```text
the package spec must not be empty
a local path must be absolute
a URL must point at a git repository or a tarball
not a package name the registry accepts
a version after @ must not be empty
```

**结论：输入框要的是“包名”，不是“命令”。** 命令行语法属于 §1.4。

另外，界面里不需要（也不能）指定 `--profile`：它装的就是这个 App 正在使用的 profile（`desktop`）。

### 2.3 第二个坑：裸名可能装到旧版本

pnpm 有**新版本冷冻期**（fresh-release hold）机制。实测现象：

| 输入 | 实际结果 |
| --- | --- |
| `dshmarket` | 装到 **1.66.5**，`package.json` 记为 `^1.66.5` |
| `dshmarket@1.66.6` | 装到 **1.66.6**，并自动在 `pnpm-workspace.yaml` 追加 `minimumReleaseAgeExclude: dshmarket@1.66.6` |

当时 1.66.6 才发布约 **15.7 小时**，未过冷冻窗口，于是裸名被解析降级到 1.66.5。

**建议：想装最新版就显式写版本号。**

---

## 3. 方法二：命令行安装

### 3.1 基本语法

```powershell
dsh plugin --profile <profile 名> add <包名或地址>
```

`dsh plugin` 会把参数**原样转发给 profile 目录里的 pnpm**，所以 `add`、`remove`、`why` 等 pnpm 子命令都可用。

### 3.2 完整路径调用（本机必须）

```powershell
$dsh = 'D:\Software\sf1\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd'

# 版本自检
& $dsh -V

# 装到 desktop profile（必须先完全退出 DSH Desktop）
& $dsh plugin --profile desktop add dshmarket@1.66.6
```

### 3.3 装到独立 profile（不碰现有环境）

```powershell
$dsh = 'D:\Software\sf1\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd'

# 从随附模板创建新 profile，并只打印组合树（不启动服务）
& $dsh --profile my-trial --from-default-profile web --dump-config

# 安装插件
& $dsh plugin --profile my-trial add dshmarket@1.66.6

# 启动该 profile 验证（另开端口，不影响正在使用的实例）
& $dsh --profile my-trial --no-open --port 31987
```

**更彻底的隔离**：给子进程单独指定 `DSH_HOME`，与主环境完全分开。

```powershell
$env:DSH_HOME = 'D:\User\Documents\deepseek-harness\default-workspace\dsh-sandbox-home'
```

> 注意：`$env:DSH_HOME` 只在该 PowerShell 进程内生效，不写入系统环境变量、不改 shell rc。
> 另：`market-trial` 这类从 web 模板建的 profile **本身就是 web 应用**，启动时不要再传 `web`（会报 `too many arguments`）。

### 3.4 其它有用的命令

```powershell
# 查看组合后的配置树（不启动）
& $dsh --profile <名字> --dump-config

# 只看默认层（排除你的 patch）
& $dsh --profile <名字> --dump-default-config

# 查看 peer 依赖问题
& $dsh plugin --profile <名字> peers check
```

---

## 4. 兼容性闸门：决定插件能不能被加载

**这是安装 DSH 插件最关键的一环。** 装得上 ≠ 能加载。

DSH 启动时由 `@deepseek-ai/dsh-app-boot` 的 `evaluatePluginCompatibility` 判定：对插件 `peerDependencies` 中所有名字匹配 `@deepseek-ai/dsh` 或 `@deepseek-ai/dsh-*` 的项，执行

```js
semver.satisfies(运行时版本, 声明范围, { includePrerelease: true })
```

**不满足时不会报错，而是静默跳过该层**，只在 stderr 留一行，例如：

```text
dsh: skipping profile bundle "dshmarket": Plugin dshmarket@1.66.3 is incompatible
with dsh 0.2.0-rc.2: peerDependencies {...}
```

如果插件界面里没出现，先去看启动日志有没有这行。

### 4.1 预发布版本的陷阱

对带预发布号的 caret 范围，node-semver 的上界会补 `-0`：

```text
^0.1.0-rc.7   ->   >=0.1.0-rc.7 <0.2.0-0
```

`0.2.0-rc.1` 大于 `0.2.0-0`，所以**「只写 0.1.x」的插件在 0.2 宿主上一定不满足**，多加几段 OR 也救不回来。跨 minor 升级时这是最常见的“插件突然消失”原因。

### 4.2 确认宿主版本

```powershell
& $dsh -V
```

或直接在安装包里检索（本机实测得到 `0.2.0-rc.2`）：

```powershell
Select-String -Path 'D:\Software\sf1\DeepSeek Harness\resources\app.asar' `
  -Pattern '"@deepseek-ai/dsh": "[^"]+"'
```

### 4.3 不满足时的处理（按优先级）

1. **升级插件到声明了新宿主线的版本**（首选）
2. **装回与宿主匹配的旧版本**：`& $dsh plugin --profile <名字> add <包名>@<旧版本>`
3. **加兼容豁免**（有风险，等于自行承担不兼容后果）：

```powershell
& $dsh plugin --profile <名字> allow-version <包名>@<版本> --dsh-version 0.2.0-rc.2 --accept-risk
& $dsh plugin --profile <名字> version-exemptions      # 查看已有豁免
& $dsh plugin --profile <名字> revoke-version <包名>@<版本> --dsh-version 0.2.0-rc.2
```

豁免会写进 `<profile>\compatibility.json`，格式：

```json
{
  "dsh-cache-billing@0.1.4": ["0.2.0-rc.2"]
}
```

> 该文件里已有的条目**不要删除或改写**，否则原本靠豁免才能加载的插件会失效。

---

## 5. 常见错误速查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `无法识别这个包名或地址：not a package name the registry accepts` | 输入框里填了整条命令行 / 含空格 / 非法字符 | 只填包名，如 `dshmarket@1.66.6` |
| `dsh: skipping profile bundle "X"` | peer 声明不覆盖当前宿主版本 | 升插件 / 降插件 / 加豁免（§1.5） |
| 提示 `incompatible-version`，并给出 `allow-version` 命令 | 兼容闸门拒绝安装 | 按提示执行 `allow-version`，或改用兼容版本 |
| `[WARN] Issues with peer dependencies found` / `missing peer @deepseek-ai/cordis` | in-box 包由宿主在启动时解析，不在 profile 内 | **通常无害**；以启动日志为准 |
| 裸名装到的版本比 `latest` 旧 | pnpm 新版本冷冻期 | 显式指定版本号 |
| `too many arguments. Expected 0 arguments but got 1: web` | 启动 profile 时多传了应用名 | 直接 `dsh --profile <名字> --no-open --port N` |
| `cannot resolve profile bundle ...` | profile 里的依赖没装 | `dsh plugin --profile <名字> install` |

---

## 6. 安装前后的安全操作

### 6.1 安装前：留快照

```powershell
$prof = '%USERPROFILE%\.dsh\profiles\desktop'
$out  = 'D:\backup\before-desktop.sha256.txt'
Get-ChildItem $prof -Recurse -File -Force |
  Where-Object { $_.FullName -notlike '*\node_modules\*' } |
  Get-FileHash -Algorithm SHA256 |
  ForEach-Object { "$($_.Hash)  $($_.Path)" } |
  Sort-Object | Set-Content $out -Encoding UTF8
```

装完再跑一次做对比，即可确认变更范围。

### 6.2 安装后：建议关闭「一键重启宿主」

在 profile 的 `cordis.patch.yml` 里追加（注意 `allowRestart` 必须在 `config:` 下，**不能与 `name:` 平级**）：

```yaml
- id: dsh-market
  name: dshmarket
  config:
    allowRestart: false
```

改完用 `--dump-config` 确认已生效（应出现 `# == dshmarket, patched by ...cordis.patch.yml`）。

### 6.3 回滚

1. 删除 `profiles\<名字>\node_modules\<包名>`
2. 从 `package.json` 的 `dependencies` 与 `dsh.profile.bundles` 中移除该包
3. 用安装前的 SHA256 快照逐个核对还原
4. 重启 DSH

若是整块独立 profile，直接删目录即可：`Remove-Item 'D:\...\profiles\<名字>' -Recurse -Force`

### 6.4 安装前三问（来源可信度）

- 仓库 owner 是组织还是个人？License 是什么？
- npm 包是否有 **provenance 签名**？发布者是否为 CI？
- README / Release Notes / Issues 是否活跃、是否有同名的仿冒仓库？

> 同名插件风险真实存在。安装前务必确认**包名 + 仓库 + 维护者**三者一致。

---

## 7. 一次完整实测记录（可复现）

以在隔离环境中安装 `dshmarket` 为例：

```powershell
$dsh = 'D:\Software\sf1\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd'
$env:DSH_HOME = 'D:\User\Documents\deepseek-harness\default-workspace\dsh-sandbox-home'

# 1. 建隔离 profile（只打印组合树，不启动）
& $dsh --profile market-trial --from-default-profile web --dump-config
#    -> exit=0，组合树约 94 KB

# 2. 安装（精确版本）
& $dsh plugin --profile market-trial add dshmarket@1.66.6
#    -> exit=0，"+ dshmarket 1.66.6"，pnpm v11.7.0，7.1s

# 3. 启动验证
& $dsh --profile market-trial --no-open --port 31987
#    -> dsh web: http://127.0.0.1:31987/?token=...
#    -> stderr 无 skipping profile bundle，即闸门通过
```

**验收要点**：

- 启动页 `__DSH_BOOT__.entries` 中应出现该插件，例如
  `{"id":"dshmarket","url":"plugins/??dshmarket/client.js&rev=...","inject":[...]}`
- 插件客户端 bundle 能取到（HTTP 200）
- 插件的 profile 目录下出现自己的状态目录（如 `.dsh-market\`）
- 原 `desktop` profile 的 SHA256 快照逐条一致、且检索不到该插件名

---

## 8. 一页速查

```text
图形界面安装：设置 → 插件 → 安装 → 输入「包名@版本」
命令行安装　：dsh plugin --profile <profile> add <包名@版本>
版本自检　　：dsh -V
组合树预览　：dsh --profile <名字> --dump-config
兼容性判据　：启动日志有没有 skipping profile bundle
豁免管理　　：allow-version / version-exemptions / revoke-version
回滚　　　　：删 node_modules\<包名> + 还原 package.json + 重启
```

**三条铁律**

1. 装到哪个 profile，就在哪个 profile 用；`web` 与 `desktop` 互不相通
2. 命令行操作 `desktop` profile 前，必须先**完全退出 DSH Desktop**
3. 安装前留 SHA256 快照，安装后核对；能 pin 版本就不要用裸名

---

## 9. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-01 | 1.0.0 | 初版：DSH 插件的图形界面与命令行两种安装方式、profile 概念、兼容性闸门与安装前后的安全操作 |
| 2026-10-03 | 1.0.1 | 补建变更记录章；本篇此前的内容改动未逐条回溯，版本号自本次起维护 |

---
