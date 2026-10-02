# Skills 安装方式

**目录**

- [[#1. 概述]]
- [[#2. 安装前准备]]
- [[#3. 方式一：npx skills（推荐）]]
- [[#4. 方式二：git clone]]
- [[#5. 方式三：脚本一键安装]]
- [[#6. 方式四：第三方 CLI 工具]]
- [[#7. 方式五：手动复制]]
- [[#8. 验证安装]]
- [[#9. 更新与卸载]]
- [[#10. 方式对比]]

## 1. 概述

### 1.1 本文目的

本文总结目前主流的 Skills 安装方式，重点覆盖 **`npx skills`** 和 **`git clone`** 两种使用最广泛的方法，同时补充其他常见安装途径。

### 1.2 适用范围

- Claude Code
- Cursor
- Codex
- OpenCode
- 其他兼容 `SKILL.md` 标准的 Agent 工具

---

## 2. 安装前准备

### 2.1 环境要求

| 依赖 | 说明 |
|---|---|
| Node.js 18+ | `npx skills` 方式必需 |
| npm / npx | 随 Node.js 一起安装 |
| Git | `git clone` 方式必需 |
| 网络 | `npx skills` 从 GitHub 下载文件，需能访问 GitHub |

### 2.2 两种作用范围

| 范围 | 说明 | 典型路径 |
|---|---|---|
| 项目级 | 仅当前项目生效 | `<项目根>/.claude/skills/` |
| 全局级 | 所有项目生效 | `~/.claude/skills/` |

---

## 3. 方式一：npx skills（推荐）

### 3.1 工具简介

`npx skills` 是 Vercel Labs 开发的 Skill 管理工具，支持 50+ 种 AI 工具，在 Skill 领域使用最广泛。它本质上是从 GitHub 下载 Skill 文件，并自动放置到正确的 Agent 目录中。

### 3.2 基本安装命令

```bash
# 安装某个仓库的全部 Skill（项目级，默认）
npx skills add <owner/repo>

# 安装某个仓库的全部 Skill（全局级）
npx skills add <owner/repo> -g

# 只安装指定 Skill
npx skills add <owner/repo> --skill <skill-name>

# 指定目标 Agent
npx skills add <owner/repo> --agent claude-code

# 跳过确认提示
npx skills add <owner/repo> -y
```

### 3.3 常用参数说明

| 参数 | 简写 | 说明 |
|---|---|---|
| `--global` | `-g` | 安装到用户目录，所有项目可用 |
| `--agent` | `-a` | 指定目标 Agent，如 `claude-code`、`cursor` |
| `--skill` | `-s` | 只安装指定 Skill |
| `--yes` | `-y` | 跳过确认提示，适合 CI/CD |

### 3.4 实际示例

```bash
# 安装 anthropics/skills 仓库的 commit 技能到 Claude Code
npx skills add anthropics/skills --skill commit --agent claude-code

# 安装到多个 Agent
npx skills add anthropics/skills --skill commit --agent claude-code cursor

# 全局安装，跳过确认
npx skills add anthropics/skills --skill commit -g -a claude-code -y

# 从任意 Git URL 安装
npx skills add git@github.com:vercel-labs/agent-skills.git
```

### 3.5 安装后的目录

`npx skills` 会将 Skill 文件放到 `.agents/skills/`（项目级）或 `~/.agents/skills/`（全局级），同时为目标 Agent 创建符号链接，例如 `.claude/skills/`[reference:2]。

### 3.6 查看已安装 Skill

```bash
npx skills list
```

---

## 4. 方式二：git clone

### 4.1 适用场景

- 网络无法稳定访问 GitHub，需要从 Gitee 等镜像克隆
- 需要固定到特定 commit 或 tag
- 需要对 Skill 源码做本地修改
- 不希望安装 Node.js

### 4.2 项目级安装

```bash
# 克隆到项目的 .claude/skills/ 目录
mkdir -p .claude/skills
git clone https://github.com/<owner>/<repo>.git .claude/skills/<skill-name>
```

示例：

```bash
git clone https://github.com/SHYXIN/skills.git .claude/skills/shyxin-skills
```

### 4.3 全局级安装

```bash
# 克隆到用户全局 Skills 目录
mkdir -p ~/.claude/skills
git clone https://github.com/<owner>/<repo>.git ~/.claude/skills/<skill-name>
```

示例：

```bash
git clone https://github.com/AWENIAI/awen-store-visual-skill.git ~/.codex/skills/awen-store-visual-skill
```

### 4.4 稀疏克隆（只下载 Skill 子目录）

如果仓库很大，但只需要其中一个 Skill 文件夹，可以使用 Git 稀疏检出：

```bash
mkdir -p .repos && cd .repos
git clone --filter=blob:none --no-checkout https://github.com/<owner>/<repo>.git
cd <repo>
git sparse-checkout init --cone
git sparse-checkout set <path/to/skill-folder>
git checkout
```

然后将检出后的 Skill 目录复制或符号链接到目标 Skills 目录[reference:3]。

### 4.5 克隆后复制

如果不需要保留 Git 历史，克隆后直接复制 Skill 文件夹到目标位置即可：

```bash
# 克隆到临时目录
git clone https://github.com/<owner>/<repo>.git /tmp/skills-repo

# 复制全部 Skill
cp -r /tmp/skills-repo/skills/* ~/.claude/skills/

# 只复制一个 Skill
cp -r /tmp/skills-repo/skills/<skill-name> ~/.claude/skills/

# 清理
rm -rf /tmp/skills-repo
```

---

## 5. 方式三：脚本一键安装

### 5.1 curl 管道脚本

部分仓库提供一键安装脚本，直接通过 `curl` 管道执行：

```bash
curl -fsSL https://raw.githubusercontent.com/<owner>/<repo>/master/install.sh | bash
```

### 5.2 克隆后运行本地脚本

```bash
git clone https://github.com/<owner>/<repo>.git
cd <repo>
./install.sh
```

脚本通常会检测已安装的 Agent 工具，并自动将 Skill 放置到正确目录[reference:4]。

---

## 6. 方式四：第三方 CLI 工具

### 6.1 skillslm

`skillslm` 是另一个支持多 Agent 的 Skill 管理工具，支持 Claude Code、Cursor、Codex、OpenCode 等 9 个 Agent[reference:5]。

```bash
# 直接使用 npx
npx skillslm install <skill-name>

# 指定 Agent 和全局安装
npx skillslm install mcp-builder --agent claude-code --global

# 批量安装
npx skillslm install anthropics/skills --skill mcp-builder --skill pdf --agent claude-code
```

### 6.2 cn-skills-cli（国内加速）

国内用户可使用 `cn-skills-cli` 从 Gitee 镜像拉取，免翻墙：

```bash
# 安装 cn-skills-cli
npm install -g cn-skills-cli --registry=https://registry.npmmirror.com

# 安装 Skill（自动路由到 Gitee 镜像）
cn-skills add <owner/repo> --yes --global --agent claude-code
```

---

## 7. 方式五：手动复制

如果已有现成的 Skill 目录，直接复制到目标位置即可：

```bash
# macOS / Linux
cp -r my-skill ~/.claude/skills/

# 项目级
cp -r my-skill .claude/skills/
```

```powershell
# Windows PowerShell 全局级
Copy-Item -Recurse my-skill\* $env:USERPROFILE\.claude\skills\
```

---

## 8. 验证安装

### 8.1 检查目录

```bash
# 项目级
ls .claude/skills/

# 全局级
ls ~/.claude/skills/
```

应能看到 `<skill-name>` 目录，且目录内有 `SKILL.md` 文件。

### 8.2 重启 Agent

安装后需要重启 Claude Code 或其他 Agent 会话，Skill 才会被加载[reference:8]。

### 8.3 测试触发

在 Agent 中调用该 Skill，例如：

```text
/<skill-name>
```

如果 Skill 正确安装，Agent 应能识别并执行。

---

## 9. 更新与卸载

### 9.1 更新

`npx skills` 方式没有单独的更新命令：对同一个仓库再执行一次安装命令（写法见 1.3.2）即可覆盖为新版本。

```bash
# 全局安装过的 Skill，用同样的全局参数重新执行
npx skills add <owner/repo> -g

# git clone 方式
cd ~/.claude/skills/<skill-name>
git pull
```

`npx skills` 也支持更新命令，具体以工具版本为准。

### 9.2 卸载

```bash
# 直接删除目录
rm -rf ~/.claude/skills/<skill-name>
rm -rf .claude/skills/<skill-name>
```

---

## 10. 方式对比

| 方式 | 需要 Node.js | 需要 Git | 自动检测 Agent | 适合场景 |
|---|---|---|---|---|
| npx skills | 是 | 否 | 是 | 快速安装、多 Agent 管理 |
| git clone | 否 | 是 | 否 | 无 Node 环境、需固定版本 |
| 脚本安装 | 视脚本而定 | 是 | 是 | 仓库提供一键脚本 |
| 第三方 CLI | 是 | 否 | 是 | 需要额外功能（国内加速等） |
| 手动复制 | 否 | 否 | 否 | 离线环境、精细控制 |

---
