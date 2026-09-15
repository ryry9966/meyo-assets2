---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 37646
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景：Agent 只信文档，不信猜测

做 Agent 与自动化实践久了会看到一个规律：模型能力是通用的，环境是私有的。同一套工作流，在笔记本上跑得通，放到家里的 Linux 小主机或 CI 容器里就报"命令不存在"。原因很简单——Agent 不靠猜，靠你给它的工具清单。在 OpenClaw 的 workspace 约定里，这份清单就是 tools.md：哪些工具可用、装在哪、怎么调用、失败怎么办。

问题在于，清单本质是"本机快照"，却常被当静态文档维护，于是漂移开始了。

## 三个典型问题

1. **文档与现实漂移**。tools.md 写着 `/usr/local/bin/uv`，重装后实际在 `~/.local/bin`。写错的清单比没有更糟，Agent 会沿错误路径反复失败。
2. **多环境无法复用**。macOS、Windows、Linux 三台机器三份清单，改一处忘两处。
3. **敏感信息混入**。为省事把 token、内网地址写进文档，随后被同步、被提交。

## 做法：模板 + 实例 + 生成

核心一句话：**能力声明进模板，环境事实靠生成，启动时合并。**

**第一步：分层。** 模板描述稳定能力与用法，进 Git；实例记录本机路径、版本，进 .gitignore。

```markdown
<!-- tools.template.md 片段 -->
## uv
- 用途: Python 依赖与脚本执行
- 调用: `{{UV_PATH}} uv run <script>`
- 兜底: 回退 python3 + venv
```

**第二步：生成实例**，而不是手写：

```bash
#!/usr/bin/env bash
# detect_tools.sh
{
  echo "# 本机实例 $(date +%F)"
  command -v uv     && echo "UV_PATH=$(dirname "$(command -v uv)")"
  command -v docker && echo "DOCKER=available"
} > tools.env
```

**第三步：启动时合并注入。** 在 OpenClaw 启动钩子或插件里先跑检测，把模板占位符替换为实例值再喂给 Agent。机器换了、工具升级了，重跑脚本即可，清单永不手编。

**第四步：约束条目格式。** 每个工具固定四件事：用途、解析后的调用方式、版本约束、失败兜底。面向模型写，讲究密度与确定性，别写成给人看的教程。

**第五步：校验闭环。** 让 Agent 首次用工具前自检一次（跑个 `--version`），或在 CI 里比对清单与真实环境，发现漂移即报错。

## 踩坑点

- **会话缓存**：Agent 缓存了旧清单，改完不生效。给文件加生成时间戳，或每次会话强制重载。
- **shell 差异**：PowerShell 与 bash 的引号、路径分隔符不同，条目里显式写明调用 shell，别让模型自己选。
- **清单过长**：tools.md 会吃上下文，只写真正会用的工具，十几个条目足够。
- **secrets 入档**：只写 `读 env: OPENCLAW_TOKEN`，永远不写值。
- **兜底缺失**：只写"有 docker"不写"没有时怎么办"，Agent 会卡在报错循环里，每条给一条替代路径。

## 可复用建议

- 团队共享模板并评审，个人维护实例且自动生成。
- 对不确定的工具标注"未验证"，比写错强。
- 把 detect 脚本与模板一起放进仓库根目录，形成"模板 + 生成器"最小组合，新机器十分钟接入。

## 总结

tools.md 的价值不在写了多少，而在和现实差多少。把它从手工文档变成"模板 + 生成 + 注入 + 校验"的流水线，环境差异就从每次踩坑的意外，变成启动时自动消化的常量。一次搭好，之后每台新机器、每次环境变更，都只是重跑一次脚本的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/92f83b68b716b834.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/a8f72ea15c2a29ac.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/11ba038891cd8e10.png)

