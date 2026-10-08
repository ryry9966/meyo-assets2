---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与自动化接入
feedId: 40965
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

最近把博客配图、Bot 输出的截图和 Agent 生成的图表统一迁到了 GitHub + jsDelivr 这套方案。起因很简单：云 OSS 要实名、要计费、要配防盗链；第三方图床存活率堪忧；自建又没必要。GitHub 仓库当源站，jsDelivr 当免费 CDN，是介于两者之间比较务实的一档。

## 问题

- `raw.githubusercontent.com` 没有 CDN 缓存，部分地区访问不稳，贴进 Markdown 加载慢；
- Agent / 插件的自动化流程需要一个能直接写进报告的稳定外链，供任意渲染器消费；
- 成本为零，日常维护成本也要低。

## 做法

1. 建一个 **public** 仓库（如 `assets`），按 `img/YYYY/MM/` 归档；
2. 链接格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`，`ref` 可以是分支（`@main`）也可以是 tag；
3. 上传走 git push，自动化就是在这之上包一层脚本或 MCP 工具——输入本地图片，压缩后 commit & push，拼出 CDN URL 返回：

```bash
IMG_DIR=img/$(date +%Y/%m)
cp "$1" "$IMG_DIR/"
git add . && git commit -m "add $(basename "$1")" && git push
echo "https://cdn.jsdelivr.net/gh/USER/assets@main/$IMG_DIR/$(basename "$1")"
```

4. 把这条链路注册成 OpenClaw 的一个 MCP tool：Agent 生成图表后直接调用，拿到 URL 写进 Markdown 报告，全程不需要人碰 git。

## 踩坑记录

- **`@main` 有缓存**。jsDelivr 对分支引用的缓存 TTL 较长，刚推的新图可能 404 或拿到旧图。用 `https://purge.jsdelivr.net/gh/...` 手动刷新，或者打 tag 用固定版本引用。
- **仓库必须 public**。private 仓库 jsDelivr 拿不到内容，所以千万别放任何敏感截图。
- **单文件上限 50MB**，但重点是先压缩（squoosh / tinypng）再提交。图片进了 git 历史就很难删干净，仓库体积会随时间无限膨胀。
- **国内可用性有过波动**。这是选型前要想清楚的边界：个人博客、Bot 输出、文档配图没问题；面向生产的强依赖业务，建议保留降级路径（raw 链接做备用 base URL，或迁到带 ICP 的 OSS）。
- 用 GitHub API 自动化时，用 fine-grained PAT，只授这个仓库的 Contents 读写权限，token 放环境变量，别进代码库。

## 可复用建议

- 文件名加日期 + 内容 hash（如 `20250115-a3f2.png`），天然避免重名和缓存歧义；
- 长期文档用 tag 引用，高频更新的才用 `@main` + 手动 purge；
- 把「压缩 → 上传 → 返回 Markdown 片段」封装成一个工具函数或 MCP tool，Agent 侧只关心「给我一个链接」，不关心背后是 jsDelivr 还是 OSS，将来换源只改一处；
- 仓库本身就是源站和备份，CDN 只是缓存层——这套架构的迁移成本基本为零。

## 总结

GitHub + jsDelivr 图床的本质是「git 当 origin，jsDelivr 当缓存」。它不完美：国内可用性要兜底，缓存刷新要自己处理。但对个人内容流和 Agent 自动化输出来说，零成本、零维护、链路透明，性价比很高。想清楚边界再用，它就很省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/c6cca09a97ee7e20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/c0eedea4507e07ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/ea2706ef6d9d8811.png)

