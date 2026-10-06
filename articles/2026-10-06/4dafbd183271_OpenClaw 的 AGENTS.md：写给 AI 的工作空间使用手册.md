---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 40702
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

AGENTS.md 是一个正在被主流 Agent 工具逐步采纳的约定：在仓库或工作区根目录放一个同名 Markdown 文件，Agent 接任务时会自动读取，作为对当前环境的先验知识。一句话概括：README 是写给人的，AGENTS.md 是写给 AI 的。

在 OpenClaw 里，agent 加载 workspace 时会把它注入上下文，相当于每次会话都预置了一份项目使用说明。它和 MCP 是互补关系：MCP 告诉 agent 有哪些工具可用，AGENTS.md 告诉它这个工作区里该怎么用。

## 问题

没有这份文件时，agent 每次都得靠猜：

- 构建用 make 还是 npm script？测试命令带不带 watch？
- 哪些目录是生成物不能改？哪些配置有副作用？
- 团队的命名、提交、目录约定，agent 从代码里推断，经常推错。

结果就是每个会话重复解释一遍，或者更糟——agent 自信地跑了错误命令。多人协作时，每个人在聊天里“调教”agent 的经验无法沉淀，换个人、换个会话就归零。

## 做法

1. **放在对的位置**。workspace 根目录建 `AGENTS.md`；monorepo 可以在子目录再放一份，靠近被改代码的那份优先生效。
2. **按 agent 的决策点组织**，而不是按文档目录组织。骨架大致是：

```markdown
# 项目概览
一两句话：这是什么、技术栈、入口在哪。

# 常用命令
构建：pnpm build
测试（单文件）：pnpm test -- path/to/test

# 约定
- src/generated 下是生成物，禁止手改
- 提交信息遵循 conventional commits
- 改 db schema 前先确认迁移脚本

# 禁区
- 不要动 deploy/，该目录改动需人工 review
```

3. **控制篇幅**。上下文是预算，几十行、100 行以内最好，重要规则放前面。
4. **迭代维护**。agent 犯了重复错误就加一条；某条规则被稳定执行或已过时就删掉。随 PR 一起改，走 code review。

## 踩坑点

- **写成给人看的散文**。“本项目致力于……”这类内容对 agent 是噪音，要用祈使句写可执行约束。
- **抄 lint/CI 已经覆盖的规则**。AGENTS.md 的甜蜜点是“linter 抓不到、但 agent 需要知道”的事，其余交给工具链。
- **指令太模糊**。“写高质量代码”无法执行；“错误信息必须带上下文字段”才行。
- **长文件稀释注意力**。塞了 500 行等于没写，关键规则会被淹没。
- **过期信息比没有更糟**。命令改了没更新，agent 连续失败几次后就会开始自由发挥。
- **指望 agent 点链接**。关键信息直接内联，超链接是给人的。

## 可复用建议

- 判断标准：同一条指令你在聊天里说过两次以上，就该进 AGENTS.md。
- 只写 agent 推断不出来、或一推断就错的内容，其余删掉。
- 把骨架做成团队模板，新项目复制起步，成本几乎为零。
- 它同时是给新人看的 onboarding 材料——能让 agent 少犯错的说明，通常也能让人少犯错。

## 总结

AGENTS.md 的价值不在“多写了一个文件”，而在于把人与 agent 的协作约定变成可版本化、可 review、可迭代的资产。别追求一步到位的结构设计，先写一个 20 行的最小版本跑起来，用真实的失败案例反向补充规则，两三周后它就会成为这个工作区最值钱的一页纸。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/3030601696d712d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/66635b43cba2c85b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/850c55eaeb35ec17.png)

