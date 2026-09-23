---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38641
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 workspace 里有几个约定俗成的 markdown：`AGENTS.md` 管行为守则，`SOUL.md` 管价值观，`USER.md` 管用户是谁，而 `IDENTITY.md` 管"我是谁"。它默认只有几行——名字、物种、vibe、emoji、头像——每次会话开始时被注入 system prompt。文件很小，作用不小：它决定了同一个 agent 在不同会话、不同频道里表现出的连续性。

## 问题

没写或随手写的后果很具体：

- **人格随机**：今天话痨明天冷淡，跨会话没有连贯性；
- **多 agent 撞脸**：两个助手都叫 Assistant，群聊里分不清谁在说话；
- **改人格只能动全局 prompt**，牵一发动全身，改完还得重启会话验证。

本质上，"这个 agent 是谁"没有被当成一个可管理的资产，而是散落在 prompt 里的临时台词。

## 做法

1. **定位文件**。单 agent 默认在 `~/.openclaw/workspace/IDENTITY.md`；多 agent 时每个 workspace 一份，天然隔离。没有就新建。
2. **最小模板起步**：

```markdown
# IDENTITY
- Name: 小螺
- Creature: 住在终端里的机械蜗牛
- Vibe: 冷静、话少、先给结论
- Emoji: 🐌
- Avatar: assets/snail.png
```

3. **加两段自定义**：语气边界（什么场合不开玩笑、哪些频道可以放松）和表达偏好（列表优先、改代码先给 diff）。
4. **让它进化**：文件尾加 `## Changelog`。每周让 agent 复盘近期对话，回答"哪句话不像我"，把修正写成一行追加进去。也可以让 agent 自己起草初版，你只做编辑。
5. **用 git 管 workspace**：改身份 = 改文件 = 提交 diff，人看一眼再合入。新人格先丢到测试频道跑两三天，观察语气再定稿。

## 踩坑点

- **别写长**。IDENTITY.md 每个会话都吃 token，控制在 30 行以内。形容词堆砌不如行为描述："回复不超过三句"比"简洁高效"管用。
- **别越界**。价值观归 `SOUL.md`，用户偏好归 `USER.md`，`IDENTITY.md` 只管身份。内容重叠时模型会在冲突处随机选边，表现就是"人格不稳定"。
- **别开放无监督自改**。自改必须落到 changelog + git diff，人工过目再提交，否则一两周后它可能给自己加戏成"赛博诗人"，而且你查不到是哪次改的。
- **小字段有大气场**：换 emoji 之后语气确实变了——可能是巧合，但这类小字段值得当成实验变量来对待。

## 可复用建议

- 身份当代码：有版本、有 review、有回滚；
- 行为描述 > 形容词；
- 每月做一次身份 retro，让 agent 参与定义自己，但决定权在你手里；
- 多 agent 场景靠 workspace 文件区分身份，别靠 prompt 里喊"你是二号助手"。

## 总结

`IDENTITY.md` 把"agent 是谁"从 prompt 里一句临时台词，变成了 workspace 里有版本、可 review、可回滚的资产。身份不是一次性的出厂设置，而是和 agent 相处过程中慢慢攒出来的东西——这也正是它值得用工程方法去维护的原因。从一个五行的模板开始，比想象中容易。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/67694b95fad883e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/5df229726adb1ee0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/fad93b693e6944db.png)

