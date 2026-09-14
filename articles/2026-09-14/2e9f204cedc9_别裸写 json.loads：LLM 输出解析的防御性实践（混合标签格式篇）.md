---
title: 别裸写 json.loads：LLM 输出解析的防御性实践（混合标签格式篇）
feedId: 37526
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

在 OpenClaw 的插件和 Agent 工作流里，"让 LLM 输出 JSON" 是最常见的模式：工具调用参数、结构化抽取结果、MCP 资源的中间表示，几乎都依赖它。但只要跑得够久，你一定会遇到这种情况：提示词里明明写了"只输出 JSON"，模型返回的却是 ```json 围栏包裹的内容，或者前面带一句"好的，以下是结果："，或者干脆是裸 JSON 和围栏格式交替出现。

我在自己的自动化流水线里统计过一段时间：同一套提示词，围栏格式约占七成，裸 JSON 约两成，剩下的是各种"JSON 混着说明文字"。这不是偶发抖动，是需要当成常态来设计的输入条件。

## 问题

裸写 `json.loads(response)` 的问题不只是报错，而是**失败方式不可预期**：

- 围栏存在时直接抛 `JSONDecodeError`，任务中断；
- 围栏里嵌套了代码块（比如 JSON 字段里包含示例代码），简单粗暴的正则会截断错位；
- 输出里混着全角引号、尾逗号、注释，解析结果时好时坏，难以复现。

本质上，这是把"格式契约"建立在概率行为上，缺少一层防御。

## 做法：分层解析，逐级降级

核心思路是把解析拆成几个独立策略，按可靠性从高到低依次尝试，并**记录命中的策略**：

```python
import json, re

def extract_json(text: str):
    candidates = []
    # 1. 直接解析
    candidates.append(("direct", text.strip()))
    # 2. 剥离围栏
    fenced = re.findall(r"```(?:json|JSON)?\s*(.*?)```", text, re.DOTALL)
    candidates += [(f"fenced#{i}", b) for i, b in enumerate(fenced)]
    # 3. 花括号配平提取（处理 JSON 混在说明文字里的情况）
    m = re.search(r"\{", text)
    if m:
        candidates.append(("balanced", _balanced_slice(text, m.start())))

    for strategy, raw in candidates:
        try:
            return True, json.loads(raw), strategy
        except json.JSONDecodeError:
            continue
    return False, text, "failed"
```

几个关键点：

1. **围栏候选收集全部，不取第一个就完事**。模型有时先输出一个示例，再输出真正的结果。
2. **花括号配平**而非正则贪婪匹配，避免字符串里出现 `}` 导致截断。
3. 解析成功后**过一遍 pydantic 模型校验**，JSON 合法但字段缺失、类型不符的情况同样常见。
4. `strategy` 字段写进日志。上线两周后你能据此判断：降级主要发生在哪个模型、哪类任务上，再针对性改提示词。

## 踩坑点

- **嵌套围栏**：JSON 字段值里包含 ` ``` ` 时，非贪婪正则会提前截断。遇到这种任务，最好在提示词里禁止模型在字段值中放代码块，或者解析失败后把原文落盘人工排查。
- **全角引号与尾逗号**：中文任务里高发。可以加一个"修复层"（替换 `""`→`""`、去掉 `,\s*[}\]]` 前的逗号），但要打上标记，修复成功的输出不要和正常输出混在同一统计口径里。
- **json.loads 默认接受 NaN/Infinity**，严格来说不是合法 JSON。如果结果要跨服务传输，用 `parse_constant` 参数拦一下。
- **静默吞错**：所有降级路径最终失败时，必须把原始输出完整记录。没有原文，你永远不知道模型当时到底输出了什么。

## 可复用建议

- 解析逻辑收敛到**一个工具模块**，全项目共用，不要每个插件各写一套正则。
- 维护一个**失败样本集**（围栏嵌套、全角符号、双块输出……），作为解析函数的单元测试用例。每次线上遇到新花样，补一条用例。
- 提示词侧仍然要约束：明确要求"只输出 JSON，不要围栏，不要解释"，并用 `response_format`（模型支持时）。防御性解析是兜底，不是替代。
- 解析失败时优先**重试一次并在提示词中附上失败原因**，比直接换修复策略更便宜。

## 总结

LLM 的输出格式本质上是概率性的，工程上的正确姿势是：**契约写在提示词里，容错写在代码里，观测写在日志里**。一个百行左右的分层解析模块，配合失败样本回归测试，能把这类问题从"随机炸任务"变成"日志里的一行降级记录"。这套做法不炫技，但让流水线在模型升级、提示词调整时依然稳得住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/de61703475cd069c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/1a58c602fac3d4aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/22b9114507f07928.png)

