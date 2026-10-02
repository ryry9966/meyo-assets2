---
title: Agent 的 tools.md：用分层文档管住本地环境差异
feedId: 40168
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

跑 Agent（OpenClaw + MCP + 一堆 CLI 工具）最烦的往往不是模型能力，而是环境差异。同一份任务清单，在台式机（Ubuntu、apt、有 GPU）上能跑通，换到笔记本（macOS、brew、没装 Docker）就报错。原因多半不是 Agent 笨，而是它根本不知道这台机器长什么样。

## 问题

常见的三种错误姿势：

1. **写进 system prompt**：换机器就得改 prompt，改完忘了同步；
2. **靠运行时自己探测**：`which ffmpeg`、`python3 --version` 试一轮，白烧 token 和轮次，还可能猜错；
3. **文档散落**：README、.env、shell rc 各存一份，README 三个月没更新，Agent 读到的是过期事实。

核心矛盾在于：环境事实是「每台机器一份、经常变动」，而 prompt 和文档默认是「一份通用、很少变动」。

## 做法

tools.md 的定位：**Agent 启动时读取的本地环境事实清单**。推荐分两层：

1. **仓库层 tools.md**：团队共识，进 git。只写跨机器成立的约定——项目入口路径、lint 和测试命令、包管理器优先级（比如「优先 uv，没有再用 pip」）。
2. **机器层 tools.local.md**：进 gitignore，每台机器一份。写 OS、shell、已装 CLI 及版本、GPU 与代理情况、端口占用习惯。

加载时 local 覆盖 base，Agent 读合并结果，挂进 OpenClaw 的会话初始化配置，别指望它「有空自己去看」。

关键原则是**生成优先于手写**：环境清单这类机器可探测的事实，用一个 snapshot 脚本生成（`which`、`--version`、`docker info` 拼一拼），登录或 CI 时刷新，附上时间戳。手写只留给探测不出来的约定。

一条判断标准：**能跑命令查到的，脚本生成；是人定的约定，写进文档；是机密，只写环境变量名。**

## 踩坑点

- **密钥入库**：tools.md 进了 git，顺手把 API key 也写进去。文档里只允许出现 `SOME_API_KEY`（从 .env 读）这种引用形式。
- **写太长**：tools.md 写成散文，每次会话固定烧几千 token。控制在事实和短句，超过一两百行就该砍。
- **过期事实比没有更糟**：文档说装了 ffmpeg，实际已卸载，Agent 会反复重试浪费轮次。所以清单必须靠脚本刷新，不靠手维护。
- **和 MCP 描述重复**：MCP server 的 tool list 运行时已经拿得到，tools.md 别再抄一遍，只记录 Agent 自己发现不了的东西。
- **命令名含糊**：写「用 python 跑」在有的机器上是 python2。写清楚 `python3`，或者写进 base 层的解析规则。

## 可复用建议

- 把 tools.md 当配置代码对待，改动走 PR review；
- 模板化新机器初始化：clone 仓库 → 跑 snapshot 脚本 → 补 local 文件，三步完成；
- 易变条目（代理地址、挂载路径）加注释，标注上次验证时间；
- 定期检查 token 开销：tools.md 的收益要大于它占用的上下文成本。

## 总结

tools.md 不是又一份文档，而是把「环境差异」从 Agent 的推理负担里挪出来，变成一份可版本化、可生成、可分层的静态事实。Agent 少猜一次，任务就少绕一圈。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/1d124593bebfacf3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/a6227b4b6e9b163d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/d4b399086790255b.png)

