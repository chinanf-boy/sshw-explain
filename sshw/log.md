## log.go

### package+import

```go
package sshw

import (
	"fmt"
	"log"
	"os"
)
```

### 接口与结构

```go
type Logger interface {
	Info(args ...interface{})
	Infof(format string, args ...interface{})
	Error(args ...interface{})
	Errorf(format string, args ...interface{})
}

type logger struct{}
```

### 变量

```go
var (
	l      Logger = &logger{}
	stdlog        = log.New(os.Stdout, "[sshw] ", log.LstdFlags)
)
```

### 公开的日志获取/设置函数

```go
func GetLogger() Logger {
	return l
}

func SetLogger(logger Logger) {
	l = logger
}
```

### logger 类型的指针方法们

```go
func (l *logger) Info(args ...interface{}) {
	l.println("[info]", args...)
}

func (l *logger) Infof(format string, args ...interface{}) {
	l.printlnf("[info]", format, args...)
}

```

#### l.Error

```go
func (l *logger) Error(args ...interface{}) {
	l.println("[error]", args...)
}

func (l *logger) Errorf(format string, args ...interface{}) {
	l.printlnf("[level]", format, args...)
}

func (l *logger) println(level string, args ...interface{}) {
	stdlog.Println(level, fmt.Sprintln(args...))
}

func (l *logger) printlnf(level string, format string, args ...interface{}) {
	stdlog.Println(level, fmt.Sprintf(format, args...))
}
```
