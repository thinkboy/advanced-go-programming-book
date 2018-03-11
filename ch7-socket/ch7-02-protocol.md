# 常见协议

客户端与服务端之间的通讯，总要有共同的语言来"沟通"，就像中国人都懂普通话，就可以轻松聊天。通讯双方必须有一个共同协议才能知道对方到底想做什么，协议越完善表达的事情就越多。开发中常用类似Json、Xml、PB(Protocol Buffers)、以及简易方便的应用在Redis上的RESP(Redis Serialization Protocol)协议。

在上一节中重点围绕传输层协议介绍，在本节我们将围绕应用层协议开展。设计一个好的Socket服务必须要有一个合适协议，在上一节中有用到`ReadLine()`方法，协议商定以\n作为结束就是一个简单的应用协议，下面介绍几个常见协议。

### Xml协议

似乎已经是很古老的协议了，在目前看来几乎很少应用再使用了，笨重的体型已经迎合不了现在的互联网时代，在这里就不过多介绍了，建议不再使用。
### Json协议

一个简单的Json协议如下：

```Json
{
    "op": "set",
    "name": "zhangsan"
}
```

优点：可读性好，扩展性好，通过协议内容就可以很清晰的知道要做什么。对于快速迭代的Web业务很青睐。

缺点：编码较为复杂(因为我们最终要把Json翻译长字节流在网络上传输)，耗损CPU、内存较高，对于高性能的简易的内部服务不适用。

当今通讯最流行的协议，相信作为开发人员无人不知了，在这里也不过多讲述。

### RESP协议(Redis)

Redis作为一个常用的cache服务，不管大小公司都会大量使用，会使用Redis也成为了每个开发人员的技能标配。在各大语言下都有设计完善的SDK包，并且Redis的通讯协议设计也极为简单，开发者可以很快的接入，因此近几年很多造的轮子为了希望可以让用户快速的接受，把自己的服务协议选型使用Redis的通讯协议。例如：Codis、LedisDB。

RESP协议支持多个类型，这里只拿比较有代表意义的**Arrays**类型举例。我们输入Redis命令：`SET name zhangsan`，实际协议格式如下：

```Redis
*3
$3
SET
$4
name
$8
zhangsan
```
在网络上传输的字节流样式如下：

```Redis
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$8\r\nzhangsan\r\n
```

可以思考：我们服务端读取的只是个网络流中，并不知道网络流中有多少字节，也不知道哪几个字节是一个组合，另外一条完整的指令也不一定是一个包，我们需要依据设定的好的协议逐个顺序读取网络字节流。RESP协议通过如下方式如下：

第一行："\*3"表示协议的长度参数，"*"后面跟数字，"3"指本条指令有三个参数。

第二行："$3"表示参数长度，"$"后面跟数字，表示下面这个参数长度为3 Bytes。

第三行："SET"指令字符，就是第二行指定3个Bytes的参数（指令也是一个参数）。

下面几个行同第二、第三行一样。我们基于RESP协议做一个简单的"Redis"通讯：

```golang
// Server.go
package main

import (
	"bufio"
	"bytes"
	"fmt"
	"net"
	"strconv"
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
	for {
		cmd, err := read(conn)
		if err != nil {
			break
		}
		fmt.Printf("收到指令: %s\n", cmd)
		conn.Write([]byte("ok"))
	}
	fmt.Println("client 退出")
}

func read(conn net.Conn) (string, error) {
	var buf = bufio.NewReaderSize(conn, 128)
	// 读取元素个数
	p, _, err := buf.ReadLine() // *3
	if err != nil {
		return "", err
	}
	ln, _ := strconv.Atoi(string(p[1]))
	arr := [][]byte{}
	for i := 0; i < ln; i++ {
		// 读取长度
		_, err = readLen(buf) // $3
		if err != nil {
			return "", err
		}
		// 读取内容
		p, _, err = buf.ReadLine() // name
		if err != nil {
			return "", err
		}
		arr = append(arr, p)
	}
	return string(bytes.Join(arr, []byte(" "))), nil
}

func readLen(b *bufio.Reader) (int, error) {
	ls, _, err := b.ReadLine()
	if err != nil {
		return 0, err
	}
	length, _ := strconv.Atoi(string(ls[1:]))
	return length, nil
}
```

在协议中，每个字段之间以`\r\n`作为间隔，如同第一节中提到的`ReadLine()`用法，每一行表示一个字段，在Redis的指令中参数个数是不固定的，要表述出多少个字段的意义，那么就需要记录有多少个参数，在第一行的参数里就说明了参数个数，这样服务端就知道了这条指令的长度。上述代码`read`方法里第一步需要先读取个数。并在后续的for循环里逐个读取参数。

```golang
// Client.go
package main

import (
	"fmt"
	"net"
)

func main() {
	var cmd = []byte("*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$8\r\nzhangsan\r\n")
	conn, err := net.Dial("tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}

	// 发送消息
	_, err = conn.Write([]byte(cmd))
	if err != nil {
		panic(err)
	}
	fmt.Println("发送消息: \n" + string(cmd))

	// 读取消息
	var resp [2]byte
	_, err = conn.Read(resp[:])
	if err != nil {
		panic(err)
	}
	fmt.Println("收到消息: " + string(resp[:]))
}
```
总结RESP协议的`Arrays`类型提供了三个元素：协议头、协议体以及协议字段分割位置。这样就可以简单解析一个多参数意义的协议。

#### 精简一下RESP

RESP协议里利用了大量的`$`、`\r\n`占据许多的字节数，这些符号除了帮助我们解析协议外其实并没有为我们服务提供什么有用信息，用过Redis的都知道`MSET`命令可以有多个字段，那么这些符号能占到几十到几百的字节，在每个字段前面其实已经有了一个以$开头的数字告诉我们字段的长度，但是如果把这些符号去掉的话，如下：
```Redis
33SET4name8zhangsan
```
想要读取的话，还需要限定一下每个表示字段长度的参数必须是1个字节，如果是不固定的话，解析协议的服务无法知道哪个字节是表示参数长度哪个字节是表示参数内容。还有一个字节的数字只能表示到9，似乎并不够用，因为这是按字符串解析。但是我们知道一个字节可以表示255个数字，如果改造为用字节表示数字，同是一个字节表示的长度会更长，对于常用的kv已经足够用了，如果你的服务需要单个字段里存储更长的内容，可以用两个或三个字节。

为什么RESP不更精简下呢？Redis多用于内网环境下的通讯，现在的服务器都是千兆万兆的网卡，对于协议里这点流量属于微乎其微，不如直接提供一个可读性好一些的协议，在这里介绍精简方法只为提供一个设计思路。

如果要设计的是一个公网的服务呢？面向移动端的用户，流量是用户最关心的，每个字节都用是Money，就很有必要花些心思扣这几个字节。下面就介绍一个以低带宽、低流量为宣传的协议：MQTT协议。

### MQTT协议
TODO
