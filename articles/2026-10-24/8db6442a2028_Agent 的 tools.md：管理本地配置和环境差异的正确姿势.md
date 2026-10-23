---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42412
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 这类本地 Agent 的基本盘，是把 workspace 里的 markdown 当上下文读进去：AGENTS.md 约束行为，记忆文件延续会话。但有一类信息经常没地方放——**本地环境事实**。同一份 workspace 往往要在多台机器上跑：开发用的 Mac、家里的 Linux 小主机、一台 VPS。工具链、包管理器、路径、端口，全都对不上。

## 问题

Agent 默认靠现场探测认识环境：`which`、`--version`、试装包、猜路径。结果有两个：

1. **重复消耗**。每次新会话都从头探测一遍，token 和轮次都在烧。
2. **猜错成本高**。在 apt 机器上按 brew 的思路装依赖，把 macOS 的路径当 Linux 用，折腾半天才发现方向反了。

更麻烦的是，这些试错经验散落在各会话的记忆里，不可 review、不可同步。换台机器，一切从零开始。

## 做法

核心思路一句话：**把环境事实从会话里搬出来，落成一个文件。**

1. 在 workspace 根目录建 `tools.md`，按机器分节，字段统一：

```markdown
# tools.md

## laptop-mac (darwin / arm64 / 2025-06 验证)
- python: /opt/homebrew/bin/python3.12，无裸 python
- 包管理: brew；node 走 nvm
- 已知坑: docker 依赖 colima，需先启动

## home-server (debian / x86_64 / 2025-06 首次探测)
- python: /usr/bin/python3.11
- 包管理: apt；禁用 sudo pip
```

2. 在 AGENTS.md 里加一条硬规则：动手前先读 tools.md；找不到当前机器的条目才允许探测，探测结论写回文件。
3. 首次落地可以让 Agent 跑一轮探测生成草稿，人工删改后提交 git。之后多机同步靠 pull，而不是靠各机器的记忆各自长歪。
4. 每条环境事实带验证日期——过期的条目比缺失更危险。

## 踩坑点

- **别写密钥值**。tools.md 会被注入上下文，等于把 secret 摊在模型面前。只写“密钥在哪个路径、怎么取”，不写内容本身。
- **别把流程塞进来**。AGENTS.md 管行为规范，tools.md 只放环境事实。两者混写，文件很快变成没人维护的大杂烩。
- **机器标识要稳定**。用 hostname 或固定机器名做 key。标识一变，条目错配，Agent 拿着 A 机器的说明书操作 B 机器。
- **控制篇幅**。一屏以内、条目化。写成百科全书，Agent 实际只会用到前几行。

## 可复用建议

- **闭环习惯**：每次 Agent 修好一个环境问题，让它顺手按固定格式写回 tools.md。走 commit 的话，改动在 diff 里一眼可见。
- **字段即契约**：工具名、路径、版本、坑，四项固定，diff 才有意义，也方便写脚本校验。
- **团队场景当基础设施代码对待**：改 tools.md 走 PR，不走口头。它是文档，更是配置。

## 总结

tools.md 做的事情很小：给 Agent 一份可版本化、可 review 的环境说明书。成本是十几行 markdown，收益是首次命中率明显上升、跨机器行为一致、经验沉淀从“玄学”变成“资产”。与其反复调 prompt，不如先把事实写清楚——这通常是更划算的那一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/238950fc18b873c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/e29a039eafca313e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/6f42d94d533ef23a.png)

