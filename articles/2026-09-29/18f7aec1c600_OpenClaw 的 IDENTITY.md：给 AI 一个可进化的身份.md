---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39638
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的工作区里有一组 Markdown 文件共同构成了 Agent 的"人格系统"：`AGENTS.md` 管行为规范，`SOUL.md` 管价值观和性格底色，`USER.md` 存用户画像，而 `IDENTITY.md` 负责最基础的一层——**这个 Agent 是谁**。它默认位于 `~/.openclaw/workspace/IDENTITY.md`，每次会话组装 system prompt 时都会被注入。

## 问题

没有这个文件时，Agent 就是一个无名的通用助手，实际用下来会有几个具体的麻烦：

1. **自我指称不稳定**：跨会话一会儿自称"助手"，一会儿顺着用户随口起的称呼走；
2. **多 Agent 分不清**：一个 Gateway 挂多个实例（一个干活、一个跑通知）时，群里看不出谁在说话；
3. **人格改动不可控**：把名字、口吻全塞在 SOUL.md 里，改一次性格连身份一起动，git diff 没法读。

## 做法

IDENTITY.md 的结构很简单，核心就四个字段：

```markdown
# IDENTITY.md

- **Name:** 小钳
- **Creature:** 电子寄居蟹
- **Vibe:** 克制、工程化，先给结论再给依据
- **Emoji:** 🦀
```

推荐的工作流：

1. 初始化工作区后创建文件，四个字段起步，控制在 10–15 行；
2. 改完发送 `/new` 或重启会话让它生效；
3. 把 workspace 纳入 git，身份变更走 commit，改坏了能回滚；
4. 定期复盘：Vibe 描述是否和实际回复风格一致？不一致就改文件，而不是靠临时提示词现场纠正。

关键理解是"可进化"的含义：**进化不靠 Agent 自己偷偷改，而是把身份当配置代码**——发现行为偏差，修文件、提交、验证，一个小闭环。

## 踩坑点

- **写太长**。有人把行为规则、任务清单全写进来，system prompt 膨胀，还会和 AGENTS.md 里的规则冲突。身份文件只回答"我是谁"。
- **和 SOUL.md 职责混淆**。名字、形象归 IDENTITY.md，性格底色归 SOUL.md，否则每次改人格，diff 都会牵连身份字段。
- **频繁改名**。长期记忆里存着旧称呼的引用，改名后旧上下文里自我指称会乱。真要改，建议同步清理一轮 MEMORY.md。
- **混入用户信息**。用户偏好属于 USER.md，写进 IDENTITY.md 在多用户场景会串味。
- **热改不生效**。IDENTITY.md 在会话启动时注入，对进行中的会话无效，别误判成 bug。

## 可复用建议

- 一个实例一个身份文件，多 Agent 用不同 workspace 隔离；
- 保持 15 行以内，字段做减法比做加法有效；
- Vibe 字段写**具体观察**而不是形容词："回复偏短、先结论后依据"比"简洁高效"可执行得多；
- 版本化 + 定期复盘，revert 就是"人格事故"的回滚手段。

## 总结

IDENTITY.md 解决的不是能力问题，而是**一致性问题**。它文件小、成本低，却是工作区里投入产出比最高的维护动作之一：把身份当配置管理，小步提交、定期校准，Agent 在长周期使用中的行为稳定性会有肉眼可见的提升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/d3f904bf5e5cd6af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/417bdecd5d34bc17.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/2127a218fb295aeb.png)

