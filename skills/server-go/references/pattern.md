# Go 服务端设计模式参考

> 基于 GoF 23 种设计模式 + 七大原则，结合 Go 语言惯用法

---

## 七大设计原则

### 1. 开闭原则（OCP）— 对扩展开放，对修改关闭

```go
// 通过接口抽象，新增图表类型无需修改已有代码
type Chart interface {
    Draw() string
}

type BarChart struct{}
func (b *BarChart) Draw() string { return "draw bar chart" }

type LineChart struct{}
func (l *LineChart) Draw() string { return "draw line chart" }

// 新增饼图只需实现接口，不修改任何已有代码
type PieChart struct{}
func (p *PieChart) Draw() string { return "draw pie chart" }

func Render(c Chart) { fmt.Println(c.Draw()) }
```

### 2. 单一职责原则（SRP）— 一个类只负责一个功能

```go
// 错误：UserService 混杂了业务逻辑和持久化
type UserService struct{}
func (s *UserService) Register(name string) error { /* 校验 + 存库 + 发邮件 */ return nil }

// 正确：职责分离
type UserValidator struct{}
func (v *UserValidator) Validate(name string) error { return nil }

type UserRepository struct{}
func (r *UserRepository) Save(name string) error { return nil }

type EmailNotifier struct{}
func (e *EmailNotifier) SendWelcome(name string) error { return nil }
```

### 3. 里氏代换原则（LSP）— 子类可替换基类

```go
// 基类接口
type Animal interface {
    Breathe() string
}

type Bird struct{}
func (b *Bird) Breathe() string { return "bird breathes via lungs" }

type Fish struct{}
func (f *Fish) Breathe() string { return "fish breathes via gills" }

// 任何接受 Animal 的地方都可传入 Bird 或 Fish
func DescribeBreathing(a Animal) {
    fmt.Println(a.Breathe())
}
```

### 4. 依赖倒转原则（DIP）— 依赖抽象，不依赖具体

```go
// 抽象
type MessageSender interface {
    Send(to, body string) error
}

// 具体实现
type EmailSender struct{}
func (e *EmailSender) Send(to, body string) error { return nil }

type SMSSender struct{}
func (s *SMSSender) Send(to, body string) error { return nil }

// 高层模块依赖抽象，通过构造注入
type NotificationService struct {
    sender MessageSender
}

func NewNotificationService(sender MessageSender) *NotificationService {
    return &NotificationService{sender: sender}
}

func (n *NotificationService) Notify(user, msg string) error {
    return n.sender.Send(user, msg)
}
```

### 5. 接口隔离原则（ISP）— 建立最小接口

```go
// 错误：接口过于臃肿
type BigStorage interface {
    Read(key string) ([]byte, error)
    Write(key string, val []byte) error
    Delete(key string) error
    Scan(prefix string) ([]string, error)
    Stats() map[string]int64
}

// 正确：按使用场景细化接口
type Reader interface {
    Read(key string) ([]byte, error)
}

type Writer interface {
    Write(key string, val []byte) error
}

type ReadWriter interface {
    Reader
    Writer
}
```

### 6. 合成/聚合复用原则（CARP）— 组合优于继承

```go
// 使用组合而非继承实现 Person HAS-A Hand
type Hand interface {
    Swing() string
}

type LeftHand struct{}
func (l *LeftHand) Swing() string { return "left hand swings" }

type RightHand struct{}
func (r *RightHand) Swing() string { return "right hand swings" }

// Go 天然支持组合，无继承，直接持有
type Person struct {
    left  Hand
    right Hand
}

func NewPerson() *Person {
    return &Person{left: &LeftHand{}, right: &RightHand{}}
}

func (p *Person) SwingLeft() string  { return p.left.Swing() }
func (p *Person) SwingRight() string { return p.right.Swing() }
```

### 7. 迪米特法则（LOD）— 最少知识，只与直接朋友通信

```go
// 中间类屏蔽细节
type Girl struct{ Name string }

type GroupLeader struct {
    girls []Girl
}

func (g *GroupLeader) CountGirls() int { return len(g.girls) }

type Teacher struct{}

// Teacher 只与 GroupLeader 交互，不直接操作 []Girl
func (t *Teacher) Command(leader *GroupLeader) {
    fmt.Printf("girls count: %d\n", leader.CountGirls())
}
```

---

## 创建型模式

### Singleton — 单例

```go
// 服务端常见：全局配置、连接池、日志器
type config struct {
    DSN string
}

var (
    instance *config
    once     sync.Once
)

func GetConfig() *config {
    once.Do(func() {
        instance = &config{DSN: os.Getenv("DB_DSN")}
    })
    return instance
}
```

**适用**：全局唯一资源（DB 连接池、配置中心客户端）。
**权衡**：增加全局状态，测试时难以替换 → 优先用依赖注入传递单例。

---

### Factory Method — 工厂方法

```go
type Logger interface {
    Log(msg string)
}

type JSONLogger struct{}
func (j *JSONLogger) Log(msg string) { fmt.Printf(`{"msg":%q}`+"\n", msg) }

type TextLogger struct{}
func (t *TextLogger) Log(msg string) { fmt.Println(msg) }

// 工厂函数根据配置创建对应实现
func NewLogger(format string) Logger {
    switch format {
    case "json":
        return &JSONLogger{}
    default:
        return &TextLogger{}
    }
}
```

**适用**：同一接口有多种实现，创建逻辑需要封装。

---

### Builder — 构建者

```go
// 适合参数多、部分可选的对象构建（HTTP Client、gRPC Server）
type ServerConfig struct {
    Addr         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    MaxConns     int
}

type ServerBuilder struct {
    cfg ServerConfig
}

func NewServerBuilder(addr string) *ServerBuilder {
    return &ServerBuilder{cfg: ServerConfig{Addr: addr, MaxConns: 1000}}
}

func (b *ServerBuilder) ReadTimeout(d time.Duration) *ServerBuilder {
    b.cfg.ReadTimeout = d
    return b
}

func (b *ServerBuilder) WriteTimeout(d time.Duration) *ServerBuilder {
    b.cfg.WriteTimeout = d
    return b
}

func (b *ServerBuilder) Build() ServerConfig { return b.cfg }

// 使用
// cfg := NewServerBuilder(":8080").ReadTimeout(5*time.Second).Build()
```

**适用**：构建复杂对象，参数多且有默认值，链式调用提升可读性。

---

## 结构型模式

### Proxy — 代理

```go
// 常见于：缓存代理、权限代理、限流代理
type UserRepo interface {
    FindByID(id int64) (*User, error)
}

type User struct{ ID int64; Name string }

type DBUserRepo struct{}
func (r *DBUserRepo) FindByID(id int64) (*User, error) {
    return &User{ID: id, Name: "alice"}, nil
}

// 缓存代理，对调用方透明
type CachedUserRepo struct {
    origin UserRepo
    cache  map[int64]*User
    mu     sync.RWMutex
}

func (c *CachedUserRepo) FindByID(id int64) (*User, error) {
    c.mu.RLock()
    if u, ok := c.cache[id]; ok {
        c.mu.RUnlock()
        return u, nil
    }
    c.mu.RUnlock()
    u, err := c.origin.FindByID(id)
    if err != nil {
        return nil, err
    }
    c.mu.Lock()
    c.cache[id] = u
    c.mu.Unlock()
    return u, nil
}
```

**适用**：在不修改原逻辑的前提下增加横切关注点（缓存、日志、鉴权、限流）。

---

### Decorator — 装饰器

```go
// HTTP middleware 本质就是装饰器
type HandlerFunc func(w http.ResponseWriter, r *http.Request)

func WithLogging(next HandlerFunc) HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    }
}

func WithAuth(next HandlerFunc) HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        next(w, r)
    }
}

// 叠加装饰：先鉴权，再记日志
// handler = WithLogging(WithAuth(myHandler))
```

**适用**：为函数/对象动态叠加行为，且行为可自由组合。Go 的 middleware 链是最典型应用。

---

### Adapter — 适配器

```go
// 旧接口
type LegacyPayment struct{}
func (l *LegacyPayment) PayWithCard(amount int, cardNo string) error { return nil }

// 新接口
type PaymentGateway interface {
    Pay(ctx context.Context, amount int, token string) error
}

// 适配器将旧接口包装成新接口
type LegacyAdapter struct {
    legacy *LegacyPayment
}

func (a *LegacyAdapter) Pay(ctx context.Context, amount int, token string) error {
    return a.legacy.PayWithCard(amount, token)
}
```

**适用**：集成第三方库或遗留系统时统一接口。

---

### Facade — 外观

```go
// 将多个子系统操作封装成单一入口，隐藏复杂性
type OrderFacade struct {
    inventory *InventoryService
    payment   *PaymentService
    notify    *NotifyService
}

func (f *OrderFacade) PlaceOrder(ctx context.Context, userID int64, itemID int64, amount int) error {
    if err := f.inventory.Reserve(ctx, itemID, 1); err != nil {
        return fmt.Errorf("reserve: %w", err)
    }
    if err := f.payment.Charge(ctx, userID, amount); err != nil {
        _ = f.inventory.Release(ctx, itemID, 1)
        return fmt.Errorf("charge: %w", err)
    }
    _ = f.notify.Send(ctx, userID, "order placed")
    return nil
}
```

**适用**：微服务内部聚合多个领域操作，对外暴露简单接口。

---

## 行为型模式

### Strategy — 策略

```go
// 运行时切换算法
type SortStrategy interface {
    Sort(data []int) []int
}

type QuickSort struct{}
func (q *QuickSort) Sort(data []int) []int { /* ... */ return data }

type MergeSort struct{}
func (m *MergeSort) Sort(data []int) []int { /* ... */ return data }

type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(st SortStrategy) { s.strategy = st }
func (s *Sorter) Execute(data []int) []int    { return s.strategy.Sort(data) }
```

**适用**：同一行为有多种实现，且需要在运行时切换（限流策略、路由策略、序列化策略）。

---

### Observer — 观察者

```go
// 事件驱动：订单状态变更通知多个下游
type EventHandler func(event string, data any)

type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]EventHandler
}

func (b *EventBus) Subscribe(event string, h EventHandler) {
    b.mu.Lock()
    b.handlers[event] = append(b.handlers[event], h)
    b.mu.Unlock()
}

func (b *EventBus) Publish(event string, data any) {
    b.mu.RLock()
    hs := b.handlers[event]
    b.mu.RUnlock()
    for _, h := range hs {
        h(event, data)
    }
}

// 使用
// bus.Subscribe("order.paid", sendInvoice)
// bus.Subscribe("order.paid", updateStats)
// bus.Publish("order.paid", orderID)
```

**适用**：一对多依赖，状态变更需通知多个下游，解耦发布者与订阅者。

---

### Chain of Responsibility — 责任链

```go
// 请求依次经过多个处理器，直到被处理
type Middleware func(ctx context.Context, req any) (any, error)

type Chain struct {
    handlers []Middleware
}

func (c *Chain) Use(m Middleware) { c.handlers = append(c.handlers, m) }

func (c *Chain) Execute(ctx context.Context, req any) (any, error) {
    var idx int
    var next func(ctx context.Context, req any) (any, error)
    next = func(ctx context.Context, req any) (any, error) {
        if idx >= len(c.handlers) {
            return nil, nil
        }
        h := c.handlers[idx]
        idx++
        return h(ctx, req)
    }
    return next(ctx, req)
}
```

**适用**：gRPC/HTTP 拦截器链、校验管道、审批流程。

---

### Command — 命令

```go
// 将请求封装为对象，支持排队、撤销、重做
type Command interface {
    Execute(ctx context.Context) error
    Undo(ctx context.Context) error
}

type TransferCommand struct {
    from, to int64
    amount   int
    ledger   *Ledger
}

func (c *TransferCommand) Execute(ctx context.Context) error {
    return c.ledger.Transfer(ctx, c.from, c.to, c.amount)
}

func (c *TransferCommand) Undo(ctx context.Context) error {
    return c.ledger.Transfer(ctx, c.to, c.from, c.amount)
}

// 命令总线，支持异步执行
type CommandBus struct {
    queue chan Command
}

func (b *CommandBus) Dispatch(cmd Command) {
    b.queue <- cmd
}
```

**适用**：操作审计、Undo/Redo、异步任务队列、CQRS 写侧。

---

### Template Method — 模板方法

```go
// Go 用函数参数或接口的钩子实现模板方法
type DataImporter interface {
    Read() ([]byte, error)
    Parse(data []byte) ([]Record, error)
    Save(records []Record) error
}

// 模板：固定流程，具体步骤由实现决定
func RunImport(ctx context.Context, importer DataImporter) error {
    data, err := importer.Read()
    if err != nil {
        return fmt.Errorf("read: %w", err)
    }
    records, err := importer.Parse(data)
    if err != nil {
        return fmt.Errorf("parse: %w", err)
    }
    return importer.Save(records)
}
```

**适用**：固定算法骨架、步骤可变（ETL 流程、数据导入导出、报表生成）。

---

## 服务端常用模式组合

| 场景 | 推荐模式 |
|------|---------|
| HTTP/gRPC 中间件 | Decorator + Chain of Responsibility |
| 多数据源/缓存 | Proxy + Strategy |
| 事件驱动解耦 | Observer + Command |
| 复杂对象初始化 | Builder + Singleton |
| 跨服务适配 | Adapter + Facade |
| 审计 / 幂等操作 | Command + Template Method |

---

## 原则应用速查

| 症状 | 对应原则 | 解法 |
|------|---------|------|
| 新增功能总要改老代码 | OCP | 抽象接口，扩展实现 |
| 一个结构体/函数做太多事 | SRP | 拆分职责 |
| 子类行为破坏父类契约 | LSP | 重新审视继承关系 |
| 高层模块依赖具体实现 | DIP | 注入接口，面向抽象编程 |
| 接口方法用不到一半 | ISP | 拆分小接口 |
| 继承关系不是 IS-A | CARP | 改用组合 |
| 两个模块强耦合 | LOD | 引入中间层/消息总线 |
