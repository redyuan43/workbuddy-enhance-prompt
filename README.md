# WorkBuddy Enhance Prompt Templates

从 WorkBuddy（腾讯 CodeBuddy 桌面版）客户端中提取的「输入增强」功能提示词模板，原样导出供研究、参考与复用。

- 来源文件：`WorkBuddy.app/Contents/Resources/app.asar` → `main/initialize.js`
- 来源模块：`packages/workbuddy-server/src/enhance-prompt/handlers.ts`
- 提取方式：直接从反编译产物中按模板变量名（`DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE` / `DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE`）原样切出，未做任何改写。

## 文件说明

| 文件 | 变量名 | 作用 |
|---|---|---|
| [enhance_system_prompt.md](enhance_system_prompt.md) | `DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE` | System Prompt。定义"Prompt Engineering Expert"角色：分析用户原始输入的目标、歧义与缺口，按 prompt 工程原则改写为更清晰、更完整、带约束与输出格式的增强版本。含硬约束（语言跟随输入、上限约 800 字符、只输出结果不加评论）与 2 个 few-shot 示例。 |
| [enhance_user_prompt.md](enhance_user_prompt.md) | `DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE` | User Prompt 包装器。内含 `{input}` 占位符，运行时被替换为用户选中的输入文本；强调"语言一致性"为最高优先级（中文进中文出），附中/英/混合语言 few-shot。 |

两个模板均为**硬编码的出厂默认值**：对所有用户、所有会话、所有任务类型生效，不注入任何会话历史、项目上下文或用户画像。模板默认下游是"代码助手"（开头即写明 `specializes in writing code`），因此用于非代码场景（如文学创作）时不会获得任务语境层面的增强，仅做通用扩写。

## 调用链路

```
┌──────────────────────────┐
│ Renderer (React 前端)      │
│                          │
│ 用户在输入框点击「输入增强」  │
│ useEnhancePrompt({       │
│   model: 当前会话选中模型   │ ← enhanceSelectedModelId =
│   sessionId,             │    remoteCurrentModelId || selectedModelId
│   ...                    │
│ })                       │
└──────────┬───────────────┘
           │  RPC: channel "llm:enhancePrompt"
           ▼
┌──────────────────────────────────────────────┐
│ Main Process (Electron 主进程)                 │
│ handleEnhancePrompt(deps, req)                │
│                                              │
│ 1. 校验 req.text 非空（空 → empty_input）      │
│ 2. model = req.model（由前端传入）             │
│ 3. endpoint = deps.getSidecarEndpoint()       │
│    （CLI sidecar 地址，不可用 →                │
│     sidecar_unavailable）                     │
│ 4. createClient(endpoint).runAgent({          │
│      systemPrompt: SYSTEM_TEMPLATE  ← 本repo   │
│      userPrompt: renderUserPrompt(text)       │
│        = USER_TEMPLATE.replace("{input}",    │
│           text)                     ← 本repo   │
│      agentName: "enhance-prompt",             │
│      model                                    │
│    })                                         │
└──────────┬───────────────────────────────────┘
           │  CliPulseClient：单次 POST
           │  {endpoint}/api/v1/llm/completions
           ▼
┌──────────────────────────────────────────────┐
│ CLI Sidecar (codebuddy --serve 进程)          │
│ 按指定 model 完成 LLM 推理                     │
│ （模型 = 当前会话选中的模型，可云端可本地）      │
└──────────┬───────────────────────────────────┘
           │  result.text
           ▼
┌──────────────────────────────────────────────┐
│ 后处理：stripWrappingQuotes()                  │
│ 去掉首尾包裹引号 → 回填到输入框                  │
└──────────────────────────────────────────────┘
```

要点：

1. **触发**：输入框的「输入增强」按钮 → renderer 通过 `llm:enhancePrompt` 通道向主进程发 RPC。
2. **模型选择**：`enhanceSelectedModelId = remoteCurrentModelId || selectedModelId`，即**跟随当前会话选中的模型**——选了本地 V100/P40 就走本地，选了云端模型就走云端。换模型不换模板。
3. **执行**：`CliPulseClient` 向 CLI sidecar 发单个 `POST /api/v1/llm/completions`，参数完全可控（systemPrompt / userPrompt / model），无需多步握手。
4. **后处理**：`stripWrappingQuotes()` 去掉模型输出首尾的包裹引号（含中文弯引号），空结果视为失败；干净文本直接回填输入框。

## 用法（复用这两个模板）

```python
# Python 示例：本地复刻输入增强
import requests

system = open("enhance_system_prompt.md", encoding="utf-8").read()
user_tpl = open("enhance_user_prompt.md", encoding="utf-8").read()

user = user_tpl.replace("{input}", "帮我写一个登录页面")

resp = requests.post(
    "http://your-llm-endpoint/v1/chat/completions",
    json={
        "messages": [
            {"role": "system", "content": system},
            {"role": "user", "content": user},
        ],
        "temperature": 0.7,   # 增强任务建议低温度
    },
)
enhanced = resp.json()["choices"][0]["message"]["content"].strip()
# 原版还有一步：去掉首尾包裹引号
enhanced = enhanced.strip("\"'“”‘’")
print(enhanced)
```

## 注意

- 模板中的 `Do NOT suggest specific technologies unless mentioned`、800 字符上限等约束是面向通用/代码场景调的，复用到其他领域时可按需调整。
- 语言一致性在两个模板里重复强调（System 约束 1 + User 最高优先级），是官方为防串语言做的双保险，复用时建议至少保留一处。
