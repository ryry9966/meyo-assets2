---
title: 子 Agent 不污染主会话：OpenClaw 的 session 隔离实践
feedId: 42537
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

在 OpenClaw 里，主会话不只是聊天记录：它同时承载上下文历史、记忆索引和工具连接状态。任务一重（批量检索、长链路浏览器操作、多文件改造），常见做法是 spawn 子 Agent 并行处理。但子 Agent 运行会产生大量中间产物——工具输出、重试记录、临时结论。这些内容如果落回主会话，就是典型的会话污染。

## 问题

我们实际踩到的表现：

- 子任务跑完没多久，主会话提前触发上下文压缩。查 token 账单，大头是子 Agent 的中间输出被算进了主历史；
- 记忆检索开始冒出无关内容，追查发现是子 Agent 把中间结论写进了主记忆；
- 有状态的 MCP（浏览器实例、DB 连接）被主/子两个 Agent 同时操作，页面和事务状态互相踩；
- 复盘日志时分不清某条工具调用是主会话还是子会话发的。

## 做法

**1. 子会话独立 key。** spawn 时显式声明隔离 scope，每个子任务拿独立 session 文件（形如 `sessions/sub/<task-id>.jsonl`），并禁止向父会话写历史：

```yaml
agents:
  subtask:
    session:
      scope: isolated
      parent_write: false
```

（字段名随版本有差异，以 `openclaw config schema` 的实际输出为准。）

**2. 只回传结构化结果。** 子 Agent 的产物收敛为一个 JSON 摘要或文件路径，由主会话以单条消息注入。全文 transcript 一律不回灌。

**3. 记忆加命名空间。** session 隔离了还不够，记忆集合也要按前缀隔离：子 Agent 只写 `mem/sub/<task-id>`，主会话检索默认不扫这个前缀，需要时显式挂载。

**4. 有状态的工具重建。** 浏览器、SSH、数据库这类带会话态的 MCP server，在子会话里重新实例化，不要复用主会话连接；纯无状态工具可以共享。

**5. 生命周期收口。** 子会话设 TTL，任务结束即 gc；主任务取消时要级联 cancel 子 Agent，否则孤儿会话会继续写盘。

## 踩坑点

- 只改了 model 和 tools、漏设 session scope，结果静默共享，日志里看到主会话冒出子任务的调用才发现；
- session 隔离了，向量库没隔离，子 Agent 的 embedding 混进主检索结果，召回质量下降；
- 并行子 Agent 写同一个工作区文件互相覆盖，应给每个子任务独立 scratch 目录；
- 图省事把子会话全文贴回主会话，隔离等于白做。

## 可复用建议

- 把子 Agent 当函数调用：参数进、一个结构化结果出，过程不外泄；
- 跨会话写默认拒绝，白名单放行；
- 命名空间覆盖 session、memory、scratch 三处，缺一不可；
- 加一条巡检：session 目录大小、孤儿会话数、记忆里的异常前缀，挂到 cron 或 CI 里跑。

## 总结

session 隔离不是“另开一个文件”，而是历史、记忆、工具状态、生命周期四条边界一起收口。收口之后，主会话上下文干净、复盘清晰，子 Agent 才敢放开跑长任务。这套模式我们后来也复制到了定时任务和插件回调上，收益一致。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/a1d4575c5b94fed9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/320f7cf8c88037da.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/29515402769de045.png)

