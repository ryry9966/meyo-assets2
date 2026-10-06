---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40695
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的工作区（workspace）里有一组会被注入 agent 上下文的 Markdown 文件：`SOUL.md`、`TOOLS.md`、`USER.md`，以及今天的主角 `IDENTITY.md`。它决定 agent 叫什么、用什么语气说话、开场白长什么样。很多人装完就用默认值，直到某个时刻——agent 在群里自我介绍混乱、语气忽冷忽热——才意识到这个文件值得认真写。

## 问题

不认真写 IDENTITY.md，常见三类症状：

1. **自我认知漂移**。长会话或重建会话后，agent 对"我是谁"的描述前后不一致，有时自报默认名，有时照抄用户消息里的临时称呼。
2. **语气不可复现**。某天的回复风格你觉得很好，但固化不下来，换个会话就变样。
3. **多 agent 串味**。同时跑几个 workspace 时，如果身份只写在临时 prompt 里，互相污染是常态。

根因相同：身份没有被当成一份可版本化、可迭代的配置，而是散落在一次性会话指令里。

## 做法

我的实践分四步：

1. **落到文件**。在 workspace 根目录建 `IDENTITY.md`，字段就几行：名字、一句话定位（什么角色、服务谁）、语气规则、emoji/头像、开场白。控制在 20 行以内。
2. **行为化描述**。不写"幽默、专业"，写具体动作："解释报错先说人话再贴日志；默认回复不超过 5 行，除非用户要求展开"。
3. **小步迭代**。准备固定的 3~5 个探针问题（自我介绍、处理模糊任务、拒绝越权操作），发现语气问题就改一行，git commit，一次只动一条规则。
4. **多 agent 隔离**。每个 workspace 一份 IDENTITY.md，命名体现分工（如 `claw-docs`、`claw-ops`），绝不共享同一份身份。

## 踩坑点

- **和 SOUL.md 抢戏**。我的划分：IDENTITY.md 管"是谁、什么风格"，SOUL.md 管"价值观和行为底线"。同一规则写两处必然漂移。
- **写成形容词清单**。"聪明、简洁、友好"模型执行不了，必须翻译成可观察的行为。
- **当记忆用**。IDENTITY.md 每次会话都会加载，但它不是记忆——你口头说"以后叫我老王"，不写进 USER.md 就会丢。
- **改了不生效**。已存在的会话里旧身份上下文权重很大，改完文件要开新会话验证。
- **格式污染**。文件里的标题层级和加粗会被模型模仿着输出到聊天里，建议用纯文本短句。

## 可复用建议

- 把整个 workspace 纳入 git，IDENTITY.md 的 diff 就是人格 changelog，可回滚、可 review。
- 坚持"一行一规则"，超过 20 行说明你在写制度而不是写身份，砍掉。
- 团队场景把 IDENTITY.md 抽成模板仓库，成员 fork 后只改称呼和语气段。
- 每月重跑一次探针问题集，尤其在换模型或升级版本后，身份退化能第一时间发现。

## 总结

IDENTITY.md 的价值不在"让 AI 更像人"，而在把 agent 的自我描述变成一份可审查、可回滚、可持续迭代的工程资产。它非常便宜——一个十几行的 Markdown 文件——但决定了你每次与 agent 交互的第一印象是否稳定。建议今天就把默认身份换成你自己的，然后交给 git 管理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/ba3cce9e8ee8e1da.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/bb6b8962b91a3180.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/c5de2fdc434d07a3.png)

