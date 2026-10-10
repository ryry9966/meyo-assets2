---
title: AI Agent 的错误恢复：外部 API 挂了怎么办
feedId: 41110
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

Agent 系统的可靠性下限，往往不由模型决定，而由它依赖的外部 API 决定。LLM 供应商、搜索接口、MCP 工具背后的 SaaS 服务……任何一个 5 分钟的抖动，都可能让一条跑了几小时的自动化流水线整体报废。接口故障不是异常路径，而是常态路径的一部分。

## 问题：三种典型的失败姿势

观察了不少社区里的 Agent 项目，出错方式基本收敛为三种：

1. **直接崩**：一个 timeout 异常打穿整个任务，前面几十分钟的中间状态全丢。
2. **死循环重试**：把「要不要重试」这个决策交给模型。模型在上下文里看到 error 就再来一次，遇到 429 越打越死，token 烧完任务也没成。
3. **静默编造**：工具的报错被当成普通文本塞进上下文，模型基于报错信息「推测」出一个看起来合理的答案——这是最危险的一种，用户还以为任务成功了。

## 做法：恢复逻辑放进代码层，分四层处理

核心原则一句话：**重试与降级是工程决策，不要交给模型决策。** 模型只负责拿到结构化结果后决定下一步。

工具调用包装器的骨架大致是：

```python
def call_tool(tool, args, budget_s=30):
    err = classify(args)                  # 1. 先分类
    if not err.retryable:                 #    401/400 直接失败
        return structured_error(err)
    for attempt in with_backoff(max=3):   # 2. 指数退避 + 抖动
        if circuit.is_open(tool):         # 3. 熔断检查
            return fallback(tool, args)
        try:
            return tool.run(args, timeout=budget_s)
        except RetryableError as e:
            continue
    return structured_error(last_err)     # 4. 结构化上抛给模型
```

四层各自要做的事：

1. **分类**：429/503/timeout 可重试；400/401/404 不可重试。注意 HTTP 200 但 body 里带错误码的情况——不少 SDK 会在这里骗你。
2. **退避重试**：指数退避加随机抖动，上限 2~3 次。重试发生在包装器里，不进模型上下文，模型只看到最终结果。
3. **熔断与降级**：连续 N 次失败就打开熔断器，直接走 fallback：备用 API、本地缓存、或显式失败。冷却期内不再打源接口，给对方喘息时间。
4. **结构化上抛**：重试耗尽后，给模型返回 `{"status": "failed", "reason": "...", "suggested_action": "skip|ask_user|use_cache"}`，而不是裸的异常堆栈。

## 踩坑点

- **对非幂等操作重试**。POST 下单超时后重试，可能下两单。非幂等调用要么带幂等键，要么超时后先查询状态再决定。
- **工具级超时缺失**。HTTP client 设了 30s，但工具内部有分页循环，一次调用实际跑了 10 分钟，把整个 Agent loop 卡死。超时要卡在工具这一层，不是网络层。
- **把报错原文喂给模型**。长堆栈既烧 token，又诱导模型脑补。喂结构化摘要就够。
- **降级不告知**。fallback 返回的缓存数据如果不打标，下游会把昨天的数据当今天的用。
- **没留现场**。失败任务没有 dead-letter 记录，事后无法复现。至少记下：时间、工具名、入参、错误类别、重试次数。

## 可复用建议

- 给所有工具统一套一层 executor（超时/重试/熔断/结构化错误），不要每个插件自己写一套。
- 为关键 API 预先写好 fallback 链：主 API → 备用 → 缓存 → 显式失败。
- 开发期做混沌测试：用本地代理注入 429、超时、畸形响应，看 Agent 的实际行为是否符合预期，而不是靠想象。
- 让 Agent 对用户诚实：失败就说失败，附上建议动作。可信度比「总能给出一个答案」值钱得多。

## 总结

外部 API 一定会挂，区别只在频率和时机。好的 Agent 不是遇不到故障，而是大部分故障被代码层消化掉，剩下的以结构化、可决策的形式交给模型，让它选择跳过、换路或问人。把错误恢复做成基础设施，而不是每次故障后的临时补丁——这是 Agent 从 demo 走向生产最实际的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/65d64d161a2bc03e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/0c30ed3aa87e7518.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/28af6e12c83749ec.png)

