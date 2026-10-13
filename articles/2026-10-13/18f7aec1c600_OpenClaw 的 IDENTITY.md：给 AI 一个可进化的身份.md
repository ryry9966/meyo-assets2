---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41391
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

OpenClaw 每次会话启动时，会从 workspace 加载一组 Markdown 文件拼装系统上下文：SOUL.md 管性格与价值观，USER.md 记录用户偏好，IDENTITY.md 负责最基础的一层——名字、形象（creature）、气质（vibe）、emoji、头像。多数人初始化时随手填一遍，之后再没打开过。但这个文件真正的价值在于：它只是一个普通文件，可以被 diff、被提交、被回滚。

## 问题

实际用下来，身份问题集中在三类：

1. **漂移**：SOUL.md 改了几轮，memory 里又积累了不少自称，时间一长 agent 的自我描述和 IDENTITY.md 对不上，同一实例在不同会话里性格不一致。
2. **无历史**：身份改了什么、为什么改、何时改的，全靠回忆；想回退时没有基线。
3. **多实例撞车**：克隆 workspace 起第二个 agent，两个实例同名同 emoji，群聊里互相认错，用户也无法从消息前缀区分来源。

## 做法

把 IDENTITY.md 当配置代码来管：

**1. 建最小基线。** 填一份克制的版本：

```markdown
- Name: 小钳
- Creature: 机械寄居蟹
- Vibe: 冷静、话少、先给结论
- Emoji: 🦀
```

控制在 15 行以内。它每个会话都会注入上下文，写长文纯属烧 token。

**2. 纳入版本控制。** workspace 下单独建 git 仓库，`.gitignore` 掉 `memory/` 和媒体目录，只跟踪根目录的 `*.md`，首次提交打 tag（如 `identity-v1`）。

**3. 写清演进规则。** 在 AGENTS.md 里明确：修改 IDENTITY.md 必须先经用户确认，一次只改一个字段。OpenClaw 的 agent 默认对 workspace 有写权限，不写规则它真的会自我发挥。

**4. 建立"提案—评审—提交"循环。** 跑一段时间后让 agent 起草修订版，你 `git diff` 审变化：合理就 commit，message 写明原因（如 `identity: vibe 调整，解决群聊刷屏`）；不合理就丢弃。所谓"进化"，实际形态就是小步、可评审、可回滚。

**5. 多实例强制差异化。** 每个 workspace 至少保证 name 和 emoji 不同，群聊里的自报家门和消息前缀才不会撞。

## 踩坑点

- **易变信息别放这里**："用户在做 X 项目"属于 USER.md 或 memory，写进身份文件会迫使你频繁改身份。
- **emoji 不是装饰**：它参与群聊消息前缀的渲染，部分渠道显示效果差，换之前先在目标渠道实测。
- **改名有过渡期**：长期记忆里存的全是旧名字，改完头几天可能自称混乱，可在 memory 里补一条更名记录。
- **别与 SOUL.md 职责重叠**：IDENTITY 管"是什么"，SOUL 管"怎么做事"，同一条描述写两处，改的时候必漏。

## 可复用建议

- 一次只改一个字段，观察两三天日志里的语气表现，再决定保留或回滚。
- 用 tag 管理人格版本（`identity-v2-reviewer`），试运行不满意直接 checkout 回退。
- 每季度做一次 identity review，让 agent 对着 git log 复盘自己的变化，产出通常比预期有意思。

## 总结

IDENTITY.md 的机制很朴素——就是启动时注入的一个 Markdown 文件。但配上版本控制和"先提案再提交"的规则，它就从一次性的填空题变成了可持续演进的配置：有基线、有历史、能回滚。维护成本很低，换来的是长线一致性，值得一开始就做对。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/f03ca093e6e9f5f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/cb64c246e25ce192.png)

