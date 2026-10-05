---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40530
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

跑 Agent 一段时间后会发现，它依赖的远不止模型本身：本地 shell、Python 环境、MCP server、浏览器驱动、各种 CLI，还有散落各处的路径和端口。OpenClaw 会把 workspace 下的 `TOOLS.md` 注入上下文，所以我的做法是把工具清单和环境事实集中写在这里，当作 Agent 的“本机说明书”。

## 问题

tools.md 很容易写坏，常见三种死法：

1. **写死绝对路径**。Mac 上是 `/opt/homebrew/bin/ffmpeg`，到 Linux 服务器变成 `/usr/bin/ffmpeg`，Agent 照文档执行，次次报错。
2. **共享与本地混杂**。用 git 同步 workspace 时，文件里既有团队通用的工具约定，又有你本机的 venv 路径和端口，diff 永远很脏，还容易把 A 机器的配置带去 B 机器。
3. **文档腐烂**。装了新工具、升了版本没人更新，Agent 按过期清单干活，出了错还以为是模型笨。

根因：把“环境事实”当成了静态文档，而环境事实是每台机器各自不同、且持续漂移的。

## 做法

核心思路三个词：**分层、生成、校验**。

1. **分层**。把内容拆成两层：共享层放仓库，只写与机器无关的约定——工具用途、调用方式、必备参数、安全红线；本地层放 `tools.local.md`，只写这台机器的事实：真实路径、版本、env 变量、可用端口。本地层进 `.gitignore`。
2. **叠加**。在共享层里声明：开工前先读本地层，后者覆盖前者。prompt 里永远只引用工具名，不出现绝对路径。
3. **生成而非手抄本地层**。写个探测脚本：`which ffmpeg`、`ffmpeg -version`、`echo $VIRTUAL_ENV`……输出成 markdown 片段。装完新东西跑一次，90 秒，腐烂问题基本消失。
4. **校验**。让 Agent 开工前做一轮 smoke check：清单里每个工具跑一个 `--version`。失败的工具当次会话降级或跳过，而不是硬猜路径。

## 踩坑点

- **密钥绝不进 tools.md**，本地层也不行。只写“凭据在 env `XXX` 中，由 .env 注入”，文件留引用不留值。
- **覆盖顺序要写死**。我踩过一次叠加顺序反了，共享层把本地修正又盖回去，排查了半小时。
- **生成脚本本身要进仓库**，否则换机器没人知道本地层是怎么来的。
- **别复述 MCP 已暴露的工具**。MCP 的工具列表是动态的，tools.md 里写一句“由某 MCP 提供，见其 schema”即可，避免两处真相互相打架。

## 可复用建议

- 共享层当“合同”，本地层当“快照”，生成脚本当“快门”，三者分开维护。
- 任何写进 prompt 的环境信息，先自问：换台机器还成立吗？不成立就下沉到本地层。
- 工具变更只改两处：跑一次生成脚本 + 提交共享层 diff，收工。

## 总结

tools.md 的价值不在“多了一份文档”，而在于把易变的环境事实从 prompt 和代码里剥离出来，用分层对抗混杂、用生成对抗腐烂、用校验对抗幻觉。Agent 出错时，你 diff 两个文件就够了，不用翻十处硬编码。配置管理的老原则，在 Agent 时代照样好使。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/cdc41883067cf6ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/c683f1610219f1b1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/da97a917bf32bae0.png)

