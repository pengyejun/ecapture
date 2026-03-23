# eCapture Code Structure Documentation

## Overview

eCapture (旁观者) is an eBPF-based tool for capturing SSL/TLS plaintext without a CA certificate. It supports multiple encryption libraries (OpenSSL, GnuTLS, NSPR, BoringSSL, GoTLS) and provides audit capabilities for Bash, MySQL, and PostgreSQL applications.

**Project Statistics:**
- Total Go files: ~126 (cli: 20, pkg: 38, user: 68)
- eBPF kernel files: 36 C files, 20 header files
- Programming languages: Go, C (eBPF), Protobuf
- Minimum kernel: Linux/Android x86_64 4.18+, aarch64 5.5+
- Requires ROOT privileges

---

## Directory Structure

```
/extend/code/github/ecapture/
├── cli/                    # Command-line interface (main entry point)
├── kern/                   # eBPF kernel-space programs (C code)
├── user/                   # User-space Go code (core logic)
│   ├── config/             # Module configurations
│   ├── event/              # Event structures and decoders
│   └── module/             # BPF probe modules
├── pkg/                    # Shared utility packages
│   ├── ecaptureq/          # eCaptureQ WebSocket server
│   ├── event_processor/     # HTTP/TLS event processing
│   ├── proc/               # Process/ELF parsing utilities
│   ├── upgrade/            # Version upgrade checking
│   └── util/              # General utilities (ebpf, kernel, etc.)
├── protobuf/               # Protocol Buffers definitions
│   ├── proto/              # .proto schema files
│   └── gen/               # Generated Go code
├── assets/                 # Generated eBPF bytecode assets
├── bin/                    # Compiled binaries output
├── build/                  # Build artifacts
├── docs/                   # Documentation
├── examples/               # Example clients
├── test/                   # Test suites
├── lib/                    # External libraries (libpcap)
├── images/                 # Documentation images
├── utils/                  # Utility scripts
├── Makefile               # Build system
├── variables.mk           # Build variables
├── functions.mk           # Build functions
└── go.mod                # Go module definition
```

---

## Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLI Layer (cli/)                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ root.go     │    │ tls.go      │    │ bash.go    ││  (Cobra commands)
│  │ gotls.go    │    │ mysqld.go   │    │ zsh.go     ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Module Layer (user/module/)                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ probe IModule│◄── │ Module Base  │◄── │ register.go  ││
│  │  interface   │    │  class      │    │             ││
│ 1. probe_openssl.go  2. probe_gotls.go  3. probe_mysqld.go│
│ 4. probe_gnutls.go  5. probe_nspr.go   6. probe_postgres.go│
│ 7. probe_bash.go    8. probe_zsh.go                 │
│ 9. probe_pcap.go    (TC/XDP packet capture)      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Configuration Layer (user/config/)              │
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
│               eBPF Layer (kern/)                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Kernel Programs (C with eBPF)                    │    │
│  │  - openssl_*_kern.c   (13 OpenSSL versions)        │    │
│  │  - boringssl_*_kern.c (4 BoringSSL versions)      │    │
│  │  - gnutls_*_kern.c    (6 GnuTLS versions)       │    │
│ugs  │  - nspr_kern.c       (NSS/NSPR)                │    │
│  │  - gotls_kern.c      (Go TLS)                  │    │
│  │  - bash_kern.c, zsh_kern.c                    │    │
│  │  - mysqld_kern.c, postgres_kern.c               │    │
│  └─────────────────────────────────────────────────────────┘    │
│  Headers:                                           │    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ ecapture.h  │    │ common.h    │    │ openssl.h   ││
│  │ boringssl.h │    │ gnutls.h   │    │ tc.h       ││
│  │ go_argument.h│    │ masterkey.h │    │            ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Event Layer (user/event/)                      │
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
│           Processing Layer (pkg/event_processor/)           │
│  - HTTP/1.0, HTTP/1.1, HTTP/2 request/response parsing  │
│  - Protocol detection and reconstruction                  │
│  - PCAP/PCAPNG packet generation                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Output Layer                             │
│  - stdout (console)                                    │
│  - file (with rotation)                                │
│  - TCP socket                                          │
│  - WebSocket (eCaptureQ protocol)                         │
│  - Protobuf encoded events                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. CLI Layer (`cli/`)

**Entry Points:**
- `cli/main.go:Start()` - Application entry point
- `cli/cmd/root.go` - Root Cobra command with global flags

**Module Commands (`cli/cmd/`):**
| File | Description | Module |
|------|-------------|---------|
| `tls.go` | OpenSSL/BoringSSL capture | OpenSSL Module |
| `gotls.go` | Go TLS capture | GoTLS Module |
| `gnutls.go` | GnuTLS capture | GnuTLS Module |
| `nspr.go` | NSS/NSPR capture | NSPR Module |
| `bash.go` | Bash command audit | Bash Module |
| `zsh.go` | Zsh command audit | Zsh Module |
| `mysqld.go` | MySQL query audit | MySQL Module |
| `postgres.go` | PostgreSQL query audit | PostgreSQL Module |
| `ecaptureq.go` | eCaptureQ server mode | WebSocket Server |

**HTTP Server (`cli/http/`):**
- `server.go` - HTTP server for runtime config updates
- `server_linux.go` / `server_androidgki.go` - Platform-specific
- `resp.go` - Response helpers
- `logger.go` - HTTP logger

**Global Flags:**
- `--debug` (-d) - Enable debug logging
- `--btf` (-b) - BTF mode (0:auto, 1:core, 2:non-core)
- `--hex` - Print byte strings as hex
- `--mapsize` - eBPF map size per CPU (KB)
- `--pid` (-p) - Target specific PID
- `--uid` (-u) - Target specific UID
- `--logaddr` (-l) - Logger address (file/tcp/ws)
- `--eventaddr` - Event collector address
- `--ecaptureq` - eCaptureQ listening server
- `--listen` - HTTP config update server port
- `--tsize` (-t) - Truncate size in text mode

### 2. Module Layer (`user/module/`)

**Base Architecture:**

**Interface (`imodule.go`) - Core methods:**
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

**Module Registry (`register.go`):**
- `RegisteFunc()` - Register module factories
- `GetModuleFunc()` - Retrieve module by name

**Base Module (`imodule.go`) - Implementation:**
- BTF mode auto-detection (kernel/container detection)
- Event reader management (perf/ringbuf)
- Event dispatching and processing
- Context-based lifecycle management

**Probe Modules:**

| Module | File | Capture Mode |
|--------|-------|--------------|
| OpenSSL | `probe_openssl.go` | text/pcap/keylog |
| OpenSSL PCAP | `probe_openssl_pcap.go` | PCAP/PCAPNG |
| OpenSSL Keylog | `probe_openssl_keylog.go` | Master Secret |
| OpenSSL Text | `probe_openssl_text.go` | Plaintext |
| OpenSSL Lib | `probe_openssl_lib.go` | Library detection |
| GoTLS | `probe_gotls.go` | text/pcap/keylog |
| GoTLS Text | `probe_gotls_text.go` | Plaintext |
| GoTLS Keylog | `probe_gotls_keylog.go` | Master Secret |
| GoTLS PCAP | `probe_gotls_pcap.go` | PCAP/PCAPNG |
| GnuTLS | `probe_gnutls.go` | text/pcap/keylog |
| GnuTLS Keylog | `probe_gnutls_keylog.go` | Master Secret |
| GnuTLS PCAP | `probe_gnutls_pcap.go` | PCAP/PCAPNG |
| GnuTLS Text | `probe_gnutls_text.go` | Plaintext |
| GnuTLS Lib | `probe_gnutls_lib.go` | Library detection |
| NSPR | `probe_nspr.go` | text/pcap/keylog |
| Bash | `probe_bash.go` | Command audit |
| Zsh | `probe_zsh.go` | Command audit |
| MySQL | `probe_mysqld.go` | Query audit |
| PostgreSQL | `probe_postgres.go` | Query audit |
| PCAP | `probe_pcap.go` | TC/XDP packet capture |

**Module Lifecycle:**
```
1. Init() - Initialize config, maps, readers
2. Run() - Start child module
3. readEvents() - Create perf/ringbuf readers
4. Event Loop:
   - Read from kernel space
   - Decode events
   - Dispatch to processor/collector
5. Close() - Cleanup resources
```

### 3. Configuration Layer (`user/config/`)

**Interface (`iconfig.go`):**
```go
type IConfig interface {
    Check() error
    GetPid() uint64
    GetUid() uint64
    GetHex() bool
    GetBTF() uint8
    GetDebug() bool
    GetByteCodeFileMode() uint8
    // ... setters and getters
    Bytes() []byte  // JSON serialization
}
```

**Base Configuration (`common.go`):**
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

**Module-Specific Configs:**
- `config_openssl.go` - OpenSSL/BoringSSL config
- `config_gnutls.go` - GnuTLS config
- `config_gotls.go` - GoTLS config
- `config_nspr.go` - NSS/NSPR config
- `config_bash.go` - Bash config
- `config_zsh.go` - Zsh config
- `config_mysqld.go` - MySQL config
- `config_postgres.go` - PostgreSQL config

**Platform Variants:**
- `common_linux.go` - Linux-specific
- `common_androidgki.go` - Android GKI-specific
- `config_openssl_linux.go` / `config_openssl_androidgki.go`
- `config_gnutls_linux.go` / `config_gnutls_androidgki.go`
- `config_nspr_linux.go` / `config_nspr_androidgki.go`

**Constants:**
```go
// Capture modes
TlsCaptureModelText   = "text"
TlsCaptureModelPcap   = "pcap"
TlsCaptureModelPcapngng = "pcapng"
TlsCaptureModelKey    = "key"
TlsCaptureModelKeylog = "keylog"

// BTF modes
BTFModeAutoDetect = 0
BTFModeCore       = 1
BTFModeNonCore    = 2

// Bytecode file modes
ByteCodeFileAll     = 0
ByteCodeFileCore    = 1
ByteCodeFileNonCore = 2
```

### 4. eBPF Kernel Layer (`kern/`)

**Core Headers:**

| Header | Description |
|--------|-------------|
| `ecapture.h` | Main include, BTF/Non-BTF mode selection |
| `common.h` | Common constants, macros, license |
| `tc.h` | TC/XDP packet capture structures |
| `openssl.h` | OpenSSL-specific definitions |
| `openssl_masterkey.h` | OpenSSL 1.1 master.x definitions |
| `openssl_masterkey_3.0.h` | OpenSSL 3.0+ master.x definitions |
| `openssl_masterkey_3.2.h` | OpenSSL 3.2+ master.x definitions |
| `boringssl_masterkey.h` | BoringSSL master.x definitions |
| `gnutls.h` | GnuTLS-specific definitions |
| `gnutls_masterkey.h` | GnuTLS master secret definitions |
| `go_argument.h` | Go runtime argument definitions |
| `core_fixes.bpf.h` | CO-RE compatibility fixes |

**Common Constants (`common.h`):**
```c
#define TASK_COMM_LEN 16
#define PATH_MAX_LEN 256
#define MAX_DATA_SIZE_OPENSSL 1024 * 16  // 16KB
#define MAX_DATA_SIZE_MYSQL 256
#define MAX_DATA_SIZE_POSTGRES 256
#define MAX_DATA_SIZE_BASH 256
#define MAX_DATA_SIZE_ZSH 256

// Target filtering
const volatile u64 target_pid = 0;
const volatile u64 target_uid = 0;
```

**Kernel Programs:**

**OpenSSL/BoringSSL (13 versions):**
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
- `boringssl_na_kern.c` - BoringSSL (native)
- `boringssl_a_13_kern.c` - BoringSSL (arm 1.3)
- `boringssl_a_14_kern.c` - BoringSSL (arm 1.4)
- `boringssl_a_15_kern.c` - BoringSSL (arm 1.5)
- `boringssl_a_16_kern.c` - BoringSSL (arm 1.6)

**Other Libraries:**
- `gotls_kern.c` - Go TLS (crypto/tls)
- `gnutls_3_6_12_kern.c` - GnuTLS 3.6.12
- `gnutls_3_6_13_kern.c` - GnuTLS 3.6.13
- `gnutls_3_7_0_kern.c` - GnuTLS 3.7.0
- `gnutls_3_7_3_kern.c` - GnuTLS 3.7.3
- `gnutls_3_7_7_kern.c` - GnuTLS 3.7.7
- `gnutls_3_8_4_kern.c` - GnuTLS 3.8.4
- `gnutls_3_8_7_kern.c` - GnuTLS 3.8.7
- `nspr_kern.c` - NSS/NSPR
- `bash_kern.c` - Bash audit
- `zsh_kern.c` - Zsh audit
- `mysqld_kern.c` - MySQL query audit
- `postgres_kern.c` - PostgreSQL query audit

**eBPF Section Types:**
- `SEC("kprobe")` - Kernel function entry
- `SEC("kretprobe")` - Kernel function exit
- `SEC("uprobe")` - User function entry
- `SEC("uretprobe")` - User function exit
- `SEC("tc")` - Traffic control (TC)
- `SEC("tracepoint")` - Kernel tracepoints

### 5. Event Layer (`user/event/`)

**Interface (`ievent.go`):**
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

**Event Types:**
- `TypeOutput` - Upload to server or write to log file
- `TypeModuleData` - Module cache data
- `TypeEventProcessor` - Display by event_processor

**Base Structure (`event_base.go`):**
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

**Module-Specific Events:**
- `event_openssl.go` - OpenSSL/BoringSSL events
- `event_gotls.go` - GoTLS events
- `event_gnutls.go` - GnuTLS events
- `event_nspr.go` - NSS/NSPR events
- `event_bash.go` - Bash command events
- `event_zsh.go` - Zsh command events
- `event_mysqld.go` - MySQL query events
- `event_postgres.go` - PostgreSQL query events

**Special Events:**
- `event_masterkey.go` - Master secret events
- `event_mastersecret_gotls.go` - GoTLS master secret
- `event_mastersecret_gnutls.go` - GnuTLS master secret
- `event_openssl_tc.go` - OpenSSL TC packet events
- `misc.go` - Miscellaneous utilities

**Collector Writer:**
```go
type CollectorWriter struct {
    logger *zerolog.Logger
}
```

### 6. Processing Layer (`pkg/event_processor/`)

**Components:**

| File | Description |
|------|-------------|
| `processor.go` | Main event processor |
| `http_request.go` | HTTP/1.x request parsing |
| `http_response.go` | HTTP/1.x response parsing |
| `http2_request.go` | HTTP/2 request parsing |
| `http2_response.go` | HTTP/2 response parsing |
| `iworker.go` | Worker interface |
| `iparser.go` | Parser interface |

**HTTP Parser Features:**
- HTTP/1.0, HTTP/1.1, HTTP/2 support
- Header parsing and display
- Body content extraction
- Protocol detection
- HPACK (HTTP/2 header compression)

### 7. Utility Packages (`pkg/`)

| Package | Description |
|---------|-------------|
| `ecaptureq/` | WebSocket server for eCaptureQ protocol |
| `event_processor/` | HTTP/TLS event processing |
| `proc/` | Process and ELF file utilities |
| `upgrade/` | Version upgrade checking (GitHub releases) |
| `util/ebpf/` | eBPF environment utilities |
| `util/kernel/` | Kernel version utilities |
| `util/ethernet/` | Ethernet/IP/TCP utilities |
| `util/hkdf/` | HKDF key derivation |
| `util/roratelog/` | Rotatable log file writer |
| `util/ws/` | WebSocket client |

**eCaptureQ Package (`pkg/ecaptureq/`):**
- `server.go` - WebSocket server
- `client.go` - WebSocket client
- `hub.go` - Connection hub

### 8. Protobuf Layer (`protobuf/`)

**Protocol Buffers for eCaptureQ communication:**

**Schema (`protobuf/proto/v1/`):**
- Define event structures for cross-platform communication
- Version 1 protocol specification

**Generated Code (`protobuf/gen/v1/`):**
- Go code generated from .proto files
- Used by eCaptureQ client/server

### 9. Build System

**Makefile Structure:**

**Variables (`variables.mk`):**
- Toolchain definitions (clang, go, make, etc.)
- Platform detection (x86_64, aarch64)
- Kernel BTF handling
- Cross-compilation support
- Library paths (libpcap)

**Functions (`functions.mk`):**
- Version checks (clang 9+, go 1.24+)
- `gobuild` macro - Go build with CGO
- `allow-override` - Environment variable override
- `release_tar` - Release package creation

**Main Makefile Targets:**
- `make all` - Full build (ebpf + assets + binary)
- `make nocore` - Non-core build (for older kernels)
- `make clean` - Clean build artifacts
- `make env` - Display build environment
- `make format` - Format C code with clang-format
- `make e2e` - Run all end-to-end tests

**Build Flow:**
```
1. make ebpf
   └── clang: Compile *.c → *.o (eBPF bytecode)

2. make assets
   └── go-bindata: Embed *.o → assets/ebpf_probe.go

3. make build
   └── go build: Compile Go + assets → bin/ecapture
```

**Bytecode Files:**
- CORE mode: `*_core.o` (CO-RE compatible)
- Non-core mode: `*_noncore.o` (legacy kernels)

---

## Data Flow

### Capture Flow (OpenSSL Example)

```
1. CLI Command: `sudo ecapture tls -m text`
   │
   ▼
2. Module Initialization: probe_openssl.Init()
   ├─ Load eBPF bytecode from assets
   ├─ Load eBPF program into kernel
   ├─ Attach kprobes/uprobes to SSL functions
   └─ Create perf/ringbuf event maps
   │
   ▼
3. Kernel Space (eBPF Program)
   ├─ Hook SSL_write() / SSL_read() functions
   ├─ Capture plaintext data
   ├─ Extract connection tuple (src/dst IP:port)
   ├─ Send event to perf/ringbuf
   └─ Return from hook
   │
   ▼
4. User Space Event Loop
   ├─ Read event from perf/ringbuf
   ├─ Decode event (event_openssl.Decode())
   ├─ Process event (event_processor)
   └─ Output to console/file/socket
   │
   ▼
5. Display
   └─ Print plaintext or save to file
```

### eCaptureQ Flow

```
1. Server Mode: `sudo ecapture --ecaptureq=:8090 tls`
   │
   ▼
2. WebSocket Server Start
   ├─ Listen on :8090
   └─ Accept eCaptureQ client connections
   │
   ▼
3. Client Connects (eCaptureQ GUI)
   └─ WebSocket handshake
   │
   ▼
4. Event Streaming
   ├─ Capture events (as above)
   ├─ Encode as Protobuf
   ├─ Send via WebSocket
   └─ Display in GUI
```

---

## Key Design Patterns

### 1. Module Registry Pattern
- Modules self-register via `RegisteFunc()`
- Lookup by name via `GetModuleFunc()`
- Enables dynamic module loading

### 2. Interface-Based Architecture
- `IModule` for all probe modules
- `IConfig` for all configurations
- `IEventStruct` for all events
- Enables polymorphism and testing

### 3. BTF Auto-Detection
- Detect kernel BTF support
- Choose CORE or non-CORE bytecode
- Fall back to non-CORE on failure

### 4. Event Processing Pipeline
```
eBPF → Event → Decode → Process → Output
  ↑                                        ↓
  └────────── Config ───────────────────────┘
```

### 5. Platform Abstraction
- Build tags for Linux/Android
- Platform-specific implementations
- Conditional compilation

---

## Security Considerations

### Privileges
- **Required**: ROOT/capabilities (CAP_BPF, CAP_NET_ADMIN)
- **Reason**: eBPF program loading, kernel hooks

### Kernel Versions
- **x86_64**: 4.18+ (BTF support)
- **aarch64**: 5.5+ (full eBPF feature set)

### Isolation
- BPF verifier ensures safety
- Memory bounds checking
- No kernel crashes on errors

---

## Development Guide

### Adding a New Module

1. **Create eBPF Program** (`kern/mylib_kern.c`)
   - Define hook functions
   - Capture and send events

2. **Create Event Structure** (`user/event/event_mylib.go`)
   - Implement `IEventStruct`
   - Define decode logic

3. **Create Configuration** (`user/config/config_mylib.go`)
   - Implement `IConfig`
   - Define module-specific flags

4. **Create Module** (`user/module/probe_mylib.go`)
   - Implement `IModule`
   - Register with `RegisteFunc()`

5. **Create CLI Command** (`cli/cmd/mylib.go`)
   - Add Cobra command
   - Call `runModule()`

6. **Update Makefile**
   - Add to `*TARGETS` or `TARGETS_NOCORE`

7. **Rebuild**
   ```bash
   make clean && make
   ```

### Debugging

**Kernel Space (eBPF):**
```bash
# Enable debug prints in eBPF
DEBUG=1 make clean && make

# View bpf_trace_printk output
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

**User Space (Go):**
```bash
# Enable debug logging
sudo ecapture tls -d

# Use Delve debugger
dlv debug bin/ecapture -- --tls
```

---

## File Statistics

| Category | Count |
|----------|--------|
| CLI Go files | 20 |
| pkg Go files | 38 |
| user Go files | 68 |
| Total Go files | ~126 |
| eBPF C files | 36 |
| eBPF headers | 20 |
| Protobuf files | variable |

---

## External Dependencies

**Go Modules:**
- `github.com/cilium/ebpf` - eBPF library
- `github.com/gojue/ebpfmanager` - eBPF manager
- `github.com/rs/zerolog` - Zero-allocation logger
- `github.com/spf13/cobra` - CLI framework
- `github.com/gin-gonic/gin` - HTTP server
- `github.com/google/gopacket` - Packet processing
- `google.golang.org/protobuf` - Protocol buffers

**System Libraries:**
- `libpcap` - Packet capture library (static)
- `libbpf` - eBPF userspace library
- `libelf` - ELF parsing (via Go)

---

## Testing

**Unit Tests:**
```bash
go test ./...
```

**End-to-End Tests:**
```bash
make e2e-tls      # TLS module
make e2e-gnutls    # GnuTLS module
make e2e-gotls     # GoTLS module
make e2e            # All tests
```

**Test Locations:**
- `test/e2e/` - E2E test scripts
- `*_test.go` - Unit tests

---

## References

- **Repository**: https://github.com/gojue/ecapture
- **Homepage**: https://ecapture.cc
- **Documentation**: https://github.com/gojue/ecapture/tree/master/docs
- **eCaptureQ GUI**: https://github.com/gojue/ecaptureq

---

*Generated: 2026-03-21*
*Project: eCapture v1.5.2*
