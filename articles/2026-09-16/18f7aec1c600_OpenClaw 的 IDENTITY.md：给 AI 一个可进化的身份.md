---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37772
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 的 workspace 里有一组不起眼的小文件：`AGENTS.md` 管行为，`SOUL.md` 管价值观，而 `IDENTITY.md` 管的是最基础的一件事，"我是谁"。它默认只有十几行：name、creature、emoji、vibe、avatar。很多人初始化完就再没打开过，但我用下来发现，这是整个 workspace 里投入产出比最高的一个文件。

原因是它在会话启动时会被注入 system prompt。你写什么，模型就用什么自称、什么语气、什么情绪基线说话，跨 Telegram、Discord、CLI 全部生效。

## 问题

不管理身份文件，实际会遇到三类问题：

1. **口吻漂移。** 长 session 里人设随上下文滑动，昨天冷静克制，今天满屏感叹号。
2. **多 agent 分不清。** 跑两三个实例后消息混在一起，用户只能靠猜。
3. **跨渠道不一致。** 同一个 agent 在 CLI 里像工程师，在群里像客服。

反过来，写太满也有代价：prompt 膨胀、人设过拟合（每句话都强行加语气词），以及改一次身份，老用户就觉得"换人了"。

## 做法

我的实践流程，供参考：

1. **从模板起步，控制在 15 行内。** name 一个词，creature 一句话，emoji 一个，vibe 两三条形容词，avatar 指向一张固定图片。
2. **用 git 管理身份。** workspace 本来就该进 git，`IDENTITY.md` 的每次修改走 commit，写清动机（如"反馈语气太跳，收敛一档"），随时可回滚。
3. **按需演进，而不是定期乱改。** 出现新职责才补：比如开始接手 cron 任务汇报，就在 vibe 里加一条"汇报直接给结论"。其余时间不动。
4. **多 agent 各自独立。** 每个实例独立 workspace，用不同 creature 和 emoji 做视觉区分，heartbeat 消息里自然带出"签名感"。
5. **和 SOUL.md 划清边界。** IDENTITY 管"你是谁"（名字、形象、语气），SOUL 管"你怎么决策"（边界、优先级）。写过界，两边都会失效。

## 踩坑点

- **vibe 写成小作文。** 三段话的人设会被模型过度演绎，回复淹没在语气词里。形容词要少而具体。
- **复制模板没改 emoji/creature。** 两个 bot 长得一模一样，多 agent 排查时极其痛苦。
- **让 agent 自己改身份且不审查。** 自我修改叠加幻觉，漂移几乎不可逆。要改，人必须过一遍 diff。
- **avatar 写了路径但没放文件。** 部分渠道渲染报错，更常见的是静默失败，很难查。
- **把身份文件当记忆用。** 用户偏好、项目上下文请去 `USER.md` / `MEMORY.md`，混写会让 `IDENTITY.md` 膨胀失控。

## 可复用建议

- 最小模板够用：`name / creature / emoji / vibe(≤3 条) / avatar`，别贪多。
- 身份变更走 PR，或至少 commit + diff review，把它当 config-as-code 对待。
- 每 2–4 周回顾一次，只问一个问题：这个身份还配得上它现在的职责吗？只跑定时任务的 bot 不需要丰富人设；直面用户的助手值得认真打磨。
- 改动小步走：一次只调语气或只调形象，方便归因。

## 总结

`IDENTITY.md` 的价值不在"写得多好"，而在"有没有被当作一份需要长期维护的配置"。它很便宜——十几行 Markdown；它也很贵，决定了用户每次交互感知到的那个"谁"。给它 git 历史、给它审查流程、给它克制的演进节奏，身份才会稳定地长成你想要的样子，而不是随机漂成某个陌生的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/345edf5da78ae608.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/eb1497abbe95c4fc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/b5d3d1d49b744a5f.png)

