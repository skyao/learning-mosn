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
```



