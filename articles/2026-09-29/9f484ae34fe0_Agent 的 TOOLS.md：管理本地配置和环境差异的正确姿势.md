---
title: Agent 的 TOOLS.md：管理本地配置和环境差异的正确姿势
feedId: 39382
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 agent 跑在本地，但每次会话启动时，它对"这台机器"的了解几乎为零——除非你告诉它。我长期在两三台机器上跑同一个 agent：一台 MacBook、一台 Linux 家庭服务器、偶尔一台 VPS。三台机器的包管理器、路径习惯、已装工具全不一样。模型不知道这些差异，只能猜，猜错就重试，重试就烧 token。

## 问题

没有环境说明之前，我的典型失败场景：

- 在 Linux 服务器上，agent 想当然地执行 `brew install`；
- 假设项目在 `~/projects`，实际在 `/srv/workspace`；
- 每次新会话都要在对话里重复一遍："这台机器 ffmpeg 在 /usr/local/bin，node 是 20，别动全局 pip。"

这些信息原本散落在聊天记录、AGENTS.md 和我自己的脑子里，既不一致，也不可迁移。本质上，这是把"机器事实"当成了"对话内容"来管理。

## 做法

OpenClaw 约定会话启动时会读取 workspace（默认 `~/.openclaw/workspace`）下的 `TOOLS.md`，正好用它承载这类信息。我的写法：

1. **建文件**：workspace 根目录放 `TOOLS.md`，当成"本机说明书"。
2. **只写差异**：机器画像（OS、shell、包管理器）、关键 CLI 及获取方式、路径约定（项目根、日志、临时目录）、这台机器的禁忌（不要全局 pip install、不要碰某目录）。
3. **动态值交给探测**：写 `node 由 nvm 管理，先 nvm current 确认`，而不是写死版本号——写死的迟早过期。
4. **分层**：跨机器通用的事实放 workspace 级 `TOOLS.md`；单项目的行为要求放项目里的 `AGENTS.md`。两份文件职责不同，别互相抄。
5. **纳入版本管理**：放进 dotfiles 仓库，换机 clone 下来，只改差异部分。
6. **验证加载**：新会话问一句"根据 TOOLS.md，这台机器该用哪个包管理器"，确认它真读进去了。

## 踩坑点

- **别写密钥**。`TOOLS.md` 每个会话都进上下文，还可能被 git 同步。只写"token 在环境变量 `XXX_TOKEN` 里，用前确认存在"，不要写值本身。
- **别写长**。这份文件每个会话都吃 token，超过一屏就该砍。它是差异清单，不是机器百科。
- **会过期**。系统升级后忘了更新，agent 会拿着错误信息更自信地犯错。我把"更新 TOOLS.md"挂进了换机和大版本升级的 checklist。
- **别和 AGENTS.md 混**。一句话判断：描述"这台机器是什么"进 TOOLS.md，描述"agent 该怎么做事"进 AGENTS.md。

## 可复用建议

给个骨架，直接抄：

```markdown
# TOOLS.md
## 机器画像   # OS / shell / 包管理器 / 硬件要点
## 工具链     # 关键 CLI 及获取方式，动态值写探测命令
## 路径约定   # 项目根 / 日志 / 下载 / 临时目录
## 禁忌       # 明确不许 agent 做的事
## 变动记录   # 一行一条，带日期
```

另外两个习惯很值：每次手动配完新环境，顺手补一行进 TOOLS.md；每月让 agent 自己跑一遍文件里的探测命令，diff 出过期条目。

## 总结

`TOOLS.md` 解决的不是技术问题，是信息归属问题：机器事实属于文件，不属于对话。一份短的、只写差异的、纳入版本管理的本机说明书，能让同一个 agent 在三台机器上表现一致，也省掉大量重复解释。它不炫技，但属于那种"配置一次、长期受益"的工程习惯。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ee02b5e8286abfaf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/9e5a9ac4cf2814ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c122745a14ca5721.png)

