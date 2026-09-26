---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 39125
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 这类常驻 Agent 有个特点：它长期在线、多入口（WhatsApp / Telegram / CLI / cron），但每次对话开始时，它对"屏幕对面的人"几乎一无所知。AGENTS.md 和 SOUL.md 回答了"Agent 该怎么做事、以什么姿态做事"，却没有一个文件回答"它在为谁做事"。

## 问题

不写用户上下文，代价是持续发生的：

- 定时任务按 UTC 排，每天早报晚到八小时；
- 本机装着 pnpm，它每次都给出 `npm install`；
- 明明约定中文交流，新会话又切回英文长篇解释；
- 最烦的是：这些事要在每个新会话里重新交代一遍。

这不是模型能力问题，是"它不认识你"。改 system prompt 治标不治本——换个入口、开个新会话就丢。

## 做法

核心就一步：在 workspace 根目录建一个 `USER.md`（如 `~/.openclaw/workspace/USER.md`），它会随系统提示词注入每一轮会话。内容我固定成四段，全部用短句和列表：

```markdown
# USER.md

## 基本事实
- 时区 Asia/Shanghai（UTC+8），排定时任务一律换算
- 中文交流；代码、注释、commit message 用英文

## 环境
- macOS arm64，shell 是 zsh
- 包管理器 pnpm，不要建议 npm / yarn
- 常用项目：~/work/openclaw-plugins

## 协作偏好
- 直接给结论和命令，不要铺垫
- 改动超过 3 个文件时，先列计划再动手

## 红线
- 生产环境操作必须先确认
- 未经确认不主动给外部联系人发消息
```

三条配套纪律：

1. **控制篇幅**：40 行以内。它每轮都占 context，多一行就多一分成本，也会稀释真正的指令。
2. **在 AGENTS.md 里挂钩**：加一句"与用户偏好冲突时以 USER.md 为准；被我纠正后，把结论写回 USER.md，而不是只在对话里道歉"。
3. **workspace 进 git**：Agent 自改 USER.md 时你能 diff 回看，改坏了随时 revert。

## 踩坑点

- **写成简历**。履历对执行没帮助，判断标准只有一条：这行能不能改变 Agent 的某个行为？不能就删。
- **放敏感信息**。USER.md 每次请求都会发往模型服务端，密钥、详细住址、证件号绝对不要写。
- **放任 Agent 自由维护**。它会往里塞"用户今天夸了我"这类流水账，固定模板加人工 diff 审查是底线。
- **职责混淆**。AGENTS.md 管怎么干活，SOUL.md 管什么性格，USER.md 只管为谁干活，三者别互相渗透。

## 可复用建议

- 一个事实一行，写成可执行的祈使句："回复用中文，代码注释用英文"比"我喜欢中英混排"好用得多。
- 每周让 Agent 对照近期对话，提议 2–3 条 USER.md 增补，人工确认后合入，让维护变成轻量例行公事。
- 工作 / 个人双身份就拆两个 workspace，各配一份，避免时区和红线互相污染。

## 总结

USER.md 是我给 OpenClaw 做过的性价比最高的一次配置：半小时的活，换来 Agent 从"每次都要重新认识你"变成"持续积累对你的理解"。它的价值不在文件本身，而在那条纪律——凡是纠正过 Agent 一次的事，都该沉淀回 USER.md。沉淀得越勤，重复解释就越少。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/b67b2d9baf50ad42.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/ec60474cea83c13a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/182e0b42f52aca79.png)

