---
title: IDENTITY.md 实践：把 Agent 的身份当成代码来养
feedId: 39413
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景：一份容易被忽略的文件

OpenClaw 的 workspace（默认 `~/.openclaw/workspace`）里有几个约定文件：`IDENTITY.md`、`SOUL.md`、`USER.md`、`AGENTS.md`。其中 IDENTITY.md 承担"我是谁"这个最基础的问题——名字、形象（creature）、emoji 签名等字段，会在每轮会话构建 system prompt 时被注入。它不是装饰，而是 Agent 行为的锚点之一。

初装时引导流程会让你填一份，多数人随手写完就再没打开过，直到出问题。

## 问题：没有管理的身份会漂移

实际用下来常见三类情况：

1. **多实例混淆**：跑了两三个 Agent（一个写代码、一个管日程），消息签名、日志、通知里分不清谁在说话。
2. **人格漂移**：长期使用后语气忽冷忽热，根源往往是身份信息只存在于历史对话里，模型每次"猜"出来的自我认知不一致。
3. **不可追溯**：身份靠对话驯化或写死在配置里，改坏了没有任何回滚手段。

## 做法：把身份文件工程化

我的做法分四步：

**1. 明确字段分工。** IDENTITY.md 只管身份事实：`name`（用户怎么称呼它）、`creature`（形象隐喻，影响自述方式）、`emoji`（消息签名）。性格倾向交给 SOUL.md，用户画像交给 USER.md，任务规范交给 AGENTS.md，不要混。

**2. 加自定义字段。** 除默认字段外，我加了 `language`（默认回复语言）、`boundary`（明确不碰的事，如"不主动删文件"）。这些同样每轮生效，比在提示词里散着写更稳定。

**3. 用 git 管 workspace。** `git init` + 每次修改 commit。身份调整本质是需求变更，有 diff 才有讨论依据，也才能回滚。

**4. 信号驱动地 review。** 触发条件不是固定周期，而是信号：语气漂了、签名乱了、自我介绍和设定不符，就对照最近的会话输出修一轮。我大约两三周修一次，每次改动不超过五行——小步提交，效果才可归因。

一份精简示例：

```markdown
# IDENTITY.md
- Name: 阿章
- Creature: 戴圆眼镜的章鱼
- Emoji: 🐙
- Language: 中文优先
- Boundary: 不执行删除类命令，除非二次确认
```

## 踩坑点

- **把它当记忆库用**。项目细节、任务上下文塞进来会无谓撑大 system prompt，且这些内容更新频繁，应交给 memory 机制。
- **人设写得长且戏剧化**。几百字的人设文会让输出浮夸，相互冲突的形容词越多越不稳定。控制在十行以内。
- **改完不生效就怀疑配置**。IDENTITY.md 在新会话构建时注入，进行中的旧会话不会自动刷新。改完开新会话验证。
- **多实例共用 workspace**。每个 Agent 要有独立目录和独立的 IDENTITY.md，或用不同 profile 隔离。
- **忘了 git**。手滑改坏一段，语气直接崩，没有历史只能凭记忆重写。

## 可复用建议

- 把 IDENTITY.md 当小型 SRS 对待：短、可测（能否稳定自报身份、遵守边界）、可版本化。
- 批量起实例时用模板 + 变量渲染，`name`/`emoji`/`creature` 保证唯一，`boundary` 和 `language` 按角色套用。
- 团队场景下身份变更走 PR review，让"AI 是谁"成为显式决策，而不是某个人的随手配置。
- 在四个约定文件开头各写一行注释标明职责边界，防止后来人（包括未来的自己）往里乱塞。

## 总结

IDENTITY.md 的价值不在于让 Agent 更"像个人"，而在于把身份从散落在对话里的隐性状态，变成一个显式、可 diff、可回滚的文件。它很短，但每轮会话都在生效。花半小时把它工程化，省掉的是无数次"这货今天怎么又变了"的排查时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/581a3309bd25fbe0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/08ded3e3d9dd1506.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/3c045ca794eb582d.png)

