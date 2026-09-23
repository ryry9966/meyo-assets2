---
title: OpenClaw session 隔离实践：让子 Agent 干完活，主会话还是干净的
feedId: 38630
source: 综合讨论
publishedAt: 2026-09-23
---

# 背景

用 OpenClaw 跑长任务时，主会话是唯一需要长期维护的资产：用户偏好、项目约定、历史决策都在里面。子 Agent（subagent）负责干脏活——检索代码、跑测试、抓文档、批量试错。问题在于，如果子 Agent 的执行过程直接写回主会话，主窗口很快就会被工具输出淹没。

# 问题

不隔离时常见的三种污染：

1. **上下文膨胀**：子 Agent 的中间 tool result（日志、堆栈、文件内容）全量回流，几千 token 说没就没，后续推理质量肉眼可见地下降。
2. **状态串扰**：子 Agent 顺手改了共享 memory、todo 列表或环境变量，留下大量"半成品状态"，主 Agent 下一步被误导。
3. **重放失控**：需要复盘或回放主会话时，子 Agent 的所有中间步骤也被拖进来，审计成本极高。

# 做法

OpenClaw 的 session 隔离核心是三件事：独立 transcript、受限回流、写权限命名空间。配置示例：

```yaml
agents:
  explorer:
    session: ephemeral          # 独立 session id，transcript 落到自己的目录
    return: summary             # 结束时只回流 final summary
    summary_budget: 2048        # 摘要 token 上限
    write_scope: .agents/explorer/   # 写操作限定在子作用域
    inherit_env: [PATH, HOME]   # 环境变量白名单继承
```

落地步骤：

1. **子 Agent 用独立 session id**，所有中间输出只写自己的 transcript 目录，主会话不可见。
2. **定义归还协议**：子 Agent 结束时输出结构化 summary（结论 + 风险 + 产物路径），主会话只注入这一段。
3. **写权限隔离**：memory / todo 的写操作指向子作用域，主 memory 对子 Agent 只读。
4. **显式提交点**：只有主 Agent 主动调用 merge 类操作，子 Agent 产物才进入主会话状态。
5. **进度走旁路**：流式进度用独立的 progress 通道，不写 transcript。

# 踩坑点

- **summary_budget 压得太小**：探索类任务结论被截断，主 Agent 拿到残缺信息做错决策。建议按任务类型分档调，探索类给到 2k 以上。
- **只读没锁死**：以为子 Agent 只读主 memory 就安全，结果某个插件在子会话里触发了写路径。上线前务必显式 `write_scope`，不要依赖默认行为。
- **并行子 Agent 共享目录**：两个任务写同一个 `.agents/` 子目录互相覆盖，task-id 命名空间必须唯一。
- **忘了超时**：僵尸子 Agent 一直占着并发额度和 token 预算，`timeout` 和 `max_tool_calls` 一定要设。
- **把 trace 当回流**：调试时开的 trace 开关忘了关，trace 混进 summary 回流。trace 应该落盘到子目录，永远不进主窗口。

# 可复用建议

- 把"回流内容"当成 API 来设计：summary + artifacts 引用，而不是全量日志。产物留在子作用域，主会话只拿路径。
- 需要审计时给子 Agent 单独开 trace，落盘不进窗口，重放主会话时干干净净。
- 写一个长会话回归脚本，对比开/关隔离后的 token 消耗和答案质量，隔离收益最好用数据说话，我们对一个 40 轮的检索任务测过，主会话 token 消耗降了约 60%。

# 总结

一句话原则：**主会话只保留决策所需的最小上下文，其余全部留在子会话，用明确的归还协议控制信息流。** 隔离不是功能开关，而是一种约定——谁产生数据，谁负责决定哪些数据值得回到主会话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/61dbcb487b70cac4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/78df998923afa448.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/dd6f43fb5d8c5e07.png)

