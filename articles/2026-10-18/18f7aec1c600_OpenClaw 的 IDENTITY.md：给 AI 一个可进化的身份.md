---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41931
source: 综合讨论
publishedAt: 2026-10-18
---

# 背景

OpenClaw 的 Agent 是长期运行的：挂在 Telegram、WhatsApp 这些渠道上，会话不断被重置，上下文隔一段时间就重建一次。系统提示词、工具说明、用户偏好分散在 config 和若干 md 文件里。时间一长你会发现一个问题——"它是谁"这件事，没有被任何一处明确定义。

IDENTITY.md 就是补这个缺口的：workspace 根目录下的一个纯 Markdown 文件，每次会话启动时随 AGENTS.md、SOUL.md 一起注入上下文，给 Agent 一个稳定、可版本化、可迭代的身份声明。

# 问题

没有它时常见的几种症状：

- **人格漂移**：今天高冷明天话痨，跨会话表现不一致；
- **渠道不一致**：同一个 Agent 在 Telegram 和 WhatsApp 里像两个人；
- **身份散落在代码里**：名字、口头禅硬编码在 prompt 字符串中，改一次要动配置，还没法追溯改了什么。

本质是：身份被当成了代码，而不是数据。

# 做法

1. 在 workspace 目录（`~/.openclaw/workspace/`）新建 IDENTITY.md，字段化地写：

```markdown
# IDENTITY
- Name: 阿钳
- Creature: 寄居蟹机器人 🦀
- Vibe: 务实、简洁，先结论后细节
- 汇报风格: 卡住超过 10 分钟必须主动上报
- 红线: 不替用户做财务决定；不确定时明说"我不确定"
```

2. **控制篇幅**。这个文件每次会话都进上下文，建议 20 行以内、几百 token 封顶。
3. **明确分工**：IDENTITY.md 管"我是谁"，SOUL.md 管"性格与边界"，AGENTS.md 管"怎么干活"。别把任务指令塞进身份文件。
4. **Git 管理，当 prompt 迭代**：Agent 表现不符合预期时，别在聊天里纠正，改文件、写清 commit message，半年后回看就是一份身份演化史。

# 踩坑点

- **写太长**：见过 80 行的 IDENTITY.md，每次会话白烧 token，关键信息反而被稀释；
- **改完不生效**：身份文件在会话启动时加载，老会话里还是旧人格，重开新会话再验证；
- **多设备不同步**：家里和 VPS 各一份，身份各自漂移，用 git 同步或软链解决；
- **人设过重**：表演性描述太多，模型会"入戏"，回答事实问题时开始添油加醋，身份要克制；
- **敏感信息入档**：这文件会进每次上下文，别写密钥、住址、私人信息。

# 可复用建议

- 把身份当 SLO 维护：每个字段都可验证——"汇报风格"这条，最近一周的对话里观察得到吗？
- 多 Agent 场景按 workspace 隔离，一份 IDENTITY.md 对应一个 Agent；
- 团队新人 onboarding 先读一遍它，比读十页文档更快理解这个 Agent 的行为约定；
- 用五字段模板起步：Name / Creature / Vibe / 汇报风格 / 红线，覆盖 90% 场景。

# 总结

IDENTITY.md 的价值不在"让 AI 有性格"，而在把身份从散落的 prompt 字符串，变成一份可版本化、可 review、可回滚的数据资产。改动成本从"改配置重启"降到"改一行 md 开个新会话"，持续迭代这才真正可行。身份是长出来的，不是一次性写死的——文件就位之后，剩下的交给时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/6faf2251e0804bbf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/bf94c9ddab55c6ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/ae355baba9fe295d.png)

