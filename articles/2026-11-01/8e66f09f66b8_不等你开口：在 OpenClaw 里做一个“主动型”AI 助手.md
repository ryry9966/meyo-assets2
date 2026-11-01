---
title: 不等你开口：在 OpenClaw 里做一个“主动型”AI 助手
feedId: 43189
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

绝大多数 agent 的使用模式仍然是“你问它答”：消息进来，模型响应，会话结束。但 OpenClaw 的 agent 是常驻的——gateway 挂着会话，插件和 MCP 工具随时可调，heartbeat/cron 内建。这意味着它完全有能力在没人说话时自己干活：早上汇总昨晚的 CI 失败、发现有人在 issue 里 @ 你、提醒一个快过期的证书。这类能力我叫它 proactive：由时间、事件或状态变化触发，而不是由用户消息触发。

## 问题

做 proactive 有两个极端都会翻车。一是哑火：心跳配了，检查逻辑写了，但触发条件太苛刻，三个月没主动说过一句话。二是骚扰：每次心跳都推送，用户第三天就把它关了。更麻烦的是安全面：agent 在无人指令时执行写操作（替你发消息、改文件），出错时责任很难界定。所以目标不是“更主动”，而是“可控的主动”。

## 做法

我的最小可行方案把每个主动任务拆成四段：触发、预检、去重、动作。

1. **触发**：低频任务挂 heartbeat（每 30 分钟），时间敏感的用 cron 表达式。
2. **预检**：唤醒后第一件事不是让大模型思考，而是跑一个只读的 check 工具（MCP 调用），返回结构化结果：有没有值得说的事。没有就静默退出，不进模型。
3. **去重**：本地 JSON 持久化 `last_sent_hash` 和 `last_sent_at`，payload 没变化就跳过，避免同一个问题推三遍。
4. **动作分级**：info 级写进每日 digest；warn 级即时推送；写操作默认关闭，按任务白名单开启。

配置大致长这样：

```yaml
proactive:
  heartbeat: "*/30 * * * *"
  quiet_hours: "23:00-08:00"
  tasks:
    - name: issue-watch
      check: mcp://github/issues?mention=me
      level: digest      # digest | push
      action: notify     # notify | tool_call
```

最后所有主动行为落一条审计日志，事后能回答“它当时为什么找我”。

## 踩坑点

- **心跳太密**。我最初配了 5 分钟一次，token 费用翻倍。现在预检用脚本/小模型，确认有事才升级到大模型。
- **没设静默时段**。凌晨三点被推送叫醒一次就够了，`quiet_hours` 必须有。
- **循环触发**。主动行为写回数据源，又触发自己形成环。给事件加 origin 标记，来源是自己的事件不再响应。
- **状态放内存**。gateway 一重启，去重记录全丢，又开始重复推送。状态必须落盘。
- **权限开太大**。第一批主动任务我只给只读权限，跑了两周确认行为符合预期，才逐步放开写操作。

## 可复用建议

- 把“主动”拆成 trigger / condition / action 三段独立配置，别写死在一个 prompt 里。
- 三个默认：默认只读、默认聚合（digest 优先于 push）、默认可一键关闭。
- 每个主动任务配一行人类可读的说明，半年后你自己需要知道它是干嘛的。
- 上线顺序永远是：提醒 → 汇总 → 代办。写操作放最后。

## 总结

proactive 不是玄学，是调度 + 预检 + 去重 + 权限边界的工程组合。OpenClaw 的常驻架构和 MCP 工具已经把地基打好，剩下的克制都在配置里：让 agent 该说话时说话，该闭嘴时闭嘴。先从“每天早上告诉我昨晚发生了什么”开始，比一上来就让它替你做事靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/80faa50423166f33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/433146c8e9dc7f29.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/c8aacd0922fa903c.png)

