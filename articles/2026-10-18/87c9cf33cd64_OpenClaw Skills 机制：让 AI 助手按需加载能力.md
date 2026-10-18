---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 41891
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 里 Agent 的行为由三样东西决定：system prompt、工具、以及 Skills。每个 Skill 是 skills 目录下的一个文件夹，核心是一份 SKILL.md，写清这个能力是干什么的、什么时候用、具体怎么做。它解决的核心问题只有一个：**上下文预算**。

## 问题

如果不做按需加载，常规做法是把所有能力的说明书一次性塞进 system prompt。技能一多，光说明就占掉大几万 token，真正任务的空间被挤压，而且大部分技能和本次对话毫无关系，纯属浪费。更糟的是，有些技能依赖的 CLI 或 API key 根本没装，注入了也执行不了，模型还会照着说明书反复试错。

## 做法

OpenClaw 的 Skills 走的是渐进披露（progressive disclosure）路线：

1. **只注入元信息**。system prompt 里每个技能只占一两行：name + description。模型根据这两行判断本次要不要用。
2. **按需读全文**。模型决定使用某技能后，用 read 工具拉取 SKILL.md 正文，才知道具体步骤；没触发的技能正文永远不进上下文。
3. **依赖门控**。frontmatter 里声明 `requires.bins`（二进制依赖）、`requires.env`（环境变量）、`requires.config`（配置项），不满足条件的技能直接从候选中剔除，不污染 prompt。

一个最小 SKILL.md 长这样：

```markdown
---
name: pdf-extract
description: 从 PDF 提取文本和表格，用于解析发票、报告类文件
metadata:
  openclaw:
    requires:
      bins: [pdftotext]
---
## 步骤
1. 先用 pdftotext -layout 保留版式
2. 表格识别失败时用 -t 模式重试
```

4. **查状态**。`openclaw skills list` 能看到每个技能是否满足依赖、为什么被排除；排查"模型为什么不用某个技能"，先看这里。个别技能也可以在配置里 allow/exclude 掉。
5. **覆盖优先级**。workspace 技能（`~/.openclaw/workspace/skills/`）优先级高于内置技能，同名直接覆盖，方便针对自己的环境改写默认行为。

## 踩坑点

- **description 是唯一的路由依据**。写得太泛（"处理文件"）触发不了，只写名字更不行。建议"动词 + 场景"：什么时候用、处理什么对象。
- **requires 漏配**。正文里用了 ffmpeg 却没声明 bins，模型会跑到报错才发现缺依赖，白费一整轮。
- **正文里的环境差异**。从别人处搬来的技能常带硬编码路径或 macOS 专属命令，加载后反而把模型带偏，装完先通读一遍。
- **别拿 Skills 当 MCP 用**。Skills 是给模型看的操作手册，不提供执行层；要真实调用外部服务还是靠 MCP/工具，Skill 只负责写清"怎么调用得漂亮"。
- **技能互相冲突**。两个技能对同一工具给出矛盾指令时，模型行为会抖动，重叠职责的技能留一个就够。

## 可复用建议

- 颗粒度按"一个可交付结果"拆，不按脚本数量拆。
- description 定期回顾：哪些任务模型没触发对应技能，多半是描述没对上用户实际的说法，改描述比改正文见效快。
- 定期 `openclaw skills list` 审计，长期不触发的禁用，候选列表越短，路由越准。
- 通用逻辑沉淀成模板技能，项目差异放进参数文件而不是复制多份。

## 总结

Skills 机制本质上是上下文的懒加载：元信息常驻、正文按需、依赖不齐就不进场。把预算还给任务本身，模型的指令遵循率反而更高。想让 Agent 靠谱，先从写好每一行 description 开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/abdf79ccdd4b3788.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/0164012b8bb451bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/8c8ea86e98c8eea2.png)

