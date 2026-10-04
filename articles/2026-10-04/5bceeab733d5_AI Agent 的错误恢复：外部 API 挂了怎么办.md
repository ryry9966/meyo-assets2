---
title: AI Agent 的错误恢复：外部 API 挂了怎么办
feedId: 40445
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

Agent 落地到生产环境后，外部 API 调用就不再是"偶尔出错"，而是常态。一个典型的自动化任务里，Agent 可能通过 MCP 工具、插件连续调用十几个甚至几十个外部接口：搜索、数据库、第三方 SaaS。网络抖动、限流、上游发版，任何一环都可能挂。区别只在于：挂掉之后，任务是整体报废，还是能继续推进。

## 问题

实际跑下来，常见的失败处理反模式有三类：

1. **模型自由重试**。把"失败了就重试"写进提示词让 LLM 自己决定，结果是紧密循环烧 token，甚至在没有成功时编造成功。
2. **盲目重试**。不分错误类型，401 和 503 一样重试三次，浪费配额还拖慢任务。
3. **错误被吞掉**。插件捕获异常后返回空字符串，模型以为这步成功了，基于空数据继续推理，输出看似完整实则错误的结果。

非幂等接口还有第四类：重试导致重复下单、重复发消息。

## 做法

核心原则一句话：**机械重试交给确定性代码，路径决策才交给模型。**

1. **错误分类**。封装层先判断：超时、429、5xx 属于可重试；400、401、403、404 属于不可重试，直接上报。
2. **指数退避 + 抖动**，设尝试上限和总时间预算，避免单次调用卡死整个任务。
3. **幂等保护**。写操作带幂等键，或先查后写，重试才安全。
4. **结构化错误回传**。不要把原始堆栈扔给模型，返回 `{ok, error_code, retriable, hint}` 这样的结构，模型据此决定换工具、降级还是询问用户。
5. **降级路径**。关键工具提前准备 Plan B：本地缓存（标注数据时间）、备用服务商、或人工确认点。连续失败达到阈值就熔断，冷却期内不再请求。

一个最小可用的封装骨架（示意，异常类需自行定义）：

```python
def call(fn, *args, max_tries=3, budget_s=30):
    deadline = time.monotonic() + budget_s
    for i in range(max_tries):
        try:
            return {"ok": True, "data": fn(*args)}
        except Retriable as e:
            if i == max_tries - 1 or time.monotonic() >= deadline:
                return {"ok": False, "error": e.code, "retriable": True,
                        "hint": "switch tool, use cache, or escalate"}
            time.sleep(min(2 ** i, 8) + random.random())
        except Fatal as e:
            return {"ok": False, "error": e.code, "retriable": False,
                    "hint": "do not retry, check credentials or params"}
```

## 踩坑点

- **退避不加抖动**：上游恢复瞬间，所有客户端同时重放，二次打挂。加随机抖动。
- **降级缓存不标新鲜度**：模型把三小时前的数据当实时数据用。hint 里明确写数据时间。
- **熔断太激进**：一两次失败就熔断，把抖动当宕机。阈值要结合错误类型，429 连续触发和偶发超时不是一个量级。
- **长任务没有断点**：第 27 步挂了从第 1 步重跑。中间状态落盘，恢复等于续跑。
- **只在成功路径上测试**：上线后第一次遇到故障就是真实故障。开发环境做故障注入，把失败路径完整跑一遍。

## 可复用建议

- 所有外部调用走统一封装：超时、分类重试、结构化返回，一处实现，所有插件复用。
- 工具的 description 里写清错误语义和降级选项，让模型在规划阶段就知道有 Plan B，而不是出错后现场发挥。
- 错误信息是写给"会读它的模型"看的：简洁、可操作、附带下一步建议。
- 长任务落 checkpoint，恢复策略是"从断点继续"，不是"从头再来"。

## 总结

错误恢复是系统设计问题，不是提示词问题。分层是关键：确定性代码负责退避、重试、熔断这些机械动作；Agent 负责读完结构化错误后做路径决策——换路、降级、求助。两层职责清晰，任务才能从"一次失败全盘报废"变成"可恢复的工程流程"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/25937f5ce1dc8fa4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/7ac63f68ecbbd850.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/cd565059062f1e1e.png)

