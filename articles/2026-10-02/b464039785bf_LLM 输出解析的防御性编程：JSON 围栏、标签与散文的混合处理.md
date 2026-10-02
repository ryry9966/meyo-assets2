---
title: LLM 输出解析的防御性编程：JSON 围栏、标签与散文的混合处理
feedId: 40105
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

在 OpenClaw 的 agent 流水线里，让模型输出 JSON 是家常便饭：MCP 工具的入参、插件之间的消息体、定时任务要落库的结构化结果。提示词里写明"只输出 JSON，不要任何解释"，大多数时候它确实照做——但"大多数时候"放在自动化场景里就等于定时炸弹。跑批几百次，总有几条输出混进了代码围栏、前缀说明，或者干脆是个被截断的半个对象，然后整条链路在凌晨挂掉。

## 问题

实际遇到的"脏输出"大致四类：

1. **围栏包裹**：` ```json ... ``` `，语言标注有时缺失，围栏有时没闭合；
2. **自带标签**：`<json>{...}</json>` 之类的 XML 风格包裹；
3. **前后缀散文**："以下是解析结果：" + JSON + "如有问题请告知"；
4. **JSON 本身有瑕疵**：尾逗号、中文引号、字符串内未转义换行、max_tokens 截断导致括号不配对。

一个裸的 `json.loads` 显然扛不住；但也不能一失败就降温重跑，成本和延迟都不划算。合理做法是承认脏输出是常态，写一个分层的容错解析器。

## 做法

核心思路：按侵入性从低到高逐层尝试，命中哪层就记录哪层，绝不静默修复。

```python
def robust_json_loads(text: str):
    raw = text.strip().lstrip("\ufeff")            # 去 BOM
    try:
        return json.loads(raw), "direct"           # L1 直接解析
    except json.JSONDecodeError:
        pass
    cands = []
    if m := re.search(r"```(?:json)?\s*([\s\S]*?)```", raw):
        cands.append(m.group(1))                   # L2 剥最外层围栏
    if m := re.search(r"<json>\s*([\s\S]*?)\s*</json>", raw):
        cands.append(m.group(1))                   # L2 剥标签
    cands.append(scan_balanced(raw))               # L3 括号配对扫描
    for c in cands:
        if obj := try_loads_with_repair(c):        # L4 尾逗号/引号/注释
            return obj, "repaired"
    raise JsonParseError(raw)                      # L5 上抛或触发一次自修复
```

四个要点：

- **L3 别用"找第一个 `{` 和最后一个 `}`"**。字符串里合法地含有花括号时会切错位置。写个二十行的状态机：维护 `in_string` 和 `escape` 两个标记，扫描时记录括号深度，深度归零处即终点。这段代码写一次，所有插件复用。
- **L4 修复要克制**。去尾逗号、BOM、零宽字符，替换中文引号，是安全的；单引号键改双引号要小心英文撇号（`it's`）；删注释必须跳过字符串内部。
- **L5 的"让模型自修"** 要把上一次的解析错误连同原文一起发回去，只重试一次，防止死循环。
- **解析成功 ≠ 数据正确**。过了 loads 必须再过 schema 校验（pydantic 或 JSON Schema），字段缺失、类型不符照样打回。

## 踩坑点

- **围栏正则的非贪婪陷阱**：模型有时在字段值里回显代码，内含 ` ``` `，非贪婪匹配会提前截断。最外层围栏优先处理，失败就回退 L3 扫描。
- **截断的 JSON 别修**：repair 库"脑补"闭合后能通过解析，但语义是错的。看到 finish_reason 是 length，直接重请求。
- **修复不留痕等于没修**：返回值里带上 repair_trace，日志能看到这条数据走了哪层，出问题才能归因。
- **提示词约束不可靠**：few-shot 示例和 JSON mode 能降低脏输出概率，但跨模型、跨网关时表现漂移，防御层不能省。

## 可复用建议

- 解析器做成独立工具函数，别散落在各插件的 try/except 里，OpenClaw 生态内共享同一份实现。
- 建"脏输出语料库"：每次线上解析失败，把原文收进去写成单测。半年后这是最值钱的资产之一。
- 按命中层统计占比并监控：direct 命中率从 95% 掉到 70%，多半是模型或网关变了，比等业务报错灵敏得多。

## 总结

把 LLM 输出当作用户输入对待：不可信、需校验、要留痕。提示词约束是第一道防线，但真正的稳定性来自"容忍提取 + schema 校验 + 命中率监控"这套组合。写一次分层解析器大概半天，换来的是自动化链路不再因为一条脏输出整批报废。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/0fbb0512d6a08686.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/ed21410eac2a7a48.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/aff8e5997ce89837.png)

