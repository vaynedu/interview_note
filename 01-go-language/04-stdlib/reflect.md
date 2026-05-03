# reflect

> Go 反射：运行时检视和操作类型与值；强大但慢，框架/序列化常用，业务热路径慎用

## 一、核心原理

### 1.1 三大法则（Russ Cox）

1. **interface → reflect.Value**：`reflect.ValueOf(x)`
2. **reflect.Value → interface**：`v.Interface()`
3. **修改 Value 必须可寻址（settable）**：传指针并 `.Elem()`

### 1.2 Type 与 Value

```go
v := reflect.ValueOf(x)  // 值的反射对象
t := reflect.TypeOf(x)   // 类型的反射对象
// 或: t := v.Type()
```

| 概念 | 描述 |
| --- | --- |
| `Type` | 类型元信息（Kind / Name / Field / Method） |
| `Value` | 实际值（可读、可能可写） |
| `Kind` | 底层类别（Int / String / Struct / Ptr / Slice / Map / ...） |

`Kind` 是最底层枚举，`Type` 是用户类型（`type MyInt int` 的 Kind=Int，Name="MyInt"）。

### 1.3 检视类型与字段

```go
type User struct {
    ID   int64  `json:"id" db:"user_id"`
    Name string `json:"name"`
}

t := reflect.TypeOf(User{})
fmt.Println(t.Name())     // "User"
fmt.Println(t.Kind())     // struct
fmt.Println(t.NumField()) // 2

for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    fmt.Println(f.Name, f.Type, f.Tag.Get("json"))
}
```

### 1.4 修���值（settable）

```go
var x int = 10
v := reflect.ValueOf(&x).Elem()  // 通过指针拿可寻址的 Value
v.SetInt(20)
fmt.Println(x)  // 20
```

**规则**：
- `reflect.ValueOf(x)` 拿到的是 x 的**副本**，不可写
- 必须 `reflect.ValueOf(&x).Elem()` 才能写
- 不可寻址的值（map element、字符串 index）不能直接 set

### 1.5 调用方法

```go
type Greeter struct{}
func (g Greeter) Hello(name string) string { return "hi " + name }

v := reflect.ValueOf(Greeter{})
m := v.MethodByName("Hello")
result := m.Call([]reflect.Value{reflect.ValueOf("world")})
fmt.Println(result[0].String())  // "hi world"
```

### 1.6 反射的代价

反射主要慢在：
1. **interface 装箱**：每次 `ValueOf` 都装箱，可能堆分配
2. **类型查找**：runtime 找 method、field 是哈希查找
3. **不可内联**：编译器无法对反射代码内联优化
4. **额外检查**：每次操作都做 Kind 校验

经验值：反射比直接调用慢 **10~100x**。但绝对值仍是 ns 级，业务代码（不是热点）影响通常可忽略。

### 1.7 典型用途

- **序列化**：encoding/json、xml、yaml 都用反射读 tag + 字段
- **ORM**：GORM 把 struct 字段映射到 DB 列
- **依赖注入**：wire / fx 解析依赖图
- **校验**：validator 读 tag 校验字段
- **DeepEqual**：reflect.DeepEqual 递归比较

## 二、八股速记

- **三大法则**：i→Value, Value→i, 改值要 Elem
- `Type` 是类型元信息，`Value` 是值，`Kind` 是底层类别
- `reflect.ValueOf(x)` 拿到副本，**只读**；要写传 `&x` 再 `.Elem()`
- 反射比直接调用慢 **10~100x**，但绝对值 ns 级
- struct tag 通过 `f.Tag.Get("json")` 读
- 反射调方法用 `MethodByName(...).Call(...)`
- 反射不能调用未导出字段/方法（首字母小写）
- `reflect.DeepEqual` 递归比较，可处理 slice/map/struct
- 泛型（Go 1.18+）能解决一些原本要反射的场景，更快更安全

## 三、面试真题

**Q1：reflect 的三大���则？**
1. **interface → reflect 对象**：`reflect.ValueOf(x)` 把任意值转成 `reflect.Value`
2. **reflect → interface**：`v.Interface()` 拿回去（通常配合类型断言）
3. **要修改值，Value 必须 settable**：拿到指针的 Elem 才行

**Q2：`reflect.ValueOf(x)` 为什么不能直接修改 x？**
Go 是值传递，`ValueOf(x)` 实际接收的是 x 的**副本**（通过 `interface{}` 装箱时复制了一份）。改这个副本对 x 没影响，所以 reflect 直接报错 "value is not addressable"。

**正确**：

```go
v := reflect.ValueOf(&x).Elem()  // &x 是地址,Elem 解引用拿到原始 x 的可寻址 Value
v.SetInt(100)
```

**Q3：怎么判断一个 interface{} 的真实类型？**

```go
// 方法 1: 类型断言(更快)
if i, ok := x.(int); ok { ... }

// 方法 2: type switch(多类型)
switch v := x.(type) {
case int: ...
case string: ...
}

// 方法 3: reflect(动态/未知类型)
t := reflect.TypeOf(x)
switch t.Kind() {
case reflect.Int: ...
}
```

业务代码优先用断言/switch，未知类型才反射。

**Q4：怎么读 struct tag？**

```go
type T struct {
    F string `json:"f" validate:"required"`
}
t := reflect.TypeOf(T{})
f, _ := t.FieldByName("F")
fmt.Println(f.Tag.Get("json"))      // "f"
fmt.Println(f.Tag.Get("validate"))  // "required"
```

`Tag` 类型是 `reflect.StructTag`，`Get` 按 key 查 value。

**Q5：反射能调用未导出方法吗？**
**不能**。反射调用未导出（小写）的方法/字段会报 "panic: reflect: call of reflect.Value.Method on zero Value" 或 settable 错。

突破方法：`unsafe`（极不推荐，破坏封装）。

**Q6：`reflect.DeepEqual` 行为？**
递归比较，规则：
- 基本类型：`==`
- slice/array：长度相等且元素 DeepEqual
- map：key 相同且 value DeepEqual
- struct：所有字段 DeepEqual
- pointer：指向同一个对象 OR 解引用后 DeepEqual
- interface：动态类型相同 + 值 DeepEqual
- func：只有都是 nil 才相等

**注意**：性能差，测试断言用没问题，业务热路径不要用。

**Q7：反射的性能开销在哪？**
1. interface 装箱（堆分配）
2. 字段/方法查找（哈希表）
3. 类型校验（每次操作都查 Kind）
4. 无法内联

对比：
- 直接 `s.Name` 读字段：< 1ns
- `v.FieldByName("Name").String()`：~100ns
- 1000x 慢，但绝对值仍小

**Q8：什么时候应该用反射？**
- 写**通用框架**：序列化（JSON/Protobuf）、ORM、配置加载、validator
- **类型不确定**且无法用泛型解决的场景
- **元编程**：根据 tag 做不同处理

**什么时候不用**：
- 业务代码热路径（特化具体类型）
- 能用泛型替代的（Go 1.18+）
- 单一类型的简单操作（用断言）

**Q9：反射怎么创建实例？**

```go
t := reflect.TypeOf(User{})
v := reflect.New(t).Elem()  // 等价于 var u User; v = reflect.ValueOf(&u).Elem()
v.FieldByName("Name").SetString("alice")
u := v.Interface().(User)
```

**Q10：反射能"破坏" Go 类型系统吗？**
某种程度上是。`Value.Interface().(T)` 类型断言失败 panic；`SetInt` 给非 int Kind 也 panic。**反射把编译期类型检查推迟到运行时**，所以反射代码必须严格测试。

## 四、手写实现

**1. 通用 struct → map[string]any（按 json tag）：**

```go
func StructToMap(v any) map[string]any {
    rv := reflect.ValueOf(v)
    if rv.Kind() == reflect.Ptr { rv = rv.Elem() }
    if rv.Kind() != reflect.Struct { return nil }

    rt := rv.Type()
    m := make(map[string]any, rt.NumField())
    for i := 0; i < rt.NumField(); i++ {
        f := rt.Field(i)
        if !f.IsExported() { continue }
        key := f.Tag.Get("json")
        if key == "" { key = f.Name }
        if key == "-" { continue }
        // 处理 omitempty
        if strings.Contains(key, ",omitempty") {
            key = strings.SplitN(key, ",", 2)[0]
            if rv.Field(i).IsZero() { continue }
        }
        m[key] = rv.Field(i).Interface()
    }
    return m
}
```

**2. 简化版 validator：**

```go
type Validator struct{}

func (Validator) Validate(v any) error {
    rv := reflect.ValueOf(v)
    if rv.Kind() == reflect.Ptr { rv = rv.Elem() }
    rt := rv.Type()
    for i := 0; i < rt.NumField(); i++ {
        tag := rt.Field(i).Tag.Get("validate")
        if tag == "required" {
            if rv.Field(i).IsZero() {
                return fmt.Errorf("%s is required", rt.Field(i).Name)
            }
        }
    }
    return nil
}

type User struct {
    Name string `validate:"required"`
    Age  int    `validate:"required"`
}
// Validator{}.Validate(User{Name: "x"}) → "Age is required"
```

**3. 通用 setter（配置加载）：**

```go
func SetField(obj any, name string, value any) error {
    rv := reflect.ValueOf(obj).Elem()
    f := rv.FieldByName(name)
    if !f.IsValid() { return fmt.Errorf("no field %s", name) }
    if !f.CanSet() { return fmt.Errorf("field %s not settable", name) }

    val := reflect.ValueOf(value)
    if f.Type() != val.Type() {
        // 尝试转换
        if !val.Type().ConvertibleTo(f.Type()) {
            return fmt.Errorf("type mismatch")
        }
        val = val.Convert(f.Type())
    }
    f.Set(val)
    return nil
}
```

**4. DeepEqual 简化版（理解原理���：**

```go
func deepEqual(a, b reflect.Value) bool {
    if a.Type() != b.Type() { return false }
    switch a.Kind() {
    case reflect.Int, reflect.Int64: return a.Int() == b.Int()
    case reflect.String: return a.String() == b.String()
    case reflect.Slice:
        if a.Len() != b.Len() { return false }
        for i := 0; i < a.Len(); i++ {
            if !deepEqual(a.Index(i), b.Index(i)) { return false }
        }
        return true
    case reflect.Struct:
        for i := 0; i < a.NumField(); i++ {
            if !deepEqual(a.Field(i), b.Field(i)) { return false }
        }
        return true
    }
    return false
}
```

## 五、踩坑与最佳实践

### 坑 1：忘记 Elem

```go
type T struct { N int }
v := T{}
reflect.ValueOf(v).FieldByName("N").SetInt(10)
// panic: reflect: reflect.Value.SetInt using unaddressable value
```

**修复**：传地址 + Elem

```go
reflect.ValueOf(&v).Elem().FieldByName("N").SetInt(10)
```

### 坑 2：Kind vs Type 混用

```go
type MyInt int
v := reflect.ValueOf(MyInt(1))
v.Type().Name()    // "MyInt"
v.Kind()            // reflect.Int
v.Type() == reflect.TypeOf(int(0))  // false (不同类型)
```

要判断"是不是整数"用 Kind；判断"是不是某具体类型"用 Type。

### 坑 3：未导出字段 panic

```go
type T struct { name string }  // 小写
reflect.ValueOf(&T{}).Elem().FieldByName("name").SetString("x")
// panic: reflect: reflect.Value.SetString using value obtained using unexported field
```

未导出字段不能 set（甚至 Interface() 也不行）。设计时让需要反射访问的字段大写。

### 坑 4：map 的 value 不可寻址

```go
m := map[string]int{"a": 1}
v := reflect.ValueOf(m).MapIndex(reflect.ValueOf("a"))
v.SetInt(2)  // panic
```

map element 不可寻址（map 内部可能 rehash 移动）。要改：取出值 → 改 → 写回 `m["a"] = 2`。

### 坑 5：反射在循环里

```go
for _, x := range items {
    v := reflect.ValueOf(x)
    // 每次都 ValueOf, 装箱 + 解析
}
```

热路径下显著拖慢。能特化就特化，不能特化考虑泛型。

### 坑 6：反射 + 接口的双重 nil

```go
var p *T = nil
var i interface{} = p  // i 不等于 nil

v := reflect.ValueOf(i)
v.IsNil()  // true (Value.IsNil 检查内部指针)
i == nil   // false (接口本身非 nil)
```

业务代码用 `v.IsNil()` 判断"装的指针是不是 nil"。

### 坑 7：DeepEqual 里 NaN ≠ NaN

`math.NaN() != math.NaN()` 即使用 DeepEqual 也是 false（IEEE 754 规定）。涉及 float 比较时要特殊处理。

### 最佳实践

- **能用断言/泛型就别用反射**
- 反射代码集中在 1~2 个工具函数里，业务侧不直接写
- struct tag 用反射读，性能可接受（启动时一次）
- **缓存 Type/Field 信息**：encoding/json 内部就缓存了，自己写框架时也缓存
- 测试用 `reflect.DeepEqual`，业务用 `==` 或自定义 Equal
- 性能敏感场景用 codegen（如 easyjson、protobuf）替代反射
- Go 1.18+ 优先泛型解决"参数化类型"需求
