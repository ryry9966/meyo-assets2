---
title: LLM 输出解析的防御性编程：一套混合 JSON 标签格式的分层处理方案
feedId: 37917
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

在 OpenClaw 的 agent 流水线里，LLM 输出经常不只是"给人看的"：它可能是下一个节点的入参、MCP 工具的调用参数、或插件回调里的结构化载荷。我们通常在 prompt 里要求模型"只输出 JSON"，但实际跑起来会发现，同一份 prompt 在不同模型、不同温度、甚至不同会话里，输出格式都会漂移：裸 JSON、```json 围栏、自定义标签包裹、前后带说明文字……都出现过。

## 问题

典型写法是正则抓第一个 ```json 块：

```python
m = re.search(r"```json\n(.*?)```", text, re.DOTALL)
data = json.loads(m.group(1))
```

演示时没问题，上量后这是三类故障的主要来源：

1. 模型没加围栏，直接输出裸 JSON → 正则匹配失败；
2. JSON 字段值本身含 ```（比如让模型生成代码片段）→ 非贪婪匹配提前截断，切出半截对象；
3. 模型先解释再输出两段 JSON → 抓到的第一个块不是我们要的。

更麻烦的是这些故障按比例发生而非必现，日志里往往只剩一条 parse error，原始输出没存，复盘无从下手。

## 做法：分层解析管线

核心思路是把"解析"当防御纵深：每层成本低、可组合，任一层成功即返回，全失败才触发重试。

**第 0 层：直接解析。** `json.loads(raw)` 能过就过，这是大多数请求的快路径。

**第 1 层：剥离围栏与标签。** 用宽匹配同时覆盖变体，收集所有候选片段而不是只取第一个：

```python
FENCE_RE = re.compile(r"```[a-zA-Z]*\s*\n(.*?)```", re.DOTALL)
TAG_RE = re.compile(
    r"<(?:json|output|result)>(.*?)</(?:json|output|result)>",
    re.DOTALL | re.IGNORECASE,
)
```

**第 2 层：括号配平扫描。** 写一个几十行的小扫描器，在忽略字符串内部引号和转义的前提下，找出配平的 `{...}` 或 `[...]`：

```python
def balanced_span(text: str, open_c="{", close_c="}") -> str | None:
    start = text.find(open_c)
    if start < 0:
        return None
    depth = in_str = esc = 0
    for i in range(start, len(text)):
        c = text[i]
        if in_str:
            if esc: esc = 0
            elif c == "\\": esc = 1
            elif c == '"': in_str = 0
        elif c == '"':
            in_str = 1
        elif c == open_c:
            depth += 1
        elif c == close_c:
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    return None
```

它不依赖任何格式约定，是围栏被模型"污染"时的兜底。对起始位置循环即可收集多个候选，逐个尝试解析。

**第 3 层：修复与校验。** 前几层全挂时用 `json_repair` 之类修尾逗号、单引号、注释；解析成功后必须再过一层 schema 校验（pydantic / jsonschema），确认字段存在且类型正确——"能 parse"和"数据正确"是两回事。

**外层：结果封装与重试。** 解析函数统一返回 `ParseResult(ok, data, raw, strategy)`，失败时原始输出落盘；需要重试时把解析错误回喂模型并收紧指令，上限设 2 次，避免 token 失控。

## 踩坑点

- **嵌套围栏**：让模型在 JSON 字段里返回代码，字段值里的 ``` 会骗过围栏正则，括号扫描是唯一可靠兜底。
- **贪婪与非贪婪都有坑**：`\{.*\}` 匹配到最后一个 `}`，`\{.*?\}` 在嵌套对象处提前收尾。这不是调参数能解决的，别用正则做配平。
- **repair 不是银弹**：修复器可能把非法输出修成"合法但语义错误"的对象（比如静默丢字段），schema 校验不能省。
- **大小写与变体**：`<JSON>`、`<Result>`、无语言标注的 ``` 都会出现，匹配一律 ignore case、语言标注可选。
- **流式场景**：分片到达时 JSON 尚未闭合，边收边 parse 必挂，要么缓冲到配平，要么用增量解析器。
- **别迷信 json mode / function calling**：换模型、换供应商、或输出经过 MCP 工具转发后格式仍可能漂移。结构化输出接口作为首选，fallback 链作为保险。

## 可复用建议

1. 解析逻辑收敛到一个模块，不要在各个插件里散落正则。格式漂移是全局问题，修复点应该只有一处。
2. 把线上真实失败样本沉淀成 fixture 测试集，每种新花样加一个 case。这套测试集的价值随时间指数增长。
3. 优先使用结构化输出能力，降低进入 fallback 链的概率，但不要删除 fallback。
4. 失败日志必须包含原始输出和命中的策略层，只打 `JSONDecodeError` 等于没打。

## 总结

LLM 输出解析本质上是和一个不稳定的对端做协议通信：对方遵守约定的概率很高但不是 100%。把原始输出当作不可信输入来处理——快路径、逐层降级、修复后强制校验、失败留痕——这套分层管线代码量不到两百行，却能把结构化链路的有效成功率从"大概率"推到工程可接受的水平。在 agent 和自动化场景里，这类不起眼的胶水层往往才是稳定性的真正来源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/2f4ed5bc6e287eeb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/0a4bf3f49df95e5e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/76b072c4df802125.png)

