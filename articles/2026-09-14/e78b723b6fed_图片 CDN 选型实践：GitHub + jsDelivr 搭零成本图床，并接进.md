---
title: 图片 CDN 选型实践：GitHub + jsDelivr 搭零成本图床，并接进自动化链路
feedId: 37549
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

做 Agent / 插件开发时经常要外发图片：bot 回复里的生成图、文档截图、README 架构图、MCP 工具产出的结果图。这些图片都需要一个稳定、免登录、带 CDN 的公网 URL。对象存储当然正规，但要么按量收费，要么备案流程繁琐，对个人项目和社区分享来说偏重。

## 问题

直接贴 `raw.githubusercontent.com` 链接，国内访问时好时坏；自建服务器又多一套要维护的东西。我的需求很朴素：

1. 图片有固定公网 URL，走 CDN 缓存；
2. 上传能脚本化，最好封装成 MCP 工具给 Agent 直接调用；
3. 成本为零，迁移成本低——本质上它就是一个 git 仓库。

方案就是标题这套：公开 GitHub 仓库存图，jsDelivr 做分发。

## 做法

**1. 建仓**：建一个 public 仓库，比如 `assets`，按年月分目录：`2025/06/xxx.png`。

**2. URL 规则**：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

`@main` 跟最新分支，`@1.0.0` 可锁定 tag。文档用 main 即可，对外服务建议锁版本。

**3. 脚本化上传**：走 Contents API，一次 PUT 搞定：

```bash
B64=$(base64 -w0 "$1")
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/foo/assets/contents/2025/06/a1b2.png \
  -d "{\"message\":\"upload\",\"content\":\"$B64\"}"
```

PAT 用 fine-grained token，只授这个仓库的 Contents 读写，权限最小化。

**4. 接进 Agent 链路**：把「本地文件 → 返回 CDN URL」封装成一个 MCP tool。Agent 生成图片后调它拿可分发链接，整条链路就闭环了。

## 踩坑点

- **同名覆盖不刷新**：CDN 按 URL 命中缓存，覆盖同名文件后旧图可能继续返回。最稳的办法是文件名用内容哈希（如 sha256 前 8 位 + 扩展名），内容寻址天然不可变；确要刷新可请求 `purge.jsdelivr.net` 的对应路径，但它也有频率限制。
- **Contents API 有 1MB 限制**：高分辨率截图常超。超限就走 Git Data API 建 blob 再提交，或干脆本地 `git push`——批量传图后者更省心。
- **用途边界**：jsDelivr 单文件上限 50MB，定位是开源项目分发。放视频、大文件、高并发外链属于滥用会被限制；图片、图标这类小文件没问题。
- **私密内容别放**：public 仓库 + 公共 CDN 等于全网可读。bot 生成的图可能带用户数据，上传前先脱敏。
- **国内可达性有波动**：jsDelivr 在大陆的可用性这几年起伏过几次。文档图无所谓，bot 回复链路建议留降级方案（备用域名或保留 raw 链接可切换）。

## 可复用建议

- **文件名 = 哈希，路径 = 日期**，这是这套方案最重要的两条纪律，缓存问题和去重问题一起解决。
- 上传逻辑收敛成一个脚本或一个工具，别散在各个流程里；配额看响应里的 rate limit header 就够。
- 仓库 README 写清用途和目录约定，半年后的自己会感谢现在的自己。

## 总结

GitHub + jsDelivr 不是新方案，但对个人项目、插件文档、bot 出图这类场景，它是成本、维护、可达性三者之间比较务实的平衡点。定位想清楚：它是免费文档级图床，不是生产级 CDN。哈希命名、锁版本、留降级，这三件事做到位，就能用得很稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/021d72ef07edc427.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/dfa88f9d577b96e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/31d9be8005b904e7.png)

