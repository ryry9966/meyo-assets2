---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 41970
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

OpenClaw 的工作区里有一组 Markdown 文件承担着"人格层"：`AGENTS.md` 管行为规则，`SOUL.md` 管价值观与底色，`MEMORY.md` 管长期记忆，而 `IDENTITY.md` 是最薄的一层——名字、形象（creature/vibe）、emoji、头像，外加几行交互风格说明。很多人初始化完就再没打开过它，于是 agent 永远是出厂默认的那个人。

## 问题

实际跑一段时间后通常会撞上这几件事：

- 每次新会话，agent 回到默认人格：语气随机、称呼漂移、开场白时冷时热；
- 多渠道（Telegram / Discord / WebChat）表现不一致，像三个人在值班；
- 你在聊天里纠正过它一百次，但上下文一滚动，纠正就蒸发——口头反馈不持久；
- 团队共用 agent 时，"它是谁"只存在于某个人的调教历史里，换人就断档。

根因只有一个：**身份没有被持久化、版本化、评审化。**

## 做法

1. **定位工作区**：默认在 `~/.openclaw/workspace`（以你的安装配置为准），没有 `IDENTITY.md` 就新建。
2. **先写最小集**：`name / creature / emoji / avatar` 四项，再加 3–5 行交互风格：语气、默认语言（比如中文优先）、面对不确定问题时的态度。不要贪多。
3. **分层放置**：身份放 `IDENTITY.md`，长期规则放 `SOUL.md` / `AGENTS.md`，用户事实放 `USER.md`，事件流水放 `MEMORY.md`。最常见的错误是把记忆往身份里塞。
4. **版本化**：整个 workspace 纳入 git，身份改动单独 commit，如 `identity: 默认称呼改为"你"，去掉卖萌开场`。回滚从此是一条命令的事。
5. **建立进化回路**：agent 表现不符合预期时，当天改文件，而不是只在对话里说。进阶玩法是约定 agent 每周（可用 heartbeat/定时任务触发）汇总"自我观察"，以 diff 形式提交 `IDENTITY.md` 修改建议，由你 approve 后合入——**agent 只提案，人在环上拍板**。
6. **变更留痕**：文件顶部维护两三行 changelog。改名、改语气都要记，避免长期用户突然觉得"它变了"。

## 踩坑点

- **写太长**。`IDENTITY.md` 每个会话都会注入上下文，控制在 30 行以内；超了既费 token，又稀释真正的关键指令。
- **与 `AGENTS.md`/`SOUL.md` 冲突**。同一件事两处说法不一，行为就会抖动。改身份时顺手全局搜一遍相关关键词。
- **无审批的自改权限**。agent 能写自己的身份文件 + 又会读网页内容，等于给提示注入开了后门。自写权限务必关在审批后面。
- **多 agent 共用 workspace** 会互相污染身份，建议一人一目录，或至少文件级隔离。
- **别假设热加载**。新内容通常在下一个会话才生效，排障第一步永远是确认"它实际读的是哪份文件"。

## 可复用建议

- 把 `IDENTITY.md` 当 **config** 而不是笔记：短、稳定、可 diff、可回滚。
- 稳定的是"我是谁"，易变的是"我经历了什么"——身份与记忆严格分离。
- 纠正流程固化为三步：对话纠偏 → 当天落文件 → git commit。
- 多实例场景先做模板再 fork，不要手抄，否则三个月后没人记得差异在哪。

## 总结

`IDENTITY.md` 本身只是几行 Markdown，真正的价值在于你围绕它建立的机制：持久化、版本化、带审批的自我修订。当纠正从聊天记录变成 commit 记录，agent 的性格才第一次拥有可回溯的历史。所谓"可进化"，不是让它随便改自己，而是给它一个有人把关的迭代节奏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/796615a680bb72d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/a9906644b30f265f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/f332cfa77193d61d.png)

