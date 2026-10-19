---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41987
source: 综合讨论
publishedAt: 2026-10-19
---

# 背景

OpenClaw 的 workspace 里有一组小而关键的 Markdown 文件：`SOUL.md`、`USER.md`、`IDENTITY.md`、`MEMORY.md`。其中 IDENTITY.md 决定 agent "是谁"——名字、语气、emoji、行为边界。它每次会话启动都会被装进系统上下文，相当于 agent 的静态引导段。

问题在于：大多数人写一次就再也不动它。

# 问题

两类典型失败：

1. **身份与行为脱节。** 跑了几个月，agent 的实际风格已经被 MEMORY 和日常调教带偏，IDENTITY.md 还停留在第一天的三行字。新会话读到的人设和真实行为对不上，排查问题时也没有参照系。
2. **放任自我改写导致漂移。** 反过来，如果让 agent 随便改自己的 identity，它会越改越讨好、越写越长，几周后边界约束消失，变成一个只会顺着你说的话痨。

本质矛盾：identity 既需要稳定（否则长期记忆和协作习惯无所适从），又需要进化（否则和实际使用脱节）。

# 做法

我的方案是把它当代码管：**agent 提议，人审核，git 记录**。

**第一步，最小骨架。** 控制在 30 行内，分两区：

```markdown
# Identity
## 不变项（改动需人工确认）
- 名字：Aka
- 硬边界：不代发消息、不删文件、不确定就问
## 可进化项
- 语气：克制、直接、少用感叹号
- 自称与 emoji：🦀
- 汇报格式：结论先行
```

**第二步，进化闭环。** 每两周固定一次：开一个独立会话，让它回顾这段时间的日志，只对"可进化项"输出 unified diff 建议 + 理由；你在 git 里 review，merge 才生效。"不变项"默认锁死，半年回顾一次。

**第三步，回归测试。** 改完用 3~5 个固定 prompt 跑一遍——一个任务型、一个边界测试型、一个闲聊型——语气和边界没跑偏才算过。

# 踩坑点

- **写太长。** identity 每轮都占上下文，越长权重越稀释，50 行是实用上限。
- **把事实记忆塞进 identity。** "用户在做某项目"这类内容会腐烂，归 `USER.md` / `MEMORY.md` 管，别混。
- **并发写冲突。** cron / heartbeat 任务和主会话同时改文件会互相覆盖。改 identity 的写权限只留给 review 会话，其他一律只读。
- **改名太随意。** 旧记忆里全是旧名字的自指，agent 引用历史时会混乱。要改就在不变项里走一次正式变更，并接受记忆检索的过渡期噪声。

# 可复用建议

- 所有变更走 commit，message 写清动机。半年后你能回答"它为什么变成现在这样"。
- 频率低于你想象的才正确：两周到一个月一次足够，高频修改本身就是漂移源。
- diff-only 原则：不接受"重写整个文件"，只接受局部 diff，review 成本才可控。
- 多 gateway / 多实例的话，identity 主文件唯一，实例差异下沉到各自的 profile，不要 fork 出两份人格。

# 总结

IDENTITY.md 不是一次性设定文档，而是你和 agent 之间的活契约。小步、可版本化、人审的进化机制，比"让它自己成长"的浪漫想象可靠得多。身份是你为长期协作定义的接口——接口可以升级，但要走变更流程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/58c924d2209ddbfa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/9723880f9abb2be8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/b5f037118cdde062.png)

