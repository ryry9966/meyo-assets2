---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 42553
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

Agent 工作流跑起来之后，图片产出明显变多：帖子的配图、生成的插画、README 和文档里的截图。这些图最终都需要一个稳定、可直链的 URL，贴进 Markdown 或让 bot 回复时才能正常显示。

对比过几条路：对象存储 + CDN 稳定，但备案和计费对个人场景太重；公共图床服务有额度限制，链接说没就没。最后落到 **GitHub 仓库做存储、jsDelivr 做 CDN**：零成本、自带 HTTPS，而且整条上传链路可以脚本化，Agent / MCP 工具能直接调用。

## 做法

**1. 建仓**：建一个公开仓库（如 `assets`），按 `年/月/` 分目录，文件名用「语义 + 内容哈希」，例如 `cover-a3f2c1.webp`，从根上杜绝重名覆盖。

**2. 引用格式**：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或tag>/<路径>
```

jsDelivr 对分支 URL 缓存较短（约 12 小时），tag / commit 哈希基本按永久缓存处理。我的习惯是：写作期用 `@main` 方便替换，文章定稿后把整批图打上 `v2025.01` 这类 tag，URL 从此不可变。

**3. 脚本化上传**：十来行 CLI：接收本地路径 → 压缩转 webp → 哈希命名 → 用细粒度 PAT（只授 Contents 写权限）push → stdout 输出 CDN URL。再包一层 MCP tool，Agent 生成图片后直接拿到可用外链，全程无人工。

注意 commit 频率要克制，批量合并上传即可，别一个 workflow 跑出上百个提交。

## 踩坑点

1. **同名覆盖不生效**：换了图但文件名没变，CDN 返回的还是旧缓存。要么去 `purge.jsdelivr.net` 手动刷新，要么（推荐）哈希进文件名让文件不可变，一劳永逸。
2. **国内直连波动**：jsDelivr 在大陆的可达性起起伏伏，不能假设 100% 可用。关键图留后手——换 `fastly.jsdelivr.net` 等备用域名，或自建兜底源，核心资源别吊死一棵树。
3. **体积红线**：GitHub 单文件超 100MB 直接拒绝，jsDelivr 也有单文件上限（约 20MB），仓库过大会被官方标记。压缩前置，别让截图原图混进来。
4. **公开即公开**：公开仓库谁都能爬。带 token 的截图、内部面板、隐私数据一律不进仓库。PAT 用细粒度最小权限，不要拿全权限的 classic token 复用。
5. **用途边界**：这套组合的定位是个人项目的静态资源。大规模商业盗链既违背社区共识也可能被封禁 CDN 入口；个人博客、文档、bot 件的量级完全没问题。

## 可复用建议

- 命名规范：`年/月/内容哈希.webp`，不可变文件省掉一切缓存纠纷。
- 发布前固定到 tag，之后「改了源图弄挂线上」这种事故不会发生。
- 上传脚本封装成 path → URL 的标准工具（CLI 或 MCP），任何 Agent 工作流即插即用。
- 压缩写进脚本里而不是靠自觉：截图超 500KB 一律转 webp。

## 总结

GitHub + jsDelivr 是一个「够用」方案：零成本、链接格式多年不变、全自动可接入，匹配个人 Agent、博客、文档的图片体量。但要清楚边界——国内连通性波动和公开仓库的隐私约束是两个硬前提，重要资源永远留备份。别拿它当企业级 CDN，个人规模下，稳定和免费就是它最大的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/819e474ee2305539.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/c529f7a0bdcb3815.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/77c62ddc3748ce0f.png)

