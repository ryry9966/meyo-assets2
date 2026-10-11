---
title: LLM 输出解析的防御性编程：JSON 围栏与标签混合格式的分层处理
feedId: 41268
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，LLM 的输出经常直接作为下游工具的输入。我们通常在 prompt 里要求"只输出 JSON"，但只要跑得够久、换过模型、调过温度，就会发现这条约束并不可靠。解析一失败，整条自动化链路就断在那里。

## 问题：格式混排远比想象的多

收集了一段时间的失败样本后，坏输出基本归为四类：

1. **代码围栏包裹**：``` ```json ... ``` ```，偶尔没有语言标注，偶尔多一层缩进；
2. **自造标签包裹**：`<json>{...}</json>`、`<result>...</result>`，不同模型有不同花样；
3. **前后闲话**：开头"好的，以下是解析结果："，结尾"如需调整可以继续"，有时两头都有；
4. **内容级污染**：全角引号和全角逗号、字符串里未转义的换行、尾逗号。

麻烦在于它们会**叠加**：围栏里套标签、标签外带闲话、某个字段本身又是一段 markdown。解析逻辑只写死一种格式，命中率大概七成，剩下三成靠人工重跑。

## 做法：分层解析管线

核心思路：不要写"一个解析函数"，写一条**按侵入性从低到高尝试的策略链，并记录每层是否命中**。

```python
import json, re

def extract_json(text: str):
    raw = text.strip()
    try:                                   # S1 直接解析
        return json.loads(raw), "direct"
    except json.JSONDecodeError:
        pass
    m = re.search(r"```(?:json)?\s*(.*?)```", raw, re.S)   # S2 剥围栏
    if m:
        try: return json.loads(m.group(1).strip()), "fence"
        except json.JSONDecodeError: pass
    m = re.search(r"<(?:json|result|answer)>(.*?)</\1>", raw, re.S)  # S3 剥标签
    if m:
        try: return json.loads(m.group(1).strip()), "tag"
        except json.JSONDecodeError: pass
    cand = balanced_slice(raw)             # S4 括号配平扫描
    if cand:
        try: return json.loads(cand), "brace"
        except json.JSONDecodeError: pass
    # S5: 有限修复（尾逗号、控制字符）后重试，失败返回 None
    ...
```

其中 `balanced_slice` 不用正则，而是逐字符扫描括号配平、跳过字符串字面量内部的括号——这是唯一能稳定处理嵌套和"字段里还有 JSON"的办法。

配套两件事必须做：

- **schema 校验**：`json.loads` 成功不等于内容正确，用 zod/pydantic 再过一道结构检查，失败走重试；
- **策略命中日志**：每次解析记录 `(strategy, ok)`。哪天 `brace` 命中率突然升高，说明上游 prompt 或模型发生了漂移——这个信号比用户报障早得多。

失败时不要静默返回 None 了事：带着原始输出和校验错误发一轮修复请求（"你的输出无法解析，请原样重发 JSON，不要任何解释"），重试仍失败再降级。

## 踩坑点

- **贪婪正则是陷阱**：`\{.*\}` 在多个对象时吞错边界，非贪婪又被嵌套截断，配平扫描是唯一稳的。
- **修复要克制**：把单引号换双引号这种"聪明"修复，会在 `it's` 这类字符串上翻车。只修结构性问题（尾逗号、控制字符），引号一律不动，宁可失败重试。
- **剥围栏可能误伤内容**：JSON 字段本身含 ``` 时，非贪婪取第一个围栏块会截断，所以 S2 失败后必须还有 S4 兜底。
- **静默降级掩盖漂移**：上层兜底让系统短期很稳，长期会让你失去对模型行为变化的感知，命中率一定要进监控。
- **全角标点**是中文场景高频雷：模型偶尔输出 `，` `“”`，直接 parse 必炸。这条规则我放在修复层，且是拿真实样本验证过才加的。

## 可复用建议

1. **建失败样本库**：每类坏格式对应一条策略，规则来自真实样本，不靠想象；
2. **策略按侵入性排序**：直解 → 剥离 → 配平 → 修复，能用低成本策略就不进下一层；
3. **prompt 侧同步加固**：few-shot 给一个精确输出样例、明确"不要围栏不要解释"，但永远不假设它 100% 生效；
4. **换模型或改 prompt 后，用样本库跑回归**，看各策略命中率的变化。

## 总结

对 LLM 输出做防御性编程，本质是把"格式正确"当成概率事件而不是承诺。分层解析 + 策略命中监控 + 克制的修复，让解析层从单点脆弱变成可观测、可回归的普通组件。这套东西不复杂，但它决定你的 agent 是每三天人工救一次火，还是安静地跑一个月。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/963f7b5042e8b4ca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/a39fd6ae8b0eec47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/17d0a7987d1fcfd4.png)

