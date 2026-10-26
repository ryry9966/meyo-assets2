---
title: Agent 的 tools.md 实践：用分层声明管住本地配置与环境差异
feedId: 42635
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

OpenClaw 的 agent 每次会话都会读取 workspace 里的固定文件，`TOOLS.md` 是其中之一，定位很明确：告诉 agent“这台机器上工具的真实情况”。MCP 和插件解决的是“工具怎么被调用”（schema、参数、返回值），而 TOOLS.md 补的是另一层：本机路径、版本、代理、配额、团队约定——这些 MCP 不会声明、模型只能靠猜的东西。

## 问题

不维护这层信息，典型症状有三个：

1. 提示词里写死 `/usr/local/bin/xxx`，换台机器或换个 agent 实例就断；
2. 工具装了 agent 不知道，或 agent 假设了实际不存在的能力——前者浪费，后者幻觉；
3. 多 agent 部署时（网关上挂几个不同 workspace），共享一份配置，环境差异互相污染。

## 做法

核心思路是**分层 + 固定结构 + 自动生成**，三步：

**1. 分层。** OpenClaw 支持每个 agent 独立 workspace，天然形成两层：

- 全局层：`~/.openclaw/workspace/TOOLS.md`，放所有 agent 通用的约定（代理地址、禁用项、安全红线）；
- agent 层：各 agent 自己 workspace 下的 `TOOLS.md`，放差异项（专属路径、设备、权限）。

覆盖规则：agent 层优先于全局层，同名条目以 agent 层为准。

**2. 固定条目结构。** 每个工具一个段落，按“状态 / 路径 / 约定”三行写：

```markdown
## ffmpeg
- status: available (v6.1, homebrew)
- path: /opt/homebrew/bin/ffmpeg
- notes: 抽音频用 -vn -acodec libmp3lame；并发不要超过 4 线程
```

关键是只写“这台机器上它怎么接”，不写“这个工具能干什么”——后者属于 MCP 和 tool schema 的职责。

**3. 生成与校验。** 手写必漂移，配两个脚本兜底：

- `probe.sh`：遍历你关心的工具（是否存在、版本、关键路径），生成 TOOLS.md 草稿，装了新东西跑一遍即可；
- `doctor.sh`：会话启动前或 CI 里比对声明与实际，不一致就报错。

## 踩坑点

- **别写密钥。** TOOLS.md 会随 workspace 备份和同步，token 一律走环境变量，文件里只写“从哪个环境变量读”。
- **控制长度。** 这份文件每个会话都占上下文，高频工具详细写，低频的一句话带过，总量尽量压在几十行内。
- **Windows/容器差异显式记。** `npx` 与 `npx.cmd` 的区别、路径分隔符、容器里没有 systemd——踩过一次就沉淀进去，这才是它最大的价值。
- **别和 MEMORY.md 混用。** 一次性的排障过程写 MEMORY，“长期成立的环境事实”才进 TOOLS.md，混了两边都会被稀释。

## 可复用建议

- 把 TOOLS.md 模板和 probe 脚本放进项目脚手架，新人 clone 后跑一次即得本地版，入职成本从“口口相传”变成“一条命令”；
- CI 加一个 job：按声明装依赖、跑 doctor，防止声明悄悄过期；
- code review 把 TOOLS.md 当代码看：改了依赖但没同步更新它，视为不通过。

## 总结

TOOLS.md 的本质，是把 agent 对环境的认知从“模型猜测”变成“显式声明”：用 workspace 分层消化差异，用生成脚本消灭手写漂移，用校验脚本防止声明过期。它不替代 MCP 和插件，而是补上“同一份 agent 配置在不同机器上行为不一致”这块最脏的空白。半小时搭好分层和脚本，之后每次换机器、加 agent、新人入职，都在替你还债。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/b3febc611f33f391.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/f8262ce0301ad931.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/5e61d22d9587dcba.png)

