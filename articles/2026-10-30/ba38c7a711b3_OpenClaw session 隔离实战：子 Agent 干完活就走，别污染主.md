---
title: OpenClaw session 隔离实战：子 Agent 干完活就走，别污染主会话
feedId: 43033
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

跑自动化的都有一个共识：主会话的上下文是最贵的资源。一旦直接在主会话里抓十几个网页、跑批处理、调一个输出啰嗦的 MCP 工具，几十轮下来上下文全是中间产物，真正要紧的决策信息反而被挤掉了。

所以标准做法是把脏活外包给子 Agent：用 `sessions_spawn` 起一个独立会话，干完只把结论交回来。但"隔离"不是 spawn 一下就自动成立的——OpenClaw 里子 Agent 默认继承的很多东西，都可能反过来污染主会话。

## 问题：污染有四种形态

1. **上下文污染**：不用 spawn、直接内联执行，或子 Agent 回传了全量日志，主会话上下文被中间输出塞满。
2. **记忆污染**：子 Agent 与主 Agent 共用 workspace 时，`MEMORY.md` 是共写的。子 Agent 把一次抓取失败总结成"该站点不可用"写进记忆，主 Agent 之后就一直带着这个错误结论干活。
3. **状态污染**：共享浏览器 profile、共享文件路径。子 Agent 登出账号、改了配置、留下半成品 git 改动，主会话接手时一脸懵。
4. **会话残留**：子会话不清理，历史无限堆积；复用同一个 agent id 时，新任务误召回旧上下文。

## 做法

**第一步：按任务类型单开 agent 条目。** 给"网页抓取""批量处理"这类任务各建一个 agent，id 和 workspace 分开。workspace 分开是记忆隔离的根本——`MEMORY.md` 按 agent 存，目录不共享就不存在互相写。示意（字段名以你版本的实际 schema 为准）：

```jsonc
{
  "agents": {
    "scraper": { "workspace": "~/openclaw-workspaces/scraper" }
  }
}
```

**第二步：工具集做减法。** 子 Agent 只挂任务必需的 MCP server；凡带写语义的工具（文件写入、浏览器登录、消息发送）默认不进白名单，查询类工具尽量只读挂载。

**第三步：约束回传格式。** spawn 时在任务提示里明确要求只返回固定结构的摘要（结论 / 数据 / 异常三段），并限制长度。完整日志留在子会话落盘，主会话需要细节时再用 `sessions_history` 主动去拉，而不是让子 Agent 全量倒回来。

**第四步：管好生命周期。** 长任务设超时；跑完的子会话定期清理，避免 sessions 目录膨胀，也避免召回时误命中陈旧上下文。

## 踩坑点

- **workspace 没分开**是最常见的：spawn 了子 Agent，但和主 Agent 同目录，`MEMORY.md` 被写脏。排障时的典型表现是"主 Agent 莫名其妙记得一些没发生过的结论"。
- **agent id 复用导致历史串台**：以为起了新会话，实际接上了上一个任务的上下文，行为变得诡异。
- 子 Agent 里调了会改登录态的浏览器工具，回头发现**主会话的浏览器被登出了**。
- 摘要没限长，子 Agent 回了三千字——隔离了个寂寞。
- 忘设超时，孤儿子会话慢慢堆满磁盘。

## 可复用建议

- 把子 Agent 当外部进程对待：**默认不共享任何可变状态，共享必须显式声明**。
- 一类任务一个 agent 配置，工具白名单最小化。
- 回传只走结构化摘要，日志留子会话，按需拉取。
- 定期巡检 sessions 目录；给 `MEMORY.md` 做 diff，发现不认识的写入立刻查是哪个 agent 干的。

## 总结

session 隔离的本质是划信任边界：子 Agent 不是"我的一部分"，而是一个受委托的外部执行者——给它最小工具集、独立工作目录、明确的生命周期，回传只留结论不留过程。做到这几点，主会话才能长期保持干净，自动化才敢往复杂了写。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/c73dff4bbce19178.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/4f623f233d6e15cf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/6aa362c6c11ccf08.png)

