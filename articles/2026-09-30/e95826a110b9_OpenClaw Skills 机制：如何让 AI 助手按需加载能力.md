---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 39697
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 的 agent 长期挂在网关上，对接 Telegram、WhatsApp 等渠道。早期的做法很直接：把所有工具用法、自动化约定全塞进 system prompt。三五个能力时还行，等你要加网页抓取、视频处理、定时任务、多渠道回复规范，prompt 很快膨胀到上万 token——费用上升是小事，真正的问题是模型注意力被稀释，该触发的指令反而漏触发。

Skills 是 OpenClaw 对这个问题的官方解法：把"能力"从 prompt 里拆出来，做成按需加载的文档模块。

## 问题拆解

按需加载要解决三件事：

1. 模型怎么知道有哪些能力？——需要一份极简的能力索引；
2. 触发后怎么拿到完整说明？——文档正文必须能被工具读取；
3. 哪些 agent 该开哪些能力？——需要配置层的开关。

## 做法

一个 Skill 就是一个目录，核心是 `SKILL.md`：

```text
~/.openclaw/skills/weather-report/
└── SKILL.md
```

frontmatter 里写 `name` 和 `description`，正文写操作步骤。关键机制是**渐进披露**：

- 启动时，网关只把所有 skill 的 name + description 注入 system prompt，通常每个只占一两行；
- 对话中命中某个能力时，agent 再用 read 工具加载对应的 SKILL.md 全文，按其中步骤执行；
- 用 `openclaw skills` 可以查看、启用/禁用；配置里 `skills.allow` / `skills.blocks` 按 agent 划分子集，多 agent 各挂各的能力包。

路径约定也简单：bundled skills 随安装自带，user skills 放 `~/.openclaw/skills`，workspace skills 跟着项目走，方便纳入版本管理。

## 踩坑点

- **description 写成功能简介而不是触发条件**。模型是拿它做匹配的，"用于天气查询"不如"当用户问明天穿什么、周末出行天气、降雨概率时使用"。
- **frontmatter 声明了 `requires.bins`（如 ffmpeg）但机器上没装**，skill 会被静默禁用。改动环境后先跑 `openclaw skills scan` 确认状态，别等到任务失败才排查。
- **拆得太细**。"读 CSV"和"读 Excel"没必要拆两个 skill，能力越多，description 之间的触发歧义反而越严重。
- **正文引用 skill 目录外的绝对路径**，换机器就断。脚本、模板尽量和 SKILL.md 同目录打包。

## 可复用建议

- 把 description 当路由规则写：写清什么样的输入该触发它，而不是它是什么。
- 先在主 workspace 用一个最小 skill 跑通"索引注入 → 触发 → 全文加载"的链路，再批量迁移存量 prompt。
- 每季度 review 一次已启用列表，长期没被触发的直接下线，别让能力索引重新变胖。

## 总结

Skills 的本质，是把 system prompt 从"能力说明书"降级成"目录页"。对长期运行的 OpenClaw agent 来说，这不只是省 token 的技巧，而是让能力可以持续增长的结构约束：能力可以一直加，上下文的体积不变。

---

