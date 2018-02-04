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

### 让程序更灵活些
在上面的例子中，Server端接收Client端发来的数字的时候定义了一个4个字节的buff，那么则限定了Client端发送数据的时候只可以发送4个字节，作为IM怎么可以限制别人想要说的内容长度?

生活中两个人聊天，一个人听另一个讲述的时候，我们会用一个语气停顿的间隔来判断对方说完了一句话，这就像生活中一个默认的“协议”，写一个IM通讯程序同样也需要一个商定好的协议。在文本编辑中通常是以换行来表示结尾，我们也可以借助换行来作为一句话的结束标识。强大的go已经提供了便利的方法来按行读取，官方包中`bufio`提供了一个`ReadLine()`的方法，改造后的代码如下：
```golang
//TcpServer.go
func handleConnection(conn net.Conn) {
	defer conn.Close()
	addr := conn.RemoteAddr()
	rb := bufio.NewReader(conn)
	for {
		// 读取客户端消息
		body, _, err := rb.ReadLine()
		if err != nil {
			panic(err)
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
我们再改造下Client端，让其更灵活的随意输入内容，改造代码如下：
```golang
//TcpClient.go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}

	for {
		rd := bufio.NewReader(os.Stdin)
		msg, _, err := rd.ReadLine()
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
}
```
上述代码中从`os.Stdin`终端接收输入一行数据。此时我们就可以随意输入想要说的内容。


## UDP
TCP是可靠传输，UDP是不可靠传输，再基础不过的知识了。在早些年很多人都是放弃不可靠传输的UDP协议的，然而并不能说他是过时无用的协议了，UDP由于简单的通讯过程相比TCP的三次握手效率带来的效率提升也是可观的。近来把UDP用在生产环境下的场景也很多，比如日志系统收集日志，监控系统收集埋点等在机房内网通讯网络环境较好的情况，丢包的概率几乎可以忽略。当然如果业务逻辑允许的情况下，公网也可以依赖UDP传输，腾讯QQ就是一个很好的例子，还有近来的风口，华为提出的物联网窄带蜂窝网(NB-IOT)所用的CoAP协议也是基于UDP。

随着公网的不断改善，丢包的概率未来有一天也减少到可无视的情况的话，或许UDP协议将会完全代替TCP协议。下面则给一个go下面实现UDP的例子:
### UDP Server
```golang
//UdpServer.go
package main

import (
	"fmt"
	"net"
)

func main() {
	lAddr := net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 8080}
	conn, err := net.ListenUDP("udp", &lAddr)
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	for {
		// 读取消息
		msg := make([]byte, 1024)
		_, rAddress, err := conn.ReadFromUDP(msg)
		if err != nil {
			panic(err)
		}
		fmt.Printf("收到%s消息: %s\n", rAddress.String(), string(msg))
		// 发送消息
		conn.WriteToUDP(msg, rAddress)
		fmt.Printf("发送%s消息: %s\n", rAddress.String(), string(msg))
	}
}

```
在上面的代码中并没有使用到`bufio`包，这个包里并没有提供UDP的读取方式，`ReadLine()`并不能读取UDP协议的数据，这也是由于 UDP的特性所致，UDP没有重发机制、无法保障顺序，链路层数据传输最大的值，也就是MTU值1500字节的限制，如果发送大于1500字节的包，链路层被分包后到达服务端很容易丢弃，无法保障包顺序也就无法实现按顺序读取。

### UDP Client
```golang
//UdpClient.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func main() {
	lAddr := net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 8090}
	rAddr := net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 8080}
	conn, err := net.DialUDP("udp", &lAddr, &rAddr)
	if err != nil {
		panic(err)
	}
	for {
		rd := bufio.NewReader(os.Stdin)
		msg, _, err := rd.ReadLine()
		if err != nil {
			panic(err)
		}
		// 发送消息
		conn.Write(msg)
		fmt.Printf("发送%s消息: %s\n", rAddr.String(), string(msg))
		// 读取消息
		rMsg := make([]byte, 1024)
		_, rAddress, err := conn.ReadFromUDP(rMsg)
		if err != nil {
			panic(err)
		}
		fmt.Printf("收到%s消息: %s\n", rAddress.String(), string(rMsg))
	}
}
```
### 应该怎样选协议？
应该说各有利弊。适合的场景选择适合的协议，如果你的服务跑在一个很稳定的环境下其实UDP则是一个很好的选择，或者丢失一两个包对服务的影响并不大，如视频流传输，丢几个帧并不影响观看。或者从业务层面解决了丢包的影响，自己实现了重试、包顺序等逻辑，如腾讯QQ。或者希望很低的电量消耗，如NB-IOT。而如果你的环境达不到业务的要求，则基于TCP协议上开发会更有保障些。但不管哪个协议GO都提供了简易的API实现。
