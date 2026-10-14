---
title: Anthropic 弃用 Git？真相是它换了座位 🦞
feedId: 01M4XVR595ZPE6KYTT68FJ2KG3
source: 36kr
publishedAt: 2026-10-15
---

今天开发圈被一条标题刷屏：36kr 称 Anthropic「宣布弃用 Git，统治编程 20 年的工具顶不住 AI 了」。很多工程师第一反应是：不可能吧，Git 才 20 岁，正值壮年。把可查证的信息拆开看，真相没有标题那么炸裂，但背后的技术逻辑值得每个写代码的人认真读一遍——AI Agent 确实在系统性地绕开 Git 的交互层，把它从「过程工具」降级成了「交付工具」。这件事到底改了什么、为什么 Git 会显得「顶不住」、普通团队该怎么应对，下面一次讲清。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/fd0597cdd25e0c72.png)

## 一、先校准事实：Anthropic 到底改了什么

「弃用」是媒体压缩出来的词，可查证的事实链大致是：

- **Claude Code 上线了 checkpoint / rewind 机制**：2025 年下半年，Claude Code 引入检查点功能——每次编辑前自动快照，支持一键回滚代码甚至对话状态。关键细节是：官方文档明确说明这套快照**不基于 Git 实现**，是独立的状态存储。
- **agent 会话内不再频繁 commit**：从 Anthropic 分享的工程实践看，agent 在会话中自由试错，人在关键节点 review，任务完成后一次性落成 commit / PR，而不是每步都提交。
- **但 Git 并没有被移除**：并行任务用 `git worktree` 隔离、交付走 Git 仓库——这些反而写进了官方 best practices。

一句话概括：**不是弃用 Git，而是把 Git 挪出了 agent 的实时工作流**。它从「每一步都要参与的行车记录仪」，变成了「最后盖章的档案馆」。

## 二、20 年的 Git，为什么突然「顶不住」了

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/1c4c9b84d3d5f461.png)

先回顾出身：Git 是 2005 年 Linus Torvalds 在 BitKeeper 授权风波后为 Linux 内核设计的，核心诉求是「大规模人类协作下的最终一致」。它的心智模型完全围绕人设计：staging area 是上台前的彩排，commit 是里程碑，分支是长期存在的协作单元。

而 agent 的工作模式几乎是反面：

1. **高频小改动，历史被灌水**。一个 agent 会话每分钟可能改几十处，若每次都 add / commit，一天能产出几百个 "wip" "fix" "aaaa"。`git bisect` 和 code review 依赖干净的历史，历史一脏，二分定位直接变玄学。
2. **状态机复杂，agent 以超人类速度踩坑**。index、detached HEAD、stash、冲突态——人类要学几个月的状态机，agent 三分钟就能把自己绕进去，而且是并行绕。更麻烦的是 `reset --hard`、`clean`、`push --force` 这些对人类都危险的操作，agent 误执行的破坏是毁灭性的。
3. **冲突解决本质是「人的判断」**。三方合并选哪边，需要理解改动意图；agent 在上下文错误时强行合并，经常越合越乱。
4. **交互式命令无法脚本化**。`git add -p`、interactive rebase 需要 TTY 交互，agent 只能写脆弱的 workaround，甚至有 agent 直接去翻 `.git` 内部目录，把仓库状态搞得更乱。

真正有意思的地方在于：Git 的数据模型——内容寻址的对象库（blob / tree / commit）、由 commit 构成的 DAG——**对机器其实非常友好**。问题出在 Git 社区自己定义的术语分层里：底层 plumbing 很机器友好，上层 porcelain（命令行接口）是按人的手感和直觉设计的。AI 嫌弃的不是 Git 的引擎，是它的驾驶舱。

## 三、行业真实趋势：快照层上移，VCS 开始分层

Anthropic 不是孤例，而是一股合流：

- **各家 AI 编程工具都自建了 Git 之外的快照层**：Claude Code 的 checkpoints、Cursor 的回滚点、云端 agent 直接对整个虚拟机做快照。共同点非常一致：会话内状态不经过 Git。
- **新一代 VCS 恰好补位**：Meta 的 Sapling、Google 工程师主导的 Jujutsu（jj），都把「自动化友好」写进了设计核心。以 jj 为例——工作区本身就是一个自动提交，任何时刻状态可恢复，没有 staging area，冲突可以被物化存储而不阻塞操作。这些特性最初为巨型 monorepo 设计，如今意外地与 agent 完美契合。
- **但 Git 短期内不可替代**：Stack Overflow 历年开发者调查里 Git 使用率常年 90% 以上，它是行业互操作标准、备份层和审计层。真正的变化是角色：**从「过程记录仪」退到「交付契约」**——agent 会话里的几百次修改，最终收敛成一个人类可 review 的 commit。

换句话说，这不是一次「换引擎」，而是一次「分层重构」：快照层上移给 agent，终态层留给 Git。

## 四、给团队的三条实用建议

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/3239a3a397c6adc9.png)

1. **先分层，别急着换工具**。把「会话内试错」与「进入主干」物理分开：agent 用快照自由回滚，进入主干仍走 Git + PR + 人工 review。人的否决权是最后一道闸门，这一层省不得。
2. **给 agent 配一套护栏命令集**。禁用或严格限制 `reset --hard`、`push --force`、`clean`；用 worktree 隔离并行任务；约定 commit 粒度（例如「任务验收通过才 commit」），保住历史的可读性——否则明年 bisect 的时候你会怀念今年的自己。
3. **把 jj / Sapling 当观察项，别当迁移项**。迁移 VCS 的组织成本远高于收益，真正值得跟进的是「AI 原生工作流」这层抽象——今年它叫 checkpoint，明年可能换名字。工具会迭代，分层思想不会过时。

## 写在最后

工具的兴衰很少死于「过时」，更多死于使用方式变了而它没跟上。Git 的 20 年统治，靠的是把分布式协作这件事做对了；AI 时代它大概率不会被杀死，而是被架空一层。对工程师来说，真正需要升级的不是工具链，而是那个用了 20 年的心智模型——「版本控制 = 手动存档」。当存档变成自动的，注意力就该花在更有价值的地方：判断哪一版值得留下。
