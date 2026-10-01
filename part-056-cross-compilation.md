# Part 56: Cross Compilation ใน Zig (Steps 551-560)

Cross compilation เป็นหนึ่งในความสามารถที่โดดเด่นที่สุดของ Zig โดย Zig มี toolchain ที่ครบครันสำหรับการ compile code ให้รันบน platform ที่แตกต่างจาก host machine ในบทนี้เราจะเรียนรู้วิธีการใช้ cross compilation อย่างละเอียด

---

## Step 551: Cross Compilation คืออะไร? Zig's Toolchain

### ความหมายของ Cross Compilation

Cross compilation หมายถึงการ compile โค้ดบนเครื่องหนึ่ง (host) เพื่อให้รันบนอีก platform หนึ่ง (target) ที่มี architecture หรือ operating system แตกต่างกัน

**ตัวอย่างเช่น:**
- Compile บน Linux x86_64 เพื่อให้รันบน Windows ARM64
- Compile บน macOS Apple Silicon เพื่อให้รันบน Linux RISC-V
- Compile บน Windows x64 เพื่อให้รันบน embedded ARM Cortex-M

### ทำไม Cross Compilation จึงสำคัญ?

```
ปัญหาในอดีต:
- ต้องมีเครื่องจริงหรือ VM ของทุก target platform
- Setup toolchain ซับซ้อน (cross-gcc, sysroot, libraries)
- Version mismatch ระหว่าง libraries
- Debug ยาก เพราะ error เกิดที่ target ไม่ใช่ host

ข้อดีของ Zig:
- ไม่ต้องติดตั้ง toolchain พิเศษ
- Zig bundled ทุกอย่างที่จำเป็น
- Single command สำหรับทุก target
- Reproducible builds
```

### Zig Toolchain Architecture

Zig มาพร้อมกับ:

```
zig compiler
├── LLVM backend (สำหรับ code generation)
├── LLD linker (สำหรับ linking)
├── libc headers สำหรับทุก target
├── compiler-rt (runtime support)
└── musl/glibc/mingw-w64 (C standard library)
```

**ตรวจสอบ version:**
```bash
zig version
# 0.13.0 (หรือ version ที่ใหม่กว่า)

zig env
# แสดง information เกี่ยวกับ Zig installation
```

### ทดสอบ Basic Cross Compilation

สร้างไฟล์ `hello.zig`:
```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello from cross-compiled world!\n", .{});
}
```

Compile สำหรับ Windows จาก Linux:
```bash
zig build-exe hello.zig -target x86_64-windows
# สร้าง hello.exe

zig build-exe hello.zig -target aarch64-linux
# สร้าง hello (binary สำหรับ ARM64 Linux)

zig build-exe hello.zig -target x86_64-macos
# สร้าง hello สำหรับ macOS Intel
```

---

## Step 552: Target Specification (CPU, OS, ABI)

### รูปแบบ Target Triple

Zig ใช้ format: `<cpu>-<os>-<abi>`

```
cpu:  สถาปัตยกรรม CPU เช่น x86_64, aarch64, riscv64
os:   ระบบปฏิบัติการ เช่น linux, windows, macos, freestanding
abi:  Application Binary Interface เช่น gnu, musl, msvc, none
```

**ตัวอย่าง targets:**
```bash
# Linux targets
x86_64-linux-gnu        # Linux x86_64 with glibc
x86_64-linux-musl       # Linux x86_64 with musl libc
aarch64-linux-gnu       # Linux ARM64 with glibc
aarch64-linux-musl      # Linux ARM64 with musl libc
riscv64-linux-musl      # Linux RISC-V 64 with musl

# Windows targets
x86_64-windows-msvc     # Windows x64 with MSVC ABI
x86_64-windows-gnu      # Windows x64 with GNU ABI (MinGW)
aarch64-windows         # Windows ARM64

# macOS targets
x86_64-macos            # macOS Intel
aarch64-macos           # macOS Apple Silicon (M1/M2/M3)

# Embedded (freestanding)
arm-freestanding-eabi   # ARM Cortex-M (bare metal)
thumb-freestanding-eabi # Thumb mode ARM
riscv32-freestanding    # RISC-V 32-bit bare metal
```

### CPU Features

นอกจาก architecture ยังสามารถระบุ CPU features ได้:

```bash
# ระบุ CPU model
zig build-exe main.zig -target x86_64-linux -mcpu skylake
zig build-exe main.zig -target aarch64-linux -mcpu cortex_a72

# Enable/disable specific features
zig build-exe main.zig -target x86_64-linux -mcpu baseline+avx2
zig build-exe main.zig -target x86_64-linux -mcpu native-avx512f

# ใช้ native (build สำหรับ current machine)
zig build-exe main.zig -target native
zig build-exe main.zig  # ค่าเริ่มต้นคือ native
```

### ใน Build Script

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Target ที่กำหนดเอง
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .x86_64,
        .os_tag = .linux,
        .abi = .musl,
    });

    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);
}
```

### ตัวอย่างการระบุ CPU Features ใน Build Script

```zig
// build.zig - กำหนด CPU features อย่างละเอียด
const std = @import("std");

pub fn build(b: *std.Build) void {
    // สร้าง target พร้อม CPU features
    var cpu_model = std.Target.Cpu.Model.generic(.x86_64);
    
    const target_query = std.Target.Query{
        .cpu_arch = .x86_64,
        .os_tag = .linux,
        .abi = .gnu,
        .cpu_model = .{ .explicit = &std.Target.x86.cpu.skylake },
    };
    
    const target = b.resolveTargetQuery(target_query);
    
    const exe = b.addExecutable(.{
        .name = "optimized_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseFast,
    });
    
    b.installArtifact(exe);
}
```

---

## Step 553: zig targets - List All Targets

### ดู Targets ที่รองรับ

```bash
# แสดงทุก target ที่รองรับ
zig targets

# Output จะแสดง:
# - CPUs ที่รองรับ
# - OSes ที่รองรับ
# - ABIs ที่รองรับ
# - CPU features
```

### ตัวอย่าง Output

```json
{
  "zig_version": "0.13.0",
  "target": {
    "cpu": {
      "arch": "x86_64",
      "name": "skylake",
      "features": {
        "sse4_2": true,
        "avx2": true,
        "aes": true
      }
    },
    "os": "linux",
    "abi": "gnu"
  },
  "native_paths": {
    "include_dirs": ["/usr/include"],
    "lib_dirs": ["/usr/lib"],
    "rpaths": []
  }
}
```

### Script ตรวจสอบ Supported Targets

```bash
#!/bin/bash
# check_targets.sh - ตรวจสอบ targets ที่รองรับ

echo "=== Common targets supported by Zig ==="

# Linux
LINUX_TARGETS=(
    "x86_64-linux-gnu"
    "x86_64-linux-musl"
    "aarch64-linux-gnu"
    "aarch64-linux-musl"
    "arm-linux-gnueabihf"
    "riscv64-linux-musl"
    "mips-linux-musl"
    "powerpc64le-linux-musl"
)

# Windows
WINDOWS_TARGETS=(
    "x86_64-windows-gnu"
    "x86_64-windows-msvc"
    "aarch64-windows"
    "x86-windows-gnu"
)

# macOS
MACOS_TARGETS=(
    "x86_64-macos"
    "aarch64-macos"
)

echo "--- Linux targets ---"
for target in "${LINUX_TARGETS[@]}"; do
    echo "Testing: $target"
    echo 'pub fn main() void {}' | zig build-exe -target "$target" - 2>/dev/null && \
        echo "  [OK]" || echo "  [FAIL]"
done
```

### Programmatic Target Discovery

```zig
// list_targets.zig - แสดง target information ด้วย Zig code
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Current target information:\n", .{});
    try stdout.print("  CPU Architecture: {s}\n", .{@tagName(builtin.cpu.arch)});
    try stdout.print("  OS: {s}\n", .{@tagName(builtin.os.tag)});
    try stdout.print("  ABI: {s}\n", .{@tagName(builtin.abi)});
    try stdout.print("  Endianness: {s}\n", .{@tagName(builtin.cpu.arch.endian())});
    
    // ตรวจสอบ CPU features
    try stdout.print("\nCPU Features:\n", .{});
    
    if (builtin.cpu.arch == .x86_64) {
        const features = builtin.cpu.features;
        const x86_features = std.Target.x86.Feature;
        
        if (std.Target.x86.featureSetHas(features, .avx2)) {
            try stdout.print("  AVX2: enabled\n", .{});
        }
        if (std.Target.x86.featureSetHas(features, .sse4_2)) {
            try stdout.print("  SSE4.2: enabled\n", .{});
        }
        if (std.Target.x86.featureSetHas(features, .aes)) {
            try stdout.print("  AES-NI: enabled\n", .{});
        }
    }
    
    // Pointer size
    try stdout.print("\nPointer size: {} bytes\n", .{@sizeOf(usize)});
    try stdout.print("Max alignment: {}\n", .{std.Target.stack_align});
}
```

---

## Step 554: Compile for Linux ARM64

### พื้นฐาน ARM64 Linux Compilation

```bash
# Compile สำหรับ ARM64 Linux (glibc)
zig build-exe main.zig -target aarch64-linux-gnu

# Compile สำหรับ ARM64 Linux (musl - static linking)
zig build-exe main.zig -target aarch64-linux-musl

# ตรวจสอบ binary
file main
# main: ELF 64-bit LSB executable, ARM aarch64...
```

### ตัวอย่าง Program สำหรับ ARM64

```zig
// arm64_demo.zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ตรวจสอบ runtime architecture
    try stdout.print("Running on: {s}-{s}\n", .{
        @tagName(builtin.cpu.arch),
        @tagName(builtin.os.tag),
    });
    
    // ARM64 specific: NEON SIMD operations
    if (builtin.cpu.arch == .aarch64) {
        try stdout.print("ARM64 NEON available\n", .{});
        // ใช้ SIMD operations
        var v1 = @Vector(4, f32){ 1.0, 2.0, 3.0, 4.0 };
        var v2 = @Vector(4, f32){ 5.0, 6.0, 7.0, 8.0 };
        const result = v1 + v2;
        try stdout.print("SIMD result: {d}\n", .{result});
    }
    
    // Cross-platform memory allocation
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer);
    
    try stdout.print("Allocated {} bytes successfully\n", .{buffer.len});
}
```

### Build Script สำหรับ ARM64 Linux

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // สร้าง targets หลายตัว
    const targets = [_]std.Target.Query{
        // ARM64 Linux glibc (dynamic linking)
        .{
            .cpu_arch = .aarch64,
            .os_tag = .linux,
            .abi = .gnu,
        },
        // ARM64 Linux musl (static linking)
        .{
            .cpu_arch = .aarch64,
            .os_tag = .linux,
            .abi = .musl,
        },
        // ARM64 Linux musl with Cortex-A72 optimization
        .{
            .cpu_arch = .aarch64,
            .os_tag = .linux,
            .abi = .musl,
            .cpu_model = .{ .explicit = &std.Target.aarch64.cpu.cortex_a72 },
        },
    };
    
    const optimize = b.standardOptimizeOption(.{});
    
    for (targets, 0..) |target_query, i| {
        const target = b.resolveTargetQuery(target_query);
        const name = b.fmt("myapp-arm64-{}", .{i});
        
        const exe = b.addExecutable(.{
            .name = name,
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        });
        
        b.installArtifact(exe);
    }
}
```

### การ Deploy บน Raspberry Pi

```bash
# Compile สำหรับ Raspberry Pi 4 (ARM64)
zig build-exe main.zig \
    -target aarch64-linux-musl \
    -mcpu cortex_a72 \
    -O ReleaseFast

# Copy ไปที่ Pi
scp main pi@raspberrypi.local:~/
ssh pi@raspberrypi.local "./main"
```

---

## Step 555: Compile for Windows x64

### Windows Cross Compilation

```bash
# Compile สำหรับ Windows x64 (GNU ABI / MinGW)
zig build-exe main.zig -target x86_64-windows-gnu

# Compile สำหรับ Windows x64 (MSVC ABI)
zig build-exe main.zig -target x86_64-windows-msvc

# สร้าง .exe file
ls -la
# main.exe  (พร้อม run บน Windows)
```

### ตัวอย่าง Windows-Specific Code

```zig
// windows_demo.zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    if (builtin.os.tag == .windows) {
        try stdout.print("Running on Windows!\n", .{});
        
        // Windows-specific: ดึง Windows version
        const windows = std.os.windows;
        
        // เปิดไฟล์ด้วย Windows API
        try windowsFileDemo();
        
        // Windows environment variables
        if (std.process.getEnvVarOwned(std.heap.page_allocator, "USERPROFILE")) |path| {
            defer std.heap.page_allocator.free(path);
            try stdout.print("User profile: {s}\n", .{path});
        } else |_| {}
        
    } else {
        try stdout.print("Not Windows - this demonstrates cross-platform code\n", .{});
    }
}

fn windowsFileDemo() !void {
    if (builtin.os.tag != .windows) return;
    
    // สร้างไฟล์บน Windows
    const file = try std.fs.cwd().createFile("test_windows.txt", .{});
    defer file.close();
    
    try file.writeAll("Hello from Windows!\r\n");
}
```

### Windows GUI Application

```zig
// windows_gui.zig - Windows GUI application
const std = @import("std");

// Windows API declarations
const HWND = *opaque {};
const HINSTANCE = *opaque {};
const WPARAM = usize;
const LPARAM = isize;
const LRESULT = isize;

extern "user32" fn MessageBoxA(
    hWnd: ?HWND,
    lpText: [*:0]const u8,
    lpCaption: [*:0]const u8,
    uType: u32,
) callconv(.winapi) i32;

pub fn main() void {
    _ = MessageBoxA(null, "Hello from Zig!", "Zig Cross Compilation", 0);
}
```

### Build Script สำหรับ Windows

```zig
// build.zig - Windows targets
const std = @import("std");

pub fn build(b: *std.Build) void {
    const windows_target = b.resolveTargetQuery(.{
        .cpu_arch = .x86_64,
        .os_tag = .windows,
        .abi = .gnu,
    });
    
    const optimize = b.standardOptimizeOption(.{});
    
    // Console application
    const console_exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = windows_target,
        .optimize = optimize,
    });
    
    // Link Windows libraries
    console_exe.linkSystemLibrary("kernel32");
    console_exe.linkSystemLibrary("user32");
    
    b.installArtifact(console_exe);
    
    // DLL
    const dll = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = windows_target,
        .optimize = optimize,
        .version = .{ .major = 1, .minor = 0, .patch = 0 },
    });
    
    b.installArtifact(dll);
}
```

---

## Step 556: Compile for macOS

### macOS Cross Compilation

```bash
# Compile สำหรับ macOS Intel
zig build-exe main.zig -target x86_64-macos

# Compile สำหรับ macOS Apple Silicon
zig build-exe main.zig -target aarch64-macos

# สร้าง Universal Binary (Fat Binary)
# ต้อง compile แยกแล้วรวมด้วย lipo
zig build-exe main.zig -target x86_64-macos -o main_x86
zig build-exe main.zig -target aarch64-macos -o main_arm
lipo -create main_x86 main_arm -output main_universal
```

### macOS Version Targeting

```bash
# ระบุ macOS version ขั้นต่ำ
zig build-exe main.zig \
    -target aarch64-macos \
    -mmacos-version-min=12.0

# Compile สำหรับ macOS 14 Sonoma
zig build-exe main.zig \
    -target aarch64-macos.14.0...14.99
```

### ตัวอย่าง macOS-Specific Code

```zig
// macos_demo.zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Platform: {s}\n", .{@tagName(builtin.os.tag)});
    
    if (builtin.os.tag == .macos) {
        try macosDemo(stdout);
    }
}

fn macosDemo(writer: anytype) !void {
    // macOS-specific paths
    const home = std.posix.getenv("HOME") orelse "/tmp";
    try writer.print("Home directory: {s}\n", .{home});
    
    // macOS Library paths
    try writer.print("Library path: {s}/Library\n", .{home});
    
    // Darwin syscalls
    const pid = std.os.linux.getpid(); // cross-platform
    try writer.print("Process ID: {}\n", .{pid});
}
```

### Universal Binary Build Script

```zig
// build.zig - สร้าง Universal Binary สำหรับ macOS
const std = @import("std");

pub fn build(b: *std.Build) void {
    const optimize = b.standardOptimizeOption(.{});
    
    // Intel macOS
    const macos_x86 = b.resolveTargetQuery(.{
        .cpu_arch = .x86_64,
        .os_tag = .macos,
    });
    
    // Apple Silicon macOS
    const macos_arm = b.resolveTargetQuery(.{
        .cpu_arch = .aarch64,
        .os_tag = .macos,
    });
    
    const exe_x86 = b.addExecutable(.{
        .name = "myapp_x86",
        .root_source_file = b.path("src/main.zig"),
        .target = macos_x86,
        .optimize = optimize,
    });
    
    const exe_arm = b.addExecutable(.{
        .name = "myapp_arm",
        .root_source_file = b.path("src/main.zig"),
        .target = macos_arm,
        .optimize = optimize,
    });
    
    b.installArtifact(exe_x86);
    b.installArtifact(exe_arm);
    
    // Step สำหรับ lipo รวม universal binary
    const lipo_step = b.addSystemCommand(&.{
        "lipo", "-create",
        "-output", "zig-out/bin/myapp",
        "zig-out/bin/myapp_x86",
        "zig-out/bin/myapp_arm",
    });
    lipo_step.step.dependOn(&exe_x86.step);
    lipo_step.step.dependOn(&exe_arm.step);
    
    const universal_step = b.step("universal", "Build universal binary");
    universal_step.dependOn(&lipo_step.step);
}
```

---

## Step 557: Compile for Embedded Targets

### Bare Metal Programming

Embedded targets ใช้ `freestanding` OS (ไม่มี OS):

```bash
# ARM Cortex-M4 (ใช้ใน STM32, nRF52)
zig build-exe firmware.zig \
    -target thumb-freestanding-eabi \
    -mcpu cortex_m4

# ARM Cortex-M7
zig build-exe firmware.zig \
    -target thumb-freestanding-eabi \
    -mcpu cortex_m7f+fp

# RISC-V 32-bit embedded
zig build-exe firmware.zig \
    -target riscv32-freestanding-none

# AVR (Arduino)
zig build-exe firmware.zig \
    -target avr-freestanding-none \
    -mcpu atmega328p
```

### ตัวอย่าง Bare Metal Firmware

```zig
// firmware.zig - ARM Cortex-M4 bare metal
const std = @import("std");

// Memory layout สำหรับ STM32F4
const GPIOA_BASE: u32 = 0x40020000;
const RCC_BASE: u32 = 0x40023800;

const GPIOA_MODER: *volatile u32 = @ptrFromInt(GPIOA_BASE + 0x00);
const GPIOA_ODR: *volatile u32 = @ptrFromInt(GPIOA_BASE + 0x14);
const RCC_AHB1ENR: *volatile u32 = @ptrFromInt(RCC_BASE + 0x30);

// Vector table
export var vector_table linksection(".vectors") = [_]usize{
    0x20020000, // Stack pointer
    @intFromPtr(&reset_handler),
    @intFromPtr(&default_handler),
    @intFromPtr(&default_handler),
};

export fn reset_handler() callconv(.C) noreturn {
    // เปิด clock สำหรับ GPIOA
    RCC_AHB1ENR.* |= (1 << 0);
    
    // ตั้ง PA5 เป็น output
    GPIOA_MODER.* = (GPIOA_MODER.* & ~(@as(u32, 0x3) << 10)) | (0x1 << 10);
    
    // LED blink loop
    while (true) {
        GPIOA_ODR.* |= (1 << 5);   // LED on
        delay(1000000);
        GPIOA_ODR.* &= ~(@as(u32, 1) << 5);  // LED off
        delay(1000000);
    }
}

fn delay(count: u32) void {
    var i: u32 = 0;
    while (i < count) : (i += 1) {
        asm volatile ("nop");
    }
}

fn default_handler() callconv(.C) noreturn {
    while (true) {}
}
```

### Linker Script สำหรับ STM32F4

```ld
/* stm32f4.ld */
MEMORY {
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1M
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
    .vectors : {
        KEEP(*(.vectors))
    } > FLASH

    .text : {
        *(.text*)
        *(.rodata*)
    } > FLASH

    .data : {
        _data_start = .;
        *(.data*)
        _data_end = .;
    } > RAM AT > FLASH

    .bss : {
        _bss_start = .;
        *(.bss*)
        *(COMMON)
        _bss_end = .;
    } > RAM
}
```

### Build สำหรับ Embedded

```zig
// build.zig - embedded target
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .thumb,
        .os_tag = .freestanding,
        .abi = .eabi,
        .cpu_model = .{ .explicit = &std.Target.arm.cpu.cortex_m4 },
    });
    
    const firmware = b.addExecutable(.{
        .name = "firmware",
        .root_source_file = b.path("src/firmware.zig"),
        .target = target,
        .optimize = .ReleaseSmall,
        .strip = false,
    });
    
    // ใช้ custom linker script
    firmware.setLinkerScript(b.path("stm32f4.ld"));
    
    // ไม่ link libc
    firmware.pie = false;
    
    b.installArtifact(firmware);
    
    // Step สำหรับสร้าง .bin file
    const objcopy = b.addObjCopy(firmware.getEmittedBin(), .{
        .format = .bin,
    });
    
    const bin_step = b.step("bin", "Create binary file");
    const install_bin = b.addInstallBinFile(objcopy.getOutput(), "firmware.bin");
    bin_step.dependOn(&install_bin.step);
}
```

---

## Step 558: Cross-Compiling C/C++ ด้วย Zig

### Zig เป็น C Compiler

Zig สามารถใช้เป็น drop-in replacement สำหรับ `cc` และ `c++`:

```bash
# Compile C code ด้วย Zig
zig cc hello.c -o hello

# Cross compile C code
zig cc hello.c -o hello -target aarch64-linux-musl

# Compile C++ ด้วย Zig
zig c++ hello.cpp -o hello

# Cross compile C++ code
zig c++ hello.cpp -o hello_arm -target aarch64-linux-gnu
```

### ตัวอย่าง C/Zig Mixed Project

```c
// helper.c - C code ที่จะ compile ด้วย Zig
#include <stdio.h>
#include <string.h>

void greet(const char* name) {
    printf("Hello, %s!\n", name);
}

int add_numbers(int a, int b) {
    return a + b;
}

size_t string_length(const char* str) {
    return strlen(str);
}
```

```zig
// main.zig - ใช้ C functions
const std = @import("std");

extern fn greet(name: [*:0]const u8) void;
extern fn add_numbers(a: c_int, b: c_int) c_int;
extern fn string_length(str: [*:0]const u8) usize;

pub fn main() !void {
    greet("World");
    
    const sum = add_numbers(10, 20);
    std.debug.print("10 + 20 = {}\n", .{sum});
    
    const len = string_length("Hello");
    std.debug.print("Length of 'Hello' = {}\n", .{len});
}
```

### Build Script สำหรับ Mixed C/Zig

```zig
// build.zig - C/Zig mixed project
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "mixed_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // เพิ่ม C source files
    exe.addCSourceFile(.{
        .file = b.path("src/helper.c"),
        .flags = &.{"-std=c11", "-Wall"},
    });
    
    // เพิ่ม include paths
    exe.addIncludePath(b.path("include"));
    
    // Link C standard library
    exe.linkLibC();
    
    b.installArtifact(exe);
}
```

### Cross-Compile CMake Project ด้วย Zig

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject C CXX)

set(CMAKE_C_COMPILER "zig")
set(CMAKE_C_COMPILER_ARG1 "cc")
set(CMAKE_CXX_COMPILER "zig")
set(CMAKE_CXX_COMPILER_ARG1 "c++")

add_executable(myapp main.c helper.cpp)
```

```bash
# Configure สำหรับ ARM64 Linux
cmake -B build \
    -DCMAKE_C_COMPILER="zig cc" \
    -DCMAKE_C_FLAGS="-target aarch64-linux-musl" \
    -DCMAKE_CXX_COMPILER="zig c++" \
    -DCMAKE_CXX_FLAGS="-target aarch64-linux-musl"

cmake --build build
```

---

## Step 559: Build Scripts สำหรับ Multiple Targets

### Comprehensive Multi-Target Build

```zig
// build.zig - Build สำหรับหลาย targets พร้อมกัน
const std = @import("std");

const Target = struct {
    name: []const u8,
    query: std.Target.Query,
};

const targets = [_]Target{
    .{
        .name = "linux-x64",
        .query = .{ .cpu_arch = .x86_64, .os_tag = .linux, .abi = .musl },
    },
    .{
        .name = "linux-arm64",
        .query = .{ .cpu_arch = .aarch64, .os_tag = .linux, .abi = .musl },
    },
    .{
        .name = "windows-x64",
        .query = .{ .cpu_arch = .x86_64, .os_tag = .windows, .abi = .gnu },
    },
    .{
        .name = "macos-x64",
        .query = .{ .cpu_arch = .x86_64, .os_tag = .macos },
    },
    .{
        .name = "macos-arm64",
        .query = .{ .cpu_arch = .aarch64, .os_tag = .macos },
    },
};

pub fn build(b: *std.Build) void {
    const optimize = b.standardOptimizeOption(.{});
    
    // Build step สำหรับ all targets
    const all_step = b.step("all", "Build for all targets");
    
    for (targets) |t| {
        const target = b.resolveTargetQuery(t.query);
        
        const exe = b.addExecutable(.{
            .name = b.fmt("myapp-{s}", .{t.name}),
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        });
        
        const install = b.addInstallArtifact(exe, .{
            .dest_dir = .{ .override = .{ .custom = "release" } },
        });
        
        all_step.dependOn(&install.step);
    }
}
```

### Build Script พร้อม Version Tagging

```zig
// build.zig - พร้อม version information
const std = @import("std");

pub fn build(b: *std.Build) void {
    const version = std.SemanticVersion{ .major = 1, .minor = 2, .patch = 3 };
    const version_str = b.fmt("{}", .{version});
    
    const options = b.addOptions();
    options.addOption([]const u8, "version", version_str);
    options.addOption([]const u8, "git_commit", getGitCommit(b) orelse "unknown");
    
    const targets = [_]struct {
        name: []const u8,
        query: std.Target.Query,
    }{
        .{ .name = "linux-amd64", .query = .{ .cpu_arch = .x86_64, .os_tag = .linux, .abi = .musl } },
        .{ .name = "linux-arm64", .query = .{ .cpu_arch = .aarch64, .os_tag = .linux, .abi = .musl } },
        .{ .name = "windows-amd64", .query = .{ .cpu_arch = .x86_64, .os_tag = .windows, .abi = .gnu } },
        .{ .name = "darwin-amd64", .query = .{ .cpu_arch = .x86_64, .os_tag = .macos } },
        .{ .name = "darwin-arm64", .query = .{ .cpu_arch = .aarch64, .os_tag = .macos } },
    };
    
    const optimize: std.builtin.OptimizeMode = .ReleaseFast;
    
    for (targets) |t| {
        const target = b.resolveTargetQuery(t.query);
        
        const exe = b.addExecutable(.{
            .name = "myapp",
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        });
        
        exe.root_module.addOptions("build_options", options);
        
        // Install ใน directory ที่มีชื่อตาม target
        const output_dir = b.fmt("dist/{s}", .{t.name});
        const install = b.addInstallArtifact(exe, .{
            .dest_dir = .{ .override = .{ .custom = output_dir } },
        });
        
        b.getInstallStep().dependOn(&install.step);
    }
}

fn getGitCommit(b: *std.Build) ?[]const u8 {
    const result = b.runAllowFail(
        &.{ "git", "rev-parse", "--short", "HEAD" },
        &.{},
        .Ignore,
    ) catch return null;
    return std.mem.trimRight(u8, result, "\n");
}
```

### Makefile สำหรับ Orchestration

```makefile
# Makefile
VERSION := 1.2.3
TARGETS := linux-x64 linux-arm64 windows-x64 macos-x64 macos-arm64

.PHONY: all clean release $(TARGETS)

all: $(TARGETS)

linux-x64:
	zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseFast
	cp zig-out/bin/myapp dist/myapp-linux-x64

linux-arm64:
	zig build -Dtarget=aarch64-linux-musl -Doptimize=ReleaseFast
	cp zig-out/bin/myapp dist/myapp-linux-arm64

windows-x64:
	zig build -Dtarget=x86_64-windows-gnu -Doptimize=ReleaseFast
	cp zig-out/bin/myapp.exe dist/myapp-windows-x64.exe

macos-x64:
	zig build -Dtarget=x86_64-macos -Doptimize=ReleaseFast
	cp zig-out/bin/myapp dist/myapp-macos-x64

macos-arm64:
	zig build -Dtarget=aarch64-macos -Doptimize=ReleaseFast
	cp zig-out/bin/myapp dist/myapp-macos-arm64

release: all
	@echo "Creating release archives..."
	for target in $(TARGETS); do \
		cd dist && tar czf myapp-$(VERSION)-$$target.tar.gz myapp-$$target; \
	done

clean:
	rm -rf zig-out zig-cache dist
```

---

## Step 560: Distribution Packaging

### สร้าง Release Package

```zig
// build.zig - สำหรับ distribution packaging
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const version = std.SemanticVersion{ .major = 2, .minor = 0, .patch = 1 };
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    b.installArtifact(exe);
    
    // Step สำหรับสร้าง release package
    const release_step = b.step("release", "Create release package");
    
    // สร้าง tarball
    const tarball_name = b.fmt(
        "myapp-{}-{s}.tar.gz",
        .{ version, target.result.zigTriple(b.allocator) catch "unknown" },
    );
    
    const tar = b.addSystemCommand(&.{
        "tar", "czf",
        b.fmt("dist/{s}", .{tarball_name}),
        "-C", "zig-out",
        "bin/myapp",
    });
    tar.step.dependOn(b.getInstallStep());
    release_step.dependOn(&tar.step);
}
```

### GitHub Actions สำหรับ Cross Compilation Release

```yaml
# .github/workflows/release.yml
name: Cross-Platform Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        target:
          - name: linux-x64
            triple: x86_64-linux-musl
          - name: linux-arm64
            triple: aarch64-linux-musl
          - name: windows-x64
            triple: x86_64-windows-gnu
          - name: macos-x64
            triple: x86_64-macos
          - name: macos-arm64
            triple: aarch64-macos
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Zig
        uses: mlugg/setup-zig@v1
        with:
          version: 0.13.0
      
      - name: Build
        run: |
          zig build \
            -Dtarget=${{ matrix.target.triple }} \
            -Doptimize=ReleaseFast
      
      - name: Package
        run: |
          mkdir -p dist
          if [[ "${{ matrix.target.name }}" == windows* ]]; then
            zip dist/myapp-${{ matrix.target.name }}.zip zig-out/bin/myapp.exe
          else
            tar czf dist/myapp-${{ matrix.target.name }}.tar.gz -C zig-out bin/myapp
          fi
      
      - name: Upload Release
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*
```

### Docker สำหรับ Cross Compilation

```dockerfile
# Dockerfile - cross compilation environment
FROM debian:bookworm-slim

# ติดตั้ง Zig
RUN apt-get update && apt-get install -y \
    wget \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

ENV ZIG_VERSION=0.13.0
RUN wget https://ziglang.org/download/${ZIG_VERSION}/zig-linux-x86_64-${ZIG_VERSION}.tar.xz && \
    tar xf zig-linux-x86_64-${ZIG_VERSION}.tar.xz && \
    mv zig-linux-x86_64-${ZIG_VERSION} /usr/local/zig && \
    rm zig-linux-x86_64-${ZIG_VERSION}.tar.xz

ENV PATH="/usr/local/zig:${PATH}"

WORKDIR /app
COPY . .

# Build สำหรับทุก targets
RUN mkdir -p dist && \
    for target in \
        x86_64-linux-musl \
        aarch64-linux-musl \
        x86_64-windows-gnu \
        x86_64-macos \
        aarch64-macos; do \
        zig build-exe src/main.zig \
            -target ${target} \
            -O ReleaseFast \
            -o dist/myapp-${target} || true; \
    done
```

### การ Checksum Verification

```bash
#!/bin/bash
# create_checksums.sh

DIST_DIR="dist"
CHECKSUM_FILE="${DIST_DIR}/checksums.sha256"

echo "Creating SHA256 checksums..."
cd "${DIST_DIR}"

# สร้าง checksums
sha256sum myapp-* > checksums.sha256

echo "Checksums created:"
cat checksums.sha256

# Verify
sha256sum -c checksums.sha256
echo "All checksums verified!"
```

### สรุป Cross Compilation ใน Zig

```
ความสามารถของ Zig Cross Compilation:
┌────────────────────────────────────────────────────────┐
│ Target Support                                         │
│ ├─ Linux: x86, x64, ARM, ARM64, RISC-V, MIPS, PPC    │
│ ├─ Windows: x86, x64, ARM64                           │
│ ├─ macOS: x64, ARM64 (Universal Binary)               │
│ ├─ FreeBSD, OpenBSD, NetBSD                           │
│ ├─ WebAssembly                                        │
│ └─ Bare metal / Embedded                              │
│                                                       │
│ ข้อดี                                                 │
│ ├─ ไม่ต้องติดตั้ง toolchain เพิ่ม                    │
│ ├─ Single command cross compilation                   │
│ ├─ Built-in libc สำหรับทุก platform                  │
│ ├─ Reproducible builds                               │
│ └─ ใช้เป็น CC/CXX สำหรับ C/C++ projects             │
└────────────────────────────────────────────────────────┘
```

---

**บทสรุป Part 56:**

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Cross Compilation ใน Zig ซึ่งเป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุด:

1. **Step 551**: เข้าใจ Cross Compilation และ Zig Toolchain ที่รวมทุกอย่างไว้แล้ว
2. **Step 552**: การระบุ target ด้วย CPU, OS, ABI และ CPU features
3. **Step 553**: การดู targets ที่รองรับทั้งหมด
4. **Step 554**: Compile สำหรับ Linux ARM64 รวมถึง Raspberry Pi
5. **Step 555**: Compile สำหรับ Windows x64 ทั้ง console และ GUI
6. **Step 556**: Compile สำหรับ macOS รวมถึง Universal Binary
7. **Step 557**: Embedded targets สำหรับ microcontrollers
8. **Step 558**: ใช้ Zig เป็น C/C++ cross compiler
9. **Step 559**: Build scripts สำหรับ multiple targets พร้อมกัน
10. **Step 560**: Distribution packaging และ release automation

Zig ทำให้ Cross Compilation เป็นเรื่องง่ายมาก เพียงระบุ target ก็สามารถ compile ได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม
