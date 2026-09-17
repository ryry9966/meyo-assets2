---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37966
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

OpenClaw 的 agent 是长跑型选手，不是一次性问答。会话由系统提示词驱动，而工作区（默认 `~/.openclaw/workspace/`）下的几个 Markdown 文件是关键输入。其中 `IDENTITY.md` 定义"我是谁"：名字、物种、气质、emoji、头像。它在会话启动时被注入 system prompt，是少数每次对话都稳定生效的"常量"。

## 问题

默认状态下这份文件基本是空模板，实际后果是：

- 同一个 agent 在不同会话里人格漂移，今天冷静明天话痨；
- 换一台机器部署，形象和称呼全部丢失；
- 个性化设定散落在各渠道配置里，改一处漏三处；
- 没有版本记录，你根本说不清它的"性格"是哪次对话被改歪的。

## 做法

1. **初始化最小字段集**。字段越少越稳，一个可直接用的例子：

```markdown
- Name: 小锚
- Creature: 一只住在终端里的螃蟹
- Vibe: 冷静、直接、偶尔冷笑话
- Emoji: 🦀
- Avatar: assets/avatar.png
```

2. **划清文件边界**。身份归 IDENTITY.md（长期不变的事实），行为与语气归 SOUL.md，用户上下文归 USER.md，别交叉污染。
3. **版本化**。把整个 workspace 纳入 git，每次身份调整就是一次 commit，演进从此有 diff 可查。
4. **引导生成、人工落盘**。OpenClaw 的系统提示词本身鼓励 agent 在首次对话中填写身份草稿，但你要做的是审一遍再接受，否决权始终在人的手里。
5. **验证生效**。身份在会话启动时注入，改完后开一个新会话，问一句"你是谁"，确认注入结果。

## 踩坑点

- **别把记忆写进身份**。身份是"我不变的部分"，任务清单和临时偏好属于 MEMORY。混写的结果是身份天天变，等于没有身份。
- **控制长度**。IDENTITY.md 进 system prompt，每多一行都是持续的 token 成本和注意力稀释，几十行以内足够。
- **头像别塞大图**。几百 KB 的小尺寸图片即可，大文件没有收益。
- **无 review 的自我进化会跑偏**。允许 agent 提议，但不允许自写自审——diff 里必须有人。

## 可复用建议

- 把 IDENTITY.md 当 config-as-code 管理：小、稳定、可 review、可回滚。
- 月度回顾一次身份相关的 git log，比"感觉它最近变了"可靠得多。
- 多机/多渠道部署时，以 git 仓库为唯一事实源，部署侧禁止手改。
- 演进历史本身就是资产：半年后回看身份的 commit 记录，比任何刻意撰写的"性格文档"都真实。

## 总结

IDENTITY.md 不是给 AI 起名字的装饰品，而是每次会话都会被读取的常量输入。它的价值不在于写得动人，而在于足够小、足够稳、可追溯。"可进化"的前提是演进可控：一次改动一次 commit，有人审，有据可查。做到这三点，身份就不是玄学，是工程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/e9bed3e87a2ab0b8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/50ede567715d82da.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/e5c02d80846b4612.png)

