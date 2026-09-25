---
title: LLM 输出解析的防御性编程：一套应对 JSON 混合格式的分层策略
feedId: 38896
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 里写 skill 和插件，经常要让模型返回结构化 JSON——比如自动化任务中把用户指令解析成 action/params，或在 MCP 工具调用前做一轮意图抽取。提示词里明明写了"只输出 JSON，不要任何解释"，但跑一段时间就会发现：格式永远在漂移。这不是某个模型的偶发问题，而是所有依赖 LLM 输出做下游处理的链路的通病。

## 问题：真实世界里的"JSON"长什么样

同一个 prompt，不同时间拿到的输出大致有这几类：

1. 纯 JSON（理想情况，占比未必最高）
2. ```json 围栏代码块包裹
3. 自定义标签包裹，如 `<result>{...}</result>`
4. 前后带寒暄："好的，以下是解析结果："
5. 中文全角引号、逗号混入：`"key"："值"，`
6. 被 max_tokens 截断的半截 JSON
7. 嵌套 JSON 被二次转义，parse 出来还是个字符串

围栏外面再套一层标签的混合情况最常见。裸写 `JSON.parse(output)`，失败率会让你怀疑人生。

## 做法：分层提取 + 校验 + 重试

把解析做成一条流水线，而不是一个正则：

**第一步：归一化。** 去 BOM、零宽字符，全角引号/逗号/冒号转半角，trim。

**第二步：候选提取，按优先级依次尝试：** 匹配 ```json 围栏 → 匹配自定义标签（按配置枚举）→ 括号平衡扫描。扫描是底线，核心是从第一个 `{` 起维护深度计数器，跳过字符串内部（注意 `\"` 转义）：

```python
def scan_json(text):
    start = text.find('{')
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(text[start:], start):
        if esc: esc = False
        elif ch == '\\': esc = True
        elif ch == '"': in_str = not in_str
        elif not in_str:
            if ch == '{': depth += 1
            elif ch == '}':
                depth -= 1
                if depth == 0:
                    return text[start:i+1]
    raise ValueError('unbalanced')
```

**第三步：修复式解析。** parse 失败后按成本从低到高尝试：去尾逗号 → 单引号转双引号 → 截断补偿（仅对可推测的简单结构补齐缺失括号）。

**第四步：schema 校验。** 用 pydantic / zod 定义结构，校验失败等同解析失败。类型对了但枚举值不在白名单也要拦。

**第五步：带错误反馈的重试。** 把 parse 的报错拼回 prompt 再要一次，通常一轮就能修正。重试上限设 2，超过就走兜底：默认值、降级或告警。

## 踩坑点

- **贪心正则**：`\{.*\}` 在多对象输出或字符串含 `}` 时不可靠，括号平衡扫描必须保留。
- **过度修复**：把脏数据"修"成类型合法但语义错误的结果，比直接失败更危险。所有修复要有日志，原始输出必须落盘。
- **嵌套转义**：parse 一次得到的是字符串时，按类型判断是否二次 parse。
- **截断别重试**：finish_reason 是 length 时直接走降级，别浪费重试次数。
- **格式漂移**：换模型、改 prompt、调温度都会改变输出习惯。解析失败率要进监控，别等用户反馈才知道坏了。

## 可复用建议

1. 解析器全局只留一份，所有 skill 共用，别各写各的正则。
2. 能用 provider 的 structured output / tool call 就优先用，防御性解析是兜底，不是首选。
3. 每次解析记录：原始输出、命中哪条分支、是否触发修复、最终结果。这份数据是后续优化 prompt 的依据。
4. 失败兜底要写进 skill 设计：返回什么默认值、如何通知，而不是让异常往上冒。

## 总结

LLM 输出解析没有银弹。prompt 约束只能降低概率，不能提供保证。把"提取 → 修复 → 校验 → 重试 → 兜底"做成一条明确的流水线，配合原始输出留痕和失败率监控，自动化链路才能在格式漂移面前保持体面。防御性编程在这里不是悲观，是对现实成本的尊重。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/f3087b6379bd48ca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/93af39b195df474d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/722b636eb2f12660.png)

