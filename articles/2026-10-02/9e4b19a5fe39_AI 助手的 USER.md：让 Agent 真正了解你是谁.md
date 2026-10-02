---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 40067
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

用过 OpenClaw 一段时间后你会发现一个尴尬的事实：Agent 每次新会话都是"失忆"的。你用 Python 3.12、主力机是 Linux、commit message 只写英文、时区 UTC+8——这些信息你得反复交代。对话式记忆（MEMORY.md）擅长记事件，但"你是谁"这类稳定的静态事实，更适合一个专门的文件。

这就是 USER.md 的定位：workspace 里描述"用户本人"的持久化档案。它和 SOUL.md（Agent 的性格）、AGENTS.md（行为规则）各管一层，互不越界。

## 问题

没有 USER.md 的典型症状：

- 每次都要重复交代环境："我的服务器是 Ubuntu 22.04，docker compose 部署……"
- Agent 给出放之四海皆准的建议，不符合你的技术栈和习惯
- 换个项目、换个设备，积累的上下文全部清零

而动手写了 USER.md 的人，又常掉进另一个坑：把它写成简历加日记的混合体，几千字塞满上下文，真正关键的几条反而被稀释。

## 做法

我的 USER.md 目前稳定在 40 行以内，结构如下：

```markdown
# User Profile

## 环境
- 主力机：Linux (Arch)，shell 是 zsh
- 部署：docker compose，一台 2C4G 的 VPS
- 时区：UTC+8，工作日 10:00-19:00 在线

## 技术偏好
- 后端 Go / Python，数据库 PostgreSQL
- 注释和 commit message 用英文
- 回答默认给可运行的最小示例，不要长篇理论

## 红线
- 不要主动重启生产容器
- 涉及删除操作，先列出影响范围再执行

## 当前上下文（随时间更新）
- 正在把博客从 Hexo 迁到 Astro
```

三条要点：

1. **写事实，不写叙事**。每条一行、可被直接引用。避免"我性子急喜欢快"这种模糊描述，改成"方案讨论给两个选项直接选"。
2. **和 MEMORY.md 分工**。USER.md 放稳定属性，MEMORY.md 放事件与决策。判断标准：一年后仍然成立的进 USER.md，否则进 MEMORY。
3. **维护节奏**。每月扫一次，删过期项。"当前上下文"一节最容易腐烂，宁缺毋滥。

## 踩坑点

- **塞敏感信息**。API key、服务器密码绝不进 USER.md——它是会被整段读进上下文的明文文件。密钥交给环境变量或 secret 管理。
- **写成愿望清单**。"希望你更主动一点"这类要求属于 SOUL.md / AGENTS.md，放错位置 Agent 会困惑优先级。
- **过期信息比没有信息更糟**。换了技术栈没更新，Agent 会持续按旧栈给方案，而且错得"很懂你"，更难察觉。
- **太长**。超过一屏后信噪比急剧下降，我的经验是硬性控制在 50 行内，超了就做减法。

## 可复用建议

- 一个简单触发器：**同一件事向 Agent 解释超过三次，就必须进 USER.md**。
- 模板四段式：环境 / 偏好 / 红线 / 当前上下文，够用且不会写飞。
- 新环境初始化时，先让 Agent 和你对话五分钟收集信息，再整理成 USER.md，比凭空回忆写得更准。
- 多设备、多 Agent 场景，把 USER.md 纳入 git 或同步盘，保持各端一致。

## 总结

USER.md 的价值不在文件本身，而在于把"让 Agent 懂我"从一次次重复对话，变成一次工程化维护的静态资产。花 20 分钟写好初版，之后每次维护只需一两分钟，换来的是所有会话里 Agent 的回答都落在你的真实语境之内。就个性化投入产出比而言，这是目前最值得先做的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/4ae0f472c24744e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/fef47663b9c19067.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/3ce1c7da0d8cf1b9.png)

