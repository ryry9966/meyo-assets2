---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的工程实践
feedId: 37543
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

写博客、维护开源文档、跑 Agent 自动化流程时，经常需要给图片一个稳定的公网 URL。在 OpenClaw 驱动的自动化 pipeline 里，Agent 生成一张插图或截图后，下一步往往就是"发布这张图，拿回一个能贴进 Markdown 的链接"。这个环节看着小，选错方案会持续踩坑。

## 问题

最初直接引用 `raw.githubusercontent.com` 链接，实践下来问题不少：

1. raw 域名本质是文件服务，不是 CDN，响应头不利于浏览器缓存；
2. 大陆访问不稳定，经常超时；
3. 部分平台会限制 raw 域名外链；
4. 每次 push 后同一 URL 内容会变，做不了 immutable 引用。

商用图床要么收费，要么有停服风险。最后选了 GitHub 仓库 + jsDelivr：仓库负责存储和版本管理，jsDelivr 负责全球分发，免费、无鉴权、带正确缓存头。

## 做法

**1. 建一个公开仓库**

专门放图，如 `pics`，按年份分目录即可。必须 public，私有仓库 jsDelivr 拿不到。

**2. 拼 CDN 地址**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

例如 `https://cdn.jsdelivr.net/gh/foo/pics@main/2024/demo.png`。

**3. 压缩后入库**

上传前用 `sharp` 或 `pngquant` 压一遍，单文件控制在几百 KB。jsDelivr 不服务超过 20MB 的文件，GitHub 也建议仓库别超 1GB。

**4. 接入自动化**

写个十几行的脚本：本地图片路径 → 压缩 → 以内容 hash 重命名 → git push → 输出 CDN URL。再包一层 MCP tool，Agent 就能"给图返回链接"一步到位。也可以用 GitHub Actions 监听目录自动完成整个流程。

**5. 缓存刷新**

同路径覆盖图片后 jsDelivr 有小时级缓存延迟。急用时可请求 `https://purge.jsdelivr.net/gh/...` 主动刷新。

## 踩坑点

- **同路径覆盖不生效**：分支引用有缓存，更新图片务必换文件名，content hash 命名天然规避这个问题。
- **`@main` 与 `@<commit>` 的区别**：`@main` 内容会随 push 变化；正式文档建议 pin 到 commit 或 tag，URL 完全 immutable。
- **合规边界**：jsDelivr 定位是开源项目 CDN，个人/社区文档体量没问题，但别把大流量、纯网盘式用途挂上去。
- **不要放敏感图**：仓库公开谁都能拉，CDN 缓存也让删除不彻底。
- **CI 推送权限**：Actions 里 push 需要细粒度 token 且有 `contents:write`，默认 GITHUB_TOKEN 在受保护分支上会被拒。

## 可复用建议

1. URL 生成逻辑收敛到一个函数或配置项，将来换 CDN 域名只改一处；
2. 维护 `manifest.json` 索引（原始名 → hash 名 → URL），方便 Agent 检索复用、避免重复上传；
3. 图片入库即视为 immutable，更新 = 新文件 + 新 URL；
4. 国内可用性需实测，重要场景预留镜像域名做 fallback；
5. 只放自己有版权的图，公开仓库没有隐私可言。

## 总结

GitHub + jsDelivr 是个人和社区场景下成本最低、可控性最强的图片分发方案：存储、版本、CDN 三件事一次解决，配合脚本或 MCP tool 能无缝嵌入自动化流程。它不适合高流量生产环境和大体积媒体，但博客插图、文档截图、Agent 产出的配图完全够用。工程上记住两点：文件名带 hash 保证 immutable，URL 生成逻辑集中管理方便迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/0ba21c4514bcdbb9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/f94d514d9d83954b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/565e027090464bc6.png)

