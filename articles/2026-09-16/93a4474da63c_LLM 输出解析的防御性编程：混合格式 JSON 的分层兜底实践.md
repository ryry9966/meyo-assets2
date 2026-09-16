---
title: LLM 输出解析的防御性编程：混合格式 JSON 的分层兜底实践
feedId: 37826
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

在 OpenClaw 的插件与 MCP 工具链里，很多环节都依赖模型输出结构化 JSON：工具参数、路由决策、批处理结果汇总。即便开了 JSON mode 或结构化输出，真实链路中仍会碰到围栏包裹、说明文字混杂、多段输出这类脏数据。提示词约束只能降低概率，解析端必须有兜底能力——这就是防御性解析的定位。

## 问题

高频出现的三类脏输出：

1. **围栏包裹**：` ```json ` 围栏，大小写、嵌套反引号各种变体；
2. **前后夹带说明**："以下是解析结果："、"希望对你有帮助"这类自然语言前后缀；
3. **格式瑕疵**：尾逗号、单引号、输出截断、一次返回多个 JSON 对象。

任何一类都会让 `json.loads` 直接抛异常，导致整个 Agent 步骤失败。

## 做法：六层解析管道

思路是把容错拆成有序的层级，逐层降级，而不是写一个巨型正则：

- **L0 归一化**：去掉 BOM、零宽字符、首尾空白。只处理不可见字符，不动业务字符串内容。
- **L1 剥围栏**：大小写不敏感地匹配 ` ```...``` `，取围栏内内容。
- **L2 平衡提取**：不要用正则抓 `{...}`，用括号计数扫描第一个平衡块，且要处理字符串内部的花括号和转义：

```python
def extract_json_block(s: str):
    start = s.find('{')
    if start < 0:
        return None
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(s[start:], start):
        if in_str:
            if esc: esc = False
            elif ch == '\\': esc = True
            elif ch == '"': in_str = False
        else:
            if ch == '"': in_str = True
            elif ch == '{': depth += 1
            elif ch == '}':
                depth -= 1
                if depth == 0:
                    return s[start:i + 1]
    return None  # 括号不平衡，大概率截断
```

- **L3 解析**：`json.loads`，严格场景用 `parse_constant` 拦截 `NaN/Infinity`。
- **L4 校验**：pydantic / jsonschema 过一遍 schema，类型和必填字段在这里把关。
- **L5 修复兜底**：前四层失败再上 json-repair 类库，触发时必须打日志。
- **L6 反馈重试**：把解析错误拼回提示词让模型自纠，限一次，避免循环烧 token。

## 踩坑点

- `\{.*\}` 贪婪会吞掉多个对象，非贪婪会在嵌套处提前截断，正则抓 JSON 基本不可救；
- 全局替换全角标点是常见错误做法——用户内容里的中文逗号会被一起改掉，污染数据。结构修复交给修复库并记录触发；
- 别只信 `response_format`：不同模型、不同调用路径（尤其工具结果回传）行为不一致，解析端照旧兜底；
- 修复库可能悄悄改类型（字符串变数字），下游逻辑敏感时要校验后再消费；
- 流式输出别套这套完整管道，要么攒完整再解析，要么用增量解析器。

## 可复用建议

- 把 L0–L6 做成共享 util，插件不要各抄一份，否则修复策略漂移后很难排查；
- 失败分类打点：`fence` / `prose` / `truncated` / `repair` / `invalid_schema`，用分布数据指导提示词迭代；
- 真实坏样本沉淀成 golden test 固定用例，改提示词或升级模型后跑回归；
- 原始输出留存（注意脱敏），坏 case 才能复盘。

## 总结

提示词负责降低脏输出概率，解析端负责兜住剩余概率，两者叠加才有稳定的结构化链路。防御性解析不是不信任模型，而是把输出侧的不确定性收敛到可观测、可回归的工程范围内。这套管道在我们的插件链路上跑了一段时间后，解析失败率明显下降，剩余问题基本都能通过打点定位到具体层级，欢迎在社区帖里交流你们的失败分类维度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/566fdd3b0b0e31b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/38c3eecd8aae095c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/6588715e3d119e6b.png)

