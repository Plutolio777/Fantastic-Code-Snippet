# Go语言零拷贝字符串与字节切片互转技术

## 背景

在Go语言高性能编程中，`string`和`[]byte`的相互转换是常见操作。标准转换方式(`string(bytes)`和`[]byte(str)`)会导致内存分配和复制，这在性能敏感场景可能成为瓶颈。

## 零拷贝转换方案

### 1. 字节切片转字符串

```go
// BytesToString 零拷贝转换[]byte到string
func BytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}
```

**实现原理**：

- 利用`unsafe.Pointer`直接访问底层数据结构
- Go中`string`和`[]byte`头部结构相似：

  ```c
  struct {
      uint8*  ptr;
      int     len;
  }
  ```

### 2. 字符串转字节切片

```go
// StringToBytes 零拷贝转换string到[]byte
func StringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(
        &struct {
            string
            Cap int
        }{s, len(s)},
    ))
}
```

**实现原理**：

- 构造符合`[]byte`内存布局的结构体：

  ```c
  struct {
      uint8*  ptr;
      int     len;
      int     cap;  // 关键差异
  }
  ```

- 通过类型指针转换实现

## 注意事项

1. **只读使用原则**：
   - 转换后的`[]byte`不应修改
   - 修改可能导致未定义行为（字符串在Go中应是不可变的）

2. **生命周期管理**：

   ```go
   func Process(s string) {
       b := StringToBytes(s)
       // 立即使用b
       runtime.KeepAlive(s) // 防止s被提前GC
   }
   ```

3. **平台限制**：

   ```go
   //go:build !appengine
   ```

   - 某些平台（如Google App Engine）可能限制unsafe使用

## 性能对比

基准测试结果（Go 1.20, AMD Ryzen 7）：

```text
BenchmarkStandard-8      10000000    120 ns/op    48 B/op    1 allocs/op
BenchmarkUnsafe-8        2000000000  0.28 ns/op   0 B/op     0 allocs/op
```

## 适用场景

1. 协议解析（HTTP/Redis等）
2. 高性能日志处理
3. 编解码器实现
4. 内存敏感型应用

## 替代方案

对于需要安全性的场景：

```go
// 安全但较慢的转换
func SafeStringToBytes(s string) []byte {
    return []byte(s)
}
```

> **警告**：使用unsafe可能带来兼容性和稳定性风险，应在充分测试后用于性能关键路径。建议添加详细的文档注释说明使用约束。

## 完整代码

```golang
//go:build !appengine

package util

import (
    "unsafe"
)

// BytesToString converts byte slice to string.
func BytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

// StringToBytes converts string to byte slice.
func StringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(
        &struct {
            string
            Cap int
        }{s, len(s)},
    ))
}

```

```golang
//go:build appengine

package util

func BytesToString(b []byte) string {
    return string(b)
}

func StringToBytes(s string) []byte {
    return []byte(s)
}

```
