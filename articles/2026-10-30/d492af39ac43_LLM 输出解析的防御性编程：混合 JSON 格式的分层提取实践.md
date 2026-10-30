---
title: LLM 输出解析的防御性编程：混合 JSON 格式的分层提取实践
feedId: 43059
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

在 OpenClaw 插件和 Agent 流程里，让模型返回 JSON 是高频需求：工具参数抽取、意图路由、批量字段提取。提示词里写明"只输出 JSON"，测试时也确实正常，但上线跑一段时间就会发现解析失败率远比预期高。原因不是模型不听话，而是输出格式本身是一个随模型版本、温度、上下文长度波动的分布，不是定值。防御性解析要解决的就是这个问题。

## 问题：你以为的 JSON，往往不是 JSON

同一套提示词，实测踩过这些形态：

1. ` ```json ` 围栏包裹；
2. `<json>...</json>` 标签，还有大写 `<JSON>` 变体；
3. 裸 JSON 前后带一句"以下是解析结果："；
4. 字段值里嵌套含反引号的代码块，把围栏正则切花；
5. 尾逗号、单引号、全角引号和全角逗号（中文上下文尤其高发）；
6. 输出两个对象，或对象外面莫名套一层数组。

裸 `json.loads` 只能覆盖少数情况。`\{.*\}` 这类正则也不行：贪婪匹配会把前后解释文字里的大括号一起吞进来，非贪婪又在嵌套 JSON 处提前截断。

## 做法：分层解析管线

核心思路：不指望单一手段通吃，而是按"可信度从高到低"搭一条管线，逐层尝试，命中即返回，并记录命中层级。

- **L0 直接解析**：先 strip 再 `loads`，干净输出直接通过；
- **L1 显式标签**：优先于围栏，因为它更不容易被内容里的代码块干扰；
- **L2 围栏提取**：处理最常见的 ` ```json ` 包裹；
- **L3 平衡扫描兜底**：从首个 `{` 扫到配对的 `}`，不依赖正则；
- **L4 修复层**：候选解析失败后再修一次（去尾逗号、全角转半角、单引号转双引号），懒得维护就上 `json_repair` 库；
- **L5 校验与重试**：解析成功只是入门，用 pydantic 按 schema 校验；不通过则把报错和原文回传模型重试一次。

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG   = re.compile(r"<json>\s*(.*?)\s*</json>", re.S | re.I)

def balanced_scan(text: str):
    out, depth, start = [], 0, None
    for i, ch in enumerate(text):
        if ch == "{":
            start = start if depth else i
            depth += 1
        elif ch == "}" and depth:
            depth -= 1
            if depth == 0:
                out.append(text[start:i + 1])
    return out  # 简易版不感知字符串内花括号，见踩坑

def extract_json(text: str):
    text = text.strip()
    try:
        return json.loads(text)            # L0
    except json.JSONDecodeError:
        pass
    candidates  = TAG.findall(text)        # L1
    candidates += FENCE.findall(text)      # L2
    candidates += balanced_scan(text)      # L3
    for cand in candidates:
        # repair(): 去尾逗号/全角转半角/单引号转双引号
        for fixed in (cand, repair(cand)):     # L4
            try:
                return json.loads(fixed)
            except json.JSONDecodeError:
                continue
    raise ValueError("no parsable JSON")       # L5 抛错，不返回 {}
```

## 踩坑记录

- **嵌套与字符串内的花括号**：上面的平衡扫描没处理字符串字面量里的 `{}`，字段值可能含 JSON 片段的场景，要么换带状态机的解析器，要么交给 `json_repair`。
- **字段值里含代码块**：围栏正则遇到值里的 ` ``` ` 必然切错位置，这是 L1 放在 L2 前面的原因；更彻底的办法是提示词里明确要求用自定义标签而非围栏。
- **全角标点**：中文上下文里全角引号、全角逗号出现率不低，修复层值得单独保留。
- **静默失败**：解析失败千万别返回 `{}` 兜底，下游会把脏数据当正常数据继续跑。要么抛错，要么走带具体报错的重试。
- **格式漂移**：temperature 偏高时波动明显，结构化任务能压低温就压低温；模型或网关支持 JSON mode / structured output 时优先用 API 层能力——解析器是兜底，不是首选。

## 可复用的建议

- 把 `extract_json` 收进公共工具包，全项目统一入口，避免每个插件各写一套正则；
- 日志里记录原始输出和命中层级，跑一周就能统计各层命中率，解析失败的原文直接沉淀为回归测试用例；
- 坏案例喂回提示词做反例 few-shot，比盲调提示词有效；
- 重试控制在 1~2 次，报错要具体——"第 12 行有尾逗号"远比"解析失败"有用。

## 总结

防御性解析的本质不是和模型斗智斗勇，而是承认输出格式是一个分布。分层提取解决"找不到 JSON"，修复层解决"JSON 不合法"，schema 校验解决"合法但不对"，日志和重试保证坏案例可观测、可回收。四件事凑齐，解析失败率基本能压到千分位，剩下的也能被日志持续捞出来修。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/74557e44fbc7933b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/27e90676ac80c76f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/c1ccb9dc0e6c2ed6.png)

