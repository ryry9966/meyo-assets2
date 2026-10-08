---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的容错管线
feedId: 40898
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，让模型输出 JSON 是家常便饭：工具调用参数、结构化抽取、agent 间消息传递。我们通常在 prompt 里写一句"只输出以下 JSON 格式"，然后 `json.loads` 一把梭。跑通 demo 没问题，但真上量后你会发现，模型对"格式承诺"的遵守是概率性的，解析层必须按"输入不可信"来设计。

## 问题

同一个 prompt，实测输出至少有这么几种形态：

- 直接裸 JSON；
- ```json 围栏包裹；
- `<json>{...}</json>` 标签包裹；
- 围栏套标签、标签前后带解释文字；
- 键用了全角引号、带尾逗号、数组里混注释。

任何一个分支没覆盖，整条工具调用就挂。更麻烦的是失败往往是低频的：联调时发现不了，生产上半夜炸。

## 做法

我们的思路是把解析拆成五段：提取、归一化、解析、校验、重试，逐级兜底。

**1. 多级提取，按命中成本排序。**

```python
def extract_candidates(text: str) -> list[str]:
    c = []
    for tag in ("json", "answer", "output"):
        c += re.findall(rf"<{tag}>(.*?)</{tag}>", text, re.S)
    c += re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
    c += balanced_braces(text)   # 字符串感知的括号扫描
    c.append(text)               # 全文兜底
    return c
```

`balanced_braces` 不能用朴素计数：要跳过引号内的 `{}` 和转义符，否则字符串里出现花括号，深度就算错了。

**2. 归一化。** 去 BOM 和零宽字符、全角引号转半角、去尾逗号：

```python
def normalize(s: str) -> str:
    s = s.strip().lstrip("\ufeff").replace("\u200b", "")
    s = s.replace("\u201c", '"').replace("\u201d", '"')
    return re.sub(r",\s*([}\]])", r"\1", s)
```

**3. 双层解析。** 候选逐个先走严格 `json.loads`，全部失败再上宽容修复（json5 类库）。宽容解析只做兜底，不做首选——修复是有语义风险的。

**4. Schema 校验。** 解析成功不等于数据可用。用 pydantic 校验字段和类型，缺字段、类型漂移一律算失败。

**5. 有界重试。** 校验失败时把具体错误回传给模型再生成，最多两次。错误信息要具体（"字段 price 应为 number，实际是 string"），笼统的"格式错了"重试成功率很低。

## 踩坑点

- `r"\{.*\}"` 配 `re.S` 是最常见的坑：输出里有多个 JSON 块时，会从第一个 `{` 贪到最后一个 `}`，拼出一个语法恰好合法、语义完全错误的对象。
- 修复别过头。把 `"null"` 修成 `null`、自动补默认字段，可能让下游拿到"看似正常实则错误"的数据，比直接报错更难排查。
- 重试不设上限，等于给 token 消耗开了无底洞。
- prompt 里给的示例本身就是模板：示例里带注释和尾逗号，模型会原样学去。示例必须是"你想收到的样子"。

## 可复用建议

- 解析收敛到一个模块，返回 `(data, strategy, raw)` 三元组，strategy 记录哪级提取命中，方便排障。
- 原始输出落盘并埋点。统计各策略命中率，裸 JSON 占比突然上升，通常意味着上游 prompt 被人改过。
- 沉淀历史坏输出做回归集，解析器每次改动跑一遍契约测试。
- 运行时若支持结构化输出 / JSON Schema 约束，优先用原生能力，解析管线是兜底，不是替身。

## 总结

LLM 输出解析不是字符串问题，是管线问题。把提取、归一化、校验、重试分层，每层可观测、可回滚，失败才会从"半夜炸"变成"有指标、有日志、可迭代"的普通工程问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/bb98175da5a96d73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/24619eeb796708f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/71c72e496881d3b5.png)

