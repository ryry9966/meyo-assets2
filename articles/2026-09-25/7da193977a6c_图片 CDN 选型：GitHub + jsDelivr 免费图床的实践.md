---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践
feedId: 38988
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

在 OpenClaw 的日常使用里，图是个绕不开的东西：agent 跑完自动化任务要留截图归档，写技术帖需要配图，MCP 插件输出的报告里也常带图表。这些图如果只靠本地路径或临时链接，换设备、发帖、分享时都会断链，配图管理就成了一个反复出现的小痛点。

## 问题

我先试过几个常见免费方案，都不太满意：

- **公共图床**（SM.MS 之类）：免费额度有限，外链稳定性和存活周期不可控；
- **GitHub raw 链接**：`raw.githubusercontent.com` 连通性和缓存策略都不适合直接当 CDN 用，页面加载会被拖慢；
- **自建对象存储**：要花钱、要维护，对"就是想贴一张图"的需求来说太重。

最后选了 **GitHub 仓库 + jsDelivr** 的组合：仓库负责存储和版本，jsDelivr 负责全球分发和缓存。零成本、API 齐全，整条链路可以全自动化。

## 做法

1. 建一个**公开仓库**专门放图（比如 `pics`），用 `main` 分支；
2. 申请 **fine-grained PAT**，只授权这个仓库的 Contents 读写，权限最小化；
3. 上传统一走 GitHub Contents API（内容 base64 编码后 PUT）：

```bash
IMG=$(base64 -w0 ./screenshot.png)
curl -s -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/pics/contents/2025/06/abc123.png \
  -d "{\"message\":\"upload\",\"content\":\"$IMG\"}"
```

4. CDN 地址格式固定为：
   `https://cdn.jsdelivr.net/gh/USER/pics@main/2025/06/abc123.png`
5. 把"压缩 → 命名 → 上传 → 返回 URL"封装成一个 shell 小工具，在 OpenClaw 里通过 exec 调用。之后 agent 生成截图后只需说一句"上传这张图"，就能拿到可直接拼进 Markdown 的稳定链接。

## 踩坑点

1. **缓存不是即时的**：`@main` 虽然指向分支，但内容会被 CDN 缓存数小时甚至更久。覆盖同名文件后需主动调用 `https://purge.jsdelivr.net/gh/USER/pics@main/...` 清缓存。更省心的做法是文件名带日期或短 hash，只新增、不覆盖。
2. **单文件 20MB 上限**：jsDelivr 对 GitHub 源超过 20MB 的文件不分发。截图一般没问题，录屏转的 GIF 要留意。
3. **只能用公开仓库**：私有仓库 jsDelivr 不代理。注意别把含敏感信息的截图传上去，PAT 更不能进仓库。
4. **定位要想清楚**：jsDelivr 是开源项目 CDN，不是图床服务，官方条款不建议纯当存储用。图先压缩（pngquant / 转 WebP），单图尽量控制在 500KB 以内，别拿它备份几个 GB 的历史文件。
5. **仓库体积**：每次提交都会让 `.git` 膨胀，仓库大了本地 clone 会很慢，这也是坚持"先压再传"的原因之一。

## 可复用建议

- 路径按 `年/月/项目名` 组织，文件名带短 hash，天然规避缓存覆盖问题；
- 上传链路统一收口成一个脚本或 MCP tool，比让 agent 每次现拼 curl 稳定得多；
- 关键图在仓库之外留一份本地备份，CDN 侧出问题时至少 raw 链接还在；
- 对国内连通性敏感的场景，让工具同时返回 raw 地址，作为 fallback 可选项。

## 总结

GitHub + jsDelivr 不是新方案，但对"个人 + agent 自动化"这个场景刚好够用：零成本、接口标准、可全自动。它的边界也很清楚——不适合大文件，不能当无限存储，缓存需要主动管理。把上传链路固化成一个小工具之后，帖子和 agent 报告里的每一张配图都有稳定外链，这件事一次做完，长期受益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ebaed73c36ade6e9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/53d404490206092b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/bde9ea70adb12f6d.png)

