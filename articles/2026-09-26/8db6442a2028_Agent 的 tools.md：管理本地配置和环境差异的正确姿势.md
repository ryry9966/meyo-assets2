---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39072
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

跑本地 Agent（OpenClaw 这类常驻本机的框架）有个容易被低估的事实：模型对你这台机器一无所知。而我们大多数人至少有两三套环境——家里的 Mac、公司的 Linux 开发机、一台跑常驻任务的 VPS，包管理器、运行时、项目路径、代理配置各不相同。OpenClaw workspace 里的 tools.md，就是给 Agent 交代"这台机器长什么样"的入口文件。

## 问题

没有这份文件时，Agent 只能猜：在 Ubuntu 上用 brew、调用只存在于你 Mac 上的脚本路径，然后 `command not found`，反复试错烧 token。也有人把机器信息写进系统提示或 AGENTS.md，结果换台机器全错，还会随 git 同步互相污染；MCP server 按机器配置各不相同，Agent 也不知道当前会话里哪些真的可用。

核心矛盾在于：skills 和提示词需要跨机器复用，环境事实却严格属于单机。混在一起，两头都不对。

## 做法

把 tools.md 当作**单机环境清单 + 使用约束**来维护，与可同步内容严格分离。

1. **摸底一次**：花十分钟记录 OS/shell、包管理器、运行时版本、常用项目路径、代理、硬件边界（有无 GPU）、已配置的 MCP server 及指向。
2. **建档**：workspace 根目录建 tools.md，只写事实与短约束，不写教程：

```markdown
# 本机：macOS 15 / arm64
- 包管理：brew，位于 /opt/homebrew/bin，无 apt
- Python：统一走 uv，用 `uv run`，不要直接调 python3
- 项目根：~/work/openclaw-demo
- 无 GPU，重计算别在本机跑
- 代理：127.0.0.1:7890，curl 需显式 --proxy
- MCP：filesystem → ~/work；playwright 已装、未配 key，先别用
```

3. **单机化**：tools.md 不进 git（`.gitignore` 掉），或按机器存 `tools.home.md` / `tools.server.md`，用符号链接对齐文件名；git 里只留一份空模板。
4. **失败即修**：Agent 因环境报错（命令不存在、路径失效），当场把结论写回 tools.md，当成 bug fix 处理。环境漂移是常态，靠自觉定期审查不现实，靠失败驱动才可持续。

## 踩坑点

- **把密钥写进 tools.md**。它会被整段读进上下文，等于明文泄漏。只写引用——"密钥在 `~/.env.server`"，不写值。
- **写太长**。每行都是 token 开销。超过百行就该拆：长操作流程移到 skills 文件，tools.md 只留环境事实。
- **只写"有什么"，不写"没有什么"**。Agent 最容易栽在默认假设上。"无 GPU""playwright 未配 key"这类负向约束，比十条正向描述更值钱。
- **和 AGENTS.md、系统提示重复**。行为规范归行为规范，环境事实归 tools.md，保持单一事实来源。
- **文件过期了还在信**。换机器、升版本、改路径之后不更新，Agent 会被旧事实稳稳带偏。

## 可复用建议

- 固定模板字段：系统/Shell、包管理器、运行时、路径、网络/代理、硬件边界、MCP 映射、已知不可用项。**模板进 git，内容不进。**
- 写个十行的审计脚本：把 tools.md 里声明的命令挨个 `which` 一遍，输出差异，在换机器或月度检查时跑一次。
- 新机器接入流程固化成一条 skill："先读环境生成 tools.md 草稿，人工审核后启用"。

## 总结

tools.md 本质是给 Agent 看的"这台机器的规格表"。它不优雅，也没什么技术含量，但它把 Agent 最不擅长的环境猜测，换成了一次确定性读取。单机维护、极简、只写事实、带上负向约束、失败即修——做到这五点，跨机器跑 Agent 的"玄学问题"会少一大半。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/0b4554f03304e4eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/e382d08ba258480d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/3a5b320ef28a68f2.png)

