---
title: OpenClaw 会话隔离实践：让子 Agent 只带结果回家
feedId: 42148
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 的主会话是整个系统的"会计室"：任务规划、用户上下文、长期记忆都记在这里。子 Agent（sub-agent）则负责脏活——批量调 MCP 工具、抓网页、跑插件流水线。问题在于，如果子 Agent 的工具调用记录原样写回主会话 transcript，一次抓取就能带进上万 token，主会话上下文迅速被冲爆，模型开始"选择性失忆"。我们团队在自动化采集场景里踩过完整的坑，这里把隔离方案整理出来。

## 问题拆解

污染分三类，处理方式不同：

1. **上下文污染**：子 Agent 的中间 tool call 全量回写主会话，最常见也最致命；
2. **状态污染**：子 Agent 写了主会话的 memory 文件，或改动了共享 workspace 里的文件；
3. **行为污染**：子 Agent 注册的临时 MCP 工具、hook 在任务结束后仍然挂在主会话里生效。

## 做法

核心原则一句话：**子 Agent 只回传结果，不透传过程**。以我们用的版本为例（字段名可能随版本有差异，思路通用）：

```yaml
subagent:
  session: isolated        # 独立 session，不共享主 transcript
  tools: [mcp:search, mcp:fetch]   # 工具白名单，只挂必需的 MCP server
  workspace: /tmp/claw-tasks/{task_id}  # 独立工作目录
  memory: readonly         # 禁止写主 memory
  return: summary          # 只回收摘要 + 结构化结果
  ttl: 30m                 # 任务结束自动销毁
```

步骤上建议按顺序做：

1. spawn 时显式声明 `session: isolated`，不要依赖默认值；
2. 用白名单收紧子 Agent 可见的工具面，没挂载的 MCP server 等于不存在；
3. 给每个子 Agent 分配独立 workspace，结束后只回收 `result` 字段；
4. 主会话侧约定解析格式，比如 JSON：`{summary, artifacts, errors}`；
5. TTL 到期自动清理 session 目录和临时注册的 hook。

## 踩坑点

- **共享 workspace**：图省事让多个子 Agent 共用一个目录，并发写同一文件直接互相覆盖。临时目录按 task_id 隔离是底线。
- **summary 粒度**：太短丢关键信息，太长等于没隔离。我们限制在 300 token 左右并强制结构化输出，超出部分落盘到 artifacts 文件，主会话里只留路径。
- **resume 语义**：隔离 session 销毁后不可恢复，别指望从子会话续聊。需要长对话的任务本来就不该拆成子 Agent。
- **插件全局 hook**：某些插件注册的是全局 hook，在子 Agent 里也会触发。如果 hook 不幂等（比如重复计数、重复发通知），记得在 spawn 配置里显式关闭。

## 可复用建议

- 临时目录统一命名规范（如 `claw-task-{id}-{date}`），方便批量清理和审计；
- 子 Agent 的完整过程日志落盘到文件，主会话需要时再按需读取，而不是预加载进上下文；
- 把"只回传结构化结果"写进团队的 spawn 模板，不靠每个人自觉；
- 监控主会话 token 增长曲线，出现突增先查最近一次 spawn 是否漏了 `session: isolated`——这是我们定位污染最快的信号。

## 总结

会话隔离不是什么高深机制，本质是把"过程"和"结论"分开记账。OpenClaw 提供的隔离开关足够用，真正的工作量在约定：工具白名单、结果格式、清理策略。这三件事定下来，主会话就能长期保持干净，长任务的稳定性会有肉眼可见的改善。建议先在一个高频自动化场景试点，跑两周再推广。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/7217be6c7b6e4d57.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/beece0d482014b6f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/01c5ba7427a6fee7.png)

