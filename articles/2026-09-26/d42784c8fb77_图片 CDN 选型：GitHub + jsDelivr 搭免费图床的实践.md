---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 39160
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

写技术文档、发社区帖、给 agent 输出配图，都绕不开图片托管。对象存储要备案和计费，各类“免费图床”随时可能关停，图挂了比代码挂了更尴尬。我用了两年多的方案是 GitHub 仓库 + jsDelivr：仓库当存储，jsDelivr 当 CDN，零成本、链路透明、天然可自动化。

## 问题

原生 GitHub 方案有两个不爽：

- `raw.githubusercontent.com` 在国内不稳定，也没有任何图片优化；
- 直接拿仓库当图床，文件一多 clone 变慢，引用链接散落在各篇文档里，迁移成本极高。

目标很朴素：一张图 push 上去，拿到一个全球可访问、带缓存、可脚本化生成的 URL，且未来要迁移时不必逐篇改文。

## 做法

1. 建一个 public 仓库，例如 `assets`，目录按 `img/YYYY/MM/` 组织。
2. 上传前本地压缩：截图走 pngquant/mozjpeg，动图尽量转 WebM；单文件控制在几百 KB——jsDelivr 对超过 20MB 的文件直接不代理。
3. 引用统一走模板：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/img/2025/06/xxx.png`
   日常用 `@main`，需要长期稳定引用时打 tag 或用 commit 短 hash。
4. 自动化：把“压缩 → commit → push → 拼 URL → 写回 Markdown”封装成脚本，并在 OpenClaw 里注册成一个 MCP 工具（如 `upload_image(local_path) -> cdn_url`），让 agent 写文档时直接调用，而不是人肉拖拽。
5. 文档中不硬编码域名，统一由配置项 `cdn_base` 渲染，日后迁移只改一处。

## 踩坑点

- **国内可用性波动**：jsDelivr 2022 年前后出现过解析异常，至今时好时坏。它适合博客/文档/笔记场景，不要作为面向国内生产服务的唯一依赖。兜底：保留 raw 链接生成能力，或在配置层随时切自建/对象存储。
- **缓存延迟**：`@main` 引用更新后有数小时到一天的缓存延迟。急着生效用 commit hash 锁版本，或向 purge.jsdelivr.net 提交刷新。
- **仓库膨胀**：截图和 GIF 是元凶。我的原则：录屏不进图床，超 500KB 先压缩，仓库超 1GB 就拆新仓。
- **隐私与历史**：public 仓库的 commit 历史删不干净，截图里带 token、内网地址的千万别传。
- **命名**：全用 kebab-case + 短哈希，中文文件名和空格会带来 URL 编码坑。

## 可复用建议

- 图床操作收敛为一个 MCP 工具/脚本，参数只有 `local_path`，返回 `cdn_url`，团队和 agent 都能复用。
- 加一条 CI 检查：对文章中所有图片 URL 发 HEAD 请求，非 200 即 fail，防“发文即裂图”。
- URL 模板集中管理，Markdown 里不要散落完整链接。

## 总结

GitHub + jsDelivr 不是最高级的方案，但工程上最可控：存储在 git、链路可脚本化、迁移成本接近零。配合 OpenClaw 把上传动作 agent 化之后，写文档的摩擦基本消失。唯一要记住的是——它适合个人与社区场景，生产流量请备好 Plan B。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/e6732416f17b62fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a9dfe1a35f597b38.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/240a928b587bd38f.png)

