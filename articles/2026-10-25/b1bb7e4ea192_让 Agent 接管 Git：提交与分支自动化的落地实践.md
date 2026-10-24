---
title: 让 Agent 接管 Git：提交与分支自动化的落地实践
feedId: 42506
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

在 OpenClaw 的典型用法里，Agent 已经能帮你写代码、跑测试，但流程经常到"提交"这一步就断档：改完的文件堆在工作区，commit message 想半天，分支开得随手，main 上混着半成品。MCP 和自定义工具让 Agent 能直接操作 git，这件事技术上可行——但"可行"和"敢用"之间隔着权限设计。

我最近在自己的几个仓库上跑了一套方案：Agent 承担提交和分支的日常操作，人只保留 review 和 merge。以下是做法和真实踩过的坑。

## 问题：裸给 git 权限是灾难

最直觉的思路是给 Agent 一个 shell，让它自己跑 `git add/commit/push`。实际试过就知道问题：

1. **危险命令无防护**：`reset --hard`、`push --force`、`clean -fd`，Agent 在纠正自己的错误时非常爱用这些。
2. **提交粒度失控**：要么一晚上 30 个碎提交，要么一次塞进十几个不相关改动。
3. **敏感文件风险**：`.env`、构建产物、临时密钥被顺手 add 进去，单靠提示词拦不住。
4. **message 质量飘**：中英混杂、语气浮夸、和 diff 内容对不上。

结论：不能让 Agent 自由发挥，要给它一把受限的杠杆。

## 做法：薄封装 + 提交契约 + 钩子兜底

方案分三层，成本都不高：

**第一层：受控工具集。** 不给裸 shell，而是接一个 Git MCP server（或自己写几十行的薄封装），只暴露有限操作：`stage_files`、`commit`、`create_branch`、`status_summary`、`diff_summary`。工作目录锁死在项目路径，`push` 单独做成需要人工确认的动作。

**第二层：提示词里的"提交契约"。** 写在 OpenClaw 工作区的 `AGENTS.md` 里，规则要可判定：

- 一个逻辑改动一个提交，跑通测试后才允许 commit，禁止每改一行就提；
- message 固定格式 `type(scope): 中文摘要`，正文写清动机；
- 只在 `agent/*` 前缀分支上工作，永远不直接动 main；
- 提交前先跑 `diff_summary` 自查，确认没有混入临时文件。

**第三层：钩子兜底。** pre-commit 挂 gitleaks 扫敏感信息，再挂 lint/format。提示词会失效，钩子不会。

另外强烈建议用 `git worktree` 给 Agent 单开一个工作区，和你手头的改动物理隔离，互不踩脚。

## 踩坑点

- **Agent 绕过工具直接跑 shell**：会话里只要还开着通用 shell 工具，它就去找捷径。解决：会话内只保留白名单工具，或对 shell 做命令审计。
- **detached HEAD 陷阱**：Agent 切分支失误后继续提交，提交挂在游离 HEAD 上，容易丢。在 `create_branch` 工具里加了状态前置校验才堵住。
- **大 diff 总结失真**：改动超过几百行时，Agent 的 message 会开始"编内容"。改为分块读 diff、按文件聚合再总结，准确率明显回升。
- **碎提交倾向**：光说"攒够再提"没用，Agent 不知道什么是"够"。换成可判定的标准（测试通过才提交）才见效。

## 可复用建议

1. 工具按"动词"设计（commit、branch），不要暴露裸 `git` 命令执行。
2. 提示词里的规则必须可判定、可检查，别写"注意规范"这种空话。
3. 钩子是最后防线，secret 扫描务必最先配好。
4. 新仓库先让 Agent 只 commit 不 push，观察一周再逐步放开。

## 总结

这套东西上线后，"忘了提交"和"message 随手写"两个老问题直接消失，分支历史也干净了。核心原则其实一条：**给 Agent 受限的杠杆，把不可逆操作留给人**。Git 自动化要落的不是"让 AI 全管"，而是把重复劳动交出去，把决定权留下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/16754911e7d0966f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/637deeee020c3459.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/302e9c89300f650d.png)

