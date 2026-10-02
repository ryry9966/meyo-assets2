---
title: 单实例双通道：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 40106
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

我的 Agent 最初只挂在 Telegram 上，个人使用为主。后来团队讨论搬到了 Discord，大家希望同一个 Agent 能查同一份知识库、共享同一段上下文。第一反应是再起一个实例，跑了一周就放弃了：两边记忆不同步、两份配置要分别维护、模型调用成本翻倍。于是改成了单实例、多通道的架构，跑了一段时间后沉淀一些经验。

## 问题在哪

跨平台路由本质上是四件事：

1. **会话路由**：不同平台、不同群/频道的消息，进同一个 session 还是各自隔离；
2. **身份归一**：Telegram 的 user id 和 Discord 的 user id，Agent 眼里算不算同一个人；
3. **格式兼容**：两个平台的 Markdown 方言、长度限制、渲染规则都不一样；
4. **限流差异**：出站频率策略必须统一管控，否则广播式回复很容易触发平台限流。

## 做法

**1. 单实例双通道。** 在 OpenClaw 配置里同时启用 `telegram` 和 `discord` 两个 channel，token 各自配，Agent、模型、工具配置共用一份。Bot 进程只有一份，通道只是 I/O 层。

**2. 会话按「通道 + 会话 ID」隔离。** session key 采用 `channel:chat_id` 的形式（如 `discord:channel:xxx`、`telegram:group:xxx`），默认互不串台。需要跨平台共享的信息走 workspace/memory，而不是合并 session。这样排障时一眼就能看出某条消息来自哪条通道。

**3. 身份与权限分层。** 平台原生 user id 各自保留，另建一层 display name 映射，让 Agent 在对话里能对上"是谁"。管理类命令按通道白名单收口：重启、清理只在 Telegram 私聊生效，Discord 只开放查询类能力，降低误操作面。

**4. 出站统一适配。** 让 Agent 输出标准 Markdown，适配层负责降级：Discord 超 2000 字符自动分段；Telegram 用 HTML parse mode 发送。所有出站消息进队列串行发送，防止双平台同时长回复时撞限流。

## 踩坑点

- **MarkdownV2 转义是最费时间的坑。** Agent 输出里的下划线、方括号经常导致 Telegram 消息直接发送失败，换 HTML parse mode 后问题基本消失，强烈建议别硬扛转义。
- **代码块嵌套渲染错乱。** Agent 输出含 ` ``` ` 时会和 Discord 适配层外包的代码块冲突，适配层要先剥内层再重新包裹。
- **工具调用并发冲突。** 两个平台同时触发写同一文件的操作会互相覆盖，加了个简单的任务队列按序执行工具调用。
- **触发词策略必须统一。** Telegram 用关键词 + 私聊直通，Discord 用 mention 触发。两边策略不一致时，出现过"该应答的平台没回、不该回的平台回了"的尴尬场面。
- **网络出口要提前规划。** Discord gateway 走长连接，Telegram webhook 需要公网回调地址，内网部署时这两条链路的要求不一样。

## 可复用的建议

- **通道薄、核心厚**：平台差异全部封在 adapter，Agent 的 system prompt 里不出现任何平台相关逻辑；
- **出站消息抽象为结构化对象**（text / code / attachment），由各通道自行渲染；
- **日志统一带 channel 前缀**，跨平台排障效率高很多；
- **新通道先灰度**：只读、小范围跑稳后再放开命令权限。

## 总结

单实例多通道的核心价值是记忆与上下文的统一，代价是必须认真处理身份映射、格式降级、限流管控这三件事。把它们压进适配层之后，后续再接 Slack 之类的通道，边际成本会低得多——本质上通道扩展变成了纯粹的 I/O 问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/a833a3bf78d90b67.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/b23e1d84a4ae08ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/8d3d35dbe51c1a0e.png)

