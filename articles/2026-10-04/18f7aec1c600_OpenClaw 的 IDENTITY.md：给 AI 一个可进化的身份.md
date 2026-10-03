---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40323
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

OpenClaw 的 workspace 里有几个不起眼但很关键的 Markdown 文件：`AGENTS.md` 管行为守则，`SOUL.md` 管价值观和语气，`USER.md` 记用户偏好，而 `IDENTITY.md` 管的是"我是谁"。每次新会话启动，这些文件会被拼进 system prompt，Agent 的自我认知就从这几百字里来。

很多人部署完就再没打开过这个文件，默认身份一路用到底。这挺可惜——IDENTITY.md 是整个 prompt 体系里改动成本最低、反馈最直接的一块。

## 问题

不认真维护身份文件时，常见三种症状：

1. **自我介绍漂移**。问"你是谁"，WhatsApp 里一个说法，Telegram 里另一个说法。因为身份不是显式定义的，模型每次即兴发挥。
2. **身份和灵魂打架**。有人把行为规则写进 IDENTITY.md，又把同类内容写进 SOUL.md，两处措辞不一致时，Agent 表现不稳定。
3. **不可追溯**。靠聊天里说"你以后叫我……"来改，重启会话可能失效，团队协作更没法同步。

## 做法

定位 workspace（默认 `~/.openclaw/workspace`，可在配置中修改），编辑 IDENTITY.md。我的写法是控制在 10 行以内：

```markdown
# IDENTITY.md

- **Name:** 阿钉
- **Creature:** 一只铁皮信鸽
- **Vibe:** 话少、直接、爱用列表
- **Emoji:** 🪶
- **Avatar:** assets/avatar.png
```

几条原则：

- **IDENTITY 管"是谁"，SOUL 管"怎么做"**。名字、形象、口头禅进 IDENTITY；语气尺度、做事原则、拒绝策略进 SOUL，两边不重复。
- **Avatar 用相对路径**指向 workspace 内文件，别引用外链，部分通道拉不到外链图片。
- 改完开新会话验证。当前会话不会热加载，这是设计使然，不是 bug。
- 把 workspace 整个放进 git。身份变更走 commit，提交信息写清改了什么、为什么改。一个月后回看 diff，你会得到一份 Agent 人设的演进史——这就是"可进化"的含义：每次调整都有据可查。

验证方法很土但有效：在新会话里问"用一句话介绍你自己"，换三个通道各问一次，回答一致即通过。

## 踩坑点

- **别写成角色小说**。见过有人写两千字的人设小传，token 成本上去了，模型反而抓不住重点。IDENTITY.md 是身份证，不是传记。
- **中文名在个别通道显示异常**，emoji 兜底很实用，它是跨通道最稳定的视觉锚点。
- **多 Agent 部署时 workspace 是隔离的**，别在 A 的目录里改完 B 的文件还以为生效了，先确认路径。
- **Avatar 别太复杂**，部分 IM 通道会压缩头像，高对比度的简洁图形存活率最高。

## 可复用建议

- 团队共用 Agent 时，IDENTITY.md 的修改收进 git review，和改代码走同一流程。
- 每月做一次"身份对账"：让 Agent 自我总结，和文件内容比对，有偏差就修文件，而不是靠聊天纠正。
- 把自己的模板沉淀成 snippet，新开 Agent 五分钟内出第一版身份，后续再迭代。

## 总结

IDENTITY.md 解决的不是能力问题，而是一致性和可维护性问题。它让"这个 Agent 是谁"从模型的即兴发挥，变成仓库里一行行有版本记录的配置。改动成本低、反馈快，很适合作为你优化 OpenClaw 的第一步。名字未必经常换，但"身份是有版本的"这个认知，值得从第一天就建立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/90958e1b01158d4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/cbb2648f263b03f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/a1c321c34f9a93cb.png)

