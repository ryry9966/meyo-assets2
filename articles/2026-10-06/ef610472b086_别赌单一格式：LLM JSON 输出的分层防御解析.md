---
title: 别赌单一格式：LLM JSON 输出的分层防御解析
feedId: 40677
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

在 OpenClaw 插件和 MCP 工具场景里，让模型返回结构化 JSON 是家常便饭：子任务分发、参数抽取、路由决策。Prompt 里写一句"只输出 JSON"，本地测几轮都正常，一接真实流量就开始频繁 `JSONDecodeError`。问题不在模型"不会"，而在输出格式的混合形态远比想象中多。

## 问题：实际会拿到什么

跑一段时间日志采样，坏样本基本是这几类：

- ```json 围栏包裹，或不带语言标记的 ``` 围栏；
- `<json>` / `<result>` / `<answer>` 之类的自造标签；
- 前后带一句"好的，结果如下："；
- 思考模型带 `<think>` 段，JSON 藏在后面；
- 尾逗号、中文引号、Python 风格的 `True`/`None`；
- `max_tokens` 截断导致 JSON 不完整；
- 最麻烦的：JSON 字符串值里还嵌着 markdown 围栏，正则会截错段。

## 做法：分层解析

思路是不要赌单一格式，把解析做成成本递增的多级过滤：

1. **源头降压**：优先用 provider 的 structured output 或 tool call schema；不行就在 prompt 里明确"不要代码块包裹、不要解释文字"，temperature 压低。
2. **预清洗**：剥掉 `<think>` 段和零宽字符。
3. **候选提取**：从便宜到贵——整体直接 loads → 围栏正则 → 自造标签正则 → 括号平衡扫描（感知字符串与转义的状态机）。
4. **受限修复**：仅在 loads 失败后做，去尾逗号、`True`/`None` 映射，每次修复打日志。
5. **schema 校验 + 修复回环**：pydantic 不过时，把错误信息和原始输出丢回模型修一轮，上限 1–2 次，仍失败就抛类型化错误。
6. **落日志**：原始输出带版本号存档，兜底命中率定期复盘。

核心骨架：

```python
import json, re

def candidates(text: str):
    yield text.strip()  # 最便宜：整体就是 JSON
    for pat in (r"```(?:json)?\s*(.*?)```",
                r"<(?:json|result|answer)>\s*(.*?)\s*</(?:json|result|answer)>"):
        for m in re.finditer(pat, text, re.S | re.I):
            yield m.group(1).strip()
    yield from balanced_spans(text)  # 兜底

def balanced_spans(text: str):
    spans, stack, in_str, esc = [], [], False, False
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch in "{[": stack.append((ch, i))
        elif ch in "}]" and stack:
            o, s = stack.pop()
            if not stack and (o, ch) in (("{", "}"), ("[", "]")):
                spans.append(text[s:i + 1])
    return spans

def loads_lenient(s: str):
    try: return json.loads(s)
    except json.JSONDecodeError: pass
    s = re.sub(r",\s*([}\]])", r"\1", s)          # 尾逗号
    s = s.replace("True", "true").replace("None", "null")
    return json.loads(s)  # 仍失败就抛，交给上层分类处理

def parse_llm_json(text: str):
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.S)
    for cand in candidates(text):
        try:
            data = loads_lenient(cand)
            if isinstance(data, (dict, list)):
                return data
        except Exception:
            continue
    raise ValueError("llm json parse failed")  # 按可重试错误处理
```

## 踩坑点

- 用正则抓 `{...}`，遇到字符串值里的花括号必截错段；平衡扫描必须维护 `in_str` 和转义状态。
- 全局去注释会误伤字符串里的 `https://`，全局替换 `True`/`None` 有伤及正文的风险——所以这些只作为失败后的兜底，且逐条记录。
- 字符串值内嵌 ``` 围栏时，非贪婪正则取到的块不一定完整，每个候选都要再过一次 loads 验证，不能提取到就直接用。
- 截断的 JSON 别试图补括号"救活"，语义多半已损，直接归为可重试错误，重发并提高 max_tokens。
- 最重要的一条：静默修复会掩盖问题。哪天兜底命中率上升，说明是 prompt 或模型变了，先看日志再改代码。

## 可复用建议

- 收敛成一个 `parse_llm_json` 工具函数，全项目所有模型输出统一走它，禁止散落各处的裸 loads。
- 失败分两类：格式错误走自动重试，schema 错误走修复回环，监控口径分开。
- 从日志攒真实坏样本做回归集，改解析器先跑一遍。
- 防御解析是兜底，能 structured output 就别手写正则。

## 总结

LLM 输出解析的本质是"不信任但理解"：承认格式会漂移，用多级过滤吸收噪声，用 schema 守住语义，用日志留住证据。这套骨架在 OpenClaw 插件和 MCP 工具里可以直接复用，代码不长，省下的排障时间很可观。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/a9875a98ee7a31a7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/662063bc939fb943.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/65517a7c40fa9cb7.png)

