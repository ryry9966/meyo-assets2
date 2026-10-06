---
title: HEARTBEAT.md：让 OpenClaw Agent 从被动应答到主动巡检
feedId: 40635
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

多数人把 Agent 当问答机用：提问、回答、关闭。但如果跑的是一个 7×24 挂在 VPS 上的个人助理 Agent，它更应该像值班运维——定时巡检，发现异常才说话。OpenClaw 的 heartbeat 机制就是干这个的：每隔一段时间自动唤醒 Agent，把工作区里的 `HEARTBEAT.md` 喂给它，由它决定"有话要说"还是"无事发生"。

## 问题

这里有两个常见的坑。一是干脆不开主动能力，Agent 退化成纯一问一答，提醒、巡检、日报全靠你自己记得去问；二是把"主动"理解成"定时输出"，结果每天被 Agent 用低价值消息刷屏，最后只能整个关掉。核心矛盾在于：怎么让 Agent **有事主动说，没事闭嘴**。

## 原理

heartbeat 的运行方式很朴素：gateway 按配置间隔（默认 30 分钟）唤醒 Agent，上下文中带上 `HEARTBEAT.md` 正文。系统约定了一个哨兵值：如果 Agent 判断没有任何需要用户知道的事，只回复 `HEARTBEAT_OK`——这条回复不会投递到任何渠道，用户侧完全无感；否则它会组织一条简短消息发到配置的目标渠道。"说不说"的判断权在模型手里，而控制判断标准的，就是你写在文件里的规则。

## 做法

1. 编辑 `~/.openclaw/workspace/HEARTBEAT.md`，把任务写成带判定条件的检查项；
2. 在 `openclaw.json` 里配置 `agents.defaults.heartbeat.every`（如 `"30m"`），用 `heartbeat.target` 指定投递渠道；
3. 用 `openclaw heartbeat status / enable / disable` 确认状态，先用短间隔（如 5m）观察几轮再调回正常频率；
4. 多步任务在文件里写明完成后标记 `HEARTBEAT_DONE`，避免每轮重复执行。

示例片段：

```markdown
- 每轮检查：若 ~/logs/error.log 自上次以来新增 FATAL/ERROR，
  摘要前 3 条发我；否则 HEARTBEAT_OK。
- 每天 9 点后第一轮心跳：汇总昨日未读邮件中含"发票/账单"
  关键词的内容，一条消息发我。
- 一次性任务：确认备份产物存在后，把本行替换为 HEARTBEAT_DONE。
```

## 踩坑点

- **成本**：沉默不等于免费。每轮心跳都是一次真实的模型调用，即使回复 `HEARTBEAT_OK` 也烧 token。固定时刻的事（每天准点日报）交给 cron 类工具做，heartbeat 只留"条件触发"类任务，间隔放到 30–60 分钟。
- **噪音**：任务描述含糊（"帮我盯着点服务器"）会显著拉高误报率，Agent 会开始跟你闲聊。每个任务写清三件事：触发条件、动作、什么情况下保持沉默。
- **上下文膨胀**：`HEARTBEAT.md` 每轮都进上下文，别写成操作手册。正文控制在十几行内，细节放 MEMORY.md 或独立文件引用。
- **多步任务卡死**：需要调工具、分几轮完成的任务，如果没写 `HEARTBEAT_DONE` 收尾逻辑，下一轮心跳会重跑，你会收到重复通知。
- **心跳不是精确调度**：重启 gateway 计时器会重置，触发有抖动，时间敏感的提醒不要指望它。
- **渠道选择**：主动消息打到 WhatsApp 这类渠道容易触发风控，target 建议指到低噪音渠道。

## 可复用建议

- **三段式任务模板**：触发条件（何时查、满足什么）→ 动作（做什么）→ 输出规则（仅在 X 时发消息，否则 `HEARTBEAT_OK`）。
- **日报模式优于事件模式**：让 Agent 累积观察，固定窗口一次性输出，而不是每次命中都响。
- **心跳日志**：让 Agent 每次真正发言时往一个日志文件追加一行，周末扫一眼就能发现误报模式，再回头收紧条件。
- **定期清理**：完成的一次性任务、失效的监控目标及时删掉——文件越短，越省 token，误报越少。

## 总结

`HEARTBEAT.md` 的价值不在"能定时跑任务"，而在把"何时该打扰用户"的判断标准显式写进文件、交给模型执行。写得越像 SLA，Agent 就越像一个可靠的值班员：安静，但有事必报。建议从一两个高价值、低频率的巡检任务起步，跑稳了再扩。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/eba7fc5922c0bdbd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/f7d033a11bdd473b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/7337d38019468f43.png)

