---
title: LLM 输出解析的防御性编程：JSON 混合格式的分层提取方案
feedId: 38040
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

在 OpenClaw 里搭 Agent 流程、对接 MCP 工具或写插件自动化时，最耗时的往往不是推理本身，而是解析模型输出。典型场景：系统提示里明确要求"输出 JSON，用代码围栏包裹"，实际拿到的结果却五花八门——带语言标注的围栏、裸围栏、模型自作主张的 `<json>` 标签、甚至 JSON 前后夹一句解释。解析代码按单一格式写，跑几天后必然开始"偶尔失败"，而且失败得很安静。

## 问题

把线上失败样本攒下来分类，混合格式大致五类：

1. **围栏变体**：```json、```、单反引号、无围栏；
2. **标签包裹**：`<json>...</json>`、`<output>...</output>`；
3. **正文夹带**：JSON 前后有一两句说明文字；
4. **语法瑕疵**：尾逗号、单引号、字符串内未转义换行；
5. **截断**：`finish_reason=length` 导致缺右括号。

前四类可以解析救回，第五类救不回，只能识别后重试。只用一条正则或一次 `json.loads` 的代码，任何一类变化都会让整条流水线断掉。

## 做法

核心思路：**分层提取，先严格后宽松，最后校验**。

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*([\s\S]*?)```")
TAG   = re.compile(r"<(?:json|output)>\s*([\s\S]*?)</(?:json|output)>")

def _loads(s):
    try: return json.loads(s)
    except Exception: return None

def extract_json(text: str):
    text = text.strip()
    if (v := _loads(text)) is not None:            # 1. 整体就是纯 JSON
        return v, "raw"
    for m in FENCE.finditer(text):                 # 2. 逐围栏块试，避免贪心
        if (v := _loads(m.group(1))) is not None:
            return v, "fence"
    if (m := TAG.search(text)) and (v := _loads(m.group(1))):
        return v, "tag"                            # 3. 标签包裹
    if (v := _brace_scan(text)) is not None:       # 4. 括号配平扫描
        return v, "scan"
    try:                                           # 5. 修复兜底
        from json_repair import repair_json
        return json.loads(repair_json(text)), "repair"
    except Exception:
        return None, "fail"
```

`_brace_scan` 自己写不难：逐字符遍历，遇到引号进入字符串状态并处理 `\` 转义，只在字符串外对花括号计数，取第一段配平且能通过 `json.loads` 的片段。解析成功后不要直接用，再过一层 pydantic 校验字段和类型；失败则携带原始输出片段抛结构化错误，并触发一次带错误反馈的重试。

## 踩坑点

- **贪心正则遇多代码块会吞错对象**：模型有时先给示例再给答案，必须 `finditer` 逐块尝试。
- **括号计数不感知字符串**，会把 `"expr": "{}}"` 这类内容截出残废 JSON。
- **别一上来就 repair**：修复会静默改写数据，严格解析失败后才兜底，并在日志标记走了 repair。
- **先查截断再解析**：`length` 截断的输出修不好，直接重试更省时间。
- **别只依赖 JSON mode**：走 MCP 桥接或换后端时未必可用，解析层要自保。
- **日志存原始输出**而非只存解析结果，否则线上失败无法离线复现。

## 可复用建议

- 把 `extract_json` 做成所有插件共用的工具模块，一处修、处处生效。
- 线上失败样本脱敏后进测试语料，模型或提示词一升级就跑一遍分层单测。
- 统计每层命中率：repair 层占比持续偏高，说明该改提示词而不是继续打补丁。
- 提示词允许时，让模型输出单行 JSON 或 XML 标签，解析成本最低——但提示词再严格也别省解析层。

## 总结

把 LLM 输出当不可信输入对待，是 Agent 自动化的基本功。分层提取解决"格式漂移"，schema 校验解决"结构漂移"，原始日志加测试语料解决"长期演化"。这套东西总共几十行代码，却决定了你的自动化是偶尔抽风，还是长期可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/6837f3e4833e9481.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/f3b5c4d0ad43e851.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/bf09de06bbdf97d1.png)

