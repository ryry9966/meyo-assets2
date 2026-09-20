---
title: Git 自动化实践：让 Agent 接管提交与分支的重复劳动，但守住三条红线
feedId: 38295
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

写 commit message、起分支名、补变更说明，这些活不难，但天天做、反复做，而且最容易敷衍。观察团队仓库：`fix`、`wip`、`update` 之类的提交信息比比皆是，分支名里躺着 `dev-backup2`、`test-final`，三个月后没人知道它们是干嘛的。

这类“低智力含量但高纪律要求”的工作，恰好适合交给接了 MCP Git 工具的 Agent。关键不是让它全自动跑，而是给它清晰的边界。

## 问题

直接给 Agent 一个能执行任意 git 命令的工具，等于把仓库权限整个交出去。实际会撞上四件事：

1. **误提交敏感文件**：Agent 习惯性 `git add -A`，`.env`、密钥、本地配置跟着进暂存区；
2. **危险操作**：`push --force`、`reset --hard`、`branch -D`，一次跑偏就是事故；
3. **提交信息失真**：Agent 拿对话里的任务描述当提交说明，而不是从实际 diff 推导，写出来“看起来对”但和改动无关；
4. **上下文爆炸**：大 diff 直接丢给模型，token 消耗大，摘要质量还下降。

## 做法

我们的方案分三层：工具层收敛、规则层约束、流程层确认。

**第一步：收敛 MCP Git 工具面。** 只暴露白名单操作：`status / diff / log / add / commit / branch / checkout` 直接可用；`push` 单独确认；`push --force`、`reset --hard`、`branch -D` 在工具配置里物理拒绝，不进模型视野。

**第二步：把规范写进仓库级指令文件。** 在仓库根目录的 agent 指令文件里明确：Conventional Commits 格式、分支命名 `feat/xxx` `fix/xxx`、提交前必须先跑 `git diff --stat` 了解范围、禁止 `add -A`、逐文件添加。Agent 每次会话自动读到这些规则，不靠临时叮嘱。

**第三步：固定工作流。** 每次提交按同一序列执行：`status` → `diff`（大 diff 先 `--stat` 再按文件分段）→ 从 diff 推导提交信息并标注 scope → 列出将暂存的文件 → 人确认 → `add` 指定文件 → `commit`。push 一律人工执行或单独确认。

**第四步：分支管理任务化。** 让 Agent 每周巡检一次：列出超过 30 天未动的分支、已合并未删的远程分支，只输出报告，删不删人来定。

## 踩坑点

- **提交信息幻觉**：最初 Agent 会把任务描述原样写进 commit message，和实际改动对不上。后来强制要求“提交信息只能来自 diff 本身，且先引用改动文件再写说明”，失真率明显下降。
- **`.env` 差点入库**：即便有 .gitignore，Agent 仍试图添加白名单外的文件。补了一条硬规则：不在白名单内的一律列出询问，不许静默加入。
- **amend 已推送提交**：Agent 有次想用 `--amend` 修正已推送的提交，这会改写远端历史。改为仅允许 amend 未推送提交，且执行前先比对远端状态。
- **token 开销**：大仓库全量 diff 一次上万 token。改成 `--stat` 优先、按文件分批读，成本降了一个量级。

## 可复用建议

1. 规范写进文件，不要靠对话叮嘱——仓库级指令文件是唯一可靠的记忆；
2. 写操作全部过机器校验：commitlint + pre-commit hook 做最终门禁，Agent 的输出必须通过检查，而不是被信任；
3. 历史改写类命令在工具层禁用，别指望提示词兜底；
4. 记录 Agent 的 git 操作日志，出问题能回溯；
5. Agent 只产出“建议”，破坏性动作保留人工确认。

## 总结

Agent 管 Git 的价值不在“全自动提交”，而在于把提交规范、分支纪律这些容易滑坡的事，变成稳定的默认行为。收益是持续的一致性和更少的上下文切换；代价是前期要把工具边界和规则文件搭好。搭好之后基本不需要维护——因为所有判断都留在了该留的地方：diff 里、规则里，和人手上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/f4c39509acabe1e4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/b89a0bd09d3aa2b5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/3f33054f1f15ebd2.png)

