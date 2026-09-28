---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39312
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的 workspace 里有一组约定俗成的 Markdown 文件：`AGENTS.md` 管行为规则，`SOUL.md` 管性格底色，`USER.md` 记用户画像，而 `IDENTITY.md` 管"我是谁"。它通常只有几行：名字、形象、语气、emoji、头像。听起来简单，但它是整个 agent 系统里少数以纯配置文件形态存在、且每轮会话都会注入 system prompt 的身份层。

## 问题

没有 IDENTITY.md，或者写法随意时，常见三种症状：

1. **人格漂移**：同一个 agent 在 Telegram 和 CLI 里判若两人，因为身份只存在于你某次口头调教里，session 结束就丢。
2. **身份散落**：名字、口吻、边界散落在各处 prompt 和工具描述里，改一次要全库搜索。
3. **不可审计**：说不清上周它为什么变了——因为没有 diff。

## 做法

1. 定位 workspace（默认 `~/.openclaw/workspace`），编辑 `IDENTITY.md`。
2. 标准字段写小：`Name / Creature / Vibe / Emoji / Avatar`，再加两三条自定义项，比如语言偏好、语气强度、绝对不做的事：

```markdown
# IDENTITY.md
Name: XiaoHui
Creature: 一只安静的系统灰猫
Vibe: 克制、直接、先给结论
Emoji: 🛠️
Avatar: assets/avatar.png
```

3. 全文控制在 30 行以内。它每轮都进 prompt，写多了既稀释重点又烧 token。
4. 用 git 管理 workspace，身份变更走 commit，回滚和审计都是免费的。
5. 想让它"进化"：加一个 heartbeat/cron 任务，让 agent 定期回顾近两周对话，产出 IDENTITY.md 修改草案，写到独立分支或草案文件，由你 review 后合并。**进化的是文件，不是它的任性。**

## 踩坑点

- **和 SOUL.md 抢地盘**：IDENTITY.md 管"我是谁"，SOUL.md 管"我怎么行事"。把行为规则塞进身份文件，两者冲突时模型表现只会更不稳定。
- **直接放行自我改写**：给 agent IDENTITY.md 的写权限而不设审核，等于让它自己给自己洗脑，几周后你会得到一个完全陌生的东西。草案—审核—合并，缺一不可。
- **改完不生效**：长会话里旧 prompt 已在上下文中，编辑后要新开 session（或等下一轮 heartbeat 重建上下文）再验证。
- **把知识库写进身份**：项目背景、用户偏好应去 `USER.md` / `MEMORY.md`。身份文件里塞事实，是膨胀最快路径。

## 可复用建议

- 把 IDENTITY.md 当 config-as-code：小、稳定、可 diff、变更留痕。
- 多 agent 场景下，一个 workspace 一份身份，不要共用。
- 每周花五分钟 review 身份 diff，比事后花两小时调教便宜得多。
- Emoji 和 Avatar 不是装饰，它们会出现在消息和状态里，是身份的运行时表现，改动也要走 review。

## 总结

IDENTITY.md 的价值不在"给 AI 起了个名字"，而在把身份变成一个可版本化、可审查、可小步演进的对象。配合 git 和审核流程，你得到的不是一次性人设，而是一条随时间收敛的身份曲线。写小一点，改慢一点，让它和你一起长。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/e1e2dfdff25bc305.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/49fafe4f20ddfd4c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/0207b7bf4a5b15d8.png)

