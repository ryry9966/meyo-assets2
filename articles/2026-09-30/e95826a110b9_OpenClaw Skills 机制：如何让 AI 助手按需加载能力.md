---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 39887
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 的 Agent 长期跑在常驻 Gateway 里，能力来源大致三类：内置工具、MCP 服务器、Skills。前两类解决"能不能做"，Skills 解决的是"什么时候让模型知道能做"。

核心机制是渐进式披露（progressive disclosure）：启动时只把每个 skill 的 `name` + `description` 注入系统提示词，正文在模型判断任务相关时才加载。这就是上下文经济学——常驻 prompt 里塞几十个工具定义和几十行 skill 摘要，差距是几千到几万 token，而且后者直接拖累工具选择的准确率。

## 问题

我早期的做法是把所有操作说明全堆进 AGENTS.md：发布流程、设备控制、报表生成……结果上下文膨胀，模型在无关任务里也容易被这些指令带偏。MCP 全量注册工具是同一个病：工具一多，选错的概率明显上升。缺的是一个"按需分发"的层。

## 做法

一个最小可用的 skill 就是一个目录加一个文件：

```
~/.openclaw/workspace/skills/release-helper/SKILL.md
```

SKILL.md 分两段。frontmatter 管元信息：

```yaml
---
name: release-helper
description: 打 tag、生成 changelog、发布 GitHub Release 时使用。
metadata:
  requires:
    bins: ["gh"]
    env: ["GITHUB_TOKEN"]
---
```

正文写操作步骤，能短则短。三个要点：

1. **description 是给模型看的触发广告**。写"什么场景该用"，而不是"这是什么"。
2. **用 `requires` 做门控**。binary 缺失、环境变量没配时，skill 直接不进候选列表，避免运行时才发现依赖坏了。
3. **正文超过几百行就该拆**。SKILL.md 只留索引，细节放同级文件，让模型按需读取。

调试用 `openclaw skills list` 看哪些 skill 被加载、为什么被禁用；`openclaw skills info <name>` 看单个 skill 的解析结果。验证触发是否可靠，最直接的办法是新开一个会话，用一句真实任务描述去问 agent，观察它是否命中对应 skill。

## 踩坑点

- **description 太抽象**（比如"辅助开发"），模型几乎不会触发。要写成"当用户要 X 时使用"。
- **技能职责重叠**，模型随机挑一个。合并或划清边界，比在 description 里加更多限定词有效。
- **把密钥写进 SKILL.md**——正文会进上下文，等于明文广播。密钥只放环境变量，frontmatter 里只声明依赖。
- **把"始终生效"的规则做成 skill**。回复风格、语言偏好这类常驻约束该放 AGENTS.md；skill 的语义是"按需"，不是"始终"。

## 可复用建议

- 一个 skill 一件事，命名用领域名词或动词短语，方便人和模型都对齐。
- 正文控制在 500 行以内，长文档拆成引用文件，让加载分层。
- 用 git 管理 `workspace/skills`，skill 也是代码，改动要可回溯。
- 每次新增 skill 做一次"冷启动测试"：新会话、只给任务描述，看触发率。触发不稳定就回来改 description，而不是改正文。

## 总结

Skills 本质上是给模型做上下文的按需分发：常驻的只有目录，详情在任务匹配时才进窗口。写好 description、用 `requires` 门控依赖、控制正文长度——这三件事做到位，agent 的工具选择准确率和 token 开销都会有可感知的改善。它不解决能力本身的问题，但解决了"能力怎么被恰当地知道"的问题，这在长时运行的 agent 里往往更关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/a2773c8f954b1b56.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/fecc2867f071b913.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/46b4425df7a9e2d2.png)

