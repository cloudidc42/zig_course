# Part 26: Standard Library ภาพรวมของ Zig (Steps 251-260)

## บทนำ

Zig Standard Library (std) คือชุดเครื่องมือที่ครอบคลุมซึ่งมาพร้อมกับภาษา Zig โดยไม่ต้องติดตั้งเพิ่มเติม ในบทนี้เราจะสำรวจโมดูลหลักต่างๆ ของ standard library เพื่อให้เข้าใจความสามารถที่มีอยู่และวิธีใช้งานอย่างมีประสิทธิภาพ

---

## Step 251: Standard Library คืออะไร? การ import

### Standard Library คืออะไร?

Zig Standard Library หรือที่เรียกย่อว่า `std` คือชุดโมดูลที่ Zig จัดเตรียมมาให้นักพัฒนาใช้งาน ประกอบด้วยฟังก์ชันและ type ที่ใช้บ่อยในการพัฒนาซอฟต์แวร์ เช่น:

- การจัดการหน่วยความจำ
- การอ่าน/เขียนไฟล์
- การจัดรูปแบบข้อความ
- คณิตศาสตร์
- การติดต่อกับระบบปฏิบัติการ
- และอื่นๆ อีกมากมาย

### โครงสร้างของ Standard Library

```
std/
├── debug.zig          - เครื่องมือ debug: print, assert, panic
├── mem.zig            - จัดการ memory
├── fmt.zig            - จัดรูปแบบข้อความ
├── io.zig             - input/output
├── fs.zig             - file system
├── os.zig             - OS interface
├── math.zig           - ฟังก์ชันคณิตศาสตร์
├── process.zig        - จัดการ process
├── builtin.zig        - builtin types และ functions
├── testing.zig        - testing framework
├── unicode.zig        - Unicode support
├── crypto.zig         - cryptography
├── hash.zig           - hash functions
├── net.zig            - networking
├── json.zig           - JSON parsing/serialization
├── time.zig           - time utilities
├── rand.zig           - random number generation
├── sort.zig           - sorting algorithms
├── ascii.zig          - ASCII utilities
└── heap.zig           - heap allocators
```

### การ Import Standard Library

วิธีการ import std มีหลายรูปแบบ:

```zig
const std = @import("std");

pub fn main() void {
    // ใช้งาน std ผ่าน namespace
    std.debug.print("Hello from std!\n", .{});
}
```

#### การ import แบบ destructuring

```zig
const std = @import("std");
const debug = std.debug;
const mem = std.mem;
const fmt = std.fmt;

pub fn main() void {
    debug.print("Hello!\n", .{});
}
```

#### การ import โมดูลเฉพาะ

```zig
const print = @import("std").debug.print;

pub fn main() void {
    print("Direct print!\n", .{});
}
```

#### ตัวอย่างการ import แบบสมบูรณ์

```zig
const std = @import("std");

// นำ type และฟังก์ชันที่ใช้บ่อยออกมา
const ArrayList = std.ArrayList;
const Allocator = std.mem.Allocator;
const print = std.debug.print;

pub fn main() !void {
    // ใช้งาน GPA (General Purpose Allocator)
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง ArrayList
    var list = ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.append(1);
    try list.append(2);
    try list.append(3);

    print("List has {} items\n", .{list.items.len});
}
```

### การอ่าน Documentation ของ std

Zig มี documentation ที่ดีมาก สามารถอ่านได้จาก:

1. **Online docs**: https://ziglang.org/documentation/master/std/
2. **Source code**: อยู่ใน `lib/std/` ของ Zig installation
3. **zig build-exe --show-builtin**: แสดง builtin functions ทั้งหมด

```bash
# ดู path ของ std library
zig env
```

### แผนที่ของ Standard Library

```
std (root)
│
├── debug          → print, assert, panic, log
├── mem            → copy, move, compare, allocate
├── fmt            → format, parseInt, parseFloat
├── io             → Reader, Writer, stdin, stdout
├── fs             → Dir, File, open, create, delete
├── os             → system calls, errno, env
├── math           → sin, cos, sqrt, pow, pi
├── process        → exit, spawn, args, env
├── builtin        → target, cpu, os, mode
│
├── heap           → allocators
├── testing        → test framework
├── json           → JSON encode/decode
├── unicode        → UTF-8 handling
├── crypto         → hash, cipher, random
├── net            → networking
├── time           → clock, sleep, duration
├── rand           → PRNG
├── sort           → sort algorithms
└── ArrayList      → dynamic array
```

---

## Step 252: std.debug - print, assert, panic

### std.debug คืออะไร?

`std.debug` เป็นโมดูลสำหรับการ debug และแสดงข้อมูลในระหว่างพัฒนา ประกอบด้วยเครื่องมือสำคัญสามอย่างหลัก:

1. **print** - แสดงข้อมูลไปยัง stderr
2. **assert** - ตรวจสอบเงื่อนไข
3. **panic** - หยุดโปรแกรมแบบฉุกเฉิน

### std.debug.print

```zig
const std = @import("std");

pub fn main() void {
    // print พื้นฐาน
    std.debug.print("Hello, World!\n", .{});

    // print พร้อม arguments
    const name = "Zig";
    const version = 0.14;
    std.debug.print("ภาษา: {s}, เวอร์ชัน: {d:.1}\n", .{ name, version });

    // print ตัวเลข
    const x: i32 = 42;
    const y: f64 = 3.14159;
    std.debug.print("x = {}, y = {d:.3}\n", .{ x, y });

    // print binary, hex, octal
    const n: u8 = 255;
    std.debug.print("decimal: {d}\n", .{n});
    std.debug.print("hex:     {x}\n", .{n});    // ff
    std.debug.print("HEX:     {X}\n", .{n});    // FF
    std.debug.print("binary:  {b}\n", .{n});    // 11111111
    std.debug.print("octal:   {o}\n", .{n});    // 377
}
```

### Format Specifiers ของ std.debug.print

```zig
const std = @import("std");

pub fn main() void {
    // {s} - string
    std.debug.print("{s}\n", .{"hello"});

    // {d} - integer (decimal)
    std.debug.print("{d}\n", .{42});

    // {f} - float
    std.debug.print("{f}\n", .{3.14});

    // {e} - scientific notation
    std.debug.print("{e}\n", .{1234567.89});

    // {c} - character
    std.debug.print("{c}\n", .{'A'});

    // {?} - optional
    const opt: ?i32 = 42;
    const null_opt: ?i32 = null;
    std.debug.print("opt: {?d}\n", .{opt});
    std.debug.print("null: {?d}\n", .{null_opt});

    // {} - default format
    std.debug.print("{}\n", .{true});
    std.debug.print("{}\n", .{@as(u8, 65)});

    // {any} - format any type
    const arr = [_]i32{ 1, 2, 3 };
    std.debug.print("{any}\n", .{arr});
}
```

### std.debug.assert

```zig
const std = @import("std");

pub fn divide(a: i32, b: i32) i32 {
    // ตรวจสอบว่า b ไม่เป็น 0
    std.debug.assert(b != 0);
    return @divTrunc(a, b);
}

pub fn fibonacci(n: u32) u32 {
    // assert ช่วยตรวจสอบ invariants
    std.debug.assert(n <= 50); // ป้องกัน overflow

    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

pub fn main() void {
    const result = divide(10, 2);
    std.debug.print("10 / 2 = {}\n", .{result});

    const fib = fibonacci(10);
    std.debug.print("fibonacci(10) = {}\n", .{fib});

    // assert นี้จะ panic ถ้า condition เป็น false
    std.debug.assert(result == 5);
    std.debug.print("All assertions passed!\n", .{});
}
```

### std.debug.panic

```zig
const std = @import("std");

pub fn safeDivide(a: f64, b: f64) f64 {
    if (b == 0.0) {
        // panic พร้อม error message
        std.debug.panic("Division by zero! a={d}, b={d}", .{ a, b });
    }
    return a / b;
}

pub fn getElement(arr: []const i32, index: usize) i32 {
    if (index >= arr.len) {
        std.debug.panic(
            "Index out of bounds: index={}, len={}",
            .{ index, arr.len },
        );
    }
    return arr[index];
}

pub fn main() void {
    const result = safeDivide(10.0, 2.0);
    std.debug.print("10 / 2 = {d}\n", .{result});

    const arr = [_]i32{ 10, 20, 30, 40, 50 };
    const elem = getElement(&arr, 2);
    std.debug.print("arr[2] = {}\n", .{elem});
}
```

### std.log - Logging System

```zig
const std = @import("std");

const log = std.log;

pub fn main() void {
    // ระดับ log ต่างๆ
    log.debug("Debug message: {s}", .{"testing"});
    log.info("Info: program started", .{});
    log.warn("Warning: low memory", .{});
    log.err("Error: file not found", .{});

    // log scoped - สำหรับ module เฉพาะ
    const myLog = std.log.scoped(.my_module);
    myLog.info("Message from my_module", .{});
}
```

---

## Step 253: std.mem - memory utilities overview

### std.mem คืออะไร?

`std.mem` เป็นโมดูลจัดการหน่วยความจำและ byte operations ซึ่งมีฟังก์ชันสำคัญหลายอย่าง

### การ Copy และ Move Memory

```zig
const std = @import("std");

pub fn main() void {
    var src = [_]u8{ 1, 2, 3, 4, 5 };
    var dst: [5]u8 = undefined;

    // copy memory
    @memcpy(&dst, &src);
    std.debug.print("dst: {any}\n", .{dst});

    // std.mem.copyForwards - สำหรับ overlapping buffers
    var buf = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    std.mem.copyForwards(u8, buf[1..], buf[0..7]);
    std.debug.print("buf after copyForwards: {any}\n", .{buf});

    // std.mem.copyBackwards
    var buf2 = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    std.mem.copyBackwards(u8, buf2[0..7], buf2[1..]);
    std.debug.print("buf2 after copyBackwards: {any}\n", .{buf2});
}
```

### การเปรียบเทียบ Memory

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]u8{ 1, 2, 3, 4, 5 };
    const b = [_]u8{ 1, 2, 3, 4, 5 };
    const c = [_]u8{ 1, 2, 3, 4, 6 };

    // เปรียบเทียบ slices
    std.debug.print("a == b: {}\n", .{std.mem.eql(u8, &a, &b)});  // true
    std.debug.print("a == c: {}\n", .{std.mem.eql(u8, &a, &c)});  // false

    // เปรียบเทียบ strings
    const str1 = "hello";
    const str2 = "hello";
    const str3 = "world";
    std.debug.print("str1 == str2: {}\n", .{std.mem.eql(u8, str1, str2)});
    std.debug.print("str1 == str3: {}\n", .{std.mem.eql(u8, str1, str3)});

    // order comparison
    const order = std.mem.order(u8, &a, &c);
    std.debug.print("order: {}\n", .{order}); // .lt
}
```

### การค้นหาใน Memory

```zig
const std = @import("std");

pub fn main() void {
    const haystack = "Hello, World! Hello, Zig!";
    const needle = "Hello";

    // ค้นหาตำแหน่งแรก
    if (std.mem.indexOf(u8, haystack, needle)) |pos| {
        std.debug.print("Found '{s}' at position {}\n", .{ needle, pos });
    }

    // ค้นหาตำแหน่งสุดท้าย
    if (std.mem.lastIndexOf(u8, haystack, needle)) |pos| {
        std.debug.print("Last '{s}' at position {}\n", .{ needle, pos });
    }

    // ค้นหา byte เดี่ยว
    if (std.mem.indexOfScalar(u8, haystack, 'W')) |pos| {
        std.debug.print("Found 'W' at position {}\n", .{pos});
    }

    // ตรวจสอบ prefix/suffix
    std.debug.print("startsWith: {}\n", .{std.mem.startsWith(u8, haystack, "Hello")});
    std.debug.print("endsWith: {}\n", .{std.mem.endsWith(u8, haystack, "Zig!")});
}
```

### การจัดการ Strings ด้วย std.mem

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // แบ่ง string ด้วย delimiter
    const csv = "apple,banana,cherry,date";
    var iter = std.mem.splitScalar(u8, csv, ',');
    while (iter.next()) |part| {
        std.debug.print("  fruit: {s}\n", .{part});
    }

    // trim whitespace
    const padded = "   hello world   ";
    const trimmed = std.mem.trim(u8, padded, " ");
    std.debug.print("trimmed: '{s}'\n", .{trimmed});

    // replace
    const original = "Hello World Hello";
    const replaced = try std.mem.replaceOwned(u8, allocator, original, "Hello", "Hi");
    defer allocator.free(replaced);
    std.debug.print("replaced: {s}\n", .{replaced});

    // แปลง case
    var buf = "Hello World".*;
    _ = std.ascii.lowerString(&buf, &buf);
    std.debug.print("lower: {s}\n", .{buf});
}
```

### Allocators ใน std.mem

```zig
const std = @import("std");

pub fn main() !void {
    // 1. GeneralPurposeAllocator - สำหรับ development/debugging
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const gpa_alloc = gpa.allocator();

    // 2. FixedBufferAllocator - allocate จาก stack buffer
    var buf: [1024]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buf);
    const fba_alloc = fba.allocator();

    // 3. ArenaAllocator - free ทุกอย่างพร้อมกัน
    var arena = std.heap.ArenaAllocator.init(gpa_alloc);
    defer arena.deinit();
    const arena_alloc = arena.allocator();

    // ใช้งาน allocators
    const arr1 = try gpa_alloc.alloc(i32, 10);
    defer gpa_alloc.free(arr1);

    const arr2 = try fba_alloc.alloc(u8, 100);
    _ = arr2;

    const arr3 = try arena_alloc.alloc(f64, 5);
    _ = arr3;
    // arr3 จะถูก free อัตโนมัติเมื่อ arena.deinit()

    std.debug.print("Allocators work correctly!\n", .{});
}
```

---

## Step 254: std.fmt - formatting overview

### std.fmt คืออะไร?

`std.fmt` จัดการการ format ข้อความและการแปลงข้อมูล ระหว่าง types ต่างๆ

### std.fmt.format และ std.fmt.allocPrint

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // allocPrint - สร้าง string ใน heap
    const str1 = try std.fmt.allocPrint(allocator, "Hello, {s}!", .{"World"});
    defer allocator.free(str1);
    std.debug.print("{s}\n", .{str1});

    // bufPrint - เขียนลงใน buffer ที่มีอยู่
    var buf: [100]u8 = undefined;
    const str2 = try std.fmt.bufPrint(&buf, "x = {d}, y = {d:.2}", .{ 42, 3.14159 });
    std.debug.print("{s}\n", .{str2});

    // countFormatted - นับความยาวที่จะใช้
    const len = std.fmt.count("Hello, {s}! You are {} years old.", .{ "Alice", 30 });
    std.debug.print("Format length: {}\n", .{len});
}
```

### การแปลงตัวเลขเป็น String

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // integer to string
    const n: i64 = -12345;
    const int_str = try std.fmt.allocPrint(allocator, "{d}", .{n});
    defer allocator.free(int_str);
    std.debug.print("int: {s}\n", .{int_str});

    // float to string
    const f: f64 = 3.14159265358979;
    const float_str = try std.fmt.allocPrint(allocator, "{d:.4}", .{f});
    defer allocator.free(float_str);
    std.debug.print("float: {s}\n", .{float_str});

    // hex
    const hex_val: u32 = 0xDEADBEEF;
    const hex_str = try std.fmt.allocPrint(allocator, "0x{X}", .{hex_val});
    defer allocator.free(hex_str);
    std.debug.print("hex: {s}\n", .{hex_str});

    // binary
    const bin_val: u8 = 0b10110101;
    const bin_str = try std.fmt.allocPrint(allocator, "0b{b:0>8}", .{bin_val});
    defer allocator.free(bin_str);
    std.debug.print("binary: {s}\n", .{bin_str});
}
```

### การแปลง String เป็นตัวเลข

```zig
const std = @import("std");

pub fn main() !void {
    // parseInt
    const int_val = try std.fmt.parseInt(i32, "12345", 10);
    std.debug.print("parsed int: {}\n", .{int_val});

    // parseInt ฐานต่างๆ
    const hex_val = try std.fmt.parseInt(u32, "DEADBEEF", 16);
    std.debug.print("parsed hex: 0x{X}\n", .{hex_val});

    const bin_val = try std.fmt.parseInt(u8, "10110101", 2);
    std.debug.print("parsed binary: {}\n", .{bin_val});

    // parseFloat
    const float_val = try std.fmt.parseFloat(f64, "3.14159");
    std.debug.print("parsed float: {d:.5}\n", .{float_val});

    // error handling
    const result = std.fmt.parseInt(i32, "not_a_number", 10);
    if (result) |v| {
        std.debug.print("value: {}\n", .{v});
    } else |err| {
        std.debug.print("error: {}\n", .{err});
    }
}
```

### Format Width และ Alignment

```zig
const std = @import("std");

pub fn main() void {
    // width - กำหนดความกว้าง
    std.debug.print("|{d:10}|\n", .{42});      // right aligned
    std.debug.print("|{d:<10}|\n", .{42});     // left aligned
    std.debug.print("|{d:^10}|\n", .{42});     // center aligned

    // fill character
    std.debug.print("|{d:0>10}|\n", .{42});    // fill with 0
    std.debug.print("|{d:*>10}|\n", .{42});    // fill with *
    std.debug.print("|{s:-^20}|\n", .{"hello"}); // center with -

    // precision สำหรับ float
    const pi = 3.14159265358979;
    std.debug.print("{d:.2}\n", .{pi});   // 3.14
    std.debug.print("{d:.5}\n", .{pi});   // 3.14159
    std.debug.print("{d:.10}\n", .{pi});  // 3.1415926536

    // combine width and precision
    std.debug.print("{d:10.3}\n", .{pi});   // right aligned, 3 decimal places
    std.debug.print("{d:<10.3}\n", .{pi});  // left aligned
}
```

---

## Step 255: std.io - input/output overview

### std.io คืออะไร?

`std.io` จัดการ input/output ผ่าน Reader และ Writer interfaces รวมถึง stdin, stdout, stderr

### stdout และ stderr

```zig
const std = @import("std");

pub fn main() !void {
    // stdout writer
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello from stdout!\n", .{});
    try stdout.print("Number: {d}\n", .{42});

    // stderr writer
    const stderr = std.io.getStdErr().writer();
    try stderr.print("Error message to stderr\n", .{});

    // writeAll - เขียน raw bytes
    try stdout.writeAll("Raw bytes\n");

    // writeByte - เขียน byte เดียว
    try stdout.writeByte('A');
    try stdout.writeByte('\n');

    // writeByteNTimes - เขียน byte ซ้ำ
    try stdout.writeByteNTimes('=', 20);
    try stdout.writeByte('\n');
}
```

### stdin - การรับ Input

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    try stdout.print("Enter your name: ", .{});

    // อ่านบรรทัด (ไม่เกิน 100 bytes)
    var buf: [100]u8 = undefined;
    const input = try stdin.readUntilDelimiterOrEof(&buf, '\n');

    if (input) |line| {
        // trim newline บน Windows (\r\n)
        const trimmed = std.mem.trimRight(u8, line, "\r\n");
        try stdout.print("Hello, {s}!\n", .{trimmed});
    }
}
```

### BufferedReader และ BufferedWriter

```zig
const std = @import("std");

pub fn main() !void {
    // BufferedWriter - เพิ่มประสิทธิภาพการเขียน
    const stdout_file = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout_file.writer());
    const stdout = bw.writer();

    // เขียนข้อมูลหลายครั้ง (จะถูก buffer ไว้ก่อน)
    for (0..10) |i| {
        try stdout.print("Line {d}\n", .{i + 1});
    }

    // flush buffer ออกจริงๆ
    try bw.flush();
}
```

### Reader Interface

```zig
const std = @import("std");

pub fn readAllFrom(reader: anytype, allocator: std.mem.Allocator) ![]u8 {
    // อ่านทุกอย่างจาก reader
    return reader.readAllAlloc(allocator, 1024 * 1024); // max 1MB
}

pub fn processLines(reader: anytype) !void {
    var buf: [1024]u8 = undefined;
    while (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
        // ประมวลผลแต่ละบรรทัด
        std.debug.print("  > {s}\n", .{line});
    }
}

pub fn main() !void {
    // ใช้กับ string เป็น reader
    const data = "line1\nline2\nline3\n";
    var stream = std.io.fixedBufferStream(data);
    try processLines(stream.reader());
}
```

### CountingWriter

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // นับจำนวน bytes ที่เขียน
    var counting_writer = std.io.countingWriter(std.io.null_writer);
    const writer = counting_writer.writer();

    try writer.print("Hello, {s}!\n", .{"World"});
    try writer.print("Number: {d}\n", .{12345});

    std.debug.print("Total bytes written: {}\n", .{counting_writer.bytes_written});

    // ArrayList เป็น writer
    var list = std.ArrayList(u8).init(allocator);
    defer list.deinit();
    const list_writer = list.writer();

    try list_writer.print("Data: {d}, {s}\n", .{ 42, "hello" });
    std.debug.print("ArrayList content: {s}\n", .{list.items});
}
```

---

## Step 256: std.fs - file system overview

### std.fs คืออะไร?

`std.fs` จัดการระบบไฟล์ รองรับการสร้าง อ่าน เขียน และลบไฟล์และโฟลเดอร์

### การเปิดและอ่านไฟล์

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // เปิดไฟล์เพื่ออ่าน
    const file = try std.fs.cwd().openFile("test.txt", .{});
    defer file.close();

    // อ่านทั้งไฟล์
    const content = try file.readToEndAlloc(allocator, 1024 * 1024);
    defer allocator.free(content);

    std.debug.print("File content:\n{s}\n", .{content});

    // ดู file metadata
    const stat = try file.stat();
    std.debug.print("File size: {d} bytes\n", .{stat.size});
    std.debug.print("Modified: {d}\n", .{stat.mtime});
}
```

### การสร้างและเขียนไฟล์

```zig
const std = @import("std");

pub fn main() !void {
    // สร้างไฟล์ใหม่
    const file = try std.fs.cwd().createFile("output.txt", .{
        .truncate = true, // ลบเนื้อหาเดิมถ้ามี
    });
    defer file.close();

    const writer = file.writer();

    // เขียนข้อมูล
    try writer.print("Hello, File!\n", .{});
    try writer.print("Line 2: {d}\n", .{42});
    try writer.writeAll("Raw content\n");

    std.debug.print("File written successfully!\n", .{});

    // append ต่อจากไฟล์เดิม
    const append_file = try std.fs.cwd().openFile("output.txt", .{
        .mode = .write_only,
    });
    defer append_file.close();
    try append_file.seekFromEnd(0);
    try append_file.writer().print("Appended line\n", .{});
}
```

### การจัดการ Directories

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง directory
    try std.fs.cwd().makeDir("my_folder");

    // สร้าง directory แบบ recursive
    try std.fs.cwd().makePath("a/b/c/d");

    // เปิด directory
    var dir = try std.fs.cwd().openDir("my_folder", .{
        .iterate = true,
    });
    defer dir.close();

    // list ไฟล์ใน directory
    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        const kind_str = switch (entry.kind) {
            .file => "FILE",
            .directory => "DIR ",
            else => "OTHER",
        };
        std.debug.print("{s} {s}\n", .{ kind_str, entry.name });
    }

    // ลบ directory
    try std.fs.cwd().deleteDir("my_folder");
    try std.fs.cwd().deleteTree("a"); // ลบแบบ recursive
}
```

### Path Operations

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ต่อ path
    const joined = try std.fs.path.join(allocator, &[_][]const u8{
        "/home", "user", "documents", "file.txt",
    });
    defer allocator.free(joined);
    std.debug.print("joined: {s}\n", .{joined});

    // basename - ชื่อไฟล์
    const base = std.fs.path.basename("/home/user/file.txt");
    std.debug.print("basename: {s}\n", .{base});

    // dirname - directory
    const dir = std.fs.path.dirname("/home/user/file.txt");
    std.debug.print("dirname: {s}\n", .{dir orelse "none"});

    // extension - นามสกุล
    const ext = std.fs.path.extension("document.pdf");
    std.debug.print("extension: {s}\n", .{ext});

    // ตรวจสอบว่าเป็น absolute path
    std.debug.print("isAbsolute: {}\n", .{std.fs.path.isAbsolute("/home/user")});
}
```

### การทำงานกับไฟล์แบบ Atomic

```zig
const std = @import("std");

pub fn writeAtomic(path: []const u8, content: []const u8) !void {
    // เขียนไฟล์ temporary ก่อน แล้ว rename เป็นไฟล์จริง
    const tmp_path = try std.fmt.allocPrint(
        std.heap.page_allocator,
        "{s}.tmp",
        .{path},
    );
    defer std.heap.page_allocator.free(tmp_path);

    const file = try std.fs.cwd().createFile(tmp_path, .{});
    try file.writeAll(content);
    file.close();

    try std.fs.cwd().rename(tmp_path, path);
}

pub fn main() !void {
    try writeAtomic("config.txt", "key=value\nfoo=bar\n");
    std.debug.print("File written atomically\n", .{});
}
```

---

## Step 257: std.os - OS interface overview

### std.os คืออะไร?

`std.os` เป็น interface กับระบบปฏิบัติการ (OS) ระดับต่ำ ให้การเข้าถึง system calls, environment variables, signals และอื่นๆ

### Environment Variables

```zig
const std = @import("std");

pub fn main() void {
    // อ่าน environment variable
    const path = std.posix.getenv("PATH");
    if (path) |p| {
        std.debug.print("PATH = {s}\n", .{p});
    } else {
        std.debug.print("PATH not set\n", .{});
    }

    const home = std.posix.getenv("HOME");
    std.debug.print("HOME = {s}\n", .{home orelse "(not set)"});

    const user = std.posix.getenv("USER");
    std.debug.print("USER = {s}\n", .{user orelse "(not set)"});
}
```

### POSIX System Calls

```zig
const std = @import("std");

pub fn main() !void {
    // getpid - Process ID
    const pid = std.os.linux.getpid();
    std.debug.print("PID: {}\n", .{pid});

    // getuid - User ID (Linux)
    const uid = std.os.linux.getuid();
    std.debug.print("UID: {}\n", .{uid});

    // sleep - หยุดรอ nanoseconds
    const ns: u64 = 100_000_000; // 100ms
    std.time.sleep(ns);
    std.debug.print("Woke up after sleep!\n", .{});
}
```

### File Descriptors

```zig
const std = @import("std");

pub fn main() !void {
    // open file ด้วย low-level API
    const fd = try std.posix.open(
        "test.txt",
        .{ .ACCMODE = .RDONLY },
        0,
    );
    defer std.posix.close(fd);

    // อ่านด้วย read syscall
    var buf: [256]u8 = undefined;
    const n = try std.posix.read(fd, &buf);
    std.debug.print("Read {} bytes: {s}\n", .{ n, buf[0..n] });
}
```

### Signals (Unix)

```zig
const std = @import("std");

// signal handler
fn handleSigInt(sig: c_int) callconv(.C) void {
    std.debug.print("\nReceived signal {}\n", .{sig});
}

pub fn main() !void {
    // ตั้ง signal handler สำหรับ SIGINT (Ctrl+C)
    // Note: ใช้กับ Unix systems เท่านั้น
    if (@import("builtin").os.tag != .windows) {
        try std.posix.sigaction(
            std.posix.SIG.INT,
            &.{
                .handler = .{ .handler = handleSigInt },
                .mask = std.posix.empty_sigset,
                .flags = 0,
            },
            null,
        );
    }

    std.debug.print("Press Ctrl+C to test signal handler\n", .{});

    // รอสักพัก
    std.time.sleep(2 * std.time.ns_per_s);
    std.debug.print("Done waiting\n", .{});
}
```

### ข้อมูลระบบ

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // ข้อมูล OS ปัจจุบัน
    std.debug.print("OS: {s}\n", .{@tagName(builtin.os.tag)});
    std.debug.print("CPU: {s}\n", .{@tagName(builtin.cpu.arch)});

    // ขนาด page
    const page_size = std.mem.page_size;
    std.debug.print("Page size: {} bytes\n", .{page_size});

    // endianness
    std.debug.print("Endian: {s}\n", .{@tagName(builtin.cpu.arch.endian())});
}
```

---

## Step 258: std.math - math functions overview

### std.math คืออะไร?

`std.math` มีฟังก์ชันคณิตศาสตร์ทั้งพื้นฐานและขั้นสูง

### ค่าคงที่ทางคณิตศาสตร์

```zig
const std = @import("std");
const math = std.math;

pub fn main() void {
    // ค่าคงที่
    std.debug.print("pi      = {d:.10}\n", .{math.pi});
    std.debug.print("e       = {d:.10}\n", .{math.e});
    std.debug.print("sqrt2   = {d:.10}\n", .{math.sqrt2});
    std.debug.print("ln2     = {d:.10}\n", .{math.ln2});
    std.debug.print("ln10    = {d:.10}\n", .{math.ln10});
    std.debug.print("log2e   = {d:.10}\n", .{math.log2e});
    std.debug.print("log10e  = {d:.10}\n", .{math.log10e});

    // ค่าพิเศษของ float
    std.debug.print("inf     = {d}\n", .{math.inf(f64)});
    std.debug.print("nan     = {d}\n", .{math.nan(f64)});

    // ตรวจสอบค่าพิเศษ
    std.debug.print("isInf:  {}\n", .{math.isInf(math.inf(f64))});
    std.debug.print("isNan:  {}\n", .{math.isNan(math.nan(f64))});
    std.debug.print("isFinite: {}\n", .{math.isFinite(3.14)});
}
```

### ฟังก์ชันพื้นฐาน

```zig
const std = @import("std");
const math = std.math;

pub fn main() void {
    // ค่าสัมบูรณ์
    std.debug.print("abs(-5) = {d}\n", .{@abs(@as(i32, -5))});
    std.debug.print("fabs(-3.14) = {d}\n", .{@abs(@as(f64, -3.14))});

    // min และ max
    std.debug.print("min(3,7) = {}\n", .{@min(3, 7)});
    std.debug.print("max(3,7) = {}\n", .{@max(3, 7)});

    // ปัดเลข
    std.debug.print("floor(3.7) = {d}\n", .{math.floor(3.7)});
    std.debug.print("ceil(3.2)  = {d}\n", .{math.ceil(3.2)});
    std.debug.print("round(3.5) = {d}\n", .{math.round(3.5)});
    std.debug.print("trunc(3.9) = {d}\n", .{math.trunc(3.9)});

    // กำลัง
    std.debug.print("pow(2,10) = {d}\n", .{math.pow(f64, 2.0, 10.0)});
    std.debug.print("sqrt(16) = {d}\n", .{math.sqrt(16.0)});
    std.debug.print("cbrt(27) = {d}\n", .{math.cbrt(27.0)});

    // logarithm
    std.debug.print("log(e)   = {d:.10}\n", .{math.log(f64, math.e, math.e)});
    std.debug.print("log2(8)  = {d}\n", .{math.log2(8.0)});
    std.debug.print("log10(100) = {d}\n", .{math.log10(100.0)});
}
```

### ฟังก์ชัน Trigonometric

```zig
const std = @import("std");
const math = std.math;

pub fn main() void {
    const pi = math.pi;

    // Trigonometric functions
    std.debug.print("sin(π/6)   = {d:.6}\n", .{math.sin(pi / 6.0)});   // 0.5
    std.debug.print("cos(π/3)   = {d:.6}\n", .{math.cos(pi / 3.0)});   // 0.5
    std.debug.print("tan(π/4)   = {d:.6}\n", .{math.tan(pi / 4.0)});   // 1.0

    // Inverse trig
    std.debug.print("asin(0.5)  = {d:.6}\n", .{math.asin(0.5)});  // π/6
    std.debug.print("acos(0.5)  = {d:.6}\n", .{math.acos(0.5)});  // π/3
    std.debug.print("atan(1.0)  = {d:.6}\n", .{math.atan(1.0)});  // π/4

    // atan2
    std.debug.print("atan2(1,1) = {d:.6}\n", .{math.atan2(1.0, 1.0)});

    // Hyperbolic
    std.debug.print("sinh(1)    = {d:.6}\n", .{math.sinh(1.0)});
    std.debug.print("cosh(1)    = {d:.6}\n", .{math.cosh(1.0)});
    std.debug.print("tanh(1)    = {d:.6}\n", .{math.tanh(1.0)});

    // แปลงองศา
    const deg_45: f64 = 45.0;
    const rad_45 = deg_45 * pi / 180.0;
    std.debug.print("45 degrees = {d:.6} radians\n", .{rad_45});
}
```

### Integer Math

```zig
const std = @import("std");
const math = std.math;

pub fn main() !void {
    // Integer operations
    const a: i32 = 100;
    const b: i32 = 7;

    // division และ modulus
    std.debug.print("100 / 7 = {}\n", .{@divTrunc(a, b)});
    std.debug.print("100 % 7 = {}\n", .{@rem(a, b)});

    // Overflow-safe operations
    const x: u8 = 200;
    const y: u8 = 100;

    if (math.add(u8, x, y)) |sum| {
        std.debug.print("200 + 100 = {}\n", .{sum});
    } else |err| {
        std.debug.print("overflow! error: {}\n", .{err});
    }

    // Saturating arithmetic
    const sat_add = math.add(u8, x, y) catch math.maxInt(u8);
    std.debug.print("saturated: {}\n", .{sat_add});

    // Bit operations
    std.debug.print("popcount(255): {}\n", .{@popCount(@as(u8, 255))});
    std.debug.print("clz(0b00001000): {}\n", .{@clz(@as(u8, 0b00001000))});
    std.debug.print("ctz(0b00001000): {}\n", .{@ctz(@as(u8, 0b00001000))});

    // power of 2 checks
    std.debug.print("isPowerOfTwo(8): {}\n", .{math.isPowerOfTwo(8)});
    std.debug.print("isPowerOfTwo(6): {}\n", .{math.isPowerOfTwo(6)});

    // nextPowerOfTwo
    std.debug.print("nextPowerOfTwo(5): {}\n", .{math.ceilPowerOfTwo(u32, 5) catch 0});
}
```

---

## Step 259: std.process - process management

### std.process คืออะไร?

`std.process` จัดการ process-level operations เช่น command line arguments, environment variables, และการออกจากโปรแกรม

### Command Line Arguments

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // อ่าน args
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    std.debug.print("Total args: {}\n", .{args.len});
    for (args, 0..) |arg, i| {
        std.debug.print("  args[{}] = {s}\n", .{ i, arg });
    }

    // args[0] คือชื่อโปรแกรม
    if (args.len > 0) {
        std.debug.print("Program: {s}\n", .{args[0]});
    }

    // ตรวจสอบ arguments
    if (args.len < 2) {
        std.debug.print("Usage: {s} <input> <output>\n", .{args[0]});
        return;
    }

    const input_file = args[1];
    std.debug.print("Input: {s}\n", .{input_file});
}
```

### ArgIterator

```zig
const std = @import("std");

pub fn main() !void {
    // ใช้ ArgIterator แทน argsAlloc เพื่อประหยัด memory
    var args_iter = try std.process.argsWithAllocator(
        std.heap.page_allocator,
    );
    defer args_iter.deinit();

    var arg_count: u32 = 0;
    while (args_iter.next()) |arg| {
        std.debug.print("arg: {s}\n", .{arg});
        arg_count += 1;
    }
    std.debug.print("Total: {} args\n", .{arg_count});
}
```

### Environment Variables

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // อ่าน environment variables ทั้งหมด
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();

    std.debug.print("Environment variables:\n", .{});
    var env_iter = env_map.iterator();
    while (env_iter.next()) |entry| {
        std.debug.print("  {s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }

    // อ่าน env variable เฉพาะ
    if (env_map.get("HOME")) |home| {
        std.debug.print("\nHOME = {s}\n", .{home});
    }

    // ใช้ getenv แบบง่าย (posix only)
    const path = std.posix.getenv("PATH") orelse "not set";
    std.debug.print("PATH = {s}\n", .{path[0..@min(50, path.len)]});
}
```

### การออกจากโปรแกรม

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Starting program...\n", .{});

    // exit ปกติ
    // std.process.exit(0);

    // exit ด้วย error code
    // std.process.exit(1);

    // abort - terminate แบบฉุกเฉิน
    // std.process.abort();

    // raise signal
    // std.posix.raise(std.posix.SIG.TERM) catch {};

    std.debug.print("Normal exit\n", .{});
}
```

### การรัน Process อื่น

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // รันคำสั่ง shell
    const result = try std.process.Child.run(.{
        .allocator = allocator,
        .argv = &[_][]const u8{ "echo", "Hello from child process" },
    });
    defer allocator.free(result.stdout);
    defer allocator.free(result.stderr);

    std.debug.print("stdout: {s}\n", .{result.stdout});
    std.debug.print("exit code: {}\n", .{result.term.Exited});

    // รันและรับ output
    const ls_result = try std.process.Child.run(.{
        .allocator = allocator,
        .argv = &[_][]const u8{ "ls", "-la" },
    });
    defer allocator.free(ls_result.stdout);
    defer allocator.free(ls_result.stderr);

    std.debug.print("Directory listing:\n{s}\n", .{ls_result.stdout});
}
```

### Process Child แบบ Interactive

```zig
const std = @import("std");

pub fn main() !void {
    // spawn process และ interact กับมัน
    var child = std.process.Child.init(
        &[_][]const u8{ "cat", "-" },
        std.heap.page_allocator,
    );
    child.stdin_behavior = .Pipe;
    child.stdout_behavior = .Pipe;

    try child.spawn();

    // เขียน input ไปยัง child
    try child.stdin.?.writer().writeAll("Hello from parent!\n");
    child.stdin.?.close();
    child.stdin = null;

    // อ่าน output จาก child
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const output = try child.stdout.?.readToEndAlloc(gpa.allocator(), 4096);
    defer gpa.allocator().free(output);

    std.debug.print("Child output: {s}\n", .{output});

    _ = try child.wait();
}
```

---

## Step 260: std.builtin - builtin types and functions

### std.builtin คืออะไร?

`std.builtin` ให้ข้อมูลเกี่ยวกับ build target, CPU, OS และ compile-time information

### Target Information

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // OS information
    std.debug.print("OS: {s}\n", .{@tagName(builtin.os.tag)});

    // CPU architecture
    std.debug.print("CPU arch: {s}\n", .{@tagName(builtin.cpu.arch)});

    // abi
    std.debug.print("ABI: {s}\n", .{@tagName(builtin.abi)});

    // build mode
    std.debug.print("Build mode: {s}\n", .{@tagName(builtin.mode)});

    // pointer size
    std.debug.print("Pointer size: {} bits\n", .{@bitSizeOf(usize)});

    // endianness
    std.debug.print("Endianness: {s}\n", .{@tagName(builtin.cpu.arch.endian())});
}
```

### Compile-time Type Information

```zig
const std = @import("std");

pub fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("Type: {s}\n", .{@typeName(T)});
    std.debug.print("Kind: {s}\n", .{@tagName(info)});
    std.debug.print("Size: {} bytes\n", .{@sizeOf(T)});
    std.debug.print("Align: {} bytes\n", .{@alignOf(T)});
}

pub fn main() void {
    printTypeInfo(i32);
    std.debug.print("---\n", .{});
    printTypeInfo(f64);
    std.debug.print("---\n", .{});
    printTypeInfo(bool);
    std.debug.print("---\n", .{});

    // ข้อมูล struct
    const Point = struct {
        x: f32,
        y: f32,
        z: f32,
    };
    printTypeInfo(Point);
    std.debug.print("---\n", .{});

    // ข้อมูล enum
    const Color = enum { red, green, blue };
    const color_info = @typeInfo(Color).@"enum";
    std.debug.print("Enum fields: {}\n", .{color_info.fields.len});
    for (color_info.fields) |field| {
        std.debug.print("  .{s} = {}\n", .{ field.name, field.value });
    }
}
```

### Builtin Functions Overview

```zig
const std = @import("std");

pub fn main() void {
    // Type conversion
    const x: i32 = -42;
    const x_u: u32 = @bitCast(x);        // reinterpret bits
    const x_abs = @abs(x);               // absolute value
    std.debug.print("bitCast: {}\n", .{x_u});
    std.debug.print("abs: {}\n", .{x_abs});

    // Arithmetic with overflow detection
    const a: u8 = 200;
    const b: u8 = 100;
    const overflow_result = @addWithOverflow(a, b);
    std.debug.print("200 + 100 = {} (overflow: {})\n", .{
        overflow_result[0],
        overflow_result[1] == 1,
    });

    // Saturating arithmetic
    const sat_result = @addWithOverflow(a, b)[0];
    _ = sat_result;

    // Bit manipulation
    const bits: u16 = 0b0000_1111_0101_1010;
    std.debug.print("popcount: {}\n", .{@popCount(bits)});
    std.debug.print("clz: {}\n", .{@clz(bits)});
    std.debug.print("ctz: {}\n", .{@ctz(bits)});
    std.debug.print("byteSwap: 0x{X}\n", .{@byteSwap(bits)});

    // Size functions
    std.debug.print("@sizeOf(i32) = {}\n", .{@sizeOf(i32)});
    std.debug.print("@bitSizeOf(bool) = {}\n", .{@bitSizeOf(bool)});
    std.debug.print("@alignOf(f64) = {}\n", .{@alignOf(f64)});
}
```

### Compile-time Features

```zig
const std = @import("std");
const builtin = @import("builtin");

// Conditional compilation
pub fn getOsName() []const u8 {
    return switch (builtin.os.tag) {
        .linux => "Linux",
        .macos => "macOS",
        .windows => "Windows",
        .freebsd => "FreeBSD",
        else => "Unknown",
    };
}

// Platform-specific code
pub fn platformSleep(ms: u64) void {
    if (builtin.os.tag == .windows) {
        // Windows implementation
        std.time.sleep(ms * std.time.ns_per_ms);
    } else {
        // POSIX implementation
        std.time.sleep(ms * std.time.ns_per_ms);
    }
}

// Debug vs Release builds
pub fn debugOnly() void {
    if (builtin.mode == .Debug) {
        std.debug.print("[DEBUG] This only shows in debug builds\n", .{});
    }
}

pub fn main() void {
    std.debug.print("Running on: {s}\n", .{getOsName()});
    debugOnly();

    // comptime ใช้งาน @compileError สำหรับ validation
    comptime {
        if (@sizeOf(usize) < 4) {
            @compileError("Need at least 32-bit system");
        }
    }

    std.debug.print("System check passed!\n", .{});
}
```

### std.builtin Types

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn inspectType(comptime T: type) void {
    const info = @typeInfo(T);
    switch (info) {
        .int => |int_info| {
            std.debug.print("Int: {s}{d}\n", .{
                if (int_info.signedness == .signed) "i" else "u",
                int_info.bits,
            });
        },
        .float => |float_info| {
            std.debug.print("Float: f{d}\n", .{float_info.bits});
        },
        .bool => std.debug.print("Bool\n", .{}),
        .void => std.debug.print("Void\n", .{}),
        .pointer => |ptr_info| {
            std.debug.print("Pointer to: {s}\n", .{@typeName(ptr_info.child)});
        },
        .array => |arr_info| {
            std.debug.print("Array[{}] of {s}\n", .{
                arr_info.len,
                @typeName(arr_info.child),
            });
        },
        .@"struct" => |struct_info| {
            std.debug.print("Struct with {} fields\n", .{struct_info.fields.len});
        },
        .@"enum" => |enum_info| {
            std.debug.print("Enum with {} variants\n", .{enum_info.fields.len});
        },
        else => std.debug.print("Other type\n", .{}),
    }
}

pub fn main() void {
    inspectType(i32);
    inspectType(f64);
    inspectType(bool);
    inspectType(*u8);
    inspectType([10]i32);
    inspectType(struct { x: f32, y: f32 });
    inspectType(enum { a, b, c });
}
```

---

## แผนที่ Standard Library ฉบับสมบูรณ์

### โมดูลหลักและการใช้งาน

```
std.debug    → การ debug
  .print()   → แสดงข้อความไปยัง stderr
  .assert()  → ตรวจสอบเงื่อนไข
  .panic()   → หยุดโปรแกรมพร้อม error message

std.mem      → จัดการ memory
  .eql()     → เปรียบเทียบ slices
  .copy()    → copy memory
  .indexOf() → ค้นหาใน slice
  .split()   → แบ่ง slice
  .trim()    → ตัด whitespace

std.fmt      → format text
  .allocPrint()  → สร้าง string ใน heap
  .bufPrint()    → เขียนลงใน buffer
  .parseInt()    → แปลง string เป็น int
  .parseFloat()  → แปลง string เป็น float

std.io       → input/output
  .getStdOut()   → stdout writer
  .getStdIn()    → stdin reader
  .getStdErr()   → stderr writer
  .bufferedWriter() → writer พร้อม buffer

std.fs       → file system
  .cwd()         → current directory
  .openFile()    → เปิดไฟล์
  .createFile()  → สร้างไฟล์
  .path.*        → path utilities

std.os       → OS interface
  .linux.*       → Linux specific
  .windows.*     → Windows specific
  posix.*        → POSIX utilities

std.math     → mathematics
  .pi, .e        → constants
  .sin/cos/tan   → trig functions
  .log/exp       → logarithm/exponential
  .sqrt/pow      → roots and powers

std.process  → process management
  .argsAlloc()   → command line args
  .getEnvMap()   → environment vars
  .exit()        → terminate program
  .Child.*       → spawn child processes

std.builtin  → compile-time info
  .os.tag        → current OS
  .cpu.arch      → CPU architecture
  .mode          → build mode (Debug/Release)
```

### Data Structures

```
std.ArrayList(T)   → dynamic array
std.ArrayListUnmanaged(T) → ArrayList without allocator stored
std.HashMap(K,V)   → hash map
std.StringHashMap(V) → hash map with string keys
std.AutoHashMap(K,V) → hash map with auto hash/eql
std.BoundedArray(T, N) → array with fixed capacity
std.MultiArrayList(S) → structure of arrays
std.SinglyLinkedList(T) → linked list
std.DoublyLinkedList(T) → doubly linked list
std.PriorityQueue(T) → priority queue
std.RingBuffer    → circular buffer
```

### Allocators

```
std.heap.GeneralPurposeAllocator(.{})  → dev/debug allocator
std.heap.c_allocator                   → malloc/free
std.heap.page_allocator                → OS pages
std.heap.FixedBufferAllocator          → fixed buffer
std.heap.ArenaAllocator                → arena allocator
std.testing.allocator                  → test allocator (leak detection)
```

---

## วิธีการอ่าน Zig Documentation

### 1. Official Documentation

```
https://ziglang.org/documentation/master/
https://ziglang.org/documentation/master/std/
```

### 2. Source Code

```bash
# หา path ของ std
zig env | grep lib_dir

# เปิด std library source
ls $(zig env | grep lib_dir | cut -d'"' -f2)/std/
```

### 3. zig-autodoc

Zig สร้าง documentation อัตโนมัติจาก source code คุณสามารถ host เองได้:

```bash
# generate docs
zig build-lib -femit-docs src/main.zig

# หรือใช้ zig build พร้อม docs step
zig build docs
```

### 4. ใช้ LSP (Language Server Protocol)

```bash
# ติดตั้ง zls
# จากนั้น editor จะแสดง documentation แบบ hover
```

### 5. Examples จาก tests ใน std source

ไฟล์ test ใน std เป็นตัวอย่างที่ดีที่สุด:

```bash
# ดูตัวอย่างใน std tests
cat $(zig env | grep lib_dir)/std/mem.zig | grep "test "
```

---

## ตัวอย่างโปรแกรมที่ใช้ std หลายโมดูล

### โปรแกรมนับคำใน File

```zig
const std = @import("std");

const WordCount = struct {
    words: u32,
    lines: u32,
    chars: u32,
    bytes: u32,
};

pub fn countWords(content: []const u8) WordCount {
    var count = WordCount{ .words = 0, .lines = 0, .chars = 0, .bytes = 0 };

    count.bytes = @intCast(content.len);

    var in_word = false;
    var iter = std.unicode.Utf8Iterator{ .bytes = content, .i = 0 };

    while (iter.nextCodepoint()) |cp| {
        count.chars += 1;

        if (cp == '\n') count.lines += 1;

        const is_space = cp == ' ' or cp == '\t' or cp == '\n' or cp == '\r';
        if (!is_space and !in_word) {
            count.words += 1;
            in_word = true;
        } else if (is_space) {
            in_word = false;
        }
    }

    return count;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 2) {
        const stderr = std.io.getStdErr().writer();
        try stderr.print("Usage: {s} <file>\n", .{args[0]});
        std.process.exit(1);
    }

    const file = try std.fs.cwd().openFile(args[1], .{});
    defer file.close();

    const content = try file.readToEndAlloc(allocator, 10 * 1024 * 1024);
    defer allocator.free(content);

    const result = countWords(content);
    const stdout = std.io.getStdOut().writer();
    try stdout.print("File: {s}\n", .{args[1]});
    try stdout.print("Words: {}\n", .{result.words});
    try stdout.print("Lines: {}\n", .{result.lines});
    try stdout.print("Chars: {}\n", .{result.chars});
    try stdout.print("Bytes: {}\n", .{result.bytes});
}
```

### โปรแกรม CSV Parser

```zig
const std = @import("std");

pub fn parseCSV(
    allocator: std.mem.Allocator,
    content: []const u8,
) ![][]std.ArrayList([]const u8) {
    var rows = std.ArrayList(std.ArrayList([]const u8)).init(allocator);

    var line_iter = std.mem.splitScalar(u8, content, '\n');
    while (line_iter.next()) |line| {
        const trimmed = std.mem.trimRight(u8, line, "\r");
        if (trimmed.len == 0) continue;

        var row = std.ArrayList([]const u8).init(allocator);
        var field_iter = std.mem.splitScalar(u8, trimmed, ',');
        while (field_iter.next()) |field| {
            try row.append(std.mem.trim(u8, field, " "));
        }
        try rows.append(row);
    }

    return rows.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const csv_data =
        \\Name, Age, City
        \\Alice, 30, Bangkok
        \\Bob, 25, Chiang Mai
        \\Charlie, 35, Phuket
    ;

    const rows = try parseCSV(allocator, csv_data);
    defer {
        for (rows) |*row| row.deinit();
        allocator.free(rows);
    }

    const stdout = std.io.getStdOut().writer();

    for (rows, 0..) |row, row_idx| {
        try stdout.print("Row {}: ", .{row_idx});
        for (row.items, 0..) |field, col_idx| {
            if (col_idx > 0) try stdout.writeAll(", ");
            try stdout.print("[{s}]", .{field});
        }
        try stdout.writeByte('\n');
    }
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Math Calculator

สร้าง calculator ที่รับ expression จาก command line:

```
zig run calculator.zig -- 3.14 * 2
zig run calculator.zig -- 100 / 7
```

**คำใบ้**: ใช้ `std.process.argsAlloc`, `std.fmt.parseFloat`, และ `std.math`

### แบบฝึกหัดที่ 2: File Statistics

สร้างโปรแกรมที่แสดงสถิติของไฟล์:
- ขนาดไฟล์
- จำนวนบรรทัด
- ตัวอักษรที่พบบ่อยที่สุด

**คำใบ้**: ใช้ `std.fs`, `std.io`, และ `std.mem`

### แบบฝึกหัดที่ 3: Environment Explorer

สร้างโปรแกรมที่แสดงข้อมูล environment:
- Environment variables ทั้งหมด
- CPU และ OS information
- Build mode

**คำใบ้**: ใช้ `std.process.getEnvMap`, `std.builtin`, `@import("builtin")`

### แบบฝึกหัดที่ 4: String Transformer

สร้างโปรแกรมที่รับ string และ transform:
- UPPER case
- lower case  
- Title Case
- reverse

```bash
zig run transformer.zig -- upper "hello world"
zig run transformer.zig -- reverse "hello"
```

**คำใบ้**: ใช้ `std.ascii`, `std.mem`, `std.fmt`

### แบบฝึกหัดที่ 5: Process Monitor

สร้างโปรแกรมที่รันคำสั่งและแสดง:
- stdout output
- stderr output
- exit code
- เวลาที่ใช้

**คำใบ้**: ใช้ `std.process.Child`, `std.time`

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Zig Standard Library อย่างครอบคลุม:

| โมดูล | ความสามารถหลัก | ใช้งานเมื่อ |
|-------|---------------|------------|
| `std.debug` | print, assert, panic, log | debugging และ development |
| `std.mem` | copy, compare, search, allocators | จัดการ memory และ slices |
| `std.fmt` | format, parse numbers | แปลงข้อมูลเป็น/จาก string |
| `std.io` | Reader, Writer, stdin/stdout | รับส่งข้อมูล |
| `std.fs` | files, directories, paths | จัดการไฟล์ |
| `std.os` | system calls, env vars | ติดต่อกับ OS |
| `std.math` | trig, log, exp, constants | คณิตศาสตร์ |
| `std.process` | args, env, exit, spawn | จัดการ process |
| `std.builtin` | type info, platform info | compile-time information |

### หลักการสำคัญ

1. **import เฉพาะที่ต้องใช้**: ใช้ `const X = std.X` เพื่อความกระชับ
2. **ตรวจสอบ errors เสมอ**: std functions ส่วนใหญ่ return errors
3. **ใช้ Allocators อย่างเหมาะสม**: เลือก allocator ให้เหมาะกับ use case
4. **อ่าน source code**: std library source code เป็นตัวอย่างที่ดีที่สุด
5. **ใช้ defer สำหรับ cleanup**: ป้องกัน resource leaks

### ขั้นตอนถัดไป

ใน Part 27 เราจะเจาะลึกโมดูล **std.testing** และการเขียน unit tests ใน Zig อย่างมืออาชีพ

---

*Part 26 จบแล้ว - ยินดีด้วย! คุณได้เรียนรู้ภาพรวมของ Zig Standard Library ครบทั้ง 10 โมดูลหลักแล้ว*
