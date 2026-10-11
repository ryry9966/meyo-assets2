---
title: OpenClaw Skills 机制：让 Agent 按需加载能力，而不是把工具箱全塞进上下文
feedId: 41234
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

给 Agent 扩能力，常见两条路：一是把操作流程全写进 system prompt，二是接一堆 MCP server 把工具全挂上。两条路的共同问题是上下文预算：挂上十来个 server 之后，仅 tool schema 和常驻说明就能吃掉数千 token，而其中绝大部分和当前这轮任务无关。能力越堆越多，基础上下文越来越脏，模型注意力也被稀释。

OpenClaw 的 Skills 机制走的是第三条路：能力以目录形式放在工作区里，启动时只注入「名字 + 一句话描述」，正文按需读取。

## 机制：三层渐进加载

1. **启动层**：扫描 skills 目录，把每个 SKILL.md frontmatter 里的 name、description 注入系统提示，单个成本几十 token。
2. **匹配层**：用户请求与某条 description 语义匹配时，Agent 主动读取该 skill 全文，操作指令进入上下文。
3. **执行层**：正文引用的脚本、模板、参考文件，只在真正执行到那一步才读取或运行。

效果是基础上下文近似恒定，能力数量与 token 成本解耦。装 50 个 skill 和装 5 个，启动开销差不了太多——前提是 description 写得好。

## 做法：写一个最小可用的 Skill

1. 在 skills 目录下新建文件夹，里面放一个 SKILL.md。
2. frontmatter 写 name 和 description。description 是唯一常驻上下文的字段，直接决定触发与否，要按「接口」来写。
3. 正文写操作步骤、命令、边界条件，建议压在 100~200 行内。
4. 复杂逻辑落成独立脚本，正文只写「执行哪个脚本、参数含义、失败时怎么处理」。
5. 重启会话后，用一组固定 prompt 验证触发路径，再投入使用。

Skill 很适合承接两类东西：团队 SOP（发布流程、值班排查手册），以及 MCP 不方便封装的本地 CLI 操作。

## 踩坑点

- **description 写成功能清单而不是触发条件**。「负责服务器管理」是无效的；「当用户要求重启服务、拉日志、发版时使用，纯咨询问题不使用」才可靠。前者触发靠运气。
- **正文过长**。每次触发全文进上下文，500 行的 skill 每触发一次就是一次 token 税。长参考资料拆成独立文件，正文里标注「需要时再读」。
- **一个 skill 塞多个不相关职责**，匹配信号被稀释，要么误触发要么不触发。
- **脚本用相对路径**，工作目录一变就断；在 SKILL.md 里显式写清执行目录，或直接用绝对路径。
- **最隐蔽的一条：skill 根本没触发**。Agent 靠通用知识「演」完了任务，结果看着对、细节错。务必查日志确认 SKILL.md 真的进了上下文。

## 可复用建议

- description 固定用「何时使用 / 何时不用」两段式。
- 一 skill 一职责；出现第二个不相关的触发场景就考虑拆分。
- 维护一个 3~5 条正例 + 3~5 条反例的测试集，改完跑一遍当回归。
- skills 目录进 git，和代码同版本管理；改 description 的改动单独审。

## 总结

Skills 机制的价值不在「能装多少能力」，而在「上下文里始终保持干净」。把 description 当接口设计，把正文当文档写，把脚本当工具实现，按需加载才真正成立。对一个需要持续扩能力的 Agent 项目来说，这比单纯堆更多工具更划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/9d0a8c497f273088.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/24cc1dc20eae2081.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/392273fcb11c22b9.png)

