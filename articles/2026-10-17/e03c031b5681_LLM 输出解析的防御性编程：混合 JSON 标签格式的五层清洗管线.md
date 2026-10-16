---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的五层清洗管线
feedId: 41734
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，让模型输出结构化 JSON 是常态操作。社区里常见的约定是让模型用 `<json>...</json>` 包裹结果，或者输出 markdown 围栏代码块。问题在于：只要换了模型版本、调高了温度、或者 prompt 被别的 agent 复用，输出格式就开始漂移。解析层如果只写了一个裸 `json.loads`，上线后大概率每周都要救火。

## 问题

把近期线上真实输出归个类，大概五类：

1. `<json>{...}</json>`，约定内的正常情况；
2. 标签里又套了一层 ```` ```json ```` 围栏；
3. 标签和围栏都没有，JSON 前后带一段解释文字；
4. 解释文字里还复述了一个示例 JSON，产生多个 `{...}` 候选块；
5. 语法瑕疵：全角引号、全角冒号、尾逗号、`//` 注释。

裸解析的命中率从演示时的 100% 掉到七成出头（小样本，仅供量级参考）。

## 做法：五层清洗管线

核心思路是"先提取、再修语法、最后校验"，每层独立、失败可降级：

```python
import json, re

def parse_llm_json(raw: str):
    text = raw.strip()
    # 1) 提取：收集所有 <json> 候选块，逐个尝试
    candidates = re.findall(r"<json>(.*?)</json>", text, re.S) or [text]
    for seg in candidates:
        s = seg.strip()
        # 2) 剥围栏：去掉开头 ```json / 结尾 ```
        s = re.sub(r"^```[a-zA-Z]*\s*|\s*```$", "", s).strip()
        # 3) 定界：第一个 { [ 到最后一个 } ]
        m = re.search(r"[{\[]", s)
        if not m:
            continue
        s = s[m.start():]
        s = s[: max(s.rfind("}"), s.rfind("]")) + 1]
        # 4) 修复：全角标点归一后先严格解析，再交宽松解析器兜底
        fixed = s.replace("“", '"').replace("”", '"') \
                 .replace("：", ":").replace("，", ",")
        for attempt in (fixed, s):
            try:
                return json.loads(attempt)
            except json.JSONDecodeError:
                pass
        try:
            import json5
            return json5.loads(s)  # 尾逗号、单引号、注释
        except Exception:
            continue
    raise ValueError("parse failed, raw output saved")
```

- **校验层**（代码外）：解析成功后还要过 pydantic / JSON Schema，缺字段、类型错都算失败；
- 全部失败时，把解析错误信息拼进 prompt 让模型自修复重试一次，同时把原始输出落盘留证。

## 踩坑点

- **贪婪取第一个 `{`**：模型在解释文字里复述示例时会切错块，必须收集候选逐个验证，"能解析且过校验"者优先；
- **盲目换单引号**：字符串值里的撇号（`it's`）会被误伤，这类交给宽松解析器，别自己写正则；
- **正则去尾逗号**：可能命中字符串内部的逗号，同上，交给解析器；
- **嵌套围栏**：JSON 字段值本身含 ` ``` ` 代码串时，朴素 strip 会从中间截断，围栏要按配对处理或取最大平衡段；
- **解析成功 ≠ 数据正确**：字段缺失、类型不对只能靠 schema 兜底，解析层管不到；
- **流式场景**：标签没闭合就急着 parse 会拿到半截 JSON，等闭合标记再做。

## 可复用建议

- 把管线做成公共 util，prompt 和解析器一起做版本管理，改 prompt 必须跑一遍坏样本回归集；
- 持续收集线上失败样本进单元测试语料，坏样本库比任何聪明正则都值钱；
- 失败时永远保留原始输出原文，只存解析结果会让你失去复盘能力；
- 低温度能减少格式漂移，但别依赖它——解析层必须独立兜底。

## 总结

这套管线的出发点只有一句话：把 LLM 输出当不可信输入。五层做下来，解析成功率回到接近 100%，残余失败都有原始日志可追。防御性解析的成本是一次性的，收益是每条自动化链路都少一类静默故障。建议直接写进你下一个插件的默认模板里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/f5da21f2404db133.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/f5857c4a6f042761.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/bc9c2a34beeec949.png)

