参考https://go.dev/doc/go1.20

### 切片转数组

现在支持将切片转为数组了，例如`*(*[4]byte)(x)`现在可以简写为`[4]byte(x)`。

### `unsafe`包新增函数

`unsafe`包提供了三个新函数`SliceData`、`String`、`StringData`，这些函数现在提供了构造和解构切片和字符串值的完整功能。例如：

```go
s := "abc"
buf := unsafe.Slice(unsafe.StringData(s), len(s))
```

就可以得到字符串`s`的底层数组。

> [!NOTE]
> 但在一般情况下，你可以放心的使用`[]byte(s)`。如果编译器检测到后续不会再用到`s`，也会直接把它的底层数组返回出来，而不是复制一份。

### 关于`comparable`约束

```go
a := map[string]any{"a": 1, "b": 2.3, "c": "c"}
b := map[string]any{"a": 1, "b": 2.3, "c": "c"}
fmt.Println(maps.Equal(a, b)) // 输出 true
```

我们来看一下`maps.Equal`函数的定义：

```go
func Equal[M1, M2 ~map[K]V, K, V comparable](m1 M1, m2 M2) bool
```

在之前的版本，由于`maps.Equal`函数接收的两个`map`要求键与值都是`comparable`，但实际上它是`map[string]any`，`any`并不一定满足`comparable`，所以编译会报错。\
在Go1.20之后，不会再因此而编译报错。如果出现不能比较的元素，则会在运行时报错：`panic: runtime error: comparing uncomparable type []int`
