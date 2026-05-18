# OLAIP 实现规范

## 解析常量

| 常量 | 值 | 说明 |
|------|----|------|
| `FRAME_OPEN` | `"[olaip][header]"` | 帧开始令牌，15 字节 |
| `HEADER_CLOSE` | `"[/header]"` | 头部关闭令牌，9 字节 |
| `BODY_OPEN` | `"[body]"` | 正文开始令牌，6 字节 |
| `END_MARKER` | `"[/body][/olaip]"` | 帧结束令牌，16 字节 |
| `OPEN_SEARCH_LIMIT` | `64` | INIT 阶段搜索 FRAME_OPEN 的最大字节数 |
| `MAX_HEADER_BYTES` | `512` | 头部段最大字节数 |
| `LOOKAHEAD_SIZE` | `15` | `len(END_MARKER)-1`，防跨 chunk 截断 |

## 内部变量

| 变量 | 类型 | 用途 |
|------|------|------|
| `scanBuffer` | string | 全局扫描 buffer，跨状态复用 |
| `headerBytes` | int | 已接收头部字节数 |
| `kvMap` | map | 头部解析出的 key:value 对 |
| `bodyAccum` | string | 正文累积 buffer，用于 DoneEvent 时 k:v 解析 |
| `frame` | Frame | 解析结果，初始为 default |
| `lookahead` | string | 末尾滚动 buffer（BODY 阶段） |

---

## 状态伪代码

### INIT

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

### HEADER

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

### BODY

```
输入 chunk(data):
  lookahead += data
  loop:
    pos = find(END_MARKER, lookahead)
    if pos >= 0:
      bodyChunk = lookahead[0..pos]
      if bodyChunk != "": emit DeltaEvent(bodyChunk)
      bodyAccum += bodyChunk
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

### DONE

终止状态，忽略后续所有输入。

---

## 示例

### 完整帧

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

### 分块流式

```
chunk 1 │ "[olaip][header]\ncard:encou"
chunk 2 │ "rage\nemotion:excited\nlang:zh\n[/header][body]\n"
chunk 3 │ "text:太棒了！你答对了这道题。\naction:nex"
chunk 4 │ "t_question\n[/body][/olaip]"
```

### 事件序列 trace

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

### 降级

```
chunk 1 │ "同学你好，今天我们来学习"    ← 64字节内无 [olaip][header]
         → 降级进 BODY，frame=default
         emit DeltaEvent("同学你好，今天我们来学习")

eof()    → emit DoneEvent(frame=default)
```
