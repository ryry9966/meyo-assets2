---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39497
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 agent 在会话启动时，会把工作区里的 `TOOLS.md`（部分版本同时读 `AGENTS.md`）注入上下文。它的定位很明确：**写给 agent 自己看的环境说明书**。大多数人的第一版 tools.md 是随手写的——装了什么、路径在哪、习惯用什么命令，想到什么写什么。直到你在第二台机器（家里的 NAS、一台 VPS、或同事的 macOS）上跑同一个 agent，才发现它开始一本正经地执行错误命令。

## 问题

环境差异导致三类典型故障：

1. **命令错配**：agent 在 Ubuntu 上用 `brew`，在 macOS 上写 systemd 单元，因为模型默认所有机器长得一样。
2. **路径幻觉**：它不知道你的项目根目录在 `/home/xx/projects` 还是 `D:\dev`，第一步猜错，后面一路错下去。
3. **配置散落**：机器事实写进了 prompt 模板，团队约定写进了 skill，改一处忘三处。

更隐蔽的是安全问题：有人图省事把 API key 直接写进 tools.md，而这份文件会被完整送进模型上下文和会话日志。

## 做法

核心思路是**分层 + 模板化**：

**1. 区分两个层级。** 机器事实（OS、包管理器、路径、服务管理方式）放全局 workspace 的 tools.md，只属于这台机器；团队约定（构建命令、测试入口、代码规范）放仓库内的文件并提交。两者职责不要混。

**2. 真实文件不入库，模板入库。** 仓库提交 `tools.md.example`，把真实 `tools.md` 加进 `.gitignore`。同事 clone 后复制一份，改成自己的环境。

**3. 用脚本生成初版，别手写。** 几行 shell 就能探测出大部分事实：

```bash
{
  echo "# Environment"
  echo "- OS: $(uname -s) $(uname -r)"
  command -v brew >/dev/null && echo "- Pkg: brew"
  command -v apt  >/dev/null && echo "- Pkg: apt"
  node -v 2>/dev/null && echo "- Node: $(node -v)"
} > tools.md
```

**4. 控制篇幅和结构。** 目标 100 行以内，只写"agent 会猜错的东西"。固定五个小节：Environment / Paths / Commands / Gotchas / Never。最后一节尤其有用——把"禁止 `sudo pip install`""不要动 workspace 外的文件"这类约束显式写死，比在对话里反复纠正省事得多。

## 踩坑点

- **写入密钥**：tools.md 会进上下文和日志。密钥放环境变量，文件里只写一句"密钥从 `XX_TOKEN` 环境变量读取"。
- **改完不生效**：文件在会话启动时注入，改完要重开会话，别以为是写错了。
- **信息过期**：一条错误的路径比没有路径危害更大。升级系统、迁移目录后记得同步。
- **与 AGENTS.md 职责重叠**：一个回答"这台机器是什么"，一个回答"在这里做事的规矩"。写重了会出现互相矛盾的指令，agent 的行为会变得不可预测。

## 可复用建议

- **让 agent 自审**：隔一段时间发一句"对照 tools.md 逐条运行验证，把失效条目直接改掉"。它有 shell 权限，这事做得比人快。
- **CI 里加一致性检查**：校验 `tools.md.example` 的章节结构与真实文件对齐，防止有人改了模板忘了同步。
- **只记差异**：能被探测到的（Node 版本）简写即可；探测不到的（"这个仓库用 pnpm 不是 npm"）才值得完整写一行。

## 总结

tools.md 不是文档，是 agent 的运行时配置。把它当代码管理：分层、模板化、可生成、可校验、定期维护。做到这几条，同一个 agent 在你的笔记本、服务器和同事机器上，表现会一致得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/7f7643faba11dbb8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/314a82c943d970ea.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/5a3101a3d2ff6995.png)

