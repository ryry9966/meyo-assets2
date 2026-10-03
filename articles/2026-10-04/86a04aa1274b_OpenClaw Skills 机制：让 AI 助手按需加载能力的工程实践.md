---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 40336
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

跑长任务的 Agent，瓶颈往往不是模型能力，而是上下文预算。早期常见做法是把所有工具说明、工作流文档一股脑塞进 system prompt，代价很明显：token 成本高、指令互相干扰、模型选错工具的概率上升。

OpenClaw 的 Skills 机制就是针对这个问题的：能力模块化，按需加载。核心是"渐进式披露"的三层结构：

1. **元数据层**：所有 skill 的 `name` + `description` 常驻上下文，单个只占几十 token；
2. **指令层**：命中触发条件时，才加载 SKILL.md 正文；
3. **资源层**：正文引用的参考文档、脚本，真正用到才读。

## 问题：skill 为什么"不触发"或"乱触发"

实际用下来，最常见的不是写不出 skill，而是加载时机不对：

- description 写得太泛（比如"处理数据"），模型判断不了何时该用，最终永远不加载；
- 两个 skill 职责重叠，模型随机选一个，输出风格时好时坏；
- 所有细节都堆进 SKILL.md 正文，一旦触发就整篇进上下文，等于变相回到全量加载。

## 做法：五步把 skill 调到稳定触发

1. **建文件**。在 skills 目录下建子目录，写 SKILL.md，frontmatter 只留 `name` 和 `description`：

```markdown
---
name: pdf-report
description: 当用户要求把结构化数据整理成带图表的 PDF 报告时使用
---
```

2. **description 按"触发条件"写**，模板：用 X 完成 Y，当用户提出 Z 类请求时使用。它是模型的路由键，不是产品简介。
3. **正文控制在 100 行以内**，只放决策规则和步骤骨架；细节下沉到 `references/*.md`，让模型按需读取。
4. **确定性逻辑写成脚本**放在 skill 目录里，正文只写"何时调用、如何传参、失败怎么办"，不要让模型现场推理这些步骤。
5. **用 10 个左右真实 prompt 做触发测试**，记录哪些命中、哪些误触，回头改 description，而不是改正文。

## 踩坑点

- description 里堆"强大的""智能的"这类形容词没有信息量，等于没写；
- skill 超过 30 个后，元数据列表本身开始挤压上下文，优先合并同类；
- 脚本含写操作（删文件、发请求）时，务必在正文写清确认步骤，自动化场景容易出事故；
- 同名 skill 同时出现在工作区和个人目录时，加载优先级容易被忽略——改动前先确认实际生效的是哪一份；
- 改完 description 别在旧会话里验证，开个新会话再测，否则看到的是残留上下文的效果。

## 可复用建议

- 一个 skill 只做一件事。模型对"小而准"的响应远好于"大而全"，想拆就拆；
- 把 description 当 API 文档的 summary 维护，每次修改后跑一遍触发测试集；
- skill 目录用 git 管理，改动走 PR——出问题时能回溯是哪次提交导致触发漂移；
- 脚本要幂等、有明确退出码和 stderr 输出，模型遇到失败才能自我纠正。

## 总结

Skills 机制的本质，是把"该在什么时候知道什么"变成一个工程问题：元数据常驻、正文按需、资源最深。实践中多数 skill 表现不好，根源不在模型，而在 description 路由信息不足和正文边界不清。把触发条件写准、细节下沉、确定性逻辑交给脚本，按需加载才会真正省下上下文，而不是换个地方堆 token。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/d656151c95ae7fef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/8bbf6d21028b1ee3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/74ae67d11f209255.png)

