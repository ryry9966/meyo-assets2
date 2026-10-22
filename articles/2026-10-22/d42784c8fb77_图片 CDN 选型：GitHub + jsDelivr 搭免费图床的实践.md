---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 42287
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

写文档、发技术帖、让 Agent 输出带图报告，图床是绕不开的基础设施。对象存储（OSS/S3/R2）稳定，但要么绑域名备案，要么算流量费；市面上的免费图床服务来去匆匆，链接失效风险高。对个人和小团队来说，GitHub 仓库当存储、jsDelivr 当分发层的组合性价比很高：零成本、版本可控、天然纳入 git 工作流。最近我把这套方案做成了自动化流程，把踩过的坑记录一下。

## 做法

1. **建仓库**：新建一个公开仓库如 `cdn-assets`，按 `img/YYYY/MM` 分目录，命名用「日期 + 短哈希」避免冲突。
2. **上传**：手动 `git push` 即可；想省事用 PicGo/PicList 的 GitHub 插件，填仓库名、分支和 URL 前缀就行。
3. **外链格式**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<版本>/<路径>`。`@main` 引用分支最新，`@1.0.0` 或 commit hash 引用固定版本。
4. **自动化**：我写了个 20 行的小脚本（也可以封装成 MCP 工具给 Agent 调用）：压缩图片 → 哈希命名 → push → 返回 CDN URL。Agent 生成博客、报告时直接产出可外链地址，省掉手动上传这一步，这是这套方案对自动化用户最有价值的地方。

## 踩坑点

- **缓存不失效**：jsDelivr 缓存很激进，同名覆盖后旧图可能长期不更新。引用一律带 commit hash 或 release tag，别裸用 `@main`。
- **大陆访问波动**：jsDelivr 主域在大陆时好时坏（历史上因备案问题多次故障）。社区镜像 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`、`testingcf.jsdelivr.net` 可作备用前缀，上传时同时记录多个 URL，方便一键切换。
- **单文件与流量限制**：jsDelivr 拒绝大于 20MB 的文件；单仓库流量异常大可能触发 abuse 机制被封。原图先压缩，长图、大动图别放这。
- **公开仓库**：push 上去的内容全部公开。敏感截图、带 token 的日志图务必先脱敏。
- **仓库膨胀**：大量二进制文件会让 clone 越来越慢。单仓库控制在几千张图以内，够用就归档、换新仓库。

## 可复用建议

- **引用带版本**：tag/hash 引用让缓存行为可预期，也方便回滚。
- **本地留清单**：一份 JSON 记录原始文件名、CDN URL、上传时间，迁移和替换不抓瞎。
- **工具化**：把「压缩 + 命名 + push + 返回 URL」封装成脚本或 MCP 工具，Agent 一句话完成配图上传，接入工作流后基本无感。
- **留退路**：免费方案别当唯一依赖。清单在手，随时可批量换前缀，或整体迁到 R2/OSS。

## 总结

GitHub + jsDelivr 不是最稳的方案，但在成本、可控性、自动化友好度上是均衡的选择。核心心法一句话：**存储用 git，分发用 CDN，引用带版本，清单本地留**。与其纠结「哪个图床能永远免费」，不如把它做成 Agent 工作流里的一个小工具，坏了能切、迁了能跑，这才是工程上站得住的做法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/c072428ee9583905.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/208b2ea9f103f725.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/e13a8a54bf8e0d23.png)

