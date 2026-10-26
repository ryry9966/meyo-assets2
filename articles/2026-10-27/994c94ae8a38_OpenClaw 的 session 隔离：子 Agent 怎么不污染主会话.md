---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42699
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

OpenClaw 里每个 agent 都有独立的 session 存储，主会话一般绑定某个 IM 渠道或定时入口，是长期运行的编排中枢。跑得久之后会遇到一个经典问题：主 agent 自己干重活——批量读文件、抓网页、长链路推理——几十条 tool call 和大段输出全部落进同一个 transcript，上下文迅速膨胀。

## 问题

不隔离的代价很具体：

- token 成本和 compaction 频率同步上升，早期指令容易被压缩掉；
- 主 agent 被中间过程淹没后丢焦点，编排质量下降；
- 重放主会话排障时噪音极大，关键决策被工具输出埋掉。

## 做法

1. **派生而不是原地执行**。用 `sessions_spawn` 把重活交给子 agent：子任务拿到全新的 session id 和独立 transcript，与主会话上下文物理分离。
2. **任务简报自包含**。子 agent 默认看不到主会话，spawn 时要把目标、输入路径、约束、交付格式、停止条件一次写全，当成给新同事发需求文档。
3. **限定回传**。明确要求只返回一段有界摘要，由主会话决定是否 announce 到渠道；中间 tool call 留在子 session 里。
4. **配置层再补一刀**。agents 各自配 workspace、模型和工具白名单；涉及文件写操作就开 Docker sandbox；memory 目录按 agent 拆开，避免互相覆盖。
5. **验证**。到 `~/.openclaw/agents/<id>/sessions/` 确认生成了独立 jsonl，主会话里只应看到 spawn 调用和最终摘要。

## 踩坑点

- 默认假设子 agent 能"看到"主对话——它看不到，缺的上下文必须显式给；
- 摘要不设上限，回传直接把渠道刷屏；
- 多个 agent 共写同一个 memory 文件，造成跨会话污染，改成 per-agent 目录或只读挂载；
- 子 agent 的 sandbox 权限和主 agent 不一致，静默失败，先查 gateway 日志再怀疑提示词；
- 子 agent 里再 spawn，形成递归，要限制深度和并发；
- 排障时发现主 transcript 里没有子 agent 的推理过程——这是特性不是 bug，去翻对应的子 session 文件。

## 可复用建议

- 固化一份 spawn 简报模板（目标 / 输入 / 约束 / 输出 / 停止条件），随用随填；
- 一次 spawn 只干一件事，不要复用子 session 塞不相关任务；
- 主会话定位成调度器：拆解、派发、验收，不亲自搬砖；
- 定期清理过期 session 文件并监控目录体积，尤其是高频自动化场景。

## 总结

子 agent 隔离的本质就三件事：**独立 transcript、显式上下文交接、有界回传**。把主会话当成 dispatcher，把脏活和噪音关进子 session，主链路才能长期保持干净可控。具体参数以你当前版本文档为准，但这套模式在多 agent 编排里是通用的，值得直接固化成团队的 spawn 规范。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/c38e78a5f6ad4d8c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/3c79c16e5a810b7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f10aef419d9045f8.png)

