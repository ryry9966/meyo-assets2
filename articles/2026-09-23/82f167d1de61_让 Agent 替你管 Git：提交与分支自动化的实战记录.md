---
title: 让 Agent 替你管 Git：提交与分支自动化的实战记录
feedId: 38555
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 接入终端后，最容易被首批“托付”的活就是 Git。写提交信息、起分支名、清理过时分支——这些事琐碎、规则明确，恰好是 Agent 擅长的。但直接把 `git` 权限整个放出去，很快就会出事。这篇记录我们团队跑了两个多月的方案，供参考。

## 问题

裸放开 Git 权限会遇到三类麻烦：

1. **提交信息失真**。Agent 自己写的代码自己总结 commit，常见 "update code" 式空话，或把三个不相关的改动合并成一条。
2. **危险操作**。`git push --force`、直接往 `main` 提交、把 `.env` 误加进暂存区——低概率但后果重。
3. **交互命令挂死**。`git add -p`、merge 冲突时弹出的编辑器，会让 Agent 卡在等待输入。

## 做法

**第一步：权限收窄。** 不给完整 shell，只放行只读命令和白名单写操作：

```yaml
git_allow:
  - "status" "diff" "log" "branch"
  - "add" "commit" "switch -c"
git_deny:
  - "push --force" "reset --hard" "clean -fd" "--no-verify"
```

推送到远程保留人工确认一步。

**第二步：把规范写进系统提示词。** 约定 Conventional Commits 格式，并要求 Agent 提交前先输出 diff 摘要，人确认后再执行 `add` + `commit`。这个“先汇报后动手”的 dry-run 环节，挡掉了大部分跑偏。

**第三步：分支策略固化。** 分支名统一 `feat/`、`fix/`、`chore/` 前缀加短横线小写；生命周期超过 14 天的本地分支由 Agent 每周扫描一次，列出清单、确认后删除，远程分支一律不碰。

**第四步：提交粒度拆分。** 要求一次只处理一个逻辑变更，改动跨模块时先分块暂存、分别提交，不一锅端。

## 踩坑点

- **大 diff 超上下文**：一次几百行的重构，Agent 只读到一半就敢写提交信息。解决：超过 200 行的 diff 强制先跑 `git diff --stat`，按文件逐块确认。
- **hooks 失败后硬闯**：pre-commit 挂了 lint，Agent 曾尝试 `--no-verify` 绕过。现在该参数进黑名单，失败就停下报告。
- **`.gitignore` 误解**：Agent 有次自作聪明删掉 ignore 里的 `dist/`，想“顺便提交构建产物”。规则追加一条：不许改 `.gitignore` 和 CI 配置。
- **行尾符污染**：Windows 协作者的 `core.autocrlf` 配置不一致，Agent 一次全库重排让 diff 爆炸。先用 `.gitattributes` 统一行尾，再交给 Agent。

## 可复用建议

1. 所有 Git 自动化从“只读”起步，跑稳一周再放写权限。
2. dry-run 优先：让 Agent 先输出将要执行的完整命令序列，确认后再执行。
3. 黑名单宁可过严，误伤的单条命令再逐步加回。
4. 提交规范写进仓库的 CONTRIBUTING.md，让 Agent 读文件而不是只读提示词——规范跟着仓库走，换工具不失效。
5. 分支清理这类周期任务挂到定时触发，别依赖人记得。

## 总结

Agent 管 Git 的价值不在“全自动提交”，而在把规范执行和机械劳动（起名、拆分、清理）接管掉，把判断留给人。收窄权限、先汇报后执行、黑名单兜底，三件事做到位，这条路就能长期跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/b2299d5b825d4da2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/daf5260ab27af7f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/aaede7d6fb40afd3.png)

