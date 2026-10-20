---
title: Agent 的 tools.md 实战：管理本地配置与环境差异的正确姿势
feedId: 42075
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

OpenClaw 这类常驻 Agent 大多不是跑在云端，而是跑在你自己的机器上：MacBook、家用 NAS、一台 VPS。每台机器的 OS、包管理器、Python 版本、代理配置、装没装 ffmpeg，全都不一样。而模型对“你的环境”的全部认知，只来自注入上下文的文本。

OpenClaw 给了两个注入点：AGENTS.md 偏行为约定（怎么干活、什么不能碰），tools.md 则留给本地环境事实——路径、版本、工具的坑。这个分工没拉开，问题就从这里开始。

## 问题

我见过三种典型翻车：

1. **全写进 AGENTS.md 并用 git 同步多机。** Mac 上的 brew 指令被带到了 Debian 上，Agent 在 apt 环境里坚持用 brew 装东西。
2. **什么都不写，每次会话口头纠正。** 同一个 venv 路径说了十遍，新会话照忘，token 白烧。
3. **写了，但和现实漂移。** 升级 Python 3.12 之后文件里还是 3.10，Agent 按旧版本选写法，排查半天。

本质矛盾：环境事实是“每台机器一份、会随时间过期”的知识，既不能全局共享，也不该硬编码进任何提示词。

## 做法

我的做法压缩成四步：

**1. 先搞清加载机制。** tools.md 的内容会被拼进 Agent 的工具上下文（具体以你所用版本的加载逻辑为准）。它是“每台机器独立维护”的文件，不要进同步仓库——或者进仓库但只放模板。

**2. 只写事实，不写行为。** “不要动生产库”这类规则归 AGENTS.md；tools.md 只放环境事实。我现在的模板长这样：

```
# machine: macbook-pro (macOS 15, arm64)
- package manager: brew; python via pyenv (3.12)
- venvs in ~/venvs/<project>, activate before pip
- proxy: http on 127.0.0.1:7890, needed for pip/npm
- ffmpeg installed; no docker on this machine
- verify: which ffmpeg && python --version
```

最后那行验证命令是关键：不确定时让 Agent 先跑再假设。

**3. 模板进 git，实例留本地。** dotfiles 仓库里放一份 `tools.md.template`，每台机器首次部署时填一遍。既不泄密，也不会把 A 机器的事实污染到 B。

**4. 当 runbook 维护。** Agent 因环境问题犯错时，别在对话里跟它辩，改 tools.md。改完顺手跑一遍验证命令，确认文件没撒谎。

## 踩坑点

- **写成小作文。** tools.md 每个会话都占上下文，我给自己定的上限是 30 行。超了说明混入了项目级信息，那些该进项目自己的 AGENTS.md 或 docs。
- **提交密钥。** 代理地址、token 一旦写进同步文件就是事故。只写“需要代理，端口见本机配置”级别的描述。
- **硬编码用户路径。** `/Users/alice` 换个用户就废，能用 `~` 就用 `~`。
- **只写不验。** 过期的事实比没有更糟——模型会自信地引用它。关键条目后附验证命令，大版本升级后强制自查一遍。
- **当万能补充用。** 它是环境备忘，不是技能文档。工具怎么用靠 MCP/插件自身的描述；你的本地差异才进 tools.md。

## 可复用建议

- 一台机器一个文件、一个 dotfiles 仓库一份模板，是最小可维护结构。
- 三个分区够用：**运行时**（OS/语言/版本）、**网络**（代理/镜像源）、**已知坑**（本机特有问题）。
- 大版本升级后花五分钟过一遍，删掉没用过的条目——上下文预算很贵。
- 多机高度相似时，提取公共段 + 机器差异段再拼接，避免三份文件改三遍。

## 总结

tools.md 的价值不在“多写”，而在**写对的东西、放在对的机器、保持新鲜**。它本质上是给 Agent 的一页环境 runbook：事实与行为分离、模板共享而实例本地化、关键事实可验证。做到这三点，Agent 在你每台机器上都能少问几次、少错几回，调试时间是实打实降下来的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/89027f91533d33fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/a93e78e036873d93.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/7192f66e5ce10015.png)

