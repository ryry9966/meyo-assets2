---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 41661
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

Agent 的价值在于"能动手"。OpenClaw 作为常驻本机的 agent runtime，能力上限基本取决于它能调到哪些外部服务：查库存、提工单、发通知、拉数据。工具调用本身不难，难的是把"demo 里能跑"做成"线上敢开"。

## 问题

多数对接翻车不在协议层，而在工程细节：

- 鉴权信息放哪，才能不进 prompt、不进日志；
- 下游接口慢或挂了，agent 会不会跟着卡死；
- 写操作一重试就重复提交；
- 错误信息模型看不懂，开始自由发挥。

## 做法

以对接一个典型 REST 服务（比如内部工单 API）为例：

1. **划边界。** 先列接口清单，标记读/写。第一版只暴露只读接口，写操作等链路稳定后再加。
2. **封装成 tool / MCP server。** 不要让模型直接拼 HTTP，写一个薄封装：入参出参用 JSON Schema 描述，字段语义写清楚（单位、格式、示例值）。支持 MCP 就挂 MCP，多个 agent 可复用。
3. **鉴权收口。** API key 放 OpenClaw 的插件配置或环境变量，由封装层读取；prompt、日志、返回值里都不出现。
4. **设防。** 每次调用显式设 timeout（建议 10s 内）；对 5xx/超时做 2 次指数退避重试；遇到 429 尊重 Retry-After。
5. **设计"给模型看的错误"。** 错误返回做成一句话 + 建议动作，例如"参数 date 格式应为 YYYY-MM-DD，请修正后重试"，别抛原始 stack trace。
6. **控制返回体积。** 大结果在封装层先裁剪/分页，只给模型需要的字段，防止上下文被撑爆或截断引发幻觉。
7. **写操作加约束。** 要求幂等键，或在 tool 内置二次确认：先返回待确认摘要，agent 向用户确认后再真正执行。
8. **留痕。** tool 调用统一记日志：入参、耗时、状态码，敏感头不落盘。

一段最小 tool 定义示例：

```json
{
  "name": "create_ticket",
  "description": "创建工单。title 必填，priority 默认 normal。",
  "parameters": { "title": "string", "priority": "low|normal|high" }
}
```

## 踩坑点

- **整份 API 文档塞进 system prompt**：token 先爆，模型还会选错接口。tool 描述写清"什么时候用、参数什么意思"就够了。
- **没设超时**：下游一抖，整条 agent 会话挂住。
- **重试不分场景**：GET 可以重试；POST 不带幂等键的重试就是重复下单。
- **错误直接透传**：模型盯着返回 JSON 某个字段瞎解释，不如一句人话管用。
- **返回整个列表**：几百条记录进上下文，既贵又容易让模型漏看关键项。

## 可复用建议

- 把 tool 描述当产品文案写：用途、参数语义、边界条件、一个示例。
- 错误信息三要素：哪里错了、期望是什么、下一步做什么。
- 新接口按"只读 → 可写带确认 → 全自动"三档灰度放量。
- 固化一份冒烟用例集：十个典型 prompt，每次改动跑一遍，重点看调用路径和错误路径。
- 能用 MCP 就用 MCP：鉴权和重试在 server 层做一次，所有 agent 受益。

## 总结

对接外部服务的本质，是给 agent 装一只"有分寸的手"。协议层用 tool/MCP 解决，剩下的功夫都在封装层——鉴权收口、超时重试、错误可读、写操作克制。把这套东西沉淀成模板，每接一个新服务就是填空，而不是重踩一遍坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/2d5ad3f7b4a6cfc3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/d648c8c4d1346330.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/0554ff17a15b5359.png)

