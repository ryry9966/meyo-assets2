---
title: Agent 的 tools.md：用两层文件管住本地配置和环境差异
feedId: 39644
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 类 agent 的能力边界，很大程度取决于它"知道"什么。项目里装了哪些脚本、venv 在哪、这台机器有没有 GPU、代理怎么走——这些信息现在大多散落在三处：system prompt、MCP 配置，或者干脆靠 agent 每次现场摸索。我们的经验是：前两处会腐化，最后一处会浪费大量 token 和试错轮次。

tools.md 是一个更朴素的方案：在项目根目录放一个 Markdown 文件，agent 启动时读入。它不算新发明，本质是把 README 的"环境部分"拆出来，专门写给 agent 看。

## 问题

真正的问题不是"要不要写 tools.md"，而是**环境差异怎么表达**。我们在三台机器上跑同一套 agent：一台 macOS、一台 Ubuntu 服务器、一台带 GPU 的工作站。Python 路径不同，ffmpeg 一个 brew 装的一个 apt 装的，代理只有内网机器需要。只维护一份文件，要么污染别人的环境，要么写得四不像。

## 做法

我们的约定是两层：

1. **`tools.md`（入库）**：机器无关的事实。有哪些工具类别、团队约定的调用方式、"环境相关条目请查 tools.local.md"。进 PR review，像代码一样对待。
2. **`tools.local.md`（gitignore）**：机器相关的事实。每个条目尽量带一条可执行的校验命令，比如 `ffmpeg → which ffmpeg`，而不是"已安装 ffmpeg"。

每个条目控制在三行内：工具名、路径/调用方式、一行校验命令。写法上有一条关键原则：**只写 agent 能自己验证的陈述**。"GPU 环境已配好"这种句子没有意义，`python -c "import torch; print(torch.cuda.is_available())"` 才有意义。

启动流程：agent 初始化读 tools.md → 遇到环境相关条目按需读 tools.local.md → 对关键条目跑校验命令 → 命令输出与文件冲突时，以输出为准并在会话中标注。

## 踩坑点

- **模糊描述比不写更糟。**含糊的"已配好"会让 agent 建立错误信心，然后一头撞墙。写进去的每一条，都要能被一条命令证实或证伪。
- **文件会腐化。**环境变了文件没变，agent 会拿着过期地图走错路。对策是把校验命令汇总成一个 `tools-doctor` 脚本，CI 或每周跑一次，失败条目直接列出来。
- **和 MCP 配置重复。**MCP server（以及插件机制）已注册的工具，不要在 tools.md 里复述第二遍，写一句"工具 X 由 MCP 提供，详见 server 配置"即可。两处维护必然漂移。
- **别把秘密写进 tools.local.md。**本地文件虽不入库，但会进 agent 上下文，还可能随会话日志被带走。路径、开关可以写，密钥一律走环境变量。
- **路径别写死。**用 `~` 或 `$HOME` 前先确认 agent 的 shell 会展开；跨 Windows 的团队直接把两种展开方式都写上。

## 可复用建议

- 把 tools.md 当代码：改环境必须连着改文件，review 时只看 diff 就能发现环境变更。
- 条目总量控制在几十行内；细节宁可让 agent 现场跑命令，也别全塞进上下文。
- 校验命令输出格式统一（成功/失败各一行），方便 tools-doctor 汇总判定。
- 换新机器时先跑一遍 tools-doctor，把失败项补进 tools.local.md 再开工，比让 agent 逐个试错快得多。

## 总结

tools.md 解决的不是"agent 有哪些工具"——那是 MCP 注册表的事——而是"**这台机器上**这些工具长什么样"。它的成本低得惊人：一个 Markdown 文件加十几条 shell 命令，却能把 agent 从"每台机器重新摸索一遍"里解放出来。核心就两条：分层表达差异，只写可验证的陈述。剩下的交给校验脚本兜底。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/084e5fe7acf47a77.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/60219dfa34c1b359.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b8bc963cfb985fd8.png)

