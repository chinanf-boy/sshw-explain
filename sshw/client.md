## client.go

### package+import
```go
package sshw

import (
	"fmt"
	"io/ioutil"
	"os"
	"os/user"
	"path"
	"time"

	"golang.org/x/crypto/ssh"
	"golang.org/x/crypto/ssh/terminal"
)
```

### 变量

```go
var (
	DefaultCiphers = []string{
		"aes128-ctr",
		"aes192-ctr",
		"aes256-ctr",
		"aes128-gcm@openssh.com",
		"chacha20-poly1305@openssh.com",
		"arcfour256",
		"arcfour128",
		"arcfour",
		"aes128-cbc",
		"3des-cbc",
		"blowfish-cbc",
		"cast128-cbc",
		"aes192-cbc",
		"aes256-cbc",
	}
)

```

### 接口与结构

``` go
type Client interface {
	Login()
}

type defaultClient struct {
	clientConfig *ssh.ClientConfig
	node         *Node
}

```

### NewClient

组织ssh请求的配置

```go
func NewClient(node *Node) Client {
	u, err := user.Current()
	if err != nil {
		l.Error(err)
		return nil
	}

	var authMethods []ssh.AuthMethod

	var pemBytes []byte
	if node.KeyPath == "" { // 读取私钥
		pemBytes, err = ioutil.ReadFile(path.Join(u.HomeDir, ".ssh/id_rsa"))
	} else {
		pemBytes, err = ioutil.ReadFile(node.KeyPath)
	}
	if err != nil {
		l.Error(err)
	} else {
		var signer ssh.Signer
		if node.Passphrase != "" { // 私钥的密码
			signer, err = ssh.ParsePrivateKeyWithPassphrase(pemBytes, []byte(node.Passphrase))
		} else {
			signer, err = ssh.ParsePrivateKey(pemBytes)
		}
		if err != nil {
			l.Error(err)
		} else {
			authMethods = append(authMethods, ssh.PublicKeys(signer))
		}
	}

	password := node.password()  // ssh密码之一

	if password != nil {
		interactive := func(user, instruction string, questions []string, echos []bool) (answers []string, err error) {
			answers = make([]string, len(questions))
			for n := range questions {
				answers[n] = node.Password
			}

			return answers, nil
		}
		authMethods = append(authMethods, ssh.KeyboardInteractive(interactive))
		authMethods = append(authMethods, password)
	}

	config := &ssh.ClientConfig{ // 原生ssh库，客户端配置
		User:            node.user(),
		Auth:            authMethods, // 密钥方法
		HostKeyCallback: ssh.InsecureIgnoreHostKey(),
		Timeout:         time.Second * 10,
	}

	config.SetDefaults()
	config.Ciphers = append(config.Ciphers, DefaultCiphers...) // 加密方式

	return &defaultClient{ // 指针返回
		clientConfig: config,
		node:         node,
	}
}

```

- [l.Error](./log.md#l.Error)

### defaultClient类型的Login指针方法

登录ssh, 新开终端

```go
func (c *defaultClient) Login() {
	host := c.node.Host
	port := c.node.port()
	client, err := ssh.Dial("tcp", fmt.Sprintf("%s:%d", host, port), c.clientConfig)
	if err != nil {
		l.Error(err)
		return
	}
	defer client.Close()

	l.Infof("connect server ssh -p %d %s@%s version: %s\n", port, c.node.user(), host, string(client.ServerVersion()))

	session, err := client.NewSession() // 新开会话
	if err != nil {
		l.Error(err)
		return
	}
	defer session.Close()

	fd := int(os.Stdin.Fd())
	state, err := terminal.MakeRaw(fd) // 终端制作
	if err != nil {
		l.Error(err)
		return
	}
	defer terminal.Restore(fd, state)

	w, h, err := terminal.GetSize(fd)
	if err != nil {
		l.Error(err)
		return
	}

	modes := ssh.TerminalModes{
		ssh.ECHO:          1,
		ssh.TTY_OP_ISPEED: 14400,
		ssh.TTY_OP_OSPEED: 14400,
	}
	err = session.RequestPty("xterm", h, w, modes)
	if err != nil {
		l.Error(err)
		return
	}

	session.Stdout = os.Stdout // 组合下下，ssh 与 终端 3大输出
	session.Stderr = os.Stderr
	session.Stdin = os.Stdin

	err = session.Shell() // 新开
	if err != nil {
		l.Error(err)
		return
	}

	// 一秒时间周期性， 获取 终端 大小
	// fix resize 问题
	go func() {
		var (
			ow = w
			oh = h
		)
		for { // 
			cw, ch, err := terminal.GetSize(fd)
			if err != nil {
				break
			}

			if cw != ow || ch != oh {
				err = session.WindowChange(ch, cw)
				if err != nil {
					break
				}
				ow = cw
				oh = ch
			}
			time.Sleep(time.Second)
		}

	}()

	session.Wait()
}
```