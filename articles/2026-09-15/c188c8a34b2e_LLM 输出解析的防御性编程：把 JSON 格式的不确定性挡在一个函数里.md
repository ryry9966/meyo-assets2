---
title: LLM 输出解析的防御性编程：把 JSON 格式的不确定性挡在一个函数里
feedId: 37594
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

在 OpenClaw 的自动化链路里，LLM 的输出经常不是给人看的，而是直接喂给下游的 MCP 工具、插件或脚本。只要中间环节有一次 `json.loads()`，格式问题就会从"显示难看"升级成"整条流水线中断"。

我在维护一个定时抓取 + 结构化入库的 agent 任务时被这个问题反复折磨，最后把解析层重写了一遍才稳定下来。这篇帖子把做法和坑整理出来。

## 问题

提示词里写了"只输出 JSON，不要任何解释"，实际拿到的输出至少有五种形态：

1. 裸 JSON，理想情况；
2. 包在 ` ```json ` 围栏里；
3. 包在 `<json>` / `<answer>` 之类的自定义标签里；
4. 前面带一句"以下是结果："，后面再补一段说明；
5. JSON 本身有尾逗号、中文引号、注释。

前四种靠几个正则还能覆盖，第五种才是真正的长尾。单靠一个 `try/except` 兜不住，因为失败模式太多。

## 做法

我的思路是分层：提示词层减量，提取层兜底，校验层把关，重试层收尾。

**1. 提示词层。** 能用 function calling / JSON mode / schema 约束就用，这是唯一能让格式接近确定的手段。用不了时，给一段完整的输出示例，比一句"请输出 JSON"有效得多。

**2. 提取层。** 不假设格式，按"最严格 → 最宽松"依次尝试：

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG   = re.compile(r"<(json|answer|result)>(.*?)</\1>", re.S | re.I)

def extract_json(text: str):
    text = text.strip().lstrip("\ufeff")
    cands = [text]
    cands += [m.group(1) for m in FENCE.finditer(text)]
    cands += [m.group(2) for m in TAG.finditer(text)]
    start = text.find("{")            # 配对扫描，应对前后夹说明文字
    if start != -1:
        depth = 0
        for i, ch in enumerate(text[start:], start):
            if ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    cands.append(text[start:i + 1]); break
    for c in cands:
        try:
            return json.loads(repair(c))
        except json.JSONDecodeError:
            continue
    raise JsonParseError(f"解析失败，原文片段: {text[:200]}")

def repair(s: str) -> str:
    s = re.sub(r",\s*([}\]])", r"\1", s)   # 尾逗号
    return s.replace("\u201c", '"').replace("\u201d", '"')  # 中文引号
```

关键点：花括号用配对扫描而不是贪婪正则 `{.*}`，避免输出说明里夹了示例 JSON 时抓错边界。

**3. 校验层。** 解析成功不等于数据可用。用 pydantic 或 jsonschema 再过一道，缺字段、类型不对在这里拦下，别让脏数据进库。

**4. 重试层。** 校验失败时，把报错信息和原始输出回喂给模型，让它按 schema 修正，上限 1 次。实测大多数格式错误一次就能修好；超过一次基本是 prompt 本身有问题，该改 prompt 而不是继续烧重试。

**5. 兜底。** 失败时抛出带原文前 200 字符的异常，原始输出完整落日志。排查问题时这两样缺一不可。

## 踩坑点

- **贪婪正则抓多**：`{.*}` 会把解释文字里的示例 JSON 一起吞掉，务必配对扫描。
- **围栏套围栏**：字符串值里出现 ` ``` ` 会让 `finditer` 截断结果，长内容输出时偶发，保留裸扫描那条候选就是兜底。
- **temperature=0 ≠ 格式确定**：只是降低波动，长输出照样可能变形。
- **流式输出别边收边 parse**：半截 JSON 永远解析失败，等终止信号再解析，或用支持增量解析的库。
- **重试无上限**：定时任务 × 每次重试的成本，月底账单很难看。
- **隐形字符**：BOM、零宽空格、全角引号，都会让"看起来一模一样"的输出解析失败。

## 可复用建议

1. 解析逻辑收敛到一个工具模块，全项目只此一份，禁止各插件各写各的 `try/except`。
2. 把真实跑出来的"坏输出"存成语料库，写进单元测试——它比任何想象出来的用例都有价值。
3. 异常信息永远带上原文片段，别只抛一个光秃秃的 `JSONDecodeError`。
4. 有结构化输出能力就优先用，手写提取层是给"没有更好选择"的场景准备的。

## 总结

LLM 输出解析没有银弹，但分层防御能把不确定性压缩到很小的范围：提示词减量、提取兜底、校验把关、重试收尾、日志留痕。整条自动化链路的稳定性上限，往往就取决于这几十行解析代码写得扎不扎实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/5716ea0a4ec1950d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/d51befee0a484204.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/2689e730d5b11c2e.png)

