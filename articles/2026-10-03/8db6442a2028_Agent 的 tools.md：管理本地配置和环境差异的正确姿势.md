---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40230
source: 综合讨论
publishedAt: 2026-10-03
---

# 背景

跑 OpenClaw 或自建 Agent 的人大多有个体验：同一个 workspace，在公司工作站上跑得好好的，换到家里的服务器或笔记本就翻车。原因不在模型，而在环境——包管理器不同、路径不同、代理不同、GPU 驱动不同。模型只能靠猜，猜错就一轮一轮试错，烧 token 还可能误删东西。

MCP 和插件解决了“工具怎么调用”的问题，但“这台机器上的工具长什么样”没人管。把这些写进系统提示词又太脆：换机器就得改 prompt，还容易把私有路径带进共享配置。

# 问题

我踩过的坑大致三类：

1. Agent 默认用 apt 装东西，机器是 macOS，折腾半小时没装上；
2. 每台机器维护一份不同的 system prompt，三台机器三份配置，改一处漏两处；
3. 新装了个工具忘了同步，Agent 坚持用旧路径，反复失败。

# 做法：写一份 tools.md

思路很朴素：把“这台机器的事实”沉淀成一份固定文件，放在 workspace 根目录（我放 `~/.openclaw/workspace/tools.md`），并在 AGENTS.md 里显式引用，保证会话启动时 Agent 会读。

我现在的模板大概长这样：

```markdown
# tools.md @ workstation-01（上次校验 2025-06-12）
- OS: Ubuntu 22.04 / bash；sudo 需要密码
- 包管理: 系统用 apt；Python 统一走 uv，禁止 pip 装进系统环境
- 路径: node 在 ~/.nvm/versions/...；ffmpeg 在 /usr/local/bin
- 网络: 出网走 http://127.0.0.1:7890，curl/wget 需显式带代理
- GPU: CUDA 12.1，torch 在 ~/envs/ml，不要重建环境
- 禁改区: /etc/nginx 改动前必须先备份
- 自检: 动手前先跑 ./scripts/env-check.sh
```

三条原则：

- **分层**：机器级差异进 tools.md（不入 git），项目级约定进项目内 AGENTS.md（入 git）。多机用户可以 `tools.md@workstation`、`tools.md@homelab` 各一份，按 host 注入对应文件。
- **写事实，不写指令**：“uv 装在 ~/...” 是事实，“请务必小心”是废话。Agent 不需要态度，需要路径、版本和边界。
- **带自检命令**：给一条 Agent 能自己跑的校验脚本。文字描述会过时，`env-check.sh` 不会。

# 踩坑点

- **过期比缺失更危险**。Agent 会无条件信任这份文件。加“上次校验”日期，或让自检脚本兜底。
- **别写密钥**。写环境变量名（如 `GITHUB_TOKEN 在 env 中`），不要写值。tools.md 会被原样读进上下文，也可能被截图分享。
- **控制篇幅**。超过一屏，模型对后半段的遵循率明显下降。我压在 60 行内，次要信息挪进自检脚本。
- **别指望自动发现**。不引用就不会读。AGENTS.md 里那句“先读 tools.md”是关键一行。

# 可复用建议

- 团队仓库放一份 `tools.md.template`，成员各自填，CI 只校验格式不校验内容；
- 让 Agent 自己维护：发现新事实就追加一行，走 PR review diff，人只做确认；
- 把“禁改区”当安全边界提前写清，比事后回滚省心得多。

# 总结

tools.md 不是新协议，就是把运维里“机器台账”的老习惯搬进 Agent 工作流。它解决的核心问题是：让 Agent 动手之前，先知道“这台机器和别的机器不一样在哪”。一份 60 行以内的实事清单，加一个自检脚本，能消掉大部分跨环境翻车。成本十分钟，值得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/60b77b0456258e37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/3617e28ee212037b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/9a6c833a56c01feb.png)

