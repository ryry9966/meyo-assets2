---
title: HEARTBEAT.md 实践：让 Agent 主动做事，而不是等你提问
feedId: 42294
source: 综合讨论
publishedAt: 2026-10-22
---

# 背景

OpenClaw 的 agent 默认是被动应答机：你不发消息，它就不动。但很多事恰恰需要它先动——盯 CI、看磁盘水位、早上汇总待办。OpenClaw 为此留了一个常驻机制：**心跳（heartbeat）**。Gateway 每隔固定时间（默认 30 分钟）向 agent 注入一次心跳请求，并把 workspace 根目录下的 `HEARTBEAT.md` 一并带上。Agent 读一遍清单，然后二选一：干活并通知你，或者沉默。

沉默协议是关键：无事可做时 agent 只回复 `HEARTBEAT_OK`，这条回复会被吞掉，不会推到你的聊天窗口。

# 问题

没有 HEARTBEAT.md，心跳基本空转；随手写几句，又会走向另一个极端——每次心跳都"发现"点什么，隔半小时给你发一条废话。主动性要可控，靠的不是一句"你可以主动一点"的 prompt，而是一份可判定、可迭代的清单文件。

# 做法

1. 在 `~/.openclaw/workspace/` 下建 `HEARTBEAT.md`，写成 checklist，一行一个任务，每行必须包含：判定条件 + 动作 + 沉默条件。
2. **心跳只做判断，执行交给 skills/脚本**。比如 CI 检查，心跳里只判断"有没有新失败"，拉日志、跑命令转交给 exec 或对应 skill。
3. 需要时间语义（如"早上 8 点后第一次"）直接写进条件行，心跳上下文里带当前时间。
4. 间隔按需调整，重任务宁可拉长间隔或改用真正的 cron；具体配置键以你版本的 `docs/configuration` 为准。
5. 上线后观察几天日志，重点看心跳 session 里除了 HEARTBEAT_OK 之外还发生了什么。

示例（可直接抄改）：

```markdown
# Heartbeat Checklist

- [ ] 仓库 xxx/yyy 的 CI：仅当出现"新"失败（上次已通报过的不算）时，简短汇报一次
- [ ] 服务器磁盘使用率 > 85%：提醒；否则沉默
- [ ] 每天 08:00 后的第一次心跳：读取 memory/ 整理今日待办，推送一条汇总
- [ ] 其余情况：回复 HEARTBEAT_OK
```

# 踩坑点

- **条件不可判定**。"关注一下行业动态"等于没写：每次心跳都会触发。要写成"RSS 中出现包含 xxx 关键词的新条目"这种能跑判断的句子。
- **没做去重**。同一个 CI 失败会连报五次。两个解法：条件里写"仅状态变化时汇报"，或让 agent 把已通报状态记到 memory 文件里做比对。
- **上下文隔离**。心跳 session 和主会话不共享即时上下文，心跳里做过的事必须落盘（memory/、日志文件），否则主会话一无所知，还会重复做。
- **心跳里跑重活**。定时触发 + 长任务 = session 被占住。心跳保持轻，重活转后台任务或脚本。
- **成本**。30 分钟一次就是一天 48 次调用。清单写短，或给心跳配更便宜的模型、拉长间隔。

# 可复用建议

把 HEARTBEAT.md 当**告警规则**来写，而不是愿望清单。每条都要能回答三个问题：什么情况算触发？触发后做什么？什么情况必须沉默？另外三条经验几乎通用：判断与执行分离、状态落盘做去重、先用长间隔（如 60m）观察一周再收紧。多 agent 场景下，一人一份清单、职责单一，别把所有事堆给一个 agent。

# 总结

HEARTBEAT.md 的本质，是把"主动性"从一次性 prompt 变成声明式配置：checklist 定义边界，HEARTBEAT_OK 定义沉默，memory 定义记忆。写得克制，它就是个可靠的值班助手；写得随意，它就是个定时打扰器。建议从一两条高价值任务起步，跑一周再迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/8c33fcc382b70620.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/ef0bdc5c8f1c6314.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/c31985ae3c4508f0.png)

