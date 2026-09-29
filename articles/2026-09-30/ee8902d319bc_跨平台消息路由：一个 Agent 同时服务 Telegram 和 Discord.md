---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 39708
source: 综合讨论
publishedAt: 2026-09-30
---

# 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord

## 背景

我们社区同时维护一个 Telegram 群和一个 Discord 服务器，早期各挂了一个 bot，prompt 和工具集各自独立演进。三个月后两边行为明显漂移：Discord 侧接了检索工具，Telegram 侧没有；同一个人在两边提问会得到不同风格的回答。这次重构的目标很朴素：Agent 内核只保留一份，Telegram 和 Discord 降级为两个"适配器"。

## 问题

两个平台的消息模型差异比想象中大：

- **会话结构**：Telegram 靠 reply_to 和话题群划界，Discord 有 thread 和 slash command，会话边界语义不同。
- **身份体系**：user id 不互通，同一个真人在两边是两个身份。
- **输出限制**：Discord 单条 2000 字符，Telegram 4096；Markdown 方言也不一样，Telegram 对 `_ * [` 需要转义。
- **限流**：两边都有 429，但配额模型和退避策略不同。

所以问题本质不是"接两个 API"，而是：消息规范化、会话路由、输出格式化这三层怎么切。

## 做法

**1. 适配器只做收发。** 每个 adapter 把平台事件转成统一的 MessageEvent：`platform / chat_id / thread_id / user_id / text / reply_to / attachments`。平台特有字段放进 extras，不进 prompt。

**2. 会话路由用 `platform:chat:thread` 做 key。** Discord 侧必须带上 thread_id，否则所有 thread 会话会串进同一个 session——这是最初踩的最大坑。

**3. Agent 内核与 MCP 工具全局共享**，权限用 per-platform allowlist 控制：

```yaml
channels:
  telegram:
    allow_groups: ["-100xxxx"]
  discord:
    allow_guilds: ["xxxx"]
    allow_threads: true
router:
  session_key: "{platform}:{chat}:{thread}"
```

**4. 输出层按平台格式化。** 同一份回答先走 formatter（转义/截断/分段），再由 adapter 发送。分段统一按段落切，不做硬字符截断。

## 踩坑点

- **编辑消息**：Telegram 用户改消息会再触发一次事件，等于重复提问。按 message_id 去重，并显式约定"编辑视为新提问"还是"忽略"，二选一。
- **slash command 的 3 秒交互超时**：Agent 跑工具经常超出这个窗口，后来干脆不用 interaction 回复，改成普通消息 + typing 状态。
- **长任务静默**：工具调用超过 20 秒用户会以为挂了。先发一条"正在检索……"占位消息，成本极低，体验提升明显。
- **附件格式**：Telegram 给 file_id，Discord 给 CDN URL，统一转成可下载 URL 再喂给 Agent；注意 Telegram bot 端有文件大小限制。
- **限流退避**：每个 channel 独立队列。Discord 的 429 涉及全局桶，重试时不要把另一个平台也堵死。

## 可复用建议

- 适配器保持"薄"，业务逻辑全部进 Agent 层；规范化 schema 加版本号，方便回放。
- 写一个离线回放脚本，把两边真实事件导出成统一 JSON，回归测试不再依赖真实账号。
- 没有跨平台上下文续聊需求，就不要急着做身份合并（同一个人 TG/Discord 归一），映射的维护成本远高于收益。
- 每个通道单独打点：消息量、首响应延迟、429 次数。出问题先看这三条曲线。

## 总结

重构完成后，接入一个新平台（比如 Slack）只需要写几百行的 adapter，Agent 和工具零改动。核心经验一句话：**把消息进出的脏活和智能分开**，路由 key 设计对了，剩下的都是体力活。大家在多渠道接入上如果有别的路由方案，欢迎评论区交流。

---

