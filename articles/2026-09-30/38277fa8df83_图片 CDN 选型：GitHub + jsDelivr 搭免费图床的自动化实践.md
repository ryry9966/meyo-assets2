---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 39703
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 日常产出里有大量图片：Agent 运行截图、MCP 工具调试过程、插件示例配图。这些图要贴进社区帖、README 或 Agent 生成的页面，就需要稳定外链。直接引用 raw.githubusercontent.com，国内加载经常超时；自建图床又多一台服务器要维护。最后我选了折中方案：GitHub 仓库做存储，jsDelivr 做分发。

## 问题

选型时有三个硬约束：

1. **链接长期稳定**——目录一调整不能全站 404；
2. **可自动化**——Agent / MCP 脚本能一键上传并直接拿到 URL，不允许有手工步骤；
3. **国内访问可用**——至少要明显好于 raw 链接。

## 做法与步骤

**1. 建一个公开仓库**，例如 `imgs`，按年分目录：`2026/xxx.png`。

**2. 统一命名规则**：文件名取内容 SHA-256 的前 12 位，如 `ab12cd34ef56.png`。好处后面讲。

**3. 上传走 GitHub Contents API**，脚本里一个 PUT 就够：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/USER/imgs/contents/2026/ab12cd34ef56.png" \
  -d '{"message":"upload","content":"<base64>"}'
```

Token 用 fine-grained PAT，只授权这一个仓库的 Contents 读写。

**4. 引用走 jsDelivr**：

```
https://cdn.jsdelivr.net/gh/USER/imgs@main/2026/ab12cd34ef56.png
```

**5. 封装成 MCP 工具**：入参本地图片路径，出参 CDN 链接。之后 Agent 发帖、写文档时自己调用即可，“截图 → 上传 → 拿链接 → 贴文”全程无人工。

## 踩坑点

- **jsDelivr 单文件上限 20MB**，超了不服务，长图录屏先压缩；
- **分支名引用有缓存问题**：同名覆盖更新后 CDN 会继续吐旧图。用内容哈希命名、永不覆盖就绕开了；确实要更新就换文件名或去官方 purge；
- **上传后立刻访问可能 404**：回源同步有几秒延迟，自动化里加一次 2 秒重试；
- **中文文件名和空格**：URL 编码全是幺蛾子，坚持全 ASCII 哈希名最省心；
- **仓库必须公开**，等于图片完全公开，隐私截图先脱敏；
- **jsDelivr 不是网盘**：官方定位是加速开源资源，量别失控，个人/社区配图规模没问题；
- **国内可达性有波动**：备好 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 两个备用域名，在脚本或前端里做降级。

## 可复用建议

1. **内容哈希命名是核心决策**：天然去重（同一张图重复上传直接返回旧链接）、缓存永久不可变、链接永不失效；
2. 仓库里维护一个 `manifest.json`，记录文件名、用途、尺寸、标签，Agent 检索历史素材时直接读它，不必挨个看图；
3. MCP 工具同时返回主域名和备用域名，渲染端按可用性自动切换；
4. Token 放环境变量，fine-grained + 单仓库授权，泄漏影响面可控；
5. 定期跑对账脚本：manifest 与仓库实际文件做 diff，防止自动化 bug 堆积垃圾文件。

## 总结

这套方案本质是“GitHub 做存储、jsDelivr 做分发”：零成本、全链路可自动化、URL 稳定。它不适合大规模生产分发，但对个人博客、社区帖、Agent 产出配图这个量级，够用且省心。工程上真正重要的就两条：用内容哈希保证不可变，用 MCP 封装让 Agent 可直接调用；再接受国内可达性的偶发波动并做好降级，免费图床这件事就算落地了。

---

