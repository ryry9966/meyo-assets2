---
title: 把 LLM 输出当不可信输入：一套 JSON 标签混合格式的防御性解析方案
feedId: 38726
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

在 OpenClaw 插件和 MCP 工具开发里，让模型输出 JSON 是常态：结构化结果回传、工具参数填充、pipeline 中间态传递。Prompt 里写了"只输出 JSON，不要任何解释"，模型大多数时候也确实照做——但对自动化流程来说，"大多数时候"远远不够。

## 问题

翻一段线上日志，输出的污染形态比想象中多：

- ` ```json ` 围栏，或没有语言标注的裸围栏；
- `<json>`、`<result>` 之类模型自造的标签；
- JSON 前后各带一句解释性文字；
- 一次输出多个 JSON 块；
- 全角标点、尾逗号、字符串里再套一层转义 JSON。

`json.loads` 直接抛异常；用正则硬抠又太脆：换模型、换 provider、甚至调一次 temperature，格式分布就会漂移。解析失败不是偶发 bug，是需要当成常态来工程化处理的问题。

## 做法

思路是分层解析、逐层降级，核心四步：

1. **剥壳**：去掉 markdown 围栏和自造标签，trim；
2. **候选提取**：不用正则抠 JSON，改用括号配平扫描器。扫描时维护"是否在字符串内"的状态，避免字符串里的 `{}` 干扰计数，返回所有顶层候选块；
3. **逐候选解析**：先原样 `loads`，失败再走修复链——全角标点转半角、去尾逗号、去行注释。修复项逐个叠加，不要一次全上；
4. **校验与兜底**：pydantic / jsonschema 校验字段；失败则记录原始输出和模型元信息，触发一次附带错误信息的重问，再失败走降级路径。

核心代码骨架：

```python
import json, re

def strip_wrappers(text: str) -> str:
    m = re.search(r"```(?:json)?\s*(.*?)```", text.strip(), re.S)
    if m:
        text = m.group(1)
    return re.sub(r"</?(json|result|output)>", "", text).strip()

def find_json_candidates(text: str):
    cands, depth, start, in_str, esc = [], 0, None, False, False
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
            continue
        if ch == '"': in_str = True
        elif ch in "{[":
            if depth == 0: start = i
            depth += 1
        elif ch in "}]":
            if depth > 0:
                depth -= 1
                if depth == 0: cands.append(text[start:i+1])
    return cands

def repair(s: str) -> str:
    s = s.replace("，", ",").replace("：", ":")
    s = re.sub(r",\s*([}\]])", r"\1", s)
    return re.sub(r"^\s*//.*$", "", s, flags=re.M)

def parse_llm_json(text: str):
    body = strip_wrappers(text)
    for cand in find_json_candidates(body):
        for variant in (cand, repair(cand)):
            try:
                obj = json.loads(variant)
                return json.loads(obj) if isinstance(obj, str) else obj
            except (json.JSONDecodeError, TypeError):
                continue
    raise ValueError("no parseable JSON in LLM output")
```

其中 `isinstance(obj, str)` 那行顺手处理了双重编码：模型把 JSON 当字符串再包一层时，递归 parse 一次。

## 踩坑点

- `r"\{.*\}"` 贪婪匹配遇到多个 JSON 块，会把中间的解释文字一起吞掉；改成非贪婪又会在嵌套处提前截断。配平扫描是更稳的基准。
- 修复是双刃剑：全局替换全角标点，可能改坏字符串字段里本该保留的中文标点。所以永远先试原样，修复只作用于解析失败后的候选。
- 静默兜底最危险：解析失败时返回空 dict，下游"看起来一切正常"，问题被推迟到更难排查的地方。宁可抛异常加完整日志。
- 别只测 happy path。模型某天给你一段前后散文夹两个 JSON，你的测试套件里就应该已经有这条用例。

## 可复用建议

- 解析层做成独立模块：输入字符串，输出校验后的对象，与业务解耦，插件和 MCP 工具直接复用。
- 从日志里攒"脏输出语料库"，每条 badcase 固化成回归用例，跑在 CI 里。
- Prompt 侧防御照样要做（few-shot 示例 + 明确禁令），但只当第一道过滤，不当唯一防线。有 function calling / 结构化输出能力就优先用，解析层保留作跨 provider 兜底。
- 日志带上 prompt 版本和模型 ID，格式漂移发生时才有得查。

## 总结

一句话：把 LLM 输出当作用户输入对待——不信任、先剥壳、逐层降级、严格校验、失败留痕。这套解析器写一次就能覆盖大部分插件场景，而随时间积累的 badcase 语料库，会成为项目里最值钱的测试资产之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/d13ce091195a0e16.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/49c7dbb5eba4e874.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/10d9655abb674be2.png)

