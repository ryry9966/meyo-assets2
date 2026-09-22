---
title: Agent 的 tools.md：用三层清单管住本地配置与环境差异
feedId: 38462
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

跑 Agent 做自动化，最磨人的往往不是模型能力，而是环境差异：同一套指令，在 macOS 上走 `brew`，到 WSL 里得绕到 `docker.exe`；MCP 服务的端口、本地工具的路径、包管理器的选择，全靠会话里口口相传。换一台机器、换一个协作者，Agent 就得重新踩一遍坑才能进入状态。

## 问题

环境知识目前散落在三处：系统提示词、聊天上下文、各类配置文件（MCP json、插件清单、shell rc）。三者互不同步，Agent 只能猜。猜错的典型症状：引用不存在的路径、发明不存在的命令行参数、在错误端口上起服务。这类问题难排查，根源是"Agent 认为的环境"和"真实环境"之间没有对账机制。

## 做法：三层 tools.md + 优先级合并

核心思路：用一份 Markdown 作为 Agent 的环境清单，分三层，就近优先合并。

1. **全局层** `~/.openclaw/tools.md`：描述这台机器本身（OS、运行时版本、常用工具路径）。
2. **项目层** `<repo>/tools.md`：进 Git，团队共享（命令、约定、端口分配、工具用途）。
3. **本地层** `<repo>/tools.local.md`：gitignore，只放本机覆盖项和敏感路径。

优先级：local > project > global。项目里附一份 `tools.md.example`，新人 clone 后复制填充。如果你的 workspace 里已有 AGENTS.md，注意分工：AGENTS.md 管"怎么干活"（流程、边界），tools.md 管"在哪干、用什么干"（环境、工具、约定）。

骨架示例，保持简短：

```markdown
## 工具
- browser: 抓取与截图；截图存 /tmp/shots
- sqlite: 库在 ~/data/app.db，默认只读

## 约定
- 包管理 pnpm；测试命令 pnpm test
- 端口：dev 3000 / mock 4000

## 例外
- WSL 下用 docker.exe
- ffmpeg 在 /opt/homebrew/bin
```

两条维护纪律：

- **写回**：Agent 在会话里探测到的环境事实（缺依赖、路径不同），要求它产出对 tools.md 的 diff 建议，而不是只留在上下文里。把易失的会话知识沉淀成配置。
- **对账**：写一个 `env doctor` 小脚本，逐条校验清单（命令存在？端口空闲？路径可达？），Agent 开工前先跑一遍，漂移项直接列出来。

## 踩坑点

- **别放密钥**。tools.md 进了 Git，密钥就跟着进了历史，洗历史很痛苦。敏感内容只进 tools.local.md。
- **别复制 MCP 配置**。tools.md 描述工具"是什么、怎么用、有什么坑"，连接参数仍以 MCP/插件配置为唯一来源。两处都写，迟早打架。
- **条目要可验证**。写"测试命令 X，期望退出码 0"，别写"跑一下测试就行"。可验证的条目才能被 env doctor 检查。
- **控制长度**。超过百行，注意力被稀释，关键例外反而被忽略。例外放前面，通用约定往后放。
- **合并顺序要实测**。别假设 Agent 按你预期的优先级合并三层，用一个带冲突的样例显式测一次。

## 可复用建议

- 每个工具条目固定四段：名称 / 用途 / 调用方式 / 注意事项，降低阅读和校验成本。
- tools.md 的变更走 PR review，像代码一样对待——它就是代码，只不过是给 Agent 读的。
- 一次只改一个事实，commit message 写清"哪个环境变了"，方便回溯是哪次升级引入的回归。

## 总结

tools.md 不是文档，是 Agent 的运行时契约。它便宜、显式、可版本化，把环境差异从口头经验变成可 review、可校验、可回滚的配置。花半小时搭好三层结构和 env doctor，换来的是换机器零重新踩坑——这是我目前见过性价比最高的 Agent 工程实践之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/74dc6b0e4e14c2d4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/e4508680246ae31a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/1c9739ea8186d88b.png)

