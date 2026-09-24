# 提取元数据 — Linux 5.5.6

本文件记录 `enhance_*.linux-5.5.6.md` 两个原始模板的提取来源与可复现方法。

## 提取目标

| 输出文件 | JS 变量名 | 字节偏移 | 读取长度 |
|---|---|---|---|
| `enhance_system_prompt.linux-5.5.6.md` | `DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE` | `124721281` | 3035 B |
| `enhance_user_prompt.linux-5.5.6.md`   | `DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE`   | `124724316` | 2706 B |

## 来源环境

| 项 | 值 |
|---|---|
| 客户端 | WorkBuddy（腾讯 CodeBuddy 桌面版）Linux 版 |
| 版本 | `5.5.6-wb.38337834.g5f969292.hbc6253c2f32f` |
| 安装路径 | `/opt/WorkBuddy/` |
| 目标文件 | `/opt/WorkBuddy/resources/app.asar` |
| 文件大小 | 298,003,615 B（约 298 MB） |
| 构建时间 | 2026-09-10 17:52 |
| 提取时间 | 2026-09-24 |
| 来源模块 | `packages/workbuddy-server/src/enhance-prompt/handlers.ts` |

## 提取方式

**未解包 asar**，直接用「字节偏移 + 精确截取」方式，秒级完成：

```bash
cd /opt/WorkBuddy/resources/

# 1) 定位两个模板的字节偏移
grep -a -b -o "var DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE" app.asar   # → 124721281
grep -a -b -o "var DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE"   app.asar   # → 124724316

# 2) 按偏移截取（SYSTEM 读 3035 B，USER 读 2706 B）
tail -c +124721282 app.asar | head -c 3035
tail -c +124724317 app.asar | head -c 2706
```

切出后再做两步清理（脚本见下）：

1. 剥掉开头 `var DEFAULT_ENHANCE_PROMPT_XXX_TEMPLATE = \``
2. 从末尾 `\n    \`;` 处截断，去掉收尾反引号与分号

**模板正文未做任何改写**，缩进原样保留（System 模板用 Tab，User 模板用 4 空格）。

## 可复现脚本

```python
import os
ASAR = '/opt/WorkBuddy/resources/app.asar'
JOBS = [
    ('enhance_system_prompt.linux-5.5.6.md', 124721281, 3035,
     'var DEFAULT_ENHANCE_PROMPT_SYSTEM_TEMPLATE = `'),
    ('enhance_user_prompt.linux-5.5.6.md',   124724316, 2706,
     'var DEFAULT_ENHANCE_PROMPT_USER_TEMPLATE = `'),
]
TAIL_MARKER = '\n    `;'

for fname, offset, length, prefix in JOBS:
    with open(ASAR, 'rb') as f:
        f.seek(offset)
        text = f.read(length).decode('utf-8')
    assert text.startswith(prefix), f'{fname}: prefix mismatch'
    text = text[len(prefix):]
    idx = text.rfind(TAIL_MARKER)
    assert idx != -1, f'{fname}: tail marker not found'
    with open(os.path.join('.', fname), 'w', encoding='utf-8') as f:
        f.write(text[:idx] + '\n')
```

## 注意事项

- **字节偏移会随客户端升级失效**。版本变化后需用上面的 `grep -b` 重新定位。
- **不要在 298 MB 的 asar 上跑复杂正则**（如 `(a|b|c).{0,200}`），会跑到数分钟甚至挂住；用 `grep -F` 或先 `-b` 定位再 `tail -c`。
- **不要改写 `app.asar`**，只读截取即可；改写会破坏完整性校验导致客户端无法启动。

## 与 macOS 版的关系

- **System 模板**：两平台逐字一致。
- **User 模板**：已分叉。macOS 版（`enhance_user_prompt.md`）仅保留语言一致性约束；Linux 5.5.6 版已升级为完整的 prompt 改写指令模板（含任务描述、语言规则、7 条输出要求、BAD/GOOD 输出对照）。
- 详细的调用链路、模型路由与上下文行为分析见 [`REPORT.md`](REPORT.md)。
