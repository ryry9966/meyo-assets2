---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 38648
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

用 OpenClaw 做日常自动化的人迟早会撞上同一个矛盾：希望助手什么都懂，又不希望 system prompt 膨胀到失控。我早期把所有操作规范、工具说明全塞进 AGENTS.md，结果上下文常驻几万 token，模型反而开始“选择性失忆”——指令越多，单条指令的权重越低。

Skills 是 OpenClaw 给出的答案：把“能力”拆成独立目录，常驻上下文的只有一行 description，正文按需加载。

## 问题

核心是两点：

1. **上下文经济学**：全量注入贵且稀释注意力；按需加载省 token，但依赖模型自己判断“要不要读”，判断错了，这个能力就等于不存在。
2. **Skills 与 MCP 工具的边界**：MCP 提供工具 schema（连接后常驻），Skills 提供程序性知识（流程、禁忌、输出格式）。搞混了就会出现“有工具没章法”或“有章法没工具”。

## 做法

最小可用的 Skill 就是一层目录：

```
~/.openclaw/workspace/skills/
└── weekly-report/
    └── SKILL.md
```

SKILL.md 分两段：frontmatter 的 `name` + `description` 常驻 system prompt，下面的 markdown 正文只在模型判断相关时通过 read 工具加载。我是按这个顺序写的：

1. **先审计**：翻聊天记录，找“反复口述同一套流程”的场景。我的第一个 Skill 是周报生成，因为每周都要重申一遍格式。
2. **description 写成路由键**：触发条件 + 做什么 + 边界。例如“当用户要求生成周报或汇总本周提交时使用；不负责数据分析本身”。
3. **正文写成操作规程**：步骤、确认点、失败兜底，而不是功能介绍。依赖外部 CLI 的在 frontmatter 里声明 requires，缺依赖时直接标为不可用，比运行时报错体面。
4. **重内容外置**：模板、清单放到同目录附加文件，SKILL.md 里只写“需要时读取 report-template.md”。
5. **验证**：问一个应该触发的问题，看 gateway 日志确认有没有去读文件。没触发就改 description，触发太频繁就收窄措辞。

装第三方的用 `clawbed skills list` / `clawbed skills install`，装完先通读一遍 SKILL.md 再留——社区 Skill 质量差异很大。

## 踩坑点

- description 写成“帮助处理数据相关任务”这种话，永远不会触发——模型路由完全依赖这段话里的关键词。
- 反过来堆太多触发词，模型什么都往这个 Skill 上靠，白白加载。
- 正文写太长。见过 300 行的 SKILL.md，按需加载变成了按需爆炸，尽量单屏以内，超出就拆文件。
- 两个 Skill 职责重叠，模型选错的概率随重叠度上升，宁可合并。
- 把 API key 写进 SKILL.md——正文会进上下文，密钥只放环境变量或配置。
- 误以为 Skill 能赋予新能力。它不能让模型调用不存在的工具，只能规范已有工具的使用方式。

## 可复用建议

- description 公式：**when（何时）+ what（做什么）+ not（不做什么）**，三段缺一不可。
- 一个 Skill 只干一件事：description 里如果用“和”连接了两个不同场景，考虑拆开。
- Skills 目录进 git，跟 workspace 一起版本化，改坏能回滚。
- 定期数一下所有 description 的常驻 token 成本——十几个 Skill 也可能吃掉上千 token。
- 必须 100% 执行的固定流程（每日备份之类）别指望 Skill 的概率性触发，用 cron / heartbeat 更可靠。

## 总结

Skills 的本质，是用一行 description 换取一次上下文加载的决策权。它不是插件系统，更像一套提示词的分发协议：常驻的尽量短，加载的尽量准。把 Skill 当成“给未来某次对话预存的操作手册”来写，而不是当功能开关来用，基本就不会跑偏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ae39bb8ad5240ed5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/140f28a703e938b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/2a130d27904a27ed.png)

