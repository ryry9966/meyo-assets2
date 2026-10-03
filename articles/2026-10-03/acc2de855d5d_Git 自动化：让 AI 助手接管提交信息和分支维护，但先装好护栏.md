---
title: Git 自动化：让 AI 助手接管提交信息和分支维护，但先装好护栏
feedId: 40278
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

写代码之外，日常 Git 操作里有相当一部分是纯机械劳动：照着 diff 编写符合规范的 commit message、把散落的文件归入合理的提交、清理已合并的本地分支、多任务切换时维护工作区。这些事不需要太多智力，但需要纪律。OpenClaw 这类常驻 agent 加上 MCP 工具链，恰好适合接手这种"低智力、高纪律"的活儿。

## 问题

但直接给 agent 一个 shell 让它随便敲 git，很快就翻过车。实际踩到的三类坑：

1. **不看 diff 就写 message**：模型根据文件名猜变更内容，描述和实际改动对不上。
2. **误伤工作区**：`git add -A` 一把梭，把调试临时文件、改了一半的代码一起提交进去。
3. **分支越管越乱**：agent 自作主张起了好几个 `fix-xxx-temp` 分支，还用 `-D` 强删过未合并的分支。

结论：不能给全能权限，要给"受限工具 + 流程约束"。

## 做法

**第一步：收敛工具面。** 不让 agent 跑任意 shell，而是通过 MCP 的 git server 只暴露 `status / diff / log / add / commit / branch -d` 这类只读和低风险命令。`push`、`rebase`、`reset` 一律不放行。

**第二步：强制先读后写。** 在 skill 提示词里写死流程：任何 commit 之前必须先执行 `git status` 和 `git diff --staged`，生成的 message 必须能从 diff 中找到依据，并附变更文件清单。这一条直接治好了幻觉式 commit message。

**第三步：message 走模板。** 约定 Conventional Commits，type 限定 feat/fix/refactor/docs/chore，scope 用模块名。agent 按模板输出，人只做最后确认。

**第四步：分支维护走汇报制。** 用 OpenClaw 的定时任务，每天跑一次 `git branch --merged`，列出可清理分支并生成带命令的建议清单。删除动作经我确认后再执行，且只用 `-d`——git 会自己拒绝未合并分支，这是天然保险丝。

**第五步：push 设人工门。** agent 的工作边界到本地 commit 为止。push 前它生成摘要（分支名、提交数、涉及文件），等我回复确认。跑了一周，误操作为零，确认成本也很低。

## 踩坑点

- **diff 太长被截断**：大变更装不进上下文时，agent 会基于半截 diff 写 message。解决：按文件分批读取，先做文件级小结再汇总。
- **多任务串扰**：我在 A 分支改了一半，让它处理 B 的事，结果 A 的脏文件也被 staged。后来约定：非本任务的未暂存文件一律不碰，遇到就停下来问我。
- **hooks 被绕过**：提示词里要明确禁止 `--no-verify`，否则 lint hook 形同虚设。
- **分支命名**：给它一个命名模板（如 `feat/<模块>-<短描述>`），否则每次起名风格都不一样。

## 可复用建议

- **最小权限起步**：先只给只读命令，让 agent 输出"建议命令清单"，人复制执行；跑稳两周再逐步放写权限。
- **写操作必须带回执**：做了什么、动了哪些文件、commit hash 是多少，落成固定格式，方便回溯。
- **流程写进 skill**，而不是每次对话临时交代。agent 的纪律来自提示词的确定性，不来自它的"自觉"。

## 总结

Git 自动化的收益不在"省下敲命令的几秒钟"，而在于把 commit 规范化、分支清理这类事从"想起来才做"变成"每天自动发生"。前提是承认 agent 会犯错：收敛权限、强制先读后写、写操作留人工门。做到这三点，AI 助手管 Git 是稳的；做不到，它就是最快把仓库搞乱的那个。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/054c2499b18135d6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/5b1ab67a4ddb8187.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/fae9c4ed1d86fb72.png)

