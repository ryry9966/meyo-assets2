---
title: 给 Agent 一份诚实的地图：用 TOOLS.md 管好本地配置与环境差异
feedId: 41522
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的 agent 跑在你自己的机器上，workspace 里的几个 Markdown 文件是它理解世界的主要来源：`AGENTS.md` 管行为准则，`SOUL.md` 管人格，而 `TOOLS.md` 的定位很朴素——告诉 agent「这台机器到底是什么样的」。但很多人的 TOOLS.md 要么空着，要么成了随手堆命令片段的杂物间。

## 问题

没有一份可信的环境说明时，agent 只能靠猜和试：

- **路径猜错**：`/Users/xxx/work` 还是 `/home/xxx/projects`？
- **工具链不一**：笔记本上 pyenv + venv，VPS 上系统 Python，台式机用 conda；
- **shell 差异**：你日常用 fish，agent 默认输出 bash 语法；
- **网络环境**：某些命令要走代理，某些直连。

结果是反复试错、烧 token，偶尔还会执行错误动作——比如在没有 venv 的机器上把包装进系统环境。笔记本、台式机、VPS 多机并用时，这些问题被成倍放大。

## 做法

**1. 把 TOOLS.md 当「环境事实清单」，按机器分节：**

```markdown
# TOOLS.md

## macbook-dev（日常开发）
- OS: macOS 15, 包管理 brew
- Python 走 pyenv，项目内用 `pyenv exec`，禁止直接 pip install
- 项目根目录: ~/work/openclaw-space
- 已知坑: fish shell，复杂命令建议显式 bash -c

## vps-tokyo（部署）
- OS: Ubuntu 22.04, apt, Docker Compose v2
- 无代理，直连
- 禁区: 不要动 nginx 配置，除非明确要求
```

**2. 写事实，不写愿望。**「Python 3.12 已装」是事实；「请小心操作」是行为规范，后者归 AGENTS.md。

**3. 可选：写一个十行的 `doctor` 脚本**，输出 JSON 格式的 OS、Python 路径、磁盘、代理变量。agent 拿不准时先跑它，比硬编码更抗过期。

**4. 用 git 管理，但绝不写秘密。** token、密码、内网地址一律不落文件，只写环境变量名占位。

## 踩坑点

- **把秘密写进去**。workspace 文件会进入上下文，泄漏面比你想象的大。
- **越写越长**。这份文件每次都会被注入，控制在一屏内，过期条目立即删。
- **过期比缺失更糟**。养成回写习惯：每次手动修完一个环境问题，顺手让 agent 更新对应条目。
- **机器标识不可靠**。hostname 会改，用「机器名 + 用途」这种稳定标识分节。
- **职责混装**。环境事实进 TOOLS.md，行为准则进 AGENTS.md，个人偏好进 USER.md，别互相塞。

## 可复用建议

- 模板固定为：机器节 → 工具链 → 关键绝对路径 → 已知坑 → 禁区；
- 「已知坑」当 changelog 写，带日期，方便判断新旧；
- 新机器初始化第一件事：让 agent 跑 doctor 脚本，生成初版 TOOLS.md；
- 每月让 agent 对照脚本输出 diff 一次，清理过期内容。

## 总结

TOOLS.md 的价值不在于写了多少，而在于 agent 还需要猜多少。把环境差异从对话里挪进文件里，agent 的动作会明显变稳。它不神秘，本质上是一份需要你持续维护的、诚实的环境快照——而这点维护成本，远低于每次会话里的反复试错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/dbd021eec9112c1d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/93392a339da454ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/54f07066651808e1.png)

