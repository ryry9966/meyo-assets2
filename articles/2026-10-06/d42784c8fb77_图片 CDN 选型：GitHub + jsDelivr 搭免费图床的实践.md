---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 40706
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

写插件文档、Agent 输出的报告、MCP 工具的演示帖，都绕不开贴图。图放哪里是个老问题：本地图路径别人看不到；对象存储要么要备案要么要钱；公共图床限制多、存活时间不可控。我的诉求很朴素：免费、链接长期稳定、能被脚本批量产出。

## 问题

直接用 `raw.githubusercontent.com` 链接有三个毛病：国内访问不稳、没有 CDN 缓存、并发下容易被限流。Agent 自动化场景里还会批量产出截图，逐张手动上传不现实。

## 做法

1. 建一个公开仓库，比如 `assets`，按 `/img/YYYYMM/` 分目录。
2. 引用走 jsDelivr：`https://cdn.jsdelivr.net/gh/<user>/assets@main/img/202506/xxx.webp`。
3. 上传自动化：用 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），body 里放 base64，三十来行脚本搞定。再包一层流水线：本地文件 → 压缩转 webp → 内容 hash 生成唯一文件名 → 上传 → 返回 jsDelivr 链接。我把它封成一个很薄的 MCP 工具，Agent 生成截图后自己调用，直接把外链写进 Markdown。
4. 鉴权用 fine-grained PAT，只授权这一个仓库、只给 Contents 读写，放环境变量注入。

## 踩坑点

- **缓存基本不可主动失效**：同名覆盖后，jsDelivr 的旧缓存可能存活很久。`purge.jsdelivr.net` 有 purge 接口，但只能当尽力而为。解法是从源头避免覆盖：文件名带 8 位内容 hash，文件天然不可变。
- **`@main` 有分钟级同步延迟**，刚传完立刻访问偶尔 404。自动化里加一次重试即可；要求严格的场合用 `@<commit-sha>` 锁定版本，代价是 URL 会变。
- **单文件上限 20MB**，仓库也别当垃圾桶，建议控制在 1GB 量级。上传前统一压缩，截图转 webp 基本都能压到几百 KB。
- 较大的文件走 Contents API 会失败，得用 git push 或 Git Data API——但图床场景本来就不该有大文件。
- **公开仓库等于全网可见**。带敏感信息的截图（面板地址、token、日志）一律打码或别传。
- jsDelivr 在国内的可访问性历史上波动过。保留 raw 域名作备胎，出问题改前缀就能切（`fastly.jsdelivr.net` 也可作备选）。

## 可复用建议

- 命名规范定死：`/{类型}/{年月}/{hash8}-{短描述}.webp`，文件名不用中文和空格。
- 把上传逻辑沉淀成 MCP 工具或插件，而不是每次让 Agent 现写代码——参数越少越稳。
- PAT 最小权限 + 环境变量注入，别进 git 历史，别进配置文件明文。
- 这套方案成本极低但不是零：仓库过大有被限制的风险。重要图片本地和仓库各留一份，把图床定位成“分发层”而不是“唯一存储”。

## 总结

这套方案不高级，但够用：GitHub 当存储，jsDelivr 当分发，脚本/MCP 当搬运工，内容 hash 解决缓存一致性。适合文档、博客、Agent 产出物这类低频写、高频读的场景。等图片量上来、要求私有化或国内访问质量有硬指标时，再迁对象存储也不迟——到时候换个域名前缀就行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/708e6427be82d646.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/15da9fa070205bbf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/1325fbfc909dab65.png)

