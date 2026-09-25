---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38917
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 这类长期运行的 Agent 网关，能力来源大致分三层：模型自身的推理、MCP/CLI 等工具、以及 Skills。前两层决定"能做什么"，Skills 决定"什么时候、按什么流程做"。

一个 Skill 本质上是一个文件夹：一份带 frontmatter 元数据的 `SKILL.md`，加上可选的脚本和资源文件。它不注册新的 API，只是给模型提供一份可读的操作手册。

## 问题

最直觉的做法是把所有能力说明全部塞进 system prompt。能力少时没问题，攒到十几个之后会出三件事：

- 常驻上下文 token 明显上涨，留给实际对话的空间变小；
- 注意力被稀释，该触发的流程没触发、不该用的被误用；
- 多份手册之间有冲突时，模型行为不稳定。

Skills 机制的核心思路，就是把这堆手册从"全量常驻"改成"索引常驻、正文按需"。

## 做法与步骤

**1. 搭目录结构。** workspace 下建 `skills/` 目录，一个能力一个文件夹：

```text
skills/
  weekly-report/
    SKILL.md
    scripts/render.py
```

**2. 写好 frontmatter。** 启动时 OpenClaw 只把每个 skill 的 `name` 和 `description` 注入索引，正文不进上下文：

```yaml
---
name: weekly-report
description: 当用户要求汇总本周消息或任务并生成周报时使用。先汇总，再运行 scripts/render.py 渲染。
---
```

**3. 把 description 当检索词写。** 用户嘴里会说出的动词、名词都要覆盖；写"何时用"，不要写"这是什么"。

**4. 正文只写动作。** `SKILL.md` 用祈使句列步骤，能脚本化的步骤直接指向脚本，让模型用执行工具去跑，而不是自己逐句推理。

**5. 验证闭环。** 跑一个典型请求，看日志确认三段都在：索引匹配 → 读取 `SKILL.md` → 执行步骤。缺任何一段都不算生效。

## 踩坑点

- **description 含糊是最常见的失效原因。** 只写"处理报告"的 skill，既不会被"整理周报"触发，也可能在用户随口提到"报告"时抢触发。
- **正文超过两三百行就该拆。** 加载是全量的，正文越长，单次触发的上下文成本越高。
- **脚本依赖要前置声明。** 容器里缺某个 CLI 时，模型会反复重试、浪费轮次；在正文开头写一句前置检查能省很多麻烦。
- **描述高度重叠的 skill 会互相抢触发**，宁可合并也不要堆叠。

## 可复用建议

- 高频重复的流程（周报、部署前检查、某类数据清洗）优先沉淀成 skill，而不是每次手贴 prompt。
- workspace skills 进版本管理，改动走提交记录，回滚有据可查。
- 定期看加载日志：长期没触发过的 skill，删掉或降级为普通文档。
- 职责分层：通用工具能力交给 MCP，流程性知识交给 Skills，两层别混着写。

## 总结

Skills 机制没有黑魔法，它只是把"手册全背下来"换成了"先看目录、用哪本翻哪本"。收益来自两处：上下文成本随能力数量增长变慢，触发准确率随 description 质量上升。把 description 当检索关键词来持续维护，是这套机制能不能用好的关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/0a8767d1c3d3ca50.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/904ad1d544f34825.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/940cf047dcdebe1b.png)

