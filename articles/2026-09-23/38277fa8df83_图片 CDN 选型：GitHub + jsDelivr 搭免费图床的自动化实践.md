---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 38547
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

在 OpenClaw 的日常使用里，图出现的场景比想象中多：Agent 生成的架构图、自动化任务的截图证据、MCP 工具产出的可视化结果、社区帖子里的示意图。图片放哪是个长期被糊弄过去的问题——本地路径发不出去，对象存储要实名和付费，第三方图床说倒就倒。

## 问题

需求其实很朴素：

1. 稳定外链，Markdown 里直接引用；
2. 零成本或接近零成本；
3. 上传流程能被 Agent / 脚本自动化，最好一条命令搞定；
4. 国内访问不能太糟。

GitHub 仓库 + jsDelivr 的组合基本满足前三条，第四条要打折扣，后面细说。

## 做法

1. **建仓库**：新建一个公开仓库（比如 `imgs`），结构随意，建议按 `YYYY/MM` 分目录，别和项目代码混在一起。
2. **上传**：手动场景直接 git push；自动化场景用 Contents API，一次 `PUT /repos/{owner}/{repo}/contents/{path}`，body 塞 base64，带 PAT 即可，不需要完整 git 流程，对 Agent 更友好。
3. **引用**：`https://cdn.jsdelivr.net/gh/<user>/imgs@main/2025/06/demo.png`。
4. **封装**：把流程包成 MCP 工具或 skill——输入本地图片路径 → 压缩（转 webp，单图控制在 500KB 内）→ 内容 hash 命名去重 → API 上传 → 返回 jsDelivr 链接。对 Agent 来说就是一个普通工具调用。

## 踩坑点

- **缓存**：jsDelivr 缓存很凶。`@main` 这类分支引用约 12 小时缓存，改图不换名基本不生效；`@<commit-sha>` 或 tag 链接内容不可变、永久缓存。实践建议：草稿期用 `@main`，定稿引用换 commit 链接；紧急刷新走 `purge.jsdelivr.net/gh/...`。
- **国内访问**：2022 年后 `cdn.jsdelivr.net` 主域名在大陆时好时坏，`fastly.jsdelivr.net` 和 `gcore.jsdelivr.net` 通常更稳。重要文档别赌单一域名，写个十几行的测速脚本，自动选可用前缀。
- **滥用风险**：jsDelivr 定位是开源 CDN，纯图床用法一直在灰色地带，历史上出现过整个账号被拒服的先例。别把它当唯一存储，原图务必留本地或再备一份。
- **大小与隐私**：单文件超过 20MB jsDelivr 不服务；公开仓库即公开内容，含敏感信息的截图不要传。

## 可复用建议

- 图片先压缩再上传，webp/avif 优先，这一步对加载速度的收益最大。
- 用内容 hash 做文件名，天然去重 + 天然缓存友好。
- PAT 只开 contents 写权限，放环境变量，别进代码库。
- 维护一个 manifest（文件名 → URL → 用途），Agent 检索图片和后续清理都方便。
- 给迁移留后路：把「上传」和「拼 URL」写成两层，将来换 R2 / OSS 只改一层。

## 总结

这套方案的价值不在「免费」，而在把图片资产变成 git 仓库里的普通文件——可版本化、可脚本化、可被 Agent 当作普通工具操作。代价是国内访问不稳定和平台依赖风险。个人博客、技术文档、Agent 产出的展示场景完全够用；商用高流量场景请直接上对象存储，别省这个钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/cebe534bb1d85b05.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/b6e4ab204d94d4cd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/d84d08afa4d67c9b.png)

