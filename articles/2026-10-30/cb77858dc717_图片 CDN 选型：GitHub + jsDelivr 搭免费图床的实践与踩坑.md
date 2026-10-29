---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 42984
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景

在 OpenClaw 社区写帖、让 Agent 生成报告、MCP 工具输出运行截图，都绕不开同一个问题：图放哪。对象存储要实名/备案还按量计费，商业图床有流量限制和外链失效风险，直接贴 `raw.githubusercontent.com` 链接加载体验又很差。我的需求很朴素：个人博客加社区帖的量级，零成本、URL 可长期引用、能被脚本自动化。最终方案：**GitHub 公开仓库存图，jsDelivr 做全球分发**。

## 做法

1. 建一个单用途公开仓库（如 `assets`），只放图片，目录按 `img/yyyy/mm` 组织，别和代码混。
2. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`，ref 可以是分支、tag 或 commit。
3. 文件名用内容哈希：`ab12cd34.webp`。上传前统一压缩到 200KB 以内、转 webp。
4. 自动化：写了个 30 行的脚本调 GitHub Contents API 上传，再封装成 MCP 工具 `upload_image(path) -> url`。Agent 写完帖直接调用，返回的 URL 自动插入 Markdown，整条链路无需人工介入。
5. 每月打一个 tag（如 `v2025.06`），对外引用统一走 tag 而不是 `@main`。

## 踩坑点

- **同名覆盖不生效**：jsDelivr 对 `@main` 这类分支引用有约 12 小时缓存，替换同名图片后 CDN 甚至浏览器都可能返回旧图（`purge.jsdelivr.net` 只能清 CDN 缓存，管不了浏览器）。内容哈希命名直接绕开这个坑。
- **别用 Git LFS**：jsDelivr 返回的是 LFS 指针文件，不是图片本身。
- **单文件上限 20MB**，超限直接拒绝，压缩是必修课。
- **仓库必须 public**，且改名、删除、转私有都会让所有外链 404。
- **域名可用性会波动**，历史上 jsDelivr 在部分地区的访问不稳定过。base URL 建议做成配置项，出问题可切 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等端点，仓库本体始终是唯一的 source of truth。

## 可复用建议

- URL 结构一次定好，仓库和 base URL 全部抽成配置，后期换 CDN 只改一行。
- MCP 工具的返回值除了 URL，最好带上文件大小和哈希，方便 Agent 自检上传结果。
- **克制用量**：个人博客、社区帖量级没问题，但 GitHub ToS 明确不允许把仓库当作主要分发渠道。别拿它当网盘堆视频和大批二进制，有封号风险。
- 本地或私有仓库留一份原图备份，公开仓库只放压缩后的发布版本，留好退路。

## 总结

这套方案的本质是"GitHub 当存储、jsDelivr 当分发"：成本为零，约束清晰——文件小、命名不可变、用量克制。对 OpenClaw 用户来说，真正的价值在最后一步：把上传动作封装成 MCP 工具后，"Agent 产出内容 → 图片自动上链 → 引用即用"完全自动化，图床从手工作业变成了基础设施。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/73aec4fc627a74f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/1d48a3a2ab8711a5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/62eb3ce800d9d240.png)

