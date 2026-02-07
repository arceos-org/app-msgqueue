# arceos-msgqueue

A standalone message-queue application running on [ArceOS](https://github.com/arceos-org/arceos) unikernel, with all dependencies sourced from [crates.io](https://crates.io). Demonstrates **cooperative multi-task scheduling** with a producer-consumer message queue and PFlash MMIO access across multiple architectures.

## What It Does

This application demonstrates cooperative scheduling and inter-task communication:

1. **Message queue**: A `VecDeque` protected by `SpinNoIrq` (interrupt-safe spinlock) serves as the shared message queue between tasks.
2. **Producer task** (worker1): Pushes numbered messages (0..64) into the queue, calling `thread::yield_now()` after each push to cooperatively hand over the CPU.
3. **Consumer task** (worker2): Pops messages from the queue, printing each one. When the queue is empty, it yields the CPU back to the producer.
4. **Cooperative scheduling**: Tasks voluntarily yield via `thread::yield_now()`, demonstrating the basic cooperative scheduling algorithm — without yielding, a task runs until completion.
5. **PFlash MMIO**: Before starting the message queue, the main task verifies PFlash access via page-table-mapped MMIO.

## PFlash Address Map

| Architecture | PFlash Unit | Physical Address | QEMU Option |
|---|---|---|---|
| riscv64 | pflash1 | `0x22000000` | `-drive if=pflash,unit=1` |
| aarch64 | pflash1 | `0x04000000` | `-drive if=pflash,unit=1` |
| x86_64 | pflash0 | `0xFFC00000` | `-drive if=pflash,unit=0` (with embedded SeaBIOS) |
| loongarch64 | pflash1 | `0x1D000000` | `-drive if=pflash,unit=1` |

## Supported Architectures

| Architecture | Rust Target | QEMU Machine | Platform |
|---|---|---|---|
| riscv64 | `riscv64gc-unknown-none-elf` | `qemu-system-riscv64 -machine virt` | riscv64-qemu-virt |
| aarch64 | `aarch64-unknown-none-softfloat` | `qemu-system-aarch64 -machine virt` | aarch64-qemu-virt |
| x86_64 | `x86_64-unknown-none` | `qemu-system-x86_64 -machine q35` | x86-pc |
| loongarch64 | `loongarch64-unknown-none` | `qemu-system-loongarch64 -machine virt` | loongarch64-qemu-virt |

## Prerequisites

- **Rust nightly toolchain** (edition 2024)

  ```bash
  rustup install nightly
  rustup default nightly
  ```

- **Bare-metal targets** (install the ones you need)

  ```bash
  rustup target add riscv64gc-unknown-none-elf
  rustup target add aarch64-unknown-none-softfloat
  rustup target add x86_64-unknown-none
  rustup target add loongarch64-unknown-none
  ```

- **QEMU** (install the emulators for your target architectures)

  ```bash
  # Ubuntu/Debian
  sudo apt install qemu-system-riscv64 qemu-system-aarch64 \
                   qemu-system-x86 qemu-system-loongarch64  # OR qemu-system-misc

  # macOS (Homebrew)
  brew install qemu
  ```

- **SeaBIOS** (required for x86_64 only)

  ```bash
  # Ubuntu/Debian
  sudo apt install seabios
  ```

- **rust-objcopy** (from `cargo-binutils`, required for non-x86_64 targets)

  ```bash
  cargo install cargo-binutils
  rustup component add llvm-tools
  ```

## Quick Start

```bash
# install cargo-clone sub-command
cargo install cargo-clone
# get source code of arceos-msgqueue crate from crates.io
cargo clone arceos-msgqueue
# into crate dir
cd arceos-msgqueue
# Build and run on RISC-V 64 QEMU (default)
cargo xtask run

# Build and run on other architectures
cargo xtask run --arch aarch64
cargo xtask run --arch x86_64
cargo xtask run --arch loongarch64

# Build only (no QEMU)
cargo xtask build --arch riscv64
cargo xtask build --arch aarch64
```

Expected output (abbreviated):

```
Multi-task message queue is starting ...
PFlash check: [0xFFFFFFC022000000] -> PFLA
Wait for workers to exit ...
worker1 (producer) ...
worker1 [0]
worker2 (consumer) ...
worker2 [0]
worker1 [1]
worker2 [1]
...
worker1 [64]
worker1 ok!
worker2 [64]
worker2 ok!
Multi-task message queue OK!
```

QEMU will automatically exit after printing the message.

## Project Structure

```
app-msgqueue/
├── .cargo/
│   └── config.toml       # cargo xtask alias & AX_CONFIG_PATH
├── xtask/
│   └── src/
│       └── main.rs       # build/run tool (pflash image creation + QEMU launch)
├── configs/
│   ├── riscv64.toml      # Platform config with PFlash MMIO range
│   ├── aarch64.toml
│   ├── x86_64.toml
│   └── loongarch64.toml
├── src/
│   └── main.rs           # Producer-consumer message queue with cooperative scheduling
├── build.rs              # Linker script path setup (auto-detects arch)
├── Cargo.toml            # Dependencies (axstd with paging + multitask features)
└── README.md
```

## How It Works

The `cargo xtask` pattern uses a host-native helper crate (`xtask/`) to orchestrate
cross-compilation and QEMU execution:

1. **`cargo xtask build --arch <ARCH>`**
   - Copies `configs/<ARCH>.toml` to `.axconfig.toml`
   - Runs `cargo build --release --target <TARGET> --features axstd`

2. **`cargo xtask run --arch <ARCH>`**
   - Performs the build step above
   - Creates a PFlash image with magic string `"PFLA"` at offset 0
   - For x86_64: embeds SeaBIOS at the end of the pflash image
   - Converts ELF to raw binary via `rust-objcopy` (except x86_64)
   - Launches QEMU with the PFlash image attached

## Key Components

| Component | Role |
|---|---|
| `axstd` | ArceOS standard library (replaces Rust's `std` in `no_std` environment) |
| `axhal` | Hardware abstraction layer, provides `phys_to_virt` for address translation |
| `axtask` | Task scheduler with run queues, enables `thread::spawn` / `thread::join` / `thread::yield_now` |
| `axsync` | Synchronization primitives, provides `SpinNoIrq` for the message queue |
| `paging` feature | Enables page table management; maps MMIO regions listed in config |
| `multitask` feature | Enables multi-task scheduler with cooperative scheduling support |

## License

GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0
