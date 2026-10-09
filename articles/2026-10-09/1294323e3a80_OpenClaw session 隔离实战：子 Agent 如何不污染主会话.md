---
title: OpenClaw session 隔离实战：子 Agent 如何不污染主会话
feedId: 41006
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

OpenClaw 的主会话通常跑长周期任务：多轮对话、工具调用记录、memory 读写都在同一个上下文里积累。子 Agent 的定位是干脏活——批量检索、代码审查、长文档摘要——干完就走。理想情况下，主会话只应该看到结论。

## 问题

如果 spawn 子 Agent 时不做 session 隔离，它会直接写入主会话的消息流：工具调用的原始返回、报错重试、中间推理全部留在主上下文里。我们实测过一个 20 次调用的检索子任务，把主会话撑大了约 40k tokens，之后每一轮请求都背着这些噪音，成本和模型注意力质量双降。更隐蔽的是，报错片段会被 memory 模块当成"事实"写进去，污染后续决策。

## 做法

OpenClaw 的隔离靠三件事：独立 session、显式输入、结构化回传。

**1. spawn 时声明隔离模式**

```yaml
subagent:
  name: repo-reviewer
  session: isolated         # 独立 session，不复用 parent 消息流
  context_inherit: none     # 不继承主会话历史
  return_policy: structured # 只回传结构化结果
```

**2. 需要什么就显式喂什么**

隔离后子 Agent 是"失忆"的，看不到主会话聊过什么。input 里把路径、范围、约束写清楚：

```json
{
  "inputs": {
    "repo": "/workspace/api-server",
    "scope": "diff:HEAD~1",
    "focus": ["error handling", "breaking changes"]
  }
}
```

**3. 回传走 schema**

`return_policy: structured` 时，子 Agent 输出必须匹配声明的 schema。主会话最终只收到一份摘要加 findings 数组，原始中间过程留在子 session 里，任务结束即回收。

验证方式很简单：跑同一任务，对比隔离前后的主会话 token 增量。我们的场景从 +40k 降到 +2k（仅摘要部分），效果直观。

## 踩坑点

- **`context_inherit` 的默认值**。部分配置模板下默认是 `partial`，会拖一份近期历史过去，等于半隔离。建议显式写 `none`，不要依赖默认。
- **memory 命名空间是共享的**。session 隔离不等于 memory 隔离，子 Agent 默认可能直写同一个 memory store。给子 Agent 单独 namespace，或设为只读。
- **异步回调绕过隔离**。子 Agent 异步完成任务后如果直接 append 到 parent 消息流，隔离等于白做。正确姿势是走结果队列，由主循环决定何时注入。
- **喂太少上下文**。隔离是双刃剑，该给的约束（语言、代码风格、路径约定）必须显式传，否则子 Agent 输出对不上主任务的口径。
- **子 session 复用**。高频短任务反复 spawn 时注意生命周期管理，复用旧 session 会带残留状态。

## 可复用建议

- 把子 Agent 当纯函数用：显式入参、结构化出参、对主会话无副作用。
- 主会话只保留"决策上下文"，执行细节全部下沉到子 session。
- return schema 一旦定下来就保持稳定，主会话下游的提示词不用跟着改。
- 定期审计 memory 写入来源，确认没有子 Agent 在直写主 namespace。

## 总结

session 隔离的本质是控制信息流向：默认不共享，需要才显式传，回传走契约。这三条做到位，主会话保持干净，子 Agent 也敢放开跑长任务，整体稳定性是肉眼可见的提升。如果你的多 Agent 流程还没做隔离，建议先从 `context_inherit: none` 开始改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/4eaa994c996b1855.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/00a279429be473dd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/0914f06551953979.png)

