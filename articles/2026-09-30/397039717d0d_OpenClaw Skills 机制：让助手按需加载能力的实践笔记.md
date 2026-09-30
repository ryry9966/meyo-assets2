---
title: OpenClaw Skills 机制：让助手按需加载能力的实践笔记
feedId: 39903
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

做过 Agent 开发的人大概都有体感：工具越多，上下文越脏。接了几个 MCP server、挂了一堆插件之后，系统提示里塞满工具 schema 和使用说明，token 消耗直线上升，模型反而开始选错工具——把该搜索的事拿去跑脚本，把该读文件的事丢给浏览器。OpenClaw 的 Skills 机制就是针对这个问题设计的：能力描述与能力实现分离，启动时只加载"目录"，真正用到时才展开"正文"。

## 问题的本质

Skills 本质上是一种渐进式披露（progressive disclosure）。每个 Skill 是一个目录，核心是 `SKILL.md`：头部 frontmatter 写 `name` 和 `description`，正文写具体操作指令。Agent 运行时只把所有 skill 的 name + description 注入上下文，正文在模型判断相关时才读取。这意味着 description 不只是注释——它是路由决策的唯一依据。

## 实践步骤

1. **建目录**：`skills/` 下每个能力一个文件夹，如 `skills/pdf-extract/`，`SKILL.md` 放根目录，配套脚本和参考文档放同级。
2. **把 description 当路由写**：不要写"处理 PDF"，要写触发条件——"当用户要求从 PDF 提取表格或文本、或合并/拆分 PDF 时使用；不用于图片 OCR。" 正例和排除项都要给。
3. **正文控制在 500 行以内**：`SKILL.md` 只写主流程，细节拆到 `reference.md`，脚本写清调用方式和依赖。模型按需读取，拆得越清楚，单次加载越省。
4. **脚本路径用相对路径**：`SKILL.md` 里引用 `scripts/run.py`，脚本自身要处理好工作目录问题。
5. **验证触发**：设计一组测试问题——三个应触发的、三个不该触发的，跑一遍看路由是否准确。

## 踩坑点

- **description 模糊 = 永不触发；过宽 = 常驻加载**。我见过一个只写了"辅助数据处理"的 skill，几乎所有任务都命中，等于没做拆分。
- **两个 skill 边界重叠**时，模型会随机选一个。宁可合并，也要把边界写清。
- **别假设模型"早就知道"**——正文未加载前它确实不知道，不要在正文里引用只有系统提示才有的上下文。
- **没有版本控制**的 skill 改动，会让你排查"行为为什么变了"时无从下手。进 git，像代码一样 review。
- **脚本没加执行权限、依赖没声明**，是"触发成功但执行失败"的高频原因。

## 可复用建议

- 一个 skill 只做一类事，粒度对齐"用户意图"，而不是"技术模块"。
- description 的写法公式：动作动词 + 触发场景 + 排除项。
- 记住三层结构：`SKILL.md`（路由 + 主流程）→ `reference.md`（细节）→ `scripts/`（可执行）。
- 定期审计：让模型列出本次会话实际加载了哪些 skill、各消耗多少 token，砍掉常驻却不触发的。

## 总结

Skills 机制的价值不在于"能力多"，而在于"上下文干净"。目录常驻、正文按需，而模型的路由准确率，几乎完全取决于你如何写那两行 description。把它当成 API 设计来做——写清触发契约、控制加载粒度、版本化管理——Agent 的稳定性和 token 成本都会有可感知的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/00b48662505984c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/711da70ae5218eb3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/e1f23d2bcd8a6855.png)

