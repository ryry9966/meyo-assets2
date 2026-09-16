---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 37860
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 这类常驻 Agent 的执行环境就是你本机：跑命令、调脚本、操作文件。模型对"一般 Linux 该怎么装东西"很熟，但它不知道你的机器是 macOS 还是 Ubuntu、Python 在哪个 venv、npm 走不走镜像、Docker 装没装。tools.md 的定位很简单：一份 Agent 每次工作前都会读的"本机说明书"，把环境事实写清楚，替代它的猜测。

## 问题

实际用下来，环境差异导致的失败集中在几类：

- **平台差异**：让 Agent 装依赖，它在 macOS 上给你 `apt-get`，或者反过来。
- **工具链版本**：`python` 指向 2.7、node 没走 nvm、conda 没激活，命令对了也跑不对。
- **网络环境**：不走代理/镜像时 pip、npm 超时，Agent 反复重试，白白烧 token。
- **路径与约定**：项目放哪、日志在哪、启动脚本是哪个，全靠猜。

共同点是：**信息存在，但不在 Agent 的上下文里**。报错后它也能试错纠正，但代价是多轮往返，甚至误操作。

## 做法

我的 tools.md 固定几块内容，供参考：

```markdown
# 本机环境（macOS 14 / Apple Silicon / zsh）
- 一律用 python3；项目统一 source ~/work/.venv/bin/activate
- node 走 nvm：先 nvm use 20，不要直接 npm -g 装东西
- 拉包走镜像，配置在 ~/.zshrc，不要改全局设置
- docker 已装；服务日志在 ~/logs/<service>/
- 改系统配置前必须先问我

# 常用命令
- 跑测试：cd ~/work/api && make test
- 重启服务：make restart && make logs
```

几个原则：

1. **写事实和命令，不写散文**。祈使句优先，"一律用 python3"比"通常建议 python3"有效得多。
2. **控制篇幅**。tools.md 会进上下文，我保持在 60 行以内，超了就砍低频内容。
3. **留验证命令**。每条约定配一个能自检的命令（如 `which python3`），让 Agent 开工前先跑一遍环境自检。
4. **多机差异用覆盖**。维护一份 base tools.md，每台机器一个 diff 文件（如 `tools.macbook.md`），换机器只换 overlay。

## 踩坑点

- **写成百科全书**。第一版我写了 200 行，Agent 明显开始漏看关键约束；精简到 60 行后遵守率高很多。
- **把密钥写进去**。tools.md 会被读取、可能被打进日志。密钥只留路径引用："凭据在 ~/.config/x，别 cat"。
- **信息过期**。升级工具链后忘了同步，Agent 按旧约定执行反而出错。我在文件头写了"最后验证日期"，升级后顺手改。
- **和 AGENTS.md 抢职责**。工具事实归 tools.md，行为规范归 AGENTS.md，两边都写必然漂移。
- **放错位置**。它必须在 Agent 加载的 workspace 里。改完先问一句"你读到了哪些本机约定"，验证生效。

## 可复用建议

- 把 tools.md 当"机器的 README"维护，进 git，但提交前 grep 一遍密钥。
- 每个工具写三行：什么版本、怎么激活、怎么验证。
- 新环境初始化时，先让 Agent 执行自检命令、把结果回填进 tools.md，比手写快且准。
- 团队场景：base 模板共享，机器差异各自维护，约定变更走 review。

## 总结

tools.md 不是什么高深机制，价值全在"把隐性环境知识显性化"。写它的过程本身也会逼你梳理清楚自己机器的真实状态。经验浓缩成四个词：**短、具体、可验证、定期维护**。做到这几点，Agent 在你机器上的执行准确率会有非常直观的提升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/062733d3769fbc42.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/e33c344f2576e706.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/059de30b39b4d0ef.png)

