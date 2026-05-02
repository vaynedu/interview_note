# slice

> Go 切片：底层数组 + 长度 + 容量的三元组；引用类型，传递的是 header 副本但共享底层数组

## 一、核心原理

### 1.1 底层结构

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 指向底层数组
    len   int            // 当前长度
    cap   int            // 容量(从 array 起到底层数组末尾)
}
```

- `len` ≤ `cap`，`s[i]` 越界检查比较 `i < len`
- `s[i:j]` 共享底层数组，新 slice 的 `len = j-i`，`cap = cap(s)-i`
- `s[i:j:k]` 三索引切片，可显式控制新 cap = `k-i`，避免后续 append 写穿原 slice

### 1.2 扩容规则（Go 1.18+）

`append` 触发扩容时调用 `runtime.growslice`：

1. 期望容量 `newcap`：
   - 旧 cap < 256：`newcap = oldcap * 2`
   - 旧 cap ≥ 256：`newcap = oldcap + (oldcap + 3*256) / 4`（约 1.25x，平滑过渡）
2. 按元素大小做内存对齐，最终 cap 可能比 `newcap` 大（向 `mallocgc` size class 对齐）
3. 分配新底层数组，拷贝旧数据，返回新 slice header

> **注意**：Go 1.17 及之前是 1024 阈值 + 2x/1.25x 规则，1.18 改为 256 + 平滑公式。面试问到要按版本回答。

### 1.3 nil slice vs 空 slice

| | `var s []int` | `s := []int{}` | `s := make([]int, 0)` |
| --- | --- | --- | --- |
| 底层指针 | nil | 非 nil(指向 zerobase) | 非 nil |
| len | 0 | 0 | 0 |
| cap | 0 | 0 | 0 |
| `s == nil` | true | false | false |
| append 行为 | 一致 | 一致 | 一致 |

JSON 序列化：nil → `null`，空 slice → `[]`。

## 二、八股速记

- 切片是 `{ptr, len, cap}` 三元组，**值传递**但共享底层数组
- 扩容：cap<256 翻倍，≥256 按 1.25x 平滑增长（Go 1.18+）
- `append` 可能返回新 slice，**必须接收返回值**
- `s[i:j]` 共享底层数组；`s[i:j:k]` 可限定 cap 防写穿
- `nil slice` 和空 slice 行为基本一致，区别在于 `==nil` 判断和 JSON
- range 取地址要小心，循环变量是值拷贝（Go 1.22 前每轮共用同一变量）
- 大 slice 截取小段会导致原数组无法 GC，需 `copy` 到新 slice

## 三、面试真题

**Q1：append 后原 slice 一定不变吗？**
不一定。如果未触发扩容，新 slice 与原 slice 共享底层数组，`append` 写入会修改原数组对应位置（前提是原 slice 的 cap 之内还有空间），可能影响其他持有该底层数组的 slice。扩容后则各自独立。

**Q2：`s := make([]int, 5, 10); s2 := s[2:4]; s2 = append(s2, 99)` 之后 `s[4]` 是多少？**
`s[4] == 99`。`s2` 的 cap = 10-2 = 8，append 不扩容，直接写到底层数组 index=4 的位置，正好是 `s[4]`。

**Q3：函数传 slice 修改长度，外部能感知吗？**
不能。slice header 是值拷贝，函数内 `append` 改的是局部 header 的 len。要么返回新 slice，要么传 `*[]T`。但**修改元素**外部能看到（共享底层数组）。

**Q4：怎么安全地从大 slice 中提取小段且释放原内存？**
```go
small := make([]T, len(big[i:j]))
copy(small, big[i:j])
// big 可被 GC
```

**Q5：slice 和数组的本质区别？**
数组是值类型，长度是类型的一部分（`[3]int` 和 `[4]int` 是不同类型）；slice 是引用语义的描述符，长度运行时可变。

## 四、手写实现

**手写一个简化的 append（仅 int）：**

```go
func myAppend(s []int, vs ...int) []int {
    need := len(s) + len(vs)
    if need <= cap(s) {
        s = s[:need]
        copy(s[len(s)-len(vs):], vs)
        return s
    }
    // 扩容
    newCap := cap(s)
    if newCap == 0 {
        newCap = need
    } else {
        for newCap < need {
            if newCap < 256 {
                newCap *= 2
            } else {
                newCap += (newCap + 3*256) / 4
            }
        }
    }
    ns := make([]int, need, newCap)
    copy(ns, s)
    copy(ns[len(s):], vs)
    return ns
}
```

**手写删除指定 index（保持顺序）：**

```go
func remove(s []int, i int) []int {
    return append(s[:i], s[i+1:]...)
}
```

**不保序删除（O(1)）：**

```go
func removeFast(s []int, i int) []int {
    s[i] = s[len(s)-1]
    return s[:len(s)-1]
}
```

## 五、踩坑与最佳实践

### 坑 1：append 共享底层数组导致数据被改写

```go
a := []int{1, 2, 3, 4, 5}
b := a[:2]            // b=[1,2], cap=5
b = append(b, 99)     // 没扩容,直接写到 a[2]
fmt.Println(a)        // [1 2 99 4 5]  ← a 被改了
```

**修复**：用三索引切片 `b := a[:2:2]`，强制扩容。

### 坑 2：range 取地址坑（Go 1.22 前）

```go
ptrs := []*int{}
for _, v := range []int{1, 2, 3} {
    ptrs = append(ptrs, &v) // 都指向同一个 v
}
// Go 1.22 前: ptrs 全指向 3
// Go 1.22+: 每轮新变量,行为符合直觉
```

老版本写法：循环内 `v := v` 创建副本。

### 坑 3：截取大 slice 导致内存泄漏

读 100MB 文件后 `result := data[0:10]`，`data` 因为被 result 引用无法释放。`copy` 到新 slice 解决。

### 坑 4：并发 append 不安全

slice 不是并发安全数据结构，多 goroutine append 同一 slice 会导致数据丢失甚至 panic。需要锁，或用 channel 收集。

### 最佳实践

- 已知容量时 `make([]T, 0, n)` 预分配，避免多次扩容
- 函数返回 slice 时考虑是否要 `copy` 隔离调用方
- 不要把 slice 当 set 用，O(n) 查找；用 `map[T]struct{}`
- 判空用 `len(s) == 0`，不要用 `s == nil`
