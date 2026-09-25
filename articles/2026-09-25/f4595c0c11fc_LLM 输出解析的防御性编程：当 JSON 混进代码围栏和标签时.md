---
title: LLM 输出解析的防御性编程：当 JSON 混进代码围栏和标签时
feedId: 38904
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，LLM 输出经常被当作结构化数据消费：Agent 决策要 JSON，工具调用参数要 JSON，流水线节点之间传递的还是 JSON。但实践下来会发现一件事——“让模型只输出 JSON”是一个愿望，不是一个保证。prompt 写得再严格，也不该被当作解析层的前提条件。

## 问题

同一份 prompt 在不同模型、不同温度、甚至不同会话里，输出格式至少有五种形态：

1. 纯 JSON（理想情况，占比未必最高）
2. ` ```json ` 代码围栏包裹
3. 自然语言开头 + JSON 结尾（“好的，以下是结果：{...}”）
4. XML 式标签包裹：`<result>{...}</result>`
5. 混合体：围栏里套围栏（JSON 字段值本身含代码块），或输出被 max_tokens 截断

直接 `json.loads` 的失败率在我们内部统计里长期在 10%~20% 徘徊。每一次失败，意味着一次任务中断，或一次无意义的重试。

## 做法

核心思路：别指望 prompt 解决问题，把解析做成一条分层流水线，每层只处理自己擅长的情况。

```python
def parse_llm_json(raw: str) -> dict:
    # L0 快路径：能直接解析就不折腾
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        pass

    # L1 剥离围栏和已知标签
    text = strip_code_fences(raw)   # 处理 ```json / ```，含嵌套
    text = strip_xml_tags(text)     # <result>、<output> 等

    # L2 括号配对提取：跳过字符串内的括号，逐字符扫描
    candidate = extract_balanced_json(text)

    # L3 轻度修复：尾逗号、中文引号、未转义换行
    candidate = repair_common_issues(candidate)

    # L4 校验：解析成功 ≠ 结构正确
    return Schema(**json.loads(candidate))  # pydantic，失败即抛出
```

配套三条规则：

- 围栏剥离用“最外层优先”：从 ` ```json ` 开头匹配到下一个独立 ` ``` `，防止字段值里的代码块被误剥
- 括号配对必须维护“是否在字符串内”的状态机，不要用贪婪正则 `\{.*\}`
- 修复层只做可枚举、可解释的替换，每次修复都记日志

## 踩坑点

1. **贪婪正则提取**：输出里出现两个对象时，会把两者粘成一个，解析出“看似合法但语义错误”的数据——这比解析失败更危险
2. **括号计数不管字符串字面量**：遇到 `"note": "use } carefully"` 直接断在半路
3. **盲信修复库**：json-repair 类工具会把截断的数字“修”成另一个值，静默产出错误数据；修复后必须过 schema 校验
4. **模型升级后格式漂移**：老特判反而帮倒忙，特判要有开关和度量
5. **截断输出不要硬修**：检查括号平衡后直接触发重试或降级，省下的调试时间远多于省下的 token

## 可复用建议

- 解析逻辑收拢到一个模块，全项目只此一份，禁止业务代码里散落正则
- 埋两个指标：解析成功率、修复触发率，按 prompt 模板分组统计。修复率突然升高，通常说明上游 prompt 被改坏了
- 收集失败样本做成测试语料，解析器每次改动都跑回归
- 优先用 provider 原生 structured output / function calling；防御性解析是保险带，不是方向盘

## 总结

LLM 输出解析的本质，是和概率性系统做工程接口。分层、可观测、可回归测试的解析流水线，比任何一段精巧的正则都可靠。当你的 Agent 凌晨三点挂掉时，你希望日志里是一段被完整保存的原始输出，而不是一个 `None`。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/7f448582886e2842.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/56cedc335b514135.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ddc1248eb8a6757d.png)

