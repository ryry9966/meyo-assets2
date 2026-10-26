---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42670
source: 综合讨论
publishedAt: 2026-10-26
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景

Agent 用久了都会遇到同一个问题：能力越接越多，MCP server 一挂就是十几个工具，system prompt 里堆满了各种操作说明。结果是上下文被大量无关内容占满，模型选工具时反而更容易犹豫和出错，token 成本也水涨船高。OpenClaw 的 Skills 机制就是针对这个问题的工程化解法：把能力拆成独立的技能包，平时只暴露名字和一句描述，真正被需要时才加载正文。

## 问题

具体痛点有三个：

1. **全量注入**：所有工具说明常驻 prompt，长会话的有效上下文被严重挤压；
2. **选择噪音**：几十条工具描述摆在面前，模型容易选错、重复确认或幻觉出用法；
3. **维护困难**：所有能力挤在一个巨型 prompt 文件里，改一处怕影响全局，无法独立迭代。

## 三级渐进加载

Skills 的核心是分级披露：

- **Level 1**：仅 `name + description` 常驻 system prompt，几十 token；
- **Level 2**：模型判断命中后，才读入 `SKILL.md` 正文指令；
- **Level 3**：正文里引用的脚本、模板、参考文档，执行阶段按需读取。

目录结构大致是：

```
workspace/skills/
  pdf-report/
    SKILL.md          # frontmatter + 操作指引
    templates/        # 被正文引用的资源
    scripts/render.py
```

`SKILL.md` 的 frontmatter 示例：

```markdown
---
name: pdf-report
description: 将 Markdown 渲染为 PDF。当用户要求"导出 PDF""生成报告文件"时使用。仅讨论内容、不导出文件时不要使用。
---
```

注意 description 的写法：它不是功能宣传，而是**触发条件**。

## 接入步骤

1. **建目录**：在 workspace 下新建 `skills/<skill-name>/SKILL.md`；
2. **写 frontmatter**：description 写清输入输出、典型触发短语、以及"何时不该用"；
3. **声明依赖**：需要的 CLI、环境变量在正文开头写明。缺依赖的 skill 会在 gating 阶段被剔除，不会污染上下文——这是很多实现里被忽略的细节；
4. **重载验证**：直接问 agent "你目前有哪些技能"，确认加载和剔除结果符合预期；
5. **用配置控开关**：内置 skill 用不到的直接禁用，压缩常驻描述的规模。

## 踩坑点

- **description 太泛**："处理文件"这种描述什么都匹配不上，也什么都可能匹配。要么不触发，要么乱触发；
- **职责重叠**：两个 skill 都声称处理"发消息"，模型会摇摆。一个 skill 一个明确任务，重叠部分合并；
- **正文塞太满**：`SKILL.md` 控制在几百行内，参考资料放引用文件，让 Level 3 真正生效；
- **路径写死**：脚本用绝对路径，换环境就断，统一用相对 workspace 的路径；
- **改名不观察**：skill 改名后，旧会话的记忆还引用旧名，建议观察一两次会话再清理。

## 可复用建议

- 把 skills 目录纳入 git，每个 skill 的演进历史本身就是最好的文档；
- 新 skill 先用最小版本（只有一个 SKILL.md）验证触发是否可靠，再补脚本和资源；
- 定期做"技能审计"：让 agent 列出当前技能，长期未触发的果断下线；
- description 用固定模板：**做什么 + 何时用 + 何时不用**，三句话以内。

## 总结

Skills 机制的价值不在于"能装多少能力"，而在于"平时不装"。三级加载把常驻成本压到几十 token，把调用决策交给触发条件。实践中你会发现，写好 description 占了效果的八成——它本质上是在为模型做检索优化，而不是写产品文案。先从一个最小的 SKILL.md 开始，比一开始就规划十个技能更靠谱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/34e695ab38a8541e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/08659611a9832a5b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/813d1c21c4225ce6.png)

