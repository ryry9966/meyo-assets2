---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 39480
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，Agent 的输出经常就是下一环节的输入：调度器要读 JSON 里的 tool 名称，自动化流水线要读参数对象。提示词里我们都会写"只输出 JSON"，模型也答应了——但答应得不彻底。

跑了几周真实任务后，攒下来的失败样本大致几类：

- 输出包在 ```json 围栏里，而不是裸 JSON
- 围栏前后多一句"以下是解析结果："
- 一次输出两段 JSON（一段说明、一段结果）
- JSON 的字符串值里本身嵌了 ```（比如让模型返回代码片段时）
- 全角引号、尾逗号、`//` 注释
- 被 max_tokens 截断的半截 JSON

## 问题

`json.loads` 是个诚实但脆弱的函数：符合规范就过，否则抛异常。而很多脚本的写法是这样的：

```python
data = json.loads(resp.split("```json")[1].split("```")[0])
```

demo 里能跑，真实流量里会以各种姿势挂掉。最典型的事故：要求模型在 JSON 字段里返回代码片段，值内嵌了 ```，`split` 按第一个围栏切下去，把合法 JSON 腰斩，下游拿到 None 还不知道错在哪一层。

## 做法

核心思路是分层：**提取 → 修复 → 校验 → 重试 → 降级**，每层独立可测。

1. **原样留存**。raw response 和 finish_reason 落日志。截断的输出别浪费时间做 repair，直接走重试。
2. **先直接 `loads`**。低温下很多模型真会给纯 JSON，别上来就加戏。
3. **贪婪剥围栏**。用"第一个 fence 到最后一个 fence"的贪婪匹配，而不是非贪婪 regex，避免被值内嵌围栏截断。
4. **括号配对兜底**。扫描第一个 `{` 或 `[`，做括号计数（跳过字符串内部和转义符），取出配对完整的片段。
5. **容错修复只做一次**。用 `json_repair` 或手写规则处理全角引号、尾逗号，修复结果必须再过一遍 `loads`。
6. **解析与校验分离**。`loads` 成功 ≠ 数据可用，用 pydantic/jsonschema 验字段类型和形状。
7. **带错误信息的重试**。把解析异常拼回 prompt 让模型自纠，最多 2 次，之后走结构化报错，而不是让 None 一路传下去。

```python
import json, re

def extract_json(text: str):
    candidates = [text]
    # 贪婪匹配：第一个围栏到最后一个围栏
    m = re.search(r"```(?:json)?\s*(.+)\s*```", text, re.S)
    if m:
        candidates.insert(0, m.group(1))
    for c in candidates:
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    from json_repair import repair_json
    return json.loads(repair_json(text))
```

## 踩坑点

- **非贪婪正则 + 嵌套围栏 = 静默截断**。`.*?` 会在第一个收尾围栏就停，且不报错，最难受。
- **括号计数不看字符串内部**。碰到含模板字符串或 `{}` 字面量的代码值，计数直接崩。
- **修复过猛比失败更难查**。比如把字符串值里的单引号也替换了，解析"成功"但数据已污染。
- **只测 happy path**。把线上真实脏样本攒成回归测试语料，比任何 mock 都有用。
- **忽略 finish_reason**。对截断输出做修复属于徒劳，先判断完整性。

## 可复用建议

- 解析器收敛成**一个**工具模块、全项目唯一入口，别到处复制 `split`。
- 四层解耦，每层单独写单测，脏样本语料持续回灌。
- 有 function calling / 结构化输出模式时优先用，但这套兜底要保留——切模型、切供应商时救命。
- 把 parse 成功率做成指标，突降往往对应模型升级或 prompt 变更。

## 总结

防御性解析的本质不是写出更聪明的正则，而是承认模型输出是脏输入，然后按层拆解：提取、修复、校验、重试、降级，每层简单、可测、可观测。在 OpenClaw 这种多工具串联的场景里，上游多写五十行解析代码，能省掉下游一堆"参数怎么是 None"的灵异问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/5e87a946b2f340ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/f31ce4c515e4a6d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b5802bfc522f4f17.png)

