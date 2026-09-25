---
title: LLM 输出解析的防御性编程：JSON 围栏与标签混合格式的分层处理
feedId: 39034
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

在 OpenClaw 的 agent、MCP 工具链和插件开发里，让 LLM 返回 JSON 几乎是日常：函数调用参数、流水线的结构化中间结果、自动化任务的产物。但即使 prompt 里明确写了「只输出 JSON，不要任何解释」，实际返回仍然五花八门：有时是裸 JSON，有时套了 ```json 围栏，有时前后各带一句「好的，以下是结果」，偶尔模型还会自创 `<json>...</json>` 标签。解析代码如果只按一种格式写，批处理任务凌晨三点挂掉并不稀奇。

## 问题

把线上实际踩到的格式归纳成三类：

1. **围栏混用**：```json、```JSON、无语言标注的 ```，甚至围栏不闭合；
2. **标签混用**：你要求用 `<result>` 包裹，模型有时遵从、有时换成 `<json>`、有时干脆不加；
3. **内容瑕疵**：尾逗号、首尾空行与 BOM、JSON 前后夹自然语言、一次吐出两个对象。

## 做法：分层降级解析

核心思路是不假设格式，准备多个候选提取路径，逐层尝试、逐层降级，任何一层成功即返回：

```python
import json, re

def extract_json(text: str):
    cands = [text.strip()]
    # L1 剥代码围栏（覆盖大小写与无语言标注）
    cands += re.findall(r"```(?:json|JSON)?\s*(.*?)```", text, re.S)
    # L2 剥自定义标签
    cands += re.findall(r"<(?:result|json|output)>(.*?)</(?:result|json|output)>",
                        text, re.S | re.I)
    for c in cands:
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    # L3 括号配平扫描，取第一个完整对象；L4 去尾逗号后二次 loads
    obj = _balanced_scan(text)
    return json.loads(re.sub(r",\s*([}\]])", r"\1", obj))
```

`_balanced_scan` 的关键是在遍历字符时维护 `in_string` 和 `escape` 两个状态位，跳过字符串内部的 `{}`——否则遇到 `"note: use {braces}"` 这类内容，计数直接就断了。所有层都失败时，抛出带上下文的异常，并把原始输出完整落盘。

## 踩坑点

- **贪心正则**：`.*` 会吞掉两个 JSON 对象之间的内容，用非贪婪 `.*?`，再让 L3 配平扫描兜底。
- **激进修复**：单引号转双引号会破坏英文所有格（如 it's）；json_repair 类库方便，但可能悄悄改值，修复后必须过一遍 schema 校验。
- **静默失败**：修复成功不等于没事，修复率突然升高往往说明模型或 prompt 漂移了，要作为指标上报，而不是闷头吞掉。
- **千万别 eval**：再像 JSON 也不行，这是安全问题，不是风格问题。

## 可复用建议

1. 解析逻辑收敛到一个模块，返回 `ParseResult(status, data, raw)`，让上游决定重试、降级还是告警，不要在业务代码里散落正则；
2. 运行时支持结构化输出或函数调用时优先用，防御性解析是兜底，不是首选；
3. 把线上失败样本沉淀成 golden test，prompt 每次改版跑一遍回归；
4. 失败必须落原始输出，否则没法复盘，也没法补充测试集。

## 总结

防御性解析的前提，是承认「模型输出不可控」。把格式不确定性圈在解析层内部消化：宽松提取、保守修复、响亮失败。做到这三点，批处理任务至少能让你睡到天亮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/16179f6b1928220b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/d933af976ef28715.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/974cb80d0b6aff82.png)

