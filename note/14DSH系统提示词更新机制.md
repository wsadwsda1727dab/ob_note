# DSH 系统提示词更新机制：追加与替换两条路径

> 文档名称：DSH 系统提示词更新机制：追加与替换两条路径
> 适用范围：DSH 中「系统提示词在中途变化时如何写入请求」的机制说明，含判定字段、两条分支、逐字节差异、缓存影响、优劣对照与排查方法
> 适用对象：本库作者，以及协助分析 DSH 会话请求、提示词缓存与插件影响的 AI
> 实测宿主：DSH Desktop 内置 `@deepseek-ai/dsh` = `0.2.1-rc.2`
> DSH_HOME：`%USERPROFILE%\.dsh`
> 当前 profile：`desktop`（provider `nexora`，协议 `openai-responses`）
> 执行环境：Windows PowerShell
> 最近更新：2026-10-25
> 关联文档：[[00note总览]]、[[07DSH插件安装]]、[[13DSH插件市场自更新限制]]

**目录**

- [[#1. 结论速览]]
- [[#2. 判定输入：一个字段、两处代码]]
- [[#3. 两条分支的实现]]
- [[#4. 逐字节对比]]
- [[#5. 缓存影响]]
- [[#6. 优劣对照]]
- [[#7. 实测：本次会话]]
- [[#8. 会与不会改变系统提示词的因素]]
- [[#9. 插件对系统提示词的影响]]
- [[#10. 常见误解澄清]]
- [[#11. 复现方法]]
- [[#12. 变更记录]]

## 1. 结论速览

同一份 agent 配置，会因**所走适配器是否声明 `systemPromptUpdate: "in-history"`**，把「系统提示词变了」落成两种不同的历史改写。

| 路线 | 适配器 | 落法 | 请求里的表现 |
|---|---|---|---|
| 官方直连 | `deepseek-official`，Anthropic Messages | 追加 | 旧提示词留在 `system` 参数，新版作为 messages 末尾的 system 条目 |
| 第三方中转 | `nexora`，`openai-responses` | 替换 | surface 第 0 个节点就地换成新版，messages 里没有 system 条目 |

两者语义等价，都以最新版为准；差异只在**线上字节**，进而决定 prompt cache 是局部失效还是整段失效。

一句话判据：看到请求里 `system` 参数被换成新文本、messages 里没有 system 条目，走的是替换路径；看到 `system` 保留旧文本、messages 尾部多一条 system，走的是追加路径。

---

## 2. 判定输入：一个字段、两处代码

### 2.1 API 层声明

适配器在解析模型时回填 `LlmResolvedModelInfo.systemPromptUpdate`。`@deepseek-ai/dsh-llm` 的 README「发现与解析模型」段原文：

> 当模型把任意位置最新的 `system` 消息读作有效系统提示词时，`LlmResolvedModelInfo.systemPromptUpdate` 为 `'in-history'`；只读取开头 system 消息时该字段缺失；`normalizeModelInfo` 以 `INVALID_MODEL_INFO` 拒绝任何其他值。

即只有 `'in-history'` 与「缺失」两种合法状态，没有第三态。

官方适配器直接在序列化时按字段判定，见 `dsh-llm-deepseek/lib/index.js:1592`：

```js
const inHistory = model?.systemPromptUpdate === "in-history";
```

第三方适配器 `dsh-llm-pi-ai` 则先查协议能力门，见 `dsh-llm-pi-ai/lib/index.js:785`：

```js
function conversationUpdates(model) {
	const compat = model.compat;
	if (compat === void 0 || !("supportsMidConvoSystemMessages" in compat) || !compat.supportsMidConvoSystemMessages) return {};
	const updates = { systemPromptUpdate: "in-history" };
	// …
}
```

`openai-responses` 路由的兼容门 `RESPONSES_COMPAT_GATE` 把 `supportsMidConvoSystemMessages` 设为 `"withhold"`，字符串为真，但语义是「不声明」，于是 `conversationUpdates` 返回空对象，`systemPromptUpdate` 与 `toolUpdate` 均为 `undefined`。`openai-completions`、`anthropic-messages`、`mistral-conversations` 同样如此。

### 2.2 agent 层消费

`dsh-agent-loop/lib/index.js:1057` 把判定结果传进投影：

```js
const commits = this.systemPrompt.project(renderedPrompt, {
	inHistory: preparedCall?.systemPromptUpdate === "in-history",
	startsSeries: startsRequestSeries || this.requestSurfaceGeneration !== this.session.surface.contentGeneration || preparedCall?.toolUpdate === void 0 && this.toolsChanged(assembly.tools)
});
```

第三方路由下 `inHistory` 恒为 `false`，且 `startsSeries` 因 `toolUpdate === undefined` 而在工具集变化时被置真。

---

## 3. 两条分支的实现

投影逻辑在 `dsh-agent-loop/lib/index.js:267` 的 `project()`：

```js
const latest = nodes.findLast((node) => node.text !== "") ?? head;
if (!input.inHistory || input.startsSeries || rendered.length === 0) {
	const updates = nodes.slice(1).filter((node) => node.text !== "").map((node) => this.replace(node.seq, ""));
	if (head.text !== rendered) updates.push(this.replace(head.seq, rendered));
	return updates;
}
if (latest.text === rendered) return [];
return [{ message: createSystemMessage(rendered), intent: { surfaceOp: "append" } }];
```

三个分支的含义：

- 第一个分支（`:272`）：把第 0 个节点替换成新文本，并把其余非空 system 节点置空
- 第二个分支（`:276`）：文本未变，返回空列表，什么都不写
- 第三个分支（`:280`）：追加一条新 system 消息

分叉条件是 `!input.inHistory`，即「协议不支持中途系统消息」时一律走替换。

### 3.1 写入表面的约束

`dsh-session/lib/index.js:419` 的 `assertSystemHeadRewrite` 限制 node 0 只能被 `system/message` 覆盖，且只能覆盖它自己：

```js
if (event.type !== "system/message" || shadowedSeqs.length !== 1) throw new Error("surface replace: node 0 holds the system prompt and may be rewritten only by a system/message over exactly that node");
```

`deriveMessages()`（`dsh-session/lib/index.js:1672`）按 `surface.nodes` 顺序重建消息，替换型 op 让新节点 shadow 旧节点。因此替换后旧文本仍留在事件日志里，只是不再进入请求；用户消息与助手消息不受这条逻辑影响。

---

## 4. 逐字节对比

设首轮 system 文本为 `S1`，中途重渲染出 `S2`，历史为 `[user: hi, assistant: hello, user: next]`。

官方追加路径的请求体：

```json
{
  "system": "S1",
  "messages": [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},
    {"role": "user", "content": "next"},
    {"role": "system", "content": "S2"}
  ]
}
```

中转替换路径的请求体：

```json
{
  "system": "S2",
  "messages": [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello"},
    {"role": "user", "content": "next"}
  ]
}
```

三处差异：

- `system` 参数的内容，`S1` 还是 `S2`
- `messages` 里有没有 system 条目
- 请求第 0 个字节是否变化，决定缓存是否整段失效

`messages` 中的历史消息两者完全相同。

### 4.1 官方序列化

`dsh-llm-deepseek/lib/index.js:1662` 起处理 system 消息：

```js
if (message.role === "system") {
	const texts = message.content.filter((block) => block.type === "text");
	if (texts.length !== message.content.length) return unsupported("non-text system message");
	const text = texts.map((block) => block.text).join("");
	if (inHistory && messages.length > 0) {
		if (text.length === 0) return unsupported("empty in-history system update");
		systemUpdates.push({ role: "system", content: [{ type: "text", text }] });
	} else historySystem = text;
	continue;
}
```

随后 `:1714` 把首个 system 与请求级提示词合并：

```js
const system = [options.system, historySystem].filter(Boolean).join("\n\n");
```

`flushSystemUpdates()`（`:1625`）要求缓冲中的更新紧跟在 user 或 tool-result 之后，否则报 `system update without a preceding user or tool-result turn`；遇 assistant 前先 flush（`:1685`），末尾再 flush 一次（`:1694`）。

### 4.2 中转序列化

`dsh-llm-pi-ai/lib/index.js:1273`：

```js
function splitSystemPrompt(options) {
	const text = [options.system];
	let end = 0;
	for (const message of options.messages) {
		if (message.role !== "system") break;
		text.push(flattenText(message));
		end++;
	}
	return {
		systemPrompt: text.filter(Boolean).join("\n\n") || void 0,
		messages: options.messages.slice(end)
	};
}
```

只把**开头连续**的 system 并进顶层 `systemPrompt`，后续 system 由 `appendSharedHistory()` 留在 messages 里。替换路径不产生后续 system，这个分支实际用不到。

---

## 5. 缓存影响

prompt cache 是**最长公共前缀匹配**：从第一个 token 开始比对，遇到第一处不同，其后全部作废。

| 项 | 官方追加 | 中转替换 |
|---|---|---|
| 提示词变化时的失效范围 | 从新 system 条目起 | 从第 0 个字节起 |
| 典型代价（5K 提示词 + 30K 历史） | 约 5K | 约 35K |
| 触发频率 | 仅在提示词文本真的变化时 | 同左 |

关键在最后一行：**不是每轮都赔，而是每次提示词变化赔一次**。提示词整场不变时，两条路径产出的请求体逐字节相同，差异一次都不触发。

---

## 6. 优劣对照

| 维度 | 官方追加 | 中转替换 |
|---|---|---|
| 缓存前缀 | 稳定，只赔增量 | 每次提示词变化归零 |
| 模型可见指令 | 能感知指令演进，旧版仍在历史 | 只看到最新版 |
| 旧指令残留 | 会残留，只能等 series 重置时清扫 | 不存在 |
| 上下文窗口 | 随提示词更新线性增长 | 恒定 |
| 依赖服务端特性 | 依赖服务端正确处理中途 system | 不依赖任何服务端特性 |
| 第三方中转兼容性 | 中转可能静默降级或报错 | 最稳 |
| 工具集变化 | 声明 `addition-only` 时新增工具不重置 series | 一律触发 series 重置 |

追加路径的唯一清扫时机是 `project()` 第一个分支，即换模型、断路或提示词清空时才把中间节点置空；追加分支本身不清扫，这是「旧指令残留」的来源。

对本机而言走的是第三方中转，无法确认上游是否把中途 system 当有效系统提示词处理，因此替换是保守但正确的选择，这也解释了 `openai-responses` 一律不声明该能力的做法。

---

## 7. 实测：本次会话

### 7.1 事件统计

| 项 | 值 |
|---|---|
| `system/message` 事件 | 1 条，seq 7，12459 字符 |
| 提示词是否变过 | 否，全程 `S1` |
| `request/header` | 7 条，全部 `initial` 或 `resume`，无 `change` |
| `request/context` | 1 条，seq 16，`contextWindow` 为 996000 |
| 工具数 | 恒定 59 |
| 实际生效路径 | 中转替换，但因文本未变而未执行替换 |

`request/header` 只在 header 变化时新增，比较基准是 `canonicalHeader()`（`dsh-session/lib/index.js:604`）与 `headerEquals()`（`:622`），只比 `config`、`adapterDefaults.reasoningEffort`、`adapterDefaults.maxTokens` 与工具 schema 顺序。

### 7.2 为什么差异没有触发

整场会话里 system 文本逐字节未变，`project()` 落到第二个分支返回空列表，一条 `system/message` 都没新增。因此追加与替换在本会话产出的请求体完全一致，缓存损失为 0。

要让差异真正发生，必须触发一次 `renderPrompt()` 输出变化，例如切换 provider 或 model、改变工具集、启用或停用注册了 section 的插件。

---

## 8. 会与不会改变系统提示词的因素

### 8.1 判定基准

「提示词不变」指 `renderPrompt()` 每步重渲染后的文本与上一版逐字节相同，由 `if (latest.text === rendered) return []` 短路。它与「发了几轮」「一轮里调了几次 API」都无关。

`dsh-system-prompt/lib/index.js:117`：

```js
function renderPrompt(assembly) {
	return assembly.sections.map((section) => section.interpolate === false ? section.text : interpolate(section, assembly.variables, "section")).filter((text) => text.length > 0).join("\n\n");
}
```

### 8.2 会改变的情形

- `persona` 的 `prefix`，预设值见 `dsh-web-app/presets/standard.patch.yml:11`：`You are a coding agent powered by the {{model}} model.`
- `{{model}}` 与 `{{provider}}` 变量，由 `dsh-agent-loop/lib/index.js:1591` 的 `ctx.systemPrompt.variable(...)` 注册
- `plan:policy`、`tool:*` 集合、`mcp-resource-servers`
- `context:file-reference`、`deployment:persona-prefix`、`harness:identity`
- 注册了 section 的插件，见第 9 章
- 切换 provider 或 model，这是本机会真的产生 `S2` 的操作

### 8.3 不会改变的情形

- 用户消息、助手消息、工具调用与结果
- AGENTS.md 内容与更新
- `runtime-context` 快照、`time-context`、技能目录、记忆注入

### 8.4 与 AGENTS.md 的边界

这是最容易混的一点：系统提示词与 AGENTS.md 是两条独立注入路径。

| 项 | 系统提示词 | AGENTS.md |
|---|---|---|
| 生成 | `renderPrompt(assembly)`，每步一次 | `dsh-agent-instructions` 生成 |
| 落点 | surface node 0，`system/message` | `user/message` 的 `agent-instructions` 来源 |
| 协议位置 | 顶层 `system` 参数 | messages 里的一条 user 消息 |
| 变化时 | 触发 `S2`，按适配器追加或替换 | 仅追加，永不触发 `S2` |
| 本次会话实例 | seq 7，12459 字符 | seq 9 与 seq 1156，各约 3.6K 字符 |

实测交叉验证：把 seq 9 与 seq 1156 的全部较长行逐条到 seq 7 中检索，命中 0 条；在 seq 7 中检索 `AGENTS.md`、`Instructions from`、`最小改动`、`安全边界` 等标志串，同样全部为否。二者互不包含。

---

## 9. 插件对系统提示词的影响

在本机 profile 的 `node_modules` 下检索注册调用 `systemPrompt.section(`，只命中一个插件：

```text
node_modules/dsh-ops/lib/index.js
```

`dsh-ops/lib/index.js:116` 注册的 section：

```js
ctx.effect(() => ctx.systemPrompt.section({
	name: `${TOOLING_SECTION}:${component}`,
	order: SECTION_ORDERS[TOOLING_SECTION] + (component === 'shell' ? 1 : 0),
	interpolate: false,
	text: context => renderToolingPolicy({ /* … */ })
}));
```

文本内容由 `dsh-ops/lib/policy.js:33` 按当前可见工具集决定标题，无相关工具时返回空串：

```js
if (!hasFiles && !hasBash && !hasPwsh && !hasDiscovery) return '';
const heading = hasBash ? '# Shell routing: bash → host PowerShell'
	: hasFiles || hasDiscovery ? '# Repository file tools' : '# Windows-native shell';
```

由此得到四个结论：

- 只有**注册了 section** 的插件才影响系统提示词，其余插件不注册
- 工具本身走 `request/header.tools`，不进入提示词正文
- 即便注册了，文本取决于当前可见工具集，小版本升级通常不改变输出
- 新装一个不注册 section 的插件，系统提示词不变

---

## 10. 常见误解澄清

| 说法 | 判定 | 实际 |
|---|---|---|
| 新提示词就是用户发送的对话 | 错 | 是 `renderPrompt()` 重渲染出的 system 文本；用户对话永远是独立 user 消息 |
| 两种做法的差别是新增内容的替换方式 | 错 | 差别是整个 `system` 参数的内容与 messages 里有无 system 条目 |
| 提示词不变指一轮内多次 API 调用 | 错 | 指每步重渲染后的文本逐字节相同，与轮次无关 |
| 替换只换 `S2`，前面的上下文就没了 | 错 | 只替换 surface node 0；历史由 `deriveMessages()` 全量重建，一条不少 |
| 本机与官方提示词内容不同 | 部分错 | 模板相同，唯一差异是 `{{model}}` 插入的字符串，且只在首条请求确定 |
| 本机提示词里含 AGENTS.md | 错 | 见第 8.4 节，两条独立路径，交叉检索命中 0 |

---

## 11. 复现方法

以下命令均在 Windows PowerShell 中执行，临时文件放在 `%TEMP%`。

### 11.1 解压会话日志

会话日志 `session.v4.jsonl.zstd` 是多帧 zstd，`zstdDecompressSync` 只解第一帧，需要循环「二分查找消费长度再前移」：

```js
const fs = require("fs"), zlib = require("zlib");
const buf = fs.readFileSync(process.argv[2]);
let off = 0;
const out = [];
while (off < buf.length) {
	const rest = buf.subarray(off);
	let lo = 1, hi = rest.length, best = -1;
	while (lo <= hi) {
		const mid = (lo + hi) >> 1;
		try { zlib.zstdDecompressSync(rest.subarray(0, mid)); best = mid; hi = mid - 1; }
		catch { lo = mid + 1; }
	}
	if (best < 0) break;
	out.push(zlib.zstdDecompressSync(rest.subarray(0, best)).toString());
	off += best;
}
fs.writeFileSync("session.jsonl", out.join(""));
```

### 11.2 统计关键事件

```powershell
node -e "const fs=require('fs');const c={};for(const l of fs.readFileSync('session.jsonl','utf8').split('\n')){if(!l)continue;try{const e=JSON.parse(l);c[e.type]=(c[e.type]||0)+1}catch{}}console.log(c)"
```

关注 `system/message`、`request/header`、`request/context` 三类计数。

### 11.3 从 app.asar 提取源码

asar 头布局：`offset 4` 为 pickle 长度，`offset 8` 为 JSON 长度，JSON 起点为 `offset 16`，文件内容基址为 `8 + pickleLength`。本机实测 pickle 长度 4779788、JSON 长度 4779780、共 18309 条目。

```js
const fs = require("fs");
const A = "<安装目录>/resources/app.asar";
const fd = fs.openSync(A, "r");
const h = Buffer.alloc(16);
fs.readSync(fd, h, 0, 16, 0);
const pickle = h.readUInt32LE(4), jsonLen = h.readUInt32LE(8);
const j = Buffer.alloc(jsonLen);
fs.readSync(fd, j, 0, jsonLen, 16);
const tree = JSON.parse(j.toString("utf8"));
const base = 8 + pickle;
const node = process.argv[2].split("/").reduce((n, p) => n && n.files && n.files[p], tree);
const out = Buffer.alloc(Number(node.size));
fs.readSync(fd, out, 0, Number(node.size), base + Number(node.offset));
fs.writeFileSync(process.argv[3], out);
```

注意 asar 内的顶层目录名是 `dsh`，包路径须写成 `dsh/node_modules/@deepseek-ai/...`。

### 11.4 路径陷阱

本机真实安装目录名带空格，写成连字符形式会失败。`Test-Path` 对连字符形式可能误报 `True`，判定存在性一律用 node 的 `fs.existsSync`。

---

## 12. 变更记录

| 日期 | 版本 | 变更 |
|---|---|---|
| 2026-10-25 | 1.0.0 | 初版，基于 DSH Desktop `0.2.1-rc.2` 与一次完整会话的实测 |

---
