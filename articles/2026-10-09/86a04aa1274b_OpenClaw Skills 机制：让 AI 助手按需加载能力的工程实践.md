---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 40967
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

做 Agent 的人大多遇到过同一个问题：能力越多，system prompt 越臃肿。工具说明、业务规则、输出规范全塞进上下文，结果 token 成本上升，模型注意力被稀释，指令遵循率反而下降。OpenClaw 的 Skills 机制就是针对这个矛盾设计的：把能力拆成独立单元，启动时只加载"目录"，命中时才加载"正文"。

## 问题：全量注入 vs 按需加载

假设接入了七八个能力，全量注入的代价很直观：

- 每次会话固定消耗大量 token，与是否实际使用无关
- 无关说明占据注意力窗口，核心指令容易被冲淡
- 能力更新要改 prompt，多人协作时冲突频繁

Skills 的思路是渐进式披露，分三级：

1. **元数据级**：name + description 常驻上下文，几十 token
2. **正文级**：SKILL.md 主体，触发条件命中才注入
3. **附件级**：references/ 和 scripts/，模型判断需要时再读

## 做法

**第一步：盘点。** 翻最近几十条会话记录，把反复出现的任务列出来。判断标准很简单——同一种事你解释过两遍以上，就值得做成 skill。

**第二步：建目录。** 每个 skill 一个文件夹，最小结构：

```
skills/
  weekly-report/
    SKILL.md          # 必需
    references/       # 可选：详细规范、历史模板
    scripts/          # 可选：确定性脚本
```

SKILL.md 头部是 frontmatter：

```yaml
---
name: weekly-report
description: 生成周报。当用户提到周报、weekly、本周总结时使用。输入为工作项列表，输出为固定四段结构。
---
```

**第三步：写正文。** 正文只写"怎么做"，不写"是什么"。规范细节挪到 references/，格式转换、数据统计这类确定性步骤写成脚本，正文里只留一行调用说明。

**第四步：验证。** 用真实会话跑触发测试：换三种说法问同一个需求，确认都能命中；再问几个无关问题，确认不误触发。

## 踩坑点

- **description 太抽象**（如"辅助处理文档"）→ 永远不触发。description 是路由键，必须包含用户真实会说的词。
- **description 太宽**（如"处理一切文本"）→ 抢走其他 skill 的触发机会。
- **所有内容堆进正文**。正文一旦触发就全量注入，300 行正文等于每次触发都交一遍 token 税。
- **脚本里写死绝对路径**。skill 可能被不同环境加载，路径要相对 workspace 解析。
- **改了 skill 不重开会话**，以为没生效，实际旧版本还留在上下文里。

## 可复用建议

- 一个 skill 只做一件事，宁可拆两个，也别做一个"万能 skill"。
- 用 git 管理 skills 目录，review 时重点看 description 改动——那是路由逻辑，改错影响面比正文大。
- 每个准备一句冒烟测试 prompt，改动后跑一遍再合并。
- 确定性逻辑交给脚本，模糊判断才交给模型，别让模型做机器更擅长的事。

## 总结

Skills 的本质不是"给 AI 更多能力"，而是控制上下文中信息的可见性：启动时轻、命中时准、深入时全。花一个下午盘点和拆分，换来的是更稳定的触发、更低的成本，以及一份可版本化、可协作的能力资产。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/99367ad77f440f33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/34cec32010a3ec49.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/0182cdfaabbbefac.png)

