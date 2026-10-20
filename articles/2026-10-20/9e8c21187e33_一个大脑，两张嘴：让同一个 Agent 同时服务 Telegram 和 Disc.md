---
title: 一个大脑，两张嘴：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 42100
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

我最早只在 Telegram 上跑了一个 OpenClaw Agent，管待办、查资料、挂几个 MCP 工具，一直很稳。后来团队日常沟通挪到了 Discord，第一反应是再部署一个实例。跑了两周就受不了了：两套会话、两份记忆、两份工具配置，改一处系统提示词要同步两遍。这篇帖子记录把它们收敛到一个 Agent 上的过程。

## 问题

核心诉求就三条：

1. 同一个 Agent（同一份系统提示词、同一组 MCP 工具、同一份 workspace 记忆）同时响应两个平台；
2. 各平台的会话上下文互相隔离，Telegram 私聊不该知道 Discord 群里聊了什么；
3. 回复永远路由回消息来源的渠道，不串线。

## 做法

OpenClaw 的架构里，渠道本质是适配器，Agent 不感知"平台"差异，只感知会话。所以多渠道接入主要是配置层面的事：

**1. 准备两个机器人凭证。** Telegram 找 @BotFather 拿 token；Discord 在开发者后台建应用、邀请机器人进服务器。

**2. 在 `~/.openclaw/openclaw.json` 的 channels 下同时启用两个渠道**（字段名以你的版本为准）：

```json
{
  "channels": {
    "telegram": { "botToken": "123:abc" },
    "discord":  { "token": "xxxx" }
  }
}
```

**3. 重启 gateway**，日志里确认两个渠道都进入 connected 状态。

**4. 验证会话键。** OpenClaw 默认按"渠道 + chatId"维度切分会话，两个平台各聊各的，互不污染。用 `/status` 可以确认当前会话归属。

**5. 工具与记忆共享。** MCP server 和 workspace 是 gateway 级配置，两个渠道天然共用。某个平台想独占工具，在渠道级配置里做覆盖即可。

**6. 平台差异化行为不要起两个 Agent。** 在系统提示词里声明"消息可能来自不同平台，注意各自的 Markdown 方言和长度习惯"，让 Agent 按消息携带的渠道元数据自适应。

跨渠道主动通知（比如 Discord 里跑完长任务后往 Telegram 推结果），走 Agent 自带的消息发送工具并指定目标渠道，别自己写脚本直连 Bot API——会绕过 OpenClaw 的会话管理，后面没法排查。

## 踩坑点

- **Discord 必须手动开启 Message Content Intent**，否则机器人在线但永远收不到消息，还不报错，我在这卡了半小时。
- **Telegram 同一 token 只能有一个轮询消费者**。之前在别处设过 webhook、或起了第二个实例，会报 409 conflict。
- **Markdown 方言**：Agent 默认输出 GitHub 风格，表格和 `***` 混用语法到 Telegram 会原样露出来。依赖适配器转换，不行就在提示词里禁表格。
- **Discord 有限速**，长回复会被切片，注意代码块被截断的问题。
- **Discord 线程会生成独立会话键**，频繁开线程会让上下文碎片化，建议在提示词里约束"尽量在原会话回复"。

## 可复用建议

- 一个 Agent + N 个渠道是默认正确姿势，除非两边人格或工具集确实需要不同；
- 平台差异交给适配器和提示词，别在业务逻辑里写 `if platform == ...`；
- 上线前过一遍固定清单：私聊、群里 @、回复引用、发图片、发语音、超长消息，六个场景各测一次。

## 总结

多渠道接入的难点从来不是"连上"，而是会话隔离、格式转换、限速这些脏活。OpenClaw 把连接层抽成了适配器，我们要做的只是把配置写对，然后老老实实把边界场景过一遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/33569380f8430957.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/f685ed6b8d8dd916.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/18d45e40986c53d3.png)

