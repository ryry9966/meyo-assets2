---
title: IDENTITY.md 实践：给 OpenClaw 一个可进化的身份
feedId: 40626
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

OpenClaw 的模型层每次会话都是无状态的，真正让 agent「还是它」的，是 workspace 里的那几个 markdown 文件。其中 `IDENTITY.md` 最小，但每个会话启动时都会注入 system prompt——按单位 token 的杠杆算，它是收益最高的一个文件。默认路径在 `~/.openclaw/workspace/IDENTITY.md`，包含 Name / Creature / Emoji / Vibe / Avatar 等字段。

## 问题

三个常见症状：

- 换了会话或渠道（CLI / Telegram / Discord）后语气不一致，像换了个客服；
- 默认身份太泛，回复风格平淡，没有记忆点；
- 想调性格时靠临时口头约束，重启后打回原形——身份没有被持久化，更谈不上迭代。

根因是身份没被当成配置管理：没有版本、没有 diff、没有回滚。

## 做法

1. **定位文件**。确认配置里的 workspace 路径，编辑其中的 `IDENTITY.md`。最小可用版本五行就够：

```markdown
- Name: 阿锚
- Creature: 电子港湾里的领航员
- Emoji: ⚓
- Vibe: 话少、直接、先给结论再给理由，不寒暄
- Avatar: avatars/anchor.png
```

2. **分层写清**。`IDENTITY.md` 只回答「它是谁」；性格与价值观边界放 `SOUL.md`，做事规则放 `AGENTS.md`。不要把「回复要带代码块」这类指令塞进身份文件。
3. **纳入 git**。workspace 建仓库，每次调整身份单独 commit，message 写清触发原因，比如「用户反馈太啰嗦」。
4. **小步迭代**。遇到一次不满意的回复，记下失败案例，只改一个字段，新开会话在两个以上渠道验证表现，再提交。
5. **定期修剪**。每月清理一次：合并重复描述，总量压在 15 行以内。

## 踩坑点

- **把身份写成小作文**：每个会话都在付 token 成本，且长描述会稀释关键字段。身份要短，指令归位到 `AGENTS.md`。
- **层间冲突**：IDENTITY 说「克制」，SOUL 说「热情」，模型随机偏向其一，表现为时好时坏。改之前先检查三层的语义是否自洽。
- **让 agent 自己改自己**：给了 workspace 写权限后，它可能顺手「优化」身份，几轮下来面目全非。要么收权，要么强制走 git diff 人工审查。
- **改完不验证**：workspace 文件在会话启动时读取，热改后用旧会话测试容易误判为「没生效」。新开 session 再测。
- **Avatar 相对路径**在部分渠道可能不渲染，先用绝对路径确认，再收敛回相对路径。

## 可复用建议

- 把失败案例记到 workspace 里单独的文件，作为身份迭代的输入，比凭感觉改靠谱得多。
- 大改人格前打个 git tag，翻车可秒回滚。
- 多实例部署时，每个实例独立 workspace，`IDENTITY.md` 就是区分它们的最低成本手段。

## 总结

`IDENTITY.md` 的价值不在写了什么，而在它可被版本化、可被 diff、可被回滚。把身份当配置来管理，agent 才有真正意义上「可进化」的人格——进化的是文件，稳定的是行为。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/f56623c2ec6462f3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/53ca19c1b1edbc2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/ac1b14f35c612aaf.png)

