---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 38010
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

在 OpenClaw 插件和 MCP 工具链开发里，让 LLM 输出结构化 JSON 是最常见的需求之一。Agent 的工具调用参数、自动化流程的中间结果、插件间的数据交换，都依赖一个隐含假设：模型返回的字符串能被稳定解析。但只要跑过足够多的真实会话就会发现，这个假设经常碎掉。

## 问题

实践中常见的混合格式至少有这几类：

1. **标准 JSON**：直接可 parse，理想情况；
2. **Markdown 代码块包裹**：```json 围栏，或无语言标注的 ``` 围栏；
3. **前后带解释文字**：“好的，以下是结果：{...}”；
4. **XML 风格标签残留**：`<answer>{...}</answer>`，被 system prompt 引导过的模型尤其容易出现；
5. **准 JSON**：尾逗号、单引号、`//` 注释；
6. **嵌套代码块**：字段值里又包含 ``` 片段，导致按围栏切分时被截断。

解析器如果只处理一种格式，线上失败率可能在 5%–20% 之间波动，且失败往往集中在长输出、多轮对话或切换模型之后。

## 做法

核心思路：不要指望 prompt 一劳永逸，解析层必须自己兜底。推荐一条五层管线：

**第一步，粗清洗**：剥掉 BOM、零宽字符；按优先级尝试匹配 ```json 围栏、普通围栏、`<tag>...</tag>` 包裹。

**第二步，边界提取**：定位第一个 `{` 或 `[`，做括号配对扫描（必须跳过字符串字面量内部的括号和转义引号），截出候选片段。这比正则贪婪匹配可靠得多。

**第三步，严格解析**：`json.loads` 直接试。

**第四步，修复解析**：失败后走修复策略——去尾逗号、单引号转双引号、去注释。Python 可用 `json_repair`，Node 有 `jsonrepair`；自己写控制在几十行内，别追求完备。

**第五步，Schema 校验 + 兜底**：解析成功不等于可用，用 pydantic / zod 校验字段类型；校验失败时记录原始输出，可选发起一次“仅重排”的重试（把原始输出回填给模型，要求纯 JSON 重出）。

精简示例（第二步的括号扫描，最容易翻车的部分）：

```python
import json

def extract_json(text: str):
    text = text.strip().lstrip("\ufeff")
    start = min((i for i in (text.find("{"), text.find("[")) if i != -1), default=-1)
    if start == -1:
        raise ValueError("no json found")
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(text[start:], start):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        else:
            if ch == '"': in_str = True
            elif ch in "{[": depth += 1
            elif ch in "}]":
                depth -= 1
                if depth == 0:
                    return json.loads(text[start:i+1])
    raise ValueError("unbalanced")
```

## 踩坑点

- **嵌套代码块**：字段值里含 ``` 时，正则非贪婪匹配会在第一个 ``` 处截断。对策：括号配对做主路径，围栏匹配只当快速通道。
- **转义引号**：括号扫描不处理 `\"`，会提前判定字符串结束，截出残缺 JSON。
- **多个 JSON 对象**：模型可能一次返回多个对象或 JSONL，先想清楚策略是取第一个还是逐个解析。
- **修复越修越坏**：激进的自动修复可能把错误数据“修成”合法但语义错误的结构，所以 schema 校验必须放在修复之后。
- **只记录 parse 失败**：解析成功但字段缺失同样要记日志，否则统计出来的失败率是假的。

## 可复用建议

- 把解析管线做成独立模块，所有插件/工具共用，不要每处各写一套正则；
- 每次解析记录原始输出、尝试的路径、最终结果，用于统计各模型的真实格式分布；
- Prompt 层的约束（“只输出 JSON”）依然要有，它降低解析层压力，但不要依赖它；
- 解析和校验引用同一份 schema 定义，避免两边漂移。

## 总结

LLM 输出解析的本质，是把概率性的文本生成接进确定性的程序逻辑，防御性编程不是可选项。五层管线（清洗 → 边界提取 → 严格解析 → 修复 → 校验）配合完整日志，能把格式问题从“线上事故”降级为“日志里的统计项”。在 OpenClaw 的插件与 MCP 开发中，这套东西一次写好，处处受益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/2613e8856d89c753.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/e87249452c229695.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/d3020567f599016e.png)

