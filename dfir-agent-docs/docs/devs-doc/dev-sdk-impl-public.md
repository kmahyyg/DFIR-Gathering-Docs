---
sidebar_position: 4
---

# SDK 文档 - 公共函数与数据结构

## 公共数据结构

输出报错使用 WrappedError 结构，收集输出使用 CollectOutput 结构。其他参见示例插件。

```go
// github.com/kmahyyg/DFIR-Gathering/pkg/abstract
package abstract

type WrappedError struct {
	Origin      string
	PluginName  string
	ErrOriginal error
}

type CollectOutput struct {
    OutputBytes         []byte
    OriginalLogLocation string
    PluginName          string
    OutputPrefix        string
}
```

## 公共工具函数

新建 Logger 日志记录与输出：

```go
// github.com/sirupsen/logrus
// github.com/kmahyyg/DFIR-Gathering/pkg/logging
package logging

func NewLogger(prefix string) *logrus.Entry {}
```

获得的 Logger 可以直接使用 `.Info()` `.Fatal()` `.Warn()` 方法打印不同级别的日志信息。


随机数据和随机字符串的生成：
```go
// github.com/kmahyyg/DFIR-Gathering/utils
package utils

func GenerateRandom(size int) []byte {}
func GenerateRandomStr(n int) string {}
```
