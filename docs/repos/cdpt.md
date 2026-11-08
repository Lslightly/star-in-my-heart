---
tags:
  - Rust
  - GC
---

# [kaist-cp/cdpt](https://github.com/kaist-cp/cdpt)

一个Rust GC项目。自动、安全、并发GC。有对应论文[Revisiting Partial Tracing for Safe, Efficient, and Concurrent Garbage Collection in Unmanaged Languages](../papers/kim2026revisitingpartialtracing.md)

特性：
- partial tracing: root采用引用计数，剩下的采用tracing

类型：
- `Shared<T>`，和`Arc<T>`类似
- `Local<'g, T>`，临时引用

局限性：
- 不能有定制化的finalizer
- 不能有unboxing。不知道什么含义，但是大概就是如果一个对象称为另一个对象的堆边，那么这个对象就不能移动回栈上
- 内部可变的的状态需要用并发安全的原语
- 相比于手动回收，更高的内存足迹

暂时还没搞清楚原理。
