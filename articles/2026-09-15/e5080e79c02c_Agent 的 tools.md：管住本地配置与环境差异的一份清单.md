---
title: Agent 的 tools.md：管住本地配置与环境差异的一份清单
feedId: 37665
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

跑 Agent 时间长了，很少只守着一台机器。笔记本、家里的小主机、CI 容器，各自装的工具不一样：这台有 ffmpeg，那台只有 ImageMagick；MCP server 一个走本地 stdio，一个走远端 SSE；Python 入口有的叫 `python`，有的叫 `python3`。Agent 的真实能力边界，就是由这些本地事实决定的。

## 问题

最常见的做法是把路径和版本硬写进 system prompt，或者干脆靠 Agent 自己 `which` 探测。前者换台机器就崩，后者每次任务都要试错，工具缺失时还容易幻觉出一个不存在的命令。而配置本身散落在 shell rc、MCP JSON、direnv 里——人都不一定说得清，何况模型。

## 做法

我的方案是给每个工作环境维护一份 `tools.md`，作为 Agent 可读的环境清单：

1. **分层放置**：全局放 `~/.agent/tools.md` 记录机器共性；项目根放 `tools.md` 记录项目依赖，项目级覆盖全局。
2. **固定结构**：环境概述（OS、包管理器）、工具清单（路径 + 版本 + 用途）、已知不可用项、关键环境变量名（只写名字不写值）、最后更新日期。
3. **脚本生成，人工补充**：写了个 `doctor.sh` 扫 PATH、核对版本，输出 markdown 片段，人工再补注意事项。换机或每月重跑一次。
4. **显式接入 Agent**：在 system prompt 里写明“调用工具前先查 tools.md，缺工具时按文件里的 fallback 处理”，不要指望它自己发现这份文件。
5. **模板入库，实例本地**：仓库里放 `tools.md.example`，真实文件进 `.gitignore`，避免机器私有信息被提交。

## 踩坑点

- **过期比缺失更糟**：Agent 对写下来的信息深信不疑，残留的旧路径会让它反复撞墙。日期戳和定期重生成是底线。
- **别把密钥写进去**：tools.md 会被读进上下文，环境变量只登记名字，值留在真正的 env 里。
- **别和 MCP 配置重复**：server 在 JSON 里声明就够了，tools.md 只标注“哪些 server 在这台机器可用”，做引用不做复制，保持单一事实源。
- **控制篇幅**：只写 Agent 真会调用的工具，超出这个范围的内容基本是在浪费上下文窗口。

## 可复用建议

- `doctor.sh` 的思路可以推广：凡是能被脚本校验的环境事实，都不该手写。
- 加一条 CI 检查：diff tools.md 与实测输出，不一致即报错，防止文件悄悄腐化。
- 多机用户可以按 host 分节，标注“仅本机”，Agent 切换环境时先读对应小节再动手。

## 总结

tools.md 做的事情很朴素：把“环境差异”从 Agent 每次的试错成本，变成一份维护成本更低的显式文档。它不解决能力问题，只解决事实问题——但在实际使用里，恰好是这一步，消掉了大部分“在我机器上是好的”类型的故障。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/daa75b2b629da2e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/c4816bc3a83345eb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/0ee0122f4cddf51d.png)

