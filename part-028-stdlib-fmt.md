# ตอนที่ 28: std.fmt ใน Zig Standard Library

## Steps 271-280: การจัดรูปแบบข้อความด้วย std.fmt

---

## บทนำ

`std.fmt` เป็นโมดูลใน Zig Standard Library ที่ให้เครื่องมือสำหรับการจัดรูปแบบข้อความ (string formatting) อย่างครอบคลุม ต่างจากภาษาอื่นๆ เช่น C ที่ใช้ `printf` หรือ Python ที่ใช้ f-strings, Zig ออกแบบระบบ formatting ให้ปลอดภัยในระดับ compile-time ตรวจสอบ format string ได้ก่อนโปรแกรมรัน

ในบทนี้เราจะเรียนรู้:
- วิธีใช้ format string พื้นฐาน
- Format specifiers ต่างๆ และความหมาย
- การ format ลง buffer และใช้ allocator
- การแปลงสตริงเป็นตัวเลข
- การสร้าง custom formatter สำหรับ struct ของเราเอง
- เทคนิคขั้นสูงในการสร้างสตริงจากข้อมูลต่างๆ

---

## Step 271: std.fmt.format - Format String พื้นฐาน

### ทำความเข้าใจ std.fmt.format

ฟังก์ชัน `std.fmt.format` เป็นหัวใจหลักของระบบ formatting ใน Zig รับ writer, format string และ arguments แล้วเขียนข้อความที่จัดรูปแบบแล้วลงใน writer

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // รูปแบบพื้นฐาน: std.fmt.format(writer, format_string, args)
    try std.fmt.format(stdout, "สวัสดี, {}!\n", .{"โลก"});

    // ใช้ผ่าน print ซึ่งเรียก format ภายใน
    try stdout.print("ตัวเลข: {d}\n", .{42});
    try stdout.print("ทศนิยม: {d:.2}\n", .{3.14159});
    try stdout.print("ข้อความ: {s}\n", .{"Hello, Zig!"});
}
```

### โครงสร้าง Format String

Format string ใน Zig ใช้ `{}` เป็น placeholder สำหรับ argument แต่ละตัว:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // {} - ใช้ format default ของ type
    try stdout.print("{}\n", .{42});         // integer
    try stdout.print("{}\n", .{3.14});       // float
    try stdout.print("{}\n", .{true});       // boolean
    try stdout.print("{}\n", .{'A'});        // character

    // หลาย arguments
    try stdout.print("{} + {} = {}\n", .{10, 20, 30});

    // format แบบมี specifier
    try stdout.print("{d} in hex is {x}\n", .{255, 255});
}
```

### ความแตกต่างระหว่าง print และ format

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // stdout.print เป็น shorthand สำหรับ std.fmt.format(stdout, ...)
    try stdout.print("ด้วย print: {d}\n", .{100});

    // std.fmt.format ต้องระบุ writer เอง
    try std.fmt.format(stdout, "ด้วย format: {d}\n", .{100});

    // ทั้งสองให้ผลเหมือนกัน
    // stdout.print เป็นแค่ syntax sugar ที่ส่ง writer เข้าไปเอง
}
```

### Compile-time Safety

Zig ตรวจสอบ format string ในระดับ compile-time:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // ✓ ถูกต้อง: จำนวน placeholders ตรงกับ arguments
    try stdout.print("{d} + {d} = {d}\n", .{1, 2, 3});

    // ✗ จะ error ตอน compile: จำนวน arguments ไม่พอ
    // try stdout.print("{d} + {d}\n", .{1});

    // ✗ จะ error ตอน compile: type ไม่ตรงกับ specifier
    // try stdout.print("{d}\n", .{"text"});  // {d} ต้องการ number ไม่ใช่ string
}
```

---

## Step 272: Format Specifiers - {d}, {s}, {x}, {b}, {o}, {e}

### ตาราง Format Specifiers หลัก

| Specifier | ความหมาย | ตัวอย่าง Input | ตัวอย่าง Output |
|-----------|---------|--------------|----------------|
| `{d}` | Decimal integer | 255 | `255` |
| `{s}` | String (slice) | "hello" | `hello` |
| `{x}` | Hex lowercase | 255 | `ff` |
| `{X}` | Hex uppercase | 255 | `FF` |
| `{b}` | Binary | 10 | `1010` |
| `{o}` | Octal | 8 | `10` |
| `{e}` | Scientific notation | 12345.0 | `1.2345e+04` |
| `{f}` | Float fixed point | 3.14 | `3.14` |
| `{c}` | Character | 65 | `A` |
| `{}` | Default formatting | any | depends on type |
| `{any}` | Debug format | any | verbose debug |

### การใช้ {d} - Decimal

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // integer ทั่วไป
    try stdout.print("{d}\n", .{42});           // 42
    try stdout.print("{d}\n", .{-100});         // -100
    try stdout.print("{d}\n", .{@as(u8, 255)});  // 255

    // กำหนด width (จำนวนตัวอักษร)
    try stdout.print("{d:5}\n", .{42});         // "   42" (right-aligned)
    try stdout.print("{d:<5}\n", .{42});        // "42   " (left-aligned)
    try stdout.print("{d:^5}\n", .{42});        // " 42  " (center-aligned)

    // กำหนด fill character
    try stdout.print("{d:0>5}\n", .{42});       // "00042" (zero-padded)
    try stdout.print("{d:*<5}\n", .{42});       // "42***"
}
```

### การใช้ {s} - String

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const name = "Zig";

    // string ทั่วไป
    try stdout.print("{s}\n", .{name});          // Zig
    try stdout.print("Hello, {s}!\n", .{name}); // Hello, Zig!

    // กำหนด width
    try stdout.print("{s:10}\n", .{name});       // "Zig       "
    try stdout.print("{s:>10}\n", .{name});      // "       Zig"
    try stdout.print("{s:^10}\n", .{name});      // "   Zig    "

    // slice ของ bytes
    const bytes = [_]u8{ 72, 101, 108, 108, 111 };
    try stdout.print("{s}\n", .{bytes[0..]});    // Hello
}
```

### การใช้ {x} และ {X} - Hexadecimal

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const value: u32 = 0xDEADBEEF;

    // hex lowercase
    try stdout.print("{x}\n", .{value});        // deadbeef
    try stdout.print("0x{x}\n", .{value});      // 0xdeadbeef

    // hex uppercase
    try stdout.print("{X}\n", .{value});        // DEADBEEF
    try stdout.print("0x{X}\n", .{value});      // 0xDEADBEEF

    // กำหนด width และ zero-pad
    try stdout.print("{x:08}\n", .{255});       // 000000ff
    try stdout.print("{X:08}\n", .{255});       // 000000FF

    // สีพิเศษ: แสดงค่า memory address
    const ptr = @as(*const i32, @ptrFromInt(0x7FFF0000));
    try stdout.print("address: 0x{x}\n", .{@intFromPtr(ptr)});
}
```

### การใช้ {b} - Binary

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // binary representation
    try stdout.print("{b}\n", .{@as(u8, 0b10110011)});   // 10110011
    try stdout.print("{b}\n", .{42});                      // 101010
    try stdout.print("{b}\n", .{255});                     // 11111111

    // กำหนด width เพื่อแสดง leading zeros
    try stdout.print("{b:08}\n", .{@as(u8, 0b00001111)}); // 00001111

    // ใช้ในการแสดง flags/bitmask
    const flags: u8 = 0b10101010;
    try stdout.print("flags: {b:08}\n", .{flags});         // flags: 10101010

    // แสดงแต่ละ bit
    var i: u3 = 7;
    while (true) : (i -= 1) {
        const bit = (flags >> i) & 1;
        try stdout.print("bit[{d}] = {d}\n", .{i, bit});
        if (i == 0) break;
    }
}
```

### การใช้ {o} - Octal

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // octal representation
    try stdout.print("{o}\n", .{8});    // 10
    try stdout.print("{o}\n", .{64});   // 100
    try stdout.print("{o}\n", .{255});  // 377

    // Unix file permissions (ใช้ octal บ่อย)
    const permissions: u32 = 0o755;
    try stdout.print("permissions: {o}\n", .{permissions});  // permissions: 755
    try stdout.print("decimal: {d}\n", .{permissions});       // decimal: 493
}
```

### การใช้ {e} และ {f} - Floating Point

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const pi: f64 = 3.14159265358979;
    const big: f64 = 123456789.0;
    const small: f64 = 0.00001234;

    // scientific notation
    try stdout.print("{e}\n", .{pi});      // 3.14159265358979e+00
    try stdout.print("{e}\n", .{big});     // 1.23456789e+08
    try stdout.print("{e}\n", .{small});   // 1.234e-05

    // fixed point
    try stdout.print("{f}\n", .{pi});      // 3.14159265358979
    try stdout.print("{d:.2}\n", .{pi});   // 3.14 (2 decimal places)
    try stdout.print("{d:.4}\n", .{pi});   // 3.1416 (4 decimal places)

    // กำหนด precision กับ scientific
    try stdout.print("{e:.3}\n", .{pi});   // 3.142e+00

    // ค่าพิเศษ
    try stdout.print("{}\n", .{std.math.inf(f64)});   // inf
    try stdout.print("{}\n", .{-std.math.inf(f64)});  // -inf
    try stdout.print("{}\n", .{std.math.nan(f64)});   // nan
}
```

### การใช้ {c} - Character

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // แสดง character จาก integer value
    try stdout.print("{c}\n", .{65});    // A
    try stdout.print("{c}\n", .{97});    // a
    try stdout.print("{c}\n", .{48});    // 0

    // loop แสดง ASCII printable characters
    var c: u8 = 32; // space
    while (c < 127) : (c += 1) {
        try stdout.print("{c}", .{c});
    }
    try stdout.print("\n", .{});

    // แปลง char เป็น code point
    const letter = 'Z';
    try stdout.print("'{c}' = {d} = 0x{x}\n", .{letter, letter, letter});
}
```

### {} vs {any} - Debug Formatting

```zig
const std = @import("std");

const Point = struct {
    x: f32,
    y: f32,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const p = Point{ .x = 1.5, .y = 2.5 };
    const arr = [_]i32{ 1, 2, 3, 4, 5 };

    // {} - ใช้ default format
    try stdout.print("{}\n", .{42});         // 42
    try stdout.print("{}\n", .{p});          // main.Point{ .x = 1.5, .y = 2.5 }

    // {any} - แสดง type ด้วย (verbose)
    try stdout.print("{any}\n", .{42});      // 42 (เหมือนกันสำหรับ int)
    try stdout.print("{any}\n", .{p});       // main.Point{ .x = 1.5e0, .y = 2.5e0 }
    try stdout.print("{any}\n", .{arr});     // { 1, 2, 3, 4, 5 }
}
```

---

## Step 273: std.fmt.bufPrint - Format ลง Buffer

### แนวคิด bufPrint

`std.fmt.bufPrint` เขียนข้อความที่จัดรูปแบบแล้วลงใน buffer ที่กำหนด ไม่ต้องการ allocator เหมาะสำหรับกรณีที่รู้ขนาด output ล่วงหน้า

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [256]u8 = undefined;

    // bufPrint คืน slice ของ buffer ที่ใช้ไป
    const result = try std.fmt.bufPrint(&buf, "Hello, {s}!", .{"World"});
    std.debug.print("result: {s}\n", .{result});
    std.debug.print("length: {d}\n", .{result.len});
}
```

### การจัดการ Error: NoSpaceLeft

```zig
const std = @import("std");

pub fn main() void {
    var buf: [10]u8 = undefined;

    // จะ error ถ้า buffer เล็กเกินไป
    const result = std.fmt.bufPrint(&buf, "This is a very long string", .{});

    if (result) |formatted| {
        std.debug.print("OK: {s}\n", .{formatted});
    } else |err| {
        switch (err) {
            error.NoSpaceLeft => std.debug.print("Buffer เล็กเกินไป!\n", .{}),
        }
    }
}
```

### ตัวอย่างการใช้งานจริง: สร้าง Filename

```zig
const std = @import("std");

pub fn makeFilename(buf: []u8, prefix: []const u8, number: u32, ext: []const u8) ![]u8 {
    return std.fmt.bufPrint(buf, "{s}_{d:04}.{s}", .{prefix, number, ext});
}

pub fn main() !void {
    var buf: [64]u8 = undefined;

    const filename = try makeFilename(&buf, "image", 42, "png");
    std.debug.print("filename: {s}\n", .{filename});  // image_0042.png

    const log_file = try makeFilename(&buf, "log", 1, "txt");
    std.debug.print("log file: {s}\n", .{log_file});  // log_0001.txt
}
```

### ตัวอย่าง: สร้าง URL Path

```zig
const std = @import("std");

pub fn buildApiPath(buf: []u8, base: []const u8, version: u8, endpoint: []const u8, id: u64) ![]u8 {
    return std.fmt.bufPrint(buf, "{s}/v{d}/{s}/{d}", .{base, version, endpoint, id});
}

pub fn main() !void {
    var buf: [128]u8 = undefined;

    const path = try buildApiPath(&buf, "https://api.example.com", 2, "users", 12345);
    std.debug.print("API path: {s}\n", .{path});
    // Output: API path: https://api.example.com/v2/users/12345
}
```

### Stack Buffer Pattern

```zig
const std = @import("std");

// Pattern ที่นิยมใช้: buffer บน stack สำหรับ temporary strings
pub fn formatTemperature(celsius: f32) !void {
    var buf: [32]u8 = undefined;
    const fahrenheit = celsius * 9.0 / 5.0 + 32.0;

    const str = try std.fmt.bufPrint(&buf,
        "{d:.1}°C = {d:.1}°F",
        .{celsius, fahrenheit}
    );

    std.debug.print("{s}\n", .{str});
}

pub fn main() !void {
    try formatTemperature(0.0);    // 0.0°C = 32.0°F
    try formatTemperature(100.0);  // 100.0°C = 212.0°F
    try formatTemperature(37.0);   // 37.0°C = 98.6°F
    try formatTemperature(-40.0);  // -40.0°C = -40.0°F
}
```

### bufPrintZ - Version ที่เพิ่ม Null Terminator

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [64]u8 = undefined;

    // bufPrintZ เพิ่ม null terminator ให้อัตโนมัติ
    // เหมาะสำหรับส่งให้ C functions
    const result = try std.fmt.bufPrintZ(&buf, "Hello from Zig: {d}", .{42});

    std.debug.print("string: {s}\n", .{result});
    std.debug.print("sentinel: {d}\n", .{result[result.len]});  // 0 (null terminator)
}
```

---

## Step 274: std.fmt.allocPrint - Format ด้วย Allocator

### แนวคิด allocPrint

`std.fmt.allocPrint` จัดสรรหน่วยความจำที่ต้องการอัตโนมัติ ส่งคืน slice ที่ต้อง free หลังใช้งาน เหมาะสำหรับกรณีที่ไม่รู้ขนาด output ล่วงหน้า

```zig
const std = @import("std");

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    // allocPrint จัดสรร memory อัตโนมัติ
    const message = try std.fmt.allocPrint(allocator,
        "User {s} has {d} items",
        .{"Alice", 42}
    );
    defer allocator.free(message);  // ต้องอย่าลืม free!

    std.debug.print("{s}\n", .{message});
}
```

### การใช้กับ ArenaAllocator

```zig
const std = @import("std");

pub fn main() !void {
    // ArenaAllocator ทำให้ไม่ต้อง free แต่ละชิ้น
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();  // free ทั้งหมดในครั้งเดียว
    const allocator = arena.allocator();

    const strings = [_][]const u8{"Alice", "Bob", "Carol"};
    var messages = std.ArrayList([]const u8).init(allocator);

    for (strings, 0..) |name, i| {
        const msg = try std.fmt.allocPrint(allocator,
            "#{d}: {s}",
            .{i + 1, name}
        );
        try messages.append(msg);
        // ไม่ต้อง free แต่ละ msg เพราะ arena จัดการให้
    }

    for (messages.items) |msg| {
        std.debug.print("{s}\n", .{msg});
    }
    // Output:
    // #1: Alice
    // #2: Bob
    // #3: Carol
}
```

### สร้าง Dynamic Error Messages

```zig
const std = @import("std");

const DatabaseError = struct {
    code: i32,
    message: []const u8,
    allocator: std.mem.Allocator,

    pub fn init(alloc: std.mem.Allocator, code: i32, table: []const u8, operation: []const u8) !DatabaseError {
        const msg = try std.fmt.allocPrint(alloc,
            "Error {d}: Failed to {s} in table '{s}'",
            .{code, operation, table}
        );
        return .{
            .code = code,
            .message = msg,
            .allocator = alloc,
        };
    }

    pub fn deinit(self: DatabaseError) void {
        self.allocator.free(self.message);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const err = try DatabaseError.init(allocator, 1001, "users", "SELECT");
    defer err.deinit();

    std.debug.print("{s}\n", .{err.message});
    // Output: Error 1001: Failed to SELECT in table 'users'
}
```

### allocPrintZ - Version ที่เพิ่ม Null Terminator

```zig
const std = @import("std");

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    // allocPrintZ เหมาะสำหรับส่งให้ C functions ที่ต้องการ null-terminated string
    const path = try std.fmt.allocPrintZ(allocator,
        "/home/{s}/.config/{s}",
        .{"user", "app.conf"}
    );
    defer allocator.free(path);

    std.debug.print("path: {s}\n", .{path});
    // ใช้ path.ptr เพื่อส่งให้ C function
    // some_c_func(path.ptr);
}
```

### เปรียบเทียบ bufPrint vs allocPrint

```zig
const std = @import("std");

pub fn demonstrateDifference(name: []const u8, value: u32) !void {
    // bufPrint: ใช้ stack buffer, เร็วกว่า, ต้องรู้ขนาดล่วงหน้า
    var buf: [128]u8 = undefined;
    const stack_str = try std.fmt.bufPrint(&buf,
        "bufPrint: {s} = {d}",
        .{name, value}
    );
    std.debug.print("{s}\n", .{stack_str});
    // หมายเหตุ: stack_str valid อยู่แค่ในขอบเขตนี้เท่านั้น

    // allocPrint: ใช้ heap, ยืดหยุ่นกว่า, ต้อง free
    const allocator = std.heap.page_allocator;
    const heap_str = try std.fmt.allocPrint(allocator,
        "allocPrint: {s} = {d}",
        .{name, value}
    );
    defer allocator.free(heap_str);
    std.debug.print("{s}\n", .{heap_str});
    // heap_str สามารถส่งคืนออกไปนอก function ได้ (ถ้าไม่ defer free)
}

pub fn main() !void {
    try demonstrateDifference("temperature", 36);
}
```

---

## Step 275: std.fmt.parseInt และ parseFloat

### parseInt - แปลงสตริงเป็น Integer

```zig
const std = @import("std");

pub fn main() !void {
    // parseInt(T, string, base) -> T
    // base: ฐานของตัวเลข (0 = auto-detect)

    // ฐาน 10
    const n1 = try std.fmt.parseInt(i32, "42", 10);
    std.debug.print("parseInt(42) = {d}\n", .{n1});

    // ฐาน 16 (hex)
    const n2 = try std.fmt.parseInt(u32, "FF", 16);
    std.debug.print("parseInt(FF, 16) = {d}\n", .{n2});  // 255

    // ฐาน 2 (binary)
    const n3 = try std.fmt.parseInt(u8, "1010", 2);
    std.debug.print("parseInt(1010, 2) = {d}\n", .{n3});  // 10

    // ฐาน 8 (octal)
    const n4 = try std.fmt.parseInt(u16, "755", 8);
    std.debug.print("parseInt(755, 8) = {d}\n", .{n4});   // 493

    // ฐาน 0 = auto-detect จาก prefix
    const n5 = try std.fmt.parseInt(u32, "0xff", 0);  // hex
    const n6 = try std.fmt.parseInt(u32, "0b1010", 0); // binary
    const n7 = try std.fmt.parseInt(u32, "0o17", 0);   // octal
    std.debug.print("0xff = {d}\n", .{n5});   // 255
    std.debug.print("0b1010 = {d}\n", .{n6}); // 10
    std.debug.print("0o17 = {d}\n", .{n7});   // 15

    // ตัวเลขลบ
    const n8 = try std.fmt.parseInt(i32, "-123", 10);
    std.debug.print("parseInt(-123) = {d}\n", .{n8});
}
```

### การจัดการ Error ใน parseInt

```zig
const std = @import("std");

pub fn parseIntSafe(str: []const u8) void {
    const result = std.fmt.parseInt(i64, str, 10);
    if (result) |value| {
        std.debug.print("OK: {d}\n", .{value});
    } else |err| {
        switch (err) {
            error.InvalidCharacter => std.debug.print("Error: มีตัวอักษรที่ไม่ใช่ตัวเลขใน '{s}'\n", .{str}),
            error.Overflow => std.debug.print("Error: '{s}' เกิน range ของ i64\n", .{str}),
        }
    }
}

pub fn main() void {
    parseIntSafe("123");        // OK: 123
    parseIntSafe("-456");       // OK: -456
    parseIntSafe("12abc");      // Error: มีตัวอักษรที่ไม่ใช่ตัวเลขใน '12abc'
    parseIntSafe("99999999999999999999"); // Error: overflow
    parseIntSafe("");           // Error: InvalidCharacter (string ว่าง)
}
```

### parseFloat - แปลงสตริงเป็น Float

```zig
const std = @import("std");

pub fn main() !void {
    // parseFloat(T, string) -> T
    const f1 = try std.fmt.parseFloat(f64, "3.14");
    std.debug.print("parseFloat(3.14) = {d}\n", .{f1});

    const f2 = try std.fmt.parseFloat(f32, "2.718281828");
    std.debug.print("parseFloat(2.718) = {d:.6}\n", .{f2});

    // scientific notation
    const f3 = try std.fmt.parseFloat(f64, "1.5e10");
    std.debug.print("parseFloat(1.5e10) = {e}\n", .{f3});

    // ค่าลบ
    const f4 = try std.fmt.parseFloat(f64, "-0.001");
    std.debug.print("parseFloat(-0.001) = {d}\n", .{f4});

    // infinity และ nan
    const f5 = try std.fmt.parseFloat(f64, "inf");
    const f6 = try std.fmt.parseFloat(f64, "nan");
    std.debug.print("inf = {d}\n", .{f5});
    std.debug.print("nan = {d}\n", .{f6});
}
```

### Parser Input แบบมีหลาย Format

```zig
const std = @import("std");

// แปลง string เป็น number โดยรองรับหลาย format
pub fn parseNumber(str: []const u8) !f64 {
    // ลอง parse เป็น float ก่อน
    if (std.fmt.parseFloat(f64, str)) |value| {
        return value;
    } else |_| {}

    // ลอง parse เป็น integer ฐาน 10
    if (std.fmt.parseInt(i64, str, 10)) |value| {
        return @floatFromInt(value);
    } else |_| {}

    // ลอง parse เป็น hex
    if (std.mem.startsWith(u8, str, "0x")) {
        if (std.fmt.parseInt(u64, str[2..], 16)) |value| {
            return @floatFromInt(value);
        } else |_| {}
    }

    return error.InvalidNumber;
}

pub fn main() !void {
    const tests = [_][]const u8{
        "42",
        "3.14",
        "-100",
        "0xff",
        "1.5e3",
    };

    for (tests) |s| {
        if (parseNumber(s)) |n| {
            std.debug.print("'{s}' = {d}\n", .{s, n});
        } else |err| {
            std.debug.print("'{s}' -> error: {}\n", .{s, err});
        }
    }
}
```

### parseCharLiteral

```zig
const std = @import("std");

pub fn main() !void {
    // parseCharLiteral แปลง Zig character literal string
    const char1 = try std.fmt.parseCharLiteral("'a'");
    const char2 = try std.fmt.parseCharLiteral("'\\n'");
    const char3 = try std.fmt.parseCharLiteral("'\\x41'");

    std.debug.print("'a' = {d}\n", .{char1});   // 97
    std.debug.print("'\\n' = {d}\n", .{char2}); // 10
    std.debug.print("'\\x41' = {d}\n", .{char3}); // 65
}
```

---

## Step 276: Custom Formatters (format method ใน struct)

### การสร้าง Custom Formatter

Zig ให้สร้าง format method ใน struct เพื่อกำหนดวิธีแสดงผลข้อมูล เมื่อใช้ `{}` กับ struct นั้น

```zig
const std = @import("std");

const Color = struct {
    r: u8,
    g: u8,
    b: u8,

    // format method signature:
    // pub fn format(self, comptime fmt: []const u8, options: std.fmt.FormatOptions, writer: anytype) !void
    pub fn format(
        self: Color,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = options;  // ไม่ได้ใช้ options ในตัวอย่างนี้

        if (std.mem.eql(u8, fmt, "hex")) {
            // {hex} format: #RRGGBB
            try writer.print("#{x:0>2}{x:0>2}{x:0>2}", .{self.r, self.g, self.b});
        } else if (std.mem.eql(u8, fmt, "rgb")) {
            // {rgb} format: rgb(r, g, b)
            try writer.print("rgb({d}, {d}, {d})", .{self.r, self.g, self.b});
        } else {
            // default format: Color{ r=R, g=G, b=B }
            try writer.print("Color{{ r={d}, g={d}, b={d} }}", .{self.r, self.g, self.b});
        }
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    const red = Color{ .r = 255, .g = 0, .b = 0 };
    const purple = Color{ .r = 128, .g = 0, .b = 128 };

    try stdout.print("{}\n", .{red});       // Color{ r=255, g=0, b=0 }
    try stdout.print("{hex}\n", .{red});    // #ff0000
    try stdout.print("{rgb}\n", .{red});    // rgb(255, 0, 0)
    try stdout.print("{hex}\n", .{purple}); // #800080
}
```

### Custom Formatter ที่ใช้ FormatOptions

```zig
const std = @import("std");

const Fraction = struct {
    numerator: i64,
    denominator: i64,

    pub fn format(
        self: Fraction,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;

        // ทำ simplification ก่อน
        const g = gcd(@abs(self.numerator), @abs(self.denominator));
        const n = @divExact(self.numerator, @as(i64, @intCast(g)));
        const d = @divExact(self.denominator, @as(i64, @intCast(g)));

        if (d == 1) {
            try writer.print("{d}", .{n});
        } else if (d < 0) {
            try writer.print("{d}/{d}", .{-n, -d});
        } else {
            try writer.print("{d}/{d}", .{n, d});
        }
    }

    fn gcd(a: u64, b: u64) u64 {
        if (b == 0) return a;
        return gcd(b, a % b);
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const f1 = Fraction{ .numerator = 1, .denominator = 2 };
    const f2 = Fraction{ .numerator = 6, .denominator = 4 };
    const f3 = Fraction{ .numerator = 3, .denominator = 1 };
    const f4 = Fraction{ .numerator = -5, .denominator = 10 };

    try stdout.print("{}\n", .{f1});  // 1/2
    try stdout.print("{}\n", .{f2});  // 3/2
    try stdout.print("{}\n", .{f3});  // 3
    try stdout.print("{}\n", .{f4});  // -1/2
}
```

### Custom Formatter สำหรับ Enum

```zig
const std = @import("std");

const Direction = enum {
    north,
    south,
    east,
    west,

    pub fn format(
        self: Direction,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = options;

        if (std.mem.eql(u8, fmt, "arrow")) {
            const arrow = switch (self) {
                .north => "↑",
                .south => "↓",
                .east => "→",
                .west => "←",
            };
            try writer.writeAll(arrow);
        } else if (std.mem.eql(u8, fmt, "thai")) {
            const thai = switch (self) {
                .north => "เหนือ",
                .south => "ใต้",
                .east => "ตะวันออก",
                .west => "ตะวันตก",
            };
            try writer.writeAll(thai);
        } else {
            try writer.writeAll(@tagName(self));
        }
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    const dir = Direction.north;

    try stdout.print("{}\n", .{dir});        // north
    try stdout.print("{arrow}\n", .{dir});   // ↑
    try stdout.print("{thai}\n", .{dir});    // เหนือ

    // วน loop แสดงทุกทิศ
    const directions = [_]Direction{ .north, .south, .east, .west };
    for (directions) |d| {
        try stdout.print("{arrow} = {thai}\n", .{d, d});
    }
}
```

### Custom Formatter ที่ซับซ้อน: Table Row

```zig
const std = @import("std");

const TableRow = struct {
    name: []const u8,
    age: u32,
    score: f32,
    col_width: u32 = 15,

    pub fn format(
        self: TableRow,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = options;

        if (std.mem.eql(u8, fmt, "header")) {
            try writer.print("| {s:<15} | {s:<5} | {s:<8} |",
                .{"Name", "Age", "Score"}
            );
        } else if (std.mem.eql(u8, fmt, "sep")) {
            try writer.print("|{s:-<17}|{s:-<7}|{s:-<10}|",
                .{"", "", ""}
            );
        } else {
            try writer.print("| {s:<15} | {d:<5} | {d:<8.2} |",
                .{self.name, self.age, self.score}
            );
        }
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const rows = [_]TableRow{
        .{ .name = "Alice", .age = 30, .score = 95.5 },
        .{ .name = "Bob", .age = 25, .score = 87.3 },
        .{ .name = "Carol", .age = 35, .score = 92.1 },
    };

    try stdout.print("{header}\n", .{rows[0]});
    try stdout.print("{sep}\n", .{rows[0]});
    for (rows) |row| {
        try stdout.print("{}\n", .{row});
    }
}
```

---

## Step 277: std.fmt.formatIntBuf และ formatFloatBuf

### formatIntBuf - Format Integer ลง Buffer โดยตรง

```zig
const std = @import("std");

pub fn main() void {
    var buf: [32]u8 = undefined;

    // formatIntBuf(out, value, base, case, options) -> []u8
    const result = std.fmt.formatIntBuf(
        &buf,
        @as(u64, 255),
        16,   // base hex
        .upper, // uppercase
        .{}   // default options
    );

    std.debug.print("formatIntBuf: {s}\n", .{result}); // FF
    std.debug.print("length: {d}\n", .{result.len});    // 2
}
```

### ใช้ formatInt กับ Writer

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // formatInt เขียนลง writer โดยตรง
    try std.fmt.formatInt(
        @as(i64, -12345),
        10,      // base decimal
        .lower,  // lowercase (สำหรับ hex)
        .{ .width = 10, .fill = '0' }, // options
        stdout
    );
    try stdout.writeByte('\n');

    // hex formatting
    try std.fmt.formatInt(
        @as(u32, 0xABCD),
        16,
        .upper,
        .{ .width = 8, .fill = '0' },
        stdout
    );
    try stdout.writeByte('\n');  // 0000ABCD
}
```

### formatFloatBuf - Format Float ลง Buffer

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [64]u8 = undefined;

    // format float ด้วย fixed decimal places
    const pi = 3.14159265358979;
    const result = try std.fmt.formatFloat(
        buf[0..],
        pi,
        .{ .mode = .decimal, .precision = 4 }
    );

    std.debug.print("pi = {s}\n", .{result}); // pi = 3.1416

    // scientific notation
    const big_num: f64 = 6.022e23;
    const result2 = try std.fmt.formatFloat(
        buf[0..],
        big_num,
        .{ .mode = .scientific, .precision = 3 }
    );
    std.debug.print("avogadro = {s}\n", .{result2}); // avogadro = 6.022e+23
}
```

### FormatOptions ในรายละเอียด

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // FormatOptions struct:
    // .fill      : character ที่ใช้เติม (default ' ')
    // .alignment : .left, .right, .center (default .right)
    // .width     : ความกว้างขั้นต่ำ
    // .precision : จำนวนทศนิยม (สำหรับ float)

    // Right-aligned, width 10, fill with spaces
    try std.fmt.formatInt(42, 10, .lower,
        .{ .width = 10, .fill = ' ', .alignment = .right },
        stdout
    );
    try stdout.writeAll(" <-- right\n");

    // Left-aligned, width 10
    try std.fmt.formatInt(42, 10, .lower,
        .{ .width = 10, .fill = ' ', .alignment = .left },
        stdout
    );
    try stdout.writeAll(" <-- left\n");

    // Center-aligned, width 10
    try std.fmt.formatInt(42, 10, .lower,
        .{ .width = 10, .fill = ' ', .alignment = .center },
        stdout
    );
    try stdout.writeAll(" <-- center\n");

    // Zero-padded
    try std.fmt.formatInt(42, 10, .lower,
        .{ .width = 10, .fill = '0' },
        stdout
    );
    try stdout.writeAll(" <-- zero-padded\n");
}
```

---

## Step 278: ชนิดข้อมูล Format Specifiers อย่างละเอียด

### Integer Types

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // unsigned integers
    const u8_val: u8 = 200;
    const u16_val: u16 = 60000;
    const u32_val: u32 = 4000000000;
    const u64_val: u64 = 18446744073709551615;

    try stdout.print("u8:  {d}\n", .{u8_val});
    try stdout.print("u16: {d}\n", .{u16_val});
    try stdout.print("u32: {d}\n", .{u32_val});
    try stdout.print("u64: {d}\n", .{u64_val});

    // signed integers
    const i8_val: i8 = -100;
    const i16_val: i16 = -30000;
    const i32_val: i32 = -2000000000;
    const i64_val: i64 = -9223372036854775808;

    try stdout.print("i8:  {d}\n", .{i8_val});
    try stdout.print("i16: {d}\n", .{i16_val});
    try stdout.print("i32: {d}\n", .{i32_val});
    try stdout.print("i64: {d}\n", .{i64_val});

    // comptime_int
    const ct_int = 1234567890;  // type: comptime_int
    try stdout.print("comptime_int: {d}\n", .{ct_int});

    // usize, isize
    const us: usize = 1024;
    const is: isize = -512;
    try stdout.print("usize: {d}\n", .{us});
    try stdout.print("isize: {d}\n", .{is});
}
```

### Float Types

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const f16_val: f16 = 3.14;
    const f32_val: f32 = 2.71828;
    const f64_val: f64 = 1.41421356237;
    const f80_val: f80 = 1.73205080757;
    const f128_val: f128 = 2.30258509299;

    // {d} สำหรับ float = fixed notation
    try stdout.print("f16:  {d:.4}\n", .{f16_val});
    try stdout.print("f32:  {d:.6}\n", .{f32_val});
    try stdout.print("f64:  {d:.10}\n", .{f64_val});
    try stdout.print("f80:  {d:.10}\n", .{f80_val});
    try stdout.print("f128: {d:.10}\n", .{f128_val});

    // {e} สำหรับ scientific notation
    const small: f64 = 0.000001234;
    const large: f64 = 123456789.0;
    try stdout.print("small: {e}\n", .{small});
    try stdout.print("large: {e}\n", .{large});

    // comptime_float
    const ct_float = 3.14;  // type: comptime_float
    try stdout.print("comptime_float: {d}\n", .{ct_float});
}
```

### Boolean Type

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const t = true;
    const f = false;

    // {} สำหรับ bool = "true" หรือ "false"
    try stdout.print("{}\n", .{t});   // true
    try stdout.print("{}\n", .{f});   // false

    // ใช้ใน conditional
    const value = 42;
    try stdout.print("value > 0: {}\n", .{value > 0});  // true
    try stdout.print("value < 0: {}\n", .{value < 0});  // false
}
```

### Optional Types

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const some: ?i32 = 42;
    const none: ?i32 = null;

    // {} สำหรับ optional
    try stdout.print("{}\n", .{some}); // 42
    try stdout.print("{}\n", .{none}); // null

    // optional string
    const opt_str: ?[]const u8 = "hello";
    const nil_str: ?[]const u8 = null;

    try stdout.print("{?s}\n", .{opt_str}); // hello
    try stdout.print("{?s}\n", .{nil_str}); // null
}
```

### Error Union Types

```zig
const std = @import("std");

pub fn riskyOperation(x: i32) !i32 {
    if (x < 0) return error.NegativeInput;
    return x * 2;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // แสดง error type
    const result1 = riskyOperation(5);
    const result2 = riskyOperation(-1);

    // ต้อง unwrap ก่อนแสดง
    if (result1) |val| {
        try stdout.print("success: {d}\n", .{val});
    } else |err| {
        try stdout.print("error: {}\n", .{err});
    }

    if (result2) |val| {
        try stdout.print("success: {d}\n", .{val});
    } else |err| {
        try stdout.print("error: {}\n", .{err});  // error: error.NegativeInput
    }
}
```

### Pointer Types

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    var x: i32 = 42;
    const ptr = &x;

    // แสดง pointer address
    try stdout.print("address: {*}\n", .{ptr});     // address: ...hex address...
    try stdout.print("address: 0x{x}\n", .{@intFromPtr(ptr)});

    // แสดงค่าที่ pointer ชี้
    try stdout.print("value: {d}\n", .{ptr.*});     // value: 42

    // array pointer
    const arr = [_]i32{1, 2, 3};
    const arr_ptr = &arr;
    try stdout.print("array: {any}\n", .{arr_ptr.*}); // array: { 1, 2, 3 }
}
```

### Struct และ Union Types

```zig
const std = @import("std");

const Point3D = struct {
    x: f32,
    y: f32,
    z: f32,
};

const Shape = union(enum) {
    circle: f32,
    rectangle: struct { w: f32, h: f32 },
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const p = Point3D{ .x = 1.0, .y = 2.0, .z = 3.0 };
    // {} ใช้ default struct formatting
    try stdout.print("{}\n", .{p});
    // Output: main.Point3D{ .x = 1.0e0, .y = 2.0e0, .z = 3.0e0 }

    const s1 = Shape{ .circle = 5.0 };
    const s2 = Shape{ .rectangle = .{ .w = 3.0, .h = 4.0 } };
    try stdout.print("{}\n", .{s1});
    try stdout.print("{}\n", .{s2});
}
```

### Slice Types

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // byte slice - ใช้ {s} สำหรับ string
    const text: []const u8 = "Hello, Zig!";
    try stdout.print("{s}\n", .{text});

    // integer slice - ใช้ {} หรือ {any}
    const numbers = [_]i32{10, 20, 30, 40, 50};
    try stdout.print("{any}\n", .{numbers[0..]});
    // Output: { 10, 20, 30, 40, 50 }

    // แสดงขนาด slice
    try stdout.print("len={d}, ptr=0x{x}\n",
        .{text.len, @intFromPtr(text.ptr)});
}
```

---

## Step 279: Writer และ print

### ทำความเข้าใจ Writer Interface

Writer ใน Zig เป็น interface ที่ใช้กับ `std.fmt` เพื่อเขียนข้อมูล ทุก type ที่มี `write` method สามารถเป็น writer ได้

```zig
const std = @import("std");

pub fn main() !void {
    // stdout writer
    const stdout = std.io.getStdOut().writer();
    try stdout.print("stdout: {s}\n", .{"Hello"});

    // stderr writer
    const stderr = std.io.getStdErr().writer();
    try stderr.print("stderr: {s}\n", .{"Error!"});

    // file writer
    const file = try std.fs.cwd().createFile("output.txt", .{});
    defer file.close();
    const file_writer = file.writer();
    try file_writer.print("file: {d}\n", .{42});

    // ArrayList writer
    var buf = std.ArrayList(u8).init(std.heap.page_allocator);
    defer buf.deinit();
    const buf_writer = buf.writer();
    try buf_writer.print("buffered: {s}\n", .{"data"});
    std.debug.print("buffer contents: {s}", .{buf.items});
}
```

### Writer Methods

```zig
const std = @import("std");

pub fn main() !void {
    var buf = std.ArrayList(u8).init(std.heap.page_allocator);
    defer buf.deinit();
    const w = buf.writer();

    // writeByte - เขียน 1 byte
    try w.writeByte('H');
    try w.writeByte('i');
    try w.writeByte('\n');

    // writeAll - เขียน slice ทั้งหมด
    try w.writeAll("Hello, ");
    try w.writeAll("World!\n");

    // writeBytesNTimes - เขียน byte N ครั้ง
    try w.writeBytesNTimes("--", 5);
    try w.writeByte('\n');

    // writeInt - เขียน integer ในรูป binary
    try w.writeInt(u32, 12345, .big);  // big-endian

    // print - format แล้วเขียน
    try w.print("value: {d}\n", .{100});

    std.debug.print("{s}", .{buf.items});
}
```

### สร้าง Custom Writer

```zig
const std = @import("std");

// Custom writer ที่นับ bytes ที่เขียน
const CountingWriter = struct {
    child_writer: std.io.AnyWriter,
    count: usize = 0,

    const Self = @This();
    const Error = std.io.AnyWriter.Error;

    pub fn write(self: *Self, bytes: []const u8) Error!usize {
        const written = try self.child_writer.write(bytes);
        self.count += written;
        return written;
    }

    pub fn writer(self: *Self) std.io.Writer(*Self, Error, write) {
        return .{ .context = self };
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    var counting = CountingWriter{ .child_writer = stdout.any() };
    const w = counting.writer();

    try w.print("Hello, {s}!\n", .{"World"});
    try w.print("Count: {d}\n", .{42});

    std.debug.print("Total bytes written: {d}\n", .{counting.count});
}
```

### BufferedWriter สำหรับ Performance

```zig
const std = @import("std");

pub fn main() !void {
    // BufferedWriter รวม writes เพื่อ performance ที่ดีกว่า
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const w = bw.writer();

    // เขียนหลายครั้งแต่จะถูก buffer ไว้
    var i: u32 = 0;
    while (i < 100) : (i += 1) {
        try w.print("line {d}\n", .{i});
    }

    // flush ออกไปยัง stdout จริงๆ
    try bw.flush();
}
```

### MultiWriter - เขียนหลาย destination พร้อมกัน

```zig
const std = @import("std");

pub fn main() !void {
    // เขียนทั้ง stdout และ file พร้อมกัน
    const stdout = std.io.getStdOut().writer();

    const file = try std.fs.cwd().createFile("log.txt", .{});
    defer file.close();
    const file_writer = file.writer();

    // สร้าง multi-writer
    var multi = std.io.MultiWriter(.{}).init(.{stdout, file_writer});
    const mw = multi.writer();

    try mw.print("This goes to both stdout and file\n", .{});
    try mw.print("Timestamp: {d}\n", .{std.time.timestamp()});
}
```

### การใช้ print ใน Debug Mode

```zig
const std = @import("std");

pub fn main() void {
    // std.debug.print - เขียนไปยัง stderr โดยตรง (ไม่ต้อง try)
    std.debug.print("debug: {d}\n", .{42});
    std.debug.print("debug: {s}\n", .{"hello"});

    // เหมาะสำหรับ debugging เพราะ:
    // 1. ไม่ต้องจัดการ error
    // 2. เขียนไป stderr (ไม่ปนกับ stdout)
    // 3. thread-safe

    // สำหรับ production code ให้ใช้ stdout.print แทน
}
```

---

## Step 280: std.fmt ใน Practice - Building Strings

### Pattern 1: String Builder ด้วย ArrayList

```zig
const std = @import("std");

pub const StringBuilder = struct {
    list: std.ArrayList(u8),

    pub fn init(allocator: std.mem.Allocator) StringBuilder {
        return .{ .list = std.ArrayList(u8).init(allocator) };
    }

    pub fn deinit(self: *StringBuilder) void {
        self.list.deinit();
    }

    pub fn append(self: *StringBuilder, str: []const u8) !void {
        try self.list.appendSlice(str);
    }

    pub fn appendFmt(self: *StringBuilder, comptime fmt: []const u8, args: anytype) !void {
        try self.list.writer().print(fmt, args);
    }

    pub fn toString(self: *const StringBuilder) []const u8 {
        return self.list.items;
    }

    pub fn toOwnedSlice(self: *StringBuilder) ![]u8 {
        return self.list.toOwnedSlice();
    }

    pub fn len(self: *const StringBuilder) usize {
        return self.list.items.len;
    }

    pub fn clear(self: *StringBuilder) void {
        self.list.clearRetainingCapacity();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var sb = StringBuilder.init(allocator);
    defer sb.deinit();

    try sb.append("สวัสดี");
    try sb.appendFmt(", {s}!", .{"โลก"});
    try sb.appendFmt(" คุณอายุ {d} ปี", .{25});

    std.debug.print("{s}\n", .{sb.toString()});
    std.debug.print("length: {d}\n", .{sb.len()});
}
```

### Pattern 2: สร้าง JSON String

```zig
const std = @import("std");

pub fn buildJson(
    allocator: std.mem.Allocator,
    name: []const u8,
    age: u32,
    active: bool,
    score: f32,
) ![]u8 {
    return std.fmt.allocPrint(allocator,
        \\{{
        \\  "name": "{s}",
        \\  "age": {d},
        \\  "active": {s},
        \\  "score": {d:.2}
        \\}}
    ,
        .{
            name,
            age,
            if (active) "true" else "false",
            score,
        }
    );
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json = try buildJson(allocator, "Alice", 30, true, 98.5);
    defer allocator.free(json);

    std.debug.print("{s}\n", .{json});
}
```

### Pattern 3: CSV Generator

```zig
const std = @import("std");

pub const CsvWriter = struct {
    writer: std.io.AnyWriter,
    first_in_row: bool = true,

    pub fn writeField(self: *CsvWriter, field: []const u8) !void {
        if (!self.first_in_row) {
            try self.writer.writeByte(',');
        }
        self.first_in_row = false;

        // Quote fields containing comma or newline
        const needs_quote = std.mem.indexOfAny(u8, field, ",\"\n") != null;
        if (needs_quote) {
            try self.writer.writeByte('"');
            for (field) |c| {
                if (c == '"') try self.writer.writeByte('"');
                try self.writer.writeByte(c);
            }
            try self.writer.writeByte('"');
        } else {
            try self.writer.writeAll(field);
        }
    }

    pub fn writeIntField(self: *CsvWriter, value: anytype) !void {
        if (!self.first_in_row) {
            try self.writer.writeByte(',');
        }
        self.first_in_row = false;
        try self.writer.print("{d}", .{value});
    }

    pub fn writeFloatField(self: *CsvWriter, value: f64, decimals: u8) !void {
        if (!self.first_in_row) {
            try self.writer.writeByte(',');
        }
        self.first_in_row = false;
        // ใช้ bufPrint เพื่อ format float ด้วย dynamic precision
        var buf: [32]u8 = undefined;
        _ = decimals;
        const str = try std.fmt.bufPrint(&buf, "{d:.2}", .{value});
        try self.writer.writeAll(str);
    }

    pub fn newRow(self: *CsvWriter) !void {
        try self.writer.writeByte('\n');
        self.first_in_row = true;
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    var csv = CsvWriter{ .writer = stdout.any() };

    // Header row
    try csv.writeField("Name");
    try csv.writeField("Age");
    try csv.writeField("Score");
    try csv.writeField("Active");
    try csv.newRow();

    // Data rows
    const data = [_]struct {
        name: []const u8,
        age: u32,
        score: f64,
        active: bool,
    }{
        .{ .name = "Alice", .age = 30, .score = 95.5, .active = true },
        .{ .name = "Bob, Jr.", .age = 25, .score = 87.3, .active = false },
        .{ .name = "Carol", .age = 35, .score = 92.1, .active = true },
    };

    for (data) |row| {
        try csv.writeField(row.name);
        try csv.writeIntField(row.age);
        try csv.writeFloatField(row.score, 2);
        try csv.writeField(if (row.active) "yes" else "no");
        try csv.newRow();
    }
}
```

### Pattern 4: Log Formatter

```zig
const std = @import("std");

pub const LogLevel = enum {
    debug,
    info,
    warning,
    @"error",
    fatal,

    pub fn toString(self: LogLevel) []const u8 {
        return switch (self) {
            .debug => "DEBUG",
            .info => "INFO ",
            .warning => "WARN ",
            .@"error" => "ERROR",
            .fatal => "FATAL",
        };
    }
};

pub const Logger = struct {
    writer: std.io.AnyWriter,
    min_level: LogLevel = .debug,

    pub fn log(
        self: Logger,
        level: LogLevel,
        comptime fmt: []const u8,
        args: anytype,
    ) void {
        if (@intFromEnum(level) < @intFromEnum(self.min_level)) return;

        const timestamp = std.time.timestamp();
        self.writer.print("[{d}] [{s}] ", .{timestamp, level.toString()}) catch return;
        self.writer.print(fmt, args) catch return;
        self.writer.writeByte('\n') catch return;
    }

    pub fn debug(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.debug, fmt, args);
    }

    pub fn info(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.info, fmt, args);
    }

    pub fn warn(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.warning, fmt, args);
    }

    pub fn err(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.@"error", fmt, args);
    }
};

pub fn main() !void {
    const stderr = std.io.getStdErr().writer();
    const logger = Logger{
        .writer = stderr.any(),
        .min_level = .debug,
    };

    logger.debug("Starting application...", .{});
    logger.info("Server listening on port {d}", .{8080});
    logger.warn("Memory usage: {d}%", .{85});
    logger.err("Failed to connect to database: {s}", .{"connection refused"});
}
```

### Pattern 5: Template Engine แบบง่าย

```zig
const std = @import("std");

// แทนที่ {{key}} ใน template ด้วยค่าจาก map
pub fn renderTemplate(
    allocator: std.mem.Allocator,
    template: []const u8,
    vars: std.StringHashMap([]const u8),
) ![]u8 {
    var result = std.ArrayList(u8).init(allocator);
    errdefer result.deinit();

    var i: usize = 0;
    while (i < template.len) {
        if (i + 1 < template.len and template[i] == '{' and template[i + 1] == '{') {
            // หา }}
            const end = std.mem.indexOf(u8, template[i + 2 ..], "}}");
            if (end) |e| {
                const key = template[i + 2 .. i + 2 + e];
                if (vars.get(key)) |value| {
                    try result.appendSlice(value);
                } else {
                    // key ไม่พบ เก็บ original
                    try result.appendSlice(template[i .. i + 2 + e + 2]);
                }
                i = i + 2 + e + 2;
                continue;
            }
        }
        try result.append(template[i]);
        i += 1;
    }

    return result.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var vars = std.StringHashMap([]const u8).init(allocator);
    defer vars.deinit();

    try vars.put("name", "Alice");
    try vars.put("language", "Zig");
    try vars.put("version", "0.13.0");

    const template = "สวัสดี {{name}}! ยินดีต้อนรับสู่ {{language}} {{version}}";
    const result = try renderTemplate(allocator, template, vars);
    defer allocator.free(result);

    std.debug.print("{s}\n", .{result});
    // Output: สวัสดี Alice! ยินดีต้อนรับสู่ Zig 0.13.0
}
```

### Pattern 6: Number Formatting Utilities

```zig
const std = @import("std");

// จัดรูปแบบตัวเลขด้วย comma separator
pub fn formatWithCommas(buf: []u8, n: u64) ![]u8 {
    var tmp: [32]u8 = undefined;
    const num_str = try std.fmt.bufPrint(&tmp, "{d}", .{n});

    var result_pos: usize = 0;
    const len = num_str.len;

    for (num_str, 0..) |c, i| {
        const remaining = len - i;
        if (remaining % 3 == 0 and i != 0) {
            buf[result_pos] = ',';
            result_pos += 1;
        }
        buf[result_pos] = c;
        result_pos += 1;
    }

    return buf[0..result_pos];
}

// จัดรูปแบบ bytes เป็น human-readable
pub fn formatBytes(buf: []u8, bytes: u64) ![]u8 {
    const units = [_][]const u8{ "B", "KB", "MB", "GB", "TB", "PB" };
    var value: f64 = @floatFromInt(bytes);
    var unit_idx: usize = 0;

    while (value >= 1024.0 and unit_idx < units.len - 1) {
        value /= 1024.0;
        unit_idx += 1;
    }

    if (unit_idx == 0) {
        return std.fmt.bufPrint(buf, "{d} {s}", .{bytes, units[0]});
    } else {
        return std.fmt.bufPrint(buf, "{d:.2} {s}", .{value, units[unit_idx]});
    }
}

// จัดรูปแบบ duration
pub fn formatDuration(buf: []u8, nanoseconds: u64) ![]u8 {
    if (nanoseconds < 1000) {
        return std.fmt.bufPrint(buf, "{d}ns", .{nanoseconds});
    } else if (nanoseconds < 1_000_000) {
        return std.fmt.bufPrint(buf, "{d:.2}µs", .{@as(f64, @floatFromInt(nanoseconds)) / 1000.0});
    } else if (nanoseconds < 1_000_000_000) {
        return std.fmt.bufPrint(buf, "{d:.2}ms", .{@as(f64, @floatFromInt(nanoseconds)) / 1_000_000.0});
    } else {
        return std.fmt.bufPrint(buf, "{d:.2}s", .{@as(f64, @floatFromInt(nanoseconds)) / 1_000_000_000.0});
    }
}

pub fn main() !void {
    var buf: [64]u8 = undefined;

    // จัดรูปแบบตัวเลขด้วย comma
    const n1 = try formatWithCommas(&buf, 1234567890);
    std.debug.print("1234567890 = {s}\n", .{n1});  // 1,234,567,890

    // จัดรูปแบบ bytes
    const sizes = [_]u64{ 512, 1024, 1048576, 1073741824, 1099511627776 };
    for (sizes) |size| {
        const formatted = try formatBytes(&buf, size);
        std.debug.print("{d} bytes = {s}\n", .{size, formatted});
    }

    // จัดรูปแบบ duration
    const durations = [_]u64{ 500, 1500, 1_500_000, 2_500_000_000 };
    for (durations) |dur| {
        const formatted = try formatDuration(&buf, dur);
        std.debug.print("{d}ns = {s}\n", .{dur, formatted});
    }
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: ระดับง่าย

**โจทย์**: เขียนฟังก์ชัน `formatBitMask` ที่รับค่า `u16` และแสดงผลเป็น bit pattern แบบมี separator ทุก 4 bits

ตัวอย่าง: `0xABCD` → `1010 1011 1100 1101`

```zig
const std = @import("std");

pub fn formatBitMask(buf: []u8, value: u16) ![]u8 {
    // TODO: implement this
    // Hint: ใช้ loop ตรวจสอบทีละ bit ด้วย bit shifting
    _ = buf;
    _ = value;
    return error.NotImplemented;
}

pub fn main() !void {
    var buf: [32]u8 = undefined;
    const result = try formatBitMask(&buf, 0xABCD);
    std.debug.print("{s}\n", .{result});
    // Expected: 1010 1011 1100 1101
}
```

**เฉลย**:
```zig
pub fn formatBitMask(buf: []u8, value: u16) ![]u8 {
    var pos: usize = 0;
    var i: u4 = 15;
    while (true) : (i -= 1) {
        const bit = (value >> i) & 1;
        buf[pos] = if (bit == 1) '1' else '0';
        pos += 1;
        if (i % 4 == 0 and i != 0) {
            buf[pos] = ' ';
            pos += 1;
        }
        if (i == 0) break;
    }
    return buf[0..pos];
}
```

### แบบฝึกหัดที่ 2: ระดับกลาง

**โจทย์**: สร้าง struct `Matrix2x2` ที่มี custom formatter แสดงผลเป็น matrix format:

```
[ 1.00  2.00 ]
[ 3.00  4.00 ]
```

```zig
const std = @import("std");

const Matrix2x2 = struct {
    data: [2][2]f32,

    pub fn format(
        self: Matrix2x2,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        // TODO: implement this
        _ = self;
        _ = fmt;
        _ = options;
        _ = writer;
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    const m = Matrix2x2{ .data = .{
        .{ 1.0, 2.0 },
        .{ 3.0, 4.0 },
    }};
    try stdout.print("{}\n", .{m});
}
```

### แบบฝึกหัดที่ 3: ระดับยาก

**โจทย์**: สร้าง `ProgressBar` struct ที่แสดงความก้าวหน้าในรูปแบบ:

```
[████████████░░░░░░░░] 60% (6/10)
```

ต้องมี:
- custom formatter
- รองรับ width ที่กำหนดได้
- แสดง percentage และ fraction

```zig
const std = @import("std");

const ProgressBar = struct {
    current: usize,
    total: usize,
    width: usize = 20,

    pub fn format(
        self: ProgressBar,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        // TODO: implement this
        _ = self;
        _ = fmt;
        _ = options;
        _ = writer;
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    var i: usize = 0;
    while (i <= 10) : (i += 1) {
        const bar = ProgressBar{ .current = i, .total = 10, .width = 20 };
        try stdout.print("\r{}", .{bar});
    }
    try stdout.writeByte('\n');
}
```

**เฉลย**:
```zig
pub fn format(
    self: ProgressBar,
    comptime fmt: []const u8,
    options: std.fmt.FormatOptions,
    writer: anytype,
) !void {
    _ = fmt;
    _ = options;

    const percent = if (self.total == 0) 0 else (self.current * 100) / self.total;
    const filled = if (self.total == 0) 0 else (self.current * self.width) / self.total;

    try writer.writeByte('[');
    var i: usize = 0;
    while (i < self.width) : (i += 1) {
        if (i < filled) {
            try writer.writeAll("█");
        } else {
            try writer.writeAll("░");
        }
    }
    try writer.print("] {d}% ({d}/{d})", .{percent, self.current, self.total});
}
```

### แบบฝึกหัดที่ 4: Real-world Application

**โจทย์**: สร้าง HTTP Request formatter ที่แสดงผล request ในรูปแบบมาตรฐาน:

```
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json
Content-Type: application/json

{"id": 42}
```

```zig
const std = @import("std");

pub const HttpMethod = enum { GET, POST, PUT, DELETE, PATCH };

pub const HttpRequest = struct {
    method: HttpMethod,
    path: []const u8,
    version: []const u8 = "HTTP/1.1",
    headers: []const struct { name: []const u8, value: []const u8 },
    body: ?[]const u8 = null,

    pub fn format(
        self: HttpRequest,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;

        // Request line
        try writer.print("{s} {s} {s}\r\n",
            .{@tagName(self.method), self.path, self.version}
        );

        // Headers
        for (self.headers) |header| {
            try writer.print("{s}: {s}\r\n", .{header.name, header.value});
        }

        // Empty line
        try writer.writeAll("\r\n");

        // Body (ถ้ามี)
        if (self.body) |body| {
            try writer.writeAll(body);
        }
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const req = HttpRequest{
        .method = .POST,
        .path = "/api/users",
        .headers = &.{
            .{ .name = "Host", .value = "example.com" },
            .{ .name = "Accept", .value = "application/json" },
            .{ .name = "Content-Type", .value = "application/json" },
        },
        .body = "{\"name\": \"Alice\", \"age\": 30}",
    };

    try stdout.print("{}\n", .{req});
}
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้เกี่ยวกับ `std.fmt` ซึ่งเป็นระบบ formatting ที่ทรงพลังของ Zig:

### สิ่งที่เรียนรู้

| Topic | Functions/Concepts |
|-------|-------------------|
| **Basic Formatting** | `std.fmt.format`, `writer.print`, `std.debug.print` |
| **Format Specifiers** | `{d}`, `{s}`, `{x}`, `{b}`, `{o}`, `{e}`, `{c}`, `{}`, `{any}` |
| **Buffer Operations** | `bufPrint`, `bufPrintZ` |
| **Heap Operations** | `allocPrint`, `allocPrintZ` |
| **Parsing** | `parseInt`, `parseFloat`, `parseCharLiteral` |
| **Custom Formatters** | `format(self, comptime fmt, options, writer)` method |
| **Low-level** | `formatIntBuf`, `formatInt`, `formatFloat` |
| **Type Specifiers** | integer, float, bool, optional, pointer, struct, slice |
| **Writer Interface** | stdout, stderr, file, ArrayList, BufferedWriter |
| **Practical Patterns** | StringBuilder, JSON builder, CSV writer, logger |

### Key Takeaways

1. **Compile-time Safety**: Zig ตรวจสอบ format string ตอน compile จึงไม่มี runtime format error

2. **bufPrint vs allocPrint**:
   - `bufPrint` - เร็วกว่า ใช้ stack buffer ต้องรู้ขนาดล่วงหน้า
   - `allocPrint` - ยืดหยุ่นกว่า ใช้ heap ต้องจำ free

3. **Custom Formatters**: การ implement `format` method ทำให้ struct ของเราแสดงผลได้สวยงามและรองรับ specifier หลากหลาย

4. **Writer Interface**: ทุก type ที่มี `write` method ใช้กับ `std.fmt.format` ได้ ทำให้โค้ด reusable มาก

5. **Performance**: ใช้ `BufferedWriter` เมื่อต้องการ write หลายครั้งเพื่อลด syscall

### Format Specifier Quick Reference

```
{d}     - decimal integer
{d:.N}  - decimal float with N decimal places
{s}     - string slice
{x}     - hex lowercase
{X}     - hex uppercase
{b}     - binary
{o}     - octal
{e}     - scientific notation (1.23e+04)
{f}     - fixed notation
{c}     - character
{}      - default (depends on type)
{any}   - verbose debug

Alignment:
{d:10}   - right-aligned, width 10
{d:<10}  - left-aligned, width 10
{d:^10}  - center-aligned, width 10
{d:0>10} - zero-padded right-aligned, width 10
{d:*<10} - star-padded left-aligned, width 10
```

---

## อ่านเพิ่มเติม

- [Zig std.fmt Documentation](https://ziglang.org/documentation/master/std/#std.fmt)
- [Zig Language Reference - Format Strings](https://ziglang.org/documentation/master/#String-Formatting)
- บทต่อไป: Part 29 - std.mem ใน Zig Standard Library

---

*Part 28 จาก 40 | Steps 271-280*
