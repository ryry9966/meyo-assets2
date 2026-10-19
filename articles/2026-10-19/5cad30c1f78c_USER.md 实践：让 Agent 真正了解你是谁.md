---
title: USER.md 实践：让 Agent 真正了解你是谁
feedId: 42017
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

用 OpenClaw 做日常助手一段时间后，你会发现一个不对称：SOUL.md 和 AGENTS.md 定义了"Agent 是谁"，但 Agent 对"你是谁"几乎一无所知。每个新会话它都要重新猜你的时区、技术栈、回复偏好，以及哪些事绝对不能自动做。这些信息其实散落在聊天记录里，但几百条消息之后，靠上下文捞回来基本不现实。

## 问题

没有稳定的用户画像时，常见症状包括：

- 每次都要重新解释环境：macOS 还是 Linux、路径习惯、用哪个包管理器；
- 风格错位：你想要结论，它给你长篇教程；
- 反复踩红线：动不动就想自动 commit、自动清理文件；
- MEMORY.md 记的是事件流水（"今天部署了 X"），而不是关于你的稳定事实。

一句话：Agent 侧有配置文件，人侧没有。

## 做法

1. 在 workspace 根目录建 `USER.md`。OpenClaw 会话启动时会把 workspace 下这些文件一起注入上下文，不需要额外接线。
2. 控制在一屏以内，只写"能改变 Agent 行为"的事实：

```markdown
# User
## 基本信息
- 称呼：老周；时区 Asia/Shanghai；中文交流，技术名词保留英文
## 环境
- macOS 15 + Homebrew，项目在 ~/code，shell 用 zsh
## 偏好
- 先给结论，细节后置；代码注释少而准；commit message 用祈使句
## 红线
- 不自动 push；不碰 /etc；任何删除操作先确认
```

3. 分清职责：SOUL.md / AGENTS.md 管 Agent 人格和行为，USER.md 管"你是谁"，MEMORY.md 管事件。同一主题只在一处维护。
4. 进 git。Agent 行为退化时，先 diff 一下画像，很多时候根因就是某次改动。
5. 写 MCP 工具或插件时，环境段尤其值钱——Agent 能据此优先选对平台的调用方式，少走弯路。

## 踩坑点

- **写成简历**：背景故事、长篇自述，除了烧 token 没用。每条事实问一句"删掉后 Agent 行为会变吗"，不会就删。
- **放时效性内容**：当前迭代任务、临时状态，放 USER.md 一周就腐烂，该进 MEMORY.md 或项目文件。
- **敏感信息**：密钥、真实住址、完整手机号不要写。USER.md 会被注入各种工具调用的上下文，泄露面比想象中大。
- **放任 Agent 自由追加**：画像会越滚越大。给自己设上限，比如 40 行，超了强制 prune。
- **与 AGENTS.md 冲突**：两边都定义回复风格，Agent 会摇摆。保持单一 source of truth。

## 可复用建议

- **冷启动**：让 Agent 采访你十个问题生成草稿，你删改定稿，比从零写快得多。
- **维护"纠错区"**：每次你纠正 Agent，让它提议一条对应事实，攒着定期合并进正文。
- **每季度清一次**：过期事实比没有事实更糟，因为它带着你的信任背书。
- **多机同步走 git**，别用网盘，避开换行符和冲突的坑。

## 总结

USER.md 是整条 Agent 链路里性价比最高的一环：十几行、十分钟，换来所有后续会话的默认体验提升。把它当"人侧的配置即代码"来维护——小、准、可版本化，和你对待任何配置文件一样。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/7f6c8697dd630212.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/6a3e07ab619fd93e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/b0f0f2ecc3871460.png)

