---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 43255
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

做 Agent、MCP 工具链或自动化脚本的同学基本都遇到过：明明在 prompt 里写了「只输出 JSON」，实际拿到的却是 ```` ```json ```` 围栏包着、前后各挂一段解释文字、偶尔还带尾随逗号或全角引号的混合体。本地调试时模型配合，一上批量任务或换模型版本，格式立刻五花八门。解析层没有防御性设计，失败率会随任务复杂度线性上升。

## 问题

最常见的脆弱写法是 `json.loads(response.strip())`，或者正则截取「第一个 `{` 到最后一个 `}`」。前者遇到任何围栏或前缀就挂；后者在模型顺带输出了含花括号的示例或说明时，截错区间。更麻烦的是**格式混合**：同一任务，不同轮次输出不一致——有时带 ```` ```json ````，有时裸 JSON，有时 JSON 还嵌在 markdown 列表里。

## 做法

核心思路是**分层降级**，每层失败才走下一层：

1. **归一化**：去 BOM、零宽字符；引号替换属于危险操作，放到修复层再做。
2. **提取候选块**：
   - 用非贪婪正则收集所有 ```` ```(json)? ... ``` ```` 围栏块；
   - 没有围栏时，用栈计数扫描所有**括号平衡**的区间，而不是首尾定位；
   - 两路候选合并，按「完整且最长」优先排序。
3. **解析降级链**：strict `json.loads` → 去围栏重试 → `json-repair` / `json5` 修复 → 全部失败则抛错并保留原文。
4. **结构校验**：解析成功 ≠ 数据可用。用 pydantic / zod 校验 schema，字段缺失或类型不对同样视为失败，进入重试或人工兜底。

参考骨架（Python）：

```python
import json, re
from json_repair import repair_json

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)

def candidates(text: str) -> list[str]:
    blocks = [m.group(1).strip() for m in FENCE.finditer(text)]
    if not blocks:
        blocks = [text.strip()]
    # TODO: 补一路"栈计数扫描平衡括号区间"的候选
    return sorted(set(blocks), key=len, reverse=True)

def parse_llm_json(text: str):
    for cand in candidates(text):
        try:
            return json.loads(cand)
        except json.JSONDecodeError:
            pass
        try:
            return json.loads(repair_json(cand))
        except Exception:
            continue
    raise ValueError(f"unparseable: {text[:200]}")
```

## 踩坑点

- 围栏正则写成贪婪匹配，会把多个代码块吞成一个；务必非贪婪 + `finditer` 收集。
- JSON 字符串字段里可能**合法包含 ```**（比如让模型生成代码），此时优先取最后一个完整围栏，而不是第一个。
- 修复要有边界：repair 库会补括号、改引号，也可能悄悄改语义。所有走修复路径的样本必须打日志、计数。
- 别用 `ast.literal_eval` 兜底单引号 JSON，遇到 `null` / `true` 直接炸，且引入执行面风险。
- 流式输出别在增量 chunk 上做完整解析，等结束信号再处理；流式阶段最多做括号深度监控。

## 可复用建议

- 解析逻辑收敛到一个工具模块，所有插件和工具调用共用，不要各写一份。
- 给每条降级路径加埋点：strict 命中率、repair 触发率。repair 占比超过阈值（比如 5%），说明该改 prompt，而不是继续加固解析器。
- Prompt 侧同步加固：明确「不要输出围栏外的任何内容」、给一个 few-shot 示例、结构化任务适当降 temperature。有 function calling / JSON mode 的模型优先用原生通道，手写解析器只作兜底。
- 解析失败时把原始输出完整落盘（注意脱敏），排查 prompt 问题比什么都快。

## 总结

把 LLM 输出当作不可信输入对待：**提取、修复、校验、留痕**，四层各司其职。防御性解析的目标不是让脏数据「凑合能用」，而是让失败可观测、可归因，最终反推上游 prompt 与模型选择变得更好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/de4aa814e9298b28.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/9b0ff0b8a1266d4e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/4a33941f65ad77e6.png)

