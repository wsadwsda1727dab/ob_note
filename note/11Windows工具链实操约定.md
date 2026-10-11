# Windows 工具链实操约定

> 文档名称：Windows 工具链实操约定
> 适用范围：本机 Windows + PowerShell 5.1 环境下写文件、跑脚本、传提交信息、读文本时的编码与执行约定，以及 DSH 沙箱的能力边界
> 适用对象：本库作者，以及在本机执行命令与提交的 AI
> 最近更新：2026-10-11
> 关联文档：[[00note总览]]、[[03Obsidian实操约定]]、[[09DSH工作区写入权限]]、[[10提交与文档同步规则]]、[[05Markdown规范]]

**目录**

- [[#1. 适用范围]]
- [[#2. PowerShell 5.1 的编码]]
- [[#3. 脚本的执行策略]]
- [[#4. 提交信息的编码传递]]
- [[#5. 搜索与文本工具]]
- [[#6. DSH 沙箱的能力边界]]
- [[#7. 变更记录]]

## 1. 适用范围

本机（Windows）的 DSH 会话里，`pwsh` 工具实际运行的是 **Windows PowerShell 5.1**（PSEdition Desktop），不是 PowerShell 7.x。本章记录由此产生的编码、执行与工具选择约定。

- 本章只讲**怎么操作**；机制层面的排查见 [[09DSH工作区写入权限]]。
- 提交的规则与流程见 [[10提交与文档同步规则]]，本章只讲其中与编码有关的操作细节。
- Obsidian 相关的编辑操作、排除文件与命令行限制见 [[03Obsidian实操约定]]，本章不重复。
- 带“已核实”的行是本机实测结论；标“推测”的行未实测。
- 这些约定只约束本机环境；换到 PowerShell 7.x 或 Linux 后多数条目不再适用。

---

## 2. PowerShell 5.1 的编码

### 2.1 新建文本文件一律 UTF-8 无 BOM

- 库内 Markdown 按 [[05Markdown规范]] 第 8.3 节必须是 UTF-8 无 BOM、LF 行尾。
- **注意**：PS 5.1 的 `-Encoding utf8` 写文件时**自带 BOM**。要写出无 BOM 的文件，用 `UTF8Encoding($false)`：

```powershell
$p = 'D:\path\to\file.md'
$enc = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($p, $text, $enc)
```

- 更简单可靠的做法是直接用 DSH 的文件写入工具（write / edit），它按 UTF-8 无 BOM 写入。已核实。

### 2.2 shell 重定向写出的日志是 UTF-16LE

- 用 `>`、`>>`、`Out-File` 抓取的输出会写成 **UTF-16LE + BOM**，DSH 的 read 工具会把它判定为 binary 而读不出内容。
- 需要把命令输出存成可读日志时，显式指定编码：

```powershell
Get-ChildItem | Out-File -FilePath .\listing.txt -Encoding utf8
```

- 已核实。

### 2.3 非 ASCII 脚本必须存为 UTF-8 带 BOM

- PS 5.1 读取**无 BOM** 的 `.ps1` 时按系统 ANSI 代码页解码。脚本里含中文注释或中文字符串时会变成乱码，并常以语法错误形式报出，很容易被误读成脚本本身写错了。
- 处理办法：**含非 ASCII 内容的 `.ps1` 一律存为 UTF-8 带 BOM**。
- 对照验证：同一份文件用 `[System.Management.Automation.Language.Parser]::ParseInput()` 按 UTF-8 显式解析时 0 错误，证明问题出在读取端的解码，不在脚本内容。已核实。
- 规避方式：能用一次性 PowerShell 命令完成的事，不落成 `.ps1` 文件——命令行参数不经过文件解码这一环。

### 2.4 编码选择速查

| 目的 | 正确做法 | 错误后果 |
|---|---|---|
| 新建 Markdown 或文本文件 | 文件写入工具，或 `UTF8Encoding($false)` | `-Encoding utf8` 会带 BOM |
| 把命令输出存成日志 | `Out-File -Encoding utf8` | 默认写成 UTF-16LE，read 工具判为 binary |
| 保存含中文的 `.ps1` | UTF-8 **带** BOM | 按 ANSI 解码，中文乱码并报语法错误 |
| 写 Git 提交信息 | UTF-8 **无** BOM 消息文件（见第 4 章） | 见第 4 章 |

---

## 3. 脚本的执行策略

- 本机拦截未签名的 `.ps1`。直接调用时报“未对文件进行数字签名”，而有效执行策略显示为 `RemoteSigned`——**两者的矛盾点尚未查明**，属未验证项。已核实（现象）。
- 临时放开方式，只作用于当前进程、不落盘、不影响机器级设置：

```powershell
Set-ExecutionPolicy -Scope Process Bypass
```

- 已核实。确需常驻时需要签名或调整机器级策略，属于系统变更，本章不涉及，也不建议为单次任务这么做。

---

## 4. 提交信息的编码传递

- 用 `git commit -m "中文信息"` 传中文有乱码风险：入参要经过 Windows PowerShell 5.1 向原生命令传参的编码路径，行为依赖控制台代码页。
- 确定做法是**写 UTF-8 无 BOM 的消息文件，再用 `git commit -F` 传入**：

```powershell
$repo = 'D:\User\Documents\Obsidian\ob_c1'
$sd = @('-c', "safe.directory=$repo", '-C', $repo)
git @sd commit -F '.\commit-msg.txt'
Remove-Item '.\commit-msg.txt'
```

- 上例的前提是消息文件已写好（UTF-8 无 BOM），提交后即删除；该文件是临时文件，不入库。
- 已核实：本机实测提交标题显示正常。

---

## 5. 搜索与文本工具

- 本机 **PATH 中没有 `rg`**（`Get-Command rg` 与 `where.exe rg` 均无输出，PATH 的 22 个路径段里都没有）。已核实。
- 搜索一律用 DSH 的 grep / glob 工具，不依赖 shell 里的 `rg` 或 `grep`。
- 需要在 PowerShell 里读 UTF-8 文档时，`Select-String` 要显式指定编码，否则中文可能匹配不上：

```powershell
Select-String -LiteralPath '.\handover\00handover总览.md' -Pattern '关键字' -Encoding UTF8
```

- 已核实。

---

## 6. DSH 沙箱的能力边界

本节只给结论；机制推导与排查过程见 [[09DSH工作区写入权限]]。

| 能力项 | 结论 | 状态 |
|---|---|---|
| 写入范围 | 沙箱**只限写、不限读** | 已核实 |
| 进程完整性级别 | shell 为 Low 完整性（`S-1-16-4096`） | 已核实 |
| 令牌特权 | 只有 `SeChangeNotifyPrivilege`；管理员组仅用于拒绝 | 已核实 |
| 能否提权 | **在能力上无法提权**，权限申请不会成功 | 已核实 |
| `danger-full-access` | 不提升管理员权限，但令牌不再被降到 Low IL，因此**确实能解除 Low 完整性带来的写入限制** | 已核实（详见下条说明） |
| 网络出口 | 一切进入上下文的内容都要经模型服务与其中继出境 | 已核实 |

- **关于 `danger-full-access`**：它不是提权，令牌里并不会多出管理员权限；变的是完整性级别不再被压低，于是原本被 Low IL 挡住的写入（如既有子目录里的 `git` 操作）得以通过。本库的同类问题最终是用提权补标签解决的（见 [[09DSH工作区写入权限]] 第 5.2 节），但“无法提权”与“无法绕过写入限制”是两件事，不要混为一谈。
- **凭据文件**：其保护水平依赖本机 DACL，沙箱不会额外阻止读取，因此凭据条款属**自律范围**，靠行为约束而不是靠系统拦截。本章不记录该文件的数值与路径。

---

## 7. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-11 | 1.0.0 | 初版：从 `handover/02`、`03`、`04` 第 9 章提取本机工具链实操约定，收编 PowerShell 5.1 编码、脚本执行策略、提交信息编码、编辑器对 diff 的影响、搜索工具与 DSH 沙箱能力边界 |
| 2026-10-11 | 1.1.0 | 按用户决定把原第 5 章「编辑器对 `git diff` 的影响」迁入 [[03Obsidian实操约定]]（Obsidian 相关内容统一落点），本篇自第 6 章起章号前移；[[#1. 适用范围]] 补分工说明 |

---
