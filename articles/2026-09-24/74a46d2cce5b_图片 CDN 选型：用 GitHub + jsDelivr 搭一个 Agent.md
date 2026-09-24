---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个 Agent 能自己用的免费图床
feedId: 38735
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

做 OpenClaw 自动化时经常要给外部世界"看图"：Agent 生成的截图要贴进报告页面、MCP 工具返回结果想带个缩略图、插件文档需要配图。国内对象存储要实名、备案、按量计费；第三方免费图床配额小、政策说变就变。最后我选了 GitHub 仓库 + jsDelivr 这条老路：零成本、全 API、纯脚本可控，适合个人和小团队的自动化链路。

## 问题

核心诉求就三条：

1. 上传能被脚本化——Agent 能在流程里自己调，不依赖人肉传图；
2. 返回的 URL 稳定、可缓存、可外链；
3. 不想多维护一台服务器。

## 做法

1. 建一个 public 仓库（比如 `assets`），专库专用，别和代码混在一起。
2. 申请 fine-grained PAT，只授予该仓库 Contents 读写权限，存进环境变量，绝不入库。
3. 上传走 Contents API，一段 curl 就够：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/me/assets/contents/2025/06/abc123.png \
  -d '{"message":"upload","content":"'"$(base64 -w0 img.png)"'"}'
```

4. 引用地址用 jsDelivr：
   `https://cdn.jsdelivr.net/gh/me/assets@main/2025/06/abc123.png`
5. 把第 3、4 步封装成 MCP 工具 `upload_image`：输入本地文件路径，输出 CDN URL。之后 Agent 在任何流程里都能自助插图。

## 踩坑点

- **同名覆盖不生效**：jsDelivr 缓存很激进，覆盖同名文件后 CDN 返回的还是旧图。干脆用内容 hash 做文件名，URL 天然不可变，问题直接消失。真要刷，手动请求 `purge.jsdelivr.net` 对应路径。
- **单文件 20MB 上限**：jsDelivr 定位是开源资源分发，别拿来堆视频和大图。上传前压一遍（pngquant、转 webp），顺带省流量。
- **public 仓库谁都能看**：带 token、带客户信息的截图先脱敏。我吃过亏，现在上传脚本里强制过一遍检查。
- **API 限流**：token 下 Contents API 约每小时 5000 次，批量迁移几千张图没问题，但别写成秒级循环灌库，容易被判滥用。
- **国内可达性波动**：jsDelivr 在大陆时好时坏，关键页面留一个 raw 链接兜底。

## 可复用建议

- 文件名规则统一为 `{日期}/{内容hash}.{ext}`，幂等、可去重、好排查。
- 仓库根目录维护一个 `index.json` 清单，Agent 查图不用遍历 API。
- 挂个 GitHub Action 在 push 时自动压缩、转格式，让上传端保持"傻白甜"。
- 这个方案最适合**文档配图、Agent 产出物、低频更新**的场景；高频大流量请直接上正规 OSS，别为难公益 CDN。

## 总结

GitHub + jsDelivr 不是什么新鲜方案，但对 OpenClaw 用户的独特价值在于：建库、传图、拿 URL 整条链路都是 API，可以完整交给 Agent 自治执行，不引入任何新的外部依赖。成本为零的代价，是自己守住三条线：脱敏、限流、缓存策略。想清楚这三点，它就是自动化工作流里最省心的一块拼图。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/17612581bfe8f147.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/854e80ce87db7890.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/2cdfab2ed1cc7579.png)

