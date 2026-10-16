---
title: OpenClaw session 隔离实战：让子 Agent 别把主会话搞脏
feedId: 41753
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 的主会话是长期运行的：用户偏好、项目上下文、历史工具调用都堆在同一个 context 里。一旦开始用子 Agent 做批量任务——检索、代码验证、多文件重构——问题就来了：如果子 Agent 和主会话共用一个 session，它的中间输出、报错重试、工具原始返回会全部灌进主上下文。

我们在一个多仓库重构任务里实测过：三个未隔离的子 Agent 跑完，主会话 compaction 提前触发两轮，早期定下的重构约定被压缩掉了，后续执行开始“失忆”。这才认真做了 session 隔离。

## 问题：污染的三条路径

1. **上下文污染**：子 Agent 的工具输出和中间推理直接写进主 transcript，挤占有效上下文。
2. **状态泄漏**：子 Agent 调用 memory 写入等有状态工具，把未验证的中间结论“刻”进主会话的长期状态。
3. **预算挤占**：主会话 token 消耗失控，compaction 边界后移，重要的旧上下文被误压缩。

## 做法：三层隔离 + 一个契约

**会话层**：spawn 时显式声明隔离，子 Agent 拿到独立 session id（建议自动加 uuid 后缀，避免并发冲突）：

```yaml
subagent:
  session: isolated
  workspace: scratch        # 独立临时目录，任务结束可清理
  tools:
    deny: [memory_write, session_emit]   # 有状态工具默认拒绝
  budget:
    max_tokens: 32000
  return: task_result       # 只回传结构化结果
```

字段名随版本略有差异，思路一致。

**工作区层**：子 Agent 的文件读写限定在自己的 scratch 目录，产物落盘后只把路径交给父会话，不把内容搬回来。

**工具层**：给子 Agent 做允许清单。能只读的不给写，能不给网络的不给网络。

**回传契约**：这是关键。父子之间只传一个 schema 化的结果对象——结论、产物路径、失败原因、耗时。完整 transcript 单独落盘到 debug 目录，主会话只留引用。排查时能回放，但不会撑大上下文。

## 踩坑点

- **忘了禁 memory 类工具**。子 Agent 把中间结论写进共享记忆，主会话下一轮直接“记得”错误答案。有状态工具默认拒绝，按需放行。
- **brief 太薄或太厚**。太薄，子 Agent 缺上下文开始自由发挥；太厚，等于没隔离。经验是只给“目标 + 约束 + 返回 schema”，不给历史对话。
- **并发 spawn 串写同一 scratch 目录**，两个子会话互相覆盖产物。session id 和 workspace 都要唯一。
- **返回结果超预算被截断**，关键字段丢失。schema 里把结论和产物路径设为必填，超限时让子 Agent 先自行压缩再返回。
- **嵌套 spawn 未继承隔离策略**，孙 Agent 的输出又回到主会话。isolation 应默认继承，除非显式放开。

## 可复用建议

1. 把父子边界当 API 设计：返回值 schema 化，传引用不传内容。
2. 子会话默认最小权限，deny 优先于 allow。
3. 写一个“噪声子 Agent”（故意刷屏输出）做回归用例，验证隔离后主会话 token 曲线平稳。
4. 预算写进配置，别靠 prompt 里一句“请节约 token”。

## 总结

session 隔离省的不只是 token，更重要的是划分了故障域：子 Agent 的失败、噪声、脏状态都停在它自己的会话里，主会话长期保持可预测。配置成本半小时，换来的是长任务链路下的稳定性。如果你的部署已经开始频繁 compaction，建议从这里下手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/b292f10bdf138b4f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/48d409aabfa4565e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/1584211d7e707838.png)

