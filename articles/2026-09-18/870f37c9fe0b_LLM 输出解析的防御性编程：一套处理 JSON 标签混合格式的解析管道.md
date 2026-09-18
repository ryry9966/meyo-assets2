---
title: LLM 输出解析的防御性编程：一套处理 JSON 标签混合格式的解析管道
feedId: 38068
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

在 OpenClaw 插件、Agent 工作流和 MCP 工具调用里，让 LLM 输出 JSON 是最常见的结构化手段：工具参数、节点间状态传递、自动化流水线的中间结果，基本都靠它。但实际跑起来你会发现，模型给的 JSON 几乎从不"干净"：有时是裸 JSON，有时包一层 ` ```json ` 围栏，有时外面又套了 `<result>` 标签，甚至 `<think>` 思考内容和正文混在一起。

写插件时如果假设"输出一定是可直接 `json.loads` 的字符串"，这套管道迟早会在某次换模型、加长上下文或者用户输入里带了几个反引号之后炸掉。

## 问题

典型脏输出大概这几种形态：

1. 纯 JSON（理想情况，实际占比没你想的高）
2. ` ```json ... ``` ` 围栏包裹
3. ` ``` ` 无语言标注的围栏
4. `<json>` / `<result>` / `<output>` 等 XML 风格标签包裹
5. 前后混着解释性文字或思考片段，片段里还带花括号
6. JSON 字段值里嵌套了代码围栏，导致你的正则提前截断
7. 尾逗号、单引号、智能引号、BOM

任何一种单独出现都不难处理，麻烦的是它们随机混合出现。

## 做法：一条五层解析管道

核心思路是**不假设格式，按优先级收集候选串，逐个尝试解析，最后强制过 schema**：

```python
import re
from pydantic import TypeAdapter

FENCE_RE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)      # 代码围栏
TAG_RE   = re.compile(r"<(?:json|result|output)>(.*?)</(?:json|result|output)>", re.S | re.I)
BRACE_RE = re.compile(r"\{.*\}|\[.*\]", re.S)                # 兜底

def extract_candidates(text: str) -> list[str]:
    text = text.replace("\ufeff", "").strip()          # 去 BOM / 零宽字符
    cands = [s.strip() for s in FENCE_RE.findall(text)]   # 1. 围栏最优先
    cands += [s.strip() for s in TAG_RE.findall(text)]    # 2. XML 标签
    cands += [text, *[s.strip() for s in BRACE_RE.findall(text)]]  # 3. 原文兜底
    return cands

def parse_llm_json(text: str, adapter: TypeAdapter):
    last_err = None
    for cand in extract_candidates(text):
        try:
            return adapter.validate_json(cand)   # 解析 + schema 校验一步完成
        except Exception as e:
            last_err = e
    raise ValueError(f"JSON 解析失败: {last_err}")
```

要点：

- **规范化**：去 BOM、零宽字符、首尾空白，智能引号可按需替换。
- **提取**：候选顺序是"围栏内 > 标签内 > 首尾大括号块 > 全文"。围栏和标签的匹配用非贪婪 `.*?`，保证多个独立块能逐个取出。
- **解析 + 校验合一**：pydantic v2 的 `validate_json` 会同时完成 JSON 解析和 schema 校验，坏结构直接在入口被拦下。
- **失败路径**：捕获异常后留原始输出到日志，把错误信息回传给模型做一次自修复重试，两次封顶，再失败就走人工队列或降级逻辑。

## 踩坑点

- **贪婪正则吞块**：用 `.*` 会把两个围栏之间的解释文字一起吞进来，必须非贪婪。
- **嵌套围栏截断**：JSON 字段值本身含 ` ``` ` 时，围栏正则会在内部提前闭合。兜底方案是保留"最后一个闭合围栏之后的整段文本"作为候选，或写括号平衡扫描。
- **思考片段里的伪 JSON**：`<think>` 内容带花括号时，兜底正则会抓到它。有候选优先级 + schema 校验双重过滤，不会误用，但日志里会多几条失败记录，属正常噪音。
- **容错解析器的副作用**：json5 / dirty-json 能修尾逗号和单引号，但也可能把坏数据"修成"合法但语义错误的 JSON。可以用于第二梯队，但校验层绝不能省。
- **流式输出**：不要边流边 parse，等完整消息落盘，或按围栏分隔符做增量缓冲再解析。

## 可复用建议

- 把提取、解析、校验做成独立纯函数，收敛成一个公共库，Agent、MCP 工具、插件共用，别每个插件各写一份正则。
- 解析层统一返回结构化结果（`ok / data / raw / err`），上层永远不接裸异常。
- 对解析成功率和失败原因分布打点监控。换模型、改提示词之后，看一眼这条曲线就知道格式稳定性有没有劣化。
- 提示词里要求"只输出 JSON、不要围栏"值得写，但架构上只能依赖解析端的容错，提示词只是降低触发概率。

## 总结

防御性解析的本质是：**不信任输出格式，信任 schema 校验**。提取层做候选排序，解析层做逐级降级，校验层一票否决，失败路径可观测、可重试。这套管道大概五十行代码，换来的是插件在模型升级和脏输入面前不再裸奔，建议所有做工具调用的同学都备一份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/34905d62a7e61795.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/127f16233365e082.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/33903c6fd9533f25.png)

