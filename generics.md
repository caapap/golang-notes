# Generics (泛型)

> 本文基于 **Go 1.23.11** 版本，全面介绍 Go 泛型的语法、实现原理和最佳实践。

## TL;DR - 面试速查

- Go 1.18 引入泛型，使用**类型参数 (Type Parameters)** 和**约束 (Constraints)**
- `any` 是 `interface{}` 的别名，允许任意类型
- `comparable` 约束支持 `==` 和 `!=` 操作，可用于 map key
- Go 泛型采用 **GCShape Stenciling** 混合实现：按 GC 形状分组 + 字典传递
- 与 C++ 模板（完全单态化）和 Java 泛型（类型擦除）不同

---

## 目录

1. [历史背景](#历史背景)
2. [基础语法](#基础语法)
3. [类型约束详解](#类型约束详解)
4. [GCShape 实现原理](#gcshape-实现原理)
5. [与其他语言的对比](#与其他语言的对比)
6. [最佳实践与性能影响](#最佳实践与性能影响)
7. [Interview Cheatsheet](#interview-cheatsheet)

---

## 历史背景

在泛型出现之前，社区里也有一些伪泛型方案，例如 [genny](https://github.com/cheekybits/genny)。

genny 本质是基于文本替换的，比如 example 里的例子：

```go
package queue

import "github.com/cheekybits/genny/generic"

// NOTE: this is how easy it is to define a generic type
type Something generic.Type

// SomethingQueue is a queue of Somethings.
type SomethingQueue struct {
  items []Something
}

func NewSomethingQueue() *SomethingQueue {
  return &SomethingQueue{items: make([]Something, 0)}
}
func (q *SomethingQueue) Push(item Something) {
  q.items = append(q.items, item)
}
func (q *SomethingQueue) Pop() Something {
  item := q.items[0]
  q.items = q.items[1:]
  return item
}
```

执行替换命令：

```shell
cat source.go | genny gen "Something=string"
```

然后 genny 会将代码中所有 Something 替换成 string，同时确保大小写与原来一致，不影响字段/类型的导出特征。

没有官方的泛型支持，社区怎么搞都是邪道。2021 年 1 月，官方的方案已经基本上成型，并释出了 [draft design](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md)。

**Go 1.18 (2022年3月)** 正式发布泛型支持，这是 Go 语言自诞生以来最重大的语法变更。

---

## 基础语法

### 类型参数 (Type Parameters)

类型参数使用方括号 `[]` 声明，放在函数名或类型名之后：

```go
// 泛型函数
func Print[T any](value T) {
    fmt.Println(value)
}

// 泛型类型
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() T {
    if len(s.items) == 0 {
        var zero T
        return zero
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}
```

### 类型实例化

```go
// 显式指定类型参数
s := Stack[int]{}
s.Push(1)
s.Push(2)

// 类型推断（编译器自动推断）
Print(42)        // T 推断为 int
Print("hello")   // T 推断为 string
```

### 多类型参数

```go
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

// 使用
nums := []int{1, 2, 3}
strs := Map(nums, func(n int) string {
    return fmt.Sprintf("%d", n)
})
```

---

## 类型约束详解

### any 约束

`any` 是 `interface{}` 的别名，允许任意类型：

```go
// 这两个声明等价
func Print[T any](v T)
func Print[T interface{}](v T)
```

使用 `any` 时，只能对值进行以下操作：
- 赋值
- 传递给其他 `any` 参数
- 类型断言/类型 switch

### comparable 约束

`comparable` 约束限制类型必须支持 `==` 和 `!=` 操作：

```go
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// 可用于 map 的 key
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

**注意**：Go 1.20 之前，`any` 不满足 `comparable`。Go 1.20+ 修复了这个问题。

### 自定义约束

使用接口定义约束：

```go
// 类型集合 (Type Set)
type Integer interface {
    int | int8 | int16 | int32 | int64
}

type Unsigned interface {
    uint | uint8 | uint16 | uint32 | uint64
}

type Float interface {
    float32 | float64
}

type Number interface {
    Integer | Unsigned | Float
}

func Sum[T Number](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}
```

### 近似约束 (~T)

`~T` 匹配底层类型为 T 的所有类型：

```go
type MyInt int

type IntLike interface {
    ~int  // 匹配 int 和所有底层类型为 int 的类型
}

func Double[T IntLike](v T) T {
    return v * 2
}

// 可以使用
var x MyInt = 5
Double(x)  // OK，MyInt 的底层类型是 int
```

### 方法约束

约束可以包含方法要求：

```go
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// 组合约束
type OrderedStringer interface {
    ~int | ~string
    String() string
}
```

### constraints 包

标准库 `golang.org/x/exp/constraints` 提供常用约束：

```go
import "golang.org/x/exp/constraints"

// Ordered: 支持 < > <= >= 的类型
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Integer: 所有整数类型
// Float: 所有浮点类型
// Complex: 所有复数类型
// Signed: 有符号整数
// Unsigned: 无符号整数
```

---

## GCShape 实现原理

Go 泛型采用 **GCShape Stenciling** 混合实现方案，这是理解 Go 泛型性能特征的关键。

### 什么是 GC Shape？

**GC Shape (GC 形状)** 指类型在垃圾回收器和内存分配器视角下的表现，由以下因素决定：

1. **大小 (Size)**：类型占用的字节数
2. **对齐 (Alignment)**：内存对齐要求
3. **指针位图 (Pointer Bitmap)**：哪些位置包含指针

```
┌─────────────────────────────────────────────────────────────┐
│                      GC Shape 决定因素                        │
├─────────────────────────────────────────────────────────────┤
│  Type        │  Size  │  Align  │  Pointers  │  GC Shape    │
├─────────────────────────────────────────────────────────────┤
│  int64       │  8     │  8      │  无        │  Shape A     │
│  uint64      │  8     │  8      │  无        │  Shape A     │
│  float64     │  8     │  8      │  无        │  Shape A     │
│  *int        │  8     │  8      │  位置0     │  Shape B     │
│  *string     │  8     │  8      │  位置0     │  Shape B     │
│  string      │  16    │  8      │  位置0     │  Shape C     │
│  []int       │  24    │  8      │  位置0     │  Shape D     │
└─────────────────────────────────────────────────────────────┘
```

### Stenciling vs Dictionary-Passing

泛型实现有两种主要策略：

```
┌─────────────────────────────────────────────────────────────┐
│              泛型实现策略对比                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 完全单态化 (Full Stenciling) - C++ 模板                  │
│     ┌─────────────────────────────────────────────────┐    │
│     │  func Max[T](a, b T) T                          │    │
│     │       ↓                                         │    │
│     │  func Max_int(a, b int) int      // 生成        │    │
│     │  func Max_string(a, b string) string // 生成    │    │
│     │  func Max_float64(a, b float64) float64 // 生成 │    │
│     └─────────────────────────────────────────────────┘    │
│     优点: 运行时零开销                                       │
│     缺点: 代码膨胀、编译慢                                   │
│                                                             │
│  2. 字典传递 (Dictionary-Passing) - 纯运行时                 │
│     ┌─────────────────────────────────────────────────┐    │
│     │  func Max(dict *TypeDict, a, b interface{}) interface{}│
│     │  // 只生成一份代码，运行时查字典                   │    │
│     └─────────────────────────────────────────────────┘    │
│     优点: 代码紧凑、编译快                                   │
│     缺点: 运行时开销、无法内联                               │
│                                                             │
│  3. GCShape Stenciling (Go 的混合方案)                       │
│     ┌─────────────────────────────────────────────────┐    │
│     │  按 GC Shape 分组生成代码 + 字典传递类型信息        │    │
│     │                                                 │    │
│     │  Max[int64], Max[uint64], Max[float64]          │    │
│     │       ↓ 共享同一份代码（Shape A）                 │    │
│     │  func Max_shapeA(dict, a, b) // 一份代码         │    │
│     └─────────────────────────────────────────────────┘    │
│     平衡: 适度代码生成 + 适度运行时开销                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 字典结构

每个泛型实例化会生成一个字典，包含：

```go
// 编译器生成的字典结构（概念示意）
type genericDict struct {
    // 类型参数的 runtime._type
    typeParams []*_type
    
    // 派生类型（如 []T, map[K]V 等）
    derivedTypes []*_type
    
    // 子字典（嵌套泛型调用）
    subDicts []*genericDict
    
    // 方法表（约束中的方法）
    methods []func()
}
```

字典在编译时计算，存储在只读数据段，运行时通过指针传递。

### 代码生成示例

```go
// 源代码
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// 调用
Max(1, 2)           // int
Max(1.0, 2.0)       // float64
Max("a", "b")       // string
```

编译器生成（概念示意）：

```
┌─────────────────────────────────────────────────────────────┐
│                    编译器代码生成                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Shape: 8字节非指针 (int64, uint64, float64 共享)            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  func Max_shape8nonptr(dict *Dict, a, b uint64) uint64│   │
│  │  {                                                   │   │
│  │      // 通过 dict 获取比较方法                        │   │
│  │      cmp := dict.methods[0]                          │   │
│  │      if cmp(a, b) > 0 { return a }                   │   │
│  │      return b                                        │   │
│  │  }                                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Shape: string (16字节，含指针)                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  func Max_shapeString(dict *Dict, a, b string) string│   │
│  │  {                                                   │   │
│  │      if a > b { return a }                           │   │
│  │      return b                                        │   │
│  │  }                                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 与其他语言的对比

### Go vs C++ 模板

| 特性 | Go 泛型 | C++ 模板 |
|------|---------|----------|
| 实现方式 | GCShape + 字典 | 完全单态化 |
| 代码膨胀 | 中等 | 严重 |
| 编译速度 | 较快 | 慢 |
| 运行时开销 | 少量（字典传递） | 无 |
| 类型检查 | 编译时（约束） | 实例化时 |
| 错误信息 | 清晰 | 复杂 |

### Go vs Java 泛型

| 特性 | Go 泛型 | Java 泛型 |
|------|---------|-----------|
| 实现方式 | GCShape + 字典 | 类型擦除 |
| 运行时类型信息 | 保留（字典） | 丢失 |
| 原始类型支持 | 支持 | 不支持（需装箱） |
| 代码生成 | 按 Shape 生成 | 无 |
| 反射支持 | 完整 | 受限 |

### Go vs Rust 泛型

| 特性 | Go 泛型 | Rust 泛型 |
|------|---------|-----------|
| 实现方式 | GCShape + 字典 | 完全单态化 |
| 零成本抽象 | 否 | 是 |
| trait/约束 | 接口约束 | trait bounds |
| 编译速度 | 较快 | 慢 |
| 二进制大小 | 中等 | 较大 |

---

## 最佳实践与性能影响

### 何时使用泛型

**适合使用泛型的场景：**

```go
// 1. 容器类型
type Set[T comparable] map[T]struct{}

// 2. 通用算法
func Filter[T any](slice []T, predicate func(T) bool) []T

// 3. 数据结构
type LinkedList[T any] struct {
    head *Node[T]
}

// 4. 函数式工具
func Reduce[T, U any](slice []T, initial U, f func(U, T) U) U
```

**不适合使用泛型的场景：**

```go
// 1. 只有一两种类型 - 直接写具体类型
func SumInts(nums []int) int  // 比泛型更清晰

// 2. 需要类型特定行为 - 使用接口
type Handler interface {
    Handle(ctx context.Context) error
}

// 3. 反射已经足够 - 如 JSON 序列化
```

### 性能考量

**1. 字典传递开销**

```go
// 泛型版本 - 有字典传递开销
func Sum[T Number](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}

// 具体类型版本 - 无开销，可能被内联
func SumInt64(nums []int64) int64 {
    var sum int64
    for _, n := range nums {
        sum += n
    }
    return sum
}
```

**2. 内联限制**

泛型函数的内联受到限制，因为需要传递字典。对于热点路径，考虑使用具体类型。

**3. 接口值装箱**

避免在泛型中使用 `interface{}` 参数，这会导致装箱开销：

```go
// 不好 - 双重开销
func Bad[T any](v interface{}) T

// 好 - 只有泛型开销
func Good[T any](v T) T
```

### 编码规范

```go
// 1. 类型参数命名
// 单个类型参数用 T
func Clone[T any](v T) T

// 多个类型参数用描述性名称
func Map[Input, Output any](slice []Input, f func(Input) Output) []Output

// Key/Value 场景
func Keys[K comparable, V any](m map[K]V) []K

// 2. 约束定义放在使用处附近
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](nums []T) T { ... }

// 3. 避免过度泛化
// 不好 - 过度泛化
func Process[T any, U any, V any](a T, b U, c V) (T, U, V)

// 好 - 只泛化必要的部分
func Process[T any](a T, b int, c string) T
```

---

## Interview Cheatsheet

### 常见面试问题

**Q1: Go 泛型是如何实现的？**

> Go 使用 GCShape Stenciling 混合方案。按类型的 GC 形状（大小、对齐、指针位置）分组，相同形状的类型共享一份生成代码，通过字典传递具体类型信息。这是单态化和字典传递的折中方案。

**Q2: `any` 和 `comparable` 有什么区别？**

> - `any` 是 `interface{}` 的别名，允许任意类型，但不能进行任何操作
> - `comparable` 约束类型必须支持 `==` 和 `!=`，可用于 map key 和相等性比较

**Q3: Go 泛型和 C++ 模板有什么区别？**

> - C++ 使用完全单态化，每个类型实例生成独立代码，运行时零开销但代码膨胀严重
> - Go 使用 GCShape 分组 + 字典传递，减少代码膨胀但有少量运行时开销
> - Go 在编译时通过约束检查类型，C++ 在模板实例化时检查

**Q4: `~T` 语法是什么意思？**

> `~T` 是近似约束，匹配底层类型为 T 的所有类型。例如 `~int` 匹配 `int` 和 `type MyInt int`。

**Q5: 泛型函数能被内联吗？**

> 受限。由于需要字典传递，泛型函数的内联比普通函数更困难。对于性能关键路径，可能需要使用具体类型版本。

### 关键概念速记

```
┌─────────────────────────────────────────────────────────────┐
│                    Go 泛型核心概念                           │
├─────────────────────────────────────────────────────────────┤
│  语法: func Name[T Constraint](params) ReturnType           │
│                                                             │
│  内置约束:                                                   │
│    - any: interface{} 别名，任意类型                         │
│    - comparable: 支持 == != 的类型                          │
│                                                             │
│  自定义约束:                                                 │
│    - 类型集合: int | int64 | string                         │
│    - 近似类型: ~int (底层类型为 int)                         │
│    - 方法要求: interface { Method() }                       │
│                                                             │
│  实现原理:                                                   │
│    - GCShape: 按 (size, align, pointers) 分组               │
│    - 字典: 存储 _type 和派生类型信息                         │
│    - 编译时计算，运行时指针传递                              │
│                                                             │
│  性能影响:                                                   │
│    - 字典传递有少量开销                                      │
│    - 内联受限                                                │
│    - 比 interface{} 更高效（避免装箱）                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 参考资料

- [Go 泛型提案](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md)
- [GCShape Stenciling 设计文档](https://go.googlesource.com/proposal/+/refs/heads/master/design/generics-implementation-gcshape.md)
- [Go 官方泛型教程](https://go.dev/doc/tutorial/generics)
- [constraints 包文档](https://pkg.go.dev/golang.org/x/exp/constraints)
