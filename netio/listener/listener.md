# listener

## listerner interface

`pkg/types/network.go`

```go
// listener interface
type Listener interface {
	// Listener name
	Name() string

	// Listener address
	Addr() net.Addr

	// Start starts listener with context
	Start(lctx context.Context)

	// Stop stops listener
	// Accepted connections will not be closed
	Stop()

	// Listener tag,
	ListenerTag() uint64

	// Return a copy a listener fd
	ListenerFD() (uintptr, error)

	// Limit bytes per connection
	PerConnBufferLimitBytes() uint32

	// Set listener event listener
	SetListenerCallbacks(cb ListenerEventListener)

	// Close listener, not closing connections
	Close(lctx context.Context) error
}
```



## listerner struct

`pkg/network/listener.go`

```go
type listener struct {
	name                                  string
	localAddress                          net.Addr
    bindToPort                            bool		//TBD: port or unix socket??
    listenerTag                           uint64	// TBD: what?
	perConnBufferLimitBytes               uint32
	handOffRestoredDestinationConnections bool
	cb                                    types.ListenerEventListener // Listener Callback
	rawl                                  *net.TCPListener
	logger                                log.Logger
	tlsMng                                types.TLSContextManager
}
```

### func Start()

```go
func (l *listener) Start(lctx context.Context) {
    // 开始监听端口
    if err := l.listen(lctx); err != nil {
    }

    if err := l.accept(lctx); err != nil {
    }
}

func (l *listener) listen(lctx context.Context) error {
	rawl, err = net.ListenTCP("tcp", l.localAddress.(*net.TCPAddr))
	l.rawl = rawl
}

func (l *listener) accept(lctx context.Context) error {
    // 在端口上做accept
	rawc, err := l.rawl.Accept()

	go func() {
		if l.tlsMng != nil && l.tlsMng.Enabled() {
            // 加密处理
			rawc = l.tlsMng.Conn(rawc)
		}

        // 成功则发送OnAccept事件，通知 ListenerEventListener
		l.cb.OnAccept(rawc, l.handOffRestoredDestinationConnections, nil)
	}()
}
```

func Close()

```go
func (l *listener) Close(lctx context.Context) error {
    // 发送OnClose事件，通知 ListenerEventListener
	l.cb.OnClose()
    // 关闭底层TCPListener
	return l.rawl.Close()
}
```



