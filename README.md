# server-go-skills

服务端 Go 开发技能集，设计原则提炼自 Martin Kleppmann《Designing Data-Intensive Applications 2nd Edition》。

## 核心原则

**可靠性** — 故障是常态，设计要容错，不假设无故障。

**可扩展性** — 先量化负载，再谈方案，不过早优化。

**可维护性** — 可观测 + 简单 + 可演化，三位一体。

## 设计自检

- 这个操作幂等吗？重试安全吗？
- 节点崩溃时数据会丢失吗？
- 并发执行有竞争条件吗？
- 数据量增长 10 倍，瓶颈在哪里？
- 这个复杂性是本质的还是偶然的？

## 参考

- [Designing Data-Intensive Applications](https://dataintensive.net/) — Martin Kleppmann
