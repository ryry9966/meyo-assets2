---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37833
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

用 OpenClaw 一段时间后，能力来源会越积越多：几个 MCP 服务器、一批 CLI 工具、自己写的自动化脚本。最初的直觉是把所有说明和流程都写进系统提示词或 `AGENTS.md`，结果上下文被大量与当前任务无关的内容占满——token 费用上去了，模型选工具反而更容易选错。

Skills 机制就是针对这个问题的：**能力不常驻，按需加载**。

## 原理：三层渐进式披露

每个 Skill 是一个目录，核心是一个 `SKILL.md`：

```markdown
---
name: video-subtitle-cleanup
description: 清理 SRT 字幕文件中的时间轴错位和重复行。Use when user 提到字幕、srt、时间轴对齐。
---
（正文：具体操作步骤、命令、注意事项）
```

加载分三层：

1. **常驻层**：只有 `name` + `description` 进入上下文，单个成本在几百字节级别；
2. **正文层**：模型判断当前任务匹配某条 description 时，才读取完整 `SKILL.md`；
3. **附件层**：正文里引用的脚本、参考文件，真正用到时才打开。

这样装 30 个 Skill，平时上下文里只有一份目录索引。

## 实操步骤

1. 在工作区 `skills/` 目录（或 `~/.openclaw/skills/`）下建子目录，名字用小写连字符；
2. 写 `SKILL.md`，frontmatter 至少包含 `name` 和 `description`；
3. 正文只写**操作过程**：流程、命令、边界情况，不写背景故事；
4. 大段参考内容（长命令表、配置样例）拆成独立文件，正文里用相对路径引用；
5. 用 `openclaw skills list` 确认可见，然后实测：问一个应该触发它的问题，观察是否加载。不触发，回去改 description。

## 踩坑点

- **description 写给人看，不是写给模型看**。要包含触发条件和关键词，"Use when…" 句式比形容词有用得多。这是最常见的失效原因：skill 永远不被触发。
- **description 太宽泛**，比如"处理文件"，会导致几乎每个任务都加载它，等于没做按需。
- **把所有东西塞进正文**，本质上只是把 system prompt 挪了个地方，没省任何预算。
- **与内置 Skill 职责重叠**，模型面临两个相似候选时路由会变得不稳定。先查重再命名。
- **只装不清理**。Skill 的 description 常驻上下文，装 50 个从不审计，成本是隐性的。

## 可复用建议

- 把 description 当作**检索入口**来打磨：写清触发场景、排除场景、关键词；
- 正文控制在 150–200 行内，更长的内容一律拆附件；
- Skill 适合承载**过程性知识**（"怎么做 X"），事实性数据（常量、清单）放普通文件更合适；
- 把 skills 目录纳入 git，改动可回滚，也能看出哪些长期没动过；
- 定期让助手列出当前可用 skills，删掉一个季度没触发过的。

## 总结

Skills 的价值不在"多"，而在"准"。整个机制可以压缩成一句话：**description 是唯一常驻的成本，所以把路由写清楚；正文是按需的代价，所以把手册写精简**。按这个原则组织能力，工具越多，上下文反而越干净。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/2174fd0463c67301.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/1cb15597fd68afd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/fecd3649dbec2618.png)

