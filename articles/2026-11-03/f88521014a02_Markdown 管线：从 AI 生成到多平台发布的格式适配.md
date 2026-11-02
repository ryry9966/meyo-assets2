---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43302
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

在 OpenClaw 的自动化场景里，「让 Agent 生成内容再分发到多个平台」是很典型的流程：Agent 产出 Markdown → 人工审校 → 发布到公众号、知乎、掘金或静态站。做过一遍就会发现，「生成」只占管线的前 20%，剩下 80% 都是格式适配。

## 问题

各平台对 Markdown 的支持差异比想象中大：

- 公众号不认 Markdown，只吃内联样式的 HTML，且会剥离 class 和大部分标签属性；
- 掘金 / GitHub 支持 GFM，但脚注、数学公式的支持参差不齐；
- 静态站生成器需要 frontmatter，而社区平台会把它原样吐进正文；
- AI 产出本身也有坑：标题从 H3 开始、代码围栏不闭合、混入 HTML、编造图片链接。

用正则做字符串替换去抹平这些差异，写到最后必然是一堆互相打架的脚本。

## 做法

我的思路是把 Markdown 当 AST 处理，管线分四层：

1. **规范化层**。Agent 输出先过 lint：统一为单个 H1、闭合代码围栏、去掉 AI 腔的填充句、校验图片链接可达。用 remark + 自定义 plugin 实现，这层可以包成 MCP tool 或发布 hook，让 Agent 产出后自检一轮。
2. **中间表示层**。规范后的 GFM + frontmatter 作为唯一事实源，所有平台版本都从这里派生，不做平台间二次转换。
3. **适配层**。每个平台一个 adapter：公众号走 remark → rehype → 内联样式（可参考 doocs/md 的思路），掘金 / GitHub 直接输出 GFM，静态站注入 frontmatter。adapter 内维护一张节点白名单，遇到不支持的节点（如脚注）要么降级转换，要么显式报错。
4. **发布层**。图片在发布时才上传并重写链接；发布前 dry-run 渲染一遍，人审通过才推送。

## 踩坑点

- **别用正则处理 Markdown**。代码块里的 `#`、嵌套列表的缩进，正则几乎必然误伤。在 mdast 层面操作，边界条件少一个数量级。
- **公众号代码高亮要预渲染**。它不支持运行时高亮，必须在构建期把代码块转成带颜色的 span，且样式全部内联。
- **中英混排的下划线**。`snake_case` 出现在链接文本里时，部分平台解析器与 GFM 行为不一致，发布前快照对比能兜底。
- **adapter 要 fail closed**。未知节点被静默丢弃是最难排查的事故来源——内容「少了」但流程全绿。宁可报错中断。
- **frontmatter 泄漏**。有一次发布忘了剥离，给 Agent 的指令模板原样出现在正文开头，相当尴尬。

## 可复用建议

- 给每个 adapter 建 golden file 快照测试：内容不变，渲染结果就不该变；
- 语法白名单随管线配置进版本库，内容仓和管线仓同仓管理；
- 图片链接统一放 frontmatter，发布时解析，避免改图就要改正文；
- 人审环节放在适配之后、发布之前，审的是「平台最终呈现」，不是原始 Markdown。

## 总结

这套管线的核心不是工具选型，而是 mindset：AI 生成的是素材，不是成品。中间格式 + AST 适配 + 快照测试，三样齐了，多平台发布就从「每次手工修格式」变成「跑一条命令」。管线本身也是代码，该测试、该版本化，一样都不能少。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/74ab4e602f27dc00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/70076f843180b1b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/fad5e6c77e43ac95.png)

