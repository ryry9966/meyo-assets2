---
title: cron vs heartbeat：OpenClaw 两种定时任务的选型实践
feedId: 39531
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景：两个看起来重叠的东西

OpenClaw 里有两条"到点干活"的路径：**cron**（定时任务）和 **heartbeat**（心跳）。刚上手时很容易纠结——每天早上 9 点推一份日报，用哪个？让它盯着收件箱、有事再叫我，又用哪个？混着用一段时间后，我总结出一套比较稳定的分工，记录在这里。

## 问题：先分清"几点做"和"要不要做"

两者的本质差异只有一条：

- **cron**：你定死时间，到点唤醒 agent。每次运行是一个**隔离的子会话**，默认看不到主会话的上下文，跑完即结束。它回答"几点做"。
- **heartbeat**：网关按固定间隔（默认 30 分钟）向主会话注入一条心跳提示，agent 读取工作区的 `HEARTBEAT.md`，**自行判断**有没有事做——没事就返回 HEARTBEAT_OK 跳过。它回答"要不要做"。

一句话：cron 是闹钟，heartbeat 是值班。

## 我的分工做法

1. **确定性调度走 cron**：日报、周报、固定时间拉数据。用 cron 表达式定义，delivery 设为 announce，结果直接推到指定会话。
2. **持续性关注走 heartbeat**：盯 CI 状态、盯邮箱、盯一个长任务的进度。这类事没有固定时间点，"有事再说"比"每小时准点跑一次"省得多。
3. **长流程用 cron 触发 + 文件衔接**：cron 会话之间没有记忆，把状态落盘到工作区文件（JSON/Markdown 均可），下一跳读文件继续。

配置要点：

- cron：注意时区（容器里经常是 UTC，显式指定）；delivery 不设 announce 就是静默跑完，你根本不知道它干过什么。
- heartbeat：间隔别设太小，token 消耗很线性；`HEARTBEAT.md` 保持几行以内，只写"待观察项 + 触发条件"，别把它写成第二份系统提示词。

## 踩坑点

1. **heartbeat 不准点**。它是"间隔检查"不是"整点触发"，宿主机休眠时心跳会停（macOS 要允许定时唤醒）。要求 9:00 准时送达的事，别交给 heartbeat。
2. **cron 会话失忆**。在隔离会话里让 agent"接着上次的做"，它不知道上次是什么。要么把上下文写进 payload，要么让它读工作区文件。
3. **心跳风暴**。`HEARTBEAT.md` 里塞了一堆"每小时检查 XX"，每次心跳都要过一遍全部条目，token 直接翻倍。高频检查项请下沉到 cron 或外部脚本/插件钩子。
4. **重入**。某次心跳任务跑得比间隔还长，下一次心跳照样进来。让 agent 开工前先写一个"进行中"标记文件，心跳时看到标记就跳过。

## 可复用建议

- 判断标准一条就够：**时间是确定的用 cron，条件是确定的用 heartbeat**。
- heartbeat 的开销是"每次必付"，cron 的开销是"到点才付"。检查项越多，越该往 cron 迁。
- 状态一律落盘：cron 会话、心跳会话、人工会话之间，工作区文件才是共享内存。
- 拿不准时先用 cron 跑通流程，确认确实需要"自主判断"后，再把逻辑挪进 `HEARTBEAT.md`。

## 总结

两者不是替代关系，而是分工：cron 负责日程表，heartbeat 负责值班桌。把"几点做"交给 cron，把"要不要做"交给 heartbeat，中间用文件传递状态，基本能覆盖个人自动化的绝大部分场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/709e7ed7bfa0f9b0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/89d520be5a96a471.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/bf6cf81df047eeff.png)

