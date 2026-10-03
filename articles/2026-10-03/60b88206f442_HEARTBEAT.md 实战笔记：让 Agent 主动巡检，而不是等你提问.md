---
title: HEARTBEAT.md 实战笔记：让 Agent 主动巡检，而不是等你提问
feedId: 40217
source: 综合讨论
publishedAt: 2026-10-03
---

# 背景

大多数时候我们把 OpenClaw 当问答和传话工具：发一条消息，agent 回一条。但它其实内置了一个容易被忽略的机制——心跳（heartbeat）：默认每 30 分钟（`agents.defaults.heartbeat.intervalMs`，默认 1800000），Gateway 会给 agent 发一条心跳提示，让它去读工作区里的 `HEARTBEAT.md`（默认在 `~/.openclaw/workspace/HEARTBEAT.md`）。文件不存在、为空或只有注释时，直接跳过，不发起模型调用；文件里有内容，agent 就会评估“现在有什么该做的”，做完把结果发到绑定的频道，没事则回 `HEARTBEAT_OK`，你不会收到任何噪音。

# 问题

cron 能定时，但不会判断；LLM 会判断，但被动等问。HEARTBEAT.md 恰好把两者拼起来：心跳负责节拍，文件里的规则负责“什么值得做、什么时候闭嘴”。我用它跑了三件事：服务器巡检（磁盘超阈值、容器异常退出才报）、每天一次的 GitHub 通知摘要、笔记 inbox 积压提醒。

# 做法

**1. 写规则，要求“可判定 + 安静优先”：**

```markdown
# 心跳任务 —— 没有异常一律回 HEARTBEAT_OK

- 每次心跳：运行 df -h / 和 docker ps；磁盘 >85%
  或有异常退出的容器才报告，附一行处理建议。
- 每天上午 9 点后第一次心跳：汇总 GitHub 上 @我的未读通知并发送；
  发送前读 memory/heartbeat-state.json 的上次发送日期，
  当天已发过则跳过，发送后写回今天日期。
- 每次心跳：数 ~/notes/inbox.md 的未处理条目，超过 3 条才提醒。
```

**2. 配置节拍与活跃时段：**

```jsonc
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "intervalMs": 1800000,
        "activeHours": { "start": "09:00", "end": "22:00" }
      }
    }
  }
}
```

**3. 给“每天一次”类任务落盘状态。** 心跳不一定记得上一轮干了什么，所以“发过没有”必须自己持久化：让 agent 发送前先读状态文件，发完写回。这一步是防重复轰炸的关键。

# 踩坑点

- **重复轰炸**：最早我没写状态文件，“每天早上的摘要”变成每 30 分钟一条。规则里必须强制“先查状态、后发送、发完记录”。
- **指令模糊**：写“帮我盯着点服务器”，agent 会自由发挥，一天报五六条正常状态。阈值和“否则回 HEARTBEAT_OK”都要写死。
- **`HEARTBEAT_OK` 语义**：安静退出依赖这个约定；如果你自定义了 heartbeat prompt，记得保留它，且要求精确匹配。
- **Token 成本**：文件非空时，每次心跳都是一次带工作区上下文的模型调用。规则控制在十几行内，重活交给脚本，agent 只读脚本输出的结论。
- **电脑睡眠**：心跳跑在 Gateway 所在机器上，Mac 合盖就没节拍。要么放常开的机器/NAS，要么接受空档。
- **主会话共享**：心跳和聊天共用上下文，长期会把巡检输出留在上下文里。让 agent 把详细输出写文件、只发一两行结论到频道，膨胀会明显减缓。

# 可复用建议

写规则时套这个模板：**一条规则 = 触发条件（每次心跳 / 时间窗）+ 动作 + 汇报格式 + 安静条件**。另外：

- “安静优先”写在文件开头，没事不寒暄、不汇报；
- 阈值、冷却时间、状态文件路径全部写明，不给 agent 猜的空间；
- 先低频跑一两周（比如 1 小时一跳），观察误报率再收紧或放宽。

# 总结

HEARTBEAT.md 的价值不在“定时任务”本身，而在把你的判断标准写成 agent 可执行的规则。它更像一份值班手册：什么时候该看、看到什么才算事、没事就闭嘴。手册写清楚了，agent 才从“工具”变成“同事”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/259e540e080e8264.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/fbb29437f95122cd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/8ea8303e74094674.png)

