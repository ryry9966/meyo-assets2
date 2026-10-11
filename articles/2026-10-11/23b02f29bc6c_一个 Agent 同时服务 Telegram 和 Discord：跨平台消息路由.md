---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 41193
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

我的 Agent 最初只挂在 Telegram 上，做个人提醒和快速问答。后来团队协作迁到 Discord，第一反应是再起一个 bot——跑了两周就拆了。原因很直接：两份配置、两套工具权限、会话记忆互不相通，每次改 prompt 和工具都要在两边同步。最后收敛到现在的形态：一个 OpenClaw Agent 实例，同时挂载 Telegram 和 Discord 两个 channel adapter，中间加一层统一路由。

## 问题

表面上是"多接一个平台"，实际要解决的是三件事：

1. **身份模型不一致**：Telegram 用 `chat_id + user_id`，Discord 用 `guild/channel/user`，直接拿 user_id 当主键必然串会话。
2. **消息模型不一致**：TG 的 MarkdownV2/HTML 实体 vs Discord 的 Markdown 和 mention 格式，同一条回复两边渲染总有一边烂掉。
3. **限流节奏不一致**：TG 有全局速率加单 chat 1 msg/s 约束，Discord 是按 channel 的桶式限流，群发时总有一边先被丢。

## 做法

架构原则一句话：**边缘适配，核心统一**。

1. **规范化消息结构**。每个 adapter 入口把平台消息压平成 canonical 结构：`source / actor_id / channel_id / content / reply_to / attachments`。平台特有字段（实体、mention 数组）一律在边缘消化。
2. **会话主键用复合 ID**。我用 `platform:channel_id:user_id` 做会话键，另加一张别名表绑定同一个人的双平台身份（手动 `/bind` 确认，不做模糊匹配）。这样单平台上下文独立，跨平台长期记忆共享。
3. **格式化下沉到边缘**。Agent 核心只输出内部 Markdown，发送时由各 adapter 转成平台方言：TG 侧转 HTML parse_mode，Discord 侧转换 mention 格式并控制长度。
4. **发送走队列**。每个 channel 一个令牌桶队列，限流集中在 gateway 层处理；超长消息降级为"截断 + 附件"策略。
5. **MCP 工具只挂一份**。MCP server 挂在 Agent 核心，两个平台的会话共享同一套工具，权限控制在核心层做，不落到 channel 层。

## 踩坑点

- **Discord 长消息切分**：2000 字符硬上限，硬切会拦腰切断代码块。按块（代码块边界优先）切再按字符切，顺序不能反。
- **TG 转义地狱**：MarkdownV2 对下划线、反引号、括号的转义极其苛刻，别恋战，直接用 HTML parse_mode 稳得多。
- **mention 单向问题**：Agent 输出的 mention 要从 canonical user_id 转成 Discord snowflake，绝不能把 `<@id>` 原样发给 Telegram。转换全部封在 adapter 里。
- **附件链接会腐烂**：Discord CDN 链接是带签名的，存进记忆一两天就是死链。两个平台的附件都应该在收到时落到本地或对象存储，记忆里只存本地引用。
- **多实例 polling 冲突**：TG 的 getUpdates 多实例并发会互相踢，单实例 + 优雅退出就够，别过度设计。
- **Reaction 语义**：Discord 的 emoji 反馈很轻量，但 TG reaction 类型有限，跨平台同步语义对不齐，我干脆不做。

## 可复用建议

- 平台方言永远在边缘解决，让核心 Agent 感知平台差异是维护灾难的开始。
- 会话键设计先于一切功能，ID 定错，后面全是增量补丁。
- 限流集中在发送队列处理，业务代码不要碰速率。
- 跨平台身份绑定哪怕只做手动命令，也比自动推断可靠。
- 日志按 canonical 结构落盘，排障时一条 grep 就能还原全链路。

## 总结

这次改造 adapter 层多了几百行代码，换来的是维护对象从"两个 bot"收敛为"一个核心 + 两个薄边缘"。跨平台路由的本质是消息规范化问题：输入归一、输出特化、状态共享。如果你的 Agent 目前只在单一平台，建议先把 channel 接口抽象出来——迁移的成本远低于日后重构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/1e2c17ccfc177b61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/c273901f1f6a28f4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/9bdaa81ae60b6b4a.png)

