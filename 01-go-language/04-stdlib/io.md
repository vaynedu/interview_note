# io

> Go IO 抽象核心：Reader/Writer/Closer 三大接口，靠组合（embedding）和适配器构建强大数据流处理

## 一、核心原理

### 1.1 三大基础接口

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}
```

**Reader 语义**：
- 把数据读入 p，返回读到的字节数和错误
- `n > 0` 时即使 err != nil 也要先处理 n 字节
- 流结束返回 `(0, io.EOF)` 或 `(n, io.EOF)`
- 阻塞直到有数据或出错

**Writer 语义**：
- 写入 p 的前 n 字节
- `n < len(p)` 必须 err != nil
- 写完整 p 才算成功

### 1.2 组合接口

通过 embedding 组合：

```go
type ReadWriter interface { Reader; Writer }
type ReadCloser interface { Reader; Closer }
type WriteCloser interface { Writer; Closer }
type ReadWriteCloser interface { Reader; Writer; Closer }
```

实战中函数参数尽量用最小接口（`io.Reader` 而不是 `*os.File`），方便 mock 和适配。

### 1.3 标准实现

| 类型 | 实现 |
| --- | --- |
| `*os.File` | Reader/Writer/Closer/Seeker |
| `*bytes.Buffer` | Reader/Writer |
| `*bytes.Reader` | Reader/Seeker/ReaderAt |
| `*strings.Reader` | Reader/Seeker |
| `*bufio.Reader` | 带缓冲的 Reader |
| `*bufio.Writer` | 带缓冲的 Writer |
| `net.Conn` | ReadWriteCloser |
| `http.Request.Body` | ReadCloser |
| `gzip.Reader` / `Writer` | 压缩 |

### 1.4 常用工具

```go
io.Copy(dst, src)              // 从 src 流式拷贝到 dst, 内部 32KB buffer
io.CopyN(dst, src, n)          // 限制 n 字节
io.CopyBuffer(dst, src, buf)   // 自带 buffer
io.ReadAll(r)                  // 读完所有, 返回 []byte (替代 ioutil.ReadAll)
io.ReadFull(r, buf)            // 必须读满 buf 才返回, 否则 ErrUnexpectedEOF
io.LimitReader(r, n)           // 限制最多 n 字节
io.TeeReader(r, w)             // 边读边写副本
io.MultiReader(r1, r2, ...)    // 串联多个 Reader
io.MultiWriter(w1, w2, ...)    // 同时写多个 Writer
io.Pipe()                      // 同步管道, 一端 write 另一端 read
io.Discard                     // 黑洞 Writer (替代 ioutil.Discard)
```

### 1.5 bufio 缓冲

```go
br := bufio.NewReader(r)
line, err := br.ReadString('\n')  // 按行读
b, _ := br.Peek(10)                // 预读不消费

bw := bufio.NewWriter(w)
bw.WriteString("hello")
bw.Flush()  // 必须 flush, 否则数据可能没真正写入底层
```

**为什么需要 bufio**：
- 减少系统调用次数（每次 read/write 都是 syscall）
- 提供按行/按字符的便利方法

`bufio.Scanner` 按行/按 token 扫描：

```go
sc := bufio.NewScanner(r)
sc.Buffer(make([]byte, 64*1024), 1<<20)  // 调大单行最大长度
for sc.Scan() {
    line := sc.Text()
}
if err := sc.Err(); err != nil { ... }
```

### 1.6 错误处理

特殊错误：
- `io.EOF`：正常结束（不是错误，是终止信号）
- `io.ErrUnexpectedEOF`：本来要读 N 字节但提前 EOF
- `io.ErrShortWrite`：Writer 写少了
- `io.ErrClosedPipe`：管道已关闭

**EOF 习惯**：

```go
for {
    n, err := r.Read(buf)
    if n > 0 {
        process(buf[:n])  // 先处理 n
    }
    if err == io.EOF { break }
    if err != nil { return err }
}
```

或更简洁地用 `io.Copy` / `io.ReadAll`。

## 二、八股速记

- 三大接口：**Reader/Writer/Closer**，组合出 ReadWriter 等
- Reader 返回 `(n, err)`：**n > 0 必须先处理**
- EOF 不是错误，是终止信号
- `bufio` 减少 syscall 次数；Writer **必须 Flush**
- `io.Copy` 流式拷贝，避免一次性读到内存
- `io.ReadAll` 替代 `ioutil.ReadAll`（Go 1.16+）
- 函数参数用**最小接口** `io.Reader` 而不是具体类型
- `io.Pipe` 实现同步管道，一端写一端读
- `LimitReader` / `MultiReader` / `TeeReader` 是常用组合工具
- 文件/网络资源**必 defer Close**

## 三、面试真题

**Q1：Reader 接口的 `Read` 方法语义？**
- 把数据读入 `p`，最多 `len(p)` 字节
- 返回 `(n, err)`：**n > 0 时即使 err != nil 也要先处理 n 字节**
- 流结束：`(0, io.EOF)` 或 `(n, io.EOF)`
- 不允许 `(0, nil)` 阻塞（除非作者明确文档说明）

**为什么这样设计**：避免每次 Read 必须返回完整的请求字节数，给 Reader 实现以灵活性（网络可能只读到一部分就返回）。

**Q2：什么时候用 `io.Copy` 而不是 `io.ReadAll`？**
- `ReadAll`：全读到内存，适合小数据（< MB）
- `Copy`：流式，适合大数据（GB 级文件、网络流）

```go
// 差: 100MB 文件全读到内存
data, _ := io.ReadAll(file)
w.Write(data)

// 好: 32KB 缓冲流式拷贝
io.Copy(w, file)
```

**Q3：bufio.Writer 不 Flush 会怎样？**

```go
bw := bufio.NewWriter(file)
bw.WriteString("hello")
// 没 Flush, 程序退出, 文件里没 "hello"
```

bufio 内部攒够 4096B 才写底层。Flush 强制刷出。

**惯用法**：

```go
defer bw.Flush()
```

> ⚠️ defer Flush 错误会被吞，关键场景手动 Flush 检查 err。

**Q4：`io.EOF` 是错误吗？**
**不是**，是终止信号。`Read` 返回 `(0, io.EOF)` 表示正常读完。当成错误处理会让正常流程报错。

```go
for {
    n, err := r.Read(buf)
    if err == io.EOF { break }   // 正常退出
    if err != nil { return err } // 真错误
    process(buf[:n])
}
```

**Q5：怎么实现一个边读边算 hash 的 Reader？**

```go
hr := io.TeeReader(r, hash)
io.Copy(dst, hr)
// hash.Sum(nil) 即文件 hash
```

`TeeReader` 像 Unix 的 `tee`：读 r 时同时写到 w。

**Q6：`io.Pipe` 怎么用？**

```go
pr, pw := io.Pipe()

go func() {
    defer pw.Close()
    json.NewEncoder(pw).Encode(largeData)
}()

// 主 g 当 reader, 流式发请求
http.Post(url, "application/json", pr)
```

**典型场景**：把生产者的 Writer 输出转成消费者的 Reader 输入，零中间内存。

注意：Pipe 是同步的，writer 阻塞直到 reader 读走。

**Q7：怎么从 Reader 限制最多读 N 字节？**

```go
limited := io.LimitReader(r, 1<<20)  // 1MB
data, _ := io.ReadAll(limited)
```

防恶意大 body / 大文件吃内存。HTTP 用 `http.MaxBytesReader`。

**Q8：怎么把多个 Reader 串成一个？**

```go
r := io.MultiReader(strings.NewReader("hello "), file, strings.NewReader(" end"))
io.Copy(os.Stdout, r)  // 顺序输出
```

或多个 Writer 同时写：

```go
w := io.MultiWriter(file, os.Stdout)  // tee 风格
fmt.Fprintln(w, "log line")
```

**Q9：bufio.Scanner 一行太长会怎样？**
默认单 token 上限 `bufio.MaxScanTokenSize = 64KB`，超过返回 `bufio.ErrTooLong`。

**修复**：

```go
sc := bufio.NewScanner(r)
sc.Buffer(make([]byte, 64*1024), 10*1024*1024)  // 10MB 上限
```

**Q10：文件 Reader/Writer 关闭顺序？**

```go
f, _ := os.Create("x.txt")
bw := bufio.NewWriter(f)
bw.WriteString("data")

// 顺序: 先 Flush, 再 Close 文件
bw.Flush()
f.Close()

// 或 defer 反序
defer f.Close()
defer bw.Flush()
```

defer LIFO，所以先注册 Close 再注册 Flush，执行时 Flush 先于 Close。

## 四、手写实现

**1. 限速 Reader（带宽控制）：**

```go
type RateLimitReader struct {
    R       io.Reader
    BPS     int  // 每秒字节数
    last    time.Time
    sent    int
}

func (r *RateLimitReader) Read(p []byte) (int, error) {
    n, err := r.R.Read(p)
    r.sent += n
    elapsed := time.Since(r.last)
    if elapsed < time.Second {
        if float64(r.sent)/elapsed.Seconds() > float64(r.BPS) {
            wait := time.Duration(float64(r.sent)/float64(r.BPS)*float64(time.Second)) - elapsed
            time.Sleep(wait)
        }
    } else {
        r.last = time.Now()
        r.sent = n
    }
    return n, err
}
```

**2. 计算文件 SHA256（流式）：**

```go
func hashFile(path string) (string, error) {
    f, err := os.Open(path)
    if err != nil { return "", err }
    defer f.Close()

    h := sha256.New()
    if _, err := io.Copy(h, f); err != nil { return "", err }
    return hex.EncodeToString(h.Sum(nil)), nil
}
```

**3. 边下载边压缩边上传：**

```go
func compressAndUpload(srcURL, dstURL string) error {
    resp, err := http.Get(srcURL)
    if err != nil { return err }
    defer resp.Body.Close()

    pr, pw := io.Pipe()
    go func() {
        defer pw.Close()
        gz := gzip.NewWriter(pw)
        defer gz.Close()
        io.Copy(gz, resp.Body)  // 边读 HTTP 边压缩到 pipe
    }()

    req, _ := http.NewRequest("POST", dstURL, pr)
    _, err = http.DefaultClient.Do(req)
    return err
}
```

**4. ProgressReader（进度回调）：**

```go
type ProgressReader struct {
    R      io.Reader
    Total  int64
    Read   int64
    OnRead func(read, total int64)
}

func (p *ProgressReader) Read(b []byte) (int, error) {
    n, err := p.R.Read(b)
    p.Read += int64(n)
    if p.OnRead != nil { p.OnRead(p.Read, p.Total) }
    return n, err
}

// 使用
pr := &ProgressReader{R: file, Total: size, OnRead: func(r, t int64) {
    fmt.Printf("\r%.1f%%", float64(r)/float64(t)*100)
}}
io.Copy(dst, pr)
```

## 五、踩坑与最佳实践

### 坑 1：bufio.Writer 没 Flush

```go
f, _ := os.Create("x")
defer f.Close()
bw := bufio.NewWriter(f)
bw.WriteString("hello")
// 文件里没 hello
```

**修复**：`defer bw.Flush()`（注意要在 `defer f.Close()` 之后注册，先执行）。

### 坑 2：Read 没处理 n > 0 + err != nil

```go
n, err := r.Read(buf)
if err != nil { return err }  // 错: 丢了 n 字节
process(buf[:n])
```

**修复**：

```go
n, err := r.Read(buf)
if n > 0 { process(buf[:n]) }
if err == io.EOF { break }
if err != nil { return err }
```

或用 `io.Copy` / `io.ReadFull` 等高层 API。

### 坑 3：`io.ReadAll` 大文件 OOM

```go
data, _ := io.ReadAll(httpBody)  // 用户上传 10GB → OOM
```

**修复**：限制大小

```go
data, err := io.ReadAll(io.LimitReader(httpBody, 10<<20))  // 10MB
```

或用 `http.MaxBytesReader`。

### 坑 4：Close 错误被吞

```go
defer f.Close()  // 写场景下 Close 错误重要(可能是 fsync 失败)
```

**修复**：写场景手动 Close 检查

```go
err = f.Close()
if err != nil { return err }
```

### 坑 5：`bufio.Scanner` 单行超长

日志文件偶尔有超长行 → `bufio.ErrTooLong`。`Scanner.Err()` 才能拿到。**修复**：调大 buffer。

### 坑 6：`io.Pipe` 一端忘记 Close

```go
pr, pw := io.Pipe()
go func() {
    pw.Write(data)
    // 没 pw.Close() → 读端永远等待
}()
io.Copy(dst, pr)  // 永久阻塞
```

**修复**：写完 `defer pw.Close()`。

### 坑 7：用 `==` 比较 EOF

```go
if err == io.EOF { ... }  // ✓ EOF 是 sentinel error
```

但被 wrap 后失效。安全写法：

```go
if errors.Is(err, io.EOF) { ... }
```

### 坑 8：Reader 不支持 Seek 还想跳过

```go
io.CopyN(io.Discard, r, 100)  // 跳过 100 字节(读丢)
```

或如果支持 Seeker：`r.Seek(100, io.SeekStart)`。

### 最佳实践

- **接口最小化**：函数参数用 `io.Reader` 而不是 `*os.File`
- **流式优先**：大数据用 `io.Copy`，小数据 `io.ReadAll`
- **限制大小**：处理外部输入必 `LimitReader`
- **bufio 必 Flush**：写完显式 Flush 检查 err
- **Pipe 双端 Close**：写完关写端，读完关读端
- **defer Close 顺序**：注册顺序反过来执行
- **避免 EOF 当错误**：业务循环里 `errors.Is(err, io.EOF)` 单独处理
- **读写场景 Close 错误**：写场景必检查（fsync 失败可能丢数据）
- 替代 `ioutil`：Go 1.16+ 用 `io.ReadAll` / `os.ReadFile` 等
