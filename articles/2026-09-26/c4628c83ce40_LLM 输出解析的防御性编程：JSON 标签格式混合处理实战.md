---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 39017
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，LLM 输出经常要被下游代码直接消费：Agent 把结构化结果交给工具调用，自动化流水线把模型回复写回数据库。Prompt 里写得再清楚——"只输出 JSON"——线上拿到的依然是各种形态的混合体。输出解析是整条链路里最不起眼、也最容易在深夜报警的环节。

## 问题

实际观测到的混合格式大致几类：

- 回复开头一句解释，中间是 ```json 码栅栏，栅栏后面又跟一句备注；
- 不加栅栏，直接裸 JSON，但两侧带空白或零宽字符；
- 推理类模型前面多了一段 `<think>…</think>`；
- JSON 本身"半合法"：全角引号、尾逗号、单引号；
- 一次输出里出现两个 JSON 块（一个草稿一个最终答案）。

任何一层用 `json.loads(response)` 裸解析，失败率都不低。更糟的是不少实现失败后静默返回 `None`，上游拿着空数据继续跑，错误被推迟到很远的地方才暴露。

## 做法

我的做法是把解析做成一条分层管道，每层只处理自己能处理的情况，层层兜底：

```python
def extract_json(text: str):
    text = strip_reasoning_tags(text)        # 1. 按 <think> 起始标签截取后半段
    try:
        return json.loads(text)              # 2. 直接解析
    except json.JSONDecodeError:
        pass
    for block in find_code_fences(text):     # 3. 逐个试码栅栏
        try:
            return json.loads(block)
        except json.JSONDecodeError:
            continue
    for cand in balanced_scan(text):         # 4. 字符串感知的花括号配平扫描
        try:
            return json.loads(repair(cand))  # 5. 修全角引号/尾逗号后再试
        except json.JSONDecodeError:
            continue
    raise ParseError(text)                   # 6. 带原始输出上抛
```

几个关键点：

1. **花括号配平要"懂字符串"**。`re.search(r'\{.*\}', text, re.S)` 会从第一个 `{` 贪婪吃到最后一个 `}`，把两个块之间的解释文本一起吞进去。正确做法是状态机扫描：记录是否在字符串内、是否转义，只统计字符串外的括号。
2. **修复是最后一步，不是第一步**。先原样解析，修复只对"差一点就合法"的输出生效。盲目把单引号换双引号，会把值里的 `it's` 一起改坏。
3. **JSON 合法 ≠ 数据可用**。解析成功后必须过 schema 校验（pydantic 或手写字段检查），拦住字段缺失、数字变字符串这类类型漂移。
4. **失败要带上下文上抛**。抛错时附带原始输出，由上层决定：带着解析错误回喂模型重试（限 1–2 次），或直接告警。

## 踩坑点

- **嵌套码栅栏**：JSON 字符串值里本身含 ``` 时，按栅栏切分会切烂。配平扫描能兜住，别把栅栏剥离当成唯一手段。
- **推理标签不总是成对**：输出截断时 `</think>` 可能缺失，strip 逻辑要按"起始标签之后截取"处理，而不是只做配对替换。
- **重试不喂错误信息**：原样重发，模型大概率再犯同样的错。重试 prompt 要写清楚上次输出错在哪。
- **流式场景**：边收边 parse 会把截断 JSON 误判为失败。要么聚合完再解析，要么用增量解析器显式处理 incomplete 状态。
- **迷信万能 repair 库**：这类工具能把任何输入"修"出结果，但容忍不等于正确，修出来的数据一样要过 schema。

## 可复用建议

- 解析器写成无副作用的纯函数模块，独立于业务代码。
- 维护脏样本回归集：线上每次解析失败，脱敏后存入用例库跑 CI。这个集子比反复调 prompt 值钱。
- 解析器是第二道防线，第一道永远是 provider 的结构化输出能力（response_format / tool call 的 schema 约束），能用就用。
- prompt 侧配合：给出精确的字段示例，明确"不要码栅栏、不要解释文字"，能消掉一半脏格式。

## 总结

LLM 输出解析的防御性编程，本质是承认"模型输出是不可信输入"。分层提取、schema 校验、带反馈的有限重试、失败可观测——这四件事做齐，解析环节的报警量会明显下降。它不性感，但它是 Agent 和自动化流水线能长期稳定运行的地基。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/cfa4a54abc3ad29f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/096833510d278606.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/74e41da8119a05a1.png)

