---
title: OpenClaw session 隔离实践：让子 Agent 不污染主会话
feedId: 40559
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

OpenClaw 的主会话是绑定渠道的长期会话：Telegram 群、WhatsApp 对话各自对应一个持续滚动的 session，历史消息、工具调用、memory 注入全在同一份上下文里。这套设计日常问答很好用，但一旦在主会话里跑重活——批量抓网页、跑评测、做大范围重构——问题就来了。

## 问题：两类污染

一是**上下文污染**。几十次工具调用的中间输出把窗口撑满，触发 compaction 之后，早期的项目约定、用户偏好被压缩掉，主 agent 之后明显“变笨”。

二是**状态污染**。子任务顺手改了 workspace 里的 AGENTS.md、memory 或共享配置，主会话后续行为被悄悄改变，而且很难追溯。

session 隔离要解决的就是这两件事。

## 做法

1. **重活用 `sessions_spawn` 派发。** 子 agent 拿到独立的 session 文件和独立上下文，中间推理、工具输出都留在子会话里，跑完只把最终结果作为一条消息回传主会话。
2. **约定返回格式。** 在 spawn 提示词里明确：只返回结论、关键数据和产物路径，控制在十几行内。回传的是摘要，不是日志。
3. **大产物落盘到独立目录。** 让子 agent 把报告、数据写到 `workspace/runs/<task-id>/` 这类约定位置，主会话只引用路径，需要时再按需读文件。
4. **裁剪子 agent 工具面。** 按任务给最小工具集：只读任务不给写工具；不给发消息类工具，避免子 agent 半路往群里插话。
5. **用 git 兜底状态。** 跑批量任务前 commit 一次，结束后 diff workspace，确认 AGENTS.md 和 memory 没被子 agent 动过。

## 踩坑点

- **session 隔离 ≠ 文件隔离。** 子 agent 的上下文是分开的，但 workspace 文件系统是共享的。两个子 agent 并发写同一个文件会互相覆盖，并发任务先分目录。
- **子 agent 默认继承工具配置。** 不显式裁剪，它可能拿到和主 agent 一样的能力，包括往外发消息。
- **回传失控。** 提示词不限制的话，子 agent 很乐意把几百行日志塞进结果——污染只是换了个会话发生。
- **中断后的产物。** 子 agent 超时被杀，主会话只收到失败信息；提前约定落盘位置，中断后还能去 runs 目录捡回半成品。

## 可复用建议

- 把 spawn 提示词做成固定模板：**目标 / 约束（允许写哪个目录、禁用哪些工具）/ 返回格式（要点式、限行数）**。
- 在 AGENTS.md 里写清路由规则：批量、可再生、与当前对话无关的任务一律走子 agent。
- 定期清理 runs 目录，别让 workspace 无限膨胀。
- 排查“主 agent 行为变了”这类玄学问题时，先 diff workspace 和 memory，再翻主会话日志。

## 总结

OpenClaw 的 session 隔离解决的是上下文层面的问题：子 agent 的思考过程和中间输出不会挤占主会话。但文件和 memory 是共享的，状态隔离要自己补。一句话：**上下文交给框架，落盘和回传纪律靠自己。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/d7cea96ce925babc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/12b7839e646256a9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/09128d6157a3debb.png)

