---
title: 一核多端：让同一个 Agent 同时值守 Telegram 和 Discord
feedId: 40454
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

我们的 Agent 最初只挂在 Telegram 上服务几个人。后来团队协作迁到 Discord，第一版的偷懒做法是再起一个实例、复制一份配置。两周后问题就暴露了：两个实例的长期记忆各自生长，同一个问题在两边答案不一致，prompt 改动还要手动同步两处。于是决定收敛成「一个核心，两个适配器」。

## 问题

跨平台不是"多填一个 bot token"那么简单，真正的差异在四层：

1. **协议层**：Telegram 适合 long polling，Discord 走 gateway websocket 或 webhook，二选一；
2. **格式层**：Telegram 的 MarkdownV2 转义规则苛刻，Discord 有 2000 字符硬限制；
3. **会话层**：群聊和私聊的会话键不同，thread 语义两边也对不齐；
4. **限流层**：两边限流桶的粒度完全不同，裸发迟早被限。

## 做法

核心思路一句话：**适配器只做翻译，核心只认信封**。

**第一步，定义统一消息信封**。所有入站消息归一化成：

```json
{
  "schema": "v1",
  "platform": "telegram | discord",
  "chat_id": "...",
  "thread_id": null,
  "user_ref": "...",
  "text": "...",
  "attachments": []
}
```

出站同理：核心产出结构化回复，由适配器负责渲染成平台格式。

**第二步，写两个薄适配器**。Telegram 侧用 long polling（省掉公网暴露），Discord 侧走 gateway。每个适配器只干三件事：收消息 → 转信封 → 入队；出队 → 渲染 → 发送。

**第三步，会话键设计**。核心 Agent 的会话键取 `platform + chat_id (+ thread_id)`，保证群聊上下文不串。跨平台的用户身份映射不做自动合并，提供一条显式绑定命令手动关联，宁可麻烦一点也不要错认人。

**第四步，路由表**。YAML 配置驱动：私聊走默认 Agent；Discord 某个论坛频道走带检索工具的专用 persona；Telegram 群里 @机器人才触发。支持热加载，改路由不重启。

**第五步，出站队列**。每个平台一条独立队列 + 各自的限速器，核心只管投递，重试和退避由队列负责。

## 踩坑点

- **MarkdownV2 转义**是最大的时间黑洞。后来干脆让核心只输出纯文本加受控的代码块标记，渲染层交给成熟的转义库，千万别手写正则。
- **Discord 分块**不能按 2000 字符硬切，会把代码块截成两半。要按块级元素（段落、代码块）聚合后再切。
- **幂等去重**：两边都可能出现消息重推，用平台 message_id 做幂等键，Redis SETNX 足够。
- **附件时效**：Discord CDN 链接和 Telegram 的 file_id 都不是永久有效的，收到后立刻下载落盘，只把本地路径传给核心。
- **长耗时任务**要先发 typing 指示，否则用户以为 bot 挂了。Discord 的 typing 状态会过期，超过 10 秒要续发。

## 可复用建议

1. 信封 schema 一定带版本号，未来接 Slack、微信时只改适配器层；
2. 保持适配器"笨"，任何业务判断下沉到核心，边界越清晰越好维护；
3. 用 trace id 串联一次对话轮的入站、核心处理、出站日志，排障效率完全不一样；
4. 把真实消息脱敏后录制成信封回放集，回归测试就不依赖平台环境了。

## 总结

多平台接入的真正工作量不在"接"，而在归一化层。把协议、格式、限流这些脏活全部锁死在适配器里，核心 Agent 对平台无感知，之后每加一个渠道的边际成本就只是一个几百行的适配器。这比养两个会各自漂移的实例，维护成本低得多，也稳得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/62c9056ab228f1b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/fadd3feb56508273.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/42942d0c9b11ac6e.png)

