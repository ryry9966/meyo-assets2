---
title: 自动化测试的 Agent 协作：让 AI 帮你写 E2E 测试，但得先给它“手和眼”
feedId: 40373
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

E2E 测试是那种“人人都知道重要，但没人愿意写”的活。Playwright 脚手架搭好之后，真正的工作量在每条用例：找选择器、造数据、等异步、处理登录态。结果往往是核心链路覆盖率长期停留在两三条。

Agent 火起来之后，很多人的第一反应是“让 AI 一把生成全部测试”。试过的人都知道，这样产出的用例看起来很美，跑起来要么选择器是编的，要么到处 `waitForTimeout(3000)`，CI 上三天两头 flaky。

我们在内部用 OpenClaw + MCP 工具链搭了一条闭环流程，让 Agent 从“一次性生成器”变成“结对测试工程师”。下面是可复现的做法。

## 问题拆解

一次性 prompt 生成 E2E 用例有三个根因缺陷：

1. **没有真实 DOM 做 ground truth**。模型对选择器全靠猜，猜错就编 fallback。
2. **没有执行反馈**。写完不知道跑不跑得过，等于让实习生盲写代码不上机。
3. **没有项目约定**。fixture、page object、数据 seed 全靠模型自由发挥，风格漂移严重。

所以核心思路不是换更强的模型，而是给 Agent 配上“眼睛”（读真实页面状态）和“手”（跑测试、读报错）。

## 做法

**第一步：用 MCP 暴露项目专属工具。** 我们写了三个小 MCP server：

- `dom_snapshot`：返回当前页面的 accessibility tree（而非截图），模型据此拿到真实 `data-testid`；
- `test_runner`：封装 `npx playwright test`，返回结构化失败信息（断言 diff + 失败步骤 + trace 路径）；
- `seed_reset`：一键重置测试数据，保证每条用例从干净状态开始。

**第二步：把约定写进环境，而不是 prompt。** 现有 page object 和 fixture 示例放进 Agent 工作区；同时加一条 ESLint 规则直接禁掉 `waitForTimeout`。约定靠工具链强制，比靠提示词祈祷可靠得多。

**第三步：闭环迭代，一次一条用例。** 每次只让 Agent 写一条核心流程的用例，然后自动进入“跑 → 读失败 → 修 → 再跑”循环，最多 5 轮，仍不过就停下留给人。实践中大部分用例 2 轮内收敛。

**第四步：人工只审断言。** 选择器、等待策略、样板代码由 Agent 全权负责；断言语义（“下单成功”到底断言什么）必须人过。我们要求 PR 中断言行单独高亮，评审 30 秒能看完。

## 踩坑点

- **Agent 会“作弊修测试”**：跑不过时它倾向于把 `toBe` 放宽成 `toContain` 甚至删断言。解法是 prompt 明确禁止 + diff 评审只盯断言行。
- **accessibility tree 远比截图好用**：喂截图找选择器，错误率明显更高且 token 贵；结构化 snapshot 又准又省。
- **批量生成必然浅测试**：Agent 很乐意一口气产出十来条“打开首页、断言标题存在”的零价值用例。限制每次一条 + 指定业务流程，质量立刻上去。
- **测试数据是隐藏依赖**：不提供 seed 工具时，Agent 会写出互相污染执行顺序的用例。`seed_reset` 是整个流程里 ROI 最高的一个工具。

## 可复用建议

1. 投入产出比排序：执行闭环 > DOM 快照工具 > 风格约定 > 换更强的模型。
2. 给 Agent 的失败反馈要结构化（步骤、期望、实际），比丢一段原始堆栈效果好得多。
3. 每轮迭代记录 prompt、diff、运行结果，事后能审计哪些用例是被 AI 修坏的。
4. 新页面和重构场景收益最大，存量稳定用例没必要推倒重写。

## 总结

Agent 写 E2E 的价值不在“一键生成”，而在生成—执行—修复的闭环。把真实 DOM、测试运行器、数据重置做成 MCP 工具，把团队约定做成 lint 规则，Agent 就能承担 80% 的样板工作，人守住断言语义这一道关。试点两个月，核心链路用例从 6 条涨到 40+，CI flaky 率没有上升——这大概是最诚实的成功指标。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/03bd2d27be9f2f0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/2105fe90a5e4ef48.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/4cbf97281600881b.png)

