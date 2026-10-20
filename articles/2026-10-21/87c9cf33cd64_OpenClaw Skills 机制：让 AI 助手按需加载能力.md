---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 42130
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 里 agent 的能力来源主要有两类：内置工具和外部 MCP。传统做法是启动时把所有工具的 schema 和说明全部塞进系统提示，模型“全知”，但代价明显：上下文膨胀、无关工具干扰判断、团队沉淀的经验也没有一个好的承载形式。Skills 机制就是针对这个问题设计的——把能力组织成一个个目录，运行时按需加载。

## 问题

我们团队最初把几十个 MCP server 全挂在一个常驻 agent 上，system prompt 轻松超过 8k tokens，模型经常选错工具，调用成本也高。真正想要的形态是：agent 平时很“轻”，只有命中特定任务时，才把对应能力的完整说明、脚本和参考资料加载进来。也就是渐进式披露（progressive disclosure）：元数据常驻，正文按需，资源文件再按需。

## 做法与步骤

**1. 建目录。** 一个 Skill 就是一个文件夹，核心是 `SKILL.md`：

```
~/.openclaw/skills/release-notes/
├── SKILL.md
├── template.md
└── gen_changelog.sh
```

**2. 写元数据。** frontmatter 只有 `name` 和 `description` 两个字段，启动时随系统提示注入——这是唯一常驻上下文的部分。description 不要写成广告词，要写成触发条件：

```yaml
---
name: release-notes
description: 当需要根据 git log 生成发版说明或 changelog 时使用。输入版本号或 tag 区间，输出 Markdown。
---
```

**3. 写正文。** 正文默认不加载，只有模型判断当前任务匹配 description 时才读入。正文里写步骤、约束和注意事项，用相对路径引用同目录的脚本与模板。

**4. 确定性逻辑下沉到脚本。** 模型负责编排和判断，重复性操作（解析 git log、格式化日期等）写进可执行脚本，正文只说明何时调用、参数是什么。

**5. 验证触发。** 用 `openclaw skills list` 确认元数据被识别，再用一句自然语言问 agent，观察它是否主动调用。不触发，先改 description 再改别的。

## 踩坑点

- **description 写成功能介绍。** “这是一个强大的发版工具”，模型无法据此判断何时使用。要写成 if 条件。
- **把所有内容塞进 SKILL.md。** 正文超过一两千字，按需加载就失去意义了。大块参考资料拆成独立文件，正文里指路即可。
- **脚本有副作用但没说明。** 会写文件的脚本，务必在正文标注影响范围；涉及删除、推送的操作，要求 agent 先确认再执行。
- **命名冲突。** 个人 skills 与项目 skills 同名时，加载顺序容易踩坑。团队共享的 skill 建议带前缀，如 `acme-release-notes`。

## 可复用建议

- description 模板：`当 <具体任务> 时使用。输入 <X>，输出 <Y>。`
- 正文控制在 500 字以内，细节拆成文件，维持三层结构：元数据常驻 → 正文按需 → 资源文件再按需。
- 把 skills 放进 git 仓库，像维护代码一样 review 和打 tag，团队共享收益最大。
- 每季度清理一次：触发率长期为零的 skill，要么改写 description，要么删掉，别让目录变成垃圾场。

## 总结

Skills 的价值不在“多”而在“准”：用一段轻量的元数据换取完整的按需上下文，再把确定性流程固化到脚本，agent 才能既轻又不弱。对我们团队来说，它已经从锦上添花变成了组织知识的主要载体。踩过坑之后，最重要的经验只有一条——**把 description 当路由规则写，而不是当简介写**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/60045678145f90d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/9a0067b3a57130d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/0e8f1725755d0acf.png)

