# eCapture 代码结构文档

## 概述

eCapture（旁观者）是一个基于 eBPF 的工具，可以在不使用 CA 证书的情况下捕获 SSL/TLS 明文。它支持多种加密库（OpenSSL、GnuTLS、NSPR、BoringSSL、GoTLS），并为 Bash、MySQL 和 PostgreSQL 应用程序提供审计功能。

**项目统计：**
- Go 文件总数：~130+ 个（cli: 20+，pkg: 38+，internal: 70+）
- eBPF 内核文件：36 个 C 文件，20 个头文件
- 编程语言：Go、C（eBPF）、Protobuf
- 最低内核版本：Linux/Android x86_64 4.18+、aarch64 5.5+
- 需要 ROOT 权限
- Go 版本：1.24.3+

---

## 目录结构

```
/Users/pengyejun/github/ecapture/
├── cli/                    # 命令行接口（主入口点）
│   ├── cmd/               # Cobra 命令定义
│   ├── http/              # HTTP 配置服务器
│   └── cobrautl/          # Cobra 工具
├── kern/                   # eBPF 内核空间程序（C 代码）
├── internal/               # 内部核心代码（新架构，原 user/）
│   ├── domain/            # 领域接口和定义
│   ├── probe/             # 探针实现
│   │   ├── base/          # 基础探针类
│   │   ├── base/handlers/ # 输出处理器（text, keylog, pcap）
│   │   ├── openssl/       # OpenSSL 探针
│   │   ├── gnutls/        # GnuTLS 探针
│   │   ├── gotls/         # GoTLS 探针
│   │   ├── nspr/          # NSPR 探针
│   │   ├── bash/          # Bash 探针
│   │   ├── zsh/           # Zsh 探针
│   │   ├── mysql/         # MySQL 探针
│   │   └── postgres/      # PostgreSQL 探针
│   ├── config/            # 配置管理
│   ├── events/            # 事件分发
│   ├── output/            # 输出处理
│   │   ├── encoders/      # 输出编码器（json, plain, protobuf）
│   │   └── writers/       # 输出写入器（file, tcp, ws, stdout）
│   ├── factory/           # 探针工厂模式
│   ├── logger/            # 日志封装
│   ├── errors/            # 错误处理
│   └── builder/           # 配置构建器
├── pkg/                    # 共享工具包
│   ├── ecaptureq/          # eCaptureQ WebSocket 服务器
│   ├── event_processor/     # HTTP/TLS 事件处理（遗留）
│   ├── proc/               # 进程/ELF 解析工具
│   ├── upgrade/            # 版本升级检查
│   └── util/              # 通用工具（ebpf、kernel 等）
├── protobuf/               # Protocol Buffers 定义
│   ├── proto/              # .proto 架构文件
│   └── gen/               # 生成的 Go 代码
├── assets/                 # 生成的 eBPF 字节码资源
├── bin/                    # 编译后的二进制文件输出
├── bytecode/               # 编译后的 eBPF 字节码 (*.o)
├── build/                  # 构建产物
├── docs/                   # 文档
├── examples/               # 示例客户端
├── test/                   # 测试套件
├── lib/                    # 外部库（libpcap）
├── images/                 # 文档图片
├── utils/                  # 工具脚本
├── Makefile               # 构建系统
├── variables.mk           # 构建变量
├── functions.mk           # 构建函数
└── go.mod                # Go 模块定义
```

---

## 架构

### 高层流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLI 层 (cli/)                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ root.go     │    │ tls.go      │    │ bash.go    ││  (Cobra 命令)
│  │ gotls.go    │    │ mysqld.go   │    │ zsh.go     ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  工厂层 (internal/factory/)              │
│         Probe Factory - 按类型创建探针实例                    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  领域层 (internal/domain/)              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Interfaces: Probe, Configuration, Event, EventDecoder  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  探针层 (internal/probe/)                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ BaseProbe   │◄── │  Probes     │◄── │   Handlers  ││
│  │  (base/)    │    │             │    │             ││
│ 1. openssl     │  2. gnutls      │  3. gotls       ││
│ 4. nspr        │  5. bash        │  6. zsh         ││
│ 7. mysql       │  8. postgres    │  Text/Keylog/Pcap││
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              配置层 (internal/config/)                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ BaseConfig  │◄── │ Module-specific configs            ││
│  │  (common)   │    │ - openssl   │    │ - gotls     ││
│  │             │    │ - gnutls    │    │ - bash      ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│               eBPF 层 (kern/)                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 内核程序（C 与 eBPF）                      │    │
│  │  - openssl_*_kern.c   (17 个 OpenSSL 版本)       │    │
│  │  - boringssl_*_kern.c (5 个 BoringSSL 版本)       │    │
│  │  - gnutls_*_kern.c    (8 个 GnuTLS 版本)        │    │
│  │  - nspr_kern.c       (NSS/NSPR)                 │    │
│  │  - gotls_kern.c      (Go TLS)                   │    │
│  │  - bash_kern.c, zsh_kern.c                    │    │
│  │  - mysqld_kern.c, postgres_kern.c               │    │
│  └─────────────────────────────────────────────────────────┘    │
│  头文件：                                             │    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ ecapture.h  │    │ common.h    │    │ openssl.h   ││
│  │ boringssl.h │    │ gnutls.h   │    │ tc.h       ││
│  │ go_argument.h│    │ masterkey.h │    │            ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              事件层 (internal/events/)                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ Dispatcher  │◄── │   Event     │◄── │   Handler   ││
│  │             │    │             │    │  Interface  ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│           输出层 (internal/output/)                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Encoders: JSON, PlainText, Protobuf                  │    │
│  │ Writers: File, TCP, WebSocket, Stdout, Keylog, Pcap  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 组件详解

### 1. CLI 层 (`cli/`)

**入口点：**
- `cli/main.go:Start()` - 应用程序入口
- `cli/cmd/root.go` - 根 Cobra 命令，包含全局标志

**模块命令 (`cli/cmd/`)：**
| 文件 | 描述 | 模块 |
|------|-------------|---------|
| `tls.go` | OpenSSL/BoringSSL 捕获 | OpenSSL 模块 |
| `gotls.go` | Go TLS 捕获 | GoTLS 模块 |
| `gnutls.go` | GnuTLS 捕获 | GnuTLS 模块 |
| `nss.go` | NSS/NSPR 捕获 | NSPR 模块 |
| `bash.go` | Bash 命令审计 | Bash 模块 |
| `zsh.go` | Zsh 命令审计 | Zsh 模块 |
| `mysqld.go` | MySQL 查询审计 | MySQL 模块 |
| `postgres.go` | PostgreSQL 查询审计 | PostgreSQL 模块 |
| `ecaptureq.go` | eCaptureQ 服务器模式 | WebSocket 服务器 |

**HTTP 服务器 (`cli/http/`)：**
- `server.go` - 用于运行时配置更新的 HTTP 服务器
- `server_linux.go` / `server_ecandroid.go` - 平台特定实现
- `config_factory.go` - 不同平台的配置工厂
- `resp.go` - 响应辅助函数
- `logger.go` - HTTP 日志器

**全局标志：**
- `--debug` (-d) - 启用调试日志
- `--btf` (-b) - BTF 模式（0：自动，1：core，2：non-core）
- `--hex` - 以十六进制格式打印字节字符串
- `--mapsize` - 每个 CPU 的 eBPF map 大小（KB）
- `--pid` (-p) - 目标特定 PID
- `--uid` (-u) - 目标特定 UID
- `--logaddr` (-l) - 日志器地址（file/tcp/ws）
- `--eventaddr` - 事件收集器地址
- `--ecaptureq` - eCaptureQ 监听服务器
- `--listen` - HTTP 配置更新服务器端口
- `--tsize` (-t) - 文本模式下的截断大小
- `--eventroratesize` - 事件收集器文件轮转大小
- `--eventroratetime` - 事件收集器文件轮转时间

### 2. 领域层 (`internal/domain/`)

**核心接口：**

```go
// Probe 定义所有 eBPF 探针的接口
type Probe interface {
    Initialize(ctx context.Context, config Configuration) error
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Close() error
    Name() string
    IsRunning() bool
    Events() []*ebpf.Map
}

// Configuration 定义探针配置的接口
type Configuration interface {
    Validate() error
    GetPid() uint64
    GetUid() uint64
    GetDebug() bool
    GetHex() bool
    GetBTF() uint8
    GetPerCpuMapSize() int
    GetTruncateSize() uint64
    EnableGlobalVar() bool
    GetByteCodeFileMode() uint8
    Bytes() []byte
    GetLoggerAddr() string
    SetLoggerAddr(addr string)
    GetEventCollectorAddr() string
    SetEventCollectorAddr(addr string)
}

// Event 定义所有事件的接口
type Event interface {
    // ... 事件方法
}

// EventDecoder 定义事件解码器的接口
type EventDecoder interface {
    Decode(em *ebpf.Map, data []byte) (Event, error)
    GetDecoder(em *ebpf.Map) (Event, bool)
}
```

### 3. 探针层 (`internal/probe/`)

**基础架构：**

**BaseProbe (`internal/probe/base/base_probe.go`) - 核心实现：**
- 通用探针初始化
- 事件读取器管理（perf/ringbuf）
- 事件分发器设置
- 基于上下文的生命周期管理
- 资源清理

**Handlers (`internal/probe/base/handlers/`)：**
- `text_handler.go` - 文本输出处理器
- `keylog_handler.go` - Keylog 输出处理器
- `pcap_handler.go` - PCAP/PCAPNG 输出处理器

**探针模块：**

| 模块 | 文件 | 捕获模式 |
|--------|-------|--------------|
| OpenSSL | `openssl/` | text/pcap/keylog |
| GnuTLS | `gnutls/` | text/pcap/keylog |
| GoTLS | `gotls/` | text/pcap/keylog |
| NSPR | `nspr/` | text/pcap/keylog |
| Bash | `bash/` | 命令审计 |
| Zsh | `zsh/` | 命令审计 |
| MySQL | `mysql/` | 查询审计 |
| PostgreSQL | `postgres/` | 查询审计 |

**探针生命周期：**
```
1. Initialize() - 设置配置、分发器、处理器
2. Start() - 加载 eBPF 字节码、附加探针、启动读取器
3. 事件循环：
   - 从内核空间读取（perf/ringbuf）
   - 解码事件
   - 分发给处理器
4. Stop() - 停止事件收集
5. Close() - 清理所有资源
```

### 4. 工厂层 (`internal/factory/`)

**探针工厂 (`probe_factory.go`)：**
```go
type ProbeType string

const (
    ProbeTypeBash     ProbeType = "Bash"
    ProbeTypeZsh      ProbeType = "Zsh"
    ProbeTypeMySQL    ProbeType = "MySQL"
    ProbeTypePostgres ProbeType = "postgres"
    ProbeTypeOpenSSL  ProbeType = "OpenSSL"
    ProbeTypeGnuTLS   ProbeType = "GnuTLS"
    ProbeTypeNSPR     ProbeType = "NSPR"
    ProbeTypeGoTLS    ProbeType = "GoTLS"
)

// CreateProbe 创建新的探针实例
func CreateProbe(probeType ProbeType) (domain.Probe, error)

// RegisterProbe 注册探针构造函数
func RegisterProbe(probeType ProbeType, constructor ProbeConstructor) error
```

### 5. 配置层 (`internal/config/`)

**基础配置 (`base_config.go`)：**
```go
type BaseConfig struct {
    Pid                uint64
    Uid                uint64
    Listen             string
    TruncateSize       uint64
    PerCpuMapSize      int
    IsHex              bool
    Debug              bool
    BtfMode            uint8
    ByteCodeFileMode   uint8
    LoggerAddr         string
    LoggerType         uint8
    EventCollectorAddr string
    EcaptureQ          string
}
```

**模块特定配置：**
- `openssl/config.go` - OpenSSL/BoringSSL 配置
- `gnutls/config.go` - GnuTLS 配置
- `gotls/config.go` - GoTLS 配置
- `nspr/config.go` - NSS/NSPR 配置
- `bash/config.go` - Bash 配置
- `mysql/config.go` - MySQL 配置
- `postgres/config.go` - PostgreSQL 配置

**平台变体：**
- `openssl/config_linux.go` / `openssl/config_ecandroid.go`
- `gnutls/config_linux.go` / `gnutls/config_ecandroid.go`

### 6. 事件层 (`internal/events/`)

**分发器 (`dispatcher.go`)：**
- 注册处理器
- 将事件分发给所有注册的处理器
- 管理处理器生命周期

### 7. 输出层 (`internal/output/`)

**编码器 (`output/encoders/`)：**
- `json_encoder.go` - JSON 格式编码
- `plain_encoder.go` - 纯文本编码
- `protobuf_encoder.go` - Protobuf 编码

**写入器 (`output/writers/`)：**
- `stdout_writer.go` - 控制台输出
- `file_writer.go` - 文件输出（支持可选轮转）
- `tcp_writer.go` - TCP 套接字输出
- `websocket_writer.go` - WebSocket 输出
- `keylog_writer.go` - SSL 密钥日志格式
- `pcap_writer.go` - PCAP/PCAPNG 格式
- `logger_writer.go` - 基于日志的输出
- `factory.go` - 写入器工厂

### 8. eBPF 内核层 (`kern/`)

**核心头文件：**

| 头文件 | 描述 |
|--------|-------------|
| `ecapture.h` | 主包含文件，BTF/非 BTF 模式选择 |
| `common.h` | 通用常量、宏、许可证 |
| `tc.h` | TC/XDP 数据包捕获结构 |
| `openssl.h` | OpenSSL 特定定义 |
| `openssl_masterkey.h` | OpenSSL 1.1 master.x 定义 |
| `openssl_masterkey_3.0.h` | OpenSSL 3.0+ master.x 定义 |
| `openssl_masterkey_3.2.h` | OpenSSL 3.2+ master.x 定义 |
| `boringssl_masterkey.h` | BoringSSL master.x 定义 |
| `gnutls.h` | GnuTLS 特定定义 |
| `gnutls_masterkey.h` | GnuTLS 主密钥定义 |
| `go_argument.h` | Go 运行时参数定义 |
| `core_fixes.bpf.h` | CO-RE 兼容性修复 |

**通用常量 (`common.h`)：**
```c
#define TASK_COMM_LEN 16
#define PATH_MAX_LEN 256
#define MAX_DATA_SIZE_OPENSSL 1024 * 16  // 16KB
#define MAX_DATA_SIZE_MYSQL 256
#define MAX_DATA_SIZE_POSTGRES 256
#define MAX_DATA_SIZE_BASH 256
#define MAX_DATA_SIZE_ZSH 256

// 目标过滤
const volatile u64 target_pid = 0;
const volatile u64 target_uid = 0;
```

**内核程序：**

**OpenSSL/BoringSSL（17+5 个版本）：**
- `openssl_1_0_2a_kern.c` - OpenSSL 1.0.2
- `openssl_1_1_0a_kern.c` - OpenSSL 1.1.0
- `openssl_1_1_1a_kern.c` - OpenSSL 1.1.1 (alpha)
- `openssl_1_1_1b_kern.c` - OpenSSL 1.1.1 (beta)
- `openssl_1_1_1d_kern.c` - OpenSSL 1.1.1 (dev)
- `openssl_1_1_1j_kern.c` - OpenSSL 1.1.1 (juliet)
- `openssl_3_0_0_kern.c` - OpenSSL 3.0.0
- `openssl_3_0_12_kern.c` - OpenSSL 3.0.12
- `openssl_3_1_0_kern.c` - OpenSSL 3.1.0
- `openssl_3_2_0_kern.c` - OpenSSL 3.2.0
- `openssl_3_2_3_kern.c` - OpenSSL 3.2.3
- `openssl_3_2_4_kern.c` - OpenSSL 3.2.4
- `openssl_3_3_0_kern.c` - OpenSSL 3.3.0
- `openssl_3_3_2_kern.c` - OpenSSL 3.3.2
- `openssl_3_3_3_kern.c` - OpenSSL 3.3.3
- `openssl_3_4_0_kern.c` - OpenSSL 3.4.0
- `openssl_3_4_1_kern.c` - OpenSSL 3.4.1
- `openssl_3_5_0_kern.c` - OpenSSL 3.5.0
- `boringssl_na_kern.c` - BoringSSL（原生）
- `boringssl_a_13_kern.c` - BoringSSL（Android 13）
- `boringssl_a_14_kern.c` - BoringSSL（Android 14）
- `boringssl_a_15_kern.c` - BoringSSL（Android 15）
- `boringssl_a_16_kern.c` - BoringSSL（Android 16）

**其他库：**
- `gotls_kern.c` - Go TLS (crypto/tls)
- `gnutls_3_6_12_kern.c` - GnuTLS 3.6.12
- `gnutls_3_6_13_kern.c` - GnuTLS 3.6.13
- `gnutls_3_7_0_kern.c` - GnuTLS 3.7.0
- `gnutls_3_7_3_kern.c` - GnuTLS 3.7.3
- `gnutls_3_7_7_kern.c` - GnuTLS 3.7.7
- `gnutls_3_8_4_kern.c` - GnuTLS 3.8.4
- `gnutls_3_8_7_kern.c` - GnuTLS 3.8.7
- `nspr_kern.c` - NSS/NSPR
- `bash_kern.c` - Bash 审计
- `zsh_kern.c` - Zsh 审计
- `mysqld_kern.c` - MySQL 查询审计
- `postgres_kern.c` - PostgreSQL 查询审计

**eBPF Section 类型：**
- `SEC("kprobe")` - 内核函数入口
- `SEC("kretprobe")` - 内核函数退出
- `SEC("uprobe")` - 用户函数入口
- `SEC("uretprobe")` - 用户函数退出
- `SEC("tc")` - 流量控制（TC）
- `SEC("tracepoint")` - 内核跟踪点

### 9. 工具包 (`pkg/`)

| 包 | 描述 |
|---------|-------------|
| `ecaptureq/` | 用于 eCaptureQ 协议的 WebSocket 服务器 |
| `event_processor/` | HTTP/TLS 事件处理（遗留） |
| `proc/` | 进程和 ELF 文件工具 |
| `upgrade/` | 版本升级检查（GitHub releases） |
| `util/ebpf/` | eBPF 环境工具 |
| `util/kernel/` | 内核版本工具 |
| `util/ethernet/` | 以太网/IP/TCP 工具 |
| `util/hkdf/` | HKDF 密钥派生 |
| `util/roratelog/` | 可轮转日志文件写入器 |
| `util/ws/` | WebSocket 客户端 |

### 10. 构建系统

**Makefile 结构：**

**变量 (`variables.mk`)：**
- 工具链定义（clang、go、make 等）
- 平台检测（x86_64、aarch64）
- 内核 BTF 处理
- 交叉编译支持
- 库路径（libpcap）

**函数 (`functions.mk`)：**
- 版本检查（clang 9+、go 1.24+）
- `gobuild` 宏 - 带有 CGO 的 Go 构建
- `allow-override` - 环境变量覆盖
- `release_tar` - 发布包创建

**主要 Makefile 目标：**
- `make all` - 完整构建（ebpf + assets + binary）
- `make nocore` - Non-core 构建（用于旧内核）
- `make clean` - 清理构建产物
- `make env` - 显示构建环境
- `make format` - 使用 clang-format 格式化 C 代码
- `make e2e` - 运行所有端到端测试

**构建流程：**
```
1. make ebpf / make ebpf_noncore
   └── clang: 编译 *.c → *.o (eBPF 字节码)

2. make assets / make assets_noncore
   └── go-bindata: 嵌入 *.o → assets/ebpf_probe.go

3. make build / make build_noncore
   └── go build: 编译 Go + assets → bin/ecapture
```

**字节码文件：**
- CORE 模式：`*_core.o`（CO-RE 兼容）
- Non-core 模式：`*_noncore.o`（传统内核）

---

## 数据流程

### 捕获流程（OpenSSL 示例）

```
1. CLI 命令：sudo ecapture tls -m text
   │
   ▼
2. 探针创建：factory.CreateProbe(ProbeTypeOpenSSL)
   ├─ 从注册表获取构造函数
   └─ 创建探针实例
   │
   ▼
3. 探针初始化：probe.Initialize(ctx, config)
   ├─ BaseProbe: 创建分发器
   ├─ BaseProbe: 创建输出处理器
   ├─ BaseProbe: 注册处理器
   └─ OpenSSLProbe: 版本特定设置
   │
   ▼
4. 探针启动：probe.Start(ctx)
   ├─ 从 assets 加载 eBPF 字节码
   ├─ 将 eBPF 程序加载到内核
   ├─ 将 uprobes/kprobes/TC 附加到函数
   ├─ 获取事件 maps
   └─ 启动 perf/ringbuf 事件读取器
   │
   ▼
5. 内核空间（eBPF 程序）
   ├─ Hook SSL_write() / SSL_read() 函数
   ├─ 捕获明文数据
   ├─ 提取连接元组（src/dst IP:port）
   ├─ 将事件发送到 perf/ringbuf
   └─ 从 hook 返回
   │
   ▼
6. 用户空间事件循环
   ├─ 从 perf/ringbuf 读取事件（perfEventLoop）
   ├─ 解码事件（decoder.Decode()）
   ├─ 分发事件（dispatcher.Dispatch()）
   └─ 处理器处理并输出
   │
   ▼
7. 输出
   └─ 打印明文或保存到文件
```

### eCaptureQ 流程

```
1. 服务器模式：sudo ecapture --ecaptureq=:8090 tls
   │
   ▼
2. WebSocket 服务器启动
   ├─ 在 :8090 上监听
   └─ 接受 eCaptureQ 客户端连接
   │
   ▼
3. 客户端连接（eCaptureQ GUI）
   └─ WebSocket 握手
   │
   ▼
4. 事件流
   ├─ 捕获事件（如上所述）
   ├─ 编码为 Protobuf
   ├─ 通过 WebSocket 发送
   └─ 在 GUI 中显示
```

---

## 关键设计模式

### 1. 工厂模式
- 探针通过 `factory.RegisterProbe()` 自注册
- 通过 `factory.CreateProbe()` 按名称查找
- 支持动态探针加载

### 2. 基于接口的架构（v2 新特性）
- 所有探针使用 `domain.Probe`
- 所有配置使用 `domain.Configuration`
- 所有事件使用 `domain.Event`
- 所有事件解码器使用 `domain.EventDecoder`
- 支持多态和测试
- 清晰的关注点分离

### 3. Handler 模式（v2 新特性）
- 输出逻辑分离为 Handlers
- TextHandler, KeylogHandler, PcapHandler
- 灵活的多输出组合
- 易于添加新的输出格式

### 4. 领域驱动设计（v2 新特性）
- `internal/domain/` - 纯接口定义
- 领域层无外部依赖
- 易于 mock 和测试

### 5. BTF 自动检测
- 检测内核 BTF 支持
- 选择 CORE 或 non-CORE 字节码
- 失败时回退到 non-CORE

### 6. 事件处理管道
```
eBPF → Event → Decode → Dispatcher → Handler(s) → Output
  ↑                                              ↓
  └────────── Config ────────────────────────────┘
```

### 7. 平台抽象
- 用于 Linux/Android 的构建标签
- 平台特定实现
- 条件编译

---

## 安全性考虑

### 权限
- **必需**：ROOT/capabilities（CAP_BPF、CAP_NET_ADMIN）
- **原因**：eBPF 程序加载、内核挂钩

### 内核版本
- **x86_64**：4.18+（BTF 支持）
- **aarch64**：5.5+（完整的 eBPF 功能集）

### 隔离
- BPF 验证器确保安全
- 内存边界检查
- 错误时不会导致内核崩溃

---

## 开发指南

### 添加新探针（新架构）

1. **创建 eBPF 程序** (`kern/mylib_kern.c`)
   - 定义 hook 函数
   - 捕获并发送事件

2. **定义领域类型**（如需要）
   - 如需要新接口，添加到 `internal/domain/`

3. **创建事件结构** (`internal/probe/mylib/event.go`)
   - 实现事件解码

4. **创建配置** (`internal/probe/mylib/config.go`)
   - 实现 `domain.Configuration`

5. **创建探针** (`internal/probe/mylib/mylib_probe.go`)
   - 嵌入 `base.BaseProbe`
   - 实现探针特定逻辑
   - 在 `init()` 中使用 `factory.RegisterProbe()` 注册

6. **创建 CLI 命令** (`cli/cmd/mylib.go`)
   - 添加 Cobra 命令
   - 调用 `runProbe()`

7. **更新 Makefile**
   - 添加到 `TARGETS`

8. **重新构建**
   ```bash
   make clean && make
   ```

### 调试

**内核空间（eBPF）：**
```bash
# 在 eBPF 中启用调试打印
DEBUG=1 make clean && make

# 查看 bpf_trace_printk 输出
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

**用户空间（Go）：**
```bash
# 启用调试日志
sudo ecapture tls -d

# 使用 Delve 调试器
dlv debug bin/ecapture -- --tls
```

---

## 文件统计

| 类别 | 数量 |
|----------|--------|
| CLI Go 文件 | 20+ |
| pkg Go 文件 | 38+ |
| internal Go 文件 | 70+ |
| Go 文件总数 | ~130+ |
| eBPF C 文件 | 36 |
| eBPF 头文件 | 20 |
| Protobuf 文件 | 可变 |

---

## 外部依赖

**Go 模块：**
- `github.com/cilium/ebpf` - eBPF 库
- `github.com/gojue/ebpfmanager` - eBPF 管理器
- `github.com/rs/zerolog` - 零分配日志器
- `github.com/spf13/cobra` - CLI 框架
- `github.com/gin-gonic/gin` - HTTP 服务器
- `github.com/google/gopacket` - 数据包处理
- `google.golang.org/protobuf` - Protocol buffers

**系统库：**
- `libpcap` - 数据包捕获库（静态）
- `libbpf` - eBPF 用户空间库
- `libelf` - ELF 解析（通过 Go）

---

## 架构迁移说明（v1 → v2）

### v1 到 v2 的主要变化：

1. **目录重组**
   - `user/` → `internal/`
   - 新增 `domain/`、`events/`、`output/`、`factory/`、`errors/`、`logger/` 包

2. **接口驱动设计**
   - `internal/domain/` 中的纯领域接口
   - 更好的关注点分离

3. **Handler 模式**
   - 输出逻辑移至 Handlers
   - 更灵活的输出组合

4. **错误包**
   - 集中的错误定义
   - 更好的错误包装和错误码

### 向后兼容性：
- 遗留的 `pkg/event_processor/` 仍然存在
- 支持渐进式迁移

---

## 测试

**单元测试：**
```bash
go test ./...
```

**端到端测试：**
```bash
make e2e-tls      # TLS 模块
make e2e-gnutls    # GnuTLS 模块
make e2e-gotls     # GoTLS 模块
make e2e-bash      # Bash 模块
make e2e            # 所有测试
```

**测试位置：**
- `test/e2e/` - E2E 测试脚本
- `*_test.go` - 单元测试

---

## 参考资料

- **代码仓库**：https://github.com/gojue/ecapture
- **主页**：https://ecapture.cc
- **文档**：https://github.com/gojue/ecapture/tree/master/docs
- **eCaptureQ GUI**：https://github.com/gojue/ecaptureq

---

*生成时间：2026-03-24*
*项目版本：eCapture v2.0.1*
*架构：v2 (internal/ 层)*
