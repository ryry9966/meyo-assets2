---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 42950
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

社区成员一半在 Telegram，一半在 Discord。最初的方案是跑两个机器人：两份 prompt、两份会话文件、两套部署。结果同一个人在两边问同一个问题，Agent 给出两份基于不同上下文的答案；改一处逻辑要同步两次，很快就受不了了。

这篇帖子记录把一个 OpenClaw Agent 同时挂到两个平台的实践，重点不在“怎么连上”，而在路由和适配。

## 问题拆解

双平台接入的核心其实是四件事：

1. **会话隔离**：同一个 Agent，Telegram 和 Discord 的会话键必须分开，同一平台内不同群组/频道/私聊也要分开。
2. **消息归一化**：两边的事件结构完全不同，进入 Agent 前要收敛成统一的内部格式。
3. **回复适配**：Telegram 用 MarkdownV2（转义极严格），Discord 用自己的 markdown 和 embed；长度上限相近，但分段逻辑不能共用。
4. **能力差异**：inline keyboard 和 Discord button 不互通；文件大小、媒体处理、限速策略各不相同。

## 做法

架构上只守一条原则：**adapter 只做翻译，不做理解**。

- **接入层**：Telegram 走 long polling（家庭网络无公网，省掉 webhook 和反代），Discord 走官方 gateway，各自作为 OpenClaw 的 channel 插件运行。
- **路由层**：入口统一生成 `platform:scene:id` 形式的会话键，例如 `telegram:group:-100123`、`discord:thread:456`。Agent 侧完全不感知平台差异。
- **渲染层**：Agent 输出统一的中间格式（纯文本 + 代码块标记），各 adapter 自行转成平台方言；超长时按平台策略分段——Telegram 在空行处切，Discord 分片发送且必须感知 fenced code block 边界。
- **身份映射**：同一个人在两边的 alias 收敛到一个 YAML 文件，为后续权限和统计留口子。

关键差异对照：

| 维度 | Telegram | Discord |
|---|---|---|
| 渲染 | MarkdownV2 / HTML | markdown + embed |
| 交互 | inline keyboard | buttons / slash command |
| 限速 | 约每群 1 条/秒 | 每频道 5 条/5 秒 |
| 接入 | long polling / webhook | gateway |

## 踩坑点

- **MarkdownV2 转义**：`_ * [ ] ( ) ~ > # + - = | { } . !` 全要转义，漏一个整条消息 400。最后正文退化为纯文本、代码块走 `<pre>`，稳定优先。
- **分段切断代码块**：按固定长度切会把代码块劈成两半，渲染全乱。分段逻辑必须识别围栏边界。
- **限速**：Discord 每频道 5 条/5 秒，Telegram 每群约 1 条/秒。共用一个带平台感知的发送队列，否则群活跃时直接被限流。
- **事件去重**：Discord gateway 偶发重发，Telegram polling 超时重拉也会重复。按 message id 做幂等，别信平台不重复。
- **差点成环**：一度想加“两边互通”，结果 A 平台的消息经 Agent 回到 B 平台，又被 B 平台的 adapter 捕获。跨平台桥接必须带来源标记，且 adapter 层直接丢弃自己发出的消息。

## 可复用建议

1. adapter 越笨越好，所有判断放进路由层或 Agent；
2. 内部消息格式加 version 字段，避免日后结构变更变成迁移灾难；
3. 每个平台准备 3–5 条录制的真实事件 fixture 做回归测试，比手动点来点去可靠；
4. 监控发送队列积压和失败率——线上问题多半先表现为“回得慢”，而不是报错；
5. 功能按平台开关（feature flag），按钮、投票这类能力不对称时，别让 Agent 误以为两边能力一致。

## 总结

双平台接入的复杂度不在连接，而在消息语义的翻译与一致性。把会话键设计、渲染适配、幂等去重这三件事想清楚，之后再接 Slack 或飞书，基本就是复制一个 adapter 的工作量。这套结构目前稳定运行数周，追加了两个群组，边际成本确实很低。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/a0ad875cc4c89ccd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/43e275db5b9964ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/7ec4ad8b61314fe8.png)

