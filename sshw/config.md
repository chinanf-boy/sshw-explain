## config.go

### package + import

``` go
package sshw

import (
	"github.com/go-yaml/yaml"
	"golang.org/x/crypto/ssh"
	"io/ioutil"
	"os/user"
	"path"
)

```

### Node类型

```go
type Node struct {
	Name       string  `json:"name"`
	Host       string  `json:"host"`
	User       string  `json:"user"`
	Port       int     `json:"port"`
	KeyPath    string  `json:"keypath"`
	Passphrase string  `json:"passphrase"`
	Password   string  `json:"password"`
	Children   []*Node `json:"children"`
}

```

### Node类型的方法

```go
func (n *Node) String() string {
	return n.Name
}

func (n *Node) user() string {
	if n.User == "" {
		return "root"
	}
	return n.User
}

func (n *Node) port() int {
	if n.Port <= 0 {
		return 22
	}
	return n.Port
}

func (n *Node) password() ssh.AuthMethod {
	if n.Password == "" {
		return nil
	}
	return ssh.Password(n.Password)
}

```

### config

```go
var (
	config []*Node
)

func GetConfig() []*Node { // 对外函数
	return config
}

```

### LoadConfig

获得当前用户的`.sshw`的yaml类型配置

```go
func LoadConfig() error {
	u, err := user.Current()
	if err != nil {
		return err
	}
	b, err := ioutil.ReadFile(path.Join(u.HomeDir, ".sshw"))
	if err != nil {
		return err
	}

	var c []*Node
	err = yaml.Unmarshal(b, &c) // 配置解析成 Node类型
	if err != nil {
		return err
	}

	config = c // 包级变量

	return nil
}
```