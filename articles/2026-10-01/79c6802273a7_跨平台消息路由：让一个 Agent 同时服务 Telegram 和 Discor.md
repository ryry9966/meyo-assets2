---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 39960
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

我们的 Agent 原本只挂在 Telegram 上做日常值守。后来社区一部分用户迁到了 Discord，需求很自然：同一套提示词、同一批 MCP 工具、同一份记忆，能不能两个平台共用一个 Agent 实例，而不是维护两份配置各自漂移。

## 问题

直接把两套 bot 代码拼在一起，会立刻撞上几个矛盾：

- 消息模型不一致：Telegram 的 update 结构和 Discord 的 gateway 事件，字段与 ID 体系完全不同；
- 回复格式不一致：MarkdownV2 的转义规则和 Discord markdown 差异很大，长度上限一个 4096、一个 2000；
- 会话边界：同一个人在两个平台各聊各的，上下文要不要合并？
- 稳定性：一个平台的轮询挂了，不能拖死另一个。

## 做法

核心思路一句话：**适配器薄、内核厚**。平台差异全部挡在 adapter 层，agent core 只认统一消息模型。

1. **定义统一消息 Schema**：`session_id = platform:chat_id`，附带 sender、content、媒体引用、reply 目标。这是整个系统的契约，先于一切代码定下来。
2. **写两个薄适配器**：Telegram 用 long polling（开发期不用配公网 HTTPS），Discord 用 gateway WebSocket。适配器只做两件事：入站解析成统一 Schema；出站把 Agent 回复渲染成平台格式（HTML parse mode、超长分段、typing 指示）。
3. **中间加路由层**：按 session_id 维护每会话的出站队列，保证同一会话消息有序，并在队列层面做平台限速（Telegram 每聊天约 1 msg/s，Discord 看 bucket）。
4. **Agent core 单实例异步消费**：LLM 调用、工具执行、记忆读写全部平台无关。回复时只声明"回给哪个 session"，投递交给路由层。
5. **故障隔离**：两个适配器各跑独立的 asyncio task，任一平台异常只重启对应 task，互不牵连。

## 踩坑点

- Telegram 同开 polling 和 webhook 会重复收消息，用 update_id 去重或干脆二选一；
- Discord 必须在开发者后台打开 Message Content Intent，否则 content 一直是空的，我们排查了很久；
- MarkdownV2 转义极易翻车，直接换 HTML parse mode 省心；
- 不要默认合并跨平台会话：同名用户的上下文串台，比记忆缺失更糟。确有需要，再做显式的身份映射表；
- 媒体下载：Telegram 文件 URL 里带 bot token，千万别打进日志；
- LLM 同步调用会阻塞另一个平台的轮询循环，全链路走 async。

## 可复用建议

- 适配器保持"哑"：零业务逻辑，只做解析和渲染。以后接 Slack、飞书，本质就是再复制一个 adapter；
- 出站队列以 (platform, chat) 为粒度，顺序和限速问题一次解决；
- 用 fixtures 录制两个平台的原始 payload 做回放测试，不必真连网回归；
- 每个平台独立健康检查与熔断，降级要允许单边发生。

## 总结

这套架构里最值钱的不是"同时上两个平台"本身，而是它逼着你把平台差异压缩进两个薄适配器。Agent core 从此与平台解耦，之后多接一个平台是增量工作，而不是重构。如果你现在手里只有单平台脚本，建议先抽统一消息模型，再谈多平台——顺序反了，返工成本很高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/31e8d1a463bbed55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/53a2a46f125f726e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a98c08291eef7cda.png)

