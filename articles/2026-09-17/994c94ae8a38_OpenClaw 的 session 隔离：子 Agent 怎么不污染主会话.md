---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37946
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

我们用 OpenClaw 跑长期自动化任务时，一个很常见的模式是：主会话里挂一个常驻 Agent 负责调度，遇到重活（批量检索、代码审查、长文档摘要）就 spawn 子 Agent 去干。好处很明显，主会话上下文干净、响应快；但前提是隔离做得对。隔离做不对，子 Agent 反而会成为主会话最大的污染源。

## 问题：污染有三种形态

实践中踩过三种"污染"，性质不同，解法也不同：

1. **上下文污染**：子 Agent 复用了主会话的 session，历史消息全部混入，几千 token 的中间过程挤占主上下文，几轮之后主 Agent 开始"失忆"。
2. **状态污染**：上下文隔离了，但子 Agent 和主会话共用同一个 workspace，随手改了共享文件、写了 memory，主 Agent 后续读到的世界已经变了。
3. **输出污染**：子 Agent 的工具调用日志和中间结果被当成回复推到了频道里，用户在群里看到一屏调试信息。

## 做法：三层各管一层

**第一层：session 隔离。** 子 Agent 必须有独立的 sessionKey，不要复用 main。我们的做法是在 spawn 时显式指定子 Agent 及其 session，子 Agent 拿到的是"干净起点 + 一段明确的任务描述"，而不是主会话的完整历史。任务描述里写清楚：目标、边界、允许动哪些文件。

```json
{
  "agents": [
    { "id": "main", "workspace": "~/claw-main" },
    { "id": "worker", "workspace": "~/claw-worker" }
  ]
}
```

**第二层：workspace 隔离。** 给子 Agent 单独的 workspace 目录。需要主会话产物时显式传入路径，需要交还结果时约定一个固定位置或直接通过工具返回值带回。子 Agent 的 memory 写入默认关闭或定向到自己的目录，避免把中间结论写进主记忆库。

**第三层：输出收敛。** 约定子 Agent 只返回结构化摘要（结论 + 关键引用 + 遗留问题），中间过程不外传。关掉子 Agent 的主动消息通道，它没有"嘴"，只有返回值。

## 踩坑点

- **默认继承比想象中多**：不显式指定 session 时，部分调用路径会沿用当前会话，等发现时上下文已经混了。规则只有一条：spawn 必带显式 session 参数。
- **文件锁与并发写**：两个子 Agent 同时改共享文件比污染更糟，会直接产生脏数据。共享资源要么只读，要么加任务级的所有权约定。
- **摘要截断失真**：子 Agent 返回的摘要太激进，丢了关键前置条件，主 Agent 拿着结论继续推理就会翻车。摘要模板里强制保留"假设与限制"字段。
- **权限放大**：子 Agent 沿用了主 Agent 的全部工具权限，隔离了上下文却没隔离能力面。能用最小工具集就给最小工具集。

## 可复用建议

- 把"子 Agent 交付物"固化成一个模板：结论、证据路径、未解决问题，三段式，所有子任务通用。
- 在主 Agent 的系统提示里写死纪律："子任务结果以外的一切过程视为不存在"，避免它主动追问子 Agent 的过程细节。
- 定期审计 workspace：主目录里出现本该属于 worker 的文件，就是隔离漏了。
- 版本行为会变，升级 OpenClaw 后重点回归测一下 spawn 的 session 归属，这类改动通常不显眼但影响大。

## 总结

session 隔离本质上不是"能不能 spawn"，而是三件事各自独立做干净：上下文隔离靠独立 session，状态隔离靠独立 workspace，输出隔离靠结构化返回值加禁用主动推送。三层里任何一层偷懒，污染都会从那条缝里渗回来。把隔离约定固化成配置和模板，而不是每次靠 prompt 现场提醒，才是能长期跑自动化的形态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/b6dd71dc792eb2a4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/1de44f80a747eb40.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/1ae7d911e1b30f04.png)

