---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38774
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 agent 不是靠一段写死的系统提示词跑起来的，而是靠 workspace 里那几份 Markdown：`SOUL.md` 管性格与价值取向，`AGENTS.md` 管行为规则，`USER.md` 管你对它说了什么，而 `IDENTITY.md` 管的是"我是谁"。每次会话启动，这些文件会被注入 system prompt，相当于给 agent 一张随身名片。

## 问题

很多人装完 OpenClaw 就用默认身份跑：名字是默认的、语气是通用助手味、隔几天自我介绍还会变。用一段时间后你会想调整——改个称呼、收敛一下 emoji 频率、让语气更干练——但改动散落在各处，改完一次会话就"忘"，或者改过头，agent 说话突然变得油腻。身份没有版本、没有边界、没有进化路径，这和配置管理没做好的服务一样不可预期。

## 做法

1. 定位 workspace（默认在 `~/.openclaw/workspace`），找到或新建 `IDENTITY.md`。
2. 用最小字段集起步：

```markdown
# IDENTITY.md
Name: 阿钳
Creature: 一只务实、话少、先给结论的机械蟹
Emoji: 🦀
Color: #1E90FF
Avatar: assets/crab.png
```

3. 控制篇幅在 10–15 行以内。全文会进上下文，多一行就多一份 token 成本。
4. 把 workspace 纳入 git，每次调整身份都 commit，`git diff` 就是身份的进化史。
5. 隔几个会话观察一次：自我介绍是否一致、语气是否符合预期。不符合就改文件，而不是在对话里反复纠正。

## 踩坑点

- **身份写成小作文**：塞几百行人设进去，挤占上下文，行为反而飘。身份文件是名片，不是传记。
- **和 SOUL.md 抢地盘**：价值观放 SOUL.md，名字和形象放 IDENTITY.md。混写会让注入的提示词内部冲突。
- **把用户事实写进身份**："用户是后端工程师"属于 `USER.md` / `MEMORY.md`，写错位置会让 agent 分不清"我是谁"和"你是谁"。
- **放任 agent 自改身份文件**：开放 workspace 写权限后，agent 可能在对话里"顺手"改 `IDENTITY.md`，造成身份漂移。关键文件建议设只读，或定期 `git diff` 审计。

## 可复用建议

- 把 IDENTITY.md 当配置文件而不是日记：小步提交、定期 review。
- 一个够用的公式：**身份 = 名字 + 一句形象 + 视觉三件套**，其他一律不进。
- 多实例部署时（手机、家用机、VPS 各挂一个），共享同一份 IDENTITY.md，各自维护 MEMORY.md——同一人格，不同记忆。
- 每月看一次 git log：身份两个月没变，多半是没用起来；一天变三次，说明当初没想清楚。

## 总结

IDENTITY.md 的价值不在于让 agent 更像人，而在于把"这个 agent 是谁"从临时的提示词碎片，变成可版本化、可审计、可迭代的工程资产。身份稳定，行为才可预期；行为可预期，自动化才敢放心交给它。从 10 行文件开始，让它跟着你的使用习惯慢慢进化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/46cbb3a00068353f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/66bbaaa5e8fbe271.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/8f48e607ce1c84b4.png)

