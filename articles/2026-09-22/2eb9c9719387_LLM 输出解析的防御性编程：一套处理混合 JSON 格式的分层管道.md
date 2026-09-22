---
title: LLM 输出解析的防御性编程：一套处理混合 JSON 格式的分层管道
feedId: 38463
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

在 OpenClaw 的 Agent 流程、MCP 工具调用和插件编排里，我们经常要求模型输出 JSON：插件间传参、自动化任务写回结构化结果、子任务下发指令。提示词里明明写了"只输出 JSON"，但真实输出五花八门——有带 ``` 围栏的，有前后夹一句解释的，还有尾逗号、全角引号、`True/None` 混进来的。演示环境一次通过，跑批就天天 `JSONDecodeError`。

## 问题

很多人的第一版解析就是 `json.loads()` 加一条贪心正则，脆弱且不可观测：失败时只剩一行报错，原始输出没落盘，无法重放；临时 `replace` 补丁越堆越多，最后没人敢动。这不是解析技巧问题，是缺一层防御性设计。

## 做法：分层降级管道

核心思路是**把"提取"和"修复"分开**，从便宜到昂贵逐层尝试，每层记录命中情况：

1. **预处理**：去 BOM、去首尾空白，先直接 `json.loads`，规范输出在这层就结束。
2. **围栏提取**：用非贪婪正则抓围栏内容（兼容有无 `json` 标记），再解析。
3. **括号配平截取**：定位第一个 `{` 或 `[`，做字符串感知的深度扫描，取第一个配平块。必须处理字符串内的引号和转义，否则内容里出现 `"}"` 就截错。
4. **受控修复**：只修高频低风险项——尾逗号、全角引号、Python 字面量；复杂损坏直接上 `json_repair` 这类库，别自己无限堆正则。
5. **Schema 校验**：解析成功后用 pydantic/jsonschema 校验字段与类型；失败时把具体错误连同原始输出回传模型，做一次（且仅一次）修复重试。

```python
import json, re

def repair(s: str) -> str:
    s = re.sub(r",\s*([}\]])", r"\1", s)   # 尾逗号
    s = s.replace("\u201c", '"').replace("\u201d", '"')
    return s

def parse_llm_json(raw: str):
    text = (raw or "").strip().lstrip("\ufeff")
    try:                                    # 1. 直接解析
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    m = re.search(r"```(?:json)?\s*([\s\S]*?)```", text)
    if m:                                   # 2. 围栏提取
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            text = m.group(1).strip()
    start = next((i for i in (text.find("{"), text.find("["))
                  if i != -1), -1)
    if start == -1:
        raise ValueError("no JSON found")
    depth, in_str, esc = 0, False, False    # 3. 括号配平
    for i in range(start, len(text)):
        c = text[i]
        if in_str:
            if esc: esc = False
            elif c == "\\": esc = True
            elif c == '"': in_str = False
        elif c == '"':
            in_str = True
        elif c in "{[":
            depth += 1
        elif c in "}]":
            depth -= 1
            if depth == 0:
                return json.loads(repair(text[start:i + 1]))
    raise ValueError("unbalanced JSON block")
```

返回值之外，把 raw 原文、命中分支、耗时一起写日志。

## 踩坑点

- **贪心正则 `\{.*\}`**：会把两个独立 JSON 块连同中间解释文字黏成一个。用配平扫描替代。
- **字符串里的花括号**：`"template": "{{var}}"` 这类值，不考虑字符串状态会提前截断。
- **修复过度**：全局把单引号换双引号，可能破坏正文里的撇号。修复要白名单化。
- **数组还是对象**：同样指令下模型有时返回顶层数组，schema 别写死。
- **重试风暴**：无上限重试且失败信息不带原文，模型只能瞎改。重试必须附带原始输出和具体错误，上限一次通常够。
- **流式截断**：半截 JSON 是另一类问题，在接收层先判完整性，别混进解析管道。

## 可复用建议

- 解析器做成独立纯函数模块，线上每个失败样本回收进单测，测试集就是你的真实分布。
- 原始输出永远落盘，能重放才能区分模型漂移和解析 bug。
- 给解析分支埋点统计命中率，某层命中突然上升，多半是模型版本或提示词被改了。
- 能用 API 的 JSON mode / structured output 就用，防御管道是兜底不是首选。
- Prompt 侧同样防守：few-shot 示例、显式声明"不要解释文字"、字段尽量扁平。

## 总结

防御性解析的本质不是让代码更会猜，而是**分层降级、受控修复、全程可观测、重试有边界**。把这四件事沉淀成一个稳定模块，下游插件和自动化任务就不用再担心上游模型的"格式心情"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/8a748cd8be06d182.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/027187faedd40c2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/2d2582f1b37d7af8.png)

