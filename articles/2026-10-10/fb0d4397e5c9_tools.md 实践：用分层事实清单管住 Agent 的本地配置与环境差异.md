---
title: tools.md 实践：用分层事实清单管住 Agent 的本地配置与环境差异
feedId: 41106
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

跑 Agent 的人很少只有一台机器。笔记本、家里的小主机、一台 VPS，再加 CI。同一套 OpenClaw 工作流搬过去，ffmpeg 的路径变了，docker 没装，代理只在公司网络里存在。写在系统提示词和脚本里的隐性假设，换台机器就开始碎。

## 问题：三种常见应对都不太对

1. **写死一份配置**，只在主跑的机器上能用，换环境必炸；
2. **写巨型通用配置**，塞满"如果是 mac 就……如果是 Linux 就……"，上下文膨胀，Agent 还经常判断错分支；
3. **让 Agent 每次自己探测**。能跑，但每轮会话浪费几个 turn 在 `which` 和 `--version` 上，结果还不稳定。

本质问题：环境事实没有被当作一份"随机器走的资产"来管理。

## 做法：分层事实清单

tools.md 的思路很简单——把"这台机器有什么、没什么、放在哪、怎么用"写成显式的事实文件，分两层维护：

- **仓库层**：放 `tools.md.example`，只写所有环境共通的基线：通用约定、scratch 目录、测试怎么跑。走 PR review，和代码同权重。
- **机器层**：每台机器一份 `tools.local.md`，gitignore 掉，写机器相关事实：路径、版本、缺什么、代理、权限。
- 启动时用一个十几行的合并脚本把两层合成 resolved 文件，Agent 配置只指向它。本地条目覆盖基线，基线条目在本地缺失时报错而非静默跳过。

内容长这样：

```markdown
## runtime
- node 22.4 at /opt/homebrew/bin/node
- python 3.12，优先 ~/work/.venv，别用系统解释器
## network
- pip / pnpm 需走代理 127.0.0.1:7890
## not-available
- 无 docker，容器一律 podman
- 无 sudo，装东西用 brew
## conventions
- scratch 统一放 /tmp/agent-scratch
```

三条内容原则：

1. **只写事实，不写指令**。"podman 4.9 at /usr/bin/podman" 是事实；"尽量用容器跑"是指令，放 Agent 的行为约定里，别混。混了之后两边都没法维护。
2. **负向约束最值钱**。"本机没有 docker，别尝试"一句话，省掉无数次失败重试。`not-available` 区块建议第一个建。
3. **能自动生成的别手写**。写个小脚本对固定工具列表 dump 路径和版本，填进 inventory 区块；手写只留脚本探不出来的部分：venv 选哪个、MCP server 哪个本机起不来、哪个目录只读。

再配一个 doctor 脚本：解析 tools.md 里所有路径和工具名，逐个验证存在性和版本，输出漂移报告。升级完工具链跑一次，十秒发现"文档写 ffmpeg 6.0，实际 7.1"。

## 踩坑点

- **写成对话体**。"你可以试试……"这种表述维护两周就没人看了，事实清单要像 `/etc/hosts` 一样干燥。
- **机器路径提交进共享仓库**。炸别人的环境，泄漏 home 目录结构，隐私一起带出去。
- **越写越长**。这文件每个会话都进上下文，目标控制在 1–2K token。过期条目比缺失更糟——Agent 会信它，然后在旧路径上白挖半小时。
- **往里塞密钥**。它会进日志和上下文缓存。写变量名和来源（"见 ~/.secrets/env"），不写值。
- **忘了更新**。inventory 区块标注生成时间，doctor 里加 mtime 检查，超过阈值就提醒。

## 可复用建议

- 区块骨架固定下来：`runtime / package managers / network / permissions / not-available / conventions`。新机器从 example 复制，跑 doctor 补空，十分钟内收工。
- 每次因为环境差异 debug 超过五分钟，回来加一条事实——这应该是文件唯一的增长方式，而不是想到什么补什么。
- 基线变更走 diff 可见的合并，本地文件随手改但要能 `git diff --no-index` 对比上次快照。

## 总结

tools.md 不是框架，就是把环境差异从散落在提示词和脚本里的隐性假设，变成一份显式、分层、可校验的机器档案。写死会碎，自动探测太贵，"声明式事实 + 本地覆盖 + doctor 校验"刚好站在这两种极端之间。今天就可以从 example 里加一个 `not-available` 区块开始，收益立竿见影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/948400efcbcf5f56.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/7926498e620997bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/904bc330a1e091c1.png)

