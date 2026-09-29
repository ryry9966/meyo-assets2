---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39522
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

跑本地 Agent 的人大多遇到过这个场景：同一套任务在自己机器上好好的，换到服务器或同事的 WSL 里就翻车，Agent 找不到 `uv`，用 `apt` 去装 macOS 的包，或者把输出写进一个不存在的路径。原因很朴素：Agent 对环境的认知靠猜测，而真实环境是异构的。

OpenClaw 工作区里的 `tools.md` 就是用来弥合这个差距的：它是 Agent 每次会话开局会读的"环境说明书"。但不少用法停留在"想到什么写什么"，时间一长，它就变成第二份没人维护的 README。

## 问题

常见的失控模式有三种：

1. **事实手写，迅速过期。** 手填的 Python 版本、模型路径，两周后就成了误导源。
2. **单文件扛所有环境。** 团队共识和个人机器差异混在一起，要么污染仓库，要么干脆不提交。
3. **和系统提示词重复。** 同一条约定在 tools.md 和 prompt 里各写一遍，改了一处忘另一处。

## 做法

我们的实践是把 tools.md 当作"分层的、部分生成的环境清单"。

**第一步：分层。** 仓库根放 `tools.md`，写团队共识：用什么包管理器、测试入口、禁止操作。旁边放 `tools.local.md` 并加入 `.gitignore`，写本机差异：路径、代理、GPU、MCP server 的可用性（headless 服务器上浏览器工具不可用这种事，Agent 自己是猜不出来的）。加载顺序约定为 local 覆盖 repo。

**第二步：易变事实用脚本生成。** OS、shell、工具版本不该手写。放一个 `scripts/gen-env.sh`，每次会话前刷一次：

```bash
{
  echo "## Generated env facts"
  echo "- os: $(uname -s) $(uname -r)"
  echo "- python: $(uv python find 2>/dev/null || echo missing)"
  echo "- git: $(git --version 2>/dev/null || echo missing)"
} > tools.local.md.tmp && mv tools.local.md.tmp tools.local.md
```

**第三步：写约定，不写教程。** 每个工具三行以内：名字、调用方式、一句注意点。细节外链到 docs，别塞进 tools.md。

**第四步：加一段自检。** 列出 Agent 可直接运行的验证命令（`which uv`、`test -d data/`），让它在动手前确认假设，而不是失败后再猜。

## 踩坑点

- **别把密钥写进去。** tools.md 会被读进上下文，可能随日志落盘。只写变量名（如 `API_KEY from env`），值留给 `.env`。
- **控制长度。** 每次会话都在消耗上下文，超过 100 行就该砍，我们一般压在 60–80 行。
- **层级冲突要显式裁决。** 文件里明确写"local 优先"，否则 Agent 遇到矛盾指令时行为不可预测。
- **别假设它一定被读。** 在启动流程里固定加载路径，并用一个探测问题（"当前 Python 管理器是什么"）验证过一次链路。

## 可复用建议

- 提交一份 `tools.example.md` 作模板，区块固定五段：Generated facts / Tool registry / Conventions / Forbidden / Self-check。
- 把 `gen-env.sh` 挂进项目 bootstrap，让"过期"变成结构上不可能，而不是靠自觉。
- 每月修剪一次：一个月没被 Agent 用到的条目先删掉，需要时再加回来。

## 总结

tools.md 本质是给 Agent 的"配置即文档"：团队共识进仓库，机器差异进 local，易变事实靠生成，关键假设靠自检。把它当代码维护——有模板、有生成脚本、有修剪节奏，Agent 在不同环境里的行为才会从碰运气变成可预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/71b6020642ff1dde.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/dec4a181bdf2cf2d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/67675c33ebb14c06.png)

