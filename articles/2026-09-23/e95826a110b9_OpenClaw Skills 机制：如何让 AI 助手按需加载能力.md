---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38614
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

在 OpenClaw 里扩展 agent 能力，常见路径有两条：接 MCP 工具，或者往 system prompt 里堆操作说明。工具和说明多了以后，每次会话的上下文里都躺着一大堆"可能用得上"的文档——大部分时间用不到，token 却一直占着。Skills 机制就是针对这个问题设计的：把能力打包成独立目录，运行时只注入一行元数据，模型判断需要时才把完整指令加载进来。

## 问题

具体痛点有三个：

1. system prompt 膨胀，几 KB 的操作手册常驻上下文，成本上升，模型注意力被稀释；
2. 工具与说明列表过长时，模型选错路径的概率明显上升；
3. 能力没有版本和边界，改一处说明要动整个 prompt 文件，回滚困难。

## 做法

Skills 的核心是"渐进式加载"三段式：

1. **元数据常驻**：每个 skill 目录下 `SKILL.md` 的 frontmatter（`name` + `description`）在会话启动时注入。这一层要小，几十个 skill 也就千把 token。
2. **按需展开**：模型根据 description 判断当前任务命中某个 skill，才去读 `SKILL.md` 正文，拿到详细的操作步骤、约束和输出格式。
3. **资源再下沉**：正文可引用同目录的 `references/` 长文档和 `scripts/` 脚本，只在真正执行时读取。

目录结构：

```
skills/
└── pdf-report/
    ├── SKILL.md        # frontmatter + 操作指令
    ├── references/     # 长文档、字段说明
    └── scripts/        # 可执行脚本
```

frontmatter 示例：

```yaml
---
name: pdf-report
description: 从结构化数据生成 PDF 周报。当用户要求导出报表、周报、PDF 文件时使用。
---
```

验证方式很直接：新开会话，问一句"帮我导出这周的周报"，看日志里是否出现对该 skill 的加载记录；再问一个无关问题，确认没有误触发。两条都过，才算接入成功。

## 踩坑点

- **description 是唯一触发依据**。写成"处理 PDF 相关任务"这种泛描述，要么不触发要么乱触发。正确写法是任务动词 + 产物 + 触发场景，一句话说清。
- skill 数量上去后，元数据本身也吃 token。超过 30~50 个就该合并同类项，或拆分成多个 agent。
- `scripts/` 里的脚本要在干净环境先跑一遍，路径用相对 skill 根目录的写法，别假设工作目录。
- skill 之间不要互相引用正文，依赖关系在主 SKILL.md 里显式说明，否则加载链路不可预测。

## 可复用建议

- 一个 skill 只做一件事，粒度以"一次完整任务"为准，不要做成大杂烩；
- description 用"当用户要求 X 时使用"的句式，触发词前置；
- 超过 300 行的说明拆到 `references/`，正文只留流程和硬约束；
- 脚本先本地执行、验证输出格式，再交给 skill 引用；
- 把 skill 当代码管理：进 git、写变更记录、review 后合并。

## 总结

Skills 机制的本质，是把"能力说明"从常驻上下文改成按需加载，用一层便宜的元数据换取上下文的干净。它不替代 MCP——工具负责"能做什么"，skill 负责"怎么做才对"，两者是分工而非竞争。实践上建议先从最常驻、最长的 prompt 说明开始迁移，收益最直接，也最容易验证效果。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/c65119d30146a94b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d10fde8253b1cf5f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/8cb8e3933e687ccd.png)

