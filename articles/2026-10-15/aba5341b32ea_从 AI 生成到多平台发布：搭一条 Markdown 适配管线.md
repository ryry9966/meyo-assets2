---
title: 从 AI 生成到多平台发布：搭一条 Markdown 适配管线
feedId: 41579
source: 综合讨论
publishedAt: 2026-10-15
---

# 背景

最近在 OpenClaw 里把内容生产半自动化了：agent 负责整理周报、技术笔记和长文初稿，模型统一产出 Markdown 存进仓库，再用 MCP tool 触发发布。目标是“写一次，发公众号、知乎、掘金和自建博客”。听起来很美好，实际做下来发现坑几乎全在格式适配这一层。

# 问题

各平台对 Markdown 的支持是“方言”级别的：

- **公众号**：不支持外链（只认白名单域名），粘贴后样式依赖内联 CSS，代码高亮基本要靠内联 `span`；
- **知乎**：会吞裸 HTML，表格和公式有自己的一套语法；
- **掘金 / 博客**：吃标准 GFM，但 front-matter 和代码块语言标记要单独处理；
- **图片**：AI 生成的相对路径或外链图床，粘到哪儿都容易裂。

直接把模型输出贴出去，十次有八次要手工返工。

# 做法

思路是把“内容”和“发布形态”拆开，管线分四层：

1. **归一化层**。所有 AI 输出先过 remark + markdownlint：标题层级统一从 h2 开始、修正列表缩进、图片补 alt、删掉模型偶尔混进来的裸 HTML。这层是幂等脚本，重复跑无副作用。
2. **资源层**。扫描 `![]()` 引用，本地图片先传 OSS，改写为外链并写缓存（同一文件不重复上传）。顺手把 SVG 转成 PNG，兼容性更稳。
3. **适配层**。每个平台一个 renderer 插件：
   - 公众号：Markdown → HTML，再走 style-inliner 把样式压成内联；外链统一转成文末引用列表；
   - 知乎：剥 HTML、表格降级为列表，LaTeX 单独走一条转换分支；
   - GFM 系：基本直出，只处理 front-matter 和围栏代码块。
4. **校验层**。发布前 CI 跑链接检查和代码块语言统计，输出一个 diff 预览页，人眼确认后再推。

工程上用一条 CLI 串起来：`md-publish build --target mp|zhihu|juejin`，同时暴露成 MCP tool，agent 自己就能调用“把这篇渲染成公众号格式”，发布动作也可以挂在定时任务里。

# 踩坑点

- 代码块内嵌 ``` 围栏会被中间层吃掉，归一化时统一改用四反引号包三层；
- style-inliner 处理不了伪类，公众号产物里干脆禁用 `:hover`，全部显式样式；
- 图床上传失败必须重试 + 输出失败清单，否则链接会静默变裂图；
- 公众号粘贴丢样式，最后是走剪贴板 API 写 rich text 才稳定，直接复制 `innerHTML` 是坑；
- 最重要的一条：**别让模型直接产出“面向公众号的 HTML”**。prompt 里只约定标准 GFM，方言转换全部留给管线，否则 source of truth 会被污染，多平台复用就废了。

# 可复用建议

- 适配逻辑永远放代码里，不要放 prompt 里；
- 发布产物（HTML）不入库，仓库只存 source；
- 每层可独立运行、可单测，renderer 写成纯函数最省心；
- 加一个 `--dry-run`，输出预览链接，发布前必看一眼。

# 总结

这套管线不到五百行，换来的是“写完即发”。核心经验一条：把 AI 生成物当成不可信输入，归一化、资源化、适配、校验各司其职。平台方言永远是 renderer 的事，不是模型的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/a54657820839a2bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/d20a858befdb4096.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/0186d310c7aaf185.png)

