---
title: 跨平台消息路由：让一个 Agent 同时值守 Telegram 和 Discord
feedId: 42037
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

我的 OpenClaw Agent 最早只挂在 Telegram 上，跑提醒、查脚本输出都够用。后来团队协作挪到了 Discord，第一反应是再起一个 Agent——但很快发现两套工作区、两份记忆、两套技能配置，维护成本直接翻倍。目标于是变成：**同一个 Agent、同一份工作区，同时值守两个平台，且会话互不串扰。**

## 问题在哪

多接一个 bot 本身不难，各渠道插件是独立的。真正要处理的是三件事：

1. **会话归属**：同一个人在两个平台，算不算同一个上下文？
2. **输出格式**：Telegram 的 MarkdownV2 和 Discord 的 markdown 是两种方言，转义规则互不兼容。
3. **平台限制**：长度上限（Telegram 4096、Discord 2000）、限流策略、媒体类型支持都不一样。

这些差异如果泄漏进 prompt，agent 就得自己“猜平台”，输出会不稳定。

## 做法

1. 两个平台各建 bot 拿 token，填入配置的 `channels` 段，都落到默认 agent：

```yaml
channels:
  telegram:
    botToken: "..."
    dmPolicy: pairing
  discord:
    token: "..."
    dmPolicy: allowlist
```

2. **路由按“渠道 + 会话”粒度隔离**。session key 自带渠道前缀，天然不串。群聊只响应 @ 提及，DM 走 pairing/allowlist，不做开放应答。
3. **约定 agent 输出平台无关**：只产普通 markdown 和纯文本。转义、分段、超长裁剪全部下沉到渠道投递层，prompt 里不出现任何平台专有语法。
4. **跨平台记忆显式落盘**：需要两边共享的结论，让 agent 写进共享 workspace 的记忆文件，不指望会话历史互通。
5. 上线前用私有测试群和测试频道各跑一轮典型请求（长文、代码块、图片、语音），验证双端投递。

## 踩坑点

- **MarkdownV2 转义**是 Telegram 侧最常见的发送失败原因，报 400。把 parse mode 降到 HTML 或纯文本后基本绝迹。
- **Discord 2000 字符上限**：长回复要硬分段，别指望 prompt 里写“请简短”就完事，模型不保证。
- **同一人两平台 = 两个会话**。有次我在 Discord 问“刚才那个脚本”，agent 一脸茫然——会话历史不互通是特性，不是 bug。
- **群聊全量响应必炸**：刷屏、限流、429 接踵而至，mention 触发是底线。
- **一个 token 别喂两个进程**，gateway 会互踢，症状是 bot 反复上下线。
- 语音消息只有 Telegram 侧能收，转写是单渠道能力，prompt 里应说明能力边界。

## 可复用建议

- 分层原则：**平台差异关在渠道层，人格与工具留在 agent 层**。以后加第三个平台只是加一个 channel，不动 agent。
- session key 命名规范第一天就定好，事后迁移很痛。
- 每个渠道单独开投递日志；发送失败先看 4xx/429，再怀疑模型。
- 灰度一周再放开真实用户，重点盯转义和限流两类报错。

## 总结

跨平台路由的本质不是“把消息转发给 agent"，而是把会话归属、格式适配、限流这些平台差异全部关进渠道层，让 agent 始终只有一份。做完之后，日常维护成本和单平台几乎没有区别——这也正是渠道插件架构该有的样子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/df445921a956b524.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/99adc27178b8a5cd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c35bf3f34e20e543.png)

