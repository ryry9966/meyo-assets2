---
title: LLM 输出解析的防御性编程：JSON 标签与混合格式的分层处理
feedId: 37414
source: 综合讨论
publishedAt: 2026-09-13
---

## 背景

在 OpenClaw 的 Agent 流程和插件自动化里，大量节点依赖 LLM 输出结构化 JSON：工具调用参数、子任务分发、状态回写……提示词里通常约定「用 ```json 代码块包裹」或「用 <result> 标签包裹」。模型大多数时候会遵守，但自动化要的是确定性，模型给的是概率。

## 问题

翻一翻线上日志，失败形态比想象中丰富：

- 有时带 ```json 围栏，有时不带，有时写成 ```JSON；
- 标签前多出一段解释文字，或者标签里又套了一层标签；
- 中文语境下出现全角引号""''、行尾逗号、`//` 注释；
- 按 `^```json\n(...)\n```$` 写的严格正则，一条格式漂移整条流水线报错，接着无脑重试三次，token 烧了，问题还在。

## 做法：分层兜底，而不是一个正则打天下

我们把解析器改成四级流水线，任一级命中即通过：

1. **标记提取**：对 `<result>`、`<json>`、`<output>` 等候选标签做大小写不敏感匹配，取最后一个闭合标签的内容（模型常在开头复述指令，末尾那段才是正经输出）。
2. **围栏剥离**：非贪婪匹配 ```` ```(\w+)?\n([\s\S]*?)``` ````，命中后取第二组。
3. **直接 parse**：什么标记都不像时，对全文直接 `json.loads`——小模型偶尔会无视所有约定裸输出。
4. **修复兜底**：用 `json_repair`（Python）或 `jsonrepair`（TS）处理尾逗号、单引号、全角引号、注释、未加引号的 key。别手写这些正则，边界情况比想象中多。

```python
from json_repair import repair_json

def parse_llm_json(raw: str):
    for candidate in extract_tags(raw) + extract_fences(raw) + [raw]:
        try:
            return json.loads(candidate)
        except Exception:
            continue
    return json.loads(repair_json(raw))
```

parse 成功后再过一层 pydantic / zod schema 校验。字段缺失或类型不对时走「带错误信息重试」：把解析报错原文拼回提示词让模型自己修，命中率远高于原样重发。

## 踩坑点

- **贪婪匹配**：`.*` 会把多个代码块之间的内容全吃进去，务必非贪婪，或按首尾配对截取；
- **标签嵌套**：模型可能在 `<result>` 里再输出一层 `<result>`，提取时按同名校验配对，或只信最外层；
- **repair 成功 ≠ 语义正确**：字符串可能被修成意外结构，schema 校验永远不能省；
- **temperature=0 也有漂移**：别指望参数解决格式问题；
- **全角标点**：中文提示词下出现率明显更高，选修复库时确认覆盖。

## 可复用建议

- 解析器做成独立 util，Agent、MCP 工具、插件共用一份，别各写各的；
- 日志保留 `raw / extracted / repaired` 三个字段和命中层级，排查时极其有用；
- 监控命中层级分布：repair 命中率持续上升，说明该改提示词了——解析兜底是止损，不是常驻方案；
- 把线上真实失败样本沉淀成回归用例集，改提示词或换模型后跑一遍。

## 总结

防御性解析不是提示词工程的替代品，而是给概率系统兜底的工程手段。分层提取 + 修复库 + schema 校验 + 带反馈重试，这套组合下来，我们的解析失败率从两位数降到万分位以下，且每次失败都有日志可查。核心一句话：永远不要假设模型的输出格式，但要让每一层偏离都有路可走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/d60401d628d9c7bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/16795e2b703eb1e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-13/47d29773353e7b9d.png)

