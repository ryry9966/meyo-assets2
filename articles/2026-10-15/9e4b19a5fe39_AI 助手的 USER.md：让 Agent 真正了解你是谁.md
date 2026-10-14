---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 41559
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

跑 OpenClaw 这类长期在线的个人 Agent，大家一般会写 AGENTS.md 约束行为、接好 MCP 工具、靠 memory 积累上下文。但有个问题常被忽略：Agent 知道该怎么做，却不知道"你"是谁。行为规范有了，用户画像还是空白。

## 问题

每次新会话，Agent 对我一无所知：

- 我在 Arch 上用 fish + neovim，它默认给 Ubuntu + VSCode 的命令
- 我要求中文回复、先结论后细节，它时不时切回英文长篇大论
- 我在 UTC+8，它排日程总按别的时区算
- 同样的自我介绍，换一个会话再说一遍

单看都是小事，但每天发生几十次。本质是"用户上下文"没有持久化载体——memory 解决"发生过什么"，解决不了"我是谁"。

## 做法

在 workspace 根目录（与 AGENTS.md 同级）建一个 `USER.md`，核心结构：

```markdown
# USER.md
## 基本信息
- 时区 UTC+8，中文优先，回复先结论后细节
## 技术环境
- OS: Arch Linux；shell: fish；编辑器: neovim
- 主力: Python / Go；部署: 自托管 Docker + Nginx
## 偏好
- 代码示例给完整可运行版本，不写伪代码
- 不确定的命令先确认再执行，尤其是 rm 和系统服务
## 边界
- 不要动 ~/archive；不要自动升级内核
```

四点经验：

1. **控制在 50~80 行**。这个文件每次进上下文，太长既费 token 又稀释重点。
2. **只写稳定事实**。当前项目进展放 memory，别塞进来，否则维护频率太高。
3. **用 git 管理**。改偏好时有据可查，回滚也方便。
4. 如果你的模板不会自动加载，就在 AGENTS.md 里加一句："每次新会话先读 USER.md 再开始工作"。写完开新会话验证：问"我的主力语言和编辑器是什么"，答不上来说明没被加载。

## 踩坑点

- **写成自传**：第一版我写了 200 行，Agent 反而抓不住重点，砍到 60 行后明显更准。
- **敏感信息**：任何有文件读取权限的 MCP/插件都可能看到这份文件。密钥、密码、具体住址，一律不放。
- **和 AGENTS.md 混写**：行为规则归 AGENTS.md，个人事实归 USER.md，两边重复迟早漂移。
- **偏好写成绝对命令**："永远只用 Python"会让 Agent 在该给 shell 脚本时也硬套。写"默认优先"比"必须"好。
- **过期信息**：换了技术栈没更新，Agent 会基于旧假设给建议。我设了每月复查一次的提醒。

## 可复用建议

- 核心四段（基本 / 环境 / 偏好 / 边界）基本通用，可以直接当模板沉淀
- 多个 Agent 共用时 symlink 到同一份，保持单一事实源
- 团队场景可派生一份 TEAM.md 放共享约定，和个人偏好分开
- 每次修改后用新会话做一次"身份抽查"，成本十秒

## 总结

USER.md 不是什么高级机制，就是一份被约定加载的纯文本。但它把"Agent 不了解我"从每个会话都要修补的问题，变成一次性维护的配置。写它大约半小时，之后每次对话都在收利息。如果你已经在维护 AGENTS.md，没有理由不同时维护 USER.md。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/ffd58f138c53aa7b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/868de6f00ef0b5a8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/bda3ebfcd855ca57.png)

