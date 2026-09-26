---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 39149
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

用 Agent 产出内容后，落地的第一步往往不是“写得好不好”，而是“发得出去吗”。同一篇 Markdown，公众号要转内联样式的 HTML，知乎、掘金吃原生 Markdown 但各有阉割，静态博客要 front matter，图片还得换源。格式适配如果靠复制粘贴，很快就会成为自动化链条上最脆的一环。

## 问题

实际跑下来，脏数据来自两侧：

1. **AI 生成侧**：模型输出经常夹带裸 HTML、标题层级跳跃（从 `##` 直接跳到 `#####`）、代码块语言标识混乱（js/javascript 混用）、智能引号、图片无 alt。
2. **平台侧**：公众号正文不允许外链（只能文末引用），外链图片会被拦截；GFM 表格和脚注在很多渲染器里原样吐出；front matter 忘了剥离就直接发布。

## 做法

我们的管线分四段，核心思路是“一份 canonical Markdown，适配器放边缘”：

1. **约束生成**。在 system prompt 里固化输出规范：标题从 `##` 起、纯 GFM、图片用相对路径占位、禁裸 HTML。生成后立刻过 remark-lint，不合格自动回炉一次。生成侧就是一个挂了发布类 MCP 工具的 Agent。
2. **AST 归一化**。用 remark/unified 做一遍规范化 pass：标题层级拉平、代码语言按映射表统一、图片补 alt、front matter 剥离后存入元数据。全程操作 AST，不碰正则。
3. **平台适配器**。每个目标平台一个 renderer：公众号走 remark-rehype 后注入内联样式，正文外链转文末引用列表，图片先传素材库再替换 URL；博客目标生成 front matter + MDX；知乎、掘金基本直出归一化后的 Markdown。
4. **发布与校验**。每个适配器有 golden file 快照测试；发布前 dry-run 输出预览，人工确认后再调发布接口；发布记录落库（slug + 内容 hash + 目标 + 时间），可追溯、可重发。

## 踩坑点

- 公众号会拦截外部图片域名，必须在适配器里强制走素材库上传，别指望“渲染完就完事”。
- YAML front matter 里含冒号的 title 不加引号会解析炸，schema 校验（zod）要放在管线最前面。
- 模型偶尔在代码块里再套代码块（嵌套 fence），lint 规则需要专门加一条。
- emoji 在部分渲染器里会吞掉后面的空格，导致段落粘联，归一化阶段统一处理。
- 改了适配器没跑快照，发出去才发现样式崩——快照测试进 CI，别靠人肉。

## 可复用建议

- 把 canonical 格式写成一份一页纸 spec，同时塞进生成侧 prompt 和 lint 配置，两边对齐。
- 所有转换基于 AST，正则只留作最后一公里的字符串修补。
- 适配器写成无状态纯函数：输入 canonical MD + 元数据，输出目标格式，天然好测。
- 保留 dry-run 和人工确认闸门。自动化到 90% 就够，最后 10% 留给人。

## 总结

多平台发布本质是个编译问题：canonical Markdown 是源码，平台适配器是后端。把格式适配从手工操作变成“有 lint、有快照、有 dry-run 的构建流程”，Agent 产出的内容才真正具备一次生成、多端发布的工程基础。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9ac708e4421538b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/54ceba1e5910f3e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1c25424f05da7deb.png)

