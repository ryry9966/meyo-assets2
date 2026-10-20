---
title: GitHub + jsDelivr 免费图床实践：给博客和 agent 配图的小基建
feedId: 42091
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

写博客、给 bot 输出贴运行截图、给文档配图——图片放哪是个绕不开的小问题。对象存储要绑卡，免费图床时不时跑路，直接传仓库又没 CDN。最后我落回一个老但好用的组合：**GitHub 公开仓库 + jsDelivr 的 gh 通道**。对 OpenClaw 用户还有个加分项：上传逻辑可以封装成脚本或技能，让 agent 自己完成「截图 → 上传 → 拿 URL」这条链路。

## 问题

直接用 `raw.githubusercontent.com` 有两个毛病：大陆访问不稳定，且 raw 没有 CDN，读者打开慢。jsDelivr 对公开 GitHub 仓库提供全球节点缓存，URL 规则简单，等于免费把仓库变成 CDN。

## 做法

1. 建一个专门的公开仓库，如 `imgs`，别和代码混放。
2. 约定目录：`img/2025/` 放截图，按年归档，方便管理。
3. 上传二选一：本地 `git push`；或脚本走 Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<user>/imgs/contents/img/2025/a1b2c3.png \
  -d '{"message":"add img","content":"<base64>"}'
```

返回值里有 sha，可顺带做幂等。Token 用 fine-grained，只给这个仓库的 contents 写权限。

4. 引用 URL：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2025/a1b2c3.png`。需要不可变版本时，打个 tag（如 `v2025.06`）替换 `@main`。
5. 封装成 OpenClaw 技能：输入本地图片路径，输出结构化结果（url、size、sha），agent 拿到就能直接用。

## 踩坑点

- **大陆访问有波动。** jsDelivr 此前 ICP 注销后，部分网络环境间歇不可达。上线前先在目标读者的网络里实测，别想当然。
- **缓存。** `@main` 的更新会被缓存，改图不换名常常「看不到变化」。换名重传，或用 `purge.jsdelivr.net/gh/...` 主动刷新；正式图用 tag 固定。
- **单文件上限 20MB**，超了直接 404，先压缩。
- **仓库是公开的**，任何截图都等于公开发布，敏感信息打码前想清楚。
- **仓库会越来越重**，别动不动 clone，脚本走 API 即可。
- jsDelivr 定位是开源项目 CDN，不是网盘。少量配图没问题，成吨刷流量有被限制的风险。

## 可复用建议

- 文件名用内容 hash（如 sha 前 8 位），天然避免缓存冲突，也方便去重。
- URL 统一成 `<prefix>/<path>` 形式，以后迁到 Cloudflare R2 或 Pages，只改前缀，链接不散。
- 给 agent 用时返回值带上 sha 和 size，agent 可以自检「传没传上、传的是不是这张」。
- 本地留一份原始素材，仓库只当发布层。

## 总结

GitHub + jsDelivr 适合个人博客、agent 配图、文档截图这类轻量公开场景：零成本、URL 干净、可自动化。代价是公开性、20MB 限制和大陆访问的不确定性。守住「少量、公开、自动化」三条，它是个值得留着的小基建；超出这个范围，就老老实实上对象存储。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/2b731c9ab940391d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/940dc75170034d1e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/24a07542bdceb3a8.png)

