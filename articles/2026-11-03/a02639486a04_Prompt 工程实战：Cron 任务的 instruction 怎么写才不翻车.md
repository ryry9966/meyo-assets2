---
title: Prompt 工程实战：Cron 任务的 instruction 怎么写才不翻车
feedId: 43310
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

OpenClaw 的 agent 支持定时任务（cron）：到点自动唤醒 agent，把一段 instruction 喂给它独立执行，跑完即结束。社区里我见得最多的现象是：聊天里调得好好的 prompt，搬进 cron 就"失踪"——要么每次都不动，要么每次都乱动，要么跑两周后变成噪音机器。

原因很简单：cron 里的 agent 是无人值守的。它没有对话上下文、不能反问你、跑完也没人盯着。这决定了 cron 的 instruction 不是聊天 prompt，而更像一份小程序规格说明。

## 问题：cron instruction 的三种常见死法

1. **上下文缺失**。聊天里你可以说"把刚才那个文件跑一下"，cron 里没有"刚才"。instruction 就是 agent 的全部世界。
2. **时间锚点模糊**。"检查一下最近的提交"——"最近"是多久？agent 的"今天"和你 cron 的时区未必一致。
3. **输出无处安放**。agent 默认会"回复"，但 cron 没有对话对象。不说清写到哪、发给谁，结果就是落空或编造。

## 做法：固定成六段来写

我沉淀了一个模板，把每条 cron 的 instruction 写成六段：

```text
【目标】一句话，只做一件事
【输入】明确数据源：确切路径 / 命令 / MCP 工具名
【步骤】1-2-3 列出动作顺序
【判断标准】什么算"值得报告"，给出具体阈值
【输出】写哪个文件 / 发到哪个渠道，格式是什么
【异常】数据为空或不可达：记日志、静默退出，禁止编造
```

举一个每天早八巡检插件仓库的例子：

```text
【目标】巡检插件仓库，报告异常
【输入】运行 git -C /srv/plugins log --since="24 hours ago" --oneline
【步骤】1) 获取提交列表 2) 检查是否涉及 src/ 目录 3) 读取 /srv/state/last_review.txt 中的上次处理时间，跳过已处理提交
【判断标准】涉及 src/ 且含 "fix" 或 "breaking" 的提交才算异常
【输出】有异常时向指定渠道发消息，最多 5 条要点，每条附 commit hash；无异常时在 /srv/state/run.log 追加一行心跳，不发消息
【异常】命令失败或输出为空：记入 run.log，本次静默结束
```

核心思想一句话：**把"决策权"和"汇报权"都写成显式规则**，让 agent 每次执行的是同一套逻辑，而不是重新猜你的意图。

## 踩坑点

- **默认无状态**。每次 cron 运行相互独立，"上次的进度"必须自己落到文件里，并在 instruction 中写清读取与恢复逻辑。这是踩得最多的坑。
- **让 agent 自己报时间**。让它执行 `date` 取当前时间，时区写死在 instruction 里，别信它的"体感时间"。
- **一个 cron 干三件事**。巡检、汇总、清理拆成独立任务，各自定频率、各自排障。instruction 越长，跑偏概率越高。
- **不做幂等**。任务重跑或触发重叠时会重复发通知。让 agent 处理前先标记、标记后再汇报。
- **直接上生产**。先以高频 + 只写日志不发包的方式 dry-run 两三天，确认输出符合预期，再接入真实渠道。

## 可复用建议

- instruction 存成文件进 git，别在 UI 里随手改。有 diff、能回滚、能 review。
- 每个 cron 配一个统一的 run.log，每次运行追加时间戳和结果摘要，排障效率提升一个量级。
- 永远写一条"静默退出"规则：没达到标准就什么都不做。宁可漏报一次，也别让它为了"完成任务"硬凑输出。
- 给输出加自检：要求消息里的数字必须来自工具返回值，查不到就不写。

## 总结

Cron 的 instruction 不是 prompt，而是给无人值守 agent 的运行手册：输入明确、逻辑确定、输出有归属、异常有出口。你写得越像程序规格，它就跑得越像程序；你写得越像聊天，它就越会用想象补全你没说清的部分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/a400e3402a76b794.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/1a4f3dc3487c38db.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/b8a462bf73753741.png)

