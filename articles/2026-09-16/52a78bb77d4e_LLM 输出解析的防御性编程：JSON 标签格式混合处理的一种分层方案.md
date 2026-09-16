---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理的一种分层方案
feedId: 37830
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

在 OpenClaw 的插件与 Agent 工作流里，让 LLM 输出结构化 JSON 是最常见的集成方式：工具调用参数、MCP 路由决策、自动化任务的中间结果，几乎都依赖 `JSON.parse`。但只要你跑过足够多的请求就会发现，模型并不总是守规矩——同一份 system prompt，有时返回干净的 JSON，有时包一层 markdown 围栏，有时前面加一句“好的，以下是结果”，有时在 JSON 后面补一段解释。格式漂移是常态，不是异常。

## 问题

假设输出是“纯 JSON”的解析逻辑，在格式漂移时直接抛异常，整条 pipeline 中断。更隐蔽的是“粗暴修复”：比如 `text.replace(/```/g, '')`，偶尔能跑，但遇到字符串值里含反引号、嵌套花括号的场景，会静默产出错误数据——这比直接报错更糟，因为它会污染下游。

实际线上会碰到三种典型形态：

1. 纯 JSON（理想情况）
2. ` ```json ` 围栏包裹
3. 前后夹杂自然语言，甚至混入多个 JSON 对象

## 做法

核心思路是**分层降级**：每层只做一件事，失败就交 给下一层，最后一层之前必须过 schema 校验。

1. **第一层：直接 parse。** 最快路径，多数请求到这里就结束。
2. **第二层：剥围栏。** 检测 ` ``` ` 边界，提取围栏内容再 parse。注意取“内容最像 JSON 的那一块”，而不是无脑全局替换。
3. **第三层：括号定位 + 平衡匹配。** 找到第一个 `{` 或 `[`，做括号计数扫描——关键是跳过字符串字面量内部的内容（正确处理 `\"` 转义），拿到平衡的闭合位置后截取再 parse。
4. **第四层：轻修复 + 严格校验。** 去尾逗号、修常见转义后重试；无论如何，parse 成功后必须过 schema 校验（ajv / zod 都行），结构不对照样算失败。

代码骨架（TS 简化）：

```ts
function extractJSON(raw: string): unknown {
  try { return JSON.parse(raw); } catch {}
  const fenced = extractBestFence(raw);
  if (fenced) { try { return JSON.parse(fenced); } catch {} }
  const span = balancedSpan(raw); // 括号平衡扫描，跳过字符串内部
  if (span) {
    return strictParse(raw.slice(span.start, span.end + 1));
  }
  throw new ParseError(raw); // 带原始输出上下文，供重试/告警
}
```

解析失败后，把错误信息和原始片段回传给模型重试一次，让模型自己修，成功率通常比本地硬猜更高。

## 踩坑点

- **正则全局删反引号**：字符串值里若有代码片段，合法 JSON 会被改坏。
- **括号计数不处理字符串**：遇到 `"pattern": "^\\{.*\\}$"` 这类值会提前截断。
- **一次响应多个 JSON 对象**：模型把“思考”和“结果”都输出了，取第一个未必对，用 schema 择优。
- **修复过头**：能定位到第几层失败就记下来，别把四层全叠成黑盒，排障会非常痛苦。
- **流式输出**：缓冲不完整前不要急着判定失败，等收尾再走完整解析链。

## 可复用建议

- 把 `extractJSON` 沉淀成公共工具模块，所有插件统一引用，别各写各的。
- 原始输出永远先落日志（截断即可），线上排障基本全靠它。
- 攒一个“脏输出测试集”，每遇到一次新的解析失败就加一条用例，回归必跑。
- 能用 API 级结构化输出 / json mode 就优先用，防御性解析是兜底，不是首选。
- 结构化任务把 temperature 压低，能显著减少格式漂移的发生频率。

## 总结

防御性解析的本质不是“更聪明的正则”，而是**分层降级 + 严格校验 + 失败可回溯**。把解析做成无状态工具函数，把失败做成带上下文的信号，Agent 链路的稳定性会有肉眼可见的提升。这套方案改动量不大，建议在新插件接入前就先铺好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/1448b967ef8ef04e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/047358f276d3ad43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/ef8ea60585cf4c3e.png)

