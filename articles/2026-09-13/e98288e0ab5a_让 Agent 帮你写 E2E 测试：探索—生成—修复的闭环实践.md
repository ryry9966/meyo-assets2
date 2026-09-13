---
title: 让 Agent 帮你写 E2E 测试：探索—生成—修复的闭环实践
feedId: 37401
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

E2E 测试的处境一直尴尬：价值高（覆盖真实用户路径），成本也高（写得慢、脆、维护累）。很多团队的现状是核心链路只有一两条冒烟用例，其余靠人肉回归。

Agent 介入后有了新解法——但实践下来必须先泼一盆冷水：「让 AI 一键生成整套 E2E」基本是伪命题。真正可行的是把 Agent 当成一个会跑会改的初级测试工程师：你给它环境和约束，它负责探索、生成、修复，你守住断言质量。

## 问题：裸生成的三个坑

直接把需求文档丢给 Agent 让它写 Playwright 用例，通常会遇到：

1. **幻觉 selector**：Agent 凭「常见写法」猜 `.btn-primary`，实际 DOM 里根本不存在；
2. **断言过弱**：只验证「流程没报错」，页面渲染成什么样不管，等于没测；
3. **脆弱等待**：`sleep(3000)` 满天飞，CI 上时快时慢，flaky 率失控。

根因是同一个：Agent 没有真实页面上下文，只能靠想象补全。

## 做法：基于真实快照的闭环

我们的流程分五步，核心工具是 Playwright + 浏览器类 MCP 工具（向 Agent 暴露导航、截图、读 accessibility tree、执行测试等能力）：

**1. 先探索，再生成。** 让 Agent 先打开目标页面，读取 accessibility snapshot（而不是原始 HTML——体积小、语义清晰），拿到真实的 role / label / testid 结构。

**2. 带约束生成。** 指令里明确：selector 只允许 `getByRole` / `getByTestId`，禁止 `sleep`，等待一律用 `expect(...).toBeVisible()` 这类自动重试断言。这些约束建议固化到 AGENTS.md 或系统提示，不要每次口述。

**3. 跑失败，喂回 trace。** 用例挂了不要手动贴报错，让 Agent 直接读 Playwright 的 trace 文件和失败截图，定位是 selector 失效、时序问题还是真实 bug。前两类让它自己修，第三类停下来说明。

**4. 人只审断言。** 修复迭代通常 2-3 轮收敛。人审的重点只有一个：断言是否符合业务规则（边界值、错误提示文案、权限跳转）。这部分 Agent 不可靠，也不该可靠——它不知道你的产品逻辑。

**5. 接入 CI。** 合入前跑受影响的 spec；失败时自动把 trace 拉给 Agent 开修复 PR，人工 review 后合入。

## 踩坑点

- **快照也别整页喂**。长页面的 accessibility tree 依然很大，多轮对话容易撑爆上下文。让 Agent 先列区块大纲，再按区域深入。
- **禁止硬编码凭证和测试数据**。账号密码走环境变量，数据用工厂函数生成并在 teardown 清理，否则用例之间互相污染。
- **别让它「顺手」加 try-catch 吞错误**。Agent 为了让测试变绿会做出各种妥协，失败必须 fail loud。
- **覆盖范围要人来定**。Agent 天然倾向写 happy path，支付失败、并发冲突、弱网这些场景要显式列进任务清单。

## 可复用建议

- 把测试规范（selector 约定、命名、禁止项）写成一份固定文档放在 Agent 的工作目录，所有会话共享；
- 沉淀一个最小 MCP 工具集：跑指定 spec、读 trace、截失败图——够用就好，别贪多；
- 失败闭环比生成能力更重要：CI 失败自动回灌给 Agent，才是这套流程真正的杠杆。

## 总结

Agent 写 E2E 的价值不在「一键生成」，而在「探索—生成—修复」这个闭环：它压低了编写和维护的成本，让你敢把覆盖面铺开。断言正确性和覆盖决策仍然归人。守住这条线，这套协作是划算的；守不住，你只是更快地生产了一批 flaky 测试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/a572b9991d7c28d3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/db6f03ff2631c82d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/3452553edbb321e8.png)

