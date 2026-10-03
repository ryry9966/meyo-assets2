---
title: 给 Agent 一份 USER.md：让它先知道你是谁，再干活
feedId: 40240
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

玩 Agent 的人大多很在意项目侧的上下文：CLAUDE.md、AGENTS.md 写得细致入微，告诉模型仓库怎么跑。但很少有人给“用户自己”写一份。结果就是：Agent 对你的了解，全靠聊天记录里零散的碎片。

OpenClaw 的 workspace 里有个约定俗成的文件：USER.md。它和 AGENTS.md（干活的规矩）、SOUL.md（人格）、MEMORY.md（流水记忆）分工不同，回答的是最基础的问题——“正在和我协作的这个人是谁”。每次会话它会被注入上下文，相当于给 Agent 挂了一份用户态配置。

## 问题

没有 USER.md 时，常见症状：

- 反复解释环境：macOS、fish shell、项目用 pnpm 不用 npm，说了一遍又一遍；
- Agent 按默认习惯干活：长篇解释、英文 commit message、自作主张加注释；
- MEMORY.md 越积越厚，关键事实淹没在对话流水里；
- 换设备或重建 workspace 后，所有积累归零。

## 做法

1. 在 workspace 根目录建 USER.md，按固定分区写，短句加 bullet，少写散文：
   - **身份与环境**：OS、常用 shell、时区、语言偏好；
   - **技术栈**：包管理器、编辑器、代码风格、明确不用的东西；
   - **协作方式**：回答详略、先给方案还是直接动手、commit/PR 规范；
   - **红线**：绝不自动 push、不动某目录、不替你做某类决定；
   - **项目索引**：一行一个项目加路径，方便 Agent 跳转。
2. 控制预算：目标 60~100 行、1k token 以内。它每轮都占上下文，长度就是成本。
3. 用 git 管理，改动走 diff；每月固定 review 一次，删掉过时项。
4. 冷启动技巧：让 Agent 反过来采访你——“问我十个问题，帮我起草 USER.md”，生成草稿后人工裁剪，比从零写快得多。

## 踩坑点

- **别写易变状态**。“本周在做 X”是 MEMORY 或 notes 的事，写进 USER.md，一个月后就是误导。
- **别放敏感信息**。明文文件会随请求发给模型 API，证件号、密钥、住址一律不进。
- **别写情绪化指令**。“多夸夸我”这类要求会系统性劣化输出质量，亲测。
- **注意优先级冲突**。约定好：项目规则（AGENTS.md）> 用户偏好（USER.md）> 模型默认，在 AGENTS.md 里写一句话声明即可。
- **别让 Agent 自动改 USER.md**。我见过半年膨胀三倍、全是客套废话的案例，变更必须由人确认。

## 可复用建议

- 每条偏好尽量带一个理由，例如“用 pnpm——团队锁了版本”，日后 review 才知道规则还成不成立。
- 把“事实”和“偏好”分区，review 时重点看偏好区，事实区让 Agent 帮你核对。
- 新机器迁移 = clone 一个 repo，五分钟恢复全部个性化。
- 结构可以在团队内互相抄，内容别抄：偏好是高度个人的东西。

## 总结

USER.md 是典型的低投入高回报：一小时的写作，换来 Agent 不再反复问你“用什么包管理器”。它不神秘，本质是把散落在聊天记录里的用户画像，收敛成一份受版本控制的配置文件。Agent 不会读心，但它会读文件——把“你是谁”写成一份它每次都会看的文档，协作质量立刻不一样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/3a0e7ab10a6d4665.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/0b55328ffa818833.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/71fcd436b13bc3f9.png)

