---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41251
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

OpenClaw 的 agent 是长驻运行的，接的是真实工作流：跑脚本、查数据、发消息、维护服务。早期常见的做法是把所有"怎么干活"的知识全堆进系统提示或 AGENTS.md，结果上下文越撑越大，模型该调工具时反而淹没在一堆无关指令里。

Skills 是 OpenClaw 给出的解法：把一项能力写成独立的 Markdown 指令文件（SKILL.md），平时上下文里只放"名字 + 一句话描述"，模型判断当前任务相关时才把完整正文加载进来。也就是所谓的按需加载、渐进式披露。

## 问题

实际用下来，主要矛盾有三个：

1. **全量注入 vs 上下文预算**。几十条 SOP 全塞进 prompt，token 成本高，注意力还被稀释。
2. **通用能力 vs 专属流程**。模型知道怎么用 git，但不知道你们团队发版要先跑哪个脚本、在哪个目录执行。
3. **MCP 工具 vs 程序性知识**。MCP 提供的是可调用的函数，但"什么时候调、按什么顺序调、失败了怎么办"这类流程知识没有地方放。

Skills 补的就是第三块。

## 做法

以一个"日志排障"技能为例：

**1. 建目录和文件**

```
~/.openclaw/skills/log-triage/
└── SKILL.md
```

**2. 写 frontmatter，重点是 description**

```markdown
---
name: log-triage
description: 排查服务日志报错时使用。当用户提到 500、超时、OOM、日志分析时触发。
metadata:
  requires:
    bins: ["journalctl"]
---
```

**3. 正文写流程，不写废话**

用编号步骤列出：确认服务名 → 抓最近 200 行 → 按 ERROR 分组 → 常见模式对照。每步给具体命令；破坏性操作（重启、清理）明确标注"需先向用户确认"。

保存后让 gateway 重新扫描技能目录，用 `/skills` 或直接问 agent"你现在有哪些技能"验证识别情况。触发测试别省：换三四种自然说法各问一遍，确认都能命中。

## 踩坑点

- **description 写得太抽象**。"辅助开发"这种描述永远不会触发。要写"触发场景 + 关键词"，它本质是给模型看的路由表。
- **一个技能塞多个职责**。日志排障和发布回滚拆开写，单个技能正文控制在几十行内。
- **假设环境存在**。技能里引用的 CLI、环境变量，用 `requires` 声明，正文第一步先做存在性检查，否则模型会硬跑然后编结果。
- **改了文件没生效**。部分版本下 gateway 会缓存技能列表，改完记得重载，别在 agent 没识别时反复调 prompt。
- **同名冲突**。工作区技能和托管技能重名时优先级容易被忽略，排障先查实际加载的是哪一份。

## 可复用建议

- skills 目录放进 git，技能就是代码，走 PR 评审。
- description 当路由规则写：一段"何时用"胜过十段"是什么"。
- 技能优先教 agent 用现成 CLI 和 MCP 工具，而不是在 prompt 里重新发明逻辑。
- 每个技能开头写两行前置条件，结尾写一行失败兜底动作。
- 定位想清楚：MCP 管能力（tools），Skills 管流程（how-to），互补而非替代。

## 总结

Skills 的价值不在"多装了几个插件"，而在于把团队里散落的操作经验沉淀成模型可按需调用的结构化知识。写好一个技能的成本，大约就是一次认真写 SOP 的工夫，但它让 agent 从"通用助手"变成"懂你们流程的助手"。建议从你最常重复解释给模型的那个流程开始，写成第一个技能。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/5dfdd5571c7e26f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/9d4d8285ce37ee81.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/ccb524f58cd8fd46.png)

