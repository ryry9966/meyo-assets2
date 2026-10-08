---
title: 图片 CDN 选型：GitHub 仓库 + jsDelivr 搭免费图床的实践
feedId: 40887
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

写博客、搭文档站、或者让 Agent 输出带图报告时，图床是绕不开的基础设施。个人项目没有预算买 OSS，主流免费图床（SM.MS、Imgur 之类）要么限流，要么有防盗链，要么某天突然失效。最后我落地了一个方案：**GitHub 仓库当存储，jsDelivr 当 CDN**，用了一年多，稳定够用。

## 问题

核心诉求就四条：

1. 外链长期有效，不能像临时图床那样过期；
2. 国内能访问（至少大概率能）；
3. 能版本管理，图片改了有据可查；
4. 能自动化——Agent 生成图片后自动上传并回填 URL，不需要人动手。

`raw.githubusercontent.com` 不是 CDN，无缓存且国内不稳定，直接排除。jsDelivr 会自动镜像所有公开 GitHub 仓库，无需注册，正好补上这一环。

## 做法

**1. 建仓。** 新建一个 public 仓库专放图，比如 `img-host`，目录按年月组织：`img/2025/03/`。

**2. 拼 URL。** 格式固定：

```
https://cdn.jsdelivr.net/gh/<用户>/<仓库>@<分支或tag>/<路径>
```

**3. 自动化上传。** 核心就一个 GitHub Contents API 的 PUT 请求：

```bash
CONTENT=$(base64 -w0 pic.webp)
curl -s -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/img-host/contents/img/2025/03/pic.webp \
  -d "{\"message\":\"add pic\",\"content\":\"$CONTENT\"}"

# 拼接 CDN 地址
echo "https://cdn.jsdelivr.net/gh/USER/img-host@main/img/2025/03/pic.webp"
```

**4. 接入 OpenClaw。** 把压缩、重命名、上传、返回 URL 这几步封装成一个 skill 或 MCP 工具。Agent 产出图片后自动走完整个链路，正文里直接写 CDN 链接，全程无人值守。这才是这套方案对自动化用户最大的价值。

## 踩坑点

- **缓存是最大的坑。** jsDelivr 缓存很激进，同名覆盖后旧图会长期命中缓存。纪律是：文件名永不重复，用 `日期-描述-8位hash` 命名；紧急刷新走 `purge.jsdelivr.net`。
- **国内可用性会波动。** jsDelivr 在大陆时好时坏，不能当唯一依赖。备用域名 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可以先试，再不行退回 GitHub 原链。
- **别把仓库当网盘。** jsDelivr 对 gh 源有单文件大小限制，仓库过大还可能触发滥用判定。上传前统一压成 webp，控制在几百 KB。
- **文件名避免中文和空格**，URL 编码后的链接又丑又易错。
- **public 仓库即公开**，敏感截图、含内网信息的图不要传。

## 可复用建议

1. 上传工具统一返回 `{url, backup_url, size}` 三个字段，上层切换域名时只改一处；
2. 加个巡检脚本，定期 HEAD 请求抽样检查 CDN 可用性，失败即告警；
3. 同一套思路可以放字体、静态 JSON 配置，但大文件一律别放；
4. 用 fine-grained token 并限定只授权这一个仓库，别拿全权限 PAT 图省事。

## 总结

GitHub + jsDelivr 的本质是「仓库即图床、git 即版本管理、URL 可拼接」，成本为零，链路完全可自动化。它不是完美的 CDN——国内访问要留后手，缓存策略要靠命名纪律兜底。对个人博客、文档站和 Agent 自动化输出这类中小流量场景，是我目前性价比最高的选择。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/c8d1b06a850087d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/475a6c3ac7f3ad11.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/766f13e5745b98a3.png)

