---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的实践
feedId: 42920
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

写博客、维护 Agent 的知识库、给插件写 README，都绕不开图片外链问题。本地路径换个环境就断，直接用 GitHub raw 链接，`raw.githubusercontent.com` 在国内的可达性又时好时坏。对象存储当然稳，但个人项目为了几十张截图走一遍开通、鉴权、计费配置，性价比不高。我的方案是：**GitHub 仓库当存储层，jsDelivr 当分发层**，零成本跑了一年多，覆盖几百张图，没出过大问题。

## 问题

核心诉求三条：

1. 链接长期稳定，换电脑、换博客框架不失效；
2. 自动化友好——Agent 生成图片后能自动完成「上传 → 拿 URL → 写进 Markdown」的闭环；
3. 在国内大概率可访问。

## 做法

**1. 建一个专用公开仓库**，比如 `assets`，图和代码分开，方便后续整体迁移。

**2. 改写链接为 jsDelivr 格式：**

```
原始: https://github.com/<user>/assets/raw/main/img/foo.png
CDN:  https://cdn.jsdelivr.net/gh/<user>/assets@main/img/foo.png
```

**3. 自动化上传。** 核心就是调 GitHub Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/assets/contents/img/20250611-a1b2c3d4.png \
  -d '{"message":"upload","content":"<base64>"}'
```

返回的 `download_url` 里把域名替换成 jsDelivr 即可。不想写代码可以用 PicGo 的 GitHub 插件；对 OpenClaw 用户，更顺的路径是把这段逻辑封装成几十行的脚本或 MCP 工具，暴露一个 `upload_image` 能力，Agent 产图后直接回写 CDN 链接。

**4. 上传前压缩**（`sharp` / squoosh），单张控制在 500KB 以内，仓库膨胀会拖慢 Git 操作。

## 踩坑点

- **缓存很凶**：jsDelivr 边缘节点缓存激进，同名覆盖后旧图可能长期命中。要么调 `purge.jsdelivr.net` 主动刷新，要么用内容哈希做文件名，走不可变命名，从根上避免覆盖。
- **仓库必须 public**，私有仓库取不到。
- **文件名只用 ASCII**，中文和空格会引入 URL 编码问题。
- gh 源有单文件大小上限（20MB 量级），且明确禁止当网盘用，视频、大压缩包别放，滥用会被封仓库的 CDN 服务。
- **国内可用性有波动历史**（ICP 注销那段时间很糟），建议对关键文章配一个 fallback，比如 Cloudflare R2 或 Pages 镜像。

## 可复用建议

- 命名规范统一为 `YYYYMMDD-<8位哈希>.png`：天然去重 + 免缓存烦恼。
- 上传脚本顺手做三件事：压缩、重命名、把 `![](url)` 格式写回剪贴板。
- 接入 Agent 后增值最大：把上传注册为技能/MCP 工具，「生成图表 → 上传 → 贴链接」全程无需人工介入，这是这套方案相对纯手工图床的核心差异。
- 仓库本身始终是 source of truth，CDN 只是派生层，哪天要换分发渠道，改一条替换规则即可，不动数据。

## 总结

GitHub + jsDelivr 不是最稳的图床方案，但在个人规模下，是成本、稳定性和自动化友好度平衡得最好的一个。本质上它把「存储」外包给 Git、把「分发」外包给公共 CDN，自己只需维护一条几十行的上传管道。如果你的 Agent 也要产图，值得花半小时把它接进去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/90af42b9ca8582e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/40354aacdde6baf2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/d5bb8edaf1bfe26b.png)

