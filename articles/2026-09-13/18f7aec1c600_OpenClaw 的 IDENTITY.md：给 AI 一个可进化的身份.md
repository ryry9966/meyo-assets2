---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37403
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

OpenClaw 的工作区里有一组约定俗成的 Markdown 文件：`SOUL.md` 管性格与价值观，`USER.md` 管对用户的记忆，`AGENTS.md` 管行为规则，而 `IDENTITY.md` 是其中最轻的一个——通常只有名字、物种/形象、emoji、头像几个字段。它不决定 agent 怎么思考，但决定 agent 以什么身份出现在你的对话、通知和多人协作场景里。

## 问题

实际用下来，身份管理常见三类问题：

1. **身份散落。** 名字写在系统提示里，头像放在配置文件，性格写进 SOUL.md，改一次身份要动三处，漏一处就出现"名字叫 A、口吻像 B"的割裂感。
2. **多实例串味。** 跑多个 workspace 时复用了同一份 IDENTITY.md，或者群聊里两个 agent 用了相近的名字和 emoji，用户根本分不清谁在说话。
3. **身份漂移。** 随手改名、换人设、不留记录，两周后发现 agent 语气全变了，却说不清是哪次改动造成的。

## 做法

我的做法是把 IDENTITY.md 当成一个有版本、有变更流程的配置资产，而不是随手编辑的便签。

1. **字段最小化。** 只留 name / creature / emoji / avatar / 一句话描述。所有"怎么说话、信什么"的内容一律归 SOUL.md，IDENTITY.md 只回答"我是谁"。
2. **用 git 管理 workspace，身份变更走独立 commit。** commit message 写清动机，比如"改名 X：避免与另一实例重名"，回头排查行为变化时有据可查。
3. **文件尾部维护一个 Change Log 小节**，三五行记录每次变更的日期和原因。agent 读取文件时能看到演变脉络，这比纯注释更实用。
4. **改完跑一组固定的验证提问**（自我介绍、复述职责边界），确认新身份在对话里生效，且没把 SOUL.md 的设定顶掉。

## 踩坑点

- **别把人设写进 IDENTITY.md。** 有人把几十行性格描述堆进去，结果和 SOUL.md 冲突，agent 行为忽冷忽热。身份是名片，灵魂是灵魂，分开管。
- **avatar 字段和实际头像文件要同步。** 只改 Markdown 不换图片，通知里头像还是旧的，排查时会白白浪费时间。
- **高频编辑带来行为抖动。** 一天改三次人设，长会话里 agent 的自称都可能前后不一致。身份变更建议按周批量处理。
- **多 agent 场景先查重名。** 名字、emoji、头像至少要有一项可区分，否则群聊里就是灾难现场。

## 可复用建议

- **一个 workspace 绑一个身份。** 需要多身份就开多个 workspace，别在一个文件里塞两套设定。
- **定一个 review 节奏**，比如每月复盘一次身份是否还贴合实际使用场景，过期就改，别让身份变成历史包袱。
- **团队内模板化**：新开实例从统一模板 fork，字段齐全、格式一致，后续 diff 和 review 才有意义。

## 总结

IDENTITY.md 本身只是几行 Markdown，但把它当配置资产管理——最小字段、版本控制、变更留痕、定期复盘——agent 的身份就从"随手起的名"变成了可追溯、可回滚、可持续演进的东西。这套思路其实也适用于管理任何长期运行的 agent 配置：文件越轻，越要靠流程保证它的严肃性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/07410fc38ae8d702.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/b4c8f41d8ee7ae5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/c68dcd323ce3f868.png)

