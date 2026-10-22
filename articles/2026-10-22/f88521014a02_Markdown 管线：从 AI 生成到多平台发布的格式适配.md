---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 42305
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

用 OpenClaw 跑内容自动化的同学大概都遇到过同一个场景：agent 生成的 Markdown 质量不错，但发不出去。GitHub 正常渲染，公众号只认内联样式的 HTML，知乎和掘金各自裁剪语法，外链图片还被防盗链拦住。模型的输出默认是"它认为的 Markdown"，而不是你下游平台要的 Markdown。于是格式返工成了每次发布前的人工环节。

我们的做法是把这一段做成一条固定的管线，让 agent 产出一键进发布流程。

## 问题

实际拆下来是三类：

1. **模型输出不稳定**：混入 `<br>`、`&nbsp;`、带 style 的 HTML 标签，脚注和任务列表也是常客；
2. **平台方言差异**：脚注、删除线、宽表格、数学公式的支持度参差不齐，公众号差异最大；
3. **资源依赖**：外链图床不可控，微信发布还要求先传素材库拿回自己的 URL。

## 做法

核心思路是 **AST 中间表示 + 平台 profile**，分四层：

1. **生成约束层**：system prompt 明确要求只输出纯 GFM、禁用 HTML 标签和脚注。这是第一道过滤，但不能信任它，只能降频。
2. **归一化层**：用 remark-parse 把 Markdown 解析成 mdast，AST 遍历清理 html 节点、裸 URL 转标准链接、超过 4 列的表格降级为列表。全部在语法树上操作，不碰字符串。
3. **渲染层**：每个平台一个 profile 配置（支持元素白名单、代码高亮方案、图片策略）。目标是公众号就走 remark-rehype 加内联样式注入；GitHub 目标直接序列化回 Markdown 透传。
4. **资源与校验层**：检测 image 节点，MCP 工具按内容 hash 去重后上传素材库并回填 URL；发布前 remark-lint 按目标平台白名单跑一遍，报错回传给 agent 自修复一轮。

在 OpenClaw 里落地为一个 skill，对外暴露 normalize / render / publish 三个 MCP tool，agent 按需编排调用。

## 踩坑点

- **正则处理 Markdown 是坑王**。第一版省事用了正则替换，结果一段内嵌 fenced code 的表格直接被打穿——代码块里的 ``` 和表格管道符会误伤一切。第二天全部重写成 AST 遍历，这条没有商量余地。
- 模型对 `<br>` 的执念比想象中强，prompt 约束只能减少不能归零，AST 层的 html 清理必须常驻。
- 公众号粘贴时 `pre` 内的换行会被吞，转 HTML 时要给代码块额外包一层带 white-space 属性的兼容 span。
- 图片转存务必按 hash 缓存，否则每次发布重复上传，素材库很快爆炸。
- remark 插件生态版本间行为有差异（gfm 表格解析就动过），除了 lockfile，归一化层建议加快照测试。

## 可复用建议

- 平台差异全部收敛进 profile 配置文件，接新平台 = 一个配置 + 一个渲染分支，不要把 if 散落在管线里；
- LLM 原始输出保留为 source of truth，各平台文件都是派生物，随时可重建；
- 发布前让 agent 自检：lint 报告贴回上下文让它自己修一轮，实测能压掉九成格式问题；
- 归一化规则全部配快照测试，升级依赖前先跑。

## 总结

这条管线没有黑科技，就是"约束生成 → AST 归一化 → 配置化渲染 → 资源兜底"四步，工作量两三天。换来的是 agent 产出可以直接进发布流程，格式返工这个环节彻底消失。对做内容型自动化的团队来说，格式适配层值得当成基础设施来养，而不是每次发布的临时补丁。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/9494a1509f5af0e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/f5cde9b02dd52dd8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/7ecc8091a0910fae.png)

