---
title: USER.md 实践：别让 Agent 每次会话都重新认识你
feedId: 39510
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 agent 默认是"金鱼记忆 + 陌生同事"的组合：每次会话几乎从零开始，它不知道你的时区、主力技术栈、命名习惯，也不知道你反感哪种回复风格。SOUL.md 解决的是"agent 是谁"，而 USER.md 解决的是"你是谁"——这个文件位于 workspace（默认 `~/.openclaw/workspace/USER.md`），每次会话作为常驻上下文注入模型。它可能是整套配置里性价比最高的一个文件，但很多人要么空着，要么塞成了流水账。

## 问题

没有 USER.md 时，损耗是隐性但持续的：

- 每隔几天重复一次"我用 UTC+8，别在半夜安排提醒"；
- agent 默认用礼貌冗长的客服腔，而你只想看结论；
- 让它写脚本，它猜你用 Python，实际你的环境是 Go + TypeScript；
- 提醒、日程类任务在错误的时间触发。

这些不是能力问题，是上下文问题。上下文问题就该用文件解决。

## 做法

**1. 建立画像骨架。** 首次创建不追求完整，先写你希望 agent 永远知道的 5–8 条事实：

```markdown
# USER

## 基本信息
- 称呼: 老张
- 时区: UTC+8，工作时间 09:00-19:00

## 技术环境
- 主力: Go / TypeScript，脚本用 Bash，不爱用 Python
- 机器: macOS + Linux 服务器，包管理 brew / apt

## 沟通偏好
- 先给结论再给细节，不要开场白和客套
- 代码注释与 commit message 用英文

## 红线
- 不要自动执行 rm / drop 类命令
- 未经确认不对外发送任何消息
```

**2. 让 agent 参与维护。** 对话中出现稳定事实时（比如"以后统一用 pnpm"），直接说"把这条偏好固化到 USER.md"，让它自己改，你 review diff。

**3. 定期修剪。** 每月扫一遍，删掉过期内容（换掉的项目、废弃的工具链）。这个文件是画像，不是日志。

## 踩坑点

- **写太长。** USER.md 每次会话都占 token，几百行的"个人百科"会稀释重点，模型反而记不住。控制在几十行内，像写 onboarding 文档一样克制。
- **塞入任务状态。** "这周要修 XX bug"属于 MEMORY 或会话记忆，写进 USER.md 三天后就变成噪音。这里只放稳定事实。
- **写入敏感信息。** 密码、API key、证件号不要出现——它会进入每一次模型调用。
- **改完不验证。** 修改后新开会话，直接问"你现在知道我的时区和主力语言吗"。另外注意：部分子 agent 如果没挂载同一个 workspace，读不到这份文件，跨 agent 场景要单独确认注入路径。
- **职责混淆。** 对 agent 行为的指令放 SOUL.md 或 AGENTS.md，USER.md 只描述你。混在一起后期没法维护。

## 可复用建议

- 把 workspace 纳入 git 或 dotfiles 管理，画像变更可追溯、可回滚；
- 工作助手、侧项目助手分开 workspace，各自维护独立 USER.md；
- 固定一段"自我介绍 prompt"：在新环境让 agent 基于对话主动起草 USER.md 初稿，你只做删改；
- 字段宁缺毋滥：agent 真正高频使用的信息通常不超过 15 条。

## 总结

USER.md 本质上是一份"给每次会话的简报"：成本是几十行 Markdown，收益是省掉重复沟通和大量错误假设。原则和写任何配置文件一样——短、准、只放稳定事实、定期维护。如果你还没建这个文件，建议今晚花十分钟写第一版，从时区和沟通偏好开始，这两个字段收益最直接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/68d4b91611ea5f71.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/4364e1eda531fe05.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/a24b6431906cb227.png)

