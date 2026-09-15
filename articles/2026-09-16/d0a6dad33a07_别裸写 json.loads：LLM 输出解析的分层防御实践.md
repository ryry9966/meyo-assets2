---
title: 别裸写 json.loads：LLM 输出解析的分层防御实践
feedId: 37769
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

做 Agent、MCP 工具调用或自动化流水线时，让模型输出 JSON 是最常见的约定。但只要链路里存在第三方网关、旧模型，或者模型需要在 markdown 回复中夹带结构化数据，"输出一定是合法 JSON"这个假设很快就会被打破。实践里同一个 prompt 在不同模型、不同温度下能产生至少五种形态。解析层不做防御，整条链路的稳定性上限就被最弱的那个模型决定。

## 问题：输出形态远比你想象的多

实际遇到的至少有这些：

- 纯 JSON 裸文本
- ` ```json ` 围栏包裹
- ` ``` ` 无语言标注的围栏
- `<json>...</json>`、`<result>...</result>` 这类 XML 风格标签
- JSON 前后带说明性文字
- 带注释、尾逗号的 JSONC
- 中文语境下混入全角引号、全角冒号
- 字符串字段里嵌套 ` ``` ` 代码块（比如让模型把代码写进某个字段）
- `max_tokens` 截断导致的半截 JSON

裸 `json.loads` 只吃得下第一种。

## 做法：从严格到宽松的分层管线

先说原则：**能走原生结构化输出（function calling / JSON Schema 约束 / JSON mode）就不要靠解析**，解析器是兜底，不是第一道防线。兜底管线按严格程度分层，逐层降级：

```python
import json, re

def extract_json(text: str):
    try:                                    # L1 直接解析
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    for pat in (                            # L2 标签/围栏，优先带语言标注
        r"<(?:json|result)>(.*?)</(?:json|result)>",
        r"```json\s*(.*?)```",
        r"```\s*(.*?)```"):
        m = re.search(pat, text, re.S | re.I)
        if m:
            try: return json.loads(m.group(1))
            except json.JSONDecodeError: continue
    frag = balanced_scan(text)              # L3 字符串感知的括号配平
    for cand in ([frag] if frag else []) + repair_variants(frag or text):
        try: return json.loads(cand)        # L4 修复层：尾逗号/注释/全角标点
        except json.JSONDecodeError: continue
    raise ParseError(text)
```

`balanced_scan` 的关键是从第一个 `{` 或 `[` 开始扫描，进入字符串字面量时跳过内部的 `{}` 和引号，并正确处理转义。解析成功之后还要接 pydantic 或 JSON Schema 校验——**解析成功不等于结构合法**。截断场景先看 `finish_reason`，是长度截断就直接走补括号或重试，别浪费解析成本。

## 踩坑点

1. **围栏正则遇到嵌套 ` ``` ` 会截半截**，字符串里含代码块时非贪婪匹配取到一半；多个代码块时还可能抓错块。优先匹配带语言标注的块，失败再试下一个。
2. **全局替换引号、冒号会破坏字段值里的中文内容**。全角标点归一化只能放最后一层，且只在严格解析全失败后启用，命中要记日志。
3. **括号配平不跳过字符串字面量**，遇到 `{"a":"}"}` 就数错。
4. **别追求 100% 解析率**。解析失败应走"带错误信息重试一次"的回路：把 `JSONDecodeError` 的行列号原样回传给模型，多数一次就能修好，比继续堆正则划算。
5. 失败样本落日志注意脱敏，原始输出可能带用户数据。

## 可复用建议

- 解析收口到一个工具模块，**记录每次命中哪一层**。某层命中率突然升高，通常说明上游 prompt 或模型版本变了，这是免费的回归信号。
- 把线上解析失败样本沉淀成 golden 测试集，改 prompt 前跑一遍。
- 四件套固定下来：结构化输出优先 → 容错解析兜底 → schema 校验 → 错误反馈重试。

## 总结

防御性解析不是为了消灭失败，而是把失败变成可观测、可分级处理的事件。管线的每一层都留遥测，让数据告诉你该修 prompt，还是该修解析器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/1f604c3010d3946c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/9ceac984678ae041.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/74ada789f24a6825.png)

