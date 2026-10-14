---
title: 图片 CDN 选型实录：GitHub + jsDelivr 给 Agent 产出搭一个免费图床
feedId: 41555
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

跑 Agent 自动化时经常有这个需求：生成图表、截屏归档、产出带图的 markdown 报告。图放哪是个现实问题——本地路径在别人打开就是裂图；对象存储要付费，有的还要签名 URL；传统免费图床说关就关。我的诉求很简单：链接长期稳定、国内能打开、上传这步能被脚本和 MCP 工具接管。

## 问题与选型

对比了几条路：

- **raw.githubusercontent.com**：国内时通时断，直链不可靠；
- **对象存储 / Cloudflare R2**：稳，但要注册付费、配自定义域，个人场景偏重；
- **GitHub 仓库 + jsDelivr**：GitHub 当存储，jsDelivr 当边缘缓存，零成本，URL 规则清晰。

个人和中小规模自动化场景，最后一条性价比最高。本质是拿 Git 仓库当 origin，jsDelivr 做全球缓存层。

## 做法

1. **建一个公开仓库**，比如 `assets`，按 `img/年月/` 分目录，别都堆根目录。
2. **上传走 GitHub Contents API**，一段 curl 就够，方便封装成 MCP tool：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/me/assets/contents/img/2024/06/a.webp \
  -d '{"message":"add image","content":"<base64>"}'
```

3. **引用地址换成 jsDelivr**：

```
https://cdn.jsdelivr.net/gh/me/assets@main/img/2024/06/a.webp
```

4. **缓存刷新**：覆盖同名文件后，对 `https://purge.jsdelivr.net/gh/...`（同路径）GET 一下即可。
5. 把第 2 步封装成一个小 MCP tool：输入本地图片，输出 CDN 链接，Agent 写报告时直接引用，全流程无人工。

## 踩坑点

1. **国内可达性波动**。jsDelivr 主域这几年时好时坏，脚本里内置 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 多域名探测，返回第一个通的。要更稳就在前面再套一层 Cloudflare Workers 兜底。
2. **缓存很激进**。同名覆盖后旧图可能还在边缘缓存里，最省心的办法是文件名带内容 hash，只增不改。
3. **单文件上限 20MB**。超限的文件 jsDelivr 直接不服务，截图先压 webp、宽度控制在 1600px 以内。
4. **仓库只增不减**。git 删了文件历史里还在，仓库体积不会降。控制上传粒度，别当无限网盘用。
5. **`@main` 是可变引用**。跟随分支最新内容，方便迭代；需要可复现的场合（比如文档归档），pin 到 commit 短 hash。
6. **服务定位是灰色地带**。jsDelivr 官方定位是开源项目 CDN，纯私人图床仓库有被限制的讨论和先例。重要长期资产别单点依赖这条路。
7. **自动化限流**。Contents API 带 token 是 5000 次/小时，高频上传记得在脚本里合并提交。

## 可复用建议

- 命名规范：`日期_内容hash.webp`，天然去重，顺手绕开缓存坑；
- 上传前统一压缩转换格式，这一步收益最大；
- 把「上传并返回 CDN 链接」做成一个原子化的 MCP tool，任何 Agent 流程都能复用；
- 脚本内置多域名 fallback 和失败重试，别裸调；
- 严肃场景留退路：迁移到 Cloudflare R2 + 自定义域，成本只是换 URL 前缀，目录结构可以原样保留。

## 总结

GitHub + jsDelivr 是零成本、五分钟跑通的方案：GitHub 管存储和版本，jsDelivr 管边缘分发。它不是企业级方案，真实短板在国内可达性和服务定位的灰色地带。但只要接受这两点、做好压缩和多域名兜底，它就是个人 Agent 产出贴图最省事的一条路—— Especially 当上传被封装成一个工具调用之后，图床这个概念本身就从流程里消失了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/32ce91bb9ee42438.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/8013be862480ff15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/1b2d07c12d71bd58.png)

