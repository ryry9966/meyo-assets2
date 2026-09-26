---
title: cron 还是 heartbeat？OpenClaw 两种定时机制的选型与踩坑
feedId: 39115
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 里想让 agent「主动干活」，官方给了两条路：**heartbeat（心跳）**和 **cron（定时任务）**。社区里常见的两种极端：一种人全靠心跳，觉得「反正它会自己判断」；另一种把所有轮询都写成 cron。两种混用不当的代价都一样——token 账单和消息打扰。

## 问题

拿三个典型需求来看：

- 每天早上 9 点把待办摘要发到 Telegram；
- 盯一个长构建，挂了再叫我；
- 收件箱出现值得关注的邮件时提醒我。

只有第一个适合 cron。后两个是**状态触发**，你无法预知时间点。用 cron 硬扫（比如每 10 分钟跑一次「检查一下」）等于人肉轮询模拟心跳，还丢了主会话上下文；反过来靠 heartbeat 做「9 点发日报」，时间点会飘，而且为了等一个时刻白烧一整天的心跳调用。

## 两种机制的本质区别

**heartbeat**：

- 按 `agents.defaults.heartbeat.every`（默认 30m）向主会话注入一条 HEARTBEAT 消息；
- 工作区的 `HEARTBEAT.md` 是常驻检查清单，agent 自行判断有没有事，没事就回 HEARTBEAT_OK，不打扰你；
- 跑在主会话里，看得见上下文；代价是**每次心跳都是一次真实的模型调用**。

**cron**：

- `openclaw cron add` 定义，支持 5 字段 cron 表达式或 `--every` 间隔；
- 每个 job 是一次独立 run，可跑在隔离会话，跑完把结果投递到指定 channel；
- 触发时间和 prompt 都显式定义，确定性强。

一句话：**cron 是日程表，heartbeat 是值班巡逻。**

## 做法

1. **先分类需求**：触发条件是「时间」还是「状态」。时间明确 → cron；状态模糊 → heartbeat。
2. **配 cron**（日报为例）：

   ```bash
   openclaw cron add --name morning-digest \
     --cron "0 9 * * *" \
     --prompt "读取 ~/notes/todo.md，输出不超过 200 字的今日摘要" \
     --deliver --channel telegram
   ```

   参数名随版本有变动，以 `openclaw cron add --help` 为准。先用 `openclaw cron run` 手动触发一次验证输出，再上线。
3. **配 heartbeat**：把具体、可判定的检查项写进 `~/.openclaw/workspace/HEARTBEAT.md`，例如「检查 CI 最近一次 run，失败才提醒」。间隔 30m 起步，观察一周再调。
4. 跑一两天，用 `/status` 和模型厂商的 usage 面板看消耗与打扰频率，再微调间隔和清单。

## 踩坑点

- **心跳间隔压到 5 分钟**：一晚几百次调用，账单感人，还会污染主会话上下文。除非真有低延迟监控需求，别低于 15m。
- **HEARTBEAT.md 写得太泛**：「帮我盯着一切」会让 agent 频繁误触发。检查项要具体、可判定，并明确写上「拿不准就忽略」。
- **cron 的 prompt 假设有上下文**：隔离会话里 agent 不知道你的项目。文件路径、参数、输出格式都要写进 prompt。
- **时区**：cron 按宿主机本地时区解析。服务器在 UTC，「每天 9 点」实际是北京时间 17 点，VPS 部署务必核对。
- **心跳和 cron 同时往主会话写**：重活让 cron 在隔离会话跑，避免与心跳排队互相堵。

## 可复用建议

- 分工口诀：**确定性产出交 cron，开放性观察交 heartbeat**。
- HEARTBEAT.md 当「关注面」维护，每月精简一次，删掉不再需要的检查项——清单越长，误触发越多。
- cron 的 prompt 按独立任务书写，不依赖会话记忆。
- 两者可以组合：cron 出日报，heartbeat 兜底盯异常，各司其职。

## 总结

不是二选一，而是分工。判断标准就一条：**触发条件能不能写成时间表达式**。能，就 cron；不能，就 heartbeat。用得克制，成本和打扰都能控住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/71d406f718e6ced4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/94efaacc4c1950f3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/069b180116e221d5.png)

