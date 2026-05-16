---
name: overlay-llm
description: OLAIP 基于大模型应用交互协议
---

## 协议命名注解

**OLAIP** — Overlay LLM App Interaction Protocol
基于大模型应用交互协议

`[olaip]` 标签名即协议缩写，帧结构自注释。接收方见开标签即知协议类型，帧边界由 `[/olaip]` 显式闭合。

---

## 协议标准格式

```
[olaip][header]k1:v1\nk2:v2\n...[/header][body]k1:v1\nk2:v2\n...[/body][/olaip]
```

展开形式（等价）：
```
[olaip][header]
card:encourage
emotion:excited
lang:zh
[/header][body]
text:太棒了！你答对了这道题。
action:next_question
[/body][/olaip]
```

**BNF（Backus-Naur Form / 巴科斯-诺尔范式）**：精确描述合法输入语法的元语言，`::=` 读作"定义为"，`*` 表示零次或多次。

```
frame   ::= "[olaip][header]" LF? header "[/header][body]" LF? body "[/body][/olaip]"
header  ::= entry*
body    ::= entry*
entry   ::= key ":" value LF
key     ::= [a-z][a-z0-9_]*
value   ::= SP* [^\n]+ SP*
LF      ::= "\n"
SP      ::= " "
```

---

## 头部字段定义

### 键名格式规则

**要求：** `[a-z][a-z0-9_]*`，纯小写，大小写敏感，未知键名静默忽略

**示例：**
```
card:encourage
emotion_level:high
lang:zh
```

**禁用：**
```
Card:encourage     ← 含大写
 card:value        ← 前导空格
123key:value       ← 数字开头
```

---

### 键值格式规则

**要求：** `:` 后到行尾，首尾空格 trim；空值行整行丢弃；值语义由业务层定义

**示例：**
```
card:encourage
lang: zh           ← trim 后为 zh，合法
```

**禁用：**
```
card:              ← 空值，整行丢弃
card:   　　       ← 全空格，trim 后为空，整行丢弃
```

---

### 重复键名处理

**要求：** 同一键名可重复出现，取最后一个值；解析方不假设键唯一

**示例：**
```
card:chat
card:encourage     ← 最终值为 encourage
```

**禁用：**
```
（重复出现合法，解析层无禁止场景）
```

---

### 键值顺序规则

**要求：** 键值行顺序无关；解析方不依赖任何顺序假设

**示例：**
```
lang:zh
card:encourage
emotion:happy
```

**禁用：**
```
（协议层无顺序要求；生成方若有约定顺序需在 prompt 中声明）
```

---

### 头部安全约束

**要求：** 头部段（`[olaip][header]` 与 `[/header]` 之间）总字节数 ≤ 512 bytes；k:v 对数 ≤ 16 条；超出部分忽略

**示例：**
```
[olaip][header]
card:encourage
emotion:happy
[/header][body]
...
[/body][/olaip]
```

**禁用：**
```
（超出 512 bytes 的头部段 → 截断，降级，frame=default）
```

---

## 正文内容定义

### 正文内容类型

**要求：** `[body]` 与 `[/body]` 之间为 k:v 键值对，格式规则与头部一致；协议层流式透传，完整正文在 DoneEvent 后结构化解析

**示例：**
```
[/header][body]
text:太棒了！你答对了这道题。
action:next_question
[/body][/olaip]
```

**说明：** 正文 k:v 与头部 k:v 使用独立键名空间，键名可重叠但语义互不干扰。

**禁用：**
```
（协议层无正文内容限制；枚举约束在业务 prompt 中声明）
```

---

### 终止标记冲突

**要求：** `[/body][/olaip]` 为帧终止令牌；正文 value 中出现字面量 `[/body]` 时须同行含其他内容；协议层不提供转义，由 prompt 规避

**示例（合法）：**
```
text:参见 [/body] 章节    ← 作为 value 一部分，不触发终止
```

**禁用：**
```
[/body][/olaip]            ← 独立出现，触发帧终止
```

---

### 分块边界处理

**要求：** chunk 边界不保证对齐标签或字符；解析器须维护跨 chunk 滚动 buffer；BODY 状态保留末尾 `LOOKAHEAD_SIZE` 字节防终止令牌跨 chunk 截断

**示例：**
```
chunk1: "text:内容\n[/bo"
chunk2: "dy][/olaip]"     ← 合并后检测到 [/body][/olaip]，正确终止
```

**禁用：**
```
假设单个 chunk 是完整标签   ← 会导致终止令牌漏检或正文截断
```

---

## 状态机全定义

### 状态集合定义

```
{ INIT, HEADER, BODY, DONE }   初始: INIT   终止: DONE
```

### 解析内部变量

| 变量 | 类型 | 用途 |
|------|------|------|
| `scanBuffer` | string | 全局扫描 buffer，跨状态复用 |
| `headerBytes` | int | 已接收头部字节数 |
| `kvMap` | map | 头部解析出的 key:value 对 |
| `bodyAccum` | string | 正文累积 buffer，用于 DoneEvent 时 k:v 解析 |
| `frame` | Frame | 解析结果，初始为 default |
| `lookahead` | string | 末尾滚动 buffer（BODY 阶段） |

### 解析常量定义

| 常量 | 值 | 说明 |
|------|----|------|
| `FRAME_OPEN` | `"[olaip][header]"` | 帧开始令牌，15 字节 |
| `HEADER_CLOSE` | `"[/header]"` | 头部关闭令牌，9 字节 |
| `BODY_OPEN` | `"[body]"` | 正文开始令牌，6 字节 |
| `END_MARKER` | `"[/body][/olaip]"` | 帧结束令牌，16 字节 |
| `OPEN_SEARCH_LIMIT` | `64` | INIT 阶段搜索 FRAME_OPEN 的最大字节数 |
| `MAX_HEADER_BYTES` | `512` | 头部段最大字节数 |
| `LOOKAHEAD_SIZE` | `15` | `len(END_MARKER)-1`，防跨 chunk 截断 |

---

### 初始等待状态

```
输入 chunk(data):
  scanBuffer += data
  pos = find(FRAME_OPEN, scanBuffer)
  if pos >= 0:
    scanBuffer = scanBuffer[pos + len(FRAME_OPEN)..]
    → HEADER; headerBytes=0 / kvMap={}
    将 scanBuffer 送入 HEADER; return
  if len(scanBuffer) > OPEN_SEARCH_LIMIT:
    → BODY(frame=default); 将 scanBuffer 送入 BODY

输入 eof():
  → DONE; emit DoneEvent(frame=default)
```

### 头部解析状态

```
输入 chunk(data):
  scanBuffer += data; headerBytes += len(data)
  if headerBytes > MAX_HEADER_BYTES:
    emit HeaderEvent(frame=default); → BODY(frame=default)
    将 scanBuffer 送入 BODY; return
  pos = find(HEADER_CLOSE, scanBuffer)
  if pos >= 0:
    headerContent = scanBuffer[0..pos]
    remainder = scanBuffer[pos + len(HEADER_CLOSE)..]
    for line in split(headerContent, "\n"):
      line = trim(line)
      if line 匹配 /^[a-z][a-z0-9_]*:.+$/:
        kvMap[key] = trim(value)         // 重复 key 取最后值
    frame = parse(kvMap)
    emit HeaderEvent(frame)
    remainder = trimLeft(remainder)
    if startsWith(remainder, BODY_OPEN):
      remainder = remainder[len(BODY_OPEN)..]
    → BODY; scanBuffer=""; lookahead=remainder; bodyAccum=""
    将 lookahead 送入 BODY

输入 eof():
  → DONE; emit DoneEvent(frame=default)
```

### 正文流式状态

```
输入 chunk(data):
  lookahead += data
  loop:
    pos = find(END_MARKER, lookahead)
    if pos >= 0:
      bodyChunk = lookahead[0..pos]
      if bodyChunk != "": emit DeltaEvent(bodyChunk)
      bodyAccum += bodyChunk
      // 解析完整正文 k:v
      for line in split(bodyAccum, "\n"):
        line = trim(line)
        if line 匹配 /^[a-z][a-z0-9_]*:.+$/:
          frame.body[key] = trim(value)
      → DONE; emit DoneEvent(frame); break
    safe_len = len(lookahead) - LOOKAHEAD_SIZE
    if safe_len > 0:
      safeChunk = lookahead[0..safe_len]
      emit DeltaEvent(safeChunk)
      bodyAccum += safeChunk
      lookahead = lookahead[safe_len..]
    break

输入 eof():
  bodyAccum += lookahead
  if lookahead != "": emit DeltaEvent(lookahead)
  → DONE; emit DoneEvent(frame)          // 无 [/body][/olaip]，降级结束
```

### 解析终止状态

终止状态，忽略后续所有输入。

---

### 状态转移总图

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

### 输出事件汇总

| 事件 | 触发时机 | 携带数据 |
|------|---------|---------|
| `HeaderEvent` | HEADER → BODY | `frame: Frame`（含头部 k:v） |
| `DeltaEvent` | BODY 每次安全输出 | `text: string`（原始文本块） |
| `DoneEvent` | 进入 DONE | `frame: Frame`（含头部 + 正文 k:v） |

---

## 协议安全约束

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

## 完整协议示例

### 完整帧示例一

```
[olaip][header]
card:encourage
emotion:excited
lang:zh
[/header][body]
text:太棒了！你答对了这道题。继续保持，下一题来了！
action:next_question
[/body][/olaip]
```

### 分块流式示例

```
chunk 1 │ "[olaip][header]\ncard:encou"
chunk 2 │ "rage\nemotion:excited\nlang:zh\n[/header][body]\n"
chunk 3 │ "text:太棒了！你答对了这道题。\naction:nex"
chunk 4 │ "t_question\n[/body][/olaip]"
```

### 事件序列示例

```
chunk 1
  INIT: 见 "[olaip][header]" → HEADER
  scanBuffer = "card:encou"

chunk 2
  HEADER: 补全行 → card=encourage, emotion=excited, lang=zh
  见 "[/header]" → emit HeaderEvent(card=encourage, emotion=excited, lang=zh)
  消费 "[body]" → BODY; lookahead = ""; bodyAccum = ""

chunk 3
  BODY: lookahead = "text:太棒了！你答对了这道题。\naction:nex"
  safe_len = len - 15
  emit DeltaEvent("text:太棒了！你答对了这道题。\naction:")
  bodyAccum = "text:太棒了！你答对了这道题。\naction:"

chunk 4
  BODY: lookahead += "t_question\n[/body][/olaip]"
  find("[/body][/olaip]") → 命中
  emit DeltaEvent("t_question\n")
  bodyAccum = "text:太棒了！...action:next_question\n"
  frame.body = {text: "太棒了！...", action: "next_question"}
  → DONE; emit DoneEvent(frame)
```

### 协议降级示例

```
chunk 1 │ "同学你好，今天我们来学习"    ← 64字节内无 [olaip][header]
         → 降级进 BODY，frame=default
         emit DeltaEvent("同学你好，今天我们来学习")

eof()    → emit DoneEvent(frame=default)
```
