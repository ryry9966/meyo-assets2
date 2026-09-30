---
title: LLM 输出解析的防御性编程：混合 JSON 格式的分层处理实践
feedId: 39908
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

在 OpenClaw 插件和 Agent 工作流里，我们经常要求 LLM 以 JSON 返回结构化结果：工具调用参数、MCP 工具入参、多步骤任务的中间状态。提示词里写一句「只输出 JSON，不要任何解释」听起来很稳，但只要换模型、调温度，或者输出内容里带中文，格式就会开始漂移。

## 问题

真实输出长这样：

````
好的，以下是解析结果：
```json
{"action": "search", "query": "OpenClaw “插件”配置", "limit": 5,}
```
如需调整请告诉我。
````

直接 `json.loads` 必挂。我翻过自己几个插件的失败日志，模式基本就几类：代码围栏包裹、前后带寒暄、中文引号、尾逗号、字符串里含 `{}`、偶尔一次吐出多段 JSONL。任何单点处理都会漏。

## 做法：分层解析管线

核心思路：把 LLM 输出当不可信输入，按成本从低到高逐层尝试。

1. **快路径**：剥掉 BOM 后直接 `json.loads`，能过就过，规整输出零开销。
2. **去围栏**：用锚定首尾的正则剥掉最外层 ``` 围栏，只剥最外层。
3. **括号配平提取**：定位第一个 `{` 或 `[`，扫描到配对位置。关键是扫描时跟踪字符串状态——引号内的大括号不计数，并处理 `\` 转义。
4. **修复层**：去尾逗号、中文引号替换成半角、`True/False/None` 转 JSON 字面量。修复动作要留日志。
5. **Schema 校验**：解析成功不代表数据可用，用 pydantic 校验字段与类型；失败就把解析错误拼回提示词重试一次，再失败走兜底分支。

最容易写错的是第 3 步，核心函数可以直接抄走：

```python
def extract_balanced(text: str, start: int) -> str:
    depth, in_str, esc = 0, False, False
    open_ch = text[start]
    close_ch = '}' if open_ch == '{' else ']'
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if esc: esc = False
            elif ch == '\\': esc = True
            elif ch == '"': in_str = False
        elif ch == '"':
            in_str = True
        elif ch == open_ch:
            depth += 1
        elif ch == close_ch:
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    raise ValueError("unbalanced JSON")
```

其余几层都是几行正则的事，建议整体封装成一个 util 模块，别散落在各插件里。

## 踩坑点

- **`\{.*\}` 贪婪正则**：遇到多段 JSON 或字符串内大括号直接错位；改非贪婪又会被嵌套截断。提取别用正则，用配平扫描。
- **数大括号不看引号**：`"pattern": "{"` 这种值会让计数崩掉，这是最常见的翻车点。
- **全局删围栏**：JSON 字符串值里可能本身含 ```，只能锚定开头结尾剥最外层那对。
- **静默修复**：修复库可能悄悄改数据，中文字符串尤其容易坏，每次修复必须记录。
- **解析成功 ≠ 语义正确**：字段缺失、类型错误靠 schema 层兜住，不要在解析层堆业务 if。

## 可复用建议

- 一个社区/一个项目共用同一份解析工具，避免各写一版、各踩一遍。
- 把线上真实坏输出沉淀成测试 fixture，每次改解析器全量回归。
- 按模型和提示词版本统计解析失败率，格式漂移能提前发现。
- 有 structured output / JSON mode 就优先用，但防御解析器保留着——你迟早会换模型。

## 总结

提示词负责「请求」格式，解析器负责「假设」格式会被破坏。快路径 → 去围栏 → 配平提取 → 修复 → schema 校验 → 一次带错误反馈的重试，这套管线在我们几个插件里把解析失败率从两位数压到接近零，代价只是几十行代码和一条日志约定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/89481db2fc1edeee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/4fafe45bbf1b46ed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/36e3492523990edd.png)

