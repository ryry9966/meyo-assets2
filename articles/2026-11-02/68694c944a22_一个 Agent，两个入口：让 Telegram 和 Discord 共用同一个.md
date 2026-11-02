---
title: 一个 Agent，两个入口：让 Telegram 和 Discord 共用同一个大脑
feedId: 43260
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

我的 Agent 最初只挂在 Telegram 上服务自己的群。后来 Discord 那边的社区也想用，第一反应是再跑一个实例——很快发现这不是"多一份配置"的事：两份记忆、两套插件状态、cron 任务跑两遍，排障时还得在两边翻日志。更合理的方向是：**一个 Agent 内核，多个消息入口**。

## 问题到底在哪

接两个 Bot API 本身不难，难的是三件事：

1. **会话语义**。Telegram 群和 Discord 频道的上下文要不要共享？默认必须隔离，否则两边的人会串进同一个会话，Agent 回复对象错乱。
2. **格式降级**。Discord 单条上限 2000 字符，Telegram 是 4096；Markdown 方言不同，Telegram 的 MarkdownV2 转义规则堪称灾难。
3. **回程路由**。Agent 回复必须带着"从哪来、回哪去"的路由元数据。异步任务（定时摘要、告警推送）尤其容易回错门。

## 做法

架构上把"入口"压成薄适配层，内核只面对统一信封：

```
Telegram adapter ─┐                     ┌─ Telegram renderer
                  ├─ normalize ─ Agent ─┤
Discord adapter ──┘                     └─ Discord renderer
```

具体步骤：

1. **双通道接入**。在 `openclaw.json` 里同时声明 `telegram` 和 `discord` 两个 channel，绑定同一个 agent。Discord 别忘了在开发者后台打开 MESSAGE CONTENT intent，否则收到的消息内容是空的——最经典的第一次翻车。
2. **统一会话键**。session key 用 `channel:chatId:userId`，平台间默认隔离。确认是同一个人跨平台使用（白名单 + 手动映射）后，再做会话合并，不要自动合并。
3. **出站渲染分离**。Agent 只产出一份标准 Markdown 正文加 route 元数据，各 renderer 各自处理：按 2000/4096 切分、表格降级为缩进文本、Telegram 侧统一用 HTML parse mode 绕开转义。
4. **出站队列 + 限速**。每个平台挂一个带退避重试的队列。Discord 单频道约 5 条/5 秒、Telegram 单 chat 约 1 条/秒，突发回复必触发 429，不要裸发。
5. **异步任务显式带路由**。cron、订阅类任务在创建时就写入目标 route，禁止"回复最后一条消息的来源"这种隐式推断。

## 踩坑点

- **切分撞上代码块**：按字符数硬切会把 ``` 围栏拦腰截断，后半段全部渲染错乱。切分逻辑要在围栏边界让步。
- **双实例重复回复**：网关进程跑了两份（比如 pm2 和手动各起了一个），两个平台都会收到双份回复。加单实例锁。
- **语音消息**：Telegram voice 是 ogg/opus，Discord 文字频道没有语音条概念，要么转文件附件，要么提前转写成文本。
- **消息编辑**：Discord 有 edit 事件，别想着全量同步，明确"编辑不回溯"这条规则即可。

## 可复用建议

- 适配层保持"哑"，所有智能（路由、会话、格式策略）收敛在内核。新平台接入只是写一个 adapter + renderer，不动业务逻辑。
- 日志里每条收发消息都带 route key（platform/chat/user），跨平台排障基本全靠它。
- 给每个平台留能力开关：桌面端合理的功能（长表格、大附件）在手机端体验未必成立。

## 总结

一个 Agent 服务两个平台，工程量的大头不在接 API，而在**会话语义**和**格式降级**这两个看不见的地方。先把会话隔离、降级渲染、异步任务显式带路由这三条定下来，之后接第三个平台（Slack、WhatsApp）基本就是复制粘贴的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/8ff98a8687ae7e17.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/827515602dd5a78c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/02a2b61fa69d347c.png)

