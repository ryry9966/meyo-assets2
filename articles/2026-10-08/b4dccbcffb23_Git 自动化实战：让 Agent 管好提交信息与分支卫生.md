---
title: Git 自动化实战：让 Agent 管好提交信息与分支卫生
feedId: 40912
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

日常开发里，真正花在 Git 上的时间往往不是解决冲突，而是重复性杂活：写 commit message、起分支名、合并后清理本地分支、定期巡检远端废弃分支。这些事简单但琐碎，也最容易被 AI 助手接管。在 OpenClaw 里把这块交给 Agent 之后，我的感受是：收益不在某一次省了 30 秒，而在于规范被机器执行后不再依赖自觉。

## 问题

- **提交信息风格漂移**：同一仓库里 `fix bug`、`修复了xx问题`、`update` 混杂，回溯历史成本高。
- **分支蔓延**：`feat-xxx-final-v2` 这类分支合完就忘，几个月后没人敢删。
- **直接让 Agent 跑 git 风险高**：`push --force`、`reset --hard` 一旦误操作，代价不小。

所以目标不是"AI 全自动管 Git"，而是让它处理高频低风险的部分，高危操作保留人工确认。

## 做法

我用 MCP 工具 + 一个受限 git 包装脚本实现，分四步：

**1. 收敛 Agent 可用的 git 能力。** 不直接暴露 shell，而是写了个 `git-proxy` 脚本：白名单放行 `status / diff / log / branch / add / commit / checkout -b / merge --ff-only` 等只读或可逆命令，`push --force`、`reset --hard`、`clean -fd` 一律拒绝，所有调用落日志。

**2. 提交信息生成。** Agent 工作流：先 `git diff --staged --stat` 看概览，再对关键文件取 diff，按约定式提交模板产出 message。commitlint 做最后校验，格式不合格打回重写。

**3. 分支卫生。** Agent 定期巡检：`git branch --merged` 找出已合并分支，结合最近提交时间生成"建议删除清单"。只生成报告不执行，我确认后手动删（或放行 `branch -d`，它本身有未合并保护）。

**4. Hook 集成。** `prepare-commit-msg` hook 里调用 Agent CLI 生成默认 message，`commit -v` 时在编辑器里过目再保存。这样即使不开 Agent 面板，习惯性 `git commit` 也能受益。

## 踩坑点

- **大 diff 撑爆上下文**：一次重构提交几千行，整段 diff 直接丢给 Agent 必然超限。先 `--stat` 定位关键文件再分段读取，message 反而更准。
- **幻觉改动**：Agent 偶尔描述 diff 里不存在的修改。约束写进提示词：只允许描述 diff 中出现的内容，hook 层再用 commitlint + 人工确认兜底。
- **squash merge 陷阱**：`branch --merged` 检测不到被 squash 合并的分支，我差点据此删掉一个已上线分支。判断逻辑要加上"远端同名 PR 已关闭"这类信号，或者干脆只报不删。
- **并发写冲突**：Agent 在 `git add` 时我同时在终端操作，撞过一次 index.lock。约定单写者原则：Agent 操作期间不手动动仓库。
- **CI 下 hook 失灵**：`prepare-commit-msg` 依赖交互式编辑器，脚本里要加 TTY 判断，非交互环境跳过，避免 CI 提交被卡住。

## 可复用建议

- **先只读，再放开**：让 Agent 第一周只跑 status/diff/log，确认行为可信后再开放 commit。
- **规则文件进仓库**：提交模板、分支命名规范、白名单列表放在仓库的 `AGENTS.md`（或等价文件）里，Agent 每次都能读到，团队成员克隆即用。
- **Agent 输出一律当草稿**：message 要人确认，删分支要人点头，把"确认"做成流程而不是自觉。
- **全部操作留日志**：出问题能回放，这是排查自动化事故的唯一线索。

## 总结

控制好命令白名单和人工确认点，Git 自动化这件事风险可控、收益稳定：提交信息统一了，分支清单每周有 Agent 看一眼。建议从"只读巡检 + 提交信息生成"起步，跑两周再加码，别一上来就追求全自动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/bdc0547bfa65bf69.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/da87309bb2dfc9cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/65f785c0fb5212d6.png)

