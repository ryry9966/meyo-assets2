---
title: 别再裸 parse：LLM 混合标签输出的防御性 JSON 解析
feedId: 41534
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

在 OpenClaw 里写插件或 MCP tool，最常见的契约是：让模型输出一段 JSON，工具端解析后执行。提示词里写明"只输出 JSON"，但只要换模型、调温度，或经过几轮 subagent 转发，输出格式就会漂移——有时是裸 JSON，有时包在 ```json 围栏里，推理模型前面还挂一段 `<think>`，偶尔混进全角标点，`max_tokens` 卡到一半直接截断。裸 `json.loads` / `JSON.parse` 的代码，线上跑一周就会开始报错。

## 问题：格式不是一种，是一碗混合物

真实样本大致五类：

- **围栏包裹**：```json ...```，有时无语言标注，有时一个回复里多个代码块（推理过程里还嵌了示例 JSON）；
- **标签与说明文字**：`<think>`、`<result>` 前后夹杂"好的，结果如下："之类的话；
- **语法瑕疵**：尾逗号、单引号、`//` 注释、全角引号逗号、BOM 和零宽字符；
- **结构损坏**：截断导致括号不平衡；
- **parse 成功但不可用**：字段缺失、类型漂移——能 parse 不等于能用。

这些情况会出现在同一个部署里，单条正则解决不了。

## 做法：分层解析，逐层降级

把解析做成流水线，任一层成功即返回，全部失败才报错：

- **L0 规范化**：去 BOM、零宽字符和首尾空白；全角标点先不动，留给修复层。
- **L1 剥壳**：先删 `<think>...</think>` 块，再按已知标签提取。
- **L2 直接 parse**：运气好到此结束。
- **L3 围栏提取**：非贪婪匹配所有围栏块，逐个尝试。
- **L4 括号配对扫描**：找到第一个 `{` 或 `[`，做字符串感知的配对扫描，截出候选逐个尝试；多个候选取最后一个完整的——推理文本里的示例 JSON 几乎总在前面。
- **L5 修复**：交给 `json_repair`（Python）/ `jsonrepair`（Node）处理尾逗号、单引号、注释和全角标点，不要手写正则改字符串内容。
- **L6 schema 校验**：zod / pydantic 过字段和类型，这步通过才算真成功。

```python
def extract_json(text: str):
    t = text.replace("\ufeff", "").strip()       # L0
    t = strip_think(t)                           # L1
    try:
        return json.loads(t)                     # L2
    except ValueError:
        pass
    cands = fenced_blocks(t) + balanced_scan(t)  # L3 + L4
    for c in reversed(cands):                    # 后面的候选优先
        try:
            return json.loads(c)
        except ValueError:
            continue
    return json_repair.loads(t)                  # L5
```

返回后再过 schema，两步都过才交给工具执行。

## 踩坑点

- **贪婪正则合并代码块**：`[\s\S]*` 会把两个围栏和中间的说明文字当成一块，用非贪婪并逐块尝试。
- **括号计数不看字符串**：`{"path": "a}b"}` 会让朴素计数器提前截断，扫描时必须维护"是否在字符串内"状态并处理 `\"` 转义。
- **全角标点全局替换**：在字符串值里把 `，` 换成 `,` 会污染中文文案字段，转换放到修复层且用现成库。
- **只取第一个 JSON**：`<think>` 里常带示例 JSON，先剥壳再取最后候选，命中率高得多。
- **截断静默通过**：括号不平衡且长度顶满基本是截断，auto-close 完别当成功，记日志并触发重试。
- **不分"没找到"和"找到但坏了"**：前者改提示词，后者重试或修复，混在一起排查很痛苦。

## 可复用建议

- **收敛成公共模块**，放在插件/MCP server 的工具结果入口，所有 subagent 共用，别每个工具各写一份。
- **提供 strict / lenient 两档**：自动化流水线用 strict 快速失败，对话场景用 lenient 宽容修复。
- **能用 provider 的结构化输出（function calling + schema 强约束）就优先用**，标签解析只做兜底层。
- **建 fixtures 目录**，把线上真实坏样本存成单测用例；模型换代后跑一遍，回归一目了然。
- **埋点记录哪一层命中**：修复层命中率突然上升，通常说明上游模型行为变了。

## 总结

LLM 的结构化输出永远处于"大概率合规、小概率花式出错"的状态。防御性解析的意义不是容忍烂输出，而是把不确定性约束在解析层，让下游工具拿到的永远是校验过的数据。分层降级 + schema 校验 + 坏样本回归，三件事做齐，插件工具调用的稳定性会有立竿见影的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/976f866708294194.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/25a8038a41d1cd50.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/9f7504203cc9891d.png)

