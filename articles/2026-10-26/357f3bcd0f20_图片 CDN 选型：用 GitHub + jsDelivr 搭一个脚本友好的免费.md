---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个脚本友好的免费图床
feedId: 42591
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

做 Agent、插件或自动化流水线时，经常要往 Markdown 文档、Bot 回复、博客里塞图：Agent 跑完任务的截图、插件示意、每日产物。可选方案不少——对象存储稳定但要钱要配置，各类免费图床随时跑路，直接引用 `raw.githubusercontent.com` 在国内访问又经常拉胯。折中下来比较务实的组合是：图片进 GitHub 公开仓库，走 jsDelivr 的 gh CDN 分发。免费、天然带版本、全流程可以用脚本操作，这三点对自动化场景很关键。

## 做法

1. **建仓库**：新建一个公开仓库如 `img-cdn`，专门放图，别和代码混。
2. **上传**：手动就 `git push`；自动化走 GitHub Contents API，对 `/repos/{owner}/{repo}/contents/{path}` 发一个 PUT，body 里带 base64 内容即可。token 只授 repo 权限，最小化。
3. **引用地址**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>`。version 用 commit 短 hash 或 release tag，**别直接用分支名**，原因见下。
4. **命名规则**：文件名用内容 hash（如 sha1 前 12 位）+ 扩展名，天然去重。
5. **接入 Agent**：把上传逻辑封成一个 MCP tool 或插件命令——输入本地图片路径，输出 CDN URL。Agent 截完图、生成完图表，一行调用拿外链。

## 踩坑点

- **分支引用有缓存坑**。`@main` 的响应同样会被 jsDelivr 缓存，图换了外链还是旧图。要么手动请求 `purge.jsdelivr.net` 刷新，要么用 hash 文件名 + 固定 tag——内容变了 URL 就变，直接绕开缓存问题，后者更省心。
- **jsDelivr 国内访问时好时坏**。它前些年经历过 ICP 与自定义域名风波，目前不承诺稳定。面向国内读者时备一条 `raw.githubusercontent.com` 回退链接，或前端 `onerror` 切换。
- **单文件 20MB 上限**，超出 jsDelivr 不服务。仓库别养太大，几百 MB 内安全；GitHub 对纯图仓库属于灰色地带，别当网盘用。
- **仓库必须公开**，私有仓库 jsDelivr 不回源。含敏感信息的截图一律不传。
- **文件名避免中文和空格**，自动化场景下 URL 编码问题能浪费你半小时。

## 可复用建议

- **幂等是这套方案最大的红利**：hash 命名 + Contents API，同一张图重复上传得到的 URL 完全一致，Agent 重试、并发都不会产生脏数据。
- 把上传逻辑做成独立 MCP tool，输入输出都是纯字符串，任何会话都能复用，也是练手 MCP 的好例子——比 echo tool 有真实价值。
- 仓库里维护一个 `index.json`，记录 hash 到原始文件名的映射，删图和审计有据可查。
- 如果图片会被频繁覆盖更新（比如每日重绘的报表图），这套方案不合适，老老实实上对象存储。

## 总结

GitHub + jsDelivr 不是完美的图床，它是“够用的图床”：零成本、版本化、可脚本化，特别适合 Agent 输出物、文档配图这类低频写、高频读的场景。记住三条——hash 命名、tag 引用、备好回退链路——剩下的就是把它包成一个顺手的工具调用，让 Agent 自己用起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/e982500bd9e857b8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/8b70312513523bf6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/6ebfbe78c5788c62.png)

