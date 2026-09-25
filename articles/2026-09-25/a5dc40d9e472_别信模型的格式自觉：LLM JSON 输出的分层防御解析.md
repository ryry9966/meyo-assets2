---
title: 别信模型的格式自觉：LLM JSON 输出的分层防御解析
feedId: 38971
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，一个很常见的模式是：让模型把结构化结果包在 `<json></json>` 标签里输出，插件解析后交给下游动作。Demo 里一次就通，上了自动化流水线才发现——同样的 prompt，模型十次里有三种写法。

## 问题：格式漂移的真实形态

跑了一段时间后，我从日志里捞出的失败样本大致五类：

1. 该用标签时输出了 ```json 围栏，甚至围栏+标签双层包裹；
2. 标签出现多次（模型把格式示例又复述了一遍），用 `split` 取第一段拿到的是示例本身；
3. 标签前后混着解释性文字，还有"以下是 JSON："这类引语；
4. 输出在 max_tokens 处截断，只剩半个对象；
5. JSON 看着像那么回事，但带尾逗号、单引号、注释。

最朴素的 `text.split("<json>")[1].split("</json>")[0]` 在这五种面前全部阵亡，而且故障是间歇性的——这比稳定报错难受得多。

## 做法：四层解析链

我把解析器改成了优先级链，逐层降级：

- **第一层：标签匹配。** 非贪婪正则 + 忽略大小写，兼容标签内外的空白，取最后一个匹配（最终答案通常在后面）。
- **第二层：围栏匹配。** 处理 ```json 围栏，兼容无语言标识的情况。
- **第三层：括号平衡扫描。** 从第一个 `{` 或 `[` 开始扫描，维护"是否在字符串内、是否转义"两个状态位，找到配对的闭合括号。这一层能兜住"前后有废话但 JSON 完整"的大多数情况。
- **第四层：校验与修复。** parse 成功不代表能用，先过 pydantic 校验 schema；失败就把具体报错连同原文喂回模型重写，只重试一次，再失败落库人工看。

核心代码不到四十行：

```python
import re

TAG   = re.compile(r"<\s*json\s*>\s*(.*?)\s*<\s*/\s*json\s*>", re.S | re.I)
FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)

def balanced_scan(text: str):
    start = next((i for i, c in enumerate(text) if c in "{["), None)
    if start is None:
        return None
    depth, in_str, esc = 0, False, False
    for j in range(start, len(text)):
        c = text[j]
        if in_str:
            if esc:
                esc = False
            elif c == "\\":
                esc = True
            elif c == '"':
                in_str = False
        else:
            if c == '"':
                in_str = True
            elif c in "{[":
                depth += 1
            elif c in "}]":
                depth -= 1
                if depth == 0:
                    return text[start:j + 1]
    return None  # 没等到闭合，多半是截断

def extract(raw: str):
    hits = TAG.findall(raw)
    if hits:
        return hits[-1].strip(), "tag"
    m = FENCE.search(raw)
    if m:
        return m.group(1).strip(), "fence"
    body = balanced_scan(raw)
    return (body, "scan") if body else (None, "none")
```

每次解析记录命中的层名。上线两周后，scan 层占比从 18% 降到 3%——因为我顺手收紧了 prompt。指标本身会告诉你该修解析器还是该修提示词。

## 踩坑点

- **贪婪正则会跨块吞内容。** `<json>(.*)</json>` 遇到两段标签时会把中间的散文一起捞进来，必须非贪婪，并想清楚取第一个还是最后一个。
- **括号计数不看字符串状态必炸。** `"note": "a}b"` 这种值里的花括号会让朴素计数提前归零，字符串/转义状态位是整个扫描器的精髓。
- **流式输出别边收边解析。** 标签和括号可能被 chunk 切开，先缓冲到闭合标记再解析，否则日志里会出现一堆"合法但残缺"的对象。
- **重试要设上限。** "解析失败→重新生成"无限循环会烧钱，我上限一次，且重试时把具体校验错误喂回去，而不是笼统说"格式错了"。
- **降级层静默成功会掩盖退化。** scan 层兜住了不代表没问题，它的占比上升要当告警看。

## 可复用建议

- 解析器独立成模块，把线上坏样本攒成回归测试集，每次改 prompt 或换模型跑一遍。
- 模型支持结构化输出或工具调用时优先用，标签解析只做兼容层。
- prompt 侧同步收紧：给一个精确的 few-shot 样例、降低 temperature、明确"标签之外不要输出任何字符"。解析健壮性和 prompt 约束是互补的，不是二选一。
- 失败时保留原始输出全文（可截断存储），离线排障全靠它。

## 总结

防御性解析的目标不是让模型更听话，而是让失败可观测、可恢复：四层链保证大多数脏输出仍能拿到合法数据，指标告诉你哪层在扛流量，重试上限保证成本可控。当降级层的占比持续下降，说明你的 prompt 和解析器终于进入了正循环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/40be3417d50d3e00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/663b2e1ac328c018.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/650255a757d83925.png)

