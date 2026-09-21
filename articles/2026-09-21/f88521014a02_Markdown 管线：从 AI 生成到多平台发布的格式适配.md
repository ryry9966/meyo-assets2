---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 38333
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

我们让 OpenClaw 起草技术内容已经跑了一段时间：agent 出稿、人工校对、多渠道分发。很快发现瓶颈不在生成，而在“最后一公里”——同一篇 Markdown 要发到博客、公众号、知乎、掘金，每个平台的格式规则都不一样，手工适配又慢又不可复现。

## 问题

AI 生成的 Markdown 离“可直接发布”有距离，两层问题叠在一起：

**生成质量**：开场白（“好的，以下是……”）、有时全文被包在 ```markdown 围栏里、标题层级从 ## 开始、代码块缺语言标注、中英文标点与空格混排。

**平台差异**：公众号不认原生 Markdown，图片必须走素材库，表格基本不可用；知乎和掘金的 HTML 白名单各不相同；标题、摘要长度限制也各有一套。

人肉适配的结果是：博客版改完，公众号版已经漂移，换个人就复现不了流程。

## 做法

我们把管线拆成三层，核心原则是 **normalize once, adapt at the edge**：

**1. 规范化**：agent 输出先过 lint 修复。入口检测并剥掉 LLM 的包裹围栏和开场白，统一标题从 h1 开始，补全代码块 language，中英文之间加空格，标点归一。所有文本变换基于 remark 的 mdast AST 而不是正则，且跳过 code 节点；修复要求幂等——同一输入跑两遍结果一致。

**2. 结构转换**：canonical Markdown 是唯一事实源，平台版本全部是构建产物。链路用 unified 生态：remark-parse → 自定义 mdast 插件（表格转列表、剔除平台不支持的节点）→ remark-rehype → 按平台 profile 走不同的 rehype 插件链。

**3. 平台适配**：每个平台一个 YAML profile：标签白名单、代码高亮方案、图片策略、标题/摘要长度上限。公众号这类只吃内联样式的，rehype 之后接 juice 做 CSS 内联；图片先传平台素材库，拿到 CDN URL 再回写正文。

最后把整条管线封成 MCP tools：`validate_markdown` / `adapt_for_platform` / `publish`，agent 编排时先 validate，人工确认后才 publish。

## 踩坑点

1. **别用正则处理 Markdown**。代码块里一段含特殊字符的示例就能把全局替换打穿，文本级变换必须在 AST 上做；
2. 标点/空格归一化同样要排除代码块，否则代码全被“格式化”；
3. 公众号外链图片预览正常、发布后裂图（防盗链），必须走素材上传并回写 URL；
4. front matter 忘剥，标题栏出现一堆 YAML；
5. 不要同时新开多个平台的 profile，先跑通一个再横向复制，否则出错难定位。

## 可复用建议

- canonical 源文件进 git，平台产物视为可重建的缓存；
- 给管线写 golden test：固定几篇样例文档断言输出快照，平台改版时 diff 一眼可见；
- 规则配置化、可覆盖：某条规则对某平台不适用，就在 profile 里覆盖，别 fork 管线；
- adapt 和 publish 分开，发布前永远留人工 checkpoint。

## 总结

一句话：生成端只负责产出一份干净的 canonical Markdown，所有平台差异收敛到 profile 层；用 AST 不用正则，产物可重建，发布前留人。跑通之后，多平台发布从每次半小时的手工活，变成了 CI 里的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/045f60bfb645bccd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/5887757a9c3598c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/9195a9d60242454b.png)

