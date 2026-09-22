---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38516
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

OpenClaw 的 Skills 是一种把"能力"从常驻上下文里拆出去的机制。一个 Skill 本质上是一个目录，核心是带 YAML frontmatter 的 SKILL.md：`name` 和 `description` 常驻系统提示，正文、参考文档、脚本全部懒加载。这是典型的渐进式披露设计——模型先看到目录，需要时再翻详情页。

## 问题

不拆的代价很直接。我们在一个接了七八个 MCP server 的实例上做过对比：全部工具定义注入后，每次请求固定多出上万 token；更糟的是，工具多了之后模型选错工具的概率明显上升——路由靠语义匹配，候选越多越容易撞车。把稳定流程固化成 Skill 之后，模型只需判断"该不该启用某个能力"，而不是在几十个 function schema 里做选择题。

## 做法

1. **建目录**：一个 Skill 一个目录，放 SKILL.md，可附 `scripts/` 与 `references/`。
2. **写 frontmatter**：`name` 用短横线命名；`description` 写成触发条件，"Use when the user asks to ..."；正文写操作步骤、参数说明、边界情况。
3. **确定性步骤下沉到脚本**：SKILL.md 只写"何时调用、如何传参"，具体执行交给脚本，避免模型每步重新发挥。
4. **长资料外置**：大段参考内容拆到 `references/`，正文以相对路径引用，模型按需读取，不进初始上下文。
5. **验证**：启动后问模型"你现在有哪些能力"，确认只有元数据被加载；再跑一次实际任务，确认正文是按需进入上下文的。

## 踩坑点

- **description 写成功能清单而非触发条件**，模型永远想不起来用。改成"when to use"句式，把用户可能说出的关键词写进去。
- **Skill 之间职责重叠**，description 互相抢触发。一个 Skill 只解决一类问题，宁可拆细。
- **把所有内容塞进正文**，等于换个地方污染上下文。正文控制在几百行内，长资料外置。
- **前置条件没声明**：脚本依赖的 CLI、环境变量，必须在正文开头写清检查方式，否则模型跑到一半才报错。
- **脚本里硬编码密钥**：Skill 目录会进版本库，凭据一律走环境变量或 secret 管理。

## 可复用建议

- 把"每次都要口头交代一遍"的重复流程做成 Skill，这是收益最高的一类。
- description 当检索 query 写，别当广告词写。
- 用 git 管理技能目录，改动可回溯；团队共享前先在隔离实例验证。
- 定期审查触发记录，长期不被触发的 Skill 要么重写 description，要么删掉。

## 总结

Skills 的价值不在"多"，在"准"：常驻的只有一行元数据，加载的永远是当下需要的那一份。写好一个 Skill，一半工作量在 description 的触发条件上，另一半在把模糊流程改写成可执行步骤。建议先从一两个高频流程试起，比一次搬几十个脚本有效得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/73f7935a34475855.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/8b2c53053491c0a3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/eafd8b3e8855db57.png)

