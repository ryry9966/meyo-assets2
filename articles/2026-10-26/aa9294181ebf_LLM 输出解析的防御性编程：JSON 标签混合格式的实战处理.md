---
title: LLM 输出解析的防御性编程：JSON 标签混合格式的实战处理
feedId: 42678
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

在 OpenClaw 插件和 MCP 工具链开发里，只要涉及“让模型输出结构化数据”，就绕不开解析环节。我们通常会在 prompt 里写死“只输出 JSON，用 ```json 包裹”，但实际跑起来会发现：换模型、调温度、上下文变长之后，模型并不总是守约。

## 问题

同一份 prompt 在一周内拿到的输出至少有这几种形态：

- 裸 JSON，没有任何包裹
- ```json 围栏，语言标注大小写不一，甚至写成 ```JSON5
- `<json>...</json>` 或 `<result>...</result>` 标签包裹
- JSON 前后带解释性文字（“好的，以下是结果：”）
- 全角标点：中文逗号、冒号、引号
- 尾逗号、`//` 注释、单引号

只按单一格式写的解析器，上线后迟早翻车，而且往往是静默失败——抛异常、任务链中断，日志里只剩一句 JSONDecodeError。

## 做法：一条分层提取流水线

核心思路：把模型输出当**不可信文本**，按成本从低到高依次尝试多种提取策略，命中即返回，全部失败再进修复环节。

```python
def extract_json(text: str):
    candidates = [text.strip()]
    m = re.search(r"```(?:json[5c]?|JSON)?\s*(.*?)```", text, re.S)
    if m:
        candidates.append(m.group(1))
    for tag in ("json", "result", "output", "answer"):
        m = re.search(rf"<{tag}>(.*?)</{tag}>", text, re.S | re.I)
        if m:
            candidates.append(m.group(1))
    candidates.extend(scan_balanced(text))  # 字符级括号配对扫描
    for c in candidates:
        try:
            return json.loads(normalize(c))
        except json.JSONDecodeError:
            continue
    return None
```

几个要点：

1. **括号配对不能靠正则**。正则匹配不了任意深度的嵌套。写一个字符级扫描器，跟踪 `in_string` 和转义状态，`{` `[` 深度加一、`}` `]` 减一，深度归零时截取。这是整条流水线里最稳的兜底。
2. **normalize 放在解析前而不是失败后**。统一做四件事：全角标点转半角、智能引号转直引号、去尾逗号、剥 `//` 注释。注意去注释必须在 string 状态外进行，否则会把字符串内容里的 `"//"` 误删。
3. **解析成功 ≠ 数据正确**。`json.loads` 通过后还要过 schema 校验（Pydantic / Zod 均可），字段缺失、类型漂移在这一层拦住。
4. **修复环节要设上限**。schema 校验失败后，可把原始输出加报错信息发回模型做一次性修复，重试不超过 2 次；仍失败就降级到错误态，并把原始输出落盘，禁止无限重试。

## 踩坑点

- `re.search(r"\{.*\}")` 贪婪匹配会吞掉多个对象之间的文字；非贪婪 `.*?` 遇到嵌套又会在第一个内部 `}` 提前截断。正则只适合剥围栏和标签，别拿它配对括号。
- `json.loads` 默认接受 `NaN` / `Infinity`，这不是合法 JSON，下游序列化会再炸一次。建议通过 `parse_constant` 直接抛错。
- 全角问题不只标点：中文输入态可能混入全角空格（U+3000）和零宽字符，肉眼看不出来，normalize 里一并处理。
- 模型偶尔输出多个粘连的 JSON 对象。取第一个还是全部，取决于业务语义，建议显式决策并记日志，别让默认行为埋雷。
- 修复重试可能“修出语义”——模型顺手改了字段值。修复后的对象必须重新过 schema，最好与失败版本做 diff 记录。

## 可复用建议

- 解析逻辑收敛到一个模块，全项目只此一处，不要各处各写一套正则。
- 每次解析记录命中的策略名（direct / fence / tag / brace-scan / repair）。积累一段时间后你会发现某类模型固定偏好好某几种格式，反过来就能收紧 prompt。
- 把线上失败样本脱敏后存成测试语料，每次改提取逻辑或改 prompt 都跑一遍回归。这套语料比任何文档都真实。
- 运行时如果支持原生结构化输出或 tool-call JSON mode，优先用；但兜底层照留——换供应商、降级模型那天你会感谢它。

## 总结

LLM 的格式输出是概率性的，解析代码必须是确定性的。防御性解析的本质不是写更聪明的正则，而是承认格式会失控，用一条多策略、可观测、有上限的流水线把不确定性收敛掉。整套实现一百多行就够了，建议每个做 Agent 插件的开发者都在工具入口处垫上这一层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/a0410f625af50b5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/0d6554edf434444d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/11c471424fd0b829.png)

