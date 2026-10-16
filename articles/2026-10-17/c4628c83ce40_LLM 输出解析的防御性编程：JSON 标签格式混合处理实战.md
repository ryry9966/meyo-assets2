---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 41749
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

在 OpenClaw 的插件和自动化流程里，让模型输出 JSON 是常见需求：MCP 工具参数、子任务结果回传、规则引擎的条件块。如果走带严格结构化输出的接口，问题不大；但实际跑起来经常遇到两类情况：模型不支持 JSON mode，或者输出经过多轮模板拼接后格式就"脏"了。同一段 prompt，十次运行能拿到至少四种格式：纯 JSON、```json 围栏、前后带说明文字、甚至被包在模型自造的 `<json>` 标签里。

## 问题

常见的初版写法是 `str.replace` 去掉围栏再 `json.loads`，跑两天就会在线上翻车。混合格式的坑主要有：

- 围栏语言标记不统一（`json` / `JSON` / 不写）；
- 回复里同时混着示例 JSON 和真正的结果 JSON；
- 中文语境带出全角引号、全角逗号；
- 字符串值内部含大括号，朴素的括号计数会错位截断；
- 尾逗号、`//` 注释这类"近似 JSON"。

## 做法：分层提取 + 分级降级

核心思路：不假设格式，按"代价从低到高"逐层尝试，每层命中情况都记录指标。

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*(.*?)\s*```", re.S)
TAG   = re.compile(r"<(?:json|result)>\s*(.*?)\s*</(?:json|result)>", re.S)

def _loads(frag):
    try:
        return json.loads(frag)
    except json.JSONDecodeError:
        # 宽松修复只做一件事：去尾逗号，保持克制
        return json.loads(re.sub(r",\s*([}\]])", r"\1", frag))

def extract_balanced(s):
    """字符串感知的括号扫描，返回第一个平衡的 {...} 或 [...]"""
    for opener, closer in (("{", "}"), ("[", "]")):
        start = s.find(opener)
        if start < 0:
            continue
        depth, in_str, esc = 0, False, False
        for i, ch in enumerate(s[start:], start):
            if in_str:
                if esc: esc = False
                elif ch == "\\": esc = True
                elif ch == '"': in_str = False
            elif ch == '"':
                in_str = True
            elif ch == opener:
                depth += 1
            elif ch == closer:
                depth -= 1
                if depth == 0:
                    return s[start:i + 1]
    return None

def parse_llm_json(text):
    text = text.strip()
    try:
        return json.loads(text)                    # L1 直解
    except json.JSONDecodeError:
        pass
    for b in reversed(FENCE.findall(text)):        # L2 围栏，取最后一块
        try: return _loads(b)
        except json.JSONDecodeError: continue
    for b in TAG.findall(text):                    # L3 自造标签
        try: return _loads(b)
        except json.JSONDecodeError: continue
    frag = extract_balanced(text)                  # L4 括号扫描
    if frag:
        return _loads(frag)
    raise ValueError("no parsable json in output")
```

解析成功后不要直接入库：先过一遍 schema 校验（pydantic 或等价物），解析成功但形状不对同样视为失败，并把原始输出落盘供复盘。

## 踩坑点

1. **贪婪正则**：`\`\`\`.*\`\`\`` 会把两个围栏之间的文字一并吞掉，务必非贪婪并 `findall`。
2. **"取第一个 JSON"常错**：很多模型先给示例再给答案，先试最后一个围栏命中率更高；但反例也存在，稳妥做法是逐块解析、取第一个通过 schema 校验的。
3. **括号计数必须跳过字符串字面量**，否则值里出现 `"}"` 就会提前截断。
4. **全角标点**建议在 prompt 侧明确要求半角输出，修复层只兜底、不依赖。
5. **静默修复会掩盖上游劣化**：给每层降级加计数上报。如果 L1 直解命中率从 90% 掉到 40%，那是模型或 prompt 模板变了，不是解析器的问题。

## 可复用建议

- 解析逻辑收口成一个工具模块，全项目唯一入口，不要各插件各写一份。
- 层级顺序固定：直解 → 围栏 → 标签 → 括号扫描 → 宽松修复，逐层带指标。
- prompt 侧配合：few-shot 给一个完整输出样例、明确"不要输出多余文字"、temperature 调低；能用结构化输出或工具调用就用，解析层只是兜底。
- 宽松修复保持克制：尾逗号、注释可以修；缺字段、类型不对不要猜，直接失败。

## 总结

JSON 解析的防御性编程，不是一个正则的胜利，而是"假设格式会脏、按代价分层处理、每层可观测、终有校验兜底"。把解析做成独立的、带指标的一层，你的自动化流程才敢在没人盯着的时候跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/84d63582c43743eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/61eab35c59e7a96b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b9a6ed8b46cc0d81.png)

