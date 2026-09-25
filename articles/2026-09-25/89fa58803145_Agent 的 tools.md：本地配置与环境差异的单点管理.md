---
title: Agent 的 tools.md：本地配置与环境差异的单点管理
feedId: 38933
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

OpenClaw 的 agent workspace 里有几份会被注入上下文的 Markdown：AGENTS.md 管行为约定，SOUL/IDENTITY 管人格，而 `tools.md` 的定位最朴素——告诉模型“在这台机器上，事情是怎么运转的”。它不定义能力（那是 skills 的事），只描述环境事实：包管理器、路径、网络、容器、禁区。

问题往往出现在第二台机器接入之后。

## 问题

典型场景：笔记本和家里的服务器都跑 OpenClaw，workspace 用 git 同步。某天 agent 在服务器上执行 `npm install -g`，而那台机器明明约定用 pnpm；或者它在容器里找不到 Docker socket，反复重试。根因都一样：**环境知识散落在 skill 描述、历史记忆和你的临时口头纠正里，没有一个权威单点。**

反过来也有坑：把所有机器的细节塞进一份共享的 tools.md，同步之后路径互相污染，agent 每次都要猜“这条适用于哪台机器”。

## 做法

我的分层原则一句话：**可跨机器复现的进 skills，机器特异的进 tools.md，tools.md 不跨机同步。**

1. **每台设备一个独立 workspace**，用 git 管理，但 `.gitignore` 掉 `tools.md`（或放 private 分支）。
2. **按固定小节写，保持“事实 + 约束”句式**：
   - 包管理：`本机用 pnpm，禁止全局 install`
   - 路径：`项目根在 ~/work，输出统一落 ~/work/out`
   - 网络：`curl 需走本机 7890 端口代理`
   - 容器：`Docker 为 rootless，socket 在用户目录下`
   - 禁区：`不要动 /etc/hosts、不要重启 nginx`
3. **敏感信息写引用不写值**：`数据库连接见环境变量 DB_URL`。上下文会发给模型端，tools.md 不是保险箱。
4. **环境一变就更新**，和改 dotfiles 同一个习惯。我把它当 runbook 的一部分，改动走 commit。
5. **验证方式很土但有效**：新起会话问 agent“根据 tools.md，本机装依赖用什么命令”，看它能否复述正确。

## 踩坑点

- **写太长**。tools.md 每轮都吃 token，超过一屏就该砍。agent 用不到的背景故事（比如你为什么从 zsh 换回了 bash）没有价值。
- **写成教程**。要的是指令不是文章，“用 pnpm” 比 “pnpm 是一个快速的包管理器……”有用得多。
- **和 skills 重复**。skill 里已写死 `pnpm install`，tools.md 再强调一遍，将来一改就漂移。约定放一处，另一处引用。
- **多机共享一份**。我曾图省事把服务器那份同步给笔记本，agent 半个月一直在找不存在的路径。
- **没写优先级**。AGENTS.md 说 A、tools.md 说 B 时 agent 会猜。显式加一句：“本文件与 AGENTS.md 冲突时，以本文件为准（仅限本机操作）”。

## 可复用建议

- **固定五段模板**：包管理 / 路径 / 网络 / 容器与硬件 / 禁区。新机器五分钟能填完，降低“懒得写”的门槛。
- **公共骨架进 dotfiles 仓库**，克隆新机后只补机器差异，不从头手写。
- **每季度删一次**：过期的约束比没有约束更危险，因为它带着“权威”的语气误导模型。

## 总结

tools.md 解决的不是能力问题，而是一致性问题：把每台机器的脾气收敛到一个短小、私有、随环境演进的注入点。原则就三条——机器特异才写进去、写得像配置不像作文、永不跨机同步。做到这三条，同一个 agent 在笔记本和服务器上，才算得上是“同一个人”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/5564c72c344cedbf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/bfc3db6871986e23.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/08fb455a8f796a9a.png)

