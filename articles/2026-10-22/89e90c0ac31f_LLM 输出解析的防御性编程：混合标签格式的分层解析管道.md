---
title: LLM 输出解析的防御性编程：混合标签格式的分层解析管道
feedId: 42290
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

在 OpenClaw 里写插件、接 MCP 工具，很多环节都依赖模型输出结构化 JSON：路由分发、参数抽取、批量标注、定时任务的决策结果。Prompt 里写一句"仅输出 JSON，不要任何解释"，听起来很稳，实测是另一回事——同一个 prompt，换个模型、调个温度、上下文再长一点，输出形态能变出五六种。解析层如果是裸的 `json.loads`，长跑任务迟早随机炸一次，而且往往炸在跑了很久之后。

## 问题：解析器要面对的野生输出

实际收集到的形态：

- 纯 JSON 直接可解析（占比没想象中高）
- Markdown 围栏包裹，带或不带语言标注
- XML 风格标签包裹：`<json>`、`<answer>`，部分模型还会自发输出 `<tool_call>`
- JSON 前后带一段"解析结果如下："式的人话
- 嵌套围栏：外层四反引号包着一段含 ```json 的示例代码，一不小心把示例当结果
- 近似 JSON：尾逗号、单引号、字符串里未转义的换行
- 被最大输出长度截断的残缺 JSON

这些格式没有对错，是不同模型训练数据的自然差异。解析层的工作不是纠正模型，而是把它们都接住。

## 做法：先便宜后昂贵的分层管道

```python
import json, re

FENCE = re.compile(r"(`{3,})[ \t]*\w*[^\n]*\n(.*?)\1", re.S)
TAG   = re.compile(r"<(?:json|answer|result|tool_call)>(.*?)</", re.S)

def extract_json(text: str):
    s = text.strip()
    try:                                # 1) 裸 JSON
        return json.loads(s)
    except json.JSONDecodeError:
        pass
    while True:                         # 2) 剥围栏，while 处理嵌套
        m = FENCE.search(s)
        if not m: break
        s = m.group(2).strip()
        try:
            return json.loads(s)
        except json.JSONDecodeError:
            continue
    m = TAG.search(s)                   # 3) 剥 XML 风格标签
    if m:
        s = m.group(1).strip()
        try: return json.loads(s)
        except json.JSONDecodeError: pass
    s = balanced_slice(s)               # 4) 括号平衡截取
    return json.loads(repair(s))        # 5) 语法修复兜底
```

几个关键点：

- **围栏正则用反向引用 `\1`**，保证开合围栏等长；外层 `while` 处理四反引号包三反引号的嵌套场景。
- **第 4 步别用贪婪正则**。`\{.*\}` 看似能取最外层，但字符串值里出现 `}` 就废了。写个二三十行的小函数：维护 in_string 和转义状态，逐字符做括号计数，从第一个未闭合的 `{` 截到它匹配的 `}`。偷懒方案是拿 `json.JSONDecoder().raw_decode` 对每个 `{` 的位置逐个尝试，解析成功的就是真 JSON。
- **修复层只做语法级**：去尾逗号、单引号换双引号、补转义。社区有现成的 `json_repair` 库，不必自己造全套。字段缺失、类型不对是 schema 层的事，交给 Pydantic 报错，解析层不要硬猜。
- **重试要喂错误信息**：解析失败后把具体报错拼回 prompt——"第 3 行末尾多了逗号，请重新输出完整 JSON"——比干巴巴的"再来一次"成功率高得多。
- **原始输出和命中的解析路径落盘**。统计一段时间你会发现不同模型的格式偏好相当稳定，这些数据反过来能优化 prompt。

## 踩坑点

- `json.loads` 默认接受 `NaN`/`Infinity`，这不是合法 JSON，下游严格序列化时会再炸一次，用 `parse_constant` 拦掉。
- Prompt 里放完整 JSON 示例，模型会复读示例内容而非任务数据。示例只给 schema 骨架，值用占位符。
- 围栏剥离后如果内容仍以反引号开头，说明还有一层没剥干净，继续循环而不是返回 None。
- 别在解析层吞异常返回默认值。宁可抛错触发重试，静默降级产出的脏数据更难查。

## 可复用建议

- 解析器做成独立模块，所有工具入口统一走，不要每个插件各写一份。
- 场景允许时，在 prompt 端约定自定义标签（如 `<json>`）包裹输出，比围栏更不容易和示例代码冲突，解析也更确定。
- 每修一个线上 case，就把那份原始输出存进 fixture 回归集，这套测试会越来越贴你的真实流量。
- 解析层和 schema 校验分层：前者负责"捞出候选 JSON"，后者负责"验证并产出可回喂的错误描述"。

## 总结

混合标签格式是 LLM 输出的常态而非异常，防御性解析的本质是把"格式不可控"当作输入约束来设计：分层剥离、语法修复、schema 兜底、错误回喂、全程留痕。管道写一次，后面每个插件、每个 MCP 工具都受益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/62e1bd8197276ec9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b6de7efd6694787b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/76dbcb8f5bd52ca4.png)

