---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 43222
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 的 agent 默认带一套固定的系统提示词和工具集。能力一多，最直接的问题是上下文：你不可能把所有操作手册都塞进 system prompt——塞进去后 token 成本高，且无关指令会稀释模型注意力。

Skills 就是 OpenClaw 对这个问题的回答。每个 Skill 是一个文件夹，核心是一个 `SKILL.md`：YAML frontmatter 声明 `name` 和 `description`，正文是具体操作指令。关键机制是**渐进式披露**：平时只有各 Skill 的名称和一句话描述进入提示词，正文只有在任务真正命中时，agent 才会去读。这就是"按需加载能力"。

## 问题：先分清 Skill、MCP、插件

社区里最常见的混淆是把三者混用。我的判断标准：

- **知道某件事怎么做**（流程、约定、避坑步骤）→ Skill，它只是说明书，不新增工具
- **要调用外部服务/API** → MCP server
- **要改运行时行为、加原生工具** → 插件/配置

把 API 封装硬写成 Skill 是典型错位：Skill 正文只能"教" agent 用它已有的工具（比如 shell）完成任务。

## 做法

1. 在 `~/.openclaw/skills/`（或 workspace 的 `skills/`）下建目录，如 `github-release/`；
2. 写 `SKILL.md` frontmatter，description 回答"什么时候用我"，而不是"我是什么"：

```yaml
---
name: github-release
description: 当需要打 tag、生成 changelog、发布 GitHub Release 时使用。触发词：发版、release。
---
```

3. 正文写可执行步骤，优先复用 agent 已有工具；辅助脚本放同目录，用相对路径引用；
4. 有依赖时在 `metadata` 里声明 `requires`（binaries / env / os），让 Skill 只在环境满足时可见；
5. 验证：开新会话，问一个应命中的问题，确认 agent 读取了正文而不是凭记忆作答。

## 踩坑点

- description 写成"这是一个发布工具"而不是"当你要发布时使用"——前者几乎永远不触发；
- 把所有 Skill 设成常驻加载，等于废掉按需机制，context 照样爆炸；
- 正文假设了 `gh`、`jq` 等不存在的依赖，要么声明 requires，要么写 fallback；
- 硬编码绝对路径，换台机器就挂；
- 多个 Skill 描述高度重叠，agent 会随机挑一个，保持职责唯一；
- 改完 SKILL.md 在旧会话里验证，旧版本可能已被缓存。

## 可复用建议

- 一个 Skill 只解决一类任务，正文控制在几十行；超了就拆成 reference 文件，需要时再读；
- description 模板：`当 <具体任务/触发词> 时使用。产出 <结果>。`
- 团队 SOP（发布、值班、数据导出）沉淀成 Skill，比往系统提示词里堆规则划算得多；
- 定期清理：长期没被命中的 Skill，合并或删除。

## 总结

Skills 的核心价值不是"给 agent 加能力"，而是控制上下文成本：把说明书和工具分开、把常驻和按需分开。描述写得准，agent 找得到；正文写得短，加载才便宜。这两点做到位，长会话下的行为稳定性会明显改善。建议从把一个高频 SOP 迁移成 Skill 开始，跑两周再评估触发率，比一次性堆十几个 skill 靠谱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/cf8bb686496ed55c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/8d94110f00790d62.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/63e1bf302eae69e3.png)

