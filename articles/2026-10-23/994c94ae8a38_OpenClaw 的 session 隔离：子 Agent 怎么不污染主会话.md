---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42315
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

OpenClaw 的典型形态是一个常驻 gateway,主 agent 在固定的 main session 里跟你对话。日常问答没问题，但一遇到跑批、长重构、探索性调试这类活，让主 agent 亲自下场跑长循环，上下文很快就会被工具输出灌满。子 agent 的设计初衷就是解决这件事：通过 `sessions_spawn` 拉起独立 session,有自己的 session key、独立的上下文窗口和 transcript,主会话历史不会被中间过程波及。

## 问题

“有隔离机制”不等于“自动干净”。实际用下来见过三类污染：

1. **上下文污染**：任务全程发生在主 session,几百条工具输出触发提前 compaction,把真正重要的历史挤掉；
2. **间接污染**：子 agent 和主 agent 共用同一个 workspace,顺手改了 AGENTS.md 或记忆文件，下次主会话加载就“串味”；
3. **通知污染**：子 agent 往主频道高频报进度，这些消息被写进主 session 的路由历史，长任务跑一夜，主会话直接膨胀。

## 做法与步骤

1. 长任务一律 `sessions_spawn`,不要让主 agent 自己跑循环。子 agent 天然拿到干净上下文；
2. **按会话传值，不按历史传值**：task prompt 里写清目标、涉及的文件路径、约束和期望产出格式，别指望子 agent 去翻主会话历史；
3. **约定返回契约**：子 agent 只回结构化摘要——结论、改动文件清单、遗留风险，细节留在自己 session。主 agent 真需要时用 `sessions_history` 按需读取，不要全量拉回；
4. **workspace 物理隔离**：探索型子 agent 指向独立目录或 sandbox,明确禁止写 AGENTS.md、MEMORY.md 等共享状态文件；
5. 一次性任务用完即弃，子 session 闲置过期即可；别复用探索型 session 干正式活，并定期清理 `sessions_list` 里的残留；
6. 大任务开跑前给主会话来一次 `/new`,让主线也保持干净。

## 踩坑点

- **session 隔离 ≠ 状态隔离**。文件系统是共享的，子 agent 改了工作区文件，污染照样发生，目录边界要在配置里卡死；
- **子 agent 继承工具权限**。“进不了主会话”不等于“干不了坏事”——不可逆操作(发消息、删文件)建议只留给主 agent,或单独立一个低权限 agent 跑子任务；
- 高频进度通知看似无害，实际都在往主 session 的历史里写，改成阶段性摘要，或结束时一次性回报；
- 子 agent 超时或被杀后的残留 session 不会主动提醒你，记得人工巡检。

## 可复用建议

把子 agent 当成“带干净白板的同事”：交接靠书面材料(task prompt 与产出报告)，不靠共享记忆；活干完白板就擦了，但他桌上落的文件(workspace)需要你提前划好范围。一句话：session 隔离解决的是上下文污染，解决不了文件与权限的共享，后者要靠 workspace 边界和权限配置兜底。

## 总结

子 agent 不是第二个“你”，而是一次性的干净执行环境。管住三件事——进什么(prompt 最小化)、出什么(摘要契约)、共享什么(workspace 边界)，主会话就能长期保持轻量、可预测。隔离是默认给的，干净是要自己设计的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/6bbbd7f9cab3f66d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/d35a912fef0f0251.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/3ca3c16118f641d5.png)

