---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42467
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 的工作区里躺着几个不起眼的 markdown 文件：`SOUL.md`、`IDENTITY.md`、`USER.md`、`MEMORY.md`。会话启动时它们会被注入 system prompt。`IDENTITY.md` 管“我是谁”——名字、形态、性格基调、emoji、一句话自我介绍。默认模板只有几行，很多人第一次跑通网关就直接跳过了，结果 agent 一直顶着默认名字和默认人设干活。

## 问题

不认真对待这个文件，会遇到三个具体问题：

1. **身份漂移**：不写死，模型每次自我介绍随机发挥。跨天、跨渠道不一致，用户建立不起稳定预期，你的自动化脚本也没法靠稳定的名字和风格做事。
2. **职责混淆**：把行为规则塞进 IDENTITY.md，和 SOUL.md 打架。system prompt 里两条互相矛盾的常驻指令，模型行为会变得不可预测。
3. **不可进化**：改一次全靠手编，没有版本记录，回滚困难，多实例之间复制粘贴还容易出错。

## 做法

一份够用的最小模板：

```markdown
# IDENTITY.md

- **Name:** 小爪
- **Creature:** 收敛的终端助手
- **Vibe:** 冷静、直接、偶尔冷幽默
- **Emoji:** 🦀
- **Intro:** 一句话自我介绍，别念简历
```

落地步骤：

1. **保持短**。5–10 行以内。它是每轮都进上下文的常驻成本，不是免费存储。
2. **只放“是谁”，不放“怎么做”**。行为偏好归 SOUL.md，用户事实归 USER.md，交互历史归 MEMORY.md。
3. **用 git 管理工作区**。identity 的每次修改都是一个 commit，可 diff、可回滚、可 review。
4. **让 agent 参与进化**。定期（比如每周）让它基于 MEMORY.md 里的真实交互记录，提一份 identity 修订建议；人工审核后合并。所谓“可进化”，不是模型自己偷偷改，而是有 review 的迭代闭环。

## 踩坑点

- **写太长**：见过有人写了 40 行人设小作文，token 成本上去，重点反而稀释。身份文件是定义，不是小说。
- **多实例共用**：两个 agent 共用一份 IDENTITY.md，会在群聊里抢同一个名字。每个实例独立 workspace，identity 各管各的。
- **放任自由进化**：不给约束让它“自己长”，几轮之后普遍滑向讨好型人格膨胀。进化必须有 diff 和人工审核两道闸。
- **改名不留痕**：换了名字，MEMORY.md 里的旧称呼还在，自我指涉混乱。改名时同步全局搜一遍记忆文件。

## 可复用建议

- 一个判断标准：**这段话如果换一个 agent 也成立，就不该出现在 IDENTITY.md 里**。身份是排他的。
- 每次改完，开一个全新会话问“你是谁、你怎么做事”，验证注入真的生效，而不是只看文件改没改。
- 身份稳定是自动化的前置条件。如果你的 webhook、路由脚本依赖 agent 名字或固定开场风格，身份漂移就是生产事故，按这个级别对待它。
- 把 identity 修订做成例行事项，绑在心跳任务或周报流程上，而不是想起来才改。

## 总结

IDENTITY.md 是个很小的文件，但它把“随机发挥的自我介绍”变成了一个有版本、可 review、持续迭代的资产。顺序很重要：先让身份稳定下来，再谈进化。稳定是进化的地基，不是对立面。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/5113395b3aa50ce0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/927bc73c626ab0c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/53cf4c8385273901.png)

