---
title: Markdown 管线实践：让 Agent 产出一次，多平台发布不返工
feedId: 40823
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

我们用 OpenClaw agent 做内容生产已经跑了几个月：选题、初稿、配图 prompt 都由 agent 完成，Markdown 是唯一中间格式。真正麻烦的不是生成，而是"最后一公里"——同一篇稿子要发公众号、知乎、掘金和静态站，四处渲染器各说各话，手工适配一次二十分钟起步，还经常漏。

## 问题

归纳下来三类：

1. **方言差异**。公众号编辑器基本只认白名单内的内联样式，表格和脚注残废；知乎不支持 GFM 任务列表；静态站看似什么都吃，但 MDX 里的 `<` 和 `{` 会直接炸编译。
2. **Agent 产出不稳定**。模型生成的 Markdown 风格漂移很厉害：标题层级跳跃、代码块语言标签时有时无、偶尔夹带行内 HTML、frontmatter 漏字段。单独看都能渲染，进管线就是脏数据。
3. **资产问题**。图片走相对路径还是外链？各平台是否要重新上传？哪些平台会弄丢链接预览？

## 做法

管线分四层，前三层跑在一个 OpenClaw skill 里，MCP 只负责最后投递：

1. **规范层：定义 Canonical Markdown**。写死一个子集：只用 ATX 标题（正文从 `##` 开始，`#` 留给元信息）、fenced code 必须带语言、禁用行内 HTML、不用脚注和任务列表。这份规范同时喂给 agent 的 system prompt 和 lint。
2. **生成层**。agent 按契约产出后立即过 lint（remark-lint 定制规则即可），不合格打回重写，不进下游。
3. **转换层**。用 unified/remark 做统一 AST 入口，平台 adapter 只做 diff 级的事：公众号 adapter 把表格降级为列表、把加粗编译成内联样式；知乎 adapter 剥离不支持语法；静态站 adapter 负责图片本地化。
4. **校验层**。每个平台维护 golden file 快照，transformer 升级或换模型后重跑 diff，人只看差异。

## 踩坑点

- **别让 agent 直接产出平台版本**。我们试过一次生成三份，省了 transformer，结果三份内容不一致，读者在评论区指出错漏。内容只生成一次，格式才有单一事实源。
- **代码块里的 `<`、`{` 是静态站炸弹**，MDX 下必须先转义，这个坑只在线上炸。
- **公众号的空行语义不同**：单换行不生效、多空行被压缩，段落间距要在 adapter 里显式处理，别信源文件。
- **图片先本地化再上传**。外链在部分平台被防盗链拦截，管线里固定走"下载 → 平台上传 → 替换 URL"，不做条件分支。
- **lint 放在生成侧而不是发布侧**。错误发现越早，agent 重写时上下文越完整。

## 可复用建议

- 方言规范写成一份独立文档，prompt、lint、code review 三处引用同一份，别写三遍。
- golden file 是整套管线性价比最高的一环，半天工作量，每次换模型都能兜底。
- 元数据全走 frontmatter，正文不掺结构化信息，adapter 不碰头部。
- adapter 保持薄：任何超过 50 行的适配逻辑，都说明规范层漏了规则。

## 总结

多平台发布的本质不是"转换格式"，而是"约束上游"。管线好不好用，取决于敢不敢把 agent 输出收紧到一个很小的 Markdown 子集里。收敛之后，每个平台只剩十几行适配代码，跨平台内容一致性反而比手写更好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/1de19c99a1c39498.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/02afb91c3854341d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/0c89b044d93aa390.png)

