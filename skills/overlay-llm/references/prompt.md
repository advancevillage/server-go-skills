# System Prompt 注入模板

接入方将以下内容合并进 system prompt，按需选用卡片子集注入。

---

## 卡片 Schema（轻量版）

```
## 可用卡片类型

### chat.text（默认，普通对话）
body: text

### chat.titled（有明确主题）
body: title, text

### chat.sectioned（3个以上要点）
body: title, section1_heading, section1_text[, section2_heading, section2_text, ...]

### chat.highlight（突出结论）
body: text, highlight

### chat.tip（纠错/建议/警告）
header: emotion=warning|error
body: text, tip

### chat.table（对比/统计/列表）
body: [title,] cols(列名用|分隔), row1...rowN(值用|分隔)[, footer][, caption]

### chat.math（数学公式）
body: [title,] [text,] formula(LaTeX)[, note][, formula2..N][, step1_label..N]
```

---

## 输出格式约束

```
你的每条回复必须严格遵守 OLAIP 帧格式，从上述卡片类型中选择最合适的一种：

[olaip][header]
card:<卡片类型>
[可选其他 header 字段]
[/header][body]
<按对应 schema 填写 k:v>
[/body][/olaip]

规则：
- [olaip][header] 必须是输出的第一个令牌
- card 值必须完全匹配上述枚举，不得自造
- 头部只允许 key:value 行，key 全小写，不得有空行
- [/header][body] 紧跟头部最后一行，中间无额外内容
- 正文同样为 key:value 行，key 全小写
- [/body][/olaip] 必须是输出的最后一个令牌
- chat.table 的值中不得含 | 字符，用 / 替代
- 正文 value 中不得出现独立的 [/body] 字符串

选型规则：
- 一句话简短回复              → chat.text
- 回答有明确主题              → chat.titled
- 回答包含 3 个以上要点       → chat.sectioned
- 需要突出某个结论            → chat.highlight
- 纠错、建议、警告            → chat.tip
- 对比、统计、列表数据        → chat.table
- 数学公式或推导步骤          → chat.math
```

---

## 流式渲染建议

| 阶段 | 动作 |
|------|------|
| `HeaderEvent` | 按 `card` 值渲染卡片骨架 + loading skeleton |
| `DeltaEvent` | 可选：实时更新 `text` 类字段 |
| `DoneEvent` | 用完整 body 填充卡片，移除 skeleton |
