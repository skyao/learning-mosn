# filterManager

## filterManager interface

`pkg/types/network.go`

```go
type FilterManager interface {
	AddReadFilter(rf ReadFilter)
	AddWriteFilter(wf WriteFilter)
	ListReadFilter() []ReadFilter
	ListWriteFilters() []WriteFilter

	InitializeReadFilters() bool

	OnRead()
	OnWrite() FilterStatus
}
```

## filterManager struct

`pkg/types/network.go`:

```go
type filterManager struct {
	upstreamFilters   []*activeReadFilter 
	downstreamFilters []types.WriteFilter
	conn              types.Connection
	host              types.HostInfo
}
```

创建filterManager：

```go
func newFilterManager(conn types.Connection) types.FilterManager {
	return &filterManager{
		conn:              conn,
		upstreamFilters:   make([]*activeReadFilter, 0, 32),
		downstreamFilters: make([]types.WriteFilter, 0, 32),
	}
}
```

添加filter的方法：

```go
func (fm *filterManager) AddReadFilter(rf types.ReadFilter) {
    //把传入的ReadFilter包装为activeReadFilter
    newArf := &activeReadFilter{
        filter:        rf,
        filterManager: fm,
    }

    // 初始化ReadFilterCallbacks，将包裹read filter的activeReadFilter传入
    rf.InitializeReadFilterCallbacks(newArf)
    // 将新生成的activeReadFilter加入到filterManager
    fm.upstreamFilters = append(fm.upstreamFilters, newArf)
}

func (fm *filterManager) AddWriteFilter(wf types.WriteFilter) {
    fm.downstreamFilters = append(fm.downstreamFilters, wf)
}
```

filterManager 中保存有ReadFilter和WriteFilter，而ReadFilter实际是包装过后的activeReadFilter。

InitializeReadFilters()方法：

```go
func (fm *filterManager) InitializeReadFilters() bool {
	if len(fm.upstreamFilters) == 0 {
		return false
	}
	// 实质是调用onContinueReading方法
	fm.onContinueReading(nil)
	return true
}

func (fm *filterManager) onContinueReading(filter *activeReadFilter) {
	var index int
	var uf *activeReadFilter

	if filter != nil {
		index = filter.index + 1
	}

	for ; index < len(fm.upstreamFilters); index++ {
		uf = fm.upstreamFilters[index]
		uf.index = index

        // 检查并在必要时做初始化
		if !uf.initialized {
			uf.initialized = true
			// 初始化时调用read filter的OnNewConnection()方法
            // 这个OnNewConnection()方法只有在这里会被调用
			status := uf.filter.OnNewConnection()

			if status == types.StopIteration {
				return
			}
		}

        // 获取read buffer
        //TBD: 在for循环中获取，buffer不会有变化吧？可以放循环外不？
		buffer := fm.conn.GetReadBuffer()

		if buffer != nil && buffer.Len() > 0 {
            // 调用read filter的OnData()方法
            // 这个OnData()方法只有在这里会被调用
			status := uf.filter.OnData(buffer)

			if status == types.StopIteration {
				//fm.conn.Write("your data")
				return
			}
		}
	}
}
```

OnRead()和OnWrite()方法：

```go
func (fm *filterManager) OnRead() {
	fm.onContinueReading(nil)
}

func (fm *filterManager) OnWrite() types.FilterStatus {
	for _, df := range fm.downstreamFilters {
		buffer := fm.conn.GetWriteBuffer()
        //为每个write filter调用OnWrite()方法
		status := df.OnWrite(buffer)

		if status == types.StopIteration {
			return types.StopIteration
		}
	}

	return types.Continue
}
```



## Filter Chain的调用方式

1. 从头开始

   参数filter为nil，index从0开始，这样for循环按照`fm.upstreamFilters[index]`获取每个read filter，然后 `uf.index = index` 将每个fiter的index属性设置为当前在fm.upstreamFilters[]数组中的实际下标。相当于对齐了read filter的index属性。

   这样的调用地方有两个， 1 是上面的filterManager的InitializeReadFilters()方法：

   ```go
   func (fm *filterManager) InitializeReadFilters() bool {
   	fm.onContinueReading(nil)
   }
   ```

   和 filterManager的OnRead()方法：

   ```go
   func (fm *filterManager) OnRead() {
   	fm.onContinueReading(nil)
   }
   ```

   ​

2. 从中间某个filter继续读

   即当前read filter已经执行完成，继续后续的read filter：

   ```go
   func (arf *activeReadFilter) ContinueReading() {
      arf.filterManager.onContinueReading(arf)
   }
   ```

   假定当前arf的index是3，则表示下标为0,1,2,3的read filter已经处理完成，onContinueReading(arf)执行时，

   `index = filter.index + 1` 得到index=4，`uf = fm.upstreamFilters[index]`拿到下标为4的read filter继续处理，刚好接上。`uf.index = index` 实际没修改index值。

