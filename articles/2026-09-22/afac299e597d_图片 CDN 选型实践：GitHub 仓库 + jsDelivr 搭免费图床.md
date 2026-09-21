---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭免费图床
feedId: 38399
source: 综合讨论
publishedAt: 2026-09-22
---

# 背景

写 OpenClaw 插件的 README、贴 Agent 运行截图、发技术帖，都绕不开一个问题：图片放哪。放本地路径没法分享；传微信公众号，外链会被替换成站内地址，迁移即失效；上对象存储要实名、备案、算流量费，对个人项目有点重。

我目前的方案是：**GitHub 仓库当存储，jsDelivr 当 CDN**。零成本、地址稳定、可以用 git 管版本，缺点也都清楚，适合文档、博客、社区发帖这类非关键业务场景。

# 做法与步骤

**1. 建一个公开仓库当图床**

新建一个仓库，比如 `assets`，可见性必须是 public（jsDelivr 只分发公开仓库）。目录建议按项目或日期组织，如 `openclaw-plugin-demo/2025-06/`，别把所有图堆在根目录。

**2. 上传图片**

三种方式按需选：
- 网页端直接拖拽，最省事；
- 本地 `git push`，适合批量；
- GitHub API，适合自动化（见第 4 步）。

**3. 用 jsDelivr 的地址引用**

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

例如 `https://cdn.jsdelivr.net/gh/me/assets@main/demo/shot.png`。需要不可变引用时把 `@main` 换成 `@v1.0` 或 commit hash，同步文档版本时很有用。

**4. 自动化：给 Agent 加一个上传 skill**

这就是图床方案和 OpenClaw 结合最顺的地方——上传本质是一个 API 调用，完全可以做成 skill 或 MCP 工具：

```text
输入本地图片路径
→ 压缩转 webp
→ base64 编码
→ PUT https://api.github.com/repos/<user>/assets/contents/<path>
   header: Authorization: Bearer <PAT>
→ 拼出 jsDelivr URL 返回，并写回 Markdown
```

不想自己写的话，PicGo、PicX 这类现成工具也是走同一套 GitHub API。

**5. 缓存刷新**

jsDelivr 对文件边缘缓存约 7 天，**同名覆盖不会生效**。要么调用 `https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>` 主动刷新，要么干脆改文件名。

# 踩坑点

- **缓存是最大的坑**。我最早把文档截图重复上传同名文件，页面始终是旧图。后来统一改成 `日期-内容哈希.png` 的命名，缓存问题直接消失，比事后 purge 靠谱。
- **国内访问会波动**。jsDelivr 在大陆的解析历史上反复出过问题，时好时坏。建议正文保留原始 GitHub 路径做兜底，重要文档别只留一条链路。
- **token 权限最小化**。用 fine-grained PAT，只给这一个仓库的 Contents 读写，别拿全局 token 图省事。
- **体积控制**。截图先用 squoosh 或 TinyPNG 压一遍再传，PNG 转 WebP 通常能省 70% 以上；单文件别太大，仓库总体积控制在几百 MB 内。
- **隐私自查**。仓库是公开的，往里推之前确认截图里没有 token、邮箱、内网地址。做自动化 skill 时尤其注意，Agent 报错截图经常带着环境变量。

# 可复用建议

1. 图床仓库独立于代码仓库，避免和项目历史混在一起，方便整体迁移。
2. 命名规范一次定好：`<项目>/<年月>/<用途>-<hash>.<ext>`，配合自动化基本不用人工干预。
3. 把「生成 → 压缩 → 上传 → 写回链接」整条流水线交给 Agent 跑，人只负责最终审阅，这是这套方案最大的杠杆点。
4. 定期跑一个断链巡检脚本（HEAD 请求遍历 Markdown 里的图片 URL），几分钟写完，长期有用。
5. 想升级的话，Cloudflare R2 是下一步的自然选项，但个人文档场景，GitHub + jsDelivr 够用到很久。

# 总结

这套方案的定位很明确：**免费、可控、可自动化，但不是生产级存储**。它解决的是文档和社区发帖里「图片能稳定打开、链接能长期有效、上传能交给 Agent」这三个诉求。接受它的边界（国内波动、无 SLA、公开可见），再配好命名规范和自动化，就是个人开发者图床的性价比天花板。

---

