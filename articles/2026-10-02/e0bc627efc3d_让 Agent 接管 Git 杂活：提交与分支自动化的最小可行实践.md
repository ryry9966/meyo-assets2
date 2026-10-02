---
title: 让 Agent 接管 Git 杂活：提交与分支自动化的最小可行实践
feedId: 40146
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

我手上长期维护着五六个中小型仓库，提交频率不低，但 git 卫生一直一般：commit message 时而一个 `fix` 时而三大段，合并完的 feature 分支一挂就是几个月，偶尔还忘了 push。代码本身不想让 AI 写，但这类高频、格式化、低创造性的杂活，正好适合交给 OpenClaw 里挂了 shell/MCP 工具的 agent 来做。这篇文章记录我落地的过程和踩的坑。

## 问题

归纳下来是三类：

1. **提交信息不规范**：没有固定格式，回溯历史时基本靠猜。
2. **分支堆积**：远端已合并的本地分支没人删，陈旧分支分不清还要不要。
3. **流程断点**：改完忘提交、提交完忘推送，跨天搁置。

还有个隐性风险：让 agent 自由跑 git，等于把 `push --force`、`reset --hard`、`clean` 这些不可逆操作交给了它。

## 做法

**第一步：收敛工具面。** 不给裸 shell，写了一个 `git-safe.sh` 包装脚本，只放行 `status / diff / log / add / commit / switch / branch -d / push`，对 `--force`、`reset --hard`、`clean` 直接拒绝。通过 MCP 暴露给 agent。靠 prompt 约束危险命令不可靠，靠白名单才可靠。

**第二步：提交两段式。** 默认流程是：agent 跑 `git diff --staged` → 按 conventional commits 风格生成 message 和改动摘要 → 我确认 → 执行 `add + commit`。对于改动小于 50 行、不涉及新文件和敏感路径的琐碎修复，允许全自动模式。

**第三步：分支清理报告。** 用定时任务每天跑一次：agent 列出本地已合并未删除的分支、30 天以上无提交的陈旧分支，输出一份报告。我看完确认后批量删除。**报告自动，删除人工**，这条红线我保留了。

**第四步：留痕。** 所有 agent 执行的 git 操作写入日志文件，带时间戳和理由，方便事后审计。

## 踩坑点

- **`git add -A` 扫进 .env**。agent 不知道哪些是密钥。后来在包装脚本里拦截了对敏感路径的 add，并加了 pre-commit 兜底。
- **message 写的是"意图"不是"事实"**。agent 看函数名就脑补了改动目的，和 diff 实际内容对不上。现在要求 message 必须逐条对应 diff，我会抽查。
- **沙箱里 worktree 不对**。有一次 agent 在容器里跑，落在 detached HEAD 上提交，commit 差点"消失"。现在脚本第一步固定校验仓库路径和当前分支。
- **历史改写**。agent 曾建议对已推送分支做 `commit --amend`，共享分支上这等于制造冲突。白名单里直接封掉。
- **大 diff 撑爆上下文**。几百行的 diff 要先按文件分块摘要，再生成 message。

## 可复用建议

- 工具面用包装脚本 + 白名单收敛，别指望 prompt 能拦住危险命令；
- "提案 → 确认 → 执行"作为默认交互，自动化只对低风险路径开放；
- 清理类操作永远做成报告，由人按下删除键；
- 提交规范写进 skill 的常驻 prompt，一次配置长期生效。

## 总结

AI 管 git 的价值不在替你做决定，而在接走高频、低风险、格式化的操作。权限白名单 + 两段确认 + 报告式清理，三个机制加起来不到一百行脚本，我每周大约两小时的 git 杂活被压到了十几分钟。建议从提交信息生成起步，跑稳两周再扩到分支报告，不要一上来就放手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/ade262ad25519283.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/dc54db38a37e8041.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/95fbb62b17ec5ec1.png)

