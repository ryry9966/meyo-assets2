---
title: cron 还是 heartbeat？OpenClaw 定时任务选型与踩坑实录
feedId: 40152
source: 综合讨论
publishedAt: 2026-10-02
---

# 背景

OpenClaw 里的自动化，最终几乎都落到 agent 的会话上：要么是 cron 按确定时刻投递一条 prompt，要么是 heartbeat 周期性唤醒 agent 自检。我早期把"每天早上推送摘要"这种需求顺手写进了 HEARTBEAT.md，结果时间不准、token 也不便宜——这就是典型的用错工具。问题不在机制本身，而在没分清两者的边界。

# 两者差在哪

**cron**：调度器按 cron 表达式（或一次性 at 任务）在确定时刻触发，把 payload 投到指定 session。时间确定，逻辑在触发器一侧，agent 是被动执行者。

**heartbeat**：网关每隔固定间隔（默认 30 分钟左右，可配置）向主会话注入一条心跳提示，agent 读取 HEARTBEAT.md 里的规则，自行判断这一轮干不干活、干多少。时间近似，判断在 agent 一侧。

一句话：cron 是"到点就做"，heartbeat 是"到点看一眼，没事就跳过"。

# 怎么选：两个问题

1. **是否要求精确时刻？** 要（每天 9:00、每周五 18:00、半小时后提醒我）→ cron。
2. **是否依赖上下文和状态判断？** 要（"有未读邮件才汇总""磁盘快满才报警"，且希望去重、记得上次干过什么）→ heartbeat。

两者都不要的：每 1–5 分钟拉数据这种高频轮询，用 cron 更可控；heartbeat 跑这个频率纯粹烧 token。

# 实操

cron（示意，具体 flag 以你版本的 `openclaw cron --help` 为准）：

```bash
openclaw cron add daily-digest \
  --schedule "0 9 * * *" \
  --session isolated \
  --message "拉取昨日 GitHub 通知与 RSS，输出 10 行内摘要"
```

heartbeat 在配置里调间隔，规则写在 HEARTBEAT.md。**规则要写成判断条件，不是命令**：

```markdown
- 仅当 ~/inbox/ 未处理文件 > 50 时整理并汇报，否则跳过
- 仅当有新的 GitHub mention 且距上次汇报 > 6h 时汇总，否则跳过
```

# 踩坑点

1. **时区**：cron 跟随网关宿主机时区。服务器是 UTC、人在 UTC+8，`0 9 * * *` 会在下午五点触发。先看宿主机 `date`，再决定表达式按谁的时间写。
2. **heartbeat 不是精确调度**：间隔是近似值，会话忙时可能顺延。别用它实现"必须 9:00 触发"。
3. **HEARTBEAT.md 写成命令**（"每小时整理收藏夹"）会每轮都执行，token 和打扰感双拉满；写成条件规则才省。
4. **投递目标要想清楚**：cron 投到 main 会和日常对话混流；隔离 session 干净但没有上下文，任务要自带状态（落文件/数据库）。
5. **重叠执行**：cron 每 5 分钟一次 + agent 单轮跑得慢，任务会在会话队列里堆积。把单轮耗时压到间隔以内，或拉长间隔。
6. **宕机补偿**：网关停机期间的 cron 基本不回补；heartbeat 恢复后下一轮继续。需要"补上停机期间的事"，就把它写进任务描述里。

# 可复用建议

- **心跳成本先测再定**：挂一天即使全部跳过也是几十次模型调用。先跑一天看 idle 成本，再调间隔；能换便宜模型就换。
- **去重状态落盘**：给 heartbeat 任务配一个状态文件（上次汇报时间、处理到的位置），agent 每轮先查再决定，幂等且可追溯。
- **同一件事别两边都配**：选了 cron 就别在 HEARTBEAT.md 再写一遍，否则必然重复触发。
- 稳定组合：**cron 做定时触发器，heartbeat 做兜底巡检**，覆盖大多数个人自动化场景。

# 总结

cron 和 heartbeat 不是替代关系，而是两种触发哲学：前者把确定性放在调度器，适合精确时刻的周期/一次性任务；后者把判断放在 agent，适合"大多数时候没事"的有状态巡检。选型只需先问两个问题——要不要精确时刻、要不要上下文判断——再用实测成本校准频率，基本不会错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/e03a5deba63ffa28.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/08369fa37621bef7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/bc8fc3fba6185f37.png)

