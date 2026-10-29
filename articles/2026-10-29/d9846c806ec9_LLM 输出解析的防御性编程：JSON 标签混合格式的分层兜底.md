---
title: LLM 输出解析的防御性编程：JSON 标签混合格式的分层兜底
feedId: 42941
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

在 OpenClaw 插件和 Agent 流程里，让模型输出 JSON 是日常操作：工具调用参数、MCP 请求体、工作流节点配置，几乎都默认模型“只输出一个 JSON 对象”。理想很整齐，现实是同一段提示词在换模型、调温度、加长上下文之后，输出格式会悄悄漂移。

## 问题：真实输出长什么样

线上抓到的失败样本大致几类：

1. 裸 JSON 外面包着解释文字：“好的，以下是解析结果：{...}”
2. 围栏和标签混用：```json 包着 `<tool_call>`，或反过来嵌套
3. 一次回复里出现多个代码块，真正的答案在最后一个
4. max_tokens 截断，JSON 只剩半截
5. 语法小毛病：尾逗号、单引号、字符串里没转义的换行

如果解析器只写 `json.loads(response)`，以上全都会炸，而且炸在插件深处，日志里只有一行 JSONDecodeError。

## 做法：分层提取 + 修复阶梯

核心思路是把“提取”和“解析”拆开，各自兜底。

**第一步，候选区域提取**，按优先级走：

```python
import re, json

def extract_json(text: str) -> str:
    # 1) 标签优先，取最后一个匹配（通常是最终答案）
    for tag in ("tool_call", "json", "answer"):
        hits = re.findall(rf"<{tag}>(.*?)</{tag}>", text, re.S)
        if hits:
            text = hits[-1]
            break
    # 2) 代码围栏
    fences = re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
    if fences:
        text = fences[-1]
    # 3) 括号配对扫描（跟踪字符串状态）
    start = text.find("{")
    if start < 0:
        raise ValueError("no JSON object found")
    depth = in_str = esc = 0
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if esc: esc = 0
            elif ch == "\\": esc = 1
            elif ch == '"': in_str = 0
        elif ch == '"':
            in_str = 1
        elif ch == "{":
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    raise ValueError("unbalanced braces (possible truncation)")
```

**第二步，解析修复阶梯**，从轻到重：

```python
def parse_lenient(raw: str):
    for cand in (raw, re.sub(r",\s*([}\]])", r"\1", raw)):  # 原样 → 去尾逗号
        try:
            return json.loads(cand)
        except json.JSONDecodeError:
            pass
    return json_repair.loads(raw)  # 兜底库
```

**第三步，schema 校验收口。** 修复库可能把截断数组“补”成合法但语义错误的 JSON，所以修复后必须过一遍 pydantic/jsonschema，字段对不上就按解析失败处理。

**第四步，Agent 循环里不要直接崩。** 把解析错误当工具错误回传给模型：“上一次输出无法解析为 JSON（错误：unbalanced braces），请只输出一个 JSON 对象，不要附加说明。”重试一次，温度调低。多数模型这一轮就能自纠。

## 踩坑点

- 正则别用贪婪匹配，`.*` 会把多个代码块吞成一个。用非贪婪加“取最后一个”的策略。
- 括号配对必须跟踪字符串状态，否则字符串里的 `{` 会把深度算崩，这是最常见的翻车点。
- 别天真地全局替换单引号，英文里的撇号（it's）会被一起替换，制造更难查的脏数据。
- 截断输出不要硬修。扫描结束时 `in_str` 为真或 depth > 0，直接判定截断，走续写或报错。
- 修复要有数。每次进入修复分支就打点，修复率突然升高，往往说明提示词或模型变了，而不是代码坏了。

## 可复用建议

- 解析逻辑收口到公共模块，所有插件共用，别在 handler 里散落 `re.findall`。
- 攒一份“脏输出语料库”：线上解析失败就存原文，脱敏后做成单元测试 fixture，回归测试比任何设计文档都实在。
- 运行时支持原生 function calling / 结构化输出时优先用，但保留这层兜底——OpenClaw 场景经常换模型，兜底层就是可移植性。
- 日志保留原文 + 命中的提取策略 + 是否触发修复，解析失败率本身就是提示词质量的度量。

## 总结

把 LLM 输出当不可信输入处理，是 Agent 工程的基本素养。分层提取、修复阶梯、schema 校验、错误回传，四层加起来不到一百行代码，换来的是插件在格式漂移面前的韧性。防御性解析不是对模型能力的否定，而是让系统能力不依赖模型的心情。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/9f5495146a3b426b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/c3811d2a8162b66c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/99b21b5b4a49f453.png)

