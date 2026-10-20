---
title: tools.md 实践：让 Agent 记住你的机器，而不是每次都猜
feedId: 42070
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

Agent 类工具（OpenClaw、各类 CLI agent、MCP client）的 system prompt 和技能描述是可移植的，但它们运行的环境不可移植。同一份指令，在你的 Mac 上能跑，到 Linux 服务器上就可能因为包管理器、路径、shell 差异直接失败。tools.md 就是用来补这个缝隙的：一份 Agent 每次会话都会读取的本地说明文件，告诉它"这台机器长什么样"。

## 问题

不写 tools.md 时，常见三类失败：

1. **硬编码**：把 `/Users/xxx/...` 写进全局 prompt，换台机器全废。
2. **猜测失败**：Agent 在没有 brew 的 Linux 上反复尝试 brew；在你没装 Docker 的机器上重试 docker 命令，浪费 token 也浪费时间。
3. **环境漂移**：你升级了 Node、挪了工作目录、换了代理端口，Agent 还按旧认知操作，行为不可预测。

## 做法

把 tools.md 当成"机器说明书"分层维护：

- **全局层**：只写跨机器成立的偏好（代码风格、提交规范、常用工作流）。
- **机器层**：每台机器一份，写 OS 与包管理器、关键路径（项目根目录、配置目录）、关键 CLI 及版本、已启用的 MCP server 及启动方式（有的机器跑 npx，有的跑本地二进制）、常驻服务与端口、代理和镜像源、已知怪癖。
- **项目层**：跟仓库走，只写这个仓库特有的命令和注意事项。

写法上守三条纪律：一行一个事实，不写散文；敏感信息不落地——写"凭据在 `~/.config/xxx/creds`，用环境变量注入"，不写凭据本身；每条标注验证日期，过期信息比没有信息更糟。

## 踩坑点

- **写成第二份 README**：超过一屏，Agent 对中间内容的遵从度明显下降。控制在 40 行内，砍掉一切"它本来就会"的内容。
- **层间冲突**：全局和项目文件都定义了构建命令，Agent 随机选一个。约定优先级——项目层覆盖机器层，机器层覆盖全局层，并写进文件开头。
- **机器层进了公共仓库**：路径泄露是小事，凭据泄露是事故。机器层只进私有 dotfiles 仓库，或用模板加本地渲染生成。
- **只加不删**：建议每月让 Agent 自己跑一次审计——"逐条核对 tools.md，报告失效项"，人工确认后更新。

## 可复用建议

一个最小骨架，可直接抄：

```markdown
# machine: macbook-pro (macOS 15, arm64)
# verified: 2025-06-01
- priority: project > machine > global
- pkg manager: brew
- node 22.x via nvm; python 3.12 via uv
- mcp: filesystem, git; start via local binary
- proxy: http://127.0.0.1:7890, git 需单独配置
- quirks: default shell is zsh, alias 在非交互 shell 不可用
```

再加一条工程习惯：把 tools.md 纳入版本控制，diff 就是环境变更记录。Agent 行为异常时，先看 tools.md 最近改了什么，往往比翻日志快。

## 总结

tools.md 的本质不是配置文件，而是给 Agent 的"环境契约"。分层、短小、可验证、有版本——四个词做到，同一份 Agent 配置就能在你的所有机器上表现一致。它解决的不是聪明问题，而是一致性问题，而这恰恰是自动化能否长期跑下去的分水岭。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c2ce375edef6159c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/2baadf793e446cf3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/82550754204a5220.png)

