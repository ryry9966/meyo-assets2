---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40291
source: 综合讨论
publishedAt: 2026-10-03
---

# Agent 的 tools.md：管理本地配置和环境差异的正确姿势

## 背景

跑 OpenClaw 或任何带工具调用的 Agent，最容易翻车的环节不是模型，是"机器"。模型不知道你的 Python 装在哪、包管理器是 brew 还是 apt、有没有代理、服务怎么启停。这些信息散落在系统提示词、README 和口头约定里，换一台机器就全对不上。

我们的做法是给每个工作区维护一个 `tools.md`：一份写给 Agent 看的本地环境事实清单。

## 问题

没有它时，常见三种翻车：

- **Agent 猜环境**：在 Arch 上试 apt，在没装 conda 的机器上反复 `conda activate`，一轮轮试错烧 token；
- **信息漂移**：提示词里写死 Node 18，实际已升到 22，Agent 按旧版本选 API 直接报错；
- **多机不一致**：笔记本、家里服务器、CI 三套环境，同一份 prompt 行为完全不同，排查全靠猜。

本质上，环境信息是配置，不该写死在 prompt 里，也不该靠 Agent 现场探索——两者都会失败，只是失败得贵一点还是便宜一点。

## 做法

1. **盘点**。只写 Agent 真正会用到的：运行时和版本、关键二进制路径、包管理器、服务启停方式、网络与代理、沙箱/权限限制。用不到的不写。
2. **写事实，不写教程**。一行一条，可被命令验证。例如：
   - `python: /usr/bin/python3 (3.11)，虚拟环境用 uv 管理`
   - `docker compose v2，免 sudo 可用`
   - `出网需走 127.0.0.1:7890`
   指导 Agent"怎么思考"的内容不要放这里，那是 system prompt 的事。
3. **分离稳定与易变**。路径、包管理器相对稳定；版本号、端口易变。易变部分交给 bootstrap 脚本生成：跑一遍 `python --version`、`node -v` 等探测命令，输出快照覆盖对应段落，文件头加生成时间。
4. **多环境用分层覆盖**。`tools.md` 作公共基底，主机差异放 `tools.local.md`（进 gitignore）或 `tools.<host>.md`，会话启动时合并。不要写一个巨型文件让模型自己做 if-else。
5. **接入会话**。初始化时把 tools.md 注入上下文，并要求 Agent"先验证后信任"：对关键事实跑一条廉价探测命令再使用。密钥只写变量名（如 `API_KEY 在环境变量`），值进 .env。

## 踩坑点

- **写成散文**。三大段背景介绍会稀释模型注意力，事实清单控制在 50 行以内。
- **塞密钥**。tools.md 会进 git、会进上下文日志，任何明文凭证都是事故。
- **烂掉比没有更糟**。环境升级后没更新，Agent 会"自信地"按过期信息操作。生成时间戳 + 定期重跑 bootstrap 是底线。
- **多份真相**。README、安装脚本、tools.md 各说各话。让 tools.md 做唯一权威，其他文档引用它或由它生成。
- **吃上下文预算**。文件膨胀后每次会话都白烧 token。按领域拆分（tools.md / services.md），按需加载。

## 可复用建议

- 定一个最小模板：运行时、包管理、路径、网络、服务、限制六个段，固定顺序，团队通用。
- bootstrap 脚本进仓库，CI 里也跑一份，保证快照可复现。
- tools.md 进版本控制，改动走 review——它改一行，Agent 行为就变一行，本质是代码。
- 排障时先看 tools.md 的生成时间，再怀疑模型。一半的"AI 不好用"，其实是环境描述过期。

## 总结

tools.md 不是写给人看的文档，而是你和 Agent 之间的环境接口契约。像对待配置即代码一样对待它：可生成、可版本化、可验证。做到这三点，同一套 Agent 配置在笔记本、服务器和 CI 上的行为就能收敛到一致。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/77375a533bb7a4ac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/6d2751cbae5f052f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/ffc3799399976ab3.png)

