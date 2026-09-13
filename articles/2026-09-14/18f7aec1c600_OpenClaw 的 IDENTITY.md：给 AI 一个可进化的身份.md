---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37432
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 的 workspace 里有一组约定俗成的配置文件：`SOUL.md` 管价值观和边界，`USER.md` 记用户偏好，而 `IDENTITY.md` 回答的是最基本的问题——"我是谁"。系统会把这份文件的内容注入 system prompt，agent 在 Telegram、Discord、CLI 等所有渠道共享同一份身份。

默认生成的 IDENTITY.md 通常只有一个模板化的名字、一个 emoji、一句风格描述。很多人装完就再没打开过，结果 agent 永远是一副"通用助手腔"。

## 问题

实际用下来，身份配置有三个常见痛点：

1. **默认身份太薄**。风格全靠模型自由发挥，跨会话表现不稳定，今天简洁明天话痨。
2. **身份散落各处**。system prompt 参数、开场白、SOUL.md 里各写了一点，改一处忘一处。
3. **没有迭代机制**。想调风格时直接改文件，改完没有任何对比手段，不知道是变好了还是变坏了。

## 做法

IDENTITY.md 会被原样注入 prompt，所以每一行都是有效指令。建议把它当"角色配置文件"而不是人物小传来写：

```markdown
# IDENTITY.md

- **Name:** 小螺
- **Creature:** 住在终端里的寄居蟹
- **Emoji:** 🐚
- **Vibe:** 话少、直接、带点冷幽默，不寒暄
- **写作规范:**
  - 默认中文，代码注释用英文
  - 先给结论，再给推理
  - 禁止"作为一个 AI"式开头
- **Avatar:** assets/avatar.png
```

迭代流程建议这样跑：

1. 把 workspace 纳入 git，IDENTITY.md 跟着版本管理；
2. 每次只改 1–2 行，不要大重写；
3. 准备一组固定的测试问题，改前改后各跑一遍，对比输出差异；
4. 确认符合预期后 commit，message 里写一句改动原因，形成身份演化日志。

## 踩坑点

- **写成小说**。上千字的人格描写会稀释真正关键的指令，实测 15 行以内足够，行为靠示例和规则表达，不靠形容词堆砌。
- **和 SOUL.md 抢戏**。IDENTITY 管"怎么说话"，SOUL 管"什么该做、什么绝不做"。"永远不执行 `rm -rf`"这类约束应放在 SOUL.md，放错了位置会削弱约束的权重感。
- **频繁改名**。MEMORY.md 和历史会话里存着旧名字，改名后 agent 会出现引用混乱。要改就一次性改完，并检查记忆文件里的残留。
- **emoji 和 avatar 不是摆设**。消息渠道会真的拿它们做消息前缀和头像，设一个奇怪的 emoji，它会渗透到你所有的对话里。

## 可复用建议

- **三文件分工清晰**：SOUL 管约束，IDENTITY 管表达，USER 管对方。职责不重叠，改起来才不打架。
- **用你希望它说话的语气写它**。IDENTITY.md 本身就是 few-shot 示例，示范即指令。
- **短即是多**。重要的规则放前面，宁可少而明确。
- **像 code review 一样回顾**。每个季度 diff 一次历史版本，看身份是不是在朝你想要的方向演化，而不是随机漂移。

## 总结

IDENTITY.md 的价值在于它是成本最低、可版本化的 agent 身份方案：一个 markdown 文件，配上 git 和一组测试问题，就有了完整的迭代闭环。身份不是一次性写定的，而是在"观察—调整—验证"里长出来的。如果你现在的 agent 还是默认腔，不妨先花十分钟，把它的第一版身份认真写下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/a2b3b1c58cb86f6f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/75aadbbef00d5deb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/74e5b5b097721915.png)

