---
title: OpenClaw Skills 机制实践：让 AI 助手按需加载能力
feedId: 39902
source: 综合讨论
publishedAt: 2026-09-30
---

# OpenClaw Skills 机制实践：让 AI 助手按需加载能力

## 背景

OpenClaw 的 Skills 本质是一套“渐进式披露”的能力加载机制。每个 Skill 是一个目录，核心是一个 `SKILL.md`：frontmatter 写 `name` 和 `description`，正文写具体操作指引。会话启动时，系统只把各 skill 的名称与描述注入上下文；模型判断当前任务命中某条描述后，才去读取完整正文。token 花在"索引”上，正文按需进入上下文。

## 问题

不用 skills 时，常见的两种困境：

- 把操作手册、团队规范全塞进系统提示词，动辄上万 token，注意力被稀释，指令遵循反而变差；
- MCP 工具全量挂载，几十个工具描述常驻，模型选错工具的概率明显上升，成本也高。

本质是上下文经济学：常驻内容越多，单条指令的权重越低。

## 做法

1. **建目录**：在 `~/.openclaw/skills/<skill-name>/SKILL.md` 创建文件（工作区 `skills/` 目录优先级更高，适合项目级定制），目录名与 `name` 保持一致。
2. **写 description**：这是触发契约，写清“做什么 + 何时用”。例如：“生成周报草稿时使用。输入为 git 提交记录与 issue 列表，输出 Markdown。”不要写成功能宣传文案。
3. **声明依赖**：依赖某个 CLI 或环境变量时，在 frontmatter 的 `requires` 中声明，缺依赖的 skill 不会误启用。
4. **写正文**：只放步骤、命令、边界条件，建议 100 行以内。更长的内容拆成子文档，正文只留索引，让模型按需再读。
5. **验证**：新开 session 执行 `openclaw skills list` 确认加载，再问一句“你有哪些 skill、什么场景用"，最后用真实任务跑一遍，观察触发时机是否准确。
6. **配合 MCP**：skill 正文可以直接编排——“先调用 mcp 的某工具，失败则回退到 shell 命令”。skill 充当编排层，工具按需调用。

## 踩坑点

- description 写成“这是一个周报工具”而非触发条件，模型要么不触发、要么乱触发。
- 正文贪长，一个 `SKILL.md` 写了五百行，等于把老问题搬进新机制。
- name 与目录名不一致，或两个 skill 重名，后者被静默覆盖。
- 正文写死绝对路径、假设特定 shell 环境，换台机器就失效。
- skill 之间隐式依赖（"先加载 A 才能用 B"），机制并不保证加载顺序。
- 积累几十个 skill 后，元数据本身又开始膨胀，需要定期合并与下线。

## 可复用建议

- 用统一模板：一句话职责 / 输入 / 输出 / 前置条件 / 步骤 / 禁止事项。
- 把团队 SOP 沉淀成 skills 进 git，评审走 PR，description 的改动单独标出重点看。
- 定期做“触发审计”：让 agent 回答“本次用了哪个 skill、为什么”，命中率低的要么改描述、要么删。
- 少而准优于多而全：一个高频场景配一个精准 skill，比十个泛化 skill 有效得多。

## 总结

Skills 机制的核心不是“给 agent 堆能力”，而是控制上下文里常驻什么、按需加载什么。description 是模型与 skill 之间的契约，写它的认真程度直接决定触发质量。建议先从一两个真实高频场景做起，跑通“触发—加载—验证”闭环，再逐步把团队经验沉淀成库。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/095713be55e5a7d8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/a0415c51bea1460c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/2e71f7cfa0749dc7.png)

