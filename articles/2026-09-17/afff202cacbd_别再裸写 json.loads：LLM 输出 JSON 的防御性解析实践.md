---
title: 别再裸写 json.loads：LLM 输出 JSON 的防御性解析实践
feedId: 37968
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

在 OpenClaw 的插件和 Agent 工作流里，让模型输出 JSON 是最常见的集成方式：MCP 工具参数、子任务路由、结构化抽取，几乎都依赖它。但只要跑过自动化流水线的人都知道，"请只输出 JSON"这句话的约束力约等于零。同一个 prompt，今天给你干净的裸 JSON，明天包一层 ```` ```json ```` 代码块，后天前面加一句"好的，以下是结果"，甚至混进 `<think>` 残留和 markdown 列表。解析代码如果是裸的 `json.loads`，流水线的稳定性就完全看模型当天的心情。

## 问题

典型的失败场景有三类：

1. **包装污染**：代码块围栏、前后闲聊文本、XML 风格标签（`<json>`、`<result>`）混着来，模型经常自创标签。
2. **语法瑕疵**：尾逗号、单引号、中文引号、未转义换行、`NaN` 字面量——语法上不合法但语义上人能读懂。
3. **结构漂移**：一个回复里出现多个 JSON 对象、数组套对象、或者键名大小写不稳定。

这类故障的特点是低频但致命：测试时看起来没问题，上线第三天凌晨两点流水线静默中断。

## 做法

核心思路是把解析做成**分层降级管线**，而不是单点尝试。我目前用的流程是四层：

```python
import json, re

def parse_llm_json(text: str):
    # 第 1 层：直接解析，赌模型听话
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # 第 2 层：剥掉代码围栏后重试
    m = re.search(r"```(?:json)?\s*([\s\S]*?)```", text)
    if m:
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            pass

    # 第 3 层：定位首个平衡的 {} 或 []（括号计数，不用贪心正则）
    for start_ch, end_ch in (("{", "}"), ("[", "]")):
        start = text.find(start_ch)
        if start == -1:
            continue
        depth = 0
        for i in range(start, len(text)):
            if text[i] == start_ch:
                depth += 1
            elif text[i] == end_ch:
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(_repair(text[start:i+1]))
                    except json.JSONDecodeError:
                        break
    # 第 4 层：返回 None，上层触发带错误上下文的重试 prompt
    return None

def _repair(s: str) -> str:
    s = re.sub(r",\s*([}\]])", r"\1", s)          # 尾逗号
    s = s.replace(""", '"').replace(""", '"')     # 中文引号
    return s
```

配套三条纪律：

- **每层失败都记原始文本**。没有 raw output 日志，排障就是玄学。
- **解析成功后走 schema 校验**（`jsonschema` 或 pydantic），格式合法但字段错的情况比想象的多。
- **第 4 层重试时把解析错误喂回模型**："你的输出无法解析，错误是 X，请重新输出"，比干巴巴重发有效得多。

## 踩坑点

- **贪心正则跨嵌套**：`\{.*\}` 遇到多个 JSON 对象会吞掉中间一切，必须用括号计数。
- **修复工具别滥用**：`json_repair` 类库能救急，但会静默改语义（比如把畸形值改成 null），修完必须过 schema。
- **流式输出是另一个问题**：部分 JSON 不是"脏 JSON"，需要增量解析器或攒够再解，别混在同一个函数里。
- **别指望 prompt 一劳永逸**：即使模型支持 structured output / tool call 强约束，也保留降级路径——换模型、换供应商时你会感谢自己。

## 可复用建议

- 把这套逻辑封装成一个纯函数 `parse_llm_json(text) -> dict | None`，全项目只此一处，别让每个插件自己写正则。
- 建一个"脏输出样本库"，把线上真实失败 case 存下来当回归测试，跑一次全样本通过率就知道解析器健不健壮。
- 解析失败率值得监控：突然从 0.5% 涨到 5%，通常说明上游换了模型版本。

## 总结

LLM 输出解析的本质是处理一个不受你控制的序列化器。防御性编程在这里不是过度设计，而是把"模型输出不可靠"当作公理来设计系统：分层降级、失败可观测、修后强校验。代码不复杂，但把这三件事做进架构里，自动化流水线的可用性会有质的差别。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/fd8fc73fa418ee49.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/cd5c778d5cb8a091.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/c53a83fdc9ae49ed.png)

