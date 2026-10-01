# Part 43: C Interoperability พื้นฐาน

> **หลักสูตร Zig ภาษาไทย** | Steps 421-430

---

## Step 421: ทำไม Zig ถึง interop กับ C ได้ดี?

Zig ถูกออกแบบมาให้ทำงานร่วมกับ C ได้อย่างราบรื่น นี่คือหนึ่งในจุดแข็งที่สำคัญที่สุดของภาษา

### เหตุผลที่ Zig + C เข้ากันได้ดี

**1. Compatible ABI**
Zig ใช้ C ABI เป็นค่าเริ่มต้น ทำให้สามารถเรียกใช้ฟังก์ชัน C ได้โดยตรง

```zig
// ฟังก์ชัน C สามารถเรียกจาก Zig ได้ตรงๆ
const c = @cImport({
    @cInclude("stdio.h");
});

pub fn main() void {
    _ = c.printf("Hello from C!\n");
}
```

**2. ไม่มี Runtime overhead**
Zig ไม่มี GC หรือ runtime หนักๆ ทำให้ integrate กับ C ได้ง่าย

**3. Compile C ในตัว**
Zig มี C compiler built-in (ใช้ LLVM) สามารถ compile C files ได้โดยตรง

```zig
// build.zig - compile C file ร่วมกับ Zig
const exe = b.addExecutable(.{ .name = "mixed", ... });
exe.addCSourceFile(.{
    .file = b.path("src/helper.c"),
    .flags = &[_][]const u8{"-std=c11", "-O2"},
});
```

**4. Cross-language LTO**
Zig รองรับ Link Time Optimization ข้ามภาษาได้

**5. เป็น C compiler ด้วย**
สามารถใช้ Zig เป็น C compiler: `zig cc` แทน `gcc`

```bash
# ใช้ zig เป็น C compiler
zig cc hello.c -o hello

# ใช้ zig เป็น C++ compiler
zig c++ hello.cpp -o hello
```

### สถานการณ์ที่ต้องการ C interop

1. **ใช้ existing C libraries** - OpenSSL, zlib, libpng, FFmpeg
2. **เรียกใช้ system APIs** - Linux syscalls, Windows API
3. **Migrate codebase** - แปลง C ไปเป็น Zig ทีละส่วน
4. **Performance critical** - ใช้ C code ที่ optimize แล้ว
5. **Plugin systems** - ระบบที่รับ C plugins

---

## Step 422: @cInclude - importing C headers

`@cInclude` นำเข้า C header files เข้ามาใน Zig

### การใช้งานพื้นฐาน

```zig
const std = @import("std");

// นำเข้า C headers
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
    @cInclude("string.h");
    @cInclude("math.h");
});

pub fn main() void {
    // ใช้ C functions โดยตรง
    _ = c.printf("Hello, %s!\n", "World");

    const len = c.strlen("Hello, World!");
    _ = c.printf("Length: %zu\n", len);

    const pi = c.M_PI;
    _ = c.printf("PI = %.5f\n", pi);
}
```

### @cImport vs @cInclude

```zig
// @cImport รับ block ของ C directives
const c = @cImport({
    // @cInclude เหมือน #include ใน C
    @cInclude("stdio.h");

    // @cDefine เหมือน #define ใน C
    @cDefine("_POSIX_C_SOURCE", "200809L");
    @cInclude("sys/types.h");

    // @cUndef ยกเลิก define
    @cUndef("DEBUG");
    @cInclude("mylib.h");
});
```

### Include Paths

```zig
// ใน build.zig - เพิ่ม include paths
exe.addIncludePath(b.path("include/"));
exe.addIncludePath(.{ .cwd_relative = "/usr/local/include" });

// สำหรับ system includes
exe.addSystemIncludePath(.{ .cwd_relative = "/usr/include" });
```

### ตัวอย่างการ include custom header

```c
// include/mylib.h
#ifndef MYLIB_H
#define MYLIB_H

#include <stdint.h>
#include <stdbool.h>

typedef struct {
    int32_t x;
    int32_t y;
    int32_t width;
    int32_t height;
} Rectangle;

Rectangle rect_new(int32_t x, int32_t y, int32_t w, int32_t h);
bool rect_contains(Rectangle r, int32_t px, int32_t py);
int32_t rect_area(Rectangle r);

#endif
```

```zig
// src/main.zig
const std = @import("std");
const c = @cImport({
    @cInclude("mylib.h");
});

pub fn main() void {
    const rect = c.rect_new(0, 0, 100, 50);
    const area = c.rect_area(rect);
    const contains = c.rect_contains(rect, 50, 25);

    std.debug.print("Area: {}\n", .{area});
    std.debug.print("Contains (50,25): {}\n", .{contains});
}
```

---

## Step 423: การใช้ C functions

วิธีการเรียกใช้ C functions ต่างๆ

### Standard C Library Functions

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
    @cInclude("string.h");
    @cInclude("time.h");
    @cInclude("math.h");
});

pub fn main() void {
    // ===== stdio functions =====
    _ = c.printf("Hello, World!\n");

    var buf: [256]u8 = undefined;
    _ = c.sprintf(&buf, "Value: %d", @as(c_int, 42));
    _ = c.puts(&buf);

    // ===== stdlib functions =====
    const ptr = c.malloc(100);
    defer c.free(ptr);

    // Random number
    _ = c.srand(@as(c_uint, @intCast(c.time(null))));
    const rand_val = c.rand();
    _ = c.printf("Random: %d\n", rand_val);

    // ===== string functions =====
    var str: [32]u8 = undefined;
    _ = c.strcpy(&str, "Hello");
    _ = c.strcat(&str, ", World!");
    _ = c.printf("String: %s\n", &str);

    const len = c.strlen(&str);
    _ = c.printf("Length: %zu\n", len);

    // ===== math functions =====
    const sqrt_val = c.sqrt(@as(f64, 16.0));
    _ = c.printf("sqrt(16) = %.1f\n", sqrt_val);

    const sin_val = c.sin(c.M_PI / 2.0);
    _ = c.printf("sin(PI/2) = %.1f\n", sin_val);
}
```

### Memory Functions

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

pub fn main() !void {
    const allocator = std.heap.c_allocator;

    // ใช้ C malloc/free ผ่าน Zig allocator
    const buf = try allocator.alloc(u8, 1024);
    defer allocator.free(buf);

    // หรือใช้ C functions โดยตรง
    const raw_ptr = c.malloc(1024) orelse {
        std.debug.print("malloc failed!\n", .{});
        return;
    };
    defer c.free(raw_ptr);

    // memset และ memcpy
    const typed_ptr: [*]u8 = @ptrCast(raw_ptr);
    _ = c.memset(raw_ptr, 0, 1024);

    const src = "Hello, World!";
    _ = c.memcpy(raw_ptr, src.ptr, src.len);

    // อ่านค่ากลับ
    std.debug.print("Data: {s}\n", .{typed_ptr[0..src.len]});
}
```

### File I/O Functions

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
});

pub fn main() void {
    // เปิดไฟล์
    const file = c.fopen("test.txt", "w") orelse {
        std.debug.print("Cannot open file!\n", .{});
        return;
    };
    defer _ = c.fclose(file);

    // เขียนไฟล์
    _ = c.fprintf(file, "Line 1\n");
    _ = c.fprintf(file, "Line 2: %d\n", @as(c_int, 42));
    _ = c.fputs("Line 3\n", file);

    std.debug.print("File written successfully\n", .{});

    // อ่านไฟล์
    const read_file = c.fopen("test.txt", "r") orelse {
        std.debug.print("Cannot read file!\n", .{});
        return;
    };
    defer _ = c.fclose(read_file);

    var line_buf: [256]u8 = undefined;
    while (c.fgets(&line_buf, 256, read_file) != null) {
        _ = c.printf("%s", &line_buf);
    }
}
```

---

## Step 424: C types mapping (c_int, c_char, etc.)

Zig มี types พิเศษสำหรับแมปกับ C types

### ตารางแมป C types กับ Zig types

| C Type | Zig Type | ขนาด (ทั่วไป) |
|--------|----------|----------------|
| `char` | `c_char` | 8-bit |
| `unsigned char` | `c_uchar` | 8-bit |
| `short` | `c_short` | 16-bit |
| `unsigned short` | `c_ushort` | 16-bit |
| `int` | `c_int` | 32-bit |
| `unsigned int` | `c_uint` | 32-bit |
| `long` | `c_long` | 32 or 64-bit |
| `unsigned long` | `c_ulong` | 32 or 64-bit |
| `long long` | `c_longlong` | 64-bit |
| `unsigned long long` | `c_ulonglong` | 64-bit |
| `float` | `f32` | 32-bit |
| `double` | `f64` | 64-bit |
| `size_t` | `usize` | platform-dependent |
| `ptrdiff_t` | `isize` | platform-dependent |
| `void *` | `*anyopaque` | pointer |
| `const char *` | `[*:0]const u8` | null-terminated string |

### การแปลง Types

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdint.h");
});

pub fn main() void {
    // c_int
    const x: c_int = 42;
    const y: i32 = @intCast(x);  // c_int -> i32
    _ = y;

    // c_uint
    const u: c_uint = 100;
    const v: u32 = @intCast(u);
    _ = v;

    // c_long (ขนาดขึ้นอยู่กับ platform)
    const l: c_long = 1000;
    _ = l;

    // size_t
    const sz: usize = 1024;
    const c_sz: c_uint = @intCast(sz);  // usize -> c_uint
    _ = c.printf("Size: %u\n", c_sz);

    // float
    const f: f64 = 3.14;
    _ = c.printf("Float: %.2f\n", f);

    // Pointer conversion
    var val: i32 = 99;
    const ptr: *i32 = &val;
    const c_ptr: *c_int = @ptrCast(ptr);
    _ = c.printf("Value via C ptr: %d\n", c_ptr.*);
}
```

### Fixed-size Integer Types

```zig
const c = @cImport({
    @cInclude("stdint.h");
});

pub fn typeExample() void {
    // Fixed-size types จาก stdint.h
    const a: c.int8_t = -127;
    const b: c.uint8_t = 255;
    const c_val: c.int16_t = -32768;
    const d: c.uint16_t = 65535;
    const e: c.int32_t = -2147483648;
    const f: c.uint32_t = 4294967295;
    const g: c.int64_t = -9223372036854775808;
    const h: c.uint64_t = 18446744073709551615;

    // เทียบกับ Zig native types
    const za: i8 = a;
    const zb: u8 = b;
    const zc: i16 = c_val;
    const zd: u16 = d;
    _ = za; _ = zb; _ = zc; _ = zd; _ = e; _ = f; _ = g; _ = h;
}
```

### bool Type

```zig
const c = @cImport({
    @cInclude("stdbool.h");
});

pub fn boolExample() void {
    // C bool
    const c_true: c.bool = c.true;
    const c_false: c.bool = c.false;

    // แปลงเป็น Zig bool
    const zig_true: bool = c_true != 0;
    const zig_false: bool = c_false != 0;

    // แปลง Zig bool เป็น C bool
    const back_to_c: c.bool = if (zig_true) 1 else 0;
    _ = c_false; _ = zig_false; _ = back_to_c;
}
```

---

## Step 425: C strings (null-terminated)

C strings ใช้ null-terminator (`\0`) บอกจุดสิ้นสุด ต่างจาก Zig strings

### C String Types ใน Zig

```zig
// ประเภทของ C strings ใน Zig:

// [*:0]const u8 - null-terminated string (read-only)
// [*:0]u8       - null-terminated string (mutable)
// [*]const u8   - pointer to bytes (ไม่มี null terminator)
// [:0]const u8  - slice ที่มี null terminator
```

### แปลง Zig string ↔ C string

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("string.h");
});

pub fn main() void {
    // ===== Zig string literal เป็น C string =====
    // String literals ใน Zig เป็น null-terminated อยู่แล้ว
    const zig_str: []const u8 = "Hello, World!";
    const c_str: [*:0]const u8 = zig_str.ptr;  // แปลงเป็น C string

    _ = c.printf("%s\n", c_str);
    _ = c.puts(c_str);

    // ===== C string เป็น Zig string =====
    const c_hello: [*:0]const u8 = "Hello from C";
    const len = c.strlen(c_hello);
    const zig_slice: []const u8 = c_hello[0..len];

    std.debug.print("Zig slice: {s}\n", .{zig_slice});
    std.debug.print("Length: {}\n", .{len});

    // ===== สร้าง C string จาก buffer =====
    var buffer: [64:0]u8 = undefined;  // null-terminated array
    _ = c.sprintf(&buffer, "Number: %d", @as(c_int, 42));

    const result: [*:0]const u8 = &buffer;
    _ = c.printf("Result: %s\n", result);
}
```

### String Functions

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("string.h");
    @cInclude("stdio.h");
    @cInclude("ctype.h");
});

pub fn main() void {
    var str1: [64:0]u8 = undefined;
    var str2: [64:0]u8 = undefined;

    _ = c.strcpy(&str1, "Hello");
    _ = c.strcpy(&str2, "World");

    // Concatenation
    _ = c.strcat(&str1, ", ");
    _ = c.strcat(&str1, &str2);
    _ = c.printf("Concat: %s\n", &str1);

    // Compare
    const cmp = c.strcmp(&str1, "Hello, World");
    if (cmp == 0) {
        _ = c.printf("Strings are equal\n");
    } else {
        _ = c.printf("Strings differ\n");
    }

    // Find
    const found = c.strstr(&str1, "World");
    if (found != null) {
        _ = c.printf("Found 'World' at offset\n");
    }

    // Length
    const len = c.strlen(&str1);
    _ = c.printf("Length: %zu\n", len);

    // Copy with limit
    var dest: [10:0]u8 = undefined;
    _ = c.strncpy(&dest, &str1, 9);
    dest[9] = 0;  // Ensure null termination
    _ = c.printf("Limited copy: %s\n", &dest);
}
```

### String Conversion ที่ Safe

```zig
const std = @import("std");

// ฟังก์ชัน helper สำหรับแปลง C string เป็น Zig slice
pub fn cstrToSlice(c_str: [*:0]const u8) []const u8 {
    return std.mem.span(c_str);
}

// ฟังก์ชัน helper สำหรับแปลง Zig string เป็น C string
// (ต้องมั่นใจว่า string เป็น null-terminated)
pub fn sliceToCstr(slice: [:0]const u8) [*:0]const u8 {
    return slice.ptr;
}

pub fn main() void {
    const c_str: [*:0]const u8 = "Hello from C!";
    const zig_str = cstrToSlice(c_str);

    std.debug.print("Zig string: {s}\n", .{zig_str});
    std.debug.print("Length: {}\n", .{zig_str.len});

    // แปลงกลับ
    const back: [:0]const u8 = "Back to C!\x00";
    const c_back = sliceToCstr(back[0..back.len - 1 :0]);
    _ = c_back;
}
```

---

## Step 426: C structs

การใช้งาน C structs ใน Zig

### Import และใช้ C structs

```c
// include/geometry.h
#include <stdint.h>
#include <stdbool.h>

typedef struct {
    float x;
    float y;
} Point2D;

typedef struct {
    Point2D center;
    float radius;
} Circle;

typedef struct {
    Point2D top_left;
    float width;
    float height;
} Rect;

float point_distance(Point2D a, Point2D b);
bool circle_contains(Circle c, Point2D p);
float rect_area(Rect r);
```

```zig
// src/main.zig
const std = @import("std");
const c = @cImport({
    @cInclude("geometry.h");
    @cInclude("math.h");
});

pub fn main() void {
    // สร้าง C structs
    const p1 = c.Point2D{ .x = 0.0, .y = 0.0 };
    const p2 = c.Point2D{ .x = 3.0, .y = 4.0 };

    const dist = c.point_distance(p1, p2);
    std.debug.print("Distance: {d:.1}\n", .{dist});

    const circle = c.Circle{
        .center = .{ .x = 5.0, .y = 5.0 },
        .radius = 3.0,
    };

    const test_point = c.Point2D{ .x = 5.0, .y = 7.0 };
    const inside = c.circle_contains(circle, test_point);
    std.debug.print("Point inside circle: {}\n", .{inside});

    const rect = c.Rect{
        .top_left = .{ .x = 0.0, .y = 0.0 },
        .width = 10.0,
        .height = 5.0,
    };
    const area = c.rect_area(rect);
    std.debug.print("Rect area: {d:.1}\n", .{area});
}
```

### C Struct Pointers

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

// Linked list ใน C style
const Node = extern struct {
    value: c_int,
    next: ?*Node,
};

pub fn createNode(value: c_int) *Node {
    const node: *Node = @ptrCast(@alignCast(c.malloc(@sizeOf(Node)) orelse @panic("OOM")));
    node.value = value;
    node.next = null;
    return node;
}

pub fn freeList(head: *Node) void {
    var current: ?*Node = head;
    while (current) |node| {
        const next = node.next;
        c.free(node);
        current = next;
    }
}

pub fn main() void {
    // สร้าง linked list
    const head = createNode(1);
    head.next = createNode(2);
    head.next.?.next = createNode(3);

    // traverse
    var current: ?*Node = head;
    while (current) |node| {
        std.debug.print("Value: {}\n", .{node.value});
        current = node.next;
    }

    freeList(head);
}
```

### Nested Structs

```zig
const c = @cImport({
    @cInclude("stdint.h");
});

// กำหนด C-compatible struct ใน Zig
const Color = extern struct {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
};

const Pixel = extern struct {
    pos: extern struct {
        x: i32,
        y: i32,
    },
    color: Color,
};

pub fn pixelExample() void {
    const pixel = Pixel{
        .pos = .{ .x = 100, .y = 200 },
        .color = .{ .r = 255, .g = 128, .b = 0, .a = 255 },
    };

    std.debug.print("Pixel at ({}, {})\n", .{ pixel.pos.x, pixel.pos.y });
    std.debug.print("Color: #{x:0>2}{x:0>2}{x:0>2}\n", .{
        pixel.color.r,
        pixel.color.g,
        pixel.color.b,
    });
}

const std = @import("std");
```

---

## Step 427: C pointers และ void*

การจัดการ pointers แบบ C ใน Zig

### Pointer Types

```zig
const std = @import("std");

pub fn pointerTypes() void {
    var x: i32 = 42;

    // Zig pointer (ไม่เป็น null)
    const zig_ptr: *i32 = &x;

    // Optional Zig pointer (อาจเป็น null)
    const opt_ptr: ?*i32 = &x;
    const null_ptr: ?*i32 = null;

    // C pointer (ไม่มีการตรวจสอบ null, เหมือน C)
    const c_ptr: [*c]i32 = &x;

    // many-item pointer
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    const many_ptr: [*]i32 = &arr;

    std.debug.print("zig: {}\n", .{zig_ptr.*});
    std.debug.print("opt: {?}\n", .{opt_ptr});
    std.debug.print("null: {?}\n", .{null_ptr});
    std.debug.print("c_ptr: {}\n", .{c_ptr.*});
    std.debug.print("many[2]: {}\n", .{many_ptr[2]});
}
```

### void* (anyopaque)

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
});

// void* ใน Zig คือ *anyopaque
pub fn voidPointerExample() void {
    // malloc return void*
    const raw_ptr: ?*anyopaque = c.malloc(100);
    defer c.free(raw_ptr);

    if (raw_ptr) |ptr| {
        // แปลง void* เป็น pointer ที่ต้องการ
        const int_ptr: *i32 = @ptrCast(@alignCast(ptr));
        int_ptr.* = 42;

        // แปลงกลับเป็น void*
        const back_to_void: *anyopaque = int_ptr;
        _ = back_to_void;

        std.debug.print("Value: {}\n", .{int_ptr.*});
    }
}

// ฟังก์ชันที่รับ void* (user_data pattern)
fn callback(user_data: ?*anyopaque, value: i32) void {
    if (user_data) |data| {
        const counter: *i32 = @ptrCast(@alignCast(data));
        counter.* += value;
    }
}

pub fn callbackExample() void {
    var total: i32 = 0;

    callback(&total, 10);
    callback(&total, 20);
    callback(&total, 30);

    std.debug.print("Total: {}\n", .{total});  // 60
}
```

### Pointer Arithmetic

```zig
const std = @import("std");

pub fn pointerArithmetic() void {
    var arr = [_]u8{ 10, 20, 30, 40, 50 };
    const ptr: [*]u8 = &arr;

    // เข้าถึงด้วย index
    std.debug.print("arr[0] = {}\n", .{ptr[0]});
    std.debug.print("arr[2] = {}\n", .{ptr[2]});

    // pointer arithmetic
    const ptr2 = ptr + 2;
    std.debug.print("*(ptr+2) = {}\n", .{ptr2[0]});

    // แปลงเป็น slice
    const slice: []u8 = ptr[0..arr.len];
    for (slice, 0..) |val, i| {
        std.debug.print("slice[{}] = {}\n", .{ i, val });
    }
}
```

---

## Step 428: การ link C libraries

การ link กับ C libraries ใน build system

### Link System Libraries

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Link กับ system libraries
    exe.linkSystemLibrary("c");      // libc (มักไม่จำเป็น - Zig link อัตโนมัติ)
    exe.linkSystemLibrary("m");      // libm (math)
    exe.linkSystemLibrary("pthread"); // pthreads
    exe.linkSystemLibrary("z");      // zlib
    exe.linkSystemLibrary("ssl");    // OpenSSL
    exe.linkSystemLibrary("crypto"); // OpenSSL crypto

    // ต้องการ libc
    exe.linkLibC();

    b.installArtifact(exe);
}
```

### เพิ่ม C Source Files

```zig
// build.zig - รวม C source files
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "mixed-project",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่มไฟล์ C เดี่ยว
    exe.addCSourceFile(.{
        .file = b.path("src/c_helper.c"),
        .flags = &[_][]const u8{
            "-std=c11",
            "-Wall",
            "-Wextra",
            "-O2",
        },
    });

    // เพิ่มหลายไฟล์พร้อมกัน
    exe.addCSourceFiles(.{
        .root = b.path("src/c"),
        .files = &[_][]const u8{
            "math_helpers.c",
            "string_utils.c",
            "file_ops.c",
        },
        .flags = &[_][]const u8{ "-std=c11", "-O2" },
    });

    // เพิ่ม include paths
    exe.addIncludePath(b.path("include"));

    exe.linkLibC();
    b.installArtifact(exe);
}
```

### Link กับ Precompiled Library

```zig
// build.zig - link กับ .a หรือ .so ที่มีอยู่แล้ว
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

    // เพิ่ม library path
    exe.addLibraryPath(b.path("libs/"));
    exe.addLibraryPath(.{ .cwd_relative = "/usr/local/lib" });

    // Link กับ library
    exe.linkSystemLibrary("mylib");   // จะหา libmylib.a หรือ libmylib.so

    // หรือ link โดยตรง
    // exe.addObjectFile(b.path("libs/mylib.a"));

    exe.addIncludePath(b.path("include/"));
    exe.linkLibC();

    b.installArtifact(exe);
}
```

---

## Step 429: Calling Zig from C

เราสามารถ export ฟังก์ชัน Zig ให้ C เรียกได้

### Export Zig Functions

```zig
// src/ziglib.zig
const std = @import("std");

// ใช้ export เพื่อให้ C เรียกได้
// ใช้ callconv(.C) สำหรับ C calling convention
export fn zig_add(a: c_int, b: c_int) c_int {
    return a + b;
}

export fn zig_fibonacci(n: c_uint) c_ulonglong {
    if (n <= 1) return n;
    var prev: u64 = 0;
    var curr: u64 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

export fn zig_greet(name: [*:0]const u8) void {
    const name_slice = std.mem.span(name);
    std.debug.print("สวัสดี, {s}! (from Zig)\n", .{name_slice});
}

// Export struct
export fn zig_create_point(x: f32, y: f32) Point {
    return .{ .x = x, .y = y };
}

const Point = extern struct {
    x: f32,
    y: f32,
};
```

### Header File สำหรับ Zig Library

```c
// include/ziglib.h
#ifndef ZIGLIB_H
#define ZIGLIB_H

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

// ประกาศ functions ที่ export จาก Zig
int zig_add(int a, int b);
unsigned long long zig_fibonacci(unsigned int n);
void zig_greet(const char* name);

typedef struct {
    float x;
    float y;
} ZigPoint;

ZigPoint zig_create_point(float x, float y);

#ifdef __cplusplus
}
#endif

#endif /* ZIGLIB_H */
```

### C Code ที่ใช้ Zig Library

```c
// src/main.c
#include <stdio.h>
#include "ziglib.h"

int main() {
    // เรียก Zig functions
    int sum = zig_add(10, 20);
    printf("10 + 20 = %d\n", sum);

    unsigned long long fib = zig_fibonacci(10);
    printf("fibonacci(10) = %llu\n", fib);

    zig_greet("World");

    ZigPoint p = zig_create_point(3.0f, 4.0f);
    printf("Point: (%.1f, %.1f)\n", p.x, p.y);

    return 0;
}
```

### build.zig สำหรับ Mixed Project

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง Zig library ที่ C เรียกได้
    const zig_lib = b.addSharedLibrary(.{
        .name = "ziglib",
        .root_source_file = b.path("src/ziglib.zig"),
        .target = target,
        .optimize = optimize,
    });
    zig_lib.linkLibC();

    // สร้าง C executable ที่ใช้ Zig library
    const c_exe = b.addExecutable(.{
        .name = "c-app",
        .target = target,
        .optimize = optimize,
    });
    c_exe.addCSourceFile(.{
        .file = b.path("src/main.c"),
        .flags = &[_][]const u8{"-std=c11"},
    });
    c_exe.addIncludePath(b.path("include/"));
    c_exe.linkLibrary(zig_lib);
    c_exe.linkLibC();

    b.installArtifact(zig_lib);
    b.installArtifact(c_exe);
}
```

---

## Step 430: C standard library functions

ฟังก์ชันสำคัญใน C standard library

### Memory Management

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

pub fn memoryFunctions() void {
    // malloc - allocate memory
    const ptr = c.malloc(256);
    defer c.free(ptr);

    if (ptr == null) {
        std.debug.print("malloc failed!\n", .{});
        return;
    }

    // calloc - allocate and zero memory
    const zero_ptr = c.calloc(10, @sizeOf(c_int));
    defer c.free(zero_ptr);

    // realloc - resize memory
    var dyn_ptr = c.malloc(100);
    dyn_ptr = c.realloc(dyn_ptr, 200);  // ขยาย
    defer c.free(dyn_ptr);

    // memset - fill memory
    _ = c.memset(ptr, 0, 256);          // zero-fill
    _ = c.memset(ptr, 0xFF, 256);       // fill with 0xFF

    // memcpy - copy memory
    const src = "Hello, World!";
    _ = c.memcpy(ptr, src.ptr, src.len);

    // memmove - copy overlapping memory safely
    const buf: [*]u8 = @ptrCast(ptr);
    _ = c.memmove(buf + 5, buf, 8);     // safe overlap copy

    // memcmp - compare memory
    const a = "Hello";
    const b_str = "Hello";
    const result = c.memcmp(a.ptr, b_str.ptr, 5);
    std.debug.print("memcmp result: {}\n", .{result});  // 0 = equal
}
```

### String Functions ทั้งหมด

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("string.h");
    @cInclude("ctype.h");
});

pub fn stringFunctions() void {
    var buf1: [64:0]u8 = undefined;
    var buf2: [64:0]u8 = undefined;

    // ===== Copy & Concatenate =====
    _ = c.strcpy(&buf1, "Hello");
    _ = c.strncpy(&buf2, "World!!!", 5);
    buf2[5] = 0;

    _ = c.strcat(&buf1, ", ");
    _ = c.strncat(&buf1, &buf2, 5);

    std.debug.print("Result: {s}\n", .{std.mem.sliceTo(&buf1, 0)});

    // ===== Search =====
    const pos = c.strchr(&buf1, 'W');
    if (pos != null) {
        std.debug.print("Found 'W'\n", .{});
    }

    const last_pos = c.strrchr(&buf1, 'l');
    _ = last_pos;

    const sub = c.strstr(&buf1, "World");
    if (sub != null) {
        std.debug.print("Found 'World'\n", .{});
    }

    // ===== Compare =====
    const cmp1 = c.strcmp("abc", "abc");  // 0
    const cmp2 = c.strcmp("abc", "abd");  // negative
    const cmp3 = c.strcmp("abd", "abc");  // positive
    _ = cmp1; _ = cmp2; _ = cmp3;

    // Case-insensitive (POSIX)
    // const icmp = c.strcasecmp("Hello", "hello");  // 0

    // ===== Conversion =====
    const num_str = "42";
    const num = c.atoi(num_str.ptr);
    std.debug.print("atoi: {}\n", .{num});

    const float_str = "3.14";
    const float_val = c.atof(float_str.ptr);
    std.debug.print("atof: {d:.2}\n", .{float_val});

    // strtol - more robust
    var end_ptr: [*c]u8 = undefined;
    const long_val = c.strtol("123abc", &end_ptr, 10);  // base 10
    std.debug.print("strtol: {}\n", .{long_val});

    // ===== Character Classification =====
    const ch: c_int = 'A';
    std.debug.print("isalpha: {}\n", .{c.isalpha(ch) != 0});
    std.debug.print("isdigit: {}\n", .{c.isdigit(ch) != 0});
    std.debug.print("isupper: {}\n", .{c.isupper(ch) != 0});
    std.debug.print("tolower: {c}\n", .{@as(u8, @intCast(c.tolower(ch)))});
}
```

### Math Functions

```zig
const c = @cImport({
    @cInclude("math.h");
    @cInclude("stdio.h");
});

pub fn mathFunctions() void {
    // Basic math
    _ = c.printf("sqrt(2)  = %.4f\n", c.sqrt(2.0));
    _ = c.printf("cbrt(8)  = %.4f\n", c.cbrt(8.0));
    _ = c.printf("pow(2,8) = %.0f\n", c.pow(2.0, 8.0));
    _ = c.printf("exp(1)   = %.4f\n", c.exp(1.0));
    _ = c.printf("log(e)   = %.4f\n", c.log(c.M_E));
    _ = c.printf("log2(8)  = %.4f\n", c.log2(8.0));
    _ = c.printf("log10(100)= %.4f\n", c.log10(100.0));

    // Trigonometry
    _ = c.printf("sin(PI/6) = %.4f\n", c.sin(c.M_PI / 6.0));
    _ = c.printf("cos(PI/3) = %.4f\n", c.cos(c.M_PI / 3.0));
    _ = c.printf("tan(PI/4) = %.4f\n", c.tan(c.M_PI / 4.0));
    _ = c.printf("asin(0.5) = %.4f\n", c.asin(0.5));
    _ = c.printf("acos(0.5) = %.4f\n", c.acos(0.5));
    _ = c.printf("atan2(1,1)= %.4f\n", c.atan2(1.0, 1.0));

    // Rounding
    _ = c.printf("floor(3.7) = %.0f\n", c.floor(3.7));
    _ = c.printf("ceil(3.2)  = %.0f\n", c.ceil(3.2));
    _ = c.printf("round(3.5) = %.0f\n", c.round(3.5));
    _ = c.printf("trunc(3.9) = %.0f\n", c.trunc(3.9));

    // Absolute value
    _ = c.printf("fabs(-5.5) = %.1f\n", c.fabs(-5.5));
    _ = c.printf("abs(-42)   = %d\n", c.abs(@as(c_int, -42)));
}
```

### Time Functions

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("time.h");
    @cInclude("stdio.h");
});

pub fn timeFunctions() void {
    // เวลาปัจจุบัน
    const now = c.time(null);
    _ = c.printf("Unix timestamp: %ld\n", now);

    // แปลงเป็น local time
    const local = c.localtime(&now);
    if (local != null) {
        var time_str: [64]u8 = undefined;
        _ = c.strftime(&time_str, 64, "%Y-%m-%d %H:%M:%S", local);
        _ = c.printf("Local time: %s\n", &time_str);
    }

    // วัดเวลาที่ใช้
    const start = c.clock();
    // ทำงานบางอย่าง...
    var sum: u64 = 0;
    for (0..1000000) |i| sum += i;
    const end = c.clock();

    const elapsed = @as(f64, @floatFromInt(end - start)) /
                    @as(f64, @floatFromInt(c.CLOCKS_PER_SEC));
    _ = c.printf("Time: %.6f seconds, sum=%llu\n", elapsed, sum);
}
```

---

## สรุป Part 43: C Interoperability พื้นฐาน

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 421 | ทำไม Zig + C | จุดแข็งของ C interop |
| 422 | @cInclude | Import C headers |
| 423 | C functions | เรียกใช้ฟังก์ชัน C |
| 424 | C types | การแมป types |
| 425 | C strings | Null-terminated strings |
| 426 | C structs | การใช้งาน structs |
| 427 | C pointers | void* และ pointer types |
| 428 | Link libraries | การ link C libraries |
| 429 | Export to C | ให้ C เรียก Zig ได้ |
| 430 | stdlib functions | Standard library |

### Tips สำหรับ C Interop

```zig
// 1. ใช้ @cImport เพียงที่เดียวในโปรเจค
// สร้างไฟล์ c_imports.zig
pub const c = @cImport({
    @cInclude("stdio.h");
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

// 2. Wrap C functions ด้วย Zig-friendly API
pub fn printLine(text: []const u8) void {
    _ = c.write(1, text.ptr, text.len);
    _ = c.write(1, "\n", 1);
}

// 3. Handle C errors อย่างถูกต้อง
pub fn openFile(path: []const u8) !*c.FILE {
    const file = c.fopen(path.ptr, "r") orelse
        return error.CannotOpenFile;
    return file;
}
```

> **ต่อไป**: Part 44 - C Interoperability ขั้นสูง (Real C libraries, SQLite wrapper)
