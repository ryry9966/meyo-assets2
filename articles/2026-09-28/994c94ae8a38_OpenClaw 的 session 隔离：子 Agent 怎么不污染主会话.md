---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39235
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的主会话（比如 `agent:main:telegram:xxx` 这样的 session key）本质是一个持续累积的上下文文件：用户消息、模型回复、每次工具调用的结果都会追加进去。单轮问答场景下这个设计没问题，但一旦开始用 `sessions_spawn` 派生子 Agent 做重活——深度调研、批量代码改造、长时间巡检——问题就暴露了。

## 问题

子 Agent 的工具调用链往往很长：搜索、抓网页、读文件、再搜索。如果这些中间过程全部写回主会话，会看到三个典型症状：

1. 主会话上下文膨胀，compaction 提前触发，历史被压缩，长期记忆质量下降；
2. token 成本翻倍，每次主会话推理都要背着这堆过程数据；
3. 注意力被污染，后续回答会引用早已过时的中间结论。

## 做法：隔离在哪几层

OpenClaw 的隔离大致三层，理解了才知道怎么用对：

**第一层：独立 session。** `sessions_spawn` 会给子 Agent 分配独立的 session 文件和 session key，子 Agent 的完整工具轨迹、多轮推理都留在自己的 session 里，主会话只看到一行"任务已派生"。

**第二层：只回传结果。** 子 Agent 跑完后，返回给主会话的是最终结果或摘要，不是完整 transcript。所以写 spawn 的任务描述时要明确要求子 Agent 输出结构化结论（JSON 或分点摘要），而不是过程叙述。

**第三层：结束后回收。** 已完成的子 Agent run 会被清理，避免 session 文件无限堆积。需要回查过程时，直接去磁盘上的 session 存档里 grep，不要把原始 transcript 塞回主会话。

推荐的任务拆分模式：

```text
主会话：判断意图 → 拆任务 → spawn 子 Agent（附最小必要上下文）
子 Agent：独立执行 → 输出结构化结果
主会话：拿到结果 → 决策 → 统一回复用户
```

## 踩坑点

- **把主会话历史整个塞进 spawn prompt。** 这等于把污染手动搬运过去，子 Agent 上下文同样爆炸。只给任务描述加上必要的文件路径和变量。
- **子 Agent 之间传大段文本。** 让它们通过 workspace 里的文件交接，会话里只传路径，不传内容。
- **不限制派生深度。** 子 Agent 再 spawn 子 Agent，几层下来成本和失控风险都会上来，自己要守住层级。
- **子 Agent 直接对外发消息。** 如果它写的是同一个频道，用户视角就是"上下文错乱"。让子 Agent 只回传结果，对外出口收归主会话。

## 可复用建议

1. 把主会话当调度器：只做意图判断、任务拆分、最终决策；
2. 一个子 Agent 一个 session 一个关注点，不要塞多任务；
3. 跨 Agent 传数据走文件，不走上下文；
4. 定期检查 session 存档目录，确认回收逻辑在正常工作；
5. spawn prompt 里显式规定输出格式——回传质量决定主会话质量。

## 总结

session 隔离不是框架的魔法，而是写入纪律：子 Agent 保留全部过程，主会话只接收结论。守住"过程留在子 session、结果才回主会话、数据走文件不走上下文"这三条，多 Agent 并发时主会话依然干净可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/dcef497c7a865e87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/32fe1cc9ccbbb1d1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/9dae6fcc998578ca.png)

