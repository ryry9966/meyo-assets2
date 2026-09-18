---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38067
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 的 agent workspace 有一套约定文件：`AGENTS.md` 管行为规则，`SOUL.md` 定价值观和语气，`USER.md` 记用户偏好，`MEMORY.md` 做长期记忆。`IDENTITY.md` 是其中最短的一个，通常只有 Name、Creature、Vibe、Emoji、Avatar 几个字段。它会被拼进系统提示，每轮对话都在场——文件虽小，但决定了开场时模型“是谁”。

## 问题

默认身份太通用，跑久了会暴露三个痛点：

1. **多实例趋同**：工作号、生活号、项目号行为越来越像，串台时分不清谁是谁；
2. **改动无据**：身份散落在各处提示词里，没有版本记录，改坏了不知道回滚到哪一版；
3. **职责混杂**：身份和行为规则混写，prompt 常驻膨胀，每轮都在烧 token。

## 做法

**1. 从最小模板起步**，控制在 10 行以内：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 一只守夜的机器蟹
- Vibe: 冷静、话少、先给结论
- Emoji: 🦀
- Avatar: avatars/claw.png
```

Vibe 用两三个短语就够，不要写人设小作文。

**2. 用 git 管 workspace，把身份当配置代码。** 每次改动一个 commit，message 写清动机（例如“太啰嗦，收窄”），回滚就是 `git revert`。

**3. 明确分工**：IDENTITY.md 只回答“我是谁”。“我该怎么做”归 AGENTS.md，“对方是谁”归 USER.md，“发生过什么”归 MEMORY.md。发现某句话属于行为规则，就搬走。

**4. 小步进化**：跑一两周，观察输出里“不像它”的部分，diff 一处改一处，不要一次性重写。改前改后用同一组任务（同一份日报、同一段 code review）对比效果，主观但有效。

**5. 多实例隔离**：每个 workspace 一份 IDENTITY.md，名字和 Vibe 必须一眼能区分。

## 踩坑点

- **身份写太长**：它每轮常驻上下文，300 字人设等于持续 token 开销，细节越多模型越顾此失彼。
- **让 agent 自己改身份**：它该写的是 MEMORY.md。身份编辑权在用户手里，否则会出现自我膨胀式漂移。
- **和 SOUL.md 冲突**：SOUL 说简洁、IDENTITY 说热情，模型会摇摆。改一处时 grep 另一处确认一致。
- **Avatar 路径写错不会报错**，只会静默回退默认图，改完记得确认实际加载结果。
- **旧会话不生效**：改完身份，正在跑的会话仍用旧版本，验证时务必开新会话。

## 可复用建议

- 固化一份 IDENTITY.md 模板，多机/团队部署时统一字段，降低协作成本；
- 每月做一次“身份 review”，配合 `git log` 回看进化轨迹——这份 log 本身就是很好的行为调试素材；
- 想量化的话，可以写个脚本对抽样输出和 Vibe 关键词做粗匹配，衡量“身份一致性”。别追求精确，够定位漂移就行。

## 总结

IDENTITY.md 的价值不在“写得多好”，而在“改得有据”。把它当配置代码对待：最小化、版本化、职责单一、小步迭代。身份稳定了，记忆和行为才有锚点，agent 才会越用越像“那一个”，而不是每次随机重启的陌生人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/1c9bea2cfc2cb1cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/e83ced59388bf1ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9dc25d8e11309291.png)

