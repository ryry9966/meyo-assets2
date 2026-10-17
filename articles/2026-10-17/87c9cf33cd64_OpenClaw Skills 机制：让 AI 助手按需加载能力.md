---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 41794
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 的形态是一个常驻网关：接消息渠道、跑定时任务、调用各种工具。跑得越久，能力越多，最常见的惯性就是——把所有操作说明塞进 system prompt，再挂一排 MCP server。结果上下文里常年躺着大量用不上的指令和工具 schema，token 成本高，模型选错工具的概率也上升。

## 问题

核心矛盾是**能力覆盖 vs 上下文预算**。一次会话里真正用到的能力往往只有一两个，但静态注入意味着你为全部能力持续付费。更麻烦的是，候选说明越多，模型越容易张冠李戴。

## 做法

Skills 的解决思路是**渐进式披露**：

1. **一个技能一个目录**。核心是 `SKILL.md`：YAML frontmatter + Markdown 正文。

```markdown
---
name: photo-cleanup
description: 批量整理本地照片目录，按拍摄日期归类并去重。当用户提到"整理照片""重复图片"时使用。
metadata:
  openclaw:
    requires:
      bins: [exiftool]
---
## 步骤
1. 先扫描目录，统计数量与疑似重复项，向用户确认后再动文件。
2. 用 exiftool 读 DateTimeOriginal，缺失时退回文件 mtime。
```

2. **只注入索引**。会话启动时，仅把每个技能的 name + description 作为一行摘要放进上下文。正文要等模型判断当前任务相关时，才主动读取。没触发 = 几乎零成本。
3. **用 requires 做门控**。`requires.bins` / `requires.env` 声明依赖，环境不满足的技能不会出现在索引里，避免"列表里有、执行时炸"。
4. **三个来源**：随安装的 bundled、`~/.openclaw/workspace/skills/` 下的 workspace 技能（同名会覆盖 bundled）、以及从 ClawHub 安装的社区技能。
5. **验证**：`openclaw skills list` 查看哪些被加载、哪些因缺依赖被隐藏，然后拿真实对话测触发。

## 踩坑点

- **description 写成功能介绍**。模型靠它做路由，务必写成"做什么 + 什么时候用"，覆盖用户实际会说的话。
- **正文过长**。一旦触发，整篇正文都进上下文。控制在 150 行以内，细节拆成子文件让 agent 按需再读。
- **不写 requires**。缺 exiftool 的机器上技能照常触发，执行到一半才报错。
- **把密钥写进 SKILL.md**。正文会进上下文和日志，敏感信息走环境变量，并在 requires.env 里声明。
- **改完不生效**。索引在会话启动时建立，重开会话（或重启 gateway）再看效果。
- **贪多**。每条 description 都常驻 token，定期审查，把不用的 bundled 技能在配置里禁掉。

## 可复用建议

- description 万能句式：*用 X 完成 Y。当出现 Z 场景时使用。*
- 一个技能只解决一类任务，别做"大而全"的万能手册。
- 把 Skill 当 MCP 工具的 playbook：MCP 出原子工具，Skill 写清组合步骤和边界条件，比让模型自由发挥稳定得多。
- 技能目录放进 git 随 workspace 分发，团队直接复用。
- 上线前至少测两轮：该触发的一次命中，不该触发的稳定不误触。

## 总结

Skills 把能力注入从"静态堆料"变成了"按需检索"，上下文成本和工具选择准确率同时受益。实际体验下来，80% 的效果取决于那段 description——写清楚"做什么、何时用"，剩下的交给机制本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/38b3bd038904a084.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/378daa74f3e33a7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b046821206b3e908.png)

