---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42729
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

用 OpenClaw 跑长任务时，主会话通常是"总调度"，需要把活拆给子 agent：批量搜索、读一堆文件、跑脚本验证。理想状态是子 agent 干完只交一份结论。但实际用下来，子 agent 的中间过程——工具原始输出、报错重试、大段文件内容——很容易原样回流到主会话，上下文迅速被撑爆。

## 问题

几个典型症状：

- 主会话 token 涨得飞快，跑几轮就触发截断/压缩，原计划丢一半；
- 子 agent 某次失败的重试记录留在主会话里，主 agent 后续决策被这条失败带偏；
- 子 agent 覆盖了共享的工作目录或记忆文件，主流程读到脏数据。

根因往往不是"子 agent 不听话"，而是三样东西没隔开：**会话、工作区、返回内容**。

## 做法

**1. 子 agent 用独立 session。** spawn 子任务时不要复用主会话的 session key，给每个子任务独立 ID（示意配置，具体参数名以你版本的文档为准）：

```json
{
  "tool": "sessions_spawn",
  "args": {
    "task": "调研 xxx 并给出结论",
    "sessionKey": "sub:research-0712",
    "mode": "summary"
  }
}
```

关键是返回模式用 summary 或 structured，主会话只收一份结论，不收过程。

**2. 大产物落盘。** 子 agent 把长文本、抓取结果写进自己的 scratch 目录，返回里只给路径加一句话结论。主会话需要细节时再按需读片段，而不是默认全量进上下文。

**3. 工具权限收敛。** 子 agent 只挂它需要的 MCP/插件，能只读就只读。不要让子 agent 拿到能写主会话状态或全局配置的权限——隔离不只为了省 token，也为了安全。

**4. 验证一下。** 跑一个任务，前后各看一眼主会话 token 数和 sessions 目录（一般在 `~/.openclaw` 下），确认子会话 transcript 落在独立文件里，主会话的增量就是那一小段摘要。没达标就说明某条链路还在漏。

## 踩坑点

- **在主会话里直接调长输出的 MCP 工具**，输出照样写进主上下文。该下沉的调用要包进子 agent，这是最容易漏的一条。
- **子 agent 默认继承主会话记忆**。有些配置这显得"贴心"，实际会串味：子 agent 拿着主会话半截计划自行发挥。显式关掉继承。
- **共享 scratch 目录撞文件名**。两个子 agent 并发写 `result.json`，谁覆盖谁全看运气。按 session key 建子目录。
- **返回模式图省事用 full**，等于没隔离。同理，重试也应该发生在子会话内部，主会话只该看到"试了三次，最终结论是 X"。

## 可复用建议

- 定个粗略阈值：预计输出超过几 KB、或步骤超过十几步的活，一律进子 agent。
- 固化一份"子 agent 契约"：输入走参数或文件，输出是一份结构化摘要，其余产物落盘。契约稳定后，主会话 prompt 都不用改。
- 主会话只做规划和汇总，定期 compact；子会话 transcript 留在磁盘上，排障时再翻。

## 总结

session 隔离的本质就三件事：**会话分开、工作区分开、返回收窄成摘要**。把主会话当调度器，子 agent 当干活的临时工——干完活只交报告，不把工地搬回办公室。做到这点，长任务的上下文曲线会平稳很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/452f5058553a0851.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/23507faecfcdd3a4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/1532016a094cf753.png)

