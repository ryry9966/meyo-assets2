---
title: Agent 的 tools.md：把环境差异写成一份可校验的契约
feedId: 42890
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

OpenClaw 的 workspace 用 AGENTS.md 约束行为，但“这台机器上的工具长什么样”是另一回事。Agent 真正干活靠 shell、MCP 工具和脚本，而每台机器都不同：macOS 还是 Linux、Homebrew 前缀、`python` 还是 `python3`、nvm/volta 的 init、`~/bin` 下的自研脚本、代理端口。这些知识模型不会凭空知道——你不写下来，它每次会话都重新猜。

## 问题

典型翻车现场：

- 只装了 `python3`，它熟练敲下 `python`；
- ffmpeg 是 Homebrew 装的，不在默认 PATH，报错后它开始瞎猜该装什么；
- GNU 版 `sed -i` 的用法在 macOS 上跑出意外结果；
- 上周口头教的部署脚本，这周忘得一干二净；
- 笔记本和服务器两套环境，口述经验来回漂移。

本质一句话：环境事实没有被**持久化、结构化、可校验**地交给 Agent。

## 做法

**1. 分层：机器事实进 tools.md，项目事实进项目文档。** workspace 里的 tools.md 是 Agent 每次会话都会读的“机器卡片”，只放与机器绑定的内容：解释器、路径、自研脚本、禁止动作。项目内的特殊约定放项目自己的 AGENTS.md，别混。

**2. 只写事实，写成条目。** 示例：

```markdown
## Interpreters
- python: 不存在，一律用 python3（3.12，pyenv 管理）
- node: volta 管理，默认 20

## Paths
- ffmpeg: /opt/homebrew/bin/ffmpeg
- deploy: ~/bin/deploy.sh（自研，只读，勿改）

## Don'ts
- 不要动 /etc/hosts；网络调试用本地映射脚本
```

**3. 让每条事实可校验。** 配一个 `env-doctor.sh`：`command -v python3`、`test -x /opt/homebrew/bin/ffmpeg`，逐条输出 OK/MISSING。让 Agent 首次接手机器或 heartbeat 时跑一遍，文档过期当场暴露。

**4. 跨机器用模板同步。** tools.md 放进 dotfiles 仓库，新机器只填差异项；可移植的逻辑下沉到脚本里，文档只“指路”，不实现逻辑。

## 踩坑点

- **写成长文。** Agent 对文档开头的遵循度明显高于结尾。关键事实放最前，条目化，一屏以内。
- **塞密钥。** tools.md 是明文且会进上下文，只写环境变量名，不写值。
- **过期条目。** 工具升级换路径后会反复失败。在 AGENTS.md 里写清规则：命令失败先跑 env-doctor，修文档，而不是连续重试。
- **越界实现。** 有人把整段 shell 逻辑抄进 tools.md，结果和脚本两份逻辑漂移。文档指路，脚本落地。
- **口述依赖。** 聊天里说过的话不是配置，“上次跟你说过”在下次会话里约等于不存在。

## 可复用建议

- 把 tools.md 当“契约 + 冒烟测试”：每条事实配一条检查命令；配不出检查的条目，多半不该出现在这里。
- 固定四段模板：Interpreters / Paths / Scripts / Don'ts，迁移新机器十分钟填完。
- 随 workspace 进 git，每次升级工具链顺手 review 一遍。

## 总结

tools.md 不是写给人看的文档，而是交给 Agent 的环境接口。环境差异永远存在，区别只是：要么花十分钟把它编码成一份可校验的短文档，要么让 Agent 每次会话重新发现一遍，还得承担猜错的成本。选前者。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/bb82c7e986a1acf2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/5a5427b39d1cdcb5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/a027b3abd48baac4.png)

