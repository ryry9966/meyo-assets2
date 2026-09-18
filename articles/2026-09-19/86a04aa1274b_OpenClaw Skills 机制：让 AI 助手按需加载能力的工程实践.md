---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 38124
source: 综合讨论
publishedAt: 2026-09-19
---

## 背景

做 Agent 的同学大概都遇到过同一个问题：能力越接越多，system prompt 越来越胖。十几个 MCP 工具、几段固定流程、若干领域知识全部塞进上下文，结果是 token 成本上升、模型注意力被稀释、选错工具的概率反而变高。

OpenClaw 的 Skills 机制就是针对这个问题设计的：能力描述不常驻上下文，而是按需加载。核心思路是渐进式披露（progressive disclosure）——平时只暴露一层轻量索引，真正干活时才把完整说明载入。

## 机制拆解

Skills 采用三层结构：

1. **元数据层**：每个 Skill 的 name + description 常驻系统提示，通常各占几十 token；
2. **正文层**：SKILL.md 的完整内容，只在模型判断"这个 Skill 可能有用"时才读入；
3. **资源层**：SKILL.md 引用的脚本、模板、参考文档，需要时再进一步加载。

也就是说，模型先"看到目录"，再"翻开某一页"，而不是把整本书背下来。

## 怎么写一个能被正确触发的 Skill

目录结构很简单：

```text
skills/
  weekly-report/
    SKILL.md
    template.md
```

SKILL.md 用 frontmatter 声明元数据：

```markdown
---
name: weekly-report
description: 当用户要求整理本周工作、生成周报、汇总提交记录时使用；单日站会纪要不适用
---
（正文写具体步骤、边界条件，模板通过相对路径引用 template.md）
```

关键在 description，它决定模型会不会在正确时机加载。写法建议：

- 用"当……时使用"的句式，写清触发场景和关键词；
- 同时给出反向条件，明确"什么时候不要用"；
- 正文控制在 100 行以内，长内容拆到资源层引用。

## 踩坑点

- **描述太泛**：写"帮你处理文档"，结果所有文档请求都触发。描述越具体，误触发越少。
- **正文太长**：把几千字资料全塞进 SKILL.md，触发一次就把省下的上下文吐回去了，渐进式披露形同虚设。
- **大而全的单 Skill**：一个 Skill 塞五种流程，模型经常只执行其中一段，拆成单一职责的多个 Skill 更稳。
- **脚本无防护**：Skill 内的自动化脚本建议加 dry-run 或人工确认步骤，避免模型直接执行破坏性操作。
- **改了不生效**：注意会话内缓存问题，调试时开新会话验证，别在旧会话里反复试。

## 可复用的建议

- 一个 Skill 只做一件事，命名用动词短语（generate-changelog、triage-issue）；
- 把 description 当"触发器"写，正反条件都要有；
- 用会话日志复盘：Skill 是否在预期时机被加载？没加载就去改 description，而不是改正文；
- 团队场景下把 skills 目录纳入 code review，和业务代码同等对待。

## 总结

Skills 机制的价值不在于"能挂多少能力"，而在于把上下文当稀缺资源来管理：索引常驻、正文按需、资源再下一层。配合克制、单一职责的 Skill 设计，Agent 才能在能力变多的同时保持响应质量。建议从一两个高频流程开始拆，跑通"触发—加载—验证"的节奏后再逐步迁移，不必一步到位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-19/8a41dab1f5be3132.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-19/ab39fd695083beec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-19/c57835e4c0d604d8.png)

