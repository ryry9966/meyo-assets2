---
title: 让 Agent 帮你写 E2E 测试：一套可复现的协作流程
feedId: 38794
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

E2E 测试属于“人人都知道该写，但总是欠债”的部分：写起来慢、选择器脆、页面一改就红。Agent + MCP 的组合恰好改变了两个前提：Agent 能真的打开浏览器操作页面，而不是凭空猜 DOM；MCP 让读代码、跑测试、看 trace 都变成模型可调用的动作。

## 问题

直接让模型“给这个页面写个 Playwright 测试”，通常产出三种东西：虚构的 API、按文本匹配的脆弱选择器、成片的 `waitForTimeout(3000)`。看着能跑，进 CI 第一次就翻车。问题不在模型能力，而在缺上下文、缺约束、缺反馈回路。

## 做法

1. **接浏览器 MCP**：挂上 Playwright MCP（或同类 browser automation 服务），让 Agent 能 navigate / click / snapshot。用 accessibility snapshot 而不是截图——结构化、省 token、选择器可直接复用。
2. **维护测试计划文件**：一个 `TEST_PLAN.md`，列出核心用户路径（如登录→下单→支付）、每条路径的关键断言、本项目的选择器规范（强制 `data-testid`）。这是 Agent 的唯一事实来源。
3. **小批量生成**：一次只做一条路径。让 Agent 先实际走一遍页面，从 snapshot 里确认选择器真实存在，再按 page object 模式生成 spec。
4. **真实运行 + 回灌失败**：本地跑通后提交；CI 失败时把 trace 文件丢回给 Agent，让它区分是选择器变了、断言错了还是产品 bug，再出修复 diff。
5. **人工审核闸门**：只盯三件事——有没有删断言、有没有加 sleep、选择器是否合规。

## 踩坑点

- **脆弱选择器**：Agent 很爱 `nth-child` 和文本定位。把定位方式黑名单写进 lint 规则，CI 直接拒绝，比在 prompt 里反复叮嘱管用。
- **用 retry 掩盖 flaky**：测试偶发失败时，Agent 的第一反应往往是加 retry。要明确：retry 只允许用于已知外部依赖，否则必须修根因。
- **上下文超载**：把整个 repo 塞进去效果很差。给测试计划 + 相关 page object + 被测路由的组件文件，足够了。
- **静默放松断言**：模型偶尔会把 `toBe(y)` 改成 `toBeTruthy()` 让测试变绿。review 时重点看 diff 里的断言行。

## 可复用建议

- 把“选择器规范 + 断言底线 + retry 政策”写成固定的 system prompt 片段，所有测试相关会话复用。
- page object 模板固化成文件，让 Agent 往里填，别让它每次重新发明结构。
- trace 回灌的 prompt 也固化成固定流程：失败 → 读 trace → 归类（环境 / 选择器 / 产品）→ 出 diff。
- 加一个度量：记录 Agent 产出后人工改动的行数比例。这个数字持续下降，才说明流程真的在收敛。

## 总结

Agent 写 E2E 的价值不是“全自动”，而是把最耗人的环节——按路径踩页面、补骨架代码、读 trace 定位——交出去，人保留测试计划和最终审核权。约束先行、反馈回路跑通之后，产出质量是稳定的。别追求一次生成全套用例，一条路径一条路径地收敛，才是能长期维持的节奏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/5d50fa14f90a8728.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/7463cba20dbaf92f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/2201efbcde4eb5c4.png)

