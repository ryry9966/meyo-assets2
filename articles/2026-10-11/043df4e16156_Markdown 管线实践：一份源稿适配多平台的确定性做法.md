---
title: Markdown 管线实践：一份源稿适配多平台的确定性做法
feedId: 41146
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

用 Agent 生成技术内容后，往往不止发一处：博客、公众号、知乎、掘金、GitHub。各平台 Markdown 方言差异很大，直接粘贴要么样式崩，要么链接失效。常见做法是每次让模型"再转一份公众号版"，但这条路不可控。我们在 OpenClaw 内容工作流里把格式适配做成了一条确定性管线，这里记录一下思路。

## 问题

- AI 输出的 Markdown 看似规范，实际常有硬伤：标题层级跳跃、代码块缺语言标注、列表缩进 2/4 空格混用。
- 平台限制各不相同：公众号不认 GFM 表格、外链不可点击、图片必须走素材接口；知乎会吞部分 HTML；静态站又强依赖 frontmatter。
- 让 LLM 做格式转换不可复现：两次输出不一致，代码块内容甚至可能被悄悄改写。

核心原则只有一条：LLM 负责内容，格式转换交给确定性代码。

## 做法

1. **规范源稿**。AI 生成后先跑 lint（remark-lint）：标题统一从 H2 起、补齐代码块语言、规范缩进。产出 `canonical.md`，这是唯一事实源。
2. **定义平台矩阵**。一个 config 声明目标平台和各自的 adapter，比如 `wechat`、`zhihu`、`juejin`、`hugo`。
3. **AST 变换，不用正则**。用 unified/remark 解析成 mdast，每个 adapter 是一组 transformer：公众号 adapter 把表格降级为列表、外链转文末引用、Mermaid 块调渲染服务换图；hugo adapter 校验 frontmatter 字段完整性。
4. **图片独立子管线**。抓取远程图 → 压缩 → 上传目标平台或 OSS → 回写 URL，映射关系存 manifest，重跑不重复上传。
5. **dry-run + 发布 ledger**。每个平台先产出最终 HTML/MD 本地预览，人工确认后才调发布接口。发布成功后把平台返回的 post_id 和内容 hash 记进 ledger，用于幂等更新。

在 OpenClaw 里整条管线包成一个 skill 即可，MCP 侧只暴露 `upload_image` 和 `publish` 两个工具，由 Agent 做编排，不参与格式逻辑。

## 踩坑点

- 用正则处理 Markdown 是重灾区：代码块里出现相似的围栏模式会把文档直接切烂，结构化操作必须走 AST。
- 公众号会把嵌套列表拍平，两层以上的列表要提前降级成编号段落。
- 忘了剥 frontmatter 就发知乎，开头一排 `---`，读者第一眼就划走。
- HTML 实体双重转义，页面出现字面量 `&amp;`。
- 没记 post_id 前重复发布，产生重复文章，只能手动删。

## 可复用建议

- 平台产物当作构建产物，绝不手改；要改就改源稿重跑。
- adapter 用 golden file 做快照测试：固定输入 MD，断言输出不变；平台规则变更时更新快照再评审。
- 失败按平台隔离：知乎挂了不影响公众号发布，失败项单独记录待重试。
- 图片上传的 token 和配额单独管理，这是管线里最容易过期的环节。

## 总结

多平台发布的本质是"一份规范源 + N 个确定性变换"。把格式转换从 LLM 手里拿回来，交给 AST 工具链，配合矩阵配置、快照测试和发布 ledger，整条链路就可控、可复现、可幂等。管线搭好之后，新增一个平台通常只是几十行的 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/ff83ea6187247215.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/55e512bbc3b8a6a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/113d35846308a5c8.png)

