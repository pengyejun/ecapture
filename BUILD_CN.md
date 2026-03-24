# eCapture 编译文档

## 项目简介

eCapture(旁观者) 是一个基于 eBPF 技术的工具，用于在不使用 CA 证书的情况下捕获 SSL/TLS 明文内容。支持 OpenSSL、BoringSSL、GoTLS、GnuTLS、NSS 等多种加密库，以及 Bash、MySQL、PostgreSQL 等应用的审计功能。

## 编译环境要求

### 系统要求
- **操作系统**: Linux / Android
- **内核版本**: x86_64 4.18+，aarch64 5.5+
- **权限**: 需要 ROOT 权限

### 工具链要求
- **Go**: 1.24 或更高版本
- **Clang**: 9 或更高版本
- **LLVM**: llc 工具
- **bpftool**: 用于生成 vmlinux.h 头文件
- **git**: 用于获取版本信息

## 主要依赖库

### Go 依赖 (go.mod)

#### 核心依赖
| 库名 | 版本 | 用途 |
|------|------|------|
| github.com/cilium/ebpf | v0.18.0 | eBPF 程序加载与管理 |
| github.com/gojue/ebpfmanager | v0.5.0 | eBPF 管理器 |
| github.com/spf13/cobra | v1.9.1 | 命令行参数解析 |
| github.com/rs/zerolog | v1.34.0 | 日志记录 |
| github.com/gin-gonic/gin | v1.10.0 | HTTP 服务器 |
| github.com/google/gopacket | v1.1.20 (替换) | 网络数据包处理 |
| github.com/jschwinger233/elibpcap | v1.1.0 | libpcap 绑定 |
| github.com/shuLhan/go-bindata | v4.0.0+incompatible | 将 eBPF 字节码嵌入 Go 代码 |

#### Go 扩展库
| 库名 | 版本 | 用途 |
|------|------|------|
| golang.org/x/sys | v0.41.0 | 系统调用接口 |
| golang.org/x/net | v0.50.0 | 网络相关 |
| golang.org/x/crypto | v0.48.0 | 加密功能 |
| golang.org/x/arch | v0.23.0 | 架构相关 |
| google.golang.org/protobuf | v1.36.6 | Protocol Buffers |

### C 依赖
- **libpcap**: 用于网络数据包捕获（静态链接）

### eBPF 内核代码
位于 `kern/` 目录下，包含多个版本和类型的 eBPF 程序：
- OpenSSL 各版本支持 (17 个版本)
- BoringSSL 支持 (5 个版本，含 Android 13-16)
- GnuTLS 支持 (8 个版本)
- GoTLS 支持
- NSS/NSPR 支持
- Bash/Zsh 审计
- MySQL/PostgreSQL 审计

## 代码架构概览

### 目录结构
```
ecapture/
├── cli/                    # 命令行接口
│   ├── cmd/               # Cobra 命令定义
│   ├── http/              # HTTP 配置服务器
│   └── cobrautl/          # Cobra 工具
├── internal/               # 内部核心代码（原 user/）
│   ├── domain/            # 领域接口定义
│   ├── probe/             # 探针实现
│   │   ├── base/          # 基础探针类
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
│   │   ├── encoders/      # 输出编码器
│   │   └── writers/       # 输出写入器
│   ├── factory/           # 工厂模式
│   ├── logger/            # 日志封装
│   ├── errors/            # 错误处理
│   └── builder/           # 配置构建器
├── kern/                   # eBPF 内核空间程序（C 代码）
├── pkg/                    # 共享工具包
│   ├── ecaptureq/         # eCaptureQ WebSocket 服务器
│   ├── event_processor/   # HTTP/TLS 事件处理（未完全迁移）
│   ├── proc/              # 进程/ELF 解析工具
│   ├── upgrade/           # 版本升级检查
│   └── util/              # 通用工具
├── protobuf/               # Protocol Buffers 定义
├── assets/                 # 生成的 eBPF 字节码资源
├── bin/                    # 编译后的二进制文件输出
├── lib/                    # 外部库（libpcap）
├── Makefile               # 构建系统
├── variables.mk           # 构建变量
├── functions.mk           # 构建函数
└── go.mod                # Go 模块定义
```

### 架构分层

1. **CLI 层** (`cli/`)
   - 命令行参数解析
   - 环境检测
   - 主控制流程

2. **领域层** (`internal/domain/`)
   - 核心接口定义 (Probe, Configuration, Event, EventDecoder)
   - 无依赖的纯接口定义

3. **探针层** (`internal/probe/`)
   - BaseProbe: 通用探针基类
   - 各具体探针实现 (OpenSSL, GnuTLS, GoTLS 等)
   - eBPF 程序加载与管理

4. **配置层** (`internal/config/`)
   - BaseConfig: 基础配置
   - 各探针特定配置

5. **事件层** (`internal/events/`)
   - 事件分发器 (Dispatcher)
   - 处理器注册与管理

6. **输出层** (`internal/output/`)
   - Encoders: 编码方式 (JSON, Plain, Protobuf)
   - Writers: 输出目标 (File, TCP, WebSocket, Stdout, Keylog, Pcap)

7. **工厂层** (`internal/factory/`)
   - ProbeFactory: 探针工厂
   - 探针注册与创建

8. **eBPF 层** (`kern/`)
   - 内核态 eBPF 程序
   - 各版本加密库支持

## 编译流程

### 整体流程图

```
1. 环境检测
   ├── 检查 Clang 版本 (>= 9)
   ├── 检查 Go 版本 (>= 1.24)
   └── 检查 bpftool 工具

2. 生成内核 BTF 头文件
   └── 使用 bpftool 生成 vmlinux.h

3. 编译 eBPF 字节码 (CO-RE 模式)
   └── 编译所有 kern/*.c 文件

4. 编译 eBPF 字节码 (非 CO-RE 模式)
   └── 编译所有 kern/*.c 文件

5. 生成 Go 资源文件
   └── 使用 go-bindata 将 eBPF 字节码嵌入 Go 代码

6. 编译 libpcap 静态库
   └── 配置并编译 libpcap

7. 编译 Go 主程序
   └── 静态链接生成最终二进制文件
```

### 详细步骤

#### 步骤 1: 环境检测
Makefile 会自动检测以下工具：
- `clang` - C 编译器
- `go` - Go 编译器
- `bpftool` - BPF 工具
- `llc` - LLVM 编译器

#### 步骤 2: 生成 BTF 头文件
- **x86_64**: 使用 bpftool 从 `/sys/kernel/btf/vmlinux` 生成 `kern/bpf/x86/vmlinux.h`
- **aarch64**: 使用预生成的 `kern/bpf/arm64/vmlinux.h`

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > kern/bpf/x86/vmlinux.h
```

#### 步骤 3: 编译 eBPF 字节码 (CORE 模式)
使用 Clang 编译每个内核源文件：

```bash
clang -D__TARGET_ARCH_x86 \
    -O2 -mcpu=v1 -nostdinc -Wno-pointer-sign \
    -I ./kern -I ./kern/bpf/x86 \
    -target bpfel -c kern/xxx_kern.c -o bytecode/xxx_kern_core.o \
    -fno-ident -fdebug-compilation-dir . -g
```

#### 步骤 4: 编译 eBPF 字节码 (非 CORE 模式)
使用 Clang + LLC 编译：

```bash
clang -emit-llvm -O2 -S \
    -D__TARGET_ARCH_x86 -xc -g -isystem \
    -D__BPF_TRACING__ -D__KERNEL__ -DNOCORE \
    -nostdinc -DKBUILD_MODNAME="eCapture" \
    -target x86_64 \
    -c kern/xxx_kern.c -o - | \
llc -march=bpf -filetype=obj -o bytecode/xxx_kern_noncore.o
```

#### 步骤 5: 生成 Go 资源文件
使用 go-bindata 将编译好的 eBPF 字节码转换为 Go 代码：

```bash
go run github.com/shuLhan/go-bindata/cmd/go-bindata \
    -pkg assets -o assets/ebpf_probe.go ./bytecode/*.o
```

#### 步骤 6: 编译 libpcap 静态库
```bash
cd lib/libpcap
CC=gcc CFLAGS="-O2 -g -gdwarf-4 -static -Wno-unused-result" \
./configure --disable-rdma --disable-shared --disable-usb \
    --disable-netmap --disable-bluetooth --disable-dbus \
    --without-libnl --without-dpdk --without-dag --without-septel \
    --without-gcc --with-pcap=linux --host=x86_64-pc-linux-gnu
make
```

#### 步骤 7: 编译 Go 主程序
```bash
CGO_ENABLED=1 \
CGO_CFLAGS='-O2 -g -gdwarf-4 -I./lib/libpcap/' \
CGO_LDFLAGS='-O2 -g -L./lib/libpcap/ -lpcap -static' \
GOOS=linux GOARCH=amd64 CC=gcc \
go build -trimpath -buildmode=pie -mod=readonly -tags 'linux,netgo,ebpfassets,dynamic' \
    -ldflags "-w -s -X 'github.com/gojue/ecapture/cli/cmd.GitVersion=...' \
              -X 'github.com/gojue/ecapture/cli/cmd.ByteCodeFiles=all' \
              -linkmode=external -extldflags -static" \
    -o bin/ecapture
```

## eBPF 字节码文件列表

### OpenSSL 系列支持
| 文件名 | OpenSSL 版本 |
|--------|--------------|
| openssl_1_0_2a_kern.c | 1.0.2a |
| openssl_1_1_0a_kern.c | 1.1.0a |
| openssl_1_1_1a_kern.c | 1.1.1a |
| openssl_1_1_1b_kern.c | 1.1.1b |
| openssl_1_1_1d_kern.c | 1.1.1d |
| openssl_1_1_1j_kern.c | 1.1.1j |
| openssl_3_0_0_kern.c | 3.0.0 |
| openssl_3_0_12_kern.c | 3.0.12 |
| openssl_3_1_0_kern.c | 3.1.0 |
| openssl_3_2_0_kern.c | 3.2.0 |
| openssl_3_2_3_kern.c | 3.2.3 |
| openssl_3_2_4_kern.c | 3.2.4 |
| openssl_3_3_0_kern.c | 3.3.0 |
| openssl_3_3_2_kern.c | 3.3.2 |
| openssl_3_3_3_kern.c | 3.3.3 |
| openssl_3_4_0_kern.c | 3.4.0 |
| openssl_3_4_1_kern.c | 3.4.1 |
| openssl_3_5_0_kern.c | 3.5.0 |

### BoringSSL 系列支持
| 文件名 | 说明 |
|--------|------|
| boringssl_na_kern.c | 通用 BoringSSL |
| boringssl_a_13_kern.c | Android 13 |
| boringssl_a_14_kern.c | Android 14 |
| boringssl_a_15_kern.c | Android 15 |
| boringssl_a_16_kern.c | Android 16 |

### GnuTLS 系列支持
| 文件名 | GnuTLS 版本 |
|--------|--------------|
| gnutls_3_6_12_kern.c | 3.6.12 |
| gnutls_3_6_13_kern.c | 3.6.13 |
| gnutls_3_7_0_kern.c | 3.7.0 |
| gnutls_3_7_3_kern.c | 3.7.3 |
| gnutls_3_7_7_kern.c | 3.7.7 |
| gnutls_3_8_4_kern.c | 3.8.4 |
| gnutls_3_8_7_kern.c | 3.8.7 |

### 其他模块
| 文件名 | 功能 |
|--------|------|
| gotls_kern.c | Go TLS 支持 |
| nspr_kern.c | NSS/NSPR 支持 |
| bash_kern.c | Bash 命令审计 |
| zsh_kern.c | Zsh 命令审计 |
| mysqld_kern.c | MySQL 查询审计 |
| postgres_kern.c | PostgreSQL 查询审计 |

## 编译命令

### 默认编译 (本机架构，包含 CORE 和非 CORE)
```bash
make all
```

### 仅编译非 CORE 模式 (需要内核源码)
```bash
make nocore
```

### 交叉编译到 ARM64
```bash
CROSS_ARCH=arm64 make all
```

### 交叉编译到 AMD64 (在 ARM64 主机上)
```bash
CROSS_ARCH=amd64 make all
```

### Android 版本编译
```bash
ANDROID=1 make all
```

### 调试模式编译
```bash
DEBUG=1 make all
```

### 清理编译产物
```bash
make clean
```

### 查看编译环境变量
```bash
make env
```

### 显示帮助信息
```bash
make help
```

### 代码格式化
```bash
make format
```

### 运行单元测试
```bash
make test-race
```

### 运行端到端测试
```bash
# 所有基础测试
make e2e-basic

# 所有高级测试
make e2e-advanced

# 所有测试
make e2e

# 单个模块测试
make e2e-tls
make e2e-gnutls
make e2e-gotls
make e2e-bash
make e2e-mysql
make e2e-postgres
```

## 输出文件

编译成功后，会在以下目录生成文件：

| 文件/目录 | 说明 |
|---------|------|
| bin/ecapture | 最终的可执行文件 |
| bytecode/*.o | 编译后的 eBPF 字节码文件 (core/noncore) |
| assets/ebpf_probe.go | 嵌入 eBPF 字节码的 Go 资源文件 |
| lib/libpcap.a | 静态链接的 libpcap 库 |

## 编译产物说明

### ecapture 二进制文件
- **位置**: `bin/ecapture`
- **链接方式**: 静态链接 (无需依赖运行时库)
- **包含**: eBPF 字节码、Go 运行时、libpcap 库
- **构建标签**: linux, netgo, ebpfassets, dynamic

### 版本信息
编译时会注入以下版本信息：
- Git Tag
- Commit Hash
- 构建日期
- 目标架构 (amd64/arm64)
- 内核版本

## 架构支持

### x86_64 (amd64)
- CO-RE 模式：支持
- 非 CORE 模式：需要内核源码
- 最低内核版本：4.18

### aarch64 (arm64)
- CO-RE 模式：支持
- 非 CORE 模式：需要内核源码
- 最低内核版本：5.5

## 常见编译问题

### 问题 1: 缺少 bpftool
```bash
# Ubuntu/Debian
sudo apt-get install linux-tools-generic

# RHEL/CentOS
sudo yum install bpftool
```

### 问题 2: 缺少 Clang/LLVM
```bash
# Ubuntu/Debian
sudo apt-get install clang llvm

# RHEL/CentOS
sudo yum install clang llvm
```

### 问题 3: Go 版本过低
```bash
# 需要安装 Go 1.24+
wget https://go.dev/dl/go1.24.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/go/bin
```

### 问题 4: BTF 信息不可用
```bash
# 确保内核已开启 BTF 支持
sudo apt-get install linux-modules-extra-$(uname -r)
```

### 问题 5: libpcap 子模块未初始化
```bash
git submodule update --init
```

## Makefile 目标说明

| 目标 | 功能 |
|------|------|
| all | 完整编译 (默认，包含 CORE 和非 CORE) |
| nocore | 仅编译非 CORE 模式 |
| ebpf | 仅编译 CORE eBPF 字节码 |
| ebpf_noncore | 仅编译非 CORE eBPF 字节码 |
| assets | 生成 Go 资源文件 (所有) |
| assets_noncore | 仅生成非 CORE 资源 |
| build | 编译主程序 |
| clean | 清理编译产物 |
| format | 格式化代码 |
| test-race | 运行单元测试 |
| env | 显示编译环境 |
| help | 显示帮助信息 |

### 测试目标
| 目标 | 功能 |
|------|------|
| e2e-bash | Bash 模块端到端测试 |
| e2e-zsh | Zsh 模块端到端测试 |
| e2e-mysql | MySQL 模块端到端测试 |
| e2e-postgres | PostgreSQL 模块端到端测试 |
| e2e-tls | TLS 模块端到端测试 |
| e2e-gnutls | GnuTLS 模块端到端测试 |
| e2e-gotls | GoTLS 模块端到端测试 |
| e2e-tls-text-advanced | TLS Text 模式高级测试 |
| e2e-tls-pcap-advanced | TLS Pcap 模式高级测试 |
| e2e-tls-keylog-advanced | TLS Keylog 模式高级测试 |
| e2e-gotls-advanced | GoTLS 高级测试 |
| e2e-bash-advanced | Bash 高级测试 |
| e2e-mysql-advanced | MySQL 高级测试 |
| e2e-edge-cases | 边界条件和错误处理测试 |
| e2e-basic | 所有基础端到端测试 |
| e2e-advanced | 所有高级端到端测试 |
| e2e | 所有端到端测试 |
| e2e-android-tls | Android TLS 测试 |
| e2e-android-gotls | Android GoTLS 测试 |
| e2e-android-all | 所有 Android 测试 |

---

文档版本: 2.0
生成日期: 2026-03-24
