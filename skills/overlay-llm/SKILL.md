---
name: overlay-llm
description: 模型交互协议
---

# 模型交互协议

**OLAIP** — Overlay LLM App Interaction Protocol，`[olaip]` 标签名即协议缩写，帧边界由 `[/olaip]` 显式闭合。

---

## 帧格式

```
[olaip][header]k1:v1\nk2:v2\n...[/header][body]k1:v1\nk2:v2\n...[/body][/olaip]
```

**BNF：**

```
frame   ::= "[olaip][header]" LF? header "[/header][body]" LF? body "[/body][/olaip]"
header  ::= entry*
body    ::= entry*
entry   ::= key ":" value LF
key     ::= [a-z][a-z0-9_]*
value   ::= SP* [^\n]+ SP*
```

---

## 字段规则

- **键名**：`[a-z][a-z0-9_]*`，纯小写，大小写敏感，未知键名静默忽略
- **键值**：`:` 后到行尾，首尾空格 trim；空值行整行丢弃
- **重复键**：取最后一个值
- **顺序**：键值行顺序无关，解析方不依赖顺序
- **命名空间**：正文 k:v 与头部 k:v 独立，键名可重叠但语义互不干扰
- **终止令牌**：正文 value 中的 `[/body]` 须与其他内容同行，否则触发帧终止

---

## 状态机

```
                   64字节内无[olaip][header]
         ┌──────────────────────────────────┐
         │                                  ▼
       INIT ──[olaip][header]──► HEADER ──[/header][body]──► BODY ──[/body][/olaip]──► DONE
         │                          │                           │
         └──────eof()───────────────┴───────eof()───────────────┘
                                                     ▼
                                                    DONE
```

**输出事件：**

| 事件 | 触发时机 | 携带数据 |
|------|---------|---------|
| `HeaderEvent` | HEADER → BODY | `frame`（含头部 k:v） |
| `DeltaEvent` | BODY 每次安全输出 | `text`（原始文本块） |
| `DoneEvent` | 进入 DONE | `frame`（含头部 + 正文 k:v） |

完整伪代码与解析常量见 @references/spec.md。

---

## 安全约束

| 约束项 | 限制值 | 违反处理 |
|--------|--------|---------|
| `[olaip][header]` 出现位置 | 前 64 字节内 | 降级进 BODY，frame=default |
| 头部段总大小 | ≤ 512 bytes | 截断，降级，frame=default |
| k:v 对数量（头部） | ≤ 16 条 | 超出部分忽略 |
| 空 value | — | 整行忽略 |
| key 未知 | — | 静默忽略 |
| value 解析失败 | — | 业务层使用枚举默认值 |
| `[/body][/olaip]` 未出现 | — | eof() 降级触发 DoneEvent |

---

## 模型约束模板

```
你的每条回复必须严格遵守以下 OLAIP 帧格式：

[olaip][header]
<key1>:<枚举值>
<key2>:<枚举值>
[/header][body]
<key1>:<内容>
<key2>:<内容>
[/body][/olaip]

规则：
- [olaip][header] 必须是输出的第一个令牌
- 头部只允许 key:value 行，key 全小写，不得有空行
- [/header][body] 紧跟头部最后一行，中间无额外内容
- 正文同样为 key:value 行，key 全小写
- [/body][/olaip] 必须是输出的最后一个令牌
- 枚举值完全匹配，不得自造值
- 正文 value 中不得出现独立的 [/body] 字符串
```

---

## 卡片系统

卡片通过 header `card` 字段指定渲染模板，命名遵循 `namespace.variant` 两段式，全一维枚举。

**请求流程：**

```
本地卡片定义 ──┐
              ├──► System Prompt ──► LLM ──► OLAIP帧 ──► 解析 ──► 渲染
用户 Query ───┘
```

**降级规则：**

- `card` 值不在白名单 → 降级为 `chat.text`
- `required` 字段缺失 → 降级为 `chat.text`
- header 解析失败 → `frame=default`，原样渲染正文

卡片注册表见 @references/cards.md，System Prompt 注入模板见 @references/prompt.md。
