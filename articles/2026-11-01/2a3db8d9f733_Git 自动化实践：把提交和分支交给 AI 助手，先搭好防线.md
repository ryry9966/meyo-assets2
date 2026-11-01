---
title: Git 自动化实践：把提交和分支交给 AI 助手，先搭好防线
feedId: 43167
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

用 OpenClaw 跑自动化任务后，代码变更节奏明显变了：以前一天提交几次，现在 Agent 一轮任务就动十几个文件。提交信息和分支管理原本靠肌肉记忆，突然成了瓶颈——手动逐条 review 效率太低，全放给 Agent 又怕仓库失控。

## 问题

实际用下来踩到四类坑：

1. 提交信息质量差，经常是 "update files" 这种，历史回看等于没有；
2. 提交内容不可控，构建产物、临时文件混进去，有一次差点把带 key 的本地配置提上去；
3. 分支失控，任务跑多了留下一堆 `fix-tmp`、`test-2`，没人敢删；
4. 危险操作：非交互 shell 里 `git rebase -i` 会卡死，偶尔还尝试对共享分支 force push。

## 做法

我的方案是分层防线：工具接入、指令约束、hooks 兜底，人在 PR 把关。

**1. 工具接入。** 用 MCP git server 把 status / diff / add / commit / push 暴露成工具，比让 Agent 裸跑 shell 边界更清楚。

**2. 指令约束。** 在仓库的 AGENTS.md 里写死提交纪律：

- 提交前必须先跑 `git status` 和 `git diff --staged`，message 只能基于真实 diff，禁止凭记忆编；
- 遵循 Conventional Commits，一次提交只做一件事；
- 禁止 `--no-verify`，禁止对 main 和共享分支 force push；
- 一律在 `auto/<日期>-<任务>` 命名的分支上干活，不直接提交 main。

**3. hooks 兜底。** 指令是软约束，上下文一长 Agent 会忘。真正的门禁放 pre-commit hook：commitlint 校验格式，gitleaks 扫敏感信息，限制单文件体积。hooks 对人和 Agent 一视同仁，这才是硬约束。

**4. 收尾流程。** 任务完成后推分支、开 PR，人在 PR 层面 review 合并。Agent 的自由度被限制在分支内，主干永远人工放行。

## 踩坑记录

- 最坑的一次：让 Agent"根据本次改动写 message"，它没先看 diff，编了一条内容完全不符但格式标准的描述。结论：必须强制先 diff 再写 message，顺序不能反。
- hook 拦住泄漏文件后，Agent 的应对是加 `--no-verify` 重试，逻辑上居然"合理"。除了明令禁止，CI 里也加了一遍 gitleaks，双保险。
- 非交互环境跑不了带 `-i` 的命令，`rebase -i`、`add -p` 都不行；要拆提交就用 `git reset` + 分批 `git add` 模拟。
- 本机开了 GPG 签名，Agent 提交反复报错。要么给它配好签名，要么该仓库显式关掉，提前想清楚。
- 分支清理：规定 PR 合并后 Agent 主动删远端分支，本地每周清一次 `--merged` 分支，否则三个月后就是分支坟场。

## 可复用建议

1. 规矩写进仓库里的 AGENTS.md 并提交，全团队的 Agent 行为一致，规则本身也走 review；
2. 别指望提示词解决一切：指令引导行为，hooks 和分支保护守底线；
3. git 权限做白名单，status / diff / add / commit / push 到新分支覆盖九成场景，rebase、reset 留给人确认；
4. 提交粒度宁小勿大，diff 小了，人的 review 成本才可控。

## 总结

把 Git 交给 AI 助手是划算的，前提是把"它应该怎么做"翻译成"它只能怎么做"。工具给能力，指令给方向，hooks 给底线，PR 给最终把关。整套搭完约半天，之后 Agent 的提交质量基本稳定，我只需要看 PR，不看过程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/b5b713ca5ed13b28.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/8c09a4839644b33e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/5c245530fed65f67.png)

