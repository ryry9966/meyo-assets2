---
title: LLM 输出解析的防御性编程：JSON 标签混杂场景的分层处理
feedId: 38348
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

在 OpenClaw 的 Agent、MCP 工具和插件自动化里，最常用的模型约定是"按固定结构返回 JSON"。开发时用固定模型测试，确实这么返回。但一上线——换模型、换版本、上下文变长——输出格式就开始漂移。

## 问题：一条指令，五种产出

同一份 prompt，实际见过的输出至少有这些形态：

1. 裸 JSON，最理想；
2. 包在 ` ```json ` 围栏里；
3. 包在 `<json>`、`<answer>` 等自定义标签里，大小写和属性不定；
4. 前后带一句"解析结果如下：""如需调整请告诉我"；
5. 半合法 JSON：尾逗号、单引号、中文全角引号、NaN。

只写一层 `json.loads` 的解析器，迟早在这几种里的某一种上挂掉。最阴险的是嵌套围栏：JSON 字段值本身含 markdown 代码块时，朴素的全局去围栏会把数据截断。

## 做法：分层解析，逐级降级

核心思路不是写一个万能正则，而是按代价从低到高排一条策略链，命中即返回，同时记录走了哪个分支：

1. **预处理**：去 BOM、零宽字符，trim；
2. **直接解析**：`json.loads`，大多数时候模型是听话的；
3. **围栏剥离**：定位第一个围栏起始行和最后一个结束行，取中间内容。不用全局正则替换，避免误伤字段值里的 ` ``` `；
4. **标签提取**：对常见标签做大小写不敏感、允许带属性的匹配；
5. **括号配平扫描**：从首个 `{` 或 `[` 起逐字符扫描，跳过字符串字面量内的括号和转义，取第一个配平的完整对象，兜住"前后有说明文字"的情况；
6. **宽松兜底**：用 json5 或 repair 逻辑处理尾逗号、单引号。注意 `json.loads` 默认放行 NaN/Infinity，最好用 `parse_constant` 显式拒绝；
7. **Schema 校验**：解析成功不等于结构正确。用 Pydantic 定义模型，失败转成结构化错误；
8. **重试闭环**：把校验错误信息拼回 prompt 再请求，限 1–2 次。

```python
def extract_json(text: str):
    s = text.strip().lstrip("\ufeff")
    for cand in (s, strip_outer_fence(s), *extract_tagged(s), balanced_scan(s)):
        if not cand:
            continue
        try:
            return json.loads(cand, parse_constant=_reject)
        except json.JSONDecodeError:
            continue
    return None  # 上层记日志，触发带错误反馈的重试
```

## 踩坑点

- 贪婪正则 `.*` 会跨字段吞内容；模型偶尔一次返回两段 JSON，应取第一个配平对象而不是最后一段。
- 中文全角引号""和全角冒号：肉眼几乎看不出，报错列号还指不准。
- `ast.literal_eval` 不是 JSON 解析器，`true/false/null` 会直接失败。
- 流式输出拿到的是不完整 JSON，别拿最终解析器去解半截流，要么等终止符，要么用增量解析。
- 静默 `except: return {}` 会让坏格式"看起来正常"，问题被推迟到下游才爆，排查成本翻倍。

## 可复用建议

- **收敛到单一模块**：解析逻辑只放一处，打点记录命中分支。分支命中率本身就是 prompt 质量的观测指标——如果长期靠兜底分支撑着，说明该改 prompt 了。
- **失败样本进测试**：每次线上解析失败，把原始输出脱敏后存成测试用例。策略链的回归测试就是这么一个个攒出来的。
- **防御不替代约束**：模型侧能开 JSON mode / 结构化输出就开。防御层是约束失效时的兜底，不该成为常态路径。
- **变更即回归**：换模型、改 prompt 后重跑这批用例，格式漂移大多能提前暴露，而不是等用户报障。

## 总结

LLM 的输出契约本质是"尽力而为"。防御性解析的目标不是修好所有坏输出，而是让合法输出走快路径、坏输出有确定的降级路径和明确的观测信号。把"格式对不对"从玄学变成可测试、可打点的工程问题，Agent 链路才谈得上稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/3ba9226027290b3f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/801b97a7a82ff3ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/ae474d8cf8584cc2.png)

