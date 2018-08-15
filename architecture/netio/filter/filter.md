# Filter

## filter status

`pkg/types/network.go`

```go
type FilterStatus string

const (
	Continue      FilterStatus = "Continue"
	StopIteration FilterStatus = "StopIteration"
)
```

StopIteration表示不再继续，直接跳出当前filter chain的循环。

详见filterManager的onContinueReading()方法。

## filter interface

`pkg/types/network.go`

```go
// 总结：ReadFilter只被filterManager直接调用，也就是filterManager完全包裹了ReadFilter
type ReadFilter interface {
    // 被filterManager在onContinueReading()方法调用，只有一个调用点
	OnData(buffer IoBuffer) FilterStatus
    // 在filterManager初始化的过程中，会被调用，只有一个调用点
	OnNewConnection() FilterStatus
    // 被filterManager的AddReadFilter()方法调用，只有一个调用点
    // 传入的ReadFilterCallbacks就是包括当前read filter的activeReadFilter
	InitializeReadFilterCallbacks(cb ReadFilterCallbacks)
}

// 总结：WriteFilter只被filterManager直接调用，也就是filterManager完全包裹了WriteFilter
type WriteFilter interface {
    // 被filterManager的OnWrite()方法调用，只有一个调用点
	OnWrite(buffer []IoBuffer) FilterStatus
}

// 被read filter调用，用来和connection通讯
type ReadFilterCallbacks interface {
	Connection() Connection
	ContinueReading()
	UpstreamHost() HostInfo
	SetUpstreamHost(upstreamHost HostInfo)
}
```

## activeReadFilter struct

`pkg/network/filtermanager.go`

activeReadFilter包装ReadFilter，但实现的是ReadFilterCallbacks接口：

```go
// as a ReadFilterCallbacks
type activeReadFilter struct {
	index         int
	filter        types.ReadFilter
	filterManager *filterManager
	initialized   bool
}
```

