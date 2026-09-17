---
title: 给 LLM 输出解析加一道防线：JSON 标签格式混合处理实录
feedId: 37916
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

在 OpenClaw 上跑 Agent 和 MCP 工具链，绕不开一件事：让模型返回结构化数据。提示词里明明写了"只输出 JSON"，但真实生产环境中返回格式五花八门。我的一个自动化插件每天处理几百次工具调用，最初解析失败率在 5% 左右，几乎全是格式问题，而不是逻辑问题。

## 问题

实际收到的输出大致有这几类：

1. 标准 JSON：直接 `json.loads` 就行，占比最高但不是全部；
2. 代码围栏包裹：` ```json ... ``` `；
3. 自定义标签包裹：`<json>...</json>`、`<result>...</result>`，模型经常自作主张；
4. 带前缀废话："好的，以下是解析结果："后面才跟 JSON；
5. 推理标签泄漏：`<think>...</think>` 之后才是正文；
6. 格式瑕疵：尾逗号、单引号、全角引号、被 max_tokens 截断。

任何一类没兜住，插件就报错。防御性编程的核心只有一句话：**不信任输出格式，按出现概率分层兜底。**

## 做法：四层解析管道

**第一层，剥包裹。** 先去掉代码围栏，再用正则抽自定义标签内容。标签抽取要覆盖常见变体，并且必须用非贪婪匹配。

**第二层，定位 JSON 边界。** 前两层失败时，找第一个 `{` 或 `[`，再用括号计数从前往后找配对的闭合符号，不能简单 `split`，否则字符串值里的花括号会干扰定位。

**第三层，容错修复。** 直接解析失败后，用支持尾逗号和注释的解析器（json5、dirty-json）重试一次，再做全角转半角等轻量清洗。

**第四层，校验与重试。** 用 pydantic / zod 校验字段和类型，失败就把解析错误拼回提示词让模型自查重试，最多两次。

核心骨架（Python）：

```python
def extract_json(text: str) -> dict | None:
    # 1. 剥代码围栏
    m = re.search(r"```(?:json)?\s*([\s\S]*?)```", text)
    candidates = [m.group(1)] if m else []
    # 2. 剥自定义标签（非贪婪）
    candidates += re.findall(
        r"<(?:json|result|output)>([\s\S]*?)</(?:json|result|output)>", text)
    candidates.append(text)
    for c in candidates:
        for cleaned in pre_clean(c):        # 3. 清洗变体
            try:
                return json.loads(cleaned)
            except json.JSONDecodeError:
                continue
    # 4. 括号计数定位 + 容错解析器兜底
    return repair_parse(find_json_block(text))
```

## 踩坑点

- **贪婪正则**：`<json>(.*)</json>` 会跨块匹配，把两段独立 JSON 拼在一起，必须非贪婪。
- **括号计数**要跳过字符串字面量内的括号，正文带示例代码时必然错位。
- **推理标签泄漏**：解析前先剔除 `<think>` 段，否则定位阶段就崩。
- **截断不是解析问题**：max_tokens 不够导致 JSON 断尾，解析层怎么兜都兜不住，要监控 finish_reason 并提前拦截。
- **清洗别丢数据**：全局把单引号替换成双引号，会破坏字符串值里本来就有的单引号内容，只对解析失败后的候选做，且记录 diff。

## 可复用建议

- 永远保留原始输出和解析日志，失败样本就是最好的测试用例，攒起来做回归集。
- 优先用 API 原生的结构化输出 / JSON mode / tool call，防御层是兜底，不是首选方案。
- 把 `extract_json` 做成独立共享工具函数，所有插件用同一份，别各自造轮子。
- 解析失败不要静默吞掉，至少打 WARN 日志，带上模型名和原文片段。

## 总结

LLM 输出解析没有银弹，但"剥围栏 → 抽标签 → 定边界 → 容错修复 → 校验重试"这条管道覆盖了我遇到的 99% 场景。防御性编程的意义不是把代码写复杂，而是让失败可预期、可观测、可恢复。欢迎在社区贴你们遇到的奇葩输出样本，一起把这套用例集攒厚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/88ae12da0a651862.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/cf50a3814fdbb3cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/3fd79329c0bdaa8d.png)

