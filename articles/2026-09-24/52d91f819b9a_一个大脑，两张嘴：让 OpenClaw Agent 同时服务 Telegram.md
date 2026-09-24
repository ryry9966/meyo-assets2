---
title: 一个大脑，两张嘴：让 OpenClaw Agent 同时服务 Telegram 和 Discord
feedId: 38749
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我的联系人分散在两个地方：技术社区在 Discord，家人朋友和一部分工作群在 Telegram。早期我给两边各跑了一个 agent 实例，很快就分叉了：同一个我要在两边重复交代上下文，笔记、待办、长期记忆各一份，行为也开始不一致——一边答得准，另一边连我的时区都记错了。于是花了一个周末，把结构收拢成一个 gateway、一个 agent、两个 channel。

## 问题

核心诉求有三个：

1. 两个平台的消息都能触发**同一个 agent**，共享同一份工作区（记忆、笔记、定时任务）。
2. 会话边界可控：私聊、群聊、不同平台之间不能互相串上下文。
3. 渲染和限制差异在 channel 层消化，不污染 agent 的输出逻辑。

## 做法

OpenClaw 的架构里，gateway 是消息枢纽，每个平台是一个 channel 插件，agent 本身不感知平台差异。我的步骤：

1. **同时配置两个 channel。** 在 `~/.openclaw/openclaw.json` 里声明 telegram（botToken + allowlist）和 discord（bot token + guild 限制），两边都只放行自己的账号 ID，绝不裸奔。
2. **规划 session 路由。** 用 bindings 把 Telegram 私聊、Telegram 群、Discord 私聊、Discord 频道分别绑到不同的 session key，但全部指向同一个 agent。效果是“一个大脑、多张嘴”，上下文却按 peer 隔离。
3. **共享状态放工作区。** 跨平台的长期记忆不要依赖会话历史，让 agent 统一读写 workspace 下的 `MEMORY.md`、笔记和每日 todo。会话是易失的，文件才是持久层。
4. **定时任务只走一个出口。** cron 和 heartbeat 固定走 Telegram 私聊，避免同一份日报在两边各发一遍。

```jsonc
// openclaw.json（节选）
"channels": {
  "telegram": { "botToken": "...", "allowFrom": ["me"] },
  "discord":  { "token": "...", "allowedGuilds": ["my-guild"] }
}
// agent 只有一个，bindings 里按 peer 分 session key
```

5. **双端实测。** 两边各发一轮消息、语音、图片、长文本，确认路由和媒体处理符合预期。

## 踩坑点

- **Telegram 群隐私模式。** 默认 bot 在群里看不到普通消息，只看到 /command。要么在 BotFather 里关掉 privacy mode，要么把路由设计成“群里以命令为主”。我踩过：群消息全被吞，排查半小时才发现。
- **Discord 频道的触发策略。** 建议只响应 @mention，否则 agent 会对每条闲聊出手，既是噪音也是 token 开销。
- **长度与格式差异。** Telegram 单条上限 4096，Discord 2000，且两边 markdown 方言不同。长回复交给 channel 层自动分段，代码块格式让插件按平台适配，别在 agent prompt 里手写两套格式规则。
- **共享 session 的隐私陷阱。** 我一度图省事把两个平台绑到同一 session，结果 Discord 里聊的内容渗进了 Telegram 的回复语境。跨平台共享的应该是“记忆”，不是“对话历史”。
- **速率限制。** Discord 同一频道约 5 条/5 秒，heartbeat 或群发场景要加节流。

## 可复用建议

- channel 保持“薄”：只做鉴权、触发、渲染适配，业务逻辑一律进 agent 和工具。
- 路由规则集中在一处配置，别散落在脚本里。
- 调试日志带上 channel + peer 标识，排障时一眼定位来源。
- 未来接第三个平台（比如 Slack）时，只加 channel 和 binding，agent 侧零改动——这是这套结构最大的红利。

## 总结

跨平台不是把 agent 跑两份，而是把“身份、记忆、任务”收敛到一处，把“接入、鉴权、渲染”分发到各 channel。OpenClaw 的 gateway + bindings 模型天然适合这个玩法。收拢之后，我不再需要向两个 bot 重复自我介绍，两边的朋友都觉得“它记得住事”——其实只是它终于只有一个大脑了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/92bfff14bf2333ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9d011403e8be9deb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f2dc9012f6d0d44e.png)

