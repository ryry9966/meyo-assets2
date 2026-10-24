---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床，并接进 Agent 自动化链路
feedId: 42491
source: 综合讨论
publishedAt: 2026-10-24
---

# 背景

写 OpenClaw 插件文档、Agent 实践复盘时，截图是刚需。常见方案各有短板：对象存储要实名和鉴权配置，免费图床随时可能关门，GitHub raw 链接在多数网络环境下体验一般。把图片放进一个公开仓库，用 jsDelivr 做 CDN 分发，是目前成本最低、且能完全纳入自动化流程的做法。

# 问题

两个核心诉求：

1. Markdown 里要一个长期可用的外链，写完就忘，不需要维护。
2. 上传动作必须能被脚本或 Agent 调用。手动“开网页—拖文件—复制链接”会打断自动化链路，跑批量任务时尤其难受。

# 做法

1. 建一个公开仓库（如 `pics`），目录按 `/img/年/月/` 组织。
2. 手动路径：push 后引用 `https://cdn.jsdelivr.net/gh/<user>/pics@main/img/2025/06/a.png`。稳定内容建议打 tag 后用 `@v1` 引用，避免主分支更新影响已发布文章。
3. 自动化路径：调 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），body 里放 base64 内容和 commit 信息，返回值中的 `download_url` 换成 jsDelivr 域名即为最终链接。
4. 把第 3 步封装成一个 MCP tool，比如 `upload_image(local_path) -> cdn_url`。OpenClaw 里的 Agent 截完图直接调用，Markdown 链接自动落稿。上传前接一步 pngquant / mozjpeg 压缩，把单图控制在几百 KB。

# 踩坑

- **国内可达性不保证**。jsDelivr 在中国大陆的节点时好时坏，重要文档留一手：本地或备用对象存储存一份原图。
- **LFS 文件拿不到**。jsDelivr 不分发 LFS 指针指向的真实文件，截图仓库别开 LFS。
- **体积限制**。GitHub 单文件硬上限 100MB，jsDelivr 只服务 20MB 以内的文件。截图场景一般碰不到，但校验逻辑要写进脚本。
- **缓存刷新**。`@main` 的 URL 被缓存后，同名覆盖不生效。最稳的做法是文件名带内容 hash，永不覆盖；紧急情况走 `purge.jsdelivr.net` 刷新，注意它有频率限制。
- **公开即公开**。仓库内容可以被枚举，带账号信息的截图务必脱敏，上传前顺手剥掉 EXIF。
- **Token 收敛**。用 fine-grained PAT，只授予这个仓库的 Contents 读写权限，不要把全权限 token 塞进 Agent 配置。

# 可复用建议

- 仓库只做分发，不做归档。git 历史里图片只增不减，重要素材另存冷备。
- 文件名用 `日期-内容hash`，天然防覆盖、便于去重。
- MCP tool 里顺带返回压缩前后体积，Agent 日志里能直接看到存储成本。
- 别拿它当视频床或跑大流量外链，jsDelivr 对滥用不客气，有连坐封仓库的先例。

# 总结

GitHub + jsDelivr 图床的本质是“用 git 做存储，用公共 CDN 做分发”。上限不高，但下限够稳，特别适合文档、博客、Agent 产出物这类轻量图片场景。封装成一个 MCP tool 之后，“截图→压缩→上传→落链接”整条链路对 Agent 完全透明——这才是选它的真正理由，而不是省那几十块钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/4a50e16787e711e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/b6858a1512730081.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/fd6483aa7fce560a.png)

