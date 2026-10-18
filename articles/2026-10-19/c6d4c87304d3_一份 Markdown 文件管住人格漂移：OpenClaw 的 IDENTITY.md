---
title: 一份 Markdown 文件管住人格漂移：OpenClaw 的 IDENTITY.md 实践
feedId: 41959
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

跑 OpenClaw 的时间一长，你会遇到一个不起眼但持续发酵的问题：Agent 的"人格"是散落的。名字写在配置里，语气藏在历史对话里，头像取决于哪个客户端加载了什么资源。换一次模型、换一台机器，那个你熟悉的"它"就不太一样了。

OpenClaw 的 workspace 约定里有一个专门的文件负责这件事：`IDENTITY.md`。思路很朴素——把"我是谁"变成 workspace 里一个可见、可编辑、可版本化的文件，每次会话启动时注入上下文。

## 问题

没有这个约定时，常见痛点有四类：

1. **身份硬编码**：写死在 system prompt 或网关配置里，改一次要动部署，且没有历史可查。
2. **人格漂移**：长期会话叠加频繁换模型后，语气和自称会悄悄走样，你没有基线可以对照。
3. **多 Agent 混淆**：同时跑几个助手时，谁该用什么口吻、什么自称，全靠人脑记忆。
4. **协作无共识**：几个人共用一个实例，"这个 bot 应该是什么样"没有单一事实来源。

## 做法

IDENTITY.md 通常位于 `~/.openclaw/workspace/IDENTITY.md`，建议保持极简，五六个字段就够：

```markdown
# IDENTITY.md
- Name: Claw
- Creature: 一只务实的机械蟹
- Vibe: 克制、直接、偶带冷幽默；不废话，不用感叹号轰炸
- Emoji: 🦀
- Avatar: assets/claw.png
```

推荐的工作流：

1. **先写最小版本**：name + vibe 两行即可跑起来，别一开始就写小作文。
2. **新会话验证**：改完后开一个全新会话问它"你是谁、你怎么说话"，确认注入生效——热编辑不一定立刻生效。
3. **小步迭代**：每次只改一处 vibe，观察几天，满意再合入，把它当 feature branch 对待。
4. **职责分清**：身份进 IDENTITY.md，行为准则进 SOUL.md / AGENTS.md，用户偏好进 USER.md，动态事实进 MEMORY.md。别把 prompt 堆进身份文件。

## 踩坑点

- **写太长**。身份文件是每次会话都要加载的，几百字的小作文会稀释重点，还挤占真正有用的上下文。一行具体的 vibe 比十行形容词有效。
- **和 USER.md 混淆**。把"用户叫我小王"写进 IDENTITY.md 是常见错误——那是用户的属性，不是 Agent 的。
- **频繁重写**。身份的价值在稳定，一天三改的 vibe 等于没有 vibe。
- **过度依赖 avatar/emoji**。部分渠道不渲染头像，核心身份必须靠文字字段成立。
- **多 workspace 直接拷贝同一份文件**，之后各改各的，几周后两边已经不是同一个"人"了——要么刻意 fork 并记录差异，要么保持单一来源。

## 可复用建议

- **把身份当代码管**：workspace 用 git 管理，身份变更有 commit message，能回滚、能 review。
- **写一句"单句测试"**：只看 IDENTITY.md，你能否用一句话向别人描述这个 bot？不能，说明文件没写好。
- **身份是常量，记忆是变量**：进化靠 MEMORY.md 累积事实，而不是靠天天改身份。两者分工清楚了，Agent 才既稳定又成长。
- **多 Agent 场景**给每个 workspace 一份独立 IDENTITY.md，模板统一、字段各异，方便横向对照。

## 总结

IDENTITY.md 不是什么复杂机制，它只是把"Agent 是谁"从隐式状态变成显式文件。工程收益很具体：身份可版本化、可回滚、可 review，漂移有基线可查。小文件、少字段、慢迭代——这大概就是给 AI 一个可进化身份的全部诀窍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/b9839aee5f801289.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/ac35814f71dfde30.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/4f1d9c0702b0a642.png)

