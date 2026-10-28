---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42830
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

Agent 用久了会发现，摩擦最大的往往不是模型能力，而是"这台机器和那台机器不一样"。在 OpenClaw 的 workspace 里，AGENTS.md 管行为约束，而 tools.md 的定位很明确：告诉 Agent 这套环境里有什么、该怎么用。实际情况是，很多人要么不写，要么把它写成了第二份 README。

## 问题

几个典型翻车场景：

- 同一 workspace 通过 git 同步到三台机器：MacBook、家里的小主机、一台 GPU 盒子。Agent 在 Mac 上习惯性 `brew install`，而 Debian 上根本没有 brew；
- 路径漂移：`/Users/a`、`/home/a`、`/mnt/data` 各不相同；
- 团队共用仓库，Python 环境管理器不统一（conda / uv / 系统 pip），Agent 按文档装包，装错了地方；
- 更隐蔽的是文档腐烂：环境早变了，tools.md 还在"说谎"，Agent 理直气壮地按错误事实行动。

根因只有一个：把 tools.md 当静态文档，而不是需要与真实环境保持一致的状态描述。

## 做法

**1. 先定纪律**：tools.md 只写"能被一条命令验证的事实"。验证不了的内容（偏好、任务流程）归 AGENTS.md。

**2. 三层结构**：

- 全局节：shell、平台、通用约定，各机一致的部分；
- 分机器节：以 hostname 为锚点，如 `## host: studio-mac`，节前注明"先执行 hostname，再读对应小节"；
- 个人覆盖：`tools.local.md` 放私有内容（内网别名、个人路径），加入 gitignore，只在本机生效。

**3. 写法用「事实 + 校验命令」，不写顺序步骤**：

```markdown
# Tools
## Global
- 密钥一律走环境变量，变量名见 .env.example，不要索要明文

## host: studio-mac
- 包管理：brew；Python 用 uv（校验：command -v uv）
- 工作区 ~/work，未列出的路径先询问

## host: lab-gpu
- 包管理：apt + conda env torch
- GPU 任务先跑 nvidia-smi 确认空闲
```

对比一下：差的写法是 `1. pip install -r requirements.txt`；好的写法是"本机用 uv，没有 uv 时先报告，不要自行 pip install"。前者是脚本，后者是事实加守门条件。

**4. 维护闭环**：装了新工具、换了路径，随手让 Agent "对照当前机器自检 tools.md，列出不符项"，人工确认后更新。文档腐烂多半是缺了这一步。

## 踩坑点

- 明文写密钥、内网 IP、token 后 push 到远端。tools.md 会进模型上下文，也会进 git 历史，删了也还在；
- 写成脚本式步骤清单，Agent 会机械顺序执行，一步失败后行为不可控；
- 无限膨胀：tools.md 每次会话都占上下文，堆到几百行会挤压真实任务，条目之间还会互相冲突；
- 多机共用 workspace 却不设分机器节，指望 Agent "自己判断"，判断错一次就是一次事故。

## 可复用建议

- 一条内容去留的判断标准：能否用一条命令验证？不能就移走或删掉；
- tools.md 管机器层事实，项目层差异放项目目录内的说明文件，别全塞一个文件；
- 新环境初始化时，先让 Agent 探测并生成 tools.md 草稿，再人工修剪，比从零手写快得多；
- 把"文档与现实的 diff"当例行维护项：环境变更后自检一次，人工确认一次。

## 总结

tools.md 的价值不在于写了多少，而在于 Agent 相信它时不会出错。把它当成环境的状态快照：分层组织、每条可验证、密钥外置，再配一个 tools.local.md 做个人覆盖，多机与协作的漂移问题基本能压住。写环境文档这件事，频率比篇幅重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/81342fc59423070c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/a76b36d8c82fb695.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/362d5c83ceac1833.png)

