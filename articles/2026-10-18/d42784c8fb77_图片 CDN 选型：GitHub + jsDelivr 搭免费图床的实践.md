---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 41923
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

写博客、文档、Agent 输出的报告，经常要贴图。方案无非几类：对象存储（要钱、绑域名）、第三方免费图床（随时跑路）、自建（要服务器）。我的诉求很简单：零成本、链接稳定、能被脚本和 Agent 自动调用。最后落在 GitHub 仓库当存储 + jsDelivr 当 CDN 的组合上，用了一年多，记录一下做法和坑。

## 原理与做法

jsDelivr 是公共 CDN，能直接镜像 GitHub 公开仓库的内容。图片推上去，就自动获得 CDN 链接，不用自己管缓存和节点。

步骤：

1. 新建一个公开仓库，如 `assets`，单独放图，别和代码混。
2. 本地 clone，按 `images/2025/06/` 这类日期目录组织。
3. 文件名用内容哈希或时间戳，纯 ASCII：`a3f9c2.png`。中文文件名和空格是 URL 编码事故的重灾区。
4. push 之后链接格式为：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<版本>/<路径>`
   `<版本>` 可以是分支名、tag 或 commit 短 hash。**建议用 tag 或 commit hash**，链接不可变，缓存行为才可预期。
5. 自动化：写一个几十行的 shell 脚本（或封装成 MCP 工具），流程是「压缩 → 重命名 → commit/push → 输出 Markdown 片段并复制到剪贴板」。Agent 侧只需调这一个工具，贴图全流程闭环。压缩用 pngquant / mozjpeg 或统一转 webp，体积通常能砍 60% 以上。

## 踩坑点

- **用分支名当版本**：`@main` 的链接内容会变，jsDelivr 对它的缓存策略和静态版本不同，更新后经常拿到旧图。要么改文件名，要么打 tag，要么主动刷缓存：`https://purge.jsdelivr.net/gh/...`。
- **替换同名文件**：push 成功不代表生效，边缘节点可能还挂着旧缓存。养成「文件名即版本」的习惯，比刷缓存省心。
- **大文件**：单文件建议控制在 20MB 内，实际上 1MB 以上的图就该压缩。git 历史里的二进制删不掉，删文件只增不减仓库体积，别把它当无限网盘。
- **国内访问波动**：jsDelivr 过去有过被降级的时期，不能当 SLA 服务用。博客场景可以接受，但重要的图建议保留 raw.githubusercontent.com 兜底链接，或双写一份到对象存储。
- **仅支持公开仓库**：私有仓库的内容它不提供，别图方便放敏感截图。

## 可复用建议

- 命名 hash 化、路径日期化，链接天然不可变，缓存问题消失一大半。
- 上传脚本的返回值直接是 Markdown 格式，Agent 拿到即可插入文档，链路零人工。
- 把「压缩、上传」做成幂等的小命令，故障重跑不产生重复文件。
- 流量预期想清楚：适合个人博客、文档站、Agent 产出物这类低到中流量场景；图要面向大量终端用户，就老实上对象存储 + 正规 CDN。

## 总结

这套方案的本质是「用 git 管图，用公共 CDN 分发」：成本为零，可控性够用，代价是守好命名规范和体积红线。一年下来没换方案的必要，也没花一分钱。它不是生产级方案，但对社区里写文档、跑 Agent、做插件的日常场景，是性价比最高的选择之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/4c4c25168ece11ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/8bdb618f84a8d611.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/cab40d041d8b25b6.png)

