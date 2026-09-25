---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与踩坑
feedId: 38964
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在社区写 Agent 实践帖、MCP 插件文档，截图和架构图是刚需。图床这条路我踩过不少：SM.MS 有限额、微博图床加防盗链、一些免费床直接跑路。后来迁到 GitHub 仓库 + jsDelivr CDN，稳定用了两年，把做法和坑整理出来。

## 问题

需求其实很朴素：

- 免费、链接不失效、可外链进 Markdown
- 能被脚本或 Agent 自动调用，上传后直接返回 URL
- 对国内读者的可访问性不能太差（这条最难）

GitHub 仓库本身可以当存储，但 `raw.githubusercontent.com` 不是 CDN，国内访问看运气。jsDelivr 的 `cdn.jsdelivr.net/gh/` 正好补上分发这一层。

## 做法

1. **建公开仓库**，如 `imgs`，按年份或项目分目录。
2. **上传即 push**：文件名用 `日期-短哈希.png`，避免重名和中文 URL 编码问题。
3. **外链格式**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。`ref` 可以是分支、tag 或 commit hash——定稿文章建议用 commit hash，链接不可变；还在改的文档用 `@main` 方便更新。
4. **自动化（重点）**：写一个十几行的 shell 脚本 `imgup`：接收本地路径或剪贴板截图 → 生成哈希文件名 → commit & push → 输出 CDN URL 并复制到剪贴板。注册成 MCP 工具或 skill 后，在对话里说一句“把这张图传图床”，Agent 调脚本拿 URL 直接插入正文。
5. 更新同名文件后，主动请求 `https://purge.jsdelivr.net/gh/...` 清缓存，或干脆换文件名。

## 踩坑点

- **单文件 20MB 上限**，动图基本超，长 GIF 转 MP4 或压帧。
- **国内可达性会波动**，jsDelivr 这几年经历过节点调整。重要文章我保留 GitHub raw 链接做备用源。
- **公开仓库 = 图片公开**。带密钥、内网地址的截图先打码；自动化流程里建议加一步本地敏感信息检查。
- **缓存顽固**：改图不换名又不 purge，读者看到的永远是旧图。
- **别当无限网盘**。jsDelivr 定位是开源项目静态资源，小体量文档图床没问题，流量大了会被限速，届时该上正经 CDN 就上。
- **PAT 用 fine-grained token**，只授权这一个仓库的 Contents 读写，脚本泄漏时损失可控。

## 可复用建议

- 命名统一 `YYYYMMDD-<hash>.<ext>`，防冲突且天然可排序。
- 仓库根维护一个 `index.json` 记录已上传文件的 URL 映射，Agent 写作时可查询，避免重复上传。
- 入库前统一压缩（pngquant / mozjpeg），仓库小，clone 和 push 都快。
- 要彻底可控可换源：仓库不动，前面挂自己的域名 + Worker 回源，jsDelivr 降级为备用。

## 总结

这套方案的性价比在于：存储用 GitHub（免费、可靠、自带版本历史），分发用 jsDelivr（免费 CDN），自动化只需一个 git 脚本。它不是生产级 CDN，但对社区写作、文档配图、Agent 生成内容的图床需求，是投入产出比最高的起步方案。先跑通，流量或合规要求上来了再演进。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/f9af3254bdbb2415.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/d85da1c4acd2b1b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/76f298c42396eb79.png)

