---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 39080
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 的会话模型很直接：一个 session key 对应一份持续累积的上下文。你在 Telegram、WhatsApp 里和主 agent 的对话、工具调用、工具输出，都落在这份上下文里。做自动化时最常见的模式是主 agent 拆任务、spawn 出子 agent 去干活——批量读文件、跑检索、调外部接口。OpenClaw 会给子 agent 分配独立的 session 和独立上下文，结束后只回传一段结果。骨架是干净的，但配置不到位，污染照样发生。

## 问题：污染的三种典型形态

一是**上下文污染**：子 agent 的中间过程——工具日志、重试报错——被整段回传，主会话 token 膨胀，无关细节还干扰后续判断。二是**状态污染**：子 agent 默认继承写权限，往 MEMORY.md 写"临时结论"、往主 workspace 写文件，主 agent 之后基于半成品做决策。三是**行为污染**：子 agent 继承了发消息类工具，干活途中直接对用户渠道说话，用户同屏看到两条口径不一致的回复。我们最早跑"每日汇总 20 个数据源"的 cron 任务时，1 和 3 同时踩了。

## 做法

按五步把隔离做实（具体字段名以你本地版本文档为准）：

1. **spawn 时给任务简报，不给现场。** task 里只写目标、交付物格式、所需文件路径，让子 agent 自己去读，不要把主会话的相关片段整段贴进去。
2. **定义回传契约。** 要求只返回结构化摘要：结论、关键数字、产物路径，并限制长度。主会话里最终只落这一段。
3. **收紧工具与目录。** 用工具 allowlist 只保留完成任务必需的工具，发消息、改配置类一律不给；工作目录指到主 workspace 的子目录，避免覆盖。
4. **隔离 memory 写入。** 子任务的临时结论不进主记忆；确有价值的信息，回到主会话由主 agent 复核后再落盘。
5. **限深度、留痕。** 禁止子 agent 再 spawn；每次派发用独立 run 标识，trace 分开存。出问题时能单独回放子会话的完整记录，而不是翻主会话流水。

## 踩坑点

- **回传摘要不限长**：一个子 agent 返回两千字"详细报告"，五个并发直接撑爆主会话。契约必须限长、限字段。
- **手动复用 session key**：图省事用同一个 key 跑并发任务，工具结果交叉写入，产出互相对不上。别复用，让框架按 run 生成。
- **递归 spawn**：子 agent 遇阻又拉了个孙 agent，空转一下午。加深度上限。
- **漏掉 cron/心跳入口**：定时路径和手动触发走的配置不同步，allowlist 没跟上，偶发越权。所有入口共用同一份 spawn 模板。

## 可复用建议

- 把派发模板固化成 skill 或脚本：目标、交付物 schema、allowlist、深度限制一次写好，所有子任务复用。
- 用 transcript 检查做回归验证：跑一次任务，用 CLI/控制台分别查看主会话与子会话记录，主会话里应只有派发记录和回传结果，出现第三种内容就说明隔离漏了。
- token 消耗按 session 分开统计，子 agent 开销异常能第一时间发现。

## 总结

session 隔离的本质是"权限跟着上下文走"：子 agent 拿到的上下文越小、可写的位置越少、回传越克制，主会话就越稳。OpenClaw 提供了隔离的骨架，allowlist、回传契约、深度上限这些边界仍要自己画。建议每个自动化流程上线前做一次 transcript 检查，确认隔离真的生效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a5c74203d5805252.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/921ef9eee8db65e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/5ef5621dd1ec0f42.png)

