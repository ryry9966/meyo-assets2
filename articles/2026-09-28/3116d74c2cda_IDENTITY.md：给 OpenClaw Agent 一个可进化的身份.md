---
title: IDENTITY.md：给 OpenClaw Agent 一个可进化的身份
feedId: 39226
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的 Agent 是无状态的：每个新会话都要靠 workspace 里的几个 Markdown 文件把"它是谁"重新装回来。SOUL.md 管性格和边界，USER.md 管你是谁，而 IDENTITY.md 管的是最基础的一层——名字、形象、自称方式。默认模板里这几行通常被一键跳过，实际用起来才发现它是最容易被低估的配置。

## 问题

跑了一段时间后常见三类情况：

1. 多个 agent（工作号 / 生活号 / 家里服务器）名字和 emoji 全是默认值，通知、日志、会话记录混在一起，分不清谁在说话；
2. 身份从不更新：起名随意，用三个月后连自己都不认，但"改名字"一直排不上优先级；
3. 想让 agent 自己"成长"，直接放开它改 IDENTITY.md，结果几次迭代后自称混乱、语气漂移，回不去了。

## 做法

1. **定位文件**：默认在 `~/.openclaw/workspace/IDENTITY.md`（Docker 部署在容器内对应挂载路径）。字段就几行：Name、Creature、Vibe、Emoji、Avatar，可自行扩展段落。
2. **先写最小身份**：不要超过 30 行。这个文件每个会话都会注入 system prompt，行数就是持续 token 成本。我的模板只加一段"自称规则"（何时用名字、何时用"我"）和一行指向 SOUL.md 的边界引用。
3. **纳入版本控制**：整个 workspace 放进 git。身份的每次变更都有 diff 和 commit message——这是"可进化"的前提，没有版本历史，进化就等于漂移。
4. **设计进化流程**：不让 agent 直接改主文件。约定它只能往 `## Proposed` 段落追加提案（比如"建议把自称从 A 调整为 B，理由是……"），人每周审一次，同意才合并进正文。可以用 cron 任务让 agent 每周发起一次回顾。
5. **改完验证**：开一个新会话问"你是谁、怎么称呼自己"，确认生效。改动通常在下一个会话才被读取，正在进行的会话不会自动感知。

## 踩坑点

- **文件名大小写**：macOS 上 `identity.md` 也能读到，Linux 上不行。迁移后身份"消失"，多半是这个原因；
- **和 SOUL.md 抢活**：把性格、语气、禁区写进 IDENTITY.md，两边指令打架，行为忽冷忽热。分工记一句话：SOUL 管它是什么性格，IDENTITY 管它叫什么、长什么样；
- **emoji 撞车**：多个 agent 用了相似 emoji，推送通知肉眼无法区分，建议给每个实例定死一个独占符号；
- **提案区没人审**：`## Proposed` 堆了几十条没人看，agent 开始自我膨胀。进化流程的核心是人工卡点，不是自动迭代。

## 可复用建议

- 身份变更有意识打 tag（v1、v2），配一行变更说明，回滚只需 `git checkout`；
- 多 agent 场景用命名前缀 + 独占 emoji 做视觉区分；
- 把"自称探针"写进日常巡检：一个固定问题，答案不符就说明身份文件没加载或被改坏；
- 写插件或自动化时不要硬编码 agent 名字，读 IDENTITY.md 的 Name 字段，保持单一事实来源。

## 总结

IDENTITY.md 是个很小的文件，但它把"agent 是谁"从一个隐式的、每会话随机的东西，变成了显式的、可审计的、可回滚的配置。克制地写、有版本地改、有卡点地进化——这三件事做到，身份才真的"活"，而不是"漂"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/ecb89db27cd846ad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/009e337de9801bd3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/14ea034bdfa18f23.png)

