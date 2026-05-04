# Go 对象创建规范

> 接口定义 + 单一职责 + 泛型 Option 模式

---

## 规范要点

1. **定义接口**：对外暴露接口而非具体类型，遵循单一职责原则。
2. **构造函数签名**：`NewXXXClient(ctx context.Context, logger logx.ILoger, opts ...XXXOption) (IXXXClient, error)`
3. **Option 模式**：使用泛型 `Option[T]` 接口，调用方以函数式选项配置，内部有默认值兜底。

---

## 完整示例

```go
package anthropic

import (
    "context"
    "time"

    "github.com/zeromicro/go-zero/core/logx"
)

// ── 1. 对外接口（单一职责）────────────────────────────────────────────

type IAnthropicClient interface {
    Complete(ctx context.Context, prompt string) (string, error)
    // 只暴露该客户端的职责方法，不混入其他领域操作
}

// ── 2. 泛型 Option 基础设施 ──────────────────────────────────────────

type Option[T any] interface {
    apply(*T)
}

type funcOption[T any] struct {
    f func(*T)
}

func (o funcOption[T]) apply(do *T) {
    o.f(do)
}

func newFuncOption[T any](f func(*T)) *funcOption[T] {
    return &funcOption[T]{f: f}
}

// ── 3. 具体 Option 类型别名 + With 函数 ──────────────────────────────

type AnthropicOption = Option[anthropicOption]

func WithAnthropicApiKey(sk string) AnthropicOption {
    return newFuncOption(func(o *anthropicOption) {
        o.sk = sk
    })
}

func WithAnthropicAuthType(ak string) AnthropicOption {
    return newFuncOption(func(o *anthropicOption) {
        o.ak = ak
    })
}

func WithAnthropicTimeout(d time.Duration) AnthropicOption {
    return newFuncOption(func(o *anthropicOption) {
        o.timeout = d
    })
}

func WithAnthropicUrl(url string) AnthropicOption {
    return newFuncOption(func(o *anthropicOption) {
        o.anthropicUrl = url
    })
}

// ── 4. 内部配置结构 + 默认值 ─────────────────────────────────────────

type anthropicOption struct {
    ak           string
    sk           string
    timeout      time.Duration
    anthropicUrl string
}

var defaultAnthropicOption = anthropicOption{
    ak:           "oauth",
    timeout:      time.Second * 300,
    anthropicUrl: "https://api.anthropic.com",
}

// ── 5. 具体实现（私有）───────────────────────────────────────────────

type anthropicClient struct {
    opt    anthropicOption
    logger logx.ILoger
}

func (c *anthropicClient) Complete(ctx context.Context, prompt string) (string, error) {
    // 实现略
    return "", nil
}

// ── 6. 构造函数 ──────────────────────────────────────────────────────

// NewAnthropicClient 构造 Anthropic 客户端。
// 参数：ctx 用于初始化阶段（如预热连接），logger 注入日志器，opts 覆盖默认配置。
func NewAnthropicClient(ctx context.Context, logger logx.ILoger, opts ...AnthropicOption) (IAnthropicClient, error) {
    opt := defaultAnthropicOption
    for _, o := range opts {
        o.apply(&opt)
    }
    return &anthropicClient{opt: opt, logger: logger}, nil
}
```

---

## 关键约定

| 约定 | 说明 |
|------|------|
| 构造函数返回接口 | `IAnthropicClient` 而非 `*anthropicClient`，方便测试 mock |
| `ctx` 作为第一个参数 | 用于初始化阶段的超时、链路追踪 |
| `logger` 显式注入 | 不使用全局 logger，保持可测试性 |
| 内部配置结构私有 | `anthropicOption` 小写，调用方只能通过 `With*` 函数配置 |
| 默认值集中定义 | `defaultXXXOption` 提供合理默认，调用方按需覆盖 |
| `Option` 泛型复用 | `funcOption[T]` 基础设施在包内复用，不重复实现 |

---

## 使用方式

```go
client, err := anthropic.NewAnthropicClient(
    ctx,
    logger,
    anthropic.WithAnthropicApiKey("sk-xxx"),
    anthropic.WithAnthropicTimeout(60*time.Second),
)
if err != nil {
    return err
}
resp, err := client.Complete(ctx, "Hello")
```

---

## 适用场景

- 第三方 SDK 封装（AI、支付、短信、OSS 等）
- 内部基础设施客户端（Redis、MQ、搜索引擎）
- 任何需要多个可选配置项的对象初始化
