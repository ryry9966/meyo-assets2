---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 38495
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

在 OpenClaw 插件和 MCP 工具链的自动化流程里，让 LLM 返回结构化 JSON 是最常见的需求：Agent 之间传参、工具调用结果落库、插件配置生成，几乎都依赖这一步。Prompt 里写「只输出 JSON」很容易，但真实流量里模型的输出经常不那么听话——代码围栏包裹、前后带解释文字、一次吐出多个 JSON 片段，甚至是带单引号和尾逗号的「宽松 JSON」。直接 `json.loads` 的失败率积累起来，就是一个不可忽视的自动化断点。

## 问题

线上观察下来，脏输出大致分三类：

1. **包裹型**：```json ...``` 围栏前后还有「以下是结果：」之类的说明文字；
2. **多段型**：一次输出里先给一段分析用的 JSON，再给一段正式结果，出现两个以上 `{...}`；
3. **宽松型**：单引号键值、尾逗号、`//` 注释、字符串内未转义换行。

用一行正则 `\{.*\}` 贪婪匹配会跨对象拼接，非贪婪又会在嵌套处截断。这类解析代码散落在各个插件里，各修各的，坑也是各踩各的。

## 做法：分层解析，逐级降级

思路是不要指望一层解析解决所有情况，而是做成瀑布式的降级链：

```python
import json, re

def extract_json(text: str):
    # 第 1 层：直接解析
    try:
        return json.loads(text), "direct"
    except json.JSONDecodeError:
        pass
    # 第 2 层：剥离代码围栏
    for block in re.findall(r"```(?:json)?\s*(.*?)```", text, re.S):
        try:
            return json.loads(block), "fenced"
        except json.JSONDecodeError:
            continue
    # 第 3 层：括号配平提取（感知字符串状态）
    start = text.find("{")
    if start != -1:
        depth, in_str, esc = 0, False, False
        for i, ch in enumerate(text[start:], start):
            if in_str:
                if esc: esc = False
                elif ch == "\\": esc = True
                elif ch == '"': in_str = False
            elif ch == '"': in_str = True
            elif ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    return json.loads(text[start:i+1]), "balanced"
    raise ValueError("no parsable json")
```

关键点：

- **括号配平必须带字符串状态机**，否则字符串里的 `{`、`"` 会把配平搞乱；
- 第 3 层拿到片段后如果仍解析失败（宽松 JSON），做**最小修复**：单引号换双引号、去尾逗号、去行注释，或者直接上 `json5` 解析；
- 修复后的结果**必须过一遍 schema 校验**（pydantic 或 JSON Schema），确认字段和类型都对才算成功；
- 全链失败时，把解析错误信息和原始输出片段拼回重试 Prompt，让模型自己纠正，通常一次就能过。

## 踩坑点

- **剥围栏别只看行首**。模型可能在字符串值里输出 ``` 字符，逐行匹配行首标记会误伤内容。
- **修复要克制**。只处理你观察到的高频模式，别写「通用 JSON 修复器」，很容易把语义改错然后静默通过校验，这比解析失败更危险。
- **「只输出 JSON」不是保证**。temperature 偏高或多轮对话后遵守率下降，防御层不能省。
- **多段输出要选对块**。配平提取默认取第一个 `{`，但有些模型习惯先给示例再给结果，建议按 schema 校验通过的那个块为准，而不是盲取第一个。
- **原始输出必须落日志**。没有 raw output，线上排障全是盲区。

## 可复用建议

- 把解析器收进一个独立模块，返回 `(obj, method)`，`method` 记录命中了哪一层，方便统计各层命中率和整体失败率，作为改进 Prompt 的依据；
- 能用模型原生的结构化输出 / 约束解码能力就优先用，防御性解析只做兜底，两者不冲突；
- schema 里加 `version` 字段，输出格式演进时不至于让老数据无法追溯；
- 重试次数设上限（2 次足够），超过就抛出带上下文的异常，交给上层决定是否人工介入。

## 总结

对 LLM 输出的解析，本质上是在和一个概率性的上游做接口。与其反复调 Prompt 祈祷输出干净，不如承认它会脏，用「直接解析 → 剥围栏 → 配平提取 → 最小修复 → schema 校验 → 带错重试」这条降级链把脏输入消化掉，再配上日志和命中率统计，整条自动化链路的稳定性会有非常直接的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/837a78fdd7648ecc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/0c7ebe18595b22d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/56db6968ee75a81b.png)

