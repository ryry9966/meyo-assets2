---
title: LLM 输出解析的防御性编程：JSON 混合格式的分层降级管线
feedId: 41383
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

在 OpenClaw 这类 Agent / 插件场景里，让模型输出 JSON 几乎是刚需：工具调用参数、MCP 消息体、插件间传递的中间结果。提示词里写"只输出 JSON，不要任何多余内容"，模型答应得很好——但线上跑一天，你就会收到各种"看起来差不多、但 `json.loads` 直接报错"的输出。

## 问题

实际混排格式大致几类：

1. Markdown 围栏包裹：```json ... ```
2. 围栏语言标注不统一：`json` / `jsonc` / `JSON5` / 空
3. 自定义标签再包一层：`<json>`、`<result>`，偶尔和围栏叠用
4. 前后缀客套话："好的，以下是……"
5. 尾逗号、全角引号、BOM、零宽字符
6. max_tokens 截断出的半截 JSON

失败率不是固定值，它随模型、温度、上下文长度波动。防御性解析的目的不是把烂输出变好，而是把"解析失败"从随机异常变成一条**有记录、可归因的路径**。

## 做法：分层提取，逐级降级，记录命中层

核心思路是构造一个候选列表，按顺序尝试，谁先成功用谁，并记录命中的是哪一层：

```python
import json, re
from json_repair import repair_json  # pip install json-repair

FENCE = re.compile(r"```[a-zA-Z0-9]*\s*(.+?)```", re.S)
TAG   = re.compile(r"<(json|result|response|output)>(.+?)</\1>", re.S | re.I)

def extract_json(text: str):
    cands = [("raw", text)]
    cands += [(f"fence{i}", m.group(1)) for i, m in enumerate(FENCE.finditer(text))]
    cands += [(f"tag{i}", m.group(2)) for i, m in enumerate(TAG.finditer(text))]
    m = re.search(r"[\[{]", text)
    if m:
        cands.append(("from_brace", text[m.start():]))
    for name, c in cands:
        try:
            return json.loads(c), name
        except json.JSONDecodeError:
            continue
    return repair_json(text, return_objects=True), "repair"
```

调用侧拿到 `(obj, layer)` 后打点。提取成功后**再过 schema 校验**（pydantic / dataclass），提取与校验职责分离：提取管"拿到合法 JSON"，校验管"字段对不对"。

## 踩坑点

- **正则贪婪**：`(.+)` 配 `re.S` 会跨围栏吞掉两个代码块。用非贪婪，且多围栏输出要逐个尝试，别只取第一个。
- **字符串内容里含 \`\`\`**：极少见但存在，围栏正则会切错位置——所以候选里永远保留 `raw` 和 `from_brace` 兜底。
- **全角引号与零宽字符**：repair 库能吃掉大部分，但先做一遍 `\u200b\ufeff""` 的 normalize，成本更低、行为更可控。
- **截断最危险**：repair 会把半截 JSON"成功"补成合法 JSON，你拿到的是静默错误数据。检测方式：对照 schema 必填字段，或检查原文是否以未闭合的字符串结尾；命中截断就走重试/续写，**不进 repair**。
- **流式场景**不要逐 chunk 解析，先做括号/引号平衡缓冲，平衡归零再喂给提取函数。

## 可复用建议

- 提取函数独立成模块、带版本号，全项目只此一处。散落在各处的 `strip("```")` 是事故来源。
- 攒 golden set：每次线上降级，把原文（可截断）存进样本库，回归测试直接吃这个库。
- 命中率打点当遥测看：fence 命中占比高，说明提示词没约束住；repair 占比高，说明该升级模型或上结构化输出了。
- 运行时支持原生结构化输出（tool schema / response_format）就优先用，手写提取只做兜底，两套并存。
- 提示词侧仍然要写清格式并给 one-shot 示例。防御性解析降低的是爆炸半径，不是对提示词要求的豁免。

## 总结

LLM 输出解析的鲁棒性不来自某个万能正则，而来自"多候选、有顺序、可观测"的降级管线，加上提取与校验的职责分离。把命中层当数据看，你多半会发现：大部分解析问题，其实是提示词问题的投影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/d53ad4512af8fc0d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/b3a2c6a768e8d066.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/a7d53149795fb3ae.png)

