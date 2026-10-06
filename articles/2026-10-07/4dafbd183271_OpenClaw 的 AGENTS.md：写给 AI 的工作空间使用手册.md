---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 40716
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的 Agent 接入一个工作空间后，真正费 token 的往往不是调工具，而是每轮对话都在重新"猜"这个项目怎么运作：测试怎么跑、临时文件放哪、哪些目录不能碰。AGENTS.md 就是解决这个问题的社区惯例——在工作空间根目录放一份 Markdown，Agent 每次进入会话都会先读它。本质上，它是给模型看的新人 onboarding 文档，读者是模型，不是人。

## 没有它会怎样

不写 AGENTS.md 的团队，通常很快遇到这些现象：

- Agent 用全局 Python 而不是项目指定的 `uv run` 跑测试，环境错乱；
- 生成的草稿文件散落在仓库根目录，混进 `git status`；
- 每次都要口头重复"别动 migrations"，换个会话就失效。

这些约束写进 system prompt 维护成本高，贴在对话里又容易漂移。AGENTS.md 的优势在于：随工作空间走、可版本化、可 review。

## 怎么写

OpenClaw 支持层级合并：根目录放全局约定，子项目可再放一份，Agent 按"就近覆盖"读取。内容上建议每条都是可执行指令，而不是介绍性散文：

```markdown
# AGENTS.md（节选）
## 项目地图
- src/core/ 是核心逻辑；dist/ 是构建产物，禁止修改
## 命令
- 测试：uv run pytest -q（不要用裸 pytest）
- Lint：uv run ruff check .
## 禁区
- 不 push 到 main；临时文件放 /tmp/workspace
## 失败处理
- 测试挂了先贴完整报错，不要自行回滚代码
```

三条原则：

1. 长度控制在百行以内。手册越长，关键指令在上下文里越容易被稀释。
2. 子目录与根目录冲突时写清优先级，例如"本目录规则覆盖根目录的测试命令"。
3. 工具的用法也写进去：某个 MCP server 的限流、内部插件的前置条件，都是模型不可能自己知道的。

## 踩坑点

- **写成 README 复制品**。全是架构介绍、没有指令，Agent 读完等于没读。
- **命令过期没人管**。手册里是半年前的命令，Agent 照跑必挂。把"同步更新 AGENTS.md"纳入 PR checklist。
- **写入敏感信息**。它会进模型上下文，也常被 commit，密钥和内网地址绝不能放。
- **只增不删**。历史决策堆满文件后，模型反而抓不住重点，定期清理失效条目。

## 可复用建议

把 AGENTS.md 当代码对待：进版本库、走 review、指定 owner。验证方法很朴素——新开一个干净会话，只给 Agent 一个任务，观察它是否按手册行事。没做到，先改手册，而不是急着换模型。

迭代节奏可以遵循一个判据：Agent 犯的错，如果根因是"它不知道"，就补一条进 AGENTS.md；如果是"它知道但不听"，才轮到调 prompt 或工具层。多数日常问题属于前者，这也是这个文件性价比最高的原因。多个项目的团队可以把通用约定做成模板，用脚本或软链接分发，项目差异部分各自维护。

## 总结

AGENTS.md 是把团队隐性规范显性化、并稳定注入 Agent 上下文的最小改动。成本是一个 Markdown 文件，收益是 Agent 行为可复现。它不需要一次写完——把每一次"AI 又猜错了"都当成补手册的素材，让它随项目一起生长，就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/b989c167e25da935.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/ebfb02cca58b90f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/010c0d18d6b2869e.png)

