# eCapture Code Structure Documentation

## Overview

eCapture is an eBPF-based tool for capturing SSL/TLS plaintext without a CA certificate. It supports multiple encryption libraries (OpenSSL, GnuTLS, NSPR, BoringSSL, GoTLS) and provides audit capabilities for Bash, MySQL, and PostgreSQL applications.

**Project Statistics:**
- Total Go files: ~130+ (cli: 20+, pkg: 38+, internal: 70+)
- eBPF kernel files: 36 C files, 20 header files
- Programming languages: Go, C (eBPF), Protobuf
- Minimum kernel: Linux/Android x86_64 4.18+, aarch64 5.5+
- Requires ROOT privileges
- Go version: 1.24.3+

---

## Directory Structure

```
/Users/pengyejun/github/ecapture/
├── cli/                    # Command-line interface (main entry point)
│   ├── cmd/               # Cobra command definitions
│   ├── http/              # HTTP config server
│   └── cobrautl/          # Cobra utilities
├── kern/                   # eBPF kernel-space programs (C code)
├── internal/               # Internal core code (NEW ARCHITECTURE, was user/)
│   ├── domain/            # Domain interfaces and definitions
│   ├── probe/             # Probe implementations
│   │   ├── base/          # Base probe class
│   │   ├── base/handlers/ # Output handlers (text, keylog, pcap)
│   │   ├── openssl/       # OpenSSL probe
│   │   ├── gnutls/        # GnuTLS probe
│   │   ├── gotls/         # GoTLS probe
│   │   ├── nspr/          # NSPR probe
│   │   ├── bash/          # Bash probe
│   │   ├── zsh/           # Zsh probe
│   │   ├── mysql/         # MySQL probe
│   │   └── postgres/      # PostgreSQL probe
│   ├── config/            # Configuration management
│   ├── events/            # Event dispatching
│   ├── output/            # Output processing
│   │   ├── encoders/      # Output encoders (json, plain, protobuf)
│   │   └── writers/       # Output writers (file, tcp, ws, stdout)
│   ├── factory/           # Factory pattern for probes
│   ├── logger/            # Logger wrapper
│   ├── errors/            # Error handling
│   └── builder/           # Config builder
├── pkg/                    # Shared utility packages
│   ├── ecaptureq/         # eCaptureQ WebSocket server
│   ├── event_processor/   # HTTP/TLS event processing (legacy)
│   ├── proc/              # Process/ELF parsing utilities
│   ├── upgrade/           # Version upgrade checking
│   └── util/              # General utilities (ebpf, kernel, etc.)
├── protobuf/               # Protocol Buffers definitions
│   ├── proto/              # .proto schema files
│   └── gen/               # Generated Go code
├── assets/                 # Generated eBPF bytecode assets
├── bin/                    # Compiled binaries output
├── bytecode/               # Compiled eBPF bytecode (*.o)
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
│                  Factory Layer (internal/factory/)          │
│  Probe Factory - creates probe instances by type          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Domain Layer (internal/domain/)            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Interfaces: Probe, Configuration, Event, EventDecoder  │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Probe Layer (internal/probe/)              │
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
│              Configuration Layer (internal/config/)          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ BaseConfig  │◄── │ Module-specific configs            ││
│  │  (common)   │    │ - openssl   │    │ - gotls     ││
│  │             │    │ - gnutls    │    │ - bash      ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│               eBPF Layer (kern/)                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Kernel Programs (C with eBPF)                      │    │
│  │  - openssl_*_kern.c   (17 OpenSSL versions)       │    │
│  │  - boringssl_*_kern.c (5 BoringSSL versions)       │    │
│  │  - gnutls_*_kern.c    (8 GnuTLS versions)        │    │
│  │  - nspr_kern.c       (NSS/NSPR)                 │    │
│  │  - gotls_kern.c      (Go TLS)                   │    │
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
│              Events Layer (internal/events/)               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐│
│  │ Dispatcher  │◄── │   Event     │◄── │   Handler   ││
│  │             │    │             │    │  Interface  ││
│  └─────────────┘    └─────────────┘    └─────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Output Layer (internal/output/)                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Encoders: JSON, PlainText, Protobuf                  │    │
│  │ Writers: File, TCP, WebSocket, Stdout, Keylog, Pcap  │    │
│  └─────────────────────────────────────────────────────────┘    │
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
| `nss.go` | NSS/NSPR capture | NSPR Module |
| `bash.go` | Bash command audit | Bash Module |
| `zsh.go` | Zsh command audit | Zsh Module |
| `mysqld.go` | MySQL query audit | MySQL Module |
| `postgres.go` | PostgreSQL query audit | PostgreSQL Module |
| `ecaptureq.go` | eCaptureQ server mode | WebSocket Server |

**HTTP Server (`cli/http/`):**
- `server.go` - HTTP server for runtime config updates
- `server_linux.go` / `server_ecandroid.go` - Platform-specific
- `config_factory.go` - Config factory for different platforms
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
- `--eventroratesize` - Event collector file rotation size
- `--eventroratetime` - Event collector file rotation time

### 2. Domain Layer (`internal/domain/`)

**Core Interfaces:**

```go
// Probe defines the interface for all eBPF probes
type Probe interface {
    Initialize(ctx context.Context, config Configuration) error
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Close() error
    Name() string
    IsRunning() bool
    Events() []*ebpf.Map
}

// Configuration defines the interface for probe configuration
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

// Event defines the interface for all events
type Event interface {
    // ... event methods
}

// EventDecoder defines the interface for decoding events
type EventDecoder interface {
    Decode(em *ebpf.Map, data []byte) (Event, error)
    GetDecoder(em *ebpf.Map) (Event, bool)
}
```

### 3. Probe Layer (`internal/probe/`)

**Base Architecture:**

**BaseProbe (`internal/probe/base/base_probe.go`) - Core implementation:**
- Common probe initialization
- Event reader management (perf/ringbuf)
- Event dispatcher setup
- Context-based lifecycle management
- Resources cleanup

**Handlers (`internal/probe/base/handlers/`):**
- `text_handler.go` - Text output handler
- `keylog_handler.go` - Keylog output handler
- `pcap_handler.go` - PCAP/PCAPNG output handler

**Probe Modules:**

| Module | File | Capture Mode |
|--------|-------|--------------|
| OpenSSL | `openssl/` | text/pcap/keylog |
| GnuTLS | `gnutls/` | text/pcap/keylog |
| GoTLS | `gotls/` | text/pcap/keylog |
| NSPR | `nspr/` | text/pcap/keylog |
| Bash | `bash/` | Command audit |
| Zsh | `zsh/` | Command audit |
| MySQL | `mysql/` | Query audit |
| PostgreSQL | `postgres/` | Query audit |

**Probe Lifecycle:**
```
1. Initialize() - Set up config, dispatcher, handlers
2. Start() - Load eBPF bytecode, attach probes, start readers
3. Event Loop:
   - Read from kernel space (perf/ringbuf)
   - Decode events
   - Dispatch to handlers
4. Stop() - Halt event collection
5. Close() - Cleanup all resources
```

### 4. Factory Layer (`internal/factory/`)

**Probe Factory (`probe_factory.go`):**
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

// CreateProbe creates a new probe instance
func CreateProbe(probeType ProbeType) (domain.Probe, error)

// RegisterProbe registers a probe constructor
func RegisterProbe(probeType ProbeType, constructor ProbeConstructor) error
```

### 5. Configuration Layer (`internal/config/`)

**Base Configuration (`base_config.go`):**
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

**Module-Specific Configs:**
- `openssl/config.go` - OpenSSL/BoringSSL config
- `gnutls/config.go` - GnuTLS config
- `gotls/config.go` - GoTLS config
- `nspr/config.go` - NSS/NSPR config
- `bash/config.go` - Bash config
- `mysql/config.go` - MySQL config
- `postgres/config.go` - PostgreSQL config

**Platform Variants:**
- `openssl/config_linux.go` / `openssl/config_ecandroid.go`
- `gnutls/config_linux.go` / `gnutls/config_ecandroid.go`

### 6. Events Layer (`internal/events/`)

**Dispatcher (`dispatcher.go`):**
- Register handlers
- Dispatch events to all registered handlers
- Manage handler lifecycle

### 7. Output Layer (`internal/output/`)

**Encoders (`output/encoders/`):**
- `json_encoder.go` - JSON format encoding
- `plain_encoder.go` - Plain text encoding
- `protobuf_encoder.go` - Protobuf encoding

**Writers (`output/writers/`):**
- `stdout_writer.go` - Console output
- `file_writer.go` - File output (with optional rotation)
- `tcp_writer.go` - TCP socket output
- `websocket_writer.go` - WebSocket output
- `keylog_writer.go` - SSL key log format
- `pcap_writer.go` - PCAP/PCAPNG format
- `logger_writer.go` - Logger-based output
- `factory.go` - Writer factory

### 8. eBPF Kernel Layer (`kern/`)

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

**OpenSSL/BoringSSL (17+5 versions):**
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
- `boringssl_a_13_kern.c` - BoringSSL (Android 13)
- `boringssl_a_14_kern.c` - BoringSSL (Android 14)
- `boringssl_a_15_kern.c` - BoringSSL (Android 15)
- `boringssl_a_16_kern.c` - BoringSSL (Android 16)

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

### 9. Utility Packages (`pkg/`)

| Package | Description |
|---------|-------------|
| `ecaptureq/` | WebSocket server for eCaptureQ protocol |
| `event_processor/` | HTTP/TLS event processing (legacy) |
| `proc/` | Process and ELF file utilities |
| `upgrade/` | Version upgrade checking (GitHub releases) |
| `util/ebpf/` | eBPF environment utilities |
| `util/kernel/` | Kernel version utilities |
| `util/ethernet/` | Ethernet/IP/TCP utilities |
| `util/hkdf/` | HKDF key derivation |
| `util/roratelog/` | Rotatable log file writer |
| `util/ws/` | WebSocket client |

### 10. Build System

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
1. make ebpf / make ebpf_noncore
   └── clang: Compile *.c → *.o (eBPF bytecode)

2. make assets / make assets_noncore
   └── go-bindata: Embed *.o → assets/ebpf_probe.go

3. make build / make build_noncore
   └── go build: Compile Go + assets → bin/ecapture
```

**Bytecode Files:**
- CORE mode: `*_core.o` (CO-RE compatible)
- Non-core mode: `*_noncore.o` (legacy kernels)

---

## Data Flow

### Capture Flow (OpenSSL Example)

```
1. CLI Command: sudo ecapture tls -m text
   │
   ▼
2. Probe Creation: factory.CreateProbe(ProbeTypeOpenSSL)
   ├─ Get constructor from registry
   └─ Create probe instance
   │
   ▼
3. Probe Initialization: probe.Initialize(ctx, config)
   ├─ BaseProbe: Create dispatcher
   ├─ BaseProbe: Create output handlers
   ├─ BaseProbe: Register handlers
   └─ OpenSSLProbe: Version-specific setup
   │
   ▼
4. Probe Start: probe.Start(ctx)
   ├─ Load eBPF bytecode from assets
   ├─ Load eBPF program into kernel
   ├─ Attach uprobes/kprobes/TC to functions
   ├─ Retrieve event maps
   └─ Start perf/ringbuf event readers
   │
   ▼
5. Kernel Space (eBPF Program)
   ├─ Hook SSL_write() / SSL_read() functions
   ├─ Capture plaintext data
   ├─ Extract connection tuple (src/dst IP:port)
   ├─ Send event to perf/ringbuf
   └─ Return from hook
   │
   ▼
6. User Space Event Loop
   ├─ Read event from perf/ringbuf (perfEventLoop)
   ├─ Decode event (decoder.Decode())
   ├─ Dispatch event (dispatcher.Dispatch())
   └─ Handlers process and output
   │
   ▼
7. Output
   └─ Print plaintext or save to file
```

### eCaptureQ Flow

```
1. Server Mode: sudo ecapture --ecaptureq=:8090 tls
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

### 1. Factory Pattern
- Probes self-register via `factory.RegisterProbe()`
- Lookup by name via `factory.CreateProbe()`
- Enables dynamic probe loading

### 2. Interface-Based Architecture (NEW in v2)
- `domain.Probe` for all probes
- `domain.Configuration` for all configurations
- `domain.Event` for all events
- `domain.EventDecoder` for event decoders
- Enables polymorphism and testing
- Clean separation of concerns

### 3. Handler Pattern (NEW in v2)
- Output logic separated into Handlers
- TextHandler, KeylogHandler, PcapHandler
- Flexible composition of multiple outputs
- Easy to add new output formats

### 4. Domain-Driven Design (NEW in v2)
- `internal/domain/` - Pure interface definitions
- No external dependencies in domain layer
- Easy to mock and test

### 5. BTF Auto-Detection
- Detect kernel BTF support
- Choose CORE or non-CORE bytecode
- Fall back to non-CORE on failure

### 6. Event Processing Pipeline
```
eBPF → Event → Decode → Dispatcher → Handler(s) → Output
  ↑                                              ↓
  └────────── Config ────────────────────────────┘
```

### 7. Platform Abstraction
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

### Adding a New Probe (NEW Architecture)

1. **Create eBPF Program** (`kern/mylib_kern.c`)
   - Define hook functions
   - Capture and send events

2. **Define Domain Types** (if needed)
   - Add to `internal/domain/` if new interfaces needed

3. **Create Event Structure** (`internal/probe/mylib/event.go`)
   - Implement event decoding

4. **Create Configuration** (`internal/probe/mylib/config.go`)
   - Implement `domain.Configuration`

5. **Create Probe** (`internal/probe/mylib/mylib_probe.go`)
   - Embed `base.BaseProbe`
   - Implement probe-specific logic
   - Register with `factory.RegisterProbe()` in `init()`

6. **Create CLI Command** (`cli/cmd/mylib.go`)
   - Add Cobra command
   - Call `runProbe()`

7. **Update Makefile**
   - Add to `TARGETS`

8. **Rebuild**
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
| CLI Go files | 20+ |
| pkg Go files | 38+ |
| internal Go files | 70+ |
| Total Go files | ~130+ |
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
make e2e-bash      # Bash module
make e2e            # All tests
```

**Test Locations:**
- `test/e2e/` - E2E test scripts
- `*_test.go` - Unit tests

---

## Architecture Migration Notes (v1 → v2)

### Key Changes from v1 to v2:

1. **Directory Restructuring**
   - `user/` → `internal/`
   - New `domain/`, `events/`, `output/`, `factory/`, `errors/`, `logger/` packages

2. **Interface-Driven Design**
   - Pure domain interfaces in `internal/domain/`
   - Better separation of concerns

3. **Handler Pattern**
   - Output logic moved to Handlers
   - More flexible output composition

4. **Error Package**
   - Centralized error definitions
   - Better error wrapping and codes

### Backward Compatibility:
- Legacy `pkg/event_processor/` still exists
- Gradual migration possible

---

## References

- **Repository**: https://github.com/gojue/ecapture
- **Homepage**: https://ecapture.cc
- **Documentation**: https://github.com/gojue/ecapture/tree/master/docs
- **eCaptureQ GUI**: https://github.com/gojue/ecaptureq

---

*Generated: 2026-03-24*
*Project: eCapture v2.0.1*
*Architecture: v2 (internal/ layer)*
