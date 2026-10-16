---
title: LLM 输出解析的防御性编程：JSON 与标签混排的处理实践
feedId: 41700
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

在 OpenClaw 里写 skill、插件或 MCP 工具时，一个高频需求是让模型按约定输出结构化数据：要么纯 JSON，要么包进 `<tool_call>`、`<result>` 这类标签。Prompt 里写得清清楚楚，本地测十条全对，一进真实自动化流程就开始报 `json.loads` 失败。

## 问题

实际采样下来，失败形态大致有这几类：

- 纯 JSON 直接返回（理想情况，占比并不高）；
- 包在 ```json 围栏里；
- 标签和围栏混用，比如 `<result>` 里再套一层围栏；
- JSON 前后带解说文字：“以下是您要的 JSON：…”；
- 中文内容带出的全角引号、尾逗号；
- 输出被 max_tokens 截断，括号没闭合。

单一策略必然漏，做法只有一个：分层降级。

## 做法

按成本从低到高排一个解析链：

1. 原文 trim 后直接 `json.loads`，快路径覆盖大部分干净输出；
2. 剥掉所有代码围栏，逐块尝试；
3. 提取已知标签的内部内容，递归走一遍；
4. 字符串感知的括号配平，从第一个 `{` 截到配对的 `}`；
5. 还不行，交给宽松修复库兜底。

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*([\s\S]*?)```")
TAG = re.compile(r"<(?:result|json|tool_call)>\s*([\s\S]*?)\s*</")

def parse_llm_json(text: str):
    text = text.strip()
    for cand in [text] + [m.strip() for m in FENCE.findall(text)]:
        try:
            return json.loads(cand)
        except Exception:
            pass
    m = TAG.search(text)
    if m:
        return parse_llm_json(m.group(1))
    s = text.find("{")
    if s != -1:  # 字符串感知的括号配平
        depth, in_str, esc = 0, False, False
        for i in range(s, len(text)):
            ch = text[i]
            if in_str:
                if esc: esc = False
                elif ch == "\\": esc = True
                elif ch == '"': in_str = False
            elif ch == '"':
                in_str = True
            elif ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    return parse_llm_json(text[s:i + 1])
    from json_repair import repair_json  # 宽松修复兜底
    return json.loads(repair_json(text))
```

解析成功不等于结构正确。拿到 dict 后立刻过一遍 pydantic 模型校验字段和类型；失败就带着“原始输出 + 具体校验错误”回喂模型重试一次，比笼统地说“请重新输出”成功率高得多。

## 踩坑点

- **括号配平必须跟踪字符串状态**。`{"note": "用 {大括号}"}` 这种内容，天真的 depth 计数会提前截断，`\"` 转义也要处理。
- **围栏正则别用贪婪匹配**。多代码块场景会一路吞到第二个块的结尾。
- **`json.loads` 是放行的**：NaN、Infinity、重复键都能过，下游走 strict 序列化时才炸，最好在解析层就拦。
- **截断输出配出来的结果是“猜”的**，结构大概率不对，宁可检测到未闭合就走续写，别硬修。
- **别一失败就重试**。先让解析器干活，重试是最后手段，也最贵。

## 可复用建议

- 收敛成单入口函数 `parse_llm_json(raw, schema)`，全项目只走这一条路，方便统计失败率、做变更灰度。
- 解析失败时把原始输出脱敏后落盘，这是改 prompt 和判断模型版本回归最直接的依据。
- 平台支持 structured output / function calling 就优先用，防御解析做兜底——工具调用的 JSON 经中间层转发同样可能被改写。
- Prompt 里给标签约定时附一个负面示例（“不要加围栏，不要加解释”），能降低混排概率，但消灭不了它。

## 总结

和 LLM 打交道，输出解析的健壮性是自动化流程能否无人值守的分水岭。分层降级、schema 校验、失败留痕，三件事做到位，大部分“模型不听话”的问题就变成了可统计、可修复的工程问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/fb42490c4872a453.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/d7b9975e4b64e9eb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/0489d09efc0382bd.png)

