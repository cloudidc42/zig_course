# Part 02: Hello World และโครงสร้างโปรแกรม Zig
## Steps 11-20: พื้นฐานการเขียนโปรแกรม Zig

---

> **หมายเหตุ**: บทนี้ครอบคลุม Steps 11-20 ซึ่งเป็นพื้นฐานสำคัญที่สุดของการเขียนโปรแกรม Zig
> ทุกโปรแกรมที่ซับซ้อนล้วนเริ่มต้นจากสิ่งที่เรียนในบทนี้

---

## Step 11: โปรแกรม Hello World แรก

### ความสำคัญของ Hello World

โปรแกรม "Hello, World!" เป็นประเพณีที่ยาวนานในการเรียนรู้ภาษาโปรแกรมมิ่ง โดยเริ่มต้นจากหนังสือ "The C Programming Language" ของ Brian Kernighan และ Dennis Ritchie ในปี 1978 การเขียน Hello World ช่วยให้เราตรวจสอบว่าสภาพแวดล้อมการพัฒนา (development environment) ถูกตั้งค่าอย่างถูกต้อง

### Hello World ในภาษา Zig

สร้างไฟล์ชื่อ `hello.zig` และเขียนโค้ดดังนี้:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, World!\n", .{});
}
```

### วิธีรัน

```bash
# Compile และรันในคำสั่งเดียว
zig run hello.zig

# หรือ compile ก่อน แล้วค่อยรัน
zig build-exe hello.zig
./hello
```

**ผลลัพธ์:**
```
Hello, World!
```

### อธิบายทีละบรรทัด

**บรรทัดที่ 1:** `const std = @import("std");`
- `const` คือ keyword ที่บอกว่าตัวแปรนี้เป็น constant (ค่าคงที่) ไม่สามารถเปลี่ยนค่าได้
- `std` คือชื่อตัวแปรที่เราตั้งให้กับ standard library
- `@import("std")` คือ built-in function ที่ทำการ import standard library ของ Zig
- เครื่องหมาย `;` ต้องอยู่ท้ายทุก statement ใน Zig

**บรรทัดที่ 2:** `pub fn main() void {`
- `pub` หมายความว่า function นี้เป็น public (สามารถเข้าถึงได้จากภายนอก)
- `fn` คือ keyword สำหรับประกาศ function
- `main` คือชื่อของ function ที่เป็นจุดเริ่มต้นของโปรแกรม
- `()` คือ parameter list ที่ว่างเปล่า (ไม่มี parameter)
- `void` คือ return type ที่บอกว่า function นี้ไม่คืนค่าอะไร
- `{` เปิด block ของ function

**บรรทัดที่ 3:** `std.debug.print("Hello, World!\n", .{});`
- `std` คือ standard library ที่เรา import มา
- `.debug` คือ module ย่อยภายใน std สำหรับการ debug
- `.print` คือ function สำหรับพิมพ์ข้อความ
- `"Hello, World!\n"` คือ format string ที่ต้องการพิมพ์ (`\n` คือ newline)
- `.{}` คือ anonymous struct ที่ว่างเปล่า (ใช้เมื่อไม่มี argument เพิ่มเติม)

**บรรทัดที่ 4:** `}`
- ปิด block ของ function main

### ข้อผิดพลาดที่พบบ่อย

**ข้อผิดพลาดที่ 1: ลืมใส่ semicolon**
```zig
// ผิด
const std = @import("std")  // ขาด ;
pub fn main() void {
    std.debug.print("Hello!\n", .{})  // ขาด ;
}
```

**ข้อผิดพลาดที่ 2: สะกด @import ผิด**
```zig
// ผิด
const std = @Import("std");   // I ตัวใหญ่ผิด
const std = import("std");    // ขาด @
const std = @import('std');   // ใช้ single quote ผิด
```

**ข้อผิดพลาดที่ 3: ลืม `.{}` argument**
```zig
// ผิด - Zig จะฟ้อง error
std.debug.print("Hello!\n");

// ถูก
std.debug.print("Hello!\n", .{});
```

---

## Step 12: โครงสร้างไฟล์ .zig

### ลำดับการจัดโครงสร้างของไฟล์ Zig

ไฟล์ `.zig` มีโครงสร้างที่ควรจัดเรียงตามลำดับดังนี้:

```zig
//! โค้ดในส่วนนี้คือ "doc comment" ระดับ module
//! อธิบายว่าไฟล์นี้ทำอะไร

// 1. Import declarations (การ import libraries)
const std = @import("std");
const mem = std.mem;
const ArrayList = std.ArrayList;

// 2. Constants (ค่าคงที่ระดับ top-level)
const MAX_SIZE: usize = 100;
const APP_NAME = "MyApp";

// 3. Type definitions (การนิยาม types ใหม่)
const Point = struct {
    x: f32,
    y: f32,
};

// 4. Global variables (ตัวแปร global - ใช้น้อยที่สุด)
var global_counter: i32 = 0;

// 5. Function declarations (การประกาศ functions)
pub fn main() void {
    // โค้ดหลักของโปรแกรม
}

fn helperFunction() void {
    // function ช่วยเหลือ
}
```

### การตั้งชื่อไฟล์ใน Zig

ไฟล์ `.zig` มีกฎการตั้งชื่อดังนี้:

| ประเภท | รูปแบบ | ตัวอย่าง |
|--------|--------|---------|
| โปรแกรมทั่วไป | snake_case | `hello_world.zig` |
| Module/Library | snake_case | `string_utils.zig` |
| Entry point | `main.zig` | `main.zig` |
| Test file | `*_test.zig` | `parser_test.zig` |

### โครงสร้างโปรเจกต์ Zig มาตรฐาน

```
my_project/
├── build.zig          # Build configuration (สำหรับ zig build)
├── build.zig.zon      # Package manifest
├── src/
│   ├── main.zig       # Entry point ของโปรแกรม
│   ├── lib.zig        # Library code หลัก
│   └── utils/
│       ├── string.zig
│       └── math.zig
└── tests/
    └── test_main.zig  # Test files
```

### ตัวอย่างไฟล์ที่มีโครงสร้างสมบูรณ์

```zig
//! โมดูลนี้สาธิตโครงสร้างไฟล์ Zig ที่ดี
//! เหมาะสำหรับการเรียนรู้ขั้นต้น

const std = @import("std");
const fmt = std.fmt;
const io = std.io;

// ค่าคงที่
const GREETING = "สวัสดี";
const MAX_NAME_LENGTH: usize = 50;

// Type ที่กำหนดเอง
const Person = struct {
    name: []const u8,
    age: u32,
};

/// ฟังก์ชันหลักของโปรแกรม
pub fn main() void {
    const person = Person{
        .name = "สมชาย",
        .age = 25,
    };

    std.debug.print("{s} คุณ{s} อายุ {d} ปี\n", .{
        GREETING,
        person.name,
        person.age,
    });
}

/// ฟังก์ชัน private สำหรับใช้งานภายใน module
fn formatGreeting(name: []const u8) []const u8 {
    _ = name;
    return GREETING;
}
```

### Comments ใน Zig

Zig มีรูปแบบ comment หลายประเภท:

```zig
// นี่คือ line comment ธรรมดา

/// นี่คือ doc comment สำหรับ declaration ต่อไป
/// ใช้สำหรับ function, struct, หรือ variable
pub fn myFunction() void {}

//! นี่คือ doc comment ระดับ module (ต้องอยู่ต้นไฟล์)
//! อธิบายทั้ง module

/* Zig ไม่มี block comment แบบ C! */
// ต้องใช้ line comment เท่านั้น
```

---

## Step 13: ฟังก์ชัน main และ return value

### ความสำคัญของ main function

`main` เป็น entry point หรือจุดเริ่มต้นของทุกโปรแกรม Zig เมื่อระบบปฏิบัติการเรียกโปรแกรมของเรา มันจะเริ่มที่ `main` เสมอ

### รูปแบบต่างๆ ของ main

**รูปแบบที่ 1: main แบบ void (ไม่คืนค่า)**
```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("โปรแกรมจบแบบปกติ\n", .{});
}
```

**รูปแบบที่ 2: main แบบคืนค่า error**
```zig
const std = @import("std");

pub fn main() !void {
    // ! หมายความว่า function นี้อาจ return error ได้
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello!\n", .{});
}
```

**รูปแบบที่ 3: main พร้อม argument (command line args)**
```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    std.debug.print("จำนวน arguments: {d}\n", .{args.len});
}
```

### Return Values และ Error Handling

```zig
const std = @import("std");

// Function ที่คืนค่า integer
fn add(a: i32, b: i32) i32 {
    return a + b;
}

// Function ที่อาจ fail (คืนค่า error union)
fn divide(a: f64, b: f64) !f64 {
    if (b == 0) {
        return error.DivisionByZero;
    }
    return a / b;
}

pub fn main() !void {
    // เรียกใช้ function ธรรมดา
    const result = add(10, 20);
    std.debug.print("10 + 20 = {d}\n", .{result});

    // เรียกใช้ function ที่อาจ fail ด้วย try
    const quotient = try divide(10.0, 2.0);
    std.debug.print("10 / 2 = {d}\n", .{quotient});

    // จัดการ error ด้วย catch
    const bad_result = divide(10.0, 0.0) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("ผลหาร: {d}\n", .{bad_result});
}
```

### Exit Codes

```zig
const std = @import("std");
const process = std.process;

pub fn main() void {
    const success = true;

    if (!success) {
        // ออกจากโปรแกรมด้วย exit code 1 (แสดงว่ามี error)
        process.exit(1);
    }

    std.debug.print("โปรแกรมสำเร็จ!\n", .{});
    // ออกจากโปรแกรมโดยอัตโนมัติด้วย exit code 0 (success)
}
```

### ความแตกต่างระหว่าง void และ !void

| รูปแบบ | ความหมาย | เมื่อใช้ |
|--------|---------|---------|
| `void` | ไม่คืนค่า ไม่มี error | โปรแกรมง่ายๆ |
| `!void` | ไม่คืนค่า แต่อาจมี error | เมื่อใช้ `try` หรือ I/O |
| `i32` | คืนค่า integer | Functions ที่คำนวณ |
| `!i32` | คืนค่า integer หรือ error | Functions ที่อาจ fail |

---

## Step 14: การ import standard library ด้วย @import

### @import คืออะไร?

`@import` เป็น built-in function ของ Zig สำหรับการโหลด module เข้ามาในโปรแกรม ต่างจากภาษาอื่นที่ใช้ `import`, `require`, หรือ `#include`

### การ import standard library

```zig
// import ทั้ง standard library
const std = @import("std");

// ใช้ผ่าน std.xxx
std.debug.print("Hello\n", .{});
```

### การ import เฉพาะส่วนที่ต้องการ

```zig
const std = @import("std");

// Method 1: ดึง sub-module มาใช้โดยตรง
const mem = std.mem;
const fmt = std.fmt;
const io = std.io;
const fs = std.fs;
const math = std.math;
const time = std.time;
const Thread = std.Thread;
const ArrayList = std.ArrayList;

pub fn main() !void {
    // ใช้ ArrayList แทน std.ArrayList
    var list = ArrayList(i32).init(std.heap.page_allocator);
    defer list.deinit();

    try list.append(1);
    try list.append(2);
    try list.append(3);

    std.debug.print("List มี {d} items\n", .{list.items.len});
}
```

### การ import ไฟล์ของเราเอง

```zig
// สมมติมีไฟล์ utils.zig อยู่ใน directory เดียวกัน
const utils = @import("utils.zig");

// หรือ import จาก subdirectory
const parser = @import("parser/json.zig");

pub fn main() void {
    utils.doSomething();
}
```

**ไฟล์ utils.zig:**
```zig
pub fn doSomething() void {
    const std = @import("std");
    std.debug.print("ทำอะไรบางอย่าง!\n", .{});
}

// function ที่ไม่มี pub จะไม่สามารถเรียกจากนอกไฟล์ได้
fn privateHelper() void {
    // ใช้ได้เฉพาะภายใน utils.zig
}
```

### @import กับ Package Manager

ใน Zig สมัยใหม่ (0.12+) สามารถ import packages จาก `build.zig.zon`:

```zig
// import package ที่กำหนดใน build.zig
const zap = @import("zap");
const httpz = @import("httpz");

pub fn main() !void {
    // ใช้ library ภายนอก
}
```

### สิ่งที่สำคัญเกี่ยวกับ @import

```zig
const std = @import("std");

pub fn main() void {
    // @import คืนค่าเป็น type ดังนั้นต้องใช้ const เสมอ
    // ไม่สามารถใช้ var ได้สำหรับ modules

    // ถูก
    const myMod = @import("mymodule.zig");
    _ = myMod;

    // ผิด - modules ไม่ใช่ runtime values
    // var myMod = @import("mymodule.zig");  // Error!

    std.debug.print("Import สำเร็จ!\n", .{});
}
```

### ตาราง Standard Library ที่ใช้บ่อย

| Module | ใช้สำหรับ | ตัวอย่าง |
|--------|----------|---------|
| `std.debug` | การ debug และ print | `std.debug.print()` |
| `std.io` | Input/Output | `std.io.getStdOut()` |
| `std.mem` | Memory management | `std.mem.copy()` |
| `std.fmt` | Formatting strings | `std.fmt.allocPrint()` |
| `std.fs` | File system | `std.fs.cwd()` |
| `std.math` | คณิตศาสตร์ | `std.math.sqrt()` |
| `std.ArrayList` | Dynamic arrays | `std.ArrayList(T)` |
| `std.HashMap` | Hash maps | `std.HashMap()` |
| `std.process` | Process control | `std.process.exit()` |
| `std.Thread` | Multi-threading | `std.Thread.spawn()` |

---

## Step 15: std.debug.print vs std.io.getStdOut

### ความแตกต่างหลัก

Zig มีสองวิธีหลักในการพิมพ์ข้อความ:

1. **`std.debug.print`** - ง่าย รวดเร็ว สำหรับ debug
2. **`std.io.getStdOut().writer()`** - ยืดหยุ่น เหมาะกับ production

### std.debug.print

```zig
const std = @import("std");

pub fn main() void {
    // วิธีที่ง่ายที่สุด - พิมพ์ไปที่ stderr
    std.debug.print("ข้อความ debug\n", .{});
    std.debug.print("ค่าตัวเลข: {d}\n", .{42});
    std.debug.print("ข้อความ: {s}\n", .{"สวัสดี"});

    // พิมพ์หลายค่าพร้อมกัน
    std.debug.print("x={d}, y={d}\n", .{10, 20});
}
```

**ข้อดีของ std.debug.print:**
- ใช้งานง่าย
- ไม่ต้อง handle errors
- เหมาะสำหรับการ debug
- Thread-safe

**ข้อเสียของ std.debug.print:**
- พิมพ์ไปที่ stderr (ไม่ใช่ stdout)
- ไม่เหมาะสำหรับ output จริงของโปรแกรม
- ไม่สามารถ redirect ผ่าน pipe ได้ง่าย

### std.io.getStdOut

```zig
const std = @import("std");

pub fn main() !void {
    // รับ writer จาก stdout
    const stdout = std.io.getStdOut().writer();

    // พิมพ์ข้อความปกติ
    try stdout.print("Hello, World!\n", .{});
    try stdout.print("ตัวเลข: {d}\n", .{42});

    // พิมพ์โดยไม่มี format
    try stdout.writeAll("ข้อความธรรมดา\n");

    // พิมพ์ทีละ byte
    try stdout.writeByte('A');
    try stdout.writeByte('\n');
}
```

### std.io.getStdErr

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    const stderr = std.io.getStdErr().writer();

    // ข้อความปกติไปที่ stdout
    try stdout.print("โปรแกรมกำลังทำงาน...\n", .{});

    // Error messages ไปที่ stderr
    try stderr.print("คำเตือน: ไม่พบไฟล์\n", .{});
}
```

### Buffered I/O สำหรับ performance ที่ดีขึ้น

```zig
const std = @import("std");

pub fn main() !void {
    const stdout_file = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout_file.writer());
    const stdout = bw.writer();

    // เขียนข้อมูลหลายบรรทัด - จะถูก buffer ไว้
    try stdout.print("บรรทัดที่ 1\n", .{});
    try stdout.print("บรรทัดที่ 2\n", .{});
    try stdout.print("บรรทัดที่ 3\n", .{});

    // Flush buffer เพื่อให้ข้อมูลออกสู่ stdout จริงๆ
    try bw.flush();
}
```

### เปรียบเทียบการใช้งาน

```zig
const std = @import("std");

pub fn main() !void {
    // === วิธีที่ 1: std.debug.print ===
    // ง่าย ไม่ต้อง try, ไปที่ stderr
    std.debug.print("Debug: {d}\n", .{42});

    // === วิธีที่ 2: stdout writer ===
    // ยืดหยุ่น ต้อง try, ไปที่ stdout
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Output: {d}\n", .{42});

    // === วิธีที่ 3: writeAll ===
    // สำหรับ strings ที่ไม่มี formatting
    try stdout.writeAll("ข้อความธรรมดา\n");

    // === วิธีที่ 4: print ด้วย std.fmt ===
    // สำหรับสร้าง string ก่อนแล้วค่อยพิมพ์
    var buf: [100]u8 = undefined;
    const msg = try std.fmt.bufPrint(&buf, "ข้อความ {d}\n", .{100});
    try stdout.writeAll(msg);
}
```

### เมื่อไหรควรใช้แบบไหน?

| สถานการณ์ | แนะนำ |
|-----------|-------|
| Debug ระหว่าง development | `std.debug.print` |
| Output ของโปรแกรม | `stdout.print` |
| Error messages | `stderr.print` |
| พิมพ์ข้อมูลเยอะ | Buffered writer |
| Log messages | `std.log.info/warn/err` |

---

## Step 16: ความแตกต่างระหว่าง print formats: {}, {s}, {d}, {any}

### Format Specifiers ใน Zig

Zig ใช้ format strings คล้ายกับ Python's f-strings หรือ Rust's format macros แต่มีวิธีการที่เฉพาะตัว

### รูปแบบพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    // {} - ใช้ default format สำหรับ type นั้น
    std.debug.print("{}\n", .{42});          // พิมพ์: 42
    std.debug.print("{}\n", .{3.14});        // พิมพ์: 3.14
    std.debug.print("{}\n", .{true});        // พิมพ์: true
    std.debug.print("{}\n", .{'A'});         // พิมพ์: A

    // {s} - สำหรับ strings (slices of bytes)
    std.debug.print("{s}\n", .{"สวัสดี"});   // พิมพ์: สวัสดี

    // {d} - สำหรับตัวเลขในระบบ decimal
    std.debug.print("{d}\n", .{42});         // พิมพ์: 42
    std.debug.print("{d}\n", .{-100});       // พิมพ์: -100

    // {any} - ใช้ได้กับทุก type รวมถึง structs ที่ซับซ้อน
    std.debug.print("{any}\n", .{42});       // พิมพ์: 42
    std.debug.print("{any}\n", .{.{1, 2, 3}}); // พิมพ์: { 1, 2, 3 }
}
```

### Format Specifiers ทั้งหมด

```zig
const std = @import("std");

pub fn main() void {
    const n: i32 = 255;

    // ระบบตัวเลขต่างๆ
    std.debug.print("Decimal: {d}\n", .{n});    // 255
    std.debug.print("Binary:  {b}\n", .{n});    // 11111111
    std.debug.print("Octal:   {o}\n", .{n});    // 377
    std.debug.print("Hex:     {x}\n", .{n});    // ff
    std.debug.print("Hex:     {X}\n", .{n});    // FF (uppercase)
    std.debug.print("Hex:     {0x}\n", .{n});   // 0xff (with prefix)

    // ตัวเลขทศนิยม
    const f: f64 = 3.14159;
    std.debug.print("Float:   {}\n", .{f});     // 3.14159e0
    std.debug.print("Float:   {d}\n", .{f});    // 3.14159
    std.debug.print("Float:   {e}\n", .{f});    // 3.14159e+00

    // Unicode
    const c: u21 = 'ก';
    std.debug.print("Char: {u}\n", .{c});       // ก

    // Pointer
    const ptr: *const i32 = &n;
    std.debug.print("Pointer: {*}\n", .{ptr});  // ที่อยู่ memory
}
```

### การกำหนด Width และ Alignment

```zig
const std = @import("std");

pub fn main() void {
    // กำหนด width
    std.debug.print("{d:10}\n", .{42});      // "        42" (right-aligned)
    std.debug.print("{d:<10}\n", .{42});     // "42        " (left-aligned)
    std.debug.print("{d:^10}\n", .{42});     // "    42    " (center-aligned)

    // กำหนด fill character
    std.debug.print("{d:0>5}\n", .{42});     // "00042" (zero-padded)
    std.debug.print("{d:*<5}\n", .{42});     // "42***"

    // ตัวเลขทศนิยม - กำหนดจำนวน decimal places
    std.debug.print("{d:.2}\n", .{3.14159}); // "3.14"
    std.debug.print("{d:.4}\n", .{3.14159}); // "3.1416"

    // รวม width และ precision
    std.debug.print("{d:10.2}\n", .{3.14});  // "      3.14"
}
```

### ความแตกต่างระหว่าง {} และ {any}

```zig
const std = @import("std");

const MyStruct = struct {
    x: i32,
    y: i32,
};

pub fn main() void {
    const point = MyStruct{ .x = 10, .y = 20 };

    // {} อาจไม่ทำงานกับ struct ที่ไม่มี custom formatter
    // std.debug.print("{}\n", .{point});  // อาจ error

    // {any} ทำงานกับทุก type
    std.debug.print("{any}\n", .{point});  // main.MyStruct{ .x = 10, .y = 20 }

    // Arrays และ Slices
    const arr = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("{any}\n", .{arr});    // { 1, 2, 3, 4, 5 }

    // Optionals
    const maybe: ?i32 = 42;
    std.debug.print("{any}\n", .{maybe});  // 42
    const none: ?i32 = null;
    std.debug.print("{any}\n", .{none});   // null
}
```

### Custom Formatters

```zig
const std = @import("std");

const Color = struct {
    r: u8,
    g: u8,
    b: u8,

    // Custom format function
    pub fn format(
        self: Color,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("rgb({d}, {d}, {d})", .{ self.r, self.g, self.b });
    }
};

pub fn main() void {
    const red = Color{ .r = 255, .g = 0, .b = 0 };

    // ใช้ custom formatter ด้วย {}
    std.debug.print("สีแดง: {}\n", .{red});   // rgb(255, 0, 0)
    std.debug.print("สีแดง: {any}\n", .{red}); // Color{ .r = 255, .g = 0, .b = 0 }
}
```

### ตาราง Format Specifiers สรุป

| Specifier | ใช้กับ | ตัวอย่าง output |
|-----------|--------|----------------|
| `{}` | Default format | ขึ้นอยู่กับ type |
| `{s}` | Strings/slices | `"hello"` |
| `{d}` | ตัวเลข decimal | `42` |
| `{b}` | Binary | `101010` |
| `{o}` | Octal | `52` |
| `{x}` | Hex lowercase | `2a` |
| `{X}` | Hex uppercase | `2A` |
| `{e}` | Scientific notation | `4.2e1` |
| `{u}` | Unicode codepoint | `*` |
| `{c}` | ASCII character | `A` |
| `{any}` | ทุก type | format อัตโนมัติ |
| `{*}` | Pointer address | `0x7fff...` |

---

## Step 17: การอ่าน input จาก stdin

### วิธีอ่าน Input พื้นฐาน

การรับข้อมูลจากผู้ใช้ใน Zig ต้องใช้ `stdin` reader:

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    // ถามผู้ใช้
    try stdout.print("กรุณาใส่ชื่อของคุณ: ", .{});

    // อ่านข้อมูล - ใช้ buffer
    var buf: [100]u8 = undefined;
    const input = try stdin.readUntilDelimiter(&buf, '\n');

    try stdout.print("สวัสดี, {s}!\n", .{input});
}
```

### การจัดการ newline characters

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    try stdout.print("ใส่ข้อความ: ", .{});

    var buf: [256]u8 = undefined;
    const raw_input = try stdin.readUntilDelimiterOrEof(&buf, '\n');

    if (raw_input) |line| {
        // ลบ \r ถ้ามี (Windows line endings)
        const trimmed = std.mem.trimRight(u8, line, "\r");
        try stdout.print("คุณพิมพ์: '{s}'\n", .{trimmed});
    }
}
```

### การอ่านตัวเลขจาก stdin

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    try stdout.print("ใส่ตัวเลข: ", .{});

    var buf: [50]u8 = undefined;
    const input = try stdin.readUntilDelimiterOrEof(&buf, '\n') orelse {
        try stdout.print("ไม่มี input!\n", .{});
        return;
    };

    // แปลง string เป็น integer
    const trimmed = std.mem.trimRight(u8, input, "\r\n ");
    const number = std.fmt.parseInt(i32, trimmed, 10) catch |err| {
        try stdout.print("Error: ไม่ใช่ตัวเลขที่ถูกต้อง ({any})\n", .{err});
        return;
    };

    try stdout.print("ตัวเลขของคุณคือ: {d}\n", .{number});
    try stdout.print("กำลังสอง: {d}\n", .{number * number});
}
```

### การอ่านหลาย inputs

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    var sum: i64 = 0;
    var count: u32 = 0;

    try stdout.print("ใส่ตัวเลข (กด Enter เพื่อจบ, ใส่ 'done' เพื่อสิ้นสุด):\n", .{});

    var buf: [50]u8 = undefined;
    while (true) {
        try stdout.print("> ", .{});

        const input = stdin.readUntilDelimiterOrEof(&buf, '\n') catch break orelse break;
        const trimmed = std.mem.trim(u8, input, " \t\r\n");

        if (trimmed.len == 0) break;
        if (std.mem.eql(u8, trimmed, "done")) break;

        const num = std.fmt.parseInt(i64, trimmed, 10) catch {
            try stdout.print("ไม่ใช่ตัวเลข ลองใหม่\n", .{});
            continue;
        };

        sum += num;
        count += 1;
    }

    if (count > 0) {
        try stdout.print("\nใส่ {d} ตัวเลข\n", .{count});
        try stdout.print("ผลรวม: {d}\n", .{sum});
        try stdout.print("เฉลี่ย: {d:.2}\n", .{@as(f64, @floatFromInt(sum)) / @as(f64, @floatFromInt(count))});
    } else {
        try stdout.print("ไม่มีตัวเลข\n", .{});
    }
}
```

### การอ่าน stdin ทั้งหมด

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdin = std.io.getStdIn();

    // อ่านทั้งหมดจาก stdin (เช่น pipe input)
    const content = try stdin.readToEndAlloc(allocator, 1024 * 1024); // max 1MB
    defer allocator.free(content);

    std.debug.print("อ่านได้ {d} bytes\n", .{content.len});
    std.debug.print("เนื้อหา:\n{s}\n", .{content});
}
```

---

## Step 18: Arguments จาก command line

### การรับ Command Line Arguments

```zig
const std = @import("std");

pub fn main() !void {
    // วิธีที่ 1: ใช้ std.process.args() - ไม่ต้องใช้ allocator
    var args = std.process.args();

    // argument แรกคือชื่อโปรแกรมเสมอ
    const program_name = args.next() orelse "unknown";
    std.debug.print("ชื่อโปรแกรม: {s}\n", .{program_name});

    // arguments ที่เหลือ
    var arg_count: u32 = 0;
    while (args.next()) |arg| {
        arg_count += 1;
        std.debug.print("Arg {d}: {s}\n", .{arg_count, arg});
    }

    std.debug.print("รวม {d} arguments\n", .{arg_count});
}
```

**วิธีรัน:**
```bash
zig run args.zig -- hello world 123
# ผลลัพธ์:
# ชื่อโปรแกรม: args
# Arg 1: hello
# Arg 2: world
# Arg 3: 123
# รวม 3 arguments
```

### การรับ Arguments แบบ Array

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // วิธีที่ 2: argsAlloc - ได้ slice ของ strings
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    std.debug.print("จำนวน arguments ทั้งหมด: {d}\n", .{args.len});

    for (args, 0..) |arg, i| {
        std.debug.print("args[{d}] = {s}\n", .{i, arg});
    }
}
```

### โปรแกรม Calculator จาก Command Line

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    var args = std.process.args();

    // ข้าม argument แรก (ชื่อโปรแกรม)
    _ = args.next();

    // รับตัวเลขแรก
    const num1_str = args.next() orelse {
        try stdout.print("การใช้งาน: calc <num1> <op> <num2>\n", .{});
        return;
    };

    // รับ operator
    const op = args.next() orelse {
        try stdout.print("กรุณาระบุ operator (+, -, *, /)\n", .{});
        return;
    };

    // รับตัวเลขที่สอง
    const num2_str = args.next() orelse {
        try stdout.print("กรุณาระบุตัวเลขที่สอง\n", .{});
        return;
    };

    // แปลงเป็นตัวเลข
    const num1 = try std.fmt.parseFloat(f64, num1_str);
    const num2 = try std.fmt.parseFloat(f64, num2_str);

    // คำนวณ
    const result = switch (op[0]) {
        '+' => num1 + num2,
        '-' => num1 - num2,
        '*' => num1 * num2,
        '/' => blk: {
            if (num2 == 0) {
                try stdout.print("Error: หารด้วยศูนย์ไม่ได้!\n", .{});
                return;
            }
            break :blk num1 / num2;
        },
        else => {
            try stdout.print("Operator ไม่รู้จัก: {s}\n", .{op});
            return;
        },
    };

    try stdout.print("{s} {s} {s} = {d:.4}\n", .{num1_str, op, num2_str, result});
}
```

**วิธีรัน:**
```bash
zig run calc.zig -- 10 + 20   # 10.0000 + 20.0000 = 30.0000
zig run calc.zig -- 15 / 3    # 15.0000 / 3.0000 = 5.0000
```

### การ Parse Arguments แบบมี Flags

```zig
const std = @import("std");

const Config = struct {
    verbose: bool = false,
    output: []const u8 = "output.txt",
    count: i32 = 1,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    var config = Config{};
    var args = std.process.args();

    // ข้าม program name
    _ = args.next();

    while (args.next()) |arg| {
        if (std.mem.eql(u8, arg, "--verbose") or std.mem.eql(u8, arg, "-v")) {
            config.verbose = true;
        } else if (std.mem.eql(u8, arg, "--output") or std.mem.eql(u8, arg, "-o")) {
            config.output = args.next() orelse {
                try stdout.print("Error: --output ต้องการค่า\n", .{});
                return;
            };
        } else if (std.mem.eql(u8, arg, "--count") or std.mem.eql(u8, arg, "-n")) {
            const count_str = args.next() orelse {
                try stdout.print("Error: --count ต้องการค่า\n", .{});
                return;
            };
            config.count = try std.fmt.parseInt(i32, count_str, 10);
        } else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
            try stdout.print("การใช้งาน: myapp [options]\n", .{});
            try stdout.print("  -v, --verbose   แสดงข้อมูลเพิ่มเติม\n", .{});
            try stdout.print("  -o, --output    ไฟล์ output (default: output.txt)\n", .{});
            try stdout.print("  -n, --count     จำนวนครั้ง (default: 1)\n", .{});
            return;
        }
    }

    if (config.verbose) {
        try stdout.print("Verbose mode: เปิด\n", .{});
        try stdout.print("Output file: {s}\n", .{config.output});
        try stdout.print("Count: {d}\n", .{config.count});
    }

    try stdout.print("กำลังทำงาน...\n", .{});
}
```

---

## Step 19: Environment Variables

### การอ่าน Environment Variables

```zig
const std = @import("std");
const process = std.process;

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // อ่าน environment variable เดี่ยว
    const path = process.getEnvVarOwned(
        std.heap.page_allocator,
        "PATH"
    ) catch |err| switch (err) {
        error.EnvironmentVariableNotFound => {
            try stdout.print("ไม่พบ PATH variable\n", .{});
            return;
        },
        else => return err,
    };
    defer std.heap.page_allocator.free(path);

    try stdout.print("PATH: {s}\n", .{path});
}
```

### การอ่านหลาย Environment Variables

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdout = std.io.getStdOut().writer();

    // รายการ env vars ที่ต้องการอ่าน
    const env_vars = [_][]const u8{
        "HOME",
        "USER",
        "SHELL",
        "TERM",
        "LANG",
    };

    for (env_vars) |var_name| {
        const value = std.process.getEnvVarOwned(allocator, var_name) catch |err| {
            if (err == error.EnvironmentVariableNotFound) {
                try stdout.print("{s}: (ไม่พบ)\n", .{var_name});
            } else {
                try stdout.print("{s}: (error: {any})\n", .{var_name, err});
            }
            continue;
        };
        defer allocator.free(value);

        try stdout.print("{s}: {s}\n", .{var_name, value});
    }
}
```

### การใช้ Environment Variables สำหรับ Configuration

```zig
const std = @import("std");

const AppConfig = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    db_url: []const u8,
};

fn loadConfig(allocator: std.mem.Allocator) !AppConfig {
    const host = std.process.getEnvVarOwned(allocator, "APP_HOST") catch
        try allocator.dupe(u8, "localhost");

    const port_str = std.process.getEnvVarOwned(allocator, "APP_PORT") catch
        try allocator.dupe(u8, "8080");
    defer allocator.free(port_str);

    const port = std.fmt.parseInt(u16, port_str, 10) catch 8080;

    const debug_str = std.process.getEnvVarOwned(allocator, "APP_DEBUG") catch
        try allocator.dupe(u8, "false");
    defer allocator.free(debug_str);

    const debug = std.mem.eql(u8, debug_str, "true") or
                  std.mem.eql(u8, debug_str, "1");

    const db_url = std.process.getEnvVarOwned(allocator, "DATABASE_URL") catch
        try allocator.dupe(u8, "sqlite://./local.db");

    return AppConfig{
        .host = host,
        .port = port,
        .debug = debug,
        .db_url = db_url,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const config = try loadConfig(allocator);
    defer {
        allocator.free(config.host);
        allocator.free(config.db_url);
    }

    const stdout = std.io.getStdOut().writer();
    try stdout.print("=== App Configuration ===\n", .{});
    try stdout.print("Host:  {s}\n", .{config.host});
    try stdout.print("Port:  {d}\n", .{config.port});
    try stdout.print("Debug: {}\n", .{config.debug});
    try stdout.print("DB:    {s}\n", .{config.db_url});
}
```

**วิธีทดสอบ:**
```bash
APP_HOST=myserver.com APP_PORT=3000 APP_DEBUG=true zig run config.zig
```

### การดู Environment Variables ทั้งหมด

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdout = std.io.getStdOut().writer();

    // รับ env map ทั้งหมด
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();

    try stdout.print("Environment Variables:\n", .{});
    try stdout.print("----------------------\n", .{});

    var it = env_map.iterator();
    var count: u32 = 0;
    while (it.next()) |entry| {
        try stdout.print("{s}={s}\n", .{entry.key_ptr.*, entry.value_ptr.*});
        count += 1;
    }

    try stdout.print("----------------------\n", .{});
    try stdout.print("รวม {d} variables\n", .{count});
}
```

---

## Step 20: การ compile และ run โปรแกรม Zig

### คำสั่งพื้นฐาน

```bash
# ติดตั้ง Zig (ดาวน์โหลดจาก ziglang.org)
# ตรวจสอบ version
zig version

# คำสั่งหลัก
zig run hello.zig         # Compile และรันทันที
zig build-exe hello.zig   # Compile เป็น executable
zig build-lib mylib.zig   # Compile เป็น library
zig test mycode.zig       # รัน tests
zig fmt hello.zig         # Format code
```

### zig run - สำหรับ Development

```bash
# รันไฟล์เดียว
zig run hello.zig

# รันพร้อม arguments
zig run hello.zig -- arg1 arg2 arg3

# รันด้วย optimization
zig run -O ReleaseFast hello.zig

# รันพร้อม additional include paths
zig run hello.zig --mod mylib=mylib.zig
```

### zig build-exe - Compile เป็น Binary

```bash
# Compile พื้นฐาน
zig build-exe hello.zig

# กำหนดชื่อ output
zig build-exe hello.zig -femit-bin=my_program

# Compile สำหรับ platform ต่างๆ (Cross-compilation!)
zig build-exe hello.zig -target x86_64-windows
zig build-exe hello.zig -target x86_64-macos
zig build-exe hello.zig -target aarch64-linux
zig build-exe hello.zig -target wasm32-freestanding
```

### Optimization Modes

```bash
# Debug mode (default) - ช้าแต่มี error checking เยอะ
zig build-exe hello.zig

# ReleaseSafe - เร็วขึ้น แต่ยังมี safety checks
zig build-exe -O ReleaseSafe hello.zig

# ReleaseFast - เร็วสุด ไม่มี safety checks
zig build-exe -O ReleaseFast hello.zig

# ReleaseSmall - ขนาดไฟล์เล็กสุด
zig build-exe -O ReleaseSmall hello.zig
```

### ไฟล์ build.zig สำหรับโปรเจกต์ใหญ่

สำหรับโปรเจกต์ที่ซับซ้อน ควรใช้ `build.zig`:

**สร้างโปรเจกต์ใหม่:**
```bash
mkdir my_project
cd my_project
zig init
```

**ไฟล์ build.zig พื้นฐาน:**
```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // กำหนด target platform
    const target = b.standardTargetOptions(.{});

    // กำหนด optimization level
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง executable
    const exe = b.addExecutable(.{
        .name = "my_project",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ติดตั้ง executable
    b.installArtifact(exe);

    // สร้าง run step
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "รันโปรแกรม");
    run_step.dependOn(&run_cmd.step);

    // สร้าง test step
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "รัน tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

**การใช้ zig build:**
```bash
# Build โปรแกรม
zig build

# Build และรัน
zig build run

# Build และรัน tests
zig build test

# Build แบบ release
zig build -Doptimize=ReleaseFast

# Build สำหรับ platform อื่น
zig build -Dtarget=x86_64-windows
```

### zig fmt - Code Formatting

```bash
# Format ไฟล์เดียว
zig fmt hello.zig

# Format ทุกไฟล์ในโปรเจกต์
zig fmt .

# ตรวจสอบโดยไม่เปลี่ยนแปลง (CI/CD)
zig fmt --check hello.zig
```

**ตัวอย่างโค้ดก่อนและหลัง fmt:**
```zig
// ก่อน fmt (โค้ด messy)
const std=@import("std");
pub fn main()void{
const x=42;
std.debug.print("{d}\n",.{x});}

// หลัง fmt (โค้ด clean)
const std = @import("std");
pub fn main() void {
    const x = 42;
    std.debug.print("{d}\n", .{x});
}
```

### zig test - การเขียนและรัน Tests

```zig
const std = @import("std");

// function ที่จะ test
fn add(a: i32, b: i32) i32 {
    return a + b;
}

fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

// Test cases
test "การบวก" {
    try std.testing.expectEqual(@as(i32, 5), add(2, 3));
    try std.testing.expectEqual(@as(i32, 0), add(-1, 1));
    try std.testing.expectEqual(@as(i32, -5), add(-2, -3));
}

test "การคูณ" {
    try std.testing.expectEqual(@as(i32, 6), multiply(2, 3));
    try std.testing.expectEqual(@as(i32, 0), multiply(0, 100));
    try std.testing.expectEqual(@as(i32, -6), multiply(-2, 3));
}

pub fn main() void {
    std.debug.print("รัน: zig test math.zig\n", .{});
}
```

**รัน tests:**
```bash
zig test math.zig
# ผลลัพธ์:
# All 2 tests passed.
```

### Cross-compilation - จุดเด่นของ Zig

Zig มีความสามารถ cross-compile ที่ยอดเยี่ยมมาก:

```bash
# Compile บน Linux เพื่อรันบน Windows
zig build-exe hello.zig -target x86_64-windows

# Compile บน Linux เพื่อรันบน macOS
zig build-exe hello.zig -target x86_64-macos-none

# Compile สำหรับ ARM Linux (เช่น Raspberry Pi)
zig build-exe hello.zig -target aarch64-linux-gnu

# Compile สำหรับ WebAssembly
zig build-exe hello.zig -target wasm32-freestanding

# ดู targets ที่รองรับทั้งหมด
zig targets
```

### การ Debug โปรแกรม Zig

```bash
# Compile แบบ debug (default)
zig build-exe hello.zig
# ไฟล์ที่ได้จะมี debug info

# ใช้ gdb
gdb ./hello

# ใช้ lldb (macOS)
lldb ./hello

# ตรวจสอบ memory leaks
valgrind ./hello
```

---

## สรุปรวม: โปรแกรมตัวอย่างที่ครอบคลุมทุก Concepts

```zig
//! โปรแกรมสาธิตการใช้งาน Steps 11-20 ครบถ้วน
//! compile: zig run complete_demo.zig -- [name] [number]

const std = @import("std");
const io = std.io;
const fmt = std.fmt;
const mem = std.mem;
const process = std.process;

/// ข้อมูล configuration ของโปรแกรม
const Config = struct {
    name: []const u8 = "ผู้ใช้",
    number: i32 = 0,
    verbose: bool = false,
};

/// แสดงวิธีใช้โปรแกรม
fn showUsage(prog_name: []const u8) !void {
    const stderr = io.getStdErr().writer();
    try stderr.print("การใช้งาน: {s} [ชื่อ] [ตัวเลข] [--verbose]\n", .{prog_name});
}

/// อ่าน configuration จาก arguments
fn parseArgs(allocator: mem.Allocator) !Config {
    const args = try process.argsAlloc(allocator);
    defer process.argsFree(allocator, args);

    var config = Config{};

    if (args.len > 1) {
        config.name = try allocator.dupe(u8, args[1]);
    }

    if (args.len > 2) {
        config.number = fmt.parseInt(i32, args[2], 10) catch 0;
    }

    for (args) |arg| {
        if (mem.eql(u8, arg, "--verbose") or mem.eql(u8, arg, "-v")) {
            config.verbose = true;
        }
    }

    return config;
}

pub fn main() !void {
    // Setup allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Setup I/O
    const stdout_file = io.getStdOut();
    var bw = io.bufferedWriter(stdout_file.writer());
    const stdout = bw.writer();

    // Step 11: Hello World
    try stdout.print("=== สาธิต Steps 11-20 ===\n\n", .{});

    // Step 13: การใช้ return value
    const greeting = "สวัสดีครับ";
    try stdout.print("Step 11-13: {s}, Hello World!\n", .{greeting});

    // Step 18: Command line arguments
    const config = parseArgs(allocator) catch Config{};

    // Step 16: Format specifiers ต่างๆ
    try stdout.print("\nStep 16: Format Specifiers\n", .{});
    try stdout.print("  ชื่อ ({{s}}):    {s}\n", .{config.name});
    try stdout.print("  ตัวเลข ({{d}}):  {d}\n", .{config.number});
    try stdout.print("  Hex ({{x}}):     {x}\n", .{@as(u32, @bitCast(config.number))});
    try stdout.print("  Bool ({{}}):     {}\n", .{config.verbose});
    try stdout.print("  Any ({{any}}):   {any}\n", .{config.number});

    // Step 19: Environment variables
    try stdout.print("\nStep 19: Environment Variables\n", .{});
    const home = process.getEnvVarOwned(allocator, "HOME") catch
        try allocator.dupe(u8, "(ไม่พบ)");
    defer allocator.free(home);

    const user = process.getEnvVarOwned(allocator, "USER") catch
        try allocator.dupe(u8, "(ไม่พบ)");
    defer allocator.free(user);

    try stdout.print("  HOME: {s}\n", .{home});
    try stdout.print("  USER: {s}\n", .{user});

    if (config.verbose) {
        try stdout.print("\n[Verbose] โปรแกรมจบการทำงานสำเร็จ\n", .{});
    }

    // Flush buffer
    try bw.flush();
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Hello ระดับง่าย
เขียนโปรแกรมที่ถามชื่อผู้ใช้จาก stdin แล้วทักทายด้วยชื่อนั้น

```
คาดหวังผลลัพธ์:
กรุณาใส่ชื่อ: สมหมาย
สวัสดีครับคุณสมหมาย! ยินดีต้อนรับสู่ Zig!
```

### แบบฝึกหัดที่ 2: Calculator
เขียนโปรแกรม calculator ที่:
- รับตัวเลข 2 ตัวและ operator จาก command line arguments
- รองรับ +, -, *, /
- แสดง error ที่เข้าใจง่ายเมื่อ input ไม่ถูกต้อง

### แบบฝึกหัดที่ 3: Environment Info
เขียนโปรแกรมที่แสดงข้อมูล environment:
- แสดง HOME, USER, PATH, SHELL
- แสดงจำนวน arguments ที่ได้รับ
- แสดงข้อมูลในตารางที่สวยงาม

### แบบฝึกหัดที่ 4: Number Formatter
เขียนโปรแกรมที่รับตัวเลขแล้วแสดงใน formats ต่างๆ:
- Decimal, Binary, Octal, Hexadecimal
- Scientific notation
- Fixed-point ด้วย precision ต่างๆ

### แบบฝึกหัดที่ 5: Config Reader
เขียนโปรแกรมที่อ่าน configuration จาก environment variables พร้อม default values:
- `APP_HOST` (default: localhost)
- `APP_PORT` (default: 8080)
- `APP_DEBUG` (default: false)
- `APP_NAME` (default: MyApp)

---

## สรุป Part 02

ในบทนี้เราได้เรียนรู้:

| Step | หัวข้อ | สิ่งสำคัญ |
|------|--------|----------|
| 11 | Hello World | โปรแกรม Zig แรก, syntax พื้นฐาน |
| 12 | โครงสร้างไฟล์ | การจัดลำดับ code, comments |
| 13 | main function | void vs !void, return values |
| 14 | @import | การโหลด modules, std library |
| 15 | stdout vs stderr | debug.print vs io writer |
| 16 | Format strings | {}, {s}, {d}, {x}, {any} |
| 17 | stdin | การรับ input จากผู้ใช้ |
| 18 | CLI args | std.process.args() |
| 19 | Env vars | std.process.getEnvVarOwned() |
| 20 | Build system | zig run, build-exe, build.zig |

### ความรู้ที่ต้องจำ

1. **`@import("std")`** ต้องอยู่ต้นไฟล์เสมอ
2. **`pub fn main() !void`** เป็น entry point มาตรฐาน
3. **`std.debug.print`** ไปที่ stderr, **`stdout.print`** ไปที่ stdout
4. **Format specifiers**: `{s}` สำหรับ string, `{d}` สำหรับตัวเลข
5. **`try`** ต้องใช้เมื่อเรียก function ที่อาจ return error
6. **`zig run`** สำหรับ development, **`zig build`** สำหรับ production
7. Zig รองรับ **cross-compilation** โดยค่าเริ่มต้น

### สิ่งที่จะเรียนในบทต่อไป

- Part 03: Types และ Variables - integer, float, bool, strings
- การจัดการ memory แบบ manual
- Pointers และ references
- Arrays และ slices

---

*Part 02 จบแล้ว - ไปต่อที่ Part 03: Types และ Variables*
