---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38769
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的 workspace 里，几个 Markdown 文件各司其职：SOUL.md 管行为与性格，USER.md 存用户画像，而 IDENTITY.md 管的是"这个 agent 是谁"——名字、形象原型、emoji、头像、主题色。文件很小，每次会话启动时随 workspace 注入上下文，决定了 agent 在对话里如何称呼自己、在群聊和不同渠道里如何被呈现。

## 问题

不配置 IDENTITY.md 时，实践中常见三类问题：

1. **自我指称不一致**：新会话里自称 A，上下文压缩后自称 B，看起来像中途换了人。
2. **多实例混淆**：同时跑"工作助手"和"家庭运维"两个 agent，群聊里分不清谁在说话。
3. **跨渠道呈现漂移**：没有固定头像和主题色，同一个 agent 在不同客户端看起来完全不同。

## 做法

1. 定位 workspace（默认 `~/.openclaw/workspace/`），若没有 IDENTITY.md，按模板新建。
2. 只填事实字段：`name`、`creature`、`avatar`（相对路径，图片放 workspace 内）、`theme`，再加一行 tagline 圈定职责边界，比如"负责家庭服务器与自动化脚本"。
3. 篇幅控制在十行以内。它是每次会话的固定 token 开销，长内容应放进 SOUL.md。
4. 开新会话验证：直接问"你是谁、负责什么"，回答应与文件一致。
5. 多实例场景下，各 workspace 分别配置，确保名字和颜色可区分。

## 踩坑点

- **写太长**：IDENTITY.md 是身份事实，不是人格说明书。行为规则放 SOUL.md，两边混写容易冲突且浪费 token。
- **措辞含糊**："你可以叫我 X"不如直接写"名字是 X"，前者在长会话中容易被稀释。
- **头像路径**：用 workspace 内相对路径，确认 gateway 进程有读取权限；绝对路径换机器即失效。emoji 选通用字符，部分终端渲染不全。
- **让 agent 自改身份文件**：我没开放这个权限。身份变更应走 git 提交 + 人工 review，否则某次对话里 agent"灵机一动"改了名，下次会话就被固化。
- **workspace 被 gitignore**：很多人忘了把 workspace 纳入版本控制，迁移环境时身份文件直接丢失。

## 可复用建议

- 把 IDENTITY.md 当 IaC 管理：提交到私有仓库，每次修改看 diff，可随时回滚。
- 一个 agent 一套身份三件套：名字 + emoji + 主题色对齐，群聊中一眼可辨。
- 小步进化：先起一个通用名字跑起来，等职责稳定后再补 tagline 和形象，比一开始虚构"完美人设"务实得多。

## 总结

IDENTITY.md 解决的不是能力问题，而是一致性问题。十行以内的版本化文件，换来的是自我指称稳定、多实例可辨、跨渠道呈现统一。先写最简版本，再随使用场景迭代，是成本最低、也最不容易跑偏的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/8ec0431cbf3de902.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f4ddf6933a3a6a99.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/b656cf31aa54435f.png)

