# slice

> Go 切片：底层数组 + 长度 + 容量的三元组；引用类型，传递的是 header 副本但共享底层数组

## 一、一句话总结(背诵版)

> **slice 是 Go 中对底层数组的轻量级动态视图**——header 三元组 `{*array, len, cap}`,**值传递但共享底层数组**;常见场景包括**动态收集结果、子切片视图、参数传递避免拷贝、预分配优化、配合 append 构建结果集**。

延伸(被追问时再展开):

> slice 真正的"坑"集中在三个点:**append 可能扩容也可能不扩容**(返回值必须接)、**子切片与原 slice 共享底层数组**(改一个影响另一个)、**header 是值拷贝**(函数内 append 不影响外部 len)。理解这三点就理解了 80% 的 slice 面试题。

---

## 二、使用场景(场景化记忆)

| 场景 | 一句话 | 最小代码 |
| --- | --- | --- |
| **动态收集结果** | 边遍历边攒,长度未知 | `res := []T{}; for ... { res = append(res, x) }` |
| **预分配优化** | 已知大致容量,一次到位免扩容 | `res := make([]T, 0, n)` |
| **子切片视图** | 不拷贝复用底层数组的某一段 | `sub := s[i:j]` |
| **参数传递** | 传 header 副本(24 字节),共享底层 | `func f(s []T)`,内部改元素影响外部 |
| **函数返回新 slice** | append 可能扩容,必须返回 | `func add(s []T, x T) []T { return append(s, x) }` |
| **隔离写入污染** | 三索引切片强制后续 append 扩容 | `sub := s[i:j:j]` 锁死 cap |
| **批量删除 / 过滤** | 用同一底层数组原地过滤 | `s = s[:0]; for _, v := range src { if ok(v) { s = append(s, v) } }` |

**判断要不要 copy**:**只要 caller 和 callee 谁都可能继续 append,就要 copy 或用三索引**。

---

## 三、常见错误(高频踩坑)

| 错误 | 现象 | 根因 | 修复 |
| --- | --- | --- | --- |
| **append 不接返回值** | `append(s, x)` 写完外部 s 长度没变 | append 返回的是新 header,旧 header 的 len 不变 | `s = append(s, x)` 必须赋值回去 |
| **子切片污染原数组** | `b := a[:2]; b = append(b, 99)` 之后 `a[2]` 被改成 99 | b 与 a 共享底层,cap 还够时 append 直接写底层数组 | 用三索引 `b := a[:2:2]` 锁死 cap,或 `copy` 出新 slice |
| **大 slice 截取小段内存泄漏** | 读 100MB 文件后只留 10 字节,整块内存不释放 | 小 slice 的 array 指针仍指向 100MB 数组,GC 无法回收 | `small := make([]T, n); copy(small, big[i:j])` |
| **并发 append 数据竞争** | 多 goroutine `append(s, x)` 出现元素丢失 / panic | slice 非并发安全,len 和底层数组写都不是原子 | 加 `sync.Mutex`,或用 channel 收集后单 G 合并 |
| **range 取 &v 都指向同一变量(Go 1.22 前)** | `ptrs = append(ptrs, &v)` 后所有指针指向最后一个值 | 循环变量 v 每轮复用同一地址(Go 1.22 才改) | Go 1.22 前显式 `v := v` 创副本;Go 1.22+ 行为已修正 |
| **nil slice 与 empty slice 混用** | `s == nil` 判断失败但 `len(s) == 0` | `[]T{}` 和 `make([]T, 0)` 不是 nil,只有 `var s []T` 是 nil | 判空统一用 `len(s) == 0`,JSON 序列化才区分(nil → null,empty → []) |
| **循环里 append 不预分配** | 10w 次 append 触发多次扩容 + 拷贝,性能差 | 默认 cap=0,翻倍扩容期间反复 malloc + copy | `make([]T, 0, expectedSize)` 一次到位 |
| **以为 slice 是引用类型** | 函数内 `s = append(s, x)` 后,外部 s 长度还是旧的 | slice 是值类型(传 header 副本),append 改的是局部 header | 函数返回新 slice,或传 `*[]T` |

---

## 四、面试常问(简答模板)

**Q1:slice 和数组的区别?**
数组是**值类型**,长度是类型的一部分(`[3]int` 和 `[4]int` 是不同类型),传参全拷贝。slice 是**对底层数组的描述符**(header 三元组),长度运行时可变,传参只拷贝 24 字节 header,但共享底层数组。

**Q2:slice header 是什么?**
`{array unsafe.Pointer, len int, cap int}` 三元组,64 位下 24 字节。array 指向底层数组,len 是当前长度,cap 是从 array 起到底层数组末尾的容量。

**Q3:append 的扩容规则?**
Go 1.18+:cap < 256 翻倍,cap ≥ 256 按 `oldCap + (oldCap + 3*256)/4`(约 1.25x,平滑过渡)。Go 1.17 及之前是 1024 阈值的 2x/1.25x。最终还要按内存分配器 size class 对齐,实际 cap 可能略大。

**Q4:append 的潜在 bug?**
两个:一是**没接返回值**——扩容后旧 header 看不到新底层数组;二是**子切片共享底层**——`b := a[:2]; b = append(b, x)` 在 cap 够时直接写到 a 的对应位置,污染原 slice。

**Q5:为什么传 slice 也"像引用"?**
header 是值拷贝,但 header 里的 array 指针指向同一底层数组,所以**改元素**外部看得到(共享底层),**改长度**外部看不到(header 是副本)。

**Q6:如何强制脱离底层数组共享?**
两种方式:一是**三索引切片** `s[i:j:j]` 锁死 cap,后续 append 必触发扩容分配新数组;二是**显式 copy**:`dst := make([]T, len(src)); copy(dst, src)`。

---

## 五、深水区:原理与源码(被追问时看)

> 下面是 slice 的 header 结构、扩容算法、内存布局等源码级内容,**正常面试用不到**,只在被深追"growslice 怎么算 newcap / 三索引底层怎么实现"时才会用到。

## 〇、核心提炼（5 段式）

### 核心机制（4 条必背）

1. **三元组 header 结构** - `{*array, len, cap}`，header 是值类型但 `*array` 指向底层数组（**多个 slice 可共享底层**）
2. **扩容规则** - cap < 256 翻倍；cap ≥ 256 按 1.25x 增长（Go 1.18+，旧版本是 1024 阈值的 2x/1.25x）
3. **append 行为** - 容量够 → 复用底层数组；容量不够 → 分配新数组 + 拷贝
4. **传递是 header 副本** - 函数内 append 可能不影响外部（如果触发扩容指向新数组），但修改元素一定影响

### 核心本质（必懂）

> slice 的本质是 **"动态数组的轻量级 wrapper"**：
>
> - **不是引用类型**（严格说）：传值传 header（24 字节：指针 + 长度 + 容量）
> - **看起来像引用**：因为 header 中的指针指向同一底层数组
> - **append 是危险操作**：可能返回新 slice（扩容），也可能修改原 slice（共享底层）
>
> **关键事实**：
> - **不同 slice 可能共享底层数组**：`s[1:3]` 与 `s` 共享，修改其一影响另一个
> - **append 是否扩容不可见**：业务代码看不出来，必须用返回值
> - **扩容会复制**：大 slice 频繁 append 会有内存分配 + 拷贝开销
> - **slice 不能比较**（除了与 nil）：因为内容是动态的

### 完整流程（面试必背）

```
slice 底层结构（runtime/slice.go）:
  type slice struct {
      array unsafe.Pointer  // 指向底层数组
      len   int             // 长度（已使用）
      cap   int             // 容量（最大可用）
  }

make([]T, len, cap):
  1. 分配大小为 cap × sizeof(T) 的底层数组
  2. 返回 header { array: 数组指针, len: len, cap: cap }

append(s, x):
  1. 如果 len(s) < cap(s):
     - 在 s[len] 位置写入 x
     - 返回新 header { array: 原指针, len: len+1, cap: cap }
     - 不分配新内存

  2. 如果 len(s) == cap(s):
     - 计算新 cap（按扩容规则）
     - 分配新数组（newCap × sizeof(T)）
     - 拷贝原数组到新数组
     - 在新数组末尾写入 x
     - 返回新 header { array: 新指针, len: len+1, cap: newCap }
     - 原 slice 不受影响（除非赋值覆盖）

扩容规则（Go 1.18+ growslice）:
  oldCap < 256:    newCap = oldCap × 2
  oldCap >= 256:   newCap = oldCap + (oldCap + 3*256)/4  // 平滑过渡到 1.25x

  最终 newCap 还要 roundup 到内存分配器的 size class

切片操作:
  s[low:high]:
    - 返回 header { array: &s.array[low], len: high-low, cap: cap(s)-low }
    - 共享底层数组（!）

  s[low:high:max]:
    - 同上但 cap 限定为 max-low
    - 限制后续 append 不会污染原 slice
```

### 4 条核心机制 - 逐点讲透

#### 1. 三元组 header（不是引用类型）

```
误解: "slice 是引用类型"
真相: slice 是值类型（24 字节 header），但 header 中的指针指向共享底层

证据:
  func f(s []int) {
      s[0] = 999       // 改元素 → 影响外部（共享底层）
      s = append(s, 1) // append → header 改变，外部不知道
      s[0] = 888       // 此时改的可能是新数组（扩容时）
  }

  var a = []int{1, 2, 3}
  f(a)
  // a[0] 一定是 999（修改元素）
  // a 的长度不变（append 在 f 内的 s 上）
  // a[0] 可能是 999 或 888（看是否扩容）

正确改外部:
  s = append(s, x)     // 函数返回新 slice 让调用者接
  func f(s *[]int)     // 传指针
```

#### 2. 扩容规则（避免误解）

```
Go 1.17 及之前:
  cap < 1024: newCap = oldCap × 2
  cap >= 1024: newCap = oldCap × 1.25

Go 1.18+:
  cap < 256: newCap = oldCap × 2
  cap >= 256: newCap = oldCap + (oldCap + 3 × 256) / 4
  → 平滑过渡（避免 1024 处的突变）

最终内存:
  newCap 还要根据 sizeof(T) 对齐到内存分配器 size class
  → 实际 cap 可能略大于计算值

预分配优化:
  s := make([]int, 0, expectedSize)  // 一次到位
  for ... { s = append(s, x) }
  → 避免多次扩容 + 拷贝
  → 大数据场景必做
```

#### 3. append 行为（陷阱）

```
共享底层的陷阱:
  s := []int{1, 2, 3, 4, 5}  // len=5, cap=5
  s1 := s[:3]                 // [1,2,3]，len=3, cap=5（共享）
  s2 := append(s1, 99)        // [1,2,3,99]，cap=5 → 不扩容

  s[3]  // 99！s2 的 append 改写了 s 的第 4 个元素

解决方案 1: 三参数切片
  s1 := s[:3:3]               // cap 限定 3
  s2 := append(s1, 99)        // cap 不够 → 扩容 → 新数组
  s[3] // 还是 4，未被改

解决方案 2: 显式拷贝
  s1 := make([]int, 3)
  copy(s1, s[:3])

为什么这个坑常见:
  range / 函数传参 / sub-slice 都共享底层
  写库函数时要特别小心
```

#### 4. 传递是 header 副本

```
24 字节 header（64 位系统）:
  - 8 字节 array 指针
  - 8 字节 len
  - 8 字节 cap

函数调用时:
  Go 按值传 header（24 字节拷贝）
  但 array 指针指向同一底层数组

性能含义:
  - slice 传参不是"全拷贝"（只 24 字节）
  - 不需要传 *[]T 来"避免拷贝"
  - 但要明白共享底层

API 设计:
  func process(data []int) error              // OK，传 header
  func processInPlace(data []int) error       // 修改 data 内容
  func extend(data []int) ([]int, error)      // 可能扩容时返回新 slice
  func extendInPlace(data *[]int) error       // 强制原 slice 反映新长度
```

### 一句话总结

> slice 的核心是：**三元组 header（指针+len+cap）+ 扩容规则（1.18+ 平滑 2x→1.25x）+ append 可能扩容也可能共享 + 传值是 header 副本**，
> 本质是**动态数组的轻量级 wrapper**：header 是值类型，但指针让多个 slice 共享底层数组。
> **三大坑**：append 是否扩容看运行时（外部接返回值最稳）；sub-slice 共享底层（修改互相影响）；并发 append 不安全（必须加锁）。
> **优化**：`make([]T, 0, cap)` 预分配，`s[:n:n]` 三参数切片防写入污染。

---

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
