---
title: 别再裸 split 了：LLM 输出 JSON 标签格式的防御性解析实践
feedId: 42755
source: 综合讨论
publishedAt: 2026-10-27
---

# 背景

在 OpenClaw 插件和 MCP 工具链里，让模型输出结构化 JSON 是常态：工具调用参数、子任务分发、评审打分……我们通常在 prompt 里约定“用 ```json 代码块包裹”。但只要换模型、升级版本、或者 prompt 稍有改动，输出格式就开始漂移：有时有围栏，有时没有；有时改用 `<json>` 标签；有时前后多一句解释；有时字符串里嵌着代码片段，把围栏提前闭合。

# 问题

最典型的故障是解析层只写了一种提取逻辑：

```python
raw = resp.split("```json")[1].split("```")[0]
data = json.loads(raw)
```

上线时能跑，跑两周就开始报 `IndexError` 或 `JSONDecodeError`。这类问题难排查在于：失败是概率性的，而且日志里往往只留了异常、没留原始输出，事后无法复盘模型当时到底输出了什么。

# 做法

把解析做成一条防御性流水线，分四层：

1. **提取层**：按优先级尝试多种候选——带语言标注的围栏、任意围栏块、`<json>`/`<result>` 类标签、配平扫描取最大的一对 `{}`。对每个候选拿去验证，取第一个能通过校验的，而不是盲目取第一个匹配。
2. **清洗层**：去 BOM、去首尾空白、去掉尾逗号。不要全局替换中文引号，原因见踩坑。
3. **解析层**：先 `json.loads` 严格解析，失败后降级到 `json_repair` 这类容错库。修复不是终点，修复结果必须再过校验。
4. **校验与重试层**：用 pydantic 定义 schema，字段缺失或类型错就带着具体解析错误回传模型重试，最多两次；仍失败则落盘原始输出，转人工队列。

```python
last_err = None
for cand in extract_candidates(resp):
    obj = try_parse(clean(cand))
    if obj is None:
        continue
    try:
        return Schema.model_validate(obj)
    except ValidationError as e:
        last_err = e
raise ParseError(last_err, raw=resp)  # raw 必须进日志
```

# 踩坑点

- 用正则 `.*?` 提取 JSON 时，字符串里含 ``` 或嵌套大括号会提前截断，配平扫描比纯正则可靠。
- 全局把中文引号换成英文引号，会误伤合法包含中文引号的字符串值；只在解析失败后做“定向修复”。
- 模型输出多个代码块时常常在回显示例，取第一个块拿到的可能是例子不是答案，逐块验证更稳。
- 流式场景别每个 chunk 都 parse，维护括号深度，finish 后再修复解析。
- `json_repair` 可能把非法输入“修”成合法但语义错误的数据，schema 校验不能省。

# 可复用建议

- 把流水线封装成纯函数工具，用线上收集的真实坏输出做单元测试集，越攒越厚。
- 解析失败率按模型、prompt 版本打点，换模型后失败率突增能立刻发现，而不是等用户报障。
- 失败时原始输出全量落盘（注意脱敏），没有 raw 的失败日志基本等于白记。
- 上游支持 JSON mode 就优先开，但容错层照留——多模型路由下，不能假设所有供应商行为一致。

# 总结

LLM 输出本质是不可信输入。和模型之间的“格式约定”是尽力而为，真正的契约是你的解析层：提取、清洗、解析、校验、重试各司其职，任何一环失败都有迹可循。防御性解析多花的几十行代码，换来的是自动化任务从“偶尔挂”变成“挂了也知道为什么”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/59a93c7d4daa7c66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f3d8e052715a49d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/6c12107bf3801d5e.png)

