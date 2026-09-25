---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与踩坑
feedId: 39040
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

写技术文档、发社区帖、给 Agent 的产出配截图——图床是个人开发者绕不开的基建。可选方案无非三类：对象存储（按量付费，绑自定义域名还要备案）；公共图床（SM.MS 之类有额度限制，外链有失效风险）；自建（要一台服务器和持续运维）。对低流量场景，GitHub 仓库当存储层、jsDelivr 当分发层，是成本最低、也最容易脚本化的组合。我用它给博客和技术帖供图一年多，没出过需要迁移级别的事故。

## 问题

三个硬需求：一是外链要过 CDN，否则每次加载都回源，打开慢；二是国内可访问——`raw.githubusercontent.com` 直连基本不可用；三是上传要能自动化，最好封装成 OpenClaw 的插件或 MCP tool，让 Agent 写帖时顺手把图传了、链接贴了。

## 做法

1. 建一个公开仓库，如 `imgbed`，按用途分目录：`blog/`、`docs/`、`agent/`。
2. 上传：网页拖拽、`git push` 都行，自动化走 Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/imgbed/contents/blog/2025-06-01-a1b2.webp \
  -d "{\"message\":\"upload\",\"content\":\"$(base64 -w0 img.webp)\"}"
```

3. 拼 jsDelivr 外链：`https://cdn.jsdelivr.net/gh/<user>/imgbed@main/blog/2025-06-01-a1b2.webp`。带分支名的链接缓存约 12 小时；固定到 tag 或 commit hash 后 TTL 更长且不受后续提交影响，重要图建议固定版本。
4. 把「压缩 → 命名 → 上传 → 返回 Markdown 片段」写成脚本，注册成 MCP tool。之后 Agent 生成帖子的同时就能自动完成贴图。

## 踩坑点

- **国内直连不稳定**：`cdn.jsdelivr.net` 因备案问题间歇性不可用。备选前缀 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可做失败回退，或用 Cloudflare Workers 反代 + 自有域名兜底。
- **缓存无法主动刷新**：覆盖同名文件后，CDN 大概率还吐旧图。命名规则用「日期 + 内容 hash」，只增不改。
- **不要用 Git LFS**：jsDelivr 不分发 LFS 文件，链接拿到的是 403 或指针文本。
- **单文件上限 20MB**：超限拿不到链接。截图压缩到几百 KB 是常规操作，webp 优先。
- **token 权限最小化**：fine-grained PAT 只授权这一个仓库的 Contents 读写，别把全权限 classic token 写进脚本。
- **公开仓库无隐私**：截图先打码，密钥、内网域名、token 都不该出现在图里。
- **认清用途边界**：jsDelivr 的定位是开源项目文件分发。个人博客量级一般没问题，但拿它给商业站点扛主流量属于滥用，有仓库被限制的风险。

## 可复用建议

- 外链 URL 前缀收口到一个模板函数或环境变量，将来迁到对象存储或自有 CDN 只改一处。
- 上传脚本统一做压缩与幂等命名，重复执行不产生垃圾文件。
- 维护一份图源清单（本地路径 ↔ 外链），方便日后批量替换。
- 发布前 `curl -I` 校验返回 200 即可，不必引入更复杂的监控。

## 总结

这套方案的本质，是借开源基础设施的余量做个人存储分发。工程上要接受两个前提：没有 SLA，政策上有边界。它适合博客、笔记、Agent 产出物的低频配图，不适合任何「挂了会影响业务链路」的场景。把 URL 管理收口、上传流程自动化之后，它就是一个近乎零成本、可以安心用很久的个人图床——真到要迁移那天，改一行前缀就行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/8057bbdc5dfc47c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/5af9590f48c43869.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/49cefb47bdb5e16c.png)

