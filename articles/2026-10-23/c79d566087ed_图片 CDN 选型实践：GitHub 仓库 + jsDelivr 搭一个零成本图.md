---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭一个零成本图床
feedId: 42318
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

在 OpenClaw 社区写教程、给 Agent 配素材、给插件仓库补 README，图片托管是绕不开的小事。第三方免费图床要么限外链、要么限速、要么说倒就倒；自建对象存储又要养域名和证书。对开发型用户，最省心的组合其实就在手边：GitHub 公开仓库当存储，jsDelivr 当免费 CDN。

## 问题

我对图床的要求很具体：

1. 外链稳定，最好走 CDN；
2. 能被脚本/Agent 自动上传并返回 URL；
3. 数据在自己仓库里，随时可迁移；
4. 零成本。

## 做法

**第一步：建仓库。** 新建公开仓库 `assets`（私有仓库 jsDelivr 拿不到），按 `yyyy/mm/` 目录组织，方便审计和清理。

**第二步：确认外链格式。**

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/2025/06/demo.webp
```

`@` 后面可跟分支、tag 或 commit hash。追求"永不变化"的缓存语义时用 commit hash，改图必然换名。

**第三步：选上传方式。** 手动拖拽或 git push 都行，但要做自动化就得走 Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/assets/contents/2025/06/demo.webp \
  -d '{"message":"upload","content":"<base64>","branch":"main"}'
```

**第四步：接入 OpenClaw。** 把上面逻辑包成一个小 MCP tool（比如 `upload_image`），入参是本地路径或图片 URL，出参直接给 markdown 格式的 `![img](...)`。之后让 Agent 写周报、生成文档时自动插图并回填链接，全流程不再碰图床网站。

## 踩坑点

- **缓存太凶，替换同名文件不生效。** 要么文件名带短 hash，要么去 purge.jsdelivr.net 提交路径强制刷新。我的做法是永远只新增文件，不覆盖。
- **别把 raw.githubusercontent.com 当外链。** 它不是 CDN，有速率限制，可达性也一般。
- **国内可达性有波动。** jsDelivr 的国内节点近年不稳定，如果读者以国内为主，务必准备镜像方案（同仓库托管到 Cloudflare Pages / R2，改一下域名前缀即可切换）。
- **体积纪律。** 单文件超 100MB 会被 GitHub 拒收；图片先压成 webp，单张控制在几百 KB。
- **公开即公开。** 仓库内容任何人可见，截图前过一遍敏感信息。
- **API 限流。** 认证后每小时 5000 次请求，高频自动化要留意，上传失败先查是不是 403。

## 可复用建议

1. 命名规范固定为 `yyyy/mm/<内容摘要>-<短hash>.webp`，天然规避缓存冲突，也方便写清理脚本；
2. 上传脚本做成幂等：先 GET 查文件 sha，已存在就跳过，Agent 重试不会产生脏提交；
3. 仓库挂一个每周跑的 GitHub Action 做体积审计，超阈值告警，防止仓库悄悄膨胀；
4. 链接拼接逻辑单独抽成函数，将来切镜像源只改一处。

## 总结

GitHub + jsDelivr 不算多先进的方案，但它把工程上的关键项都占了：数据自有、协议标准、成本为零、天然可自动化。对社区写作者和 Agent 工作流来说，它是一个"足够好"的默认选项；唯一需要认真评估的是目标读者的网络环境，必要时配一份镜像方案兜底。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/27996f994ecb9ca5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/613117afea6e4d71.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/c90bbe5ab539a76e.png)

