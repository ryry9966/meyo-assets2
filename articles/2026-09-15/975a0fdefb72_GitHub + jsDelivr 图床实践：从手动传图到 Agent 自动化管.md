---
title: GitHub + jsDelivr 图床实践：从手动传图到 Agent 自动化管线
feedId: 37704
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

个人博客、文档站，以及让 Agent 生成带截图的周报和复盘，都会撞上同一个问题：图片放哪。对象存储按量计费还要备案，第三方免费图床说挂就挂。GitHub 仓库 + jsDelivr 是零成本方案里最经久的一个：仓库当存储层，jsDelivr 当缓存分发层，分工清晰。

## 问题

直接引用 `raw.githubusercontent.com` 不是正经做法：缓存策略不适合图片，国内可达性差，也没有压缩空间。结果就是首屏一张截图几百 KB 到几 MB，页面被拖垮。要解决的其实是两件事：一个稳定的引用域名，一条能自动化、最好能让 Agent 代劳的上传链路。

## 做法

1. 建一个公开仓库，比如 `images`，按 `yyyy/mm` 分目录，控制单仓库规模。
2. 上传前统一处理：压缩一遍、转 WebP，命名用 `yyyyMMdd-HHmmss-短hash`，天然去重。
3. 引用规则固定为 `https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag>/<path>`。写草稿用 `@main`，页面正式发布前打 tag，换成 `@vx.y`。
4. 改图后如果引用的是 `@main`，请求一次 `https://purge.jsdelivr.net/gh/...` 刷缓存；最稳的还是靠 tag 切换版本。
5. 自动化：轻量用法是 PicGo 或自写脚本走 GitHub API；在 OpenClaw 里可以用 MCP 的 git/GitHub 工具，把「收到截图 → 压缩 → commit/push → 回填 URL」串成一条管线。PAT 用 fine-grained token，只授这个仓库的 Contents 读写。

## 踩坑点

- jsDelivr 有单文件 20MB 上限，长 GIF 会直接失败，先压缩再传。
- 单仓库文件数建议控制在几千以内，文件太多 jsDelivr 索引会出问题，跑久了按年份拆仓。
- `@main` 有缓存延迟，push 完不等于立刻生效，别在自动化脚本里假设即时可见。
- 仓库必须公开，jsDelivr 才读得到。所以截图里带 token、内网地址就是事故——我在管线里加了一步正则扫描，命中就拦截。
- 近两年国内对 `cdn.jsdelivr.net` 可达性有波动，重要页面要有降级预案（自建反代或备用源）。

## 可复用建议

- URL 生成写成函数，不要手拼：域名、tag、路径集中管理，将来换源只改一处。
- git 仓库本身就是异地备份，本地保留同名镜像目录即可。
- 把它定位成「博客与文档的图床」，不要当生产级 CDN 用——高流量分发不符合 GitHub 的合理使用边界，量级上来老老实实迁对象存储。

## 总结

这套方案的价值不在「免费」，而在可控和可自动化：存储是自己的仓库，管线是自己的代码，Agent 接入只需要一个受限 token。边界同样清晰：国内可达性波动、公开性约束、20MB 与文件数上限。用在个人博客、文档站、Agent 报告这类低流量场景，可以稳定跑很多年；越界使用才会撞上它的天花板。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/b904007828261871.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/887701b58719b8b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/f0b788b5b14b28ed.png)

