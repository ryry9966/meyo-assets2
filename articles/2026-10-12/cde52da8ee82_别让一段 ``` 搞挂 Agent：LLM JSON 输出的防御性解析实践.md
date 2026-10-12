---
title: 别让一段 ``` 搞挂 Agent：LLM JSON 输出的防御性解析实践
feedId: 41294
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，"让模型返回 JSON" 是最高频的契约之一：skill 调工具要传参、Agent 之间要交接状态、自动化流水线要机器可读的中间结果。但这个契约几乎没有模型会稳定遵守——即便 prompt 里写了"只输出 JSON"，实际返回仍可能是围栏包裹、前后带解释、甚至混着自定义标签。解析层不做防御，故障只是时间问题。

## 问题

同一个 prompt 在不同模型、温度、上下文长度下，输出形态至少有这几类：

- 纯 JSON（理想情况，占比反而最低）
- ` ```json ... ``` ` 围栏，或无语言标注的 ` ``` `
- JSON 前后带一句"好的，以下是结果"
- 嵌在 `<answer>` 之类的标签里
- 带尾逗号、全角引号、`True/False/None`

直接 `json.loads()` 会在生产环境随机炸。最麻烦的是这是**低频失败**：测试十次九次过，上线后换模型或上下文变长才暴露，排查时往往连原始输出都没留。

## 做法：分层解析，逐级降级

核心思路是不指望一次解析成功，而是写一条流水线，从最便宜的手段开始逐级加强：

```python
import json, re

def extract_json(text: str):
    # 1) 快速路径
    try:
        return json.loads(text)
    except (json.JSONDecodeError, TypeError):
        pass

    # 2) 剥代码围栏：逐块尝试，防止贪婪匹配吞掉多个代码块
    for m in re.finditer(r"```[a-zA-Z]*\s*(.*?)```", text, re.S):
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            continue

    # 3) 字符串感知的括号配对提取
    candidate = _balanced_span(text)
    if candidate:
        try:
            return json.loads(candidate)
        except json.JSONDecodeError:
            pass
        # 4) 轻度修复后重试
        fixed = re.sub(r",\s*([}\]])", r"\1", candidate)   # 尾逗号
        fixed = fixed.replace("\u201c", '"').replace("\u201d", '"')
        fixed = re.sub(r"\bTrue\b", "true", fixed)
        fixed = re.sub(r"\bFalse\b", "false", fixed)
        fixed = re.sub(r"\bNone\b", "null", fixed)
        try:
            return json.loads(fixed)
        except json.JSONDecodeError:
            pass

    raise ValueError(f"json parse failed, raw={text[:500]!r}")

def _balanced_span(text: str):
    for open_ch, close_ch in (("{", "}"), ("[", "]")):
        start = text.find(open_ch)
        if start == -1:
            continue
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):
            c = text[i]
            if in_str:
                if esc: esc = False
                elif c == "\\": esc = True
                elif c == '"': in_str = False
            else:
                if c == '"': in_str = True
                elif c == open_ch: depth += 1
                elif c == close_ch:
                    depth -= 1
                    if depth == 0:
                        return text[start:i + 1]
    return None
```

解析拿到合法 JSON 后，再交给 pydantic / jsonschema 做结构校验——解析层只负责"拿到"，不负责"对不对"。全部失败时抛出带原始输出前 500 字符的错误，交由上层决定是否带着解析错误信息回炉重试。

## 踩坑点

1. **正则贪婪匹配**：`.*``` ` 会把"示例代码块 + 答案代码块"合成一段。模型很爱先给个 demo 再给正式答案，务必非贪婪 + 逐块尝试。
2. **括号计数必须跳过字符串内部**：`"desc": "a } b"` 这种内容会让朴素计数器提前截断，上面 `_balanced_span` 里的 `in_str/esc` 状态机就是为此。
3. **不要过度修复**：静默把单引号替成双引号，容易破坏内容里的撇号。修复手段要留日志、留开关，出问题能关。
4. **失败必须保留原文**：把 raw output 吞掉，等于放弃了事后复盘的唯一证据。
5. **流式增量解析是另一个课题**：partial JSON 解析和这里的一次性解析逻辑完全不同，别混在一个函数里。

## 可复用建议

- 收敛成一个共享 util 模块或插件公共库，不要每个 skill 各抄一份，改一处漏一处。
- 沉淀"坏输出语料库"：把线上真实的畸形输出存下来做回归测试，比拍脑袋造的测试用例有价值得多。
- prompt 侧照常收紧（明确禁止围栏和解释），解析侧照常兜底，双保险但互不依赖。
- 按层级统计解析命中率：哪一层生效的比例突然变化，往往意味着上游模型升级导致的格式漂移，比报错更早暴露问题。

## 总结

防御性解析的本质，是承认"模型输出格式是概率性的"这个前提。工程上的正确姿势是四件事：prompt 收紧、分层解析、失败可观测、可重试。做完这四件事，JSON 解析失败就从偶发故障变成了可控、可度量的事件——这才是 Agent 系统能长期跑在自动化里的底气。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/0668cce0a9b1dc46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/ccbc588b60e8720c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/736467a615bdb56b.png)

