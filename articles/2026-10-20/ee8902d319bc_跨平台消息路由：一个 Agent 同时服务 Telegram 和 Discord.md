---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 42039
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

我们组有个基于 OpenClaw 的运维 Agent，最初只挂在 Telegram 群里值班，查日志、跑巡检都靠它。后来 Discord 那边也要用。第一反应是复制一套部署过去，但很快发现：两份 prompt 要同步维护、会话记忆割裂、改一个工具调用两边都得动。于是把消息接入抽成「适配层 + 路由层」，让 Agent 核心完全不感知平台差异。这篇帖子记录这次改造的思路和踩坑。

## 问题

把两个平台直接塞给同一个 Agent，会撞上三类问题：

1. **消息模型不一致**：Telegram 有 chat_id、回复引用、编辑消息；Discord 有频道、thread、slash command，还有 interaction 必须在 3 秒内 ack 的硬约束。
2. **会话归属混乱**：session key 按什么粒度定？按 chat、按用户还是按 thread？
3. **输出格式冲突**：两边 Markdown 方言不同，长度上限也不同（Telegram 4096，Discord 2000）。

## 做法

结构上分四层，Agent 只看到中间一层：

1. **统一信封**：定义内部消息结构，字段大致是 `platform / user_id / channel_id / text / reply_to / attachments / raw`。两个适配器只负责把各自 payload 翻译成信封，不做任何业务判断。
2. **路由与会话键**：路由层根据信封生成 session key。我们用 `platform:channel_id`（群聊场景），Discord thread 场景追加 `:thread_id`。同一个 key 进同一个 Agent 会话，天然实现隔离。
3. **Agent 核心**：OpenClaw 侧的 prompt、MCP 工具、记忆只有一份。prompt 里禁止出现任何平台相关措辞，输出约定为「结构化意图 + 正文」。
4. **渲染回程**：出站方向各有一个 renderer——Telegram 用 HTML parse mode，Discord 用原生 Markdown；超长消息按块切分。出站统一过一个带限速的队列（Telegram 全局约 30 msg/s，Discord 频道级 5 条/5 秒）。

## 踩坑点

- **Discord 3 秒超时**：Agent 跑一次工具调用经常超过 3 秒，必须先 ack/defer，再用 webhook followup 或编辑原消息回复。这段逻辑要关死在 Discord 适配器里，不能泄漏到路由层。
- **Telegram MarkdownV2 转义地狱**：字符集太怪，最后换 HTML parse mode 才稳定。
- **切消息不能按字数硬切**：会把代码块切烂。先按块（段落/代码块）聚合，块超限再降级切。
- **附件不可跨平台**：Telegram 的 file_id 只对当前 bot 有效，Discord CDN 链接会过期。信封里带附件的，务必在入站时就转存。
- **身份是两个身份**：同一个人在两边就是两个 session。跨平台 identity merge 我们还没做，目前靠人工在会话里互相知会，列在下一步。
- **编辑/撤回不回放**：用户编辑了消息，Agent 是否重新执行？我们选择不回放，只在日志里标记，避免重复跑工具。

## 可复用建议

- 先定信封 schema，再写适配器，顺序不要反。
- 平台怪癖全部关进 adapter/renderer，Agent prompt 保持平台无关——这是这次改造收益最大的一条。
- 用真实的原始 payload 攒一批 fixture 做回放测试，比端到端手测省一半时间。
- 每条日志都带 `platform` 标签，排查「哪边出的问题」效率完全不同。

## 总结

跨平台的本质不是「多接一个 bot」，而是把**消息进出**和 **Agent 决策**解耦。信封 + 路由 + 渲染这套结构两天搭完，之后接第三个平台（比如 Slack）基本只是再写一个 adapter。代码不复杂，但每条边界都要想清楚：会话键怎么定、超时怎么 ack、消息怎么切。想清楚了，后面的平台都是体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/fa9bf271697bb44f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/bf9b0679c8513db2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/ab098aa7fa953acb.png)

