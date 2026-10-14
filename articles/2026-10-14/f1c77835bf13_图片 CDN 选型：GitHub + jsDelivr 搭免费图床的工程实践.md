---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的工程实践
feedId: 41557
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

最近给社区写帖子、以及跑自动化产物时（Agent 生成的报告、MCP 工具输出里的截图），都需要稳定的图片外链。常规选项各有痛点：对象存储要实名加按量计费；SM.MS 这类免费床有配额和域名波动；微信图床有防盗链，外链基本不可用。最后选了 GitHub 公共仓库 + jsDelivr 的组合，用了小半年，把实践和坑记录一下。

## 问题

核心诉求就三条：Markdown 里能长期引用；脚本和 Agent 能自动上传、不需要人工干预；零成本免实名。前提是接受它"非生产级 SLA"的定位。

## 做法

**1. 单独建公共仓库**（如 `assets`），和代码仓库分离，不污染 commit 历史。目录按 `img/{yyyyMM}/` 组织。

**2. 手动场景**用 PicGo 的 GitHub 插件，配好 repo、分支、token，拖图上传自动复制链接。

**3. 自动化场景**走 GitHub Contents API，一次 PUT 就能上传：

```bash
KEY="img/202506/$(sha1sum pic.webp | cut -c1-12).webp"
curl -s -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/USER/assets/contents/$KEY" \
  -d "{\"message\":\"upload\",\"content\":\"$(base64 -w0 pic.webp)\"}"
```

注意 `base64 -w0` 不能换行（macOS 用 `base64 | tr -d '\n'`）。

**4. 外链格式：**

```
https://cdn.jsdelivr.net/gh/USER/assets@main/img/202506/xxx.webp
```

建议把 `@main` 换成固定的 release tag 或 commit 短哈希，语义上更接近"不可变资源"，也减少 jsDelivr 回源解析分支的开销。

## 踩坑点

1. **缓存几乎不可变。** jsDelivr 对同名文件缓存很激进，覆盖上传后旧图可能长期不更新。正确姿势是文件名带内容哈希，把每个文件当不可变对象，要改就换名。确实需要刷新可请求 `purge.jsdelivr.net` 对应路径。
2. **大陆访问有波动。** 这是本方案最大风险，jsDelivr 国内可用性时好时坏。选型前先在目标网络实测；把 CDN 前缀抽成配置项，必要时一键切到 Cloudflare 代理或国内 OSS，文档里的相对路径不用动。
3. **体积限制。** GitHub 单文件上限 100MB，仓库建议控制在 1GB 量级。上传前强制压缩：截图转 webp、质量 80，一般能从 2MB 压到 200KB 以内。
4. **ToS 定位。** jsDelivr 官方定位是开源项目文件分发，不是通用图床。个人博客和文档的量级没问题，别拿去给高 QPS 业务当生产 CDN，容易连坐封仓库。
5. **自动化细节。** 匿名 API 只有 60 次/小时，务必用 token；用 fine-grained token 只授权这一个仓库；更新已存在文件会返回 409，要带原文件 sha 走另一条路径——"只新增不覆盖"的哈希命名顺便把这个坑也绕开了。

## 可复用建议

- 命名规范 `{yyyyMM}/{contenthash}.webp`，天然不可变、可去重；
- CDN 域名放进环境变量，迁移只改一行配置；
- 流水线固定为：压缩 → 算哈希 → 上传 → 回写 Markdown 链接，MCP 工具可以按这个四步封装成一个小工具；
- 仓库定期 `git clone` 做本地镜像，GitHub 侧出问题随时有底。

## 总结

这套方案的本质是"GitHub 做存储、jsDelivr 做边缘"：成本为零，对自动化管线极其友好，适合博客、文档、Agent 产物这类低频写入、中低频读取的场景。但它没有 SLA，国内可达性是最大变量——把它当 best-effort 的免费层来用，并在架构上预留好换 CDN 的能力，就基本不会被坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/f05ac9c8c32212a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/d1c77e77fd7ec37a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/a904bae183aefe8b.png)

