---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42614
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

Agent 用得越深，一个矛盾越明显：能力要丰富，上下文却有限。早期常见做法是把所有操作说明、工具文档、领域知识全塞进 system prompt——结果 token 成本直线上升，关键指令被淹没在噪音里，模型表现反而下降。

OpenClaw 的 Skills 机制是对这个问题的一个工程化回答，核心思路一句话：**把"知道有这件事"和"知道具体怎么做"拆开，后者按需加载。**

## 机制简述

每个 Skill 是一个目录，核心是一个 `SKILL.md`：

```markdown
---
name: log-triage
description: 排查网关日志中的异常。当用户提到"日志、报错、连接失败"时使用。
metadata:
  emoji: 🧾
  requires:
    bins:
      - jq
---
（正文：具体排查步骤、常用命令、输出解读）
```

Agent 启动时，上下文里只注入各 skill 的 name + description 组成的紧凑清单；判断某条相关时，才通过工具读取完整正文。这就是渐进式披露——常驻开销是每条一行，几百行的正文只在真正用到时才付费。

## 实践步骤

1. **选位置**：个人能力放 `~/.openclaw/skills/<name>/`，项目相关流程放工作区 skills 目录，随仓库同步给团队。
2. **写 description**：这是决定触发的唯一入口，写清楚"做什么 + 什么时候用"。推荐触发条件句式："当用户提到 X / 需要做 Y 时使用"。
3. **控制正文**：几百行以内，步骤化、可直接复制的命令优先，别写成散文。
4. **声明依赖**：`requires` 里列出可执行文件和环境变量，缺依赖时 agent 能提前判断，而不是跑到一半才失败。
5. **验证**：重载后确认清单已注入，再用一句真实请求测触发时机，而不是只看列表在不在。

## 踩坑点

- **description 太泛或太窄**。"处理文件"这种描述会导致不该触发时也触发；反过来，只写团队黑话，该用的时候模型想不起来。把它当 prompt 迭代，看一周实际触发记录再调。
- **把 SKILL.md 当配置文件**。正文会被读进上下文，密钥、内网地址、token 都别写进去，敏感值走环境变量。
- **一个 skill 塞多个主题**。大而全的 skill 触发率反而低，一个目录只解决一类问题，宁可拆小。
- **和 MCP 职责不清**。MCP 管"能不能调"（工具接口），Skill 管"该怎么调、调完怎么处理"（流程知识）。在 skill 里重复罗列工具 schema 是浪费上下文。
- **改了没重载**。以为生效了，实际跑的还是旧版。改完重载再验证，别省这一步。

## 可复用建议

- SKILL.md 只做索引：长参考资料拆成同级文件，正文里写明"需要时读取 xxx.md"，进一步压缩常驻成本。
- 团队协作时把通用 skill 收敛进一个仓库，review description 的改动要像 review prompt 一样认真。
- 新增 skill 前先问：这件事值不值得常驻一条清单？一年用两次的流程，让模型现场推理可能更划算。

## 总结

Skills 不是一个炫技特性，本质是上下文预算管理：常驻部分越瘦，模型注意力越集中，成本越低。写好一条 description，比堆十个功能更影响实际体验。建议从把你最常重复口述的那个操作流程落成第一个 skill 开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/904402e92e366659.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/c693f687b7f2c135.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/5d70e9af9cfd827d.png)

