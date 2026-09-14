---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践（附 MCP 自动化接入）
feedId: 37488
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

写技术帖、跑 Agent 工作流时经常需要外链图片：文档配图、bot 回复里带的截图、生成结果展示。图片放本地没法分享，贴到聊天群会被压缩。之前用过几个免费图床，隔一阵就关站或限速，链接批量失效，修复成本远高于当初省下的钱。

## 问题

候选方案其实不多：

- **传统免费图床**：随时跑路，无法自定义；
- **对象存储 + CDN**：稳定，但要备案域名、按量付费，个人博客和小项目不划算；
- **GitHub raw 链接**：免费且稳定，但 raw 域名在国内访问慢，也没有 CDN 缓存。

最后选了「GitHub 仓库 + jsDelivr」的组合：托管在 GitHub（跑路风险比第三方图床低一个量级），分发走 jsDelivr 全球 CDN，大陆可达性虽波动但总体可用，还有备用源可切。

## 做法

1. 建一个公开仓库（私有仓库 jsDelivr 不支持），例如 `pics`，按 `yyyy/mm` 分目录。
2. 上传走 GitHub Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_PAT" \
  https://api.github.com/repos/USER/pics/contents/2025/06/demo.png \
  -d '{"message":"upload","content":"'$(base64 -w0 demo.png)'"}'
```

3. 拼接 CDN 地址：

```
https://cdn.jsdelivr.net/gh/USER/pics@main/2025/06/demo.png
```

4. 自动化接入：把上传和拼接封装成一个 MCP 工具（比如 `image_upload`），入参是本地路径或剪贴板图片，出参直接是 CDN URL。这样 OpenClaw 的 Agent 在写帖、回帖工作流里可以自己插图，不需要人工搬运。

细节：PAT 用 fine-grained，只授这一个仓库的 Contents 读写；Contents API 上传单文件上限 1MB，更大的图要走 git push。

## 踩坑点

- **缓存不可控**：jsDelivr 缓存激进，同名覆盖后 CDN 仍是旧图。急删可打 `purge.jsdelivr.net`，但更省事的做法是文件名带内容 hash，天然不可变。
- **大陆可达性波动**：主域名偶发解析异常，可备用 `fastly.jsdelivr.net` / `gcore.jsdelivr.net`。建议把 CDN 域名做成配置项而不是硬编码。
- **@main vs commit hash**：`@main` 跟随分支最新提交，配激进缓存可能出现不一致；对确定性要求高的场景用 commit hash 锁定。
- **仓库膨胀**：GitHub 不适合当媒体库，几百 MB 后 clone/CI 都变慢。图片先压缩（png 转 webp 通常省一半以上），过期素材定期清理。
- **限速**：匿名 GitHub API 每小时 60 次，自动化场景必须带 PAT。

## 可复用建议

- 文件名用 `sha1(内容)` 前 12 位，重名即去重，天然幂等；
- 维护一份 JSON 索引（路径、hash、上传时间），Agent 去重和检索直接查索引，不重复上传；
- 前端渲染留降级：`onerror` 时切 raw 地址，再不行显示占位；
- 上传、拼 URL、写索引三步收进同一个工具里，别散落在各处。

## 总结

GitHub + jsDelivr 不是生产级高可用方案，但对个人博客、社区帖子和 Agent 工作流里"能稳定外链一张图"的需求来说，成本、可控性、可自动化程度的综合分最高。核心心得是把图床当基础设施的一部分来设计——hash 命名、配置化域名、MCP 工具化——而不是一个粘贴图片的网站。后续若 jsDelivr 可达性进一步恶化，切换方案的成本也只是改一个配置项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/693c14e420e161da.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/68d77d7c841ab1c9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/a907795a4c78e524.png)

