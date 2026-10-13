# Obsidian 实操约定

> 文档名称：Obsidian 实操约定
> 适用范围：本库在 Obsidian 中的编辑操作与命名约定（改名与双链、同名冲突）、必须排除跟踪的界面状态文件与本库的 GitHub 接入做法，以及 `obsidian` 命令行的使用限制
> 适用对象：本库作者，以及在本库中执行编辑、改名、提交与渲染核验的 AI
> 最近更新：2026-10-13
> 关联文档：[[00note总览]]、[[01GitCommand]]、[[02GitUse]]、[[05Markdown规范]]、[[11Windows工具链实操约定]]

**目录**

- [[#1. 适用范围]]
- [[#2. 编辑操作与双链]]
- [[#3. 库的命名与同名冲突]]
- [[#4. 必须排除跟踪的文件]]
- [[#5. 接入 GitHub 的两种做法]]
- [[#6. obsidian 命令行的使用限制]]
- [[#7. 变更记录]]

## 1. 适用范围

本篇是本库 Obsidian 相关内容的统一落点，四个部分各管一段：

- **记录文件的界面状态**（工作区布局、关系图谱、反向链接开关等）不属知识内容，一律排除跟踪，见第 4 章。
- **编辑操作与双链**：改名会不会同步、反引号路径要不要手工改、同名双链怎么处理、表格被重排怎么办，见第 2、3 章。
- **接入 GitHub**：把本库连到远端、停止跟踪已入库的界面状态文件，见第 4、5 章。
- **命令行核验**：用 `obsidian` 命令读库、开文档、看渲染结果时的边界，见第 6 章。

分工与边界：

- 命令写法查 [[01GitCommand]]，原理与回退流程查 [[02GitUse]]；本章只在「落到本库的实际做法」这一层展开。
- 本机 PowerShell 与编码约定、DSH 沙箱能力边界见 [[11Windows工具链实操约定]]。
- 写作格式以 [[05Markdown规范]] 为准，本章不重复格式条款。
- 带“已核实”的行是本机实测结论；标“推测”的行未实测。

---

## 2. 编辑操作与双链

### 2.1 改名会不会同步双链，取决于用哪个入口

- **在 Obsidian 应用内改名**：应用会自动更新全库指向该文档的 `[[双链]]`。已核实。
- **用文件工具或 `git mv` 改名**：**不会**更新任何双链，必须手工或脚本同步，否则留下死链。
- 结论：改名一律走 Obsidian 应用内入口；确需用文件工具改名时，改完必须全库排查 `[[旧名]]`。

### 2.2 反引号里的纯文本路径不受改名影响

- 写成反引号包起来的纯文本（如 `` `note/05Command.md` ``）不参与双链解析，改名时**不会被自动更新**，也不会在死链检查里暴露。
- 这类残留必须单独排查：搜旧文件名，逐个判断该处是历史记录还是活跃引用。
- 已核实。

### 2.3 双链的读取口径

- 双链是导航与相关性提示，不是读取清单。发现双链后按需读取，不递归展开整条关系链。
- 规则正文见本库 `AGENTS.md` 第 2.7 节，本篇不重复。

### 2.4 Obsidian 会重排 Markdown 表格

- Obsidian 打开含表格的文档后会重排表格，形态固定为**补齐单元格空格 + 拉长分隔行**。实测某次改动多出 68 字节，忽略空白差异后只剩分隔行长度变化。
- 后果：工作区出现与内容无关的未提交改动，混进 `git add` 会污染提交内容。
- 处理方式：**提交前先确认 staged diff 是否只属这类变化**，是则按需要纳入或还原；不要为了消掉它而回退，回退后编辑器再存仍会变回。
- 经核查这类重排只改对齐填充，**单元格文字一字未改**，两种写法按 [[05Markdown规范]] 第 5.4 节均合规，故保留现状。
- 触发者**推测**为 Obsidian 的表格格式化插件，未实测；但同一形态在 `03`、`04`、`06`、`08` 四篇交接文档里独立记录过，证据已够。

---

## 3. 库的命名与同名冲突

### 3.1 同名双链在 Obsidian 中指向不明

- 库内存在两个同名文件时，`[[名称]]` 形式的双链**指向不明确**，Obsidian 无法据此唯一解析。已核实。
- 因此库级文件与目录内文件不得撞名。已发生的实例：`templates/` 目录的说明文件原拟命名为 `README.md`，会与库根 `README.md` 撞名，故改用 `templates/00模板说明.md`。
- 需要指向特定文件时，用带路径的链接形式，不要依赖裸名。

### 3.2 总览类文件不带日期

- 总览与规范类文件（`00note总览.md`、`00handover总览.md`、`01交接文档写作格式.md`）**文件名不带日期**，因为它们不随阶段冻结。
- 命名里带日期即表示「这是一份阶段快照」，见 `handover/` 的 `NN主题-YYYY-MM-DD.md` 约定。
- `00note总览.md` 的 `note` 保持小写，属命名例外，理由见 [[00note总览]] 第 1.1 节。

---

## 4. 必须排除跟踪的文件

在库根维护 `.gitignore` 与 `.gitattributes` 两个文件。

### 4.1 `.gitignore`

本库用的规则如下：

```gitignore
# OS 元数据
.DS_Store
Thumbs.db
desktop.ini

# Obsidian 本地 UI 状态（频繁变动，必须排除）
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/workspace*.json
.obsidian/backlink.json

# Obsidian 缓存与索引
.obsidian/cache/
.obsidian/graph.json
.obsidian/copilot-index*.json

# Obsidian 回收站
.trash/

# 项目级 Skills 与它的清单
.agents/
skills-lock.json

# Agent 工具与会话数据
.workbuddy/
```

规则语法（`*.log`、`/dist/`、`!important.log`）与生效范围见 [[01GitCommand]] 第 6 章。

### 4.2 `.gitattributes`

本库用的规则如下：

```gitattributes
* text=auto eol=lf
*.md text eol=lf
*.canvas text eol=lf
*.json text eol=lf
*.png binary
*.jpg binary
```

属性含义与 `git add --renormalize .` 的用法见 [[01GitCommand]] 第 1 章与第 5 章；行尾符改动的原因见 [[02GitUse]]。

### 4.3 排除的判据：是否只记录本机界面状态

排除与否不看文件在不在 `.obsidian/` 下，而看**它记录的是不是随本机操作频繁变动、且换设备后无意义的界面状态**：

| 文件 | 内容 | 处置 | 理由 |
|---|---|---|---|
| `workspace.json`、`workspace-mobile.json`、`workspace*.json` | 当前打开的标签、面板布局、光标位置 | 排除 | 每次操作都更新，提交历史混乱，多设备同步极易冲突；Pull 后缺少它不影响使用 |
| `graph.json` | 关系图谱的颜色、物理参数、节点坐标 | 排除 | 高度依赖屏幕分辨率，不同设备会重新计算；不影响笔记内容与链接结构 |
| `backlink.json` | 反向链接面板的开关状态 | 排除 | 33 字节的界面开关（内容为 `{"backlinkInDocument": false}`），与 `graph.json` 同类 |
| `cache/`、`copilot-index*.json` | 缓存与索引 | 排除 | 可由内容重新生成 |
| `.obsidian/` 下其余文件 | 插件配置、主题、快捷键等 | **跟踪** | 属有意维护的配置，需多设备保持一致 |

- `.agents/` 与 `skills-lock.json` 属工具目录与清单，同样排除；技能来源与接入记录见 `handover/13技能接入与模板目录-2026-10-08`。`.workbuddy/` 属 Agent 工具与会话数据，同样排除。
- `.gitignore` 的补充由用户自行决定，Agent 不主动修改。

### 4.4 如果已经提交过这些文件，如何停止跟踪

```bash
git rm --cached .obsidian/workspace.json
git rm --cached .obsidian/graph.json
# 如有 mobile 版本：
# git rm --cached .obsidian/workspace-mobile.json
git commit -m "停止跟踪本地界面状态文件"
git push
```

本地文件会保留，只是不再被 Git 跟踪。取消跟踪的完整流程、`.gitignore` 配合方式和常见问题见 [[02GitUse]] 第 7 章。

---

## 5. 接入 GitHub 的两种做法

### 5.1 方法 A：使用 Obsidian Git 插件（推荐）

1. 在 Obsidian 中进入 **设置 → 第三方插件**，关闭安全模式。
2. 浏览并安装 **Obsidian Git** 插件，启用。
3. 在库根目录打开终端，完成本地仓库初始化与首次推送：

```bash
   git init
   git remote add origin https://github.com/你的用户名/你的仓库名.git
   git add .
   git commit -m "初始提交"
   git push -u origin main
```

4. 进入 **设置 → Obsidian Git**，开启自动提交、自动拉取和启动时拉取。
5. 以后 Obsidian 会自动同步，也可在命令面板执行 `Commit-and-sync`。

### 5.2 方法 B：手动使用 Git 命令行

1. 在库根目录执行：

```bash
   git init
   git remote add origin https://github.com/你的用户名/你的仓库名.git
```

2. 创建 `.gitignore` 和 `.gitattributes`（内容见第 4 章）。
3. 提交并推送：

```bash
   git add .
   git commit -m "初始提交"
   git push -u origin main
```

4. 以后每次修改后：

```bash
   git add .
   git commit -m "更新笔记"
   git push
```

> 注意：
>
> - `git remote add origin <url>` 只对当前文件夹有效，每个库需单独设置。
> - 示例统一用 `main`；若远程默认分支仍是 `master`，把命令里的分支名换成 `master`，或按 [[02GitUse]] 第 1 章把分支改名为 `main`。

### 5.3 本库的实际选择

- 本库不走插件自动同步：Agent 只在用户明确要求时执行本地提交，远端推送一律由用户在沙箱外操作。规则见 [[10提交与文档同步规则]]。
- 提交前的同步检查与敏感值扫描按 [[10提交与文档同步规则]] 第 4 章执行，本篇不重复。

---

## 6. `obsidian` 命令行的使用限制

以下各条均针对本机 Obsidian 1.13.7；环境细节见 `handover/13技能接入与模板目录-2026-10-08` 第 9 章。

| 限制 | 现象或原因 | 应对 |
|---|---|---|
| **必须在沙箱外执行** | 沙箱内报 `The CLI is unable to find Obsidian. Please make sure Obsidian is running and try again.`，退出码 1 | 需要跑 CLI 时先向用户申请授权 |
| 应用必须处于打开状态 | 未打开时同样报找不到 Obsidian | 先确认 Obsidian 在运行 |
| `eval` 顶层不能写 `return` | 报 `Error: Illegal return statement` | 把代码包成 IIFE；异步用 `async () => {}` |
| `eval` 里不能导入 Obsidian 模块 | `await import('obsidian')` 报 `Error: Failed to resolve module specifier 'obsidian'`；`window.MarkdownRenderer` 等全局不存在 | 不要指望在 `eval` 里直接调用渲染 API |
| `open` 默认打开编辑模式 | 想核验阅读视图时看不到渲染结果 | 用 `setState` 切到 `preview` |
| 阅读视图是虚拟滚动 | 长文档只渲染视口内区块（实测 `clientHeight` 699、`scrollHeight` 4714），一次性取 DOM 会漏 | 按步长滚动收集并去重，才能完整核对 |

- 可用命令：`obsidian help` 列出完整清单；常用的是 `read`、`open`、`dev:dom`、`dev:screenshot`、`vault`、`eval`。已核实。
- **与其他工具的分工**：`knap validate` 只做静态检查，不校验运行时取值；`defuddle` 对含代码块的页面保真度不足。两者的实测范围见 `handover/15规则验证与提交同步观察-2026-10-10` 第 5 章。

---

## 7. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-03 | 1.0.0 | 初版：本库接入 GitHub 的两种做法、必须排除跟踪的文件与原因 |
| 2026-10-11 | 2.0.0 | 按用户决定扩容为 Obsidian 相关内容的统一落点并更名（原 `03GitObsidian.md`）：新增大纲、编辑操作与双链（含自 [[11Windows工具链实操约定]] 迁入的表格重排）、库的命名与同名冲突、排除判据表、`obsidian` 命令行的使用限制；`.gitignore` 补齐 `backlink.json`、`.agents/`、`skills-lock.json`；原第 1、2 章降为第 4、5 章 |
| 2026-10-13 | 2.0.1 | 第 4.1 节 `.gitignore` 规则块与第 4.3 节排除清单补入 Agent 工具与会话数据目录 `.workbuddy/` |

---
