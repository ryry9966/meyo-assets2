---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 39285
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

Agent 要产生实际价值，光会聊天不够，得能"动手"：查库存、开工单、发通知。OpenClaw 对接外部服务有三条常用路径：原生 tool calling、MCP server、插件/回调。路径不同，但本质是同一件事——**一次握手**：认证方式、参数契约、错误语义、超时与重试策略，双方都要先谈拢。很多"模型不听话"的问题，根因其实是握手没做完。

## 问题

三种典型症状：

1. **传参靠猜**：tool 描述只有一句"创建订单"，模型不知道字段单位、枚举值，只能瞎填。
2. **错误不会处理**：服务返回 401 或 429，agent 拿到原始报错后原样重试，轻则烧 token，重则死循环。
3. **慢接口拖全场**：一个 20 秒的同步接口把整个任务卡住，上游还以为 agent 挂了。

## 做法

按五步走：

**1. 先划边界。** 列出 agent 真正需要的动作清单（查询、创建、取消），不要把整套 REST API 一股脑暴露。tools 超过 20 个，选择准确率会明显下降。

**2. 用 MCP 包一层薄封装。** 一个 tool 对应一个明确动作，描述写清参数含义、单位、必填项和返回结构：

```python
@mcp.tool()
async def create_ticket(title: str, priority: Literal["low","medium","high"],
                        idempotency_key: str):
    """创建工单。priority 指业务紧急度。重复调用请复用同一 idempotency_key。"""
    r = await client.post(f"{API}/tickets", headers=auth(),
                          json={"title": title, "priority": priority},
                          params={"key": idempotency_key})
    if r.status_code in (401, 403):
        return err("凭证无效或过期，请检查 token，不要重试", retryable=False)
    if r.status_code == 429:
        return err("触发限流，请稍等后重试", retryable=True)
    r.raise_for_status()
    return ok(r.json())
```

**3. 错误翻译成人话。** 把 HTTP 状态码映射成带决策建议的结构化错误：401 提示"检查凭证、不要重试"，429 提示"稍后重试"。模型一次就能做对动作。

**4. 凭证最小化。** secrets 从环境变量或密钥服务注入，按 workspace 隔离，scope 只开用到的接口，绝不提交进仓库。

**5. 长任务异步化。** p95 超过 5 秒的接口，改成"立即返回 task_id + 提供查询工具"，或走 webhook 回调，别让 agent 干等。

## 踩坑点

- **tool 描述省字省出 bug**：字段含义、默认值、边界条件必须在描述里说清，这是模型唯一的"说明书"。
- **写操作没带幂等键**：重试一次就多建一条资源，对账时才发现。
- **原始异常堆栈直接丢回上下文**：既占 token 又误导模型。
- **只测 happy path**：上线前务必造一批 401 / 429 / 500 / 超时场景，观察 agent 行为是否符合预期。

## 可复用建议

- 每个 tool 用一句话说清"做什么、什么时候用"，说不清就拆或合。
- 错误返回固定三件套：`code` / `human_hint` / `retryable`。
- 所有写操作默认带幂等键。
- secrets 不落盘、不进日志。
- 维护一组固定回归 prompt，改动 tool 定义后跑一遍再上线。

## 总结

对接外部服务，"接上"只是第一步。真正决定 agent 稳定性的，是握手的契约质量：清晰的 tool 描述、会说人话的错误、可控的超时和幂等。把这四件事做扎实，模型的表现往往会比预期好一截——多数时候不是模型不行，是接口没教会它。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/1a8091fe03979a45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/51be50a32589cacd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/e1c4ca53282f937c.png)

