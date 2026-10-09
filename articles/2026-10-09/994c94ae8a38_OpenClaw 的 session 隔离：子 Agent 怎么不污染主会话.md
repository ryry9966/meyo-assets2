---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41039
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

OpenClaw 的主会话是长期上下文的载体：和用户的日常对话、系统提示、workspace 注入的内容，都在同一个 session 里滚动。窗口有限，塞满了就只能靠 compaction 压缩，细节会丢。

子 Agent（通过 `sessions_spawn` 拉起的会话）的设计初衷就是干脏活：批量读文件、抓网页、跑长检索。它有独立的 session 和独立上下文，跑完后只把一条 result 回传主会话。这套机制好用，但前提是你真的在用它。

## 问题

实际跑下来，“污染”通常发生在三种情况：

1. **该 spawn 的没 spawn**。主 agent 直接在主会话里批量执行工具，几十条 tool result 全部进主上下文，紧接着触发 compaction，把真正重要的对话细节挤掉。
2. **spawn 了但串了**。sessionKey 复用旧值，子 agent 起来时带着上一轮任务的残留上下文，结论里混进不相关信息。
3. **写共享资源**。子 agent 往 MEMORY.md 或共享 workspace 写文件，长期记忆被中间产物占据。

## 做法

以我们网关上的配置为例（键名以你的版本为准）：

1. **开启 sandbox 隔离**。`~/.openclaw/openclaw.json` 里把 `agents.defaults.sandbox.mode` 设为 `all`（至少 `non-main`）。spawn 出的子会话跑在一次性容器里，文件系统天然隔离，任务结束即销毁。
2. **任务描述里写死“只回摘要”**。spawn 的 task 中显式要求：输出不超过 N 行、只给结论和关键路径、不复述过程。主会话收到的就只有这一条 final message。
3. **在 AGENTS.md 里立规矩**：批量操作一律 spawn；主会话禁止直接调用会产生大输出的工具；子 agent 不写 MEMORY.md，需要沉淀的结论由主会话确认后落盘。
4. **用 `sessions_list` 定期体检**。看有没有僵尸子会话、主会话是否异常膨胀，必要时 reset。

## 踩坑点

- **“摘要”也可能很长**。有一次抓取任务回传了两千字，主会话照样被撑一截。后来把字数上限写进 task 模板才稳定。
- **调试时关 sandbox 图方便**，子会话落回主 agent 的 session 目录，排查时极易看混。调完记得改回来。
- **子 agent 带消息发送权限时**，可能直接回复到主频道，用户视角就是“人格分裂”。要么收权，要么在提示里禁止它主动发消息。

## 可复用建议

- 把“什么任务必须 spawn”写成团队约定放进 AGENTS.md，不靠模型自觉。
- sessionKey 用语义化命名，比如 `research-<日期>`，方便事后审计和清理。
- 大 IO、大检索、大文件分析，一律视为子 agent 任务；主会话只做编排和确认。

## 总结

Session 隔离的本质是上下文预算管理：主会话的每一个 token 都该花在用户关心的东西上。spawn + sandbox + 回传摘要，三件事做到位，子 Agent 就是干完即走的干净临时工，而不是把泥带进客厅的客人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/3260c7034e33f56f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/2e29cfbb431a7bae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/f498dfb24cab1c79.png)

