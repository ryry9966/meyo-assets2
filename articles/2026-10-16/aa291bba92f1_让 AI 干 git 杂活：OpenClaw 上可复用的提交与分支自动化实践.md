---
title: 让 AI 干 git 杂活：OpenClaw 上可复用的提交与分支自动化实践
feedId: 41673
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

仓库维护里有一类活：写 commit message、清理分支、汇总提交记录。单次不多，日积月累很烦。这类工作规则明确、机械重复，正好适合交给 OpenClaw——它有 shell 和 MCP 工具，跑 git 命令没有门槛。问题只在于：怎么安全地放权。

## 问题

直接让 agent 碰 git，常见的三个坑：

1. **提交信息随缘**：agent 图省事写 "update"、"fix"，历史比人写的还乱；
2. **误操作风险**：`push --force`、`reset --hard`、把 `.env` 一起 add 进去；
3. **颗粒度失控**：五六个不相干的改动被打包成一个 commit，review 没法看。

## 做法

**1. 限制工具面。** 不给完整 shell，接 git MCP server 或封装一小撮命令：`status / diff / log / commit / branch`。`push --force`、`clean`、`reset --hard` 从根上不可达。

**2. 规则写进仓库的 AGENT.md：**
- commit message 遵循 Conventional Commits，body 必须说清改了什么、为什么；
- 分支命名 `feat/`、`fix/`、`chore/` 前缀；
- 三条禁令：不直接提交 main、不用 `--no-verify` 绕钩子、禁止 `git add -A`（必须按文件名 add）；
- 破坏性操作一律先停下等人工确认。

**3. 提交流程。** agent 跑 `status → diff`，把改动拆成逻辑独立的几个 commit，写好 message 后把 staged diff 列给我确认，点头才执行。前两周全程确认模式，跑稳后才放开小改动自动提交。

**4. 分支清理做成定时任务。** 每天扫一遍：已合入 main 的本地分支、14 天无活动的 feature 分支，输出清单由我勾选后删除。**只列不删**，这条后来救过我们。

**5. 给 agent 的提交留痕。** 加 `Co-authored-by` trailer，半年后审计分得清哪些是人写的。

## 踩坑点

- **`add -A` 差点提交密钥**：`.env` 进了暂存区，靠 pre-commit 的密钥扫描拦下。之后规则里强制按文件 add。
- **agent 自作主张 rebase**：它想"整理历史"，试了 force push。教训是禁令必须走工具黑名单，光写在 prompt 里不可靠。
- **squash merge 识别不到**：`git branch --merged` 对 squash 合并的分支无效，得拿分支名去匹配 main 上的 commit message 兜底。
- **误伤在途分支**：一次定时任务把出差同事两周没动的分支标成可删。从此清理任务只出清单。
- **并发冲突**：agent 提交撞上我正在 rebase。把定时任务挪到早上十点前，撞车基本消失。

## 可复用建议

1. **权限分级放**：先只读（汇总 log、生成报告），再放 commit，最后才是分支管理。
2. **规则文件当代码维护**：进仓库、走 review，团队所有人受益。
3. **碰远程或破坏性的事，用"清单 + 确认"替代"自动执行"**。
4. message 写不具体，多半是 commit 拆得太粗——把这条当信号反过来约束颗粒度。

## 总结

这套自动化 value 不在"无人值守"，而在把机械劳动卸载掉的同时，给危险动作留人工卡点。git 杂活场景下，从"只读 + 建议"起步就能省下不少时间；你信不过的事就写死成规则和工具限制，比指望模型自觉靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/1d7a3dff7ad1ad77.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/591fbb7f3e92225a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/26cbc5497f37259c.png)

