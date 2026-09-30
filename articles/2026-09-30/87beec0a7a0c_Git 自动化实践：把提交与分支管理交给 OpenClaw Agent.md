---
title: Git 自动化实践：把提交与分支管理交给 OpenClaw Agent
feedId: 39888
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

一个人维护两三个项目时，最耗精力的往往不是写代码，而是收尾：改完一坨文件懒得拆提交，message 全是 `update` 和 `fix`；本地堆着十几个 `feat/xxx` 分支，哪个合过全靠记忆。这些活没技术含量，但最容易因为"顺手"出事故——比如在 main 上直接 commit。

我的 OpenClaw 本来就常驻一台小主机跑巡检脚本，于是尝试把 Git 日常也交给它。跑了一个多月，收敛出一套还算稳的做法。

## 问题：裸奔的 Agent 很危险

直接让 Agent 执行 git 命令，观察到的风险很具体：

1. 它会"好心"地帮你 `git push --force`；
2. commit message 经常基于文件名脑补，和实际 diff 对不上；
3. rebase 遇到冲突会陷入循环重试；
4. 权限给大了，等于把仓库写权限交给一个概率系统。

所以目标不是全自动，而是分层：重复性环节交给 Agent，不可逆操作保留人工确认。

## 做法：四步收敛

**第一步，环境收口。** 确保 OpenClaw 运行环境有 `git` 和 `gh`，`gh` 用细粒度 PAT 只授权目标仓库；在运行用户下单独配 `git config user.name/email`。这步很基础，但我第一次就栽在 agent 的系统账户没有 git identity 上，提交作者全乱了。

**第二步，固化规则。** 写一个持久化的 git-ops skill（或并入 AGENTS.md）：

- 分支命名统一 `feat/`、`fix/`、`chore/` 前缀；
- 提交信息遵循 Conventional Commits；
- 提交前必须真实执行 `git diff --staged`，禁止只看文件名写 message；
- 默认只 commit 不 push；push 共享分支、任何 `--force`、改写历史，一律先询问；
- 禁止直推 main，一律 feature 分支 + PR。

关键：规则必须放持久文件并纳入版本管理，靠聊天记忆三天就丢。

**第三步，定义三个高频场景。** 提交整理：agent 读 staged diff，把混杂改动拆成多个逻辑提交，我确认后执行；分支清理：每周用 `git branch --merged` 与 `gh pr list` 交叉核对，生成清理清单，删除由我确认；PR 与变更摘要：基于 `git log` 区间生成草稿，人只审校。

**第四步，加两道兜底。** 服务端开 branch protection 禁止直推 main——这是真正的安全网，比任何 prompt 约束可靠；本地 pre-commit hook 跑 lint 和 secret 扫描，防止 `.env` 之类被带进去。

## 踩坑记录

- **脑补式 message**：早期允许它"根据改动文件猜内容"，一个只改配置的提交被写成"重构认证模块"。必须强制读真实 diff。
- **冲突死循环**：它曾反复尝试自动解冲突，差点把两侧改动都丢了。现在规则写明：遇冲突立即停下报告。
- **token 权限过大**：图省事用过 classic PAT 全仓库权限，后来换成细粒度、限定仓库的写权限。
- **会话间遗忘**：约定写在聊天里必然失效，全部迁入 skill 文件，改约定等于改代码。

## 可复用建议

1. 权限灰度上线：先只读（status/log/diff）跑一周，再加本地 commit，最后才放 push 和 PR；
2. 要求 agent 在执行变更类命令前先完整输出命令本身，等于自带 dry-run；
3. 用包装脚本记录 agent 执行过的 git 命令，出问题能回溯；
4. 规则文件跟仓库走，不同项目可以有不同约定。

## 总结

一个多月下来，`update` 式提交基本消失，本地分支稳定在个位数。Agent 擅长的是读取、归纳、按规范生成这类确定性流程；不擅长的是判断"这次改动该不该合"。把前者交出去，把后者留住，这套东西才敢长期开着。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/f915961010ceb69e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/bee7830716d17828.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/de53e943253ecb07.png)

