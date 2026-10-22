---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 42328
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

只要你的 OpenClaw skill 或 MCP 工具链里有一环是"让模型返回 JSON"，就会遇到同一个事实：同一个 prompt，换个模型、换个温度，返回的包装格式就不一样。我们在一条自动化流水线里先后接了通用对话模型和推理模型，解析层第一周崩了三次，最后把解析重写成一条防御性流水线才稳定下来。

## 问题：四种"合法"的脏输出

同样是"请输出 JSON"，实际拿到手的至少有四种形态：

1. 围栏包裹：```json … ```，偶尔是 ```、```JSON、甚至 ```jsonc；
2. 裸 JSON 前后带话："好的，以下是结果：{…} 如有疑问请告诉我"；
3. 推理模型先吐一段 `<think>…</think>`，再接围栏或裸 JSON；
4. 半残 JSON：尾部多逗号、中文引号，或被 max_tokens 截断。

任何一层直接 `json.loads`，本质都是把概率性输出当确定性输入用。

## 做法：解析流水线，五步走

```python
def extract_json(text: str) -> str:
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.S)  # 1. 预处理
    text = text.strip().lstrip("\ufeff")
    m = re.search(r"```(?:json|jsonc)?\s*(.*?)\s*```", text, re.S | re.I)
    if m:
        text = m.group(1)                                       # 2. 剥围栏
    return scan_json_block(text)  # 3. 字符串感知的括号配对，取最外层块
```

之后两步：

4. **修复**：trailing comma、中文引号可以安全替换；但截断的 JSON 不要硬补，先看响应里的 finish_reason，截断就走重试或报错；
5. **校验**：过一遍 pydantic / zod 的 schema，字段缺失、类型不对直接判失败；失败时把原始输出带回 prompt 重试一次，仍失败进死信队列。

## 踩坑点

- 贪婪正则 `r"\{.*\}"` 遇到嵌套对象会从第一个 `{` 一路匹配到最后一个 `}`，中间夹着 prose 也照单全收；
- 括号扫描如果不处理字符串态，JSON 字符串值里的 `"{"` 会把深度计数搞崩；
- 流式场景把 chunk 直接拼接，可能得到两个粘连的 JSON 对象，loads 报"额外数据"；
- 修复类库对截断输出会"脑补"出一个合法 JSON，静默产出脏数据——比抛异常危险得多；
- 日志全量打印原始输出，既刷屏又有敏感信息风险，建议截断到几百字符。

## 可复用建议

- 解析逻辑收敛到一个模块，所有 skill 复用，别在各自 handler 里散落 strip 和正则；
- 模型侧优先用 structured output / tool call，但外层兜底流水线照留——供应商降级、上下文污染时它救过我们；
- 每次解析失败把原始样本落盘，一个月后就是现成的回归测试集；
- prompt 和 parser 当作一个版本单元管理，改输出格式约定时两边同步。

## 总结

LLM 输出解析的本质，是在概率系统和确定性系统之间加一层收敛。格式混排不是异常，是常态。把"剥标签 → 定位 → 修复 → 校验 → 重试"做成固定流水线，Agent 链路才能从"偶尔能跑"变成"长期能跑"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/2b49b7080a1d271d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/a2c12f585bfe0aa8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/668bdb0d1f16b81f.png)

