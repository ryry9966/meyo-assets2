---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 42267
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

跑 OpenClaw 一段时间后，最常见的症状不是"不会做"，而是"上下文太挤"。系统提示词里堆满各种工作流说明、格式约定、工具用法，几千 token 常驻，每个会话都背着跑。结果是模型在无关任务里被这些指令干扰，注意力被稀释，成本也随之上浮。

Skills 机制就是针对这个问题的：把"能力"从常驻上下文里拆出去，按需加载。

## 问题：常驻指令的三个代价

1. **成本线性增长**——每条指令每个会话都要付费，不管今天用不用得上；
2. **指令越多，路由越差**——模型面对一堆常驻规则，选错工作流的概率上升；
3. **维护困难**——改一条规则，要在巨型提示词里翻找，改完还容易引入副作用。

## Skills 如何工作：渐进式披露

核心是三层加载：

- **第一层（常驻）**：启动时只有每个 skill 的 name + description 进入上下文，单个通常几十 token；
- **第二层（按需）**：模型判断当前任务匹配某个 skill 后，才读取其 SKILL.md 正文；
- **第三层（引用）**：正文可再引用 references/ 下的长文档和 scripts/ 下的脚本，真正用到时才读入。

写一个最小 skill 只要三步：

1. 在 workspace 的 skills 目录下建文件夹，例如 `weekly-report/`；
2. 写 SKILL.md：frontmatter 给 name 和 description，正文写触发条件、步骤、约束；
3. description 决定"何时触发"，正文负责"怎么做"。

比如"周报生成"skill，description 写"当用户要求汇总本周工作、生成周报时使用"，正文放固定模板和输出格式。平时它只占一行描述的空间，周五被触发时才展开。需要确定性逻辑的步骤放进 scripts/ 让 agent 调脚本执行；支持声明本机依赖（命令、环境变量），环境不满足时自动禁用，避免运行期失败。

注意边界：skill 是**指令层**，只提供"怎么做的知识"；真正的执行走 tools/MCP。两者是配合关系，不要混为一谈。

## 踩坑点

- **description 含糊 = 不触发**。"处理文档相关任务"这种写法基本等于没写，模型判断不了边界；
- **description 太泛 = 误触发**。多个 skill 抢同一场景，模型随机选一个，行为不稳定；
- **正文塞满长文档**。SKILL.md 一旦几千字，"按需"名存实亡，长内容应下沉到 references；
- **职责重叠**。两个 skill 内容八成重复，要么合并，要么抽公共 reference；
- **改了没生效**。部分部署方式下 skill 列表有加载时机，改动后确认会话是否重新加载；
- **把 skill 当工具用**。它不提供执行接口，执行能力该交给 MCP/tools。

## 可复用建议

- 一个 skill 只管一个任务域，命名即边界；
- description 用"当……时使用"句式，写清触发场景和排除条件——它就是这个 skill 的 API 签名；
- 正文控制在几百字内，长内容全部下沉到 references；
- 确定性步骤脚本化，skill 里只写调用方式和参数约定；
- skills 目录进 git 管理，description 变更走 review，触发行为变化要能被追溯。

## 总结

Skills 的价值不在"能装多少能力"，而在"同一时刻上下文里有多少能力"。把知识从常驻改为按需，换来的是更低的 token 成本、更少的指令干扰和更稳定的任务路由。建议先挑一个高频工作流拆成 skill 试水，观察触发率是否符合预期，再逐步迁移存量提示词——渐进式披露的收益，只有在你忍住不把所有东西都塞进正文时才会显现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/4237d2ad90106fba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/6c3847b68639d156.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/3b8eb05f4e1ed8d7.png)

