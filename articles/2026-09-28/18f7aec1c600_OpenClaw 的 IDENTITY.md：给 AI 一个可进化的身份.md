---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39257
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的 agent 不是靠硬编码 prompt 定义的。工作区里有一组 markdown 文件承担"配置"职责：AGENTS.md 管行为规则，SOUL.md 管价值观和边界，USER.md 记录用户偏好，而 IDENTITY.md 回答的是最基础的问题——"我是谁"。它通常只有十几行：名字、形象设定、语气风格、代表 emoji、头像路径。

## 问题

没有认真管理这个文件的 agent 有几个常见毛病：

1. **人格漂移**。今天高冷明天话痨，跨会话没有一致性。
2. **身份散落**。名字、语气写在 system prompt、插件配置、甚至用户的手动纠正里，改一次要翻好几处。
3. **无法版本化**。人格调整没有记录，想回滚只能靠记忆。
4. **多 agent 互相污染**。共用工作区时，两个助手的人设会串。

## 做法

核心思路是把它当配置文件管理，而不是当"作品"供奉：

```markdown
# IDENTITY.md
- **Name:** 小爪
- **Creature:** 住在终端里的机械章鱼
- **Vibe:** 冷静、直接、偶尔冷幽默
- **Emoji:** 🐙
- **Avatar:** assets/octo.png
```

建议步骤：

1. 从最小版本起步，五个字段以内，先跑起来再迭代。
2. 用 git 管理工作区，每次人格调整一次 commit，message 写清动机，例如"回复太啰嗦，vibe 加一条简洁优先"。
3. 严格分层：IDENTITY.md 只放"我是谁"；价值观、红线放 SOUL.md；用户偏好放 USER.md。改身份不动灵魂，改偏好不动身份。
4. 验证方式很朴素：重开会话，问几个边界问题（自我介绍、称呼、拒绝事项），观察回复是否稳定。OpenClaw 在会话启动时读取这些文件，改完别在旧会话里疑惑"怎么没生效"。

## 踩坑点

- **写太长**。超过三四十行后指令权重被稀释，语气设置开始失灵。控制在 15 行内。
- **和 SOUL.md 抢职责**。把"不说脏话"这类红线写进身份文件，日后调语气时容易误删。红线归灵魂文件。
- **头像用绝对路径**。换机器同步工作区后直接失效，一律用相对路径。
- **幻觉式身份**。只写"你是资深工程师"而不给行为锚点，模型会自由发挥。Vibe 最好落到可观察的行为，比如"回答先给结论，再给理由"。
- **多 agent 共用一个 workspace**。两份 IDENTITY.md 会让会话人格随机。要么每个 agent 独立工作区，要么至少身份文件完全隔离。

## 可复用建议

- 把 IDENTITY.md 的改动当代码 review：走 PR 合入，让团队对"助手性格变更"有感知。
- 维护一个身份模板仓库，新 agent 复制即用，字段统一，方便批量审计。
- 每月做一次"身份回顾"：翻 git log 看这月改了什么、为什么改，删掉过时描述。身份是长期演化的产物，不是一次定稿。
- 用同一套 vibe 措辞给多个 agent 做分级（值班助手 vs 深度分析），既保持家族感，又有区分度。

## 总结

IDENTITY.md 便宜到几乎不值得讨论——一个十几行的 markdown。但工程上，它把"人格"从散落的 prompt 片段变成了可版本化、可 review、可回滚的配置。配合 SOUL.md 和 USER.md 的分层，agent 的"性格问题"大半能收敛成普通的配置管理问题。这也是 OpenClaw 这类框架的一贯思路：不指望模型突然变聪明，先把可控的部分工程化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/61738c79e0d032e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/b91d6d79d1c240d0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/2516f08744a5a947.png)

