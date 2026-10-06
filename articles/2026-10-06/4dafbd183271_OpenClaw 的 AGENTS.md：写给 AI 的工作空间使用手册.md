---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 40647
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

如果你同时用 OpenClaw 的主 agent、MCP 工具和各种插件跑同一个仓库，大概率遇到过同一个尴尬：每个 AI 都要重新"摸一遍"这个项目。构建命令是什么、测试怎么跑、哪些目录是生成物不能碰，这些信息全靠口头喂。AGENTS.md 是为解决这个问题而生的社区约定：在工作区根目录放一个 Markdown 文件，agent 每次新会话自动读取，作为它的"工作空间使用手册"。它跟着仓库走、能进 Git、能过 code review，天然比写在系统提示词里可维护。

## 问题

没有这份手册时，典型失败模式是：agent 在 pnpm 项目里跑了 npm；不知道 `tests/` 里的集成测试要连数据库，改完代码直接跑全量测试然后连环报错；重构时顺手动了 `generated/`。这些不是模型能力问题，是上下文缺失。团队场景下更麻烦：每个人教 agent 的方式不一样，同一个仓库里 agent 行为漂移，出问题很难判断是"规则没写"还是"规则没被读到"。

## 做法：五步搭起来

1. **建文件**：在 OpenClaw 工作区根目录新建 `AGENTS.md`，确认 agent 配置的 workspace 指向该目录，新开会话生效。
2. **只写四类内容**：项目一句话概览；可直接复制执行的命令（build/test/lint 的准确写法）；硬性约束（禁止触碰的目录、必须遵守的风格）；不确定时的行为（先问，不要猜）。
3. **分层**：子目录可放局部 AGENTS.md，就近覆盖根目录规则。monorepo 里给 `packages/api` 单独写一条"改接口前先对照 openapi.yaml"。
4. **验证生效**：在会话里问 agent "你读到了哪些工作区约定"，或翻会话日志确认文件内容进了上下文。
5. **迭代**：agent 每犯一个新错，就把对应规则回写成一行。这份文件是活的。

一个精简到能贴进 PR 的示例：

```markdown
# AGENTS.md
- Node 20 + pnpm，禁止使用 npm/yarn
- 测试：pnpm test（全量）；单包：pnpm --filter <pkg> test
- generated/、dist/ 为构建产物，禁止手改
- 提交信息遵循 Conventional Commits
- 端口 3000 被占用时改用 .env 里的 PORT，不要 kill 进程
```

## 踩坑点

- **写太长**。超过两三屏，关键规则被淹没，token 也贵。手册的价值在"被读到并执行"，不在全。
- **写软建议**。"尽量写好注释"没有可判定性，agent 会直接忽略。要写成"导出函数必须带类型注解"。
- **命令过时**。AGENTS.md 会随重构腐烂，agent 会忠实执行错误命令然后连环失败。把它当代码：改动进同一个 PR。
- **规则冲突**。根目录说 pnpm、子目录说 npm，agent 不一定按你以为的优先级裁决。保持单向覆盖，且不要重复声明同一条规则。
- **写入敏感信息**。文件内容会进模型上下文，API key、内网地址绝不能出现。
- **高估边界**。AGENTS.md 管约定和禁令，复杂流程（部署、数据迁移）应沉淀为 skill 或 MCP 工具，不是往 Markdown 里堆步骤。

## 可复用建议

- **错误回写法**：每次 agent 犯错且根因是缺上下文，就加一条规则；如果三条规则都治不好同一个错，说明该写工具了，而不是继续堆文档。
- **起步模板不超过 15 行**，宁可少而准。
- **加个 10 行的 CI 脚本**，解析 AGENTS.md 里的命令逐个跑 `--help`，防止命令悄悄失效。
- **一份文件多端共用**：AGENTS.md 是跨工具的开放约定，各家 CLI agent 都认，不必每个工具单独维护一套配置。

## 总结

AGENTS.md 是我目前性价比最高的 agent 治理手段：一个文件、十几行、进版本库，就把"口头调教"变成了"工程约定"。它的上限取决于你回写规则的勤快程度——把它当代码维护，agent 的下限会稳定很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/035095ac7cd075e7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/4cd4456e630c9720.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/abb0691810d7133f.png)

