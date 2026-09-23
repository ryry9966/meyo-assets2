---
title: OpenClaw Skills 机制实践：让 Agent 按需加载能力，而不是堆满上下文
feedId: 38605
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

用 Agent 做自动化一段时间后，几乎都会撞上同一个问题：能力越多，system prompt 越胖。把所有 SOP、命令模板、API 约定全写进提示词，静态内容占据大量上下文，模型对当前任务反而“看不清”，token 成本还一路上涨。

OpenClaw 的 Skills 采用两段式加载：常驻的只有每个 skill 的 `name` 和 `description`（几十个 token），完整正文在模型判断“当前任务相关”时才注入。本质上，这是把提示词工程从“预加载”改成了“懒加载”。

## 问题

我最初的用法很典型：二十多个流程文档全堆在 workspace 里，指望 agent 自己翻。结果要么它根本不看，要么翻半天拿错文件。迁移到 skills 之后又出现新问题——有的 skill 从不触发，有的却什么请求都抢。

## 做法

1. 建目录：`~/.openclaw/skills/<skill-name>/SKILL.md`，项目级的放 workspace 的 skills 目录。
2. frontmatter 至少写清 `name` 和 `description`：

```yaml
---
name: weekly-report
description: 当用户要求汇总本周提交、生成周报或周度变更摘要时使用。不要用于单次 commit 说明。
---
```

3. 正文只写三块：触发后的执行步骤、可直接复用的命令或模板、明确的边界（什么情况不要用）。
4. 有外部依赖时，用 `metadata.requires` 声明所需的 CLI 或环境变量，让 agent 在缺依赖的环境里直接跳过，而不是空转重试。
5. 新开会话，用 `/skills` 确认已加载，再用一个真实任务验证触发路径。

## 踩坑点

- **description 写成“这是什么”，而不是“什么时候用”。** 模型选 skill 靠匹配当前意图，名词式介绍几乎不会命中。
- **description 太宽泛。** 一个自称“处理所有文本转换”的 skill，会抢走本该路由给其他 skill 的请求。
- **正文塞满长示例。** 示例放外部文件，SKILL.md 只留入口和路径，否则命中一次就烧掉一大段上下文。
- **与内置 skill 重名。** 覆盖行为很隐蔽，调试半天才发现名字撞了。
- **依赖的 CLI 未安装。** agent 会反复尝试再失败，务必做 requires 门控。

## 可复用建议

- description 公式：**触发场景 + 具体动作 + 排除项**，一句话说清，像写函数签名的 docstring。
- 一个 skill 只做一件事，宁可拆小，也不要做“万能流程”。
- 每写一个 skill，用三个问法测触发：正向问法、同义变体、近似但不该触发的请求。第三个最容易被忽略，却是防止误路由的关键。

## 总结

Skills 的价值不在“多”，而在“准”。把静态知识从上下文里挪出去，只保留精准的入口描述，agent 的响应质量和 token 成本都会有肉眼可见的改善。建议从把你重复粘贴次数最多的那篇 SOP 改造成第一个 skill 开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/86e1a867c12575c4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/1766bd46ef2d61d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/c8b36d13fa3a9837.png)

