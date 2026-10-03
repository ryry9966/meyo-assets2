---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 40335
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

OpenClaw 里让 agent "自己动起来"有两条路：**heartbeat（心跳）** 和 **cron（计划任务）**。两者都能触发定时行为，但设计意图完全不同。最常见的翻车姿势，就是拿 heartbeat 干 cron 的活，或者反过来。

## 两者到底差在哪

- **heartbeat**：按固定间隔唤醒主会话，agent 醒来先读工作区的 `HEARTBEAT.md`，有事就干，没事保持沉默。本质是"巡更"——到点睁眼看一圈，见机行事。
- **cron**：cron 表达式精确触发，为每个任务开独立会话，执行写死的 prompt，跑完可选把结果投递到指定频道。本质是"闹钟"——到点必响，响完就走。

一句话选型：**需要固定时刻、要推送结果 → cron；需要"看看情况再说"的观察类任务 → heartbeat。**

## 做法

**heartbeat 两步：**
1. 配置里设置唤醒间隔，例如 `heartbeat.every = "30m"`；
2. 在工作区写 `HEARTBEAT.md`，每条任务一行，写清判断条件和动作，比如："若 `logs/error.log` 新增未处理的 ERROR，则整理摘要发到频道"。

**cron 两步：**
1. 对话里用 `/cron` 建任务，或直接编辑任务文件；表达式示例 `30 8 * * *`，注意把时区设为 `Asia/Shanghai`；
2. prompt 写成自包含的指令，并配置 delivery，让产物直接推到你的 IM 频道。

## 踩坑点

1. **heartbeat 调太密**。每次唤醒都是一次模型调用，哪怕最后什么都不做。从 30 分钟起步，按需收紧。
2. **把大任务塞给 heartbeat**。它跑在主会话里，长任务会挤占日常上下文；超过几轮工具调用的活，交给 cron。
3. **cron 会话是隔离的**。它不记得主会话聊过什么，prompt 必须自包含；依赖的状态写成文件，让它自己去读。
4. **时区翻车**。服务器多为 UTC，表达式没配时区时，"每天早上 8 点"会漂移 8 小时。
5. **cron 失败是静默的**。任务没配投递时，跑挂了你都不知道；重要任务务必配置失败通知。
6. **HEARTBEAT.md 变垃圾场**。过期任务不删，agent 每次醒来都白判断一遍。

## 可复用建议

- 固定节拍给 cron：晨报、周报、备份提醒、定时抓取。产物落盘到工作区，主会话随时可读。
- 巡检观察给 heartbeat：盯依赖更新、盯收件箱、盯一个"等它出现再说"的文件。
- `HEARTBEAT.md` 条目带优先级和失效条件，定期清理，宁可少放。
- 两者可以组合：cron 每天固定生成日报，heartbeat 期间发现异常时顺手补充，而不是重复造一份巡检。

## 总结

heartbeat 是"自主判断的巡更"，cron 是"精确触发的闹钟"。用"到点必做"还是"看了再说"来分界，绝大多数场景就能选对；再把隔离上下文、时区、静默失败这三件事处理好，OpenClaw 的定时自动化基本不会踩雷。

---

## 配图

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/acf5eb471318c29a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/78eeec57f7092ab4.png)

