# GDB Stub for D3OS

A **GDB stub for the D3OS operating system**, built as a bachelor's thesis at
Heinrich Heine University Düsseldorf. It lets you debug the D3OS kernel with
the GNU Debugger (GDB) **on real hardware**, from a second computer connected
by a serial cable, as well as in QEMU.

The stub runs inside the kernel and speaks the
[GDB Remote Serial Protocol](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Remote-Protocol.html)
over a serial port. It supports:

- software breakpoints (`break`) and up to four hardware breakpoints (`hbreak`)
- single-stepping by source line (`step`, `next`) and by instruction (`stepi`, `nexti`)
- reading and writing registers and memory
- inspecting variables and struct fields (`print`)
- listing kernel threads and switching between them
- stopping the running system with **Ctrl+C**

[D3OS](https://github.com/hhu-bsinfo/D3OS) is an operating system written in
Rust by the operating systems group at HHU Düsseldorf. The protocol handling
is done by the [`gdbstub`](https://crates.io/crates/gdbstub) crate. This
project implements the target side: CPU state, memory, breakpoints, stepping,
interrupts, and the serial connection.

> The thesis itself is written in German:
> [`document/Thesis Abgabe/thesis.pdf`](document/Thesis%20Abgabe/thesis.pdf).

## Repository layout

```text
document/
├── Thesis Abgabe/     the submitted thesis: LaTeX sources (chap/*.tex) and thesis.pdf
└── ...                the university's LaTeX thesis template
project/               D3OS with the GDB stub integrated (the version that runs on real hardware)
└── os/kernel/src/
    ├── gdbstub/
    │   ├── event_loop.rs           stub thread and the blocking event loop
    │   ├── gdbtarget.rs            the gdbstub Target: registers, memory, breakpoints, stepping, trap handlers
    │   ├── gdbconnection.rs        serial connection used by the stub (COM3, 115200 baud)
    │   └── debug_state.rs          state shared by the trap handlers and the stub thread
    ├── gdb_interrupt_handler.asm   assembly entry points for #DB and #BP
    ├── device/serial.rs            serial port driver (extended with polled reading)
    └── serialtest/testserial.rs    gdb_break_here(): sample code for trying out the stub
```

## Building

Build on Linux (tested on Ubuntu 24.04) or macOS. You need the D3OS build
dependencies:

```sh
# Debian / Ubuntu
apt install rustup build-essential nasm dosfstools wget qemu-system-x86
cargo install --no-default-features cargo-make cargo-license
```

The toolchain used for the thesis was `nightly-2026-01-01`
(rustc 1.94.0-nightly). rustup downloads a nightly toolchain automatically on
the first build. Apart from `gdbstub` and `gdbstub_arch`, no libraries were
added to D3OS.

The stub is behind the Cargo feature **`gdbstub`**. When enabled, it halts
the system at boot until GDB connects. A normal D3OS build therefore doesn't
include it, and you build the debug-enabled kernel with a separate task:

```sh
cd project
cargo make --no-workspace gdbstub
```

This builds the kernel with the `gdbstub` feature, creates the boot image
`d3os.img`, and starts it in QEMU. QEMU is given three serial ports: COM1 on
the terminal, COM2 to a log file, and **COM3 on TCP port 4321**, where the
stub listens.

## Usage

### In QEMU

Start D3OS with the command above. It halts at boot and waits for GDB. In a
second terminal:

```sh
cd project
rust-gdb loader/kernel.elf
(gdb) target remote localhost:4321
(gdb) continue
```

### On real hardware

1. Build as above. The resulting `project/d3os.img` is the image to boot.
2. Write it to a USB stick, e.g. with [balenaEtcher](https://etcher.balena.io).
3. Connect the target computer to your development computer with a serial
   (null-modem) cable. A USB-to-serial adapter on the development computer
   works.
4. In the target's BIOS/UEFI settings, enable the serial port **COM3**
   (I/O port `0x3E8`). The tests for the thesis ran with Hyper-Threading on
   and all cores enabled.
5. Boot the target from the USB stick. It halts and waits for GDB.
6. On the development computer:

   ```sh
   rust-gdb project/loader/kernel.elf
   (gdb) set serial baud 115200
   (gdb) target remote /dev/ttyUSB0
   (gdb) continue
   ```

On real hardware, the stub must be the **only** user of its serial port, so
loggers and anything else writing to that port must be turned off. To use a
different port, change `Com3` in `os/kernel/src/gdbstub/gdbconnection.rs`.

### From VS Code

`project/.vscode/launch.json` contains a **gdbstub** debug configuration
(requires the C/C++ extension's `cppdbg` debugger). It loads
`loader/kernel.elf` with `rust-gdb` and connects to `localhost:4321`. For
real hardware, replace that address with your serial device.

### GDB commands

| Action | Command |
|---|---|
| Connect | `target remote <device or host:port>` |
| Set the serial speed | `set serial baud 115200` |
| Show protocol packets (for debugging the connection) | `set debug remote 1` |
| Continue | `continue` / `c` |
| Stop the running system | **Ctrl+C** |
| Step into / step over (source line) | `step` / `s`, `next` / `n` |
| Step one instruction | `stepi` / `si`, `nexti` / `ni` |
| Software breakpoint | `break <function / file:line / *address>` |
| Hardware breakpoint (max. 4) | `hbreak <function / file:line / *address>` |
| Delete a breakpoint | `delete <n>` |
| Registers | `info registers` |
| Variable / struct field | `print <name>`, `print <instance>.<field>` |
| Memory | `x/<count><format><size> <address>`, e.g. `x/2dw <addr>` |
| Threads | `info threads`, `thread <n>` |
| Don't step into a function | `skip <function / file>` |
| Load symbols of a user-space program | `add-symbol-file <path>` |

To try these out quickly, set a breakpoint on `gdb_break_here` in
`os/kernel/src/serialtest/testserial.rs`. That function creates variables
and structs and calls a few functions, which makes it convenient for testing
stepping, `print` and memory reads.

## How it works

### Overview

```text
 Development computer                      Target computer (D3OS kernel)
┌──────────────────┐   serial / TCP   ┌─────────────────────────────────────────┐
│ GDB (rust-gdb)   │◄────────────────►│ stub thread (gdbstub crate event loop)  │
└──────────────────┘   GDB RSP        │       ▲ events         │ stop / resume  │
                                      │       │                ▼                │
                                      │ #BP / #DB trap handlers   scheduler     │
                                      │       ▲                                 │
                                      │ int3 (0xCC), TF, DR0–DR7                │
                                      └─────────────────────────────────────────┘
```

### The stub thread

When the kernel boots with the `gdbstub` feature enabled, `boot.rs` starts a
dedicated kernel thread, `init_gdb_stub`. The thread:

1. records its own thread ID in the global `GDB_DEBUG_STATE`, so the trap
   handlers know which thread must keep running while everything else is
   stopped,
2. opens a `GdbStubConnection` on COM3 at 115200 baud,
3. builds a `gdbstub::GdbStub` with a 4 KiB packet buffer and calls
   `run_blocking` with a custom `BlockingEventLoop`.

The event loop's `wait_for_stop_reason` checks two things in turn:

- **Incoming bytes** from GDB, by peeking at the serial port. They are passed
  to the `gdbstub` crate, which parses the protocol packets and calls into
  the target.
- **Debug events** posted by the trap handlers (`SwBreakpoint`,
  `HwBreakpoint`, `SingleStep`, `CtrlC`). Each one is turned into the matching
  stop reason (`SwBreak`, `HwBreak`, `SIGTRAP`, `SIGINT`) and reported to GDB.

When neither is available, the thread yields to the scheduler.

### The serial connection

The `gdbstub` crate needs a byte-oriented connection with `read`, `write` and
`peek`. `GdbStubConnection` implements those on top of D3OS's serial port
driver. Reading is **polled** (`try_read_polled`) instead of interrupt-driven,
and a one-byte buffer provides `peek`. The thesis task description notes that
the existing driver didn't process incoming data reliably, which is why the
driver was analyzed and extended for this work.

### Trap handling

The IDT entries for **vector 1 (#DB, debug exception)** and **vector 3 (#BP,
breakpoint)** point to the assembly stubs in `gdb_interrupt_handler.asm`. Each
stub pushes all 15 general-purpose registers on top of the frame the CPU
already pushed, which gives a complete `GdbTrapFrame`. It then aligns the
stack to 16 bytes and calls the Rust function
`gdb_interrupt_handler(frame, vector)`. After that function returns, it
restores the registers, which may have been changed by GDB in the meantime,
and returns with `iretq`.

The Rust handler:

1. disables interrupts,
2. stores a pointer to the trap frame in the current thread, so GDB can read
   and write that thread's exact register state,
3. posts a debug event to `GDB_DEBUG_STATE`,
4. stops every thread except the stub thread (`debug_stop_all_except`), and
5. switches to the stub thread, which reports the stop to GDB.

When GDB resumes, the stopped threads are released and execution continues
from the (possibly modified) trap frame.

### Breakpoints and stepping

| Feature | Implementation |
|---|---|
| **Software breakpoint** | The original byte at the address is saved and replaced with `0xCC` (`int3`). When the #BP trap fires, the handler moves `rip` back by one byte to point at the breakpoint address. Removing the breakpoint writes the original byte back. |
| **Hardware breakpoint** | The address goes into one of the debug registers **DR0–DR3**, and the matching global-enable bit is set in **DR7**. The CPU raises #DB when it executes that address, without modifying code, so this also works on code that can't be patched. At most four can be active. |
| **Single step** | The **Trap Flag** (bit 8 of `RFLAGS`) is set in the stopped thread's saved state. The CPU executes one instruction and raises #DB. The handler clears the flag again. |
| **Telling #DB causes apart** | The #DB handler reads **DR6**. If one of the bits `TRAP0`–`TRAP3` is set, a hardware breakpoint was hit; otherwise the trap came from a single step. |
| **Ctrl+C** | GDB sends an interrupt byte. The `on_interrupt` callback posts a `CtrlC` event and stops all threads except the stub thread. |

### Threads, registers and memory

The target implements `gdbstub`'s **multi-threaded** API, so every D3OS
kernel thread appears as a thread in GDB (`info threads`).

- **Registers** (`X86_64CoreRegs`) come from the trap frame if the thread was
  stopped by a trap. Otherwise they come from the context the scheduler saved
  when it switched the thread out. Writes go back to the same place.
- **Memory** is read and written directly in the kernel's address space.
- **Scheduler locking** is supported, so GDB can resume only the current
  thread while stepping.

## Limitations

The thesis evaluation lists these limitations:

- Most features only work for **kernel-space** code.
- Breakpoints and step-into don't work in functions on the **exception
  handler path**, since the stub depends on those handlers itself.
- The stub must be the only component using its serial port.
- **Version split:** the version in this repository runs on real hardware
  but is based on an older D3OS. A port to the newest D3OS with its
  multi-core scheduler works only in QEMU with Hyper-Threading off and a
  single core. On real hardware it still crashes because of serial
  transmission problems. That port is in the `development` branch of the
  university's GitLab repository and isn't included here.

Possible extensions named in the thesis are multi-core support, further
investigation of the serial port issues, and **hardware watchpoints**
(breaking on memory access), which the `gdbstub` crate already provides an
interface for.

## License

D3OS (the `project/` directory) is licensed under the GPLv3. See
[`project/LICENSE`](project/LICENSE).
