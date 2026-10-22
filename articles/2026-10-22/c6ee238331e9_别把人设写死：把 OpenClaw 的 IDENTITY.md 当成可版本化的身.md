---
title: 别把人设写死：把 OpenClaw 的 IDENTITY.md 当成可版本化的身份系统来养
feedId: 42286
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

OpenClaw 的 Agent 行为，很大程度由 workspace 里的几个 Markdown 文件决定：`SOUL.md` 管价值观和边界，`IDENTITY.md` 管名字、性格与语气，用户偏好则归 `USER.md`。这些文件不是编译进代码的配置，而是每次会话注入上下文的一部分——改文件就是改人格。这个设计给了我们一个机会：把 Agent 的身份从"一次性 prompt 调教"变成可以持续演进的工程对象。

## 问题

默认的 `IDENTITY.md` 往往只有一两行，用下来会有三个典型症状：

1. **跨渠道人格漂移**：Telegram 里冷静克制，CLI 里变成话痨，同一个 Agent 像两个人。
2. **长会话语气稀释**：跑了几轮自动化任务后，回复风格逐渐退化成通用助手腔。
3. **人设改不动**：想调整语气只能凭记忆手动改，改坏了也不知道哪次改的，没有演进机制。

## 做法

以 workspace 在 `~/.openclaw/workspace` 为例（路径以你的 `openclaw.json` 配置为准）：

1. **定骨架**。`IDENTITY.md` 固定四段：核心（名字 / 形象 / 一句话自我介绍）、语气（3 条以内硬规则）、示例（2–3 段标准回复，起 few-shot 作用）、演进记录（CHANGELOG 式几行）。
2. **控长度**。整文件压在 40 行以内。它每个会话都会进上下文，写成小作文是纯烧 token，还会稀释真正的关键指令。
3. **分层不重复**。"我是谁"放 `IDENTITY.md`，"我遵守什么"放 `SOUL.md`，"这个用户是谁"放 `USER.md`。同一句话只允许出现在一个文件里，否则改一处漏一处。
4. **设计进化机制**。把字段显式分成"不变项"（名字、底线）和"可进化项"（口头禅、示例回复）。每周让 Agent 基于近一周对话提交一份 `IDENTITY.md` 的修改 diff，人工 review 后合入。workspace 放 git 里，漂移了就 revert。

## 踩坑点

- **示例回复写太长**：模型模仿示例的冗长度比遵守任何规则都灵，示例必须短，宁可少给。
- **让 Agent 直接自改文件**：没有 diff review 的自改几乎必然人格漂移，而且方向一定是越来越讨好你。必须走"提案 → review → 合入"。
- **把用户信息写进 IDENTITY.md**：它对所有会话共享，多用户场景会串，用户相关内容一律进 `USER.md` 或 memory。
- **改完不验证**：维护一组固定探针 prompt（如"用一句话介绍自己""拒绝一个不合理请求"），每次改动后跑一遍对比输出，把人格漂移当回归 bug 对待。

## 可复用建议

- 把 `IDENTITY.md` 当代码管理：进 git、小步提交、有 CHANGELOG、有 review 流程。
- 语气规则用否定句最稳："不使用感叹号""不主动道歉"，比"要简洁专业"可执行得多。
- 心跳任务（HEARTBEAT）里挂一个每周身份提案步骤，让进化变成例行公事而非拍脑袋。
- 每月跑一次探针集，输出存档，肉眼可见地看到身份曲线。

## 总结

`IDENTITY.md` 的价值不在于"给 AI 起个名字"，而在于把人格从隐式、易漂移的调教经验，变成显式、可版本化、可 review 的工程资产。身份越稳定，你在自动化上才越敢放开手脚——这是养 Agent 和调 API 最大的区别。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/1804dda0dab48fac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/ce222a00f1fca013.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/a198e99a2c56abce.png)

