---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 42536
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

OpenClaw 的 agent 行为不是写死在代码里的，而是由 workspace 下的一组 Markdown 决定：`SOUL.md` 管行为准则，`USER.md` 管用户偏好，`MEMORY.md` 管记忆，而 `IDENTITY.md` 管"它是谁"。默认模板只有几行——name、emoji、creature、vibe。很多装完就跑的人直接跳过了它，结果 agent 名字一直是默认值，几轮对话下来语气也飘忽不定。

## 问题

实践中我们遇到三个具体的痛点：

1. **跨会话人格漂移**：今天像客服，明天像诗人，system prompt 里没有稳定锚点。
2. **多实例难以归因**：两个 agent 挂在同一渠道，输出没有标识，翻日志时分不清谁说的。
3. **调 persona 全靠临时改提示词**：改完即丢，没法版本化，更没法回滚。

## 做法

1. **找到文件**：workspace 根目录下的 `IDENTITY.md`（默认在 `~/.clawd/workspace/`，多实例各自独立一份）。
2. **写最小可用版本**，控制在十行左右：

```markdown
# IDENTITY
- name: 阿钳
- emoji: 🦀
- creature: 一只工程师养的机械蟹
- vibe: 克制、务实，先给结论再给依据
- 语言: 默认中文，技术术语保留英文
```

3. **和 SOUL.md 分工**：IDENTITY 回答"我是谁"（名字、形象、口吻），SOUL 回答"我怎么做"（边界、价值观）。两份文件不要重复写，重复反而互相稀释。
4. **建立进化机制**：把 workspace 纳入 git。发现语气不对就改 IDENTITY.md，提交信息写清楚改了什么、为什么改——commit 历史就是人格的变更日志。改完开新会话生效。
5. **多实例场景**：每个 agent 用独立 workspace 和不同的 emoji，群聊和日志一眼就能区分来源。

## 踩坑点

- **别写长篇人设**。IDENTITY.md 每次会话都会注入上下文，几十行的"性格描写"会稀释关键指令，还烧 token。实测 15 行以内足够。
- **别把任务和事实混进来**。"每天爬一次数据"属于 TOOLS/AGENTS，"用户喜欢简洁回复"属于 USER.md，身份文件只放身份。
- **改完不生效**：workspace 文件在会话建立时注入，旧会话不会热更新，要新开会话（或重启网关）才能看到变化。
- **emoji 填了非法字符、avatar 写了绝对路径**，换台机器就断，尽量用相对 workspace 的路径。
- **全英文人设配中文用户**，语气容易发怪；vibe 直接用中文写，实测更贴。

## 可复用建议

- 把 IDENTITY.md 当**配置代码**对待：git 管、diff 审、小步改，一次只动一两个 trait，方便定位是哪次改动引起的语气变化。
- 文件尾部加一个 changelog 小节，人能读懂，回滚也快。
- 定一个回顾节奏，比如每两周翻一遍近期会话，各记一条"希望它更像 / 不希望它像"，再决定要不要动文件。
- 团队共用同一人设时，用 PR 评审改人格，避免有人悄悄把 agent 调成段子手。

## 总结

IDENTITY.md 看起来只是个自我介绍，实际是把"人格"从一次性提示词升级成受版本控制的配置。四个要点：**写短、分工清楚、进 git、按会话生效**。做到这几点，agent 的性格就既稳定又可迭代——这比任何花哨的 system prompt 模板都更接近"养一个长期助手"的正确姿势。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/85d4817e0fbbd4c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/7b70431ea77f4d58.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/01f1111971258b79.png)

