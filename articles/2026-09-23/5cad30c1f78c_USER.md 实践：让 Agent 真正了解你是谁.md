---
title: USER.md 实践：让 Agent 真正了解你是谁
feedId: 38539
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 workspace 里躺着几个固定的记忆文件，大多数人把精力花在 MEMORY.md 的会话沉淀上，却经常忽略一个更简单的文件：USER.md。它不是日志，而是一份关于"你是谁"的静态档案，每次会话启动时被注入上下文，相当于 system prompt 里的一段用户侧写。

## 问题

没有 USER.md 时，Agent 对你的认知是零起点：

- 每次都要重新解释"我用的是 Arch，别给我 Ubuntu 的命令"
- 时区、称呼、语言偏好反复确认
- 明明说过很多遍"回复要短"，它还是长篇大论

MEMORY.md 能积累事实，但它是概率性的：检索靠相似度，内容有噪声。而"用户是谁"属于高置信度的长期信息，放在一个手工维护、每次必载入的小文件里，比指望记忆检索靠谱得多。

## 做法

USER.md 放在 workspace 根目录（`~/.openclaw/workspace/USER.md`），纯 Markdown，建议按分区组织：

```markdown
# USER
## 基本信息
- 称呼：老周
- 时区：Asia/Shanghai，中文交流
## 环境
- Arch Linux + Wayland，zsh；无独立 N 卡，别推 CUDA 方案
## 偏好
- 回复直接给结论和命令，不要铺垫，不用 emoji
- 默认 pnpm；动生产环境前必须先向我确认
## 当前
- 在做家庭服务器自动化，树莓派 5 上跑
```

三条原则：用列表不用段落；只放长期稳定的事实；控制在 50 行以内，这段内容每次会话都要付 token 成本。写完用 git 管起来，改动可追溯。

## 踩坑点

1. **写成自传**。三个自然段不如十个短句，模型对结构化列表的遵循度明显更好。
2. **和 SOUL.md 混淆**。SOUL 定义 Agent 的人格，USER 定义你。"助手要简洁"属于 SOUL；"用户讨厌 emoji"才属于 USER。
3. **塞进易变状态**。待办、项目进度该去 MEMORY.md 或独立文件。USER.md 一旦变成垃圾场，信噪比就崩了。
4. **敏感信息裸奔**。这个文件会随上下文发给模型服务商，密钥、密码、精确住址别写。确有隐私约束，写规则而不是裸数据。
5. **以为改了立刻生效**。下次会话才载入，且遵循程度取决于模型。改完可以问一句"你目前知道我哪些信息"做验证。

## 可复用建议

- **把维护外包给 Agent**：会话结束前让它"根据本次确认的关于我的长期事实，给出 USER.md 的合并建议"，人工审后再合入。文件能持续生长，又不会失控。
- **分层记忆模型**：USER.md 是高置信层（必载入），MEMORY.md 是概率层（按需检索），职责分开，别互相兜底。
- **团队场景复制该模式**：做一个 TEAM.md 沉淀团队约定（部署流程、review 规矩），比 wiki 离 Agent 更近一步。

## 总结

USER.md 没什么黑魔法，就是一份被约定载入上下文的 Markdown。但它把"了解用户"从玄学变成了可版本管理、可 review、可 diff 的工程问题。花二十分钟写第一版，之后每月修订一次，你会发现 Agent 终于不再问你"请问您用的是哪个系统"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/b70726aaf8801ed4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/c12bf5132ef9ee09.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ec49bf748c2b53fa.png)

