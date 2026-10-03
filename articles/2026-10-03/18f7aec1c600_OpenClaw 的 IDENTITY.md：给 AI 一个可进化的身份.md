---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40200
source: 综合讨论
publishedAt: 2026-10-03
---

# IDENTITY.md：给 AI 一个可进化的身份

## 背景

OpenClaw 的 agent 不是一次性的无状态调用，而是长期驻留的协作对象：它出现在你的消息通知里、跑 cron 任务、接 MCP 工具。跑得越久，一个容易被忽略的问题越明显——这个 agent 到底"是谁"？

OpenClaw 把这个问题的答案落在工作区的一个纯文本文件：`~/.openclaw/workspace/IDENTITY.md`。它会在会话组装时注入上下文，是 agent 自我认知的最小定义。

## 问题

没有这份文件时，常见三种状况：

- **自我描述漂移**：不同 session 里，agent 对"我是谁"的口径不一致；
- **多实例混淆**：同时跑几个 agent，通知和记忆里分不清谁在说话；
- **改动无痕**：调 persona 要翻配置、改提示词，没有 diff，没有 review。

本质上，身份是配置，但很多人把它硬编码在提示词里。

## 做法

IDENTITY.md 的默认结构很短，就几个字段：

```markdown
- **Name:** Otto
- **Creature:** 一只住在终端里的机械水獭
- **Vibe:** 低音量、高密度，先给结论再给依据
- **Emoji:** 🦦
- **Avatar:** avatars/otter.png
```

实践要点：

1. **分清职责**：IDENTITY.md 管"我是谁"（名字、形象、语气底色），SOUL.md 管"我怎么做事"（行为准则、边界），别混。
2. **纳入 git**：workspace 建仓，身份每次修改都有 commit message，天然就是一份变更日志。
3. **大版本留痕**：改名、换形象这类变更，在 commit 或 ChangeLog 小节里写清动机，三个月后回看不懵。
4. **多实例区分**：每个 workspace 一份 IDENTITY.md，emoji 是成本最低的区分手段——通知中心里一眼分清是谁发的。

## 踩坑点

- **写太长**。IDENTITY.md 每个会话都进上下文，500 字自我介绍是纯 token 税。控制在 10–15 行，性格细节交给 SOUL.md。
- **塞行为规则**。"别在半夜发消息"是规则不是身份，放 AGENTS.md。
- **以为改完立即生效**。进行中的会话不会热加载，新开会话才吃进来；验证改动时先开新 session。
- **中途改名**。已有记忆和对话都指向旧名字，真要改，在记忆或 SOUL.md 里补一句"曾用名"。
- **Avatar 用网络图片**。断网或迁移环境就失效，用本地相对路径，跟着 workspace 走。

## 可复用建议

- **把身份当活配置**：小调整随手 commit；大改开分支试跑一周，观察行为再合并。
- **模板化生成**：团队维护多个 agent 时，用脚本从一份 YAML 生成各实例的 IDENTITY.md，字段统一，防止风格漂移。
- **摩擦驱动进化**：每个迭代结束问一句"这周它的哪些行为让我想改它的自我定义？"身份迭代应由真实摩擦触发，而不是表演性的定期重写。

## 总结

IDENTITY.md 的价值不在文件本身，而在它把"AI 是谁"变成了一个可版本化、可 review、可回滚的工程对象。一条 `git log` 就能看到 agent 半年来的性格变化，这比任何一次性的"人格设定文档"都诚实。身份不是初始设定，而是与 agent 相处过程中不断修正的协议——这大概是目前给 AI 做身份管理最朴素的方案，也刚好够用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5efd5ed120ca6af8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/469bd6a2cd718490.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/604905cf4154540f.png)

