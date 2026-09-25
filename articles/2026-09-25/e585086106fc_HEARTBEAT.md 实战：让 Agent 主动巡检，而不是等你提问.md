---
title: HEARTBEAT.md 实战：让 Agent 主动巡检，而不是等你提问
feedId: 38965
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

多数 Agent 的使用方式仍是"你问一句，它答一句"。OpenClaw 里有一个容易被忽略的机制：心跳（heartbeat）。Gateway 按固定间隔（默认 30 分钟，可配置）唤醒一次 Agent，Agent 随后读取工作区里的 `HEARTBEAT.md`。文件为空时只回一个 `HEARTBEAT_OK` 并跳过本轮，几乎零成本；文件里有内容时，它会按清单逐项执行——读文件、调工具、发通知。这就是把 Agent 从被动应答改造成主动巡检的入口。

## 问题

实际用下来，大多数人的第一版 HEARTBEAT.md 都写偏了：要么写成聊天式的"帮我留意一下服务器"，要么塞满一次性任务。结果是每半小时烧一轮 token，深夜还往聊天窗口推送一堆"一切正常"。心跳的价值在**持续的条件检查**，不在定时闲聊。

## 做法

1. 找到工作区的 `HEARTBEAT.md`（Gateway 首次启动会生成占位文件）。
2. 在 `openclaw.json` 里确认间隔：`agents.defaults.heartbeatIntervalM`，建议先设 30–60 分钟。
3. 用"条件 → 动作 → 汇报规则"的结构写清单，例如：

```markdown
# Heartbeat checklist
- 读取 ~/monitor/status.txt；若为 DOWN 且 ~/monitor/.notified 不存在：通知我，并创建该标记文件
- 统计 ~/inbox/ 文件数；大于 5 才汇报，否则静默
- 22:00–08:00 之间只写日志，不推送通知
- 以上都不命中时，回复 HEARTBEAT_OK
```

4. 验证：最简单的方式是直接在对话里让 Agent"立刻执行一轮心跳检查"，观察它的动作顺序、日志与 token 消耗，再逐步收紧规则。

## 踩坑点

- **当聊天 prompt 写**：描述越模糊，Agent 每轮自由发挥越多，token 越贵。每一条都应是可判定、可执行的动作。
- **缺"异常才汇报"**：每轮都说"一切正常"，一周后你就会无视它。
- **缺去重状态**：服务宕机一小时会收到 N 次同样告警。用标记文件落一个"已报告"状态，恢复后再清除。
- **与 cron 混淆**：固定时间点的事（每天 9 点发摘要）交给 cron；心跳适合"不知道何时发生、但要持续盯着"的状态类检查。时间驱动用 cron，状态驱动用心跳。
- **文件过长**：整个文件每轮都会注入上下文，控制在 20 行以内，重逻辑放到它引用的脚本里。
- **多 Agent 场景**：每个 Agent 有自己的工作区和 HEARTBEAT.md，别指望一份清单管所有实例。

## 可复用建议

- 把它当 **ops runbook** 写，不是 prompt：条件明确、动作具体、汇报条件清晰。
- **例外汇报（report by exception）** 是核心原则：正常即静默。
- 让心跳动作落日志文件，事后可审计它每轮做了什么。
- 需要跨轮记忆的判断（如"连续 3 次失败才告警"），显式写入磁盘状态文件，不要指望模型记得上一轮。
- 间隔宁可先粗后细，观察一两周成本和噪音后再调。

## 总结

HEARTBEAT.md 本质是给 Agent 一个固定频率的巡检循环：写得像 runbook，它就是一名不知疲倦的值班员；写得像闲聊，它就是噪音源。从一两个高价值的状态检查起步，配上异常汇报和去重，你会发现主动式 Agent 的价值不在多能说，而在**该说话时才说话**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/373fe613be40d120.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/4416fe72df99d2be.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/e05031170694f282.png)

