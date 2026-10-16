---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41732
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 用得越久，攒下来的能力越多：日报生成、内网巡检、播客转写、发布检查……最早的方案是把所有操作手册塞进系统提示词，结果上下文越滚越长，单轮 token 消耗上涨，模型选错工具的概率反而变高。

Skills 机制给出的解法是**渐进式披露**：常驻上下文里只保留每条技能的 `name` 和 `description`，正文（步骤、脚本路径、注意事项）在模型判断当前任务相关时才加载。相当于把能力从"全部随身携带"改成"建一个可检索的仓库"。

## 问题

实际跑下来有两类反面模式：

1. **手册全量进提示词。** AGENTS.md 写到几百行，任何一轮对话都要为其买单，且注意力被稀释。
2. **万物皆 MCP。** 很多能力并不需要新工具，只是一套操作指引——把"说明文"包装成工具，反而让工具列表变得难选。

## 做法 / 步骤

1. **建 skill**：在 workspace 的 `skills/` 目录下为每个能力建文件夹，核心是 `SKILL.md`，frontmatter 写 `name` 和 `description`，正文写操作步骤。
2. **把 description 当触发面写**：说清"什么场景用、输入是什么、依赖哪些命令"，一两句即可。这是模型唯一常驻可见的部分，决定技能会不会被选中。
3. **声明依赖**：在 `metadata.openclaw.requires` 里写清 `bins`、`env`、`config`，环境不满足的技能不进候选池，避免模型调用注定失败的东西。
4. **区分常驻与按需**：核心工作流可加 `always: true` 常驻，其余一律按需，常驻数量控制在两三条以内。
5. **显式验证再放手**：新 skill 先用 `/skill <name>` 手动加载，确认正文和脚本路径无误，再让模型自动触发。
6. **脚本外置**：脚本放 skill 目录里，正文只写路径和调用方式，不要把代码内联进 SKILL.md。

## 踩坑点

- **description 太笼统 → 永远不触发；关键词堆太多 → 什么场景都抢着触发。** 写完拿真实任务测：直接问 agent"这个问题你会用哪个 skill"，观察它的判断依据。
- **技能静默消失。** `requires.env` 配了但变量没注入，技能会从候选中隐藏，表现为"写好的 skill 找不到了"。先用 `/skills list` 确认可见性再排查。
- **改完不生效。** SKILL.md 的修改通常要新开会话（或重载）才会被读到。
- **三层同名覆盖。** workspace、managed、bundled 三个位置可能出现同名 skill，优先级覆盖会让"我明明改了"变成排查黑洞，先确认命中的是哪一份。
- **别把 MCP 和 Skills 对立。** MCP 适合结构化工具调用，Skills 适合沉淀流程经验；skill 正文里完全可以指导模型去调某个 MCP 工具，两者是互补关系。

## 可复用建议

- 一个 skill 只做一件事，正文超过两三百行就该拆。
- workspace skills 用 git 管理，改动可回溯、可同步。
- 定期审计：把所有 description 拉出来读一遍，删掉长期没触发过的。
- 团队共用的 skill，写清前置条件比写清步骤更重要——前置不满足的技能宁可不出现。

## 总结

Skills 的价值不在"多"，在"按需"。把常驻上下文当预算花：description 是货架标签，正文是仓库里的货物，`requires` 是入库质检。写 skill 的本质，是给未来的自己和团队写一份能被 agent 主动检索到的 runbook——这份投入会在每一次自动触发里还回来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/dbe24b88be4e9d9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/5ad97386c6101ab1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/1553945292c6a425.png)

