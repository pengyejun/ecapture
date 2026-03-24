# eCapture 性能优化分析与改进方案

本文档分析 eCapture 项目中新架构下存在的性能瓶颈和优化机会，并提供具体的改进方案。

---

## 一、优化概览

基于新的 internal/ 架构，以下是潜在的优化点：

| 优化类别 | 优先级 | 预计收益 | 实现难度 |
|---------|-------|---------|---------|
| 事件分发锁竞争 | 高 | 高 | 中 |
| 输出缓冲区复用 | 高 | 中 | 低 |
| Handler 注册查找 | 中 | 低 | 低 |
| eBPF Map 预分配 | 中 | 中 | 低 |
| 事件对象池 | 中 | 高 | 中 |
| 字符串拼接优化 | 中 | 低 | 低 |
| 日志输出优化 | 低 | 低 | 低 |

---

## 二、新架构特点分析

### 2.1 架构改进带来的优势

新的 internal/ 架构相比旧的 user/ 架构有以下优势：

1. **清晰的职责分离**
   - domain: 纯接口定义，无依赖
   - probe: 探针实现
   - events: 事件分发
   - output: 输出处理

2. **更好的可测试性**
   - 接口抽象便于 mock
   - 依赖注入模式

3. **Handler 模式**
   - 灵活的输出格式组合
   - 易于扩展新的输出方式

### 2.2 潜在的性能瓶颈

#### 2.2.1 事件分发层 (internal/events/)

**当前实现**:
```go
// Dispatcher 将事件分发给所有注册的 Handler
func (d *Dispatcher) Dispatch(event domain.Event) error {
    d.mu.RLock()
    defer d.mu.RUnlock()

    for _, handler := range d.handlers {
        if err := handler.Handle(event); err != nil {
            // 记录错误但继续
        }
    }
    return nil
}
```

**问题分析**:
- 所有 Handler 串行执行
- 单个慢 Handler 会阻塞整个分发
- 读锁在高并发下仍有竞争

**优化方案**:
1. **并行分发**
   - 使用 worker pool 并行调用 Handler
   - 适用于 Handler 之间无依赖的场景

2. **异步分发**
   - 使用 channel 缓冲事件
   - 后台 goroutine 处理分发

3. **Handler 优先级**
   - 关键 Handler 优先执行
   - 非关键 Handler 可以异步

---

## 三、详细优化方案

### 3.1 事件对象池优化 [高优先级]

#### 问题分析

**位置**: `internal/probe/openssl/` 等探针的事件解码

**当前模式**:
```go
// 每次解码都创建新的事件对象
func (d *tlsEventDecoder) Decode(_ *ebpf.Map, data []byte) (domain.Event, error) {
    event := &Event{}  // 🔴 每次都创建新对象
    if err := event.DecodeFromBytes(data); err != nil {
        return nil, err
    }
    return event, nil
}
```

**问题**:
1. 高频事件（如 TLS 数据）导致大量小对象分配
2. 增加 GC 压力
3. 事件对象结构相对固定，适合复用

#### 优化方案：使用 sync.Pool

```go
package openssl

import "sync"

// 事件对象池
var eventPool = sync.Pool{
    New: func() interface{} {
        return &Event{}
    },
}

var connDataEventPool = sync.Pool{
    New: func() interface{} {
        return &ConnDataEvent{}
    },
}

var masterSecretEventPool = sync.Pool{
    New: func() interface{} {
        return &MasterSecretEvent{}
    },
}

var packetEventPool = sync.Pool{
    New: func() interface{} {
        return &PacketEvent{}
    },
}

// 获取事件对象
func getEvent() *Event {
    return eventPool.Get().(*Event)
}

// 归还事件对象
func putEvent(e *Event) {
    // 重置事件状态
    *e = Event{}
    eventPool.Put(e)
}

func getConnDataEvent() *ConnDataEvent {
    return connDataEventPool.Get().(*ConnDataEvent)
}

func putConnDataEvent(e *ConnDataEvent) {
    *e = ConnDataEvent{}
    connDataEventPool.Put(e)
}

func getMasterSecretEvent() *MasterSecretEvent {
    return masterSecretEventPool.Get().(*MasterSecretEvent)
}

func putMasterSecretEvent(e *MasterSecretEvent) {
    *e = MasterSecretEvent{}
    masterSecretEventPool.Put(e)
}

func getPacketEvent() *PacketEvent {
    return packetEventPool.Get().(*PacketEvent)
}

func putPacketEvent(e *PacketEvent) {
    *e = PacketEvent{}
    packetEventPool.Put(e)
}

// 修改解码器使用池
func (d *tlsEventDecoder) Decode(_ *ebpf.Map, data []byte) (domain.Event, error) {
    event := getEvent()  // 📦 从池中获取
    if err := event.DecodeFromBytes(data); err != nil {
        putEvent(event)  // 出错时归还
        return nil, err
    }
    if err := event.Validate(); err != nil {
        putEvent(event)  // 出错时归还
        return nil, err
    }
    // 注意：需要在 Handler 处理完成后归还
    return event, nil
}
```

**使用注意事项**:
- 需要在 Handler 处理完成后显式归还对象
- 可以通过包装 Event 接口实现自动归还
- 或者使用引用计数管理

**进阶方案：带引用计数的事件池**

```go
// PooledEvent 包装事件，添加引用计数
type PooledEvent struct {
    domain.Event
    refCount int32
    pool     *sync.Pool
}

func (pe *PooledEvent) IncRef() {
    atomic.AddInt32(&pe.refCount, 1)
}

func (pe *PooledEvent) DecRef() {
    if atomic.AddInt32(&pe.refCount, -1) == 0 {
        // 重置并归还
        pe.pool.Put(pe.Event)
    }
}
```

**预期收益**: 减少 30-50% 的事件对象分配，降低 GC 压力

---

### 3.2 输出缓冲区复用优化 [高优先级]

#### 问题分析

**位置**: `internal/output/writers/`

**当前实现**:
```go
// FileWriter 每次写入可能分配新的缓冲区
func (w *FileWriter) Write(data []byte) (int, error) {
    // 直接写入，没有缓冲区复用
    return w.file.Write(data)
}
```

**问题**:
1. 小批量写入导致频繁的系统调用
2. 没有缓冲区复用
3. 可以使用 bufio.Writer 但需要正确管理

#### 优化方案：带缓冲池的写入器

```go
package writers

import (
    "bufio"
    "sync"
)

// 缓冲区大小
const bufferSize = 32 * 1024  // 32KB

// 缓冲区对象池
var bufferPool = sync.Pool{
    New: func() interface{} {
        buf := make([]byte, 0, bufferSize)
        return &buf
    },
}

// BufferedFileWriter 带缓冲和池的文件写入器
type BufferedFileWriter struct {
    writer   *bufio.Writer
    file     File
    pool     *sync.Pool
}

func NewBufferedFileWriter(config FileWriterConfig) (*BufferedFileWriter, error) {
    // ... 创建文件 ...

    return &BufferedFileWriter{
        file:   file,
        writer: bufio.NewWriterSize(file, bufferSize),
    }, nil
}

func (w *BufferedFileWriter) Write(data []byte) (int, error) {
    return w.writer.Write(data)
}

func (w *BufferedFileWriter) Flush() error {
    return w.writer.Flush()
}

func (w *BufferedFileWriter) Close() error {
    if err := w.writer.Flush(); err != nil {
        _ = w.file.Close()
        return err
    }
    return w.file.Close()
}
```

**预期收益**: 减少 40-60% 的系统调用次数，提高写入吞吐量

---

### 3.3 事件分发优化 [高优先级]

#### 问题分析

**位置**: `internal/events/dispatcher.go`

**当前实现**:
```go
func (d *Dispatcher) Dispatch(event domain.Event) error {
    d.mu.RLock()
    defer d.mu.RUnlock()

    var errs []error
    for _, handler := range d.handlers {
        if err := handler.Handle(event); err != nil {
            d.logger.Warn().Err(err).Str("handler", handler.Name()).Msg("Handler error")
            errs = append(errs, err)
        }
    }

    if len(errs) > 0 {
        return fmt.Errorf("%d handlers failed", len(errs))
    }
    return nil
}
```

**问题**:
1. 串行调用所有 Handler，一个慢 Handler 拖慢整体
2. 每次都获取读锁，高并发下有竞争
3. 没有利用多核并行处理

#### 优化方案：并行分发器

```go
package events

import (
    "sync"
    "sync/atomic"
)

// ParallelDispatcher 支持并行分发的分发器
type ParallelDispatcher struct {
    mu         sync.RWMutex
    handlers   []Handler
    logger     Logger
    workerPool chan struct{}  // 限制并发数
    errChan    chan error     // 错误收集
}

func NewParallelDispatcher(logger Logger, maxWorkers int) *ParallelDispatcher {
    if maxWorkers <= 0 {
        maxWorkers = 4  // 默认 4 个 worker
    }
    return &ParallelDispatcher{
        handlers:   make([]Handler, 0, 4),
        logger:     logger,
        workerPool: make(chan struct{}, maxWorkers),
        errChan:    make(chan error, 16),
    }
}

func (d *ParallelDispatcher) Dispatch(event domain.Event) error {
    d.mu.RLock()
    handlers := make([]Handler, len(d.handlers))
    copy(handlers, d.handlers)
    d.mu.RUnlock()

    if len(handlers) == 0 {
        return nil
    }

    var wg sync.WaitGroup
    var errCount int32

    for _, h := range handlers {
        wg.Add(1)

        // 获取 worker 槽位
        select {
        case d.workerPool <- struct{}{}:
            // 继续执行
        default:
            // 槽位满，同步执行
            if err := h.Handle(event); err != nil {
                d.logger.Warn().Err(err).Str("handler", h.Name()).Msg("Handler error")
                atomic.AddInt32(&errCount, 1)
            }
            wg.Done()
            continue
        }

        // 异步执行
        go func(h Handler) {
            defer func() {
                <-d.workerPool  // 释放槽位
                wg.Done()
            }()

            if err := h.Handle(event); err != nil {
                d.logger.Warn().Err(err).Str("handler", h.Name()).Msg("Handler error")
                atomic.AddInt32(&errCount, 1)
            }
        }(h)
    }

    wg.Wait()

    if errCount > 0 {
        return fmt.Errorf("%d handlers failed", errCount)
    }
    return nil
}
```

**预期收益**: 在多核系统上，分发吞吐量提升 2-4 倍

---

### 3.4 字符串拼接优化 [中优先级]

#### 问题分析

**位置**: 各 Event 的 String() 方法

**当前模式**:
```go
func (e *Event) String() string {
    return fmt.Sprintf("PID:%d, Comm:%s, Tuple:%s, ...",
        e.Pid, e.Comm, e.Tuple)
}
```

**问题**:
1. `fmt.Sprintf` 需要解析格式字符串
2. 每次都分配新的字符串
3. 对于高频事件，这是可观的开销

#### 优化方案：使用 strings.Builder 并预分配

```go
import (
    "strconv"
    "strings"
)

func (e *Event) String() string {
    var sb strings.Builder

    // 预估计容量，减少扩容
    sb.Grow(256)

    sb.WriteString("PID:")
    sb.WriteString(strconv.FormatInt(int64(e.Pid), 10))
    sb.WriteString(", Comm:")
    sb.WriteString(e.Comm)
    sb.WriteString(", Tuple:")
    sb.WriteString(e.Tuple)
    // ... 其他字段

    return sb.String()
}
```

**更优方案：直接写入 Writer 接口**

```go
// WriteTo 直接写入 Writer，避免中间字符串
func (e *Event) WriteTo(w io.Writer) (int64, error) {
    var total int64

    n, _ := w.Write([]byte("PID:"))
    total += int64(n)

    // 使用 strconv.AppendInt 避免分配
    buf := make([]byte, 0, 32)
    buf = strconv.AppendInt(buf, int64(e.Pid), 10)
    n, _ = w.Write(buf)
    total += int64(n)

    n, _ = w.Write([]byte(", Comm:"))
    total += int64(n)
    n, _ = w.Write([]byte(e.Comm))
    total += int64(n)

    return total, nil
}
```

**预期收益**: 减少 20-30% 的字符串分配开销

---

### 3.5 Map 容量预分配 [低优先级]

#### 问题分析

**位置**: `internal/probe/base/base_probe.go` 等

**当前模式**:
```go
type BaseProbe struct {
    // ...
    readers []closer
    closers []closer
}

func NewBaseProbe(name string) *BaseProbe {
    return &BaseProbe{
        name:    name,
        readers: make([]closer, 0),      // 🔴 默认容量为 0
        closers: make([]closer, 0),      // 🔴 默认容量为 0
    }
}
```

**问题**:
1. 切片初始容量为 0，首次添加时分配
2. 后续扩容需要复制数据
3. 已知通常的容量，可以预分配

#### 优化方案：预分配合理容量

```go
const (
    defaultReadersCapacity = 8
    defaultClosersCapacity = 4
)

func NewBaseProbe(name string) *BaseProbe {
    return &BaseProbe{
        name:    name,
        readers: make([]closer, 0, defaultReadersCapacity),
        closers: make([]closer, 0, defaultClosersCapacity),
    }
}
```

**预期收益**: 减少 5-10% 的切片扩容开销

---

### 3.6 日志输出优化 [低优先级]

#### 问题分析

**位置**: `internal/logger/` 和各探针的日志输出

**当前模式**:
- 每次日志调用都可能进行序列化
- 调试日志在生产环境仍有开销

#### 优化方案：懒加载和条件编译

```go
// 使用 zerolog 的 Level 检查
if e.logger.Debug().Enabled() {
    e.logger.Debug().
        Str("map", em.String()).
        Int("size_mb", mapSize/1024/1024).
        Msg("Perf event reader started")
}

// 或者使用封装的方法
func (l *Logger) DebugIfEnabled(msg string, fn func(e *zerolog.Event)) {
    if !l.debugEnabled {
        return
    }
    e := l.logger.Debug()
    fn(e)
    e.Msg(msg)
}
```

---

## 四、优化实施建议

### 4.1 实施优先级

1. **第一阶段（立即实施）**：
   - 事件对象池（sync.Pool）
   - 输出缓冲区复用
   - Map 容量预分配

2. **第二阶段（短期实施）**：
   - 并行事件分发
   - 字符串拼接优化

3. **第三阶段（中期实施）**：
   - 异步分发模式
   - 更细粒度的锁优化

4. **第四阶段（长期优化）**：
   - 无锁数据结构
   - 内存分配器定制
   - 其他专项优化

### 4.2 性能测试建议

实施优化后，建议进行以下性能测试：

1. **基准测试**：
```go
func BenchmarkEventDispatch(b *testing.B) {
    // 测试事件分发吞吐量
}

func BenchmarkEventDecode(b *testing.B) {
    // 测试事件解码性能
}
```

2. **压力测试**：
- 使用 wrk 等工具模拟高并发 HTTPS 请求
- 监控 CPU、内存、GC 指标

3. **pprof 分析**：
```go
import _ "net/http/pprof"
```
- CPU profile 分析热点
- Memory profile 分析内存分配

### 4.3 回滚方案

1. 使用编译标签控制优化开关：
```go
//go:build !optimization
// ... 原始代码
```

2. 添加配置选项：
```go
type Optimizations struct {
    UseEventPool    bool
    UseBufferPool   bool
    UseParallelDispatch bool
}
```

---

## 五、新架构的天然优势

### 5.1 易于优化的设计

1. **接口抽象**
   - domain.Event 接口便于包装和优化
   - 可以在不改变调用方的情况下替换实现

2. **Handler 模式**
   - 可以独立优化每个 Handler
   - 支持 Handler 的组合和装饰

3. **依赖注入**
   - 便于注入优化后的组件
   - 支持 A/B 测试不同实现

### 5.2 推荐的优化组合

1. **事件池 + 缓冲池**
   - 对象复用 + IO 缓冲
   - 最大程度减少分配

2. **并行分发 + 异步写入**
   - 利用多核 + 减少阻塞
   - 高吞吐场景最佳组合

---

## 六、总结

通过以上优化方案的实施，基于新的 internal/ 架构，预期可以获得以下收益：

| 指标 | 优化前 | 优化后 | 提升 |
|------|-------|-------|------|
| 事件吞吐量 | 基准 | +40-70% | 显著 |
| 内存分配次数 | 基准 | -40-60% | 显著 |
| GC 压力 | 基准 | -30-50% | 显著 |
| CPU 使用率 | 基准 | -15-25% | 中等 |
| 平均延迟 | 基准 | -10-20% | 中等 |

---

文档版本: 2.0
生成日期: 2026-03-24
