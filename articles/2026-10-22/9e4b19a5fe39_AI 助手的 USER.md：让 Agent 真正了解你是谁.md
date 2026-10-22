---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 42254
source: 综合讨论
publishedAt: 2026-10-22
---

# 背景

Agent 的会话本质上是短期的。即便有 MEMORY.md 记录过往交互，"你是谁"这件事——你的角色、环境、偏好、红线——依然散落在一次次重复说明里。OpenClaw 在 workspace 里为此留了一个专门槽位：`USER.md`。它和另外两个文件分工明确：AGENTS.md 管"怎么干活"，SOUL.md 管"Agent 是谁"，USER.md 管"用户是谁"。

# 问题

没有 USER.md 时，典型摩擦包括：

- 每个新会话开头都在自我介绍：包管理器用 pnpm、注释写中文、别加 emoji——说了一遍又一遍；
- Agent 靠默认假设干活：给你 apt 而你在 macOS，给你长篇铺垫而你要结论；
- 偏好信息混进事件记忆，检索全凭运气。

# 做法

1. **位置**：`~/.openclaw/workspace/USER.md`。OpenClaw 在会话启动时会连同 AGENTS.md / SOUL.md 一起读入。
2. **结构**（建议控制在 60 行内）：

```markdown
## 我是谁
后端工程师，主要写 Go 和 Python。
## 环境
macOS + homebrew；Neovim；git 默认 rebase。
## 偏好
- 结论先行，解释放最后
- 注释用中文，标识符英文
- 不要主动加 emoji
## 当前项目
- 每个项目一句话，细节放项目内文件
## 红线
- 不要动 ~/.ssh 和生产配置
- 未经确认不要 git push
```

3. **写事实，别写愿望**。"注释用中文"是事实；"我希望更自律"是愿望——后者会让 Agent 周期性提醒你，很烦。
4. **纳入 git**。workspace 建仓库，USER.md 的每次改动可审计、可回滚。
5. **更新节奏**。环境大改时手动改；平时让 Agent 出 diff："根据最近两周会话，给一版 USER.md 更新建议，只列改动。"确认后再合并，避免它自作主张。

# 踩坑点

- **太长**。USER.md 是每次会话的固定 token 开销，超过百行就该拆：项目细节下沉到各项目目录的说明文件，让 Agent 按需读。
- **敏感信息**。API key、内网拓扑、客户名不要写。这份文件会被完整注入上下文，可能随日志或导出外流。
- **职责混淆**。行为规则进 AGENTS.md，人格进 SOUL.md，USER.md 只放"关于用户的事实"。混着写，后期没法维护。
- **过期信息比没有更糟**。Agent 会自信地引用你三个月前写的技术栈和路径，错误画像比空白画像危害更大。

# 可复用建议

- **验证生效**：开一个全新会话，问 Agent"你现在对我的了解有哪些？"，看它复述的是否是你写的重点。
- **全局 + 项目分层**：全局 USER.md 放稳定事实；项目级信息放仓库内说明文件，并在 AGENTS.md 里加一条规则引导 Agent 进入项目时先读它。
- **定期审计**：让 Agent 当 reviewer——"读 USER.md，列出你认为可能过时的三条"，比自己回忆省力。

# 总结

USER.md 的价值不在"写得多"，而在"最小充分"：一段让 Agent 不再问蠢问题、不再用错环境的信息就够。短、准、新，再加上版本化，它就是你能给 Agent 的性价比最高的一份配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/0a1b635f0ac2f355.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b8227aa1e29a9142.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/300236d974d6913c.png)

