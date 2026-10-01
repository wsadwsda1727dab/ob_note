
---
# 一、分支管理：将 `master` 改为 `main`

## 1. 查看当前分支和远程信息

```bash
git branch -vv          # 查看本地分支及上游跟踪关系
git remote show origin  # 查看远程仓库详细信息，包括默认分支
```

## 2. 本地重命名分支

```bash
git branch -m master main
```

## 3. 推送新分支并设置上游

```bash
git push -u origin main
```

## 4. 在 GitHub 上修改默认分支

进入仓库 **Settings → Branches**，将 **Default branch** 从 `master` 改为 `main`。

## 5. 删除远程旧的 `master` 分支

```bash
git push origin --delete master
```

## 6. 清理本地对远程已删除分支的引用

```bash
git fetch --prune
```

## 7. 以后新建仓库默认使用 `main`

```bash
git config --global init.defaultBranch main
```

---

# 二、基于整个旧版本修改并保留的正确流程

如果你想基于历史中的某个旧版本进行修改，并且希望这些修改最终合并回 `main` 分支，**不要直接 `git checkout <旧提交>` 后修改提交**，否则会进入 detached HEAD 状态，提交容易悬空。

正确做法是：**先基于旧提交创建一个新分支，在新分支上修改提交，再合并回 main。**

## 1. 找到旧版本的提交哈希

```bash
git log --oneline
```

记下要基于的旧提交哈希，例如 `6b30a27`。

## 2. 基于旧提交创建新分支

```bash
git switch -c fix-old 6b30a27
```

或旧版命令：

```bash
git checkout -b fix-old 6b30a27
```

- `fix-old` 是临时分支名，可自定义。
- 执行后 HEAD 指向新分支，不会进入 detached HEAD。

## 3. 在新分支上修改文件并提交

```bash
git add .
git commit -m "基于旧版本的修改"
```

## 4. 切回 main 分支

```bash
git switch main
```

## 5. 将修改合并到 main

有两种方式可选：

**方式 A：合并分支**

```bash
git merge fix-old
```

**方式 B：cherry-pick 单个提交**

```bash
git cherry-pick fix-old
```

## 6. 推送到远程

```bash
git push
```

## 7. 删除临时分支（可选）

```bash
git branch -d fix-old
```

---

# 三、merge 与 cherry-pick 的区别

| 对比项      | `git merge fix-old`         | `git cherry-pick fix-old` |
| -------- | --------------------------- | ------------------------- |
| **作用**   | 把整个分支合并进当前分支                | 只把分支上的某个提交复制到当前分支         |
| **历史结构** | 保留分支分叉，生成一个合并提交             | 不保留分叉，直接生成新提交，历史线性        |
| **提交数量** | 保留原分支所有提交 + 一个 merge commit | 只复制指定提交（默认最后一个）           |
| **提交哈希** | 原提交哈希不变                     | 生成新的提交哈希                  |
| **适合场景** | 想保留分支历史，或分支有多个提交需整体合并       | 只想拿一个或几个提交，让历史保持线性        |
| **冲突处理** | 一次解决多个提交的冲突                 | 逐个提交解决冲突                  |

### 历史结构对比

假设 `main` 和 `fix-old` 历史如下：

```text
main:    A --- B --- C
                  \
fix-old:           D --- E
```

**merge 后：**

```text
main:    A --- B --- C ------- M
                  \         /
fix-old:           D --- E
```

**cherry-pick（假设只复制 D）后：**

```text
main:    A --- B --- C --- D'
```

### 选择建议

- 如果 `fix-old` 只提交了一次，推荐 **cherry-pick**，历史更干净。
- 如果 `fix-old` 有多个提交且想整体保留分支轨迹，用 **merge**。

---

# 四、detached HEAD 的避免与恢复

### 为什么会进入 detached HEAD？

执行 `git checkout <提交哈希>` 或 `git switch --detach <提交>` 时，HEAD 会直接指向某个提交，而不是分支，从而进入 detached HEAD 状态。

### 在此状态下提交会怎样？

提交会创建一个新提交，但**没有任何分支指向它**。一旦你切回其他分支，这个提交就会变成悬空提交（dangling commit），只能用 `git reflog` 找回。

### 如何避免？

- 不要直接 `git checkout <旧提交>` 后修改提交。
- 如果想基于旧版本修改，请用 `git switch -c 新分支 <旧提交>` 创建新分支，流程见第二章。
- 如果只是想查看旧版本文件，用 `git show <提交>:<文件>` 或 `git checkout <提交> -- <文件>`。
- 只想临时看看旧提交，用 `git switch --detach <提交>`，看完用 `git switch main` 回来。

### 已经悬空了怎么办？

1. 用 `git reflog` 找到悬空提交的哈希。
2. 切回目标分支（如 `main`）。
3. 用 `git cherry-pick <哈希>` 将其应用到当前分支。
4. 推送即可。

---

# 五、checkout 与 switch 使用的比较

`git checkout` 是 Git 的“瑞士军刀”，一个命令能做很多事；`git switch` 是 Git 2.23 引入的专用命令，只负责切换分支。两者功能有重叠，但推荐分工使用。

## 1. 功能对比

| 操作                            | 旧命令（checkout / reset）       | 新命令（switch / restore）              | 说明                   |
| ------------------------------- | --------------------------------- | --------------------------------------- | ---------------------- |
| 切换分支                        | `git checkout <分支>`             | `git switch <分支>`                     | 切换到已有分支         |
| 创建并切换分支                  | `git checkout -b <新分支>`        | `git switch -c <新分支>`                | 基于当前提交创建新分支 |
| 基于指定提交创建并切换          | `git checkout -b <新分支> <提交>` | `git switch -c <新分支> <提交>`         | 从旧提交拉出新分支     |
| 切换到某个提交（detached HEAD） | `git checkout <提交>`             | `git switch --detach <提交>`            | 进入分离头指针状态      |
| 恢复文件到某版本                | `git checkout <提交> -- <文件>`   | `git restore --source=<提交> -- <文件>` | 只改文件，不移动 HEAD  |
| 恢复文件到最近提交              | `git checkout -- <文件>`          | `git restore -- <文件>`                 | 丢弃工作区修改         |
| 取消暂存                        | `git reset HEAD <文件>`           | `git restore --staged <文件>`           | 从暂存区撤出           |

## 2. `git restore` 的方向

恢复文件时，分清"从哪个区域覆盖哪个区域"就不会用错。三个区域的关系：

```text
HEAD（最近提交）
   │
   │  git restore --staged
   ▼
暂存区（index）
   │
   │  git restore
   ▼
工作区
```

- **`git restore <文件>`**：从**暂存区**覆盖**工作区**。它不碰暂存区，也不直接从提交里拿内容。
- **`git restore --staged <文件>`**：从**HEAD（最近提交）** 覆盖**暂存区**。它不碰工作区。
- **`git restore --source=<提交> --staged --worktree <文件>`**：才从指定提交同时覆盖暂存区和工作区。

| 命令 | 源 | 目标 | 效果 |
| --- | --- | --- | --- |
| `git restore 文件` | 暂存区 | 工作区 | 丢弃工作区修改，回到暂存区状态 |
| `git restore --staged 文件` | HEAD | 暂存区 | 取消暂存，暂存区回到 HEAD，工作区不变 |
| `git restore --source=HEAD~1 -- 文件` | 指定提交 | 工作区 | 工作区变成旧提交版本，暂存区不变 |
| `git restore --source=HEAD --staged --worktree -- 文件` | HEAD | 暂存区 + 工作区 | 彻底回到最近提交，两边都覆盖 |

## 3. 核心区别

- **`git checkout`**：多功能，既能切分支，又能恢复文件，还能切提交。功能多但容易混淆，也容易误操作。
- **`git switch`**：只用于切换分支（包括创建分支、切到指定提交）。语义清晰，减少误用。
- **`git restore`**：专门用于恢复文件内容或取消暂存，替代 `checkout` 的文件恢复功能。
- **`git reset`**：主要用来移动分支指针、撤销提交，也能取消暂存。

Git 2.23 之后，官方把 `checkout` 的职责拆成了 `switch` 和 `restore`，新项目推荐用 `switch` + `restore`。

常用写法：

```bash
git switch main                 # 切换到 main 分支
git switch -c new-feature       # 创建并切换到 new-feature
git switch -c fix-old 6b30a27   # 基于旧提交创建并切换
git switch --detach 6b30a27     # 进入 detached HEAD，只看不提交
git restore 文件                # 丢弃工作区修改，回到暂存区状态
git restore .                   # 丢弃所有工作区修改
git restore --staged 文件       # 取消暂存，改动回到工作区
git checkout main               # 旧版切换分支
git checkout 6b30a27            # 旧版切到旧提交，进入 detached HEAD
git reset --soft HEAD~1         # 撤销最近提交，改动保留在暂存区
git reset --mixed HEAD~1        # 撤销最近提交，改动保留在工作区（默认）
git reset --hard HEAD~1         # 撤销最近提交，丢弃所有改动（危险）
git reset HEAD 文件             # 取消暂存（旧用法）
```

- `git switch` 不会恢复文件，也不支持 `-- <文件>`。
- `git reset` 的 `--soft` 只移动 HEAD；`--mixed` 还会重置暂存区；`--hard` 连工作区一起重置。

## 4. 常见场景该用哪个？

| 场景 | 推荐命令 |
| --- | --- |
| 切换分支 | `git switch <分支>` |
| 丢弃工作区修改 | `git restore <文件>` |
| 取消暂存 | `git restore --staged <文件>` |
| 恢复文件到旧提交 | `git restore --source=<提交> -- <文件>` |
| 撤销最近一次提交（本地未推送） | `git reset --soft/--mixed/--hard HEAD~1` |
| 撤销已推送的提交 | `git revert HEAD` + `git push` |
| 只想看看旧提交 | `git switch --detach <提交>`，看完 `git switch main` 回来 |

具体操作步骤见第六章。

## 5. 注意事项

- `git switch` **不支持** `-- <文件>` 语法，不能用来恢复文件。
- 如果你的 Git 版本低于 2.23，可能没有 `git switch` 和 `git restore`，只能用 `git checkout`。
- 在脚本或旧教程中常见 `git checkout`，理解其多功能性即可，新项目建议用 `switch` + `restore`。
- 已推送到远程的分支不要随便 `reset` + 强推，协作时用 `revert` 更安全。

## 6. 一句话总结

**`checkout` 是全能旧命令，`switch` 是专用新命令。** 切换分支用 `switch`，恢复文件用 `restore`；旧环境继续用 `checkout` 也没问题，但要注意它可能同时改变 HEAD 和文件。

------

# 六、回退到最近一次提交

如果你修改了文件，发现改错了，想回到最近一次提交（HEAD）的状态，可以按下面情况处理。

## 1. 修改了文件，但还没 `git add`

恢复单个文件：

```bash
git restore 文件名
```

或旧版命令：

```bash
git checkout -- 文件名
```

恢复所有被修改的文件：

```bash
git restore .
```

或：

```bash
git checkout -- .
```

执行后，工作区文件会回到最近一次提交的样子，修改被丢弃。

## 2. 已经 `git add`，但还没 `git commit`

需要先取消暂存，再恢复文件：

```bash
git restore --staged 文件名
git restore 文件名
```

或者一步到位，用 `HEAD` 覆盖暂存区和工作区：

```bash
git checkout HEAD -- 文件名
```

恢复所有文件：

```bash
git restore --staged .
git restore .
```

## 3. 已经 `git commit`，但还没 `git push`

如果想撤销最近一次提交，并丢弃这次提交的所有修改：

```bash
git reset --hard HEAD~1
```

- `HEAD~1` 表示回到上一次提交。
- `--hard` 会丢弃工作区和暂存区的所有改动，**不可恢复**，确认后再用。

如果只是想撤销提交，但保留修改内容：

| 命令 | 提交 | 暂存区 | 工作区 |
| --- | --- | --- | --- |
| `git reset --soft HEAD~1` | 撤销 | 保留改动 | 不动 |
| `git reset --mixed HEAD~1`（即 `git reset HEAD~1`） | 撤销 | 重置 | 保留改动 |
| `git reset --hard HEAD~1` | 撤销 | 重置 | 丢弃改动 |

## 4. 已经 `git push` 到远程

不要用 `git reset --hard` 后强推，除非你确定只有自己使用这个分支。  
更安全的做法是用 `git revert` 生成一个反向提交：

```bash
git revert HEAD
git push
```

这会抵消最近一次提交的改动，历史保留完整。

## 5. 操作前建议

先查看当前状态：

```bash
git status
```

再查看你改了什么：

```bash
git diff
```

确认确实要丢弃，再执行恢复命令。

## 6. 总结

| 状态                   | 命令                                             |
| ---------------------- | ------------------------------------------------ |
| 改了文件，没 `add`     | `git restore 文件` 或 `git checkout -- 文件`     |
| 已 `add`，没 `commit`  | `git restore --staged 文件` + `git restore 文件` |
| 已 `commit`，没 `push` | `git reset --hard HEAD~1`（丢弃提交和修改）      |
| 已 `push`              | `git revert HEAD` + `git push`（安全）           |

**最常用的是：`git restore 文件名`，直接回到最近一次提交的状态。**

> `checkout`、`reset`、`restore`、`switch` 四者的分工、各命令完整用法与三区关系，见第五章。

---

# 七、Git 取消本地和远程仓库文件/文件夹跟踪指南
## 1. 核心概念

- **跟踪（tracked）**：文件已被 Git 纳入版本控制，修改会被记录。
- **取消跟踪（untrack）**：让 Git 不再管理该文件，但**本地文件仍然保留**。
- **`git rm` 与 `git rm --cached` 的区别**：
  - `git rm <文件>`：删除本地文件，同时从 Git 跟踪中移除。
  - `git rm --cached <文件>`：**只从 Git 跟踪中移除，本地文件保留**。

取消跟踪后，需要**提交并推送**，远程仓库才会同步删除该文件（但历史记录中仍保留）。

---

## 2. 取消本地跟踪（通用步骤）

### 2.1 取消单个文件的跟踪

```bash
git rm --cached 文件路径
#例如：
git rm --cached config.ini
```
### 2.2 取消整个文件夹的跟踪

```bash
git rm -r --cached 文件夹路径
# `-r` 表示递归处理文件夹内所有文件。例如：
git rm -r --cached logs/

```
### 2.3 取消多个文件或文件夹

```bash
git rm --cached 文件1 文件2
git rm -r --cached 文件夹1 文件夹2
```
### 2.4 提交并推送

```bash
git commit -m "停止跟踪 xxx"
git push
```

推送后，远程仓库的最新版本中就不再包含这些文件，但**历史提交里仍然存在**。

---

## 3. 如果已经 push 了，怎么让远程也取消跟踪？

操作与上面完全一样，因为 `git rm --cached` + `commit` + `push` 本身就会更新远程仓库。

推送成功后，远程仓库的最新版本中该文件/文件夹就会消失，但本地文件完好无损。

> **注意**：这不会删除历史记录中的文件。如果有人 clone 旧提交，仍然能看到它。若要从所有历史中彻底删除，需使用 `git filter-repo` 或 BFG，操作复杂且有风险，一般不推荐。

---

## 4. 配合 `.gitignore` 防止再次被跟踪

取消跟踪后，建议把该文件/文件夹加入 `.gitignore`，避免以后不小心又 `git add` 进去。
在项目根目录编辑 `.gitignore`，添加：
```gitignore
# 忽略单个文件
config.ini

# 忽略文件夹
logs/
```

如果该文件之前已经被跟踪，仅加 `.gitignore` 是无效的，必须先执行 `git rm --cached` 停止跟踪。

---

## 5. 只想停止跟踪，但连本地文件也一起删除？

如果确实想删除本地文件，同时取消跟踪：
```bash
git rm 文件
# 或
git rm -r 文件夹
```
然后提交推送即可。本地文件会被删除，请谨慎使用。

---

## 6. 常见问题

### 6.1 执行 `git rm --cached` 后，`git status` 显示 `deleted`，但本地文件还在？

这是正常的。`--cached` 只从暂存区移除，本地文件保留。提交后，Git 认为该文件被删除，但工作区里还有。

### 6.2 推送后，远程仓库的文件不见了，但本地还在？

这正是预期效果。远程最新版本不再包含该文件，本地因为使用了 `--cached` 所以保留。

### 6.3 想恢复跟踪怎么办？

直接 `git add 文件` 即可重新跟踪。

### 6.4 已经 push 了，还能彻底从历史删除吗？

可以，但需要重写历史：
```bash
git filter-repo --path 文件 --invert-paths
```
或使用 BFG Repo-Cleaner。之后需要强制推送，并通知所有协作者重新克隆。操作风险高，请谨慎。

---

## 7. 总结

| 目的            | 命令                                 |
| ------------- | ---------------------------------- |
| 停止跟踪单个文件，保留本地 | `git rm --cached 文件`               |
| 停止跟踪文件夹，保留本地  | `git rm -r --cached 文件夹`           |
| 提交并同步到远程      | `git commit -m "..."` + `git push` |
| 防止再次跟踪        | 在 `.gitignore` 中添加对应路径             |
| 彻底从历史删除       | `git filter-repo` 或 BFG（高风险）       |

**核心操作：`git rm --cached` → `git commit` → `git push`。** 这样本地文件保留，远程最新版本中删除，历史记录不受影响。

---

