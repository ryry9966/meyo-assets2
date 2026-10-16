---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41766
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 的 Agent 长期运行时面对的是一个有限的上下文窗口。早期的做法是把所有工具说明、操作手册全部塞进系统提示词，结果是上下文很快被占满，模型注意力被稀释，简单对话也变得又慢又贵。

Skills 机制借鉴了渐进式披露（progressive disclosure）的思路：技能平时只以「名字 + 一句描述」的形式存在于上下文中，只有当 Agent 判断当前任务相关时，才加载完整的 SKILL.md 正文。本质上是给能力做懒加载。

## 问题

按需加载成立的前提是「触发得准」。实践中常见三个矛盾：

1. 描述写得太泛，技能永远不被触发；写得太窄，又会错过相邻场景。
2. 技能数量多了以后，描述列表本身也在吃上下文预算。
3. 技能与 MCP 工具、内置命令职责重叠，Agent 不知道该信谁。

## 做法

以添加一个自定义技能为例：

1. **建目录**。在 `~/.openclaw/workspace/skills/` 下建文件夹，比如 `git-release-helper/`，工作区技能会随会话自动被发现。
2. **写 SKILL.md**。YAML frontmatter 里必须有 `name` 和 `description`。description 是路由的关键，建议写成「什么时候用我 + 能做什么」，例如：「当用户要求整理提交记录、生成发布说明或打 tag 时使用。调用本地 git 完成分支整理与变更汇总。」
3. **正文写操作手册**。正文只在被触发时进入上下文，可以放开写：步骤、边界条件、失败回退。几百行以内为宜，脚本放同目录，正文里引用相对路径。
4. **依赖门控**。技能如果依赖某个 CLI（如 `gh`），在 frontmatter 的 `metadata.requires.bins` 里声明；环境缺失时 OpenClaw 会把技能标记为不可用，而不是让 Agent 半路报错。
5. **装别人的技能**。`clawhub search / install` 从社区仓库安装，装完重启会话生效。第三方技能建议先通读正文再启用，重点检查有没有让它联网上传或写敏感文件的步骤。

与 MCP 的分工：MCP 提供工具接口（API 层），Skills 提供流程知识（什么时候、按什么顺序调）。我们的经验是——能用现有 MCP 工具拼出来的流程，写成 skill，而不是再造一个 server。

## 踩坑点

- **描述里没有触发词**。写「帮我处理 git 的事」这种描述，Agent 匹配不上用户原话。要写用户实际会说出的词：release notes、changelog、打标签。
- **改了 SKILL.md 不生效**。多数情况是旧会话仍持有缓存，新开会话或重启 gateway 即可。
- **正文写成小作文**。触发后整段进入上下文，过长反而拖慢响应。把细节拆到附属文件，正文只留决策主干。
- **命名冲突**。自定义技能与内置技能同名时，加载顺序不一定如你所愿，加个人前缀最省心。

## 可复用建议

- 一个技能只解决一类任务，宁可拆两个，不要做成大而全。
- 把 description 当作检索 key 来打磨，用真实请求反复测触发率。
- 团队协作时把 skills 目录纳入 git，技能变更像代码一样走 PR 评审。
- 定期清点：半年没触发过的技能直接删，描述列表的每一行都是成本。

## 总结

Skills 机制的价值不在「能装插件」，而在把上下文预算从静态堆料改成动态调度。写好一句 description，比堆十个技能更有效。建议从一两个真实高频任务起步，先跑通「触发—加载—执行」闭环，再考虑规模化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/5ac5a2ed69fbc3cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/1f962779fa648a47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/360a5520b6c730ab.png)

