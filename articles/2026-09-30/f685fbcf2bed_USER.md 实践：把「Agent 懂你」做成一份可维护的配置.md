---
title: USER.md 实践：把「Agent 懂你」做成一份可维护的配置
feedId: 39895
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 这类常驻 Agent 的上下文不是凭空来的。会话启动时，workspace 里的几个 Markdown 文件会被注入 system prompt：AGENTS.md 约束行为，SOUL.md 定语气，而 USER.md——很多人的 workspace 里它还是空的——负责回答最基础的一个问题：对面这个人是谁。

## 问题

没有 USER.md 的 Agent 是这样工作的：每个新会话都问一遍“你用什么系统”“项目在哪个目录”；默认给 Debian 的命令，实际跑在 macOS 上；写个临时脚本也给你上完整的工程化配置。时区、命名习惯、常用工具全靠猜，“了解你”完全依赖 memory 命中的运气。

常见反模式是把人和机器的信息混写进 AGENTS.md。行为约束要跟着人迁移，环境事实要跟着机器迁移，两者应该分开：AGENTS.md 解决“怎么做事”，USER.md 解决“为谁做事”。

## 做法

我的 USER.md 控制在一屏以内，分四段：

```markdown
# About the user
## Environment
- OS: Ubuntu 24.04, shell: zsh, editor: neovim
- ~/work 放正式项目，~/lab 放实验性代码
## Preferences
- 中文交流，代码与报错信息保持英文
- 命令给可复制的整段，不要拆成多段解释
## Current focus
- 正在做网关压测，目标 p99 < 200ms
## Boundaries
- 不要主动 git push；rm -rf 类命令先复述再执行
```

几个要点：

- **环境事实放最前**：OS、shell、目录结构、硬件，这是被引用频率最高的部分。
- **偏好写成可执行规则**：不说“我喜欢简洁”，说“回答不超过 10 行，除非在 debug”。模糊形容词 Agent 没法执行。
- **Current focus 每周更新**：这是唯一需要持续维护的段落，其余基本不动。

## 踩坑点

- **把 secrets 写进去**。USER.md 会进每一次请求的上下文，API key、内网地址、客户名都不要放。我把 workspace 用 git 管理，敏感内容靠 `.gitignore` 和 review 习惯双保险。
- **写成简历**。几百字的自我介绍每次会话都在烧 token，且大半用不上。超过一屏就该砍。
- **与 AGENTS.md 冲突**。两边都定义“如何回复”，Agent 行为会抖动。规则归 AGENTS.md，事实归 USER.md，重叠处以 AGENTS.md 为准并写明。
- **只写不验证**。USER.md 是 prompt 的一部分，不是日记。写完跑两个真实任务，观察 Agent 是否真的引用了里面的信息，没被引用的段落直接删。

## 可复用建议

- 模板化后放进 dotfiles，换新机器十分钟就能恢复 Agent 对你的全部认知。
- 让 Agent 参与维护：在 AGENTS.md 里约定“当我纠正你时，把结论追加到 USER.md 对应小节”。把纠错从重复口播变成一次性沉淀。
- 用 git 管版本，`git log` 天然就是你偏好的变更史，回滚也方便。

## 总结

USER.md 的价值不在文件本身，而在于把“Agent 懂我”从玄学变成一份可 review、可 diff、可迁移的配置。它不提升模型能力，但能消掉绝大部分重复沟通成本。如果你的 workspace 里它还是空的，建议花二十分钟写第一版——从环境事实加三条硬性偏好开始，就够用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/87fffa3e9bf47d52.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/ec6e8335f68f9fa9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/b27202280bf9c856.png)

