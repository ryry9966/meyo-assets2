---
title: Git 自动化实践：把提交信息和分支清理交给 Agent
feedId: 40407
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

在多数团队里，真正消耗心流的不完全是写代码，而是围绕代码的机械动作：写提交信息、切分支、合并后清理旧分支、推送前同步主干。每件事只花两三分钟，但一天被打断十几次。OpenClaw 的常驻 agent 本来就能调 shell 和 MCP 工具，正适合接管这类"规则明确、创造性低"的活。

## 问题

我们遇到的具体问题有三个：一是 commit message 风格漂移，三个月后 `git log` 基本不可读；二是本地和远程堆着几十个早已合并的分支，没人敢批量删；三是长期分支忘了同步主干，最后一次 rebase 冲突多到没人愿意处理。这些问题的共同点是：不需要智力，需要纪律——恰好是 agent 该干的。

## 做法

**第一步，给 agent 一个受限的 git 封装，而不是裸 shell。** 核心是命令白名单加全量日志：

```bash
ALLOW="status diff show log branch switch commit stash"
case " $ALLOW " in
  *" $1 "*) exec git "$@" ;;
  *) echo "[blocked] $*" && exit 1 ;;
esac
```

`push`、`reset --hard`、`clean` 一律不进白名单。如果你用的是 MCP git server，思路相同，把限制放在 server 暴露的工具集上即可。所有执行过的命令落盘，出问题可回放。

**第二步，提交信息生成。** agent 先跑 `git diff --staged --stat` 定位改动范围，再挑关键文件看完整 diff，按仓库里的 COMMIT_STYLE.md 生成 conventional commits 草稿。流程必须是：草稿 → 你确认 → 执行 commit，不要让它一步到位。

**第三步，分支体检。** 做成每天一次的定时任务：`git fetch --prune`、`git branch --merged main`、`git branch -vv`，输出一份报告——哪些已合并可删、哪些远程已不存在、哪些落后主干超过十个提交。报告归报告，删除动作仍需人工确认。

**第四步，写操作走确认机制。** agent 只产出命令清单，你回复确认后才执行。

## 踩坑点

- **squash merge 的陷阱**：`git branch --merged` 检测不到 squash 合并的分支，会误报"未合并"。稳妥做法是先重命名加 `archive/` 前缀观察一周，或结合 PR 状态再删。
- **上下文膨胀**：大仓库的完整 diff 很容易撑爆上下文。固定流程是先 `--stat`，再选文件，禁止把整个 diff 丢给模型。
- **幻觉**：agent 会编造 issue 编号和 scope。要求每条 message 的引用必须能在 diff 里找到依据，找不到就不写。
- **死循环**：如果 `prepare-commit-msg` hook 也调 agent，用环境变量做互斥标记，避免 hook 触发 agent、agent 再触发 hook。
- **提交签名**：走封装提交时确认 GPG/SSH 签名仍生效，用 `git log --show-signature` 抽查几条。

## 可复用建议

- 最小权限 + 全量日志，这是敢让 agent 碰 git 的前提。
- 一切写操作默认 dry-run，apply 需要显式确认。
- 规范沉淀在仓库文件里（COMMIT_STYLE.md、分支命名约定），提示词只引用路径，避免两处维护。
- 先在个人仓库跑一两周，把提示词和白名单调稳，再推给团队。

## 总结

这套方案里，agent 接管的是那 80% 的机械动作——生成提交信息、体检分支、产出命令清单；判断权和确认权留在人手里。权限收紧、默认 dry-run、日志可回放，落地之后 git 卫生问题基本消失，也没有引入新的风险面。建议从"提交信息生成"这一个点起步，跑稳了再扩展到分支管理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/6b16abfe1858319a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/310e51f0c0cae385.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/9f32b4b6676d89fe.png)

