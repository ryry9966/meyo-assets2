---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 42988
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

Agent 的价值不在会聊，而在能"动手"。在 OpenClaw 里，Agent 触达外部服务通常有三条路：让模型直接跑 shell/curl、接入 MCP server、写本地插件把服务封装成 tool。第一条适合探索期验证可行性，真正要长期跑的对接，基本都落在后两条上。

## 问题

很多人把"能调通"当成"对接完成"。demo 里一次调用成功，放到日常任务里问题就来了：鉴权过期没人管、接口偶发超时、Agent 重试导致重复写操作、返回体太大把上下文撑爆。这些才是对接的真实成本，得在设计阶段就想清楚。

## 做法：以一个 MCP server 为例

假设要把内部的工单 REST 服务交给 Agent 使用，我的步骤是：

1. **按动作切 tool，不按接口切。** 不要把整个服务包成一个万能 tool，而是拆成 `create_ticket` / `query_ticket` / `close_ticket`。tool 粒度对应 Agent 的"意图"，模型选起来才不容易错。
2. **鉴权下沉到 server 侧。** API key 放环境变量，tool 参数里永远不出现凭证字段，Agent 只传业务参数。
3. **把约束写进 description。** 必填项、取值范围、格式示例都写进 schema 描述。模型是靠描述理解工具的，这里省一句话，线上就多一类报错。
4. **错误返回要"可行动"。** 返回结构化错误加下一步建议，而不是抛堆栈。模型看到 "timeout，可先 query 确认是否已创建" 是能自己决策的。
5. **返回体做裁剪。** 只回字段白名单，列表强制分页，别让一次调用吃掉半个上下文窗口。

骨架大致长这样：

```python
@mcp.tool()
def create_ticket(title: str, priority: str = "normal") -> dict:
    """创建工单。priority 仅接受 low / normal / high。"""
    try:
        resp = http.post(f"{BASE}/tickets", json={"title": title, "priority": priority},
                         headers=auth_header(), timeout=8)  # 凭证从环境变量读
        resp.raise_for_status()
        d = resp.json()
        return {"id": d["id"], "status": d["status"]}
    except requests.Timeout:
        return {"error": "timeout",
                "hint": "服务 8s 未响应，可稍后用 query_ticket 确认是否已创建"}
```

## 踩坑点

- **写操作 + 自动重试 = 重复下单。** 没有幂等键时，一次超时重试就是两笔数据。要么服务端支持 idempotency key，要么写操作不做自动重试，交给确认环节。
- **超时要对齐。** tool 侧超时设得比上游网关还长，等于没设。宁可 fast fail，把决策权还给 Agent。
- **schema 漂移。** 上游改了字段，tool 不报错但行为变怪。对关键响应做最小校验（必填字段存在、类型正确），早失败早暴露。
- **别在 tool 里堆业务逻辑。** tool 只做"翻译"，判断留给 Agent 或独立工作流。混在一起，出问题时说不清是模型的错还是代码的错。

## 可复用建议

- 一个外部服务对应一套薄 tool：鉴权、超时、重试策略收敛在同一处，改起来只动一处。
- 读操作放开，写操作默认 dry_run 或 require_confirm。
- 配一个 `ping`/healthcheck tool，Agent 拿到新环境能先自检连通性，省掉大量来回试探。
- 错误信息模板化：发生了什么 + 为什么 + 下一步建议，三段式。

## 总结

对接外部服务，本质是给 Agent 一份**有边界的权限**，再加一份**能读懂的说明书**。tool 做得越薄、约束写得越显式、错误越可行动，Agent 的行为就越可预测。OpenClaw 侧多花的每一分收敛功夫，最后都会变成线上少熬的夜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/301331209c0e790d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/e7c8da9e35e5e0ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/eda75f4a6286162b.png)

