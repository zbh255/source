# The LittleRpc Handout - 从零开始的高性能RPC框架设计与实现

## Self-Introduction

`rainstorm`，学生，开源爱好者

## Project-Introduction

`LittleRpc`是临时起意的项目，它的最初想法来自于在学校时我刚刚写完`ddio`之后，想要研究`RPC`的底层原理的想法，看了一些其它的实现确总是感觉少了一些拼图，最后决定自己造一个。最初的设计很简单且没有任何用处，`v0.10`设计了类`thrift`协议。随着8个月的迭代和打磨，一些想法逐渐成熟起来

[Readme.md](https://github.com/nyan233/littlerpc/blob/main/README.md)

## Example

[link](https://github.com/nyan233/littlerpc/tree/main/example)

## Benchmark

> `benchmark`因为模拟的负载比较死板，所以大家看个乐就好

`Platfrom`

```go
Server
CPU 		: AMD EPYC 7T83 16Core
Memory  	: 16GB * 4 ECC
Network 	: 7.5G
NumaNode	: 0~0

Client
CPU 		: AMD EPYC 7T83 16Core
Memory  	: 16GB * 4 ECC
Network 	: 7.5G
NumaNode	: 0~0
```

`Mock 10us`

<img src="https://raw.githubusercontent.com/zbh255/source/main/rpc-bench1.svg" alt="svg"  />

## Transport Layer

- [`Reacotr`](https://en.wikipedia.org/wiki/Reactor_pattern)

- [`Proactor`](https://en.wikipedia.org/wiki/Proactor_pattern)

> 传输层的设计参照了`Reactor`的处理模式，同时设计了一个可以让`RPC`框架来处理读事件的接口，避免了一次内存拷贝。
>
> 触发`OnMessage`时的数据已经在网络框架里经过一次拷贝了，性能方面不如`OnRead`好，不过兼容性方面要比`OnRead`要好，`OnRead`通常需要编写与框架配对的读取函数。
>
> 注：`OnRead`这层优化通常在小数据量下并没有效果

```go
type EventDriveInter interface {
	OnRead(func(conn ConnAdapter))
	OnMessage(func(conn ConnAdapter, data []byte))
	OnOpen(func(conn ConnAdapter))
	OnClose(func(conn ConnAdapter, err error))
}

type ConnAdapter interface {
	// SetSource Source主要用于server/client将连接的相关资源
	// 比如: parser/writer/buffer等附加到conn中, 可以降低map查找的开销
	SetSource(s interface{})
	Source() interface{}
	// Close 不管因为何种原因导致了连接被关闭, ServerTransportBuilder设置的OnClose
	// 应该被调用, 从而让LittleRpc能够清理残余数据
	Close() error
	// Conn 其它的接口遵循net.Conn的定义
	net.Conn
}
```

> `Server`/`Client`用的都是同一套抽象[code](https://github.com/nyan233/littlerpc/blob/main/core/common/transport/transport.go)，有助于复用代码

```go
type ServerBuilder interface {
	Server() ServerEngine
	EventDriveInter() EventDriveInter
}

type ClientBuilder interface {
	Client() ClientEngine
	EventDriveInter() EventDriveInter
}
```

> 客户端多了一个新建连接的接口，因为客户端往往要主动创建连接

```go
NewConn(NetworkClientConfig) (ConnAdapter, error)
```

> `nbio`/`ddio`这些都是`Reacotr`模型的网络框架，为了将标准库接入，所以将标准库模拟成`Reacotr`使用[link](https://github.com/nyan233/littlerpc/blob/main/core/common/transport/std_tcp.go)

## Message Design

- [lmsg](https://github.com/nyan233/littlerpc/blob/main/core/protocol/message/message.go)

- [mux](https://github.com/nyan233/littlerpc/blob/main/core/protocol/message/mux/mux.go)

  > `mux-block`中存储`msgId`是为了能够将后续的数据拼接起来

### Head Block

![svg](https://raw.githubusercontent.com/zbh255/source/main/head-block.svg)

> `mux`协议的实现使得在`stream`中大块消息的传输对小块消息的延迟影响降低，默认的`parser`允许你混用`mux`&`no-mux`消息，这意味着你可以只为一些较大的请求选用`mux`

## Message Parser Design

消息解析器的设计, 消息解析器的目标是可以适配不同的`RPC`协议, 同时以高性能达成这个目标。

```go
type Parser interface {
	// ParseOnReader 用于处理读事件可减少一次memcopy, reader返回的错误会停止解析, 所以
	// reader是包装了非阻塞的syscall的话不应该返回(syscall.EWOULDBLOCK | syscall.EINTR)等相关的错误
	ParseOnReader(reader func([]byte) (n int, err error)) (msgs []ParserMessage, err error)
	// Parse 处理数据的接口必须能够正确处理half-package
	// 也必须能处理有多个完整报文的数据, 在解析失败时返回对应的error
	Parse(data []byte) (msgs []ParserMessage, err error)
	// FreeMessage 用于释e返回的数据, 在Parse返回error时这个过程
	// 绝对不能被调用
	FreeMessage(msg *message.Message)
	FreeContainer(c []ParserMessage)
	inters.Reset
}
```

### FSM Design

`LittleRpc`默认实现了一个称为`trait parser`的`message parser`，也就是将消息开头的`1 B`当做一个消息的特征，以下是每个阶段所做的事情和`FSM`

- `init`识别特征并调用注册好的`Handler`
- `parse1`解析出消息的基本信息，比如长度/id等
  - `single parse`是为了给一些没有标明长度或很难识别长度的消息使用的，这类消息的底层承载通常是`HTTP`/`WebSocket`这类协议，不需要考虑半包
- `parse2`解析出完整消息，并根据`mux`/`no-mux`分别进行处理，一些并不是完整长度的`mux`消息需要暂存起来等待下一个`Block`到达，所有`Block`到达以后即可拼接成完整的消息。`no-mux`消息在完成`parse2`之后就是一个完整的消息

![fsm](https://raw.githubusercontent.com/zbh255/source/main/lrpc-parser-fsm.svg)

### Slow Connection

![svg](https://raw.githubusercontent.com/zbh255/source/main/slow-conn2.svg)

> 慢连接的情况下，如果`Parser`不是被设计为可暂停且可恢复最近状态的话，那么就会影响到在事件循环中的其它连接，即使是可暂停的`Parser`但是每次重启都要重头开始的话也会一定情况下影响效率

### Overflow Check

> `rubust`的保证之一，序列化/反序列化需要针对各种输入进行溢出检查，`parser`能够在错误时清理资源，并且这个不能影响到其它逻辑

[code](https://github.com/nyan233/littlerpc/blob/main/core/protocol/message/serialization.go)

### Json-RPC2 Adapt

> `jsonrpc-2.0`的支持并不是为性能而生的，只是为了调试方便

- [`Protocol`](http://wiki.geekdream.com/Specification/json-rpc_2.0.html)
- [`Parser`](https://github.com/nyan233/littlerpc/blob/main/core/common/msgparser/jsonrpc2_handler.go)
- [`Writer`](https://github.com/nyan233/littlerpc/blob/main/core/common/msgwriter/writer_jsonrpc2.go)

### Benchmark

> [`code`](https://github.com/nyan233/littlerpc/blob/main/core/common/msgparser/bench_test.go)中有很多暂停/启动计时器的代码，所以真实世界中的性能是要比`benchmark`中的表现要优异的
>
> 注：现实中的数据包大小都是比较随机的，以框架的视角来看。在高并发下只要`profile`里`parse time`占比控制在一个合理的范围内即可

```go
b.StopTimer()
for _, v := range parseMsgs {
    parser.FreeMessage(v.Message)
}
parser.FreeContainer(parseMsgs)
b.StartTimer()
```

`Small <= 50`

```go
goos: windows
goarch: amd64
pkg: github.com/nyan233/littlerpc/core/common/msgparser
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkParser
BenchmarkParser/LRPCProtocol-OneParse-1Message
BenchmarkParser/LRPCProtocol-OneParse-1Message-10                 952580
              1270 ns/op         456.56 MB/s        952580 RunCount          336
 B/op         13 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-4Message
BenchmarkParser/LRPCProtocol-OneParse-4Message-10                 555918
              2496 ns/op         810.50 MB/s        555918 RunCount          960
 B/op         38 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-16Message
BenchmarkParser/LRPCProtocol-OneParse-16Message-10                169754
              6727 ns/op         962.14 MB/s        169754 RunCount         4531
 B/op        155 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-64Message
BenchmarkParser/LRPCProtocol-OneParse-64Message-10                 43791
             24468 ns/op        1194.60 MB/s         43791 RunCount        19012
 B/op        682 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-256Message
BenchmarkParser/LRPCProtocol-OneParse-256Message-10                12184
            100493 ns/op        1192.98 MB/s         12184 RunCount        78170
 B/op       2918 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-1024Message
BenchmarkParser/LRPCProtocol-OneParse-1024Message-10                2664
            497628 ns/op         950.28 MB/s          2664 RunCount       312600
 B/op      11683 allocs/op
PASS
```

`Medium <= 500`

```go
goos: windows
goarch: amd64
pkg: github.com/nyan233/littlerpc/core/common/msgparser
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkParser
BenchmarkParser/LRPCProtocol-OneParse-1Message
BenchmarkParser/LRPCProtocol-OneParse-1Message-10                 723850
              2690 ns/op         643.98 MB/s        723850 RunCount         1712
 B/op         17 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-4Message
BenchmarkParser/LRPCProtocol-OneParse-4Message-10                 206637
              6018 ns/op        1231.71 MB/s        206637 RunCount         3802
 B/op         44 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-16Message
BenchmarkParser/LRPCProtocol-OneParse-16Message-10                 64680
             17869 ns/op        1971.36 MB/s         64680 RunCount        18075
 B/op        178 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-64Message
BenchmarkParser/LRPCProtocol-OneParse-64Message-10                 23430
             49992 ns/op        2690.37 MB/s         23430 RunCount        70509
 B/op        717 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-256Message
BenchmarkParser/LRPCProtocol-OneParse-256Message-10                 6163
            198202 ns/op        2957.98 MB/s          6163 RunCount       300419
 B/op       2980 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-1024Message
BenchmarkParser/LRPCProtocol-OneParse-1024Message-10                1141
            916712 ns/op        2535.26 MB/s          1141 RunCount      1198164
 B/op      11808 allocs/op
PASS
```

`Large <= 5000`

```go
goos: windows
goarch: amd64
pkg: github.com/nyan233/littlerpc/core/common/msgparser
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkParser
BenchmarkParser/LRPCProtocol-OneParse-1Message
BenchmarkParser/LRPCProtocol-OneParse-1Message-10                 429360
              5139 ns/op        4499.36 MB/s        429360 RunCount        10853
 B/op         12 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-4Message
BenchmarkParser/LRPCProtocol-OneParse-4Message-10                  80762
             14432 ns/op        5256.86 MB/s         80762 RunCount        33658
 B/op         41 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-16Message
BenchmarkParser/LRPCProtocol-OneParse-16Message-10                 20949
             64842 ns/op        4892.87 MB/s         20949 RunCount       147633
 B/op        180 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-64Message
BenchmarkParser/LRPCProtocol-OneParse-64Message-10                  3222
            351254 ns/op        3798.79 MB/s          3222 RunCount       687040
 B/op        742 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-256Message
BenchmarkParser/LRPCProtocol-OneParse-256Message-10                  788
           1405583 ns/op        3672.61 MB/s           788.0 RunCount    2646963
 B/op       2972 allocs/op
BenchmarkParser/LRPCProtocol-OneParse-1024Message
BenchmarkParser/LRPCProtocol-OneParse-1024Message-10                 178
           6514426 ns/op        3201.06 MB/s           178.0 RunCount   10654489
 B/op      11791 allocs/op
```



## LoadBalancer Design

[code](https://github.com/nyan233/littlerpc/tree/main/core/client/loadbalance)

> `wait-free bounded`实现的负载均衡器，不包含`consistent-hash`，`consistent-hash`采用了缓存

### Benchmark

![svg](https://raw.githubusercontent.com/zbh255/source/main/rpc-loadbalance.svg)

## reflect.Value.Call Optimit

[code](https://github.com/nyan233/littlerpc/blob/main/internal/reflect/call_test.go)

```shell
// gcflags "-l"
goos: windows
goarch: amd64
pkg: github.com/nyan233/littlerpc/internal/reflect
cpu: 12th Gen Intel(R) Core(TM) i7-12700H
BenchmarkCall
BenchmarkCall/ReflectCall
BenchmarkCall/ReflectCall-10             2636827               406.6 ns/op
BenchmarkCall/MethodValueCall
BenchmarkCall/MethodValueCall-10         6254766               194.1 ns/op
BenchmarkCall/AnonymousFunctionCall
BenchmarkCall/AnonymousFunctionCall-10          551793904                2.114 ns/op
BenchmarkCall/UnsafeCall
BenchmarkCall/UnsafeCall-10                     990944419                1.205 ns/op
BenchmarkCall/FunctionCall
BenchmarkCall/FunctionCall-10                   1000000000               0.9373ns/op
PASS
```

> 第一套方案，根据in/out list将对应的方法通过`hack`的方式替换成匿名函数，就能获得极大的性能提升。但是因为`in/out list`排列众多，会造成识别困难。又因为众多的排列造成实际生成的函数列表很大，`swtich`匹配开销很大，而`RPC`面对的目标函数实际是随机的，根据客户端的请求而定，这样就会造成分支预测算法的效果不好，`IPC`下降。不是一套好的方案

```go
type CallxC  func(r unsafe.Pointer,context.Context,args ...unsafe.Pointer) (...unsafe.Pointer,error)
type CallxCS func(r unsafe.Pointer,context.Context,stream.LStream,args ...unsafe.Pointer) (...unsafe.Pointer,error)
type CallxS  func(r unsafe.Pointer,stream.LStream,args ...unsafe.Pointer) (...unsafe.Pointer,error)
type Callx   func(r unsafe.Pointer,args ...unsafe.Pointer) (...unsafe.Pointer,error)
```

> 第二套方案，`Hijacker`，劫持器是在初始化时可以被用户设置到对应的示例中。在劫持器中可以将`req`数据绑定到设定的参数中。以下是一个案例和劫持器的挂载点。虽然比第一套方案侵入性强，但是理论性能也更高，也更加容易适配。作为一个可选选项存在是合理的

```go
func (t *HelloTest) Setup() {
	err := t.HijackProcess("GetCount", func(stub *server2.Stub) {
		assert.NoError(t.t, stub.Write(atomic.LoadInt64(&t.count)))
		assert.NoError(t.t, stub.Write(nil))
		assert.NoError(t.t, stub.WriteErr(nil))
	})
	assert.NoError(t.t, err)
	err = t.HijackProcess("WaitSelectUserHijack", func(stub *server2.Stub) {
		var uid int
		assert.NoError(t.t, stub.Read(&uid))
		// wait
		<-stub.Done()
		user, _, err := t.SelectUser(stub, uid)
		assert.NoError(t.t, stub.Write(&user))
		assert.NoError(t.t, stub.WriteErr(err))
	})
	assert.NoError(t.t, err)
}
```

```go
// Process v0.40从Method更名到Process
// 过程的名字更适合同时描述方法和函数
type Process struct {
	Value reflect.Value
	// v0.4.6开始, 事先准备好的Args List, 避免查找元数据的开销
	// client不包含opts -> []CallOption
	ArgsType []reflect.Type
	// v0.4.6开始, 实现准备好的Results List, 避免查找元数据的开销
	// 不包含err -> error
	ResultsType []reflect.Type
	// 用于复用输入参数的内存池
	Pool sync.Pool
	// 是否为匿名函数, 匿名函数不带接收器
	// TODO: v0.4.0按照Service&Source为维度管理每个API, 这个字段被废弃
	// AnonymousFunc bool
	// 和Stream一起在注册时被识别
	// 是否支持context的传入
	SupportContext bool
	// 是否支持stream的传入
	SupportStream bool
	// Option中的参数都是用于定义的, 在Process中的其它控制参数用户
	// 并不能手动控制
	Option ProcessOption
	// 过程是否被劫持, 被劫持的过程会直接调用劫持器
	Hijack bool
	// type -> *func(stub *server.Stub)
	Hijacker unsafe.Pointer
}
```



