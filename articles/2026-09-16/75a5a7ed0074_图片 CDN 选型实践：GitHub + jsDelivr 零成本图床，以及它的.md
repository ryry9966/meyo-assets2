---
title: 图片 CDN 选型实践：GitHub + jsDelivr 零成本图床，以及它的边界
feedId: 37845
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

在社区里写 Agent 实践记录、插件文档、MCP 工具评测，截图是刚需。对象存储（OSS/COS）要实名 + 按量付费，各类免费图床随时跑路。GitHub 仓库本身就能存图，jsDelivr 在上面套了一层全球 CDN，组合起来是个零成本的折中方案。我自己用它托管了几百张文档配图，跑了快两年，把流程和坑整理一下。

## 问题

直接贴 GitHub raw 链接（`raw.githubusercontent.com`）有三个毛病：没有 CDN 加速、国内访问不稳定、大图加载慢，不适合作为跨站引用的图片源。我们要的是：链接稳定、可自动化生成、未来能低成本迁移。

## 做法

**1. 建一个公开仓库**，比如 `yourname/pics`，只放图片，不放代码。

**2. 上传图片。** 手动场景直接网页拖拽；自动化场景走 Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/yourname/pics/contents/2025/06/a3f9c2.webp \
  -d '{"message":"upload","content":"<base64>"}'
```

**3. 拼 jsDelivr 链接：**

```
https://cdn.jsdelivr.net/gh/yourname/pics@main/2025/06/a3f9c2.webp
```

用 `curl -I` 验证 200 和缓存响应头。

**4. 接入 Agent 工作流（OpenClaw 用户重点）。** 把「压缩图片 → 算哈希命名 → API 上传 → 返回 Markdown 链接」封装成一个 MCP tool。给 Agent 配上之后，写文档时说一句“截图上传”，链接就直接插进正文，不用人肉搬运。

## 踩坑点

1. **国内访问不稳定是最大短板。** jsDelivr 的 ICP 备案 2022 年被吊销后，大陆直连时好时坏。备用域名 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`，或换 `cdn.statically.io/gh/...`。对国内读者重要的页面，给双链接。
2. **缓存与覆盖冲突。** `@main` 分支引用缓存约 12 小时，同名覆盖后 CDN 未必立刻刷新。两条路：内容哈希命名（推荐，一劳永逸），或手动刷新 `https://purge.jsdelivr.net/gh/...`。
3. **仓库体积红线。** 单文件 100MB 硬限制，仓库建议控制在 1GB 内。上传前压缩到 200KB 以内（WebP 质量 80 对文档截图足够），旧图定期归档。
4. **别当网盘用。** jsDelivr 对纯文件分发性质的仓库有过封禁先例。保持“项目附属静态资源”的定位，图片内容本身也是公开的，别放隐私和版权敏感素材。
5. **自动化上传的偶发失败。** API 偶发 422/403（文件冲突或触发滥用检测），哈希命名 + 指数退避重试基本能规避。

## 可复用建议

- 文件名规则统一为 `日期-描述-短哈希.webp`，杜绝缓存问题
- 把 `https://cdn.jsdelivr.net/gh/` 当成**可配置前缀**，未来迁自有 OSS 只改前缀，正文里的链接不用动——这是这套方案最大的隐性收益
- 压缩、上传、返回链接三步做成一个 tool，Agent 一次调用完成贴图
- GitHub 仓库本身就是原始备份，jsDelivr 只是加速层，图永远不会丢

## 总结

GitHub + jsDelivr 适合个人博客、社区文档、Agent 生成内容的配图托管：零成本、自带版本管理、迁移成本近乎为零。它不适合生产关键路径和大流量分发，国内访问质量也要如实评估。接受这些边界，做好前缀可替换和备用域名，这就是一个够用很多年的务实方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/5046ce0a39960f20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/f2399839f29245a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/425f1034aa580038.png)

