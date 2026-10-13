---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 41408
source: 综合讨论
publishedAt: 2026-10-13
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

OpenClaw 的 agent 每次会话启动时，会把 workspace 里的一组固定文件注入上下文：`AGENTS.md` 定义行为规范，`SOUL.md` 定义人格，`USER.md` 回答一个更基础的问题——**对面这个人是谁**。很多人把精力花在前两者上，USER.md 却常常是空的，然后抱怨 agent 不懂自己。

## 问题

大模型的长期记忆检索是概率性的。你三个月前随口提过"我用 fish 不是 bash"，这条信息可能躺在记忆库里，但下次不一定被召回。于是 agent 反复问同样的问题：时区、主力语言、包管理器、命名习惯。

USER.md 的价值在于**确定性**：它是每轮必注入的静态上下文，不依赖召回，不需要祈祷。

## 怎么做

1. 在 workspace 根目录创建 `USER.md`（默认在 `~/.openclaw/workspace/` 下）
2. 分区写事实，控制在 60–80 行以内
3. 用 git 管起来，每次改动走 diff
4. 让 agent 参与维护：说"把这个偏好记进 USER.md"，让它给出 diff，你审核后落盘

一个能直接抄的骨架：

```markdown
# USER.md
## 基本信息
- 称呼：老周；时区：Asia/Shanghai；中文交流，术语保留英文
## 技术环境
- macOS + fish shell；brew / pnpm；主力栈：TypeScript + Python
## 偏好
- 先给方案和取舍，确认后再写代码；回复直给结论
## 边界
- rm -rf、生产环境变更、对外发布：必须先确认
```

## 踩坑点

- **写太长**。它是每轮固定 token 开销，也会稀释注意力。超过一屏就该裁剪。
- **放错层**。USER.md 放"你是谁"这类慢变信息；进行中的任务属于 `MEMORY.md`。把本周待办写进 USER.md 是最常见的错位。
- **留过期事实**。因为是系统级注入，agent 对它信任度很高，一条过期的"我不用 Docker"比没有更糟。建议每月 prune 一次。
- **写敏感信息**。这份文件会随上下文发给模型服务商，也会进你的 git 仓库——密钥、住址、证件号一律不要出现。
- **免审直写**。记忆漂移就是这么来的：agent 自己改、小错累积，最后变成"它以为的你"。所有写入必须过 diff 审核。

## 可复用建议

- **一屏原则**：装不下一屏就说明你把快变信息混进来了，拆出去。
- **只写事实和规则，不写叙事**："回复直给结论"，而不是"我是个喜欢简洁的人，因为……"。
- **三层分工**：USER.md（用户画像）/ AGENTS.md（行为规范）/ 项目级配置（项目上下文），职责不重叠，不互相覆盖。
- **约定维护口令**：比如 `/remember` 触发 propose-diff 流程，把维护变成固定动作而不是随口一说。

## 总结

USER.md 是成本最低、收益最确定的 agent 个性化手段。它不智能——就是一份每轮必读的 markdown——但正因为确定，才可靠。花二十分钟写好第一版，之后当配置文件维护，agent 的"懂你"就会从玄学变成工程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/725275630ba46277.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/b9cc7459542e1733.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/4dda7502a4b05513.png)

