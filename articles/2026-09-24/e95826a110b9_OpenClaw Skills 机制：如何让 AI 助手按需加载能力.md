---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38675
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

做 Agent 的人迟早会撞上同一个问题：上下文是稀缺资源。把所有工具说明、操作手册、领域知识全塞进系统提示，结果是上下文膨胀、注意力被稀释、token 账单变厚，而模型真正用到的可能不到两成。

OpenClaw 的 Skills 走的是渐进式加载（progressive disclosure）路线：启动时只有每个 skill 的 `name` 和 `description` 进入上下文，正文默认不加载；只有当模型判断当前任务与某个 skill 相关时，才把对应的 SKILL.md 正文读进来，需要时再读取捆绑的脚本和参考文件。

## 问题

思路简单，写起来容易跑偏。最常见的两类翻车：

一是把 SKILL.md 当文档写，内容详尽，但 description 写成了功能简介，模型读不出"什么时候该用它"，于是永远不触发；二是 description 太宽泛，什么都沾一点，结果处处触发、互相打架。本质是同一件事：**description 不是摘要，是路由。**

## 做法

以一个"周报生成"skill 为例：

1. **建目录。** 放在 `~/.openclaw/skills/weekly-report/` 下，核心就是 SKILL.md 一个文件。
2. **写 frontmatter。** name 用小写加连字符；description 明确写触发条件和边界，比如 "Use when 用户要求生成或汇总周报；Not for 日报或即时纪要"。中英混写没问题，关键是条件具体。
3. **正文只写操作指令。** 步骤、参数、注意事项，控制在几十行。大段参考内容（如字段规范）拆成 `reference.md`，正文里指示模型"需要时再读"。
4. **确定性逻辑交给脚本。** 日期计算、模板填充这类不需要模型发挥的步骤，写成 bundle 里的脚本，用相对路径调用，并在 SKILL.md 里声明依赖（如需要 jq）。
5. **验证触发。** 开一个全新会话，先问边界外的问题（确认不触发），再问边界内的（确认触发且正文被读入）。这步别省。

## 踩坑点

- description 写成给人看的介绍。它是给模型看的路由信号，第一句就该是触发条件。
- 多个 skill 描述重叠，触发抢占。在 description 里主动写"不适用于什么"，把边界划清。
- 正文太长。每次触发都会整段进上下文，上千 token 的正文应拆成文件按需读。
- 脚本里写死绝对路径。换台机器或进容器就断，统一用相对路径。
- 依赖不声明。skill 跑到一半发现缺工具，返工成本比提前一句声明高得多。

## 可复用建议

- 一个 skill 只做一件事，做不好就拆。
- description 按路由写：什么时候用、什么时候不用，各一句。
- 能脚本化的不要靠提示词。模型负责判断，脚本负责执行，职责分开。
- 用 git 管理 skills，每个 skill 配一条固定的 smoke-test 提问，改动后跑一遍。
- 先少后多。三五个高命中的 skill，价值远超二十个互相干扰的。

## 总结

Skills 的本质是按需注入上下文的开关。它解决的不是"能力从哪来"，而是"能力何时进入模型的视野"。写好一个 skill 不靠篇幅和数量，靠触发边界的精确——description 是路由，正文是手册，脚本是确定性。按这个分层去组织，Agent 的上下文会干净很多，命中率也稳定得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/c46b56ccb56483b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/321168c816818f8a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/eade3512f2da47a9.png)

