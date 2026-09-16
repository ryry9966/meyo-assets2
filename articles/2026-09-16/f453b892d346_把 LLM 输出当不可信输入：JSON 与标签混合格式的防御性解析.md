---
title: 把 LLM 输出当不可信输入：JSON 与标签混合格式的防御性解析
feedId: 37850
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

在 OpenClaw 插件和 Agent 工作流里，LLM 输出经常被当作结构化数据消费：工具调用参数、任务拆解结果、记忆条目、MCP 工具的元数据。提示词里写"只输出 JSON"，模型大多数时候也配合。但只要跑得够久、换过模型、改过几版提示词，下游解析迟早炸一次——要么 `json.loads` 抛异常中断流水线，要么被某个宽容的兜底逻辑吞掉、悄悄丢了字段，错误在几层之后才暴露。

## 常见的"脏输出"形态

实际见过的混合格式大致几类：

- 带 `json` 代码围栏，围栏外还有一句解释性废话；
- 用自造标签包裹，如 `<output>{...}</output>`、`<result>...</result>`；
- 纯 JSON 但前后混入 Markdown 列表或分隔线；
- 同一条回复里混着多段 JSON（一段说明 + 一段结果）；
- 伪 JSON：单引号、尾逗号、未加引号的 key；
- JSONL 被当单个 JSON 解析。

单看每一种都有简单解法，麻烦在于它们会随机组合出现。

## 做法：四层解析梯子

核心思路是解析做成逐层降级的函数，而不是一次正则赌运气：

```python
import json, re
import json_repair  # pip install json-repair

def extract_json(text: str):
    t = text.strip().lstrip("\ufeff")
    try:
        return json.loads(t)                    # L1 直接解析
    except Exception:
        pass
    fence = "`" * 3
    t2 = re.sub(fence + r"[a-zA-Z]*", "", t).strip()  # L2 剥围栏
    try:
        return json.loads(t2)
    except Exception:
        pass
    # L3 括号配对扫描，取第一个完整 JSON 块
    for start_ch, end_ch in (("{", "}"), ("[", "]")):
        start = t2.find(start_ch)
        if start == -1:
            continue
        depth, in_str, esc = 0, False, False
        for i, ch in enumerate(t2[start:], start):
            if in_str:
                if ch == '"' and not esc:
                    in_str = False
                esc = (ch == "\\" and not esc)
                continue
            if ch == '"':
                in_str = True
            elif ch == start_ch:
                depth += 1
            elif ch == end_ch:
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(t2[start:i + 1])
                    except Exception:
                        break
    return json_repair.loads(t2)               # L4 修复兜底
```

L5 不是代码而是流程：四层都失败时，带着原始输出与报错重试一次，提示词明确要求"只输出 JSON，不要围栏"，重试结果走同一套解析。

三个细节值得强调：括号扫描必须处理字符串内的引号，否则 JSON 值里出现 `{` 就会截断错位（`in_str/esc` 两个状态位就是干这个的）；不要用贪婪正则匹配 `\{.*\}`，多段 JSON 时会从第一个括号吞到最后一个；围栏匹配用 `[a-zA-Z]*` 宽松处理，模型会输出 `JSON`、`jsonc` 甚至 `js`。

## 踩坑点

- **示例即污染**：few-shot 里给了一个带围栏的示例，模型就会持续输出围栏。想要干净输出，示例本身别带围栏。
- **嵌套围栏**：让模型生成含代码字符串的 JSON 时，字符串里的三反引号会破坏所有按围栏切分的逻辑，基本只能靠 L3 的括号扫描救。
- **智能引号与零宽字符**：从聊天界面复制来的样本常带 U+201C / U+200B，本地通过、线上失败，排查半天。入口统一做字符规范化。
- **JSONL 歧义**：逐行能解析就按 JSONL 处理，但要先确认下游要的是列表还是单对象，两种语义别混。
- **修复库不是银弹**：`json_repair` 对严重损坏的输入会"脑补"出一个结构，字段可能对不上。L4 之后必须接 schema 校验，不能直接消费。

## 可复用建议

- 解析收敛到**一个工具模块**，禁止业务代码各自手写正则，全项目只有一处要改。
- 每层加计数埋点：L1 占比 98% 说明提示词健康；L4 占比升高就该回去改提示词，或优先用结构化输出（function calling / schema 约束解码），解析梯子只是纵深防御。
- 攒一个**脏输出语料库**：线上解析失败的原始文本脱敏后存下来做回归测试，这比拍脑袋写的用例值钱得多。
- 解析后用 pydantic / jsonschema 校验，顺手做 key 归一化（camelCase / snake_case），把模型命名习惯差异挡在边界层。

## 总结

LLM 文本输出本质上和外部 API 返回一样，是不可信输入。防御性解析不是替代提示词工程，而是承认"模型偶尔不守规矩"这个现实：结构化输出做主路径，解析梯子做兜底，埋点做观测，语料做回归。这四件事做完，凌晨三点被 `JSONDecodeError` 叫醒的次数会显著下降。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/74c3967159a8e458.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/4ee6c03336f7cc32.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/8c61183d9fb80fa5.png)

