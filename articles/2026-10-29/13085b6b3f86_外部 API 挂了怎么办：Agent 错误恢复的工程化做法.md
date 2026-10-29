---
title: 外部 API 挂了怎么办：Agent 错误恢复的工程化做法
feedId: 42915
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

Agent 化之后，工具调用链变长了：MCP 插件、搜索、数据库、支付、LLM 供应商本身，全是外部依赖。对外部 API 来说，故障不是异常而是常态——限流、超时、5xx、配额耗尽、网络抖动，总有一款会在生产环境撞上。一个没有错误恢复设计的 Agent，在 API 挂掉时的典型表现是三种：死循环重试烧 token、把报错原文喂给模型导致幻觉、或者干脆卡死整个流程。

## 问题拆解

实践中最常见的是三类失败模式：

1. **卡死**：没设超时，工具调用永远不返回，Agent 流程整体阻塞。
2. **重试风暴**：失败立刻重试，无退避、无上限，把限流升级成封禁。
3. **静默错误**：API 明明返回失败，但错误信息被模型"脑补"成结果，输出看起来成功、数据却是错的。

## 做法

建议给所有工具调用套一层统一封装，按这个顺序处理：

1. **错误分类**。429 / 503 / 超时 = 可重试；400 / 401 / 403 = 不可重试，直接上抛。只有可重试错误才进重试逻辑。
2. **有限重试 + 指数退避 + 抖动**。比如 3 次，间隔 1s / 2s / 4s 加随机抖动。响应带 `Retry-After` 头时优先尊重它。
3. **每次调用设超时**。HTTP 用显式 timeout，MCP 工具调用同样要包一层。宁可快速失败，不要挂起。
4. **幂等保护**。写操作带幂等键（idempotency key），重试才安全；API 本身不支持幂等的，重试前先想清楚副作用。
5. **熔断**。连续 N 次失败后短路，冷却期内不再请求，避免拖垮自己流程，也给对方恢复时间。
6. **降级路径**。读操作返回缓存的上次成功数据并打上 degraded 标记；有备选 provider 就切换；都没有，就把结构化错误交还给模型，让它改走别的计划。

伪代码大致长这样：

```python
def call_tool(fn, payload, max_retry=3):
    for attempt in range(max_retry):
        try:
            return fn(payload, timeout=10)
        except RetryableError:
            sleep(backoff(attempt) + jitter())
        except FatalError as e:
            return structured_err(e.code, retryable=False)
    return degrade_or_return_error()
```

关键点：返回给模型的错误必须**结构化**，比如 `{ok: false, code: "TIMEOUT", retryable: true, hint: "改用本地缓存或稍后重试"}`，而不是一段原始堆栈。模型拿到清晰契约才知道该怎么应对。

## 踩坑点

- 对非幂等的 POST 盲目重试，出现重复下单、重复通知。
- 把 429 当普通错误处理，不读 `Retry-After`，越重试越被封。
- 降级返回了过期数据但没打标，下游当成新鲜数据用。
- 重试预算按单次调用算，没按整个工作流算——一个流程几十个工具调用全在重试，token 成本直接翻倍。
- 报错原文直接丢给 LLM，模型进入道歉循环，或者开始编造替代数据。

## 可复用建议

- **统一错误契约**：所有插件/工具返回 `{ok, code, retryable, hint}`，Agent 侧只认这一个格式，排查问题也省心。
- **读多写少的 API 维护"最后成功值"缓存**，降级时救急。
- **熔断状态要可观测**：触发时打日志、发告警。最怕 Agent 默默降级了一整天，没人知道。
- **在系统提示里写明规则**：工具失败时优先换路径，而不是反复重试同一个工具。

## 总结

外部 API 挂掉是必然事件，错误恢复不是"加个 try/except"就完事，而是分类、退避、超时、幂等、熔断、降级这条完整链路。目标不是让 Agent 永不失败，而是失败时快速、可观测、有替代路径。落地节奏建议：先给所有工具调用套上统一错误封装（半天工作量），再逐步补缓存和熔断。这套东西做扎实了，Agent 才敢放到生产上跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/045bc67d9ff0d46e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/bb7b6903cf34d3d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/686062cdbcb620af.png)

