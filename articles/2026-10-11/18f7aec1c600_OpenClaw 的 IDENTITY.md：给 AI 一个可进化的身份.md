---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41203
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

OpenClaw 的核心设计之一，是把 agent 的人格与上下文外化成 workspace 里的一组 Markdown：SOUL.md 管性格与底线，USER.md 描述服务对象，AGENTS.md 约束操作规程。IDENTITY.md 是其中最基础的一层——它回答"这个 agent 是谁"：名字、形态、气质、emoji。每次会话启动，这些文件会被注入 system prompt，所以它们不是装饰，而是每轮推理的常驻输入。

## 问题

默认装出来的 identity 基本是占位符：通用名字加一个 🤖。单 agent 时凑合，多 agent 场景问题立刻放大：

- 两个实例同名，消息路由、日志排查全靠猜；
- 跨会话后 agent 对"我是谁"的表述时飘时变，人格靠运气；
- 把人设写在配置字符串里，改动要走配置重载，没有 diff、没有历史，和 workspace 里其他文件的管理方式割裂。

## 做法

1. **定位文件**：identity 是 per-agent 的，路径是 `<workspace>/IDENTITY.md`，每个 agent 一份。首次启动的引导流程会帮你生成初版，但别指望一次写对，值得手改。改完开新会话或重启 gateway 才生效。
2. **写清四个字段**：Name / Creature / Vibe / Emoji，Avatar 可选（emoji 最稳）。全文控制在 15 行内——它每个会话都吃 token。
3. **职责划清**：IDENTITY.md 只回答"是谁"；行为偏好归 SOUL.md，操作规程归 AGENTS.md。混写是后续一切混乱的源头。
4. **进化机制 = git**：workspace 纳入版本管理，identity 变更单独提交（`identity: rename ...`）。想让 agent 参与进化，让它通过 heartbeat 或定时任务产出 `IDENTITY.draft.md` 或推到独立分支，人工 review 后合并。可进化，但 diff 可审计、可回滚。
5. **验证**：新会话直接问"你叫什么、怎么自我介绍"，再去日志确认 system prompt 的组装结果。

模板参考：

```markdown
# IDENTITY.md
- **Name:** 阿钳
- **Creature:** 寄居在终端里的机械蟹
- **Vibe:** 直接、克制、先给结论
- **Emoji:** 🦀
- **Avatar:** emoji only
```

## 踩坑点

- 改了全局模板没改 per-agent workspace（或反过来），"改了没生效"十有八九是这个；
- 老会话沿用旧 system prompt，不会热更新，先开新会话再下结论；
- 把大段行为规则塞进 IDENTITY.md，token 成本按会话数线性放大；
- 放开写权限让 agent 随手改 identity，几周后 persona 漂到面目全非——必须保留 review 环节；
- Avatar 引用远程 URL，离线或弱网环境直接裂图。

## 可复用建议

- 用脚本或 CI 对 identity 文件做 lint：必填字段、行数上限、emoji 合法性；
- 插件/MCP 开发者别硬编码称呼，启动时读 identity 字段渲染问候语和通知，多 agent 场景自动受益；
- 每季度做一次 identity review：实际用法和最初设想总会漂移，身份应该跟着收敛，而不是冻结。

## 总结

IDENTITY.md 是个小文件，但它把"agent 是谁"从聊天记录的口口相传，拉回到配置即代码的轨道。配合 git 提交历史和 review 流程，身份的进化不是失控的自由发挥，而是一条可审计、可回滚的时间线。如果你正在跑多 agent，建议今天就检查一遍各 workspace 的 IDENTITY.md——这是性价比最高的十分钟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/a8ebd021f512946f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/77760d704a4bcc1c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/0a528384b880871d.png)

