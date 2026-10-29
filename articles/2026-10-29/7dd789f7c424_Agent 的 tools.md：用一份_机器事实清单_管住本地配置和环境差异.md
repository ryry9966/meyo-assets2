---
title: Agent 的 tools.md：用一份"机器事实清单"管住本地配置和环境差异
feedId: 42909
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

跑 Agent（无论 OpenClaw 还是自建 harness）最大的隐性成本不是模型调用，而是环境差异。同一份任务提示词，在开发机能跑通，到家里的服务器就因为 `brew` 换成 `apt`、Docker 没装、端口被占而翻车。MCP 和插件解决了"工具怎么接"，但"这台机器上的工具长什么样"一直没有落点。社区里目前比较收敛的做法，是用一份 `tools.md` 承接这件事。

## 问题

痛点主要有三类：

1. **知识散落**：工具路径写在 shell 配置里，注意事项留在某次对话里，Agent 每次都要重新踩坑。
2. **硬编码漂移**：提示词里写死 `/usr/local/bin/xxx`，换台机器就失效；版本号写死后升级即过期。
3. **Agent 猜环境**：没有事实来源时，模型会自己"脑补"包管理器和路径，猜错就可能执行破坏性命令。

## 做法

核心原则一句话：**tools.md 是给 Agent 看的机器事实清单，不是给人看的文档。**

推荐结构：

```markdown
# tools.md
## Facts（可由脚本生成）
- OS: Ubuntu 22.04, 包管理器: apt
- python 由 uv 管理，版本查看: uv python list
- docker: 可用，需要 sudo

## Overrides（本机差异）
- 项目 A 的 venv 在 /data/venvs/a，不要用根目录 .venv
- 8080 被 frps 占用，服务统一用 8090

## Do / Don't
- 不要全局 pip install，一律走 uv
- 删除前先 echo 目标路径确认
```

落地步骤：

1. **全局一份，项目一份**：全局放 `~/.config/openclaw/tools.md` 描述机器级事实；项目根目录放项目差异，项目级覆盖全局。
2. **用脚本生成 Facts**：写一个 20 行左右的 `tools doctor`，输出 OS、包管理器、关键工具是否存在及调用方式，粘进 Facts 段。环境变更后重跑一次。
3. **写"怎么查"而非"是什么"**：不写 "python 3.12.1"，写 "版本查看：python --version"。前者过期即误导，后者永不过期。
4. **与 MCP 配置解耦**：tools.md 只描述本机约束和入口，MCP 工具的参数说明留给各自的 server manifest，不要抄第二遍。
5. **进 git，但本机差异单独放**：`tools.md` 提交共享，`tools.local.md` 进 `.gitignore`，放绝对路径、密钥位置等不宜分发的内容。

## 踩坑点

- **写成散文**。Agent 对长篇叙述的遵循度很差，Facts 必须是一行一条、可校验的陈述。
- **太长**。超过一屏，注意力被稀释，关键约束反而被忽略。控制在 60 行以内，历史交给 git。
- **只写不验**。约束没有配套验证命令就会腐化，每条 Don't 尽量配一条检查命令。
- **忘了更新**。从 conda 换到 uv 后没改 tools.md，Agent 按旧事实执行——比没有文档更糟。

## 可复用建议

- 把 `tools doctor` 做成 Agent 可调用的小工具，开场自动刷新 Facts。
- 团队场景下 review 重点看 Overrides 段，那是环境差异最容易失控的地方。
- 每条事实行内标注更新日期，成本极低但很有效。

## 总结

tools.md 的本质，是你和 Agent 之间关于"环境现状"的契约：事实机器可查、差异单独隔离、约束可验证。它不解决工具接入问题（那是 MCP 的活），但能让 Agent 在任何一台机器上少猜、少错、少重跑。花半小时写第一版，之后每次翻车往里补一条，两周就能看到收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/cbbe31d37375c0d6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/a15fc149dd0b60f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/e188fd3d87dea562.png)

