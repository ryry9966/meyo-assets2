---
title: 让 Agent 当仓库管家：Git 提交与分支自动化实践
feedId: 42480
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

个人和小团队仓库的日常维护，大量是机械劳动：写 commit message、删已合并分支、整理错位提交。这些事不难，但高频且容易被敷衍，三个月后 history 就没法看了。OpenClaw 的 agent 本身具备执行 shell、读文件、跑定时任务的能力，很适合把这部分接过去。最近两个月我把多个仓库的提交和分支管理交给 agent，这里记录可行做法和踩过的坑。

## 问题

难点不是"AI 能不能跑 git 命令"，而是三件事：

1. **提交质量**：agent 生成的 message 容易脱离实际 diff，写得比代码还漂亮；
2. **安全边界**：`push -f`、`reset --hard`、误提交密钥，任何一条都是事故；
3. **判断归属**：哪些分支该删、冲突怎么解，agent 不该替人拍板。

## 做法

**第一步，权限收敛。** 不给 agent 裸 shell 的 git 权限，而是写了一个包装脚本 `gitctl`，白名单子命令：`status / diff / log / add（限路径）/ commit / branch -d / --merged`。危险命令在包装层直接拒绝并记日志，agent 只能通过 skill 或 MCP 调它。

**第二步，仓库约定落盘。** 每个仓库放一份 conventions 文件（进 AGENTS.md 或 skill 上下文）：Conventional Commits 格式、分支命名规则、禁止 amend 已推送提交、冲突即停。

**第三步，diff 驱动的提交流程。** 流程固定为：`git status` → 人确认要提交的文件 → `git diff --cached` → 基于真实 diff 生成 message → gitleaks 预检 → 默认 dry-run 输出"将要提交什么"，确认后执行。message 里不允许出现 diff 中不存在的描述。

**第四步，分支清理。** 定时任务每天跑一次：列出 `--merged main` 的本地分支和超 30 天未动的远程分支，生成报告推送出去。agent 只报不删，删除动作等人确认后触发白名单命令。

## 踩坑记录

1. 早期让 agent 自己 `git add -A`，一次把 `.env` 提了进去。此后 add 一律限路径，gitleaks 进必经链路。
2. agent 偶尔"顺手" amend 已 push 的提交，导致团队历史分叉。规则文件写死还不够，我在包装层也加了检测：分支有 upstream 且未领先时，拒绝 amend。
3. 分支清理按"30 天未动"直接删，会误伤长期开发的 feature 分支——即使其旧版本已合并。改成只清理"已合并且名不以 `feat/` 开头"的分支。
4. 两个定时任务并发操作同一仓库时撞过 `index.lock`。现在每次调 `gitctl` 前先 flock。
5. rebase 遇到冲突时，agent 的第一反应是反复尝试自动解决，越解越乱。规则改为：检测到冲突，立即退出并输出冲突文件清单。

## 可复用建议

- `gitctl` 这类"白名单包装 + 审计日志"可以做成通用 skill，换仓库零成本复用；
- 一切先 dry-run：agent 输出"计划"而非"结果"，人再放行；
- 让 agent 产出周报（未推送提交、落后分支、陈旧分支），零风险，但最能建立信任；
- 放权节奏：只读 → 本地提交 → 分支删除，每档跑两周再进下一档。

## 总结

AI 管 git 的价值不在全自动，而在把机械部分（message、清理、汇总）稳定外包，把判断留给人。权限白名单、diff 驱动、dry-run 加人工确认，这三件事做扎实，agent 就是可靠的仓库管家；做不扎实，它就是出事最快的同事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/da2d268d9d0acdeb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/d9c60fadb0e961ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/da74d335fad5f48d.png)

