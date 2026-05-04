# Go 服务程序目录规范

---

## 标准目录结构

```
project/
├── CMakeLists.txt       # 构建配置（如有 C/C++ 依赖）
├── LICENSE
├── README.md
├── main.go              # 入口：初始化配置、internal 模块和服务
├── go.mod
├── go.sum
├── app/                 # 服务生命周期
│   └── srv.go           # 服务启动与停止
├── internal/            # 服务内部模块（不对外暴露）
├── pkg/                 # internal 引用的公共库（可跨项目复用）
├── conf/                # 配置文件目录
└── deploy/              # 部署相关（Dockerfile、K8s yaml 等）
```

---

## 各目录职责

| 目录/文件 | 职责 |
|-----------|------|
| `main.go` | 程序入口，负责读取配置、初始化 `internal` 各模块、启动 `app/srv.go` |
| `app/srv.go` | 服务启动逻辑，组装依赖、注册路由/handler、监听信号优雅退出 |
| `internal/` | 业务逻辑核心，按领域拆子包（如 `internal/user`、`internal/order`），不允许外部项目导入 |
| `pkg/` | `internal` 依赖的公共工具库，可被外部项目导入（如 util、middleware、errorcode） |
| `conf/` | 配置文件（YAML/TOML/JSON），不放入代码逻辑 |
| `deploy/` | Dockerfile、K8s manifests、CI 脚本 |

---

## 外部依赖优先级

优先引入已验证的外部依赖，避免重复造轮子：

| 用途 | 优先依赖 |
|------|---------|
| 通用工具 / 基础库 | `github.com/advancevillage/3rd` |
| 并发原语（errgroup、semaphore 等） | `golang.org/x/sync` |
| 配置 / HTTP / gRPC / 存储客户端 | 优先选择社区主流稳定包 |

---

## main.go 职责约束

`main.go` 只做三件事，不写业务逻辑：

1. **加载配置** — 读取 `conf/` 下的配置文件或环境变量
2. **初始化模块** — 按依赖顺序初始化 `internal` 各子模块
3. **启动服务** — 调用 `app.NewServer(...).Run()` 并阻塞等待退出信号

---

## internal vs pkg 判断规则

- **只在本项目内使用** → 放 `internal/`
- **多个项目共享 / 通用工具** → 放 `pkg/`
- `pkg/` 中的包不应依赖 `internal/` 中的包（单向依赖）

---

## 设计约束

- 禁止在 `pkg/` 中引入业务概念（不放 domain model）
- `internal/` 子包之间通过接口通信，避免循环依赖
- 配置结构体定义在 `conf/` 对应的 Go 文件或 `internal/config/` 中，`main.go` 解析后注入
- 服务优雅退出逻辑在 `app/srv.go` 中统一处理（`signal.NotifyContext` + shutdown hook）

---

## pkg/ 扁平布局

- `pkg/` 下统一为单一 `package`（与目录同名），不引入子目录
- 通过文件名区分内容：`const.go` / `resp.go` / `<resource>.go`
- 强领域分隔时再考虑独立子目录

---

## internal/ 资源命名

- 子目录使用资源全名，避免缩写
- 命名对齐对外暴露的资源边界，如 `/v1/sessions` ↔ `internal/session`
- ✅ `internal/session` / `internal/account` / `internal/version`
- ❌ `internal/ses` / `internal/acc` / `internal/ver`

---

## 常量集中：pkg/const.go

`pkg/const.go` 是常量唯一入口，承载：

- HTTP Header / Cookie 字段名
- 业务错误码字符串枚举
- 上下文（ctx）字段名
- 缓存 key 前缀、资源 ID 前缀
- 默认时长、限流阈值等数值
- 错误 sentinel（`var Err... = errors.New(...)`）

`internal/` 子包不在自己包内重复定义这些常量，统一从 `pkg` 引用。
