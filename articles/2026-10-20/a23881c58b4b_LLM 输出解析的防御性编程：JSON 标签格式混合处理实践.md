---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实践
feedId: 42061
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

在 OpenClaw 插件和 Agent 流程里，工具调用参数、结构化记忆写入、MCP 传参几乎都要求 LLM 输出 JSON。提示词里写"只输出 JSON，不要任何解释"，听起来很明确，实践里模型并不会严格执行。这不是模型的 bug，而是自然语言承诺的固有脆弱性——格式约定无法等价于程序约束。防御性解析不是可选项，是管线里必须的一层。

## 问题：同一份提示词，多种"标准格式"

实际日志里常见的几种：

1. **Markdown 代码围栏**：带 `json` 标注的，或不带标注的裸围栏；
2. **伪 XML 标签**：`<json>...</json>`、`<output>...</output>`；
3. **裸 JSON 前后带说明**："结果如下：{…}，如需调整请告知"；
4. **语法漂移**：尾逗号、单引号键、智能引号、全角冒号。

前三种是包装层差异，第四种是内容层差异。很多实现只处理了其中一种，换模型或模型升级后失败率突增，才回头补课。

## 做法：五层解析管线

原则：便宜、确定性的手段放前面，昂贵、有风险的修复放最后。

1. **预处理**：非贪婪正则剥掉围栏和伪标签；多个匹配取最后一个，模型习惯先写说明再写结果。
2. **括号配对定位**：找第一个 `{` 或 `[`，用字符串感知的扫描器（跳过引号内内容和转义字符）找到配对括号，截出候选串。这一层能兜住标签、围栏、说明文字的所有组合。
3. **标准解析**：`json.loads` 直接过，多数情况到此结束。
4. **语法修复兜底**：仅第三层失败才触发——去尾逗号、替换智能引号。修复是有损操作，必须打日志标记"经修复解析"。
5. **校验与重试**：用 JSON Schema 或 pydantic 验证字段与类型，解析成功不等于数据正确。校验失败时，把原始输出和具体报错拼回提示词，temperature 降为 0 重试一次；仍失败就 fail closed 抛出明确异常，绝不用默认值蒙混。

核心骨架：

```python
FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG = re.compile(r"<(?:json|output|result)>(.*?)</(?:json|output|result)>", re.S | re.I)

def extract_json(text: str) -> dict:
    for pat in (TAG, FENCE):
        hits = pat.findall(text)
        if hits:
            text = hits[-1]                     # 取最后一个匹配
            break
    start = min((i for i in (text.find("{"), text.find("[")) if i >= 0), default=-1)
    if start < 0:
        raise JsonParseError("no candidate found")
    body = match_balanced(text, start)          # 字符串感知的括号配对
    try:
        return json.loads(body)
    except json.JSONDecodeError:
        return json.loads(repair(body))         # 有损，必须记日志
```

## 踩坑点

- **贪婪正则**：`\{.*\}` 配 `re.S` 会吞掉多个对象或截断嵌套结构，是最常见的事故源。
- **修复器改变语义**：单引号转双引号会误伤字符串里的撇号（it's），修复后要比对关键字段。
- **围栏套围栏**：让模型输出代码片段时，字符串值里可能出现连续三个反引号，朴素剥围栏会截在半路，括号配对层能救回来。
- **流式截断**：流式输出中途断连会留下半个 JSON，扫描发现不闭合就直接走重试，不要试图"补全"。
- **模型升级改变格式分布**：上周日志里没出现的格式不代表不会出现，解析层要按"永远会有新花样"来设计。

## 可复用建议

- 把解析器做成独立模块，收集每次失败的真实输出，沉淀成单测语料库，回归测试比任何文档都可靠。
- 解析失败时永远落盘原始输出，排查时最有价值的就是这段。
- 平台的结构化输出（JSON mode、tool schema）能从源头消掉大半问题，优先用，但保底解析器留着——总有要走纯文本接口的一天。
- 修复逻辑宁可保守：无法确定语义时选择报错，把重试决策交给上层。

## 总结

LLM 的输出格式承诺，本质是概率而非契约。防御性解析做的事很朴素：用分层管线把格式不确定性收敛为确定数据，用校验把"解析成功"升级为"数据可用"，用受控重试和 fail closed 收尾。提示词负责提高成功率，代码负责守住下限，两者各司其职，管线才稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/33d00286887b66c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/df2c4f71a75bc8d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/186927f449cfeeb2.png)

