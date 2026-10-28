---
title: 跨平台消息路由实战：一个 Agent 同时服务 Telegram 和 Discord
feedId: 42822
source: 综合讨论
publishedAt: 2026-10-28
---

### 背景

我们的 Agent 在 Telegram 上跑了小半年，最近社区日常聊天搬到了 Discord。摆在面前两条路：再起一个 Agent 实例，或者让现有 Agent 同时服务两边。前者意味着配置、插件、记忆各一份，人设和上下文还会分叉；后者只需要一层消息路由。我们选了后者，把过程和坑记下来。

### 问题

两个平台的差异比想象中大：

- **身份体系**：Telegram 是 user_id / chat_id，Discord 是 guild / channel / user 三层结构；
- **消息格式**：MarkdownV2 和 Discord markdown 不兼容，转义规则各管各的；
- **长度限制**：4096 vs 2000 字符；
- **接收方式**：一边 long polling，一边 gateway websocket；
- **媒体与限流**：下载、转发、速率全都不一样。

最朴素的写法是两个适配器各自直连 Agent，结果就是会话分裂、上下文写入竞态、格式报错三连。路由层要解决的正是这些。

### 做法

1. **统一消息抽象**。适配器只做“翻译”，把两边消息归一到同一个结构：

```json
{
  "platform": "telegram | discord",
  "session_key": "u:10086",
  "msg_id": "telegram:12345",
  "text": "...",
  "media": ["/cache/xxx.jpg"]
}
```

2. **会话键归一**。我们用全局用户身份做 session_key（SQLite 存映射表），记忆跨平台共享；如果需要隔离，改成 `platform:user_id` 即可。
3. **按会话串行**。路由层给每个 session_key 挂一个 FIFO 队列，用户从两边同时发消息也不会写坏上下文。
4. **回复渲染层分离**。Agent 输出统一为纯 markdown，由各平台 renderer 负责转义、分片、parse_mode 选择。
5. **媒体走本地缓存**。入站媒体落到磁盘，Agent 拿到路径；出站由适配器负责重新上传。
6. **先单进程**。两个适配器跑在同一事件循环里完全够用；等队列有积压再拆 Redis。

### 踩坑点

- Telegram 群组默认开启 privacy mode，收不到普通消息，要么关闭 privacy，要么约定 /cmd 触发；
- Discord 必须显式开启 MESSAGE CONTENT intent，否则收到空文本——这个坑排查了一晚上；
- 去重 key 不能只用 message_id，要用 `platform:id` 复合键，webhook 重试会造成重复回复；
- Telegram 的 MarkdownV2 转义极碎，下划线、星号都能触发 400，直接用 HTML parse_mode 省心得多；
- 长回复分片按段落切，别把代码块腰斩；
- 广播类回复记得按平台限速，Telegram 30 msg/s，Discord 的限制更碎。

### 可复用建议

- 适配器零业务逻辑，所有判断放路由层，新平台接入只需新写一个适配器；
- 接入顺序固定为：收发打通 → 会话接入 → 媒体处理，别贪快一步到位；
- 监控盯三个数：入站到出站的延迟、各平台失败率、队列积压长度；
- token 分文件管理并进 .gitignore，Discord 的 bot token 泄一次就够喝一壶。

### 总结

路由层最终不到 400 行，换来的是 Agent 核心完全感知不到平台的存在。真正的成本不在架构，而在格式兼容和限流这些边角——渲染层设计好了，一半的排障时间都能省下来。下一个平台接入时，理论上只差一个适配器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/6b6c08c38de36430.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/ad7a5de57e6ee81f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/2cd60fc758f26105.png)

