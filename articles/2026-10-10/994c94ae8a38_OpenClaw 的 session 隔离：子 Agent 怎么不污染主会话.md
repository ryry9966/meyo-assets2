---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41087
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

跑多 Agent 自动化流水线时，最常见的问题不是子 Agent 干不了活，而是它干完活之后——几十条工具调用、中间推理、报错堆栈全部回流进主会话。主上下文迅速膨胀，主 Agent 开始“失忆”、重复决策、甚至把子 Agent 的脏数据当成事实。我们在一条持续运行的抓取+代码检索流水线上踩了两周坑，最后靠 OpenClaw 的 session 隔离机制稳定下来，这里把做法整理出来。

## 问题：污染的四种形态

1. **上下文串台**：子 Agent 的中间推理被追加进主会话历史，主 Agent 的注意力被过程性内容稀释；
2. **工具记录回流**：子 Agent 每次调用的原始输出（动辄几千 token）全部进入主上下文；
3. **记忆写入泄漏**：子 Agent 往共享 memory store 写入低质量摘要，后续被主 Agent 检索到；
4. **配置继承**：子 Agent 默认拿到主会话的环境变量和全部 MCP 连接，权限面过大。

## 做法：五步隔离

```yaml
subagent:
  session: ephemeral          # 一次性独立会话，不复用主 session_id
  inherit_context: summary    # none / summary / selective
  tools:
    allow: [web.fetch, code.search]   # 最小工具集
  mcp:
    inherit: false            # 不继承主会话的 MCP 连接
  ttl: 15m                    # 到期即销毁
  return:
    mode: structured          # 只回传结构化最终结果
    max_tokens: 800
```

1. **独立 session_id**：子 Agent 用一次性 id 创建，结束即销毁，杜绝历史残留；
2. **context 策略收窄**：`summary` 模式下只传摘要，长任务配合 `max_tokens` 限制回传体积；
3. **工具与 MCP 白名单**：按任务下发，而不是继承全量；
4. **结构化回传协议**：子 Agent 只返回约定格式的结果 JSON，过程性内容留在子会话里；
5. **TTL + 异常捕获**：超时销毁，错误不透传堆栈，只返回错误码和一句话原因。

## 踩坑点

- `inherit_context: summary` 不等于零污染——摘要本身也占主会话预算，一定要限 `max_tokens`；
- **共享文件系统是隐性通道**：子 Agent 写了中间文件，主 Agent 读到了“看似合理”的脏数据，这种污染最隐蔽；
- 子 Agent 异常未捕获时，默认行为是把完整堆栈回流，务必在 return 层拦截；
- 图省事复用 session_id，上一轮的历史会残留，这是我们发现频率最高的事故；
- 并发多个子 Agent 共享限流和密钥，一个把配额打满，全体超时。

## 可复用建议

- 把子 Agent 当**函数**对待：输入参数化、输出结构化，不读也不改全局状态；
- 主会话只保留决策上下文，一切过程性上下文下沉到子会话；
- 给子 Agent 的 memory 写入加独立 namespace，定期审计；
- 灰度阶段先开 session 级日志确认隔离生效，再逐步收窄权限。

## 总结

session 隔离的本质是一句话：**过程留在子会话，结论回到主会话**。配置本身不复杂，难在守住三条纪律——一次性 session id、最小工具集、结构化回传。做到这三条，主会话在长任务下的稳定性会有肉眼可见的改善。欢迎在评论区交流你们的隔离策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/45adf0d3acd6e422.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/ebc6b2e1dc4972ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/09333128f205c930.png)

