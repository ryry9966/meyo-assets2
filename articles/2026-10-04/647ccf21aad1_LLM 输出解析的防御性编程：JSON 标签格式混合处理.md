---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 40322
source: 综合讨论
publishedAt: 2026-10-04
---

# 背景

在 OpenClaw 的插件和 MCP 工具链里，让 LLM 返回结构化 JSON 是最常见的集成方式。提示词里写了"只输出 JSON"，模型也"基本听话"，但跑长任务、多轮调用时输出格式会漂移：有时是裸 JSON，有时包一层 ``` 围栏，有时套 `<json></json>` 标签，偶尔还混一句"好的，以下是结果"。解析器只按一种格式写，跑一百次总有几次崩。

# 问题

典型症状：

- `json.loads` 直接抛异常，任务链中断；
- 用正则提取 `<json>(.*?)</json>`，模型却改用了围栏，匹配不到；
- 拿到 JSON 了，但带尾逗号、单引号或注释，反序列化失败；
- 输出被 `max_tokens` 截断，JSON 只剩半截。

# 做法

把解析拆成四层，逐层兜底：

**第一层：提取。** 不假定单一格式，按顺序尝试：剥掉代码围栏 → 提取标签内容 → 从首个 `{` 做括号平衡扫描到匹配的 `}`（逐字符计数深度，维护字符串内外状态）。三种都失败就落盘原始输出。

**第二层：修复。** 对提取结果做清洗：去 BOM 和零宽字符、智能引号转直引号、去尾逗号、去行注释。每个修复步骤独立成小函数，一个失败不影响其他。

**第三层：校验。** 解析成功后用 pydantic/jsonschema 按 schema 校验字段与类型，缺字段给默认值或标记失败原因。

**第四层：重试。** 校验失败时，把原始输出和错误信息拼进修复提示词再调一次（temperature=0），明确要求"只输出 JSON，不要任何解释"。最多重试一次，仍失败就走降级路径。

骨架大致是：

```python
def robust_parse(raw: str, schema) -> dict | None:
    for candidate in extract_candidates(raw):  # 围栏/标签/括号扫描
        fixed = clean_and_repair(candidate)
        try:
            data = json.loads(fixed)
            return validate(data, schema) or None
        except Exception:
            continue
    return None  # 交由上层重试或降级
```

# 踩坑点

1. 别用正则匹配嵌套 JSON——字符串内容里含 `{` 或 `}` 时必翻车，括号平衡扫描更稳。
2. 围栏语言标注大小写不一：```` ```json ````、```` ```JSON ```` 都见过，剥围栏时别精确匹配 "json" 这个词。
3. 字符串内有转义引号时，简单按引号切分会破坏内容，扫描时要维护 `in_string` 状态。
4. 截断产生的半截 JSON 基本修不回来，检测到截断（finish_reason）应直接触发重试，别浪费时间做修复。
5. 一定要保存原始输出日志。没有原始日志，格式问题永远靠猜。

# 可复用建议

- 解析器做成独立工具模块，所有插件共用，别在每个工具里复制粘贴正则。
- structured output / JSON mode 能开就开，但别完全信任，兜底层照留。
- 结构化抽取固定 temperature=0，降低格式漂移概率。
- schema 关键字段的校验写宽松一点、配明确默认值，比"校验失败即崩溃"更抗造。

# 总结

LLM 输出格式漂移是常态而非异常。防御性解析的核心思路是：提取不假定格式、修复步骤隔离、校验按 schema 走、失败走有限重试和降级。多花半天写好这四层，比每次线上崩了再补正则划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/ae9917fd03ed34fc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/e2a8958a96f98b82.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/5e8739432b5adbe0.png)

