# 卡片注册表

命名规范：`namespace.variant` 两段式，`.` 分隔，全一维枚举，渲染层按 `card` 值直接查表。

---

## chat.text

**触发**：普通对话，简短回复

```
[olaip][header]
card:chat.text
[/header][body]
text:你好！有什么可以帮你的吗？
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `text` | ✅ | 回复正文 |

---

## chat.titled

**触发**：回答有明确主题

```
[olaip][header]
card:chat.titled
[/header][body]
title:光合作用是什么？
text:光合作用是植物利用阳光将二氧化碳和水转化为葡萄糖和氧气的过程，发生在叶绿体中。
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `title` | ✅ | 卡片标题 |
| `text` | ✅ | 正文 |

---

## chat.sectioned

**触发**：回答包含 3 个以上要点

```
[olaip][header]
card:chat.sectioned
[/header][body]
title:如何提高睡眠质量
section1_heading:规律作息
section1_text:每天同一时间入睡和起床，帮助身体建立生物钟。
section2_heading:睡前环境
section2_text:保持房间黑暗、安静、凉爽，避免睡前使用手机。
section3_heading:避免刺激物
section3_text:咖啡因半衰期约6小时，下午三点后建议不再摄入。
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `title` | ✅ | 整体标题 |
| `section1_heading` | ✅ | 第一节小标题 |
| `section1_text` | ✅ | 第一节内容 |
| `section2_heading` ~ `sectionN_heading` | ❌ | 更多节，连续编号 |
| `section2_text` ~ `sectionN_text` | ❌ | 对应内容 |

---

## chat.highlight

**触发**：需突出某个结论或重点

```
[olaip][header]
card:chat.highlight
[/header][body]
text:根据你描述的症状，最可能是睡眠不足导致的注意力下降。
highlight:建议每晚保证 7–8 小时睡眠，坚持两周后再观察变化。
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `text` | ✅ | 正文说明 |
| `highlight` | ✅ | 突出结论，语义色块展示 |

---

## chat.tip

**触发**：纠错、建议、警告

```
[olaip][header]
card:chat.tip
emotion:warning
[/header][body]
text:这道题的解法方向有点偏，乘法分配律在这里不适用。
tip:试着先通分，把两个分数化为相同分母再运算。
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `text` | ✅ | 正文说明 |
| `tip` | ✅ | 提示/建议内容，色块展示 |

| header 字段 | 可选值 | 说明 |
|------------|--------|------|
| `emotion` | `warning`（默认）/ `error` | 控制色块颜色语义 |

---

## chat.table

**触发**：对比、统计、列表类数据

```
[olaip][header]
card:chat.table
[/header][body]
title:2024年各季度营收对比
cols:季度|营收(万)|环比增长|备注
row1:Q1|1200|—|基准期
row2:Q2|1450|+20.8%|新品上线
row3:Q3|1380|-4.8%|促销减少
row4:Q4|1890|+36.9%|双十一拉动
footer:全年合计|5920|+32.4%|—
caption:数据来源：内部财务系统
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `cols` | ✅ | 列标题，`\|` 分隔，决定列数 |
| `row1` ~ `rowN` | ✅ | 数据行，`\|` 分隔，连续编号不得跳号 |
| `title` | ❌ | 表格标题 |
| `footer` | ❌ | 汇总行，样式与数据行区分 |
| `caption` | ❌ | 底部注释 |

**解析规则**：某行列数不足补 `—`，超出截断；值中不得含 `|`，用 `/` 替代；行数建议 ≤ 20。

---

## chat.math

**触发**：数学公式、推导步骤

```
[olaip][header]
card:chat.math
[/header][body]
title:求解一元二次方程
text:对于标准形式的一元二次方程，可以用求根公式直接求解。
formula:x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
note:其中 a≠0，判别式 Δ=b²-4ac 决定根的类型
[/body][/olaip]
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `formula` | ✅ | 主公式，LaTeX 字符串，block 级渲染（KaTeX） |
| `title` | ❌ | 卡片标题 |
| `text` | ❌ | 公式前说明 |
| `note` | ❌ | 公式下方补充说明 |
| `formula2` ~ `formulaN` | ❌ | 多公式依次编号 |
| `step1_label` ~ `stepN_label` | ❌ | 推导步骤标注，与 `formulaN` 对应 |

**渲染说明**：客户端用 KaTeX 渲染，`formula` 值为合法 LaTeX 字符串。
