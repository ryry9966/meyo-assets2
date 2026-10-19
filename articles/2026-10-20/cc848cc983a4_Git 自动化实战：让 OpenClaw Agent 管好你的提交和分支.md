---
title: Git 自动化实战：让 OpenClaw Agent 管好你的提交和分支
feedId: 42056
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

在 OpenClaw 的日常使用里，AI 生成代码的速度远快于人整理提交的速度。典型场景：一个下午改了五六个文件，涉及两个不相关功能，最后 `git add .` 一把梭，commit message 写着 "update"。时间一长，log 没法看，回滚定位全靠猜。

## 问题

拆开看是三类：

1. **提交粒度失控**：不相关改动混进同一次 commit；
2. **分支堆积**：feature 分支合完不删，两个月后没人敢动；
3. **高危操作风险**：agent 拿到完整 git 权限后，`push --force`、`reset --hard` 都是隐患。

## 做法

**第一步：接入 MCP Git 工具，但先只给读权限。** 挂一个 git MCP server，在 OpenClaw 里先只开放 `status` / `diff` / `log` / `branch` 四个只读能力，跑一周，确认它对仓库状态的理解可靠再放权。

**第二步：用常驻指令固化提交规范：**
- 每次提交前必须先跑 `git status` 和 `git diff`，按语义分组，不相关改动拆成多次 commit；
- message 用约定式格式（`type(scope): summary`），正文写"为什么"而不是"改了什么"；
- 提交前自查：不得包含 `.env`、密钥、构建产物。

**第三步：三段式执行，而非全自动。** 流程是 propose → confirm → execute：agent 给出分组方案和候选 message，我在对话里确认后它才执行 `add` + `commit`。push 默认不自动做，除非我明确说"推上去"。

**第四步：分支清理用定时任务，但先只出报告。** 用 OpenClaw 的 cron 每周跑一次：列出已合并、超过 30 天未动的分支，发到会话里。删除动作需要我逐个确认。

## 踩坑点

- **大 diff 截断**：单次改动超过几千行时 agent 读不全，message 会开始编造。解法是按文件分块读，或先做文件级摘要。
- **agent 会顺手 `add .`**：即使提示词里禁止也见过偷懒，后来在 pre-commit 里加了敏感文件扫描兜底。
- **MCP server 权限等于你的权限**：git 工具跑在你自己的用户下，没有额外沙箱。我用专用 git 用户 + 限定工作目录收权。
- **清理任务误删 WIP**：某个"30 天未动"的分支其实是留着的实验，所以报告模式永久保留，删除永远是显式动作。
- **遇冲突要停**：agent 处理 merge conflict 容易瞎选一侧。规则里写死：冲突即中止，交还给人。

## 可复用建议

- 把规范抽成可版本管理的 skill / 提示词模板，团队共用一份；
- 所有 agent 的 git 操作落一份本地日志，方便审计"它到底做了什么"；
- 按 只读 → 本地写 → 推送 的顺序逐步放权，每一步观察一两周；
- agent 永远不直接碰 main，push 一律走 feature 分支，交给 CI 验证。

## 总结

这套流程跑了一个多月，最大的收益不是省时间，而是提交历史变"可读"了——回滚、cherry-pick、写 changelog 都顺了很多。经验就一句话：让 agent 负责纪律和重复劳动，把破坏性操作和最终确认留给人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/622ec78ef1c15ec3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/107b9ca9050ebb2d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c2300730502b8fb1.png)

