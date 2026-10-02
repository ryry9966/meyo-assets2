---
title: Markdown 管线：让 AI 生成的内容一次成型、多端落地
feedId: 40185
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

用 OpenClaw 跑内容类 agent，产物几乎都是 Markdown：LLM 输出它最稳，diff 友好，也能直接进 git。但真正的终点往往不是一个 `.md` 文件，而是公众号、知乎、掘金、GitHub Pages 这些平台。于是问题从“生成”变成了“适配”——同一段 Markdown，在不同平台上的渲染结果差别很大。

## 问题

三个高频痛点：

1. **方言差异**。公众号不认 Markdown、只吃带内联样式的 HTML；知乎会吞掉部分语法；表格、脚注、任务列表各家支持度参差。
2. **AI 输出不守约**。即使 prompt 里写了“只用二级标题”，模型偶尔仍会输出 HTML 片段、四层嵌套列表，或残留 frontmatter。
3. **资源问题**。本地相对路径图片、外链防盗图，发布时直接挂。

## 做法

我们把它拆成四段式管线，每段职责单一、可单独测试：

**1. 生成约束。** 在 agent 的 system prompt 里固化一个“最小方言”：仅 h2–h4、有序/无序列表、标准表格、带语言标注的代码块、图片用占位符 `[img:描述]`。方言规范单独写成一个 md 文件注入 prompt，而不是散落在对话里。

**2. AST 校验。** 用 remark 把产物 parse 成 mdast，遍历检查节点类型是否在白名单内。违规节点走两条路：能自动降级的（h5→加粗段落、脚注→括号内联）直接改写；改不了的把该段落打回 agent 重写，而不是整篇重来。

**3. 归一化。** 发布前触发占位符的真实上传（OSS/图床）并回写 URL；代码块语言映射到各家高亮都稳定的主流标签；清理多余空行与转义。

**4. 平台 adapter。** 核心是 remark-rehype 加自定义 handler：公众号 adapter 把所有样式内联、禁用 class；掘金 adapter 直出原始 Markdown；知乎 adapter 把宽表格降级为列表。每个 adapter 是纯函数，输入 mdast、输出目标格式，配 snapshot 测试。

最后用 MCP 把管线包成一个 `publish` 工具暴露给 agent：调用后先返回各平台预览快照，人工确认再真正提交。

## 踩坑点

- **别用正则处理 Markdown**。嵌套结构一多就崩，mdast 是底线。
- 公众号粘贴 HTML 会丢弃部分标签属性，跨度大的排版宁可截图，别硬转。
- 表格超过三列在移动端必溢出，提前在归一化阶段拦截。
- frontmatter 残留的 `---` 会被部分平台解析成分隔线，入口处先剥一次元数据。
- 预览快照要人工抽检——自动化省掉的是搬运，不是审校。

## 可复用建议

- 方言规范版本化，prompt 与校验器共用同一份定义，避免两边漂移。
- 保留原始 md 作为 artifact，任何一步失败都可重放，不必让 LLM 重新生成。
- adapter 按平台隔离目录，新增平台只是加一个转换函数加测试，不动主体管线。

## 总结

这条管线本质上是把“格式适配”从发布时刻提前到了生成与校验时刻：AI 只负责产出受限方言，机械转换全部交给确定性代码。上线以来格式相关的人工干预基本归零，出问题也能按段定位。思路不新鲜，但对 agent 落地内容工作流来说，这一层的投入产出比很高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/837c369e6103056b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/981ce18c8b5360e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/7fdaf88f159ea875.png)

