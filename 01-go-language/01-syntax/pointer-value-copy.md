# Go 指针、值传递与拷贝

> 这类题表面问指针，实际考 Go 的内存模型、方法接收者、逃逸分析和线上性能坑。核心结论：Go 只有值传递，只是有些值本身保存了指向底层数据的指针。

## 一、一句话总结(背诵版)

> **Go 全部是值传递**——指针只是把"地址"作为值复制一份;**需要改原对象、大 struct、含 Mutex/连接池**用指针接收器,**小值类型、不可变语义**用值接收器,**同一类型方法集统一**避免接口实现混乱。

延伸(被追问时再展开):

> Go 没有"引用传递"概念。slice / map / channel 看似引用,实质是把 **header(内含指向底层数据的指针)**按值复制——所以修改元素影响外部,但 `append` 扩容或 `m = make(...)` 重赋值不会影响外部。指针接收器 vs 值接收器的核心差异是**方法集**:`*T` 的方法集 = 值接收 + 指针接收全部;`T` 的方法集 = 仅值接收。**含 sync.Mutex / sync.WaitGroup / noCopy 的 struct 必须指针接收**,否则 `go vet` 会报 `copylocks` 警告。

---

## 二、使用场景(场景化记忆)

| 场景 | 一句话 | 典型例子 |
| --- | --- | --- |
| **需要修改原对象** | 值接收器只改副本,必须指针 | `func (u *User) SetName(n string)`,Repo 模式更新实体字段 |
| **大 struct 避免拷贝** | 拷贝几 KB 的 struct 在热路径浪费 CPU/带宽 | `Order{Payload [4096]byte}`、`http.Request`、Protobuf 大 message |
| **含状态/同步字段必须指针** | Mutex/WaitGroup/atomic 不可复制 | `sync.Mutex`、`sync.WaitGroup`、`bytes.Buffer`、`sql.DB` |
| **方法集统一(同类型不混用)** | 一个类型的方法要么全值要么全指针 | 不要 `func (u User) GetName()` 和 `func (u *User) SetName()` 混用 |
| **小值类型用值接收器** | 拷贝成本低于指针间接寻址 | `time.Time`、`uuid.UUID`、自定义 `Money` / `Coord` |
| **返回错误时小对象值返回** | 简单 DTO 值返回更清晰,避免 nil 检查 | `func parse() (Config, error)` 而非 `*Config` |

**选指针 / 值的判断顺序**:**改原对象?→ 指针**;**含锁/不可复制资源?→ 指针**;**大于 4 字段 / 64 字节?→ 倾向指针**;**否则用值**。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **for range 取 &v 复用变量** | `append(s, &v)` 后所有元素指向同一对象(末值) | Go 1.22 前 `v` 是循环变量复用,地址固定 | 用 `&list[i]` 或 Go 1.22+ 默认每轮新变量 |
| **值接收器修改不生效** | `func (u User) SetName(n)` 调用后字段没变 | 操作的是副本,函数返回副本就丢了 | 改成指针接收器 `(u *User)` |
| **指针接收器导致接口实现集变化** | `var i I = s`(s 是 T 值,方法在 *T)编译错 | T 方法集不含 `*T` 方法,值不可寻址 | 传 `&s`,或把方法改值接收器 |
| **map 元素不可寻址** | `&m["k"]` 编译错;直接改字段也不行 | map 内部 rehash 会搬迁元素,不允许取地址 | 先取出 → 改 → 写回:`v := m[k]; v.X=1; m[k]=v` |
| **slice 传值"以为不共享"** | 函数里改 `s[0]` 外部跟着变 | header 是副本,底层数组是同一份 | 改元素是预期共享;真要隔离做 `copy()` |
| **返回局部变量地址(其实安全)** | 担心栈帧释放后悬空 | Go 逃逸分析会把它放堆,自动 GC | 放心用 `return &local`,Go 编译器自己处理 |
| **nil 指针解引用 panic** | `var p *T; p.X` 直接 crash | 零值是 nil,没指向任何对象 | 解引用前判 `if p == nil` 或确保已初始化 |

---

## 四、面试常问(简答模板)

**Q1:Go 有没有引用传递?值类型和引用类型怎么分?**
**没有引用传递,只有值传递**。所谓"引用类型"(slice/map/channel/func/interface)只是它们的**值本身内含指针**——传参时复制的是包含指针的 header,所以修改底层数据影响外部,但重新赋值变量本身不影响。严格来说 Go 没有"引用类型"这个术语,官方叫**复合类型**。

**Q2:何时用指针接收器,何时用值接收器?**
**指针接收器**:需要修改对象、struct 较大(>64 字节经验值)、含 sync.Mutex / 连接池等不可复制资源、方法集要统一时。**值接收器**:小不可变值对象(time.Time / uuid.UUID / Money)、不需要修改、想要值语义清晰时。**经验法则**:**一个类型的方法接收器要么全值要么全指针,不要混用**——否则方法集不一致,实现接口时容易踩坑。

**Q3:为什么 slice / map / channel 看起来像引用传递?**
它们的"值"本身就是一个 **header 结构**:slice = `{Data, Len, Cap}`,map = `*hmap`,channel = `*hchan`。传参时**复制 header**,但 header 里的指针指向同一份底层数据,所以**修改元素影响外部,重新赋值变量不影响**。这是"值内含指针"≠"引用传递"。

**Q4:大对象一定要用指针吗?**
**不一定**。指针避免拷贝,但有代价:**对象可能逃逸到堆增加 GC 压力**、**间接访问 cache miss**、**多一次内存解引用**。热路径需要 `go test -bench` + `pprof` 实测,有时小对象值传递反而快(栈上分配 + 寄存器传递)。**经验:>64 字节倾向指针,关键路径以 benchmark 为准**。

**Q5:interface 内部存的是值还是指针?差异在哪?**
**iface = (itab, data)**,data 是 `unsafe.Pointer`。**接口内永远存指针**:**值类型装箱时会拷贝一份到堆**,data 指向堆上副本;**指针类型直接存指针**,无额外拷贝。差异:**值装箱触发堆分配 + GC 压力**;指针装箱不分配但需保证原对象生命周期。**性能敏感场景优先传指针给接口**。

**Q6:函数返回局部变量地址安全吗?**
**安全**。Go 有**逃逸分析**:编译器发现局部变量地址逃出函数作用域,会把它分配到堆上,函数返回后变量仍存活,由 GC 管理。可以 `go build -gcflags="-m"` 查看逃逸结果。**和 C/C++ 完全不同**——C 里返回栈上局部变量地址是悬空指针,Go 编译器替你解决了。

---

## 五、深水区:原理与源码(被追问时看)

> 下面是指针 / 接收器 / 拷贝成本 / 逃逸的原理细节,被追问时使用。

## 六、一句话总结

```text
Go 函数参数永远是值传递。
传 struct 会复制整个 struct。
传 pointer 会复制一个地址值。
传 slice / map / channel 会复制 header，但 header 里指向同一份底层数据。
```

## 七、先把“值传递”讲明白

很多人说 Go 里 slice、map 是引用传递，这个说法不准确。

```go
func change(x int) {
    x = 100
}

func main() {
    n := 1
    change(n)
    fmt.Println(n) // 1
}
```

函数调用时，`n` 的值被复制一份给 `x`。`x` 怎么改，都不会影响外面的 `n`。

指针也是值传递：

```go
func change(p *int) {
    *p = 100
}

func main() {
    n := 1
    change(&n)
    fmt.Println(n) // 100
}
```

这里不是“引用传递”，而是把地址值复制了一份。两个指针值不同变量，但指向同一块内存。

```text
main.n  <----+
             |
p(copy) -----+
```

## 八、slice / map / channel 为什么像引用

### 1. slice

slice 本身是一个三字段 header：

```go
type sliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

调用函数时复制的是 header。

```go
func update(s []int) {
    s[0] = 100
}

func main() {
    a := []int{1, 2, 3}
    update(a)
    fmt.Println(a) // [100 2 3]
}
```

图示：

```text
调用前:
a header -> array[1,2,3]

调用后:
a header ----+
             +-> array[100,2,3]
s header ----+
```

但 append 可能触发扩容，导致坑：

```go
func appendOne(s []int) {
    s = append(s, 4)
}

func main() {
    a := []int{1, 2, 3}
    appendOne(a)
    fmt.Println(a) // [1 2 3]
}
```

原因：`s` 这个 header 是副本，append 后新 header 没有返回给调用方。

正确写法：

```go
func appendOne(s []int) []int {
    return append(s, 4)
}
```

### 2. map

map 变量底层是指向 runtime hash 表的指针。

```go
func update(m map[string]int) {
    m["a"] = 100
}
```

复制的是 map header，但底层 hash 表同一份，所以修改元素能影响外部。

但如果在函数里重新赋值一个新 map，不会影响外部变量：

```go
func reset(m map[string]int) {
    m = make(map[string]int)
    m["x"] = 1
}
```

### 3. channel

channel 变量内部也是一个指向 `hchan` 的引用值。传 channel 会复制 channel 值，但底层队列同一份。

```go
func send(ch chan int) {
    ch <- 1
}
```

## 九、值接收者和指针接收者

### 1. 值接收者

```go
type User struct {
    Name string
}

func (u User) Rename(name string) {
    u.Name = name
}
```

`Rename` 拿到的是 `User` 的副本，外部不会变。

### 2. 指针接收者

```go
func (u *User) Rename(name string) {
    u.Name = name
}
```

`u` 是地址副本，但指向原对象，所以能改原对象。

### 3. 怎么选

| 场景 | 推荐 |
| --- | --- |
| 需要修改对象 | 指针接收者 |
| struct 较大 | 指针接收者 |
| 包含 mutex / atomic / noCopy | 指针接收者 |
| 小值对象、不可变语义 | 值接收者 |
| 要保持方法集一致 | 通常统一用指针接收者 |

典型值语义：

```go
time.Time
```

典型指针语义：

```go
bytes.Buffer
sync.Mutex
http.Client
sql.DB
```

## 十、struct 拷贝的线上坑

### 坑 1：复制带锁对象

```go
type SafeCounter struct {
    mu sync.Mutex
    n  int
}

func bad(c SafeCounter) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

`sync.Mutex` 被复制后，锁状态和保护的数据关系被破坏。实际项目里复制包含锁、连接池、buffer、context cancel func 的对象都要非常谨慎。

### 坑 2：大 struct 在热路径反复复制

```go
type Order struct {
    ID      int64
    Payload [4096]byte
}

func handle(o Order) {}
```

如果 QPS 很高，大对象值传递会增加 CPU 和内存带宽压力。可以改成 `*Order`，但不要为了“省拷贝”把所有东西都改成指针，指针也可能带来逃逸和 GC 压力。

### 坑 3：range 变量地址复用

```go
var users []*User
for _, u := range list {
    users = append(users, &u) // 旧版本 Go 常见坑
}
```

老版本 Go 中 `u` 是循环变量，地址会复用，最后可能都指向同一个变量。更稳妥写法：

```go
for i := range list {
    users = append(users, &list[i])
}
```

## 十一、Go vs Java vs C++ 指针差异

| 维度 | Go | Java | C++ |
| --- | --- | --- | --- |
| 参数传递 | 全部值传递 | 全部值传递，对象变量复制引用值 | 值、指针、引用都可选 |
| 指针运算 | 不支持 | 无显式指针 | 支持 |
| 内存释放 | GC | JVM GC | 手动 / RAII |
| 空指针 | nil pointer panic | NullPointerException | 未定义行为或崩溃 |
| 对象拷贝 | struct 默认值拷贝 | 对象变量复制引用 | 拷贝构造 / 移动语义 |

Java 里也不是“引用传递”：

```java
void reset(User u) {
    u = new User();
}
```

`u` 只是引用值的副本，重新赋值不会影响外部变量。

## 十二、面试真题

**Q1：Go 是值传递还是引用传递？**

Go 只有值传递。区别在于被复制的值是什么：int 复制数值，pointer 复制地址，slice 复制 header，map/channel 复制指向 runtime 对象的引用值。所以 slice/map/channel 修改底层数据会影响外部，但重新赋值变量本身不会影响外部。

**Q2：slice 传参后 append 为什么外部不一定看到？**

因为传进去的是 slice header 副本。append 如果没有扩容，修改元素能共享底层数组；append 如果扩容，会生成新数组和新 header，调用方原来的 header 不变。需要返回新 slice。

**Q3：什么时候用指针接收者？**

需要修改对象、对象较大、包含锁/连接池/buffer、希望方法集统一时，用指针接收者。小的不可变值对象可以用值接收者。

**Q4：指针一定更快吗？**

不一定。指针减少拷贝，但可能导致对象逃逸到堆上，增加 GC 压力，也会带来间接访问成本。热路径要用 benchmark 和 pprof 判断。

## 十三、面试表达

```text
Go 里不要把 slice/map/channel 简单说成引用传递。严格说 Go 只有值传递，只是这些类型的值内部持有指向底层数据结构的指针。
我判断值还是指针，主要看三点：是否需要修改原对象、对象大小和是否包含不可复制资源，比如 mutex、连接池、buffer。
在线上性能问题里，指针不是银弹。减少拷贝和增加 GC 压力之间要用 pprof 和 benchmark 做取舍。
```
