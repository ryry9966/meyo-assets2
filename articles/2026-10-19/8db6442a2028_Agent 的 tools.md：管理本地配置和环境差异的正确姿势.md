---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 41961
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

跑 OpenClaw Agent 的人大多不止一台机器：开发用的 macOS 笔记本、家里的小主机、一台 VPS，可能还有 CI。Agent 对"这台机器长什么样"的认知来自两个来源：运行时探测，以及 tools.md 里的静态描述。tools.md 相当于 Agent 的《宿主机使用手册》——shell 是什么、包管理器用哪个、工作目录在哪、本地 MCP 服务跑在哪个端口、哪些事不许做。

问题在于，很多人图省事，把所有环境信息写进一份 tools.md，或者散落在系统提示词和插件配置里。换台机器就出事：Agent 在 mac 上敲 `apt`，在无 GPU 的主机上尝试 CUDA，或者更糟——把内网地址和 API Key 随 tools.md 一起提交进了仓库。

## 做法：分三层

核心思路是分层，把"处处为真的事实"和"本机特有的事实"分开：

1. **base 层**：跨平台通用的约定与边界。比如"长任务用后台任务加轮询""密钥只从环境变量读取，禁止写入任何文件"。进 Git，团队共享。
2. **host 层**：只写本机差异。OS、包管理器、绝对路径、本地服务端口、硬件限制。一台机器一份，放 `~/.config/openclaw/tools.d/` 之类的位置，不进 Git。
3. **secrets 纪律**：tools.md 里只写"从哪个环境变量读"，绝不写值本身。

加载顺序：先 base，再 host，冲突时 host 覆盖。可以用简单的 include 机制，也可以在 Agent 启动前用一个几十行的脚本做合并。

base 层示例（节选）：

```markdown
## Shell 约定
- 优先写 POSIX 兼容命令
- 长任务放后台，轮询而非死等

## 秘密
- API Key 只从 env `MODEL_API_KEY` 读取
```

host 层示例（家里的小主机）：

```markdown
- OS: Debian 12，apt 可用，brew 不存在
- 工作目录: /srv/agent-workspace
- 本地服务: browser MCP 在 127.0.0.1:9222
- 无 GPU，禁止尝试 CUDA 相关操作
```

## 踩坑点

- **host 层误提交**。`.gitignore` 必须在第一次提交前配好，仓库里只放 base 层和模板。
- **重复描述**。同一个工具在两层各写一遍，信息冲突时 Agent 会摇摆。约定：host 层只写差异，不复述 base 已有的内容。
- **信息过期**。服务换了端口、目录挪了位置，tools.md 还是旧的。把它当"起点"而不是"事实源"——首次使用前让 Agent 探测验证一次。
- **越写越长**。tools.md 直接吃上下文，控制在几十行。细节交给脚本的 `--help`，tools.md 只写"去哪找"。
- **跨平台细节**。路径分隔符、行尾符、shell 语法差异，在 host 层显式标注，别指望 Agent 自己猜对。

## 可复用建议

- 维护一份 `tools.template.md`，新机器部署时复制出来填空，十分钟搞定。
- 写个探测脚本（OS、包管理器、常见路径、监听端口），自动生成 host 层草稿，人工校对一遍。
- 给 base 层加个 lint：检查高熵字符串，防止有人手滑把 Key 写进去。
- 每次改 base 层，提交信息里写清改动原因，方便回溯"Agent 行为为什么变了"。

## 总结

tools.md 不是文档，是运行时配置的一部分。管好它的关键是分层与最小化：base 进仓库、host 留本机、secrets 只走环境变量，每层只写必要信息。做完这几件事，Agent 换机器的表现会稳定很多，审计和排障也都有了抓手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/14603709f6ff4e6c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/f3b44c7c116d93b3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/020c2f6b8bef0449.png)

