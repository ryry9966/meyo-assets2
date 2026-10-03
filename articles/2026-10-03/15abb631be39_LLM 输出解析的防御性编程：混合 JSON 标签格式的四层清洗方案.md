---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的四层清洗方案
feedId: 40234
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

在 OpenClaw 的插件和自动化流里，只要让 LLM 做"决策"或"抽取"，就绕不开同一个动作：把模型的自由文本变成可执行的结构。路由 agent 要吐 `{"tool": "web_search", "args": {...}}`，抽取 agent 要吐字段化结果。我们习惯假设"prompt 里写清楚只输出 JSON 就够了"，实践中这个假设经常半夜被打脸。

## 问题：同一段 prompt，五种输出

同一个 system prompt（已包含"直接输出 JSON，不要任何多余内容"），换模型、换版本、甚至只换采样参数，实测至少见过这些形态：

1. 纯 JSON——理想情况，占比反而未必最高；
2. ` ```json ... ``` ` 围栏包裹；
3. 前后带解释："以下是解析结果："……"希望对你有帮助"；
4. 自定义标签包裹：`<answer>{...}</answer>`、`<tool_call>...</tool_call>`；
5. 推理类模型的 `<think>...</think>` 混在正文里。

再叠加 JSON 本身不合法的情况——尾逗号、单引号、字符串里未转义的换行——任何一处裸写 `json.loads`，整条 workflow 就断。粗暴重试会把全量 prompt 重发一遍，贵，而且失败模式经常原样复现。难的不是处理某一种格式，而是你无法预知这一次是哪种。

## 做法：四层防御，逐级降级

核心思路：不指望规则穷尽所有形态，而是按"破坏性最小 → 最后兜底"排一条处理链，命中即返回。

- **第 0 步：先删 think。** 非贪婪 + `re.DOTALL` 剥掉 `<think>...</think>`。必须放最前面，否则 think 里的 `{` 会污染后面的括号定位——这是我们踩过最隐蔽的坑。
- **第一层：剥已知包装。** 按优先级提取代码围栏、白名单标签（answer / json / result / tool_call）。
- **第二层：括号定位切片。** 取第一个 `{` 或 `[` 到最后一个 `}` 或 `]` 之间的内容。比正则稳健，对嵌套和前后废话免疫，能兜住大部分"围栏写法没对上"的场景。
- **第三层：宽松解析兜底。** `json.loads` 失败后用 json5 等宽松解析器；`ast.literal_eval` 只在确认是 Python 风格时用。不建议手写正则"修复"JSON，误伤率高。
- **第四层：校验 + 定向重试。** 解析成功不等于结果可用，用 pydantic 校验字段类型和枚举；失败时把"原始输出 + 具体 parse error"追加进对话重试一轮，只重发错误上下文。

```python
import json, re

FENCE_RE = re.compile(r"```(?:json)?\s*(.*?)```", re.DOTALL)
THINK_RE = re.compile(r"<think>.*?</think>", re.DOTALL)
TAGS = ("answer", "json", "result", "tool_call")

def _try(s):
    try:
        return json.loads(s)
    except Exception:
        return None

def _outer_slice(s: str) -> str:
    starts = [i for i in (s.find("{"), s.find("[")) if i != -1]
    if not starts:
        return s
    l, r = min(starts), max(s.rfind("}"), s.rfind("]"))
    return s[l:r + 1] if r > l else s

def parse_llm_json(text: str):
    text = THINK_RE.sub("", text).strip()      # 0. 先删 think
    obj = _try(text)                           # 1. 直连
    if obj is not None:
        return obj
    for m in FENCE_RE.finditer(text):          # 2. 围栏
        obj = _try(m.group(1).strip()) or _try(_outer_slice(m.group(1)))
        if obj is not None:
            return obj
    for tag in TAGS:                           # 3. 白名单标签
        m = re.search(rf"<{tag}>(.*?)</{tag}>", text, re.DOTALL)
        if m:
            obj = _try(m.group(1).strip())
            if obj is not None:
                return obj
    obj = _try(_outer_slice(text))             # 4. 括号定位
    if obj is not None:
        return obj
    try:                                       # 5. 宽松兜底
        import json5
        return json5.loads(_outer_slice(text))
    except Exception as e:
        raise ValueError(f"unparseable output: {text[:80]!r}") from e
```

## 踩坑点

- 正则忘了 `re.DOTALL`：跨行围栏一个都匹配不上，症状是"本地好线上挂"。
- 字符串值里含 `{}`：首尾括号切片在极端情况会切进字符串内部，简单修正法是准备多个候选切片逐个尝试。
- 清洗规则硬编码进业务代码：模型一升级规则全废，务必抽成独立工具函数。
- 流式场景别边收边 `json.loads`，等结束标记或用带缓冲的流式 JSON 解析。
- `temperature=0` 不等于格式稳定，别把它当防御手段。

## 可复用建议

- 清洗逻辑沉淀为插件公共库里的 util，所有 workflow 复用同一份，修一处全生效。
- 每次解析前把原始输出落盘。攒下来的 badcase 就是现成的回归集，改清洗规则前先跑一遍历史样本。
- 打一个分布指标：fence / 标签 / 括号切片 / 宽松兜底各占多少比例。兜底占比突然上升，大概率是上游模型变了——这比报错更早暴露问题。
- prompt 端同步做减法：明确"第一行开始就是 JSON，禁止围栏和解释"，并给一个形状示例；有条件优先用 API 层的 JSON schema 约束，防御代码降级为兜底。

## 总结

防御性解析的前提是承认输出不可控。剥离、定位、宽松解析、校验，这四层的本质是把无限可能的脏输出收窄成有限几种已知形态逐个处理，再靠 badcase 回归和分布监控维持长期有效。改造之后，我们那条链路从"每天要人工捞几次解析失败"降到一周偶尔一两次，而且失败可解释、可复现。代码不长，值得沉淀进自己的工具库。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/4d18782795c5c802.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/24b2513c852f316f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/335df6a92ccd4f48.png)

