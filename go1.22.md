## 语言的变化

Go 1.22 对`for`循环进行了两项更改，和一项实验性功能。

### 循环变量的调整

以前，`for`循环声明的变量只创建一次，并在每次迭代时更新。在 Go 1.22 中，循环的每次迭代都会创建新变量，以避免意外共享错误。举个例子：

```go
i := 0
ss := []string{"123", "234", "345"}
for _, s := range ss {
    i++
    go func() {
        fmt.Println(s)
    }()
}
```

对于以前的版本，上述代码大概率会输出三个"345"，因为在每一遍`for`循环时，`s`是同一个变量，只不过每次循环把它更新成了新值。而使用`go`语句启动协程一般是需要一些时间的，协程真正开始执行会比`for`循环要慢一些的，此时的`s`已经变成了最后一个值"345"，所以三个协程都会输出"345"。

而在 Go 1.22 中，上述代码会将三个字符串各输出一遍，三个字符串的输出顺序会因为协程执行的顺序而不同。

### 整数范围循环

现在，range后面允许是一个整数了，例如：

```go
for i := range 10 { // 等价于 for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

### （实验性功能）函数迭代器

想要使用这个功能，需要在编译时启用环境变量`GOEXPERIMENT=rangefunc`。

```go
ss := []string{"123", "234", "345"}
f := func(cb func(int, string) bool) {
    for i, s := range ss {
        if !cb(i, s) {
            return
        }
    }
}
for i, s := range f {
    fmt.Println(i, s)
}
```

从上述代码中可以看到，`f`是一个函数，它的格式为`func(cb func(xxx) bool)`，其中xxx可以为零到两个参数，两个参数支持任意类型，我们将其称为“函数迭代器”。当我们`for i, s := range f`时，会将for下面的大括号体中的内容作为`cb`函数传入`f`函数中去，然后前面的`i, s := `两个参数对应`cb`的两个形参，并且变量类型和`cb`的形参声明的类型也一致。

通过这种语法，我们可以方便的生成一些迭代器，例如切片的逆序迭代器：

```go
func Backward[E any](s []E) func(func(int, E) bool) {
    return func(yield func(int, E) bool) {
        for i := len(s)-1; i >= 0; i-- {
            if !yield(i, s[i]) {
                return
            }
        }
    }
}

func main() {
    s := []string{"hello", "world"}
    for i, x := range Backward(s) {
        fmt.Println(i, x)
    }
}
```

## go vet 工具

### 对循环变量的引用

由于上文的[（实验性功能）函数迭代器](#实验性功能函数迭代器)的变化，导致上述代码不会再有类似的风险，所以`vet`工具不再会报告这些错误。

### append的新警告

现在，像`s = append(s)`这样不产生任何效果的`append`代码，会被`vet`工具报告错误。

### 在difer中错误使用time的警告

参考这样一个例子：

```go
t := time.Now()
defer log.Println(time.Since(t)) // 事实上，time.Since并不会在defer的时候才调用。我们实际defer的是log.Println
tmp := time.Since(t); defer log.Println(tmp) // 同上

defer func() {
  log.Println(time.Since(t)) // 正确的time.Since写法
}()
```

现在，`vet`工具会报告出上述错误的写法。

### 对于log/slog的警告

`slog`库的正确用法是`slog.Info(message, key1, v1, key2, v2)`，如果在key的位置填写的既不是一个`string`，又不是一个`slog.Attr`，现在`vet`工具会报告这个错误。

## 核心库

- 新增了`math/rand/v2`包
- 新增了`go/version`包，用以比较 Go 版本号，例如：`version.Compare("go1.21rc1", "go1.21.0")`
- `net/http.ServeMux`现在有了更多的支持，已经支持了传入请求方法和通配符
- 一些库的小变化
  - `archive/tar`包新增了`Writer.AddFS`方法
  - `archive/zip`包新增了`Writer.AddFS`方法
  - `cmp`包新增了`Or`函数，用以返回一系列变量中的第一个非零值变量
  - `log/slog`包新增了`SetLogLoggerLevel`函数
  - `net/http`包新增了`ServeFileFS`, `FileServerFS`, `NewFileTransportFS`函数
  - `reflect`包新增了`Value.IsZero`方法
  - `slices`包新增了`Concat`函数，用以合并多个切片。`Delete`, `Compact`, `Replace`等函数现在会把切片末尾空出来的位置置为零值。
  - 还有一些影响不大的变化，就不一一列举了
