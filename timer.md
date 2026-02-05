# 定时器

## 简单用法

什么 Date，Parse，ParseInLocation，就不说了。主要关注下面几个函数：

### ticker 相关

```go
func Tick(d Duration) <-chan Time
func NewTicker(d Duration) *Ticker
func (t *Ticker) Stop()
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for t := range time.Tick(time.Second * 2) {
        fmt.Println(t, "hello world")
    }
}
```

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ticker := time.NewTicker(time.Second * 2)
    for {
        select {
        case t := <-ticker.C:
            fmt.Println(t, "hello world")
        }
    }
}
```

需要注意的是，ticker 在不使用时，应该手动 stop，如果不 stop 可能会造成 timer 泄露，比如下面这样的代码:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for {
        select {
        case t := <-time.Tick(time.Second * 2):
            fmt.Println(t, "hello world")
        }
    }
}
```

泄露之后会在时间堆中累积越来越多的 timer 对象，从而发生更麻烦的问题。

### timer 相关

```go
func After(d Duration) <-chan Time
func NewTimer(d Duration) *Timer
func (t *Timer) Reset(d Duration) bool
func (t *Timer) Stop() bool
```

time.After 一般用来控制某些耗时较长的行为，在超时后不再等待，以使程序行为可预期。如果不做超时取消释放资源，则可能因为依赖方响应缓慢而导致本地资源堆积，例如 fd，连接数，内存占用等等。从而导致服务宕机。

这里用阻塞 channel 读取会永远阻塞的特性来模拟较长时间的行为，在超时后会从 select 中跳出。

```go
package main

import "time"

func main() {
    var ch chan int
    select {
    case <-time.After(time.Second):
        println("time out, and end")
    case <-ch:
    }
}
```

time.After 和 time.Tick 不同，是一次性触发的，触发后 timer 本身会从时间堆中删除。所以一般情况下直接用 `<-time.After` 是没有问题的，不过在 for 循环的时候要注意:

```go
package main

import "time"

func main() {
    var ch = make(chan int)
    go func() {
        for {
            ch <- 1
        }
    }()

    for {
        select {
        case <-time.After(time.Second):
            println("time out, and end")
        case <-ch:
        }
    }
}
```

上面的代码，<-ch 这个 case 每次执行的时间都很短，但每次进入 select，`time.After` 都会分配一个新的 timer。因此会在短时间内创建大量的无用 timer，虽然没用的 timer 在触发后会消失，但这种写法会造成无意义的 cpu 资源浪费。正确的写法应该对 timer 进行重用，如下:

```go
package main

import "time"

func main() {
    var ch = make(chan int)
    go func() {
        for {
            ch <- 1
        }
    }()

    timer := time.NewTimer(time.Second)
    for {
        timer.Reset(time.Second)
        select {
        case <-timer.C:
            println("time out, and end")
        case <-ch:
        }
    }
}
```

和 Ticker 一样，如果之前的 timer 没用了，可以手动 Stop 以使该 timer 从时间堆中移除。

## 源码分析

### 数据结构

```
                                          ┌────────┐                                                                                                     
                                          │ timers │                                                                                                     
                                          ├────┬───┴┬────┬────┬────┬────┬────┬───────────────────────┬────┐                                              
                                          │    │    │    │    │    │    │    │                       │    │                                              
                                          │  0 │  1 │  2 │  3 │  4 │  5 │  6 │            ...        │ 63 │                                              
                                          └────┴────┴────┴────┴────┴────┴────┴───────────────────────┴────┘                                              
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │                                                         │                                                
                                              │ │                                          │            │    │                                          │
                                              │ │          ┌────────────────────┐          │            │    │          ┌────────────────────┐          │
                                              │ │────────▶ │   cacheline size   │ ◀────────┤            │    │────────▶ │   cacheline size   │ ◀────────┤
                                              │ │          └────────────────────┘          │            │    │          └────────────────────┘          │
                                              │ ├─────────────┬─────────────────┬──────────┤            │    ├─────────────┬─────────────────┬──────────┤
                                              └▶│timersBucket │                 │          │            └───▶│timersBucket │                 │          │
                                                ├─────────────┴─────────────────┤   pad    │                 ├─────────────┴─────────────────┤   pad    │
                                                │          lock mutex           │          │                 │          lock mutex           │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │             gp *g             │          │                 │             gp *g             │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │         created bool          │          │                 │         created bool          │          │
                                                ├───────────────────────────────┤          │    ........     ├───────────────────────────────┤          │
                                                │         sleeping bool         │          │                 │         sleeping bool         │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │       rescheduling bool       │          │                 │       rescheduling bool       │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │       sleepUntil int64        │          │                 │       sleepUntil int64        │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │         waitnote note         │          │                 │         waitnote note         │          │
                                                ├───────────────────────────────┤          │                 ├───────────────────────────────┤          │
                                                │          t []*timer           │          │                 │          t []*timer           │          │
                                                └───────────────────────────────┴──────────┘                 └───────────────────────────────┴──────────┘
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                ▼                                                                                        
                                                              ┌───┐                                                                                      
                                                              │ 0 │                                                                                      
                                                              └───┘                                                                                      
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                │                                                                                        
                                                                ▼                                                                                        
                                                        ┌───┬───┬───┬───┐                                                                                
                                                        │ 1 │ 2 │ 3 │ 4 │                                                                                
                                                        └─┬─┴─┬─┴─┬─┴─┬─┘                                                                                
                        ┌─────────────────────────────────┘   │   │   └──────────────────────────────────────────┐                                       
                        │                              ┌──────┘   └────────┐                                     │                                       
                        ▼                              │                   │                                     ▼                                       
                ┌───┬───┬───┬───┐                      ▼                   ▼                             ┌───┬───┬───┬───┐                               
                │   │   │   │   │              ┌───┬───┬───┬───┐   ┌───┬───┬───┬───┐                     │   │   │   │   │                               
                └─┬─┴─┬─┴───┴───┘              │   │   │   │   │   │   │   │   │   │                     └───┴───┴─┬─┴─┬─┘                               
                  │   │                        └───┴───┴───┴───┘   └───┴───┴───┴───┘                               │   │                                 
        ┌─────────┘   └─────────┐                                                                        ┌─────────┘   └─────┐                           
        │                       │                                                                        │                   │                           
        │                       │                                                                        │                   │                           
        ▼                       ▼                                                                        ▼                   ▼                           
┌───┬───┬───┬───┐       ┌───┬───┬───┬───┐                                                        ┌───┬───┬───┬───┐   ┌───┬───┬───┬───┐                   
│   │   │   │   │       │   │   │   │   │                  .................                     │   │   │   │   │   │   │   │   │   │                   
└───┴───┴───┴───┘       └───┴───┴───┴───┘                                                        └───┴───┴───┴───┘   └───┴───┴───┴───┘                   
```

下面的结论都可以结合上面的图来看。

在 `runtime/time.go` 中定义了 timers 数组:

```go
var timers [timersLen]struct {
    timersBucket
    pad [sys.CacheLineSize - unsafe.Sizeof(timersBucket{})%sys.CacheLineSize]byte
}
```

在 Go 的早期实现中是全局一个 timer 的，但操作全局的 timer 堆要加锁，所以多核心会暴露出因为争锁而性能低下的问题。从某个版本(嗯，我也不知道哪个，大概是 1.10)起，Go 的 timers 修改成了这种多个时间堆的实现方式，目前在 runtime 里写死为 64:

```go
const timersLen = 64
```

嗯，官方表示如果和 GOMAXPROCS 相等的话，那么会在 procresize 的时候重新分配和修改这些时间堆。写死成 64 是内存使用和性能上的一个折衷。如果 GOMAXPROCS 比 64 大的话，那么可能多个 P 会公用同一个时间堆。当然，实际场景中我还没有见过 64 核以上的 CPU。

timers 数组的元素是一个匿名 struct，包含 timersBucket 和 pad 两个成员，这个 pad 是为了填充 struct 到 cacheline 的整数倍，以避免在不同的 P 之间发生 false sharing。在多核心的编程场景中较为常见。timerBucket 的结构:

```go
//go:notinheap
type timersBucket struct {
    lock         mutex
    gp           *g
    created      bool
    sleeping     bool
    rescheduling bool
    sleepUntil   int64
    waitnote     note
    t            []*timer
}
```

其中的 t 就是我们的时间堆了，不过这个和我们传统的 heap 结构稍微有所不同，是分四个叉的，这种设计第一次见。这里的 timersBucket 还有个特殊的注释 `go:notinheap`，官方的说明:

> go:notinheap applies to type declarations. It indicates that a type must never be allocated from the GC'd heap. Specifically, pointers to this type must always fail the runtime.inheap check. The type may be used for global variables, for stack variables, or for objects in unmanaged memory (e.g., allocated with sysAlloc, persistentalloc, fixalloc, or from a manually-managed span). Specifically:
> 1. new(T), make([]T), append([]T, ...) and implicit heap allocation of T are disallowed. (Though implicit allocations are disallowed in the runtime anyway.)
> 2. A pointer to a regular type (other than unsafe.Pointer) cannot be converted to a pointer to a go:notinheap type, even if they have the same underlying type.
> 3. Any type that contains a go:notinheap type is itself go:notinheap. Structs and arrays are go:notinheap if their elements are. Maps and channels of go:notinheap types are disallowed. To keep things explicit, any type declaration where the type is implicitly go:notinheap must be explicitly marked go:notinheap as well.
> 4. Write barriers on pointers to go:notinheap types can be omitted.
> The last point is the real benefit of go:notinheap. The runtime uses it for low-level internal structures to avoid memory barriers in the scheduler and the memory allocator where they are illegal or simply inefficient. This mechanism is reasonably safe and does not compromise the readability of the runtime.

嗯，了解一下就行了，在用户代码中基本不会用得到。

### 四叉小顶堆性质

四叉堆高度上比二叉堆要矮一些。一个节点的所有(最多有4个)孩子节点都比这个节点要大。一个节点的(只有一个)父节点一定比当前节点小。下面是填好值之后的一个典型的四叉堆:

```
                                                             ┌─────┐                                                         
                                                             │     │                                                         
                                                             │  0  │                                                         
                                                             └─────┘                                                         
                                                                │                                                            
                                                                │                                                            
                                                                │                                                            
                                                                ▼                                                            
                                                    ┌─────┬─────┬─────┬─────┐                                                
                                                    │     │     │     │     │                                                
                                                    │  3  │  2  │  2  │  10 │                                                
                                                    └─────┴─────┴─────┴─────┘                                                
                                                       │     │     │     │                                                   
                                                       │     │     │     │                                                   
                    ┌──────────┐                       │     │     │     │                                                   
   ┌────────────────┤  4*i+1   ├───────────────────────┘     │     │     └─────────────────────────────┐                     
   │                └──────────┘         ┌───────────────────┘     └───┐                               │                     
   │                                     │                             │                               │                     
   │                                     │                             │                               │                     
   ▼                                     │                             │                               ▼                     
┌─────┬─────┬─────┬─────┐                │                             │                            ┌─────┬─────┬─────┬─────┐
│     │     │     │     │                ▼                             ▼                            │     │     │     │     │
│  20 │  4  │  5  │  13 │             ┌─────┬─────┬─────┬─────┐     ┌─────┬─────┬─────┬─────┐       │ 99  │ 13  │ 11  │  12 │
└─────┴─────┴─────┴─────┘             │     │     │     │     │     │     │     │     │     │       └─────┴─────┴─────┴─────┘
                                      │ 12  │ 14  │ 15  │  16 │     │ 3   │ 10  │ 3   │  3  │                                
                                      └─────┴─────┴─────┴─────┘     └─────┴─────┴─────┴─────┘                                
```

和二叉堆一样，对于一个节点的要求只有和其父节点以及子节点之间的大小关系。相邻节点之间没有任何关系。

### 时间堆插入

```go
// 分配 timerBucket
// 加锁，添加 timer 进时间堆
func addtimer(t *timer) {
    tb := t.assignBucket()
    lock(&tb.lock)
    tb.addtimerLocked(t)
    unlock(&tb.lock)
}

// 太简单了，就是用 g.m.p 的 id 模 64
// 然后分配对应的 timerBucket
func (t *timer) assignBucket() *timersBucket {
    id := uint8(getg().m.p.ptr().id) % timersLen
    t.tb = &timers[id].timersBucket
    return t.tb
}

// 向时间堆中添加一个 timer，如果时间堆第一次被初始化或者当前的 timer 比之前所有的 timers 都要早，那么就启动(首次初始化)或唤醒(最早的 timer) timerproc
// 函数内假设外部已经对 timers 数组加锁了
func (tb *timersBucket) addtimerLocked(t *timer) {
    // when 必须大于 0，否则会在计算 delta 的时候溢出并导致其它的 runtime timer 永远没法过期
    if t.when < 0 {
        t.when = 1<<63 - 1
    }
    t.i = len(tb.t)
    tb.t = append(tb.t, t)
    siftupTimer(tb.t, t.i)
    if t.i == 0 {
        // 新插入的 timer 比之前所有的都要早
        // 向上调整堆
        if tb.sleeping {
            // 修改 timerBucket 的 sleep 状态
            tb.sleeping = false
            // 唤醒 timerproc
            // 使 timerproc 中的 for 循环不再阻塞在 notesleepg 上
            notewakeup(&tb.waitnote)
        }
        // 同一个 P 上的所有 timers 如果
        // 都在 timerproc 中被弹出了
        // 该 rescheduling 会被标记为 true
        // 并且启动 timerproc 的 goroutine 会被 goparkunlock
        if tb.rescheduling {
            // 该标记会在这里和 timejumpLocked 中被设置为 false
            tb.rescheduling = false
            goready(tb.gp, 0)
        }
    }
    // 如果 timerBucket 是第一次创建，需要启动一个 goroutine
    // 来循环弹出时间堆，内部会根据需要最早触发的 timer
    // 进行相应时间的 sleep
    if !tb.created {
        tb.created = true
        go timerproc(tb)
    }
}
```

插入 timer 到堆中的时候的逻辑是先追加到数组末尾(append)，然后向上 adjust(siftup) heap 以重新恢复四叉小顶堆性质。

### 时间堆删除

```go
// 从堆中删除 timer t
// 如果 timerproc 提前被唤醒也没所谓
func deltimer(t *timer) bool {
    if t.tb == nil {
        // t.tb can be nil if the user created a timer
        // directly, without invoking startTimer e.g
        //    time.Ticker{C: c}
        // In this case, return early without any deletion.
        // See Issue 21874.
        return false
    }

    tb := t.tb

    lock(&tb.lock)
    // t may not be registered anymore and may have
    // a bogus i (typically 0, if generated by Go).
    // Verify it before proceeding.
    i := t.i
    last := len(tb.t) - 1
    if i < 0 || i > last || tb.t[i] != t {
        unlock(&tb.lock)
        return false
    }
    // 把 timer[i] 替换为 timer[last]
    if i != last {
        tb.t[i] = tb.t[last]
        tb.t[i].i = i
    }
    // 删除 timer[last]，并缩小 slice
    tb.t[last] = nil
    tb.t = tb.t[:last]

    // 判断是不是删的最后一个
    // 如果不是的话，需要重新调整堆
    if i != last {
        // 最后一个节点当前来的分叉可能并不是它那个分叉
        // 所以向上走或者向下走都是有可能的
        // 即使是二叉堆，也是有这种可能的
        siftupTimer(tb.t, i)
        siftdownTimer(tb.t, i)
    }
    unlock(&tb.lock)
    return true
}
```

### timer 触发

```go
// timerproc 负责处理时间驱动的事件
// 在堆中的下一个事件需要触发之前，会一直保持 sleep 状态
// 如果 addtimer 插入了一个更早的事件，会提前唤醒 timerproc
func timerproc(tb *timersBucket) {
    tb.gp = getg()
    for {
        // timerBucket 的局部大锁
        lock(&tb.lock)
        // 被唤醒，所以修改 sleeping 状态为 false
        tb.sleeping = false
        // 计时
        now := nanotime()
        delta := int64(-1)
        // 在处理完到期的 timer 之前，一直循环
        for {
            // 如果 timer 已经都弹出了
            // 那么不用循环了，跳出后接着睡觉
            if len(tb.t) == 0 {
                delta = -1
                break
            }
            // 取小顶堆顶部元素
            // 即最近会被触发的 timer
            t := tb.t[0]
            delta = t.when - now // 还差多长时间才需要触发最近的 timer
            if delta > 0 {
                // 大于 0 说明这个 timer 还没到需要触发的时间
                // 跳出循环去睡觉
                break
            }
            if t.period > 0 {
                // 这个 timer 还会留在堆里
                // 不过要调整它的下次触发时间
                t.when += t.period * (1 + -delta/t.period)
                siftdownTimer(tb.t, 0)
            } else {
                // 从堆中移除这个 timer
                // 用最后一个 timer 覆盖第 0 个 timer
                // 然后向下调整堆
                last := len(tb.t) - 1
                if last > 0 {
                    tb.t[0] = tb.t[last]
                    tb.t[0].i = 0
                }
                tb.t[last] = nil
                tb.t = tb.t[:last]
                if last > 0 {
                    siftdownTimer(tb.t, 0)
                }
                t.i = -1 // 标记 timer 在堆中的位置已经没有了
            }
            // timer 触发时需要调用的函数
            f := t.f
            arg := t.arg
            seq := t.seq
            unlock(&tb.lock)
            // 调用需触发的函数
            f(arg, seq)
            // 把锁加回来，如果下次 break 了内层的 for 循环
            // 能保证 timeBucket 是被锁住的
            // 然后在下面的 goparkunlock 中被解锁
            lock(&tb.lock)
        }
        if delta < 0 || faketime > 0 {
            // 说明时间堆里已经没有 timer 了
            // 让 goroutine 挂起，去睡觉
            tb.rescheduling = true
            goparkunlock(&tb.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
            continue
        }
        // 说明堆里至少还有一个以上的 timer
        // 睡到最近的 timer 时间
        tb.sleeping = true
        tb.sleepUntil = now + delta
        noteclear(&tb.waitnote)
        unlock(&tb.lock)
        // 内部是 futex sleep
        // 时间睡到了会自动醒
        // 或者 addtimer 的时候，发现新的 timer 更早，会提前唤醒
        notetsleepg(&tb.waitnote, delta)
    }
}
```

可以留意一下这里的 period，有 period 的 timer 会从 when 开始，每隔 period 段时间，就再次触发。

```
                                                                                
                                          when+period                           
                                                          when+period*3         
                                             │                                  
                                             │               │                  
                                             │               │                  
                                             │  when+period*2│                  
                                  when       │               │                  
                                             │       │       │                  
                                     │       │       │       │        .....     
                                     │       │       │       │                  
┌───────────┐                        │       │       │       │                  
│ timeline  ├────────────────────────┼───────┼───────┼───────┼─────────────────▷
└───────────┘                        │       │       │       │                  
                                     ▼       ▼       ▼       ▼                  
                                                                                
                                  trigger          trigger                      
                                          trigger          trigger              

```

### 时间堆调整

之前的代码也看到了，时间堆调整有向上调整和向下调整两种调整方式。

#### 向上调整

```go
func siftupTimer(t []*timer, i int) {
    // 先暂存当前刚插入到数组尾部的节点
    when := t[i].when
    tmp := t[i]

    // 从当前插入节点的父节点开始
    // 如果最新插入的那个节点的触发时间要比父节点的触发时间更早
    // 那么就把这个父节点下移
    for i > 0 {
        p := (i - 1) / 4 // parent
        if when >= t[p].when {
            break
        }
        t[i] = t[p]
        t[i].i = i
        i = p
    }

    // 如果发生过移动，用最新插入的节点
    // 覆盖掉最后一个下移的父节点
    if tmp != t[i] {
        t[i] = tmp
        t[i].i = i
    }
}
```

#### 向下调整

```go
func siftdownTimer(t []*timer, i int) {
    n := len(t)
    when := t[i].when
    tmp := t[i]
    for {
        c := i*4 + 1 // 最左孩子节点
        c3 := c + 2  // 第三个孩子节点
        if c >= n {
            break
        }
        w := t[c].when
        if c+1 < n && t[c+1].when < w {
            w = t[c+1].when
            c++
        }
        if c3 < n {
            w3 := t[c3].when
            if c3+1 < n && t[c3+1].when < w3 {
                w3 = t[c3+1].when
                c3++
            }
            if w3 < w {
                w = w3
                c = c3
            }
        }
        if w >= when {
            break
        }
        t[i] = t[c]
        t[i].i = i
        i = c
    }
    if tmp != t[i] {
        t[i] = tmp
        t[i].i = i
    }
}
```

这段代码实在是称不上优雅，其实就是在所有孩子节点中先找出最小的那一个，如果最小的比当前要下移的节点还要大，那么就 break。反之，则将最小的节点上移，然后再判断这个最小节点的 4 个子节点是否都比要下移的节点大。以此类推。用图来模拟一下这个过程:

```
                         │            ┌───┐                                       
                         │            │ 5 │                                       
                         │            └───┘                                       
                         │              │                                         
                         │        ┌─────┘                                         
                         │        ▼                                               
                         │      ┌───┬───┳━━━┳───┐                                 
                         │      │ 7 │ 3 │ 2 ┃ 6 │                                 
                         │      └───┴───┻━━━┻───┘                                 
┌───────────────────┐    │                │                                       
│   siftdownTimer   │    │                └──────────┐                            
└───────────────────┘    │                           ▼                            
     .─────────.         │                         ┌───┬───┬───┳━━━┓              
    (  before   )        │                         │ 4 │ 5 │ 9 │ 3 ┃              
     `─────────'         │                         └───┴───┴───┻━━━┛              
                         │                                       │                
                         │                                       └─────────────┐  
                         │                                                     ▼  
                         │                                       ┌───┬───┬───┳━━━┓
                         │                                       │ 6 │ 6 │ 6 │ 4 ┃
                         │                                       └───┴───┴───┻━━━┛
                         │                                                        
                         ▼                                                        
                                                                                  
                                                                                  
                                                                                  
                         │            ┌───┐                                       
                         │            │ 2 │                                       
                         │            └───┘                                       
                         │              │                                         
                         │        ┌─────┘                                         
                         │        ▼                                               
                         │      ┌───┬───┳━━━┳───┐                                 
                         │      │ 7 │ 3 │ 3 ┃ 6 │                                 
                         │      └───┴───┻━━━┻───┘                                 
┌───────────────────┐    │                │                                       
│   siftdownTimer   │    │                └──────────┐                            
└───────────────────┘    │                           ▼                            
    .─────────.          │                         ┌───┬───┬───┳━━━┓              
   (   after   )         │                         │ 4 │ 5 │ 9 │ 4 ┃              
    `─────────'          │                         └───┴───┴───┻━━━┛              
                         │                                       │                
                         │                                       └─────────────┐  
                         │                                                     ▼  
                         │                                       ┌───┬───┬───┳━━━┓
                         │                                       │ 6 │ 6 │ 6 │ 5 ┃
                         │                                       └───┴───┴───┻━━━┛
                         │                                                        
                         ▼                                                        
```

### 流程

#### timer.After 流程

```go
func After(d Duration) <-chan Time {
    return NewTimer(d).C
}

// NewTimer creates a new Timer that will send
// the current time on its channel after at least duration d.
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime,
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}

func startTimer(*runtimeTimer)
```

这个 startTimer 的实现是在 `runtime/time.go` 里:

```go
// startTimer adds t to the timer heap.
// 把 t 添加到 timer 堆
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
    addtimer(t)
}
```

addtimer 后面的流程之前已经看过了。

#### timer.Tick 流程

```go
func Tick(d Duration) <-chan Time {
    if d <= 0 {
        return nil
    }
    return NewTicker(d).C
}
```

```go
// NewTicker 会返回一个 Ticker 对象，其 channel 每隔 period 时间
// 会收到一个时间值
// 如果 receiver 接收慢了，Ticker 会把不需要的 tick drop 掉
// d 必须比 0 大，否则 panic
// Stop ticker 才能释放相关的资源
func NewTicker(d Duration) *Ticker {
    if d <= 0 {
        panic(errors.New("non-positive interval for NewTicker"))
    }
    c := make(chan Time, 1)
    t := &Ticker{
        C: c,
        r: runtimeTimer{
            when:   when(d),
            period: int64(d),
            f:      sendTime,
            arg:    c,
        },
    }
    startTimer(&t.r)
    return t
}
```

可以看到， Ticker 和 Timer 的 r 成员就只差在 period 这一个字段上，每隔一个 period 就往 channel 里发数据的就是 Ticker，而 fire and disappear 的就是 Timer。

#### Stop 流程

```go
func (t *Ticker) Stop() {
    stopTimer(&t.r)
}

func (t *Timer) Stop() bool {
    if t.r.f == nil {
        panic("time: Stop called on uninitialized Timer")
    }
    return stopTimer(&t.r)
}
```

Timer 和 Ticker 都是调用的 stopTimer。

```go
func stopTimer(t *timer) bool {
    return deltimer(t)
}
```

deltimer 在上面也看到过了。

#### Reset 流程

```go
func (t *Timer) Reset(d Duration) bool {
    if t.r.f == nil {
        panic("time: Reset called on uninitialized Timer")
    }
    w := when(d)
    active := stopTimer(&t.r)
    t.r.when = w
    startTimer(&t.r)
    return active
}
```

都是见过的函数，没啥特别的。

## 最后

本篇内容主要是讲 Go 的定时器实现，工业界的定时器实现并不只有一种。如果你还想知道其它系统，比如 nginx 里是怎么实现定时器的，可以参考[这一篇](https://www.jianshu.com/p/427dfe8ad3c0)。

---

## Timer 实现演进 (Go 1.14 - Go 1.23)

> 本节基于 **Go 1.23.11** 版本，介绍 Timer 从 Go 1.14 到最新版本的实现变化。

### TL;DR - 面试速查

- **Go 1.14**: 从全局堆改为 P 本地四叉堆，大幅减少锁竞争
- **Go 1.23**: Timer channel 变为无缓冲，未停止的 timer 可被 GC 回收
- Timer 使用四叉堆（而非二叉堆）以平衡插入/删除性能

---

### Timer 架构演进

```
┌─────────────────────────────────────────────────────────────┐
│                    Timer 架构演进历史                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Go 1.9 之前: 单一全局四叉堆                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Global Timer Heap                   │   │
│  │  ┌─────────────────────────────────────────────┐    │   │
│  │  │              Single 4-ary Heap              │    │   │
│  │  │                    │                        │    │   │
│  │  │  G1 ──┐            │                        │    │   │
│  │  │  G2 ──┼── lock ───▶│                        │    │   │
│  │  │  G3 ──┘            │                        │    │   │
│  │  │                    │                        │    │   │
│  │  │  问题: 严重的锁竞争                          │    │   │
│  │  └─────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Go 1.10 - 1.13: 64 个全局四叉堆                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ┌────┐ ┌────┐ ┌────┐     ┌────┐                    │   │
│  │  │Heap│ │Heap│ │Heap│ ... │Heap│  (64 个)           │   │
│  │  │ 0  │ │ 1  │ │ 2  │     │ 63 │                    │   │
│  │  └────┘ └────┘ └────┘     └────┘                    │   │
│  │                                                     │   │
│  │  改进: 减少竞争，但仍有跨 P 访问问题                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Go 1.14+: P 本地四叉堆 (当前实现)                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │   P0    │  │   P1    │  │   P2    │  ...        │   │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │             │   │
│  │  │ │Heap │ │  │ │Heap │ │  │ │Heap │ │             │   │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  │                                                     │   │
│  │  优势: 无锁竞争，本地访问                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Go 1.14 P 本地堆实现

从 Go 1.14 开始，每个 P 维护自己的 timer 堆：

```go
// runtime/runtime2.go
type p struct {
    // ...
    
    // Timer 相关字段
    timersLock mutex
    timers     []*timer
    
    // 堆中最早触发的 timer 时间
    timer0When uint64
    
    // 需要调整的 timer 数量
    timerModifiedEarliest uint64
    
    // ...
}
```

### Timer 数据结构

```go
// runtime/time.go
type timer struct {
    // 保护 timer 的互斥锁
    mu mutex
    
    // 原子状态位，用于无锁快速路径
    astate atomic.Uint8
    
    // timer 状态
    state uint8
    
    // 是否在堆中
    isChan bool
    
    // 是否被阻塞
    blocked uint32
    
    // 触发时间 (纳秒)
    when int64
    
    // 周期 (对于 Ticker)
    period int64
    
    // 触发时调用的函数
    f func(arg any, seq uintptr, delay int64)
    
    // f 的参数
    arg any
    
    // 序列号，用于区分不同的 timer 实例
    seq uintptr
    
    // 在 P 的 timers 堆中的索引
    heapIndex int
}
```

### 四叉堆 vs 二叉堆

Go 使用四叉堆（4-ary heap）而非二叉堆：

```
┌─────────────────────────────────────────────────────────────┐
│                    四叉堆 vs 二叉堆                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  二叉堆 (Binary Heap):                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              ┌───┐                                  │   │
│  │              │ 1 │                                  │   │
│  │              └───┘                                  │   │
│  │            ┌───┴───┐                                │   │
│  │          ┌───┐   ┌───┐                              │   │
│  │          │ 3 │   │ 2 │                              │   │
│  │          └───┘   └───┘                              │   │
│  │         ┌─┴─┐   ┌─┴─┐                               │   │
│  │       ┌───┐ ┌───┐ ┌───┐ ┌───┐                       │   │
│  │       │ 7 │ │ 4 │ │ 5 │ │ 6 │                       │   │
│  │       └───┘ └───┘ └───┘ └───┘                       │   │
│  │                                                     │   │
│  │  高度: log₂(n)                                      │   │
│  │  每层比较: 1 次                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  四叉堆 (4-ary Heap):                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    ┌───┐                            │   │
│  │                    │ 1 │                            │   │
│  │                    └───┘                            │   │
│  │          ┌─────┬─────┼─────┬─────┐                  │   │
│  │        ┌───┐ ┌───┐ ┌───┐ ┌───┐                      │   │
│  │        │ 3 │ │ 2 │ │ 5 │ │ 4 │                      │   │
│  │        └───┘ └───┘ └───┘ └───┘                      │   │
│  │                                                     │   │
│  │  高度: log₄(n) = log₂(n)/2                          │   │
│  │  每层比较: 最多 4 次                                 │   │
│  │                                                     │   │
│  │  权衡:                                              │   │
│  │  - 树更矮，减少内存访问次数                          │   │
│  │  - 每层比较次数增加，但都在 cache line 内           │   │
│  │  - 对于 timer 场景，插入/删除更频繁，四叉堆更优     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Go 1.23 Timer Channel 变更

Go 1.23 对 Timer channel 做了重大改变：

```
┌─────────────────────────────────────────────────────────────┐
│                Go 1.23 Timer Channel 变更                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Go 1.22 及之前:                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  c := make(chan Time, 1)  // 缓冲 channel           │   │
│  │                                                     │   │
│  │  问题 1: Reset 后可能读到旧值                        │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  timer := time.NewTimer(1 * time.Hour)      │   │   │
│  │  │  timer.Reset(1 * time.Second)               │   │   │
│  │  │  <-timer.C  // 可能立即返回旧的触发值!       │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  问题 2: 未停止的 timer 无法被 GC                    │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  for {                                      │   │   │
│  │  │      <-time.After(time.Second)  // 泄漏!    │   │   │
│  │  │  }                                          │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Go 1.23+:                                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  c := make(chan Time)  // 无缓冲 channel (同步)     │   │
│  │                                                     │   │
│  │  改进 1: Reset/Stop 后不会有旧值                     │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  timer := time.NewTimer(1 * time.Hour)      │   │   │
│  │  │  timer.Reset(1 * time.Second)               │   │   │
│  │  │  <-timer.C  // 保证是新的触发值              │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                                                     │   │
│  │  改进 2: 未停止的 timer 可被 GC                      │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  for {                                      │   │   │
│  │  │      <-time.After(time.Second)  // 安全!    │   │   │
│  │  │  }                                          │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Go 1.23 Timer 使用建议

```go
// Go 1.23+ 推荐用法

// 1. time.After 现在可以安全地在循环中使用
for {
    select {
    case <-time.After(time.Second):
        // 不再泄漏，未触发的 timer 会被 GC
    case <-done:
        return
    }
}

// 2. Reset 后不需要 drain channel
timer := time.NewTimer(time.Hour)
// ...
timer.Reset(time.Second)  // 不需要先 drain channel
<-timer.C  // 保证是新的触发

// 3. Stop 后不需要 drain channel
timer := time.NewTimer(time.Hour)
if timer.Stop() {
    // 不需要 <-timer.C 来 drain
}
```

### Timer 调度流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Timer 调度流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 添加 Timer                                              │
│     ┌─────────────────────────────────────────────────────┐│
│     │  time.NewTimer(d)                                   ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  addtimer(t)                                        ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  将 t 添加到当前 P 的 timers 堆                      ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  siftupTimer (上浮调整堆)                           ││
│     └─────────────────────────────────────────────────────┘│
│                                                             │
│  2. 检查 Timer (在调度循环中)                               │
│     ┌─────────────────────────────────────────────────────┐│
│     │  schedule()                                         ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  checkTimers(pp, now)                               ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  if timers[0].when <= now:                          ││
│     │      runTimer(timers[0])                            ││
│     └─────────────────────────────────────────────────────┘│
│                                                             │
│  3. 触发 Timer                                              │
│     ┌─────────────────────────────────────────────────────┐│
│     │  runTimer(t)                                        ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  t.f(t.arg, t.seq, delay)  // 执行回调              ││
│     │       │                                             ││
│     │       ▼                                             ││
│     │  if t.period > 0:  // Ticker                        ││
│     │      t.when += t.period                             ││
│     │      siftdownTimer (下沉调整堆)                     ││
│     │  else:  // Timer                                    ││
│     │      deltimer(t)  // 从堆中删除                     ││
│     └─────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Interview Cheatsheet

**Q1: Go 1.14 对 timer 做了什么优化？**

> 将全局 timer 堆改为 P 本地堆。每个 P 维护自己的 timer 四叉堆，消除了跨 P 的锁竞争，大幅提升了高并发场景下的 timer 性能。

**Q2: 为什么使用四叉堆而不是二叉堆？**

> 四叉堆的树高度更低（log₄(n) vs log₂(n)），减少了内存访问次数。虽然每层需要比较更多节点，但这些节点通常在同一 cache line 内，对于 timer 这种频繁插入删除的场景更优。

**Q3: Go 1.23 对 timer channel 做了什么改变？**

> 1. Timer channel 从缓冲（容量1）改为无缓冲（同步）
> 2. Reset/Stop 后不会有旧值残留
> 3. 未停止的 timer 现在可以被 GC 回收

**Q4: time.After 在循环中使用有什么问题？**

> Go 1.22 及之前：每次调用创建新 timer，未触发的 timer 不会被 GC，导致内存泄漏。
> Go 1.23+：未触发的 timer 可被 GC，循环中使用 time.After 是安全的。

---

### 参考资料

- [Go 1.23 Timer Channel Changes](https://go.dev/wiki/Go123Timer)
- [runtime/time.go 源码](https://go.dev/src/runtime/time.go)
- [How Go timers are scheduled](https://sobyte.net/post/2022-03/go-timer)
