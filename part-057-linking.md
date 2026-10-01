# Part 57: Linking และ Libraries ใน Zig (Steps 561-570)

การ linking เป็นขั้นตอนสำคัญใน compilation process ที่รวม object files และ libraries เข้าด้วยกันเป็น executable หรือ library ขั้นสุดท้าย ในบทนี้เราจะเรียนรู้เกี่ยวกับ linking และการสร้าง libraries ใน Zig อย่างละเอียด

---

## Step 561: Static vs Dynamic Linking

### ความแตกต่างระหว่าง Static และ Dynamic Linking

**Static Linking:**
```
Source Code → Compile → Object Files → Static Linker → Executable
                                           ↑
                                    Static Libraries (.a, .lib)
                                    (รวมอยู่ใน executable)

ผลลัพธ์: Executable ขนาดใหญ่กว่า แต่ self-contained
```

**Dynamic Linking:**
```
Source Code → Compile → Object Files → Dynamic Linker → Executable
                                            ↑
                                    Dynamic Libraries (.so, .dll, .dylib)
                                    (อ้างอิง ไม่ได้รวมใน executable)

ผลลัพธ์: Executable ขนาดเล็กกว่า แต่ต้องการ shared library ที่ runtime
```

### การเลือก Static หรือ Dynamic Linking

```
Static Linking ดีสำหรับ:
✓ Deployment ง่าย (single binary)
✓ ไม่มี dependency hell
✓ Performance ดีกว่าเล็กน้อย (ไม่ต้อง resolve symbols ที่ runtime)
✓ Cross-platform containers (Docker)
✓ Embedded systems

Dynamic Linking ดีสำหรับ:
✓ ประหยัด memory เมื่อหลาย processes ใช้ library เดียวกัน
✓ Update library โดยไม่ต้อง recompile executable
✓ Plugin systems
✓ ปฏิบัติตาม OS license requirements (LGPL libraries)
```

### ตัวอย่างการเลือก Linking Mode ใน Zig

```zig
// build.zig - เลือก linking mode
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Static linking (default สำหรับ musl target)
    const static_exe = b.addExecutable(.{
        .name = "static_app",
        .root_source_file = b.path("src/main.zig"),
        .target = b.resolveTargetQuery(.{
            .cpu_arch = .x86_64,
            .os_tag = .linux,
            .abi = .musl,  // musl = static linking
        }),
        .optimize = optimize,
    });
    static_exe.linkLibC();
    
    // Dynamic linking (default สำหรับ gnu target)
    const dynamic_exe = b.addExecutable(.{
        .name = "dynamic_app",
        .root_source_file = b.path("src/main.zig"),
        .target = b.resolveTargetQuery(.{
            .cpu_arch = .x86_64,
            .os_tag = .linux,
            .abi = .gnu,   // gnu = dynamic linking กับ glibc
        }),
        .optimize = optimize,
    });
    dynamic_exe.linkLibC();
    
    b.installArtifact(static_exe);
    b.installArtifact(dynamic_exe);
}
```

### ตรวจสอบ Linking Mode

```bash
# ตรวจสอบว่า binary เป็น static หรือ dynamic
ldd ./static_app
# output: not a dynamic executable (static binary)

ldd ./dynamic_app
# output:
#   linux-vdso.so.1 => (0x...)
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6

# ตรวจสอบ size
ls -lh static_app dynamic_app
# static_app จะใหญ่กว่า dynamic_app
```

---

## Step 562: Creating Static Library (.a)

### สร้าง Static Library ใน Zig

Static library คือ archive ของ object files ที่ linker จะนำมา include ใน executable

```zig
// src/mathlib.zig - library code
const std = @import("std");

/// บวกสองจำนวน
pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

/// ลบสองจำนวน
pub fn subtract(a: i32, b: i32) i32 {
    return a - b;
}

/// คูณสองจำนวน
pub fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

/// หารสองจำนวน (return error ถ้าหารด้วยศูนย์)
pub fn divide(a: i32, b: i32) !i32 {
    if (b == 0) return error.DivisionByZero;
    return @divTrunc(a, b);
}

/// หาค่า factorial
pub fn factorial(n: u64) u64 {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

/// หาค่า GCD (Greatest Common Divisor)
pub fn gcd(a: u64, b: u64) u64 {
    var x = a;
    var y = b;
    while (y != 0) {
        const temp = y;
        y = x % y;
        x = temp;
    }
    return x;
}

/// Matrix 2x2 multiplication
pub fn matMul2x2(
    a: [2][2]f64,
    b: [2][2]f64,
) [2][2]f64 {
    return .{
        .{ a[0][0] * b[0][0] + a[0][1] * b[1][0],
           a[0][0] * b[0][1] + a[0][1] * b[1][1] },
        .{ a[1][0] * b[0][0] + a[1][1] * b[1][0],
           a[1][0] * b[0][1] + a[1][1] * b[1][1] },
    };
}
```

### Build Script สำหรับ Static Library

```zig
// build.zig - สร้าง static library
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // สร้าง static library
    const lib = b.addStaticLibrary(.{
        .name = "mathlib",
        .root_source_file = b.path("src/mathlib.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Install library
    b.installArtifact(lib);
    
    // สร้าง executable ที่ใช้ library
    const exe = b.addExecutable(.{
        .name = "calculator",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Link กับ static library
    exe.linkLibrary(lib);
    
    b.installArtifact(exe);
    
    // Unit tests
    const lib_tests = b.addTest(.{
        .root_source_file = b.path("src/mathlib.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    const run_lib_tests = b.addRunArtifact(lib_tests);
    
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_lib_tests.step);
}
```

### ใช้ Static Library ใน Main Program

```zig
// src/main.zig - ใช้ static library
const std = @import("std");
const math = @import("mathlib");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Math Library Demo\n", .{});
    try stdout.print("─────────────────\n", .{});
    
    // Basic operations
    try stdout.print("5 + 3 = {}\n", .{math.add(5, 3)});
    try stdout.print("10 - 4 = {}\n", .{math.subtract(10, 4)});
    try stdout.print("6 * 7 = {}\n", .{math.multiply(6, 7)});
    
    // Division with error handling
    const result = try math.divide(20, 4);
    try stdout.print("20 / 4 = {}\n", .{result});
    
    // Division by zero
    if (math.divide(10, 0)) |_| {
        try stdout.print("Should not reach here\n", .{});
    } else |err| {
        try stdout.print("Error: {}\n", .{err});
    }
    
    // Factorial
    try stdout.print("10! = {}\n", .{math.factorial(10)});
    
    // GCD
    try stdout.print("GCD(48, 18) = {}\n", .{math.gcd(48, 18)});
}
```

### สร้าง Static Library ที่มี C ABI

```zig
// src/clib.zig - C-compatible library
const std = @import("std");

// Export functions สำหรับ C users
export fn math_add(a: c_int, b: c_int) c_int {
    return a + b;
}

export fn math_multiply(a: c_int, b: c_int) c_int {
    return a * b;
}

export fn math_sqrt(x: f64) f64 {
    return std.math.sqrt(x);
}

// C header file ที่ต้องสร้างด้วย
// mathlib.h:
// int math_add(int a, int b);
// int math_multiply(int a, int b);
// double math_sqrt(double x);
```

---

## Step 563: Creating Shared Library (.so, .dll)

### สร้าง Shared Library

```zig
// src/shared_lib.zig - shared library
const std = @import("std");

// Version information
pub const VERSION_MAJOR = 2;
pub const VERSION_MINOR = 0;
pub const VERSION_PATCH = 1;

// Public API
export fn get_version() [*:0]const u8 {
    return "2.0.1";
}

export fn create_buffer(size: usize) ?[*]u8 {
    const allocator = std.heap.c_allocator;
    const buf = allocator.alloc(u8, size) catch return null;
    return buf.ptr;
}

export fn free_buffer(ptr: [*]u8, size: usize) void {
    const allocator = std.heap.c_allocator;
    const slice = ptr[0..size];
    allocator.free(slice);
}

export fn process_data(
    input: [*]const u8,
    input_len: usize,
    output: [*]u8,
    output_len: usize,
) usize {
    const in_slice = input[0..input_len];
    const out_slice = output[0..output_len];
    
    // ตัวอย่าง: แปลงเป็น uppercase
    const copy_len = @min(input_len, output_len);
    for (in_slice[0..copy_len], out_slice[0..copy_len]) |in_byte, *out_byte| {
        out_byte.* = std.ascii.toUpper(in_byte);
    }
    
    return copy_len;
}
```

### Build Script สำหรับ Shared Library

```zig
// build.zig - สร้าง shared library
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // สร้าง shared library
    const lib = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/shared_lib.zig"),
        .target = target,
        .optimize = optimize,
        .version = .{
            .major = 2,
            .minor = 0,
            .patch = 1,
        },
    });
    
    lib.linkLibC();
    
    b.installArtifact(lib);
    
    // ติดตั้ง header files
    b.installFile("include/mylib.h", "include/mylib.h");
}
```

### สร้าง Shared Library สำหรับ Windows (.dll)

```zig
// src/windows_dll.zig
const std = @import("std");
const builtin = @import("builtin");

// DllMain สำหรับ Windows
pub export fn DllMain(
    hinstDLL: std.os.windows.HINSTANCE,
    fdwReason: std.os.windows.DWORD,
    lpvReserved: std.os.windows.LPVOID,
) std.os.windows.BOOL {
    _ = hinstDLL;
    _ = lpvReserved;
    
    switch (fdwReason) {
        1 => {}, // DLL_PROCESS_ATTACH
        0 => {}, // DLL_PROCESS_DETACH
        2 => {}, // DLL_THREAD_ATTACH
        3 => {}, // DLL_THREAD_DETACH
        else => {},
    }
    return 1; // TRUE
}

export fn hello_from_dll() void {
    std.debug.print("Hello from DLL!\n", .{});
}

export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

### ตัวอย่าง C Header สำหรับ Shared Library

```c
// include/mylib.h
#pragma once

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

// Version
const char* get_version(void);

// Buffer management
uint8_t* create_buffer(size_t size);
void free_buffer(uint8_t* ptr, size_t size);

// Data processing
size_t process_data(
    const uint8_t* input,
    size_t input_len,
    uint8_t* output,
    size_t output_len
);

#ifdef __cplusplus
}
#endif
```

---

## Step 564: Linking External Libraries

### Link กับ System Libraries

```zig
// build.zig - link กับ system libraries
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "linked_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Link กับ C standard library
    exe.linkLibC();
    
    // Link กับ C++ standard library
    exe.linkLibCpp();
    
    // Link กับ system library (เช่น OpenSSL)
    exe.linkSystemLibrary("ssl");
    exe.linkSystemLibrary("crypto");
    
    // Link กับ math library
    exe.linkSystemLibrary("m");
    
    // Link กับ pthread
    exe.linkSystemLibrary("pthread");
    
    // เพิ่ม include path
    exe.addIncludePath(b.path("third_party/include"));
    
    // เพิ่ม library path
    exe.addLibraryPath(b.path("third_party/lib"));
    
    b.installArtifact(exe);
}
```

### Link กับ Local Library Files

```zig
// build.zig - link กับ pre-built libraries
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Link กับ .a file โดยตรง
    exe.addObjectFile(b.path("libs/libfoo.a"));
    
    // หรือระบุ path และชื่อ
    exe.addLibraryPath(b.path("libs"));
    exe.linkSystemLibrary("foo");
    
    // เพิ่ม rpath สำหรับ shared libraries
    exe.addRPath(b.path("libs"));
    
    b.installArtifact(exe);
}
```

### ตัวอย่าง FFI กับ OpenSSL

```zig
// src/ssl_demo.zig - ใช้ OpenSSL
const std = @import("std");
const c = @cImport({
    @cInclude("openssl/sha.h");
    @cInclude("openssl/evp.h");
});

pub fn sha256(data: []const u8) [32]u8 {
    var hash: [32]u8 = undefined;
    var sha256_ctx: c.SHA256_CTX = undefined;
    
    _ = c.SHA256_Init(&sha256_ctx);
    _ = c.SHA256_Update(&sha256_ctx, data.ptr, data.len);
    _ = c.SHA256_Final(&hash, &sha256_ctx);
    
    return hash;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const data = "Hello, World!";
    const hash = sha256(data);
    
    try stdout.print("SHA256 of \"{s}\":\n", .{data});
    for (hash) |byte| {
        try stdout.print("{x:0>2}", .{byte});
    }
    try stdout.print("\n", .{});
}
```

### pkgconfig Integration

```zig
// build.zig - ใช้ pkg-config
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "gtk_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // ใช้ pkg-config สำหรับ GTK4
    exe.linkSystemLibrary2("gtk4", .{ .use_pkg_config = .force });
    exe.linkLibC();
    
    b.installArtifact(exe);
}
```

---

## Step 565: Symbol Visibility

### Control Symbol Visibility

Symbol visibility กำหนดว่า function/variable ใดจะมองเห็นได้จากภายนอก library

```zig
// src/visibility_demo.zig
const std = @import("std");

// Public symbol (exported)
pub export fn public_function() i32 {
    return internal_helper(42);
}

// Internal symbol (ไม่ export)
fn internal_helper(x: i32) i32 {
    return x * 2;
}

// Explicitly hidden (สำหรับ shared library)
pub fn hiddenFunction() void {
    std.debug.print("This is internal\n", .{});
}
```

### Symbol Visibility ใน C interop

```c
// ใน C code ที่ compile ด้วย Zig
// GCC/Clang visibility attributes

__attribute__((visibility("default")))
int public_api(int x);  // มองเห็นได้จากภายนอก

__attribute__((visibility("hidden")))
int private_helper(int x);  // ไม่มองเห็นจากภายนอก

__attribute__((visibility("protected")))
int semi_public(int x);  // override ได้จาก preload แต่ไม่ interpose ได้
```

### Export List

```bash
# ดู exported symbols
nm -D libmylib.so | grep " T "

# ดู symbols ทั้งหมด
objdump -T libmylib.so

# ดู undefined symbols (dependencies)
nm -u libmylib.so
```

### Version Script สำหรับ Symbol Visibility

```
# mylib.map - version script
MYLIB_2.0 {
    global:
        public_function;
        create_buffer;
        free_buffer;
        process_data;
        get_version;
    local:
        *;
};
```

```zig
// build.zig - ใช้ version script
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const lib = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // เพิ่ม version script
    lib.linkerScript(b.path("mylib.map"));
    
    b.installArtifact(lib);
}
```

---

## Step 566: Link-Time Optimization (LTO)

### LTO ช่วยอะไร?

LTO (Link-Time Optimization) ช่วยให้ optimizer มองเห็น code ทั้งหมดพร้อมกันในขั้นตอน linking แทนที่จะ optimize แต่ละ compilation unit แยกกัน

```
ประโยชน์ของ LTO:
├─ Inline functions ข้าม translation units
├─ Dead code elimination ที่แม่นยำกว่า
├─ Better constant propagation
├─ Improved devirtualization
└─ Binary size ลดลง 10-20%
```

### เปิดใช้ LTO ใน Zig

```zig
// build.zig - เปิดใช้ LTO
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    
    // LTO ใช้กับ Release modes
    const exe = b.addExecutable(.{
        .name = "optimized_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseFast,
    });
    
    // Zig เปิดใช้ LTO อัตโนมัติกับ ReleaseFast และ ReleaseSmall
    // สำหรับ Debug mode ไม่ใช้ LTO เพื่อ build เร็ว
    
    b.installArtifact(exe);
}
```

### ตัวอย่าง LTO Effect

```zig
// math_utils.zig
pub fn square(x: f64) f64 {
    return x * x;
}

pub fn cube(x: f64) f64 {
    return x * x * x;
}

pub fn distance(x1: f64, y1: f64, x2: f64, y2: f64) f64 {
    const dx = x2 - x1;
    const dy = y2 - y1;
    return @sqrt(square(dx) + square(dy));
}
```

```zig
// main.zig - LTO จะ inline square() และ optimize
const std = @import("std");
const math = @import("math_utils.zig");

pub fn main() void {
    // ด้วย LTO, compiler รู้ว่า square() และ cube() เป็น pure functions
    // และสามารถ compute ที่ compile time ได้
    const d = math.distance(0, 0, 3, 4);
    std.debug.print("Distance: {d}\n", .{d}); // 5.0
}
```

### Thin LTO

```bash
# Full LTO (ช้ากว่าแต่ optimize ดีกว่า)
zig build -Doptimize=ReleaseFast

# Thin LTO (เร็วกว่า, เหมาะกับ CI)
# Zig ใช้ Thin LTO โดยอัตโนมัติสำหรับบาง targets
```

---

## Step 567: Dead Code Elimination

### Dead Code Elimination คืออะไร?

Dead code elimination ลบ code ที่ไม่ถูกใช้งานออกจาก binary สุดท้าย ทำให้ binary มีขนาดเล็กลง

```zig
// library_with_dead_code.zig
pub fn usedFunction() i32 {
    return 42;  // จะถูก include
}

pub fn unusedFunction() i32 {
    return 0;   // อาจถูกลบถ้าไม่มีใครเรียก
}

pub fn anotherUnused() void {
    // Code path ที่ไม่มีทางถึง
}
```

### ตรวจสอบ Dead Code Elimination

```bash
# Build แบบ release เพื่อดู DCE
zig build-exe main.zig -O ReleaseSmall

# ดู symbols ที่เหลืออยู่
nm --defined-only main | grep -v "^0000000000000000"

# ดูขนาดของ sections
size main

# เปรียบเทียบ Debug vs Release
zig build-exe main.zig -O Debug -o main_debug
zig build-exe main.zig -O ReleaseSmall -o main_release
ls -lh main_debug main_release
```

### Force Include Symbols

```zig
// ถ้าต้องการ keep symbol ที่ไม่ถูกเรียกตรงๆ
// (เช่น สำหรับ plugins หรือ dynamic loading)
comptime {
    _ = unusedFunction;  // ทำให้ compiler คิดว่าใช้งาน
}

// หรือใช้ export
pub export fn unusedButExported() i32 {
    return 0;
}
```

### Build Options สำหรับ Size Optimization

```zig
// build.zig - optimize สำหรับ size
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    
    const exe = b.addExecutable(.{
        .name = "tiny_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseSmall,  // เน้น size ไม่ใช่ speed
    });
    
    // Strip debug information
    exe.strip = true;
    
    b.installArtifact(exe);
}
```

---

## Step 568: ABI Compatibility

### ABI คืออะไร?

ABI (Application Binary Interface) กำหนดวิธีที่ functions ถูกเรียกในระดับ binary:
- Calling conventions (register/stack usage)
- Data structure layout
- Name mangling
- Exception handling

### Calling Conventions ใน Zig

```zig
// calling_conventions.zig
const std = @import("std");

// C calling convention (default สำหรับ C interop)
fn cFunction(a: i32, b: i32) callconv(.C) i32 {
    return a + b;
}

// Windows specific (stdcall)
fn windowsFunction(x: i32) callconv(.winapi) i32 {
    return x * 2;
}

// Naked function (ไม่มี prologue/epilogue)
fn nakedFunction() callconv(.Naked) void {
    asm volatile ("ret");
}

// Inline function
inline fn fastAdd(a: i32, b: i32) i32 {
    return a + b;
}

// Async calling convention
fn asyncFunction() callconv(.Async) void {
    // suspend/resume ได้
}
```

### Data Layout Compatibility

```zig
// data_layout.zig - ควบคุม data layout สำหรับ ABI compat
const std = @import("std");

// Packed struct - ไม่มี padding
const PackedStruct = packed struct {
    a: u8,
    b: u16,
    c: u32,
};

// Extern struct - ใช้ C layout
const CCompatStruct = extern struct {
    x: c_int,
    y: c_double,
    z: [4]c_char,
};

// Zig default struct (อาจมี padding เพื่อ alignment)
const ZigStruct = struct {
    a: u8,
    b: u32,  // Zig อาจเพิ่ม padding ก่อน b
    c: u16,
};

pub fn main() void {
    std.debug.print("PackedStruct size: {}\n", .{@sizeOf(PackedStruct)});
    std.debug.print("CCompatStruct size: {}\n", .{@sizeOf(CCompatStruct)});
    std.debug.print("ZigStruct size: {}\n", .{@sizeOf(ZigStruct)});
    
    // ตรวจสอบ offsets
    std.debug.print("CCompatStruct.x offset: {}\n", .{@offsetOf(CCompatStruct, "x")});
    std.debug.print("CCompatStruct.y offset: {}\n", .{@offsetOf(CCompatStruct, "y")});
}
```

### ABI-Stable Interface Design

```zig
// stable_api.zig - design API ที่ ABI stable
const std = @import("std");

// ใช้ opaque types เพื่อ hide implementation
pub const Handle = opaque {};

// Versioned API
pub const API_VERSION: u32 = 1;

// C-compatible error codes
pub const Error = enum(c_int) {
    ok = 0,
    invalid_arg = -1,
    out_of_memory = -2,
    not_found = -3,
    _,  // unknown errors
};

// C-compatible function signatures
export fn create_handle(size: usize) ?*Handle {
    const allocator = std.heap.c_allocator;
    const ptr = allocator.create(InternalState) catch return null;
    ptr.* = InternalState{ .size = size };
    return @ptrCast(ptr);
}

export fn destroy_handle(handle: *Handle) void {
    const state: *InternalState = @alignCast(@ptrCast(handle));
    std.heap.c_allocator.destroy(state);
}

export fn get_size(handle: *const Handle) usize {
    const state: *const InternalState = @alignCast(@ptrCast(handle));
    return state.size;
}

// Private implementation (ซ่อนจาก API)
const InternalState = struct {
    size: usize,
    data: ?[]u8 = null,
};
```

---

## Step 569: Symbol Versioning

### Library Versioning บน Linux

```bash
# Version format: libname.so.MAJOR.MINOR.PATCH
# Soname: libname.so.MAJOR

# ดู soname ของ library
readelf -d libmylib.so | grep SONAME
# 0x000000000000000e (SONAME) Library soname: [libmylib.so.2]

# ดู version dependencies
readelf -d executable | grep NEEDED
```

### Zig Library Versioning

```zig
// build.zig - library versioning
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const lib = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
        .version = .{
            .major = 2,   // soname จะเป็น libmylib.so.2
            .minor = 3,
            .patch = 1,
        },
    });
    
    b.installArtifact(lib);
}
```

### Symbol Versioning ใน Version Script

```
# lib_versions.map - ควบคุม symbol versions
MYLIB_1.0 {
    global:
        old_function_v1;
    local: *;
};

MYLIB_2.0 {
    global:
        new_function_v2;
        create_handle;
        destroy_handle;
} MYLIB_1.0;
```

### Backward Compatibility

```zig
// compat.zig - maintain backward compatibility
const std = @import("std");

// Version 1 function (deprecated แต่ยังต้อง support)
export fn old_function_v1(x: c_int) c_int {
    return new_function_v2(x, 0);
}

// Version 2 function (ใหม่กว่า)
export fn new_function_v2(x: c_int, flags: c_int) c_int {
    _ = flags;
    return x * 2;
}

// Aliases สำหรับ forward compatibility
comptime {
    @export(old_function_v1, .{ .name = "function@@MYLIB_1.0" });
    @export(new_function_v2, .{ .name = "function@@MYLIB_2.0" });
    @export(new_function_v2, .{ .name = "function@MYLIB_2.0" });
}
```

---

## Step 570: Library Packaging

### สร้าง Complete Library Package

```
mylib/
├── include/
│   ├── mylib.h           # C header
│   └── mylib.hpp         # C++ wrapper
├── lib/
│   ├── libmylib.a        # Static library
│   ├── libmylib.so       # Symlink to libmylib.so.2
│   ├── libmylib.so.2     # Symlink to libmylib.so.2.3.1
│   └── libmylib.so.2.3.1 # Actual shared library
├── cmake/
│   └── mylibConfig.cmake # CMake package config
├── pkgconfig/
│   └── mylib.pc          # pkg-config file
├── zig/
│   └── mylib.zig         # Zig module
└── README.md
```

### Build Script สำหรับ Complete Package

```zig
// build.zig - สร้าง complete package
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const version = std.SemanticVersion{ .major = 2, .minor = 3, .patch = 1 };
    
    // Static library
    const static_lib = b.addStaticLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    static_lib.linkLibC();
    
    // Shared library
    const shared_lib = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
        .version = version,
    });
    shared_lib.linkLibC();
    
    // Install libraries
    b.installArtifact(static_lib);
    b.installArtifact(shared_lib);
    
    // Install headers
    b.installFile("include/mylib.h", "include/mylib.h");
    b.installFile("include/mylib.hpp", "include/mylib.hpp");
    
    // Generate pkg-config file
    const pc_content = b.fmt(
        \\prefix={s}
        \\exec_prefix=${{prefix}}
        \\libdir=${{exec_prefix}}/lib
        \\includedir=${{prefix}}/include
        \\
        \\Name: mylib
        \\Description: My Library
        \\Version: {}.{}.{}
        \\Libs: -L${{libdir}} -lmylib
        \\Cflags: -I${{includedir}}
    , .{
        b.install_prefix,
        version.major,
        version.minor,
        version.patch,
    });
    
    const write_pc = b.addWriteFiles();
    const pc_file = write_pc.add("mylib.pc", pc_content);
    
    b.installFile(pc_file.path, "lib/pkgconfig/mylib.pc");
    
    // Tests
    const lib_tests = b.addTest(.{
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    const run_tests = b.addRunArtifact(lib_tests);
    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&run_tests.step);
}
```

### CMake Package Config

```cmake
# cmake/mylibConfig.cmake
cmake_minimum_required(VERSION 3.15)

find_library(MYLIB_LIBRARY
    NAMES mylib
    PATHS "${CMAKE_CURRENT_LIST_DIR}/../lib"
)

find_path(MYLIB_INCLUDE_DIR
    NAMES mylib.h
    PATHS "${CMAKE_CURRENT_LIST_DIR}/../include"
)

include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(mylib
    REQUIRED_VARS MYLIB_LIBRARY MYLIB_INCLUDE_DIR
)

if(mylib_FOUND AND NOT TARGET mylib::mylib)
    add_library(mylib::mylib UNKNOWN IMPORTED)
    set_target_properties(mylib::mylib PROPERTIES
        IMPORTED_LOCATION "${MYLIB_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${MYLIB_INCLUDE_DIR}"
    )
endif()
```

### Zig Package สำหรับ Zig Users

```zig
// zig/mylib.zig - Zig module wrapper
const std = @import("std");

// Re-export public API
pub const add = @import("../src/mathlib.zig").add;
pub const subtract = @import("../src/mathlib.zig").subtract;
pub const multiply = @import("../src/mathlib.zig").multiply;
pub const divide = @import("../src/mathlib.zig").divide;

// Zig-idiomatic wrapper
pub const Math = struct {
    pub fn compute(op: Operation, a: i32, b: i32) !i32 {
        return switch (op) {
            .add => add(a, b),
            .subtract => subtract(a, b),
            .multiply => multiply(a, b),
            .divide => try divide(a, b),
        };
    }
};

pub const Operation = enum { add, subtract, multiply, divide };
```

### Packaging Script

```bash
#!/bin/bash
# package.sh - สร้าง release package

VERSION="2.3.1"
PACKAGE_NAME="mylib-${VERSION}"

echo "Building mylib ${VERSION}..."

# สร้าง release builds สำหรับทุก platforms
for TARGET in \
    x86_64-linux-musl \
    aarch64-linux-musl \
    x86_64-windows-gnu \
    aarch64-macos; do
    
    echo "Building for ${TARGET}..."
    zig build \
        -Dtarget="${TARGET}" \
        -Doptimize=ReleaseFast \
        --prefix "dist/${PACKAGE_NAME}-${TARGET}"
    
    # สร้าง archive
    if [[ "${TARGET}" == *"windows"* ]]; then
        (cd dist && zip -r "${PACKAGE_NAME}-${TARGET}.zip" "${PACKAGE_NAME}-${TARGET}/")
    else
        (cd dist && tar czf "${PACKAGE_NAME}-${TARGET}.tar.gz" "${PACKAGE_NAME}-${TARGET}/")
    fi
done

# สร้าง checksums
cd dist
sha256sum *.tar.gz *.zip > checksums.sha256
echo "Package created successfully!"
echo "Files:"
ls -lh *.tar.gz *.zip
```

### build.zig.zon สำหรับ Zig Package Manager

```zig
// build.zig.zon - package manifest
.{
    .name = "mylib",
    .version = "2.3.1",
    .minimum_zig_version = "0.13.0",
    
    .dependencies = .{
        // ไม่มี external dependencies
    },
    
    .paths = .{
        "src/",
        "include/",
        "build.zig",
        "build.zig.zon",
        "LICENSE",
        "README.md",
    },
}
```

### สรุป Linking และ Libraries

```
การใช้ Libraries ใน Zig:
┌─────────────────────────────────────────────────────┐
│ Static Library (.a)                                 │
│ ├─ b.addStaticLibrary(...)                         │
│ ├─ exe.linkLibrary(lib)                            │
│ └─ ผล: executable ขนาดใหญ่, self-contained         │
│                                                    │
│ Shared Library (.so/.dll/.dylib)                   │
│ ├─ b.addSharedLibrary(...)                         │
│ ├─ exe.linkSystemLibrary("name")                   │
│ └─ ผล: executable เล็ก, ต้องมี .so ที่ runtime    │
│                                                    │
│ Best Practices                                     │
│ ├─ ใช้ musl สำหรับ portable static binaries       │
│ ├─ ออกแบบ stable C API สำหรับ shared libraries    │
│ ├─ ใช้ version scripts เพื่อควบคุม symbol export  │
│ ├─ เปิด LTO ใน release builds                    │
│ └─ สร้าง complete package พร้อม headers          │
└─────────────────────────────────────────────────────┘
```

---

**บทสรุป Part 57:**

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Linking และ Libraries ใน Zig:

1. **Step 561**: ความแตกต่างระหว่าง static และ dynamic linking
2. **Step 562**: การสร้าง static library (.a) พร้อม C interop
3. **Step 563**: การสร้าง shared library (.so, .dll) พร้อม DLL support
4. **Step 564**: การ link กับ external libraries รวมถึง OpenSSL
5. **Step 565**: Symbol visibility และ version scripts
6. **Step 566**: Link-Time Optimization (LTO)
7. **Step 567**: Dead code elimination และ size optimization
8. **Step 568**: ABI compatibility, calling conventions, data layout
9. **Step 569**: Symbol versioning สำหรับ backward compatibility
10. **Step 570**: Complete library packaging สำหรับ distribution

การเข้าใจ linking อย่างถ่องแท้จะช่วยให้สามารถสร้าง libraries ที่ใช้งานได้อย่างยืดหยุ่นและ distribute ได้อย่างถูกต้อง
