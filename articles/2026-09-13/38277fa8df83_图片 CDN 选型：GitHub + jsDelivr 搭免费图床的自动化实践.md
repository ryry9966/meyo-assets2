---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 37413
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

OpenClaw 的自动化工作流里经常要产出带图内容：Agent 写的博客配图、bot 回复里的截图、MCP 工具生成的图表。这些图需要一个稳定、可外链、可编程获取的 URL。裸用 `raw.githubusercontent.com`，国内访问基本不可靠；上商业 OSS 要付费、要管生命周期；第三方免费图床随时可能限外链，数据还不好迁走。

折中方案：**GitHub 仓库做存储，jsDelivr 做分发**。零成本、纯静态、天然适合自动化。

## 做法

1. 建一个公开仓库（如 `cdn-assets`），按年份/用途分目录。
2. 传图三选一：
   - 本地 git push（手工场景）；
   - GitHub Contents API，base64 直传，适合脚本，不用 clone 仓库；
   - 封装成脚本或 MCP 工具暴露给 Agent：输入本地路径，输出 CDN 链接。
3. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。ref 用 tag/commit 最稳，用分支名要留意缓存。

脚本骨架大致是：

```bash
SHA=$(sha1sum out.webp | cut -c1-8)
NAME="$(date +%Y%m%d)-$SHA.webp"
B64=$(base64 -w0 out.webp)
# PUT https://api.github.com/repos/<user>/cdn-assets/contents/imgs/2025/$NAME
# 拼接返回：https://cdn.jsdelivr.net/gh/<user>/cdn-assets@main/imgs/2025/$NAME
```

在 OpenClaw 流程里，Agent 只关心一个函数：`put_image(path) -> url`，其余细节全部收进工具里。

## 踩坑点

1. **同名覆盖 + 分支引用 = 旧缓存**。jsDelivr 对 `@main` 也会缓存，覆盖同名文件后线上可能长时间不更新。解法：文件名带内容哈希、永不覆盖；或改用 `@<commit>` 引用；紧急情况走 `purge.jsdelivr.net` 的 purge 接口。
2. **大陆可用性有波动**。jsDelivr 的国内节点状态这些年起伏过，别把它当唯一出口。重要场景留一条备用链路（比如同步推一份到对象存储），仓库本身就是数据源，切换只改 URL 前缀。
3. **体积限制**。单文件超过 20MB jsDelivr 不回源服务，GitHub 单文件硬上限 100MB。图片先压成 webp、控制合理宽度再上传。
4. **公开仓库无隐私**。敏感截图一律本地先脱敏，这个要写进工具的前置检查里。
5. **token 权限收敛**。用 fine-grained PAT，只授予这一个仓库的 contents 写权限，别拿全局 token 喂给自动化流程。

## 可复用建议

- 把「上传 → 哈希命名 → 拼 URL」固化成一个工具函数（shell / MCP tool 均可），任何 Agent 流程都能复用。
- 命名规范 `日期-哈希` 一举两得：内容去重 + 天然防缓存事故。
- 仓库即 source of truth，CDN 只是边缘层。哪天要换分发方式，数据零迁移。
- 只放图片类小文件，别把它当网盘，也别拿来分发视频。

## 总结

GitHub + jsDelivr 适合个人博客、文档、bot 产出这类低频中量的图片分发，不适合当生产级 OSS 或隐私存储。工程上真正的价值在于**存储与分发解耦**：配合一个十几行的小工具，自动化流程里「出图」就退化成一次函数调用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/cff13410ea79980d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/f54f9938b53763c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/7365af167cfeafc8.png)

