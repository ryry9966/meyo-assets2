---
title: HEARTBEAT.md 实践：让 Agent 主动做事，而不是等你提问
feedId: 42059
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 里的 agent 默认是被动模型：你不发消息，它就不动。对问答类场景这没问题，但"每天早上汇总一次 issue""日志出现异常要主动喊人"这类需求，被动模式天然缺位。OpenClaw 内置了 heartbeat 机制：网关按固定间隔向 agent 发送一次内部唤醒（默认约 30 分钟，可配置），agent 被要求读取工作区根目录的 `HEARTBEAT.md`——有事就执行，没事就回一个 `HEARTBEAT_OK` 并保持静默，不会推送到你的聊天渠道。

## 问题

用外部 cron + MCP 脚本当然也能实现主动任务，但配置散落在系统各处，任务和 agent 的上下文是割裂的。HEARTBEAT.md 的价值在于把"什么时候、满足什么条件、做什么、怎么汇报"收敛成一个 agent 自己能读的 Markdown 文件：改动即生效，不用碰调度器，也不用为每件事单独写脚本。

## 做法

1. **开心跳、调间隔**。先用较大间隔（比如 60 分钟）跑几天，观察日志里实际触发次数和 token 消耗，再决定收紧。
2. **创建 `HEARTBEAT.md`**，用 checklist 写法，每条任务固定三段：触发条件、动作、汇报策略。示例：

```markdown
## 每日
- 每天 09:00（Asia/Shanghai）：汇总仓库过去 24h 新增 issue 的标题，
  发到 Telegram；没有新增就不发。

## 巡检
- 每次心跳：检查 gateway.log 最近 100 行，出现 ERROR 且 30 分钟内
  未通知过才发一条摘要；其余情况一律回 HEARTBEAT_OK。
```

3. **让动作可执行**。条目里引用的能力要和已接入的 MCP 工具对齐（fetch、filesystem、github 之类），agent 才有手可用。
4. **看日志验证**。空转时确认日志里是 `HEARTBEAT_OK` 且没有外发消息，说明汇报策略写对了。

## 踩坑点

- **任务写得模糊**。"帮我盯着服务器"会让 agent 每次心跳都在猜，要么不动作要么乱动作。必须写成可判定条件：阈值、时间窗、去重规则。
- **当成真 cron 用**。心跳是 LLM 驱动的软调度，可能提前也可能延后几分钟。强时效任务老老实实上系统 cron，HEARTBEAT.md 只放"软"任务。
- **输出失控**。不限定汇报策略的话，巡检类任务可能每次都把正常结果刷进聊天。每条都要写清"仅异常时通知，正常不打扰"。
- **去重缺失**。"日志有 ERROR 就通知"不加时间窗，一次故障能给你连发一晚上。
- **文件膨胀**。整个文件会随唤醒 prompt 进入上下文，写长了每次心跳都在烧钱。已完成的条目及时删。

## 可复用建议

- 每条任务强制三段式：**触发条件 / 动作 / 汇报策略**，缺一不收。
- 显式写上"无事则 HEARTBEAT_OK"——这是省 token 最有效的一句话。
- 先大间隔跑一周，用日志统计真实触发频率，再决定收紧还是放宽。
- HEARTBEAT.md 管"周期性 + 可判定"的事，精确调度交给系统 cron，两者是分工不是替代。

## 总结

HEARTBEAT.md 解决的不是调度精度，而是主动性的入口问题：agent 由此拥有一个固定、由你亲手维护的待办清单，每次醒来先看它。写小、写准、写清汇报策略，它就能成为可靠的无人值守层；写糊了，它就是一个定时烧 token 的随机数发生器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/952ecdc64f15b68e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/8bae805a25ea1a39.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/a8ee0062b89fb927.png)

