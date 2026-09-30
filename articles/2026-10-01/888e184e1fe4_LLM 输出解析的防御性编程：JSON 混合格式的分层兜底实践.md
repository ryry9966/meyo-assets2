---
title: LLM 输出解析的防御性编程：JSON 混合格式的分层兜底实践
feedId: 39929
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，让模型返回 JSON 几乎是标配：工具调用参数、结构化抽取、Agent 之间的中间结果传递。提示词里写一句"只输出 JSON"，大部分时候也确实能拿到。问题恰恰出在"大部分时候"——换模型、升级版本、调温度、上下文变长之后，输出格式会悄悄漂移。解析层是整条自动化链路里最不起眼、也最容易在半夜把人叫起来的一层。

## 问题

实际观测到的输出形态至少有六种：

1. 裸 JSON（理想情况）；
2. ` ```json ` 围栏包裹；
3. `<json>`、`<tool_call>`、`<answer>` 等标签包裹（部分模型训练模板自带）；
4. 推理模型前置 `<think>…</think>`，真正的 JSON 在后面；
5. JSON 前后带一句"好的，以下是结果："；
6. 伪 JSON：尾逗号、全角引号、单引号、Python 风格的 `True/None`。

裸写 `json.loads(resp)` 只在形态 1 下成立。同一段代码在不同模型、不同 session 下轮流命中不同分支，这就是解析层脆弱的根源。

## 做法

思路：把解析做成一条**由宽到窄的抽取管道**，每层只处理自己认识的形态，命中即返回，全部失败再走修复。

```python
import json, re

THINK = re.compile(r"<think>.*?</think>", re.S)
FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG   = re.compile(r"<(json|tool_call|answer)>\s*(.*?)\s*</\1>", re.S)

def balanced_slice(s: str):
    """截取第一段括号配平的片段，忽略字符串内部的括号"""
    depth, start, in_str, esc = 0, None, False, False
    for i, c in enumerate(s):
        if in_str:
            if esc: esc = False
            elif c == "\\": esc = True
            elif c == '"': in_str = False
        elif c == '"': in_str = True
        elif c in "{[":
            if depth == 0: start = i
            depth += 1
        elif c in "}]":
            if depth > 0:
                depth -= 1
                if depth == 0: return s[start:i + 1]
    return None

def repair(s: str) -> str:
    s = re.sub(r",\s*([}\]])", r"\1", s)          # 尾逗号
    s = s.replace("“", '"').replace("”", '"')     # 全角引号
    return re.sub(r"\b(True|None)\b",
                  lambda m: {"True": "true", "None": "null"}[m.group(1)], s)

def parse_llm_json(text: str):
    meta = {"strategy": None}
    s = THINK.sub("", text).strip()               # ① 预清洗
    if (m := FENCE.search(s)): s = m.group(1).strip()   # ② 围栏
    if (m := TAG.search(s)):   s = m.group(2).strip()   # ③ 标签
    for name, cand in (("direct", s), ("balanced", balanced_slice(s))):
        if not cand: continue
        try:
            return json.loads(cand), {**meta, "strategy": name}   # ④⑤
        except json.JSONDecodeError:
            try:
                return json.loads(repair(cand)), {**meta, "strategy": name,
                                                  "repairs": ["repair"]}  # ⑥
            except json.JSONDecodeError:
                continue
    raise ValueError("LLM 输出中未找到可解析的 JSON")
```

管道之后还有两步：用 pydantic/jsonschema 做 **schema 校验**，字段对不上才算真失败；失败时把解析错误信息拼回提示词**带反馈重试一次**，封顶一次，仍失败就走告警/降级。每次调用把命中的策略写进日志和指标。

## 踩坑点

- **正则 `.*?` 提取 `{...}` 会翻车**：嵌套对象、字符串里的花括号都会截断。必须用配平扫描，且扫描时要处理字符串内的引号转义。
- **单引号转双引号要极度谨慎**：字符串内容本来就可能有撇号（`"it's"`），一转就修坏。所以修复永远放在直接解析失败之后，而不是无脑前置。
- **全角引号肉眼难查**：中文语境下模型偶发输出 `"键"："值"`，报错信息还指不到具体字符，修复层统一替换是最省心的。
- **管道顺序很重要**：若正文本身包含花括号示例，配平扫描可能截错段，所以围栏和标签提取必须排在它前面。
- **重试无上限 = 成本爆炸**：带错误反馈的重试一次封顶。
- **温度压不下来，漂移压不住**：结构化任务把 temperature 设到 0~0.3，能消掉大部分格式抖动。
- **模型一升级，分布就变**：各策略命中率要当指标看，"balanced/repaired" 占比突然上升，就是模型行为漂移的信号。

## 可复用建议

- 做成共享 util 或插件内公共函数，别每个 tool 各抄一份，坏一处漏一处。
- 返回值带 meta（命中策略、修复动作），日志保留原始输出，出问题能回放。
- schema 校验只放在边界处一次，内部调用点信任已校验对象。
- 提示词要求模型用固定标签包裹并在 few-shot 里示范——提示词降低漂移概率，解析层兜底，两者缺一不可。
- 把"LLM 输出一律按不可信输入处理"写进团队的 code review 清单。

## 总结

解析层本质上是在和一个不确定性对手签合同：提示词是君子协定，解析管道才是法律条文。30 行左右的防御性解析，换来的是深夜告警少一半、模型可替换、指标可观测。这类投入小、回报稳定的工程习惯，值得每个写 Agent 的人先补上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/caf742dbbbcafee5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/40e80c675516053e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/4f4b129a3bfc0382.png)

