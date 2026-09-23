---
title: LLM 输出解析的防御性编程：围栏、大写标签与裸 JSON 的混合处理
feedId: 38558
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

做 Agent 或 MCP 工具链时，LLM 的输出经常被当结构化数据用：插件要从回复里解析出工具名和参数，自动化脚本要把模型结论写回下游。理想情况是模型直接吐一份合法 JSON，但现实里它给的东西五花八门：有时包在代码围栏里，有时围栏语言标签是大写、有时干脆不写，有时前面先来一段"好的，以下是结果："。当 JSON 字段里还嵌着代码块时，解析难度再上一个台阶。

## 问题

很多人的第一版解析长这样：

```python
m = re.search(r'```json\n(.*?)```', output, re.S)
data = json.loads(m.group(1))
```

Demo 里能跑，生产上必挂：大写标签匹配不上、没围栏直接漏空、贪婪匹配吞掉多个代码块、字段值里嵌套围栏把 JSON 拦腰截断、还有尾逗号和全角标点。更麻烦的是模型升级会悄悄改变输出习惯，管线在某个周二突然开始报错。核心问题是：把解析当成一次性字符串操作，而不是一条分层的防御管线。

## 做法：五层递进

**1. 结构化输出优先。** 如果模型/供应商支持 JSON mode 或 tool call，参数本身就是结构化的，直接用。但跨模型回退和第三方 MCP 工具不总能保证，所以兜底层必须存在。

**2. 提取层。** 用大小写不敏感的正则收集所有围栏块（`r"```[a-zA-Z0-9]*\s*(.*?)```"` + `re.S`），不要只认 `json` 标签；找不到围栏再走裸文本。

**3. 修复层。** `json.loads` 失败后交给 `json_repair` 这类库，处理尾逗号、智能引号、注释。

**4. 校验层。** 用 Pydantic 做字段级校验。解析成功 ≠ 数据正确，缺字段、类型漂移要在这里拦住。

**5. 反馈重试。** 失败时把解析错误原文回传给模型重试一次，重试上限 1–2 次，原始输出永远落日志。

一个可直接抄的收敛版：

```python
import re, json

FENCE = re.compile(r"```[a-zA-Z0-9]*\s*(.*?)```", re.S)

def extract_json(text: str):
    # 1. 先试裸解析，也避免嵌套围栏误伤
    try:
        return json.loads(text.strip())
    except json.JSONDecodeError:
        pass
    # 2. 逐个尝试围栏块
    for m in FENCE.finditer(text):
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            continue
    # 3. 兜底：大括号截取 + 修复库
    i, j = text.find("{"), text.rfind("}")
    if 0 <= i < j:
        from json_repair import repair_json
        return repair_json(text[i:j + 1])
    raise ValueError("unparseable LLM output")
```

## 踩坑点

- **先裸解析再拆围栏**。一上来就剥围栏，遇到字段值里嵌代码块会把合法 JSON 截断。
- **贪婪 `.*`** 遇到多个围栏会跨块匹配，用非贪婪加 `finditer`，逐块尝试而不是赌第一个。
- **全角标点和中文引号**：只作为最后一道修复，别提前全局替换，容易破坏字符串内容。
- **流式场景**：半截 JSON 无法 `loads`，要么等终止信号，要么用增量解析器，别拿同一套代码硬套。
- **一次成功让你误以为永远成功**：模型换版本后输出习惯会变。

## 可复用建议

- 全项目收敛到一个 `parse_llm_json()`，别让每个插件各写一套正则。
- 建"脏输出语料库"：线上每次解析失败，把原文脱敏存档，回填成单测用例。
- 日志里记录命中了哪一层策略，排障省一半时间。
- 校验失败要带上下文抛出：哪一步、什么策略、原始片段前 200 字符。
- 有 tool call 就用 tool call，这套兜底只作为跨模型兼容层，不替代它。

## 总结

LLM 输出解析的本质是"不信任输入"：默认它会给你围栏、废话和非法逗号。把不确定性压进固定的管线——结构化输出优先、提取、修复、校验、带反馈重试——系统才能在模型升级和提示词调整时保持稳定。防御性代码不是不信任模型，而是让系统对模型的"口音变化"免疫。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/36b82b9ffd4accf5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/c28d078e98fba1dd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/eb5f6950d38d918f.png)

