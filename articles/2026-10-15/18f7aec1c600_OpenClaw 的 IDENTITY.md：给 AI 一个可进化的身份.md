---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41607
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

OpenClaw 把每个 Agent 的工作区当作它的“家”，除了记忆和工具配置，还有一个不起眼但作用很大的文件：`IDENTITY.md`。它通常只包含 Name、Creature、Vibe、Emoji、Avatar、Hue 几个字段，每次会话启动时被注入上下文，Agent 回答“你是谁”时就从这里取材。

听起来只是个昵称配置，但在长期运行的 Agent 场景里，身份其实是一致性的锚点。

## 问题

没有这个文件时，实践中常见三类麻烦：

1. **自我介绍漂移**。今天叫“小助手”，明天被问到又临时编一个，多渠道（Telegram / Discord / Web）下尤其明显。
2. **身份散落在 prompt 里**。人设写死在系统提示词或插件代码中，改一次要动多处，无法 review，也无法回滚。
3. **身份与能力纠缠**。角色描述和工具清单、记忆内容混在一起，调人设时容易误伤行为逻辑。

## 做法

我的实践是把 `IDENTITY.md` 当成一个版本化的配置文件来管理：

1. **最小化字段**。只保留那六个字段，每个一两句话。身份是约束，不是传记。
2. **纳入 git**。工作区本身在仓库里，身份每次变更都有 diff 和 commit message，相当于一份现成的“身份变更日志”。
3. **Agent 提案，人裁决**。定期让它基于近期对话给出一版身份修订建议（比如发现它总被问某类问题，可以补进 Vibe），我 review 后再 commit。避免它自己改自己造成循环。
4. **与相邻文件分层**。IDENTITY.md 管“我是谁”，SOUL.md 管行为气质，MEMORY.md 管事实状态。三者分离后，改动互不干扰。

## 踩坑点

- **别往里塞状态**。“当前在做的项目”这类易变内容应进 MEMORY.md，写进身份文件很快会过期并误导模型。
- **Vibe 别写太长**。超过半屏的人设描述会稀释关键指令，实测效果反而下降。
- **Avatar 与 Hue 要一致**。多渠道渲染时头像色相和 Emoji 不统一，用户会感知成“换了个人”。
- **自改必须加护栏**。让 Agent 修改身份文件时，务必走 git 提交加人工确认，否则一次会话内的反复改写很难追溯。

## 可复用建议

- 把身份文件当作产品里的品牌页：改动频率低，每次改动都有明确理由。
- 每月做一次身份 review：翻一遍 git log，删掉过时描述。
- 字段命名保持稳定。下游插件（头像渲染、消息签名等）依赖这些字段，改名就是 breaking change。
- 多 Agent 部署时，每个工作区一份独立 IDENTITY.md，不要共享。

## 总结

IDENTITY.md 的价值不在“给 AI 起名字”，而在于把身份从散落的提示词收敛成一个可版本化、可 review、可回滚的单一事实来源。工程上它只是个小配置文件，但对长期运行的 Agent 来说，稳定的身份是稳定行为的前提。先用最小字段跑起来，再按需进化，就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/6d4db68568340bdb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/1785e9e7e0460a18.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/b7e02edf68f144aa.png)

