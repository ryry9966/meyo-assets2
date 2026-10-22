---
title: LLM 输出解析的防御性编程：当 JSON 标签格式开始“混搭”
feedId: 42279
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

在 OpenClaw 这类 agent / MCP / 插件体系里，LLM 的结构化输出是工具调用的粘合剂。我们与模型约定：输出用 ```json 围栏包裹的 JSON。但翻了一周的线上日志后发现，格式漂移比想象中严重得多——同一个 prompt，模型能给出至少六种形态。

## 问题

实际遇到的输出形态：

1. 纯 JSON，理想情况；
2. ` ```json ... ``` ` 围栏包裹；
3. `<json>...</json>` 这类 XML 风格标签包裹；
4. JSON 前后带一段“好的，以下是结果：”之类的说明文字；
5. 一次输出里出现两个 JSON 对象；
6. JSON 里带注释、尾逗号、智能引号（`"` 而不是 `"`）。

任何一层只按约定格式写解析，上线第一周就会开始报警。

## 做法：分层解析管线

核心思路：prompt 约定保留，但只当第一道防线，解析器按“从最严格到最宽松”逐层降级。

```python
import json, re

def parse_llm_json(text: str):
    candidates = []
    candidates.append(text.strip())                                   # L0 直接解析
    candidates += re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)  # L1 剥围栏
    candidates += re.findall(r"<(?:json|result)>(.*?)</(?:json|result)>",
                             text, re.S)                              # L2 剥标签
    candidates.append(_balanced_scan(text))                           # L3 配平扫描
    for c in candidates:
        if not c:
            continue
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    return None

def _balanced_scan(text: str):
    start = text.find("{")
    if start < 0:
        return None
    depth, in_str, esc = 0, False, False
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        else:
            if ch == '"': in_str = True
            elif ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    return text[start:i + 1]
    return None
```

之后固定三步：

1. **Schema 校验**：pydantic 过一遍，字段类型不对等同解析失败；
2. **有限重试**：失败后把具体解析错误（不是“重试一次”）拼进 prompt 再要一次，封顶一轮；
3. **全量落日志**：无论成败，原始输出都存下来。

## 踩坑点

- **正则贪婪匹配**会把两个 JSON 对象粘成一个，务必用非贪婪并收集全部候选。
- **配平扫描必须感知字符串**：字符串值里的 `}` 和被转义的 `\"` 不处理，一定截错位置。
- `json.loads` 默认接受 `NaN`/`Infinity`，下游序列化会炸，用 `parse_constant` 钩子拦掉。
- 修复类库（如 json_repair）好用，但它在“猜”，猜错会静默改变语义。关键链路修复后必须过 schema 校验，别裸信。
- 候选的尝试顺序是个策略问题：标签和围栏同时出现时谁优先，建议实测自己日志里的真实分布再定。
- 重试别无限循环，也别只说“再试一次”——要带上具体错误，否则模型大概率原样再来一遍。

## 可复用建议

- 解析器收敛成一个公共模块，所有插件和工具调用共用，杜绝各写各的。
- 把线上解析失败的原文脱敏后存进 golden 测试集，每次改动解析器跑一遍回归——这是最有价值的资产。
- 给解析器加版本号写进日志，出问题能回溯“当时用的是哪版逻辑”。
- 盯三个指标：L0 直接命中率、各降级层命中率、修复后校验失败率。模型或 prompt 一变，这些数字最先漂移。

## 总结

Prompt 是约定，解析器才是契约。分层降级、schema 校验、有限重试、全量落日志，四件事做齐之后，结构化输出这条链路的稳定性会有数量级的改善。防御性解析不是不信任模型，而是承认输出天然是概率性的——工程要接住这个概率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/bdc751ac2a75cb4a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/953e8a7ca38dcb96.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/2518b3a7a7fa2dd4.png)

