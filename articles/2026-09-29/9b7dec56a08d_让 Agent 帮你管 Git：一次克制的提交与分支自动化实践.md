---
title: 让 Agent 帮你管 Git：一次克制的提交与分支自动化实践
feedId: 39439
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的典型用法是让 agent 常驻开发机、能执行 shell 命令。跑起来之后，最先被"顺手自动化"的就是 Git：一天几十次小改动，手动写 commit message、切分支、清理已合并分支，全是机械劳动。

但直接把 git 权限丢给 agent 是危险的。这篇帖子记录我把自己仓库的提交和分支管理交给 OpenClaw skill 的过程，重点不在"怎么写 prompt"，而在"怎么给权限"。

## 问题

裸跑 agent 管 Git，我踩过三类事故：

1. **暂存区污染**：agent 习惯性 `git add -A`，把 `.env`、调试日志带进了提交。
2. **高危操作**：rebase、`reset --hard`、force push，agent 在"修复问题"的名义下都可能执行。
3. **提交信息空洞**：生成的 message 全是 "update code"，历史没法读。

## 做法

核心思路：**不让 agent 直接敲 git，而是给它一组包装过的 skill 脚本**，脚本内部限定命令和参数。

1. 封装 `git_commit` skill：流程固定为 `git status → git diff --stat → 读取相关变更 → 生成 conventional commit message → 列出待提交文件清单 → 等我确认 → git add <显式文件列表> && git commit`。add 永远用明确文件列表，不接受 `-A`。
2. 封装 `branch_clean` skill：只做 `git branch --merged` 列出后逐个删除，且强制排除 main/master/develop。
3. 在 commit-msg hook 里校验 message 格式，不合规直接拒绝。agent 收到报错会自己修正重试，比 prompt 里写十遍规矩有效。
4. 推送和合并永远不自动执行，skill 只输出建议命令，我在终端里自己跑。

再配合 OpenClaw 的工具白名单，agent 在进程层面就跑不了 `git push -f`、`git reset --hard`——prompt 约束是软的，白名单是硬的。

## 踩坑点

- **不要让 agent 做交互式 rebase**。哪怕很想让它"整理历史"，改写历史的事留给人类。我的白名单里没有 rebase。
- **大仓库的 diff 会撑爆上下文**。先 `--stat` 看范围，再只读相关文件的 hunk，token 消耗能降一个量级。
- **hook 拒绝后 agent 可能死循环重试**。给 skill 加重试上限（3 次），超限就把原始状态报给我。
- **`[skip ci]` 这类标记要让 agent 知道**，否则自动提交会触发一轮无意义的流水线。
- **确认环节不能省**。前期我试过"完全托管"，结果它把一个调试文件提交进了历史，回滚花的时间比手动提交省下的还多。

## 可复用建议

- 权限靠白名单和脚本兜底，不要只依赖 prompt 约束。
- 所有 skill 设计成"输出计划 → 人工确认 → 执行"三段式，确认这步永远保留。
- 执行结果落日志（时间、命令、返回码），出问题能回溯。
- 先在个人仓库跑两周再考虑团队仓库；团队场景至少把 push 权限留在人手上。

## 总结

Agent 管 Git 的价值不是"全自动"，而是把 message 生成、分支清理这类低判断密集的活接走，同时用脚本和白名单把破坏面压到最小。自动化程度可以渐进：先只读，再写本地，最后才是远端——远端那一步我到现在也没放开。克制一点，这套东西才跑得长久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/390a6194d002deba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c7f437b99ca1184c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/a53a71c79dc38816.png)

