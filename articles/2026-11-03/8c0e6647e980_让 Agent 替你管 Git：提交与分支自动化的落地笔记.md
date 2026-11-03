---
title: 让 Agent 替你管 Git：提交与分支自动化的落地笔记
feedId: 43327
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

我们平时用 OpenClaw 的 agent 做编码任务，任务结束经常留下半成品：diff 没提交、commit message 随手写、分支越攒越多。既然 agent 本来就有 shell 能力，把 git 日常操作交给它是顺理成章的事。但"让 AI 操作版本库"和"让 AI 写代码"的风险等级完全不同，这篇记录一下实际使用中的收敛过程。

## 问题

三个高频痛点：

1. commit message 质量不稳定，回看历史基本靠猜；
2. 分支只增不减，已合入的分支没人删，本地远端各一套；
3. agent 改完代码经常忘记提交，下次会话上下文全丢。

## 做法

分三步，核心思路是**限制写权限，扩大读权限**。

**第一步：划定操作边界。** 在工作区规则（AGENTS.md 或等效配置）里写死硬约束：只在 `feat/*`、`fix/*` 分支上工作；禁止对 main 和共享分支执行 push、`reset --hard`、rebase；禁止 force push。规则对所有会话生效，不依赖对话里反复叮嘱。如果不想开 shell，也可以用社区的 git MCP server，把暴露的 tool 收敛到少量只读查询加受限写入。

**第二步：提交走"提案制"。** agent 完成任务后不直接 commit，而是先输出三样东西：diff 摘要、Conventional Commits 格式的建议 message、影响面说明。人确认后才执行，执行统一走 git wrapper 脚本做白名单校验：

```bash
# git-wrapper.sh 节选
case "$1" in
  push)
    if [[ "$*" == *"--force"* ]]; then
      echo "blocked: force push not allowed" >&2; exit 1
    fi ;;
  add)
    if [[ "$*" == *"-A"* || "$*" == *"."* ]]; then
      echo "blocked: no bulk add, stage files explicitly" >&2; exit 1
    fi ;;
esac
exec git "$@"
```

**第三步：分支清理做成定时任务。** 用 cron 每周五跑一个脚本：列出已合入 main 的本地与远端分支，生成删除建议清单发给用户，人回复确认后 agent 才批量执行。清理永远停在"建议"这一步。

## 踩坑点

- **message 幻觉**：一开始只喂文件名列表，agent 编出和实际改动完全对不上的提交说明。后来强制它先读 `git diff` 再写 message，质量立刻上来。上下文不够时按目录拆分提交。
- **误提交敏感文件**：agent 一次性 `git add -A` 差点把 .env 提交进去。现在 wrapper 拦截批量 add，并且 pre-commit 挂了 secret 扫描。
- **凭证暴露面过大**：最初 agent 用的是用户全局 git 配置。后来给它单独签发一个只授权单仓库的 SSH key，并关掉 credential helper 缓存。即使被提示注入，能碰到的也只有这一个仓库。
- **清理误判**：有分支 squash merge 后原分支仍显示"未合入"，脚本差点删掉活分支。判断逻辑改成同时核对 `git log --grep` 与 PR 状态。

## 可复用建议

- 所有 git 约束写进工作区规则文件，一次性生效，别靠会话内口头约束；
- 危险命令走 wrapper 白名单，而不是信任模型自觉；
- 提交与删除都用"提案—确认—执行"三段式，人在环上成本很低（回一个字），换来的安全性完全值得；
- 全部 git 操作落日志，出问题能回放。

## 总结

git 自动化的收益不在"快"，而在**一致性**：message 规范、分支及时清理、每次改动有据可查。关键是把 agent 的写权限压到最小——能建分支、能提交到 feature 分支，到此为止。任何触碰 protected branch 的操作都留给人。做到这一步，这套自动化才敢长期开着。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/4538ba23bdf70dbd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/4e0df46b628aea58.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/faa8c8a921692ccd.png)

