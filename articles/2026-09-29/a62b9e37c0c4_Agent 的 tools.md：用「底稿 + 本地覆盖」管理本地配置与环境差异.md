---
title: Agent 的 tools.md：用「底稿 + 本地覆盖」管理本地配置与环境差异
feedId: 39375
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

跑 Agent 的机器往往不止一台：日常开发机、家里的 homelab、CI 容器。每台机器上可用的 MCP server、脚本路径、shell 类型、代理设置都不一样。OpenClaw 的 agent 启动时会读取 tools.md 来了解「我能用什么、约定是什么」，这份文件实际上是 agent 与本地环境之间的契约。

## 问题

常见做法有三种，都有问题：

1. 全部写进 tools.md 并提交到仓库——换台机器路径就失效，同事 clone 下来跑不通；
2. 把机器细节写进 system prompt——散落各处，无法 review，改一次动全身；
3. 靠会话里临时口头约定——agent 每次启动都重新「猜」环境，行为不可复现。

更隐蔽的是密钥问题：为了让 agent 能调某个内部工具，有人直接把 token 写进文档，然后整份文件进了 git。

根因是：**「能力约定」和「机器事实」被混在了一起。**

## 做法：两层拆分 + 启动校验

把 tools.md 拆成两层：

**第一层：tools.md（仓库内，提交）**。只写与机器无关的能力契约：有哪些工具类别、优先用哪个 MCP server、调用时的团队约定（比如「大文件先分块」「不要直连 prod 库」）。用表格，控制在几百 token 内。

**第二层：tools.local.md（本地，gitignore）**。只写机器事实：python 路径、docker socket 位置、代理端口、shell 是 bash 还是 pwsh、本机 GPU 情况。仓库里放一份 `tools.local.example.md` 作模板。

合并顺序在 agent 的启动指令里写死：`base → local → 会话临时覆盖`，后者覆盖前者。

再加一个兜底：写一个 `doctor` 技能（或十几行脚本），启动后抽查 tools.local.md 里的关键声明——路径是否存在、MCP 端口是否可达——不一致就输出差异清单。文档保持诚实靠的是校验，不是自觉。

具体迁移步骤：

1. 盘点 agent 实际会调用的东西（shell、MCP、插件、自定义脚本）；
2. 把现有 tools.md 按「能力 / 事实」拆成两份；
3. local 文件加入 .gitignore，example 模板入库；
4. 启动 prompt 里明确合并顺序；
5. 跑一遍 doctor，把差异修掉。

## 踩坑点

- **把 MCP schema 抄进 tools.md**。工具参数描述 server 自己会暴露，文档里重复一遍只会过期。tools.md 该记的是「约定和怪癖」：哪个 server 更快、哪个有速率限制、哪个只在特定网络可用。
- **local 文件里写密钥明文**。即使不提交，也建议只写环境变量名，值放 .env 或系统钥匙串，agent 拿到变量名自己去读。
- **base 文件里出现绝对路径**。看到 `/home/xxx/...` 出现在提交的文件里，就该挪到 local 层。
- **文档写成散文**。agent 每次会话都要读它，长文档是持续的成本。表格 + 短句，删掉一切背景介绍。
- **忽略 shell 差异**。pwsh 和 bash 的管道、引号行为不同，shell 类型必须显式写进 local。

## 可复用建议

- 判断标准一句话：**换一台机器，这句话还成立吗？** 成立 → base，不成立 → local。
- 这套分层可以直接推广：团队共享 base，CI 和 homelab 各自一份 local 覆盖；新人入职 = 拷模板 + 跑 doctor + 修差异。
- 把 base 文件的修改纳入 code review——它和代码一样影响 agent 行为。

## 总结

tools.md 不是文档，是配置。用「能力底稿 + 本地覆盖 + 启动校验」三层把它管起来，agent 在哪台机器上都能拿到一致且真实的环境认知，你的仓库里也不会再出现第二个 token 泄漏事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/3fb62629633a2361.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/f8e92360264666fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/5c22c75803fdcce2.png)

