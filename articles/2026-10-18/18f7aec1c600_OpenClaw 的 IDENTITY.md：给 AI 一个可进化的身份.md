---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41945
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 的 agent 不靠一大坨硬编码的 system prompt 描述"我是谁"，而是把上下文拆成 workspace 里的一组 Markdown：`AGENTS.md` 管操作规则，`SOUL.md` 管行为底线，`USER.md` 记用户偏好，`IDENTITY.md` 只回答一个问题——我是谁。每次会话启动，这些文件随 system prompt 注入。身份单独成文，本质是把"我是谁"从"我该怎么做"里解耦出来。

## 问题

跑过长期 agent 的人大概都见过这三种症状：

1. **身份漂移**：换个模型或清了会话，自我称呼就变了，日志里一会儿一个名字；
2. **人设串味**：同一网关下多个 agent，prompt 片段互相抄，语气和自称混在一起；
3. **身份散落**：定时任务、技能提示词里各写一份 "you are..."，改个名字要动五个地方。

根因都一样：身份没有单一事实来源。

## 做法

先看一个最小可用的 IDENTITY.md（放在 workspace 根目录）：

```markdown
# IDENTITY.md

- **Name:** Claw
- **Creature:** 一只机械小螃蟹
- **Emoji:** 🦀
- **Vibe:** 克制、直接、偶尔冷幽默
```

落地步骤：

1. **首启仪式**：第一次启动时 OpenClaw 会引导 agent 自己起名、选 emoji，你也可以直接手写这个文件覆盖结果；
2. **守住四要素**：名字、物种、emoji、vibe，控制在 15 行以内，不写传记；
3. **纳入版本管理**：workspace 整体 git 化，`.gitignore` 掉 `USER.md` 等带隐私的文件，只留身份和行为配置；
4. **定义演进流程**：模型可以提议改身份，但必须人审后提交，git log 就是身份的 changelog；
5. **多 agent 场景**：每个 workspace 一份 IDENTITY.md，用不同 emoji 做日志里的视觉标记，肉眼即可分辨是谁在说话。

## 踩坑点

- **别塞行为规则**。IDENTITY.md 每次会话都注入，把"消息要简短"这类规则写进来是纯浪费 token，那是 `SOUL.md`/`AGENTS.md` 的职责；
- **搜一遍残留 prompt**。改完身份后全局搜 `you are` 和旧名字，自动化与技能里散落的旧人设会和它打架；
- **注意生效时机**。文件在会话启动时读取，改完当场追问是看不到变化的，开新会话再验；
- **别让模型自由改**。放开自我编辑身份文件，一两周就会漂成它最喜欢的样子，必须走人审；
- **重建 workspace 前备份**。这文件只有十几行，丢了之后重建的"新它"可能已经不是同一个它。

## 可复用建议

- 四要素最小模板够用了，行数开始膨胀，说明内容该拆去别的文件；
- 把 SOUL/IDENTITY/USER 的分工当架构约束：SOUL 管怎么做，IDENTITY 管是谁，USER 管对方是谁；
- 身份变更走 commit 或 PR，可随时回滚，也方便复盘"它什么时候变的"；
- 每季度回顾一次 vibe 是否还匹配实际使用场景，和依赖升级放进同一个维护窗口。

## 总结

IDENTITY.md 是一个十几行的 Markdown 文件，换来的是跨会话、跨模型的稳定自我认知，以及一条可控的身份演进链路。说到底就一句话：**身份即配置，演进即版本管理**。给 agent 一个稳定的名字，你反复打磨的那些 prompt 才有一个一致的角色来承接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/c967d1b92ee1380a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/5ea11aa2d88eef73.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/ccb3ecd1609e8370.png)

