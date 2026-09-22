---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 38508
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

Agent（比如 OpenClaw 这类常驻本地的 assistant，或任何接了 shell / MCP 的 agent）干活的前提是"认识"你的机器。但每台机器都不一样：macOS 上是 `gsed`，Linux 上是 `sed`；node 走 nvm，python 走 pyenv；ffmpeg 装在非标准路径；公司内网还得走代理。这些知识不在任何官方文档里，只在你脑子里。

## 问题

常见的三种错误姿势：

1. **口头喂**：每次新会话重复解释"我的 xx 在哪"，费 token，还容易忘。
2. **混进 AGENTS.md**：把环境事实和行为规范写在一起，换台机器整份配置跟着失效。
3. **让 agent 自己探测**：每次 `which` 一轮，慢，猜错路径后开始自由发挥。

根本原因是：环境知识没有声明化。它应该在文件里，而不是在对话里。OpenClaw 的约定是 workspace 根目录下的 `TOOLS.md`，每次会话自动读取，专门承载"这台机器上有什么"。

## 做法

1. **盘点**：用 `command -v` 过一遍常用工具，记下非标准路径和版本管理器的加载方式。
2. **建模板**：固定几个小节——工具路径、平台差异、自定义脚本、网络与代理、禁区。
3. **只写事实**：一行一条，写成 agent 可直接执行的形态，不写教程。
4. **分层管理**：脱敏的 TOOLS.md 入 git；机器相关的敏感内容拆到 `TOOLS.local.md`，进 .gitignore。
5. **校验**：写个十几行的 smoke check 脚本核对路径是否存在，迁移或升级后跑一遍。

最小模板参考：

```markdown
# TOOLS.md
## 工具路径
- ffmpeg: /opt/homebrew/bin/ffmpeg（不在 PATH，用绝对路径调用）
- node: 走 nvm，先 `source ~/.nvm/nvm.sh && nvm use 22`
## 平台差异
- macOS: GNU 工具带 g 前缀（gsed/gawk）；无 timeout，用 gtimeout
## 自定义脚本
- ~/bin/sync-notes.sh: 同步笔记，--dry-run 可试运行
## 网络
- 代理: http://127.0.0.1:7890（仅终端，export 后再用）
## 禁区
- 不要动 /etc/hosts；密钥一律读环境变量，不落文件
```

## 踩坑点

- **写成说明书**：超过一屏，agent 抓不住重点还挤占上下文。超过 60 行就该砍。
- **绝对路径硬编码入库**：`/Users/xxx/...` 提交到仓库，换人换机全是噪音。用 `~/` 相对家目录。
- **密钥和内网地址写进去**：这个文件会被反复读取、可能被日志带出。敏感值只放环境变量，TOOLS.md 里写"从 XX 变量读"。
- **和 AGENTS.md 职责混淆**：一句话分工——AGENTS.md 管"怎么干活"，TOOLS.md 管"机器上有什么"。
- **信息过期**：升级后路径变了没更新，agent 按旧信息执行然后开始猜。把"发现路径变化先更新 TOOLS.md"写成约定，让 agent 参与维护。

## 可复用建议

- 命名分层固化：`TOOLS.md`（入库、脱敏、跨机通用）+ `TOOLS.local.md`（本机覆盖、不入库），后者优先。
- 多机用户按机器分节（`## macbook` / `## vps`），避免维护多份文件。
- 让 agent 自举：任务中发现新装的工具、报错里出现路径变化，先补一行进 TOOLS.md 再继续干活。
- 定期跑一次 smoke check，删掉已不存在的条目。这个文件的生命力在于"可信"，宁可条目少。

## 总结

tools.md 的本质，是把环境知识从"对话记忆"变成"版本化的声明式配置"。三条原则：写事实不写教程，写少不写多，敏感信息外置。配置好之后你会发现，换机器、新会话的冷启动成本明显下降，需要你重复解释的话也越来越少——这才是配置文件该干的活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/33e7bca0cfad7ab6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/b7dcb3cf8eaf867b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/dc5651bc317486c5.png)

