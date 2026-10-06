---
title: OpenClaw session 隔离实战：子 Agent 干完活就走，别污染主会话
feedId: 40666
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的主会话（你绑定的聊天入口）是长期上下文的载体：对话历史、workspace 里的 `AGENTS.md` / `MEMORY.md`、压缩后的摘要都在里面。子 Agent 通过 `sessions_spawn` 拉起，拿到的是独立的 transcript、独立的上下文窗口和一份裁剪过的 system prompt，跑完只把最终一条消息回传给父会话。这套机制的设计意图很明确：过程噪音留在子会话，主会话只消费结论。

## 问题：不隔离会发生什么

实践里常见的三种“污染”：

1. **上下文膨胀**。子任务的工具输出（批量读代码、抓网页）直接灌进主会话，token 消耗暴涨，触发 compaction，把早期真正重要的对话摘要掉——表现为“聊着聊着忘了之前的决定”。
2. **记忆污染**。子 Agent 往共享 workspace 的 `MEMORY.md` 写了任务临时笔记，主会话下次冷启动把它当成用户偏好。
3. **越权输出**。没收掉 message 类工具的子 Agent 中途直接给用户发消息，主会话与子任务的输出交织，事后无法排查。

## 做法

1. **重活一律 spawn**。判断标准：预计工具调用超过十几次、或会产生大段中间输出的任务，单独开 session；纯问答才留在主会话。
2. **收窄工具面**。子 Agent 通常只需要读、搜索、计算几类工具；在 agent 配置里 deny 掉 `message` / `sessions_send` 这类能对外说话的工具（不同版本键名有差异，用 `openclaw doctor` 核对当前 schema）。
3. **隔离 workspace**。高频子任务固化为独立 agent id + 独立 workspace，它要写记忆就写在自己目录里；主会话的 `MEMORY.md` 永远只由主 Agent 维护。
4. **约定返回格式**。父会话只能看到子 Agent 的最后一条消息，所以 prompt 里要写死输出：

```text
任务：…
约束：不给用户发消息；不编辑 MEMORY.md/AGENTS.md；
     中间产物写进自己的 workspace。
返回：≤200 字结论 + 关键路径/数据，按固定 JSON 结构。
```

5. **跑完核对**。`openclaw sessions` 看会话列表；主会话的 jsonl 里不应出现子任务的工具噪音，异常时 `openclaw doctor` 排查。

## 踩坑点

- **“回传只有一条”被忽略**：子 Agent 最后一句含糊（“完成了，详见上面”），父会话等于白跑。返回格式必须显式约束。
- **spawn 出来的 session 忘了收尾**：会话列表越积越多，排查时误导自己，定期 reset 废弃子会话。
- **在群里测隔离**：同一群聊触点共享主会话，验证隔离效果先用私聊单 Agent 跑通。
- **deny 完就不管了**：工具面收窄后，还要确认子 Agent 的 prompt 里没有“随时汇报进度”这类引导，否则它会绕路（比如往共享目录写文件让主 Agent 去读）。

## 可复用建议

- 一句话决策：产生中间输出 → spawn；纯问答 → 内联。
- 子 Agent prompt 三件套：任务边界 + 禁写共享资源 + 固定返回格式。
- 隔离验证两步：`openclaw sessions` 查列表；在主会话 transcript 里搜子任务关键词，应为空。
- 周期性重活（每日抓取、巡检）直接配成常驻独立 agent，比每次临时 spawn 更可控。

## 总结

Session 隔离不是“开个子进程”，而是三条边界：**上下文边界**（独立 transcript）、**权限边界**（收窄工具）、**记忆边界**（独立 workspace）。主会话只收结论、不碰过程。把这三条固化成自己的 spawn 模板，子 Agent 才是助力而不是污染源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/d75a86bdacd41b7b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/c751a1e6f7bd1841.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/e3558a9eeb9131b3.png)

