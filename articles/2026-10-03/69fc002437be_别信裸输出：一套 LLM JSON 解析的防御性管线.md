---
title: 别信裸输出：一套 LLM JSON 解析的防御性管线
feedId: 40176
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

在 OpenClaw 的日常开发里，“让模型只输出 JSON”几乎无处不在：MCP 工具的入参约定、子 agent 之间的结构化通信、自动化流水线的中间结果。Prompt 里写一句“仅输出 JSON，不要代码块”当然必要，但实践反复证明它不可靠。防御性编程的立场很简单：**把模型输出当作不可信的外部输入，解析层必须兜住所有形态。**

## 问题：三类常见污染

线上积累下来，脏输出大致三类：

1. **栅栏包裹**：` ```json ... ``` `，偶见 ` ```js ` 甚至无语言标记；
2. **前后缀散文**：“好的，以下是结果：{...}如需调整请告诉我”；
3. **语法瑕疵**：中文引号、尾逗号、BOM、未转义换行、流式场景下的截断半截 JSON。

最阴险的是嵌套栅栏——JSON 字符串值里本身含 ` ``` `，朴素正则会从中间断开。

## 做法：四层降级管线

解析收敛成一个 `parse_llm_json()`，内部四层，逐层降级：

**第一层 规范化**：去 BOM、trim、去掉可能的 "JSON:" 前缀。

**第二层 括号配对提取**：不用正则，用状态机找到第一个 `{` 或 `[`，扫描时跟踪“是否在字符串内、是否转义”，配对出完整片段：

```python
def extract_json(text: str) -> str:
    start = min(x for x in (text.find("{"), text.find("[")) if x != -1)
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(text[start:], start):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"':
            in_str = True
        elif ch in "{[":
            depth += 1
        elif ch in "}]":
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    raise ValueError("unbalanced json")
```

这一层同时解决散文前后缀和嵌套栅栏两个问题，是整条管线的性价比之王。

**第三层 修复**：交给 `json-repair` 一类库，处理尾逗号、单引号、裸键。原则：只修“明确无害”的语法问题。

**第四层 校验**：pydantic / jsonschema 按业务 schema 验字段、验类型。解析成功不等于数据可用，这层才是最终把关。

四层都失败再重试：把具体解析错误喂回模型——“上次输出无法解析，第 3 行有尾逗号，请只输出 JSON”。错误越具体，重试成功率越高，通常一到两次就收敛。

## 踩坑点

- **正则偷懒**：` ```json(.*?)``` ` 遇嵌套栅栏会静默截断且不报错，污染下游数据。要么不用，要么配异常路径。
- **修复过猛**：把 `"3.14"` 擅自修成 `3.14`、单引号串强行转双引号，都可能改变语义。修复层只管语法，语义交给 schema。
- **取第一个 `{` 未必对**：有的模型输出两段 JSON，或先数组后对象，先看整体结构再提取。
- **流式截断**：流式拿到的可能是半截 JSON，配对必然失败，要么等完成信号，要么单独走增量解析。
- **静默降级是反模式**：每层都要打点，否则你不知道线上 80% 的请求死在第几层，prompt 永远改不对地方。

## 可复用建议

1. 全项目只用一个 `parse_llm_json()`，别让每个插件自己写正则；工具函数集中维护、带版本。
2. 打点分类统计：栅栏 / 散文 / 修复 / 重试各占多少。若栅栏占大头，往 prompt 里补一个 few-shot 输出示例，比改代码见效快。
3. 运行时支持结构化输出（function calling / JSON mode / response_format）就优先用，但解析兜底照留——不同模型接 MCP 时行为差异不小。
4. 失败样本落盘保留原文，排查和回归测试都用得上。

## 总结

防御性解析的目标不是把单次成功率顶到 100%，而是让失败**可观测、可降级、可重试**。四层管线加 schema 校验加打点，核心代码不到百行，能消掉绝大多数线上解析事故。模型会一直给你惊喜，解析层的工作就是让惊喜只停留在日志里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/b6f52c60a478d52a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/010cca00271980a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5fcab71fa2a5a3c1.png)

