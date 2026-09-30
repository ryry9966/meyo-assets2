---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39799
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 的工作区里有一组 Markdown 文件承担了 agent 的"人格层"：`SOUL.md` 管价值观与行为边界，`AGENTS.md` 管任务规则，`MEMORY.md` 管长期记忆。`IDENTITY.md` 是其中最短的一个，但它解决的问题最基础：**这个 agent 是谁**。

我的工作区在 `~/.openclaw/workspace/`，每次新会话启动时，IDENTITY.md 会被注入 system prompt。它很小，但它决定了长期交互里用户对 agent 的稳定认知。

## 问题

没有这个文件也能跑，但会遇到几个具体的麻烦：

- 新会话的自我介绍每次不一致，长对话里"我是谁"会漂移；
- 多 agent 部署（一个干活、一个值守）在群里发言风格雷同，用户分不清谁在说话；
- 日志和通知里全是默认名，回溯某次决策时缺一个锚点；
- 想调人设时只能改散落在各处的 prompt 字符串，改一处漏一处。

本质上是身份缺少单一代码源。

## 做法与步骤

1. 在工作区新建 IDENTITY.md，结构尽量简单：

```markdown
# IDENTITY

- Name: Clawbit
- Creature: 机械章鱼
- Emoji: 🐙
- Vibe: 先给结论；不确定就明说；代码示例必须可运行
- Avatar: assets/octo.png
```

2. 保持克制。IDENTITY.md 只放"是什么"，不放"怎么做"——行为规则归 AGENTS.md，价值边界归 SOUL.md。
3. 用 git 管这个文件。身份会变，变更历史本身就是 agent 的履历：

```bash
cd ~/.openclaw/workspace && git log --follow IDENTITY.md
```

4. 改完开新会话（或重启 gateway）再验证——它是会话启动时读取的，当轮不热加载。
5. 多 agent 场景给每个实例独立的 Name + Emoji + Creature，名字要能在语音里自然读出来。

## 踩坑点

- **写太长。** 有人把几十条行为指令塞进去，结果每轮会话固定多烧 token，还和 SOUL.md、AGENTS.md 打架。控制在 15 行以内。
- **改完不生效就去 debug。** 大概率只是没开新会话，先确认加载时机。
- **频繁重写不留痕。** 没有版本历史，出了"上个月它还挺靠谱"这种体感问题就无从回溯。
- **Vibe 写成形容词堆砌。** "有趣、聪明、强大"没有约束力，不如写可验证的倾向："回复先给结论，拒绝时给出理由"。
- **多 agent 共用一份身份。** 消息路由和用户认知都会乱，身份必须一一对应。

## 可复用建议

- 把 IDENTITY.md 当产品决策而不是装饰：名字定了就少改，要改就走 git 提交，写清动机；
- 字段固定为 Name / Creature / Emoji / Vibe / Avatar 五项，团队内统一格式，方便脚本解析和 diff；
- 每季度回顾一次，把日常积累的"它总是……"式抱怨，沉淀成 Vibe 里一条具体、可验证的描述；
- 想深度定制行为时优先改 SOUL.md 和 AGENTS.md，IDENTITY.md 只在"这个 agent 该长什么样"真正变化时才动。

## 总结

IDENTITY.md 的价值不在文件本身，而在它把"agent 是谁"变成一个有版本、有边界、可演进的工程对象。它很便宜——十几行 Markdown；也很贵——身份一旦在长期交互里被用户记住，就是所有记忆和风格的锚点。建议今天就把它纳入 git：先短后长，按需进化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/d3b17ae0e205b9b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/b832933acada3d10.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/9595a4976f94205e.png)

