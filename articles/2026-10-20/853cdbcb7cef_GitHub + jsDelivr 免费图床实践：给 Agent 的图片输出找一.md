---
title: GitHub + jsDelivr 免费图床实践：给 Agent 的图片输出找一条低成本链路
feedId: 42085
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

在 OpenClaw 的自动化流程里，图片是高频产物：Agent 生成的图表、浏览器自动化截的屏、插件输出的卡片图，最终都要以 URL 形式发到 IM 或写进文档。方案不少，但常见免费图床要么有防盗链、要么链接会过期，对象存储则要绑卡计费。我最后选了 GitHub 仓库 + jsDelivr 这条老路，用下来发现它特别契合自动化场景：仓库即存储，上传走 API，链接走全局 CDN，一切皆 git。

## 问题

我对图床的核心诉求就四条：

1. 免费且没有隐性过期；
2. 支持外链，不被防盗链拦截；
3. 上传可脚本化——Agent/插件能自己完成，不依赖人工传图；
4. 链接稳定，至少在 bot 回复和文档的生命周期内不挂。

## 做法

**1. 建一个公开仓库。** 比如 `openclaw-assets`，必须 public，jsDelivr 只回源公开仓库。目录按 `imgs/{workflow}/{yyyymmdd}/` 组织。

**2. 申请 Fine-grained PAT。** 只勾选这一个仓库的 `Contents: Read and write`，权限收窄到最小，存进 OpenClaw 的环境变量，不要写进任何仓库或配置文件。

**3. 写一个上传工具。** 核心就是 GitHub Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/{owner}/openclaw-assets/contents/imgs/bot/20250611/a1b2.png \
  -d '{"message":"upload","content":"'"$(base64 -w0 img.png)"'"}'
```

在 OpenClaw 里把它封成一个 MCP 工具或 skill：输入本地文件路径，输出 CDN URL，Agent 就能自主完成「生成图 → 拿到外链 → 发出去」的闭环。

**4. 用 jsDelivr 域名访问。**

```
https://cdn.jsdelivr.net/gh/{owner}/openclaw-assets@main/imgs/bot/20250611/a1b2.png
```

`@main` 换成 commit SHA 或 tag 即可钉住版本。

## 踩坑点

- **同路径覆盖 = 缓存灾难。** jsDelivr 按 URL 缓存，覆盖同名文件后旧图可能残留数天。解法是文件名带内容 hash 或时间戳，从设计上禁止覆盖；确需刷新可到 jsDelivr 的 purge 入口按 URL 手动清理。
- **别拿 raw.githubusercontent.com 当图床。** 它不是 CDN，有限速，偶发 Content-Type 不对会直接触发下载而不是渲染。
- **国内访问不稳定。** jsDelivr 在大陆部分运营商下时好时坏，这是选型前就要接受的现实。面向国内用户的正式页面，还是上对象存储；这套方案更适合 bot 输出、内部文档这类场景。
- **单文件 20MB 上限。** 传图前先压缩，顺便也省仓库体积；仓库建议控制在 1GB 内，超大会收到 GitHub 提醒。
- **截图可能带敏感信息。** 仓库是公开的，Agent 上传前最好过一层脱敏或人工确认，把这个检查直接写进 skill 流程里。

## 可复用建议

- 文件命名用 `{workflow}/{日期}/{内容hash}.png`，不可变命名一次性解决缓存和去重两个问题。
- 上传成功后把 URL 落一份本地 JSON 索引，Agent 后续能检索「我之前传过哪些图」。
- PAT 坚持三原则：fine-grained、单仓库、只写权限，泄露了损失也可控。
- 配一个每月跑的 GitHub Actions，统计体积、归档旧图，防止仓库无限膨胀。

## 总结

GitHub + jsDelivr 不是新方案，但对 OpenClaw 用户的优势在于「一切皆 git」：上传是 API 调用，版本是 commit，回滚是 revert，Agent 的每张图都有迹可循。它不适合当正经生产 CDN，但作为个人自动化流水线里的图片中转层，成本为零、够稳、够简单，值得写进你的工具箱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/92e50b59097af118.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/f24dc69341025322.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/c9ebdbd132f6594d.png)

