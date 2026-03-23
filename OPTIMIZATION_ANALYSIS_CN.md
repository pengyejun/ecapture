# eCapture 性能优化分析与改进方案

本文档分析 eCapture 项目中存在的性能瓶颈和优化机会，并提供具体的改进方案。

---

## 一、优化概览

| 优化类别 | 优先级 | 预计收益 | 实现难度 |
|---------|-------|---------|---------|
| 连接查找锁竞争 | 高 | 高 | 低 |
| 字节缓冲区复用 | 中 | 中 | 低 |
| 字符串拼接优化 | 中 | 中 | 低 |
| TC 包缓冲重用 | 中 | 中 | 低 |
| HTTP 解析缓冲复用 | 低 | 低 | 低 |
| Map 容量预分配 | 低 | 低 | 低 |

---

## 二、详细分析与改进方案

### 2.1 连接查找锁竞争优化 [高优先级]

#### 问题分析

**位置**: `user/module/probe_openssl.go:472-488`

```go
func (m *MOpenSSLProbe) GetConn(pid, fd uint32) *ConnInfo {
    if fd <= 0 {
        return nil
    }
    m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Msg("GetConn")
    m.pidLocker.Lock()              // 🔴 每次调用都获取锁
    defer m.pidLocker.Unlock()
    connMap, f := m.pidConns[pid]
    if !f {
        return nil
    }
    connInfo, f := connMap[fd]
    if !f {
        return nil
    }
    return &connInfo
}
```

**问题**:
1. `GetConn()` 在每次 SSL 数据事件处理时都会被调用
2. 高并发场景下，大量 SSL 数据事件会导致严重的锁竞争
3. 每次只读一个连接信息，却锁住了整个 `pidConns` map

#### 优化方案：使用 sync.RWMutex

```go
type MOpenSSLProbe struct {
    MTCProbe
    // ... 其他字段 ...

    // 使用读写锁替代互斥锁
    pidConnsLock sync.RWMutex
    // ... 其他字段 ...
}

func (m *MOpenSSLProbe) GetConn(pid, fd uint32) *ConnInfo {
    if fd <= 0 {
        return nil
    }
    m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Msg("GetConn")

    // � 读操作使用读锁，允许多个 goroutine 并发读取
    m.pidConnsLock.RLock()
    defer m.pidConnsLock.RUnlock()

    connMap, f := m.pidConns[pid]
    if !f {
        return nil
    }
    connInfo, f := connMap[fd]
    if !f {
        return nil
    }
    return &connInfo
}

func (m *MOpenSSLProbe) AddConn(pid, fd uint32, tuple string, sock uint64) {
    if fd <= 0 {
        m.logger.Info().Uint32("pid", pid).Uint32("fd", fd).Str("tuple", tuple).Msg("AddConn failed")
        return
    }

    // 📝 写操作使用写锁
    m.pidConnsLock.Lock()
    defer m.pidConnsLock.Unlock()

    connMap, f := m.pidConns[pid]
    if !f {
        connMap = make(map[uint32]ConnInfo, 16)  // 预分配容量
    }
    connMap[fd] = ConnInfo{tuple: tuple, sock: sock}
    m.pidConns[pid] = connMap

    m.sock2pidFd[sock] = [2]uint32{pid, fd}

    m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Uint64("sock", sock).Str("tuple", tuple).Msg("AddConn success")
}

func (m *MOpenSSLProbe) DestroyConn(sock uint64) {
    // 📝 写操作使用写锁
    m.pidConnsLock.Lock()
    defer m.pidConnsLock.Unlock()

    if sock > 0 {
        m.processor.WriteDestroyConn(sock)
    }

    pidFd, ok := m.sock2pidFd[sock]
    if !ok {
        return
    }

    delete(m.sock2pidFd, sock)
    pid, fd := pidFd[0], pidFd[1]

    connMap, ok := m.pidConns[pid]
    if !ok {
        return
    }

    connInfo, ok := connMap[fd]
    if ok {
        if connInfo.sock != sock {
            m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Uint64("sock", sock).Uint64("storedSock", connInfo.sock).Msg("DestroyConn skip")
            return
        }
        delete(connMap, fd)
        if len(connMap) == 0 {
            delete(m.pidConns, pid)
        }
        m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Uint64("sock", sock).Str("tuple", connInfo.tuple).Msg("DestroyConn success")
    }
}
```

**预期收益**: 在高并发 TLS 流量场景下，吞吐量可提升 30-50%

---

### 2.2 字节缓冲区复用优化 [中优先级]

#### 问题分析

**位置**: 多处创建新的 `bytes.Buffer`

```go
// 问题代码示例
func (m *MOpenSSLProbe) saveMasterSecret(...) {
    // ...
    var b = bytes.NewBuffer(nil)  // 🔴 每次调用都创建新的 Buffer
    // ...
}
```

**问题**:
1. 高频事件处理中频繁创建和销毁 bytes.Buffer
2. 导致大量内存分配和 GC 压力
3. bytes.Buffer 内部有可增长的字节数组，重用可以减少分配

#### 优化方案：使用 sync.Pool

```go
import "sync"

type MOpenSSLProbe struct {
    MTCProbe
    // ... 其他字段 ...

    // 📦 字节缓冲区对象池
    bufferPool sync.Pool
    // ... 其他字段 ...
}

// 在 Init 中初始化
func (m *MOpenSSLProbe) Init(...) error {
    // ... 其他初始化代码 ...

    // 初始化字节缓冲区对象池
    m.bufferPool = sync.Pool{
        New: func() interface{} {
            // 预分配 1KB 的缓冲区，减少后续扩容
            buf := make([]byte, 0, 1024)
            return &buf
        },
    }

    return nil
}

// 辅助函数：从池中获取 Buffer
func (m *MOpenSSLProbe) getBuffer() []byte {
    return *(m.bufferPool.Get().(*[]byte))
}

// 辅助函数：放回 Buffer 到池中
func (m *MOpenSSLProbe) putBuffer(buf []byte) {
    // 重置长度但保留容量
    buf = buf[:0]
    m.bufferPool.Put(&buf)
}

// 使用示例
func (m *MOpenSSLProbe) saveMasterSecret(secretEvent *event.MasterSecretEvent) {
    k := fmt.Sprintf("%02x", secretEvent.ClientRandom)

    _, f := m.masterKeys[k]
    if f {
        return
    }

    // 📦 从池中获取缓冲区
    buf := m.getBuffer()
    defer m.putBuffer(buf)

    buf = append(buf, []byte(hkdf.KeyLogLabelTLS12)...)
    buf = append(buf, ' ')
    buf = append(buf, k...)
    buf = append(buf, ' ')
    buf = append(buf, secretEvent.MasterKey...)
    buf = append(buf, '\n')

    v := event.TlsVersion{Version: secretEvent.Version}

    switch m.eBPFProgramType {
    case TlsCaptureModelTypePcap:
        e := m.savePcapngSslKeyLog(buf)
        if e != nil {
            m.logger.Warn().Err(e).Str("TlsVersion", v.String()).Str("CLientRandom", k).Str("eBPFProgramType", m.eBPFProgramType.String()).Msg("CLIENT_RANDOM save failed")
            return
        }
        m.logger.Info().Str("TlsVersion", v.String()).Str("CLientRandom", k).Int("bytes", len(buf)).Msg("CLIENT_RANDOM save success")
    case TlsCaptureModelTypeKeylog:
        l, e := m.keylogger.Write(buf)
        if e != nil {
            m.logger.Warn().Err(e).Str("TlsVersion", v.String()).Str("CLientRandom", k).Str("eBPFProgramType", m.eBPFProgramType.String()).Msg("CLIENT_RANDOM save failed")
            return
        }
        m.logger.Info().Str("TlsVersion", v.String()).Str("CLientRandom", k).Str("eBPFProgramType", m.eBPFProgramType.String()).Int("bytes", l).Msg("CLIENT_RANDOM save success")
    }
}

func (m *MOpenSSLProbe) Close() error {
    // ... 其他清理代码 ...
    return m.Module.Close()
}
```

**预期收益**: 减少 20-30% 的内存分配，降低 GC 压力

---

### 2.3 字符串拼接优化 [中优先级]

#### 问题分析

**位置**: `user/event/event_openssl.go` 等多处

```go
// 问题代码示例
func (event *SSLDataEvent) String() string {
    // ...
    // 🔴 使用 fmt.Sprintf 进行字符串拼接，效率较低
    return fmt.Sprintf("EventName:%s, PID:%d, Comm:%s, Tuple:%s",
        event.EventNameSSLData, event.Pid,
        CToGoString(event.Comm[:]), event.Tuple)
}
```

**问题**:
1. `fmt.Sprintf` 需要解析格式字符串，性能开销大
2. 对于简单的字符串拼接，可以使用更高效的方式

#### 优化方案：使用 strings.Builder 或预分配

```go
import "strings"

func (event *SSLDataEvent) String() string {
    // 📝 使用 strings.Builder 替代 fmt.Sprintf
    var builder strings.Builder

    // 预估计最终字符串长度，减少扩容
    builder.Grow(256)

    builder.WriteString("EventName:")
    builder.WriteString(event.EventNameSSLData)
    builder.WriteString(", PID:")
    builder.WriteString(strconv.FormatUint(uint64(event.Pid), 10))
    builder.WriteString(", Comm:")
    builder.WriteString(CToGoString(event.Comm[:]))
    builder.WriteString(", Tuple:")
    builder.WriteString(event.Tuple)

    return builder.String()
}
```

**更优方案**：对于高频事件，直接写入 Writer 而不是返回字符串

```go
func (event *SSLDataEvent) WriteTo(w io.Writer) error {
    // 直接写入 Writer，避免中间字符串分配
    _, err := w.Write([]byte("EventName:"))
    if err != nil {
        return err
    }
    _, err = w.Write([]byte(event.EventNameSSLData))
    if err != nil {
        return err
    }
    // ... 其他字段
    return nil
}
```

**预期收益**: 减少 10-20% 的字符串分配开销

---

### 2.4 TC 包缓冲重用优化 [中优先级]

#### 问题分析

**位置**: `user/module/probe_pcap.go:151-177, 295`

```go
// save pcapng file
func (t *MTCProbe) savePcapng() (int, error) {
    // ...
    t.tcPacketLocker.Lock()
    defer func() {
        t.tcPacketLocker.Unlock()
    }()

    for _, packet := range t.tcPackets {
        err = t.pcapWriter.WritePacket(packet.info, packet.data)
        i++
        if err != nil {
            return
        }
    }

    // 🔴 切片清空但没有重用底层数组
    // 下次使用时会重新分配
}

func (t *MTCProbe) ServePcap() {
    // ...
    // 🔴 每次都用新切片重置，会重新分配
    t.tcPackets = t.tcPackets[:0]
}
```

**问题**:
1. 切片清空 `t.tcPackets[:0]` 保留底层数组
2. 但在某些场景下，数组容量可能不足，导致重新分配
3. 没有针对不同负载大小的动态调整

#### 优化方案：动态容量调整

```go
func (t *MTCProbe) Init(...) error {
    // ...
    // 初始容量设为 256，后续动态调整
    t.tcPackets = make([]*TcPacket, 0, 256)
    // ...
}

func (t *MTCProbe) savePcapng() (int, error) {
    if t.masterKeyBuffer.Len() > 0 {
        err = t.pcapWriter.WriteDecryptionSecretsBlock(pcapgo.DSB_SECRETS_TYPE_TLS, t.masterKeyBuffer.Bytes())
        if err != nil {
            return
        }
    }
    t.masterKeyBuffer.Reset()

    t.tcPacketLocker.Lock()
    defer func() {
        t.tcPacketLocker.Unlock()
    }()

    for _, packet := range t.tcPackets {
        err = t.pcapWriter.WritePacket(packet.info, packet.data)
        i++
        if err != nil {
            return
        }
    }

    if i == 0 {
        return 0, nil
    }
    err = t.pcapWriter.Flush()

    // 📦 动态调整切片容量
    // 如果使用超过容量的 75%，增加容量
    // 如果使用低于容量的 25%，减少容量
    currentCap := cap(t.tcPackets)
    if currentCap > 256 {
        if i > currentCap*3/4 {
            // 使用超过 75%，扩容到 2x
            newCap := currentCap * 2
            if newCap > 4096 {
                newCap = 4096  // 设置上限
            }
            newPackets := make([]*TcPacket, 0, newCap)
            // 注意：reset 在下面进行
            t.tcPackets = newPackets
        } else if i < currentCap/4 && currentCap > 256 {
            // 使用低于 25%，缩容到 1/2
            newCap := currentCap / 2
            newPackets := make([]*TcPacket, 0, newCap)
            t.tcPackets = newPackets
        }
    }

    // 重置切片长度为 0
    t.tcPackets = t.tcPackets[:0]

    return i, err
}
```

**预期收益**: 减少内存分配 15-25%，适应不同流量负载

---

### 2.5 HTTP 请求解析缓冲复用优化 [低优先级]

#### 问题分析

**位置**: `pkg/event_processor/http_request.go:37-40, 54-68`

```go
func (hr *HTTPRequest) Init() {
    hr.reader = bytes.NewBuffer(nil)  // 🔴 创建新的 Buffer
    hr.bufReader = bufio.NewReader(hr.reader)
}

func (hr *HTTPRequest) Write(b []byte) (int, error) {
    if !hr.isInit {
        n, e := hr.reader.Write(b)
        // ...
        req, err := http.ReadRequest(hr.bufReader)
        // ...
        hr.request = req
        hr.isInit = true
        return n, nil
    }
    // ...
}
```

**问题**:
1. 每次创建新的 HTTPRequest 都会创建新的 Buffer
2. HTTP 事件高频时导致大量分配
3. Buffer 在事件处理完成后没有重用

#### 优化方案：使用对象池

```go
import "sync"

var httpRequestPool = sync.Pool{
    New: func() interface{} {
        return &HTTPRequest{
            reader: bytes.NewBuffer(make([]byte, 0, 1024)),
            bufReader: nil,  // 延迟初始化
        }
    },
}

func GetHTTPRequest() *HTTPRequest {
    req := httpRequestPool.Get().(*HTTPRequest)
    if req.bufReader == nil {
        req.bufReader = bufio.NewReader(req.reader)
    } else {
        req.bufReader.Reset(req.reader)
    }
    return req
}

func PutHTTPRequest(req *HTTPRequest) {
    req.isInit = false
    req.reader.Reset()
    httpRequestPool.Put(req)
}

// 在 EventProcessor 中使用
func ep.dispatch(e event.IEventStruct) error {
    var uuid = e.GetUUID()
    found, eWorker := ep.getWorkerByUUID(uuid)
    if !found {
        eWorker = NewEventWorker(uuid, ep)
        ep.addWorkerByUUID(eWorker)
    }

    err := eWorker.Write(e)
    eWorker.Put()
    if err != nil {
        // ...
    }
    return nil
}
```

**预期收益**: 减少 HTTP 事件处理的内存分配 20-30%

---

### 2.6 Map 容量预分配优化 [低优先级]

#### 问题分析

**位置**: `user/module/probe_openssl.go:414-418`

```go
func (m *MOpenSSLProbe) AddConn(pid, fd uint32, tuple string, sock uint64) {
    // ...
    connMap, f := m.pidConns[pid]
    if !f {
        connMap = make(map[uint32]ConnInfo)  // 🔴 默认容量为 0
    }
    // ...
}
```

**问题**:
1. map 默认容量为 0，首次插入时分配
2. 对于高并发场景，频繁的 map 扩容会影响性能
3. 已知通常每个进程的连接数，可以预分配

#### 优化方案：预分配合理容量

```go
func (m *MOpenSSLProbe) AddConn(pid, fd uint32, tuple string, sock uint64) {
    if fd <= 0 {
        m.logger.Info().Uint32("pid", pid).Uint32("fd", fd).Str("tuple", tuple).Msg("AddConn failed")
        return
    }

    m.pidConnsLock.Lock()
    defer m.pidConnsLock.Unlock()

    connMap, f := m.pidConns[pid]
    if !f {
        // 📦 预分配容量为 16，减少扩容
        connMap = make(map[uint32]ConnInfo, 16)
    }
    connMap[fd] = ConnInfo{tuple: tuple, sock: sock}
    m.pidConns[pid] = connMap

    m.sock2pidFd[sock] = [2]uint32{pid, fd}

    m.logger.Debug().Uint32("pid", pid).Uint32("fd", fd).Uint64("sock", sock).Str("tuple", tuple).Msg("AddConn success")
}
```

**预期收益**: 减少 5-10% 的 map 扩容开销

---

### 2.7 事件 Worker 对象池优化 [中优先级]

#### 问题分析

**位置**: `pkg/event_processor/iworker.go:91-98`

```go
func NewEventWorker(uuid string, processor *EventProcessor) IWorker {
    eWorker := &eventWorker{}
    eWorker.init(uuid, processor)
    go func() {
        eWorker.Run()
    }()
    return eWorker
}

func (ep *EventProcessor) dispatch(e event.IEventStruct) error {
    var uuid = e.GetUUID()
    found, eWorker := ep.getWorkerByUUID(uuid)
    if !found {
        // 🔴 每次都创建新的 eventWorker
        eWorker = NewEventWorker(uuid, ep)
        ep.addWorkerByUUID(eWorker)
    }
    // ...
}
```

**问题**:
1. 高频事件（如 WebSocket、HTTP）频繁创建/销毁 Worker
2. Worker 结构体包含多个字段，分配成本较高
3. Worker 完成后没有被回收复用

#### 优化方案：实现 Worker 对象池

```go
import "sync"

// 在 EventProcessor 中添加
type EventProcessor struct {
    sync.Mutex
    isClosed bool
    incoming chan event.IEventStruct
    outComing chan []byte
    destroyConn chan uint64
    workerQueue map[string]IWorker
    logger io.Writer
    closeChan chan bool
    errChan chan error
    isHex bool
    truncateSize uint64

    // 📦 Worker 对象池
    workerPool sync.Pool
}

func (ep *EventProcessor) init() {
    ep.incoming = make(chan event.IEventStruct, MaxIncomingChanLen)
    ep.outComing = make(chan []byte, MaxIncomingChanLen)
    ep.destroyConn = make(chan uint64, MaxIncomingChanLen)
    ep.closeChan = make(chan bool)
    ep.errChan = make(chan error, 16)
    ep.workerQueue = make(map[string]IWorker, MaxParserQueueLen)

    // 初始化 Worker 对象池
    ep.workerPool = sync.Pool{
        New: func() interface{} {
            ew := &eventWorker{}
            // 预分配 incoming channel
            ew.incoming = make(chan event.IEventStruct, MaxChanLen)
            return ew
        },
    }
}

func (ep *EventProcessor) getWorkerByUUID(uuid string) (bool, IWorker) {
    ep.Lock()
    defer ep.Unlock()
    var eWorker IWorker
    var found bool
    eWorker, found = ep.workerQueue[uuid]
    if !found {
        return false, eWorker
    }
    eWorker.Get()
    return true, eWorker
}

func (ep *EventProcessor) dispatch(e event.IEventStruct) error {
    var uuid = e.GetUUID()
    found, eWorker := ep.getWorkerByUUID(uuid)
    if !found {
        // 📦 从池中获取 Worker
        poolObj := ep.workerPool.Get()
        eWorker = poolObj.(*eventWorker)
        eWorker.init(uuid, ep)
        go func() {
            eWorker.Run()
        }()
        ep.addWorkerByUUID(eWorker)
    }

    err := eWorker.Write(e)
    eWorker.Put()
    if err != nil {
        select {
        case ep.errChan <- err:
        default:
        }
    }
    return nil
}

func (ep *EventProcessor) delWorkerByUUID(worker IWorker) {
    ep.Lock()
    defer ep.Unlock()
    delete(ep.workerQueue, worker.GetUUID())
    // 📦 归还 Worker 到池中
    ep.workerPool.Put(worker.(*eventWorker))
}
```

**预期收益**: 减少 25-40% 的高频事件处理内存分配

---

## 三、优化实施建议

### 3.1 实施优先级

1. **第一阶段（立即实施）**：
   - 连接查找锁竞争优化（RWMutex）
   - Map 容量预分配

2. **第二阶段（短期实施）**：
   - 字节缓冲区复用（sync.Pool）
   - TC 包缓冲重用

3. **第三阶段（中期实施）**：
   - 事件 Worker 对象池
   - 字符串拼接优化

4. **第四阶段（长期优化）**：
   - HTTP 请求解析缓冲复用
   - 其他专项优化

### 3.2 性能测试建议

实施优化后，建议进行以下性能测试：

1. **基准测试**：
   ```go
   func BenchmarkGetConn(b *testing.B) {
       // ... 并发获取连接信息的测试
   }
   ```

2. **压力测试**：
   - 使用 ab、wrk 等工具模拟高并发 HTTPS 请求
   - 监控 CPU、内存、GC 指标

3. **pprof 分析**：
   ```go
   import _ "net/http/pprof"
   ```
   - CPU profile 分析热点
   - Memory profile 分析内存分配

### 3.3 回滚方案

1. 使用编译标签控制优化开关：
   ```go
   //go:build !optimization
   // ... 原始代码
   ```

2. 添加配置选项：
   ```go
   type Optimizations struct {
       UseRWMutex    bool
       UseBufferPool  bool
       UseWorkerPool  bool
   }
   ```

---

## 四、总结

通过以上优化方案的实施，预期可以获得以下收益：

| 指标 | 优化前 | 优化后 | 提升 |
|------|-------|-------|------|
| 高并发 TLS 吞吐量 | 基准 | +30-50% | 显著 |
| 内存分配次数 | 基准 | -20-30% | 显著 |
| GC 压力 | 基准 | -20-30% | 显著 |
| CPU 使用率 | 基准 | -10-15% | 中等 |
| 平均延迟 | 基准 | -5-10% | 中等 |

---

文档版本: 1.0
生成日期: 2026-03-23
