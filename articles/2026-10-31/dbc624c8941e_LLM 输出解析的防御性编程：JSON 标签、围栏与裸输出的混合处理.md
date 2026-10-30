---
title: LLM 输出解析的防御性编程：JSON 标签、围栏与裸输出的混合处理
feedId: 43070
source: 综合讨论
publishedAt: 2026-10-31
---

## 背景

在 OpenClaw 的 Agent、插件和 MCP 自动化流水线里，让模型返回结构化 JSON 几乎是刚需：动作参数、路由决策、结果汇总、节点间传递，都依赖这一步。常见做法是在提示词里约定一个标签，比如 `<result>{...}</result>`，再用正则提取。上线头几天很稳，跑久了就开始偶发解析失败。

## 问题

模型输出会漂移。同一套提示词，换模型、调温度、甚至只是换了个会话，输出格式至少出现五种变体：

1. 严格按标签包裹；
2. 标签前后多一句"好的，结果如下"；
3. 不用标签，直接上 ```json 围栏；
4. 围栏写成大写 ```JSON，或干脆省略语言标注；
5. 标签里再嵌一层围栏。

而解析代码通常只处理第一种。结果就是最讨厌的故障形态：偶发、难复现，日志里的 raw output 看起来"明明是 JSON"。

## 做法：分层降级解析

思路是把"提取"和"解析"拆开，各自从严格到宽松做多层降级，命中即返回：

```python
import json, re

def extract_json(text: str):
    cands = [text.strip()]                          # 1) 直接整体解析
    m = re.search(r"<(?:result|json|output)>(.*?)</(?:result|json|output)>",
                  text, re.I | re.S)                # 2) 标签提取，宽容标签名
    if m:
        cands.append(m.group(1).strip())
    for m in re.finditer(r"```(?:json)?\s*(.*?)```",
                         text, re.I | re.S):        # 3) 代码围栏提取
        cands.append(m.group(1).strip())
    cands.append(scan_balanced_braces(text))        # 4) 括号配对兜底
    for c in cands:
        try:
            return json.loads(c), True
        except ValueError:
            continue
    return None, False

def scan_balanced_braces(text: str) -> str:
    depth, start, in_str, esc = 0, -1, False, False
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch == "{":
            if depth == 0: start = i
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0: return text[start:i + 1]
    return ""
```

`scan_balanced_braces` 是个小状态机：遍历字符，维护 in_string、escape、depth 三个状态，跳过字符串字面量里的花括号，返回第一对配平的完整 `{...}`。十几行代码，能救回大量"裸 JSON 前后带说明文字"的输出。解析成功后再过一层 schema 校验——解析成功不等于内容正确，这两件事要分开验证。

## 踩坑点

- 正则漏了 `re.DOTALL` 或 `re.IGNORECASE`：跨行 JSON 和大写围栏直接漏掉，这是我见得最多的两类低级失误。
- 用贪婪匹配 `.*` 遇到输出里有两段围栏时会吃进多余内容，务必非贪婪并优先取首段。
- 花括号计数不处理字符串内的 `{}`，字段值里一出现花括号就断错位置，必须上状态机。
- 提取顺序有讲究：先标签后围栏，否则嵌套场景会把内层围栏先剥出来。
- 偶尔出现全角引号、尾逗号，可以上 json5 / json-repair 做最后修复，但修复触发时要打日志，别静默吞掉。

## 可复用建议

- 把 `extract_json` 收成公共工具，返回值带上"命中了哪一层"，按天统计各层命中率。某层占比突然变化，往往意味着模型或提示词在漂移，可以直接当告警信号用。
- 把线上解析失败的原始输出脱敏后存进脏样本库，做成单元测试回归集，比凭空编造边界用例有效得多。
- 解析兜底是保险丝，不是借口：提示词里仍要明确"只输出 JSON，不要围栏、不要解释"；模型支持结构化输出或 JSON Schema 约束时优先用约束，而不是靠解析层硬扛。
- 流式场景别逐 chunk 解析，等结束信号后走统一入口，避免半截 JSON 反复抛错刷屏。

## 总结

LLM 输出的不确定性没法靠提示词根除。防御性解析的目标不是把脏输出洗干净，而是把不确定性收敛到可观测、可回归的代码路径里。四层降级加命中率统计加脏样本回归，这套组合拳成本不到五十行代码，却能把"偶发 JSON 解析失败"从玄学问题变成有数据、有测试的工程问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/82366b8989986e3c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/bcf698df6e841f57.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/e086039f72e03a26.png)

