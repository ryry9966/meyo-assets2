---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39974
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的 workspace 里有一组以 Markdown 为载体的"软配置"：`AGENTS.md` 管行为约束，`SOUL.md` 管性格与行事方式，`MEMORY.md` 管记忆，而 `IDENTITY.md` 管"我是谁"。不少人是跳过 bootstrap 直接用的，或者首次运行随手填了个名字，之后再没看过。直到 agent 在不同会话、不同渠道里表现飘忽，才发现这份文件其实是身份的唯一事实来源。

## 问题

实际用下来，身份管理有四个典型痛点：

1. 默认身份是"通用助手"，回答风格随模型状态漂移，今天话痨明天冷淡；
2. 身份写死在系统提示词里，改一次要动配置，且没有任何变更历史；
3. 多 agent 共用一个 workspace 时互相串味，A 的语气污染 B；
4. 身份和记忆混写，本该稳定的部分跟着记忆一起漂。

## 做法

我的落地步骤，供参考：

1. **定位文件**：workspace 根目录下的 `IDENTITY.md`，建议字段为 name / creature / vibe / emoji / avatar；
2. **最小可用版本先跑**，不要一次写满，跑一周再定稿；
3. **用 git 管理 workspace**，身份变更有 diff、可回滚；
4. **建立进化节奏**：每次模型大版本升级或使用满一个月，让 agent 复盘自己的回答，提交身份修订建议，人工审核后合并；
5. **多 agent 场景**：每个 agent 独立 workspace，身份物理隔离。

一个最小示例：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 一只住在终端里的机械蟹，动作慢但从不放弃
- Vibe: 克制、工程化、偶尔冷幽默
- Emoji: 🦀
```

## 踩坑点

- **把事实性记忆写进 IDENTITY.md**。该去 MEMORY.md 的内容混进来，身份会越写越臃肿，且每个会话都白白消耗 token；
- **与 SOUL.md 职责重叠**。IDENTITY 回答"我是谁"，SOUL 回答"我怎么行事"，两边重复定义会导致语气打架；
- **写太长**。这份文件每个会话都注入上下文，控制在 30 行以内为宜；
- **让 agent 自主改身份且不留审核**。身份漂移是一种难以逆转的气质污染，务必走 PR / 人工确认；
- **emoji 和 creature 不是装饰**。它们实实在在地影响输出语气，改之前先想清楚。

## 可复用建议

- 模板字段保持稳定，只改取值，diff 才干净；
- 把"身份修订"做成定期任务（cron 或 heartbeat 触发），agent 自评 + 人审合并，形成闭环；
- 不要在文件里维护 changelog，`git log` 就是 changelog；
- 冷启动新 agent 时先复制模板跑一周，观察输出风格再定稿，比一开始精雕细琢效率高。

## 总结

IDENTITY.md 的价值不在于"写了一个名字"，而在于把身份从提示词里散落的描述，变成一份可版本化、可评审、可进化的配置。身份稳定了，行为才可预期；行为可预期，长期协作才有基础。它解决的不是能力问题，而是稳定性问题——而这恰恰是自托管 agent 最容易忽略的一环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/7c87cfe29af1e622.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/82f767074cdb293f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/cee368b705168a13.png)

