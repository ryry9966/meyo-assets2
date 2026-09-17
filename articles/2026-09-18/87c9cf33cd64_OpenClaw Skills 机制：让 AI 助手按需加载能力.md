---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 38006
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

Agent 跑久了都会遇到同一个问题：能力越堆越多，系统提示词越来越臃肿。早期我把常用工作流全部塞进 system prompt——部署步骤、日志排查、数据整理规范，加起来上万 token。结果上下文被静态指令占满，模型选工具反而更容易出错，推理成本也上去了。

OpenClaw 的 Skills 机制就是为了解决这个矛盾：能力拆成独立模块，平时只在上下文里保留一段元数据（名称 + 描述），任务真正匹配时才加载完整内容。本质上是渐进式披露（progressive disclosure）。

## 问题

不用 Skills 机制时，痛点集中在三处：

1. **上下文浪费**：90% 的指令在 90% 的对话里用不到；
2. **工具选择混乱**：指令越多，模型越容易张冠李戴；
3. **维护困难**：所有能力耦合在一份 prompt 里，改一处要全文回归。

## Skills 的结构与加载逻辑

一个 Skill 就是一个目录：

```
skills/
└── log-triage/
    ├── SKILL.md      # frontmatter + 指令正文
    ├── scripts/
    │   └── scan.sh   # 可执行脚本
    └── references/
        └── patterns.md
```

`SKILL.md` 的 frontmatter 里写 `name` 和 `description`。OpenClaw 在会话启动时扫描 skills 目录，只把元数据注入上下文；当用户任务与描述匹配，agent 再去读完整正文，需要时执行 `scripts` 里的脚本或翻 `references`。

三层结构对应三级加载：**元数据（常驻）→ SKILL.md（按需）→ references / scripts（更深一层的按需）**。

## 落地步骤

1. **建目录，写 SKILL.md**。description 用"触发条件"句式：解决什么问题、什么场景用、包含哪些关键词。例如："排查网关日志中的连接异常时使用，适合 timeout、断连、握手失败类问题。"
2. **确定性逻辑下沉到脚本**。过滤、格式化、判断这类事不要写成散文让模型自由发挥，写进 `scripts` 让 agent 通过 exec 调用，输出稳定且省 token。
3. **控制体积**。SKILL.md 正文建议几百行以内，更细的内容拆到 references，让 agent 按需翻。
4. **新开会话验证**。我这套部署上，改动不会反映到已开的会话，元数据是会话启动时注入的——这是我踩的第一个坑。验证方法：新开会话，问一个匹配描述的问题，观察它是否去读 SKILL.md。

## 踩坑点

- **description 太泛**（"处理各种任务"）→ 永远不触发；**太宽** → 什么任务都往这个 skill 上凑。写具体场景加关键词。
- **多个 skill 描述重叠**，agent 的选择近乎随机。能力边界尽量互斥，一个 skill 只做一件事。
- **脚本路径与环境变量**：host 和容器环境不一致，本地测通、容器里跑挂。脚本内尽量用相对 skill 目录的路径。
- **权限问题**：skill 里的脚本以 agent 的权限执行，第三方 skill 先审脚本再装，当作 review 代码对待。
- **引用 references 文件时的相对路径**，解析基准可能是 skill 目录也可能是 workspace，拿不准就写绝对路径说明。

## 可复用建议

- 把 skills 目录纳入 git：skill 就是代码，走 review、留变更记录。
- 团队公共能力沉淀成一个 skills 仓库，成员同步到各自 workspace。
- 定期清理不触发的 skill：连续几周没命中，要么改描述要么删掉。
- "元数据 → 正文 → 引用文件"的分层思路可以推广到所有 agent 能力设计，不止 OpenClaw 适用。

## 总结

Skills 机制的价值不在"多装几个插件"，而在把能力的**注册**和**加载**分开：注册成本是几十个 token，加载成本只在真正用到时支付。对长期运行的 agent 来说，这个结构直接决定上下文质量和行为稳定性。建议从拆一个最常用的 prompt 开始，验证触发可靠后，再逐步迁移其余能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/b48c1d85e2af8f1d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/91ac72e672bef448.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/938a6b7903c05dd3.png)

