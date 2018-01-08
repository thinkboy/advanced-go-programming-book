随着服务端的结构越来越复杂，服务之间通讯也越来越多，基于该场景下对于性能的需求，服务端内部服务常直接采用Socket编程方式来做通讯，Socket连接也常被叫“长连接”、“tcp连接”等。

golang是面向互联网的开发语言，原生包就带有一个名"net"的包抽象成：连接(dial)->读/写(read/write)->关闭(close)这种API模式。先看下一个在golang里的demo：

## TCP
为了后文的扩展，在这里我们先看一个网上常见的最简单的模型，就以IM为场景。
### TCP Server
```golang
//TcpServer.go
package main

import (
	"fmt"
	"net"
)

func main() {
	ln, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}
	for {
		conn, err := ln.Accept()
		if err != nil {
			panic(err)
		}
		// 每个Client一个Goroutine
		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	defer conn.Close()
	var body [4]byte
	addr := conn.RemoteAddr()
	for {
		// 读取客户端消息
		_, err := conn.Read(body[:])
		if err != nil {
			break
		}
		fmt.Printf("收到%s消息: %s\n", addr, string(body[:]))
		// 回包
		_, err = conn.Write(body[:])
		if err != nil {
			break
		}
		fmt.Printf("发送给%s: %s\n", addr, string(body[:]))
	}
	fmt.Printf("与%s断开!\n", addr)
}

```

在上面的Demo里，网络流的读写放在了handleConnection函数里，由于Read方法是阻塞的，为了支持更多的客户端链接同时在线，要为每个客户端链接单独起一个goroutine异步处理，这样借助了golang的“高并发”的特性，并且goroutine是轻量级的，即可以维护大量的客户端链接。注意在函数退出后，说明该链接的读写完成要关闭链接释放资源，此操作就交给`defer conn.Close()`来处理。

### TCP Client

```golang
//TcpClient.go
package main

import (
	"fmt"
	"net"
)

func main() {
	var msg = []byte("abcd")
	conn, err := net.Dial("tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}
	// 发送消息
	_, err = conn.Write([]byte(msg))
	if err != nil {
		panic(err)
	}
	fmt.Println("发送消息: " + string(msg))
	// 读取消息
	_, err = conn.Read(msg[:])
	if err != nil {
		panic(err)
	}
	fmt.Println("收到消息: " + string(msg))
}

```

从另一个角度看相比Server端代码，网络流的读写其实都是一种相同的读写方式。所以在设计长链接的工程中可以不用过多的在意哪边是Server端哪边是Client端，因为对于TCP链接里任何一端都可以主动Write消息或者阻塞Read消息，不要因为两个端的“名词”限制了程序设计的想象力。

首先启动Server端，再启动Client端执行结果如下：

```shell
$ ./TcpServer
收到127.0.0.1:61404消息: abcd
发送给127.0.0.1:61404: abcd
与127.0.0.1:61404断开!
```
```shell
$ ./TcpClient 
发送消息: abc
收到消息: abc
```


## UDP
### UDP Server
TODO
### UDP Client
TODO
