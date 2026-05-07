# Go 指针、值传递与拷贝

> 这类题表面问指针，实际考 Go 的内存模型、方法接收者、逃逸分析和线上性能坑。核心结论：Go 只有值传递，只是有些值本身保存了指向底层数据的指针。

## 一、一句话总结

```text
Go 函数参数永远是值传递。
传 struct 会复制整个 struct。
传 pointer 会复制一个地址值。
传 slice / map / channel 会复制 header，但 header 里指向同一份底层数据。
```

## 二、先把“值传递”讲明白

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

## 三、slice / map / channel 为什么像引用

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

## 四、值接收者和指针接收者

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

## 五、struct 拷贝的线上坑

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

## 六、Go vs Java vs C++ 指针差异

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

## 七、面试真题

**Q1：Go 是值传递还是引用传递？**

Go 只有值传递。区别在于被复制的值是什么：int 复制数值，pointer 复制地址，slice 复制 header，map/channel 复制指向 runtime 对象的引用值。所以 slice/map/channel 修改底层数据会影响外部，但重新赋值变量本身不会影响外部。

**Q2：slice 传参后 append 为什么外部不一定看到？**

因为传进去的是 slice header 副本。append 如果没有扩容，修改元素能共享底层数组；append 如果扩容，会生成新数组和新 header，调用方原来的 header 不变。需要返回新 slice。

**Q3：什么时候用指针接收者？**

需要修改对象、对象较大、包含锁/连接池/buffer、希望方法集统一时，用指针接收者。小的不可变值对象可以用值接收者。

**Q4：指针一定更快吗？**

不一定。指针减少拷贝，但可能导致对象逃逸到堆上，增加 GC 压力，也会带来间接访问成本。热路径要用 benchmark 和 pprof 判断。

## 八、面试表达

```text
Go 里不要把 slice/map/channel 简单说成引用传递。严格说 Go 只有值传递，只是这些类型的值内部持有指向底层数据结构的指针。
我判断值还是指针，主要看三点：是否需要修改原对象、对象大小和是否包含不可复制资源，比如 mutex、连接池、buffer。
在线上性能问题里，指针不是银弹。减少拷贝和增加 GC 压力之间要用 pprof 和 benchmark 做取舍。
```
