---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 40134
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景：Agent 进了工作区，但没看手册

过去一年工作方式变了：AI 不再只是补全代码，而是直接进入工作区——跑命令、改文件、提交 PR、通过 MCP 和插件执行自动化。问题是，所有约定（怎么构建、怎么测试、哪些文件不能碰）都是写给人看的。README 面向新同事，wiki 面向团队，而 agent 什么都不读，它只看仓库本身和你的对话。

## 问题

没有统一说明文件时，每个会话都在重新推导上下文：构建命令猜错、手改了生成文件、绕过 lint 直接提交、commit 风格漂移。你不得不在对话里反复纠正，换个会话又重来。规则散落在系统提示、各种 rules 文件和聊天记录里，互相冲突且无人维护。

AGENTS.md 解决的就是这件事：一份放在仓库根目录、面向 agent 的约定文件。主流 coding agent 和 OpenClaw 的 workspace 机制都会自动读取它，不需要额外配置。

## 做法

1. 在仓库根目录建 `AGENTS.md`，控制在 50–150 行。
2. 只写五类内容：
   - 项目三句话概述（agent 只需要定位，不需要介绍文案）；
   - 精确命令：构建、测试、lint，可复制直接执行的那种，不要描述性语言；
   - 目录地图：只标注"不看会踩坑"的部分，比如 `generated/` 禁止手改；
   - 与默认习惯不同的约定：commit 格式、分支命名、错误处理风格；
   - 完成定义：改完代码必须跑通什么才算 done。
3. Monorepo 在子目录放嵌套 `AGENTS.md`，就近覆盖根文件的局部规则。
4. 关键习惯：agent 犯错后，除了修代码，往 AGENTS.md 加一行防止复发。把它当 runbook 迭代，进版本库、走 PR review。

与 MCP 的分工要说清：MCP 管"能做什么"（工具能力），AGENTS.md 管"该怎么做"（工作约定）。工具的参数和调用方式应由 MCP server 自身定义，不要复制一份进 AGENTS.md，否则必然过期。

## 踩坑点

- **写成散文**。"请尽量使用 pnpm"不如 `pnpm test --filter api`。agent 需要可执行的事实，不是建议。
- **太长**。超过一屏，模型会截断或忽略，且段落间容易互相冲突。
- **和 README 重复**。两份文档各自演化，三个月后必漂移。README 里链接到 AGENTS.md 即可。
- **写死环境**。绝对路径、本机端口、密钥相关内容不要进文件。
- **过期比没有更糟**。重构后规则没更新，agent 会拿着旧地图自信地走错路。可以在 CI 加个轻量检查：AGENTS.md 引用的路径必须存在。

## 可复用建议

- 判断标准：同一件事你给两个不同的 agent 或会话讲过两遍，它就该进 AGENTS.md。
- 起步模板五段：Overview / Commands / Structure / Conventions / Definition of Done，各写 5–10 行即可。
- 每次修改保持 PR 尺寸，一条规则一个 commit，方便日后回溯"这条规则为什么存在"。
- 季度过一遍，删掉不再成立的规则——负维护也是维护。

## 总结

AGENTS.md 的价值不在单次会话，而在复利：一次写清，之后每个 agent、每次自动化任务都受益。它本质上是给非人类同事的 onboarding 文档——成本一小时，回报是少掉无数轮"不是这个命令"的纠正。建议现在就给主力仓库补一份，从命令清单开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/3fab03f176753bc1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/c9ba2e1e805806da.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/50bd9d0fe07f6750.png)

