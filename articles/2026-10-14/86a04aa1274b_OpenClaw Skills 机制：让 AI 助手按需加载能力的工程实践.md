---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 41536
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的 agent 长期运行在一份固定的 system prompt 上。把所有能力都塞进提示词，很快会撞上两个硬约束：上下文预算有限，且大量无关指令会稀释模型对当前任务的注意力。Skills 就是为此设计的机制：把"某类任务该怎么做"封装成独立目录，平时只暴露一行元信息，agent 判断相关时才读全文。

和插件、MCP 的分工值得先说清楚：MCP 解决"能调用什么工具"，skill 解决"什么时候、按什么流程用这些工具"。前者是能力，后者是操作手册。

## 问题

实际使用中，痛点集中在三处：

1. 同一类做法反复在对话里纠正，比如周报格式、服务器巡检步骤；
2. 把所有 SOP 写进 system prompt，token 成本高，而 90% 的会话根本用不到；
3. 好用的提示词散落在聊天记录里，无法沉淀、复用和版本化。

## 做法

一个最小可用的 skill 就是一个目录加一个 SKILL.md：

```
~/.openclaw/workspace/skills/
└── weekly-report/
    ├── SKILL.md
    └── template.md        # 可选，按需读取的附属文件
```

SKILL.md 分两部分：

```markdown
---
name: weekly-report
description: 撰写或整理每周工作汇报。当用户提到"周报""本周总结"时使用。
---

## 步骤
1. 读取 template.md 作为输出结构
2. 从最近的对话和日志中提取本周完成事项
3. ...
```

工作原理是三级渐进披露：

- 一级：frontmatter 的 name + description 常驻 system prompt，让 agent 知道"有这个能力"；
- 二级：agent 判断当前任务相关，才主动读 SKILL.md 正文；
- 三级：正文里引用的 template.md 等文件，需要时再读。

落地步骤：

1. 在 workspace/skills 下建目录，写好 frontmatter 和正文；
2. description 认真写——它是触发的唯一依据，建议"做什么 + 什么时候用 + 关键触发词"；
3. 新开会话，说一句"帮我写周报"，通过日志确认 agent 实际读取了对应 skill；
4. 用配置按 agent 启停：用不到的 skill 直接关掉，避免误触发和 token 浪费。

## 踩坑点

- **description 写成一句话概括**（如"周报技能"）：太含糊，agent 匹配不上就不会加载，必须写成触发条件式描述；
- **正文越写越长**：SKILL.md 控制在百行内，细节拆到引用文件，让第三级披露真正发挥作用；
- **多个 skill 职责重叠**：触发时互相打架，一个 skill 只做一件事，宁可拆细；
- **workspace 与内置 skill 同名**：workspace 优先级更高，会静默覆盖，改名时要有意识；
- **在 skill 里写密钥或敏感路径**：skill 内容会被注入上下文，一律按明文处理；
- **改了 skill 不生效**：description 在会话开始时注入，验证要开新会话，别在旧会话里反复调试。

## 可复用建议

- 把"第二次纠正同一件事"当作沉淀信号：出现即建 skill；
- 用 git 管理 workspace/skills，代码评审的 PR 流程同样适用于提示词资产；
- 定期让 agent 自报："最近你实际触发过哪些 skill"，长期未触发的考虑下线或重写 description；
- MCP 工具的 description 尽量短，编排逻辑写进 skill，两者不要重复。

## 总结

Skills 的价值不在"多一种插件"，而在于把上下文管理变成显式的工程决策：常驻的只有元信息，细节按需加载。把高频流程沉淀成小而准的 skill，配合严格的 description 写法，长期运行的 agent 才不会在提示词膨胀中失控。建议从一个最痛的重复流程开始，先跑通一个 skill，再谈体系。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/2ecdfd7cfeba30b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/de5a1b4807d857a4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/f8675f09bb5240be.png)

