---
title: LLM 输出解析的防御性编程：当围栏、标签和裸 JSON 混在一起
feedId: 43170
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

在 OpenClaw 这类 Agent 网关和插件体系里，LLM 的输出经常不只是给人看的：工具调用参数、MCP 结果汇总、自动化流水线的中间状态，都要求模型吐出结构化 JSON。我们通常在提示词里要求"只输出 JSON"，甚至指定包在 `<tool_call>` 这类标签里。但实践中的共识是：**提示词约束不是契约**，你无法保证模型每次都照做。

## 问题

翻一下真实会话日志，常见形态至少有五种：

- 标准的 ` ```json ` 围栏代码块；
- 裸 JSON，前后还带着一段解释性文字；
- JSON 外面再套一层 XML 风格标签，如 `<answer>{...}</answer>`；
- 语法上"长得像" JSON 但 `json.loads` 直接炸：中文引号、尾逗号、行内注释；
- `max_tokens` 截断，JSON 只有一半。

任何一种单独出现都好办，麻烦在于它们**混合出现**，且不同模型、不同版本行为不同。很多插件第一步就 `json.loads(response)`，一次格式漂移，整条自动化链路挂掉。

## 做法

核心思路：把解析做成**分层管道**，每层只处理自己认识的形态，失败了往下一层走，而不是直接抛异常。

```python
import re

def extract_json(text: str) -> str:
    # 1. 优先精确匹配标签
    m = re.search(r"<(\w+)>\s*(\{.*?\})\s*</\1>", text, re.S)
    if m:
        text = m.group(2)
    # 2. 剥围栏（取最后一个，避免嵌套误伤）
    fences = re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
    if fences:
        text = fences[-1].strip()
    # 3. 兜底：从首个 { 或 [ 做括号配平扫描
    if not text.lstrip().startswith(("{", "[")):
        text = _balanced_slice(text)  # 自实现，跳过字符串内的括号
    return text
```

拿到候选字符串后再走两步：先用 `json-repair` 这类容错库修尾逗号和中文引号，再用 pydantic / jsonschema 做结构校验。校验失败就带着**具体错误**重试一次（"你上次输出有 X 问题，请只输出 JSON"），重试次数写死上限；超限走降级路径——转人工或返回固定错误结构，不要静默吞掉。

无论成败，把原始输出和命中的解析分支写进日志。排障时这比任何猜测都有用。

## 踩坑点

- **正则切嵌套 JSON 不可靠**：字符串里含反引号或多层大括号时 `.*?` 会切错，配平扫描比正则稳。
- **注意解析器语义**：`eval` 别碰；`ast.literal_eval` 不认 `true`/`null`；`json.loads` 反而默认接受 `NaN`——先确认你用的解析器接受什么。
- **流式输出别边收边 parse**：半截 JSON 白白消耗一次重试，等结束标记或用增量解析。
- **重试不换策略等于没重试**：同样的提示词连发三次，大概率三次都错，重试时必须附上失败原因。

## 可复用建议

- 把 `extract_json` 沉淀为插件公共库的统一函数，别每个插件各写一份，行为不一致最难查。
- 用真实故障样本建黄金测试集，每次换模型跑一遍解析通过率。
- 能用 function calling / 结构化输出就用，但保留文本解析兜底——不同后端支持程度不一。
- 按模型维度监控解析失败率，格式漂移往往先在指标上现形，再反映到用户投诉。

## 总结

LLM 的文本输出本质上是不可靠输入，应该像对待用户输入一样做防御。分层解析、容错修复、结构校验、有限重试、原始日志——这五件事成本不高，却能把"偶发挂掉"变成"偶发降级"，自动化流程的稳定性会上一个台阶。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/1b37aca557008b09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/f8e46107960b1b21.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/d2efe300f348f5b3.png)

