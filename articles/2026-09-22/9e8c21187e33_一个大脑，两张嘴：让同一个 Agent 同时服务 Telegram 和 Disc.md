---
title: 一个大脑，两张嘴：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 38454
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

起因很简单：用户一半在 Telegram 群，一半在 Discord 服务器。最早的做法是跑两个 Agent 实例、各接一个 channel。一个月后问题暴露——同一个用户在两边是两份互不相识的上下文，记忆文件写了两套，prompt 改一处漏一处。Agent 只有一套人格设定，却像分裂成了两个人。

所以这次的目标很明确：**一份 workspace、一份会话记忆、一个 Agent 进程，同时挂 Telegram 和 Discord 两个通道**。

## 问题在哪

两个平台各自跑通不难，难的是“同时、同一个”。真正的工程问题集中在四点：

1. **会话隔离粒度**：session key 按平台+chatId 隔离，还是按“人”合并；
2. **消息格式差异**：Telegram 的 MarkdownV2 转义规则和 Discord markdown 互不兼容，mention、code block 的解析方式完全不同；
3. **出站限制不同**：Telegram 单条 4096 字符，Discord 2000，且每频道限速 5 条/5 秒；
4. **附件引用机制不同**：Discord 的 CDN 链接带签名会过期，不能直接长期引用。

## 做法

分层思路只有一句话：**Agent 永远不感知平台差异，所有差异在 channel 层被抹平**。

1. **单 Agent 双通道**：在 OpenClaw 配置里把 telegram 和 discord 两个 channel 都指向同一个 agent workspace（示意，字段以你所用版本的实际 schema 为准）：

```json
{
  "agents": { "main": { "workspace": "~/agent-workspace" } },
  "channels": {
    "telegram": { "botToken": "***", "dmPolicy": "pairwise" },
    "discord":  { "botToken": "***" }
  }
}
```

2. **session key 按“人”归一**：在 channel 层做一次身份映射，把 `telegram:12345` 和 `discord:67890` 映射到同一个内部用户 ID，同一个人换平台提问，Agent 依然记得他。群聊例外：群 session 仍按 `platform:chatId` 隔离，避免两个群的上下文串味。

3. **统一消息信封**：内部定义规范信封（文本、附件、reply 引用、发送者）。入站时 channel 把平台消息翻译成信封，出站时再翻译回去，Agent 只看信封。

4. **出站适配器**：按平台做 chunking——在代码块边界切分，而不是硬切 2000 字符；限速用队列加退避；Telegram 出站统一用 HTML parse mode，躲开 MarkdownV2 的转义地狱。

## 踩坑点

- **MarkdownV2 转义几乎必踩**：`_` `*` `[` 出现在代码内容里就炸。结论：能用 HTML mode 就别用 MarkdownV2。
- **跨平台同时消息的竞态**：同一用户两边几乎同时发消息，两个事件并发写同一份记忆。加 per-session 串行队列即可，不必上复杂的锁。
- **Discord 附件链接过期**：早期直接把 CDN URL 喂给 Agent，隔天就读不了。改为入站时下载到 workspace 的 media 目录，消息体里只留本地路径。
- **斜杠命令语义冲突**：两个平台 `/help` 的约定不同，统一由 Agent 的命令层处理，channel 只透传。

## 可复用建议

- **先跑影子通道**：新平台先以只读/白名单模式接入观察一周，再放开。
- **失败日志按平台分开**：入站正常、出站被限速这类问题，混在一份日志里根本查不动。
- **信封结构尽早定**：字段宁多勿少，宁可先冗余 reply/quote，也别事后补格式。

## 总结

这件事的本质是“一个大脑、多个嘴”。channel 适配器只做翻译和搬运，不做决策；平台差异在边界处消化。做完后最大的收益不是省了一个进程，而是记忆、prompt、工具配置只有一份——Agent 的行为终于可预测。下一步计划把第三个通道（邮件）也挂上来，验证这套信封模型的扩展性，有结果再来同步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/da4c291ca1a42feb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/fdafaf0f4a2e0a2f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/219a2895d170fbd3.png)

