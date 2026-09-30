---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 39875
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

在 OpenClaw 的插件和自动化管线里，让 LLM 输出 JSON 是高频操作：工具调用参数、MCP 结果结构化、多 Agent 之间的消息传递。我们通常在 prompt 里约定一种格式——"用 ```json 围栏输出"或"包在 `<output>` 标签里"——然后就按这一种格式写解析。

现实是：同一份 prompt，在不同模型版本、不同温度、甚至长对话的不同轮次里，会返回好几种格式。裸 JSON、带 `json` 语言标注的围栏、裸 ``` 围栏、自定义标签包裹、围栏外面还带一句"以下是结果："，都见过。

## 问题

单一解析路径在格式漂移时整条管线挂掉，而报错往往只有一句 `JSONDecodeError: Expecting ',' delimiter`，不带原始输出，定位成本很高。解析本质上是系统边界上的"不可信输入处理"，应该按防御性编程来做。

## 做法：分层降级解析

核心思路是把解析收敛成一个模块，按命中率从高到低逐层尝试，命中即返回：

```python
import json, re

FENCE_RE = re.compile(r"```(?:json|JSON)?\s*(.*?)```", re.S)
TAG_RE = re.compile(r"<(?:json|output|result)>\s*(.*?)\s*</(?:json|output|result)>",
                    re.S | re.I)

def extract_json(text: str):
    # 1. 直接解析（模型听话时最快路径）
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # 2. 代码围栏，逐块尝试
    for m in FENCE_RE.findall(text):
        try:
            return json.loads(m.strip())
        except json.JSONDecodeError:
            continue
    # 3. 自定义标签
    for m in TAG_RE.findall(text):
        try:
            return json.loads(m)
        except json.JSONDecodeError:
            continue
    # 4. 括号配平扫描（字符串感知版）
    frag = balanced_scan(text)
    if frag:
        try:
            return json.loads(frag)
        except json.JSONDecodeError:
            pass
    # 5. 修复兜底（json_repair 类库），失败则抛出带原始输出的异常
    return repair_and_load(text)
```

第 4 层的 `balanced_scan` 必须跟踪字符串状态和转义，否则 JSON 值里出现 `{` 或 `"` 就会深度错乱：

```python
def balanced_scan(text):
    start, depth, in_str, esc = None, 0, False, False
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == '\\': esc = True
            elif ch == '"': in_str = False
            continue
        if ch == '"': in_str = True
        elif ch in '{[':
            if depth == 0: start = i
            depth += 1
        elif ch in '}]':
            depth -= 1
            if depth == 0 and start is not None:
                return text[start:i + 1]
    return None
```

解析成功不等于数据正确。出口处再加一层 schema 校验（pydantic / jsonschema），字段缺失、类型不符在这里拦截，报错带上原始输出片段。

## 踩坑点

1. **贪婪正则撕数据**：`.*` 会从第一个 ``` 吃到最后一个 ```。模型在 JSON 字符串里输出代码片段（内含围栏）时，整段直接报废。用非贪婪加 `findall`，逐块尝试。
2. **括号计数忽略字符串**：深度计数不跟踪 `in_str` 和转义符，遇到含引号、大括号的值必挂。
3. **全角字符**：中文 prompt 场景下偶发全角引号""和全角逗号，`json.loads` 直接报错。修复层处理，但必须记日志，不要静默修。
4. **`json.loads` 接受 `NaN`/`Infinity`**：这不是严格 JSON，下游数值计算会埋雷，可用 `parse_constant` 拒绝。
5. **格式契约会漂移**：换模型、升版本后输出格式可能变。别假设标签格式永远稳定。

## 可复用建议

- 解析收敛到单一模块，禁止业务代码里到处裸 `json.loads`。
- 每层 fallback 打遥测点：记录命中了第几层、失败时原始输出落盘（截断 + 脱敏）。哪层命中率上升，就是模型行为漂移的早期信号。
- prompt 侧仍然要做契约：明确的格式要求 + 一个 one-shot 示例 + 低温，能显著降低混合格式比例；有 JSON mode / 结构化输出 API 就优先用。但这些都只是降低概率，不是保险。
- 修复兜底（第 5 层）要保守：宁可失败抛错让人介入，也不要静默"修"出错误数据。

## 总结

LLM 输出解析本质是边界信任问题。分层降级解决"格式不统一"，遥测解决"格式在漂移"，schema 校验解决"格式对但内容错"。三层齐了，管线才敢长期无人值守地跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/199d605a905a0c97.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/0441cbeb268fed53.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/420fcfc1d05d57c7.png)

