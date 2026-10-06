---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 40658
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

写技术博客、给插件补文档、或者让 Agent 自动产出带图报告时，图床是个绕不开的小问题。付费对象存储要充值还要考虑备案，免费图床跑路风险高，GitHub raw 域名国内直连又慢。目前成本最低的组合就是：**GitHub 公开仓库当存储，jsDelivr 当分发层**。它不是完美方案，但作为个人级 CDN 够用，下面是实践中验证过的做法和坑。

## 做法

1. **建仓库**：新建一个 public 仓库（比如 `assets`）。注意私有仓库 jsDelivr 不认，必须公开。
2. **定规范**：图片统一放 `img/` 目录，命名用 `日期-内容-短hash.png`（如 `20250601-arch-3f9c.png`），避免重名覆盖。
3. **拼外链**：格式为 `https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag或分支>/<路径>`。建议锁 tag 或 commit hash，缓存命中更稳定；用 `@main` 也能用，但更新后缓存刷新不可预期。
4. **自动化上传**：本地先写个 20 行脚本：压缩 → git push → 拼外链 → 复制到剪贴板。更进一步的玩法是封成一个 MCP tool，输入本地路径或图片 URL，输出 jsDelivr 外链，同 hash 幂等不重复提交。这样在 OpenClaw 流水线里，Agent 写文档时自己调工具传图、拿回链接，全程不需要人碰 git。
5. **压缩前置**：提交前用 squoosh 或 tinypng 压一道，PNG 截图压完通常只剩 1/3 体积，加载体验差距明显。

## 踩坑点

1. **缓存不可控**：jsDelivr 边缘缓存周期很长，文件删了外链可能还活着。想换图就换文件名，不要原地覆盖同名文件。
2. **仓库只增不减**：git 删图不会缩小仓库体积，历史里永远在。别把仓库当相册，垃圾图攒多了将来迁移成本很高。
3. **国内解析波动**：`cdn.jsdelivr.net` 这两年解析并不稳（ICP 相关原因），打不开时先试 `fastly.jsdelivr.net` / `gcore.jsdelivr.net`。所以 Markdown 里务必把域名做成可配置项，别写死。
4. **硬限制**：单文件 ≤ 20MB，且不支持 Git LFS，大图和视频别往这儿塞。
5. **合规边界**：jsDelivr 官方条款并不鼓励纯图床用法，属于灰色地带。个人博客无所谓，生产关键路径别压在上面。

## 可复用建议

- **域名抽象**：静态站点里配一个 `IMG_CDN_PREFIX` 环境变量，哪天要迁到别家，改一行重构建即可，文章里的图片链接不用挨个改。
- **双写兜底**：本地保留原图目录，jsDelivr 只是分发层，不是备份。仓库 + CDN 挂了，你有退路。
- **工具化优先**：如果图片由 Agent/自动化流程产出，把「压缩 → 命名 → push → 返回外链」做成一个幂等 CLI 或 MCP 工具。Agent 调用固定接口比人肉操作可靠得多，也方便日后整体换后端。

## 总结

GitHub + jsDelivr 适合个人博客、文档站、低频分享这类场景：成本几乎为零，代价是缓存不可控、国内稳定性一般、合规上属于灰色地带。工程上的结论一句话：**把它当「顺手的 CDN」，而不是「存储底座」**——域名可配置、本地有原图、上传已自动化，这套组合在个人项目里能稳稳跑很久，出问题时你永远有退路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/aaf167874c56439c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/da7cdcbe8338dbfc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/c410f85b0144fef6.png)

