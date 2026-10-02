# Codex 断电故障恢复记录

> 记录时间：2026-10-02
> 故障源：2026-10-01 22:54 突然断电（非正常关机），Codex 桌面版当时正在更新
> 恢复方式：全程只读排查 + 定点修复，未使用"重装/重置"这类破坏性手段
> 结论：**所有对话数据零丢失**（46 条全部找回并重新归入项目）

---

## 一、故障总览：一次断电，四处"全 0 字节"文件

这次断电的本质是 **NTFS 数据没有落盘**：Windows 来不及把文件缓存刷进磁盘，恢复后这些文件的内容一律读成 `0x00`（文件大小元数据还在，所以看起来"文件还在"，实际内容全空）。它同时打坏了 4 处：

| 受损对象 | 受损表现 | 症状 |
|---|---|---|
| `~\.codex\config.toml` | 4316 字节全 0 | Codex 打不开，弹"无法加载组织设置" |
| `~\.codex\.codex-global-state.json` | 从 95 KB 缩到 3.4 KB，项目相关键全丢 | 侧边栏"项目"分区消失 |
| `~\.codex\skills\.system\` 43 个文件 | 全 0 | 5 个内置技能不可用 |
| `~\.codex\state_5.sqlite` 中线程的项目归属 | `project_id` 全为 NULL | 项目点进去是空的 |

还有一个**衍生问题**：cc-switch 数据库里存的 Codex 配置模板含旧版本路径，它是把损坏路径反复写回来的源头。

**关键判断**：这四处看起来是 4 个独立 bug，实际是**同一次断电的不同侧面**。所以不能只修一个——尤其不能只重装 Codex，因为损坏的是用户数据目录，重装不会碰它。

---

## 二、问题 1：Codex 无法启动，提示"无法加载组织设置"

**现象**：开机后打开 Codex 弹出对话框——"应用已暂停，等待安全加载组织设置后才能恢复。网络连接、代理或防火墙可能导致无法访问……"，提供"重试/退出登录/退出"三个按钮。点重试无效。

**分析**：对话框里说的"网络、代理、防火墙"是**误导性文案**，它是所有"配置加载失败"共用的兜底提示，实际与网络无关。真正的证据在日志里：

```
failed to read configuration layers:
C:\Users\34936\.codex\config.toml:1:4317: key with no value, expected `=`
```

`config.toml` 是 4316 字节、**全是 0x00 空字节**（无换行、无任何字符），TOML 解析器把它当成"一个 4316 字符的键名，然后遇到文件结尾却没等到 `=`"，于是在第 4317 列报错。Codex 的 Rust 内核每次启动都要解析这个"配置层"，解析失败就拿不到 config requirements，按安全策略**暂停整个应用**——这就是那个对话框的真正来源。

事件日志能对上时间：`Kernel-Power 41`（系统未正常关机就重启）+ `EventLog 6008`（上次关机时间 22:54:41 属意外关机）。

**解决**：
1. 先备份坏文件（留证）：`config.toml` → `config.toml.corrupt-20261001`
2. 用 **cc-switch** 的"切换/应用"功能重新生成 `config.toml`（cc-switch 数据库里存着 DeepSeek 供应商配置）
3. 生成后校验三件套：**无 BOM**（首 3 字节 `6D 6F 64` = `mod`）、**无 0 字节**、能被 `tomllib` 正常解析

**踩过的坑**：用 PowerShell 5.1 的 `Set-Content` 默认会写成 UTF-16 带 BOM，TOML 解析必然失败。手工写配置必须显式指定 UTF-8 无 BOM：
```powershell
[IO.File]::WriteAllText($p, $text, (New-Object Text.UTF8Encoding($false)))
```

---

## 三、问题 2：配置能读了，但 MCP/电脑操作功能报错（cc-switch 模板含失效路径）

**现象**：Codex 能启动了，但 `notify` 钩子和 `node_repl` MCP 服务器会因路径不存在而失败。

**分析**：cc-switch 写出的 `config.toml` 里有一批路径指向**旧版本的目录哈希**，这些目录在 Codex 更新后已被删除：

| 配置里的旧路径 | 实际是否存在 |
|---|---|
| `cua_node\b63ee7ee40c23b77\...\codex-computer-use.exe` | ❌ |
| `cua_node\b474a88d5d105afa\bin\node_repl.exe` / `node.exe` | ❌ |
| `bin\8e5b6932251c2c1c\codex.exe` | ❌ |
| `browser\26.901.51231\scripts\browser-service.mjs` | ❌ |

本机真实存在的是 `cua_node\154806497bb51bae`、`bin\de8a38d2100ae498`、`browser\26.928.31416`。也就是说，**cc-switch 存的模板本身已经过期**——这是后面反复出问题的根子。

**解决**：分两层修，只改路径、不动其他配置。
- **第一层（config.toml）**：按上表替换 6 处路径（`notify` 钩子、MCP `command`、`CODEX_CLI_PATH`、`NODE_REPL_NODE_PATH`、`NODE_REPL_NODE_MODULE_DIRS`、`NODE_REPL_TRUSTED_CODE_PATHS`、`BROWSER_USE_CODEX_APP_VERSION`）
- **第二层（cc-switch 数据库）**：改 `settings.common_config_codex`、`mcp_servers.node_repl`、`providers.codex-official` 三张表里的同一批哈希，否则下次一点"应用"就又写回坏路径

**技术细节**：`notify` 那行用的是双反斜杠转义写法（`cua_node\\b63ee...`），普通字符串替换匹配不上，得用正则按哈希替换。改完用 `PRAGMA integrity_check` 确认数据库完好。

---

## 四、问题 3：43 个内置技能文件被断电打成空文件

**现象**：`~\.codex\skills\.system\` 下 imagegen / openai-docs / skill-creator / skill-installer / review-agent 五个技能共 43 个文件全 0 字节，写入时间与 `config.toml` 同一时刻（23:10:21）。

**分析**：这些是 **Codex 自带的系统技能**，不是用户数据。我扫描了 `codex.exe` 二进制，确认技能内容**内置在可执行文件里**（字符串里能看到 `create system skills subdir`、`write system skill file`、`skills.codex-system-skills.marker`、`imagegen` 等）。旁边那个 `.codex-system-skills.marker` 是"内置技能指纹"，只要它有效，应用就认为"已释放过、无需重写"——**所以文件坏了它也不管**，这才是关键。

**解决**：把整个 `.system` 目录**改名**（不是删除，方便回退）：
```powershell
Rename-Item "$env:USERPROFILE\.codex\skills\.system" "system.bak-20261002"
```
然后启动一次 Codex，它会重新释放。**验证结果：49 个文件、全 0 损坏 0 个** ✅

---

## 五、问题 4：左侧"项目"分区整个消失，项目名称一个都不见

**现象**：Codex 能正常启动、对话也能看到，但侧边栏"项目"下面写着"**没有项目**"，c1 / c2 / READ / PythonGEE 全部不见。

**分析**：这是本次最难的一环，因为它**不在数据库层**。查证过程：
- 数据库 `projects` 表里 **5 个项目都在** ✅
- 应用日志也证明它读到了：`Local app-server project migration completed projectCount=5`
- 但 `.codex-global-state.json` 从 95 KB 缩成 **3.4 KB（只剩 20 个键）**，其中：

| 键 | 断电后 | 作用 |
|---|---|---|
| `local-projects` | `{}` ← **侧边栏就是读它** | 项目列表数据源 |
| `thread-project-assignments` | 整个键不存在 | 对话归属映射 |
| `project-order` / `active-workspace-roots` / `projectless-thread-ids` | 全部不存在 | 顺序与分组 |
| `app-server-project-id-by-legacy-project-id-by-host` | 不存在 | 新旧 ID 对照 |

我读了应用代码确认：侧边栏项目列表构建函数取的就是 `local-projects`。**所以无论怎么修数据库，侧边栏都读不到**——必须重建状态文件。

**解决**：用"旧状态文件 + 数据库"双权威源重建这些字段。
- 数据来源一：9/26 那份**完好的旧状态文件**（`..codex-global-state.json.tmp-…`，95 KB，断电时正在写而留下的临时文件）——里面有原始的 4 个项目、`project-order`、14 条归属映射、legacy→新 ID 对照表
- 数据来源二：数据库（项目、根目录、线程归属的最终事实）
- 写入方式：原子写入（临时文件 + rename）、UTF-8 无 BOM、保留所有未知键

**这里踩了一个大坑**：我第一次写入时 **Codex 正在运行**，它的内存里持有旧状态，一保存就把我的内容盖掉了（13605 字节 → 3514 字节，`local-projects` 又被清空）。**教训：写这个文件前必须确认 Codex 完全退出**（包括托盘；只有 `codex-windows-sandbox-service` 是常驻服务，可以忽略）。

**第二个坑**：我第一版脚本里**手写了 12 个线程 ID**（从错乱编码里反推的），实际都不存在。后来改成**全部从数据库取真实 ID** → 无效 0 条。教训：这种唯一标识一律不要手写。

**还有个衍生问题**：我重建时误把旧文件的 4 个 **legacy 条目也写进了 `local-projects`**（变成 9 个条目），用户下次打开 Codex 时，应用按这些条目**又新建了 4 个重复的空项目**。用 `project/delete` 删掉，并把 `local-projects` 改成**严格只来自数据库**（5 个）。

**防复发验证（影子测试）**：把 `state_5.sqlite` + 当前状态文件复制到隔离目录，用它启动一次 app-server，结果 **项目数 5 → 5 未变**、名称不变，证明不会再产生重复项目。

---

## 六、问题 5：项目里没有对话（46 条线程归属丢失）

**现象**：项目（即使能看到）点进去是空的，"项目 X 里 0 条对话"。

**分析**：`threads` 表 **46 条记录的 `project_id` 全部为空**。旧状态文件里保存着迁移进度：

```json
"app-server-projects-migration-by-host": {
  "local:C:\\Users\\34936\\.codex": {
    "projectsMigrated": true,            // 项目迁完了
    "threadAssignmentsMigrated": false   // 线程归属没迁完
  }
}
```

**断电正好打断在"项目迁移成功、线程归属未开始"的位置**。我更读了应用的 `migrate()` 代码，逻辑是：它只把"旧映射表里已有的归属"推送进数据库——而映射表已被清零，所以**它跑了个空**，而且跑完还会把标记写成"已完成"，之后再也不会自动补。

**可行性验证（只读，先确认数据是否还在）**：

| 核对项 | 结果 |
|---|---|
| 数据库线程数 / 磁盘 rollout 文件数 | 46 / 46 |
| 双向孤儿（磁盘有库中无、库中有磁盘无） | 0 / 0 |
| 文件名 UUID == `threads.id` | **46/46 完全一致** |
| 内容抽样 10 条（98 KB ~ 7 MB） | 4133 行、坏行 0，全部可正常解析 |
| 字段完整性（cwd/标题/时间戳/model） | 无缺失；`archived = 1` 为 0 |

**结论：对话数据一条没丢，丢的只是"哪个对话属于哪个项目"这张映射表。**

**解决**：走应用自己的接口 `thread/metadata/update`（不是直接改数据库），顺序是：
1. `project/delete` 删掉重复项目（`accuracy` 与 `c2` 根目录完全相同）
2. `project/create` 新建 `PythonGEE`（旧状态里它是独立项目，数据库里已丢失）
3. 按"当前工作目录 → 项目根目录"匹配，逐条 `thread/metadata/update` 写归属：先 38 条，再给 `c1` 追加 6 个归档会话目录后补完剩余 8 条
4. 最后把状态文件同步成与数据库一致

**最终结果**：`c1` 21 条、`c2` 17 条、`PythonGEE` 7 条、`READ` 1 条，**合计 46 条，未归属 0 条**。

---

## 七、可复用的判断方法与经验

**判断"是不是断电导致的数据损坏"**：文件大小正常、内容却是**全 0x00**，且时间戳集中在同一个时刻，同时事件日志有 `Kernel-Power 41` / `EventLog 6008`。这是 NTFS 未落盘写入的典型特征，不要往"病毒/软件 bug/硬盘坏道"方向查。

**排查顺序（先只读、后动手）**：事件日志看断电 → 应用日志定位报错文件 → 数据库核对数据是否还在 → 状态文件核对 UI 数据源 → 最后才改。**每一步先确认"数据还在不在"，再决定"要不要修"**，这样能避免把没坏的东西也动了。

**三条硬教训**：
1. **改配置文件前先确认应用完全退出**——否则内存里的旧状态会把你的修改覆盖掉，而且覆盖时**不留痕迹**（文件大小变回去、时间戳更新）
2. **唯一标识（线程 ID、项目 ID）一律从数据库/接口读，绝不手写**
3. **写 JSON/TOML 一律 UTF-8 无 BOM**，PowerShell 5.1 的默认编码会把文件写坏

**工具位置备忘**（本机）：
- sqlite3：`D:\Software\sf1\miniconda\Library\bin\sqlite3.exe`
- Python：`D:\Software\sf1\miniconda\python.exe`
- Codex 自带 app-server：`%LOCALAPPDATA%\OpenAI\Codex\bin\de8a38d2100ae498\codex.exe app-server`
- 协议定义：`codex.exe app-server generate-json-schema --experimental --out <目录>`

---

## 八、备份清单（本次恢复过程中建立）

| 备份 | 内容 | 用途 |
|---|---|---|
| `~\.codex\config.toml.corrupt-20261001` | 4316 字节全 0 的坏文件 | 留证 |
| `~\.codex\config.toml.bak-20261002` | 修正路径前的配置 | 回退配置层 |
| `~\.codex\state_5.sqlite.bak-20261002` | 改归属前的数据库 | 回退数据库 |
| `~\.codex\.codex-global-state.json.bak-project-restore` | 动手前的空项目状态 | 回退状态文件 |
| `~\.cc-switch\cc-switch.db.bak-20261002` | 修正旧路径前的 cc-switch 库 | 回退源头模板 |
| `~\.codex\skills\.system.bak-20261002` | 改名前那批损坏技能 | 确认重建无误后可删 |

---

## 九、待办与提醒

1. **不要轻易在 cc-switch 里点"切换/应用"**：这个动作会用 cc-switch 数据库里的模板**覆盖** `config.toml`。本次已修好模板里的旧路径，但如果以后再更新 Codex 导致运行时目录哈希变化，点一下就会把失效路径写回来。真要动之前，先备份 `config.toml`。
2. **`config.toml` 至今没有自动备份机制**（`.codex-global-state.json` 有 `.bak`，配置没有）。建议手工做一份带日期戳的快照。
3. **`c1` 项目现在混了两类目录**：项目目录 `2026-09-08` + 6 个按日期归档的会话目录。如果希望归档那批单独成组，可以用 `project/create` 单建一个"Codex 归档"项目再挪过去。
4. **考虑给笔记本接电源或用 UPS**：这次是笔记本断电，一次性损坏了 4 类文件，恢复花了不少功夫。
