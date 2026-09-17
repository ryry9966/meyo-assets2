---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 37935
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

用 Agent 做自动化久了会发现一个规律：任务失败很少是因为模型不够聪明，更多是因为它对本地环境的认知是错的。路径猜错、包管理器用混、macOS 和 Linux 命令差异、venv 没激活……这些事实散落在 README、shell history 和人的脑子里，Agent 每次都要重新猜。

OpenClaw 的工作区约定里，AGENTS.md 管"怎么做事"，tools.md 管"在什么环境里做事"。前者是行为规范，后者是环境契约，两者不该混。MCP 能覆盖标准化工具的发现，但 MCP 之外的本地脚本、系统命令、路径约定，恰恰是 tools.md 的地盘。

## 问题

没有 tools.md 时，常见三类故障：

1. **命令幻觉**：Agent 假设项目用 npm，实际是 pnpm；假设测试命令是 `make test`，实际根本不存在。
2. **环境差异**：同一份指令在 mac 上是 `sed -i ''`，在 Linux 上是 `sed -i`；日志路径、Python 版本、端口占用各不相同。
3. **知识不可迁移**：换台机器、换个协作者，所有坑重新踩一遍。

## 做法

**1. 固定四段结构**：环境概览、已验证命令、差异矩阵、禁区。模板：

```markdown
# tools.md — 本环境事实（Agent 与维护者共同遵守）

## 环境
- OS: macOS 14（本地）/ Ubuntu 22.04（测试机）
- Node 20 via fnm；Python 3.11，venv 在 ./.venv，先 source 再跑脚本

## 已验证命令（2025-06-12）
- 测试：pnpm test
- 本地网关：pnpm dev --port 3100

## 差异矩阵
- sed：mac 用 `sed -i ''`，Linux 用 `sed -i`
- 日志：~/logs/openclaw（mac）、/var/log/openclaw（linux）

## 禁区
- 不全局安装依赖；不动 .env*；不重装系统级工具
```

**2. 只写跑通过的命令**。每条命令入档前先执行一遍，标注验证日期。tools.md 的核心价值不是全面，而是"写了就为真"。

**3. 分层管理**。共享部分进 tools.md 提交仓库；机器相关的路径、别名放 tools.local.md，加入 `.gitignore`。密钥永远不进这两个文件。

**4. 让 Agent 参与维护**。任务开始先读 tools.md；结束时把"文档与现实不符"的地方汇报出来，人工确认后更新。文档即代码，改命令走 PR。

## 踩坑点

- **写"应该能跑"的命令**。文档一旦失真，Agent 会自信地反复失败，比没有文档更糟。
- **和 README 重复维护**。README 给人看，tools.md 给 Agent 看，内容冲突时 Agent 不知听谁的。原则：tools.md 只写执行相关的事实，其余链接出去。
- **文件膨胀**。塞满细节会浪费上下文窗口，控制在几十行内，长内容拆子文档。
- **平台差异不写明**。`sed -i`、路径大小写、`readlink -f` 在 macOS 缺失，都是高频坑，值得单独一节。

## 可复用建议

- 记住三原则：**验证过、最短必要、机器相关外置**。
- 配一个 20 行的校验脚本放进 CI：逐条检查 tools.md 里的命令是否可执行、路径是否存在，防止文档腐化。
- 团队场景下，把 tools.md 的 diff 纳入 code review——它改变了所有 Agent 的执行行为，权限上应等同配置文件。

## 总结

tools.md 不是又一份文档，而是一份可验证的环境契约。它的价值不在于写得多全，而在于每一条都被验证过、被分层隔离、被持续校验。做到这三点，Agent 在你机器上的表现会稳定得多，排障时也有据可查。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/c87673f7705c61b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/77d4bd36ee7edc01.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/fe664916cb57f03a.png)

