---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 43293
source: 综合讨论
publishedAt: 2026-11-03
---

# 背景

Agent 跑了一段时间后，最烦人的往往不是模型能力，而是“这台机器和那台机器不一样”。笔记本上 ffmpeg 在 `/opt/homebrew/bin`，服务器上在 `/usr/bin`；公司内网要走代理，家里不用；同一套 MCP 在 A 机器有 Python 3.11，B 机器只有 3.9。这些差异最后都去了哪里？大部分进了系统提示词、聊天记录，以及某位同事的脑子里。

# 问题

把环境信息直接写进提示词，有三个后果：

1. **提示词膨胀**：每换一台机器就要改一版，改着改着就没人维护了；
2. **环境漂移**：同一份 prompt 在 A 机正常，B 机上 Agent 一执行就报路径不存在，而你不知道它依据的是哪条过时信息；
3. **不可审计**：Agent“以为”的环境和真实环境脱节，且你查不到这些认知的来源。

tools.md 的定位很朴素：**一份给 Agent 读（也给人读）的本地环境清单，可版本化、可 review 的事实来源（source of truth）**。

# 做法

按三层组织，加载时自上而下合并：

- **全局层** `~/.agent/tools.md`：跨项目的机器事实（包管理器、代理、常用 CLI 路径与版本）；
- **项目层** `<repo>/tools.md`：项目专属工具、脚本入口、端口约定；
- **会话层**：临时覆盖，用完即弃。

每个条目固定字段：名称、调用方式、版本约束、前置条件、已知怪癖、验证命令。示例：

```markdown
## ffmpeg
- 调用: `which ffmpeg` 解析，不写死绝对路径
- 版本: >= 6.0（mac 上 brew 装的是 7.x）
- 怪癖: 编码 hevc 需加 `-tag:v hvc1`，否则 QuickTime 打不开
- 验证: `ffmpeg -version | head -1`
- last_verified: 2025-06-12
```

然后让 Agent 在会话开始时读取清单，并加一条硬约束：**只有在实际验证过之后，才允许向 tools.md 追加条目**（比如它装了新工具、探测到了真实路径），diff 提交给你 review。能用环境变量解决的（路径、代理地址），tools.md 里记录“怎么解析”，而不是硬编码值。

# 踩坑点

- **写满绝对路径**。换台机器全废。记录解析方式（`which xxx`、`brew --prefix`）比记录结果可靠得多。
- **放秘密**。tools.md 会进 git，token 一律只写环境变量名，值留在 `.env` 或系统里。
- **让 Agent 随意改写**。不加约束它会把猜测当事实写进去，三个月后没人知道哪条是真的。限定“只追加已验证条目 + 人工 review diff”。
- **和 MCP 配置重复定义**。tools.md 只引用 MCP server 的名字和用途，参数留在 MCP 配置里，避免两处真相打架。
- **写太长**。它是要进上下文的。单份控制在几十行，按“Agent 真的会用到”裁剪；人看的细节写成普通 README。

# 可复用建议

- 每条加 `last_verified` 日期，超期的条目让 Agent 先跑验证命令再使用，可疑就标记而非删除；
- 新机器初始化时，让 Agent 按模板逐项探测生成初稿，人工校对后作为基线，比手写快一个量级；
- tools.md 和 AGENTS.md 之类的行为约定分开维护：一个管“环境是什么”，一个管“该怎么干活”，职责别混。

# 总结

tools.md 不是什么新协议，本质是把“每台机器都不一样”这件事，从散落的提示词和口口相传，收敛成一份**分层、可版本化、可 review** 的事实清单。它消除不了环境差异本身，但能让 Agent 在任何一台机器上第一次就找对路，并且让你随时能审计它“以为”的世界长什么样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/e5d587562524c86b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/d87dc537ff4c2f7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/5cc35008d393db18.png)

