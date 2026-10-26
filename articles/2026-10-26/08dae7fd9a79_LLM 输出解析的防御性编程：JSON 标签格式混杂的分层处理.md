---
title: LLM 输出解析的防御性编程：JSON 标签格式混杂的分层处理
feedId: 42643
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，LLM 的输出经常不是"纯数据"，而是要被下游脚本直接消费的结构化结果。我们的惯例是在 prompt 里要求模型"只输出 JSON"，但实测中模型输出经常带 markdown 围栏、前后缀说明文字，甚至几种格式混着来。解析层不做防御，自动化流水线就会间歇性失败，而且失败点很随机，排查成本高。

## 问题

把线上抓到的失败样本归类，大致五种：

1. **围栏混杂**：```json 围栏、裸 ``` 围栏、无围栏纯 JSON 三种交替出现；
2. **围栏外有话**：先来一句"以下是结果"，再给 JSON，末尾再补解释；
3. **多段输出**：一个回复里出现两个 JSON 块，一段中间推理、一段最终结果；
4. **轻微语法损伤**：尾逗号、单引号、中文引号、伪注释；
5. **流式场景**：读到一半拿到不完整 JSON。

麻烦在于它们会随机组合，单点解法不够用。

## 做法

核心思路：**不要试图用 prompt 把格式管死，而是在解析层做分层兜底**。

1. **预处理**：去 BOM、中文引号映射回 ASCII、trim；
2. **剥围栏**：正则匹配 ```json / ``` 围栏提取内容；注意非贪婪，围栏本身可能不成对；
3. **括号配平提取**：对带杂文的文本，从第一个 `{` 或 `[` 起做深度扫描（跳过字符串字面量内部的括号），取配平的完整片段；多个候选时按任务约定取最长或最后一个；
4. **修复与校验**：先走严格 `json.loads`，失败才走修复路径（去尾逗号、单引号转义），修复后必须重新校验，再过 pydantic/jsonschema 验业务字段；
5. **带反馈重试**：校验失败时把解析错误连同原始输出回传模型，最多重试 2 次，仍失败则落盘原始输出供人工排查。

核心伪代码：

```python
def extract_json(text: str):
    text = normalize(text)             # 归一化
    blocks = strip_fences(text)        # 剥围栏，可能为空
    for c in (blocks or [text]):
        for frag in balanced_scan(c):  # 括号配平扫描
            for j in (frag, repair(frag)):
                try:
                    return json.loads(j)
                except JSONDecodeError:
                    continue
    raise ParseError(text)
```

## 踩坑点

- **贪婪正则会吞掉两个 JSON 块之间的杂文**，务必非贪婪，或干脆用括号扫描；
- 括号扫描若不跳过字符串内部的 `{}`，遇到值里含 `"}"` 会提前截断；
- 别迷信 ```json 标签一定在，同一会话内格式就会漂移；
- 修复逻辑只能是最后手段，**修复率升高往往说明 prompt 契约在退化**，要报警而不是庆幸兜底成功；
- 流式场景别套用完整解析，要么攒齐再解，要么换增量解析器。

## 可复用建议

- 把 `extract_json` 收进插件公共库，全项目只维护一份，别各写各的；
- prompt 侧照样写清契约（给一个纯 JSON 的 few-shot 示例），解析侧兜底，**两头都做，缺一不可**；
- 记录每条输出的解析路径（fenced / plain / repaired / retry），按天统计占比，能提前发现模型行为变化。

## 总结

LLM 输出解析的稳定性，不靠把 prompt 写得更凶，而靠解析层的分层防御：归一化 → 剥围栏 → 配平提取 → 修复校验 → 带反馈重试。每一层都很简单，组合起来就能把间歇性失败压到很低。这也是写 Agent / 插件时的一个通用心法：**对外部不可控输入，永远假设格式会漂移**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/7705dc15f81c3cc6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/0fbc317c63c38150.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/19c4ecbfa9bdbe3a.png)

