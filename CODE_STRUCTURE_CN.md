# eCapture 代码结构文档

## 概述

eCapture（旁观者）是一个基于 eBPF 的工具，可以在不使用 CA 证书的情况下捕获 SSL/TLS 明文。它支持多种加密库（OpenSSL、GnuTLS、NSPR、BoringSSL、GoTLS），并为 Bash、MySQL 和 PostgreSQL 应用程序提供审计功能。

**项目统计：**
- Go 文件总数：~126 个（cli: 20，pkg: 38，user: 68）
- eBPF 内核文件：36 个 C 文件，20 个头文件
- 编程语言：Go、C（eBPF）、Protobuf
- 最低内核版本：Linux/Android x86_64 4.18+、aarch64 5.5+
- 需要 ROOT 权限

---

## 目录结构

```
/extend/code/github/ecapture/
├── cli/                    # 命令行接口（主入口点）
├── kern/                   # eBPF 内核空间程序（C 代码）
├── user/                   # 用户空间 Go 代码（核心逻辑）
│   ├── config/             # 模块配置
│   ├── event/              # 事件结构和解码器
│   └── module/             # BPF 探针模块
├── pkg/                    # 共享工具包
│   ├── ecaptureq/          # eCaptureQ WebSocket 服务器
│   ├── event_processor/     # HTTP/TLS 事件处理
│   ├── proc/               # 进程/ELF 解析工具
│   ├── upgrade/            # 版本升级检查
│   └── util/              # 通用工具（ebpf、kernel 等）
├── protobuf/               # Protocol Buffers 定义
│   ├── proto/              # .proto 架构文件
│   └── gen/               # 生成的 Go 代码
├── assets/                 # 生成的 eBPF 字节码资源
├── bin/                    # 编译后的二进制文件输出
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
│                  模块层 (user/module/)                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ probe IModule│◄── │ Module Base  │◄── │ register.go  ││
│  │  interface   │    │  class      │    │             ││
│ 1. probe_openssl.go  2. probe_gotls.go  3. probe_mysqld.go│
│ 4. probe_gnutls.go  5. probe_nspr.go   6. probe_postgres.go│
│ 7. probe_bash.go    8. probe_zsh.go                 │
│ 9. probe_pcap.go    (TC/XDP 数据包捕获)      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              配置层 (user/config/)                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ IConfig     │◄── │ BaseConfig  │◄── │ Module-specific│
│  │  interface   │    │  (common)   │    │ configs     ││
│  │   - openssl │    │   - gotls   │    │   - bash   ││
│  │   - gnutls │    │   - gnutls  │    │   - mysqld ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│               eBPF 层 (kern/)                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 内核程序（C 与 eBPF）                      │    │
│  │  - openssl_*_kern.c   (13 个 OpenSSL 版本)        │    │
│  │  - boringssl_*_kern.c (4 个 BoringSSL 版本)      │    │
│  │  - gnutls_*_kern.c    (6 个 GnuTLS 版本)       │    │
│  │  - nspr_kern.c       (NSS/NSPR)                │    │
│  │  - gotls_kern.c      (Go TLS)                  │    │
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
│              事件层 (user/event/)                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ IEventStruct │◄── │ Base struct  │◄── │ Events      ││
│  │  interface   │    │  (common)   │    │ for each    ││
│  │             │    │             │    │ module      ││
│  │ Decode()    │    │ String()    │    │ - event_openssl.go   │
│  │ Payload()   │    │ PayloadLen() │    │ - event_gotls.go     │
│  │ ToProtoBuf()│    │ Clone()     │    │ - event_gnutls.go    │
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│           处理层 (pkg/event_processor/)                │
│  - HTTP/1.0、HTTP/1.1、HTTP/2 请求/响应解析          │
│  - 协议检测和重构                                   │
│  - PCAP/PCAPNG 数据包生成                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  输出层                                │
│  - stdout（控制台）                                     │
│  - file（支持轮转）                                   │
│  - TCP 套接字                                         │
│  - WebSocket（eCaptureQ 协议）                          │
│  - Protobuf 编码事件                                    │
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
| `nspr.go` | NSS/NSPR 捕获 | NSPR 模块 |
| `bash.go` | Bash 命令审计 | Bash 模块 |
| `zsh.go` | Zsh 命令审计 | Zsh 模块 |
| `mysqld.go` | MySQL 查询审计 | MySQL 模块 |
| `postgres.go` | PostgreSQL 查询审计 | PostgreSQL 模块 |
| `ecaptureq.go` | eCaptureQ 服务器模式 | WebSocket 服务器 |

**HTTP 服务器 (`cli/http/`)：**
- `server.go` - 用于运行时配置更新的 HTTP 服务器
- `server_linux.go` / `server_androidgki.go` - 平台特定实现
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

### 2. 模块层 (`user/module/`)

**基础架构：**

**接口 (`imodule.go`) - 核心方法：**
```go
type IModule interface {
    Init(context.Context, *zerolog.Logger, config.IConfig, io.Writer) error
    Name() string
    Run() error
    Start() error
    Stop() error
    Close() error
    SetChild(module IModule)
    Decode(*ebpf.Map, []byte) (event.IEventStruct, error)
    Events() []*ebpf.Map
    DecodeFun(p *ebpf.Map) (event.IEventStruct, bool)
    Dispatcher(event.IEventStruct)
}
```

**模块注册器 (`register.go`)：**
- `RegisteFunc()` - 注册模块工厂函数
- `GetModuleFunc()` - 通过名称获取模块

**基础模块 (`imodule.go`) - 实现：**
- BTF 模式自动检测（内核/容器检测）
- 事件读取器管理（perf/ringbuf）
- 事件分发和处理
- 基于上下文的生命周期管理

**探针模块：**

| 模块 | 文件 | 捕获模式 |
|--------|-------|--------------|
| OpenSSL | `probe_openssl.go` | text/pcap/keylog |
| OpenSSL PCAP | `probe_openssl_pcap.go` | PCAP/PCAPNG |
| OpenSSL Keylog | `probe_openssl_keylog.go` | 主密钥 |
| OpenSSL Text | `probe_openssl_text.go` | 明文 |
| OpenSSL Lib | `probe_openssl_lib.go` | 库检测 |
| GoTLS | `probe_gotls.go` | text/pcap/keylog |
| GoTLS Text | `probe_gotls_text.go` | 明文 |
| GoTLS Keylog | `probe_gotls_keylog.go` | 主密钥 |
| GoTLS PCAP | `probe_gotls_pcap.go` | PCAP/PCAPNG |
| GnuTLS | `probe_gnutls.go` | text/pcap/keylog |
| GnuTLS Keylog | `probe_gnutls_keylog.go` | 主密钥 |
| GnuTLS PCAP | `probe_gnutls_pcap.go` | PCAP/PCAPNG |
| GnuTLS Text | `probe_gnutls_text.go` | 明文 |
| GnuTLS Lib | `probe_gnutls_lib.go` | 库检测 |
| NSPR | `probe_nspr.go` | text/pcap/keylog |
| Bash | `probe_bash.go` | 命令审计 |
| Zsh | `probe_zsh.go` | 命令审计 |
| MySQL | `probe_mysqld.go` | 查询审计 |
| PostgreSQL | `probe_postgres.go` | 查询审计 |
| PCAP | `probe_pcap.go` | TC/XDP 数据包捕获 |

**模块生命周期：**
```
1. Init() - 初始化配置、maps、读取器
2. Run() - 启动子模块
3. readEvents() - 创建 perf/ringbuf 读取器
4. 事件循环：
   - 从内核空间读取
   - 解码事件
   - 分发到处理器/收集器
5. Close() - 清理资源
```

### 3. 配置层 (`user/config/`)

**接口 (`iconfig.go`)：**
```go
type IConfig interface {
    Check() error
    GetPid() uint64
    GetUid() uint64
    GetHex() bool
    GetBTF() uint8
    GetDebug() bool
    GetByteCodeFileMode() uint8
    // ... setter 和 getter
    Bytes() []byte  // JSON 序列化
}
```

**基础配置 (`common.go`)：**
```go
type BaseConfig struct {
    Pid          uint64
    Uid          uint64
    Listen       string
    TruncateSize uint64
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
- `config_openssl.go` - OpenSSL/BoringSSL 配置
- `config_gnutls.go` - GnuTLS 配置
- `config_gotls.go` - GoTLS 配置
- `config_nspr.go` - NSS/NSPR 配置
- `config_bash.go` - Bash 配置
- `config_zsh.go` - Zsh 配置
- `config_mysqld.go` - MySQL 配置
- `config_postgres.go` - PostgreSQL 配置

**平台变体：**
- `common_linux.go` - Linux 特定
- `common_androidgki.go` - Android GKI 特定
- `config_openssl_linux.go` / `config_openssl_androidgki.go`
- `config_gnutls_linux.go` / `config_gnutls_androidgki.go`
- `config_nspr_linux.go` / `config_nspr_androidgki.go`

**常量：**
```go
// 捕获模式
TlsCaptureModelText   = "text"
TlsCaptureModelPcap   = "pcap"
TlsCaptureModelPcapngng = "pcapng"
TlsCaptureModelKey    = "key"
TlsCaptureModelKeylog = "keylog"

// BTF 模式
BTFModeAutoDetect = 0
BTFModeCore       = 1
BTFModeNonCore    = 2

// 字节码文件模式
ByteCodeFileAll     = 0
ByteCodeFileCore    = 1
ByteCodeFileNonCore = 2
```

### 4. eBPF 内核层 (`kern/`)

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

**OpenSSL/BoringSSL（13 个版本）：**
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
- `boringssl_a_13_kern.c` - BoringSSL（arm 1.3）
- `boringssl_a_14_kern.c` - BoringSSL（arm 1.4）
- `boringssl_a_15_kern.c` - BoringSSL（arm 1.5）
- `boringssl_a_16_kern.c` - BoringSSL（arm 1.6）

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

### 5. 事件层 (`user/event/`)

**接口 (`ievent.go`)：**
```go
type IEventStruct interface {
    Decode(payload []byte) (err error)
    Payload() []byte
    PayloadLen() int
    String() string
    StringHex() string
    Clone() IEventStruct
    EventType() Type
    GetUUID() string
    Base() Base
    ToProtobufEvent() *pb.Event
}
```

**事件类型：**
- `TypeOutput` - 上传到服务器或写入日志文件
- `TypeModuleData` - 模块缓存数据
- `TypeEventProcessor` - 通过 event_processor 显示

**基础结构 (`event_base.go`)：**
```go
type Base struct {
    Timestamp int64  `json:"timestamp"`
    UUID      string `json:"uuid"`
    SrcIP     string `json:"src_ip"`
    SrcPort   uint32 `json:"src_port"`
    DstIP     string `json:"dst_ip"`
    DstPort   uint32 `json:"dst_port"`
    PID       int64  `json:"pid"`
    PName     string `json:"pname"`
    Type      uint32 `json:"type"`
    Length    uint32 `json:"length"`
}
```

**模块特定事件：**
- `event_openssl.go` - OpenSSL/BoringSSL 事件
- `event_gotls.go` - GoTLS 事件
- `event_gnutls.go` - GnuTLS 事件
- `event_nspr.go` - NSS/NSPR 事件
- `event_bash.go` - Bash 命令事件
- `event_zsh.go` - Zsh 命令事件
- `event_mysqld.go` - MySQL 查询事件
- `event_postgres.go` - PostgreSQL 查询事件

**特殊事件：**
- `event_masterkey.go` - 主密钥事件
- `event_mastersecret_gotls.go` - GoTLS 主密钥
- `event_mastersecret_gnutls.go` - GnuTLS 主密钥
- `event_openssl_tc.go` - OpenSSL TC 数据包事件
- `misc.go` - 杂项工具

**收集器写入器：**
```go
type CollectorWriter struct {
    logger *zerolog.Logger
}
```

### 6. 处理层 (`pkg/event_processor/`)

**组件：**

| 文件 | 描述 |
|------|-------------|
| `processor.go` | 主事件处理器 |
| `http_request.go` | HTTP/1.x 请求解析 |
| `http_response.go` | HTTP/1.x 响应解析 |
| `http2_request.go` | HTTP/2 请求解析 |
| `http2_response.go` | HTTP/2 响应解析 |
| `iworker.go` | Worker 接口 |
| `iparser.go` | Parser 接口 |

**HTTP 解析器功能：**
- HTTP/1.0、HTTP/1.1、HTTP/2 支持
- 头部解析和显示
- 主体内容提取
- 协议检测
- HPACK（HTTP/2 头部压缩）

### 7. 工具包 (`pkg/`)

| 包 | 描述 |
|---------|-------------|
| `ecaptureq/` | 用于 eCaptureQ 协议的 WebSocket 服务器 |
| `event_processor/` | HTTP/TLS 事件处理 |
| `proc/` | 进程和 ELF 文件工具 |
| `upgrade/` | 版本升级检查（GitHub releases） |
| `util/ebpf/` | eBPF 环境工具 |
| `util/kernel/` | 内核版本工具 |
| `util/ethernet/` | 以太网/IP/TCP 工具 |
| `util/hkdf/` | HKDF 密钥派生 |
| `util/roratelog/` | 可轮转日志文件写入器 |
| `util/ws/` | WebSocket 客户端 |

**eCaptureQ 包 (`pkg/ecaptureq/`)：**
- `server.go` - WebSocket 服务器
- `client.go` - WebSocket 客户端
- `hub.go` - 连接中心

### 8. Protobuf 层 (`protobuf/`)

**用于 eCaptureQ 通信的 Protocol Buffers：**

**架构 (`protobuf/proto/v1/`)：**
- 定义用于跨平台通信的事件结构
- 版本 1 协议规范

**生成代码 (`protobuf/gen/v1/`)：**
- 从 .proto 文件生成的 Go 代码
- 被 eCaptureQ 客户端/服务器使用

### 9. 构建系统

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
1. make ebpf
   └── clang: 编译 *.c → *.o (eBPF 字节码)

2. make assets
   └── go-bindata: 嵌入 *.o → assets/ebpf_probe.go

3. make build
   └── go build: 编译 Go + assets → bin/ecapture
```

**字节码文件：**
- CORE 模式：`*_core.o`（CO-RE 兼容）
- Non-core 模式：`*_noncore.o`（传统内核）

---

## 数据流程

### 捕获流程（OpenSSL 示例）

```
1. CLI 命令：`sudo ecapture tls -m text`
   │
   ▼
2. 模块初始化：probe_openssl.Init()
   ├─ 从 assets 加载 eBPF 字节码
   ├─ 将 eBPF 程序加载到内核
   ├─ 将 kprobes/uprobes 附加到 SSL 函数
   └─ 创建 perf/ringbuf 事件 maps
   │
   ▼
3. 内核空间（eBPF 程序）
   ├─ Hook SSL_write() / SSL_read() 函数
   ├─ 捕获明文数据
   ├─ 提取连接元组（src/dst IP:port）
   ├─ 将事件发送到 perf/ringbuf
   └─ 从 hook 返回
   │
   ▼
4. 用户空间事件循环
   ├─ 从 perf/ringbuf 读取事件
   ├─ 解码事件（event_openssl.Decode()）
   ├─ 处理事件（event_processor）
   └─ 输出到控制台/文件/socket
   │
   ▼
5. 显示
   └─ 打印明文或保存到文件
```

### eCaptureQ 流程

```
1. 服务器模式：`sudo ecapture --ecaptureq=:8090 tls`
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

### 1. 模块注册器模式
- 模块通过 `RegisteFunc()` 自注册
- 通过 `GetModuleFunc()` 按名称查找
- 支持动态模块加载

### 2. 基于接口的架构
- 所有探针模块使用 `IModule`
- 所有配置使用 `IConfig`
- 所有事件使用 `IEventStruct`
- 支持多态和测试

### 3. BTF 自动检测
- 检测内核 BTF 支持
- 选择 CORE 或 non-CORE 字节码
- 失败时回退到 non-CORE

### 4. 事件处理管道
```
eBPF → Event → Decode → Process → Output
  ↑                                        ↓
  └────────── Config ───────────────────────┘
```

### 5. 平台抽象
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

### 添加新模块

1. **创建 eBPF 程序** (`kern/mylib_kern.c`)
   - 定义 hook 函数
   - 捕获并发送事件

2. **创建事件结构** (`user/event/event_mylib.go`)
   - 实现 `IEventStruct`
   - 定义解码逻辑

3. **创建配置** (`user/config/config_mylib.go`)
   - 实现 `IConfig`
   - 定义模块特定标志

4. **创建模块** (`user/module/probe_mylib.go`)
   - 实现 `IModule`
   - 使用 `RegisteFunc()` 注册

5. **创建 CLI 命令** (`cli/cmd/mylib.go`)
   - 添加 Cobra 命令
   - 调用 `runModule()`

6. **更新 Makefile**
   - 添加到 `*TARGETS` 或 `TARGETS_NOCORE`

7. **重新构建**
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
| CLI Go 文件 | 20 |
| pkg Go 文件 | 38 |
| user Go 文件 | 68 |
| Go 文件总数 | ~126 |
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

*生成时间：2026-03-21*
*项目版本：eCapture v1.5.2*
