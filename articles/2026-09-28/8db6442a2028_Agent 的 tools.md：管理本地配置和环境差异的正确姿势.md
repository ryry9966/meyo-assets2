---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39221
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 这类常驻 agent 的一个典型工作方式：模型本身并不"住"在你的机器上，它对环境的了解全部来自注入的上下文。训练语料给它的只是"平均世界"——大多数人是 Ubuntu、路径在 /home、用 apt。而你的实际环境可能是 macOS + Homebrew、公司内网代理，或者一台 arm64 的开发板。这些差异如果不显式告诉它，它就只能靠猜。

workspace 里的 tools.md 就是干这个的：一份专门给 agent 读的、关于你本地环境的说明。

## 问题

不管理环境信息的常见症状：

- agent 写出 `apt install`，而你的机器是 macOS；
- 每次会话都要口头重复"我这台是 xx，路径在 xx"；
- 同一套指令在公司机器和家里机器行为不一致；
- 这些信息散落在对话历史、memory、系统提示词里，换机器、多 agent 时无法同步，也没法 review。

本质上这是一个配置管理问题，只是配置的"读者"是 agent。

## 做法

**1. 建文件。** 在 workspace 根目录放 tools.md，它会随 workspace 上下文注入给 agent。

**2. 只写事实，不写教程。** 推荐分区：

```markdown
## System
- macOS 14, Apple Silicon (arm64)
## Toolchain
- Homebrew at /opt/homebrew/bin/brew, no apt
## Network
- 走公司代理，git 操作前先执行代理脚本
## Gotchas
- ffmpeg 是自编译的，在 ~/bin，不在默认 PATH
```

**3. 每条尽量带验证命令。** 比如 `- jq: command -v jq`。agent 不确定时可以自己确认，而不是信一份可能过期的文档。

**4. 分清职责。** 机器级事实放 tools.md；行为规范（"提交前跑测试"）放 AGENTS.md，别混在一起。

**5. 版本控制。** tools.md 进 git。注意：这份文件会被注入模型上下文，token、内网密码绝不写入，走环境变量或密钥管理——写进去就等于交出去了。

**6. 形成反馈闭环。** agent 犯了环境相关的错，修 tools.md，而不是在对话里纠正一次就完。同一类错误出现第二次，说明文档缺条目。

## 踩坑点

- **写太长。** 我见过 200 行的 tools.md，结果 agent 反而漏读关键条目。控制在 30–60 行，超了就说明该拆分或删减。
- **写易变的事实。** DHCP 分配的 IP、临时端口，第二天就过期。要么写稳定标识（主机名、ssh config 别名），要么写"怎么查"。
- **多机共用一份。** 网关管多台机器时按 host 分节，并在每节开头写清适用范围。
- **忘记过期清理。** 机器重装后旧路径残留，agent 会一直用错。给关键条目加"最后验证：2025-03"这类时间戳，定期跑一遍验证命令。

## 可复用建议

- 模板五段：System / Toolchain / Paths / Network / Gotchas，够用且不臃肿。
- 基础事实用脚本生成：`uname -a`、`echo $SHELL`、`command -v` 一把梭，人工只补 Gotchas。
- 团队共用环境的，把 tools.md 的 diff 放进 code review，环境变更和代码变更一样可追溯。

## 总结

tools.md 的价值在于把隐性环境知识显性化、版本化、可验证化。原则就三条：短、准、稳。它是给 agent 的"环境说明书"，不是给人看的文档——写每一条时问自己一句：这条信息能帮 agent 少猜一次吗？不能就删掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/815a2ff18c8b6e1d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/a1e071190a65bda5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/78db223cf5c98f3d.png)

