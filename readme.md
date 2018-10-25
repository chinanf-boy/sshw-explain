# yinheli/sshw [![explain]][source] [![translate-svg]][translate-list]

<!-- [![size-img]][size] -->

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list

「 ssh 客户端包装器,用于自动登录. 」

---

## explain ✅

<!-- doc-templite START generated -->
<!-- time = '2018-10-16' -->
<!-- name = 'yinheli' -->
<!-- repo = 'sshw' -->
<!-- commit = '54736d83523db4c394c13fc08d98e6028e11f770' -->

| 版本     | 与日期        | 最新更新   | 更多               |
| -------- | ------------- | ---------- | ------------------ |
| [commit] | ⏰ 2018-10-16 | ![version] | [源码解释][source] |

[commit]: https://github.com/yinheli/sshw/tree/54736d83523db4c394c13fc08d98e6028e11f770
[version]: https://img.shields.io/npm/v/sshw.svg

<!-- doc-templite END generated -->

### 中文

[中文Readme.md](zh.md)

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---


## 安装

``` 
go get -u github.com/yinheli/sshw/cmd/sshw
```

## cmd/sshw

### 目录

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [package + import](#package--import)
- [常量+变量](#%E5%B8%B8%E9%87%8F%E5%8F%98%E9%87%8F)
- [main](#main)
- [choose](#choose)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### package + import
``` go
package main

import (
	"flag"
	"fmt"
	"os"
	"runtime"
	"strings"

	"github.com/manifoldco/promptui" // 终端 提示输入
	"github.com/yinheli/sshw"
)

```

### 常量+变量

```go
const prev = "-parent-"

var (
	Build = "devel"
	V     = flag.Bool("version", false, "show version")
	H     = flag.Bool("help", false, "show help")

	log = sshw.GetLogger()

	templates = &promptui.SelectTemplates{
		Label:    "✨ {{ . | green}}",
		Active:   "➤ {{ .Name | cyan }} {{if .Host}}{{if .User}}{{.User | faint}}{{`@` | faint}}{{end}}{{.Host | faint}}{{end}}",
		Inactive: "  {{.Name | faint}} {{if .Host}}{{if .User}}{{.User | faint}}{{`@` | faint}}{{end}}{{.Host | faint}}{{end}}",
	}
)
```

- `&promptui.SelectTemplates`: 终端 提示模版

### main

``` go
func main() {
	flag.Parse()
	if !flag.Parsed() {
		flag.Usage()
		return
	}

	if *H { // help 输出
		flag.Usage()
		return
	}

	if *V { // 版本输出
		fmt.Println("sshw - ssh client wrapper for automatic login")
		fmt.Println("  git version:", Build)
		fmt.Println("  go version :", runtime.Version())
		return
	}

	err := sshw.LoadConfig() // 加载配置
	if err != nil {
		log.Error("load config error", err)
		os.Exit(1)
	}

	node := choose(nil, sshw.GetConfig()) // 选择 ssh 其中的一个配置
	if node == nil {
		return
	}

	client := sshw.NewClient(node) // 准备发送ssh登录请求
	client.Login() // 登录
}
```

- [LoadConfig](./sshw/config.md#loadconfig)
- [x] [NewClient](./sshw/client.md#NewClient)
- [x] [Login](./sshw/client.md#Login)

### choose

主要使用，promptui库 做到用户能自由选择ssh配置

```go
func choose(parent, trees []*sshw.Node) *sshw.Node {
	prompt := promptui.Select{
		Label:     "select host",
		Items:     trees, // sshw.Node 类型数组
		Templates: templates,
		Size:      20,
		Searcher: func(input string, index int) bool {
			node := trees[index]
			content := fmt.Sprintf("%s %s %s", node.Name, node.User, node.Host)
			if strings.Contains(input, " ") {
				for _, key := range strings.Split(input, " ") {
					key = strings.TrimSpace(key)
					if key != "" {
						if !strings.Contains(content, key) {
							return false
						}
					}
				}
				return true
			}
			if strings.Contains(content, input) {
				return true
			}
			return false
		},
	}
	index, _, err := prompt.Run()
	if err != nil {
		return nil
	}

	node := trees[index]
	if len(node.Children) > 0 {
		first := node.Children[0]
		if first.Name != prev {
			first = &sshw.Node{Name: prev}
			node.Children = append(node.Children[:0], append([]*sshw.Node{first}, node.Children...)...)
		}
		return choose(trees, node.Children)
	}

	if node.Name == prev {
		return choose(nil, parent)
	}

	return node
}
```