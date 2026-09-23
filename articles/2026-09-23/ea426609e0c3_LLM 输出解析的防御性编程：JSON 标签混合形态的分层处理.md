---
title: LLM 输出解析的防御性编程：JSON 标签混合形态的分层处理
feedId: 38643
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

在 OpenClaw 的插件和自动化流程里，LLM 的输出往往不是终点，而是下游的输入：MCP 工具调用的参数、Agent 循环中的结构化决策、定时任务里落库的数据。常见做法是在 prompt 里约定格式，比如"只输出 JSON，用 `<json></json>` 包裹"。但这个约定只在理想模型、理想温度、理想上下文下成立。

## 问题

同一套 prompt 换个模型或换个 system prompt 组合，实际输出至少有这几种形态：fenced code block（```json、```JSON）、XML 风格标签（`<json>`、`<output>`，大小写不定，偶尔还带属性）、裸 JSON 前后带一句"以下是结果"，甚至出现两个候选片段。更麻烦的是截断：max_tokens 打断后拿到半个对象。

严格解析器遇到这些形态直接抛异常，触发整轮重试——烧 token、拉高延迟。最糟的情况是某些"清洗"逻辑解析出一个语法合法但内容错误的片段，静默污染下游。

## 做法

我们把解析拆成三层，每层独立可测：

**1. 提取**：按优先级尝试多条路径，命中即返回候选文本：
- fenced code block，兼容语言标注大小写；
- XML 风格标签，标签名走配置列表，容忍属性和大小写；
- 都没有时，从第一个 `{` 做括号平衡扫描——逐字符跟踪字符串内状态与转义，深度归零即为终点。核心逻辑十几行：

```python
def extract_balanced(s, start):
    depth, in_str, esc = 0, False, False
    for i in range(start, len(s)):
        c = s[i]
        if in_str:
            if esc: esc = False
            elif c == '\\': esc = True
            elif c == '"': in_str = False
        else:
            if c == '"': in_str = True
            elif c == '{': depth += 1
            elif c == '}':
                depth -= 1
                if depth == 0:
                    return s[start:i+1]
    return None
```

**2. 修复**：解析失败再走宽松层——去尾逗号、给截断输出补右括号、处理 NaN/Infinity。可以用 json-repair 类库，但要清楚它可能"修"出合法却语义错误的结果。

**3. 校验**：pydantic / zod 做 schema 校验。存在多个候选片段时，取第一个通过校验的，而不是最长的。校验失败就把具体报错拼进 prompt，temperature 降 0 重试，最多 2 次。

另外对每次命中哪条提取路径打点。格式漂移率是很有用的信号：某天 fenced 占比从 90% 掉到 40%，大概率是模型或上游 prompt 变了。

## 踩坑点

- 平衡扫描必须跟踪字符串内状态，否则 `"key": "a}b"` 会让深度提前归零。
- 单引号转双引号的清洗会破坏 `it's` 这类合法内容，不要做全局替换。
- 截断补全能通过语法校验，但字段可能缺一半，schema 必须校验必填字段，而不是只看能不能 parse。
- 非贪婪正则在字符串包含 ``` 时会截断，贪婪正则会把两个代码块粘成一个。正则只适合初筛，平衡扫描才是兜底。
- 重试时只说"格式错了"没用，要把 parser 的具体报错和原始输出片段一起回传给模型。

## 可复用建议

- 提取器写成纯函数；线上每次解析失败把原始输出落盘，攒成 badcase 语料库，跑回归单测。
- 标签列表、fence 约定做成配置，不同 MCP 工具、不同模型可以注入各自的约定。
- 三层保持职责独立：提取错了修复层救不回来，修复层也不要偷偷干提取的活。
- 把格式漂移率当作模型更换的回归指标之一。

## 总结

防御性解析的核心不是把 prompt 写得更凶，而是承认输出格式天然不可控。提取、修复、校验三层各司其职，配合 badcase 语料库和漂移监控，Agent 管线对模型切换和 prompt 变更的容忍度会明显提高。我们踩过的坑，几乎都源于"相信了某一种格式不会变"——把不确定性当常态来设计，解析层自然就稳了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/6fda6b9c34657571.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/92654e105404fb83.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/db795074f6855857.png)

