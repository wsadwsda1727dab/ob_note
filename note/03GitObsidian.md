# Obsidian 与 GitHub 连接及排除文件指南

> 文档名称：Obsidian 与 GitHub 连接及排除文件指南
> 适用范围：把本库接入 GitHub 的两种做法，以及必须排除跟踪的文件与原因
> 适用对象：本库作者，以及协助配置同步与忽略规则的 AI
> 最近更新：2026-10-03
> 关联文档：[[00总览]]（全库导航）、[[01GitCommand]]（逐条命令）、[[02GitUse]]（原理与回退流程）

## 1. 将 Obsidian 库连接到 GitHub

### 1.1 方法 A：使用 Obsidian Git 插件（推荐）

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

### 1.2 方法 B：手动使用 Git 命令行

1. 在库根目录执行：

```bash
   git init
   git remote add origin https://github.com/你的用户名/你的仓库名.git
```

2. 创建 `.gitignore` 和 `.gitattributes`（内容见第 2 章）。
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

---

## 2. 必须排除跟踪的文件

在库根目录创建以下两个文件。

### 2.1 `.gitignore`

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

# 关系图谱布局（依赖屏幕分辨率，易冲突）
.obsidian/graph.json

# 缓存与索引
.obsidian/cache/
.obsidian/copilot-index*.json

# 回收站
.trash/
```

规则语法（`*.log`、`/dist/`、`!important.log`）与生效范围见 [[01GitCommand]] 第 6 章。

### 2.2 `.gitattributes`

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

### 2.3 为什么要排除 `workspace.json` 和 `graph.json`

- **`workspace.json`**：记录当前打开的标签、面板布局、光标位置等。Obsidian 每次操作都会更新，提交历史会混乱，多设备同步极易冲突。Pull 后缺少它不影响使用，只需重新打开笔记、调整布局。
- **`graph.json`**：记录关系图谱的颜色、物理参数、节点坐标等。高度依赖屏幕分辨率，不同设备会重新计算，频繁变动且易冲突。不影响笔记内容和链接结构。

### 2.4 如果已经提交过这些文件，如何停止跟踪

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
