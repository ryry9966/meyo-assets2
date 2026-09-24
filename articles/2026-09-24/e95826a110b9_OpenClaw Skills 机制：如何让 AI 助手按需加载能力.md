---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38781
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 agent 常驻运行，要同时应付写脚本、管设备、跑自动化这些杂事。早期的直觉做法是把所有操作说明堆进 system prompt 或 AGENTS.md，结果就是上下文越写越长：token 成本上升，模型对真正相关那条指令的注意力反而被稀释。Skills 机制就是针对这个问题设计的：能力以独立模块存在，agent 平时只看到「目录」，需要时才翻开「正文」。

## 问题

没有 skills 时常见的三个症状：

1. **prompt 膨胀**。几十条操作规范全量注入，每轮对话都在为它们付费。
2. **能力不可见**。你配好的某个流程，agent 在该用的时候想不起来，因为它淹没在长上下文里。
3. **规则互相打架**。两套处理 PDF 的说明同时生效，模型随机选一个，行为不稳定。

核心矛盾是：能力要「在场」，但不该「常驻」。

## 做法：三个层级，按需展开

一个 skill 就是一个目录，核心是 `SKILL.md`：

```
my-skill/
├── SKILL.md        # 入口：frontmatter + 指导正文
├── scripts/        # 可选：确定性脚本
└── references/     # 可选：细节文档，按需再读
```

加载分三步（渐进式披露）：

1. **会话启动时**，只把每个 skill 的 `name` 和 `description` 注入上下文——这一层是「货架标签」，几十个 skill 也只占几百 token。
2. **agent 判断任务匹配**后，读取 `SKILL.md` 全文，拿到具体操作步骤。
3. **正文里引用的 scripts / references**，agent 再按需去读、去执行。

实操步骤：

1. 在 `~/.openclaw/skills/` 或 workspace 的 `skills/` 下建目录；
2. 写 `SKILL.md`：name 用小写连字符，description 一句话写清「什么场景用、能做什么」；
3. 正文控制在两百行以内，长细节拆到 `references/`，可预测的步骤写成 `scripts/` 里的脚本；
4. 用 `/skills` 查看已装载清单，确认 description 被正确解析；
5. 改动后重启 gateway 生效。

description 写法对比：

- 差：`处理文件的工具`
- 好：`Use when the user asks to batch-convert or merge PDF files. Handles splitting, OCR and format conversion via python scripts.`

## 踩坑点

- **description 太泛或太窄**。太泛会误触发（什么任务都加载它），太窄永远不触发。写触发场景，不要写实现。
- **正文写成论文**。SKILL.md 超过几百行，「按需加载」就失去意义，该拆就拆。
- **脚本依赖没声明**。skill 里的 Python 脚本缺包，agent 会原地排错很久。在正文开头加一段环境检查。
- **多个 skill 描述重叠**。两个 skill 都声称管「图片处理」，agent 选择近乎随机。在 description 里划清边界，比如一个只管压缩、一个只管 OCR。
- **忘了重启**。skills 目录变更不会热加载到常驻的 gateway，改完不重启，容易怀疑人生。

## 可复用建议

- **把重复 SOP 沉淀成 skill**。同一个流程口头教 agent 超过三次，就值得写成 SKILL.md。
- **确定性逻辑进脚本，判断逻辑进正文**。LLM 负责决策，脚本负责执行，别让模型现场推每一步命令。
- **skills 目录进 git**，像 review 代码一样 review 每次对 description 的改动。
- **新 skill 先小范围验证**：开一个隔离会话，用典型任务问一遍，确认它在正确的时机被加载、内容被完整执行。

## 总结

Skills 本质上是把提示词工程模块化：元数据常驻保证 agent「知道自己会什么」，正文按需加载保证上下文保持轻量。一句话 description 的质量，比十行正文更决定这个 skill 能不能被真正用起来。把你在 OpenClaw 里重复调教过三次以上的流程抽成 skill，是性价比最高的下一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/74d0e2090d52e3bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/c16ad25c815edd0d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/313e8e5ab85a667a.png)

