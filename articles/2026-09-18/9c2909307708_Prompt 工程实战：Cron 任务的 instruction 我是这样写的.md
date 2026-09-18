---
title: Prompt 工程实战：Cron 任务的 instruction 我是这样写的
feedId: 38063
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 的 cron 任务本质上是"定时投递的一条 instruction"：到点之后，agent 拿着这段文字冷启动执行——没有会话上下文，也没有你在旁边确认。很多人把 cron instruction 当成聊天消息来写，任务跑了，产出却不可控。

## 问题

我早期写过这样的任务："每天早上帮我看看关注的仓库有什么更新，总结一下。"跑了两周发现问题：总结有时发到当前会话（没人看），有时去翻了根本没配 token 的仓库，有一次直接空跑。根源都一样——instruction 信息不足，agent 每次都在自行补全，补全方向随机。

Cron instruction 和聊天 prompt 最大的区别是：**它是无人值守的**。所有你在对话里会临时口头补充的信息（数据从哪来、结果写到哪、失败了怎么办），都必须写进 instruction 本身。

## 做法

我把每条 cron instruction 当成一个微型 SOP，固定五段：

```text
[目标] 一句话说清产出什么
[输入] 数据从哪来：哪个 MCP 工具、哪个文件路径
[步骤] 按序号列动作，每步动词开头
[输出] 结果写到哪、什么格式
[异常] 工具失败/数据为空时怎么做，明确"不要提问"
```

改造后的例子：

```text
[目标] 汇总 workspace/logs/ 下 24h 内的 ERROR 日志
[输入] 读取 workspace/logs/，只看修改时间在 24h 内的文件
[步骤] 1. 列出文件；2. 过滤 ERROR 行；3. 按文件去重统计
[输出] 追加写入 workspace/reports/daily-error.md，含日期标题
[异常] 目录为空则写"无错误"；读取失败则记录原因后结束，重试不超过 1 次
```

改造后连续跑了一个月，没再空跑。配套三个习惯：

1. **上线前手动跑一次**：instruction 原样发给 agent，确认结果再挂 cron。
2. **先高频试跑**：用 10 分钟间隔观察 3~5 次，稳定后再改正式周期。
3. **留运行痕迹**：让任务每次在固定 journal 文件追加一行（时间 + 结果摘要），排障全靠它。

## 踩坑点

- **引用不存在的上下文**："接着上次继续"——cron 每次独立执行，没有"上次"。状态要么落盘，要么让任务重建。
- **依赖交互确认**：凡可能出现"请确认"的流程，要在 instruction 里明确授权或禁止，否则任务会卡在等待。
- **路径和工具名写模糊**："工作目录里的报告"不如 `workspace/reports/`；MCP 工具写全称，别让 agent 猜。
- **一个任务塞多件事**：抓取、分析、通知拆开，或只产出中间文件，比几百字巨型 instruction 可靠得多。
- **时区**：cron 表达式按服务器/容器时区算，上线前和本地时间对一遍。

## 可复用建议

把五段式存成模板，新任务先填模板。instruction 控制在一眼能读完的长度，细节 SOP 单独放文档，在 [输入] 里用绝对路径引用让 agent 先读它。每季度审一遍存量任务：工具改名、路径变更、API 失效，instruction 不会自己更新。

## 总结

Cron instruction 的写作，本质是把"有人在场的对话"压缩成"无人在场的说明书"。写完自问一句：一个完全不了解背景的 agent，拿到这段话能否不看聊天记录就把事做对、把结果放到该放的地方？能，这条任务才算写完。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/59a0c58c2465dcec.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/5a82152b6138b93d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9ebd7934b3613e7b.png)

