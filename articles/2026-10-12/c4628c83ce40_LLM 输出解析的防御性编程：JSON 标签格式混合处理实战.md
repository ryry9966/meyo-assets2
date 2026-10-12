---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 41296
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

在 OpenClaw 插件和 MCP 工具链开发里，让模型按固定格式输出是常态：用 `<tool_call>...</tool_call>` 包 JSON、用 ` ```json ` 围栏包结构化结果，或者在系统提示里约定"只在标签内输出"。约定很美好，现实很骨感——换模型、换量化版本、调高 temperature，甚至只改提示词一行字，输出格式就会漂移。

## 问题

我们在线上遇到的混合格式大致有几类：

- 标签内又套了一层 ` ```json ` 围栏；
- 标签前后夹着"好的，以下是结果："之类的客套话；
- 尾随逗号、单引号、中文引号""、`True/None` 这类非标准 JSON；
- 少写右标签，或者把 few-shot 示例里的标签原样复述了一遍；
- 流式输出中途截断，拿到半截 JSON。

只按一种格式写解析器，失败率会随模型切换明显波动，而且失败往往是静默的——插件拿到 `None`，上游毫无感知。

## 做法

我们的思路是**分层解析**，每层只做一件事，失败就进下一层：

1. **提取候选区**：用非贪婪正则拿标签内容，容忍大小写和空白；找不到闭合标签时退化到"从开标签截到末尾"；多次命中时取最后一次完整配对（避免命中复述的示例）。
2. **剥围栏**：去掉 ` ```json ` / ` ``` `，嵌套围栏取最后一组闭合。
3. **直接解析**：`json.loads` 成功就走 schema 校验。
4. **受控修复**：只做确定性修复——去尾逗号、全角引号转半角、`True/False/None` 映射，修完再试一次。
5. **括号配平扫描**：带字符串感知的栈扫描，取第一段配平的 `{...}`，处理"前有废话、后有废话"的情况。
6. **校验与兜底**：pydantic 校验字段类型；全失败则记录原始输出，按错误类型决定带错误信息重试还是告警。

骨架大概十几行：

```python
def parse_llm_json(raw: str):
    for cand in extract_candidates(raw):   # 标签区 → 围栏区 → 全文
        for text in strip_fences(cand):
            try:
                return validate(json.loads(text))
            except json.JSONDecodeError:
                fixed = deterministic_repair(text)
                try:
                    return validate(json.loads(fixed))
                except json.JSONDecodeError:
                    continue
    log_and_raise(raw)  # 失败必须可见
```

## 踩坑点

- **贪婪正则会跨标签吞内容**，务必用非贪婪匹配并尽量加锚点；
- `json.loads` 默认放行 `NaN/Infinity`，下游序列化会炸，要显式拦截；
- **修复要克制**：确定性规则之外的异常（比如模型输出了两个 JSON 对象）应该失败重试，而不是猜一个"看起来对的"；
- 流式场景别边收边 parse，等闭合标签确认或流结束后再走管线，半截 JSON 交给修复层没有好结果；
- 日志里必须保留原始输出，否则永远不知道模型到底输出了什么，语料也沉淀不下来。

## 可复用建议

- 解析器收敛到一个公共模块，不要每个插件各写一份、各烂一份；
- 从日志沉淀"脏输出语料"，做成回归测试集，每次换模型或改提示词先跑一遍；
- 运行时支持 structured output / json_schema 约束解码就优先用，标签 + 解析只做兜底，而不是唯一防线；
- 按模型、按提示词版本记录解析失败率——格式漂移应该是一个能被观测到的指标，而不是用户报障才知道的事故。

## 总结

防御性解析的本质，是承认"模型一定会违反格式约定"，然后把违规变成可观测、可恢复的事件：提取、剥壳、解析、修复、校验、重试，六层各司其职。每一层都要能独立失败，而失败本身要留下证据。这套管线不增加多少代码量，但能把"偶尔抽风"变成"可控降级"，在多模型混跑的 Agent 环境里尤其值回票价。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/57cbacd28d7213af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/f3f74a9efad1d259.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/909de9f070fc503d.png)

