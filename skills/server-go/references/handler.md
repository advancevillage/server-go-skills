# Go 服务端 Handler 层设计

> 按资源拆分 + 四模式标配，避免巨型 Service 聚合

---

## 反模式：单一 Service 聚合

```go
// ❌ 一个接口装载所有业务
type Service interface {
    AuthToken(ctx, r)   (Response, error)
    CreateUser(ctx, r)  (Response, error)
    QueryUser(ctx, r)   (Response, error)
    CreateOrder(ctx, r) (Response, error)
    // ... 20+ 方法
}

type service struct {
    user, order, payment, asset Module  // 全量依赖
}
```

**问题**：

- 违反 ISP — 调用方只用 1/N，却被迫依赖整个接口
- 违反 SRP — 一个 struct 装载所有领域，演化与测试困难
- 违反 OCP — 新增功能必修改大接口
- 中间件方法（AuthToken）与业务方法（CreateOrder）混杂，无清晰边界

---

## 正模式：按资源拆 + 四模式标配

```
HTTP Request
   │
   ▼
[Decorator]   middleware（鉴权/日志/限流），独立于 handler
   │
   ▼
[Template Method]   Pipeline 骨架：Parse → Validate → Execute → Render
   │
   ▼
[Facade]   handler struct，仅持需要的 internal 子集
   │
   ▼
[Strategy]   ErrorMapper：internal error → HTTP 响应（统一兜底）
```

### 1. Template Method — Pipeline 骨架

```go
type Pipeline interface {
    Parse(ctx context.Context, r *http.Request)        (any, error)        // 请求 → DTO
    Validate(ctx context.Context, dto any)             error               // 业务前置
    Execute(ctx context.Context, dto any)              (any, error)        // 调 internal
    Render(ctx context.Context, dto, result any)       (Response, error)   // 装配响应
}

// Run 适配 Pipeline 为 HTTP handler；任一步出错统一进 ErrorMapper
func Run(p Pipeline, mapper ErrorMapper) HandlerFunc { ... }
```

**收益**：四步骨架统一，子类只填 hook，新增 API 不复制流程代码。

### 2. Strategy — ErrorMapper

```go
type ErrorMapper interface {
    Map(err error) Response
}

// 默认实现按 errors.Is 匹配 sentinel → 各类错误响应
type defaultErrorMapper struct{}
```

**收益**：错误转换集中，新增错误码扩 mapper 即可（OCP）。

### 3. Facade — Handler 聚合

```go
// handler struct 只持自己用到的 internal 接口（ISP）
type sessionHandler struct {
    code    code.Code        // 仅 AuthCode
    user    user.User        // 仅 UpsertByPhone
    session session.Session  // 仅 CreateToken
}
```

**收益**：通过组合编排业务（CARP），handler 不感知底层存储/SDK（DIP），最少知识（LOD）。

### 4. Decorator — Middleware

```go
// middleware 独立目录，与 handler 解耦
type Auth struct {
    session session.Session  // 仅持自身依赖
}
```

**收益**：横切关注点（鉴权/日志/限流）可任意叠加，handler 不需关心。

---

## 七大原则对齐

| 原则 | Handler 层落地 |
|------|----------------|
| OCP  | 新增 API：加 Pipeline 实现 + 路由配置 |
| SRP  | handler 只编排；internal 单一资源；middleware 单一横切 |
| LSP  | Pipeline 多实现可互换 |
| DIP  | handler 依赖 internal 接口，不依赖具体实现 |
| ISP  | 每个 handler 持最小接口子集 |
| CARP | 全组合，无继承 |
| LOD  | handler 只与直接依赖通信 |

---

## 与既有模式参考的关系

- 横切关注点（限流/缓存/鉴权代理）的 Proxy 实现 → 见 `pattern.md`
- 对象构造与 Option 模式 → 见 `object.md`
- 目录布局与常量收口 → 见 `software.md`
