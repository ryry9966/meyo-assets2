---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 38272
source: 综合讨论
publishedAt: 2026-09-20
---

# LLM 输出解析的防御性编程：JSON 标签格式混合处理

## 背景

在 OpenClaw 插件、MCP 工具链和 Agent 流水线里，“让模型输出 JSON”几乎是最基础的约定。但只要跑过批量任务就会发现：同一份 prompt，模型可能返回纯 JSON、包在代码围栏里、前面加一句“以下是结果”、尾部多一个逗号，甚至把 `<think>` 推理段一起吐出来。提示词里写“只输出 JSON，不要任何其他内容”能降低概率，但工程上不能依赖概率。正确的姿势是：**把模型输出当作不可信的外部输入**，像处理用户上传文件一样做防御性解析。

## 问题

典型脏输出至少有五种形态：围栏包裹（` ```json ` 及大小写、空格变体）、前导/尾随说明文字、JSON 内部的尾逗号与单引号、推理标签混入、中文语境下的全角弯引号。任何一种都会让 `json.loads` 直接抛异常。单次对话里这只是一次重试；在自动化流水线里，一个节点的解析失败会让整条任务链雪崩，且失败往往集中发生在模型升级或 prompt 改动的当天——恰恰是最没空救火的时候。

## 做法

按五步搭一条解析链：

1. **Schema 先行**：用 Pydantic（或 zod）定义输出结构。解析的目标不是“能 load”，而是“能过 schema”。
2. **清洗**：剥掉 `<think>` 标签等包裹物。
3. **字符串感知的括号配对提取**：定位第一个 `{`，扫描到配对为止，天然兼容围栏和前后缀噪声。
4. **分级解析**：严格 `model_validate_json` → 容错修复库兜底。逐级降级，不要一上来就 repair。
5. **受控重试**：schema 失败时把原始输出和错误信息一并回传给模型重试，最多 1–2 次；无论成败都落日志。

核心代码（简化版）：

```python
import re, json_repair
from pydantic import BaseModel, ValidationError

THINK = re.compile(r"<think>[\s\S]*?</think>", re.I)

def parse_llm_json(raw: str, schema: type[BaseModel]):
    text = THINK.sub("", raw)
    start = text.find("{")
    if start < 0:
        raise ValueError("no JSON object found")
    depth, in_str, esc, body = 0, False, False, None
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        else:
            if ch == '"': in_str = True
            elif ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    body = text[start:i + 1]
                    break
    if body is None:
        raise ValueError("unbalanced braces")
    try:
        return schema.model_validate_json(body)   # 第一级：严格解析
    except ValidationError:
        repaired = json_repair.loads(body)        # 第二级：容错修复
        return schema.model_validate(repaired)    # 修复后仍必须过 schema
```

顶层是数组时把花括号换成方括号扫一遍即可，逻辑同构。

## 踩坑点

- **括号配对必须感知字符串**：JSON 字段值里出现 `{` 或 `}` 很常见，naive 的 `find`/计数会截出残缺 JSON，这也是“模型明明返回了合法 JSON 却解析失败”的高发原因。
- **不要在提取前就按围栏切**：模型有时在 JSON 字符串里再嵌一个代码块，非贪婪正则会提前截断。先找第一个 `{` 做配对，比先找围栏稳。
- **不要盲目信任 repair 库**：修复可能“语法正确但语义错误”，比如把数字猜成字符串。修复后必须再过一次 schema，否则脏数据会静默流入下游。
- **全角弯引号救不了**：中文上下文里模型偶尔输出 “ ”，连 repair 库也无能为力，只能靠 schema 失败后的重试兜底。
- **重试要有上限且带错误信息**：不带错误信息的重试大概率原样复现；不带上限的重试会无限烧 token，还拿不到失败原因。

## 可复用建议

- **解析器独立成模块，配失败样本回归集**：每次线上解析失败，把原始输出脱敏后存入测试集，在 CI 里跑。半年后这个集合就是最贵重的资产——换模型时跑一遍就知道行为差异。
- **能走结构化输出就走结构化输出**：模型原生 JSON mode / MCP structured output 能从源头消灭大半脏格式，防御解析留给不支持的场景。
- **监控按“模型版本 × prompt 版本”聚合解析失败率**：失败率突增几乎总是意味着上游变了，通常比用户报障早几天。

## 总结

对 LLM 输出做解析，本质上是对不可信数据源做协议协商。清洗 → 提取 → 分级解析 → schema 校验 → 受控重试，这条链路写一次，之后换模型、改 prompt 都只需改配置而不是救火。防御性解析不追求让脏输出“看起来能过”，而是保证：能过的真过了，过不了的有记录、有原因、有边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/1791e35fe279e982.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/38f48a550887f8ea.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/dd986a66aa44acce.png)

