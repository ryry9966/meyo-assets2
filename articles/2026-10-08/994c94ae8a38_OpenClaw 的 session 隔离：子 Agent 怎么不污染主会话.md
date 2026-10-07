---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 40850
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

在 OpenClaw 里，主会话承载着用户侧的长期上下文：历史消息、工具调用记录、memory 写入。当我们派子 Agent 去跑搜索、代码执行、批量处理这类"脏活"时，最常见的问题不是子 Agent 干不好活，而是它把过程垃圾全倒进了主会话——几十条中间 tool result、临时文件路径、失败重试日志。结果就是主会话 context 被撑爆，主 Agent 的判断被无关细节干扰，token 成本也跟着涨。

## 问题

典型症状有三个：

1. 主会话里出现大量子 Agent 的中间输出，单轮 token 消耗翻倍；
2. 子 Agent 写 memory / 临时文件时和主会话互相覆盖；
3. 子 Agent 的失败重试记录留在主会话历史里，后续轮次模型被误导，反复走弯路。

## 做法

OpenClaw 的隔离机制核心是三层，示意配置如下（字段名以你本地版本为准）：

```yaml
agent:
  name: researcher
  session:
    type: ephemeral            # 独立 session，不落主会话
    inherit: summary_only      # 只继承任务摘要，不继承全量历史
    max_turns: 12              # 硬性轮数上限
    on_finish: return_summary  # 结束时只回传压缩结果
```

**第一步：session 分层。** spawn 子 Agent 时显式声明 `ephemeral`，子 Agent 拿到独立 session id，生命周期跟随任务，结束即回收。这是隔离的骨架。

**第二步：回传契约。** 子 Agent 结束时只回传一个结构化 summary（结论 + 关键引用 + 状态码），中间 tool 调用全部留在自己的 session 里。主会话看到的只是"任务完成，结果如下"一条消息。

**第三步：写入命名空间。** memory 和 workspace 文件给子 Agent 加前缀隔离，比如 `ns: task/researcher-<run_id>`，避免和主会话的 memory 互相踩。

## 踩坑点

- **inherit 全量历史**：最贵的坑。默认继承会让子 Agent 带着主会话全部上下文跑，token 成本直接乘以子 Agent 数量。改成 `summary_only`。
- **异步回调写回主会话**：子 Agent 异步执行时，回调必须走 return 通道，别直接 append 主会话 history，否则隔离形同虚设。
- **临时文件共享目录**：子 Agent 往同一个 `/tmp` 写同名文件会互相覆盖，用 run_id 做目录隔离。
- **不设 max_turns**：失控的子 Agent 会把 ephemeral session 自己烧穿，设个上限兜底。
- **摘要压缩过狠**：`return_summary` 丢掉关键约束会让主 Agent 二次返工，把"必须保留的字段"写进任务描述里。

## 可复用建议

- 把子 Agent 当函数调用设计：明确的输入、输出 schema、超时和重试策略；
- 主会话只做编排和决策，脏活全部下沉到 ephemeral session；
- 每类子任务固定一个 prompt 模板 + 回传模板，方便回归测试和 diff；
- 定期审计主会话的 context 构成，一旦发现子 Agent 泄漏，立刻补隔离规则。

## 总结

Session 隔离的本质是：**主会话只保留决策所需的最小信息**。OpenClaw 提供了 ephemeral session、summary 回传、命名空间三件套，但默认值不一定适合你的场景——`inherit` 策略和异步回调是最容易翻车的两处。先隔离，再观察主会话 context 的构成，逐步收紧，比一次性上复杂方案更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/377ee2f8275a2ef6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/d9a20be774b318c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/203bf1af2352cb72.png)

