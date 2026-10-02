---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40136
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

OpenClaw 的 workspace 里有几个固定加载的 markdown：AGENTS.md 管行为，SOUL.md 管性格底色，USER.md 记用户信息，IDENTITY.md 管"我是谁"。这些文件在每次会话启动时被拼进 system prompt，相当于 agent 的常驻配置。IDENTITY.md 是其中最小的一个，也是最少被认真写的一个。

## 问题

没有身份文件的 agent 有几个实际痛点：

1. **人设漂移。** 换个渠道（Telegram、CLI、Discord）重开会话，模型按底层模型的默认习惯重新"猜"一个语气，名字和自称都不稳定。
2. **多 agent 混淆。** 跑两个 agent 分别对接不同群时，回复风格趋同，分不清谁在说话。
3. **调教不可沉淀。** 你在对话里纠正过它十次"别这么客套"，但下次会话全忘了——修正只存在于历史消息里，不在配置里。

根源是同一个：身份没有被当作一份可版本管理的配置。

## 做法

**1. 建文件。** 在 workspace 根目录（默认 `~/.openclaw/workspace/`）创建 IDENTITY.md，五行模板足够：

```markdown
# IDENTITY.md
- **Name:** 小钳
- **Creature:** 机械蟹
- **Emoji:** 🦀
- **Vibe:** 资深同事口吻，直接、简短，先给结论再给理由，默认不超过三句
- **Avatar:** assets/avatar.png
```

**2. 理解加载时机。** 文件在会话启动时读取，改完需要 `/new` 重开会话（或重启 gateway）才生效，进行中的会话不会热更新。

**3. 分清 IDENTITY 和 SOUL 的边界。** IDENTITY 是表层：叫什么、什么形象、什么语气；SOUL 是里层：价值观、做事原则。我踩过的坑是把"不要过度道歉"同时写进两个文件，措辞略有出入，模型随机遵循其中一个。现在只在一处写。

**4. 建立进化循环。** 把整个 workspace 纳入 git。每周复盘一次：哪些回复不像"它"，把结论压缩成一行改进 vibe 描述，小步提交。三个月下来，这个文件的 git log 基本就是 agent 的调教史。

**5. 多 agent 场景。** 每个 agent 独立 workspace、各自一份 IDENTITY.md，用不同 creature/emoji 区分，群聊里一眼能认出谁在发言。

## 踩坑点

- **写太长。** 我见过 60 行的人设，一半是"要幽默但不过分幽默"这种互相打架的描述，模型不会稳定遵循，还每会话白吃 token。控制在 15 行以内。
- **形容词不可验证。** "聪明、有趣、有温度"没法执行，改成可判定的规则："回复默认 ≤3 句；报错先贴日志片段"。
- **身份与习惯冲突。** vibe 写了"简洁"，但你天天让它出长报告，冲突时它跟着任务走。要么改文件，要么接受干活时角色让位于任务。
- **让 agent 自己改身份。** 试过让它"自己优化 IDENTITY.md"，两周后人设漂到不认识。身份编辑保持人工审批，最多让它提 diff。
- **改了没生效先查路径。** 确认配置里 agent 实际加载的 workspace 路径，别改错目录。

## 可复用建议

- IDENTITY.md 作为唯一事实源，其他文件不要复述语气类内容。
- 用"跨渠道测试"验证：新会话里问"你是谁、你怎么说话"，在两个渠道各跑一遍，回答一致才算及格。
- 身份字段保持统一的键值格式，机器可读，方便脚本做多 agent 状态看板。

## 总结

IDENTITY.md 是 OpenClaw 里成本最低、收益最确定的文件：十几行，换来跨渠道稳定的人设和可沉淀的调教过程。把它当 config-as-code 对待——小步改、走版本控制、定期复盘——身份就不再是每次会话碰运气的东西，而是随使用持续进化的资产。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/2121c55d89c37827.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/3fb5c180a770f1d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/139e30c6ca13a464.png)

