---
title: 一个 Agent 喂两条管道：Telegram + Discord 跨平台路由实战
feedId: 39626
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

社区里很常见的处境：技术讨论沉淀在 Discord，移动端和国内用户聚在 Telegram。最初的方案是跑两个 bot 实例，各接一套 prompt、各养一份记忆。跑了一个月就露馅了——同一个 Agent 在两个平台人格不一致，上下文互不相通，插件配置改一处忘一处。

这篇帖子记录一次收敛：**两个平台接到同一个 Agent 核心，只在入口和出口做平台适配**。

## 问题拆解

“能不能连上”不是重点，真正的难点是三件事：

1. 消息格式差异：Discord 的 embed 和 Markdown 子集 vs Telegram 的 MarkdownV2 转义；
2. 会话身份与记忆共享：同一用户跨平台算一个会话还是两个；
3. 出站节奏：两边的频率限制机制完全不同，不能共用一套发送逻辑。

## 做法

架构上只做一层抽象：**统一信封（envelope）+ 平台适配器**。在 OpenClaw 里落地分五步：

1. **定义统一信封**。所有入站消息先归一化成 `{platform, chat_id, user_id, text, attachments, reply_to, ts}`。平台细节（Telegram 的 update 结构、Discord 的 gateway 事件）在适配器里消化掉，不外泄到核心。
2. **单 Agent 进程，双网关 worker**。Telegram 走 long polling，Discord 走 websocket，两者只管收发，业务判断全在核心。两个网关进程隔离，一个挂了不影响另一个。
3. **会话键用 `platform:chat_id`**。Agent 的长期记忆和工作区共享，短期会话按平台隔离。跨平台身份合并做成显式命令，不自动猜——猜错一次代价很高。
4. **出站走统一格式化管线**：Agent 输出中间格式 → 按目标平台渲染（长度切分、Markdown 降级、附件映射）→ 进入每平台独立的限速队列再发出。
5. **路由规则配置化**。白名单会话、前缀触发、哪些频道静默，全部进配置文件，不写死在代码里。

## 踩坑点

- **Markdown 转义是最大的坑**。Telegram MarkdownV2 对 `_ * [ ] ( ) ~` 几乎全要转义，Agent 输出含代码块的内容时极易 400。最后我们砍到“公共子集”：加粗、行内代码、代码块，其余一律降级纯文本。
- **限速必须按 chat 维度排队**，不是全局。Telegram 单 chat 约 1 msg/s，Discord 按 channel 窗口限制，长回答切分后必须串行投递，否则尾段全部被吞。
- **长度切分选在段落边界**。Discord 2000 字符、Telegram 4096，代码块绝不从中间断开。
- **别用编辑消息做“流式输出”**。两边对编辑都有频控，高频编辑直接 429。老老实实生成完再发，或按段投递。
- **文件大小限制不对称**（Telegram bot API 上传 50MB，Discord 视订阅 8–25MB），超限走外链，别硬传。
- **错误域隔离**。早期把两个网关写在同一棵 async 任务树里，Discord 一次重连风暴把 Telegram 也拖死。拆成独立 worker 加各自健康检查后才稳定。

## 可复用建议

- 信封 schema 加 `version` 字段，后续接 Slack、QQ 时只写适配器，核心不动；
- 限速器和格式化器做成独立模块，平台作为参数传入，业务代码里不出现 if-else；
- 日志统一带 `trace_id + platform`，一条消息从入站到出站能串起来，排障效率完全不一样；
- 用线上抓的真实 payload 做回放测试，比手写 fixture 靠谱得多。

## 总结

跨平台路由本身不难，难的是**格式降级策略**和**出站节奏控制**。把平台差异关进适配器和格式化管线里，Agent 核心保持平台无感，之后新增一个渠道的成本就是“一个适配器 + 一份限速参数”。这是这次改造最值得的部分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/fb63f6724833b764.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/10313c6c01de91af.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/1b54ba74c801712c.png)

