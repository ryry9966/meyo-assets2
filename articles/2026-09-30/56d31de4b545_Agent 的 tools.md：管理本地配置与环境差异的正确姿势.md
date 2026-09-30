---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 39877
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

跑 Agent 的工作区往往不止一台机器：日常笔记本、家里的服务器、CI 容器。skills 和 prompt 可以用 git 同步，但环境事实没法同步——macOS 上是 brew，Linux 上是 apt；ffmpeg 有的机器在 `/usr/local/bin`，有的根本没装；代理、端口、Python 版本各不相同。

结果就是同一个 skill 在 A 机器跑得顺，到 B 机器 Agent 开始瞎猜：用错包管理器、调用不存在的命令、撞上被占的端口。这些试错每次都烧 token，还拖慢任务。

## 问题

Agent 对环境的认知只能来自上下文。你没告诉它的事实，它就靠常识补——而"常识"在你的机器上经常是错的。把环境细节写进 skill 不合适（换台机器就过期），提交到共享仓库又会污染队友。缺的是一个"每台机器一份、只描述本机事实"的载体，tools.md 就是干这个的：AGENTS.md 管"怎么干活"，tools.md 管"这里有什么"。

## 做法

**1. 定一个固定模板。** 只写事实和约束，标题固定方便 Agent 定位：

```markdown
# tools.md（本机环境事实）
## 系统与 shell
## 运行时（python/node/uv 版本与路径）
## 包管理器（brew/apt/pipx 怎么用）
## 常用 CLI（ffmpeg/ripgrep 是否存在、在哪）
## 服务与端口（占用约定）
## 网络与代理
## 已知限制（别尝试什么）
```

**2. 用脚本生成初稿。** 手写容易漏。让 Agent 或 bootstrap 脚本跑一遍 `uname -a`、`python3 --version`、`which ffmpeg` 之类，整理成首版，人工再裁剪。

**3. 分层与 git 策略。** 仓库只提交 `tools.md.example`（模板加示例），真实的 `tools.md` 进 `.gitignore`，每台机器自己生成。workspace 级放通用事实，项目级只放差异项。

**4. 接入加载链路。** 在 AGENTS.md 里写明"开始任务前先读 tools.md"。否则它只是个普通文件，Agent 不一定主动看。

**5. 带维护约定。** 装了新工具顺手更新；关键条目标注"最后验证时间"——过期信息比没有更糟。

## 踩坑点

- **写成说明书。** 大段叙述 Agent 抓不住重点，保持条目化、可 grep。
- **塞密钥。** tools.md 会进上下文。密钥只写变量名（"API key 在 env KEY_X"），值放环境变量。
- **写死绝对路径。** 机器一变就废，优先写"命令是否存在 + 用 which 核实"。
- **只写不删。** 文件越长越占上下文预算，定期让 Agent 做一次"环境审计"，对照现实核对并清理。
- **和 skills 抢地盘。** 边界不清楚就会出现两处漂移，tools.md 只做环境事实的单一来源。

## 可复用建议

- 把生成脚本和 example 模板放进仓库，新机器十分钟完成冷启动。
- 关键工具留一行验证命令（如 `ffmpeg -version | head -1`），让 Agent 自行核实而非盲信文档。
- 换机器或重装系统后，第一件事跑重建流程，而不是等报错再补。

## 总结

tools.md 的本质，是把"环境事实"从 Agent 的猜测变成显式输入。它不解决能力问题，只解决信息问题——但这往往是最便宜、见效最快的一层。模板固定、脚本生成、git 隔离、定期审计，四件事做到位，同一套 skills 就能在你的几台机器上稳定复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/04846dddcc9fe007.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/b994c3c35385c78d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/db025515da1b13c7.png)

