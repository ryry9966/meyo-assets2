---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38475
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景：上下文预算是稀缺资源

跑过一段时间 OpenClaw 的人多半有类似体验：为了让助手"会做事"，我们习惯把操作手册、常用命令、注意事项一股脑塞进 system prompt 或 AGENTS.md。结果常驻上下文越滚越大，真正被用到的内容不到两成。MCP 解决的是"工具怎么接进来"，但工具列表和描述本身也常驻 token。Skills 机制针对的正是这个问题：把能力拆成独立文件，启动时只暴露一行描述，模型判断相关后才加载全文——也就是渐进式披露（progressive disclosure）。

## 一个 Skill 长什么样

一个 Skill 就是一个目录，核心是 SKILL.md：

```markdown
---
name: daily-report
description: 生成每日运维日报。当用户要求"出日报""汇总今天的告警"时使用。
---

## 步骤
1. 读取今日告警列表 ...
2. 按服务分组统计 ...
```

frontmatter 的 `description` 是模型开机就能看到的全部信息；正文只有在模型决定调用时才进入上下文。常驻成本是几十 token，正文写多长都不影响日常会话。

## 实操步骤

1. 建目录：workspace 下建 `skills/daily-report/SKILL.md`（全局能力可放 `~/.openclaw/skills/`）。
2. 写 description：建议"做什么 + 何时触发"的句式，这是决定命中率的唯一检索入口。
3. 正文只写模型不知道的东西：内部约定、路径、命令序列，别复述常识。
4. 验证：`openclaw skills list` 确认已加载，`openclaw skills info <name>` 看解析详情。
5. 实测触发：用自然语言问一句，观察模型是否命中并展开正文。

## 踩坑点

- **description 写成关键词堆砌**（"日报 报告 汇总 统计"）：容易误触发。写成一句自然语言、带上触发场景，命中率明显更好。
- **正文塞满背景故事**：skill 被加载后同样占上下文，只保留可执行步骤和硬信息。
- **多个 skill 职责重叠**：模型会犹豫或选错，定期合并、删旧的。
- **正文里引用脚本用了相对路径**：执行时的工作目录未必是你以为的那个。脚本要么放进 skill 目录、在正文写清路径约定，要么统一走 workspace 路径。
- **frontmatter 格式错误**：`---` 分隔符缺失、字段名拼错，整个 skill 不被识别。`skills list` 里看不到，先查格式。

## 可复用建议

- **分工原则**：MCP 管"连什么工具"，skill 管"怎么组合这些工具完成某类事"。把稳定流程从 prompt 迁到 skill，是性价比最高的瘦身。
- 一个 skill 一个职责，宁可多建几个小的，不要造大而全的"百科 skill"。
- description 是写给模型的检索接口，按实际命中情况持续迭代，它和代码一样需要维护。
- 团队场景把 skills 目录放进仓库一起版本化，新人 clone 下来，助手就自带全套作业规范。

## 总结

Skills 机制的本质，是把 prompt 工程从"一篇大文档"变成"可索引的能力库"：常驻成本压到最低，需要时才展开。我的体感是，把十几个常用流程迁进 skills 后，常驻 prompt 缩了一半以上，触发准确率反而因为 description 更聚焦而变好。如果你的助手也开始"什么都知道一点、什么都记不全"，值得花一个下午完成这次迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/0170b224eca61edc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/b6bd7b2eeb9fab94.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/070bcb9bd79ddb1e.png)

