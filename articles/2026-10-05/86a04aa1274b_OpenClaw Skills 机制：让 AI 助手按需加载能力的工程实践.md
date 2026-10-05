---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 40580
source: 综合讨论
publishedAt: 2026-10-05
---

# 背景

OpenClaw 的 agent 本体其实很薄，能力靠外挂：MCP server 提供工具，Skills 提供操作知识。一个 Skill 本质上就是一个带 `SKILL.md` 的文件夹，frontmatter 里的 `name` 和 `description` 常驻上下文，正文只有在模型判断相关时才注入——这就是渐进式披露（progressive disclosure）的设计。

# 问题

我们的 workspace 曾经把所有流程说明和工具用法全塞进 system prompt：30 多个工具加上一堆内部 SOP，每轮固定 token 开销上万。更糟的是注意力被稀释，agent 在该调内部导出脚本的时候，经常选了通用方案绕远路。上下文不是免费的，也不是越大越好。

# 做法

在 `~/.openclaw/skills/`（或 workspace 的 skills 目录）下，每个能力一个文件夹：

```
skills/
└── weekly-report-export/
    ├── SKILL.md
    └── scripts/export.py
```

`SKILL.md` 长这样：

```markdown
---
name: weekly-report-export
description: 当用户要求导出周报、月度数据或统计报表时使用。
  覆盖 CSV 与 Markdown 两种格式。
---

# 周报导出

## 步骤
1. 运行 scripts/export.py --range <start>..<end>
2. 校验行数非零，否则向上游反馈而不是输出空表
3. 默认输出到 workspace/exports/，文件名带日期

## 边界
- 跨季度数据先确认范围，避免全量拉取
- 用户没指定格式时先问，不要默认 CSV
```

改完重启 gateway（或触发重新扫描）即可生效。关键机制：**description 是索引，正文是按需分页**。模型每轮只看几十个字的索引，命中了才把几百字的正文拉进上下文。

# 踩坑点

- **description 写成名词短语**（"报表工具"）而不是触发条件（"当用户要导出周报时"），结果永远不会被命中。
- **正文太长**：一旦触发就吃几千 token，比不加载还贵。控制在一屏内，长内容拆成脚本或附属文件。
- **多个 skill 描述重叠**，模型随机挑一个。用更具体的动词和场景区分，别写两个"数据处理助手"。
- **写死本机绝对路径**，换台机器就挂。用相对 workspace 的路径，或让脚本自己探测。
- **改了 SKILL.md 忘了 reload**，以为生效了，跑的还是旧版。排查时先确认加载时间。

# 可复用建议

1. **一个 skill 只干一件事**。判断标准：description 能不能用一句"当……时"说清楚。
2. **description 里放用户真实会说出的关键词**（周报、回滚、部署），这是路由命中率的主要来源。写 skill 的功夫，八成应该花在 description 上。
3. **可执行部分抽成脚本**，正文只写"何时调、怎么调、怎么验证结果"。
4. **skills 目录进 git**，改 description 当改代码看待：跑一组固定的 probe prompt（"帮我导出上周数据"），验证命中是否如预期。
5. **定期清理**：一个月没被命中的 skill，大概率是描述有问题，或者真的没人需要。

# 总结

Skills 的价值不在于"给助手更多能力"，而在于让能力**可索引**。把上下文当预算花：description 是常驻的索引行，正文是命中后才加载的一页。索引写得准，助手才会在对的时候拿出对的本事——这比堆工具数量重要得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/a7cf9a836585a5bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/7a87f71d284b5b55.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/f0930e70e9859bfe.png)

