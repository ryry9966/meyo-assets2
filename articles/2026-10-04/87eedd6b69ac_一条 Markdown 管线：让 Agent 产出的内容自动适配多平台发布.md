---
title: 一条 Markdown 管线：让 Agent 产出的内容自动适配多平台发布
feedId: 40341
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

用 Agent 产出技术帖已经是常态：模型输出的正文天然是 Markdown，结构清楚、写作快。但真正麻烦的是发布环节——公众号、知乎、掘金、GitHub Pages 各家对 Markdown/HTML 的支持差异很大，手工逐平台排版耗时且容易走样。我们在 OpenClaw 的内容自动化实践里，把这条链路收敛成了一条 Markdown 管线，这里整理一下思路和坑。

## 问题

AI 生成的 Markdown 直接投递平台，会撞上四类问题：

1. **方言差异**：脚注、任务列表、扩展表格，各家渲染器支持程度不一；
2. **样式剥离**：公众号编辑器会剥掉 `<style>` 和 class，只认内联样式，代码高亮全丢；
3. **图片链路**：外链图床有防盗链，平台图床要单独上传，URL 必须重写；
4. **模型输出不稳定**：偶尔混入裸 HTML、标题层级跳跃、转义混乱。

## 做法

核心原则是**单一事实源 + 适配器模式**。

**第一步：约定规范子集。** 在给 Agent 的 prompt 契约里明确"只允许 CommonMark + GFM 安全子集"：不用脚注和任务列表，表格单元格不嵌套，图片一律标准占位形式。这是管线的地基，比后面任何转换都重要。

**第二步：归一化。** Agent 写入 `drafts/` 后，先跑 lint（裁剪版 markdownlint 规则）加 remark 归一化：统一换行、修复标题层级、剥掉模型偶尔吐出的裸 HTML 节点，产出 canonical Markdown。

**第三步：平台适配器。** 基于 unified 生态转换：`remark-parse → remark-gfm → remark-rehype → rehype-*`。每个平台一个 adapter，本质是纯函数：`(canonicalMD, platformConfig) => output`。

- 掘金/GitHub：直出 Markdown，只做图片 URL 重写；
- 知乎：转 HTML，收紧标签白名单；
- 公众号：转 HTML 后用 `rehype-inline` 全量内联样式，代码块用 shiki 预渲染成带内联颜色的 `<span>`。

**第四步：接入 OpenClaw。** 适配器包成 MCP tool：`render(platform, source)`，带 `dryRun` 参数返回预览。发布前必须 dryRun 人工确认再调 publish。图片上传独立成 `upload_assets` 工具，先传图拿回 URL，再渲染正文。

## 踩坑点

- 不要在 prompt 里要求模型直接输出内联样式，效果差且不可控，样式统一放适配层；
- 公众号对 `<table>` 支持极差，宽表会溢出。我们超过 4 列自动降级为定义列表；
- shiki 渲染产物体积大，长文几十个代码块会让公众号编辑器卡顿，注意裁剪主题 token；
- 图片 URL 重写要在渲染前完成，渲染后正则替换会被代码块里的示例链接误伤；
- golden file 快照测试必须有。适配器是纯函数，正好给每个平台维护输入/输出快照，改样式跑一遍就知道影响面。

## 可复用建议

1. 规范子集写进 Agent 系统提示词，同时用 lint 双重校验，别只靠提示词；
2. 适配器保持无状态无 IO，网络操作（传图、发布）外置成独立工具；
3. 新增平台只写 adapter 加快照，不动管线主体；
4. canonical Markdown 永久归档，平台规则变了随时重渲染，不必回炉让模型重新生成。

## 总结

这条管线没有黑科技，本质是编译器思路：规范子集是语法约束，归一化是中间表示，适配器是后端目标。把格式适配从 Agent 的职责里剥离后，模型只管产内容，管线负责"长什么样"。跑顺之后，多平台发布从每次半小时的手工活，变成一次 dryRun 确认。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/1080e16df45994f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/08426579cba37f92.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/2c49e38ab089fa67.png)

