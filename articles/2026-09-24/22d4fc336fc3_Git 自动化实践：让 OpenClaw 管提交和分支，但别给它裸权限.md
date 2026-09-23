---
title: Git 自动化实践：让 OpenClaw 管提交和分支，但别给它裸权限
feedId: 38708
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

我手头大多是单人项目加几个小仓库，提交信息、分支清理这类事占比不高但很磨人：写 commit message 要来回看 diff，合并完忘删分支，WIP 提交越堆越多。OpenClaw 本来就有 shell 和 MCP 工具能力，索性把这部分交给它，人只守关键闸门。

## 问题

直接让 agent 跑 git 有三类风险：

1. **误操作**：`reset --hard`、`push -f`、切分支丢工作区改动；
2. **提交内容失控**：`git add -A` 把 `.env`、日志一起带进去；
3. **信息失真**：模型写的 message 与实际 diff 不符，事后考古很痛苦。

所以目标不是"全自动"，而是 **agent 干活，人守闸门**。

## 做法

### 1. 单一入口：包一层 git 包装脚本

不让 agent 直接敲 git，给它一个 `agent-git` 脚本做白名单校验：

```bash
#!/usr/bin/env bash
allowed=(status diff add commit branch switch log stash)
deny=(reset push rebase clean checkout -- .)
cmd=$1; shift
[[ " ${deny[*]} " == *" $cmd "* ]] && { echo "denied: $cmd"; exit 1; }
exec git "$cmd" "$@"
```

`push`、`rebase` 不进白名单，由我手动执行。agent 的全部 git 行为 `tee -a` 到一份日志，出问题能回溯。

### 2. 用 Skill 固化提交规范

在 OpenClaw 里写一个 git-commit skill，核心约束：

- 提交前必须先跑 `status` + `diff`，message 只允许基于 diff 内容描述，禁止臆测；
- 遵循 Conventional Commits 前缀（feat/fix/chore/refactor）；
- 暂存前检查是否含 `.env`、`*.log`、密钥类文件，命中即中止并报告；
- commit 前把文件列表和 message 发给我，确认后才执行。

### 3. 分支管理的定时任务

用 cron 触发每日任务：列出已合并到 main 的本地分支生成清理清单（只列不删）；检查有超过 N 天未提交改动的仓库并提醒；分支命名不符合 `feat/|fix/|chore/` 前缀的，列出来让我改名。

## 踩坑点

1. **`git add -A` 是最大隐患**。后来把 `add` 限定为显式文件路径，并在 skill 里强制先跑一次密钥扫描，才真正堵住。
2. **commit message 幻觉**。早期它写过"refactor: 优化模块结构"，但 diff 只是改了配置。之后规定 message 必须引用 diff 里的具体文件/函数名，确认时我把 diff 一并贴出来人工核对。
3. **rebase 冲突别让它自己解**。试过让它处理冲突，结果它为了"解决"冲突改了业务逻辑。现在遇冲突一律停手交还给人。
4. **hook 失败引发重试循环**。pre-commit 挂了它就反复重试，加一条规则：同一命令失败两次即停止并报告。
5. **跨仓库上下文混淆**。多仓库并行时它把 A 仓库的 message 用到了 B 上，skill 里强制第一步输出当前仓库路径做校验。

## 可复用建议

- 危险操作不要靠 prompt 约束，要靠脚本白名单。prompt 是软约束，shell 才是硬约束。
- 所有 agent 的 git 行为留日志，成本低、收益大。
- "只列不做"是个好模式：清理、改名、合并建议都先输出清单，人点头再执行。
- 确认粒度对齐风险：本地 commit 可以抽查，push 和分支删除必须逐次确认。

## 总结

这套流程跑下来，commit message 质量比我随手写的高，分支也常年干净。关键经验一句话：自动化不等于放手——把 agent 权限收窄到包装脚本里，把人的确认留在 push、删除这类不可逆动作上。代码还是我的，它只是个勤快的助手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/65d5b3d14cab6e2c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/36781edf5c52e07c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/95446c80c7391153.png)

