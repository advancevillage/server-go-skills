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

## 输出格式约束（System Prompt 段）

```
【硬性输出协议 — 违反即视为故障】
1. 每条回复 = 单个 OLAIP 帧。
2. 首字符必须是 [，末字符必须是 ]。帧外不得有任何字符（含空格、换行、emoji、寒暄、解释）。
3. 即使只是"好"/"嗯"/"你好" 也必须用 chat.text 包裹。
4. 每个 key:value 独占一行，用真实换行符（不得用空格代替）。

错误（裸文本，绝对禁止）：
配对相加真聪明！

正确：
[olaip][header]
card:chat.text
[/header][body]
text:配对相加真聪明！
[/body][/olaip]

补充规则：
- card 值必须完全匹配上述枚举，不得自造
- 头部 key 全小写，不得有空行
- [/header][body] 紧跟头部最后一行，中间无额外内容
- chat.table 的值中不得含 | 字符，用 / 替代
- 正文 value 中不得出现独立的 [/body] 字符串

选型规则：
- 一句话简短回复              → chat.text（默认）
- 回答有明确主题              → chat.titled
- ≥2 个并列要点或分步骤       → chat.sectioned
- 需要突出某个结论            → chat.highlight
- 纠错、建议、警告            → chat.tip
- 对比、统计、列表数据        → chat.table
- 数学公式或推导步骤          → chat.math
```

System Prompt 末尾必须追加 recap（吃 recency bias，对抗中段注意力衰减）：

```
【回复前自检】
回复必以 [olaip][header] 开头、以 [/olaip] 结尾。默认 chat.text；其他卡片仅当明确匹配 trigger 时使用。
```

---

## 用户消息尾部锚定（Sandwich Prompting）

仅靠 system prompt 不可靠。长 system + 长 history 会让中段注意力衰减；历史里的旧裸文本会作为隐式 few-shot 反向带歪模型。**必须在每次发送的最后一条 user content 末尾追加 anchor**，把约束贴到生成位最近的位置（tail anchoring / 末位敏感性）。

**Anchor 模板：**

```
【输出锁】回复立即从 [olaip][header] 开始，不要任何前言、寒暄或裸文本；不知道选哪张卡时默认 chat.text。
```

**接入要点：**
- 只 wrap 最末位的 user message；history 中的 user turn 原样保留
- anchor 仅出现在 outgoing HTTP body，**不写入本地 storage**（避免污染 UI 展示与下一轮 history）
- 与 system prompt 内的硬性协议 + 末尾自检三者协同（system 开头 + system 末尾 + user 末尾，构成 sandwich）

---

## 约束强度梯度

按强度从弱到强排列。**长会话 / 弱模型 / 复杂格式至少 L5**。

| 级别 | 措施 | 失败模式 | 何时升级 |
|---|---|---|---|
| L1 | 一句"必须 OLAIP" | 模型多半忽略 | 立即 |
| L2 | + 完整 schema + example | 短回复时仍可能裸文本 | 立即 |
| L3 | + 硬性协议（编号 + WRONG/RIGHT 反例） | 中段注意力衰减仍有概率漏 | 长 system 时 |
| L4 | + System 末尾自检 recap | 历史污染仍能带歪 | history ≥ 4 轮 |
| L5 | + 用户消息尾部锚定（sandwich） | 历史里的裸文本仍可能作为 few-shot | 默认基线 |
| L6 | + 历史消毒（outgoing 前 wrap 旧 assistant 裸文本） | — | 跨会话长期使用 |
| L7 | + 真 prefill（`prefix:true`） | — | API/端点支持时 |

---

## 已验证不可用的方案

| 方案 | 状态 | 备注 |
|---|---|---|
| DeepSeek `prefix:true` via Tencent tokenhub | 不可用 | 代理不透传该字段；模型重新生成而非续写。等官方端点直连后再启用 |

---

## 流式渲染建议

| 阶段 | 动作 |
|------|------|
| `HeaderEvent` | 按 `card` 值渲染卡片骨架 + loading skeleton |
| `DeltaEvent` | 可选：实时更新 `text` 类字段 |
| `DoneEvent` | 用完整 body 填充卡片，移除 skeleton |
