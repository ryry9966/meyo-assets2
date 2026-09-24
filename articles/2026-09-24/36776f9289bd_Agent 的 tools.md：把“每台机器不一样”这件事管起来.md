---
title: Agent 的 tools.md：把“每台机器不一样”这件事管起来
feedId: 38806
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

跑 Agent 久了都会遇到同一个尴尬：同一套 skill 和提示词，在公司 Mac 上好好的，到了家里的 Linux 主机或 CI 沙箱里就开始翻车——`python3` 变成 `python`，ffmpeg 没装，Docker socket 权限不通，代理地址也对不上。问题不在模型，而在于我们把“我这台机器长什么样”写死在了提示词里。

## 问题

环境差异本质上是三类事实的错配：

1. **工具清单**：哪些 CLI、MCP server 可用，怎么调用；
2. **路径与版本**：venv 在哪，包管理器是 apt 还是 brew；
3. **约束条件**：有无 GPU，是否走代理，哪些端口不能占。

这些事实散落在 `.bashrc`、`.env`、README 和提示词里，Agent 每次只能靠猜。猜错的代价是反复重试、装错依赖，甚至改坏文件。

## 做法

我的方案是给每个工作区放一份 `tools.md`，约定 Agent 在会话开始时先读它。结构分三层：

```markdown
# machine
- OS: Ubuntu 22.04 / apt；本机没有 brew，不要用
- GPU: 无，训练类任务请直接拒绝并说明原因
- network: 走 http://127.0.0.1:7890，pip 已配内网镜像

# toolchain
- ffmpeg: /usr/bin/ffmpeg（7.x）；缺失时用 apt 装，不要 snap
- docker: 可用，socket 权限已配好，无需 sudo

# project
- venv: ./venv，使用前先 source
- 测试: 只跑 pytest -q，全量跑会超时
```

三个要点：

1. **写“怎么查”而不只写“是什么”。** 比如“缺 ffmpeg 用 apt 装”，Agent 拿到的是动作，不是死数据。
2. **分层与覆盖。** 机器级事实放全局（如 `~/.config/openclaw/tools.md`），项目级放仓库根目录，项目覆盖全局。前者进私有 dotfiles 仓库，后者随代码提交。
3. **配套一个 doctor 脚本。** 几十行 shell，探测工具是否存在、版本是否匹配，输出 diff。环境变了跑一次，比手写可靠。

## 踩坑点

- **过期比缺失更危险。** tools.md 一旦写错，Agent 会深信不疑。加生成时间戳、用 doctor 校验，是底线。
- **防止 Agent 自己改它。** 我见过 Agent“好心”把幻觉出来的路径写回去。把它设成只读，或在首行注明“本文件由脚本生成，禁止编辑”。
- **别放密钥。** 它常被随手提交，顺手贴 API key 的事故我见过两次。只写名字，值进 `.env`。
- **控制篇幅。** 全塞进去会吃掉上下文预算。一条工具一个块，五行以内，教程链接也不要放。

## 可复用建议

- 模板固定四段：machine / toolchain / network / project。团队内格式统一，Agent 解析才稳定。
- 新机器初始化流程：跑 doctor → 生成草稿 → 人工删减 → 提交。
- 每条写成“事实 + 校验命令 + 缺失时的动作”三元组，这才是这份文件真正的价值。
- 与 MCP 配置对照维护：MCP 声明能力，tools.md 声明环境，两者不重复。

## 总结

tools.md 不是又一份文档，而是把“环境”变成 Agent 可读的契约。它对纪律性要求很高：小、准、可校验、不许 Agent 乱写。做到这四点，同一套 skill 在笔记本、家里主机和 CI 上都能跑通，排障时也多了一个可以 diff 的锚点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/742909b39161dcf5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/91c33ac0f2a037b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9c5d461df0b7e4fa.png)

