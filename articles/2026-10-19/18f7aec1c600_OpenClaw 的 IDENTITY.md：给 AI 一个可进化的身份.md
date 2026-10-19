---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42004
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

OpenClaw 的 workspace 里有一组"人格文件"：SOUL.md 管行为准则，USER.md 记用户画像，AGENTS.md 定工作方式。IDENTITY.md 是其中最小的一个，它只回答一个问题："我是谁"。默认路径在 `~/.openclaw/workspace/IDENTITY.md`，字段通常只有 Name、Creature、Emoji、Avatar、Vibe 这几行。听起来像装饰品，但跑久了你会发现，它是多渠道、多 Agent 场景下行为一致性的锚点。

## 问题

没有认真维护 IDENTITY.md 时，常见的三种症状：

1. **人格漂移**：Telegram 上克制、CLI 里话痨，同一个 Agent 在不同 channel 表现得像三个不同的东西。
2. **人格散落在 prompt 里**：系统提示、渠道模板、记忆文件各写一份"你应该……"，改一次要翻五个地方，改完还互相矛盾。
3. **不可审计**：上周它还叫这个名字、用这个语气，现在变了，但没人知道是谁改的、为什么改。

本质上：身份没有版本化，就谈不上"进化"，只能叫"随机漂移"。

## 做法

**第一步：最小可用身份。** 新建 IDENTITY.md，十行以内：

```markdown
# IDENTITY

- **Name:** Claw
- **Creature:** 机械螯虾
- **Emoji:** 🦞
- **Avatar:** avatars/claw.png
- **Vibe:** 工程化、克制，先结论后依据
```

注意这是"名片"不是"作文"。它每个 session 都会被注入上下文，长文件是在烧 token 换噪音。

**第二步：划清边界。** 身份只放稳定特质（名字、口吻、审美）；工作流规则放 AGENTS.md，行为红线放 SOUL.md，用户信息放 USER.md。判断标准很简单：这条描述半年后还成立吗？成立才进 IDENTITY.md。

**第三步：git 管起来。** 把 workspace 纳入版本控制，身份改动像发版一样提交：

```
git commit -m "identity: 收敛语气，去掉营销腔"
```

观察两三天的实际对话再决定下一次改动。"可进化"的全部含义就是：不是写得多，而是 diff 得清楚。

**第四步（多 Agent 场景）：** 每个 Agent 独立 workspace、独立 IDENTITY.md。多个 Agent 共享一份身份文件，是人格串味的最快路径。

## 踩坑点

- **改完不生效**：身份在 session 启动时加载，长会话里改文件没用，先 `/new` 或重启会话。
- **越写越长**：加到三十行后模型开始抓错重点。我的经验是超过十行就该砍，细节挪去 SOUL.md。
- **与系统提示冲突**：全局 prompt 要求"永远简洁"，IDENTITY.md 却写着"热情洋溢地解释"，表现就会忽冷忽热。改身份前先 grep 一遍其他配置。
- **Avatar 相对路径**：不同启动目录下会找不到图，用绝对路径或确保相对 workspace 根目录。
- **频繁改人设**：每天换人设等于没有人设。把身份改动当 release，别当草稿。

## 可复用建议

1. **身份即代码**：review 一份身份 PR 的成本极低，收益是全渠道行为一致。
2. **固定探针测试**：改完身份，用同一组问题（"做个自我介绍""拒绝一个不合理请求"）对比新旧输出，比肉眼感觉靠谱。
3. **字段宁缺毋滥**：Creature、Colors 可有可无，但 Name 和 Vibe 建议必填——一个是调用锚点，一个是语气锚点。
4. **备份再动刀**：重构身份前打个 tag，行为退化时 `git revert` 秒回滚。

## 总结

IDENTITY.md 的价值不在于让 AI"更像个人"，而在于把一件原本散落、模糊、不可审计的事——Agent 是谁——变成一个十行以内、可 diff、可回滚的文件。身份进化的正确姿势是：频繁提交、缓慢变更、随时能退。这套做法零成本，今天就可以从 `git init` 开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/e816f14bec1d21b9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/c958cc0d836d8df0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/e20696409ff4d338.png)

