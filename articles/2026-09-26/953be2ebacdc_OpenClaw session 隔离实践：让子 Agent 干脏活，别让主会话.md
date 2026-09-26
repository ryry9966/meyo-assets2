---
title: OpenClaw session 隔离实践：让子 Agent 干脏活，别让主会话买单
feedId: 39145
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 里每个 channel/agent 默认绑定一个主 session，你的消息、agent 的工具调用、工具返回，全部写进同一份 transcript（`~/.openclaw/agents/<agentId>/sessions/` 下的 jsonl 文件）。日常问答没问题，但一旦让它跑重活——批量抓网页、跑分析脚本、多文件重构——所有中间产物都堆在主会话上下文里。

## 问题

直接在主会话里执行重任务，会遇到三个典型症状：

1. **上下文膨胀**：几轮工具调用后 context 被输出塞满，早期指令被稀释，agent 开始"忘事"；
2. **成本上升**：每轮都带着历史里的全量工具输出重复计费；
3. **垃圾留存**：失败的尝试、重试的中间输出永久留在主 session 历史里，后续对话持续背锅。

这就是所谓"主会话污染"。解法不是省着用，而是把上下文边界划清楚。

## 做法

**1. 重活派生独立 session。** 让主 agent 通过 `sessions_spawn` 起一个子 agent，分配独立的 agentId 和 workspace。子 agent 有自己的 transcript，干完即弃，主会话 history 不动。

**2. 显式注入任务上下文。** 子 agent 看不到主会话历史。spawn 时的 prompt 必须写清：目标、输入路径、期望输出格式。不要指望它"自己明白"。

**3. 约定回传协议。** 子 agent 只回结构化摘要：结论 + 关键文件路径 + 失败项清单，并限制长度。全量输出落在子 agent 自己 workspace 的文件里，主会话只拿指针，不拿内容。

**4. 执行环境隔离。** 涉及跑脚本、装依赖的任务，套 Docker sandbox，文件变动收敛在独立工作区内，避免误伤主 workspace。

**5. 收尾清理。** 任务结束用 `sessions_list` 检查，归档或删除一次性子 session，防止孤儿堆积。

## 踩坑点

- **共用 workspace 互相覆盖**：没给子 agent 独立目录时，它可能和主会话写同一个文件。强制约定 `workspaces/<taskId>/` 子目录。
- **回传不设限等于换个地方污染**：子 agent 摘要写太长，主会话照样爆。回传前先约定条目数和字数上限。
- **调试时的 session 错觉**：同一个聊天窗口默认同一个 session。你以为新开了会话测 prompt，其实还挂在旧上下文里。先 `sessions_list` 确认，或干脆换 agentId。
- **长任务变孤儿**：子 agent 卡死没有超时控制，session 文件越积越多。给 spawn 的任务加超时，定期清 sessions 目录。

## 可复用建议

- 把高频重活做成**固定模板**：固定系统提示 + 固定工具白名单 + 固定输出 schema，spawn 时只填参数。
- 主会话定位是**调度与决策**，执行一律下沉到子 agent。
- 回传统一用 JSON 摘要，主会话拿到后可以继续编排下一步，而不是解析自然语言。
- 重要结论**落盘到 memory/workspace 文档**，不要只留在 transcript 里——session 可以丢，结论不能丢。

## 总结

session 隔离的本质是控制上下文边界：主会话保持轻、稳定、可预期，子 agent 负责脏活、可丢弃、可复现。上下文是预算，隔离是最便宜的省钱手段——与其事后压缩历史，不如一开始就别让脏东西进主会话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a8f38c0a3c58d3a8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/94e06219c63fe99a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/d41ff66eb3463b7a.png)

