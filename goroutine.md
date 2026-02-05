# goroutine

> 本文基于 **Go 1.23.11** 版本。

## coroutine

## stackless, stackful

---

## Go 1.22 循环变量语义变更

> 这是 Go 语言历史上最重要的语义变更之一，修复了困扰 Go 开发者多年的经典陷阱。

### TL;DR - 面试速查

- **Go 1.22 之前**: 循环变量在所有迭代中共享，闭包捕获的是同一个变量
- **Go 1.22 之后**: 每次迭代都有独立的循环变量副本
- 变更由 `go.mod` 中的 `go` 版本控制
- 这是**非向后兼容**的变更，但对大多数代码是安全的

---

### 经典问题

在 Go 1.22 之前，以下代码是一个经典的 bug：

```go
// Go 1.21 及之前的行为
func main() {
    done := make(chan bool)
    values := []int{1, 2, 3, 4, 5}
    
    for _, v := range values {
        go func() {
            fmt.Println(v)  // 问题: 所有 goroutine 打印的都是 5
            done <- true
        }()
    }
    
    for range values {
        <-done
    }
}

// 输出 (Go 1.21): 5 5 5 5 5
// 输出 (Go 1.22+): 1 2 3 4 5 (顺序可能不同)
```

### 问题根源

```
┌─────────────────────────────────────────────────────────────┐
│                Go 1.21 及之前的循环变量行为                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  for _, v := range values {                                 │
│      go func() { fmt.Println(v) }()                         │
│  }                                                          │
│                                                             │
│  内存布局:                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  v (单一变量，地址固定)                               │   │
│  │  ┌───┐                                              │   │
│  │  │ 1 │ ← 第1次迭代                                   │   │
│  │  │ 2 │ ← 第2次迭代 (覆盖)                            │   │
│  │  │ 3 │ ← 第3次迭代 (覆盖)                            │   │
│  │  │ 4 │ ← 第4次迭代 (覆盖)                            │   │
│  │  │ 5 │ ← 第5次迭代 (覆盖)                            │   │
│  │  └───┘                                              │   │
│  │                                                     │   │
│  │  所有 goroutine 闭包都引用同一个 v                    │   │
│  │  当 goroutine 执行时，v 已经是 5                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                Go 1.22+ 的循环变量行为                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  for _, v := range values {                                 │
│      go func() { fmt.Println(v) }()                         │
│  }                                                          │
│                                                             │
│  内存布局:                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  每次迭代创建新的 v                                   │   │
│  │                                                     │   │
│  │  迭代1: v₁ ┌───┐                                    │   │
│  │           │ 1 │ ← goroutine 1 捕获                  │   │
│  │           └───┘                                     │   │
│  │  迭代2: v₂ ┌───┐                                    │   │
│  │           │ 2 │ ← goroutine 2 捕获                  │   │
│  │           └───┘                                     │   │
│  │  迭代3: v₃ ┌───┐                                    │   │
│  │           │ 3 │ ← goroutine 3 捕获                  │   │
│  │           └───┘                                     │   │
│  │  ...                                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 旧版本的修复方式

在 Go 1.22 之前，开发者需要手动修复这个问题：

```go
// 方式 1: 创建局部变量副本
for _, v := range values {
    v := v  // 创建新的局部变量，shadowing 外层的 v
    go func() {
        fmt.Println(v)
    }()
}

// 方式 2: 通过参数传递
for _, v := range values {
    go func(val int) {
        fmt.Println(val)
    }(v)
}
```

### Go 1.22+ 的行为

Go 1.22 改变了循环变量的语义，**每次迭代都会创建新的变量副本**：

```go
// Go 1.22+ 自动修复
for _, v := range values {
    go func() {
        fmt.Println(v)  // 现在正确捕获每次迭代的值
    }()
}
```

### 适用范围

变更适用于以下两种循环：

```go
// 1. for-range 循环
for i, v := range slice {
    // i 和 v 每次迭代都是新的
}

// 2. 三段式 for 循环
for i := 0; i < n; i++ {
    // i 每次迭代都是新的
}
```

**不适用**于使用已存在变量的循环：

```go
var i int
for i = 0; i < n; i++ {
    // i 仍然是同一个变量（因为是赋值，不是声明）
}
```

### 版本控制

变更由 `go.mod` 中的 `go` 版本控制：

```go
// go.mod
module myproject

go 1.22  // 使用新语义

// 或
go 1.21  // 使用旧语义
```

**实验性启用 (Go 1.21)**：

```bash
GOEXPERIMENT=loopvar go run main.go
```

### 潜在影响

虽然这个变更对大多数代码是有益的，但在某些边缘情况下可能有影响：

#### 1. defer 中捕获循环变量

```go
// Go 1.21 行为
for i := 0; i < 3; i++ {
    defer fmt.Println(i)
}
// 输出: 2 2 2 (LIFO 顺序)

// Go 1.22 行为
for i := 0; i < 3; i++ {
    defer fmt.Println(i)
}
// 输出: 2 1 0 (LIFO 顺序，每个 defer 捕获不同的 i)
```

#### 2. 取循环变量地址

```go
// Go 1.21: 所有指针指向同一地址
var ptrs []*int
for i := 0; i < 3; i++ {
    ptrs = append(ptrs, &i)
}
// ptrs[0] == ptrs[1] == ptrs[2]

// Go 1.22: 每个指针指向不同地址
var ptrs []*int
for i := 0; i < 3; i++ {
    ptrs = append(ptrs, &i)
}
// ptrs[0] != ptrs[1] != ptrs[2]
```

#### 3. 性能考虑

对于大型循环变量，每次迭代创建副本可能有轻微性能影响：

```go
type LargeStruct struct {
    data [1024]byte
}

// 可能有性能影响
for _, v := range largeStructs {
    // v 每次迭代都是新副本
}

// 优化: 使用索引访问
for i := range largeStructs {
    v := &largeStructs[i]  // 使用指针避免复制
}
```

### 检测工具

Go 提供了工具来检测可能受影响的代码：

```bash
# 使用 loopclosure 分析器
go vet -loopclosure ./...

# 使用 loopvar 实验检测
GOEXPERIMENT=loopvar go build -gcflags="-d=loopvar=2" ./...
```

### Interview Cheatsheet

**Q1: Go 1.22 对 for 循环做了什么改变？**

> Go 1.22 改变了循环变量的作用域：每次迭代都会创建新的变量副本，而不是所有迭代共享同一个变量。这修复了闭包捕获循环变量的经典 bug。

**Q2: 这个变更会破坏现有代码吗？**

> 理论上可能，但实际上对大多数代码是有益的。变更由 go.mod 中的版本控制，只有 go 1.22+ 的模块才会使用新语义。

**Q3: 如何在 Go 1.21 中提前使用新语义？**

> 设置环境变量 `GOEXPERIMENT=loopvar`。

**Q4: 这个变更对性能有影响吗？**

> 对于大型循环变量，每次迭代创建副本可能有轻微开销。可以通过使用索引和指针来优化。

---

### 参考资料

- [Go 1.22 Release Notes](https://go.dev/doc/go1.22)
- [Loop Variable Experiment](https://go.googlesource.com/wiki/+/refs/heads/master/LoopvarExperiment.md)
- [for Loop Semantic Changes in Go 1.22](https://go101.org/blog/2024-03-01-for-loop-semantic-changes-in-go-1.22.html)
