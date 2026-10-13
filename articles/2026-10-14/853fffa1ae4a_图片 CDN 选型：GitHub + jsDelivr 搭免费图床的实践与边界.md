---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与边界
feedId: 41495
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

在 OpenClaw-CN 写帖、写文档、跑 agent 自动化输出时，贴图是高频需求。截图、流程图、agent 运行结果都要外链。个人项目用对象存储显得重：要实名、要域名备案、要计费；而大多数人的图早已在 GitHub 仓库里。这篇帖记录我基于 GitHub 仓库 + jsDelivr 搭图床的完整做法，以及踩过的坑。

## 问题

直接用 `raw.githubusercontent.com` 做图链有几个实际痛点：

1. **无 CDN 缓存**，每次回源，加载慢；
2. 部分渲染器和预览环境对 raw 域名的直链加载不稳定；
3. 没有 `Cache-Control` 和图片处理能力，重复流量全打到 GitHub。

我们需要的是：稳定 URL、有缓存、可自动化上传、零成本。jsDelivr 的 `/gh/` 源恰好补上了这几块。

## 做法

**1. 建专用公开仓库**

单独建一个 `assets` 仓库放图，不要混进代码仓库。按 `images/2025/06/20250610-a3f2.png` 的日期+短哈希规则命名。

**2. 拼接 CDN 链接**

```
https://cdn.jsdelivr.net/gh/<user>/assets@main/images/2025/06/xxx.png
```

日常发帖用 `@main` 就够；正式项目的关键资源建议锁到 release tag 或 commit hash，保证链接不可变。

**3. 自动化上传**

手动流程是：压缩 → push → 拼链接 → 改 markdown。写了个 20 行的脚本固化它，OpenClaw 用户更直接的做法是封装成一个 MCP tool 或 shell skill，契约很简单：输入本地图片路径，输出可用的 CDN URL，中间自动做压缩和链接重写。这样 agent 在生成 markdown 时可以自己完成"贴图"动作，人不用碰。

**4. 缓存刷新**

同名覆盖更新后，把 URL 里的 `cdn.jsdelivr.net` 换成 `purge.jsdelivr.net` 访问一次即可清缓存。但我更推荐唯一文件名，从源头绕开这个问题。

## 踩坑点

- **国内访问不稳定**。jsDelivr 的 ICP 备案 2022 年后出了状况，国内可达性时好时坏。上生产前一定先实测自己的用户群，重要项目别单点依赖，备选是 Cloudflare Pages 或自建反代。
- **缓存残留**。同名替换后旧图可能残留在边缘节点很久，配合 purge 或改名解决。
- **公开即公开**。仓库是 public 的，截图里的 token、内网地址、客户信息都会被同步，上传前必须脱敏。这个交给 agent 自动化时要特别小心。
- **别当大文件床**。jsDelivr 明确不适合分发视频和大文件，滥用会被封源。单图控制在几百 KB。
- **仓库体积**。图多了 clone 会变慢，定期归档，或干脆用裸 push 不 clone 本地副本。

## 可复用建议

1. **定位要清晰**：GitHub 仓库是源，jsDelivr 是缓存层。原图自己留一份，图床挂了随时可迁。
2. **域名收敛**：所有链接走一个常量或配置项，将来迁到对象存储时批量替换成本低。
3. **命名即缓存策略**：`日期-哈希` 的唯一文件名比事后 purge 靠谱得多。
4. **工具契约化**：上传能力做成幂等的 tool（path in, URL out），人和 agent 共用同一入口，才是长期可维护的形态。
5. **定期抽检**：写个 cron 抽样 HEAD 请求历史 URL，及时发现可达性劣化。

## 总结

GitHub + jsDelivr 是个人博客、社区帖、agent 文档产出的够用方案：零成本、有缓存、能自动化。但它的定位是"免费缓存"，不是"高可用存储"——国内可达性和滥用封禁是两颗定时炸弹。工程上的正确姿势是：仓库为源、CDN 为缓存、域名为抽象，随时可换后端。如果你的项目开始有商业流量，直接上对象存储，别在免费方案上做过度优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/b8c74204f132179b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/3d53c3daf62e92df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/055cede610156aef.png)

