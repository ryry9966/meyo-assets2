---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的分层处理
feedId: 37696
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

在 OpenClaw 的技能和 MCP 工具链里，让 LLM 输出结构化 JSON 是最常见的需求：Agent 之间传参、插件拿到可执行配置、自动化流水线消费结果。Prompt 里写得很清楚"只输出 JSON，不要代码块"，但跑久了你会发现，模型的输出分布有一条很长的尾巴——不是模型不行，是格式漂移永远会发生。

## 问题：约定敌不过采样

实际踩到的混合格式包括：

- 整体被 ```json 围栏包住（训练语料惯性）
- 用 `<json>...</json>` 或 `<JSON>` 标签自包裹
- JSON 前面带一句"以下是解析结果："之类的中文前缀
- 尾逗号、单引号、字符串里混着注释、`NaN`/`None` 裸值
- 一个回复里出现两段 JSON（一段解释、一段结果）

`json.loads()` 直接抛异常，整条链路中断。如果你在多个插件里各自散落着 `json.loads` 和一两行正则，每个地方的处理逻辑都不一样，排障成本极高。

## 做法：分层解析，收敛到一个模块

核心思路是**逐层降级**，每一层只处理一类漂移：

```python
import re, json
from json_repair import repair_json

def extract_json(raw: str):
    # L1 前置清洗：剥围栏和 <json> 标签
    text = re.sub(r"```[a-zA-Z]*|</?json>", "", raw, flags=re.I).strip()
    try:
        return json.loads(text)              # L2: 清洗后整体就是 JSON
    except json.JSONDecodeError:
        pass
    for opener, closer in (("{", "}"), ("[", "]")):
        start = text.find(opener)
        if start < 0:
            continue
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):    # L3: 字符串感知的括号配对截取
            c = text[i]
            if in_str:
                if esc: esc = False
                elif c == "\\": esc = True
                elif c == '"': in_str = False
            elif c == '"':
                in_str = True
            elif c == opener:
                depth += 1
            elif c == closer:
                depth -= 1
                if depth == 0:
                    frag = text[start:i + 1]
                    return json.loads(repair_json(frag))  # L4: 修复尾逗号/引号
    raise ValueError("no json object found")
```

解析成功后还有 **L5：schema 校验**（pydantic 或 jsonschema），字段缺失、类型不对在这里拦截。

依赖：`pip install json-repair`。

## 踩坑点

1. **贪婪正则截取**：`\{.*\}` 会横跨前缀说明甚至吞掉第二段 JSON，必须用括号配对而不是正则贪婪匹配。
2. **括号计数不看字符串状态**：JSON 字符串值里的 `{` 和 `"` 会让截断点错位，配对循环里必须维护 `in_str` 状态。
3. **修复库会"善意"改数据**：`repair_json` 可能把你没意图的 `'null'` 修成字符串 `"null"` 而不是 `null`，自动化链路必须靠 schema 校验兜底，不能裸信修复结果。
4. **重试不带反馈**：解析失败就原样重发，模型看不到上一轮错在哪，命中率很低。正确做法是把解析错误信息拼回 prompt 重试 1–2 次。
5. **只存解析结果不存 raw output**：事后想统计"哪个模型漂移率高"时无据可查。

## 可复用建议

- **收敛**：全项目只用这一个解析模块，禁止业务代码里散落 `json.loads`。
- **双保险**：prompt 侧给 few-shot 边界示例、明确"不要围栏"，同时解析器照旧支持围栏——约定和防御互不替代。
- **主路径不靠修复**：优先用模型的结构化输出/工具调用能力，这套解析器定位是降级路径。
- **可观测**：日志记录 raw output + 实际走了哪一层，漂移率本身就是评估模型更换影响的指标。

## 总结

防御性解析不是不信任模型，而是承认输出分布有尾巴。把不确定性收敛进一个分层模块，prompt 侧降低漂移概率，解析侧兜住漂移后果，schema 校验守住数据契约——三层下来，链路的稳定性会比"再多改一版 prompt"可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/f7fcbdac5c10d9f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/dc91bdef7e42892c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/ff0a763630f65766.png)

