---
title: HEARTBEAT.md：让 Agent 主动做事，而不是等你提问
feedId: 39322
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

大多数 Agent 的默认工作模式是被动的：你发消息，它回复，然后挂起。OpenClaw 里有个不太起眼但很好用的机制——heartbeat。网关按固定间隔（默认约 30 分钟，配置见 `agents.defaults.heartbeat`，不同版本键名略有差异）唤醒一次 Agent，让它读取工作区里的 `HEARTBEAT.md`，自己判断"现在有没有该做的事"。这让 Agent 从应答器变成了带周期巡检的服务。

## 问题

在用上 heartbeat 之前，我的自动化只有两条路：写 cron 脚本，或者想起来再问 Agent。前者僵硬——脚本只能跑死逻辑，做不了"判断今天日志值不值得汇报"这种事；后者不可靠——主动权在人，忘了就没了。heartbeat 补的是中间一层：不需要精确时刻，但需要周期性"看一眼 + 拿主意"的任务。

## 做法

1. 在工作区（默认 `~/.openclaw/workspace/`）建 `HEARTBEAT.md`，用清单写任务：

```markdown
# Heartbeat
- 检查 app/error.log 是否有新增 ERROR，仅在出现新错误时提醒我
- 每天第一次心跳：汇总 GitHub Actions 昨夜失败的 run
- 检查 notes.md 中标记"待提醒"且已到期的条目，提醒我
- 以上都不满足时，仅回复 HEARTBEAT_OK，不要解释
```

2. 按需调整间隔。巡检类任务 30~60 分钟足够，别贪密。

3. 关键约定：心跳回复若严格等于 `HEARTBEAT_OK`，OpenClaw 会吞掉这条消息，不推送到聊天渠道；只有"真有事"的回复才送达。这就是"主动但不吵"的核心。

4. 心跳做分诊，不做重活：让它产出一条摘要或一个触发建议，重任务交给 cron、hook 或后续会话接管。

## 踩坑点

- **心跳是真金白银的调用**。一天几十次模型请求，清单越长，token 开销和走形式的概率一起涨。控制在 3~5 条。
- **`HEARTBEAT_OK` 必须严格匹配**。Agent 有时会"热心地"多说一句，结果整条推送原样发到你手机，半夜尤其感人。清单末尾务必写明静默条件。
- **heartbeat 是 best-effort**。会话忙或锁定时会跳过，不保证准时。精确调度用 cron，事件驱动用 webhook，heartbeat 只负责周期性判断，三者别混用。
- **别把长任务写进心跳**。我曾在清单里写"每天重建笔记索引"，结果一次心跳占住会话十几分钟。改成：心跳只检查索引是否过期，过期才触发重建。
- 多渠道用户注意推送 target 配置，避免巡检汇报发到不该发的群里。

## 可复用建议

- 任务写成"条件触发式"：**"检查 X，仅在 Y 时提醒"**，而不是"每 30 分钟做 X"。
- 把 `HEARTBEAT.md` 当作版本化的小型 SLO 文件提交进 git，改动有据可查，回滚也方便。
- 分层组合：cron 管精确时刻，webhook 管事件，heartbeat 管判断。职责越单一，越不容易互相踩。
- 先跑 3~5 天，看心跳日志的实际命中率和 token 消耗，再决定调间隔还是砍条目，比拍脑袋靠谱。

## 总结

HEARTBEAT.md 的价值不在于定时执行——那是 cron 的活——而在于给了 Agent 一个低成本的自主判断入口。写好它的诀窍恰好是克制：条目要少、静默条件要明确、重活外移。当你发现 Agent 能在你不问的时候安静补上一句"日志有新错误"时，这个机制才算真正用对了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/6bed43b954e3de8f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/7d955fbd20cd30fb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/e9d511706c62a6ef.png)

