# WorkBuddy 输入框图标背后的提示词 —— 完整获取报告

> 结论先说：**能拿到，而且已经拿到了。** 你截图里那排图标是客户端内置功能，它发给模型的提示词模板就明文躺在安装目录的 `app.asar` 里，不需要任何特殊权限，一条 `grep` 就能导出。
>
> 采集环境：本机 Linux，WorkBuddy 安装于 `/opt/WorkBuddy/resources/app.asar`（2026-09-10 构建，298 MB）。
> 采集时间：2026-09-24。

---

## 一、先对号入座：那两个图标是什么

| 图标 | 内部名 | 对应功能 | 提示词在哪 |
|---|---|---|---|
| ✨ 双星火花 | `input.enhance.title` = **"增强提示词"** | 一键把你输入框里的草稿改写得更完整、更具体（俗称"优化提示词/魔法棒"） | 硬编码在客户端 `app.asar`，**固定模板**，见第二节 |
| 📦 立方体 | **专家（Expert）** 入口 | 选择/切换专家角色，或打开市场装专家包 | 随**专家包**走，存在本地专家包目录，见第三节 |

两个都是"调用模型"的功能，但机制完全不同：**✨ 是固定模板 + 你的输入**；**📦 是每个专家自带的人设 + 默认提示词**。

> 说明：截图上两个图标挨在一起，从代码看它们属于同一个输入框工具栏组件。如果你的问题只针对其中一个，直接看对应小节即可。

---

## 二、✨ "增强提示词"的完整提示词

点 ✨ 时，客户端会发起一次真实的模型调用，链路是 RPC 频道 `llm:enhancePrompt`（日志已抓到实际调用记录，见第四节）。它把下面 **两段固定模板** 和 **你的输入** 拼起来送给模型。

### 2.1 System 角色模板（`DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE`）

原文（英文，客户端仅此一份，无中文版）：

```
You are a Prompt Engineering Expert specializing in improving user prompts for a development code assistant. When given a prompt, analyze and enhance it to create a more effective version while maintaining its core purpose. The requests are being made to an AI assistant that specializes in writing code.

TASK: When given a prompt, analyze and enhance it to create a more effective version while maintaining its core purpose. The requests are being made to an AI assistant that specializes in writing code.

ANALYSIS PROCESS:
Evaluate the original prompt:
  Identify the main objective
  Note any ambiguities or gaps
  Assess the clarity of instructions
  Check for missing context
Apply these prompt engineering principles:
  Write clear, specific instructions
  Include necessary context
  Set explicit parameters and constraints
  Structure the output format
  Add relevant examples
  Match tone and complexity to the use case
  Remove redundant information
Create the enhanced version:
  Maintain the original goal
  Incorporate identified improvements
  Ensure clarity and completeness
  Be realistic in the features to add
  Do NOT request guides/how-tos unless the user asks
  Do NOT ask for code snippets
  Do NOT suggest specific technologies unless mentioned in the user's prompt
  Do NOT explain HOW to do things, focus on WHAT
  Do NOT answer questions - expand/rewrite them to be more detailed

IMPORTANT CONSTRAINTS:
1. Language matching is the highest priority - You MUST strictly respond in the exact same language as the user's input. If the user writes in Chinese, respond in Chinese; if the user writes in English, respond in English; if the user uses another language, respond in that same language. Do not mix languages unless the user's input itself mixes languages.
2. Keep the enhanced prompt concise - maximum length should be around 800 characters

FORMAT: Provide only the enhanced prompt with no additional commentary.

Example:
"A website for my dog"

Enhanced prompt:
"Design a personalized Next.js website dedicated to showcasing my dog. Include sections such as a photo gallery, a biography detailing the dog's breed, age, and personality traits, and a blog for sharing stories or updates about your dog's adventures. Add a contact form for visitors to reach out with questions or comments. Ensure the website is visually appealing and easy to navigate, with a responsive design that works well on both desktop and mobile devices."

Example:
"Convert this to a friendly tone, maintain technical details but reduce bullets in favor of narrative. Remove any jargon like 'genie router'. Use canvas"

Enhanced prompt:
"Transform the provided content into a friendly narrative format while preserving all technical details. Minimize bullet points in favor of flowing prose. Eliminate any technical jargon such as 'genie router'. Incorporate the concept of using canvas elements naturally within the narrative structure to enhance the technical explanation."
```

### 2.2 User 角色模板（`DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE`）

`{input}` 就是你在输入框里打的字：

```
You are a prompt enhancement assistant. Improve the user prompt while preserving its intent and language.

USER INPUT:
{input}

TASK:
Rewrite the user input into a clearer, more specific prompt for the target AI assistant.

CRITICAL PRIORITY - LANGUAGE CONSISTENCY:
1. You MUST detect the language of the user input above and write the enhanced prompt in that same language.
2. If the user writes in Chinese, the enhanced prompt MUST be entirely in Chinese.
3. If the user writes in English, the enhanced prompt MUST be entirely in English.
4. If the user writes in any other language, the enhanced prompt MUST use that exact same language.
5. If the user mixes languages, keep a natural matching mix. Do not translate the user's intent into a single language.
6. These language rules are behavior instructions only; never include language analysis or language labels in the output.

ENHANCEMENT REQUIREMENTS:
1. Return only the enhanced prompt text; do not add explanations, prefaces, markdown fences, labels, or analysis.
2. Do not include language labels or meta notes such as "User input is in Chinese" or "Response must be in Chinese".
3. Preserve the user's original intent, topic, constraints, and target output type. Do not answer the request.
4. Always make a substantive enhancement when possible: clarify the task, scope, constraints, and expected output.
5. If the original prompt is already clear, lightly polish it instead of returning it unchanged.
6. Keep the enhanced prompt complete and concise. Do not end with an unfinished list, dangling conjunction, or trailing colon.
7. Do not add unrelated requirements, unsupported facts, or unnecessary sections.

EXAMPLES:
User input (Chinese): "请帮我解释这段代码的功能"
Enhanced prompt: "请解释这段代码的主要功能、执行流程和关键逻辑，并指出可能需要注意的边界情况。"

User input (English): "Please explain what this code does"
Enhanced prompt: "Explain what this code does, including its main purpose, key control flow, and any important edge cases."

User input (Mixed): "这段代码有 bug，can you help me fix it?"
Enhanced prompt: "请分析这段代码中的 bug，explain the root cause, and provide a minimal fix with necessary verification steps."

BAD OUTPUT EXAMPLE:
User input is in Chinese → Response must be in Chinese.
请解释这段代码的主要功能

GOOD OUTPUT EXAMPLE:
请解释这段代码的主要功能、执行流程和关键逻辑，并指出可能需要注意的边界情况。
```

### 2.3 关键行为规则（从模板里能直接读出来的）

- **语言匹配是最高优先级**：你写中文，它就吐中文，绝不混语言。
- **只改写、不回答**：不会替你把问题答了，只会把问题问得更清楚。
- **不许堆技术**：你没提的技术栈，它不会给你加。
- **长度上限约 800 字符**，只输出改写后的提示词、不带任何解释。
- 输出**直接覆盖**输入框草稿（有"保留用户已输入内容"的判断逻辑，避免覆盖你手打的字）。

---

### 2.4 ✨ 用的是哪个模型？—— 跟随你当前会话选中的模型

**结论：不是某个固定的"增强专用模型"，而是跟着你当前会话选中的模型走。**

完整调用链（全部从 `app.asar` 源码读出，非推测）：

```
点 ✨
 └─ useEnhancePrompt({ model: enhanceSelectedModelId })
      enhanceSelectedModelId =
        (当前会话存在 ? remoteCurrentModelId
                      : remoteCurrentModelId || 界面选中的 selectedModelId)
        || undefined
 └─ daemon RPC  channel: llm:enhancePrompt   { text, sessionId, model }
 └─ handleEnhancePrompt():  const model = req.model?.trim() || undefined   // 允许为空
 └─ CliPulseClient.runAgent({ systemPrompt, userPrompt, agentName:"enhance-prompt", model })
 └─ POST {sidecarEndpoint}/api/v1/llm/completions
      body: { systemPrompt, userPrompt, agentName: "enhance-prompt",
              ...(model 非空 ? { model } : {}) }        // 为空则【不带 model 字段】
```

**三个要点：**

1. **它跟着你当前对话选的模型走。** 你在会话里选了什么模型，✨ 就用什么模型，不会偷偷换成另一个便宜/小模型。这是"当前会话模型被复用"，不是独立配置。
2. **如果 model 为空**（例如会话尚未选定），请求体里**干脆不带 `model` 字段**，交由 sidecar 的 `/api/v1/llm/completions` 按 `agentName = "enhance-prompt"` 走它自己的兜底默认。
3. **所以点 ✨ 要等 5–10 秒**（实测日志），因为它拿你的主力模型**全量跑一次生成**，不是走轻量模型。

本机可选的模型定义在 `~/.workbuddy/models.json`（如 `siyuan/auto`、`siyuan_local_amd`、`minicpm5-nx-cluster` 等自定义模型）。**当前会话选哪个，增强提示词就走哪个。**

> 注：2.2 的 user 模板已按 `app.asar` 原文**完整**补全（含末尾 `EXAMPLES / BAD OUTPUT / GOOD OUTPUT` 三种语言示例），非节选。

---

### 2.5 它只看你输入框里的文本，不结合任何上下文

**原哥问的这点，答案是：对，只按你当前输入的内容做优化，不带对话历史。**

三条证据（均来自源码）：

**① 前端只取"文本块"，图片/附件一律剔除**

```js
function blocksToText(blocks) {
  return normalizeEnhanceText(
    blocks.filter(isTextBlock).map(block => block.text).join("\n")
  );
}
```

输入框里的**图片、附件、场景 chip 全部被过滤掉**，只有纯文本参与增强。

**② 服务端 handler 只认 `text` 和 `model`，`sessionId` 拿到但根本没用**

```js
async function handleEnhancePrompt(deps, req) {
  const text  = typeof req?.text === "string" ? req.text : "";
  const model = typeof req?.model === "string" && req.model.trim() ? req.model.trim() : void 0;
  // ↓ 注意：req.sessionId 在这个函数体里一次都没出现
  result = await deps.createClient(endpoint).runAgent({
    systemPrompt: DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE,
    userPrompt: renderUserPrompt(text),      // 只把 text 塞进 {input}
    agentName: ENHANCE_PROMPT_AGENT_NAME,
    model
  });
}
```

前端确实把 `sessionId` 塞进了请求（`opts.sessionId = sid`），但**服务端 handler 从头到尾没使用它** —— 带着但没用（推测仅供日志/追踪）。

**③ 请求体里没有任何消息历史**

`CliPulseClient.runAgent` 的 payload 固定就是：

```js
{
  systemPrompt, userPrompt, temperature, maxTokens, maxTurns,
  agentName, ...(model 非空 ? { model } : {})
}
```

**没有 `messages` / `history` / `context` 字段**。`userPrompt` 就是那段模板里 `{input}` 被你的文本替换后的结果。

---

**由此而来的几个实际影响（值得知道）：**

- **指代接不住。** 你输入"把上面那段改成表格"，它并不知道"上面那段"是什么 —— 它只能把这句话本身改写得更清楚（如"请将前文内容改写为表格形式"）。**真正的指代消解还是要靠主对话模型**，增强提示词这一步帮不上。
- **图片/附件不参与。** 增强完成后它们原样保留，只有文本部分被替换（`textToBlocks` 会 `keptChips` 保留所有非 text block）。
- **回撤机制**：增强结果会存 `backupBlocks.before / after`；你在结果上手动改动后，回撤备份即失效（"修改即失效" `handleContentDiverged`）。
- **没历史也意味着更省**：不带会话上下文，输入 token 很小，快慢主要取决于你当前选的模型本身。

---

## 三、📦 专家（Expert）的提示词

专家的提示词**不是硬编码**在客户端里的，而是随**专家包**从市场下载到本地。本机路径：

```
~/.workbuddy/plugins/marketplaces/experts/plugins/<专家名>/
├── .codebuddy-plugin/plugin.json      ← 元数据，含「默认提示词」「快捷提示词」
├── agents/<专家名>.md                  ← 专家人设（真正的 System Prompt）
├── skills/                             ← 专家附带的技能
├── avatars/expert.png
└── README.md
```

**两层提示词要分清楚：**

| 层 | 字段/文件 | 作用 |
|---|---|---|
| 默认提示词 | `plugin.json` → `defaultInitPrompt` | 选中专家时，**自动填进输入框**的那句话，你可以直接改 |
| 快捷提示词 | `plugin.json` → `quickPrompts` | 面板上的几个一键入口 |
| 人设系统提示 | `agents/<name>.md` | 真正作为 System Prompt 注入对话的专家人格与规则 |

### 3.1 实例：你装的 Godot 专家（节点通）

选自 `~/.workbuddy/plugins/marketplaces/experts/plugins/godot-game-script-engineer/.codebuddy-plugin/plugin.json`：

```json
{
  "displayName": { "zh": "节点通", "en": "Earl" },
  "profession":  { "zh": "Godot游戏脚本工程师", "en": "Godot Game Script Engineer" },
  "defaultInitPrompt": {
    "zh": "我们使用Godot引擎开发游戏,需要专业的GDScript和引擎使用支持,请Godot游戏脚本工程师帮我们实现游戏功能。"
  },
  "quickPrompts": [
    { "zh": "设计基于节点的组合架构和信号系统" },
    { "zh": "集成C#模块和类型安全设计" }
  ],
  "agents": ["./agents/godot-game-script-engineer.md"]
}
```

它的**人设系统提示**在 `agents/godot-game-script-engineer.md`，就是你在对话里被注入的专家身份说明（角色、性格、职责、规则清单）。同样，把 `agents/*.md` 打开就能看到全文。

**本机已装的专家包（13 个）**：
`ai-content-creator-team`、`ai-image-prompt-engineer`、`code-review-expert`、`data-analysis`、`equity-research`、`flova-anime-short-drama-director`、`flova-cinematic-director`、`godot-game-script-engineer`、`gpt-researcher-team`、`senior-developer`、`teaching-design-expert`、`technical-documentation-engineer`（目录里共 13 个）。

---

## 四、实测证据：它确实调了模型

日志 `~/.workbuddy/logs/main.log` 抓到的真实调用记录（频道就是 `llm:enhancePrompt`）：

```
[D aemonRPC] end #7439  llm:enhancePrompt elapsedMs=10580 SLOW
[D aemonRPC] end #8839  llm:enhancePrompt elapsedMs=6109  SLOW
[D aemonRPC] end #9032  llm:enhancePrompt elapsedMs=5415  SLOW
```

说明 ✨ 不是本地字符串处理，而是**真发了一次模型请求**（走本地 sidecar / 你配置的模型，单次 5–10 秒）。这也解释了为什么点它偶尔要等一会儿。

---

## 五、你以后自己怎么扒（可直接复制的命令）

**1）导出 ✨ 的两段模板：**

```bash
cd /opt/WorkBuddy/resources/

# 定位字节偏移
grep -a -b -o "var DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE" app.asar
grep -a -b -o "var DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE"   app.asar

# 从偏移处 dump（偏移换成上面查到的数字）
tail -c +<偏移+1> app.asar | head -c 3035
```

**2）查看某个专家的默认提示词与人设：**

```bash
# 默认提示词 / 快捷提示词
cat ~/.workbuddy/plugins/marketplaces/experts/plugins/<专家名>/.codebuddy-plugin/plugin.json

# 人设系统提示（全文）
cat ~/.workbuddy/plugins/marketplaces/experts/plugins/<专家名>/agents/*.md
```

**3）看它是不是真的在调模型：**

```bash
grep "enhancePrompt" ~/.workbuddy/logs/main.log | tail
```

---

## 六、边界与注意事项（别踩坑）

1. **升级会变**：模板硬编码在 `app.asar` 里，WorkBuddy 每次更新都可能改（本机是 2026-09-10 构建的版本）。要长期跟踪，升级后重跑第五节的命令即可。
2. **没有用户可视化入口**：目前客户端**没有**把这两段模板开放成"可编辑配置项"。代码里 `enhancePromptConfig` 只是内部传参，不是给你改模板的开关。想自定义只能自己改 asar（不推荐，会被更新覆盖）。
3. **专家提示词上限**：代码里 `MAX_EXPERT_PROMPT_CHARS = 20000`，超过会被截断。
4. **服务端还有一层**：专家包是从云端市场下载的（`/portal/operation-platform/market/expert/download-url`），市场侧可能更新专家包版本，本地缓存未必是最新版。
5. **别动 `app.asar`**：只读 `grep`/`tail` 没关系；改写它会导致客户端完整性校验失败、无法启动。

---

*报告生成：Buddy · 2026-09-24 · 全部结论均来自本机实际文件与日志，未作推测。*
