---
title: GitHub + jsDelivr 搭免费图床：给 Agent 流水线一个够用的图片 CDN
feedId: 43275
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

做 Agent 自动化时经常要处理图片：截图工具输出运行报告、MCP 插件生成的图表、bot 推送内容里的配图。这些图片需要一个可外链、稳定、免运维的 URL。对象存储当然正规，但要备案、要付费、要维护密钥；公共免费图床又随时可能关停。对个人和小团队，GitHub 仓库 + jsDelivr 是一个投入产出比很高的默认选项。

## 问题

GitHub 的 raw 链接（raw.githubusercontent.com）有两个短板：国内访问不稳定，且没有 CDN 缓存，当外链体验差。jsDelivr 补上了这一层：它会把 GitHub 仓库内容镜像到全球边缘节点，还提供主动刷新接口。核心思路一句话：**仓库只当存储层，jsDelivr 当分发层**。

## 做法

1. 建一个公开仓库，比如 `imgs`，按 `yyyy/mm/` 建目录，文件名用时间戳或 uuid 避免冲突和缓存混淆。
2. 手动用就 git push；自动化场景建议走 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，body 里带 base64 内容和 fine-grained token（只授该仓库的 contents:write）。一次 HTTP 请求完成上传，比依赖 git 环境更适合塞进 MCP 工具或 CI。
3. 外链格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。
   - 文档、报告等要长期不变的引用，`@` 后面钉 commit hash 或 tag，jsDelivr 永久缓存，天然防篡改；
   - 需要“最新版”的场景用 `@main`，但注意分支引用的缓存最长 12 小时才刷新。
4. 更新图片后想立刻生效，请求 `https://purge.jsdelivr.net/gh/...`（路径同上）主动清缓存。

## 踩坑点

- **国内访问不稳定**。jsDelivr 2022 年后国内节点时好时坏，不是彻底挂了，但别把它当面向国内读者的唯一来源。好在仓库本身就是源文件，随时可以换一层分发，迁移成本为零。
- **单文件上限 20MB**，仓库也别当网盘用，GitHub 对超大仓库同样会限制。上传前压缩，转 webp 通常能砍掉 60% 以上体积。
- **仓库必须公开**，所有图片任何人可见。含 token、内网地址的截图一律打码或不上传。
- **API 限流**：Contents API 匿名只有 60 次/小时，自动化务必带 token（5000 次/小时），且最小权限只授这一个仓库，别让 token 出现在 CI 日志里。
- **`@main` 的缓存陷阱**：同名路径覆盖更新后，CDN 和浏览器缓存都可能让你看到旧图。要么改文件名，要么主动 purge。

## 可复用建议

- 把“上传 → 返回 URL”封装成一个 MCP 工具：输入本地路径或 base64，输出 jsDelivr 链接，Agent 就能直接把图写进报告和消息里。
- 命名约定固定下来：日期目录 + uuid，配一个 `manifest.json` 索引方便反查历史图片。
- 引用策略二分：对外发布钉 commit，调试期用 `@main`。
- 存储与分发解耦的价值在于可替换——哪天 jsDelivr 不行了，仓库还在，换 Cloudflare 或自建镜像即可，业务代码里的上传逻辑不用动。

## 总结

这套方案的本质是“GitHub 做源站、jsDelivr 做边缘”：零成本、免运维、自带版本化。它的上限不高——不适合大流量站点和隐私内容，但对个人 Agent 流水线、博客配图、开源项目文档来说，是一个克制而可靠的工程选择。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/f6b493451b8298b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/13b79d379821d3ea.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/6414fc120f411d05.png)

