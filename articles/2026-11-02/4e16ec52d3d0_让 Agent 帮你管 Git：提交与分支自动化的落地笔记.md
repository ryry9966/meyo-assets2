---
title: 让 Agent 帮你管 Git：提交与分支自动化的落地笔记
feedId: 43263
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

OpenClaw 这类常驻型 agent 的特点是：有稳定的 shell 执行能力、能跑定时任务、能通过 skill 固化流程。Git 恰好是"流程固定但琐碎"的典型场景——commit message 要规范、分支要命名、提交前要检查状态。这些事单独看都不难，难在长期坚持。我把一部分 Git 例行操作交给 agent 跑了一个多月，下面是实际经验。

## 问题

我遇到的真实痛点有三个：

1. 个人项目里 commit message 全是 `fix`、`update` 这类无信息量的词；
2. 经常忘记切分支，直接在 main 上改了一堆文件才反应过来；
3. 多个仓库并行推进时，`git status` 巡检靠自觉，经常有改动躺一周没提交。

## 做法

前提：agent 只拿到本地工作目录的 shell 权限，凭据不含 push 权限。

**第一步：固化提交规范。** 写一个 skill，内容是 conventional commits 的简化版（feat / fix / chore / docs 四类足够），并强制流程：先 `git status` → 再 `git diff` → 基于真实 diff 起草 message → 列出将 add 的具体文件 → 提交前输出一份摘要给我确认。

**第二步：分支纪律。** 约定 agent 接到任务后第一件事是检查当前分支；如果不在预期分支上，先报告再动手。新任务默认 `git switch -c feat/任务名`，命名由 agent 按规范生成。

**第三步：定时巡检。** 用 agent 的 cron 能力，每天固定时间跑一遍所有登记仓库的 `git status`，汇总未提交变更和未推送分支，发一条消息给我。这一步价值最大——它把"想起来才看"变成了被动推送。

**第四步：划定边界。** 所有写操作只到 commit 为止。push、merge、rebase 由人执行；agent 遇到冲突时直接停下并汇报，不允许自行解决。

## 踩坑点

- **`git add -A` 是大坑。** 早期 agent 图省事全量 add，把本地临时脚本和一份带密钥的 env 文件带进了暂存区，幸好确认时拦下。现在的规则：只允许 add 它逐个列出的文件。
- **message 幻觉。** 不看 diff 直接写 message 时，agent 会"脑补"它认为你做了的改动。强制先 diff 再写，准确率明显提升，但关键提交我仍会人工复核。
- **detached HEAD。** agent 在 checkout 历史 commit 做对比后忘了切回来，后续 commit 全部悬空。现在每次 git 操作前它必须先报当前分支和 HEAD 状态。
- **自动解决冲突不可靠。** 试过让 agent 自行 rebase 处理冲突，结果它选了"保留两边"这种和稀泥方案。冲突必须人来判断，这条已写死在 skill 里。

## 可复用建议

1. 规范写进 skill 文件，别靠对话记忆——skill 是幂等的，记忆不是；
2. Git 权限按"最小可用"配置：能读、能 commit、不能 push；
3. 新流程先在沙箱仓库跑一周，再上真实项目；
4. 所有 git 命令落日志，出问题能回溯是哪一步起的头。

## 总结

一个多月下来，agent 承担的是"流程执行者"角色：规范的 commit message、一致的分支命名、每日状态巡检，这三件事的执行质量比手动操作更稳定。但 push、merge、冲突处理这些有不可逆风险的动作，仍然留在人手里。Agent 管 Git 的合理定位不是替代判断，而是消灭遗漏——这个边界想清楚，这套流程就很省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/fed9a7cd78dcfebb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/eb4ca8d8a60fe71a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/848f707d60ebec34.png)

