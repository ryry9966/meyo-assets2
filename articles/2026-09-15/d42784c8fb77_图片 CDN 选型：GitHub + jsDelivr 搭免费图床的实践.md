---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 37670
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

写技术帖、贴自动化流水线的产出截图、给 Agent 生成的报告配图，都绕不开同一个问题：图放哪。付费对象存储当然稳，但对低频写作场景属于过度配置。GitHub 仓库本身能存图，jsDelivr 在前面加了一层全球 CDN，组合起来就是一个零成本、可脚本化的图床。这套方案我在社区帖和几个 MCP 插件的文档里用了大半年，把经验和坑整理如下。

## 问题

直接用 `raw.githubusercontent.com` 做外链有三个问题：

1. 部分网络环境下访问慢或不稳定，读者打开帖子图挂了很伤体验；
2. 没有缓存加速，大图加载慢；
3. 对自动化流程不友好——需要一套稳定规则把"本地文件"变成"可引用的 URL"。

## 做法

整体链路：本地图片 → 压缩 → push 到公开仓库 → 通过 jsDelivr 域名引用。

1. **建专库**：新建一个公开仓库，比如 `openclaw-assets`，只放图，不和代码混。
2. **固定引用格式**：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch或tag>/<path>`
   例：`https://cdn.jsdelivr.net/gh/foo/openclaw-assets@main/2024/05/demo.png`
3. **上传自动化**：写十几行脚本（或封装成 MCP 工具），走 GitHub Contents API，base64 编码上传，返回拼好的 jsDelivr markdown 链接。我在 OpenClaw 里把它做成一个小技能，Agent 生成报告时直接调用，产出就是可粘贴的 `![](url)`。
4. **push 前压缩**：单图控制在几百 KB 内，别把原图直接推上去。

## 踩坑点

- **同名覆盖不生效**：jsDelivr 对 `@main` 这类分支引用有较长缓存，覆盖同名文件后 CDN 上还是旧图。要么用 purge 接口手动刷新（purge.jsdelivr.net），要么更省心——文件名带日期 + 短 hash，只增不改。
- **追新不如钉版本**：正式图建议引用 `@<commit 短 hash>` 或 release tag。分支引用在仓库改名、分支删除时会失效。
- **单文件 20MB 上限**：超限文件 jsDelivr 不服务；仓库也别无限膨胀，一年一两 GB 就该考虑归档旧图。
- **公开仓库无隐私**：任何猜到 URL 的人都能访问，截图注意打码，密钥类内容绝对不要走这条路。
- **可用性有波动**：jsDelivr 在不同地区可达性有起伏，不要当成有 SLA 的服务。

## 可复用建议

1. **URL 前缀做成配置项**：自动化工具里把 `cdn.jsdelivr.net/gh` 抽出来，将来换镜像或换对象存储，改一行配置整体迁移。
2. **Token 走环境变量**：GitHub PAT 只给目标仓库的权限，不要硬编码进脚本。
3. **保留原图**：CDN 链接只是衍生物，git 历史里的原图才是资产本体的备份。
4. **封装通用化**：MCP 工具输入同时支持本地路径和 URL，输出同时给 markdown 和 HTML 两种格式，其他场景直接复用。

## 总结

GitHub + jsDelivr 适合个人和社区场景的轻量图床：零成本、版本化、天然可脚本化，和 OpenClaw 的自动化流程契合度很高。但它的本质是"尽力而为"的静态文件分发，不是图片服务。明确这一点后，做好文件命名规范、压缩入库、可切换的 URL 前缀，它就能长期稳定地用下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/88b076434a732827.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/c52fa7ca797aa67a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/cdd5c5b479c23009.png)

