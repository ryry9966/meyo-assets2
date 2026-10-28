---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 免费图床的工程化用法
feedId: 42806
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

写博客、维护插件文档、给 Agent 的 README 配图，都需要一个能稳定外链的图床。常见选项里，对象存储要备案实名，商用 CDN 要花钱，自建 Nginx 要机器。对个人开发者来说，"GitHub 仓库当存储、jsDelivr 当 CDN" 是成本最低的组合：仓库即备份，CDN 即加速，全程零费用。

## 问题

直接外链 `raw.githubusercontent.com` 有几个硬伤：没有 CDN 边缘缓存，部分地区访问不稳定，且 raw 链接的缓存策略不友好，重复加载慢。我们需要一个对热链友好、有全球边缘节点的分发层。

## 做法

1. 建一个 public 仓库，比如 `pics`。
2. 上传图片后，通过 jsDelivr 引用：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch-or-tag>/<path>
```

例如 `https://cdn.jsdelivr.net/gh/alice/pics@main/2024/cover.webp`。

3. 缓存策略二选一：
   - 引用 `@main`：文件更新后约 12 小时内生效，急用可手动刷缓存：`https://purge.jsdelivr.net/gh/<user>/<repo>@main/<path>`；
   - 打 tag/release 引用：URL 不可变、缓存永久，适合文档里长期引用。
4. 自动化：写一个几十行的脚本（或挂成 OpenClaw 的 skill / MCP 工具），输入本地图片路径，调 GitHub Contents API 上传，返回 CDN 链接。核心就两步：base64 编码 → PUT `https://api.github.com/repos/<user>/<repo>/contents/<path>`。配上剪贴板监听，截图即出链接。

## 踩坑点

- **单文件 20MB 上限**：jsDelivr 不分发超过 20MB 的 GitHub 文件，大图也不该进 git。上传前先压一遍，WebP 通常能砍 60% 以上体积。
- **文件名规范**：中文文件名、空格、`#`、`?` 会在 URL 里变成编码灾难，还可能缓存错乱。统一用 `日期-短哈希.webp` 这类命名。
- **别把仓库当网盘**：public 仓库任何人可见可遍历，敏感截图千万别传；且 git 历史只增不减，图片删了也还在历史里。
- **国内可用性**：jsDelivr 主域近年在大陆时好时坏，属已知问题。预留 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 等备用域，前端做一层域名降级，别把主域写死。
- **同 URL 覆盖更新**：覆盖 `@main` 下同名文件后，边缘可能仍是旧图。要么 purge，要么直接换文件名——实践中后者更省心。

## 可复用建议

- 文档里长期引用的图走 tag，README 里常换的走 branch，两层缓存语义分开。
- GitHub token 用 fine-grained 权限，只授予这一个仓库的 Contents 写权限，别用全量 classic token。
- 把"上传→拿链接"封装成插件后，可以进一步让 Agent 在写文档时自动完成配图插入，这是这套方案对自动化用户最大的价值。
- 仓库是唯一真源，CDN 只是缓存层。就算 jsDelivr 哪天不可用，切回 raw 链接或其他镜像即可，数据不丢。

## 总结

这套方案的本质是用 GitHub 做存储层、jsDelivr 做分发层，成本为零、数据自有。它适合个人博客、项目文档、插件资源这类中小规模场景；不适合大流量商用分发或大量二进制投放——那既违反公平使用原则，也撑不住 SLA。配好命名规范和自动化脚本后，"贴图出链接"的体验不输付费图床，值得作为个人技术栈的默认选项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/11c8162f9074e72f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/ecb58dcb9a5d9d02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/01800fac7ad6019d.png)

