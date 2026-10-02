---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40137
source: 综合讨论
publishedAt: 2026-10-02
---

用 OpenClaw 跑了一段时间 agent 之后，你会发现决定它好不好用的，往往不是模型本身，而是 workspace 里那几个 markdown 文件。今天聊其中最小、也最容易被糊弄过去的一个：`IDENTITY.md`。

## 背景：身份是上下文的一部分

OpenClaw 每次会话都会把 workspace 中的一组文件注入系统提示：`AGENTS.md` 管流程规则，`SOUL.md` 管价值观和语气底色，`USER.md` 记用户偏好，而 `IDENTITY.md` 回答的是最基础的问题——"你是谁"。它通常只有几个字段：名字、生物原型（creature）、emoji、vibe、头像。文件很小，但直接影响 agent 的自我介绍、消息签名、通知风格和跨会话一致性。

## 问题：三个常见症状

1. **所有 agent 长一个样**。默认身份太通用，你接了三个 bot，开场白全是同一套话。
2. **身份散落在提示词里**。写死在启动脚本或某个 MCP server 的 system prompt 中，想改一次要翻好几处代码。
3. **变化不可追溯**。某天觉得 agent 语气不对，但说不清是哪次改动引入的——因为没有版本记录。

## 做法：五步把身份变成配置

1. **找到文件**：`~/.openclaw/workspace/IDENTITY.md`（多 agent 场景下每个 agent 有独立 workspace，别改错）。
2. **只填最小字段**，每个字段一句话：

```markdown
- Name: 小钳
- Creature: 寄居蟹工程师
- Emoji: 🦀
- Vibe: 话少，先给命令，再给原因
- Avatar: ./avatar.png
```

3. **划清边界**：语气规则进 `SOUL.md`，操作流程进 `AGENTS.md`，`IDENTITY.md` 只回答"我是谁"。混淆边界是最常见的起点性错误。
4. **纳入 git**。身份文件和代码一样要可回滚，commit message 写清动机（例如"通知场景太啰嗦，vibe 改短"）。
5. **建立进化循环**：跑两三周，记下真实摩擦点，一次只改一个字段，观察一段时间再决定去留。

## 踩坑点

- **写成小作文**。身份超过 20 行就开始稀释——它会被整体注入上下文，长 ≠ 有个性。
- **avatar 写了路径但文件不存在**，或 emoji 与 `SOUL.md` 里的自称对不上，会看到 agent 时不时"人格分裂"。
- **频繁换 vibe**。语气底色一月三改，用户对它的信任建立不起来。身份应该比功能更稳定。
- **把密钥、内网地址写进去**。这个文件的内容会进提示词，请按公开材料的标准来写。
- **多 agent 复制粘贴同一份身份**。给每个 agent 不同的 creature 和 emoji，通知和日志一眼可辨，排障成本低得多。

## 可复用建议

- 把 `IDENTITY.md` 当产品而不是配置：小步 diff、有理由、可回滚。
- review 节奏与真实痛感绑定（两周或一个月一次即可），不要为改而改。
- 新想法先进 `MEMORY.md` 观察一阵，被验证后再固化进身份。
- 团队协作时，identity 的改动应该走 code review——它和 prompt 一样影响行为。

## 总结

`IDENTITY.md` 是 OpenClaw 里最小的文件之一，但它是"一个聊天机器人"和"一个有名字的同事"之间的区别。它的价值不在于写了多少字，而在于给了你一个可版本化、可评审、可演化的锚点：agent 的其他一切都在变，身份的变化至少是有记录的。花十分钟认真填好它，往往比换一次模型省事得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/9a9f12fc31d71ec2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/6125658ce31bbe02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/d2074f4016659087.png)

