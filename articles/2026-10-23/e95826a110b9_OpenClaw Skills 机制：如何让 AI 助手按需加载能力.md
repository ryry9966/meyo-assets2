---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42327
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

跑 Agent 时间稍长的人都会撞到同一个瓶颈：能力越多，系统提示越肥。MCP 工具描述、插件说明、操作手册全堆在常驻上下文里，模型注意力被稀释，token 成本也难看。OpenClaw 的 Skills 机制就是针对这个问题：把"知识"从常驻上下文里挪出去，改成按需注入。

## 问题

传统做法无非两条路：把操作说明直接写进系统提示（稳定但昂贵），或者全靠工具描述让模型自己猜（便宜但容易翻车）。技能数量到了十几个，前者的常驻开销线性增长；后者则经常出现不该调的时候乱调、该用的时候想不起来。本质矛盾在于：**索引和正文没有分离**。

## Skills 机制怎么工作

OpenClaw Skills 的核心设计就是把这两层拆开。每个技能是一个文件夹，入口是 `SKILL.md`：

- frontmatter 里的 `name` 和 `description` 构成**索引**，以单行摘要的形式常驻系统提示；
- Markdown 正文是**正文**，只有当用户请求匹配描述、或 agent 自行判断需要时才注入；
- frontmatter 的 metadata 可声明 `os`、`requires`（bins / env / config），运行时据此判定技能是否 eligible——缺依赖的技能根本不进索引，避免模型"看得到、用不了"。

## 做法

写一个自定义技能大致四步：

1. 在技能目录（如 `~/.openclaw/skills/<name>/`）下建 `SKILL.md`：

```markdown
---
name: log-triage
description: 当用户要求排查服务日志、定位报错原因时使用。
  覆盖 grep/journalctl 分析流程，不覆盖代码修改。
metadata:
  openclaw:
    requires:
      bins:
        - jq
---
# 步骤
1. 先确认时间窗口，再按关键字 grep……
2. 用 jq 结构化提取字段……
```

2. `description` 按"什么场景用 + 覆盖什么 + 不覆盖什么"来写，它就是触发路由。
3. 正文写成可执行手册：步骤、命令、边界条件，末尾放一段自检命令。
4. `openclaw skills list` 确认加载状态，`openclaw skills check` 看缺哪些依赖；补齐后重开会话验证。

## 踩坑点

- **description 写成名词短语**（"日志分析工具"）基本不触发，要写成带场景的句子；
- **触发词太宽**，正文频繁注入，token 反而更费；
- **忘声明 requires**，agent 拿着技能却跑不了命令，反复试错浪费轮次——先 `skills check` 再进会话；
- **workspace 与 managed 技能同名**会覆盖，排查半天发现加载的不是自己那份；
- 改完 `SKILL.md` 不重开会话，误以为没生效；
- 正文里写死本机绝对路径，换台机器就废，尽量用环境变量或相对约定。

## 可复用建议

- 把 description 当**检索 query** 写，不是当简介写；
- 一个技能只管一个领域，宁可拆小；
- 正文末尾固定一段"完成标准"，让 agent 能自检闭环；
- 技能入库 git，description 的每次改动都值得 diff——它就是你的路由表；
- 心里有个账：常驻成本 ≈ 技能数 × description 长度，几十个技能以内索引仍然很轻，正文才是大头。

## 总结

Skills 本质是给 agent 的延迟加载文档：索引常驻、正文按需、依赖前置校验。用好的关键不在写了多少内容，而在 description 的精度和依赖声明的完整性。把这三件事做扎实，Agent 的上下文才真正"长得起"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/6034fa21afc6296b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/386196d8b9437e74.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/369c5a45f99899d2.png)

