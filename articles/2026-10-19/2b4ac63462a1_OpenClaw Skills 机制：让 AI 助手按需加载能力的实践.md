---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的实践
feedId: 42029
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

用 OpenClaw 做长期运行的 agent，瓶颈往往不在模型本身，而在上下文管理。接了几个 MCP server、挂了一堆工具之后，system prompt 越来越胖，单次请求 token 成本上去了，模型面对几十条工具说明时选错、漏选的概率也明显上升。Skills 机制解决的就是这个问题：**常驻上下文的只有一份轻量索引，具体能力按需加载**，也就是所谓的渐进式披露（progressive disclosure）。

## 问题

传统做法是把所有能力说明全量塞进 prompt，带来三个实际痛点：

- 上下文膨胀，成本和延迟线性上涨；
- 注意力被稀释，工具越多路由越不准；
- 每台机器环境不同（有没有 ffmpeg、有没有配 key），但能力清单是同一份。

## Skills 是什么

一个 Skill 就是一个目录，核心是 `SKILL.md`：frontmatter 写元信息，正文写操作指引，旁边可以放脚本和参考文档。会话启动时只注入 name 和 description（每个几十 token）；模型判断当前任务命中某个 skill，才会去读完整正文。

```yaml
---
name: video-clip
description: 用 ffmpeg 裁剪视频片段、压缩与转码。当用户要求剪辑、截取、转码视频时使用。
metadata:
  requires:
    bin: [ffmpeg]
---
```

`requires` 里声明的二进制、env、config 不满足时，这个 skill 根本不会进索引，环境差异被天然消化掉了。

## 做法

1. 在 workspace 的 `skills/` 目录下建文件夹，写 `SKILL.md`；
2. description 写清"做什么 + 什么时候用"，它是路由的唯一依据；
3. 长流程、参数表拆到 `references/` 里，正文只留入口和核心步骤；
4. 用 `requires` 做门控，别让本机跑不起来的能力出现在索引里；
5. 用 `openclaw skills list` 确认加载状态，配合日志看实际命中了哪个。

## 踩坑点

- **description 写给人看而不是写给模型看**。"强大的视频处理工具"这种描述基本不会触发；写明任务类型和触发词才有效。
- **什么都塞进正文**。一旦命中就吃掉几千 token，等于把膨胀问题推迟了一步。超过 200 行就该拆参考文件。
- **同名覆盖**。用户级 skill 会盖掉内置的，行为悄悄变化，排查半天才发现是目录里有个旧版本。
- **脚本细节**。没加执行权限、相对路径写错，是最常见的两类"skill 命中了但跑不通"。
- **数量失控**。索引本身也有体积，隔段时间删掉三个月没命中的。

## 可复用建议

- 把 description 当广告写：一行说清任务类型，附上用户可能会说的触发词；
- 重复性、确定性操作写成脚本放进 skill 目录，比一堆 prose 指引稳定得多；
- skills 进 git，像代码一样 review 和回滚；
- 团队内沉淀通用 skill，机器相关的东西交给 `requires` 门控。

## 总结

Skills 本质上是一道上下文经济学题：用极小的索引成本，换一个可扩展的能力面。机制本身很简单，真正决定效果的是 description 的路由精度和文件的拆分纪律。把"什么时候该被想起"写清楚，比堆功能重要得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/8c8d4804228a339f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/8d1916281aa6a798.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/5c38c40cc7711636.png)

