---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 41272
source: 综合讨论
publishedAt: 2026-10-12
---

# 背景

在本地跑 OpenClaw / Agent，最常见的低级失败往往不是模型能力问题，而是环境问题：想调 ffmpeg，路径不在 PATH；想跑 python3，机器上只有 conda 里的 python；macOS 和 Linux 连 `sed -i` 的参数都不兼容。于是每个会话 agent 都要重新"摸底"你的机器，靠猜测和试错逼近正确命令。tools.md 的思路很简单：把摸底结果固化成一个 agent 开局会读的文件，一次记录，反复复用。

# 问题：环境差异是常态，而 agent 默认不知道

- **平台差异**：macOS / Linux / WSL 下同名工具行为不同，Homebrew、apt、pacman 各有脾气。
- **路径差异**：pyenv、conda、nvm 都会改写 PATH，agent 拿到的 shell 环境未必和你终端一致。
- **习惯差异**：你用 pnpm，agent 默认 npm；你的 docker 其实是 podman 的别名。
- **配置散落**：这些事实埋在 .zshrc、README 和你脑子里，agent 默认一个都不读。

没有 tools.md 的典型症状：重复试错浪费 token、选错包管理器、在错误目录执行，甚至跑出 `brew upgrade`、`docker system prune -a` 这类你根本不想让它碰的命令。

# 做法：写一份"事实清单"，分两层管

1. **建两个文件**。用户级放个人机器事实，项目级放仓库根目录描述本 repo 的工具约定，项目级覆盖用户级。
2. **只写事实，不写教程**。推荐四段结构：
   - **Toolchain**：工具名 + 调用方式 + 约束。例："python 一律用 python3（pyenv 管理），别碰系统 python。"
   - **Paths**：非默认路径，如自定义 GOPATH、conda env 位置、需要 source 的脚本。
   - **Platform quirks**：本机特有的坑，如 macOS 下 `sed -i` 要加 `''` 占位、`date` 不可用需换 `gdate`。
   - **Do-not-run**：明确禁令，如"禁止执行 brew upgrade、docker system prune -a"。
3. **当作代码维护**。随 dotfiles 或仓库提交，环境一变顺手更新，文件头加一行最后核对日期。
4. **冒烟验证**。新会话直接问 agent："列出你已知的本机工具和禁忌命令"，确认它真的读到了。

# 踩坑点

- **写成散文**。agent 上下文不是无限的，控制在两屏以内；写不下说明你在写文档，不是事实清单。
- **过期信息比缺失更糟**。agent 会信任这个文件：文件说 python 3.10，环境已升到 3.12，它就拿着旧结论继续错。所以必须有核对日期和更新习惯。
- **别写一条命令就能查到的东西**。`ffmpeg -version` 能查到的版本号没价值；要写的是"查不到或查起来贵"的：别名、禁令、非默认路径、已知冲突。
- **别放密钥**。tools.md 常被随手提交，token、内网地址一律不出现。
- **团队仓库别带个人路径**。`/Users/yourname/...` 这类机器相关内容留在用户级，项目级只放可移植的约定。

# 可复用建议

- 把四段结构（Toolchain / Paths / Platform quirks / Do-not-run）做成模板，新机器二十分钟能填完。
- 装新工具、改 PATH、升大版本之后，把"更新 tools.md"当作收尾动作，像写 CHANGELOG 一样对待。
- 团队场景：项目级 tools.md 随仓库走，新人 clone 即得；个人差异留在本地层，避免互相污染。
- 如果 agent 权限体系支持，把 Do-not-run 的禁令同时落到权限配置里——文档管习惯，配置管底线。

# 总结

tools.md 的价值不在文件本身，而在把"agent 猜环境"变成"agent 读环境"。写事实、分两层、跟着环境更新，一次性成本不到一小时，换来的是每次会话少一轮试错、少一次误操作。对天天和 agent 打交道的工程实践来说，这是性价比最高的环境治理动作之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/87ab3e116c73489c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/2804128097cba9ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/cfe0ec47dbf100f5.png)

