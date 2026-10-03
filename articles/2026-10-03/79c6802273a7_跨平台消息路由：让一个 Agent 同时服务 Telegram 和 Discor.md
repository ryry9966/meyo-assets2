---
title: 跨平台消息路由：让一个 Agent 同时服务 Telegram 和 Discord
feedId: 40202
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

我的 Agent 一直跑在一台小 VPS 上，日常在 Telegram 里当私人助手。但团队协作都在 Discord，同事也想直接问它。与其让人切换平台，不如让 Agent 同时接入两边。OpenClaw 的 channel 机制本身就是多渠道设计，接入本身不难，难的是接完之后不串台、不炸格式。

## 问题

真正要解决的是四件事：

1. **会话隔离**：两边用户共享 session 会上下文互相污染，完全隔离又失去共享记忆。
2. **回复路由**：回复必须回到消息来源的 channel，不能 A 平台提问、答案发到 B 平台。
3. **格式适配**：Telegram 的 MarkdownV2 转义严格，Discord 的 markdown 和代码块行为不同。
4. **速率与长度**：两边的 rate limit 和单条消息长度上限不一样。

## 做法

1. **双 channel 配置**：Telegram 用 bot token + polling，Discord 用 bot token + gateway，都指向同一个 agent 实例，共享同一份 workspace 和模型配置。
2. **session key 按 `platform:chat_id` 设计**：`telegram:12345` 和 `discord:67890` 是独立会话，长期记忆放在共享 workspace，跨平台生效。
3. **回复走原路**：不自己写路由逻辑，依赖消息模型里 reply 绑定发起 session 的特性，天然不会串台。只有跨平台主动广播（比如定时摘要）才显式指定目标。
4. **格式适配收敛到一个 helper**：Telegram 侧降级为纯文本或基础 Markdown，避免转义地狱；Discord 侧代码块统一用三反引号包裹。适配层不做任何业务逻辑。
5. **长消息按平台切分**：Discord 按 2000 字符切，Telegram 按自身上限切，切段时检测不要把代码块拦腰截断。

## 踩坑点

- Telegram MarkdownV2 对 `_`、`*`、`[]` 都要求转义，直接把模型输出塞进去就是 400。最省事的做法是改用 HTML parse，或者干脆不解析。
- Discord 侧只监听了普通消息，slash command 一直是哑的，排查半天才发现是两套入口。
- 附件大小限制不同：Discord 明显更小，大文件要么在 Telegram 侧处理，要么改走外部链接。
- 定时任务的输出默认打到了“主 session”，结果进了一个没人看的会话。所有定时输出必须显式指定目标 channel。
- 同一个真人在两边都有账号，先不做身份合并，只在备注字段里关联，避免权限互相污染。

## 可复用建议

- session key 永远带平台前缀，给第三个平台（Slack、飞书）留好扩展位。
- 格式 helper、切分逻辑写成独立模块，channel 层保持薄。
- 给每个 channel 加心跳探测，掉线要能第一时间发现，而不是等用户来报障。
- 回复生成端不假设平台特性：图片、按钮、embed 全部可降级为纯文本。

## 总结

多渠道接入在 OpenClaw 里是配置层面的体力活，真正的工作量集中在三件事：**会话隔离、格式适配、回复路由**。把这三层想清楚并做成独立模块，之后加第三个渠道，就只是重复一遍配置、多写一个 helper 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/cbb54084c06336f4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/d60377918e5c0664.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/15c552ebc9f2773b.png)

