---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39508
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 这类常驻 Agent 和普通脚本最大的区别，是它长期跑在"你的机器"上，会自己执行命令。它表现好不好，很大程度上取决于它对本地环境的认知。`TOOLS.md`（工作区约定文件，通常在 `~/.openclaw/workspace/` 下）就是为此准备的：Agent 需要环境信息时读取它，了解这台机器上有什么、路径在哪、惯用哪种包管理器。

我的 Agent 部署在三处：日常开发机（macOS）、一台家庭服务器（Debian）、一台 VPS。同一个任务"跑一下测试"，三台机器的正确做法完全不同。

## 问题

没有 tools.md 或写法不对时，常见翻车方式：

- Agent 在 macOS 上用 `apt`，在 Debian 上用 `brew`；
- Python 路径靠猜，包装进了系统环境；
- 假设 docker 存在，实际没装；
- 把所有机器的差异堆进一个文件，写满"如果是 mac 就……如果是 linux 就……"，Agent 读得一知半解；
- 环境升级后文件过期，Agent 按旧事实行动，且不会告诉你它依据的是过期信息。

## 做法

1. **一机一文件，不做条件分支。** 每台机器的 `TOOLS.md` 只描述当前环境。差异在文件级别解决，而不是文件内写 if-else。
2. **只写"可执行的事实"。** 每一条都对应 Agent 能验证的命令：包管理器是 `brew`、Python 用 `~/dev/py311/bin/python`、测试命令是 `make test`。不写散文式介绍。
3. **固定分区。** runtime、包管理、关键路径、常用命令、已知坑（quirks），每区三五行，全文件控制在 60 行左右，省上下文。
4. **用脚本生成初稿。** 十几行 shell 探测 OS、包管理器、主要工具版本，输出 draft，人工删改。机器重装后重跑一次即可。
5. **Git 策略。** 仓库里提交 `TOOLS.md.example` 作模板，真实文件进 gitignore；内网地址、账号路径这类敏感信息绝不写入。
6. **和 AGENTS.md、MCP 配置对齐。** AGENTS.md 里加一行指引"环境信息先看 TOOLS.md"；文件里声称的工具，应与实际启用的 MCP server / 插件一致，避免 Agent 以为自己有某个工具。

## 踩坑点

- **写成了 README：** 大段项目背景介绍，Agent 要的是命令和路径，不是文章。
- **信息过期：** 保留那个探测脚本，怀疑不对时直接让 Agent 重跑核对，比人工回忆靠谱。
- **混入密钥：** token 写进去一时方便，泄漏时全盘完蛋。
- **堆太长：** 超过一屏的内容，Agent 遵循度明显下降。细节移到独立文档，TOOLS.md 只留索引。
- **没被读到：** 路径或命名不符合加载约定，Agent 根本没打开过，等于白写。改完先问一句"你从 TOOLS.md 里看到了什么"验证。

## 可复用建议

- 把 TOOLS.md 当"机器清单"而非"文档"，**生成优于手写**。
- 模板分区固定，新机器五分钟完成初始化。
- 基础设施变更（换 shell、升级 Python、新增 MCP server）的提交里附带更新 TOOLS.md，当作 review 项。
- 跨机器同步的是**模板和生成脚本**，不是具体文件内容。

## 总结

TOOLS.md 的价值不在文件本身，而在于把"环境差异"从 Agent 的猜测变成显式输入。一机一文件、只写可验证事实、脚本生成、模板入库——四件事做下来，Agent 在哪台机器上都表现得像本地人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/0549f86eb6c4fd05.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/611b88b37bd90bbd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/86aeceb348f326eb.png)

