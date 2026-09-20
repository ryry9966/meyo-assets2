---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 38286
source: 综合讨论
publishedAt: 2026-09-21
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

OpenClaw 的 agent 长期驻留在你的机器上，跨会话、跨任务工作。但默认状态下它只是个"礼貌的通用助手"——每次会话的人格都由系统提示临时拼装。OpenClaw 的做法是把身份外置到工作区的一个 Markdown 文件：`IDENTITY.md`（默认在 `~/.openclaw/workspace/` 下），随引导文件在新会话启动时一起读入上下文。它通常只有几行：名字、形象设定、气质、一个 emoji、头像描述。

## 问题

没有这个文件时，常见的三种尴尬：

1. 人格写在启动命令或配置字符串里，改一次要动代码，没有 diff、没有历史、没法 review；
2. 多 agent 共用一台机器时互相串味，A 的语气出现在 B 的回复里；
3. 你希望它"随着相处慢慢变"，但通用助手的提示是静态的，没有留出进化的入口。

## 做法

1. **定位工作区**。默认 `~/.openclaw/workspace/`，多 agent 场景下每个 agent 有独立 workspace，先确认你改的是哪一个。
2. **写最小模板**：

```markdown
- **Name:** Atlas
- **Creature:** 沉稳的工程向助手
- **Vibe:** 简洁、直接、先给结论
- **Emoji:** 🛠️
- **Avatar:** 拿着扳手的几何小机器人
```

20 行以内足够。它是身份声明，不是作文。

3. **和相邻文件分清职责**：`IDENTITY.md` 管"我是谁"，`SOUL.md` 管"我怎么做事"，`USER.md` 管"我为谁服务"。混写之后 diff 无法 review。
4. **验证生效**：新开会话，问一句"你是谁、你的风格是什么"，对照文件确认加载的是当前版本。
5. **进化流程**：把它当代码管理——workspace 进 git，每次修改单独 commit；想让 agent 参与时，让它基于最近的摩擦点起草修改，你 review diff 后再合入。

## 踩坑点

- **写成小作文**。上下文按 token 结算，800 字的自我描述每场会话都要付一遍，模型转述时还不稳定。字段化、短句。
- **把行为规则塞进身份文件**。"不要用感叹号"这类约束属于 `SOUL.md` / `AGENTS.md`，塞错位置后你永远说不清哪次行为变化是谁引起的。
- **放任 agent 自我改写**。模型天然倾向给自己加形容词，无人审核的自我进化很快漂移成"每个会话都是新人"。必须由人来 commit。
- **改了没生效就到处找 bug**。`IDENTITY.md` 在新会话引导时读入，改完当前会话不一定刷新，先重开会话再排查。
- **多 agent 共用一个 workspace**。身份互相污染，git 历史混在一起，也没法单独回滚某个 agent。

## 可复用建议

- 把 `IDENTITY.md` 当配置代码：字段化、短、进版本库、改动走 review。
- 进化节奏对齐"观察到的失配"而不是日历：只有确实觉得某次回复不对劲时才改，而不是每月定时折腾。
- 单一事实来源：emoji、头像这类字段如果全局配置里也有，只留一处，否则迟早不一致。
- 模板本身值得沉淀：团队多个 agent 可共享一套 IDENTITY 模板，只在差异化字段上分叉，降低维护成本。

## 总结

`IDENTITY.md` 的价值不在"给 AI 起个名字"这种拟人化趣味，而在工程上把"人格"从散落的提示字符串收敛成一个可版本化、可 review、可回滚的文件。改动有 diff，历史有 commit，进化有门槛——这才是"可进化的身份"的真正含义：不是让 AI 随便变，而是让变化可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/fd5171595f9357f3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/4dae150125cf592f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/8bfb34ba42091555.png)

