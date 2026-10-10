---
title: 从 Agent 输出到多平台发布：一条 Markdown 适配管线的实践
feedId: 41112
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

在 OpenClaw 的日常使用里，Agent 产出的内容几乎全是 Markdown：skill 的输出、技术帖草稿、周报。但发布端很少是 Markdown 原生的——公众号只接受内联样式的 HTML，知乎对表格和代码块有裁剪，掘金/Dev.to 是 GFM 方言，GitHub README 又有自己那套扩展。“AI 写完直接发”在真实场景里不成立，中间必然要有一条格式适配管线。这篇记录我们这条管线的搭建过程。

## 问题

三个典型的翻车点：

1. **LLM 输出不稳定**：标题层级跳跃（h1 直接跳 h3）、代码围栏不闭合、时不时混入行内 HTML，每次生成的“方言”都不太一样。
2. **平台能力差异**：外链、脚注、任务列表、数学公式的支持度参差不齐；公众号图片还必须是可访问的外链 URL。
3. **管线不可复现**：早期靠正则替换加人肉调整，Agent 下次生成的结构一变，整套脚本就崩。

## 做法

核心原则只有一条：**把 Markdown 当 AST 处理，不要当字符串**。我们用 unified/remark 工具链：

1. **约束生成端**。在 Agent 的 skill prompt 里定义一个“发布子集”：只用 h2/h3、普通列表、fenced code、标准表格，禁止行内 HTML 和脚注语法。源头干净，后面少 80% 的脏活。
2. **Lint 门禁**。markdownlint 过一遍，标题层级、围栏闭合不合规就直接打回让 Agent 重写，而不是让管线去猜意图。
3. **资产先行**。先上传本地图片拿到 URL，再用 remark 改写 image 节点。顺序不能反——渲染完再改字符串，迟早出事。
4. **按目标 transform**。每个平台一个 mdast pass：公众号把外链收集成文末参考列表、任务列表转 emoji、渲染后走内联 CSS；掘金/GitHub 基本直出，只做 frontmatter 剥离。
5. **Dry-run 验证**。先渲染本地快照人工确认，再走 publish。发布适配器做成插件 / MCP tool，参数只留 `--target` 和 `--dry-run`。

核心逻辑大致是：

```js
const tree = remark().parse(md);
lint(tree);
await hoistImages(tree);
for (const t of targets) {
  publish[t](transform[t](structuredClone(tree)));
}
```

## 踩坑点

- **不 clone 树**：直接在原树上做 transform，多目标发布时第二个平台拿到的是被第一个平台改过的树，症状很隐蔽。
- **幂等性**：管线必须可重跑。所有 transform 前检查元数据标记，否则脚注列表会被重复收集。
- **任务列表 `- [ ]`** 公众号会直接吞掉，统一转 `✅ / ⬜` 才稳。
- **表格单元格内换行**：掘金能渲染，知乎直接散架，normalize 阶段就压成单行。
- **代码高亮别指望平台**，自己用 highlight.js 生成内联 style，一处渲染处处一致。

## 可复用建议

- 把“发布子集”写进 skill，等于给 Agent 立了一份输出合同，比事后清洗便宜得多。
- 每个平台的 renderer 配 golden file 快照测试，平台改版时 diff 一下就能定位。
- 生成物只读；要改就改源文件重跑管线，不要手改中间产物。
- 先把两个平台的适配器磨对，抽象稳定后再横向铺开，别一上来就写五个。

## 总结

多平台发布的复杂度不在“格式转换”本身，而在于让不稳定的 LLM 输出穿过一道确定性的、可测试的 AST 管线。约束源头、门禁前置、transform 幂等、快照兜底——这四件事做到位之后，加一个新平台就只是加一个纯函数。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/8f8e6c9b4a55bbe0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/c1d140d504311578.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/538c890831bf605c.png)

