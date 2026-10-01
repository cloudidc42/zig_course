# Part 09: Arrays และ Slices ใน Zig (Steps 81-90)

ยินดีต้อนรับสู่ Part 09 ของคอร์ส Zig ภาษาไทย! ในบทนี้เราจะเรียนรู้เกี่ยวกับ Arrays และ Slices ซึ่งเป็นโครงสร้างข้อมูลพื้นฐานที่สำคัญมากใน Zig

---

## Step 81: Arrays พื้นฐาน - การสร้างและเข้าถึง

### Arrays คืออะไร?

Array ใน Zig คือกลุ่มของข้อมูลชนิดเดียวกันที่จัดเก็บอยู่ในหน่วยความจำแบบต่อเนื่องกัน (contiguous memory) โดยขนาดของ Array ต้องรู้ตั้งแต่ตอน compile time

**รูปแบบประกาศ Array:**
```
[N]T
```
- `N` = จำนวนสมาชิก (ต้องเป็น comptime value)
- `T` = ชนิดข้อมูลของสมาชิกแต่ละตัว

### การสร้าง Array

```zig
const std = @import("std");

pub fn main() void {
    // สร้าง Array แบบต่างๆ
    const numbers: [5]i32 = [5]i32{ 10, 20, 30, 40, 50 };
    
    // Zig สามารถ infer ขนาด Array ได้ด้วย underscore
    const colors: [3][]const u8 = [_][]const u8{ "red", "green", "blue" };
    
    // สร้าง Array ของ bool
    const flags: [4]bool = [_]bool{ true, false, true, false };
    
    std.debug.print("numbers[0] = {}\n", .{numbers[0]});
    std.debug.print("colors[1] = {s}\n", .{colors[1]});
    std.debug.print("flags[2] = {}\n", .{flags[2]});
}
```

### การเข้าถึงสมาชิก Array

```zig
const std = @import("std");

pub fn main() void {
    const primes = [_]u32{ 2, 3, 5, 7, 11, 13, 17, 19 };
    
    // เข้าถึงด้วย index (เริ่มจาก 0)
    std.debug.print("จำนวนเฉพาะตัวที่ 1: {}\n", .{primes[0]});
    std.debug.print("จำนวนเฉพาะตัวที่ 5: {}\n", .{primes[4]});
    
    // ขนาดของ Array
    std.debug.print("ขนาด Array: {}\n", .{primes.len});
    
    // Loop ผ่าน Array
    for (primes) |prime| {
        std.debug.print("{} ", .{prime});
    }
    std.debug.print("\n", .{});
    
    // Loop พร้อม index
    for (primes, 0..) |prime, i| {
        std.debug.print("primes[{}] = {}\n", .{ i, prime });
    }
}
```

### Memory Layout ของ Array

Array ใน Zig เก็บข้อมูลแบบ stack-allocated และสมาชิกทุกตัวอยู่ติดกันในหน่วยความจำ:

```
Array: [5]i32 = { 10, 20, 30, 40, 50 }

หน่วยความจำ (แต่ละ i32 = 4 bytes):
┌────────┬────────┬────────┬────────┬────────┐
│  10    │  20    │  30    │  40    │  50    │
│ [0]    │ [1]    │ [2]    │ [3]    │ [4]    │
└────────┴────────┴────────┴────────┴────────┘
 addr+0   addr+4   addr+8  addr+12  addr+16
```

### ตรวจสอบ Bounds

Zig ตรวจสอบ array bounds โดยอัตโนมัติ (ใน Debug/ReleaseSafe mode):

```zig
const std = @import("std");

pub fn main() void {
    const arr = [_]i32{ 1, 2, 3 };
    
    // ต่อไปนี้จะ panic ถ้าเกิด runtime
    // const x = arr[5]; // index 5 out of bounds for array of length 3
    
    // ใช้งานอย่างปลอดภัย
    const idx: usize = 2;
    if (idx < arr.len) {
        std.debug.print("arr[{}] = {}\n", .{ idx, arr[idx] });
    }
}
```

---

## Step 82: Array Initialization - การกำหนดค่าเริ่มต้น

### วิธีต่างๆ ในการ Initialize Array

```zig
const std = @import("std");

pub fn main() void {
    // 1. กำหนดค่าทุกตัว
    const explicit = [5]u32{ 1, 2, 3, 4, 5 };
    
    // 2. ใช้ underscore ให้ Zig นับขนาดเอง
    const inferred = [_]u32{ 10, 20, 30 };
    
    // 3. กำหนดค่าเริ่มต้นเป็น 0 ทุกตัว
    const zeros = [_]u32{0} ** 5;
    // ผลลัพธ์: { 0, 0, 0, 0, 0 }
    
    // 4. ใช้ pattern การ repeat
    const pattern = [_]u8{ 0xFF } ** 8;
    // ผลลัพธ์: { 255, 255, 255, 255, 255, 255, 255, 255 }
    
    // 5. undefined - ไม่กำหนดค่าเริ่มต้น (ระวัง! ค่าอาจเป็นอะไรก็ได้)
    var uninitialized: [10]i32 = undefined;
    // ต้องกำหนดค่าก่อนใช้!
    for (&uninitialized, 0..) |*elem, i| {
        elem.* = @intCast(i * i);
    }
    
    std.debug.print("explicit: {any}\n", .{explicit});
    std.debug.print("inferred: {any}\n", .{inferred});
    std.debug.print("zeros: {any}\n", .{zeros});
    std.debug.print("pattern: {any}\n", .{pattern});
    std.debug.print("squares: {any}\n", .{uninitialized});
}
```

### การสร้าง Array จาก Comptime Computation

```zig
const std = @import("std");

// สร้าง Array ของ fibonacci ณ comptime
fn makeFibArray(comptime n: usize) [n]u64 {
    var result: [n]u64 = undefined;
    if (n > 0) result[0] = 0;
    if (n > 1) result[1] = 1;
    var i: usize = 2;
    while (i < n) : (i += 1) {
        result[i] = result[i - 1] + result[i - 2];
    }
    return result;
}

pub fn main() void {
    const fib10 = comptime makeFibArray(10);
    std.debug.print("Fibonacci 10 ตัวแรก: {any}\n", .{fib10});
    
    // สร้าง Array ของ squares
    const squares = comptime blk: {
        var result: [10]u32 = undefined;
        for (&result, 0..) |*elem, i| {
            elem.* = @intCast(i * i);
        }
        break :blk result;
    };
    std.debug.print("กำลังสอง 0-9: {any}\n", .{squares});
}
```

### การ Modify Array (Mutable Arrays)

```zig
const std = @import("std");

pub fn main() void {
    // ต้องประกาศ var เพื่อให้แก้ไขได้
    var scores: [5]u32 = [_]u32{ 85, 90, 78, 92, 88 };
    
    std.debug.print("ก่อนแก้ไข: {any}\n", .{scores});
    
    // แก้ไขค่า
    scores[2] = 95;
    
    // ใช้ pointer เพื่อแก้ไขผ่าน loop
    for (&scores) |*score| {
        score.* += 5; // เพิ่มคะแนนทุกคน 5 คะแนน
    }
    
    std.debug.print("หลังแก้ไข: {any}\n", .{scores});
    
    // หาค่าเฉลี่ย
    var sum: u32 = 0;
    for (scores) |score| {
        sum += score;
    }
    const average = @as(f64, @floatFromInt(sum)) / @as(f64, @floatFromInt(scores.len));
    std.debug.print("คะแนนเฉลี่ย: {d:.2}\n", .{average});
}
```

### Array Concatenation และ Repetition

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]u32{ 1, 2, 3 };
    const b = [_]u32{ 4, 5, 6 };
    
    // ต่อ Array สองตัวเข้ากัน (เฉพาะ comptime)
    const c = a ++ b;
    std.debug.print("a ++ b = {any}\n", .{c});
    // ผลลัพธ์: { 1, 2, 3, 4, 5, 6 }
    
    // Repeat Array
    const d = a ** 3;
    std.debug.print("a ** 3 = {any}\n", .{d});
    // ผลลัพธ์: { 1, 2, 3, 1, 2, 3, 1, 2, 3 }
    
    // ใช้กับ strings
    const hello = "Hello" ++ ", " ++ "World!";
    std.debug.print("{s}\n", .{hello});
    
    const line = "-" ** 20;
    std.debug.print("{s}\n", .{line});
}
```

---

## Step 83: Multi-dimensional Arrays

### การสร้าง 2D Array

```zig
const std = @import("std");

pub fn main() void {
    // Matrix 3x3
    const matrix: [3][3]i32 = [_][3]i32{
        [_]i32{ 1, 2, 3 },
        [_]i32{ 4, 5, 6 },
        [_]i32{ 7, 8, 9 },
    };
    
    // เข้าถึงสมาชิก
    std.debug.print("matrix[1][2] = {}\n", .{matrix[1][2]}); // 6
    
    // แสดงผลทั้ง Matrix
    std.debug.print("Matrix:\n", .{});
    for (matrix) |row| {
        for (row) |elem| {
            std.debug.print("{:3} ", .{elem});
        }
        std.debug.print("\n", .{});
    }
}
```

### Memory Layout ของ 2D Array

```
Matrix [3][3]i32:

row[0]: │ 1 │ 2 │ 3 │
row[1]: │ 4 │ 5 │ 6 │
row[2]: │ 7 │ 8 │ 9 │

ในหน่วยความจำ (row-major order):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

### Matrix Operations

```zig
const std = @import("std");

// Matrix multiplication ขนาด 2x2
fn matMul2x2(a: [2][2]f64, b: [2][2]f64) [2][2]f64 {
    var result: [2][2]f64 = undefined;
    for (0..2) |i| {
        for (0..2) |j| {
            result[i][j] = 0;
            for (0..2) |k| {
                result[i][j] += a[i][k] * b[k][j];
            }
        }
    }
    return result;
}

fn printMatrix(matrix: [2][2]f64) void {
    for (matrix) |row| {
        std.debug.print("│ ", .{});
        for (row) |elem| {
            std.debug.print("{:6.2} ", .{elem});
        }
        std.debug.print("│\n", .{});
    }
}

pub fn main() void {
    const A: [2][2]f64 = .{
        .{ 1, 2 },
        .{ 3, 4 },
    };
    
    const B: [2][2]f64 = .{
        .{ 5, 6 },
        .{ 7, 8 },
    };
    
    const C = matMul2x2(A, B);
    
    std.debug.print("Matrix A:\n", .{});
    printMatrix(A);
    
    std.debug.print("Matrix B:\n", .{});
    printMatrix(B);
    
    std.debug.print("A × B:\n", .{});
    printMatrix(C);
    // ผลลัพธ์:
    // │  19.00  22.00 │
    // │  43.00  50.00 │
}
```

### 3D Array

```zig
const std = @import("std");

pub fn main() void {
    // 3D array สำหรับ RGB image 2x2 pixels
    // [height][width][channels]
    var image: [2][2][3]u8 = undefined;
    
    // กำหนดสี
    image[0][0] = .{ 255, 0, 0 };   // Red
    image[0][1] = .{ 0, 255, 0 };   // Green
    image[1][0] = .{ 0, 0, 255 };   // Blue
    image[1][1] = .{ 255, 255, 0 }; // Yellow
    
    std.debug.print("Image pixels:\n", .{});
    for (image) |row| {
        for (row) |pixel| {
            std.debug.print("RGB({}, {}, {}) ", .{ pixel[0], pixel[1], pixel[2] });
        }
        std.debug.print("\n", .{});
    }
}
```

### Jagged Arrays (ใช้ Slice)

```zig
const std = @import("std");

pub fn main() void {
    // Array ของ slices (แต่ละแถวมีขนาดต่างกัน)
    const row1 = [_]i32{ 1 };
    const row2 = [_]i32{ 2, 3 };
    const row3 = [_]i32{ 4, 5, 6 };
    
    const triangle: [3][]const i32 = .{
        &row1,
        &row2,
        &row3,
    };
    
    std.debug.print("Triangle array:\n", .{});
    for (triangle, 0..) |row, i| {
        std.debug.print("row[{}]: {any}\n", .{ i, row });
    }
}
```

---

## Step 84: Arrays ใน Comptime

### Comptime Arrays

Zig มีระบบ comptime ที่ทรงพลัง ทำให้เราสร้าง Array ได้ตั้งแต่ตอน compile time โดยไม่เสีย runtime cost:

```zig
const std = @import("std");

// ตารางการคูณ comptime
fn multiplicationTable(comptime n: usize) [n][n]u32 {
    var table: [n][n]u32 = undefined;
    for (0..n) |i| {
        for (0..n) |j| {
            table[i][j] = @intCast((i + 1) * (j + 1));
        }
    }
    return table;
}

pub fn main() void {
    // สร้าง ณ comptime - ไม่มี runtime cost!
    const table = comptime multiplicationTable(5);
    
    std.debug.print("ตารางสูตรคูณ 5x5:\n", .{});
    std.debug.print("   │", .{});
    for (1..6) |i| {
        std.debug.print(" {:2} │", .{i});
    }
    std.debug.print("\n───┼", .{});
    for (0..5) |_| std.debug.print("────┼", .{});
    std.debug.print("\n", .{});
    
    for (table, 0..) |row, i| {
        std.debug.print(" {} │", .{i + 1});
        for (row) |val| {
            std.debug.print(" {:2} │", .{val});
        }
        std.debug.print("\n", .{});
    }
}
```

### Comptime String Tables

```zig
const std = @import("std");

// สร้าง lookup table สำหรับชื่อเดือนภาษาไทย ณ comptime
const MONTHS_TH = comptime blk: {
    break :blk [_][]const u8{
        "มกราคม",
        "กุมภาพันธ์",
        "มีนาคม",
        "เมษายน",
        "พฤษภาคม",
        "มิถุนายน",
        "กรกฎาคม",
        "สิงหาคม",
        "กันยายน",
        "ตุลาคม",
        "พฤศจิกายน",
        "ธันวาคม",
    };
};

const DAYS_TH = comptime [_][]const u8{
    "อาทิตย์",
    "จันทร์",
    "อังคาร",
    "พุธ",
    "พฤหัสบดี",
    "ศุกร์",
    "เสาร์",
};

pub fn main() void {
    const month: usize = 9; // ตุลาคม (0-based)
    const day: usize = 4;   // พฤหัสบดี
    
    std.debug.print("เดือน: {s}\n", .{MONTHS_TH[month]});
    std.debug.print("วัน: {s}\n", .{DAYS_TH[day]});
    
    std.debug.print("\nรายการเดือนทั้งหมด:\n", .{});
    for (MONTHS_TH, 1..) |month_name, num| {
        std.debug.print("{:2}. {s}\n", .{ num, month_name });
    }
}
```

### Comptime Array ด้วย @Type

```zig
const std = @import("std");

// สร้าง lookup table สำหรับ powers of 2
fn powersOf2(comptime n: usize) [n]u64 {
    var result: [n]u64 = undefined;
    for (&result, 0..) |*elem, i| {
        elem.* = @as(u64, 1) << @intCast(i);
    }
    return result;
}

pub fn main() void {
    const POW2 = comptime powersOf2(16);
    
    std.debug.print("Powers of 2:\n", .{});
    for (POW2, 0..) |val, i| {
        std.debug.print("2^{:2} = {:6}\n", .{ i, val });
    }
    
    // ใช้เป็น constant lookup
    const bit_position: usize = 7;
    const bit_value = POW2[bit_position];
    std.debug.print("\n2^{} = {}\n", .{ bit_position, bit_value });
}
```

### Comptime Array Type Generation

```zig
const std = @import("std");

// ฟังก์ชันที่สร้าง Array type ณ comptime
fn ArrayOf(comptime T: type, comptime n: usize) type {
    return [n]T;
}

pub fn main() void {
    // สร้าง type ณ comptime
    const IntArray5 = ArrayOf(i32, 5);
    const FloatArray3 = ArrayOf(f64, 3);
    
    const ints: IntArray5 = .{ 1, 2, 3, 4, 5 };
    const floats: FloatArray3 = .{ 1.1, 2.2, 3.3 };
    
    std.debug.print("ints: {any}\n", .{ints});
    std.debug.print("floats: {any}\n", .{floats});
    
    // ตรวจสอบ type info ณ comptime
    const info = @typeInfo(IntArray5);
    std.debug.print("Array length: {}\n", .{info.array.len});
    std.debug.print("Array child type: {}\n", .{info.array.child});
}
```

---

## Step 85: Slices พื้นฐาน - []T vs [*]T

### Slice คืออะไร?

Slice ใน Zig คือ "มุมมอง" (view) ไปยังส่วนหนึ่งของหน่วยความจำ ประกอบด้วยสองส่วน:
1. **pointer** - ตำแหน่งเริ่มต้นในหน่วยความจำ
2. **length** - จำนวนสมาชิก

```
Slice []T ประกอบด้วย:
┌──────────────────────┬──────────────────┐
│ ptr: [*]T            │ len: usize       │
│ (pointer ไปยัง data) │ (จำนวนสมาชิก)   │
└──────────────────────┴──────────────────┘
ขนาดรวม = 16 bytes (บน 64-bit system)
```

### ความแตกต่างระหว่าง []T และ [*]T

```zig
const std = @import("std");

pub fn main() void {
    const arr = [_]i32{ 10, 20, 30, 40, 50 };
    
    // []T = Slice (มี pointer + length)
    // ปลอดภัยกว่า เพราะรู้ขนาด
    const slice: []const i32 = &arr;
    std.debug.print("slice.len = {}\n", .{slice.len});
    std.debug.print("slice[2] = {}\n", .{slice[2]});
    
    // [*]T = Many-item pointer (มีแค่ pointer ไม่มี length)
    // ต้องรู้ขนาดด้วยตนเอง เหมือน pointer ใน C
    const many_ptr: [*]const i32 = &arr;
    // many_ptr.len ไม่มี! ต้องรู้ขนาดเอง
    std.debug.print("many_ptr[0] = {}\n", .{many_ptr[0]});
    std.debug.print("many_ptr[4] = {}\n", .{many_ptr[4]});
    
    // แปลง [*]T เป็น []T ด้วยการระบุขนาด
    const slice_from_ptr: []const i32 = many_ptr[0..arr.len];
    std.debug.print("slice from ptr len = {}\n", .{slice_from_ptr.len});
}
```

### ประเภทของ Pointer ใน Zig

```zig
// 1. *T - Single-item pointer (pointer ไปยัง item เดียว)
var x: i32 = 42;
const single: *i32 = &x;

// 2. [*]T - Many-item pointer (ไม่รู้จำนวน, เหมือน pointer ใน C)
var arr = [_]i32{ 1, 2, 3 };
const many: [*]i32 = &arr;

// 3. []T - Slice (pointer + length, ปลอดภัย)
const slice: []i32 = &arr;

// 4. [*:sentinel]T - Sentinel-terminated many-item pointer
// ใช้กับ C strings (จบด้วย null)
const cstr: [*:0]const u8 = "hello";
```

### Slice เป็น Fat Pointer

```zig
const std = @import("std");

pub fn main() void {
    const arr = [_]u8{ 1, 2, 3, 4, 5 };
    const slice: []const u8 = &arr;
    
    // ดู internal ของ Slice
    std.debug.print("slice.ptr = {*}\n", .{slice.ptr});
    std.debug.print("slice.len = {}\n", .{slice.len});
    
    // Slice size = 2 * @sizeOf(usize)
    std.debug.print("@sizeOf([]u8) = {}\n", .{@sizeOf([]u8)});
    std.debug.print("@sizeOf(*u8) = {}\n", .{@sizeOf(*u8)});
    std.debug.print("@sizeOf(usize) = {}\n", .{@sizeOf(usize)});
    
    // เทียบกับ Array - Array มีขนาดตามข้อมูล
    std.debug.print("@sizeOf([5]u8) = {}\n", .{@sizeOf([5]u8)});
    std.debug.print("@sizeOf([10]u8) = {}\n", .{@sizeOf([10]u8)});
}
```

### Mutable vs Const Slice

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    
    // []T - mutable slice (สามารถเปลี่ยนค่าได้)
    const mut_slice: []i32 = &arr;
    mut_slice[0] = 100;
    std.debug.print("หลังแก้ไข: {any}\n", .{arr});
    
    // []const T - const slice (อ่านอย่างเดียว)
    const const_slice: []const i32 = &arr;
    // const_slice[0] = 200; // Error! cannot assign to const slice
    std.debug.print("const_slice[0] = {}\n", .{const_slice[0]});
    
    // pointer เป็น const vs ข้อมูลเป็น const
    // *const T = pointer ที่เปลี่ยนไม่ได้ แต่ข้อมูลเปลี่ยนได้
    // *T const = pointer ที่เปลี่ยนได้ แต่ข้อมูลเปลี่ยนไม่ได้ (ไม่มีใน Zig)
}
```

---

## Step 86: การสร้าง Slice จาก Array

### วิธีต่างๆ ในการสร้าง Slice

```zig
const std = @import("std");

pub fn main() void {
    const arr = [_]u32{ 10, 20, 30, 40, 50, 60, 70 };
    
    // 1. Slice ทั้ง Array
    const full_slice: []const u32 = &arr;
    std.debug.print("full: {any}\n", .{full_slice});
    
    // 2. Slice บางส่วน [start..end] (รวม start, ไม่รวม end)
    const partial: []const u32 = arr[1..4];
    std.debug.print("arr[1..4]: {any}\n", .{partial}); // { 20, 30, 40 }
    
    // 3. Slice จากต้น
    const from_start: []const u32 = arr[0..3];
    std.debug.print("arr[0..3]: {any}\n", .{from_start}); // { 10, 20, 30 }
    
    // 4. Slice ถึงท้าย
    const to_end: []const u32 = arr[4..];
    std.debug.print("arr[4..]: {any}\n", .{to_end}); // { 50, 60, 70 }
    
    // 5. Slice ทั้งหมดด้วย arr[0..arr.len]
    const all: []const u32 = arr[0..arr.len];
    std.debug.print("all: {any}\n", .{all});
    
    // 6. Sentinel slice (จะอธิบายใน Step 88)
    const with_sentinel: [:0]const u8 = "hello";
    _ = with_sentinel;
}
```

### Slice ที่ Overlap กัน

```zig
const std = @import("std");

pub fn main() void {
    var data = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    
    // สองก Slice ที่ชี้ไปยัง Array เดียวกัน
    const first_half: []u8 = data[0..4];
    const second_half: []u8 = data[4..8];
    
    std.debug.print("first_half: {any}\n", .{first_half});
    std.debug.print("second_half: {any}\n", .{second_half});
    
    // แก้ไขผ่าน first_half จะกระทบ data
    first_half[0] = 100;
    std.debug.print("data หลังแก้ไข first_half[0]: {any}\n", .{data});
    
    // Sliding window
    std.debug.print("\nSliding window ขนาด 3:\n", .{});
    var i: usize = 0;
    while (i + 3 <= data.len) : (i += 1) {
        const window = data[i .. i + 3];
        std.debug.print("window[{}..{}]: {any}\n", .{ i, i + 3, window });
    }
}
```

### สร้าง Slice จาก Pointer

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 5, 3, 8, 1, 9, 2, 7 };
    
    // สร้าง slice จาก pointer
    const ptr: *i32 = &arr[2]; // ชี้ไปที่ arr[2] = 8
    
    // แปลง single pointer เป็น slice
    const slice_of_one: []i32 = @as([*]i32, @ptrCast(ptr))[0..1];
    std.debug.print("slice_of_one: {any}\n", .{slice_of_one}); // { 8 }
    
    // สร้าง slice จาก element ที่ 2 ไปถึงสิ้นสุด
    const from_elem2: []i32 = arr[2..];
    std.debug.print("from_elem2: {any}\n", .{from_elem2}); // { 8, 1, 9, 2, 7 }
    
    // ตรวจสอบว่า slice ชี้ไปที่ไหน
    std.debug.print("arr.ptr = {*}\n", .{&arr[0]});
    std.debug.print("from_elem2.ptr = {*}\n", .{from_elem2.ptr});
}
```

### Slice ใน Heap (ใช้ Allocator)

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // จัดสรรหน่วยความจำสำหรับ slice
    const heap_slice: []u32 = try allocator.alloc(u32, 10);
    defer allocator.free(heap_slice);
    
    // กำหนดค่า
    for (heap_slice, 0..) |*elem, i| {
        elem.* = @intCast(i * 10);
    }
    
    std.debug.print("heap_slice: {any}\n", .{heap_slice});
    std.debug.print("heap_slice.len: {}\n", .{heap_slice.len});
}
```

---

## Step 87: Slice Operations (len, ptr, indexing)

### คุณสมบัติและ Operations ของ Slice

```zig
const std = @import("std");

pub fn main() void {
    var numbers = [_]i32{ 10, 20, 30, 40, 50 };
    const slice: []i32 = &numbers;
    
    // 1. .len - ความยาวของ slice
    std.debug.print("len: {}\n", .{slice.len});
    
    // 2. .ptr - pointer ไปยังข้อมูล
    std.debug.print("ptr: {*}\n", .{slice.ptr});
    
    // 3. Indexing
    std.debug.print("slice[0]: {}\n", .{slice[0]});
    std.debug.print("slice[4]: {}\n", .{slice[slice.len - 1]});
    
    // 4. Slicing a slice
    const sub_slice = slice[1..3];
    std.debug.print("slice[1..3]: {any}\n", .{sub_slice});
    
    // 5. เปรียบเทียบ slices
    const a: []const i32 = &[_]i32{ 1, 2, 3 };
    const b: []const i32 = &[_]i32{ 1, 2, 3 };
    
    // Pointer comparison (ไม่ใช่ value comparison)
    std.debug.print("a.ptr == b.ptr: {}\n", .{a.ptr == b.ptr}); // false (คนละที่)
    
    // ใช้ std.mem.eql สำหรับ value comparison
    std.debug.print("std.mem.eql: {}\n", .{std.mem.eql(i32, a, b)}); // true
}
```

### การ Iterate Slice

```zig
const std = @import("std");

pub fn main() void {
    const fruits = [_][]const u8{ "แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง" };
    const slice: []const []const u8 = &fruits;
    
    // 1. for loop แบบ value
    std.debug.print("ผลไม้ทั้งหมด:\n", .{});
    for (slice) |fruit| {
        std.debug.print("  - {s}\n", .{fruit});
    }
    
    // 2. for loop พร้อม index
    std.debug.print("\nพร้อม index:\n", .{});
    for (slice, 0..) |fruit, i| {
        std.debug.print("  [{}] {s}\n", .{ i, fruit });
    }
    
    // 3. while loop ด้วย index
    std.debug.print("\nด้วย while:\n", .{});
    var i: usize = 0;
    while (i < slice.len) : (i += 1) {
        std.debug.print("  fruits[{}] = {s}\n", .{ i, slice[i] });
    }
    
    // 4. for loop พร้อม pointer (สำหรับ mutable slice)
    var scores = [_]u32{ 70, 80, 90, 85 };
    const score_slice: []u32 = &scores;
    
    for (score_slice) |*score| {
        score.* = score.* + 10; // เพิ่ม 10 ทุกตัว
    }
    std.debug.print("\nคะแนนหลังเพิ่ม: {any}\n", .{score_slice});
}
```

### Slice ใน Functions

```zig
const std = @import("std");

// รับ slice เป็น parameter (efficient - ไม่ copy ข้อมูล)
fn sum(numbers: []const i32) i64 {
    var total: i64 = 0;
    for (numbers) |n| {
        total += n;
    }
    return total;
}

fn max(numbers: []const i32) ?i32 {
    if (numbers.len == 0) return null;
    var maximum = numbers[0];
    for (numbers[1..]) |n| {
        if (n > maximum) maximum = n;
    }
    return maximum;
}

fn reverse(slice: []i32) void {
    var left: usize = 0;
    var right: usize = slice.len - 1;
    while (left < right) {
        const temp = slice[left];
        slice[left] = slice[right];
        slice[right] = temp;
        left += 1;
        right -= 1;
    }
}

fn contains(slice: []const i32, value: i32) bool {
    for (slice) |elem| {
        if (elem == value) return true;
    }
    return false;
}

pub fn main() void {
    var data = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6, 5, 3 };
    const slice: []i32 = &data;
    
    std.debug.print("ข้อมูล: {any}\n", .{slice});
    std.debug.print("ผลรวม: {}\n", .{sum(slice)});
    std.debug.print("ค่าสูงสุด: {?}\n", .{max(slice)});
    std.debug.print("มี 9 ไหม: {}\n", .{contains(slice, 9)});
    std.debug.print("มี 7 ไหม: {}\n", .{contains(slice, 7)});
    
    reverse(slice);
    std.debug.print("กลับหัว: {any}\n", .{slice});
}
```

### Slice ของ Struct

```zig
const std = @import("std");

const Student = struct {
    name: []const u8,
    score: f64,
};

fn averageScore(students: []const Student) f64 {
    if (students.len == 0) return 0;
    var total: f64 = 0;
    for (students) |s| {
        total += s.score;
    }
    return total / @as(f64, @floatFromInt(students.len));
}

fn topStudent(students: []const Student) ?Student {
    if (students.len == 0) return null;
    var top = students[0];
    for (students[1..]) |s| {
        if (s.score > top.score) top = s;
    }
    return top;
}

pub fn main() void {
    const class = [_]Student{
        .{ .name = "สมชาย", .score = 85.5 },
        .{ .name = "สมหญิง", .score = 92.0 },
        .{ .name = "สมศรี", .score = 78.3 },
        .{ .name = "สมพงษ์", .score = 96.5 },
    };
    
    const students: []const Student = &class;
    
    std.debug.print("นักเรียนทั้งหมด:\n", .{});
    for (students) |s| {
        std.debug.print("  {s}: {d:.1}\n", .{ s.name, s.score });
    }
    
    std.debug.print("คะแนนเฉลี่ย: {d:.2}\n", .{averageScore(students)});
    
    if (topStudent(students)) |top| {
        std.debug.print("นักเรียนยอดเยี่ยม: {s} ({d:.1})\n", .{ top.name, top.score });
    }
}
```

---

## Step 88: Sentinel-terminated Arrays/Slices [*:0]u8

### Sentinel-terminated คืออะไร?

Sentinel-terminated array/slice คือ array ที่มี "ตัวบอกจบ" (sentinel value) ต่อท้าย ตัวอย่างที่พบบ่อยที่สุดคือ C string ที่จบด้วย null byte (`\0`)

```
C String "hello\0":
┌───┬───┬───┬───┬───┬───┐
│ h │ e │ l │ l │ o │\0 │
└───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5

*:0 หมายถึง: pointer ที่ชี้ไปยังข้อมูลที่จบด้วย null (0)
```

### ประเภทต่างๆ ของ Sentinel-terminated

```zig
const std = @import("std");

pub fn main() void {
    // [N:0]u8 - Array ที่จบด้วย null, รู้ขนาด ณ comptime
    const arr: [5:0]u8 = "hello".*;
    std.debug.print("arr.len = {} (ไม่นับ null)\n", .{arr.len}); // 5
    std.debug.print("arr[5] = {} (null byte)\n", .{arr[5]}); // 0
    
    // [*:0]u8 - Many-item pointer จบด้วย null (เหมือน char* ใน C)
    const c_style: [*:0]const u8 = "world";
    // นับความยาวต้อง iterate ถึง null
    var len: usize = 0;
    while (c_style[len] != 0) : (len += 1) {}
    std.debug.print("c_style length: {}\n", .{len}); // 5
    
    // [:0]u8 - Slice ที่รู้ขนาดและมี null terminator
    const sentinel_slice: [:0]const u8 = "sentinel";
    std.debug.print("sentinel_slice.len = {}\n", .{sentinel_slice.len}); // 8
    std.debug.print("null at end: {}\n", .{sentinel_slice[8]}); // 0
    
    // String literals ใน Zig เป็น *const [N:0]u8
    const greeting = "สวัสดี";
    std.debug.print("type of string literal: {}\n", .{@TypeOf(greeting)});
}
```

### String Literals ใน Zig

```zig
const std = @import("std");

pub fn main() void {
    // String literal = *const [N:0]u8
    const str1 = "Hello";
    // type: *const [5:0]u8
    
    // แปลงเป็น []const u8 (slice ไม่มี sentinel)
    const str2: []const u8 = str1;
    
    // แปลงเป็น [:0]const u8 (slice มี sentinel)
    const str3: [:0]const u8 = str1;
    
    std.debug.print("str1: {s} (len: {})\n", .{ str1, str1.len });
    std.debug.print("str2: {s} (len: {})\n", .{ str2, str2.len });
    std.debug.print("str3: {s} (len: {})\n", .{ str3, str3.len });
    
    // ตรวจสอบ null terminator
    std.debug.print("str3[5] = {} (null)\n", .{str3[5]});
    
    // multiline string
    const multiline =
        \\บรรทัดที่ 1
        \\บรรทัดที่ 2
        \\บรรทัดที่ 3
    ;
    std.debug.print("{s}\n", .{multiline});
}
```

### การทำงานกับ C Strings

```zig
const std = @import("std");

// ฟังก์ชันที่รับ C string
fn cstrlen(s: [*:0]const u8) usize {
    var i: usize = 0;
    while (s[i] != 0) : (i += 1) {}
    return i;
}

fn cstrprint(s: [*:0]const u8) void {
    var i: usize = 0;
    while (s[i] != 0) : (i += 1) {
        std.debug.print("{c}", .{s[i]});
    }
    std.debug.print("\n", .{});
}

pub fn main() void {
    const msg: [*:0]const u8 = "Hello from C-style!";
    
    std.debug.print("ความยาว: {}\n", .{cstrlen(msg)});
    cstrprint(msg);
    
    // แปลง [*:0]const u8 เป็น []const u8
    const slice = std.mem.span(msg);
    std.debug.print("slice: {s} (len: {})\n", .{ slice, slice.len });
    
    // สร้าง sentinel slice จาก array
    var buf: [32:0]u8 = undefined;
    const written = std.fmt.bufPrint(&buf, "สวัสดีโลก!", .{}) catch unreachable;
    buf[written.len] = 0; // ใส่ null terminator
    
    std.debug.print("buf (as c string): {s}\n", .{&buf});
}
```

### Sentinel-terminated Slices ในทางปฏิบัติ

```zig
const std = @import("std");

pub fn main() void {
    // ใช้กับ std.process.args
    // (โดยทั่วไป args จะส่งมาเป็น [][*:0]u8)
    
    // ตัวอย่างการแปลงระหว่าง types
    const original: [:0]const u8 = "test string";
    
    // [:0]const u8 -> []const u8 (ใช้ได้เลย)
    const as_slice: []const u8 = original;
    
    // []const u8 -> [:0]const u8 (ต้องระวัง)
    // ถ้าแน่ใจว่ามี null terminator
    // const back: [:0]const u8 = original[0..original.len :0];
    
    std.debug.print("original: {s}\n", .{original});
    std.debug.print("as_slice: {s}\n", .{as_slice});
    
    // ตัวอย่าง: parse null-terminated string
    const env_like: [*:0]const u8 = "PATH=/usr/bin:/bin";
    const env_slice = std.mem.span(env_like);
    
    if (std.mem.indexOf(u8, env_slice, "=")) |eq_pos| {
        const key = env_slice[0..eq_pos];
        const value = env_slice[eq_pos + 1 ..];
        std.debug.print("Key: {s}\n", .{key});
        std.debug.print("Value: {s}\n", .{value});
    }
}
```

---

## Step 89: Array vs Slice - เมื่อไหรควรใช้อะไร?

### ตารางเปรียบเทียบ

| คุณสมบัติ | Array [N]T | Slice []T |
|-----------|-----------|-----------|
| ขนาด | รู้ ณ comptime | รู้ ณ runtime |
| ที่เก็บ | Stack (โดยปกติ) | ชี้ไปที่ไหนก็ได้ |
| ขนาดใน stack | N × sizeof(T) | 2 × sizeof(usize) |
| Copy semantics | Copy ทั้ง array | Copy แค่ ptr+len |
| Bounds checking | ณ comptime ถ้าทำได้ | ณ runtime |
| ใช้เป็น parameter | ไม่นิยม (copy ทั้ง array) | นิยม (efficient) |

### เมื่อไหรควรใช้ Array

```zig
const std = @import("std");

// 1. เมื่อขนาดคงที่และรู้ตั้งแต่ต้น
const RGB = [3]u8; // เสมอมี 3 channels
const IPv4 = [4]u8; // เสมอมี 4 octets
const ChessBoard = [8][8]u8; // กระดาน Chess 8x8

// 2. เมื่อต้องการ stack allocation สำหรับข้อมูลขนาดเล็ก
fn computeHash(data: []const u8) [32]u8 {
    var hash: [32]u8 = [_]u8{0} ** 32;
    // ... compute hash
    for (data, 0..) |byte, i| {
        hash[i % 32] ^= byte;
    }
    return hash; // Return by value OK เพราะขนาดคงที่
}

// 3. เมื่อต้องการ comptime-known size สำหรับ optimization
fn processFixed(data: [16]f32) [16]f32 {
    var result: [16]f32 = undefined;
    for (data, 0..) |v, i| {
        result[i] = v * 2.0;
    }
    return result;
}

pub fn main() void {
    const color: RGB = .{ 255, 128, 0 }; // Orange
    const ip: IPv4 = .{ 192, 168, 1, 1 };
    
    std.debug.print("Color: RGB({}, {}, {})\n", .{ color[0], color[1], color[2] });
    std.debug.print("IP: {}.{}.{}.{}\n", .{ ip[0], ip[1], ip[2], ip[3] });
    
    const hash = computeHash("hello world");
    std.debug.print("Hash (first 8 bytes): {any}\n", .{hash[0..8]});
}
```

### เมื่อไหรควรใช้ Slice

```zig
const std = @import("std");

// 1. เมื่อขนาดไม่แน่นอนหรือกำหนดตอน runtime
fn processVariable(data: []const f64) f64 {
    var sum: f64 = 0;
    for (data) |v| sum += v;
    return sum / @as(f64, @floatFromInt(data.len));
}

// 2. เมื่อต้องการ function ที่รับ array ขนาดต่างๆ
fn printAll(items: []const i32) void {
    for (items, 0..) |item, i| {
        if (i > 0) std.debug.print(", ", .{});
        std.debug.print("{}", .{item});
    }
    std.debug.print("\n", .{});
}

// 3. เมื่อทำงานกับ heap memory
fn createRange(allocator: std.mem.Allocator, start: usize, end: usize) ![]usize {
    const len = end - start;
    const result = try allocator.alloc(usize, len);
    for (result, 0..) |*elem, i| {
        elem.* = start + i;
    }
    return result;
}

// 4. เมื่อต้องการ view ส่วนหนึ่งของข้อมูล (ไม่ต้อง copy)
fn skipFirstN(data: []const u8, n: usize) []const u8 {
    if (n >= data.len) return data[data.len..];
    return data[n..];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const small = [_]i32{ 1, 2, 3 };
    const large = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    
    // Function เดียวรับทั้งสองได้!
    printAll(&small);
    printAll(&large);
    
    // Heap-allocated slice
    const range = try createRange(allocator, 1, 11);
    defer allocator.free(range);
    std.debug.print("Range: {any}\n", .{range});
    
    // Skip operation ไม่ copy ข้อมูล
    const text: []const u8 = "Hello, World!";
    const skipped = skipFirstN(text, 7);
    std.debug.print("After skip: {s}\n", .{skipped}); // "World!"
}
```

### Array vs Slice: Copy Semantics

```zig
const std = @import("std");

pub fn main() void {
    // Array - copy ทั้ง array
    const arr1 = [_]i32{ 1, 2, 3, 4, 5 };
    var arr2 = arr1; // Copy ทั้ง 5 elements!
    arr2[0] = 100;
    
    std.debug.print("arr1[0] = {} (ไม่เปลี่ยน)\n", .{arr1[0]}); // 1
    std.debug.print("arr2[0] = {} (เปลี่ยน)\n", .{arr2[0]}); // 100
    
    // Slice - copy แค่ ptr+len
    var data = [_]i32{ 1, 2, 3, 4, 5 };
    const s1: []i32 = &data;
    const s2: []i32 = s1; // Copy แค่ ptr+len (8-16 bytes)
    s2[0] = 100;
    
    std.debug.print("s1[0] = {} (เปลี่ยนด้วย!)\n", .{s1[0]}); // 100
    std.debug.print("data[0] = {} (เปลี่ยนด้วย!)\n", .{data[0]}); // 100
    
    // ระวัง! Slice หลายตัวชี้ไปที่เดียวกัน
    std.debug.print("\nส่ง slice ไปยัง function:\n", .{});
    std.debug.print("ก่อน: {any}\n", .{data});
    
    // ส่งแค่ ptr+len ไม่ copy data
}
```

---

## Step 90: std.mem Functions กับ Arrays/Slices

### std.mem Overview

`std.mem` เป็น module ที่มีฟังก์ชันสำหรับทำงานกับหน่วยความจำ arrays และ slices:

```zig
const std = @import("std");
const mem = std.mem;

pub fn main() void {
    var arr = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    const slice: []u8 = &arr;
    
    // แสดงข้อมูลเบื้องต้น
    std.debug.print("ข้อมูลเริ่มต้น: {any}\n", .{slice});
}
```

### std.mem.copy และ std.mem.copyForwards/Backwards

```zig
const std = @import("std");

pub fn main() void {
    const source = [_]u32{ 10, 20, 30, 40, 50 };
    var destination: [5]u32 = undefined;
    
    // copyForwards: copy จากซ้ายไปขวา (ปลอดภัยเมื่อ dst อยู่หลัง src)
    @memcpy(&destination, &source);
    std.debug.print("หลัง copy: {any}\n", .{destination});
    
    // สำหรับ overlapping memory ใช้ copyForwards หรือ copyBackwards
    var data = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8 };
    
    // ย้ายข้อมูล 3 bytes แรกไปที่ตำแหน่ง 2
    // (overlapping: src=data[0..3], dst=data[2..5])
    std.mem.copyForwards(u8, data[2..5], data[0..3]);
    std.debug.print("หลัง copyForwards: {any}\n", .{data});
    // { 1, 2, 1, 2, 3, 6, 7, 8 }
}
```

### std.mem.eql - เปรียบเทียบ Slices

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{ 1, 2, 3, 4, 5 };
    const b = [_]i32{ 1, 2, 3, 4, 5 };
    const c = [_]i32{ 1, 2, 3, 4, 6 };
    
    // เปรียบเทียบค่า
    std.debug.print("a == b: {}\n", .{std.mem.eql(i32, &a, &b)}); // true
    std.debug.print("a == c: {}\n", .{std.mem.eql(i32, &a, &c)}); // false
    
    // เปรียบเทียบ strings
    const s1: []const u8 = "hello";
    const s2: []const u8 = "hello";
    const s3: []const u8 = "world";
    
    std.debug.print("s1 == s2: {}\n", .{std.mem.eql(u8, s1, s2)}); // true
    std.debug.print("s1 == s3: {}\n", .{std.mem.eql(u8, s1, s3)}); // false
    
    // เปรียบเทียบ case-insensitive (ต้องทำเอง)
    const upper = "HELLO";
    const lower = "hello";
    
    // ตรวจสอบทีละตัว
    var same = upper.len == lower.len;
    if (same) {
        for (upper, lower) |u, l| {
            if (std.ascii.toLower(u) != std.ascii.toLower(l)) {
                same = false;
                break;
            }
        }
    }
    std.debug.print("case-insensitive equal: {}\n", .{same}); // true
}
```

### std.mem.indexOf และ std.mem.lastIndexOf

```zig
const std = @import("std");

pub fn main() void {
    const text: []const u8 = "The quick brown fox jumps over the lazy dog";
    
    // หาตำแหน่งของ substring
    if (std.mem.indexOf(u8, text, "fox")) |pos| {
        std.debug.print("พบ 'fox' ที่ตำแหน่ง {}\n", .{pos});
    }
    
    // หาจากท้าย
    if (std.mem.lastIndexOf(u8, text, "the")) |pos| {
        std.debug.print("พบ 'the' ครั้งสุดท้ายที่ตำแหน่ง {}\n", .{pos});
    }
    
    // หาไม่เจอ
    if (std.mem.indexOf(u8, text, "cat")) |_| {
        std.debug.print("พบ 'cat'\n", .{});
    } else {
        std.debug.print("ไม่พบ 'cat'\n", .{});
    }
    
    // หา byte เดียว
    if (std.mem.indexOfScalar(u8, text, 'q')) |pos| {
        std.debug.print("พบ 'q' ที่ตำแหน่ง {}\n", .{pos});
    }
    
    // นับจำนวนครั้งที่พบ
    const haystack = "banana";
    var count: usize = 0;
    var start: usize = 0;
    while (std.mem.indexOfPos(u8, haystack, start, "an")) |pos| {
        count += 1;
        start = pos + 1;
    }
    std.debug.print("พบ 'an' ใน 'banana': {} ครั้ง\n", .{count});
}
```

### std.mem.startsWith และ std.mem.endsWith

```zig
const std = @import("std");

pub fn main() void {
    const filename = "document.pdf";
    
    // ตรวจสอบ prefix
    if (std.mem.startsWith(u8, filename, "doc")) {
        std.debug.print("ไฟล์เริ่มด้วย 'doc'\n", .{});
    }
    
    // ตรวจสอบ suffix
    if (std.mem.endsWith(u8, filename, ".pdf")) {
        std.debug.print("ไฟล์เป็น PDF\n", .{});
    }
    
    // ตัวอย่าง: filter files by extension
    const files = [_][]const u8{
        "main.zig",
        "build.zig",
        "README.md",
        "config.json",
        "lib.zig",
    };
    
    std.debug.print("\nไฟล์ Zig:\n", .{});
    for (files) |file| {
        if (std.mem.endsWith(u8, file, ".zig")) {
            std.debug.print("  {s}\n", .{file});
        }
    }
}
```

### std.mem.sort

```zig
const std = @import("std");

fn lessThan(context: void, a: i32, b: i32) bool {
    _ = context;
    return a < b;
}

fn greaterThan(context: void, a: i32, b: i32) bool {
    _ = context;
    return a > b;
}

pub fn main() void {
    var numbers = [_]i32{ 5, 2, 8, 1, 9, 3, 7, 4, 6 };
    
    std.debug.print("ก่อน sort: {any}\n", .{numbers});
    
    // เรียงจากน้อยไปมาก
    std.mem.sort(i32, &numbers, {}, lessThan);
    std.debug.print("เรียงน้อย->มาก: {any}\n", .{numbers});
    
    // เรียงจากมากไปน้อย
    std.mem.sort(i32, &numbers, {}, greaterThan);
    std.debug.print("เรียงมาก->น้อย: {any}\n", .{numbers});
    
    // Sort strings
    var names = [_][]const u8{ "Charlie", "Alice", "Bob", "David" };
    
    std.mem.sort([]const u8, &names, {}, struct {
        fn lessThan(ctx: void, a: []const u8, b: []const u8) bool {
            _ = ctx;
            return std.mem.lessThan(u8, a, b);
        }
    }.lessThan);
    
    std.debug.print("\nชื่อเรียง: {s}\n", .{names});
}
```

### std.mem.set และ @memset

```zig
const std = @import("std");

pub fn main() void {
    var buffer: [10]u8 = undefined;
    
    // ตั้งค่าทุกตัวเป็น 0
    @memset(&buffer, 0);
    std.debug.print("หลัง @memset(0): {any}\n", .{buffer});
    
    // ตั้งค่าทุกตัวเป็น 0xFF
    @memset(&buffer, 0xFF);
    std.debug.print("หลัง @memset(0xFF): {any}\n", .{buffer});
    
    // ตั้งค่าส่วนหนึ่ง
    @memset(buffer[3..7], 42);
    std.debug.print("หลัง @memset(buffer[3..7], 42): {any}\n", .{buffer});
    
    // สร้าง string ซ้ำ
    var line: [20]u8 = undefined;
    @memset(&line, '-');
    line[19] = '\n';
    std.debug.print("{s}", .{&line});
}
```

### std.mem.span - แปลง C string เป็น Slice

```zig
const std = @import("std");

pub fn main() void {
    // แปลง null-terminated string เป็น slice
    const c_str: [*:0]const u8 = "Hello, Zig!";
    const slice = std.mem.span(c_str);
    
    std.debug.print("c_str length (manual): ", .{});
    var len: usize = 0;
    while (c_str[len] != 0) : (len += 1) {}
    std.debug.print("{}\n", .{len});
    
    std.debug.print("slice.len: {}\n", .{slice.len});
    std.debug.print("slice: {s}\n", .{slice});
    
    // ใช้กับ environment variables (simulation)
    const env_vars: []const [*:0]const u8 = &[_][*:0]const u8{
        "HOME=/home/user",
        "PATH=/usr/bin:/bin",
        "LANG=th_TH.UTF-8",
    };
    
    std.debug.print("\nEnvironment variables:\n", .{});
    for (env_vars) |env| {
        const env_slice = std.mem.span(env);
        std.debug.print("  {s}\n", .{env_slice});
    }
}
```

### std.mem.split และ std.mem.tokenize

```zig
const std = @import("std");

pub fn main() void {
    // split: แยกด้วย delimiter (ผลที่ได้อาจเป็น string ว่าง)
    const csv = "apple,banana,,cherry,";
    var split_iter = std.mem.split(u8, csv, ",");
    
    std.debug.print("split:\n", .{});
    while (split_iter.next()) |item| {
        std.debug.print("  '{s}'\n", .{item});
    }
    
    // tokenize: แยกและข้ามส่วนที่ว่าง
    std.debug.print("\ntokenize:\n", .{});
    var token_iter = std.mem.tokenize(u8, csv, ",");
    while (token_iter.next()) |token| {
        std.debug.print("  '{s}'\n", .{token});
    }
    
    // แยก path
    const path = "/usr/local/bin/zig";
    std.debug.print("\nส่วนประกอบของ path:\n", .{});
    var path_iter = std.mem.split(u8, path, "/");
    while (path_iter.next()) |part| {
        if (part.len > 0) {
            std.debug.print("  '{s}'\n", .{part});
        }
    }
}
```

### std.mem.replace

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const original = "Hello World, Hello Zig!";
    
    // นับว่ามีกี่ครั้ง
    const count = std.mem.count(u8, original, "Hello");
    std.debug.print("พบ 'Hello' {} ครั้ง\n", .{count});
    
    // Replace ทุกตำแหน่ง
    const replaced = try std.mem.replaceOwned(u8, allocator, original, "Hello", "Hi");
    defer allocator.free(replaced);
    
    std.debug.print("ก่อน: {s}\n", .{original});
    std.debug.print("หลัง: {s}\n", .{replaced});
}
```

### std.mem.trim

```zig
const std = @import("std");

pub fn main() void {
    const input = "  \t  สวัสดีครับ  \n  ";
    
    // ตัด whitespace จากซ้ายและขวา
    const trimmed = std.mem.trim(u8, input, " \t\n\r");
    std.debug.print("ต้นฉบับ: '{s}'\n", .{input});
    std.debug.print("หลัง trim: '{s}'\n", .{trimmed});
    
    // ตัดแค่ซ้าย
    const left_trimmed = std.mem.trimLeft(u8, input, " \t\n\r");
    std.debug.print("ตัดซ้าย: '{s}'\n", .{left_trimmed});
    
    // ตัดแค่ขวา
    const right_trimmed = std.mem.trimRight(u8, input, " \t\n\r");
    std.debug.print("ตัดขวา: '{s}'\n", .{right_trimmed});
    
    // ตัด characters เฉพาะ
    const numbers_only = std.mem.trim(u8, "###data###", "#");
    std.debug.print("ตัด #: '{s}'\n", .{numbers_only}); // "data"
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Matrix Operations

```zig
// TODO: เขียนฟังก์ชันต่อไปนี้
// 1. transpose: สลับแถวและคอลัมน์ของ matrix 3x3
// 2. trace: หาผลรวมของ diagonal elements
// 3. isIdentity: ตรวจสอบว่าเป็น identity matrix หรือไม่

const std = @import("std");

fn transpose(matrix: [3][3]f64) [3][3]f64 {
    // เขียนโค้ดที่นี่
    var result: [3][3]f64 = undefined;
    for (0..3) |i| {
        for (0..3) |j| {
            result[j][i] = matrix[i][j];
        }
    }
    return result;
}

fn trace(matrix: [3][3]f64) f64 {
    // เขียนโค้ดที่นี่
    return matrix[0][0] + matrix[1][1] + matrix[2][2];
}

fn isIdentity(matrix: [3][3]f64) bool {
    // เขียนโค้ดที่นี่
    for (0..3) |i| {
        for (0..3) |j| {
            const expected: f64 = if (i == j) 1.0 else 0.0;
            if (matrix[i][j] != expected) return false;
        }
    }
    return true;
}

pub fn main() void {
    const A: [3][3]f64 = .{
        .{ 1, 2, 3 },
        .{ 4, 5, 6 },
        .{ 7, 8, 9 },
    };
    
    const I: [3][3]f64 = .{
        .{ 1, 0, 0 },
        .{ 0, 1, 0 },
        .{ 0, 0, 1 },
    };
    
    const AT = transpose(A);
    
    std.debug.print("Transpose of A:\n", .{});
    for (AT) |row| {
        std.debug.print("{any}\n", .{row});
    }
    
    std.debug.print("Trace of A: {}\n", .{trace(A)});
    std.debug.print("A is identity: {}\n", .{isIdentity(A)});
    std.debug.print("I is identity: {}\n", .{isIdentity(I)});
}
```

### แบบฝึกหัดที่ 2: Slice Algorithms

```zig
// TODO: เขียน algorithms ต่อไปนี้ด้วย slices
// 1. binarySearch: หาค่าใน sorted slice
// 2. bubbleSort: เรียงลำดับ slice
// 3. removeDuplicates: ลบค่าซ้ำ (assume sorted)

const std = @import("std");

fn binarySearch(sorted: []const i32, target: i32) ?usize {
    var left: usize = 0;
    var right: usize = sorted.len;
    
    while (left < right) {
        const mid = left + (right - left) / 2;
        if (sorted[mid] == target) {
            return mid;
        } else if (sorted[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    return null;
}

fn bubbleSort(arr: []i32) void {
    const n = arr.len;
    var i: usize = 0;
    while (i < n) : (i += 1) {
        var j: usize = 0;
        while (j < n - i - 1) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

pub fn main() void {
    const sorted = [_]i32{ 2, 5, 8, 12, 16, 23, 38, 56, 72, 91 };
    
    const targets = [_]i32{ 23, 7, 91, 1 };
    for (targets) |target| {
        if (binarySearch(&sorted, target)) |idx| {
            std.debug.print("พบ {} ที่ index {}\n", .{ target, idx });
        } else {
            std.debug.print("ไม่พบ {}\n", .{target});
        }
    }
    
    var unsorted = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    std.debug.print("\nก่อน sort: {any}\n", .{unsorted});
    bubbleSort(&unsorted);
    std.debug.print("หลัง sort: {any}\n", .{unsorted});
}
```

### แบบฝึกหัดที่ 3: String Operations

```zig
// TODO: เขียนฟังก์ชันทำงานกับ strings (slice ของ bytes)
// 1. isPalindrome: ตรวจสอบว่าเป็น palindrome หรือไม่
// 2. countWords: นับจำนวนคำ
// 3. reverseWords: กลับลำดับคำ

const std = @import("std");

fn isPalindrome(s: []const u8) bool {
    if (s.len == 0) return true;
    var left: usize = 0;
    var right: usize = s.len - 1;
    while (left < right) {
        if (s[left] != s[right]) return false;
        left += 1;
        right -= 1;
    }
    return true;
}

fn countWords(text: []const u8) usize {
    var count: usize = 0;
    var in_word = false;
    for (text) |c| {
        if (c == ' ' or c == '\t' or c == '\n') {
            in_word = false;
        } else if (!in_word) {
            in_word = true;
            count += 1;
        }
    }
    return count;
}

pub fn main() void {
    // Test palindrome
    const words = [_][]const u8{ "racecar", "hello", "level", "world", "madam" };
    for (words) |word| {
        std.debug.print("{s}: {s}\n", .{
            word,
            if (isPalindrome(word)) "palindrome" else "ไม่ใช่ palindrome",
        });
    }
    
    // Test word count
    const text = "  The quick   brown fox  jumps over  ";
    std.debug.print("\nข้อความ: '{s}'\n", .{text});
    std.debug.print("จำนวนคำ: {}\n", .{countWords(text)});
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 09

**Arrays [N]T:**
- ขนาดคงที่ กำหนด ณ comptime
- เก็บบน stack ข้อมูลอยู่ติดกัน
- Copy ทั้ง array เมื่อ assign
- เหมาะสำหรับข้อมูลขนาดคงที่ เช่น RGB, IPv4, hash

**Slices []T:**
- ประกอบด้วย pointer + length (fat pointer)
- ชี้ไปยัง array หรือ heap memory
- Copy แค่ ptr+len เมื่อ assign (ไม่ copy data)
- เหมาะสำหรับ function parameters และข้อมูลขนาดไม่แน่นอน

**Sentinel-terminated [:sentinel]T:**
- มี sentinel value ต่อท้าย (เช่น null byte)
- ใช้กับ C interop และ strings
- `[*:0]const u8` เหมือน `char*` ใน C
- String literals เป็น `*const [N:0]u8`

**std.mem:**
- `eql`: เปรียบเทียบ slices
- `copy/copyForwards/copyBackwards`: คัดลอก
- `indexOf/lastIndexOf`: หาตำแหน่ง
- `startsWith/endsWith`: ตรวจสอบ prefix/suffix
- `sort`: เรียงลำดับ
- `split/tokenize`: แยก string
- `trim/trimLeft/trimRight`: ตัด whitespace
- `span`: แปลง C string เป็น slice

### Memory Layout สรุป

```
Array [5]i32:             Slice []i32:
┌─────────────────────┐   ┌──────────┬──────────┐
│ 1 │ 2 │ 3 │ 4 │ 5   │   │ ptr      │ len = 5  │
│ (20 bytes on stack) │   │ (16 bytes│ on stack)│
└─────────────────────┘   └──────────┴──────────┘
                                  │
                                  ▼
                          ┌─────────────────────┐
                          │ 1 │ 2 │ 3 │ 4 │ 5   │
                          │ (ที่ไหนก็ได้ใน RAM) │
                          └─────────────────────┘
```

### Quick Reference

```zig
// Array
const arr: [5]i32 = .{ 1, 2, 3, 4, 5 };
const arr2 = [_]i32{ 1, 2, 3 }; // inferred size
const filled = [_]u8{0} ** 100; // fill with value

// Slice
const slice: []i32 = &arr;
const part = arr[1..3];  // partial slice
const all = arr[0..];    // full slice

// Sentinel
const cstr: [*:0]const u8 = "hello";
const sslice: [:0]const u8 = "world";

// Common patterns
for (slice) |elem| { ... }              // iterate values
for (slice, 0..) |elem, i| { ... }      // with index
for (&slice) |*elem| { elem.* = ...; }  // mutable
```

---

ในบทต่อไป (Part 10) เราจะเรียนรู้เกี่ยวกับ **Structs และ Enums** ใน Zig ซึ่งเป็นพื้นฐานสำหรับการสร้างโครงสร้างข้อมูลที่ซับซ้อนยิ่งขึ้น!
