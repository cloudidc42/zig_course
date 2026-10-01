# Part 44: C Interoperability ขั้นสูง

> **หลักสูตร Zig ภาษาไทย** | Steps 431-440

---

## Step 431: C libraries - static vs dynamic linking

การเลือกวิธี link libraries มีผลต่อ portability และขนาดของ binary

### Static Linking

**Static linking** รวม library code เข้าไปใน binary โดยตรง

```
โปรแกรม.exe
├── code ของเรา
├── code จาก libssl.a   ← รวมเข้ามาแล้ว
├── code จาก libcrypto.a ← รวมเข้ามาแล้ว
└── code จาก libc.a     ← รวมเข้ามาแล้ว
```

**ข้อดี:**
- ไม่ต้องติดตั้ง libraries เพิ่มเติม
- รันได้ทุกเครื่อง
- ไม่มีปัญหา version conflict

**ข้อเสีย:**
- ไฟล์ binary ขนาดใหญ่
- อัพเดต library ต้อง recompile ทั้งหมด

```zig
// build.zig - static linking
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "static-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Static link กับ library
    exe.linkSystemLibrary2("ssl", .{ .preferred_link_mode = .static });
    exe.linkSystemLibrary2("crypto", .{ .preferred_link_mode = .static });
    exe.linkLibC();

    b.installArtifact(exe);
}
```

### Dynamic Linking

**Dynamic linking** โหลด library ตอน runtime

```
โปรแกรม.exe
└── references to: libssl.so, libcrypto.so, libc.so

Runtime:
system/
├── libssl.so.3
├── libcrypto.so.3
└── libc.so.6
```

**ข้อดี:**
- ไฟล์ binary เล็ก
- หลาย programs ใช้ library ร่วมกัน
- อัพเดต security patches ง่าย

**ข้อเสีย:**
- ต้องมี library ในระบบ
- อาจเกิด "DLL hell"
- ช้ากว่าเล็กน้อยตอนโหลด

```zig
// build.zig - dynamic linking (ค่าเริ่มต้น)
exe.linkSystemLibrary("ssl");    // จะ prefer dynamic
exe.linkSystemLibrary("crypto");
```

### Musl Libc - Fully Static Binary

```bash
# Build เป็น static binary ด้วย musl libc
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseSafe

# ตรวจสอบ
ldd zig-out/bin/my-app
# output: not a dynamic executable   ← fully static!
```

```zig
// build.zig สำหรับ fully static
const std = @import("std");

pub fn build(b: *std.Build) void {
    // ใช้ musl target สำหรับ static binary
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .x86_64,
        .os_tag = .linux,
        .abi = .musl,
    });

    const exe = b.addExecutable(.{
        .name = "static-binary",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseSafe,
    });

    b.installArtifact(exe);
}
```

---

## Step 432: pkg-config integration

`pkg-config` เป็นเครื่องมือมาตรฐาน Unix สำหรับหา compiler/linker flags ของ libraries

### ทำงานอย่างไร?

```bash
# ดู flags ของ library
pkg-config --cflags openssl
# -I/usr/include/openssl

pkg-config --libs openssl
# -lssl -lcrypto

pkg-config --modversion openssl
# 3.0.7
```

### ใช้ pkg-config ใน build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ใช้ pkg-config สำหรับ library
    exe.linkSystemLibrary("openssl");

    // หรือระบุ pkg-config name โดยตรง
    // pkg-config ใช้ชื่อ "openssl" หรือ "libssl"

    exe.linkLibC();
    b.installArtifact(exe);
}
```

### Custom pkg-config

```zig
const std = @import("std");

// ฟังก์ชัน helper สำหรับ pkg-config
fn pkgConfig(b: *std.Build, pkg: []const u8) !struct {
    cflags: []const []const u8,
    libs: []const []const u8,
} {
    // รัน pkg-config --cflags
    const cflags_result = try b.exec(&[_][]const u8{
        "pkg-config", "--cflags", pkg,
    });

    // รัน pkg-config --libs
    const libs_result = try b.exec(&[_][]const u8{
        "pkg-config", "--libs", pkg,
    });

    _ = cflags_result;
    _ = libs_result;

    // parse และ return flags
    return .{
        .cflags = &[_][]const u8{},
        .libs = &[_][]const u8{},
    };
}

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Manual pkg-config flags
    exe.addIncludePath(.{ .cwd_relative = "/usr/include/openssl" });
    exe.linkSystemLibrary("ssl");
    exe.linkSystemLibrary("crypto");
    exe.linkLibC();

    b.installArtifact(exe);
}
```

---

## Step 433: Using OpenSSL, zlib, SQLite

ตัวอย่างการใช้ C libraries ยอดนิยม

### zlib - Compression

```zig
// build.zig
const exe = b.addExecutable(.{ .name = "compress", ... });
exe.linkSystemLibrary("z");  // zlib
exe.linkLibC();
```

```zig
// src/compress.zig
const std = @import("std");
const c = @cImport({
    @cInclude("zlib.h");
});

pub fn compress(data: []const u8, allocator: std.mem.Allocator) ![]u8 {
    // คำนวณขนาด output สูงสุด
    var dest_len = c.compressBound(data.len);

    // Allocate output buffer
    const dest = try allocator.alloc(u8, dest_len);
    errdefer allocator.free(dest);

    // Compress
    const result = c.compress(
        dest.ptr,
        &dest_len,
        data.ptr,
        data.len,
    );

    if (result != c.Z_OK) {
        return error.CompressionFailed;
    }

    // Resize to actual size
    return allocator.realloc(dest, dest_len);
}

pub fn decompress(compressed: []const u8, allocator: std.mem.Allocator, max_size: usize) ![]u8 {
    var dest_len: c_ulong = @intCast(max_size);
    const dest = try allocator.alloc(u8, max_size);
    errdefer allocator.free(dest);

    const result = c.uncompress(
        dest.ptr,
        &dest_len,
        compressed.ptr,
        compressed.len,
    );

    if (result != c.Z_OK) {
        return error.DecompressionFailed;
    }

    return allocator.realloc(dest, dest_len);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const original = "Hello, World! This is a test of zlib compression. " **
        "Repeated text repeated text repeated text repeated text.";

    std.debug.print("Original size: {} bytes\n", .{original.len});

    const compressed = try compress(original, allocator);
    defer allocator.free(compressed);

    std.debug.print("Compressed size: {} bytes ({d:.1}%)\n", .{
        compressed.len,
        @as(f64, @floatFromInt(compressed.len)) /
        @as(f64, @floatFromInt(original.len)) * 100.0,
    });

    const decompressed = try decompress(compressed, allocator, original.len * 2);
    defer allocator.free(decompressed);

    std.debug.print("Decompressed: {s}\n", .{decompressed});
}
```

### OpenSSL - Cryptography

```zig
// src/crypto.zig
const std = @import("std");
const c = @cImport({
    @cInclude("openssl/sha.h");
    @cInclude("openssl/md5.h");
    @cInclude("openssl/evp.h");
    @cInclude("openssl/rand.h");
});

// SHA-256 Hash
pub fn sha256(data: []const u8) [32]u8 {
    var hash: [32]u8 = undefined;
    _ = c.SHA256(data.ptr, data.len, &hash);
    return hash;
}

// MD5 Hash (ไม่แนะนำสำหรับ security)
pub fn md5(data: []const u8) [16]u8 {
    var hash: [16]u8 = undefined;
    _ = c.MD5(data.ptr, data.len, &hash);
    return hash;
}

// Random bytes
pub fn randomBytes(buf: []u8) !void {
    const result = c.RAND_bytes(buf.ptr, @intCast(buf.len));
    if (result != 1) return error.RandomGenerationFailed;
}

pub fn main() !void {
    const message = "Hello, Cryptography!";

    // SHA-256
    const sha256_hash = sha256(message);
    std.debug.print("SHA-256: ", .{});
    for (sha256_hash) |byte| {
        std.debug.print("{x:0>2}", .{byte});
    }
    std.debug.print("\n", .{});

    // Random bytes
    var rand_buf: [16]u8 = undefined;
    try randomBytes(&rand_buf);
    std.debug.print("Random: ", .{});
    for (rand_buf) |byte| {
        std.debug.print("{x:0>2}", .{byte});
    }
    std.debug.print("\n", .{});
}
```

---

## Step 434: C callbacks และ function pointers

การใช้ function pointers และ callbacks ระหว่าง C และ Zig

### Function Pointer Types

```zig
const std = @import("std");

// กำหนด function pointer type
const Callback = *const fn (value: c_int) callconv(.C) void;
const Comparator = *const fn (a: *const anyopaque, b: *const anyopaque) callconv(.C) c_int;
const ErrorHandler = *const fn (code: c_int, message: [*:0]const u8) callconv(.C) void;

// ฟังก์ชันที่รับ callback
fn processItems(items: []const i32, callback: Callback) void {
    for (items) |item| {
        callback(@intCast(item));
    }
}

// Callback implementation
fn printCallback(value: c_int) callconv(.C) void {
    std.debug.print("Value: {}\n", .{value});
}

fn squareCallback(value: c_int) callconv(.C) void {
    std.debug.print("Square: {}\n", .{value * value});
}

pub fn main() void {
    const numbers = [_]i32{ 1, 2, 3, 4, 5 };

    std.debug.print("Print callback:\n", .{});
    processItems(&numbers, printCallback);

    std.debug.print("Square callback:\n", .{});
    processItems(&numbers, squareCallback);
}
```

### qsort กับ Custom Comparator

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
});

// Comparator สำหรับ integer
fn compareInts(a: *const anyopaque, b: *const anyopaque) callconv(.C) c_int {
    const ia: *const c_int = @alignCast(@ptrCast(a));
    const ib: *const c_int = @alignCast(@ptrCast(b));
    if (ia.* < ib.*) return -1;
    if (ia.* > ib.*) return 1;
    return 0;
}

// Comparator สำหรับ string
fn compareStrings(a: *const anyopaque, b: *const anyopaque) callconv(.C) c_int {
    const sa: *const [*:0]const u8 = @alignCast(@ptrCast(a));
    const sb: *const [*:0]const u8 = @alignCast(@ptrCast(b));
    return c.strcmp(sa.*, sb.*);
}

pub fn main() void {
    // Sort integers
    var numbers = [_]c_int{ 5, 2, 8, 1, 9, 3, 7, 4, 6 };

    c.qsort(
        &numbers,
        numbers.len,
        @sizeOf(c_int),
        compareInts,
    );

    std.debug.print("Sorted: ", .{});
    for (numbers) |n| std.debug.print("{} ", .{n});
    std.debug.print("\n", .{});

    // Binary search หลัง sort
    const target: c_int = 7;
    const found = c.bsearch(
        &target,
        &numbers,
        numbers.len,
        @sizeOf(c_int),
        compareInts,
    );

    if (found != null) {
        const result: *c_int = @alignCast(@ptrCast(found));
        std.debug.print("Found: {}\n", .{result.*});
    }
}
```

### Event System ด้วย Callbacks

```zig
const std = @import("std");

// Event system
pub const EventType = enum(c_int) {
    click = 0,
    keypress = 1,
    resize = 2,
};

pub const Event = extern struct {
    event_type: EventType,
    x: c_int,
    y: c_int,
    key: c_int,
};

pub const EventHandler = *const fn (event: *const Event, user_data: ?*anyopaque) callconv(.C) void;

pub const EventSystem = struct {
    handlers: [3]?EventHandler = .{null} ** 3,
    user_data: [3]?*anyopaque = .{null} ** 3,

    pub fn on(self: *EventSystem, event_type: EventType, handler: EventHandler, data: ?*anyopaque) void {
        const idx = @intFromEnum(event_type);
        self.handlers[idx] = handler;
        self.user_data[idx] = data;
    }

    pub fn emit(self: *EventSystem, event: *const Event) void {
        const idx = @intFromEnum(event.event_type);
        if (self.handlers[idx]) |handler| {
            handler(event, self.user_data[idx]);
        }
    }
};

// Handler implementations
fn onClickHandler(event: *const Event, user_data: ?*anyopaque) callconv(.C) void {
    _ = user_data;
    std.debug.print("Click at ({}, {})\n", .{ event.x, event.y });
}

fn onKeyHandler(event: *const Event, user_data: ?*anyopaque) callconv(.C) void {
    _ = user_data;
    std.debug.print("Key pressed: {c}\n", .{@as(u8, @intCast(event.key))});
}

pub fn main() void {
    var sys = EventSystem{};

    sys.on(.click, onClickHandler, null);
    sys.on(.keypress, onKeyHandler, null);

    const click = Event{ .event_type = .click, .x = 100, .y = 200, .key = 0 };
    const key = Event{ .event_type = .keypress, .x = 0, .y = 0, .key = 'A' };

    sys.emit(&click);
    sys.emit(&key);
}
```

---

## Step 435: C variadic functions

การจัดการ variadic functions (`printf` style)

### เรียก Variadic Functions จาก C

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdarg.h");
});

pub fn main() void {
    // เรียก printf ซึ่งเป็น variadic function
    _ = c.printf("Hello, %s! You are %d years old.\n", "World", @as(c_int, 25));
    _ = c.printf("Float: %.2f, Hex: %x\n", @as(f64, 3.14), @as(c_uint, 255));

    // sprintf
    var buf: [128:0]u8 = undefined;
    _ = c.sprintf(&buf, "(%d, %d)", @as(c_int, 10), @as(c_int, 20));
    _ = c.printf("Point: %s\n", &buf);
}
```

### สร้าง Variadic Function ใน Zig

```zig
const std = @import("std");

// Zig ไม่รองรับ variadic functions แบบ C โดยตรง
// แต่สามารถใช้ anytype หรือ tuples แทน

// วิธีที่ 1: ใช้ anytype
pub fn printAny(value: anytype) void {
    const T = @TypeOf(value);
    switch (@typeInfo(T)) {
        .int, .comptime_int => std.debug.print("{d}", .{value}),
        .float, .comptime_float => std.debug.print("{d:.2}", .{value}),
        .bool => std.debug.print("{}", .{value}),
        .pointer => |ptr| {
            if (ptr.child == u8) {
                std.debug.print("{s}", .{value});
            } else {
                std.debug.print("{any}", .{value});
            }
        },
        else => std.debug.print("{any}", .{value}),
    }
}

// วิธีที่ 2: ใช้ format string แบบ Zig
pub fn myPrint(comptime fmt: []const u8, args: anytype) void {
    std.debug.print(fmt, args);
}

// วิธีที่ 3: export variadic function ไปยัง C (ผ่าน VaList)
export fn zig_vprintf(fmt: [*:0]const u8, args: *std.c.va_list) c_int {
    _ = fmt;
    _ = args;
    // ยาก - ควรใช้ Zig's own formatting แทน
    return 0;
}

pub fn main() void {
    printAny(42);
    std.debug.print("\n", .{});
    printAny(3.14);
    std.debug.print("\n", .{});
    printAny("Hello");
    std.debug.print("\n", .{});

    myPrint("Name: {s}, Age: {d}\n", .{ "Alice", 30 });
}
```

### va_list ใน C Interop

```c
// src/logger.c - C library ที่ใช้ va_list
#include <stdio.h>
#include <stdarg.h>
#include <time.h>

void log_message(const char* level, const char* fmt, ...) {
    time_t now = time(NULL);
    char time_str[64];
    strftime(time_str, sizeof(time_str), "%H:%M:%S", localtime(&now));

    printf("[%s] [%s] ", time_str, level);

    va_list args;
    va_start(args, fmt);
    vprintf(fmt, args);
    va_end(args);

    printf("\n");
}
```

```zig
// src/main.zig
const c = @cImport({
    @cInclude("logger.h");
});

pub fn main() void {
    c.log_message("INFO", "Server started on port %d", @as(c_int, 8080));
    c.log_message("WARN", "Memory usage: %d%%", @as(c_int, 75));
    c.log_message("ERROR", "Connection failed: %s", "timeout");
}
```

---

## Step 436: Error handling จาก C (errno)

C ใช้ `errno` global variable สำหรับ error reporting

### errno Basics

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("errno.h");
    @cInclude("string.h");
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
    @cInclude("fcntl.h");
});

pub fn main() void {
    // เปิดไฟล์ที่ไม่มี
    const file = c.fopen("/nonexistent/file.txt", "r");

    if (file == null) {
        // อ่าน errno
        const err_num = c.errno;
        const err_msg = c.strerror(err_num);

        std.debug.print("Error {}: {s}\n", .{ err_num, err_msg });
        // Output: Error 2: No such file or directory
    }
}
```

### Zig Error Set จาก errno

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("errno.h");
    @cInclude("stdio.h");
    @cInclude("string.h");
});

pub const CError = error{
    NoSuchFile,
    PermissionDenied,
    AlreadyExists,
    NotADirectory,
    IsADirectory,
    TooManyOpenFiles,
    OutOfMemory,
    BrokenPipe,
    Unknown,
};

pub fn errnoToZigError(err: c_int) CError {
    return switch (err) {
        c.ENOENT => CError.NoSuchFile,
        c.EACCES, c.EPERM => CError.PermissionDenied,
        c.EEXIST => CError.AlreadyExists,
        c.ENOTDIR => CError.NotADirectory,
        c.EISDIR => CError.IsADirectory,
        c.EMFILE, c.ENFILE => CError.TooManyOpenFiles,
        c.ENOMEM => CError.OutOfMemory,
        c.EPIPE => CError.BrokenPipe,
        else => CError.Unknown,
    };
}

pub fn openFile(path: []const u8) CError!*c.FILE {
    // Reset errno ก่อน
    c.errno = 0;

    const file = c.fopen(path.ptr, "r");

    if (file == null) {
        return errnoToZigError(c.errno);
    }

    return file;
}

pub fn readLine(file: *c.FILE, buf: []u8) CError!?[]u8 {
    c.errno = 0;
    const result = c.fgets(buf.ptr, @intCast(buf.len), file);

    if (result == null) {
        if (c.feof(file) != 0) return null;  // EOF
        return errnoToZigError(c.errno);
    }

    return std.mem.sliceTo(buf, 0);
}

pub fn main() !void {
    const file = openFile("test.txt") catch |err| {
        std.debug.print("Error opening file: {}\n", .{err});
        return;
    };
    defer _ = c.fclose(file);

    var buf: [256]u8 = undefined;
    while (try readLine(file, &buf)) |line| {
        std.debug.print("Line: {s}\n", .{line});
    }
}
```

### POSIX Error Handling

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("errno.h");
    @cInclude("unistd.h");
    @cInclude("fcntl.h");
    @cInclude("sys/stat.h");
});

pub fn main() !void {
    // open() returns -1 on error
    const fd = c.open("test.txt", c.O_RDONLY);
    if (fd == -1) {
        const err = c.errno;
        std.debug.print("open failed: {s} (errno={})\n", .{
            c.strerror(err),
            err,
        });
        return error.OpenFailed;
    }
    defer _ = c.close(fd);

    // read()
    var buf: [1024]u8 = undefined;
    const n = c.read(fd, &buf, buf.len);
    if (n == -1) {
        std.debug.print("read failed: {s}\n", .{c.strerror(c.errno)});
        return error.ReadFailed;
    }

    std.debug.print("Read {} bytes\n", .{n});
    std.debug.print("Content: {s}\n", .{buf[0..@intCast(n)]});
}
```

---

## Step 437: Memory ownership boundaries

การจัดการ ownership ของ memory ระหว่าง Zig และ C

### กฎสำคัญ

```
Memory ที่ C จัดสรร → C ต้องคืน (ด้วย free())
Memory ที่ Zig จัดสรร → Zig ต้องคืน (ด้วย allocator.free())
```

### Pattern 1: C จัดสรร, C คืน

```zig
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

// C function ที่ return allocated memory
// caller ต้องเรียก free()
fn c_strdup(s: [*:0]const u8) [*:0]u8 {
    return c.strdup(s);
}

pub fn useCMemory() void {
    // C จัดสรร
    const dup = c_strdup("Hello, World!");
    defer c.free(dup);  // C ต้องคืน

    std.debug.print("Dup: {s}\n", .{dup});
}

const std = @import("std");
```

### Pattern 2: Zig จัดสรร, Zig คืน

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("string.h");
});

// Zig allocate memory แล้วส่งให้ C ใช้
pub fn zigAllocForC(allocator: std.mem.Allocator) !void {
    // Zig จัดสรร
    const buf = try allocator.alloc(u8, 256);
    defer allocator.free(buf);  // Zig ต้องคืน

    // ส่ง pointer ให้ C ใช้ (ห้าม C free!)
    _ = c.memset(buf.ptr, 0, buf.len);
    _ = c.strcpy(buf.ptr, "Hello from Zig memory");

    std.debug.print("Buffer: {s}\n", .{buf});
}
```

### Pattern 3: Wrapper ที่จัดการ Ownership ชัดเจน

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

// Wrapper ที่ชัดเจนเรื่อง ownership
pub const CString = struct {
    ptr: [*:0]u8,

    pub fn init(s: []const u8) !CString {
        const dup = c.strndup(s.ptr, s.len) orelse return error.OutOfMemory;
        return .{ .ptr = dup };
    }

    pub fn deinit(self: CString) void {
        c.free(self.ptr);
    }

    pub fn slice(self: CString) []const u8 {
        return std.mem.span(self.ptr);
    }
};

pub fn main() !void {
    const str = try CString.init("Hello, World!");
    defer str.deinit();  // ชัดเจนว่า C ต้อง free

    std.debug.print("String: {s}\n", .{str.slice()});
}
```

### Arena Allocator สำหรับ C Calls

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
});

pub fn main() !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();  // คืนทั้งหมดครั้งเดียว
    const allocator = arena.allocator();

    // Allocate หลาย strings สำหรับ C
    const paths = [_][]const u8{
        "/usr/local/lib",
        "/home/user/.config",
        "/tmp/work",
    };

    for (paths) |path| {
        // Copy เป็น null-terminated
        const c_path = try allocator.dupeZ(u8, path);
        const file = c.fopen(c_path.ptr, "r");
        if (file != null) {
            _ = c.fclose(file);
            std.debug.print("Exists: {s}\n", .{path});
        }
    }
    // arena.deinit() คืน memory ทั้งหมดพร้อมกัน
}
```

---

## Step 438: Build system กับ C libraries

การตั้งค่า build system สำหรับโปรเจคที่ใช้ C libraries

### สร้าง C Library จาก Source

```zig
// build.zig - build C library จาก source
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง static library จาก C source
    const cjson = b.addStaticLibrary(.{
        .name = "cjson",
        .target = target,
        .optimize = optimize,
    });

    cjson.addCSourceFiles(.{
        .root = b.path("deps/cJSON"),
        .files = &[_][]const u8{
            "cJSON.c",
            "cJSON_Utils.c",
        },
        .flags = &[_][]const u8{
            "-std=c11",
            "-DCJSON_HIDE_SYMBOLS",
            "-O2",
        },
    });
    cjson.addIncludePath(b.path("deps/cJSON"));
    cjson.linkLibC();

    // สร้าง main executable
    const exe = b.addExecutable(.{
        .name = "json-parser",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe.linkLibrary(cjson);
    exe.addIncludePath(b.path("deps/cJSON"));
    exe.linkLibC();

    b.installArtifact(exe);
}
```

### Vendored vs System Libraries

```zig
// build.zig - เลือกระหว่าง vendored และ system library
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Option: ใช้ system sqlite หรือ bundled version
    const use_system_sqlite = b.option(
        bool,
        "system-sqlite",
        "Use system SQLite instead of bundled version",
    ) orelse false;

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    if (use_system_sqlite) {
        // ใช้ system library
        exe.linkSystemLibrary("sqlite3");
        std.debug.print("Using system SQLite\n", .{});
    } else {
        // ใช้ bundled (vendored) version
        const sqlite = b.addStaticLibrary(.{
            .name = "sqlite3",
            .target = target,
            .optimize = optimize,
        });
        sqlite.addCSourceFile(.{
            .file = b.path("deps/sqlite3/sqlite3.c"),
            .flags = &[_][]const u8{
                "-DSQLITE_THREADSAFE=1",
                "-DSQLITE_ENABLE_FTS5",
                "-O2",
            },
        });
        sqlite.addIncludePath(b.path("deps/sqlite3"));
        sqlite.linkLibC();

        exe.linkLibrary(sqlite);
        exe.addIncludePath(b.path("deps/sqlite3"));
    }

    exe.linkLibC();
    b.installArtifact(exe);
}
```

---

## Step 439: Wrapping C library with Zig API

การ wrap C library ให้มี API แบบ Zig-idiomatic

### ตัวอย่าง: Wrap C JSON Library

```c
// deps/cjson/cjson.h (simplified)
typedef struct cJSON {
    char *string;
    int type;
    char *valuestring;
    int valueint;
    double valuedouble;
    struct cJSON *child;
    struct cJSON *next;
} cJSON;

cJSON* cJSON_Parse(const char* value);
cJSON* cJSON_GetObjectItem(const cJSON* object, const char* string);
char* cJSON_GetStringValue(const cJSON* item);
double cJSON_GetNumberValue(const cJSON* item);
void cJSON_Delete(cJSON* c);
char* cJSON_Print(const cJSON* item);
```

```zig
// src/json.zig - Zig wrapper สำหรับ cJSON
const std = @import("std");
const c = @cImport({
    @cInclude("cjson.h");
});

pub const JsonError = error{
    ParseFailed,
    InvalidType,
    KeyNotFound,
    OutOfMemory,
};

/// JSON Value wrapper
pub const JsonValue = struct {
    ptr: *c.cJSON,
    owned: bool,

    /// Parse JSON string
    pub fn parse(json_str: []const u8) JsonError!JsonValue {
        // สร้าง null-terminated string
        const c_str = json_str.ptr;
        const parsed = c.cJSON_Parse(c_str) orelse
            return JsonError.ParseFailed;

        return .{ .ptr = parsed, .owned = true };
    }

    /// Free memory (เฉพาะที่ owned)
    pub fn deinit(self: JsonValue) void {
        if (self.owned) {
            c.cJSON_Delete(self.ptr);
        }
    }

    /// Get object field
    pub fn getField(self: JsonValue, key: []const u8) JsonError!JsonValue {
        const item = c.cJSON_GetObjectItem(self.ptr, key.ptr) orelse
            return JsonError.KeyNotFound;

        return .{ .ptr = item, .owned = false };
    }

    /// Get string value
    pub fn getString(self: JsonValue) JsonError![]const u8 {
        const str = c.cJSON_GetStringValue(self.ptr) orelse
            return JsonError.InvalidType;
        return std.mem.span(str);
    }

    /// Get number value
    pub fn getNumber(self: JsonValue) f64 {
        return self.ptr.*.valuedouble;
    }

    /// Get int value
    pub fn getInt(self: JsonValue) i64 {
        return @intFromFloat(self.ptr.*.valuedouble);
    }

    /// Convert to string (caller owns memory)
    pub fn toString(self: JsonValue, allocator: std.mem.Allocator) ![]u8 {
        const printed = c.cJSON_Print(self.ptr) orelse
            return error.OutOfMemory;
        defer c.free(printed);

        const str = std.mem.span(printed);
        return allocator.dupe(u8, str);
    }
};

// ตัวอย่างการใช้
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{"name":"Alice","age":30,"city":"Bangkok"}
    ;

    const json = try JsonValue.parse(json_str);
    defer json.deinit();

    const name = try (try json.getField("name")).getString();
    const age_val = try json.getField("age");
    const age = age_val.getInt();

    std.debug.print("Name: {s}\n", .{name});
    std.debug.print("Age: {}\n", .{age});

    const json_str_out = try json.toString(allocator);
    defer allocator.free(json_str_out);
    std.debug.print("JSON: {s}\n", .{json_str_out});
}
```

---

## Step 440: Real project: Zig wrapper สำหรับ SQLite

โปรเจคจริง: สร้าง Zig wrapper สำหรับ SQLite database

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // SQLite as static library (vendored)
    const sqlite3 = b.addStaticLibrary(.{
        .name = "sqlite3",
        .target = target,
        .optimize = optimize,
    });
    sqlite3.addCSourceFile(.{
        .file = b.path("deps/sqlite3/sqlite3.c"),
        .flags = &[_][]const u8{
            "-DSQLITE_THREADSAFE=0",      // single-threaded
            "-DSQLITE_DEFAULT_MEMSTATUS=0",
            "-DSQLITE_DEFAULT_WAL_SYNCHRONOUS=1",
            "-DSQLITE_LIKE_DOESNT_MATCH_BLOBS",
            "-DSQLITE_MAX_EXPR_DEPTH=0",
            "-DSQLITE_OMIT_DECLTYPE",
            "-DSQLITE_OMIT_DEPRECATED",
            "-DSQLITE_OMIT_PROGRESS_CALLBACK",
            "-DSQLITE_OMIT_SHARED_CACHE",
            "-DSQLITE_USE_ALLOCA",
            "-O2",
        },
    });
    sqlite3.addIncludePath(b.path("deps/sqlite3"));
    sqlite3.linkLibC();

    // Main library
    const zig_sqlite = b.addStaticLibrary(.{
        .name = "zig-sqlite",
        .root_source_file = b.path("src/sqlite.zig"),
        .target = target,
        .optimize = optimize,
    });
    zig_sqlite.linkLibrary(sqlite3);
    zig_sqlite.addIncludePath(b.path("deps/sqlite3"));
    zig_sqlite.linkLibC();

    // Example executable
    const exe = b.addExecutable(.{
        .name = "sqlite-example",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.linkLibrary(zig_sqlite);
    exe.addIncludePath(b.path("deps/sqlite3"));
    exe.linkLibC();
    b.installArtifact(exe);

    // Run
    const run = b.addRunArtifact(exe);
    run.step.dependOn(b.getInstallStep());
    b.step("run", "Run example").dependOn(&run.step);

    // Tests
    const tests = b.addTest(.{
        .root_source_file = b.path("src/sqlite.zig"),
        .target = target,
        .optimize = optimize,
    });
    tests.linkLibrary(sqlite3);
    tests.addIncludePath(b.path("deps/sqlite3"));
    tests.linkLibC();
    b.step("test", "Run tests").dependOn(&b.addRunArtifact(tests).step);
}
```

### src/sqlite.zig - Wrapper หลัก

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("sqlite3.h");
});

// ===== Error Types =====
pub const SqliteError = error{
    OpenFailed,
    PrepareFailed,
    BindFailed,
    StepFailed,
    FinalizeFailed,
    NotFound,
    Constraint,
    ReadOnly,
    IoError,
    OutOfMemory,
    Unknown,
};

fn resultToError(rc: c_int) SqliteError {
    return switch (rc) {
        c.SQLITE_CONSTRAINT => SqliteError.Constraint,
        c.SQLITE_READONLY => SqliteError.ReadOnly,
        c.SQLITE_IOERR => SqliteError.IoError,
        c.SQLITE_NOMEM => SqliteError.OutOfMemory,
        else => SqliteError.Unknown,
    };
}

// ===== Database =====
pub const Database = struct {
    db: *c.sqlite3,
    allocator: std.mem.Allocator,

    /// เปิด database (สร้างใหม่ถ้าไม่มี)
    pub fn open(path: []const u8, allocator: std.mem.Allocator) SqliteError!Database {
        var db: ?*c.sqlite3 = null;

        const rc = c.sqlite3_open(path.ptr, &db);
        if (rc != c.SQLITE_OK) {
            if (db) |d| _ = c.sqlite3_close(d);
            return SqliteError.OpenFailed;
        }

        return .{
            .db = db.?,
            .allocator = allocator,
        };
    }

    /// เปิด in-memory database
    pub fn openMemory(allocator: std.mem.Allocator) SqliteError!Database {
        return open(":memory:", allocator);
    }

    /// ปิด database
    pub fn close(self: Database) void {
        _ = c.sqlite3_close(self.db);
    }

    /// Execute SQL (ไม่ return results)
    pub fn exec(self: Database, sql: []const u8) SqliteError!void {
        var err_msg: ?[*:0]u8 = null;

        const rc = c.sqlite3_exec(
            self.db,
            sql.ptr,
            null,
            null,
            &err_msg,
        );

        if (err_msg) |msg| c.sqlite3_free(msg);

        if (rc != c.SQLITE_OK) {
            return resultToError(rc);
        }
    }

    /// Prepare statement
    pub fn prepare(self: Database, sql: []const u8) SqliteError!Statement {
        var stmt: ?*c.sqlite3_stmt = null;

        const rc = c.sqlite3_prepare_v2(
            self.db,
            sql.ptr,
            @intCast(sql.len),
            &stmt,
            null,
        );

        if (rc != c.SQLITE_OK or stmt == null) {
            return SqliteError.PrepareFailed;
        }

        return Statement{
            .stmt = stmt.?,
            .db = self,
        };
    }

    /// Last insert rowid
    pub fn lastInsertRowid(self: Database) i64 {
        return c.sqlite3_last_insert_rowid(self.db);
    }

    /// Error message
    pub fn errorMsg(self: Database) []const u8 {
        return std.mem.span(c.sqlite3_errmsg(self.db));
    }
};

// ===== Statement =====
pub const Statement = struct {
    stmt: *c.sqlite3_stmt,
    db: Database,

    pub fn deinit(self: Statement) void {
        _ = c.sqlite3_finalize(self.stmt);
    }

    pub fn reset(self: Statement) void {
        _ = c.sqlite3_reset(self.stmt);
    }

    /// Bind integer
    pub fn bindInt(self: Statement, col: c_int, value: i64) SqliteError!void {
        const rc = c.sqlite3_bind_int64(self.stmt, col, value);
        if (rc != c.SQLITE_OK) return SqliteError.BindFailed;
    }

    /// Bind text
    pub fn bindText(self: Statement, col: c_int, value: []const u8) SqliteError!void {
        const rc = c.sqlite3_bind_text(
            self.stmt,
            col,
            value.ptr,
            @intCast(value.len),
            c.SQLITE_STATIC,
        );
        if (rc != c.SQLITE_OK) return SqliteError.BindFailed;
    }

    /// Bind float
    pub fn bindFloat(self: Statement, col: c_int, value: f64) SqliteError!void {
        const rc = c.sqlite3_bind_double(self.stmt, col, value);
        if (rc != c.SQLITE_OK) return SqliteError.BindFailed;
    }

    /// Bind null
    pub fn bindNull(self: Statement, col: c_int) SqliteError!void {
        const rc = c.sqlite3_bind_null(self.stmt, col);
        if (rc != c.SQLITE_OK) return SqliteError.BindFailed;
    }

    /// Execute (no result)
    pub fn exec(self: Statement) SqliteError!void {
        const rc = c.sqlite3_step(self.stmt);
        if (rc != c.SQLITE_DONE) {
            return resultToError(rc);
        }
    }

    /// Step to next row
    pub fn step(self: Statement) SqliteError!bool {
        const rc = c.sqlite3_step(self.stmt);
        return switch (rc) {
            c.SQLITE_ROW => true,
            c.SQLITE_DONE => false,
            else => resultToError(rc),
        };
    }

    /// Get integer column
    pub fn columnInt(self: Statement, col: c_int) i64 {
        return c.sqlite3_column_int64(self.stmt, col);
    }

    /// Get text column
    pub fn columnText(self: Statement, col: c_int) []const u8 {
        const text = c.sqlite3_column_text(self.stmt, col);
        if (text == null) return "";
        const len = c.sqlite3_column_bytes(self.stmt, col);
        return text[0..@intCast(len)];
    }

    /// Get float column
    pub fn columnFloat(self: Statement, col: c_int) f64 {
        return c.sqlite3_column_double(self.stmt, col);
    }

    /// Column count
    pub fn columnCount(self: Statement) c_int {
        return c.sqlite3_column_count(self.stmt);
    }

    /// Column name
    pub fn columnName(self: Statement, col: c_int) []const u8 {
        return std.mem.span(c.sqlite3_column_name(self.stmt, col));
    }
};

// ===== Tests =====
test "basic database operations" {
    const allocator = std.testing.allocator;

    // เปิด in-memory database
    const db = try Database.openMemory(allocator);
    defer db.close();

    // สร้าง table
    try db.exec(
        \\CREATE TABLE users (
        \\  id INTEGER PRIMARY KEY AUTOINCREMENT,
        \\  name TEXT NOT NULL,
        \\  age INTEGER,
        \\  score REAL
        \\)
    );

    // Insert
    const insert_stmt = try db.prepare(
        "INSERT INTO users (name, age, score) VALUES (?, ?, ?)"
    );
    defer insert_stmt.deinit();

    try insert_stmt.bindText(1, "Alice");
    try insert_stmt.bindInt(2, 30);
    try insert_stmt.bindFloat(3, 95.5);
    try insert_stmt.exec();
    insert_stmt.reset();

    try insert_stmt.bindText(1, "Bob");
    try insert_stmt.bindInt(2, 25);
    try insert_stmt.bindFloat(3, 87.3);
    try insert_stmt.exec();

    // Query
    const select_stmt = try db.prepare("SELECT id, name, age, score FROM users ORDER BY id");
    defer select_stmt.deinit();

    var count: usize = 0;
    while (try select_stmt.step()) {
        count += 1;
        const id = select_stmt.columnInt(0);
        const name = select_stmt.columnText(1);
        const age = select_stmt.columnInt(2);
        const score = select_stmt.columnFloat(3);

        std.debug.print("User {}: {s}, age={}, score={d:.1}\n", .{ id, name, age, score });
    }

    try std.testing.expectEqual(@as(usize, 2), count);
}
```

### src/main.zig

```zig
const std = @import("std");
const sqlite = @import("sqlite.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // เปิด database
    const db = try sqlite.Database.open("data.db", allocator);
    defer db.close();

    std.debug.print("SQLite version: {s}\n", .{std.mem.span(
        @import("sqlite3.h").sqlite3_version,
    )});

    // สร้าง schema
    try db.exec(
        \\CREATE TABLE IF NOT EXISTS products (
        \\  id INTEGER PRIMARY KEY AUTOINCREMENT,
        \\  name TEXT NOT NULL,
        \\  price REAL NOT NULL,
        \\  stock INTEGER DEFAULT 0
        \\)
    );

    // เพิ่มสินค้า
    const insert = try db.prepare(
        "INSERT OR IGNORE INTO products (name, price, stock) VALUES (?, ?, ?)"
    );
    defer insert.deinit();

    const products = [_]struct { name: []const u8, price: f64, stock: i64 }{
        .{ .name = "Apple",  .price = 15.0,  .stock = 100 },
        .{ .name = "Banana", .price = 8.5,   .stock = 150 },
        .{ .name = "Orange", .price = 20.0,  .stock = 80  },
    };

    for (products) |p| {
        try insert.bindText(1, p.name);
        try insert.bindFloat(2, p.price);
        try insert.bindInt(3, p.stock);
        try insert.exec();
        insert.reset();
    }

    // แสดงสินค้า
    const select = try db.prepare(
        "SELECT name, price, stock FROM products ORDER BY price"
    );
    defer select.deinit();

    std.debug.print("\nProducts:\n", .{});
    std.debug.print("{s:<20} {s:>10} {s:>8}\n", .{ "Name", "Price", "Stock" });
    std.debug.print("{s}\n", .{"-" ** 40});

    while (try select.step()) {
        const name = select.columnText(0);
        const price = select.columnFloat(1);
        const stock = select.columnInt(2);

        std.debug.print("{s:<20} {d:>10.2} {d:>8}\n", .{ name, price, stock });
    }
}
```

---

## สรุป Part 44: C Interoperability ขั้นสูง

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 431 | Static vs Dynamic | เลือก link mode ที่เหมาะสม |
| 432 | pkg-config | หา library flags |
| 433 | OpenSSL, zlib, SQLite | ใช้ real C libraries |
| 434 | Callbacks | Function pointers |
| 435 | Variadic functions | printf-style functions |
| 436 | errno | Error handling จาก C |
| 437 | Memory ownership | จัดการ memory ข้ามภาษา |
| 438 | Build system | ตั้งค่า build สำหรับ C |
| 439 | Wrapping C API | สร้าง Zig-friendly API |
| 440 | SQLite wrapper | โปรเจคจริง |

### Best Practices สำหรับ C Interop

```zig
// ✓ ดี: ชัดเจนว่าใครเป็น owner
const c_str = c.strdup("hello");
defer c.free(c_str);

// ✓ ดี: reset errno ก่อนใช้
c.errno = 0;
const result = c.some_function();
if (result < 0) {
    // ตรวจสอบ errno
}

// ✓ ดี: Wrap C API ให้ใช้งานง่าย
pub fn openFile(path: []const u8) !*File {
    const f = c.fopen(path.ptr, "r") orelse
        return error.CannotOpen;
    return f;
}

// ✗ ไม่ดี: ไม่ handle null return
const file = c.fopen("x.txt", "r");
_ = c.fread(buffer, 1, 100, file); // crash ถ้า file เป็น null!
```

> **ต่อไป**: Part 45 - Async ใน Zig
