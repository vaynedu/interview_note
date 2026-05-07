# defer

> 延迟调用：函数返回前按 LIFO 执行；用于资源释放、解锁、recover、统计耗时

## 一、核心原理

### 1.1 三种实现方式（演进）

| 版本 | 实现 | 性能 |
| --- | --- | --- |
| Go ≤ 1.12 | 堆分配 _defer + 链表 | 慢（每次 ~50ns 分配） |
| Go 1.13 | 栈分配 _defer + 链表 | 快约 30% |
| Go 1.14+ | **open-coded defer**（编译器内联展开） | 接近零开销 |

**open-coded defer** 条件：函数内 `defer` 数量 ≤ 8 且不在循环中。编译器把 defer 展开成位图 + 直接调用，不走 _defer 链表。否则退化到栈分配链表。

### 1.2 _defer 结构（兜底实现）

```go
type _defer struct {
    started bool
    heap    bool        // 是否堆分配
    sp      uintptr     // 注册时栈指针
    pc      uintptr     // defer 语句的 PC
    fn      *funcval    // 要调用的闭包
    link    *_defer     // 链到 g._defer (LIFO 链表)
}
```

每次 `defer f()` 把 _defer 节点头插到 g._defer 链表。函数返回时 `runtime.deferreturn` 取链表头依次调用，符合 LIFO。

### 1.3 参数求值时机

**defer 语句执行时立即求值实参**，但函数体延迟到返回前执行：

```go
i := 1
defer fmt.Println(i)  // 立即捕获 i=1
i = 2
// 输出: 1
```

闭包则不同——闭包内部读外部变量是引用：

```go
i := 1
defer func() { fmt.Println(i) }()
i = 2
// 输出: 2
```

### 1.4 与 return 的执行顺序

`return` 不是原子操作，可以理解成函数有一个隐藏的“返回值槽位”：

```text
1. 计算 return 后面的表达式
2. 把结果赋给返回值槽位
3. 按 LIFO 执行所有 defer
4. 真正执行 RET，把返回值槽位里的值带回调用方
```

所以 defer 能不能改返回值，关键看它能不能访问到这个返回值槽位。

**命名返回值**情况下，defer 可以修改返回值：

```go
func foo() (n int) {
    defer func() { n++ }()
    return 1  // 实际返回 2
}
```

匿名返回值：

```go
func foo() int {
    n := 1
    defer func() { n++ }()
    return n  // 返回 1 (n++ 改的是局部 n,返回值已被赋值)
}
```

### 1.5 与 panic/recover 的关系

panic 触发后开始 unwind 栈，**沿途执行所有 defer**，直到某个 defer 里调 `recover()` 拦下，panic 流程结束，函数从该 defer 所在层正常返回。

只有在 defer 里直接调用 `recover()` 才有效，包一层函数就失效（不是 deferred function 本身）。

## 二、八股速记

- **LIFO** 顺序，最后注册先执行
- 实参在 `defer` 语句执行时**立即求值**
- 闭包内变量是**引用**，会读到执行时的值
- Go 1.14+ open-coded defer 接近零开销（数量 ≤8 且不在循环）
- 命名返回值 + defer 可改返回值
- `recover` 必须在 defer 函数体内**直接调用**才有效
- 循环里大量 defer 会膨胀链表，资源释放放循环外或显式调用

## 三、面试真题

**Q1：defer 什么时候执行？**
所在函数 return 时（包括 panic unwind）。**每个函数作用域**结束才执行，不是 block 结束。所以 for 循环里的 defer 要等整个函数结束才执行，可能造成资源积压。

**Q2：以下输出？**

```go
func main() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
    fmt.Println("end")
}
```

```
end
2
1
0
```

defer 实参立即求值，分别捕获 0/1/2；LIFO 输出 2/1/0。

**Q3：以下输出？**

```go
func foo() (n int) {
    defer func() { n++ }()
    defer func() { n *= 2 }()
    return 1
}
```

返回 **3**。

- `return 1`：n = 1
- 注册顺序：先 `n++` 后 `n*=2`，LIFO 执行：先 `n*=2` → n=2，再 `n++` → n=3
- 最终返回 **3**

（面试时小心，按 LIFO 仔细推）

**Q4：defer recover 的常见误用？**

```go
func badRecover() {
    defer recover()  // 无效! recover 必须在 defer 函数体内调用
    panic("x")
}
```

`defer recover()` 把 recover 的返回值存到 _defer 但不会真的拦截 panic。正确写法：

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("recovered: %v", r)
    }
}()
```

**Q5：defer 性能开销有多大？**
- 老版本（≤1.12）：约 50ns/次（堆分配）
- Go 1.13：约 35ns/次（栈分配）
- Go 1.14+：满足 open-coded 条件时几乎免费（编译期展开）

热路径每秒亿次调用才需要在意，业务代码尽情用。

**Q6：什么场景 defer 会退化到链表实现？**
1. defer 数量 > 8
2. 在循环里 defer
3. 函数有 `recover` 但被 inline 了（少见）
触发后退化到栈分配 _defer 链表，每次 ~30ns。

**Q7：defer 和 finally / try-with-resources 比较？**
| | Go defer | Java try-finally / try-with-resources |
| --- | --- | --- |
| 作用域 | 整个函数 | 单个 try block |
| 注册顺序 | 任意位置（运行到 defer 才注册） | 静态结构 |
| 灵活性 | 可条件 defer，可关闭函数动态 defer | 静态绑定 |
| 性能 | open-coded 后接近零开销 | JVM 优化通常更好 |

## 四、手写实现

**1. 优雅释放多资源（替代多层嵌套 if err != nil）：**

```go
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()

    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return err
    }
    defer db.Close()

    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer func() {
        if err != nil {
            tx.Rollback()
        }
    }()

    // ... 业务逻辑

    return tx.Commit()
}
```

**2. 计时辅助：**

```go
func timed(name string) func() {
    start := time.Now()
    return func() {
        log.Printf("%s took %v", name, time.Since(start))
    }
}

func handle() {
    defer timed("handle")()  // 注意立即调用 timed("handle"), 返回的闭包被 defer
    // ...
}
```

**3. 用 defer 防止 panic 让 g 退出：**

```go
func GoSafe(f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic: %v\n%s", r, debug.Stack())
            }
        }()
        f()
    }()
}
```

**4. 锁的安全释放：**

```go
mu.Lock()
defer mu.Unlock()
// 即使中间 panic, Unlock 也会执行
```

## 五、踩坑与最佳实践

### 坑 1：循环里 defer 资源泄漏

```go
func badLoop(paths []string) error {
    for _, p := range paths {
        f, err := os.Open(p)
        if err != nil { return err }
        defer f.Close()  // 全部堆积到函数末尾才执行
        // 处理 f
    }
    return nil
}
```

文件句柄到函数返回才释放。**修复**：包成函数

```go
func processOne(p string) error {
    f, err := os.Open(p)
    if err != nil { return err }
    defer f.Close()
    // ...
    return nil
}
for _, p := range paths { processOne(p) }
```

### 坑 2：defer 改返回值翻车

```go
func foo() int {
    n := 1
    defer func() { n++ }()
    return n  // 返回 1
}
```

匿名返回值时，`return n` 已经把 n 的值存到返回寄存器，再改局部 n 没用。要改用命名返回值。

### 坑 3：defer 闭包捕获循环变量

```go
for i := 0; i < 3; i++ {
    defer func() { fmt.Println(i) }()
}
// Go 1.22 前: 全是 3
// Go 1.22+: 0/1/2 (但 LIFO → 2/1/0)
```

老版本：循环内 `i := i` 创建副本。

### 坑 4：defer 顺序导致解锁前 panic

```go
mu.Lock()
defer mu.Unlock()
defer cleanup()  // 这个先执行(LIFO)
```

清理逻辑在锁内执行，可能死锁或数据竞争。注意 LIFO 顺序设计 defer 注册顺序。

### 坑 5：defer 调用 nil 函数

```go
var f func()
defer f()  // panic: invalid memory address (deferproc 时就 panic? 不, 是 deferreturn 时)
```

实际行为：`defer f()` 注册时不 panic，函数返回执行 defer 时 panic。

### 最佳实践

- **资源获取后立即 defer 释放**：Open + defer Close、Lock + defer Unlock
- **循环内的 defer 拆函数**
- 修改返回值场景明确用**命名返回值**
- recover 必须**直接**写在 defer 函数体内
- 性能关键热点用 `go test -bench` 验证 defer 开销，老版本可改写为显式调用
- defer 不能跨 goroutine（每个 g 独立 defer 栈）
