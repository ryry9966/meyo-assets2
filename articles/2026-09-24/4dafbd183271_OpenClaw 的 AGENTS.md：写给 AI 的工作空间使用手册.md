---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 38788
source: 综合讨论
publishedAt: 2026-09-24
---

# 背景

OpenClaw 的 agent 每次会话都是"失忆"重启：模型上下文清空，唯一跨会话稳定存在的，是工作空间里那几个 Markdown 文件。其中 `AGENTS.md` 每次会话都会被注入上下文——`SOUL.md` 决定它是谁，`AGENTS.md` 决定它怎么干活。

类比一下：新同事入职，你不会只说"好好干"，而是给一份 onboarding 文档：环境怎么配、常用命令是什么、哪些目录不能碰。AGENTS.md 就是这份文档，只是读者换成了 AI。

# 问题

不写或乱写 AGENTS.md 的典型症状：

- 每次都要重新交代项目路径、包管理器、部署方式；
- agent 擅自重装依赖、改了不该动的配置；
- 同一个坑反复踩，因为没有沉淀机制；
- 或者反过来：把所有想法都堆进去，文件几千字，关键规则被稀释，模型注意力跟不上。

# 做法

我的 AGENTS.md 控制在 80–120 行，分五段：

1. **环境**：Linux 还是 macOS、项目根目录、pnpm/uv、Python/Node 版本。一行一条，可直接执行。
2. **Runbook**：高频操作的准确命令。跑测试、重启服务、查日志，直接给命令而不是描述。
3. **护栏**：禁止项与需确认的操作。写具体行为，如"git push 前必须确认""不要动 nginx 配置"，别写"要小心"。
4. **记忆协议**：规定何时写 MEMORY.md。例如"每次踩坑修好后，把结论按 现象/原因/处理 三段追加进去"。
5. **指针**：人格指向 SOUL.md，工具细节指向 TOOLS.md。AGENTS.md 只放过程性规则，不和人格文件抢职责。

迭代方式比初版更重要：每次你纠正 agent，都让它自己把教训写进对应文件，你只做 review。这是 OpenClaw 工作空间最划算的用法。

# 踩坑点

- **写成散文**。"希望你乐于助人"这类话零信息量，每条规则都应能对应到具体行为差异。
- **文件太长**。超过 150 行就开始稀释关键指令。任务级的拆进 skills，一次性的进 memory。
- **和 SOUL.md 打架**。语气要求写两处且不一致，行为就会漂移。人格归 SOUL，流程归 AGENTS。
- **塞敏感信息**。token、内网地址不要写进去，workspace 建议 git 管理，敏感项走环境变量。
- **改完不重开**。AGENTS.md 在会话开始时读取，中途改完要 `/new` 才生效。以为改了就生效、结果 agent 按旧规则跑，这个坑我踩过不止一次。

# 可复用建议

- workspace 放进 git，每次修改有 diff、可回滚；
- 验收标准一句话：新会话的 agent 只靠这个文件接手，能不能不出错；
- 每月 review 一次，删过时规则——规则库和代码一样会腐烂；
- 护栏类规则宁可啰嗦也要具体，风格类规则宁可简短。

# 总结

AGENTS.md 是 OpenClaw 里投入产出比最高的文件：不写它，你每次会话都在重新做入职培训；写好它，纠正一次就永久生效。把它当配置不如当文档——像维护 README 一样维护它，agent 的表现会稳定得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/d5945ec41845a2db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/fbdc17176b14bd2d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9b86cea1e449da9c.png)

