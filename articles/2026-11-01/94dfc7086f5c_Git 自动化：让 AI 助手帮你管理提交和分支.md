---
title: Git 自动化：让 AI 助手帮你管理提交和分支
feedId: 43123
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

个人项目一多，Git 纪律就靠自觉：commit message 全是 "fix"、"update"，合并完的分支没人删，下班忘了提交，第二天上下文全丢。OpenClaw 天然有 shell 执行能力和定时任务（cron），让 agent 直接操作 git 是成本很低的自动化切入点——不需要写复杂插件，一套清晰的规则加一个白名单就够了。

## 问题拆开看

- 提交质量差：agent 不看 diff 凭文件名编 message；
- 工作区失控：半成品被顺手 commit，`.env` 险些进历史；
- 分支堆积：`git branch` 一列几十条 stale branch；
- 人肉巡检不存在：没人每天挨个仓库跑 `git status`。

## 做法：三步走，风险递增

**第一步：只读日报（先建信任）**

用 cron 每晚跑一次，把结果汇总发到聊天窗口：

```bash
git -C ~/projects/<repo> status --porcelain
git -C ~/projects/<repo> log --oneline -5
```

零风险，但你会立刻看到问题的规模。

**第二步：规范化提交**

在 agent 的技能/系统提示里固化规则：

1. 提交前必须先读 `git diff`，禁止凭文件名想象改动；
2. 遵循 Conventional Commits（feat / fix / chore / docs / refactor）；
3. 单次提交只做一件事。

之后日常只需说“提交一下”，agent 生成 message 并给出预览，你确认后落盘。

**第三步：分支卫生**

每周任务：列出 `git branch --merged main` 和最后提交超过 30 天的分支，生成清理清单，等你确认后删除。feature 分支的 push 可以授权，`main` 永远人工。

## 踩坑点

1. **误提交敏感文件**：agent 默认 `git add -A`，`.env` 直接进历史。先补全 `.gitignore`，并在规则里写死"遇到 .env / key / 大文件停下询问"。
2. **message 幻觉**：不给 diff 上下文，agent 会编造改动内容。流程里必须强制 `git diff` 先行。
3. **提交半成品**：你写到一半，agent 把工作区收走了。解法是只做"提醒"不做"代提交"，或约定只在你说"收尾"时触发。
4. **冲突越权**：pull 遇到冲突时 agent 可能自作主张 `checkout --theirs`。规则写明：冲突即停，交还给人。
5. **hooks 卡住**：pre-commit 里有交互式脚本会让 agent 挂住，务必设超时，并把报错原文带回来。

## 可复用建议

- **权限分三级**：只读汇总 → 确认后写入 → 授权范围内自动执行，逐级放开，别一步到位。
- **仓库白名单**：写在一个固定文件里，绝不让 agent 全盘扫家目录。
- **破坏性命令全部二次确认**：`reset --hard`、`branch -D`、`push --force` 一律人工点头，操作记日志便于审计。
- **提交规范做成独立 skill**：多个项目复用，不要散落在某次对话里靠 agent 自己回忆。

## 总结

Git 自动化是 agent 落地里"低风险、高回报"的一类场景：动作基本可逆、结果可审计、收益每天可见。核心原则一句话——agent 管琐碎和提醒，人管决策和越权动作。建议从每晚一份未提交清单开始，跑两周建立信任，再逐步放开提交和分支清理。自动化不是让 AI 替你写代码，而是让它把你不想操心的版本管理杂事接走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/8ab301a82bf58028.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/83c0ac8a316b220d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/ae7acee0f0ee3aee.png)

