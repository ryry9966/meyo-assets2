---
title: Agent 的 TOOLS.md：管理本地配置和环境差异的正确姿势
feedId: 41292
source: 综合讨论
publishedAt: 2026-10-12
---

# TOOLS.md：把“这台机器”讲给 Agent 听

## 背景

OpenClaw 的 workspace 里有一组分工明确的 Markdown 文件：AGENTS.md 管行为规范，SOUL.md 管人格语气，MEMORY.md 管长期记忆。其中 **TOOLS.md 的定位最容易被误解**——它不是工具清单，而是“本机说明书”：这台机器上哪些命令能用、怎么用、有什么坑。

模型知道 Docker 是什么，但不知道你机器上的 docker 需不需要 sudo；知道虚拟环境的通用用法，但不知道你的 venv 在哪个目录、用 uv 还是 pip。这些差异不在任何预训练语料里，只能靠我们显式写下来。

## 问题

TOOLS.md 缺失或写得不好时，常见的失败模式：

- Agent 默认用 npm，而你的仓库统一用 pnpm；
- 起 docker compose 时权限报错，来回试错烧掉大量 token；
- 换机器或升级环境后，旧的隐含约定全部失效，每次都要在对话里重新解释；
- 这些信息被随手塞进 MEMORY.md，和用户偏好、项目进度混在一起，越积越乱。

本质问题：**环境相关的事实，缺一个明确、稳定、会被注入上下文的落点。**

## 做法

1. **确认落点**。TOOLS.md 放在 workspace 根目录（默认 `~/clawd`，具体以 openclaw.json 里的 workspace 配置为准），随会话注入上下文。
2. **按域分区，不写散文**。推荐骨架：

```markdown
# Machine
# Shell & Paths
# Languages & Version Managers
# Containers
# Network / Proxy
# Custom Scripts
# Known Quirks
```

3. **一条一个事实，附可执行命令**。写“验证过的原话”，而不是描述性文字：

```markdown
- docker 需 sudo：sudo docker compose up -d
- Python 统一走 uv：uv run python xxx.py
- 出网走代理：export HTTPS_PROXY=http://127.0.0.1:7890
```

4. **控制体量**。它每个会话都占 token，目标 100 行以内；仓库级约定放到各项目自己的 AGENTS.md，别都堆在这。
5. **建立回写循环**。Agent 试出一条可行命令（比如需要 `sudo -E`）时，让它把结论追加到 TOOLS.md 对应分区，你只负责 review。workspace 用 git 管理，改动可追溯。
6. **多机策略**：每台机器一个独立 workspace，共享部分抽成模板文件同步，不要多机共用同一份 TOOLS.md。

## 踩坑点

- **当记忆桶用**：偏好、日程、项目进度请去 MEMORY.md / USER.md，TOOLS.md 只回答“这台机器怎么用”。
- **过期信息比没有更糟**：venv 改了名没更新，Agent 会按错误命令反复失败。建议定期让 Agent 自己通读一遍，标出“与当前环境不符”的条目。
- **写入敏感信息**：TOOLS.md 会进模型上下文，token、密码、内网地址别写；只写“密钥在 shell 环境变量 `XXX_KEY` 中，直接引用即可”。
- **复制通用知识**：“Docker 是一种容器技术”这类内容是纯浪费，只写差异。

## 可复用建议

- 每条记录自问：换个新同事接手这台机器，这条能让他少踩一个坑吗？不能就删。
- 写个十几行的环境探测脚本（uname、shell、python/docker 版本、代理状态），新机器初始化时跑一遍，让 Agent 据此生成初版 TOOLS.md。
- 分区名事先约定好，git 同步时的冲突会少很多。

## 总结

TOOLS.md 的价值不在“写了多少”，而在“每一条都省掉一次试错”。把它当成给 Agent 的本机 README：**短、准、只写差异、可执行、勤维护**。环境差异不会消失，但可以被一份克制的文档稳定吸收。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/c3962dc7b3ba11b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/a3064be6a341d11c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/1c7b5a2e40732174.png)

