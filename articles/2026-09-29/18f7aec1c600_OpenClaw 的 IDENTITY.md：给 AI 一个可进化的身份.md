---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 39490
source: 综合讨论
publishedAt: 2026-09-29
---

# 背景

跑 OpenClaw 一段时间后，你会发现 agent 的“人格”是散落的：system prompt 里有一段、记忆文件里有一段、临时调 prompt 时又加一段。换个名字要翻配置，两个实例在同一个群里分不清谁是谁。IDENTITY.md 就是把这件事收口的文件——工作区根目录（默认 `~/clawd`）下，集中声明“我是谁”。

# 问题

实际用下来有四类麻烦：

1. 人设硬编码在 prompt 或配置里，每次调整都要动代码，diff 噪音大；
2. 模型即兴发挥，不同会话的自我介绍不一致，名字和语气会漂移；
3. 多实例共用默认人格，日志和群聊里难以区分；
4. 人设演进没有留痕，改坏了不知道改了什么，想回滚也难。

# 做法

**1. 写一个最小版 IDENTITY.md：**

```markdown
- Name: 阿钳
- Creature: 机械钳虾
- Vibe: 克制、直接、偶尔冷幽默；中文优先，术语保留英文
- Emoji: 🦞
- Avatar: assets/avatar.png
```

**2. 分层。** IDENTITY.md 管“我是谁”，SOUL.md 管“怎么做事、价值观”，USER.md 管“服务对象”。行为规则别塞进 IDENTITY.md，否则职责重叠、两头维护。

**3. 版本化。** workspace 本身是 git 仓库，identity 变更单独 commit，message 写清动机，比如 `identity: 降低表情包频率`。

**4. 进化回路。** 一两周看一次真实对话日志 → 列出人格偏差（太话痨、太卑微、术语混用）→ 只改一到两处 → commit → 开新会话观察效果。小步提交，拒绝一次性重写。

**5. 多实例。** 每个实例独立 workspace，用不同 Emoji 做快速识别。

注意：修改对**新会话**生效，进行中的会话不会热加载，验证时别被旧会话误导。

# 踩坑点

- **文件膨胀**：IDENTITY.md 每个会话都注入上下文，膨胀直接变成 token 成本，建议压在 30 行内，密度优先于面面俱到；
- **无人把关的自我修改**：可以让 agent 在 heartbeat 里起草人设调整建议，但必须人工 review 后再合并，否则怪癖会被自我强化不断放大；
- **Avatar 路径**：gateway 跑在服务器上，引用你本机的绝对路径必然找不到，资源放进 workspace 用相对路径；
- **多实例复制模板忘改名**：群里两个同名 agent 互相抢答，场面一度尴尬；
- **中途大改人设**：老用户会感觉“性格突变”，重要变更最好提前公告或在测试频道灰度。

# 可复用建议

- 把 IDENTITY.md 当代码对待：固定字段顺序、语义化 commit、变更走 diff review；
- 保留 v1 作为回滚锚点，提交历史本身就是一份人设 changelog；
- 这套“身份 = 版本化 Markdown”的模式不限于 OpenClaw，迁移到其他 agent 运行时或 MCP 服务同样成立：身份显式、可版本、可回滚，比散落在 prompt 里可靠得多；
- 新实例启动时，把 IDENTITY.md / SOUL.md / USER.md 三件套作为 bootstrap checklist 固定下来。

# 总结

IDENTITY.md 是个很小的文件，但它把“AI 是谁”从口头约定变成了受版本控制的资产。显式声明、小步演进、人工把关——做到这三点，人设就是一个可维护的工程对象，而不是随缘的玄学。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/08b3a41fd877a747.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/47c91f047e401fcd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/fddb080952c134c0.png)

