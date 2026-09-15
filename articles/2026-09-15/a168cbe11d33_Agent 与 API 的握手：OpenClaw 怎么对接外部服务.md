---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 37673
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

Agent 本身只会"想"，不会"做"。OpenClaw 的价值在于通过 MCP / 插件机制把外部服务变成 Agent 可调用的工具。社区里常见两类写法：一类把 REST API 逐条全量包成工具，一类干脆把调用细节写死在 prompt 里。两种方案在 demo 里都能跑，到了真实业务里都会翻车。

## 问题

对接外部服务，真正难的从来不是"调通一次请求"，而是四件事：

- **工具粒度**：一个 endpoint 一个工具，还是按业务动作聚合？
- **认证**：API key 放哪，怎么避免进日志和上下文？
- **错误语义**：429、5xx、超时，Agent 该重试还是该放弃并如实汇报？
- **返回体积**：几十 KB 的原始 JSON 塞进上下文，token 直接爆炸。

## 做法

以对接一个工单系统为例，我最终的收敛路径是：

1. 用 MCP 写一个**薄封装 server**，只暴露 4 个语义化工具：`create_ticket`、`query_ticket`、`add_comment`、`close_ticket`，而不是十几个裸接口。
2. schema 用精确类型 + enum + 明确 description，参数约束越强，模型乱填概率越低：

```json
{
  "name": "create_ticket",
  "parameters": {
    "title":    { "type": "string", "maxLength": 120 },
    "priority": { "type": "string", "enum": ["low", "mid", "high"] },
    "dry_run":  { "type": "boolean", "default": true }
  }
}
```

3. 认证走环境变量，server 启动时读取，不落盘、不回显。
4. 错误统一映射成结构体 `{ok, error_code, message, retryable}`，是否重试由 OpenClaw 侧的策略决定，不留给模型"自由判断"。
5. 返回值做裁剪：只回业务字段，长文本截断并附一个 id，需要时 Agent 再用 `fetch_detail` 二次查询。

## 踩坑点

- **工具数太多**：一个 endpoint 一个工具，模型面对 30 个工具时选择准确率明显下降。按业务动作聚合，工具数控制在个位数到十几个。
- **重试没有幂等键**：网络超时后盲目重试，结果建了两张一样的单。所有写操作必须带 idempotency key。
- **超时不对齐**：MCP server 内部超时大于 Agent 侧超时，会出现"工具还在跑、Agent 已判失败"的幽灵状态，两层超时要显式配置且外层大于内层。
- **把原始异常栈返回给模型**：模型会拿 stack trace"脑补"，必须给清洗后的错误语义。
- **没有 dry-run**：联调期直接打真实环境属于事故预备役，写操作默认带 `dry_run`。

## 可复用建议

- 工具边界按业务动作划，不按 API 结构划；
- 写操作一律幂等，读操作考虑缓存与限流；
- 错误结构化 + retryable 标记，重试策略收敛在框架侧；
- 建一套固定的调用用例集，每次改 prompt、换模型版本就跑一遍对拍，防止工具调用静默劣化。

## 总结

对接外部服务的本质，是给 Agent 一份清晰的**服务契约**：明确的工具边界、结构化的错误语义、可控的返回体积。OpenClaw + MCP 只是通道，契约的质量才决定 Agent 的可靠度。先把契约写好，再谈自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/a614b94ef766f2b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/fde7d24ccfbc84f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/0be0ef40d7506790.png)

