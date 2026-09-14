---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 37480
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

用 OpenClaw 这类常驻 Agent 时，一个反复出现的不满是：换了个会话，它就不认识你了。时区、主力语言、命名习惯、正在推进的项目，每次都得重新交代。系统提示词是通用的，模型本身没有关于"你"的先验。OpenClaw 的解法很朴素——把用户的自我描述放进工作区的一个 Markdown 文件：`USER.md`。每次会话启动时它会被注入上下文，Agent 从第一句话起就带着你的背景干活。

## 问题

没有 USER.md 时的典型体验：

- 每个新会话重复自我介绍，成本高且容易遗漏关键约束；
- Agent 给出"正确但平庸"的回答，因为它不知道你的上下文；
- 手机、桌面、cron 任务触发的不间断会话行为不一致，像三个人在干活。

## 做法与步骤

1. 在工作区根目录创建 `USER.md`（默认路径 `~/.openclaw/workspace/USER.md`）。
2. 只写**能改变 Agent 行为**的信息，而不是个人简历。我的模板分五段：

```markdown
# USER.md
## 基本信息
- 称呼：阿黄；时区：Asia/Shanghai；语言：中文优先
## 技术栈
- 主力：TypeScript / Node；部署：自建 Docker + Caddy
## 偏好
- 先给最小可用实现，不做提前抽象
- 回复先给结论，推理过程放后面
## 当前在做
- 家庭服务器自动化（Home Assistant + 自研 MCP 工具）
## 红线
- 不要主动 git push；不要动 /etc 下的配置
```

3. 控制篇幅。我的版本长期保持在 60 行以内，越短命中率越高。
4. 纳入 git 管理。Agent 犯同样的错，就把纠正写回去。

一个判断标准：USER.md 里的每一条，都应该能被观察到"它改变了 Agent 的输出"。写不出来就删掉。

## 踩坑点

- **塞太多**。有人把整个项目历史写进去，关键信息被稀释，模型反而抓不住重点。USER.md 只管"我是谁"，项目细节放 AGENTS.md 或各项目自己的文档。
- **写了敏感信息**。API key、内网地址、密码不要放这里，它会随每次上下文注入。
- **信息过期**。换栈、换岗后忘了更新，Agent 会拿着三个月前的假设自信输出。
- **写得模糊**。"我喜欢简洁的代码"没有信息量，"Python 用 ruff，不用 black"才能被执行。

## 可复用建议

- 让 Agent 帮你写第一版：直接让它"采访我五分钟，然后起草 USER.md"，比从零写快得多。
- 职责分开：USER.md 管"你是谁"，AGENTS.md 管"怎么干活"，SOUL.md 管"语气人格"，三者别混。
- 可以授权 Agent 自己维护 USER.md，但建议在 AGENTS.md 里约束它**只追加、不重写**，避免一次误操作清掉你的积累。
- 维护节奏：每当你第二次纠正 Agent 同一件事，就追加一条。这是成本最低的迭代方式，每月 git diff 复查一次即可。

## 总结

USER.md 的价值不在文件本身，而在于它把"调教 Agent"从一次次重复的对话，变成一份可版本控制、可复查、可跨设备迁移的静态资产。花二十分钟写好它，之后的每个会话都在为你省时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0297c477e41fd1c2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/7753e1cb769ec660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/41bee7e2e7798d6d.png)

