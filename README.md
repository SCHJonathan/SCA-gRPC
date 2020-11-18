# Source Code Adventure - gRPC

> gRPC (gRPC Remote Procedure Calls) is an open source remote procedure call (RPC) system initially developed at Google in 2015. It uses HTTP/2 for transport, Protocol Buffers as the interface description language, and provides features such as authentication, bidirectional streaming and flow control, blocking or nonblocking bindings, and cancellation and timeouts. It generates cross-platform client and server bindings for many languages. Most common usage scenarios include connecting services in microservices style architecture and connect mobile devices, browser clients to backend services.
> (quoted from [Wikipedia](https://en.wikipedia.org/wiki/GRPC))

Hope this brief quote from [Wikipedia](https://en.wikipedia.org/wiki/GRPC) can give you a quick understanding of this popular framework. It's an RPC Framework carrying HTTP/2 proctol with (even bidirectional) data streaming support. Protocol Buffers play the role of IDL, which means you can write your server completely in `Java` andœ the client using `C++` can invoke your service functions like it's written in `C++` too. This framework gets increasingly more attention since the past few years and this article will offer you an objective and detailed breakdown of some important parts of this framework.

## Some Nonsenses

This source code adventure project is aiming at people that have some understanding of the Computer Network, the Distributed System, and some basic Design Patterns. In this source code adventure project, we will enjoy our journey with [Golang](https://golang.org/). I strongly suggest you to be familiar with `Golang`'s `net` library (at least be comfortable with `Golang`). If you are not quite familiar with any of thoses stuffs that I mentioned, you might find this adventure is a little bit tough to go through.

During this adventure, we will not just limit our scope within the gRPC project, whenever we come across some external dependencies or third-party packages that are worth to take look at, we will also stop by and admire the greatness.

## Adventure Started

### 0. What is the RPC and why is it so important?

One of the questions you might ask is why do we even consider using RPC rather than API for the communication between processes or services because API also allow connect different services with different programming languages and dependencies with uniform and lightweight(maybe) proctol.  

Let's first read the definition of RPC from wikipedia.

> In distributed computing, a remote procedure call (RPC) is when a computer program causes a procedure (subroutine) to execute in a different address space (commonly on another computer on a shared network), which is coded as if it were a normal (local) procedure call, without the programmer explicitly coding the details for the remote interaction. That is, the programmer writes essentially the same code whether the subroutine is local to the executing program, or remote. This is a form of client–server interaction (caller is client, executor is server), typically implemented via a request–response message-passing system. In the object-oriented programming paradigm, RPCs are represented by remote method invocation (RMI). The RPC model implies a level of location transparency, namely that calling procedures are largely the same whether they are local or remote, but usually they are not identical, so local calls can be distinguished from remote calls. Remote calls are usually orders of magnitude slower and less reliable than local calls, so distinguishing them is important.
> (quoted from [Wikipedia](https://en.wikipedia.org/wiki/Remote_procedure_call))

The benefits of distributed computing bring more and more people focusing on improving the communication cost among each endpoints in the distributed network. Nowadays, Together with REST, GraphQL, and SOAP, RPC is one of the predominant architectural styles for APIs. 

The most important sentence from above Wikipedia quote is: **RPC is when a computer program causes a procedure (subroutine) to execute in a different address space, which is coded as if it were a normal (local) procedure call**. RPC should abstract this interaction between the **server** and the **client**. That is, the client(calling side) first invokes that remote function call as if calling a local routine. The passing argument and metadata related to the function will be processed by the proxy infrastructure of the RPC framework to convert it into a network request. It then gets directed over the internet under some protocol(HTTP for gRPC) to the server side. On the receiving side, similar infrastructure transforms the received network request back into routine invocation by using the identifier within the request to look up the registered function. Once we got the result, we will again transform the result back into a network request and the client's proxy infrastracture will manage to convert the result request into client's data representation type. As you can see, the proxy infrasctures from both the server and the client side abstract away most network operations, it makes the RPC easier to use compared to other API protocols.

Bringing convenience to the programmers is one of the advantage of the RPC framework. However, it's also one of the disadvantage of the RPC.

### 1. Create three main components using protocol buffer

Just like other RPC frameworks, the core components of the gRPC consist of `Server`, `Client`, and `Service`. gRPC obviously uses Google protocol buffer as the IDL language and I really want to include IDL as one of the greatness of `gRPC` because google protocol buffer is just way better designed compared to other IDLs. Yeah, Apache Thrift, I am talking to you, you prolong crap. (I'm sorry. I will talk more about why I hate Apache Thrift in details later)

We declare the RPC request and  response structure as well as the service functions inside protocol buffer. Followed by running 'protocol' compiler with `grpc` plugin installed will automatically generate the server and client portion of the code. The generated code abstract plenty of details of service discovery, load balancing, etc and we will unveil those beauties step by step.



```go
// Server is a gRPC server to serve RPC requests.
type Server struct {
	opts serverOptions

	mu       sync.Mutex 							// guards following
	lis      map[net.Listener]bool
	conns    map[transport.ServerTransport]bool
	serve    bool
	drain    bool
	cv       *sync.Cond              				// signaled when connections close for GracefulStop
	services map[string]*serviceInfo 				// service name -> service info
	events   trace.EventLog

	quit               *grpcsync.Event
	done               *grpcsync.Event
	channelzRemoveOnce sync.Once
	serveWG            sync.WaitGroup 				// counts active Serve goroutines for GracefulStop

	channelzID int64 								// channelz unique identification number
	czData     *channelzData

	serverWorkerChannels []chan *serverWorkerData
}
```

```go
// ClientConn represents a virtual connection to a conceptual endpoint, to
// perform RPCs.
//
// A ClientConn is free to have zero or more actual connections to the endpoint
// based on configuration, load, etc. It is also free to determine which actual
// endpoints to use and may change it every RPC, permitting client-side load
// balancing.
//
// A ClientConn encapsulates a range of functionality including name
// resolution, TCP connection establishment (with retries and backoff) and TLS
// handshakes. It also handles errors on established connections by
// re-resolving the name and reconnecting.
type ClientConn struct {
	ctx    context.Context
	cancel context.CancelFunc

	target       string
	parsedTarget resolver.Target
	authority    string
	dopts        dialOptions
	csMgr        *connectivityStateManager

	balancerBuildOpts balancer.BuildOptions
	blockingpicker    *pickerWrapper

	mu              sync.RWMutex
	resolverWrapper *ccResolverWrapper
	sc              *ServiceConfig
	conns           map[*addrConn]struct{}
	// Keepalive parameter can be updated if a GoAway is received.
	mkp             keepalive.ClientParameters
	curBalancerName string
	balancerWrapper *ccBalancerWrapper
	retryThrottler  atomic.Value

	firstResolveEvent *grpcsync.Event

	channelzID int64 // channelz unique identification number
	czData     *channelzData

	lceMu               sync.Mutex // protects lastConnectionError
	lastConnectionError error
}
```