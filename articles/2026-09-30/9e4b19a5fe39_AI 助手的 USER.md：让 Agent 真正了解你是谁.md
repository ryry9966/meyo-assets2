---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 39838
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

玩 Agent 这一年多，大家在"让 Agent 更强"上的投入基本都集中在工具侧：接 MCP、写插件、堆 skills。但很少有人花十分钟告诉 Agent 一个更基础的事实——我是谁。

OpenClaw 的 workspace 模板里其实预留了 USER.md，和 AGENTS.md、MEMORY.md 并排。多数人的 USER.md 要么空着，要么只写了一行昵称。这份文件恰恰是 Agent 个性化里收益成本比最高的地方。

## 问题

没有稳定的用户画像时，Agent 的行为全靠猜：

- 排日程按服务器 UTC 算，下午五点的提醒在早上八点响；
- 不知道你用 macOS 还是 Armbian，给的命令混着 brew 和 apt；
- 不知道你在维护哪个仓库，每次回答都从零开始；
- "回复要短、代码带注释"这类偏好，每个会话都得重复说一遍。

MEMORY.md 是动态记忆，记"发生了什么"，事件越积越多，稳定事实反而被淹没。用户画像需要一份人工维护、低噪音的静态配置。

## 做法

1. **建文件**。在 workspace 根目录（默认 `~/.openclaw/workspace`）新建 USER.md，控制在一页以内。
2. **按五节填**：

```markdown
# 用户画像
## 基本信息与作息
称呼：老周；时区：Asia/Shanghai；工作时间 9:00–19:00，22 点后不安排提醒。
## 环境
主力机 macOS + Armbian 服务器；语言 Python/Go；部署用 Docker Compose。
## 偏好
回答先给结论；代码默认带中文注释；不用客套。
## 当前项目
- openclaw-gateway：自用网关，仓库在 ~/code/xxx
## 禁区
不要动 crontab；不要替我回复群消息。
```

3. **挂载**。Agent 开会话读的是 AGENTS.md，在末尾加一条规则："涉及日程、称呼、环境或个人偏好的任务，先读 USER.md 再执行。"没有这条，文件就是摆设。
4. **验证**。新开会话问一句"你现在知道我的哪些信息、你在哪个时区"，答不上来说明没加载成功。
5. **维护**。文件末尾加一节"候选事实"，日常让 Agent 把新学到的稳定信息追加进去，你每周审一次：同意的上移，过期的删掉。

## 踩坑点

- **别放敏感信息**。USER.md 每次都进上下文，也就意味着会进日志、备份和模型调用。密钥、证件号、住址一律不写，最多留引用路径。
- **别写成日志**。小作文稀释上下文，还会被 Agent 过度加权。只写稳定事实，一条一行。
- **和 MEMORY.md 划清边界**：USER.md 管长期稳定的"我是谁"，MEMORY.md 管动态的"发生过什么"。职责重叠会导致同一件事出现两个版本。
- **信息要带保鲜期**。项目三个月一换，画像里留着旧仓库，Agent 会一本正经地引用错误上下文。条目注明"截至 YYYY-MM"。
- **多机用户要同步**。workspace 放进 git，否则两台机器的画像会各自漂移。

## 可复用建议

- 画像控制在 30 行内，超了说明该拆分，而不是继续堆。
- 多重身份（公司/个人）用分节区分，不要分文件，避免 Agent 拿错语境。
- "候选事实"工作流同样适用于 TEAM.md：Agent 提议、人裁决，文件只收敛不发散。
- 把 USER.md 当代码对待：git 管理、有 review、有清理周期。

## 总结

USER.md 本质上是对自己做的一次 prompt engineering：把"我是谁"从会话里的口头约定，变成版本化的静态配置。十分钟的维护成本，换来的是 Agent 不再反复问你同一个问题。工具链越复杂，这份画像的价值越大——因为所有上下文里，唯一不变的就是你自己。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/5f8826dc17beabf0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/bc1912afb9d95dfd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/40aaa87523fdeadf.png)

