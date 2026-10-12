---
title: 别让 Agent 猜你的机器：tools.md 管理本地配置与环境差异的正确姿势
feedId: 41336
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

OpenClaw 里的 agent 不只是聊天，它会真的在你的机器上动手：装依赖、起服务、调 MCP 工具、跑脚本。但它对"这台机器"几乎一无所知——macOS 还是 Linux、用 pnpm 还是 npm、Docker 装没装、端口被谁占了、MCP server 配了哪几个。它只能猜，猜错就报错，然后你在对话里解释一遍，下个会话再解释一遍。

OpenClaw 的工作区约定里有一个 `TOOLS.md`（工作区根目录），用途就是给 agent 一份本机说明书。很多人把它写成流水账，或者干脆空着，结果环境知识散落在聊天记录里，每次新会话都从零开始踩坑。

## 问题

环境差异导致的失败基本是三类：

- **默认值不对**：agent 按通用习惯给命令（apt vs brew、python vs python3、`docker-compose` vs `docker compose`）
- **上下文丢失**：上次怎么修好的只存在于那一次对话里
- **配置散落**：shell rc、README、聊天记录、脑子里各存一份，互相矛盾

这不是模型能力问题，是信息没放在 agent 每次都能读到的位置。

## 做法

**1. 分层存放。** 整机事实写工作区根目录的 TOOLS.md（OS、包管理器、运行时版本、代理、GPU、全局 MCP 配置）；项目差异写项目内的说明或并入 AGENTS.md（构建命令、端口、测试方式）。两层不要混。

**2. 只写事实和差异，不写教程。** 每条一句话，命令给完整可复制形式：

```markdown
## 环境
- macOS 15, Apple Silicon (arm64)
- Node 由 nvm 管理，先 `nvm use` 再用 node
- 容器：Docker Desktop，compose v2（`docker compose`，无连字符）
- 出网走代理：`export HTTPS_PROXY=http://127.0.0.1:7890`

## 惯例
- 包管理用 pnpm，不要 npm install
- 服务起在 :3000，测试：`pnpm test --run`
- 不要 sudo，不要动 /etc/hosts
```

**3. 给出验证路径。** 告诉 agent：拿不准时先 `which` / `--version` 确认，再执行；确认结果回写进这份文件，形成闭环。

**4. 进版本管理。** TOOLS.md 纳入 git，换机器或重装系统后直接 diff，比记忆可靠。

## 踩坑点

- **写太长**。agent 对超长文档的遵从度会明显下降，控制在几十行，超出部分拆到项目级文件。
- **放机密**。token、密钥绝不进 TOOLS.md，写"从环境变量 X 读取"即可。
- **当成强制机制**。它是提示不是约束，真正要禁止的操作应走权限配置，别指望 agent"读到就不做"。
- **信息过期**。环境升级后不改文档，agent 会理直气壮按旧信息执行。养成"命令失败一次，顺手修一次文档"的习惯。
- **和 AGENTS.md 抢职责**。行为规范进 AGENTS.md，机器事实进 TOOLS.md，一个信息只存一处。

## 可复用建议

- 固定四段模板：环境 / 惯例 / 已知坑 / 验证命令，够用且不臃肿
- 每条尽量附一条自查命令（`--version`、`curl -s localhost:3000/health`），让 agent 能自己核实
- 多机用户按机器各维护一份，在主文件里写明"当前这台是哪种"
- 团队共享时只共享去敏版本，机密一律指向环境变量

## 总结

tools.md 解决的不是模型问题，而是信息架构问题：把"这台机器怎么用"从聊天记录和人的脑子里，搬进 agent 每次都能读到的固定位置。写得短、只写事实、随失败持续更新——坚持一段时间，agent 的执行成功率和你的重复解释次数，都会有肉眼可见的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/a267291d4e850158.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/79342f896d940846.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/9c6a4c6569a022e4.png)

