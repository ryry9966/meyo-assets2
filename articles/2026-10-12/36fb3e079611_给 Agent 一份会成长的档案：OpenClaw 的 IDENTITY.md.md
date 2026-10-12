---
title: 给 Agent 一份会成长的档案：OpenClaw 的 IDENTITY.md 实践
feedId: 41278
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

跑 OpenClaw 一段时间后你会发现，Agent 的"性格"其实是散落的：系统提示词里写一点、SOUL.md 里带一点、聊天中口头纠正一点。模型一升级、或者换了个渠道接入，之前调出来的语气就没了。IDENTITY.md 就是 OpenClaw 用来收拢这件事的文件：它和工作区里的 SOUL.md、USER.md 一样，每次会话都会注入上下文，专门回答一个问题——"我是谁"。

## 问题

实际用下来，没有它的痛点很具体：

1. **身份碎片化**：人设散在 system prompt 和历史对话里，不可版本化，也不可迁移。
2. **多 Agent 分不清**：工作号、私人号各跑一个 Agent，群聊和通知里看不出哪条消息是谁发的。
3. **行为漂移不可追溯**：这周它说话风格突然变了，想查"什么时候变的、为什么变"，没有任何 diff 可看。

## 做法

1. **初始化**：引导流程会生成 IDENTITY.md 模板，核心字段就四个：`name`、`creature`、`vibe`、`emoji`。名字定一个你平时叫得出口的；vibe 用三五个形容词，别写成小作文。
2. **保持极小**：这个文件每轮都进上下文，建议控制在 20 行以内。身份是"常量"，任务状态、临时偏好不要塞进来——那是 MEMORY.md 和 USER.md 的事。
3. **纳入 git**：工作区本身就是一个目录，直接 `git init`。每次改身份都提交，commit message 写清动机，比如"群聊里太啰嗦，vibe 加 concise"。
4. **建立进化流程**：Agent 做得不对照旧方式是当场骂它，正确方式是事后改文件。可以让它在心跳任务里定期复盘，propose 一份 IDENTITY.md 的 diff，人工 review 后合并。**身份进化是 PR 驱动的，不是聊天驱动的。**

## 踩坑点

- **写太多**。有人把人设写成两百行小说，结果每轮多烧几千 token，任务表现反而下降——过强的人设会压过工具调用指令。
- **和 SOUL.md 抢地盘**。IDENTITY.md 管"我是谁"（名字、气质、emoji），SOUL.md 管"我怎么做"（边界、原则）。两边都写语气规则，模型就会来回摇摆。
- **emoji 随便定**。多 Agent 场景里这个 emoji 会出现在消息前缀，是区分"谁在说话"的关键，选辨识度高、不重复的。
- **只进化不留痕**。没有 git 的进化等于漂移，出问题时你答不上来"它是什么时候变成这样的"。

## 可复用建议

- 把 IDENTITY.md 当**配置文件**而不是人设文档：小、稳定、可 review、可回滚。
- 不同渠道可以 fork：工作渠道用克制版，私人渠道用放松版，本质是同一身份的不同 profile。
- 模型升级后跑一次回归：用固定一组问题对比输出语气是否符合 vibe 字段，不符再改文件，而不是凭感觉重调。
- 变更日志靠 commit message 承担，不要在文件里再维护"更新记录"段落——那只会让文件膨胀，背离它的定位。

## 总结

IDENTITY.md 的价值不在于让 Agent 更"像个人"，而在于把身份从不可控的对话流里抽出来，变成一份可版本化、可 review、可 diff 的配置。身份调整的粒度，从"今天心情不好重新调教一遍"变成"提一个 PR"。这一步做扎实了，多 Agent 路由、长期记忆、渠道个性化才有共同的锚点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/f948d4b0328c8df0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/1af22136a59ffd3b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/493d9a1b0b9b9355.png)

