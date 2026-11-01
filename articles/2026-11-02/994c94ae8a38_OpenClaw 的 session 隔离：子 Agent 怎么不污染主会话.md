---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 43212
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 的主会话是你和 agent 长期对话的地方。问题在于：每一次工具调用——读文件、抓网页、搜代码——输出都会进主会话上下文。跑得久了上下文膨胀，触发压缩（compaction）后模型开始"忘事"，指令遵循和召回质量肉眼可见地下滑。这是长跑型 agent 的通病。

OpenClaw 的解法之一是子 agent：通过 `sessions_spawn` 拉起一个独立 session，它有自己的上下文窗口和完整的工具调用轨迹，跑完后只把最终结果回传主会话，中间过程对主会话不可见。

## 问题

第一次用子 agent 通常踩两类坑：一是以为它"知道"主会话里聊过什么，任务描述写得太省，子 agent 开始自由发挥；二是任由它回传大段原文，隔离了个寂寞，主会话照样被撑爆。这篇讲清楚隔离边界在哪、怎么用才真正省上下文。

## 做法

1. **判断什么活值得派。** 经验值：预计超过 5–10 次工具调用，或要吞大量原文（长网页、多文件扫描、批量重构）的任务，值得 spawn。单次工具调用的小事直接在主会话做，spawn 的调度开销不值。
2. **任务简报写全。** 子 agent 看不到主会话，`task` 里要包含：目标、相关路径/URL、约束（比如"只读不改"）、期望输出格式。关键是显式写死输出上限。
3. **加 label 追踪。** spawn 时带 `label`，之后用 `sessions_list` 看状态，跑偏了能及时发现。
4. **需要续聊的用 `sessions_send`** 往同一 session 追加消息，而不是每次新开一个。

```
sessions_spawn:
  task: 阅读 docs/spec/*.md，总结与支付流程相关的变更点；
        只读不改；输出不超过 15 行 bullet，
        每条标注文件名和行号
  label: spec-review-payments
```

## 踩坑点

- **回传体积失控：** 简报里不限格式，子 agent 可能贴回几百行原文——主会话一样爆炸。永远在 task 里写死行数/token 上限。
- **并发写冲突：** 并行 spawn 两个子 agent 同时改同一个仓库，git 状态互相踩。写操作要么串行，要么按文件/目录切开边界。
- **递归限制：** 子 agent 默认不能再 spawn 子 agent，别设计层层分叉的任务树，两层的活拆成两次顺序调度。
- **不能向用户提问：** 子 agent 跑完即回，任务有歧义它会自己猜。歧义要在简报阶段消掉。
- **session 残留：** 大量短命子 agent 会在磁盘留下 session 记录，定期清理，别让排查日志时翻到一堆僵尸记录。

## 可复用建议

- 固化一个四段式简报模板：**目标 / 上下文与路径 / 约束 / 输出格式与上限**，写全再 spawn。
- 给"重活"单独配一个 agent profile，工具白名单收窄到只读类，降低子 agent 误写主工作区的风险。
- 把"该不该 spawn"写成团队规则：预计消耗超过主会话单轮预算的任务，一律派出去。

## 总结

session 隔离的价值不在"多开几个 agent"，而在控制主会话的上下文增速。子 agent 本质是一个自包含、一次性、结果压缩回传的执行单元——把它当带隔离外壳的外包工人用：简报写全、结果限长、写操作管好并发，主会话就能长时间保持干净，agent 的长程表现也稳定得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/0ca94046ccab520d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/1ecfd48b62dd150b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/36007f32dc279730.png)

