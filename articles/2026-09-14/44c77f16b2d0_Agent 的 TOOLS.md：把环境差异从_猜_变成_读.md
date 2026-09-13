---
title: Agent 的 TOOLS.md：把环境差异从"猜"变成"读
feedId: 37424
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

跑 OpenClaw 或同类 Agent 的人，大多不止一台机器：macOS 笔记本、家里一台 Linux 服务器、可能还有台云上的 VM。同一套指令在不同机器上表现不一致，十有八九不是模型的问题，而是环境差异：Python 版本、包管理器、路径约定、代理、权限。

OpenClaw 的 workspace 约定里，`AGENTS.md` 定义 Agent 的行为准则，`TOOLS.md` 则用来描述"这台机器长什么样"。这个区分看起来琐碎，但恰恰是多机使用体验的分水岭。

## 问题

常见的三种错误姿势：

1. **环境信息写进 AGENTS.md 或系统提示**。换台机器就失效，还把机器无关的行为规则污染了。
2. **靠 Agent 每次现场探测**。`which python`、`ls ~/` 跑一轮，浪费轮次不说，结果还可能是错的——conda base、pyenv shim、shell alias 都会让探测结果骗人。
3. **硬编码在脚本或 prompt 里**。环境一变就静默出错，比如 `/usr/bin/python3` 在两台机器上指向不同版本。

本质问题：环境事实是"这台机器的属性"，既不该和行为准则混在一起，也不该每次会话都重新发现一遍。

## 做法

一份好用的 TOOLS.md 大致长这样：

```markdown
# 本机环境

## 系统概览
- OS: Ubuntu 22.04 (WSL2)
- 包管理: apt + uv，禁止 pip 全局装包

## 路径约定
- 工作区根: ~/work
- 项目数据: ~/work/data，勿写到其他位置

## 常用命令
- 跑测试: make test
- 重启服务: systemctl --user restart gateway

## 已知坑
- python3 是 3.10，项目要 3.12，一律用 uv run
- 出网走 http://127.0.0.1:7890 代理
```

落地步骤：

1. **一次性盘点**。自己跑一遍探测命令，人工确认结果后写进文件。不建议让 Agent 直接生成第一版——它探到什么写什么，容易把 shim 和 alias 当真相。
2. **分层**。判断标准就一条：换一台机器，这行还成立吗？成立就放 AGENTS.md，不成立放 TOOLS.md。
3. **每台机器一份**，随 workspace 留在本地。可以用 dotfiles 或私有仓库同步模板，但保留机器间的差异。
4. **版本化 + 更新习惯**。环境变了顺手改文件，不靠记忆。

## 踩坑点

- **把任务说明写进 TOOLS.md**。它是环境事实清单，不是使用手册，混写会稀释关键信息的权重。
- **写了不维护**。过期的 TOOLS.md 比没有更糟，Agent 会非常自信地执行错误路径。
- **一股脑全写**。文件越长，关键条目越容易被淹没。只写 Agent 真会用、且探测容易出错的项。
- **明文 token 写进文件**。写"凭证存放在 `~/.config/xxx`，通过环境变量读取"即可。
- **放任 Agent 自动重写**。建议它只提案 diff，人确认后落盘，否则文件会被悄悄改走样。

## 可复用建议

- 一句口诀：AGENTS.md 管"怎么做事"，TOOLS.md 管"这里是什么"。
- 新机器冷启动六项检查：OS、包管理、运行时版本、代理、路径、权限。
- 团队内共享固定字段的模板，各机填值，review diff 很快。
- 每次大版本升级后，让 Agent 通读 TOOLS.md 并逐条验证命令仍可执行，输出差异报告——这比人肉回忆可靠得多。

## 总结

TOOLS.md 的价值不在文件本身，而在于把环境事实变成显式、可版本化、可审查的资产。Agent 不需要聪明到每次都猜对你的机器，它需要一份准确、简短、被持续维护的说明书。花半小时写好第一版，之后每次变更顺手更新，比任何精巧的动态探测方案都省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c20922b56eceb4d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/65a1e604899a0fff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0c82249792fc0a70.png)

