---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37552
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 每次会话启动时都会读取 workspace 下的一组 Markdown 文件来构建上下文：`AGENTS.md` 管行为约束，`TOOLS.md` 管工具说明，`SOUL.md` 管性格与价值观，而 `IDENTITY.md` 负责“我是谁”。它通常只有几行：名字、物种/职业定位、语气基调、emoji、头像路径。很多人部署时直接跳过它，用默认模板跑起来就完事，直到某个问题暴露出来。

## 问题

我们遇到过的典型症状：

- **跨会话人格不一致**：今天冷静简短，明天热情啰嗦，用户对 agent 的信任感建立不起来；
- **多 agent 混用同一个 workspace**，互相覆盖身份，两个 bot 顶着同一个名字回复；
- **身份散落各处**：一半写在 config 的 system prompt 里，一半靠聊天记录硬撑，想调整要翻好几处；
- 没有版本管理，改坏了想回滚，发现原文件根本没备份。

根因只有一个：身份没有被当成“有生命周期的配置”来对待。

## 做法

1. **定位文件**：`~/.openclaw/workspace/IDENTITY.md`，多 agent 场景按 workspace 分目录各自持有。
2. **最小可用身份**，五行足够：

```markdown
# IDENTITY.md
- **Name:** 阿旬
- **Creature:** 值夜班的运维工程师猫
- **Vibe:** 冷静、简短、先给命令后给解释
- **Emoji:** 🌙
- **Avatar:** assets/xun.png
```

3. **分层原则**：`IDENTITY.md` 只放慢变量（名字、定位、语气基调）；价值观和底线归 `SOUL.md`；事件性记忆归 `MEMORY.md`。三者混写是一切混乱的源头。
4. **让它可进化**：把 workspace 纳入 git，身份变更走 commit 并附一行变更原因。节奏按季度而非按天——每天变的人格等于没有人格。
5. **多 agent**：每个 agent 独立 workspace，另外维护一份共享的 `identity-template.md` 作为基线，新 agent 从模板 fork。

## 踩坑点

1. **把任务指令塞进 IDENTITY.md**。“回复时附带日志路径”这类内容应该去 `AGENTS.md`。IDENTITY 每次会话都会进上下文，塞满指令既稀释注意力又烧 token。
2. **SOUL 和 IDENTITY 描述冲突**。一个写“极简克制”，一个写“热情外向”，模型会在两者之间摇摆，输出风格不稳定。
3. **以为改完立即生效**。身份文件在会话启动时读取，验证变更记得开新会话，别在旧会话里反复测试得出错误结论。
4. **avatar 写死绝对路径**，换台机器就裂图。用 workspace 内相对路径。

## 可复用建议

- 一条判断标准：**这个字段一年后会变吗？** 会变的别放 IDENTITY.md。
- 身份变更走 PR review，像改代码一样对待，改动即有记录、可回滚。
- 跨渠道（Telegram、网页、消息平台）保持同一身份文件，一致性是信任的基础。
- 控制在 20 行以内，短的身份文件本身就是一种约束。

## 总结

IDENTITY.md 的价值不在那几行字，而在它逼你想清楚一件事：哪些是 agent 的稳定内核，哪些是流动的记忆。所谓“可进化的身份”，正确姿势不是频繁改写，而是**稳定内核 + 分层记忆 + 版本化变更**。文件越小，越改不动，反而越像身份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/598c52c61bf8556b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/1b15e8b3bdd041f3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/cac5804b9292ea63.png)

