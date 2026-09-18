---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 38052
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

做 Agent / 自动化的人迟早会遇到这个场景：流程跑了一半，某个外部 API 突然 503。传统服务挂了会抛异常、触发告警、执行回滚，但 Agent 不太一样——它会把错误信息当作“上下文”继续推理。轻则编造一个看起来合理的答案交差，重则开始盲目重试，把配额烧光，甚至对下游造成重复写入。

## 问题在哪

外部 API 故障对 Agent 的伤害通常分三层：

1. **感知层**：LLM 拿到原始报错后可能“脑补”出修复方案，而不是如实上报失败；
2. **行为层**：Agent 自主决定重试，没有退避策略，遇到限流反而雪上加霜；
3. **状态层**：部分成功 + 部分失败，留下不一致的中间状态。

## 实践做法

**第一步：统一错误分类。** 所有工具调用走同一个封装层，把异常归为三类：

- 可重试：429、503、超时、网络抖动
- 不可重试：401、403、参数错误
- 待观察：HTTP 200 但 body 是错误语义——这类最容易被漏掉

**第二步：带抖动的指数退避。** 可重试错误统一走 retry + backoff + jitter，上限 3 次。关键是把重试逻辑放在代码层，而不是让 Agent 自己决定——LLM 对“等多久再试”没有概念。

```python
for attempt in range(MAX_RETRIES):
    try:
        return call_api(...)
    except RetryableError:
        sleep(min(base * 2**attempt, cap) + random.uniform(0, 1))
raise ToolUnavailable(...)
```

**第三步：熔断 + 降级。** 连续 N 次失败后熔断，一段时间内快速失败，不再打 API。降级路径按优先级排：备用供应商 → 本地缓存 → 明确告知用户“该能力暂不可用”。宁可暴露失败，不要让 Agent 编结果。

**第四步：提示词层面的防护。** 工具返回给模型的是结构化状态，不是原始堆栈：

```json
{"status": "unavailable", "retry_after": 60, "fallback": "cache"}
```

同时在 system prompt 里写死规则：工具返回 unavailable 时，如实说明并给替代方案，禁止虚构数据、禁止重试超过上限。

**第五步：幂等。** 所有写操作带 idempotency key，重试才是安全的。

## 踩坑点

- **让 Agent 自己重试**：它会立刻连发十次，没有退避，还烧 token；
- **把原始报错直接喂给模型**：它会煞有介事地“解释”这个错误并编造修复方案；
- **备用 API schema 不一致**：降级表面成功，数据结构变了，下游悄悄出错；
- **只看 HTTP 状态码**：200 + error body 的坑非常常见；
- **熔断阈值太敏感**：一次抖动就熔断，误伤正常流量。

## 可复用建议

1. 所有工具调用收敛到统一的执行器，错误分类、重试、熔断、日志都在这一层做；
2. 每个工具的重试策略做成配置项，而非硬编码；
3. 给关键工具加健康检查和手动 kill switch；
4. 定期演练：故意断掉某个 API，观察 Agent 的实际表现——这比写十页文档有用。

## 总结

错误恢复不是“多加一个 try/catch”，而是把失败当成一等公民来设计：分类、退避、熔断、降级、幂等，五件事缺一不可。核心原则只有一条：让 Agent 对用户诚实，让代码对故障负责。API 会挂，这很正常；挂了之后系统是可预期地降级，还是不可预期地胡说，才是工程能力的分水岭。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/b63de53e3e302a7e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/73ab013d87de9588.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9f846aba6f2891dd.png)

