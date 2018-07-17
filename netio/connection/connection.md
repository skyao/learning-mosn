# connection

## connection interface

`pkg/types/network.go`

```go
type Connection interface {
	Id() uint64
	Start(lctx context.Context)
	Write(buf ...IoBuffer) error
	Close(ccType ConnectionCloseType, eventType ConnectionEvent) error
    //忽略部分方法

	AddConnectionEventListener(cb ConnectionEventListener)
	AddBytesReadListener(cb func(bytesRead uint64))
	AddBytesSentListener(cb func(bytesSent uint64))
    
	FilterManager() FilterManager
}
```

## connection struct

```go
type connection struct {
	id         uint64
	localAddr  net.Addr
	remoteAddr net.Addr
    rawConnection        net.Conn
    
    connCallbacks        []types.ConnectionEventListener
	bytesReadCallbacks   []func(bytesRead uint64)
	bytesSendCallbacks   []func(bytesSent uint64)
	filterManager        types.FilterManager
    
    readBuffer          *buffer.IoBufferPoolEntry
	writeBuffer         *buffer.IoBufferPoolEntry
}
```

### func NewServerConnection()

```go
func NewServerConnection(rawc net.Conn, stopChan chan bool, logger log.Logger) types.Connection {
	id := atomic.AddUint64(&idCounter, 1)

	conn := &connection{
		id:               id,
		rawConnection:    rawc,
		localAddr:        rawc.LocalAddr(),
		remoteAddr:       rawc.RemoteAddr(),
		......
	}
	// 创建当前连接的filterManager
	conn.filterManager = newFilterManager(conn)
	return conn
}

func newFilterManager(conn types.Connection) types.FilterManager {
	return &filterManager{
		conn:              conn,
		upstreamFilters:   make([]*activeReadFilter, 0, 32),
		downstreamFilters: make([]types.WriteFilter, 0, 32),
	}
}
```

start()方法：

start方法实际干了两个事情：startReadLoop和startWriteLoop。简化后的代码逻辑为：

```go
func (c *connection) Start(lctx context.Context) {
	c.startOnce.Do(func() {
		c.internalLoopStarted = true
        c.startReadLoop()
        c.startWriteLoop()
    })
}
```

startReadLoop()方法：

```go
func (c *connection) startReadLoop() {
	for {
		......
		select {
		case <-c.stopChan:
			return
		case <-c.internalStopChan:
			return
		case <-c.readEnabledChan:
		default:
			if c.readEnabled {
                // 如果c.readEnabled为true，就一直通过c.doRead()做读取
				err := c.doRead()
                ......//忽略错误处理代码
			} else {
                // 如果c.readEnabled为false，就会阻塞在c.readEnabledChan 直至100毫秒超时。
				select {
				case <-c.readEnabledChan:
				case <-time.After(100 * time.Millisecond):
				}
			}

			runtime.Gosched()
		}
	}
}
```

因为startReadLoop()是一个无限的for循环，因此调用startReadLoop()是通过goroutine。

