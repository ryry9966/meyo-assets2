---
title: GitHub + jsDelivr 免费图床实践：从选型到 MCP 工具化
feedId: 42864
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

写博客、维护文档、给 Agent 产出的内容配图，都绕不开一个问题：图放哪。直连 GitHub raw 在国内基本不可用；对象存储要花钱，绑自定义域名还要备案；各类免费图床跑路风险高。我的方案是 GitHub 公开仓库 + jsDelivr，个人博客用了近两年，几千张图片稳定可用。

## 选型

需求很朴素：免费、URL 稳定、能被脚本和 Agent 自动调用。对比下来，jsDelivr 的优势是直接把 GitHub 仓库当源站，零注册零配置，URL 规则：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<版本>/<路径>
```

版本可以是分支、tag 或 commit hash。

## 做法

1. 建公开仓库（如 `cdn`），按目录分类图片；
2. 提交前统一压缩：转 WebP、宽度压到 1600px 内，单图控制在 200KB 上下；
3. 引用优先用 tag 而非 `@main`。分支引用缓存约 12 小时，改图不生效很常见；tag 视为不可变，缓存更久。图片变更就发新 tag 或换文件名，别原地覆盖；
4. 需要立即生效时手动刷缓存：浏览器访问 `https://purge.jsdelivr.net/gh/user/repo@main/xxx.png` 即可。

对 OpenClaw 用户，更有价值的是把链路工具化。我写了个约 60 行的 Python 脚本：接收本地图片 → 内容哈希重命名 → 压缩 → push → 返回 jsDelivr URL，再包一层 MCP server 暴露成 `upload_image` 工具。写作 Agent 在成文流程里就能“截图—上传—拿链接—插图”全自动完成。凭证用 fine-grained PAT，只授这个仓库的 Contents 写权限。

## 踩坑

- **国内访问不稳定是最大变量**。jsDelivr 2022 年后备案失效，国内解析时好时坏。个人博客可接受，生产业务别押注它；
- 公开仓库人人可访问，截图带 token、内网地址的先脱敏；
- 删除图片不会缩小仓库，git 历史永久保留。控制提交量，单仓库建议 1GB 内；
- jsDelivr 定位是开源项目文件分发，不是无限量商用图床，超大批量搬运有封禁风险，自用文档图片没问题；
- 掘金、知乎等平台会转存或替换外链，自建站不受影响。

## 可复用建议

- URL 模板写成常量，方便整体切换到 statically.io（同类服务，URL 结构几乎一致）做备用源；
- 命名用“内容哈希 + 日期”，天然去重、便于追溯；
- 博客模板里给 img 加 `loading="lazy"` 和宽高属性，弱网体验差距明显。

## 总结

GitHub + jsDelivr 不是图床最优解，但它是“零成本 + 可自动化 + 不锁定”的平衡点。个人博客、开源文档、Agent 内容配图这类轻量场景足够好；可用性要求高的业务，把它当备用即可。工具化的关键不在图床本身，而在“上传即得 URL”这条链路——封装成 MCP 工具后，配图才真正成为自动化流程里的一等公民。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/efd07a56853fbd95.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/f2cd8f78d0f9766f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/9a55aea189bc8038.png)

