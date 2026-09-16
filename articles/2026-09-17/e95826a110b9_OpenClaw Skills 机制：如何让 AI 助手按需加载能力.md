---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37907
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

接了一堆 MCP server 和插件之后，Agent 的能力清单会越拉越长。OpenClaw 的 Skills 机制就是为了解决这个问题：把每类能力封装成独立目录，靠 SKILL.md 描述"这是什么、什么时候用、怎么用"，并且不是全部常驻上下文，而是按需注入。

## 问题：上下文塞满不等于能力变强

把所有工具说明一次性塞进系统提示词，会遇到三个实际麻烦：

1. **Token 固定开销大**。几十个能力的完整说明常驻上下文，每轮对话白烧几万 token；
2. **选择精度下降**。说明越长越相似，模型在相近能力之间误选的概率反而升高；
3. **维护成本线性增长**。每加一个能力都要改系统提示词，改一处容易碰坏别处。

## Skills 的分层加载

Skills 采用渐进式披露，大致三层：

- **常驻层**：每个 Skill 只有 name + description 进入上下文，单条几十 token，模型据此判断要不要调用；
- **触发层**：命中后，对应 SKILL.md 的完整正文才被注入；
- **资源层**：正文里引用的脚本、参考文档，真正执行时才读取。

也就是说，平时挂 50 个 Skill，可能只占 2000 token 左右的目录开销，真正干活的说明只在需要时进场。

## 实操步骤

1. 在 skills 目录下建文件夹，创建 SKILL.md：

```markdown
---
name: pdf-report
description: Use when the user asks to generate, merge, or extract data from PDF files.
---

# PDF Report

1. 批量合并优先调用 scripts/merge.py
2. 输出前确认页码范围与用户一致
```

2. description 写成触发条件，不是功能介绍；
3. 正文写关键步骤，能落到脚本的就引用脚本路径，别堆长文；
4. 重载后实测：问一句相关问题，确认是否命中；
5. 看注入日志，确认常驻层开销符合预期。

## 踩坑点

- description 写成广告词（"强大的 PDF 处理工具"），模型基本不触发；改成 "Use when..." 句式后命中率明显提升；
- 描述太宽泛会误触发，比如任何提到 PDF 的对话都拉起技能，浪费 token 还带偏上下文；
- 正文里写死本机绝对路径，换台机器就失效，尽量用相对 skills 目录的路径；
- 把需要确定性执行的事写成 Skill。Skill 是提示词级注入，适合流程性知识；确定性逻辑应下沉到脚本或 MCP 工具；
- 两个 Skill 职责重叠时，模型选择接近随机，要主动合并或划清边界；
- Skill 注入不等于沙箱，涉及敏感操作仍要走审批流程。

## 可复用建议

- 一个 Skill 只解决一类任务，宁可拆小；
- description 模板：`Use when <具体场景>. Handles <产出物>.`；
- 正文控制在几百 token 内，细节外置到引用文件，吃满第三层；
- 把重复出现的 SOP 沉淀成 Skill，而不是反复改系统提示词；
- 定期查触发日志，清理从未命中或频繁误触发的技能。

## 总结

Skills 的价值不在于"能挂多少能力"，而在于把进入上下文的信息量控住：目录常驻、正文触发、资源执行时加载。写好一条 description，比堆十个技能更能决定 Agent 的实际表现。先把触发条件写准，再把流程沉淀进正文——上下文干净了，能力反而更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/ae6e2148ecc1f4d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/7c7211574c64b772.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/0247136d28906540.png)

