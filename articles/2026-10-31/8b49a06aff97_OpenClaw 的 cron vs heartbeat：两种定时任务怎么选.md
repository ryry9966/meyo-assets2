---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43065
source: 综合讨论
publishedAt: 2026-10-31
---

## 背景

OpenClaw 的 agent 有两种被"定时叫醒"的方式：cron 和 heartbeat。刚上手时很容易只注意到 heartbeat——默认就开着，"agent 每 30 分钟自己巡逻一次"看起来很省心；等任务多了才发现，真正可控的调度其实是 cron。这篇帖结合一段时间的实践，梳理两者的差异和选型依据。

## 两种机制的本质差异

- **cron**：确定性调度。定义 cron 表达式 + payload（一段发给 agent 的指令），到点触发一次 agent turn。什么时候跑、跑什么，完全由配置决定，跑完可把结果投递到指定 channel。
- **heartbeat**：非确定性巡检。agent 按固定间隔（默认 30 分钟）被唤醒，读 workspace 里的 HEARTBEAT.md，自己判断有没有值得做的事；没事就回 HEARTBEAT_OK，网关把它吞掉不外发。

## 怎么选：四个判断维度

1. **时间驱动还是状态驱动？** 固定时间点的任务（每天早报、每周汇总）→ cron；"每隔一段时间盯一眼状态"（盯邮件、盯进程、盯 CI）→ heartbeat。
2. **任务能否一句话写死？** 能写死 → cron；需要结合上下文临时判断 → heartbeat。
3. **token 成本**：heartbeat 每次唤醒都有开销，哪怕最终啥也没干；cron 只在触发时花钱。
4. **会话污染**：heartbeat 默认跑在主会话，产生大量短 turn；cron 可以用隔离会话，不搅动主上下文。

## 具体做法

**cron 三步：**

1. 在网关配置里建 job：起名字、写 cron 表达式、写 payload。payload 说清楚"做什么、输出到哪、失败怎么办"。
2. 决定投递：需要通知人就 deliver 到 channel；纯后台任务只落日志或记忆。
3. 重活放隔离会话跑，避免搅浑主会话上下文。

示例（字段名以当前版本文档为准）：

```json
{
  "cron": {
    "jobs": {
      "morning-brief": {
        "schedule": "30 8 * * *",
        "payload": "汇总昨天的对话要点和今天的日程，输出不超过 10 行的简报",
        "deliver": true
      }
    }
  }
}
```

**heartbeat 三步：**

1. 在 workspace 根目录建 HEARTBEAT.md，写成 3–5 条以内的 checklist，每条是"判断式"而非"任务式"。
2. 调整间隔（如 30m → 60m）和 activeHours，避开深夜。
3. 观察一两天心跳日志，把"每次都做同样的事"的条目迁出去变成 cron。

## 踩坑记录

1. **把明确任务塞进 HEARTBEAT.md**。最常见的错误。每次心跳都要读一遍、判断一遍，还可能因模型"觉得没到时间"而漏跑。凡能写成"某时刻做某事"的，都该是 cron。
2. **HEARTBEAT.md 越写越长**。它每次唤醒都全文进 prompt，30 分钟一次就是一个月上千次。控制条数，删掉模糊项。
3. **cron 表达式的时区**。调度按网关主机时区解释，容器里没设 TZ，早报变成晚上八点发。部署时统一固定 TZ。
4. **隔离会话没有上下文**。cron job 在隔离会话里看不到主会话聊过什么，要么把必要上下文写进 payload，要么让它读记忆文件。
5. **重复触发**。网关重启补跑、手动触发叠加，可能重复发消息。任务里加"先查再发"的幂等判断。

## 可复用建议

- 一句话法则：**时间驱动用 cron，状态驱动用 heartbeat**。
- 心跳是"发现"机制，cron 是"执行"机制。heartbeat 巡逻中发现的固定需求，沉淀成 cron job，心跳清单随之变短。
- 对成本敏感时优先 cron：它的开销可预测（次数 × 单次 token），heartbeat 的开销跟清单长度和判断摇摆正相关。
- 两者都建议留痕：cron 结果统一投到一个不吵人的 channel，heartbeat 的关键动作让它写进记忆，方便事后审计。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工。cron 负责确定性——时间准、行为可预期、成本可预算；heartbeat 负责开放性——巡逻、发现、兜底。我们目前的实践是 cron 为主、heartbeat 为辅：心跳间隔拉长，HEARTBEAT.md 只留真正需要"看情况"的条目。选型错了不可怕，可怕的是每次心跳都在为一份放错位置的清单付费。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/4610d92fdada91ab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/cfe92400e30ba018.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-31/5ce5bde68e0817da.png)

