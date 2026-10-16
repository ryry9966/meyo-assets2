---
title: OpenClaw session 隔离实践：让子 Agent 别弄脏主会话
feedId: 41689
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

OpenClaw 的主会话（比如绑定 Telegram 的那个 session）是长跑型选手：用户消息、工具调用、MCP 返回、插件输出，全部追加在同一条 JSONL 里。跑久了有两个典型症状：一是上下文膨胀，compaction 一压缩，早期细节就丢；二是子任务的过程性输出把主会话污染——你只是让 agent 查个日志，它翻了几十个工具来回，中间噪音全记在主会话账上。

OpenClaw 给出的解法是子 agent（`sessions_spawn`）：子 agent 有自己独立的 session 文件，过程留在子会话，只把结论回传主会话。原理不复杂，但实践里有几个坑，不踩一次不知道。

## 做法

1. **调度下沉**。主会话只做三件事：理解意图、拆任务、汇总答复。凡是需要多轮工具调用的活，一律 spawn 子 agent 去做。
2. **最小上下文传递**。给子 agent 的 prompt 只写任务目标、约束和期望输出格式，不要把主会话历史整段贴进去。既省 token，也缩小敏感信息的扩散面。
3. **约定回传格式**。在 prompt 里明确“只回传十行以内的结论 + 产物文件路径”。这样子会话里几十次工具调用，出不了子会话的边界。
4. **写操作隔离**。子 agent 需要改文件或写 memory 时，要么指向独立 workspace，要么只给读权限；memory 写入收敛到主 agent 一处。
5. **验证**。跑完后用 `/sessions`（或直接看 `~/.openclaw/agents/<agent-id>/sessions/` 下的 JSONL）确认：子会话独立成文件，主会话只新增了一条结构化结果。

## 踩坑点

- **taskKey 复用**：同名 taskKey 会撞“任务已存在”，或者复用旧 session 拿到陈旧上下文。命名带上日期和序号，别偷懒。
- **子 agent 越权回复**：如果不收敛消息类工具，子 agent 可能直接往频道发消息，用户收到两条口径不一的回复。子 agent prompt 里显式禁止对外发送。
- **memory 并发写**：父子会话同时写 MEMORY.md 会互相覆盖，而且没有任何提示。立规矩：memory 只允许主会话写。
- **回传内容过大**：子 agent 把整个文件内容塞回主会话，等于白隔离。回传限长，大产物落盘给路径。
- **定时任务走主会话**：cron 任务如果在主 session 里跑长活，就和用户对话挤同一条上下文。定时任务同样应该落在独立子会话。

## 可复用建议

- 一条纪律记住就够：**控制流在主会话，数据流在子会话**。
- 按来源分域：一个 MCP server 对应一个专职子 agent，工具集天然不串。
- 回传契约固定成模板，子 agent 的输出才可 diff、可审计。
- 定期看 sessions 目录体积和子会话数量，异常膨胀说明隔离在某个环节漏了。

## 总结

Session 隔离不是一个配置开关，更像是使用习惯：主会话只做调度和最终答复，过程性输出全部下沉，写操作收敛。把“结论回传、过程留痕、写操作收敛”这三条固化下来，主会话能长期保持干净，多 agent 协作也才真正可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/d1d769fc402fd912.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/d523fb0eaf29823e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/740a9fc0665263f1.png)

