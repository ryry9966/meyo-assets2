---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40819
source: 综合讨论
publishedAt: 2026-10-07
---

# 背景

跑 OpenClaw 的 Agent 时，最容易被低估的不是模型能力，而是"本机常识"。Agent 知道 `brew` 和 `apt` 的区别，但不知道你家里那台服务器是 Debian 还是 Ubuntu，不知道服务是 systemd 管的还是 docker compose 起的，不知道部署脚本叫 `deploy.sh` 还是某个你三年前写完就忘的 make target。它只能猜，猜错一次就浪费一轮对话。

这就是 workspace 里 `TOOLS.md` 的用途：一份写给 Agent 看的本机说明书。但它经常被两种极端用法毁掉——要么空着，要么被当成第二个系统提示词塞满所有东西。

# 问题

我在三台机器（笔记本、家庭服务器、一台 VPS）上跑同一个 workspace，踩过的坑：

- Agent 在笔记本上学会的命令，到服务器上路径全错；
- 改了端口没更新文档，Agent 反复用旧端口失败，还"贴心"地建议我回滚；
- 早期图省事把 token 直接写进了 TOOLS.md，同步 workspace 时差点泄漏。

# 做法

我的原则是四条：只写事实、可验证、不写秘密、进版本管理。

**1. 分层组织。** TOOLS.md 只放所有机器通用的内容：包管理器习惯、通用约定（比如"重启服务用 `systemctl --user`，别用 sudo"）。机器差异单独拆成 `tools.<host>.md`，并在文件头用一小段说明告诉 Agent 按主机名读取对应文件。

**2. 每条带验证命令。** 与其写"服务跑在 8080"，不如写"服务应在 8080，验证：`curl -s localhost:8080/health`"。Agent 会先验证再行动，失败时也有明确的排错起点。

**3. 秘密只写指针。** 写"token 在密码管理器的某个条目"或"在 `~/.config/xxx/env`"，绝不写值本身。内网拓扑可以写，凭据不行。

**4. 用脚本生成机器差异部分。** 我写了一个 `tools-gen.sh`，从当前系统探测发行版、服务管理方式、关键路径，生成 markdown 片段填入 TOOLS.md 的专属节。新机器初始化只需跑一次。

# 踩坑点

- **别把它当 prompt。** 塞满"回复风格""注意事项"之后，真正重要的环境事实反而被稀释。行为规范归 AGENTS.md，事实归 TOOLS.md，两者别混。
- **过时比缺失更糟。** 错误的文档会让 Agent 自信地失败。我在文件头维护一个"最后验证日期"，并定期让 Agent 逐条跑验证命令、汇报失效项——这本身就是个很好的自动化巡检任务。
- **别让 Agent 免费改这份文件。** 允许它提议修改，但走 git，人工 review 后再合入。否则它会"顺手"把自己的错误结论写进去，形成自我强化的幻觉。

# 可复用建议

- 起步模板四节就够：环境概况 / 常用命令 / 部署流程 / 禁止事项。
- 一条内容准入标准：能复制到终端直接执行、且配一句话说明原因，才配写进去。
- 多机同步用 git 做主干 + 生成脚本处理差异，不要手工维护多份副本。

# 总结

TOOLS.md 的价值不在长短，而在准确率和可信度。一份精简、可验证、无秘密、有版本历史的说明书，才能让 Agent 在任何一台机器上少猜一次、多对一次。花一小时整理它，回报是之后每一次对话的确定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/5d30037fbaf0980f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/bd74b4370641662d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/ae12c34f1fc184e8.png)

