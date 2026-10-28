---
title: LLM 输出解析的防御性编程：JSON 标签混杂格式的分层处理实践
feedId: 42820
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

在 OpenClaw 的 Agent / MCP 插件开发里，让模型按约定输出 JSON 是家常便饭：工具路由要模型选 tool + args，子任务结果要结构化回传，自动化流水线靠这些字段做下游分发。但只要跑得够多，你会发现"要求输出 JSON"和"拿到可解析的 JSON"之间隔着一条很宽的沟。

## 问题：输出格式远比想象中杂

同一份 prompt 在不同模型、不同上下文长度下，实测至少会遇到这些变体：

- markdown 围栏：```json { ... } ```
- 伪 XML 标签：`<json>{...}</json>`，甚至 `<JSON>`、`<output>`
- 裸 JSON，前后带一句"好的，以下是结果："
- 输出多个 JSON 块（先解释一遍，再给一遍）
- 全角引号、全角冒号（中文 prompt 场景高频）
- 被 max_tokens 截断的半截 JSON
- 字符串值里本身含 ``` 围栏或引号

正则一刀切在这些场景都会翻车。在自动化链路里，解析失败不只是报错，而是整条流程卡住或走错分支。

## 做法：分层解析，逐级降级

核心思路：把解析当输入清洗，而不是格式断言。按成本从低到高排一条流水线，任一层成功即返回，全部失败才触发重试：

```
1. 直接 json.loads（理想路径）
2. 剥围栏/标签：提取 ```json、<json> 块再 loads
3. 括号配平扫描：全文找第一个语法配平的 {...} / [...]
4. 修复层：去尾逗号、全角转半角、json_repair 兜底
5. schema 校验：pydantic 验字段，不只是能 parse
6. 失败则带错误信息回喂模型，重试预算 1 次
```

第 3 层是关键。不要用 `\{.*\}`（贪婪）或 `\{.*?\}`（非贪婪）去抠 JSON——遇到嵌套对象两种都会切错。正确姿势是一个手写扫描器：

```python
def scan_balanced(text):
    depth, start, in_str, esc = 0, None, False, False
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == '\\': esc = True
            elif ch == '"': in_str = False
            continue
        if ch == '"': in_str = True
        elif ch in '{[':
            if depth == 0: start = i
            depth += 1
        elif ch in '}]':
            depth -= 1
            if depth == 0 and start is not None:
                return text[start:i+1]
    return None
```

它正确处理字符串内的括号和转义，对"说明文字 + 裸 JSON"和"字符串里嵌围栏"都稳。第 4 层的修复要克制：只做确定性规范化（尾逗号、全角标点、BOM），语义级修复交给 json_repair 这类库，并记录修复日志。

## 踩坑点

1. **截断修复是静默炸弹。** 给半截 JSON 补右括号能得到"合法但错误"的数据，下游照单全收。扫描结束时 depth != 0 应走重试或缩短输出，而不是硬补。
2. **多 JSON 块别默认取第一个。** 常见"先解释后输出"两份，策略应是全部提取、逐一过 schema 校验、取通过的那份。
3. **重试别开循环。** 带 parse error 回喂模型通常一轮就修好，预算设 1，超了就落盘原始输出人工看，否则成本和延迟双双失控。
4. **全角标点是中文场景的特色坑。** prompt 为中文时，模型输出全角引号/冒号的概率明显上升，规范化层要常备。

## 可复用建议

- 收敛成一个 util 模块：`extract_json(raw) -> ParseResult(data, strategy, repaired, raw)`，全项目只走这一个入口。
- 把线上真实脏输出攒成测试语料，解析器改动跑一遍回归，比拍脑侻可靠得多。
- 模型支持结构化输出 / tool schema 时优先用，解析降级链是给长尾场景兜底的，不是首选。
- 埋个统计：各层命中率分布是最诚实的 prompt 质量指标，第 1 层命中率下滑，多半是 prompt 或模型变了。

## 总结

LLM 输出本质是不可信输入。防御性解析不是和模型斗智斗勇，而是承认"格式保证不存在"，用分层降级 + schema 校验 + 有限重试 + 观测把不确定性框住。这套方案在一个十几个 MCP 工具的生产流水线上跑了数月，解析失败率从两位数降到基本只剩截断场景，核心代码不到百行。欢迎在社区帖子里贴你们遇到的更脏的真实输出样本，一起把语料攒厚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/b741c2ad906e1c8f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/77f1c0922f972745.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/36f6f569ddf9af6c.png)

