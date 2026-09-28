---
title: LLM 输出解析的防御性编程：一套 JSON 标签格式混合处理管线
feedId: 39389
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

在 OpenClaw 里写 skill、插件或 MCP 工具时，大量场景需要模型返回结构化结果——路由决策、工具参数抽取、批量结论。JSON 事实上是唯一通用格式，但实践中模型输出的"干净程度"远不如文档承诺：同一个 prompt，不同模型、不同温度、甚至同一模型的相邻两次会话，返回可能带 ```json 围栏、带 `<JSON>` 标签、带"以下是结果："式前后缀，也可能直接是裸 JSON。

## 问题

解析失败通常分三层：

1. **包装层**：代码围栏、XML 风格标签、前后缀闲话；
2. **格式层**：尾逗号、单引号、`True/None`、中文引号、字符串内未转义换行；
3. **结构层**：字段缺失、类型漂移（数字变字符串）、枚举值超纲。

很多人的起点是一条 `re.search(r"```json(.*?)```")`，然后在每个新样本上打补丁，最后正则堆成屎山，还盖不住"输出里出现两个代码块"这种 case。

## 做法

思路是**候选生成 + 逐个尝试 + 校验兜底**的管线，而不是单条匹配：

**第一步，源头先收敛**。Prompt 明确"只输出 JSON，不要解释"；接口支持原生 structured output / JSON Schema 的优先用——这是唯一从根上解决的手段，但不要只依赖它。

**第二步，按优先级生成候选串，每个候选拿去解析，成功即返回**：

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG   = re.compile(r"<(?:json|JSON|output)>\s*(.*?)\s*</(?:json|JSON|output)>", re.S)

def candidates(t: str):
    yield t.strip()
    yield from (m.strip() for m in TAG.findall(t))
    yield from (m.strip() for m in FENCE.findall(t))
    yield from balanced_scan(t)  # 字符串感知的括号配对，从首个 { 或 [ 取平衡子串

def parse_llm_json(t: str):
    for c in candidates(t):
        try:
            return json.loads(repair(c))
        except Exception:
            continue
    raise ValueError("unparseable llm output")
```

**第三步，repair 环节处理格式层脏数据**：尾逗号、单引号、Smart Quotes、`True/False/None`。别手写全套规则，社区现成的 `json_repair`（`pip install json-repair`）覆盖很全，直接引入。

**第四步，结构层用 Pydantic / JSON Schema 做校验和归一化**：默认值补齐、类型强转、枚举映射。校验失败不要静默吞掉，把 raw 原文和命中的解析分支一起打日志。

## 踩坑点

- **正则贪心匹配**：非贪婪 `.*?` 遇到模型顺手输出的解释性代码块会取错。所有候选全试一遍，比"只匹配第一个"可靠。
- **括号扫描不感知字符串**：`"note": "a } b"` 里的花括号会把平衡扫描搞崩，扫描时必须跟踪是否处于字符串内、是否处于转义态。
- **输出截断**：max_tokens 偏小时 JSON 被腰斩，任何 repair 都救不回来。检测到"括号不平衡 + 触达长度上限"应直接走重试，而不是进入修复流程。
- **双重编码**：有的模型会把 JSON 再序列化成字符串塞进字段，解析出一层后如果值仍是字符串，要递归再 parse 一次。
- **别让修复掩盖问题**：某模型或某版本失败率突然升高，先改 prompt 或换模型，修复管线是兜底，不是遮羞布。

## 可复用建议

- 把 `parse_llm_json` 沉淀成工具库单一入口，全项目禁止散落手写正则；
- 收集解析失败的 raw 样本，做成回归测试集，新模型接入先跑一遍；
- 管线分层（提取 → 修复 → 校验）各自独立，方便单独替换和打点；
- 每次解析记录命中了哪个候选分支，分布数据能告诉你哪种格式是主流、prompt 该怎么改。

## 总结

LLM 输出解析没有银弹：源头用 structured output 收敛，落地用"多候选 + 修复 + 校验"的防御管线承接，失败样本回流成测试集。防御性编程在这里的价值不是把脏输出变对，而是让脏输出**可观测、可恢复、可回归**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/11701cdb4c27f2f2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c095f71782e32986.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/09262109cb222ffe.png)

