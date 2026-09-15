---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 37655
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

跑 OpenClaw 一段时间后，大多数人都会遇到同一个局面：同一套 agent 配置，散落在好几台机器上——日常笔记本、家里常开的 mini 主机、一台云上的 VPS。gateway 配置可以同步，但每台机器的"物理现实"没法同步：路径不同、shell 不同、装没装 ffmpeg、MCP server 起没起、Python 是 3.10 还是 3.12。

OpenClaw 的 workspace 里有几个约定文件，其中 AGENTS.md 管"怎么做事"，TOOLS.md 管"这台机器有什么"。把这两者的边界划清楚，就是管理环境差异的核心。

## 问题

不划边界的代价很具体：

1. **工具幻觉**。agent 在 A 机器上学会了用 `yt-dlp`，到了没装的 B 机器上直接调用，报错后开始瞎猜替代方案，越猜越偏。
2. **路径硬编码**。`/Users/xxx/...` 写进系统提示词，Linux 网关上第一次跑就挂。
3. **排查靠试错**。换环境后不知道 agent 能干什么，只能看报错一个个补。

这些问题我全踩过，根因一样：环境事实散落在提示词、配置和历史会话记忆里，没有单一事实来源。

## 做法

**第一步：定分工。** AGENTS.md 放跨机器一致的规则和流程，交给 git 同步；TOOLS.md 只放"这台机器的事实"，原则上不跨机器复用。两条线分开，同步就不会互相污染。

**第二步：按固定结构写 TOOLS.md。** 我现在只用五个段落：

- 系统底座：OS、shell、包管理器
- 已装 CLI 及版本（ffmpeg、node、gh……），各附一句可用场景
- 可用的 MCP server 及启动方式
- 关键路径：工作目录、下载区、日志位置
- 红线：不许动的目录和服务

最后留一行自检命令，比如 `ffmpeg -version`，agent 不确定时先跑再调。

**第三步：能生成的不要手写。** 写个十几行的脚本，dump 各 CLI 版本和路径，追加到 TOOLS.md 的 generated 段，用注释和手写部分隔开。装了新工具跑一次脚本，信息不腐化。

**第四步：把验证习惯写进 AGENTS.md。** 明确告诉 agent："用到文件里没提的工具前，先 `which` / `--version` 探测"。启动时读文件、运行时探测，双保险。

## 踩坑点

- **别放密钥**。TOOLS.md 会进上下文、可能被同步，只写"凭据由 xx 管理"，永远不写 token 本身。
- **控制长度**。这个文件每次会话都吃 token，我的上限是一屏（60 行内），细节外链到别的文档。
- **事实和流程别混**。"怎么转码"是流程，进 AGENTS.md；"本机有 ffmpeg 6.x"是事实，留 TOOLS.md。混了之后多机同步必然出事。
- **描述要可执行**。"有 ffmpeg"不如"ffmpeg 6.1，转码示例：`ffmpeg -i in.mp4 ...`"。可执行的描述 agent 才能直接复用。

## 可复用建议

把 TOOLS.md 当"机器档案"提交到私有仓库，一台一个文件，排障时对照看，比翻 shell history 快得多；换新机器时顺手当装机 checklist 用。一个 gateway 跑多个 agent 时，各 workspace 各自维护，公共事实宁可在 AGENTS.md 里写"环境以 TOOLS.md 为准"，也不要复制粘贴两份。

## 总结

TOOLS.md 的本质，是把环境探测从运行时挪到启动时：用一份人可以 review、机器每次必读的文本，替掉模型的即时试错。改动成本几乎为零，换来的是跨机器行为的确定性。如果你的 agent 已经跑在两台以上机器上，值得今天就划清这条边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/46c7236b3801621a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/514b4ced0b5a5d0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/43a3f9cac29daad0.png)

