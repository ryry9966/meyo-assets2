---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40679
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

在 OpenClaw 的 workspace 里，几个 markdown 文件构成了 agent 的“运行时人格”：`AGENTS.md` 管行为约束，`SOUL.md` 管价值取向，`USER.md` 记用户偏好，而 `IDENTITY.md` 只回答一个问题——**它是谁**。名字、emoji、形象、气质，通常十来行。很多人在 onboarding 时随手填完就再没打开过，但它可能是 workspace 里杠杆最高的一个文件。

## 问题

身份缺失或散落各处时的典型症状：换个会话、换个渠道，自我介绍不一致；人格硬编码在系统提示词里，改一次要动代码、重启服务，还没有 diff 可看；多 agent 共用记忆和日志时，分不清“这句话是谁说的”。

更本质的问题是所有权：身份到底属于开发者，还是属于 agent？OpenClaw 给了第三种答案——身份是一个普通文件，agent 可读，也可以在授权下自己改，于是它可以进化，而进化过程可以被审计。

## 做法

1. **定位与精简**。workspace（默认 `~/.openclaw/workspace`）下的 `IDENTITY.md`，保留 name / emoji / creature / vibe / description 几个字段，控制在 10 行以内。它每个会话都会注入上下文，多一行就多烧一份 token。
2. **划清边界**。事实类信息（名字、形象、自称）放 IDENTITY.md；行为准则放 AGENTS.md；价值观放 SOUL.md。不要把“执行前必须确认”这类策略写进身份文件。
3. **设计写入权限**。三种模式：完全手动；首次启动的 bootstrap 仪式让 agent 自己取名（想可控就先手写好再首启）；定期（比如每月）让 agent 提案、你审 diff。推荐第三种：进化，但不失控。
4. **用 git 管 workspace**。身份被改了什么、什么时候改的，`git log` 一目了然，`git checkout` 即可回滚。这是“可进化”的安全网。
5. **多 agent 场景**，每个 agent 独立 workspace，名字和 emoji 显著区分，共享记忆和日志里才不会串。

## 踩坑点

- **写成小作文**。800 字的人设每轮进上下文，成本高，还会稀释关键信息。
- **开放自改却没版本控制**。半年后发现名字变了，查无实据，只能认。
- **改完不生效**。身份在会话启动时载入，编辑对正在跑的会话不热加载，开新会话即可。
- **按渠道 fork 身份**。Telegram 一个人格、CLI 一个人格，很快失控；渠道差异应放渠道级指令，身份保持单一来源。
- **emoji 和 creature 别乱填**。它们会影响头像生成和 agent 的自称方式，不是纯装饰。

## 可复用建议

- 把身份**当数据、不当代码**：markdown + git + 最小写入权限。这套治理模式同样适用于 `USER.md`、`MEMORY.md`。
- 定一个“身份评审”节奏：每季度 diff 一次，问一句“这还是它吗”。
- 插件和 MCP 工具**不要硬编码任何 persona 提示词**，身份只从 workspace 流入上下文，避免第二事实源。

## 总结

IDENTITY.md 很小，但它是“配置”变成“自我”的那条接口线。写短一点、进 git、给 agent 一条受审计的修改通道，身份就能随实际使用自然生长，而不是上线即冻结。这大概比精心编写的人设更接近我们想要的东西：一个用得越久越像它自己的 agent。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/6364458f552c7234.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/36415cdf142ac57f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/7e61b47cdb69c7cc.png)

