---
title: 让 Agent 接管 Git 提交与分支：一套可落地的自动化做法
feedId: 41402
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

日常开发里，Git 操作有相当比例是纯重复劳动：写 commit message、把一坨改动拆成几个提交、按规范命名分支、清理已合并的旧分支。这些活规则明确、上下文局部——看一眼 diff 和 log 就能判断——恰好是带 shell 或 MCP 能力的 agent 最擅长的。OpenClaw 这类能直接调工具的助手，理论上完全可以接管这部分。

## 问题

实际让它动手前，有三个现实障碍：

1. **message 质量不稳定**：人自己都常写 `update`、`fix`，agent 如果没有明确规范，只会更笼统。
2. **分支状态黑盒**：几十个 `feat/*` 分支，哪些已合并、哪些还活着，没人记得清。
3. **写操作风险**：agent 有 shell 就意味着可能执行 `push --force`、`reset --hard`，一次误操作就是事故。

## 做法

**第一步，收窄工具面。** 不给裸 shell，用 MCP git server 或封装一个白名单脚本，只暴露 `status` / `diff` / `log` / `add` / `commit` / `branch` / `checkout` / `merge --ff-only`。`push` 单独走人工确认，危险命令根本不暴露。

**第二步，规范进仓库。** commit 格式、分支命名（`feat/xxx`、`fix/xxx`）写进 CONTRIBUTING.md，系统提示只负责让 agent 先读文档再动手。规范更新不用改提示词。

**第三步，提交流程固定为三段式：**

```
status + diff → 判断是否拆分 → 按 hunk 分组 add
→ 逐个生成符合规范的 message → 列出待执行命令
→ 人工确认 → 执行
```

关键是"先列计划再执行"，不搞一步到位。

**第四步，分支清理做成周期任务。** 每周让 agent 跑一次 `branch --merged` 加远端状态比对，输出待删清单，确认后删除。合并统一走 PR 或 `--no-ff`，保留回滚线索。

## 踩坑点

- **白名单优于黑名单**。别想着靠 prompt 劝阻 agent 别跑危险命令，直接不给权限才是可靠的。
- **`git add -A` 是密钥泄漏高发区**。agent 很容易把 `.env`、调试脚本一起提交。pre-commit 里放 secret 扫描，这是底线，别指望提示词约束。
- **大 diff 一次提交，message 必然空洞**。强制 agent 先出拆分方案，单个 commit 只承载一个意图。
- **detached HEAD 下提交会丢**。执行写操作前先校验当前分支状态。
- **人和 agent 抢同一个工作区会打架**。用 `git worktree add` 给 agent 划独立工作区，最省心。
- 中文 message 注意终端编码，否则 CI 日志里全是乱码。

## 可复用建议

- 把"允许的 git 命令清单"做成独立配置文件，团队共用一份，agent 启动时加载。
- 所有写操作走"计划 → 确认 → 执行"，执行日志落盘，出问题可审计。
- hook 是最后防线：`commit-msg` 校验格式、`pre-commit` 扫密钥，agent 再聪明也绕不过 hook。
- 拆分提交通常比生成 message 更有价值，优先让 agent 干这件事。

## 总结

AI 管 Git 的价值不在"全自动"，而在把规则明确的部分自动化、把需要判断的部分留给确认环节。工具面收窄、规范进仓库、三段式执行——这三件事做到位，agent 管提交和分支是稳的；反过来图省事给裸 shell，迟早还账。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/1a02b28fc3f20e53.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/534eb825807f745b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/39c1cf033ff20c30.png)

