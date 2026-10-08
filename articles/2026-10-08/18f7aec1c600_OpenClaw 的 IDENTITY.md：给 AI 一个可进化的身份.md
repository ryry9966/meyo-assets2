---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40934
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

OpenClaw 的 agent 不是一次性对话机器人，而是长期驻留、跨会话的个人助理。每次会话启动时，workspace 里的一组 markdown 文件会被注入 system prompt，其中 `IDENTITY.md` 负责回答"我是谁"。默认模板只有几行：名字、形象 emoji、语气风格。很多人装完就忘了它，但这其实是整个系统里投入产出比最高的文件之一。

## 问题

没认真维护 `IDENTITY.md` 时，常见症状有三类：

- **人格漂移**：周一的它冷静克制，周五开始满口网络梗，因为人格全靠 SOUL.md 里的模糊描述撑着；
- **多渠道不一致**：Telegram 上简洁，Discord 上话痨，用户对"它"没有稳定预期；
- **多 agent 混淆**：两个 workspace 用了同一套人设，消息里谁是谁分不清。

根因是身份信息缺少一个稳定、可版本化的载体。

## 做法

1. 定位 workspace（默认 `~/.openclaw/workspace`），打开 `IDENTITY.md`；
2. 只写**稳定字段**：name、emoji、vibe、两三句的自我定位，控制在 30 行以内——它是每次会话的固定上下文开销，写得越花哨，token 税越高；
3. 和 `SOUL.md` 明确分工：IDENTITY 管"是谁"（名字、形象、边界），SOUL 管"怎么处事"（价值观、语气）。不要两边重复写，冲突时模型表现是随机的；
4. 放进 git。身份的每次调整对应一个 commit，diff 和回滚都有据可查；
5. 验证：改完后开新会话，分别在 Telegram 和 WebChat 让它自我介绍，确认加载生效、口吻一致。

## 可进化：让 agent 参与修订

"可进化"不是自动改写。我的流程是：季度性让 agent 基于近期对话复盘，输出一份身份修订草案——哪些设定被频繁打破、哪些新偏好值得固化；然后我自己审 diff、改写、提交。**agent 起草，人合并。** 用 git 管这个流程，比任何"自动人格进化"的玄学都可靠。

## 踩坑点

- **写成小作文**：800 字人设不会让人格更立体，只会稀释指令权重，30 行封顶；
- **塞入易变信息**：当前项目、情绪状态属于 MEMORY.md 或日记，不是身份；
- **只测一个渠道**：emoji 和长段落在部分客户端渲染不同，发出去才知道；
- **改了文件却复用旧会话**：workspace 文件在新会话才重新读取，记得 `/new`；
- **多 agent 共用 workspace**：身份必须隔离，不同 agent 用不同 workspace 路径，在 config 里分别指定。

## 可复用建议

- 把 `IDENTITY.md` 当接口契约：字段稳定，改动走 git review，小步提交；
- 一个 agent 一份身份，用 workspace 天然隔离；
- 名字和 emoji 全渠道一致，这是用户识别你的 agent 成本最低的手段；
- 让 agent 提案、人拍板的修订节奏，比频繁手改更能保持人设连贯。

## 总结

`IDENTITY.md` 很小，但它是 agent 长期一致性的锚点。工程化地对待它：短、稳定、可版本化、人审进化。花一个下午写清楚这几十行，之后每个月都能省下纠正人格漂移的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/bb136e19daf18a5a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/055b7344e3fd7159.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/0cc000c823bcce61.png)

