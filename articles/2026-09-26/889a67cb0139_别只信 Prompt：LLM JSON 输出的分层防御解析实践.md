---
title: 别只信 Prompt：LLM JSON 输出的分层防御解析实践
feedId: 39124
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，让模型输出结构化 JSON 是常态：Agent 决策、工具参数、批处理任务都依赖它。提示词里写一句"只输出 JSON"，听起来就够了，实际远远不够。

## 问题

同一个 prompt，换个模型、调高温度、上下文变长之后，输出格式就开始漂移：

- 有时是 ```json 围栏，有时是无语言标注的 ``` 围栏，有时干脆裸输出；
- 前面偶尔带一句"好的，以下是结果："；
- 偶发尾逗号、单引号、中文引号、注释，甚至一次吐出两个 JSON 块。

按单一格式写的解析器，上线后隔三差五崩一次，崩法每次还不一样。这类故障靠重试救不了——重试的是同一个 prompt，格式错误往往会复现。

## 做法：分层解析，逐级降级

核心思路是把解析做成流水线，每层只处理自己擅长的失败模式：

```python
def parse_llm_json(raw: str):
    attempts = [
        direct_parse,      # 1. 直接 json.loads
        strip_fences,      # 2. 剥离 markdown 围栏后逐块尝试
        extract_balanced,  # 3. 字符串感知的括号匹配，截取第一个平衡的 {...}
        repair_and_parse,  # 4. jsonrepair / json5 兜底
    ]
    for fn in attempts:
        r = fn(raw)
        if r.ok:
            return validate(r.data)   # 5. schema 校验
    return ParseResult(ok=False, raw=raw)
```

几个实现细节：

1. **围栏剥离用非贪婪正则**：```` ```(?:json)?\s*([\s\S]*?)``` ````，取出所有候选块逐个尝试，不要只取第一个。
2. **括号匹配必须感知字符串**：扫描时跟踪是否处于引号内以及转义状态，否则 JSON 字符串里的 `{` 会把深度算错。这是最容易写错的一层。
3. **修复库放最后**：jsonrepair 能救尾逗号、缺引号，但它可能"猜"出与原意不同的结构，修复后必须再过 schema 校验。
4. **parse 成功 ≠ 数据正确**：统一用 pydantic / zod 校验字段，把"解析失败"和"语义错误"分成两类指标统计。

## 踩坑点

- `{.*}` 贪婪匹配会把前后多个块吞成一个，几乎必然出错；
- Python 的 `json.loads` 默认接受 `NaN/Infinity`，不是合法 JSON，下游序列化会炸；
- 流式场景拿到的是半截 JSON，需要增量解析或攒齐缓冲，别拿分片走完整流水线；
- 修复中文引号（""）要谨慎，盲目替换可能破坏字符串内容本身；
- 失败日志记得脱敏，raw 输出里可能带着用户数据。

## 可复用建议

- 收敛到一个统一模块，全项目只此一处，不要在业务代码里散落 try/except；
- 返回结构化结果 `{ok, data, error, raw}`，让调用方决定降级策略，而不是抛异常打断流程；
- 把线上真实失败样本沉淀成测试用例集，回归时全量跑一遍；
- 记录各模型、各 prompt 版本的解析失败率——格式漂移是可以被观测到的；
- 模型侧能开 JSON mode / tool calling 就开，但防御解析器照留，它保的是下限。

## 总结

LLM 的输出格式无法契约化，解析层就必须契约化。分层降级 + schema 校验 + 失败观测，三件事做齐，结构化输出的可用性会从"基本能跑"变成"可以托付自动化流程"。Prompt 负责提高正确率，解析器负责兜住错误，两者缺一不可。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/d5557c1b9d1edce5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/87f90d33d5cb5781.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/105cca57353638d4.png)

