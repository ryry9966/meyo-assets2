---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 38765
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 agent 在一个 workspace 里干活，但每次会话的上下文都是从头开始的：上次怎么定的规矩、哪个目录放什么、哪些事不能碰，agent 一概不记得。人接入新项目有 README，agent 对应的就是工作空间根目录的 `AGENTS.md`——每次会话启动时自动读入 system context。它本质上是一份写给 AI 的使用手册：我是谁、东西在哪、什么能做、什么不能做。

## 问题

没有这份手册的典型日常：

- 每次开新会话，在聊天里重复交代"先看 scripts/ 目录，别动 main 分支"；
- agent 自己猜约定，按它的习惯来，你第二天再返工；
- 规矩散落在历史对话里，无法沉淀，同一个坑反复踩；
- 多个 agent 实例各干各的，风格互相打架。

根因是：**稳定的规则不该住在易失的对话上下文里，而该住在工作空间文件里。**

## 做法

第一版只要五个板块，全部用短句列表：

```markdown
# 我是谁
- 跑在 macOS，时区 Asia/Shanghai，用中文回复
- 不碰 .env 和 credentials/，除非我明确要求

# 项目地图
- notes/ — 每日记录，文件名 YYYY-MM-DD-主题.md
- scripts/ — 维护脚本，用前先跑 --help

# 常用命令
- 备份：./scripts/backup.sh
- 检查：./scripts/lint.sh

# 工具边界
- filesystem MCP 只用于读取，写文件走内置工具

# 硬规则
- 不执行 git push / git commit
- 拿不准就先问，别按猜的来
```

几个关键点：

1. **只有根目录这份会被自动加载**。子目录里的嵌套 AGENTS.md 不保证被读，要么在根文件里写明"改 notes/ 前先读 notes/AGENTS.md"，要么干脆不嵌套。
2. **把纠错沉淀成规则**。agent 同一个错犯两次，别再在聊天里骂，把结论压成一行写进 AGENTS.md。这是这个文件性价比最高的维护动作。
3. **分层存放**：稳定章程进 AGENTS.md，每日流水进 MEMORY.md，人格气质进 SOUL.md。混着写的结果是每次记笔记都在"修宪"。
4. **进 git，像审代码一样审它**。OpenClaw 自己的文档仓库也维护着一份 AGENTS.md，可以参考其写法，但别整段照抄。

## 踩坑点

- **写太长**。它每个会话都占上下文，百行以内为宜。什么都重要等于什么都不重要。
- **写空话**。"小心一点""聪明点干"没有约束力；要写成可判定的动作："删除文件前先列出清单，等我确认。"
- **写机密**。这份文件会被发给模型，密钥一行都不能出现，写"凭据在 .env，不要读取"就够。
- **当硬约束用**。AGENTS.md 是强先验，不是沙箱。真正的红线靠网关权限、工具白名单和沙箱配置兜底，文件只负责把 agent 引向正确方向。

## 可复用建议

- 最小骨架先跑起来，五条起步，不要追求一次写完美。
- 给自己定一条元规则：同一个纠错发生两次，才写进 AGENTS.md，防止文件膨胀。
- 命令全部保持可复制粘贴的形态，agent 会直接执行。
- 每季度清理一次：删掉过期路径和失效规则，删除与新增同样重要。

## 总结

AGENTS.md 的写造成本很低，收益是每次会话都复用一次。它把"聊天里再教一遍"变成"写一次长期生效"，把个人经验变成 workspace 资产，也让多个 agent 之间有了统一的行为基准。如果你现在只有一个能跑的 OpenClaw 却没有 AGENTS.md，今天就花十分钟写下前五行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/3df5b650e5886b87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/e4c7fc8ce3686fdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/b56663f5bf88b3f7.png)

