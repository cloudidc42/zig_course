# ตอนที่ 27: std.mem ใน Zig Standard Library

## ขั้นตอนที่ 261-270: การจัดการหน่วยความจำและ String ด้วย std.mem

ใน Zig Standard Library โมดูล `std.mem` เป็นหัวใจสำคัญของการจัดการหน่วยความจำและการประมวลผล slice รวมถึง string โมดูลนี้มีฟังก์ชันที่ครอบคลุมทุกความต้องการตั้งแต่การเปรียบเทียบ การคัดลอก การค้นหา ไปจนถึงการเรียงลำดับ

---

## ขั้นตอนที่ 261: std.mem.eql - การเปรียบเทียบ

### แนวคิดพื้นฐาน

`std.mem.eql` เป็นฟังก์ชันสำหรับเปรียบเทียบ slice สองตัวว่ามีเนื้อหาเหมือนกันหรือไม่ ซึ่งแตกต่างจากการเปรียบเทียบด้วย `==` ที่จะเปรียบเทียบ pointer ไม่ใช่เนื้อหา

### นิยาม (Signature)

```zig
pub fn eql(comptime T: type, a: []const T, b: []const T) bool
```

- `T` คือชนิดของข้อมูลใน slice (เช่น `u8` สำหรับ string)
- `a` และ `b` คือ slice ที่ต้องการเปรียบเทียบ
- คืนค่า `true` ถ้าทั้งสองมีขนาดเท่ากันและทุก element เหมือนกัน

### ตัวอย่างที่ 261.1: เปรียบเทียบ String พื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const str1 = "สวัสดี";
    const str2 = "สวัสดี";
    const str3 = "hello";

    // เปรียบเทียบ string สองตัว
    if (std.mem.eql(u8, str1, str2)) {
        std.debug.print("str1 และ str2 เหมือนกัน\n", .{});
    } else {
        std.debug.print("str1 และ str2 ต่างกัน\n", .{});
    }

    if (std.mem.eql(u8, str1, str3)) {
        std.debug.print("str1 และ str3 เหมือนกัน\n", .{});
    } else {
        std.debug.print("str1 และ str3 ต่างกัน\n", .{});
    }
}
```

**ผลลัพธ์:**
```
str1 และ str2 เหมือนกัน
str1 และ str3 ต่างกัน
```

### ตัวอย่างที่ 261.2: เปรียบเทียบ Array และ Slice ของตัวเลข

```zig
const std = @import("std");

pub fn main() void {
    const arr1 = [_]i32{ 1, 2, 3, 4, 5 };
    const arr2 = [_]i32{ 1, 2, 3, 4, 5 };
    const arr3 = [_]i32{ 1, 2, 3, 4, 6 };

    // เปรียบเทียบ slice ของ i32
    if (std.mem.eql(i32, &arr1, &arr2)) {
        std.debug.print("arr1 และ arr2 เหมือนกันทุก element\n", .{});
    }

    if (!std.mem.eql(i32, &arr1, &arr3)) {
        std.debug.print("arr1 และ arr3 ต่างกัน\n", .{});
    }

    // ทดสอบกับ slice บางส่วน
    const slice1 = arr1[0..3]; // { 1, 2, 3 }
    const slice2 = arr2[0..3]; // { 1, 2, 3 }
    const slice3 = arr3[0..3]; // { 1, 2, 3 }

    std.debug.print("slice1 == slice2: {}\n", .{std.mem.eql(i32, slice1, slice2)});
    std.debug.print("slice1 == slice3: {}\n", .{std.mem.eql(i32, slice1, slice3)});
}
```

**ผลลัพธ์:**
```
arr1 และ arr2 เหมือนกันทุก element
arr1 และ arr3 ต่างกัน
slice1 == slice2: true
slice1 == slice3: true
```

### ตัวอย่างที่ 261.3: การใช้ eql ในการตรวจสอบ Command

```zig
const std = @import("std");

fn processCommand(cmd: []const u8) void {
    if (std.mem.eql(u8, cmd, "start")) {
        std.debug.print("เริ่มระบบ...\n", .{});
    } else if (std.mem.eql(u8, cmd, "stop")) {
        std.debug.print("หยุดระบบ...\n", .{});
    } else if (std.mem.eql(u8, cmd, "status")) {
        std.debug.print("ระบบทำงานปกติ\n", .{});
    } else {
        std.debug.print("คำสั่งไม่รู้จัก: {s}\n", .{cmd});
    }
}

pub fn main() void {
    processCommand("start");
    processCommand("status");
    processCommand("stop");
    processCommand("restart");
}
```

**ผลลัพธ์:**
```
เริ่มระบบ...
ระบบทำงานปกติ
หยุดระบบ...
คำสั่งไม่รู้จัก: restart
```

### จุดสำคัญของ std.mem.eql

1. **Type Safety**: ต้องระบุชนิดข้อมูล `T` ให้ถูกต้อง
2. **ขนาดต้องเท่ากัน**: ถ้าขนาดต่างกัน จะคืนค่า `false` ทันที ไม่ต้องเปรียบเทียบ element
3. **ใช้กับ Unicode ได้**: เนื่องจาก UTF-8 ใน Zig เป็นแค่ `[]const u8`
4. **Performance**: การตรวจสอบขนาดก่อนทำให้เร็วในกรณีที่ขนาดต่างกัน

---

## ขั้นตอนที่ 262: std.mem.copy และ @memcpy

### แนวคิดพื้นฐาน

การคัดลอกข้อมูลใน Zig มีสองแบบหลัก:
- `std.mem.copyForwards` / `std.mem.copyBackwards`: สำหรับ slice ที่อาจ overlap กัน
- `@memcpy`: builtin ที่เร็วกว่าสำหรับกรณีที่ไม่ overlap

### นิยาม (Signature)

```zig
// คัดลอกจาก source ไป destination (จากหน้าไปหลัง)
pub fn copyForwards(comptime T: type, dest: []T, source: []const T) void

// คัดลอกจาก source ไป destination (จากหลังไปหน้า - ป้องกัน overlap)
pub fn copyBackwards(comptime T: type, dest: []T, source: []const T) void

// Builtin สำหรับคัดลอกที่ไม่ overlap
@memcpy(dest: anytype, source: anytype) void
```

### ตัวอย่างที่ 262.1: การใช้ copyForwards พื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const source = [_]u8{ 72, 101, 108, 108, 111 }; // "Hello"
    var dest: [5]u8 = undefined;

    // คัดลอกข้อมูลจาก source ไป dest
    std.mem.copyForwards(u8, &dest, &source);

    std.debug.print("source: {s}\n", .{source});
    std.debug.print("dest:   {s}\n", .{dest});
    std.debug.print("เหมือนกัน: {}\n", .{std.mem.eql(u8, &dest, &source)});
}
```

**ผลลัพธ์:**
```
source: Hello
dest:   Hello
เหมือนกัน: true
```

### ตัวอย่างที่ 262.2: การใช้ @memcpy

```zig
const std = @import("std");

pub fn main() void {
    const message = "Zig Programming";
    var buffer: [15]u8 = undefined;

    // @memcpy ต้องการให้ dest และ source ไม่ overlap กัน
    @memcpy(&buffer, message);

    std.debug.print("Original: {s}\n", .{message});
    std.debug.print("Copied:   {s}\n", .{buffer});
}
```

**ผลลัพธ์:**
```
Original: Zig Programming
Copied:   Zig Programming
```

### ตัวอย่างที่ 262.3: copyBackwards สำหรับ Overlapping Memory

```zig
const std = @import("std");

pub fn main() void {
    var data = [_]u8{ 1, 2, 3, 4, 5, 0, 0, 0 };

    std.debug.print("ก่อน: ", .{});
    for (data) |byte| {
        std.debug.print("{d} ", .{byte});
    }
    std.debug.print("\n", .{});

    // ย้ายข้อมูลไปข้างหน้า 3 ตำแหน่ง (overlap)
    // ต้องใช้ copyBackwards เพื่อป้องกันการเขียนทับก่อนอ่าน
    std.mem.copyBackwards(u8, data[3..8], data[0..5]);

    std.debug.print("หลัง:  ", .{});
    for (data) |byte| {
        std.debug.print("{d} ", .{byte});
    }
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
ก่อน: 1 2 3 4 5 0 0 0
หลัง:  1 2 3 1 2 3 4 5
```

### ตัวอย่างที่ 262.4: การสร้าง String Copy Function

```zig
const std = @import("std");

fn duplicateString(allocator: std.mem.Allocator, original: []const u8) ![]u8 {
    const copy = try allocator.alloc(u8, original.len);
    @memcpy(copy, original);
    return copy;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const original = "Hello, Zig World!";
    const copy = try duplicateString(allocator, original);
    defer allocator.free(copy);

    std.debug.print("Original: {s}\n", .{original});
    std.debug.print("Copy:     {s}\n", .{copy});
    std.debug.print("Pointer เดียวกัน: {}\n", .{original.ptr == copy.ptr});
    std.debug.print("เนื้อหาเหมือนกัน: {}\n", .{std.mem.eql(u8, original, copy)});
}
```

**ผลลัพธ์:**
```
Original: Hello, Zig World!
Copy:     Hello, Zig World!
Pointer เดียวกัน: false
เนื้อหาเหมือนกัน: true
```

### ความแตกต่างระหว่าง copy และ @memcpy

| คุณสมบัติ | copyForwards/Backwards | @memcpy |
|-----------|------------------------|---------|
| Overlap | รองรับ | ไม่รองรับ |
| Speed | ปานกลาง | เร็วมาก |
| Type Check | Compile-time | Compile-time |
| ใช้บ่อยในกรณี | Memory ซ้อนทับกัน | Memory แยกกัน |

---

## ขั้นตอนที่ 263: std.mem.set และ @memset

### แนวคิดพื้นฐาน

`std.mem.set` และ `@memset` ใช้สำหรับเติมข้อมูลใน slice ด้วยค่าเดียวกันทุก element ซึ่งมีประโยชน์มากในการ initialize หน่วยความจำ

### นิยาม (Signature)

```zig
// เติมทุก element ใน slice ด้วยค่า value
pub fn set(comptime T: type, dest: []T, value: T) void

// Builtin สำหรับเติม byte
@memset(dest: anytype, value: u8) void
```

### ตัวอย่างที่ 263.1: การ Initialize Buffer ด้วย Zero

```zig
const std = @import("std");

pub fn main() void {
    var buffer: [10]u8 = undefined;

    // เติม 0 ทั้ง buffer
    @memset(&buffer, 0);

    std.debug.print("Buffer หลัง memset(0): ", .{});
    for (buffer) |byte| {
        std.debug.print("{d} ", .{byte});
    }
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
Buffer หลัง memset(0): 0 0 0 0 0 0 0 0 0 0
```

### ตัวอย่างที่ 263.2: การใช้ std.mem.set กับชนิดข้อมูลต่างๆ

```zig
const std = @import("std");

pub fn main() void {
    // เติม array ของ i32 ด้วยค่า -1
    var int_array: [5]i32 = undefined;
    std.mem.set(i32, &int_array, -1);

    std.debug.print("i32 array: ", .{});
    for (int_array) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // เติม array ของ bool ด้วยค่า false
    var bool_array: [6]bool = undefined;
    std.mem.set(bool, &bool_array, false);

    std.debug.print("bool array: ", .{});
    for (bool_array) |val| {
        std.debug.print("{} ", .{val});
    }
    std.debug.print("\n", .{});

    // เติม array ของ f32 ด้วยค่า 3.14
    var float_array: [4]f32 = undefined;
    std.mem.set(f32, &float_array, 3.14);

    std.debug.print("f32 array: ", .{});
    for (float_array) |val| {
        std.debug.print("{d:.2} ", .{val});
    }
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
i32 array: -1 -1 -1 -1 -1
bool array: false false false false false false
f32 array: 3.14 3.14 3.14 3.14
```

### ตัวอย่างที่ 263.3: การใช้ memset ในการสร้าง Pattern

```zig
const std = @import("std");

pub fn main() void {
    var screen: [5][20]u8 = undefined;

    // เติม border ด้วย '*' และ พื้นที่ว่างด้วย ' '
    for (&screen, 0..) |*row, i| {
        if (i == 0 or i == 4) {
            // แถวบนและล่าง เติมด้วย '*'
            @memset(row, '*');
        } else {
            // แถวกลาง เติมด้วย ' ' แล้วใส่ '*' ที่ขอบ
            @memset(row, ' ');
            row[0] = '*';
            row[19] = '*';
        }
    }

    // พิมพ์ผลลัพธ์
    for (screen) |row| {
        std.debug.print("{s}\n", .{row});
    }
}
```

**ผลลัพธ์:**
```
********************
*                  *
*                  *
*                  *
********************
```

### ตัวอย่างที่ 263.4: การ Clear Sensitive Data

```zig
const std = @import("std");

pub fn main() void {
    // สถานการณ์จริง: ล้างข้อมูลที่ sensitive หลังใช้งาน
    var password: [32]u8 = undefined;

    // จำลองการเก็บ password
    const pwd = "MySecretPassword123!";
    @memcpy(password[0..pwd.len], pwd);

    std.debug.print("Password ก่อนล้าง: {s}\n", .{password[0..pwd.len]});

    // ล้าง password ด้วย zero
    @memset(&password, 0);

    std.debug.print("Password หลังล้าง: {d}\n", .{password[0]});
    std.debug.print("ข้อมูลถูกล้างแล้ว\n", .{});
}
```

**ผลลัพธ์:**
```
Password ก่อนล้าง: MySecretPassword123!
Password หลังล้าง: 0
ข้อมูลถูกล้างแล้ว
```

---

## ขั้นตอนที่ 264: std.mem.indexOf และ lastIndexOf

### แนวคิดพื้นฐาน

`std.mem.indexOf` ค้นหาตำแหน่งแรกที่พบ subsequence ใน slice ส่วน `lastIndexOf` ค้นหาตำแหน่งสุดท้าย ทั้งสองคืนค่า `?usize` (optional) เนื่องจากอาจไม่พบ

### นิยาม (Signature)

```zig
// ค้นหาตำแหน่งแรกของ needle ใน haystack
pub fn indexOf(comptime T: type, haystack: []const T, needle: []const T) ?usize

// ค้นหาตำแหน่งสุดท้ายของ needle ใน haystack
pub fn lastIndexOf(comptime T: type, haystack: []const T, needle: []const T) ?usize

// ค้นหา element เดียว (scalar)
pub fn indexOfScalar(comptime T: type, slice: []const T, value: T) ?usize
pub fn lastIndexOfScalar(comptime T: type, slice: []const T, value: T) ?usize
```

### ตัวอย่างที่ 264.1: การค้นหา String พื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const text = "Hello, World! Hello, Zig!";

    // ค้นหาคำว่า "Hello"
    if (std.mem.indexOf(u8, text, "Hello")) |pos| {
        std.debug.print("พบ 'Hello' ครั้งแรกที่ตำแหน่ง: {d}\n", .{pos});
    } else {
        std.debug.print("ไม่พบ 'Hello'\n", .{});
    }

    // ค้นหา "Hello" ครั้งสุดท้าย
    if (std.mem.lastIndexOf(u8, text, "Hello")) |pos| {
        std.debug.print("พบ 'Hello' ครั้งสุดท้ายที่ตำแหน่ง: {d}\n", .{pos});
    }

    // ค้นหาสิ่งที่ไม่มี
    if (std.mem.indexOf(u8, text, "Python")) |pos| {
        std.debug.print("พบ 'Python' ที่ตำแหน่ง: {d}\n", .{pos});
    } else {
        std.debug.print("ไม่พบ 'Python' ในข้อความ\n", .{});
    }
}
```

**ผลลัพธ์:**
```
พบ 'Hello' ครั้งแรกที่ตำแหน่ง: 0
พบ 'Hello' ครั้งสุดท้ายที่ตำแหน่ง: 14
ไม่พบ 'Python' ในข้อความ
```

### ตัวอย่างที่ 264.2: การใช้ indexOfScalar

```zig
const std = @import("std");

pub fn main() void {
    const numbers = [_]i32{ 10, 20, 30, 40, 50, 30, 60 };
    const text = "Hello, World!";

    // ค้นหาตัวเลข 30 ตัวแรก
    if (std.mem.indexOfScalar(i32, &numbers, 30)) |pos| {
        std.debug.print("พบ 30 ครั้งแรกที่ index: {d}\n", .{pos});
    }

    // ค้นหาตัวเลข 30 ตัวสุดท้าย
    if (std.mem.lastIndexOfScalar(i32, &numbers, 30)) |pos| {
        std.debug.print("พบ 30 ครั้งสุดท้ายที่ index: {d}\n", .{pos});
    }

    // ค้นหา character ใน string
    if (std.mem.indexOfScalar(u8, text, ',')) |pos| {
        std.debug.print("พบ ',' ที่ตำแหน่ง: {d}\n", .{pos});
    }

    // ค้นหา character ที่ไม่มี
    const result = std.mem.indexOfScalar(u8, text, 'x');
    std.debug.print("ค้นหา 'x': {?}\n", .{result});
}
```

**ผลลัพธ์:**
```
พบ 30 ครั้งแรกที่ index: 2
พบ 30 ครั้งสุดท้ายที่ index: 5
พบ ',' ที่ตำแหน่ง: 5
ค้นหา 'x': null
```

### ตัวอย่างที่ 264.3: การนับความถี่ของ Pattern

```zig
const std = @import("std");

fn countOccurrences(haystack: []const u8, needle: []const u8) usize {
    var count: usize = 0;
    var start: usize = 0;

    while (start < haystack.len) {
        if (std.mem.indexOf(u8, haystack[start..], needle)) |pos| {
            count += 1;
            start += pos + needle.len;
        } else {
            break;
        }
    }

    return count;
}

pub fn main() void {
    const text = "the quick brown fox jumps over the lazy dog near the river";
    const word = "the";

    const count = countOccurrences(text, word);
    std.debug.print("พบคำว่า '{s}' จำนวน {d} ครั้งในข้อความ\n", .{ word, count });
}
```

**ผลลัพธ์:**
```
พบคำว่า 'the' จำนวน 3 ครั้งในข้อความ
```

### ตัวอย่างที่ 264.4: Parser อย่างง่าย

```zig
const std = @import("std");

fn parseKeyValue(line: []const u8) ?struct { key: []const u8, value: []const u8 } {
    const eq_pos = std.mem.indexOfScalar(u8, line, '=') orelse return null;
    const key = line[0..eq_pos];
    const value = line[eq_pos + 1 ..];
    return .{ .key = key, .value = value };
}

pub fn main() void {
    const lines = [_][]const u8{
        "name=Zig",
        "version=0.12",
        "author=Andrew Kelley",
        "invalid_line",
    };

    for (lines) |line| {
        if (parseKeyValue(line)) |kv| {
            std.debug.print("key='{s}', value='{s}'\n", .{ kv.key, kv.value });
        } else {
            std.debug.print("ไม่สามารถ parse: '{s}'\n", .{line});
        }
    }
}
```

**ผลลัพธ์:**
```
key='name', value='Zig'
key='version', value='0.12'
key='author', value='Andrew Kelley'
ไม่สามารถ parse: 'invalid_line'
```

---

## ขั้นตอนที่ 265: std.mem.startsWith และ endsWith

### แนวคิดพื้นฐาน

`startsWith` ตรวจสอบว่า slice เริ่มต้นด้วย prefix ที่กำหนด ส่วน `endsWith` ตรวจสอบว่าจบด้วย suffix ที่กำหนด ทั้งสองคืนค่า `bool`

### นิยาม (Signature)

```zig
pub fn startsWith(comptime T: type, haystack: []const T, needle: []const T) bool
pub fn endsWith(comptime T: type, haystack: []const T, needle: []const T) bool
```

### ตัวอย่างที่ 265.1: ตรวจสอบ File Extension

```zig
const std = @import("std");

fn getFileType(filename: []const u8) []const u8 {
    if (std.mem.endsWith(u8, filename, ".zig")) {
        return "Zig source file";
    } else if (std.mem.endsWith(u8, filename, ".c")) {
        return "C source file";
    } else if (std.mem.endsWith(u8, filename, ".rs")) {
        return "Rust source file";
    } else if (std.mem.endsWith(u8, filename, ".py")) {
        return "Python script";
    } else if (std.mem.endsWith(u8, filename, ".md")) {
        return "Markdown document";
    } else {
        return "Unknown file type";
    }
}

pub fn main() void {
    const files = [_][]const u8{
        "main.zig",
        "hello.c",
        "server.rs",
        "script.py",
        "README.md",
        "Makefile",
    };

    for (files) |file| {
        std.debug.print("{s}: {s}\n", .{ file, getFileType(file) });
    }
}
```

**ผลลัพธ์:**
```
main.zig: Zig source file
hello.c: C source file
server.rs: Rust source file
script.py: Python script
README.md: Markdown document
Makefile: Unknown file type
```

### ตัวอย่างที่ 265.2: ตรวจสอบ URL Protocol

```zig
const std = @import("std");

fn getProtocol(url: []const u8) []const u8 {
    if (std.mem.startsWith(u8, url, "https://")) {
        return "HTTPS (Secure)";
    } else if (std.mem.startsWith(u8, url, "http://")) {
        return "HTTP (Insecure)";
    } else if (std.mem.startsWith(u8, url, "ftp://")) {
        return "FTP";
    } else if (std.mem.startsWith(u8, url, "file://")) {
        return "Local File";
    } else {
        return "Unknown Protocol";
    }
}

pub fn main() void {
    const urls = [_][]const u8{
        "https://ziglang.org",
        "http://example.com",
        "ftp://files.server.com",
        "file:///home/user/doc.txt",
        "ssh://user@host.com",
    };

    for (urls) |url| {
        std.debug.print("{s}\n  -> Protocol: {s}\n", .{ url, getProtocol(url) });
    }
}
```

**ผลลัพธ์:**
```
https://ziglang.org
  -> Protocol: HTTPS (Secure)
http://example.com
  -> Protocol: HTTP (Insecure)
ftp://files.server.com
  -> Protocol: FTP
file:///home/user/doc.txt
  -> Protocol: Local File
ssh://user@host.com
  -> Protocol: Unknown Protocol
```

### ตัวอย่างที่ 265.3: การ Strip Prefix และ Suffix

```zig
const std = @import("std");

fn stripPrefix(s: []const u8, prefix: []const u8) []const u8 {
    if (std.mem.startsWith(u8, s, prefix)) {
        return s[prefix.len..];
    }
    return s;
}

fn stripSuffix(s: []const u8, suffix: []const u8) []const u8 {
    if (std.mem.endsWith(u8, s, suffix)) {
        return s[0 .. s.len - suffix.len];
    }
    return s;
}

pub fn main() void {
    const commands = [_][]const u8{
        "CMD_start",
        "CMD_stop",
        "CMD_status",
        "run",
    };

    for (commands) |cmd| {
        const stripped = stripPrefix(cmd, "CMD_");
        std.debug.print("'{s}' -> '{s}'\n", .{ cmd, stripped });
    }

    std.debug.print("\n", .{});

    const filenames = [_][]const u8{
        "document.txt",
        "image.png",
        "script.sh",
        "Makefile",
    };

    for (filenames) |name| {
        const no_ext = if (std.mem.lastIndexOfScalar(u8, name, '.')) |dot|
            name[0..dot]
        else
            name;
        std.debug.print("'{s}' -> '{s}'\n", .{ name, no_ext });
    }
}
```

**ผลลัพธ์:**
```
'CMD_start' -> 'start'
'CMD_stop' -> 'stop'
'CMD_status' -> 'status'
'run' -> 'run'

'document.txt' -> 'document'
'image.png' -> 'image'
'script.sh' -> 'script'
'Makefile' -> 'Makefile'
```

---

## ขั้นตอนที่ 266: std.mem.trim, trimLeft, trimRight

### แนวคิดพื้นฐาน

ฟังก์ชัน trim ใช้สำหรับตัด character ที่ไม่ต้องการออกจากขอบของ slice โดย:
- `trim`: ตัดทั้งสองด้าน
- `trimLeft`: ตัดด้านซ้าย (ต้น)
- `trimRight`: ตัดด้านขวา (ท้าย)

### นิยาม (Signature)

```zig
pub fn trim(comptime T: type, slice: []const T, values_to_strip: []const T) []const T
pub fn trimLeft(comptime T: type, slice: []const T, values_to_strip: []const T) []const T
pub fn trimRight(comptime T: type, slice: []const T, values_to_strip: []const T) []const T
```

**หมายเหตุ**: `values_to_strip` คือ set ของ character ที่ต้องการตัด ไม่ใช่ substring

### ตัวอย่างที่ 266.1: ตัด Whitespace

```zig
const std = @import("std");

pub fn main() void {
    const messy = "   Hello, World!   \n\t";

    const trimmed = std.mem.trim(u8, messy, " \n\t\r");
    const trimmed_left = std.mem.trimLeft(u8, messy, " \n\t\r");
    const trimmed_right = std.mem.trimRight(u8, messy, " \n\t\r");

    std.debug.print("Original:     '{s}'\n", .{messy});
    std.debug.print("trim:         '{s}'\n", .{trimmed});
    std.debug.print("trimLeft:     '{s}'\n", .{trimmed_left});
    std.debug.print("trimRight:    '{s}'\n", .{trimmed_right});
}
```

**ผลลัพธ์:**
```
Original:     '   Hello, World!   
	'
trim:         'Hello, World!'
trimLeft:     'Hello, World!   
	'
trimRight:    '   Hello, World!'
```

### ตัวอย่างที่ 266.2: ตัด Custom Characters

```zig
const std = @import("std");

pub fn main() void {
    // ตัดเครื่องหมายวรรคตอนออกจากขอบคำ
    const punctuated = "...Hello, World!!!";
    const cleaned = std.mem.trim(u8, punctuated, ".,!?;:");

    std.debug.print("ก่อน: '{s}'\n", .{punctuated});
    std.debug.print("หลัง: '{s}'\n", .{cleaned});

    // ตัดตัวเลขออกจากขอบ
    const numbered = "123abc456";
    const letters_only = std.mem.trim(u8, numbered, "0123456789");

    std.debug.print("\nก่อน: '{s}'\n", .{numbered});
    std.debug.print("หลัง: '{s}'\n", .{letters_only});

    // ตัด quotes ออก
    const quoted = "\"Hello, World!\"";
    const unquoted = std.mem.trim(u8, quoted, "\"'");

    std.debug.print("\nก่อน: {s}\n", .{quoted});
    std.debug.print("หลัง: {s}\n", .{unquoted});
}
```

**ผลลัพธ์:**
```
ก่อน: '...Hello, World!!!'
หลัง: 'Hello, World'

ก่อน: '123abc456'
หลัง: 'abc'

ก่อน: "Hello, World!"
หลัง: Hello, World!
```

### ตัวอย่างที่ 266.3: การ Parse Config File

```zig
const std = @import("std");

fn parseConfigLine(line: []const u8) ?struct { key: []const u8, value: []const u8 } {
    // ตัด whitespace รอบๆ
    const trimmed = std.mem.trim(u8, line, " \t");

    // ข้ามบรรทัด comment และบรรทัดว่าง
    if (trimmed.len == 0 or trimmed[0] == '#') return null;

    // ค้นหา '='
    const eq_pos = std.mem.indexOfScalar(u8, trimmed, '=') orelse return null;

    // ตัด whitespace รอบๆ key และ value
    const key = std.mem.trim(u8, trimmed[0..eq_pos], " \t");
    const value = std.mem.trim(u8, trimmed[eq_pos + 1 ..], " \t");

    return .{ .key = key, .value = value };
}

pub fn main() void {
    const config_lines = [_][]const u8{
        "# This is a comment",
        "name = My Application",
        "version   =   1.0.0",
        "  debug  =  true  ",
        "",
        "port=8080",
    };

    std.debug.print("Parsed config:\n", .{});
    for (config_lines) |line| {
        if (parseConfigLine(line)) |kv| {
            std.debug.print("  {s} => {s}\n", .{ kv.key, kv.value });
        }
    }
}
```

**ผลลัพธ์:**
```
Parsed config:
  name => My Application
  version => 1.0.0
  debug => true
  port => 8080
```

---

## ขั้นตอนที่ 267: std.mem.split และ tokenize

### แนวคิดพื้นฐาน

`std.mem.split` และ `std.mem.tokenize` ใช้สำหรับแบ่ง slice เป็นส่วนย่อยๆ ความแตกต่างสำคัญ:
- `split`: คืน empty slice เมื่อพบ delimiter ติดกัน
- `tokenize`: ข้าม delimiter ที่ติดกัน (ไม่คืน empty token)

### นิยาม (Signature)

```zig
// split ด้วย sequence เดียว
pub fn split(comptime T: type, buffer: []const T, delimiter: []const T) SplitIterator(T, .sequence)

// split ด้วย any character ใน delimiter set
pub fn splitAny(comptime T: type, buffer: []const T, delimiter: []const T) SplitIterator(T, .any)

// split ด้วย scalar (character เดียว)
pub fn splitScalar(comptime T: type, buffer: []const T, delimiter: T) SplitIterator(T, .scalar)

// tokenize ข้าม delimiter ที่ติดกัน
pub fn tokenize(comptime T: type, buffer: []const T, delimiter: []const T) TokenIterator(T, .any)
pub fn tokenizeAny(comptime T: type, buffer: []const T, delimiter: []const T) TokenIterator(T, .any)
pub fn tokenizeScalar(comptime T: type, buffer: []const T, delimiter: T) TokenIterator(T, .scalar)
pub fn tokenizeSequence(comptime T: type, buffer: []const T, delimiter: []const T) TokenIterator(T, .sequence)
```

### ตัวอย่างที่ 267.1: การแยก CSV

```zig
const std = @import("std");

pub fn main() void {
    const csv_line = "Alice,30,Bangkok,Engineer";

    std.debug.print("CSV line: {s}\n", .{csv_line});
    std.debug.print("Fields:\n", .{});

    var iter = std.mem.splitScalar(u8, csv_line, ',');
    var field_num: usize = 1;

    while (iter.next()) |field| {
        std.debug.print("  Field {d}: '{s}'\n", .{ field_num, field });
        field_num += 1;
    }
}
```

**ผลลัพธ์:**
```
CSV line: Alice,30,Bangkok,Engineer
Fields:
  Field 1: 'Alice'
  Field 2: '30'
  Field 3: 'Bangkok'
  Field 4: 'Engineer'
```

### ตัวอย่างที่ 267.2: ความแตกต่างระหว่าง split และ tokenize

```zig
const std = @import("std");

pub fn main() void {
    const text = "one  two   three  four";

    std.debug.print("Text: '{s}'\n\n", .{text});

    // split: คืน empty string สำหรับ space ที่ติดกัน
    std.debug.print("split(' '):\n", .{});
    var split_iter = std.mem.splitScalar(u8, text, ' ');
    var count: usize = 0;
    while (split_iter.next()) |token| {
        count += 1;
        std.debug.print("  [{d}]: '{s}'\n", .{ count, token });
    }
    std.debug.print("  รวม: {d} tokens\n\n", .{count});

    // tokenize: ข้าม space ที่ติดกัน
    std.debug.print("tokenize(' '):\n", .{});
    var tok_iter = std.mem.tokenizeScalar(u8, text, ' ');
    count = 0;
    while (tok_iter.next()) |token| {
        count += 1;
        std.debug.print("  [{d}]: '{s}'\n", .{ count, token });
    }
    std.debug.print("  รวม: {d} tokens\n", .{count});
}
```

**ผลลัพธ์:**
```
Text: 'one  two   three  four'

split(' '):
  [1]: 'one'
  [2]: ''
  [3]: 'two'
  [4]: ''
  [5]: ''
  [6]: 'three'
  [7]: ''
  [8]: 'four'
  รวม: 8 tokens

tokenize(' '):
  [1]: 'one'
  [2]: 'two'
  [3]: 'three'
  [4]: 'four'
  รวม: 4 tokens
```

### ตัวอย่างที่ 267.3: การ Split บรรทัดของไฟล์

```zig
const std = @import("std");

pub fn main() void {
    const file_content =
        \\Line 1: Hello
        \\Line 2: World
        \\Line 3: Zig
        \\Line 4: Programming
    ;

    std.debug.print("ประมวลผลบรรทัด:\n", .{});
    var line_iter = std.mem.splitScalar(u8, file_content, '\n');
    var line_num: usize = 1;

    while (line_iter.next()) |line| {
        std.debug.print("  {d:>3}: {s}\n", .{ line_num, line });
        line_num += 1;
    }
}
```

**ผลลัพธ์:**
```
ประมวลผลบรรทัด:
    1: Line 1: Hello
    2: Line 2: World
    3: Line 3: Zig
    4: Line 4: Programming
```

### ตัวอย่างที่ 267.4: การ Parse Path

```zig
const std = @import("std");

pub fn main() void {
    const path = "/home/user/projects/zig/main.zig";

    std.debug.print("Path: {s}\n", .{path});
    std.debug.print("Components:\n", .{});

    var iter = std.mem.splitScalar(u8, path, '/');
    var depth: i32 = -1; // -1 เพราะ path เริ่มด้วย '/'

    while (iter.next()) |component| {
        if (component.len > 0) {
            depth += 1;
            std.debug.print("  depth {d}: {s}\n", .{ depth, component });
        }
    }
}
```

**ผลลัพธ์:**
```
Path: /home/user/projects/zig/main.zig
Components:
  depth 0: home
  depth 1: user
  depth 2: projects
  depth 3: zig
  depth 4: main.zig
```

---

## ขั้นตอนที่ 268: std.mem.replace และ replaceOwned

### แนวคิดพื้นฐาน

`std.mem.replace` ใช้แทนที่ pattern ใน slice ด้วยค่าใหม่ มีสองเวอร์ชัน:
- `replace`: เขียน result ลงใน buffer ที่มีอยู่แล้ว (ต้องคำนวณขนาดก่อน)
- `replaceOwned`: จัดสรร memory ใหม่และคืน slice ที่ได้ (ต้องจัดการ free เอง)

### นิยาม (Signature)

```zig
// แทนที่ needle ด้วย replacement ใน input และเขียนลง output
// คืนจำนวนครั้งที่แทนที่
pub fn replace(comptime T: type, input: []const T, needle: []const T, replacement: []const T, output: []T) usize

// คำนวณขนาด output ที่ต้องการ
pub fn replacementSize(comptime T: type, input: []const T, needle: []const T, replacement: []const T) usize

// แทนที่และจัดสรร memory ใหม่
pub fn replaceOwned(comptime T: type, allocator: std.mem.Allocator, input: []const T, needle: []const T, replacement: []const T) std.mem.Allocator.Error![]T
```

### ตัวอย่างที่ 268.1: การแทนที่ด้วย Buffer ที่กำหนดเอง

```zig
const std = @import("std");

pub fn main() void {
    const original = "Hello, World! Hello, Zig!";
    const needle = "Hello";
    const replacement = "Hi";

    // คำนวณขนาด output ที่ต้องการ
    const output_size = std.mem.replacementSize(u8, original, needle, replacement);
    std.debug.print("ขนาด output ที่ต้องการ: {d}\n", .{output_size});

    var output_buf: [64]u8 = undefined;
    const output = output_buf[0..output_size];

    const count = std.mem.replace(u8, original, needle, replacement, output);

    std.debug.print("Original:    {s}\n", .{original});
    std.debug.print("Result:      {s}\n", .{output});
    std.debug.print("แทนที่ {d} ครั้ง\n", .{count});
}
```

**ผลลัพธ์:**
```
ขนาด output ที่ต้องการ: 21
Original:    Hello, World! Hello, Zig!
Result:      Hi, World! Hi, Zig!
แทนที่ 2 ครั้ง
```

### ตัวอย่างที่ 268.2: การใช้ replaceOwned

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const template = "Dear {name}, your order #{order_id} is ready.";

    // แทนที่ placeholder ทีละตัว
    const step1 = try std.mem.replaceOwned(u8, allocator, template, "{name}", "คุณสมชาย");
    defer allocator.free(step1);

    const result = try std.mem.replaceOwned(u8, allocator, step1, "{order_id}", "ORD-12345");
    defer allocator.free(result);

    std.debug.print("Template: {s}\n", .{template});
    std.debug.print("Result:   {s}\n", .{result});
}
```

**ผลลัพธ์:**
```
Template: Dear {name}, your order #{order_id} is ready.
Result:   Dear คุณสมชาย, your order #ORD-12345 is ready.
```

### ตัวอย่างที่ 268.3: Template Engine อย่างง่าย

```zig
const std = @import("std");

const Replacement = struct {
    placeholder: []const u8,
    value: []const u8,
};

fn renderTemplate(
    allocator: std.mem.Allocator,
    template: []const u8,
    replacements: []const Replacement,
) ![]u8 {
    var current = try allocator.dupe(u8, template);

    for (replacements) |rep| {
        const new = try std.mem.replaceOwned(u8, allocator, current, rep.placeholder, rep.value);
        allocator.free(current);
        current = new;
    }

    return current;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const template = "สวัสดี {{user}}! คุณมีข้อความ {{count}} ฉบับ จาก {{sender}}";

    const replacements = [_]Replacement{
        .{ .placeholder = "{{user}}", .value = "อลิซ" },
        .{ .placeholder = "{{count}}", .value = "5" },
        .{ .placeholder = "{{sender}}", .value = "บ็อบ" },
    };

    const result = try renderTemplate(allocator, template, &replacements);
    defer allocator.free(result);

    std.debug.print("Template: {s}\n", .{template});
    std.debug.print("Result:   {s}\n", .{result});
}
```

**ผลลัพธ์:**
```
Template: สวัสดี {{user}}! คุณมีข้อความ {{count}} ฉบับ จาก {{sender}}
Result:   สวัสดี อลิซ! คุณมีข้อความ 5 ฉบับ จาก บ็อบ
```

---

## ขั้นตอนที่ 269: std.mem.sort และ lessThan

### แนวคิดพื้นฐาน

`std.mem.sort` เป็น in-place sorting algorithm ใน Zig ที่ใช้ comparison function ที่กำหนดเอง สามารถเรียงลำดับข้อมูลชนิดใดก็ได้

### นิยาม (Signature)

```zig
// เรียงลำดับ slice in-place
pub fn sort(comptime T: type, items: []T, context: anytype, comptime lessThanFn: fn (context: @TypeOf(context), lhs: T, rhs: T) bool) void

// lessThan functions สำหรับชนิดข้อมูลพื้นฐาน
pub fn lessThan(comptime T: type, context: void, lhs: T, rhs: T) bool  // deprecated
```

### ตัวอย่างที่ 269.1: การเรียงลำดับตัวเลข

```zig
const std = @import("std");

pub fn main() void {
    var numbers = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };

    std.debug.print("ก่อนเรียงลำดับ: ", .{});
    for (numbers) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});

    // เรียงลำดับจากน้อยไปมาก
    std.mem.sort(i32, &numbers, {}, std.sort.asc(i32));

    std.debug.print("น้อย -> มาก:    ", .{});
    for (numbers) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});

    // เรียงลำดับจากมากไปน้อย
    std.mem.sort(i32, &numbers, {}, std.sort.desc(i32));

    std.debug.print("มาก -> น้อย:    ", .{});
    for (numbers) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
ก่อนเรียงลำดับ: 64 34 25 12 22 11 90
น้อย -> มาก:    11 12 22 25 34 64 90
มาก -> น้อย:    90 64 34 25 22 12 11
```

### ตัวอย่างที่ 269.2: การเรียงลำดับ String

```zig
const std = @import("std");

fn stringLessThan(_: void, a: []const u8, b: []const u8) bool {
    return std.mem.lessThan(u8, a, b);
}

pub fn main() void {
    var words = [_][]const u8{
        "banana",
        "apple",
        "cherry",
        "date",
        "elderberry",
    };

    std.debug.print("ก่อนเรียง: ", .{});
    for (words) |w| std.debug.print("{s} ", .{w});
    std.debug.print("\n", .{});

    std.mem.sort([]const u8, &words, {}, stringLessThan);

    std.debug.print("หลังเรียง:  ", .{});
    for (words) |w| std.debug.print("{s} ", .{w});
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
ก่อนเรียง: banana apple cherry date elderberry
หลังเรียง:  apple banana cherry date elderberry
```

### ตัวอย่างที่ 269.3: การเรียงลำดับ Struct ด้วย Custom Comparator

```zig
const std = @import("std");

const Student = struct {
    name: []const u8,
    score: f64,
    age: u32,
};

fn byScoreDesc(_: void, a: Student, b: Student) bool {
    return a.score > b.score;
}

fn byNameAsc(_: void, a: Student, b: Student) bool {
    return std.mem.lessThan(u8, a.name, b.name);
}

fn byAgeAsc(_: void, a: Student, b: Student) bool {
    return a.age < b.age;
}

pub fn main() void {
    var students = [_]Student{
        .{ .name = "อลิซ", .score = 92.5, .age = 20 },
        .{ .name = "บ็อบ", .score = 87.0, .age = 22 },
        .{ .name = "ชาร์ลี", .score = 95.0, .age = 19 },
        .{ .name = "ไดอาน่า", .score = 87.0, .age = 21 },
        .{ .name = "อีฟ", .score = 78.5, .age = 23 },
    };

    std.debug.print("เรียงตามคะแนน (มาก -> น้อย):\n", .{});
    std.mem.sort(Student, &students, {}, byScoreDesc);
    for (students) |s| {
        std.debug.print("  {s}: {d:.1} (อายุ {d})\n", .{ s.name, s.score, s.age });
    }

    std.debug.print("\nเรียงตามชื่อ (ก -> ฮ):\n", .{});
    std.mem.sort(Student, &students, {}, byNameAsc);
    for (students) |s| {
        std.debug.print("  {s}: {d:.1} (อายุ {d})\n", .{ s.name, s.score, s.age });
    }

    std.debug.print("\nเรียงตามอายุ (น้อย -> มาก):\n", .{});
    std.mem.sort(Student, &students, {}, byAgeAsc);
    for (students) |s| {
        std.debug.print("  {s}: {d:.1} (อายุ {d})\n", .{ s.name, s.score, s.age });
    }
}
```

**ผลลัพธ์:**
```
เรียงตามคะแนน (มาก -> น้อย):
  ชาร์ลี: 95.0 (อายุ 19)
  อลิซ: 92.5 (อายุ 20)
  บ็อบ: 87.0 (อายุ 22)
  ไดอาน่า: 87.0 (อายุ 21)
  อีฟ: 78.5 (อายุ 23)

เรียงตามชื่อ (ก -> ฮ):
  ...
```

### ตัวอย่างที่ 269.4: การใช้ std.mem.lessThan

```zig
const std = @import("std");

pub fn main() void {
    const a = "apple";
    const b = "banana";
    const c = "apple";

    std.debug.print("lessThan('apple', 'banana'): {}\n", .{std.mem.lessThan(u8, a, b)});
    std.debug.print("lessThan('banana', 'apple'): {}\n", .{std.mem.lessThan(u8, b, a)});
    std.debug.print("lessThan('apple', 'apple'): {}\n", .{std.mem.lessThan(u8, a, c)});

    // เปรียบเทียบ byte arrays
    const bytes1 = [_]u8{ 1, 2, 3 };
    const bytes2 = [_]u8{ 1, 2, 4 };
    const bytes3 = [_]u8{ 1, 2 };

    std.debug.print("\nlessThan([1,2,3], [1,2,4]): {}\n", .{std.mem.lessThan(u8, &bytes1, &bytes2)});
    std.debug.print("lessThan([1,2,3], [1,2]): {}\n", .{std.mem.lessThan(u8, &bytes1, &bytes3)});
    std.debug.print("lessThan([1,2], [1,2,3]): {}\n", .{std.mem.lessThan(u8, &bytes3, &bytes1)});
}
```

**ผลลัพธ์:**
```
lessThan('apple', 'banana'): true
lessThan('banana', 'apple'): false
lessThan('apple', 'apple'): false

lessThan([1,2,3], [1,2,4]): true
lessThan([1,2,3], [1,2]): false
lessThan([1,2], [1,2,3]): true
```

---

## ขั้นตอนที่ 270: std.mem.Allocator Functions (alloc, dupe, concat)

### แนวคิดพื้นฐาน

`std.mem.Allocator` คือ interface สำหรับการจัดสรรหน่วยความจำใน Zig มีฟังก์ชันหลักที่ต้องรู้จัก:
- `alloc`: จัดสรร memory ใหม่
- `dupe`: คัดลอกและจัดสรร memory ให้สำเนา
- `concat`: รวม slice หลายอันเข้าด้วยกัน

### นิยาม (Signature)

```zig
// จัดสรร memory สำหรับ n elements ของ type T
pub fn alloc(self: Allocator, comptime T: type, n: usize) Error![]T

// คืน memory
pub fn free(self: Allocator, memory: anytype) void

// จัดสรร memory และ resize
pub fn realloc(self: Allocator, old_mem: anytype, new_n: usize) Error![]@typeInfo(@TypeOf(old_mem)).pointer.child

// คัดลอก slice และจัดสรร memory ใหม่
pub fn dupe(self: Allocator, comptime T: type, m: []const T) Error![]T

// รวม slice หลายอัน
pub fn concat(self: Allocator, comptime T: type, slices: []const []const T) Error![]T
```

### ตัวอย่างที่ 270.1: การใช้ alloc และ free

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // จัดสรร memory สำหรับ 10 ตัวเลข
    const numbers = try allocator.alloc(i32, 10);
    defer allocator.free(numbers);

    // เติมข้อมูล
    for (numbers, 0..) |*n, i| {
        n.* = @as(i32, @intCast(i * i));
    }

    std.debug.print("Squares: ", .{});
    for (numbers) |n| {
        std.debug.print("{d} ", .{n});
    }
    std.debug.print("\n", .{});

    // จัดสรร memory สำหรับ string
    const message = try allocator.alloc(u8, 50);
    defer allocator.free(message);

    const written = std.fmt.bufPrint(message, "Sum = {d}", .{
        blk: {
            var sum: i32 = 0;
            for (numbers) |n| sum += n;
            break :blk sum;
        },
    }) catch unreachable;

    std.debug.print("{s}\n", .{written});
}
```

**ผลลัพธ์:**
```
Squares: 0 1 4 9 16 25 36 49 64 81
Sum = 285
```

### ตัวอย่างที่ 270.2: การใช้ dupe

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const original = "Hello, Zig World!";

    // dupe string
    const copy = try allocator.dupe(u8, original);
    defer allocator.free(copy);

    std.debug.print("Original: {s} (ptr: {*})\n", .{ original, original.ptr });
    std.debug.print("Copy:     {s} (ptr: {*})\n", .{ copy, copy.ptr });
    std.debug.print("เนื้อหาเหมือนกัน: {}\n", .{std.mem.eql(u8, original, copy)});
    std.debug.print("Pointer ต่างกัน: {}\n", .{original.ptr != copy.ptr});

    // แก้ไข copy โดยไม่กระทบ original
    copy[0] = 'h';
    std.debug.print("\nหลังแก้ไข:\n", .{});
    std.debug.print("Original: {s}\n", .{original});
    std.debug.print("Copy:     {s}\n", .{copy});
}
```

**ผลลัพธ์:**
```
Original: Hello, Zig World! (ptr: ...)
Copy:     Hello, Zig World! (ptr: ...)
เนื้อหาเหมือนกัน: true
Pointer ต่างกัน: true

หลังแก้ไข:
Original: Hello, Zig World!
Copy:     hello, Zig World!
```

### ตัวอย่างที่ 270.3: การใช้ concat

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // รวม string หลายอัน
    const parts = [_][]const u8{
        "สวัสดี",
        ", ",
        "โลก",
        "! ",
        "ยินดีต้อนรับสู่ Zig",
    };

    const result = try std.mem.concat(allocator, u8, &parts);
    defer allocator.free(result);

    std.debug.print("Result: {s}\n", .{result});
    std.debug.print("Length: {d}\n", .{result.len});
}
```

**ผลลัพธ์:**
```
Result: สวัสดี, โลก! ยินดีต้อนรับสู่ Zig
Length: ...
```

### ตัวอย่างที่ 270.4: Dynamic Array Builder

```zig
const std = @import("std");

const StringBuilder = struct {
    allocator: std.mem.Allocator,
    buffer: []u8,
    len: usize,

    pub fn init(allocator: std.mem.Allocator) StringBuilder {
        return .{
            .allocator = allocator,
            .buffer = &[_]u8{},
            .len = 0,
        };
    }

    pub fn deinit(self: *StringBuilder) void {
        if (self.buffer.len > 0) {
            self.allocator.free(self.buffer);
        }
    }

    pub fn append(self: *StringBuilder, str: []const u8) !void {
        const new_len = self.len + str.len;

        if (new_len > self.buffer.len) {
            // ขยาย buffer
            const new_cap = @max(new_len, self.buffer.len * 2 + 8);
            const new_buf = try self.allocator.alloc(u8, new_cap);
            if (self.buffer.len > 0) {
                @memcpy(new_buf[0..self.len], self.buffer[0..self.len]);
                self.allocator.free(self.buffer);
            }
            self.buffer = new_buf;
        }

        @memcpy(self.buffer[self.len..new_len], str);
        self.len = new_len;
    }

    pub fn toString(self: *const StringBuilder) []const u8 {
        return self.buffer[0..self.len];
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var sb = StringBuilder.init(allocator);
    defer sb.deinit();

    try sb.append("Hello");
    try sb.append(", ");
    try sb.append("World");
    try sb.append("!");

    std.debug.print("Result: {s}\n", .{sb.toString()});
    std.debug.print("Length: {d}\n", .{sb.len});
}
```

**ผลลัพธ์:**
```
Result: Hello, World!
Length: 13
```

### ตัวอย่างที่ 270.5: การจัดการหน่วยความจำอย่างปลอดภัย

```zig
const std = @import("std");

const DataProcessor = struct {
    allocator: std.mem.Allocator,
    data: []i32,

    pub fn init(allocator: std.mem.Allocator, size: usize) !DataProcessor {
        const data = try allocator.alloc(i32, size);
        @memset(std.mem.sliceAsBytes(data), 0); // initialize to zero
        return .{ .allocator = allocator, .data = data };
    }

    pub fn deinit(self: *DataProcessor) void {
        self.allocator.free(self.data);
    }

    pub fn fill(self: *DataProcessor, value: i32) void {
        std.mem.set(i32, self.data, value);
    }

    pub fn sum(self: *const DataProcessor) i64 {
        var total: i64 = 0;
        for (self.data) |n| {
            total += n;
        }
        return total;
    }

    pub fn resize(self: *DataProcessor, new_size: usize) !void {
        self.data = try self.allocator.realloc(self.data, new_size);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var processor = try DataProcessor.init(allocator, 5);
    defer processor.deinit();

    processor.fill(10);
    std.debug.print("Size: {d}, Sum: {d}\n", .{ processor.data.len, processor.sum() });

    try processor.resize(10);
    // เติมข้อมูลส่วนใหม่
    for (processor.data[5..]) |*n| {
        n.* = 20;
    }

    std.debug.print("After resize - Size: {d}, Sum: {d}\n", .{ processor.data.len, processor.sum() });
}
```

**ผลลัพธ์:**
```
Size: 5, Sum: 50
After resize - Size: 10, Sum: 150
```

### ตัวอย่างที่ 270.6: การเปรียบเทียบ Allocator ต่างๆ

```zig
const std = @import("std");

pub fn main() !void {
    // 1. GeneralPurposeAllocator - สำหรับ development
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const gpa_alloc = gpa.allocator();

    // 2. FixedBufferAllocator - สำหรับ stack-based allocation
    var fixed_buf: [1024]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&fixed_buf);
    const fba_alloc = fba.allocator();

    // 3. ArenaAllocator - free ทีเดียวทั้งหมด
    var arena = std.heap.ArenaAllocator.init(gpa_alloc);
    defer arena.deinit(); // free ทุก allocation ในคราวเดียว
    const arena_alloc = arena.allocator();

    // ทดสอบ GPA
    const gpa_str = try gpa_alloc.dupe(u8, "Hello from GPA!");
    defer gpa_alloc.free(gpa_str);
    std.debug.print("{s}\n", .{gpa_str});

    // ทดสอบ FBA
    const fba_str = try fba_alloc.dupe(u8, "Hello from FBA!");
    // ไม่ต้อง free เพราะ FBA manage เอง
    std.debug.print("{s}\n", .{fba_str});

    // ทดสอบ Arena
    const arena_str1 = try arena_alloc.dupe(u8, "Hello from Arena! (1)");
    const arena_str2 = try arena_alloc.dupe(u8, "Hello from Arena! (2)");
    // ไม่ต้อง free แต่ละ allocation
    // arena.deinit() จะ free ทั้งหมด
    std.debug.print("{s}\n", .{arena_str1});
    std.debug.print("{s}\n", .{arena_str2});
}
```

**ผลลัพธ์:**
```
Hello from GPA!
Hello from FBA!
Hello from Arena! (1)
Hello from Arena! (2)
```

---

## ฟังก์ชันเสริมใน std.mem ที่น่ารู้

### ฟังก์ชันอื่นๆ ที่ใช้งานบ่อย

```zig
const std = @import("std");

pub fn main() void {
    // std.mem.reverse - กลับลำดับ slice
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    std.mem.reverse(i32, &arr);
    std.debug.print("Reversed: ", .{});
    for (arr) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});

    // std.mem.rotate - rotate slice
    var rot = [_]i32{ 1, 2, 3, 4, 5 };
    std.mem.rotate(i32, &rot, 2); // rotate ไปซ้าย 2
    std.debug.print("Rotated(2): ", .{});
    for (rot) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});

    // std.mem.count - นับ occurrences
    const text = "hello world hello zig hello";
    const count = std.mem.count(u8, text, "hello");
    std.debug.print("'hello' พบ {d} ครั้ง\n", .{count});

    // std.mem.containsAtLeast - ตรวจสอบว่ามีอย่างน้อย n copies
    const has_two = std.mem.containsAtLeast(u8, text, 2, "hello");
    std.debug.print("มี 'hello' อย่างน้อย 2 ครั้ง: {}\n", .{has_two});

    // std.mem.zeroes - สร้าง zero-initialized struct
    const ZeroStruct = struct {
        x: i32,
        y: i32,
        z: f32,
    };
    const zero_s = std.mem.zeroes(ZeroStruct);
    std.debug.print("Zero struct: x={d}, y={d}, z={d:.1}\n", .{ zero_s.x, zero_s.y, zero_s.z });
}
```

**ผลลัพธ์:**
```
Reversed: 5 4 3 2 1
Rotated(2): 3 4 5 1 2
'hello' พบ 3 ครั้ง
มี 'hello' อย่างน้อย 2 ครั้ง: true
Zero struct: x=0, y=0, z=0.0
```

---

## โปรเจกต์ประยุกต์: Text Processing Library

```zig
const std = @import("std");

/// ตัวอย่าง Text Processor ที่ใช้ std.mem หลายฟังก์ชัน
const TextProcessor = struct {
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) TextProcessor {
        return .{ .allocator = allocator };
    }

    /// นับจำนวนคำ (word count)
    pub fn wordCount(self: *const TextProcessor, text: []const u8) usize {
        _ = self;
        var count: usize = 0;
        var iter = std.mem.tokenizeAny(u8, text, " \t\n\r");
        while (iter.next()) |_| count += 1;
        return count;
    }

    /// หาคำที่ยาวที่สุด
    pub fn longestWord(self: *const TextProcessor, text: []const u8) ?[]const u8 {
        _ = self;
        var longest: ?[]const u8 = null;
        var iter = std.mem.tokenizeAny(u8, text, " \t\n\r.,!?;:");
        while (iter.next()) |word| {
            if (longest == null or word.len > longest.?.len) {
                longest = word;
            }
        }
        return longest;
    }

    /// นับความถี่ของ character
    pub fn charFrequency(self: *const TextProcessor, text: []const u8, ch: u8) usize {
        _ = self;
        return std.mem.count(u8, text, &[_]u8{ch});
    }

    /// แทนที่ทุก occurrence ของคำ
    pub fn replaceWord(
        self: *const TextProcessor,
        text: []const u8,
        old_word: []const u8,
        new_word: []const u8,
    ) ![]u8 {
        return std.mem.replaceOwned(u8, self.allocator, text, old_word, new_word);
    }

    /// ตรวจสอบว่าเป็น palindrome
    pub fn isPalindrome(self: *const TextProcessor, text: []const u8) bool {
        _ = self;
        if (text.len == 0) return true;
        var left: usize = 0;
        var right: usize = text.len - 1;
        while (left < right) {
            if (text[left] != text[right]) return false;
            left += 1;
            right -= 1;
        }
        return true;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const processor = TextProcessor.init(allocator);

    const sample_text = "The quick brown fox jumps over the lazy dog";

    std.debug.print("Text: {s}\n\n", .{sample_text});

    // word count
    std.debug.print("จำนวนคำ: {d}\n", .{processor.wordCount(sample_text)});

    // longest word
    if (processor.longestWord(sample_text)) |word| {
        std.debug.print("คำที่ยาวที่สุด: '{s}' ({d} ตัวอักษร)\n", .{ word, word.len });
    }

    // character frequency
    const e_count = processor.charFrequency(sample_text, 'e');
    std.debug.print("ความถี่ของ 'e': {d} ครั้ง\n", .{e_count});

    // replace word
    const replaced = try processor.replaceWord(sample_text, "fox", "cat");
    defer allocator.free(replaced);
    std.debug.print("หลังแทนที่: {s}\n", .{replaced});

    // palindrome
    const words = [_][]const u8{ "racecar", "hello", "level", "world" };
    std.debug.print("\nตรวจสอบ palindrome:\n", .{});
    for (words) |word| {
        std.debug.print("  '{s}': {}\n", .{ word, processor.isPalindrome(word) });
    }
}
```

**ผลลัพธ์:**
```
Text: The quick brown fox jumps over the lazy dog

จำนวนคำ: 9
คำที่ยาวที่สุด: 'quick' (5 ตัวอักษร)
ความถี่ของ 'e': 3 ครั้ง
หลังแทนที่: The quick brown cat jumps over the lazy dog

ตรวจสอบ palindrome:
  'racecar': true
  'hello': false
  'level': true
  'world': false
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: String Statistics

เขียนโปรแกรมที่รับ string และพิมพ์:
- จำนวนตัวอักษรทั้งหมด
- จำนวนคำ (แยกด้วย whitespace)
- จำนวนบรรทัด
- คำที่สั้นที่สุดและยาวที่สุด
- ความถี่ของ vowel (a, e, i, o, u)

```zig
// ฝึกทำเอง
const std = @import("std");

fn stringStats(text: []const u8) void {
    // TODO: ใช้ std.mem.tokenizeAny, std.mem.splitScalar, std.mem.count
    _ = text;
}

pub fn main() void {
    const text =
        \\The quick brown fox
        \\jumps over the lazy dog
    ;
    stringStats(text);
}
```

### แบบฝึกหัดที่ 2: CSV Parser

เขียน CSV parser ที่:
- อ่านบรรทัดแรกเป็น header
- Parse แต่ละแถวเป็น record
- รองรับ field ที่มี spaces
- ตัด whitespace รอบๆ แต่ละ field

```zig
// ฝึกทำเอง
const std = @import("std");

const CsvRecord = struct {
    fields: [][]const u8,
    allocator: std.mem.Allocator,

    pub fn deinit(self: *CsvRecord) void {
        for (self.fields) |field| {
            self.allocator.free(field);
        }
        self.allocator.free(self.fields);
    }
};

fn parseCsvLine(allocator: std.mem.Allocator, line: []const u8) !CsvRecord {
    // TODO: ใช้ std.mem.splitScalar, std.mem.trim, allocator.dupe
    _ = allocator;
    _ = line;
    unreachable;
}

pub fn main() !void {
    // TODO: test the CSV parser
}
```

### แบบฝึกหัดที่ 3: Sort Benchmark

เขียนโปรแกรมที่:
- สร้าง array ของตัวเลขสุ่ม 1,000 ตัว
- เรียงลำดับจากน้อยไปมาก
- เรียงลำดับจากมากไปน้อย
- ตรวจสอบว่าผลลัพธ์ถูกต้อง

```zig
// ฝึกทำเอง
const std = @import("std");

fn isSorted(comptime T: type, arr: []const T, ascending: bool) bool {
    // TODO: ตรวจสอบว่า array เรียงลำดับถูกต้อง
    _ = arr;
    _ = ascending;
    return false;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();

    // TODO: สร้างตัวเลขสุ่มและทดสอบ sort
    _ = allocator;
    _ = random;
}
```

### แบบฝึกหัดที่ 4: String Builder

เขียน StringBuilder ที่สมบูรณ์กว่าตัวอย่าง โดยมีฟังก์ชัน:
- `append(str)`: เพิ่ม string
- `appendFmt(fmt, args)`: เพิ่ม formatted string
- `prepend(str)`: เพิ่มที่ต้น
- `insertAt(pos, str)`: แทรกที่ตำแหน่งที่กำหนด
- `clear()`: ล้างข้อมูล
- `toString()`: คืนค่าเป็น string

```zig
// ฝึกทำเอง
const std = @import("std");

const AdvancedStringBuilder = struct {
    // TODO: implement
};
```

---

## สรุป (Summary)

### ฟังก์ชันหลักที่ได้เรียนรู้

| ฟังก์ชัน | การใช้งาน | ความซับซ้อน |
|----------|-----------|-------------|
| `std.mem.eql` | เปรียบเทียบ slice | O(n) |
| `std.mem.copyForwards/Backwards` | คัดลอก memory (รองรับ overlap) | O(n) |
| `@memcpy` | คัดลอก memory (ไม่ overlap) | O(n) |
| `std.mem.set` | เติมค่าซ้ำ | O(n) |
| `@memset` | เติม byte ซ้ำ | O(n) |
| `std.mem.indexOf` | ค้นหาตำแหน่งแรก | O(n*m) |
| `std.mem.lastIndexOf` | ค้นหาตำแหน่งสุดท้าย | O(n*m) |
| `std.mem.startsWith` | ตรวจสอบ prefix | O(m) |
| `std.mem.endsWith` | ตรวจสอบ suffix | O(m) |
| `std.mem.trim` | ตัด character รอบขอบ | O(n) |
| `std.mem.split` | แบ่ง slice (รวม empty) | Iterator |
| `std.mem.tokenize` | แบ่ง slice (ข้าม empty) | Iterator |
| `std.mem.replace` | แทนที่ pattern | O(n) |
| `std.mem.replaceOwned` | แทนที่ + จัดสรร memory | O(n) |
| `std.mem.sort` | เรียงลำดับ | O(n log n) |
| `std.mem.lessThan` | เปรียบเทียบแบบ lexicographic | O(min(n,m)) |
| `allocator.alloc` | จัดสรร memory | O(1)* |
| `allocator.dupe` | คัดลอก + จัดสรร | O(n) |
| `std.mem.concat` | รวม slice | O(n) |

### หลักการสำคัญ

1. **Memory Safety**: ใน Zig คุณต้องจัดการ memory เอง แต่ compiler จะช่วยตรวจสอบ
2. **No Hidden Allocations**: ฟังก์ชันที่ต้องการ memory จะรับ `Allocator` เสมอ
3. **Slice vs Pointer**: Zig ใช้ slice (pointer + length) แทน null-terminated string
4. **comptime T**: ฟังก์ชันส่วนใหญ่ใช้ generic type ผ่าน comptime
5. **Error Handling**: ฟังก์ชันที่อาจ fail จะคืน `!T` (error union)

### Pattern ที่ควรจำ

```zig
// Pattern 1: Safe String Operations
const result = std.mem.trim(u8, input, " \t\n");

// Pattern 2: Iterator Pattern
var iter = std.mem.splitScalar(u8, text, ',');
while (iter.next()) |part| {
    // process part
}

// Pattern 3: Arena Allocator Pattern
var arena = std.heap.ArenaAllocator.init(allocator);
defer arena.deinit();
const a = arena.allocator();
// allocate freely, all freed when arena.deinit() is called

// Pattern 4: Defer Pattern
const buffer = try allocator.alloc(u8, size);
defer allocator.free(buffer);
```

### ขั้นตอนต่อไป

หลังจากเรียนรู้ `std.mem` แล้ว ขั้นตอนต่อไปแนะนำให้ศึกษา:
- `std.fmt`: การจัดรูปแบบ string และการ print
- `std.io`: การอ่านและเขียน I/O
- `std.fs`: การจัดการ file system
- `std.ArrayList`: Dynamic array ที่ใช้ Allocator
- `std.HashMap`: Hash map สำหรับ key-value storage

---

## อ้างอิง (References)

- [Zig Standard Library Documentation](https://ziglang.org/documentation/master/std/)
- [Zig Language Reference - Memory](https://ziglang.org/documentation/master/#Memory)
- [std.mem source code](https://github.com/ziglang/zig/blob/master/lib/std/mem.zig)
- [Zig Guide - Allocators](https://zig.guide/standard-library/allocators)

---

*ตอนที่ 27 จบสิ้น - ตอนต่อไป: std.fmt และการจัดรูปแบบข้อมูล*
