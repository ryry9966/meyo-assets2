---
title: 子 Agent 不污染主会话：OpenClaw session 隁离的四层边界实践
feedId: 40404
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

用 OpenClaw 跑长任务时，主会话是唯一带“连续记忆”的地方：对话历史、工具输出、memory 索引都堆在这里。一旦开始用子 Agent 做并行检索、批量处理、MCP 长链路调用，很快会撞上同一个问题——主会话越来越脏。

## 问题：污染的三条路径

1. **上下文膨胀**。子 Agent 的中间过程写回主会话，几千 token 的工具输出挤占窗口，模型开始“忘”早期的关键约定。
2. **记忆污染**。子任务的临时结论被自动写入 memory，几天后主 Agent 引用了一条过时甚至错误的“事实”。
3. **状态串台**。两个子 Agent 共享同一个 workspace 或同一个 MCP server 实例，文件和缓存互相覆盖。

## 做法：把隔离拆成四层边界

**1. 上下文边界。** spawn 子 Agent 时不传完整历史，只传任务简报：目标、约束、输入路径、返回格式。子 Agent 有自己独立的 session 文件（以我用的版本为例，在 sessions 目录下按独立 id 存放），过程记录全部留在子 session 里。

**2. 返回契约。** 子 Agent 结束时不回灌原始 transcript，只返回结构化摘要，比如固定 schema：`{"status", "summary", "artifacts", "errors"}`。主会话只吃摘要，想看细节去读子 session 文件。

**3. 工具与文件边界。** 给子 Agent 分配独立 workspace 子目录（按 session id 命名），工具白名单收窄到任务必需的 MCP server，并且不授予 memory 写权限。

**4. 记忆提升显式化。** 子 Agent 的结论默认不进长期记忆，由主 Agent 在主会话里判断“这条值得留”再写入。宁可多一步确认。

一个伪配置示意（字段名以你手上的版本为准）：

```yaml
spawn:
  context: brief        # 不携带父会话历史
  session: isolated     # 独立 session 文件
  workspace: per-session
  tools: [search, fs:scoped]
  return: schema        # 只回结构化摘要
```

## 踩坑点

- **默认继承**：不显式声明 context 时，部分版本会带上父会话尾部历史，省了 token 也把脏上下文带了过去。升级后记得回归测一下。
- **自动索引**：workspace 里的临时文件会被 memory 索引任务扫到。把子目录排除出索引路径，或统一加临时后缀。
- **MCP 全局状态**：有些 MCP server 是进程级单例，自带缓存和会话态，多个子 Agent 共用一个实例就会串数据。要么按 session 传 namespace 参数，要么每个 session 起独立实例。
- **过度隔离**：简报写太省，子 Agent 缺关键约定，返回结果不可用，重试反而制造更多垃圾。隔离不等于信息饥饿，验收标准必须写进简报。
- **子 session 无限增长**：子 Agent 自己也会爆上下文，长任务在子 Agent 内部同样要分段，或设 TTL 定期归档。

## 可复用建议

- 把“任务简报 + 返回 schema”做成团队模板，比每次手写稳定得多。
- 主会话定期体检：统计 token 分布，工具输出占比过半，就考虑把这类调用下沉为子 Agent。
- 排障时先读子 session 文件再问模型——它是最诚实的日志。
- memory 写入加“来源”字段，标明来自哪个子 session，事后好清理。

## 总结

session 隔离本质是四条边界：上下文、工具、文件、记忆。原则一句话——**默认隔离、显式回流**：子 Agent 的一切产出必须过一道契约才能进主会话。做到这一点，新增并行任务时，主会话依然是干净、可预测的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/59b647effa2ea2c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/950bdd071534d85f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/b7ef41c90522a898.png)

