# Source Code Adventure - gRPC

> gRPC (gRPC Remote Procedure Calls) is an open source remote procedure call (RPC) system initially developed at Google in 2015. It uses HTTP/2 for transport, Protocol Buffers as the interface description language, and provides features such as authentication, bidirectional streaming and flow control, blocking or nonblocking bindings, and cancellation and timeouts. It generates cross-platform client and server bindings for many languages. Most common usage scenarios include connecting services in microservices style architecture and connect mobile devices, browser clients to backend services.
> (quoted from [Wikipedia](https://en.wikipedia.org/wiki/GRPC))

Hope this brief quote from [Wikipedia](https://en.wikipedia.org/wiki/GRPC) can give you a quick understanding of this cutie. Yes, it's a RPC Framework with HTTP/2 carrying the transportation. Yes, it supports data streaming, even bidirectional streaming. Yes, it uses Protocol Buffers, which means you can write your server completely in Java and the client using `C++` can call and communicate with your service like it's written in `Java` too. This framework is a heaven for distributed system. So, oh boy, are you ready to go to the heaven? (do you wanna die?)

## Some Nonsenses

This project is aiming at people that have kinda deep understanding of the Computer Network, the Distributed System, and some basic design patterns. In this source code adventure project, we will enjoy our journey with [Golang](https://golang.org/). We will read the source code of gRPC or some other related network packages written in `Golang`. If you are not quite familar with any of those, you might find this adventure is a little bit tough to go through but this actually means you're learning. So.... yeah!

During this adventure, we will not just limit our scope within gRPC project, whenever we come across some external dependencies or thrid party packages that are worth to take look at, we will also stop by and admire the greatness.

## Let's begin our journey, shall we?

Just like other RPC frameworks, the core components of the gRPC consist of `Server`, `Client`, and `Service`. I really want to include IDL because google proctol buffer is just way better designed compared to other IDLs. Yeah, Apache Thrift, I am talking to you, you prolong crap. (I'm sorry and I will talk more about why I hate Apache Thrift in details later)

gRPC obviously uses Google proctol buffer as the IDL language. We declare the RPC request and  reponse structure as well as the service functions inside proctol buffer. Followed by running 'proctol' compiler with `grpc` plugin installed will automatically generate the server and client portion of the code. The generated code abstract plent of details of service discovery, load balancing, etc and we will unveil those beauties step by step.

## 1. We get our server baby.


```go
// Server is a gRPC server to serve RPC requests.
type Server struct {
	opts serverOptions

	mu       sync.Mutex // guards following
	lis      map[net.Listener]bool
	conns    map[transport.ServerTransport]bool
	serve    bool
	drain    bool
	cv       *sync.Cond              // signaled when connections close for GracefulStop
	services map[string]*serviceInfo // service name -> service info
	events   trace.EventLog

	quit               *grpcsync.Event
	done               *grpcsync.Event
	channelzRemoveOnce sync.Once
	serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop

	channelzID int64 // channelz unique identification number
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