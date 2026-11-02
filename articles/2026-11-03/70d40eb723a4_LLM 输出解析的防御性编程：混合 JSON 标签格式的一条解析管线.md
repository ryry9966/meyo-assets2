---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的一条解析管线
feedId: 43305
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

在 OpenClaw 插件和自动化流程里，让模型返回结构化 JSON 是最常见的集成方式：提示词写一句“请只输出 JSON，不要包裹代码块”，下游 `json.loads` 直接吃。Demo 阶段一切正常，跑上几天就开始出现偶发解析失败——而且每次失败的样本长得都不一样。

## 问题

实际线上输出不是一种格式，而是多种格式随机混合，大致三类：

1. **围栏包裹**：```` ```json ... ``` ````，提示词明明说了不要围栏，模型有时还是加；
2. **自造标签**：`<answer>{...}</answer>`，标签名不固定，可能是 `output`、`result`，甚至模型临场发挥；
3. **半残 JSON**：裸 JSON 前后带说明文字，或者尾逗号、`//` 注释、中文引号、被 max_tokens 截断的半个对象。

解析器只认一种格式，失败率通常在几个百分点，低到测试测不出来，高到足以打断定时任务。这类问题最难的地方在于难以稳定复现，排查成本远高于写解析代码本身。

## 做法：一条分层解析管线

核心思路是**不要赌格式，而是按命中成本从低到高收集候选、逐一尝试**：

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG   = re.compile(r"<[a-zA-Z_]+>\s*(\{.*?\})\s*</[a-zA-Z_]+>", re.S)

def balanced_json(text):
    """找第一个配对的 {...}，跳过字符串内部的括号和转义"""
    start = text.find("{")
    while start != -1:
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
                    return text[start:i + 1]
        start = text.find("{", start + 1)
    return None

def parse_llm_json(raw: str):
    candidates = FENCE.findall(raw) + TAG.findall(raw)
    m = balanced_json(raw)
    if m:
        candidates.append(m)
    for c in candidates:
        fixed = re.sub(r",\s*([}\]])", r"\1", c.strip())  # 去尾逗号
        fixed = fixed.replace("“", '"').replace("”", '"')
        try:
            return json.loads(fixed)
        except json.JSONDecodeError:
            continue
    return None
```

在这之上还有三层，缺一不可：

- **结构校验**：`json.loads` 成功不等于“是你要的结构”，用 pydantic 或 jsonschema 再验一遍字段；
- **带纠错的重试**：解析失败时，把**原始输出和报错信息**一起喂回去重试一次，只说“格式错了”模型大概率会重复同样的错误；
- **落盘原始输出**：无论成败，原始返回必须进日志。没有它，线上问题等于无法复盘。

## 踩坑点

- **用 `text.find("{")` 到 `rfind("}")` 截取**是常见错误：输出里如果有两个对象或说明文字里带花括号，就会混进脏数据。必须做配对计数，且处理字符串内部的引号转义。
- **标签正则写死标签名**会翻车。模型会自己发明 `<output_json>` 这类名字，用通配的 `[a-zA-Z_]+` 留弹性。
- **`json.loads` 默认放行 `NaN` / `Infinity`**，下游如果对接严格序列化器会再炸一次，用 `parse_constant` 参数拦掉。
- **手写的“修复”逻辑要克制**。去尾逗号、换引号这类可以自己写；更深的修复（截断补全、单引号键名）建议直接用 `json_repair` 这类现成库，自己造轮子的边界情况写不完。
- **重试不超过一次**。解析失败往往伴随内容本身的问题，反复重试只是烧 token。

## 可复用建议

- 把解析器做成独立模块，**用真实失败样本建回归测试集**，每次遇到新 case 就加进去，这套用例库的价值会随时间指数增长；
- 提示词端也做防御：给 schema 示例、明确禁止围栏——但把它当作降低概率的手段，**永远不要当作唯一防线**；
- 如果所用模型支持结构化输出 / JSON mode，优先用，防御解析降级为第二层；
- 给解析成功率加监控，突然下跌通常意味着模型版本变了，或者上游有人改了提示词。

## 总结

LLM 的输出格式漂移不是 bug，是常态。防御性解析的本质是承认这一点：候选提取 + 渐进修复 + 结构校验 + 一次纠错重试 + 原始输出落盘，五层各管一件事。写一次，之后所有插件和自动化流程都能复用，几个百分点的偶发失败就会从“深夜告警”变成“日志里的一行记录”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/3ca4928394cea051.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/e7e4b61dbf120832.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/7c34024063bb5bd9.png)

