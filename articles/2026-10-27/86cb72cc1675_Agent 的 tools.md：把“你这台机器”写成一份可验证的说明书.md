---
title: Agent 的 tools.md：把“你这台机器”写成一份可验证的说明书
feedId: 42702
source: 综合讨论
publishedAt: 2026-10-27
---

# 背景

OpenClaw 的价值在落地执行：agent 直接跑 shell、调 MCP 工具、触发本地脚本。但模型对“你这台机器”一无所知——它知道 ffmpeg 是什么，不知道你的 ffmpeg 在不在 PATH 里、是 6.1 还是 4.4、走不走代理。

我在笔记本（macOS）、家里的 Debian 小主机和一台 VPS 上跑了同一个实例、同一个工作区，问题很快出现。

# 问题

典型翻车现场：

- agent 想压个视频，直接 `ffmpeg -i ...`，报 command not found，接着试图 `apt install`——没权限，也不该装；
- 同一条 `node script.js`，笔记本上是 nvm 管的 22，VPS 的非交互 shell 里根本没有 node；
- 这些“机器事实”散落在聊天记录里，换一个会话就丢，agent 重新试错一遍，每一步都在烧 token。

本质上这是文档问题，不是模型问题：机器相关知识需要一个稳定、可验证、随环境走的存放点。

# 做法：一份四行封顶的机器说明书

在工作区放一个 `tools.md`（比如 `~/.openclaw/workspace/tools.md`），并在 AGENTS.md 里加一句：执行任何工具命令前先查 tools.md；命令失败时先用条目里的验证命令复查，再换方案。

条目结构建议：

```markdown
## ffmpeg
- 路径: /opt/homebrew/bin/ffmpeg（仅本机，VPS 无此工具）
- 验证: ffmpeg -version | head -1
- 用法: 转码一律加 -threads 2，机器只有 2 核
- 禁止: 不要尝试安装，不要往 /tmp 写大文件
```

配套习惯：

1. **按机器分文件**：`tools.macbook.md`、`tools.vps.md`，不要互相同步——差异本身就是内容。
2. **写验证命令而不是只写结论**。agent 先跑验证，结果对不上就当条目过期处理，不要盲信。
3. **脚本生成初稿**：把 `which ffmpeg`、`node -v`、`docker info` 之类的探测命令塞进一个小 shell 脚本，装完新东西跑一遍，输出贴进 tools.md，避免手抄路径出错。
4. **单列禁区**：sudo、可删目录、计费类 API，集中写一节 "don't"。

# 踩坑点

- **过期比缺失更糟**。brew 升级后路径或版本变了，agent 仍信旧条目。所以验证命令是必填项，不是加分项。
- **别放密钥**。有人顺手把 API key 写进去——tools.md 会被装进模型上下文和日志。密钥只放环境变量，文件里最多写一句“代理配置见 `~/.config/xxx.env`”。
- **别写长**。这是注入模型的上下文，尽量 100 行以内；超了就拆文件、按需引用。
- **注意非交互 shell**。cron/systemd 不加载 `.zshrc`，nvm 全部失效。条目里写绝对路径，比写“用 nvm 的 node”可靠得多。
- **不替代脚本**。一段流程超过三步就写成脚本，tools.md 只登记“脚本在哪、干什么用”。

# 可复用建议

- **分层约定**：机器相关 → tools.md；项目相关 → 项目内文档；会话相关 → 留在对话里。三者不要混。
- 写个 `doctor` 脚本：比对 tools.md 声明的路径/版本与实际输出，有 diff 就列出来。长任务前跑一次，成本极低。
- 每台机器的 tools.md 进私有 git，升级出问题时 diff 一下就知道环境改了什么。

# 总结

tools.md 做的事情很朴素：把“这台机器长什么样”从聊天记录里捞出来，变成一份短小、可验证、按机器隔离的说明书。写它不到半小时，省掉的是 agent 每次会话的试错、误装和误判。让 agent 少猜，是它稳定干活的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f5a08e007e826c9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/aea8ca17a8a57cdf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/0eafb8aa2ccd99a1.png)

