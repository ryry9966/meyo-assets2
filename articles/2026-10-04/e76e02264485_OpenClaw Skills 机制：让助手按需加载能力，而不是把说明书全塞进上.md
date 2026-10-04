---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是把说明书全塞进上下文
feedId: 40465
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

Agent 长期运行时有一个现实约束：上下文窗口有限，而"能力手册"会越积越多。早期常见做法是把所有操作说明、命令模板全写进系统提示词，结果 prompt 膨胀到几千 token，真正用到的不到两成。

Skills 机制就是针对这个问题的解法。它的本质不是代码插件，而是带元数据的 Markdown 指令文件（SKILL.md），核心是渐进式披露：会话开始时只注入每个 skill 的名称和一句描述（每个几十 token），正文由 agent 判断相关后再自行读取。

## 问题

没有按需加载时，痛点很具体：

- 无关指令稀释注意力，模型更容易跑偏；
- 能力越多，system prompt 越臃肿，成本和延迟同步上涨；
- "某工具什么时候该用、流程怎么串"这类知识没有合适的承载位置——MCP 解决了工具接入，但使用策略缺少归宿。

## 做法

1. **建目录**。托管级放 `~/.openclaw/skills/<name>/SKILL.md`，项目级放 `<workspace>/skills/`，后者建议直接进 git。
2. **写 frontmatter**。`name`、`description` 必填；`requires` 声明依赖（env 环境变量 / binaries 可执行文件 / config 配置项）。缺失依赖时该 skill 会被门控标记为不可用，而不是运行时报错。
3. **description 按触发条件写**。不要写"GitHub 操作工具"，要写"当用户要求创建 issue、查询 PR 状态或批量打标签时使用；不适用于代码 review"。agent 靠这句话决定是否加载。
4. **正文保持精简**。步骤、精确命令、护栏、期望输出格式，控制在百行以内；更细的参考文档拆成子文件，正文写明路径让 agent 二次按需读取——渐进式披露贯彻到底。
5. **验证**。`openclaw skills list` 确认加载状态和依赖门控；开新会话，用一个应触发、一个不应触发的请求各测一遍。

## 踩坑点

- **description 写成文档标题**：永远不会被命中。它是给 agent 看的路由信号，不是给人看的简介。
- **大杂烩 skill**：十件事塞进一个 skill，一旦触发整份加载，等于没有按需。
- **密钥写进 SKILL.md**：文件内容会进上下文。用 `requires.env` 声明，值走环境变量。
- **误以为 skill 会自动执行**：agent 是自主决策。如果普通 bash 也能完成任务，而描述里没写出差异化价值，它可能直接绕过 skill。
- **requires.binaries 只查 PATH**：npx、docker 包装的工具查不到，要在正文里写清实际调用方式。

## 可复用建议

- description 用固定三段式：何时使用 / 覆盖范围 / 不适用场景。
- skill 与 MCP 分工：MCP 提供 what（工具本身），skill 约定 when 和 how（使用策略）。
- 定期翻会话日志，统计哪些 skill 真正被触发，砍掉僵尸条目——每条描述都在常驻消耗 token。
- 把 SKILL.md 当提示词工程产物对待，进 code review，而不是当随手笔记。

## 总结

Skills 的价值不在"多"，而在"准"：描述写得越像路由规则，agent 的加载决策越可靠。把常驻上下文压到最小、把细节后置到按需读取，是让助手能力持续增长而不失控的关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/80d575762da7da36.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/1450b72329e5061e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/376ca04e2f8e8fac.png)

