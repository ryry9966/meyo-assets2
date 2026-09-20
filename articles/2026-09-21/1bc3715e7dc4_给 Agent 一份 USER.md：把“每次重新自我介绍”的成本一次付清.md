---
title: 给 Agent 一份 USER.md：把“每次重新自我介绍”的成本一次付清
feedId: 38297
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

玩 Agent 这套东西久了，大家习惯在项目里养 `AGENTS.md` / `CLAUDE.md`，告诉模型这个仓库怎么构建、测试怎么跑。但很少有人给“自己”写一份文件。结果是：Agent 知道项目的所有细节，却对你一无所知——每次会话都像新同事入职，你要重新交代“我用 pnpm 不用 npm”“回复用中文、先给结论”“服务器是 Ubuntu 22.04，路径在 /srv/app”。

OpenClaw 的 workspace 里预留了这个位置：`USER.md`。它和 `MEMORY.md` 的分工是：MEMORY 记过程（聊过什么、做过什么决定），USER.md 记稳定事实（你是谁、你的环境、你的偏好）。

## 问题

不写 USER.md 的代价很具体：

1. **重复解释**。偏好类指令说了这次生效，下次归零。
2. **错误的默认值**。Agent 猜你是 macOS + npm，实际是 Linux + pnpm，生成的命令永远差一步。
3. **语气不合拍**。你要简洁的技术判断，它给面面俱到的客服式回复。
4. **跨项目不迁移**。A 项目调教好的习惯，到 B 项目从零开始。

## 做法

我的 USER.md 约 40 行，分五段：

```markdown
# 基本设定
- 时区 Asia/Shanghai；中文交流，术语保留英文

# 环境
- Ubuntu 22.04；Python 3.12 + uv；Node 用 pnpm
- 部署走 Docker Compose，配置在 /srv/xxx

# 偏好
- 先结论后理由；能一行代码说清就不写三段
- 不用 emoji，不用"希望这对你有帮助"

# 禁忌
- 不主动重构我没要求改的代码
- 不在回复里重复贴我已提供的报错全文

# 当前在做
- 数据管道迁移，Q2 前下线旧服务
```

注入方式按你的用法选：

- **OpenClaw workspace**：直接放 `USER.md`，会话启动自动进上下文；
- **CLI 类工具**：写在全局 `CLAUDE.md` 里 import，或启动脚本里 cat 进 system prompt；
- **走 MCP**：做一个极简 `get_user_profile` tool 按需返回，省 token。

## 踩坑点

- **写太长**。超过一屏就开始挤占真正的任务上下文，模型还会“过度服从”你两年前的偏好。控制在 50 行内。
- **信息过期**。旧项目路径、已完成的任务不删，Agent 会一直围着它们打转。把“当前在做”当看板，两周清一次。
- **塞敏感信息**。API key、内网 IP、真实客户名不要写——这文件大概率会被同步和备份。
- **和 AGENTS.md 打架**。项目级规则（用哪个包管理器）放项目文件；USER.md 只放跨项目成立的稳定事实。两边冲突时模型行为不可预测。
- **写成许愿清单**。“希望你更聪明”这类零信息句子别写，每条都该是可执行的约束或事实。

## 可复用建议

1. 建个 git 仓库托管它，改动走 commit，能追溯偏好怎么演化的。
2. 让 Agent 自己维护：发现它猜错了，直接说“把这条加进 USER.md”，比手动改快。
3. 分层：全局 USER.md（身份、语言、风格）+ 项目 AGENTS.md（技术栈、命令），别混。
4. 把“长期事实”和“临时状态”分成两段，临时状态过期就删。

## 总结

USER.md 本质上是把“每次会话重新自我介绍”的成本一次性付清。它不玄学，就是一个 40 行的纯文本文件，但在我用过的 Agent 优化手段里投入产出比排前三——因为你调的是模型输入里最稳定的部分：偏好不会天天变，环境不会天天变。写一次，维护成本趋近于零，之后每次会话都在吃复利。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/418bc24a6add7cd9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/9b17ad51447c841f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/5bf6a7eaf749b3fb.png)

