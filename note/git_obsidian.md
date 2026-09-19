# Obsidian 与 GitHub 连接及排除文件指南（含常用 Git 命令补充）

> 本文档总结如何将 Obsidian 库连接到 GitHub，必须排除跟踪的文件，以及相关 Git 命令的使用。

---

## 一、将 Obsidian 库连接到 GitHub

### 方法 A：使用 Obsidian Git 插件（推荐）

1. 在 Obsidian 中进入 **设置 → 第三方插件**，关闭安全模式。
2. 浏览并安装 **Obsidian Git** 插件，启用。
3. 在库根目录打开终端，执行：
   ```bash
   git init
   git remote add origin https://github.com/你的用户名/你的仓库名.git
   git add .
   git commit -m "初始提交"
   git push -u origin master
   ```
4. 进入 **设置 → Obsidian Git**，开启自动提交、自动拉取和启动时拉取。
5. 以后 Obsidian 会自动同步，也可在命令面板执行 `Commit-and-sync`。

### 方法 B：手动使用 Git 命令行

1. 在库根目录执行：
   ```bash
   git init
   git remote add origin https://github.com/你的用户名/你的仓库名.git
   ```
2. 创建 `.gitignore` 和 `.gitattributes`（内容见第二部分）。
3. 提交并推送：
   ```bash
   git add .
   git commit -m "初始提交"
   git push -u origin master
   ```
4. 以后每次修改后：
   ```bash
   git add .
   git commit -m "更新笔记"
   git push
   ```

> 注意：`git remote add origin <url>` 只对当前文件夹有效，每个库需单独设置。

---

## 二、必须排除跟踪的文件

在库根目录创建以下两个文件：

### 1. `.gitignore`

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

### 2. `.gitattributes`

```gitattributes
* text=auto eol=lf
*.md text eol=lf
*.canvas text eol=lf
*.json text eol=lf
*.png binary
*.jpg binary
```

### 3. 为什么要排除 workspace.json 和 graph.json

- **`workspace.json`**：记录当前打开的标签、面板布局、光标位置等。Obsidian 每次操作都会更新，提交历史会混乱，多设备同步极易冲突。Pull 后缺少它不影响使用，只需重新打开笔记、调整布局。
- **`graph.json`**：记录关系图谱的颜色、物理参数、节点坐标等。高度依赖屏幕分辨率，不同设备会重新计算，频繁变动且易冲突。不影响笔记内容和链接结构。

### 4. 如果已经提交过这些文件，如何停止跟踪

```bash
git rm --cached .obsidian/workspace.json
git rm --cached .obsidian/graph.json
# 如有 mobile 版本：
# git rm --cached .obsidian/workspace-mobile.json
git commit -m "停止跟踪本地界面状态文件"
git push
```

本地文件会保留，只是不再被 Git 跟踪。

---

## 三、Git 分支管理：将 `master` 改为 `main`

### 1. 查看当前分支和远程信息

```bash
git branch -vv          # 查看本地分支及上游跟踪关系
git remote show origin  # 查看远程仓库详细信息，包括默认分支
```

### 2. 本地重命名分支

```bash
git branch -m master main
```

### 3. 推送新分支并设置上游

```bash
git push -u origin main
```

### 4. 在 GitHub 上修改默认分支

进入仓库 **Settings → Branches**，将 **Default branch** 从 `master` 改为 `main`。

### 5. 删除远程旧的 `master` 分支

```bash
git push origin --delete master
```

### 6. 清理本地对远程已删除分支的引用

```bash
git fetch --prune
```

### 7. 以后新建仓库默认使用 `main`

```bash
git config --global init.defaultBranch main
```

---

## 四、其他常用命令补充

### 1. 重新暂存最新改动

`git add .` 后如果又修改、新增或删除了文件，再次执行即可：

```bash
git add .
```

然后用 `git status` 确认，再 `git commit`。

### 2. 提交说明的写法

`git commit -m "..."` 引号内可以自由写，概括所有改动。例如：

```bash
git commit -m "添加 .gitattributes，更新代码和数据文件"
```

也可以用多个 `-m` 写详细说明：

```bash
git commit -m "标题" -m "详细说明第一行\n详细说明第二行"
```

### 3. 查看暂存区与工作区差异

```bash
git diff          # 工作区 vs 暂存区
git diff --cached # 暂存区 vs 最近提交
```

### 4. 取消暂存

```bash
git restore --staged <文件>
```

### 5. 查看提交历史

```bash
git log --oneline
git log --stat
```

### 6. 撤销提交（本地未推送）

```bash
git reset --soft HEAD~1   # 保留改动在暂存区
git reset --mixed HEAD~1  # 保留改动在工作区（默认）
git reset --hard HEAD~1   # 丢弃所有改动（危险）
```

---

## 五、总结

- **连接方式**：Obsidian Git 插件（自动）或手动 Git 命令。
- **必须排除**：`.obsidian/workspace.json`、`graph.json` 等本地 UI 状态文件。
- **分支统一**：将本地和远程默认分支统一为 `main`，避免推送和协作混乱。
- **常用命令**：重命名分支、删除远程分支、清理引用、设置默认分支名。
- **提交习惯**：提交说明写清楚改动内容，便于回溯。

---

*文档结束。可直接复制保存为 `obsidian-github-full.md`。*