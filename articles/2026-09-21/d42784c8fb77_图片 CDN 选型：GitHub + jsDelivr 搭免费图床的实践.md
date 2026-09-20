---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 38313
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

在 OpenClaw 的日常使用里，图片需求比想象中多：Agent 生成的长文要配图、插件文档要截图、MCP 工具跑完要回传结果图。这些图如果贴本地路径，换台机器就断链；放对象存储，要么持续付费，要么免费额度说清就清。折腾过几个方案后，我把项目级静态资源收敛到了一条最省事的链路：**GitHub 仓库 + jsDelivr CDN**。

## 问题

目标很朴素：

- 零成本、长期有效，不依赖某个平台的免费额度
- URL 稳定，Markdown 里可以直接引用
- 上传过程能被脚本或 MCP 工具调用，最好一条命令出链接
- 可缓存，别让发布页每次都回源拉图

## 做法

**1. 建一个公开仓库。** 比如 `assets`，按 `images/{yyyy-mm}/` 分目录。仓库必须 public，私有仓库 jsDelivr 不回源，这是硬前提。

**2. 上传自动化。** 申请细粒度 PAT，只给这个仓库的 Contents 读写权限，然后走 GitHub Contents API：本地文件读出来转 base64，`PUT` 到 `/repos/{owner}/{repo}/contents/{path}`。这步我包成了一个十几行的 CLI，后来顺手接成 MCP 工具——Agent 生成图后直接调用，返回现成的 Markdown 片段，工作流里即产即贴。

**3. 引用走 jsDelivr 的 gh 源：**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/images/2025/06/a1b2c3-shot.webp
```

**4. 缓存管理。** 分支名引用会跟随仓库最新内容，但 CDN 有缓存延迟；改完图没生效，把域名换成 `purge.jsdelivr.net` 访问一次即可强制刷新。追求绝对稳定就用 tag 或 commit hash pin 住版本。

## 踩坑点

- **单文件 20MB 上限**：jsDelivr 对 gh 源有单文件大小限制。截图入库前务必压缩，我统一转 WebP，一般能压到原图三分之一。
- **公开即永久**：只要 push 过，内容就在 git 历史里，事后删除也未必干净。任何带隐私的图都不要进这个仓库，写成团队规矩。
- **大陆访问波动**：jsDelivr 在大陆的可用性时有起伏，备用域名 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可以做降级。对外发布的页面建议做 fallback 或本地兜底。
- **文件命名**：别用中文和空格，特殊字符在 URL 和缓存 key 上都会出问题。我用 `{8位hash}-{描述}.webp` 的格式，同名不同图也不会撞缓存。
- **别当网盘用**：GitHub 对仓库用途有社区规范，纯当大文件 CDN 刷流量不合适，控制在对项目资产合理的量级。

## 可复用建议

- 把上传动作封装成 MCP 工具或 CLI，入参本地路径、出参 markdown，Agent 就能自助完成"生成—上传—引用"闭环。
- 定一份命名与压缩规范（按月分目录、WebP、单图体积预算），写进仓库 README，让所有自动化遵守同一套约定。
- 维护一个 `index.json` 清单记录已入库资源，上传前先查重，避免 Agent 重复传同一张图。
- 如果发布场景对大陆可达性敏感，把这个方案当"作者侧图床"，发布时再同步一份到目标平台。

## 总结

GitHub + jsDelivr 不是新玩法，但对个人和中小项目来说，它把"图放哪"的成本压到接近零，而且整条链路——压缩、上传、命名、出链——都能被脚本和 Agent 接管。约束同样清楚：内容公开、体积要小、用量克制。把边界想清楚，它就是自动化写作流里最省心的一环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/33d37e9000f3682c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/4fb15f1f61b10123.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/99a101101941ed38.png)

