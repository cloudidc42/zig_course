# ภาค 04: ชนิดข้อมูลพื้นฐานใน Zig (Steps 31–40)

> **ระดับ**: ผู้เริ่มต้น–ระดับกลาง  
> **เวลาโดยประมาณ**: 3–4 ชั่วโมง  
> **สิ่งที่ต้องรู้ก่อน**: ภาค 01–03 (การติดตั้ง, โครงสร้างโปรแกรม, ตัวแปรและ const)

---

## บทนำ

ชนิดข้อมูล (Type) คือรากฐานของภาษา Zig ทุกค่าในโปรแกรม Zig มีชนิดข้อมูลที่แน่นอนซึ่งคอมไพเลอร์รู้ล่วงหน้าตั้งแต่เวลา compile Zig เป็นภาษาที่มี **type system แบบ static และ strong** หมายความว่า:

1. ชนิดข้อมูลถูกตรวจสอบตอน compile ไม่ใช่ตอน runtime
2. การแปลงชนิดข้อมูล (type conversion) ต้องทำอย่างชัดเจน (explicit)
3. ไม่มี implicit conversion ที่ซ่อนเร้นเหมือนภาษา C

ในภาคนี้เราจะเรียนรู้ชนิดข้อมูลพื้นฐาน (primitive types) ทั้งหมดของ Zig ตั้งแต่ integer, float, boolean ไปจนถึงการจัดการ overflow และ bitwise operations

---

## Step 31: Integer Types — จำนวนเต็มแบบมีเครื่องหมาย (Signed Integers)

### ความหมายของ Signed Integer

**Signed integer** คือจำนวนเต็มที่สามารถเป็นค่าบวกหรือลบได้ ใน Zig ชื่อชนิดข้อมูลขึ้นต้นด้วย `i` ตามด้วยจำนวนบิต เช่น `i8` คือ signed integer ขนาด 8 บิต

### ตารางชนิด Signed Integer มาตรฐาน

| ชนิดข้อมูล | ขนาด (บิต) | ขนาด (bytes) | ค่าต่ำสุด | ค่าสูงสุด |
|-----------|-----------|-------------|----------|----------|
| `i8`      | 8         | 1           | -128     | 127      |
| `i16`     | 16        | 2           | -32,768  | 32,767   |
| `i32`     | 32        | 4           | -2,147,483,648 | 2,147,483,647 |
| `i64`     | 64        | 8           | -9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |
| `i128`    | 128       | 16          | -170,141,183,460,469,231,731,687,303,715,884,105,728 | 170,141,183,460,469,231,731,687,303,715,884,105,727 |
| `isize`   | platform  | 4 หรือ 8    | ขึ้นอยู่กับ platform | ขึ้นอยู่กับ platform |

### สูตรคำนวณช่วงค่า

สำหรับ signed integer ขนาด N บิต:
- ค่าต่ำสุด = -(2^(N-1))
- ค่าสูงสุด = 2^(N-1) - 1

ตัวอย่าง: `i8` (8 บิต)
- ค่าต่ำสุด = -(2^7) = -128
- ค่าสูงสุด = 2^7 - 1 = 127

### ตัวอย่างโค้ด: การประกาศและใช้งาน Signed Integers

```zig
const std = @import("std");

pub fn main() void {
    // การประกาศ signed integer แต่ละชนิด
    const a: i8 = -100;
    const b: i16 = -30000;
    const c: i32 = -2_000_000;    // ใช้ _ คั่นตัวเลขเพื่อให้อ่านง่าย
    const d: i64 = -9_000_000_000;
    const e: i128 = -170_000_000_000_000_000_000_000_000_000_000_000_000;

    // isize ขึ้นอยู่กับ platform (32-bit หรือ 64-bit)
    const f: isize = -1000;

    std.debug.print("i8:    {}\n", .{a});
    std.debug.print("i16:   {}\n", .{b});
    std.debug.print("i32:   {}\n", .{c});
    std.debug.print("i64:   {}\n", .{d});
    std.debug.print("i128:  {}\n", .{e});
    std.debug.print("isize: {}\n", .{f});
}
```

### การใช้ค่าสูงสุด/ต่ำสุดจาก std.math

```zig
const std = @import("std");

pub fn main() void {
    // ดึงค่า min/max จาก standard library
    const min_i8 = std.math.minInt(i8);
    const max_i8 = std.math.maxInt(i8);
    const min_i32 = std.math.minInt(i32);
    const max_i32 = std.math.maxInt(i32);
    const min_i64 = std.math.minInt(i64);
    const max_i64 = std.math.maxInt(i64);

    std.debug.print("i8  range: {} ถึง {}\n", .{ min_i8, max_i8 });
    std.debug.print("i32 range: {} ถึง {}\n", .{ min_i32, max_i32 });
    std.debug.print("i64 range: {} ถึง {}\n", .{ min_i64, max_i64 });
}
```

### isize และการใช้งานจริง

`isize` เป็น signed integer ที่มีขนาดเท่ากับ pointer ของ platform:
- บน 32-bit system: เท่ากับ `i32`
- บน 64-bit system: เท่ากับ `i64`

ใช้ `isize` เมื่อต้องการแทน:
- ความแตกต่างระหว่าง pointer สองตัว (pointer arithmetic)
- ดัชนีที่อาจเป็นค่าลบ
- ขนาดที่ต้องการ signed

```zig
const std = @import("std");

pub fn main() void {
    const arr = [_]i32{ 10, 20, 30, 40, 50 };

    // isize เหมาะสำหรับ signed index
    var i: isize = @as(isize, @intCast(arr.len)) - 1;
    while (i >= 0) : (i -= 1) {
        std.debug.print("arr[{}] = {}\n", .{ i, arr[@intCast(i)] });
    }
}
```

---

## Step 32: Unsigned Integers — จำนวนเต็มไม่มีเครื่องหมาย

### ความหมายของ Unsigned Integer

**Unsigned integer** คือจำนวนเต็มที่เป็นค่าบวกหรือศูนย์เท่านั้น ไม่มีค่าลบ ใน Zig ขึ้นต้นด้วย `u` ตามด้วยจำนวนบิต เช่น `u8` คือ unsigned integer ขนาด 8 บิต

### ตารางชนิด Unsigned Integer มาตรฐาน

| ชนิดข้อมูล | ขนาด (บิต) | ขนาด (bytes) | ค่าต่ำสุด | ค่าสูงสุด |
|-----------|-----------|-------------|----------|----------|
| `u8`      | 8         | 1           | 0        | 255      |
| `u16`     | 16        | 2           | 0        | 65,535   |
| `u32`     | 32        | 4           | 0        | 4,294,967,295 |
| `u64`     | 64        | 8           | 0        | 18,446,744,073,709,551,615 |
| `u128`    | 128       | 16          | 0        | 340,282,366,920,938,463,463,374,607,431,768,211,455 |
| `usize`   | platform  | 4 หรือ 8    | 0        | ขึ้นอยู่กับ platform |

### สูตรคำนวณช่วงค่า

สำหรับ unsigned integer ขนาด N บิต:
- ค่าต่ำสุด = 0
- ค่าสูงสุด = 2^N - 1

ตัวอย่าง: `u8` (8 บิต)
- ค่าต่ำสุด = 0
- ค่าสูงสุด = 2^8 - 1 = 255

### ตัวอย่างโค้ด: Unsigned Integers

```zig
const std = @import("std");

pub fn main() void {
    // การประกาศ unsigned integer แต่ละชนิด
    const a: u8 = 255;
    const b: u16 = 65_535;
    const c: u32 = 4_294_967_295;
    const d: u64 = 18_446_744_073_709_551_615;

    // usize ใช้บ่อยมากสำหรับ array index และขนาด
    const arr = [_]u32{ 1, 2, 3, 4, 5 };
    const len: usize = arr.len;

    std.debug.print("u8  max: {}\n", .{a});
    std.debug.print("u16 max: {}\n", .{b});
    std.debug.print("u32 max: {}\n", .{c});
    std.debug.print("u64 max: {}\n", .{d});
    std.debug.print("array length: {}\n", .{len});

    // การใช้ u8 กับข้อมูล byte
    const byte_data: u8 = 0xFF;  // เลขฐาน 16
    const binary_data: u8 = 0b11001010;  // เลขฐาน 2
    const octal_data: u8 = 0o177;  // เลขฐาน 8

    std.debug.print("hex: {}\n", .{byte_data});
    std.debug.print("binary: {}\n", .{binary_data});
    std.debug.print("octal: {}\n", .{octal_data});
}
```

### usize และการใช้งานจริง

`usize` เป็น unsigned integer ที่มีขนาดเท่ากับ pointer — นี่คือชนิดข้อมูลที่ Zig ใช้สำหรับ:
- ดัชนี (index) ของ array และ slice
- ขนาด (size/length) ของ container
- การคำนวณ memory address

```zig
const std = @import("std");

pub fn main() void {
    const data = [_]u32{ 100, 200, 300, 400, 500 };

    // usize เป็น default สำหรับ array index
    var i: usize = 0;
    while (i < data.len) : (i += 1) {
        std.debug.print("data[{}] = {}\n", .{ i, data[i] });
    }

    // การหาขนาดของชนิดข้อมูล
    const size_of_u32 = @sizeOf(u32);
    const size_of_u64 = @sizeOf(u64);
    std.debug.print("u32 size: {} bytes\n", .{size_of_u32});
    std.debug.print("u64 size: {} bytes\n", .{size_of_u64});
}
```

### เปรียบเทียบ i vs u: เลือกใช้อะไรดี?

| สถานการณ์ | ชนิดที่แนะนำ |
|----------|------------|
| ดัชนี array | `usize` |
| ขนาด/ความยาว | `usize` |
| ค่าที่ไม่มีทางเป็นลบ | unsigned (`u8`, `u32`, ...) |
| ค่าที่อาจเป็นลบ | signed (`i8`, `i32`, ...) |
| การแทนสีใน RGB | `u8` |
| อุณหภูมิ | `i32` หรือ `f32` |
| timestamp | `i64` หรือ `u64` |

---

## Step 33: Arbitrary Width Integers — Integer ความกว้างอิสระ

### คุณสมบัติเฉพาะของ Zig

หนึ่งในคุณสมบัติที่น่าทึ่งของ Zig คือ **arbitrary width integers** — คุณสามารถสร้าง integer ที่มีขนาดกี่บิตก็ได้ตั้งแต่ 0 ถึง 65535 บิต!

ไม่ใช่แค่ 8, 16, 32, 64 — แต่เป็น **ทุกขนาด** เช่น:
- `i1` — signed 1 bit (ค่าได้ -1 หรือ 0)
- `u1` — unsigned 1 bit (ค่าได้ 0 หรือ 1)
- `u3` — unsigned 3 bit (ค่า 0–7)
- `i7` — signed 7 bit (ค่า -64 ถึง 63)
- `u24` — unsigned 24 bit (ค่า 0–16,777,215)
- `i256` — signed 256 bit
- `u4096` — unsigned 4096 bit

### ตัวอย่างขนาดที่ไม่ปกติ

| ชนิด | บิต | ค่าต่ำสุด | ค่าสูงสุด | ใช้งานทั่วไป |
|------|-----|----------|----------|------------|
| `u1`  | 1   | 0        | 1        | แทน boolean ใน bit-packed struct |
| `u2`  | 2   | 0        | 3        | 4 สถานะ |
| `u3`  | 3   | 0        | 7        | 8 สถานะ |
| `u4`  | 4   | 0        | 15       | nibble (ครึ่ง byte) |
| `u7`  | 7   | 0        | 127      | ASCII character |
| `u24` | 24  | 0        | 16,777,215 | RGB color (8 bit per channel) |
| `i24` | 24  | -8,388,608 | 8,388,607 | audio samples |
| `u48` | 48  | 0        | 281,474,976,710,655 | MAC address |

### ตัวอย่างโค้ด: Arbitrary Width Integers

```zig
const std = @import("std");

pub fn main() void {
    // u1: เก็บได้แค่ 0 หรือ 1
    const flag: u1 = 1;
    std.debug.print("u1 flag: {}\n", .{flag});

    // u3: เก็บ 0-7 (3 บิต)
    const small: u3 = 7;
    std.debug.print("u3 max: {}\n", .{small});

    // i7: signed 7-bit (-64 ถึง 63)
    const mid: i7 = -64;
    std.debug.print("i7 min: {}\n", .{mid});

    // u24: เหมาะกับ RGB color
    const color_r: u8 = 255;
    const color_g: u8 = 128;
    const color_b: u8 = 0;

    // รวม RGB เป็น u24 เดียว
    const rgb: u24 = (@as(u24, color_r) << 16) |
                     (@as(u24, color_g) << 8) |
                     @as(u24, color_b);
    std.debug.print("RGB color: #{X:0>6}\n", .{rgb});

    // i256: ตัวเลขขนาดใหญ่มาก
    const big: i256 = 1_000_000_000_000_000_000_000_000_000_000;
    std.debug.print("i256: {}\n", .{big});
}
```

### การใช้ใน Packed Struct (บันทึก Memory)

```zig
const std = @import("std");

// Packed struct ใช้ arbitrary-width integer เพื่อประหยัด memory
const Flags = packed struct {
    is_active: u1,
    is_admin: u1,
    permission_level: u3,  // 0-7 (8 ระดับ)
    priority: u3,          // 0-7 (8 ระดับ)
    // รวมทั้งหมด = 8 บิต = 1 byte เท่านั้น!
};

pub fn main() void {
    var flags = Flags{
        .is_active = 1,
        .is_admin = 0,
        .permission_level = 5,
        .priority = 3,
    };

    std.debug.print("Size of Flags: {} byte(s)\n", .{@sizeOf(Flags)});
    std.debug.print("is_active: {}\n", .{flags.is_active});
    std.debug.print("permission_level: {}\n", .{flags.permission_level});

    flags.is_admin = 1;
    std.debug.print("is_admin (after): {}\n", .{flags.is_admin});
}
```

### ข้อควรระวัง

- Arbitrary-width integers จะถูก align ให้เป็น power-of-2 เมื่ออยู่ใน struct ปกติ
- ใน `packed struct` จะใช้พื้นที่จริงตามบิตที่กำหนด
- การคำนวณ performance อาจต่างกันบนฮาร์ดแวร์ต่างๆ

---

## Step 34: Float Types — ชนิดข้อมูลทศนิยม

### มาตรฐาน IEEE 754

Zig รองรับ floating-point types ตามมาตรฐาน **IEEE 754** ซึ่งเป็นมาตรฐานสากลสำหรับการแทนเลขทศนิยมในคอมพิวเตอร์

### ตารางชนิด Float ทั้งหมด

| ชนิดข้อมูล | ขนาด (บิต) | ขนาด (bytes) | ความแม่นยำ (decimal digits) | มาตรฐาน |
|-----------|-----------|-------------|--------------------------|--------|
| `f16`     | 16        | 2           | ~3.3 หลัก               | IEEE 754 half |
| `f32`     | 32        | 4           | ~7.2 หลัก               | IEEE 754 single |
| `f64`     | 64        | 8           | ~15.9 หลัก              | IEEE 754 double |
| `f80`     | 80        | 10          | ~18.5 หลัก              | x86 extended |
| `f128`    | 128       | 16          | ~34.0 หลัก              | IEEE 754 quad |

### โครงสร้างภายในของ f32 และ f64

**f32 (32 บิต)**:
- 1 บิต: เครื่องหมาย (sign)
- 8 บิต: exponent
- 23 บิต: mantissa (fraction)

**f64 (64 บิต)**:
- 1 บิต: เครื่องหมาย (sign)
- 11 บิต: exponent
- 52 บิต: mantissa (fraction)

### ตัวอย่างโค้ด: Float Types

```zig
const std = @import("std");

pub fn main() void {
    // การประกาศ float แต่ละชนิด
    const a: f16 = 3.14;
    const b: f32 = 3.14159265;
    const c: f64 = 3.14159265358979323846;
    const d: f128 = 3.14159265358979323846264338327950288;

    // แสดงผลด้วย format specifier ที่ต่างกัน
    std.debug.print("f16:  {d:.4}\n", .{a});
    std.debug.print("f32:  {d:.8}\n", .{b});
    std.debug.print("f64:  {d:.16}\n", .{c});
    std.debug.print("f128: {d:.34}\n", .{d});

    // ค่าพิเศษ: infinity และ NaN
    const pos_inf: f64 = std.math.inf(f64);
    const neg_inf: f64 = -std.math.inf(f64);
    const nan_val: f64 = std.math.nan(f64);

    std.debug.print("positive infinity: {}\n", .{pos_inf});
    std.debug.print("negative infinity: {}\n", .{neg_inf});
    std.debug.print("NaN: {}\n", .{nan_val});

    // ตรวจสอบค่าพิเศษ
    std.debug.print("is inf: {}\n", .{std.math.isInf(pos_inf)});
    std.debug.print("is nan: {}\n", .{std.math.isNan(nan_val)});
    std.debug.print("is finite: {}\n", .{std.math.isFinite(b)});
}
```

### ตัวอย่างโค้ด: การคำนวณทางวิทยาศาสตร์

```zig
const std = @import("std");
const math = std.math;

pub fn main() void {
    const pi: f64 = math.pi;
    const e: f64 = math.e;

    std.debug.print("pi = {d:.10}\n", .{pi});
    std.debug.print("e  = {d:.10}\n", .{e});

    // ฟังก์ชันทางคณิตศาสตร์
    const angle_rad: f64 = pi / 4.0;  // 45 องศา
    std.debug.print("sin(45°) = {d:.6}\n", .{math.sin(angle_rad)});
    std.debug.print("cos(45°) = {d:.6}\n", .{math.cos(angle_rad)});
    std.debug.print("tan(45°) = {d:.6}\n", .{math.tan(angle_rad)});

    // รากที่สอง
    const x: f64 = 2.0;
    std.debug.print("sqrt(2) = {d:.10}\n", .{math.sqrt(x)});

    // ลอการิทึม
    std.debug.print("log(e) = {d:.10}\n", .{math.log(f64, math.e, math.e)});
    std.debug.print("log2(8) = {d:.1}\n", .{math.log2(8.0)});

    // ยกกำลัง
    std.debug.print("2^10 = {d:.0}\n", .{math.pow(f64, 2.0, 10.0)});
}
```

### ความแม่นยำและการเลือกใช้

```zig
const std = @import("std");

pub fn main() void {
    // ปัญหาความแม่นยำของ float
    const a: f32 = 0.1;
    const b: f32 = 0.2;
    const sum_f32: f32 = a + b;

    const c: f64 = 0.1;
    const d: f64 = 0.2;
    const sum_f64: f64 = c + d;

    // 0.1 + 0.2 ไม่ได้เท่ากับ 0.3 พอดี!
    std.debug.print("f32: 0.1 + 0.2 = {d:.20}\n", .{sum_f32});
    std.debug.print("f64: 0.1 + 0.2 = {d:.20}\n", .{sum_f64});

    // การเปรียบเทียบ float ต้องใช้ epsilon
    const epsilon: f64 = 1e-9;
    const is_equal = @abs(sum_f64 - 0.3) < epsilon;
    std.debug.print("0.1 + 0.2 ≈ 0.3: {}\n", .{is_equal});
}
```

### คำแนะนำการเลือก Float Type

| สถานการณ์ | ชนิดที่แนะนำ |
|----------|------------|
| การคำนวณทั่วไป | `f64` |
| Game graphics, shader | `f32` |
| Machine learning weights | `f32` หรือ `f16` |
| Financial calculation (ห้ามใช้ float!) | ใช้ integer แทน |
| Scientific computing สูงสุด | `f128` |
| Embedded system ประหยัด memory | `f32` หรือ `f16` |

---

## Step 35: Boolean Type — ชนิดข้อมูลตรรกศาสตร์

### ความหมายของ bool

`bool` คือชนิดข้อมูลที่มีค่าได้เพียงสองค่า: `true` หรือ `false`

- ขนาด: **1 byte** (8 บิต) แม้จะใช้แค่ 1 บิตก็ตาม
- ใน `packed struct` สามารถใช้ `u1` เพื่อประหยัดพื้นที่

### ตัวอย่างโค้ด: Boolean Operations

```zig
const std = @import("std");

pub fn main() void {
    // การประกาศและใช้งาน bool
    const is_running: bool = true;
    const is_paused: bool = false;
    var is_complete: bool = false;

    std.debug.print("is_running: {}\n", .{is_running});
    std.debug.print("is_paused: {}\n", .{is_paused});

    // Logical operators
    const and_result = is_running and !is_paused;  // AND
    const or_result = is_running or is_paused;      // OR
    const not_result = !is_running;                 // NOT

    std.debug.print("running AND not_paused: {}\n", .{and_result});
    std.debug.print("running OR paused: {}\n", .{or_result});
    std.debug.print("NOT running: {}\n", .{not_result});

    // Bool จากการเปรียบเทียบ
    const x: i32 = 42;
    const y: i32 = 100;

    const eq = x == y;      // เท่ากับ
    const neq = x != y;     // ไม่เท่ากับ
    const lt = x < y;       // น้อยกว่า
    const gt = x > y;       // มากกว่า
    const lte = x <= y;     // น้อยกว่าหรือเท่ากับ
    const gte = x >= y;     // มากกว่าหรือเท่ากับ

    std.debug.print("{} == {}: {}\n", .{ x, y, eq });
    std.debug.print("{} != {}: {}\n", .{ x, y, neq });
    std.debug.print("{} < {}: {}\n", .{ x, y, lt });
    std.debug.print("{} > {}: {}\n", .{ x, y, gt });
    std.debug.print("{} <= {}: {}\n", .{ x, y, lte });
    std.debug.print("{} >= {}: {}\n", .{ x, y, gte });

    // การใช้ bool ใน if
    if (is_running and !is_complete) {
        std.debug.print("Program is actively running\n", .{});
        is_complete = true;
    }

    std.debug.print("is_complete: {}\n", .{is_complete});
}
```

### Short-circuit Evaluation

```zig
const std = @import("std");

fn expensive_check() bool {
    std.debug.print("expensive_check() ถูกเรียก!\n", .{});
    return true;
}

fn is_valid_index(i: usize, len: usize) bool {
    return i < len;
}

pub fn main() void {
    // Short-circuit AND: ถ้า condition แรกเป็น false จะไม่เรียก condition ที่สอง
    const result1 = false and expensive_check();  // expensive_check ไม่ถูกเรียก!
    std.debug.print("false AND ...: {}\n", .{result1});

    std.debug.print("---\n", .{});

    // Short-circuit OR: ถ้า condition แรกเป็น true จะไม่เรียก condition ที่สอง
    const result2 = true or expensive_check();  // expensive_check ไม่ถูกเรียก!
    std.debug.print("true OR ...: {}\n", .{result2});

    std.debug.print("---\n", .{});

    // ใช้ประโยชน์ short-circuit เพื่อ safe array access
    const arr = [_]i32{ 1, 2, 3 };
    const idx: usize = 5;

    // ตรวจสอบ boundary ก่อนเสมอ
    if (is_valid_index(idx, arr.len) and arr[idx] > 0) {
        std.debug.print("valid positive element\n", .{});
    } else {
        std.debug.print("index {} is out of bounds\n", .{idx});
    }
}
```

### ขนาดของ bool

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("sizeof bool: {} byte\n", .{@sizeOf(bool)});
    std.debug.print("sizeof u1:   {} byte\n", .{@sizeOf(u1)});  // ยังคง 1 byte ใน regular context

    // ใน packed struct u1 ใช้แค่ 1 bit
    const S1 = struct { a: bool, b: bool, c: bool };
    const S2 = packed struct { a: u1, b: u1, c: u1, _pad: u5 };

    std.debug.print("struct with bool fields: {} bytes\n", .{@sizeOf(S1)});
    std.debug.print("packed struct with u1 fields: {} byte\n", .{@sizeOf(S2)});
}
```

---

## Step 36: Character และ Unicode — ตัวอักษรและ Unicode

### การแทนตัวอักษรใน Zig

Zig ไม่มีชนิดข้อมูล `char` แบบ C แต่ใช้:
- `u8` สำหรับ ASCII byte (0–127)
- `u21` สำหรับ Unicode code point (0–1,114,111)
- String literal เป็น `[]const u8` (slice ของ bytes)

### ASCII และ Unicode

**ASCII**: มาตรฐานเก่า ใช้ 7 บิต รองรับตัวอักษรภาษาอังกฤษ ตัวเลข และสัญลักษณ์พื้นฐาน (128 ตัวอักษร)

**Unicode**: มาตรฐานใหม่ รองรับภาษาทั่วโลก มากกว่า 140,000 ตัวอักษร

**UTF-8**: การ encoding ที่ใช้บ่อยที่สุด ใช้ 1–4 bytes ต่อ character:
- ASCII (U+0000 ถึง U+007F): 1 byte
- ภาษาตะวันออกกลาง (U+0080 ถึง U+07FF): 2 bytes
- ภาษาเอเชีย รวมถึงภาษาไทย (U+0800 ถึง U+FFFF): 3 bytes
- Emoji และอื่นๆ (U+10000 ถึง U+10FFFF): 4 bytes

### ตัวอย่างโค้ด: Characters และ Bytes

```zig
const std = @import("std");

pub fn main() void {
    // Character literal ใช้ single quote
    const char_a: u8 = 'A';       // ASCII 65
    const char_z: u8 = 'z';       // ASCII 122
    const char_0: u8 = '0';       // ASCII 48
    const newline: u8 = '\n';     // escape sequence
    const tab: u8 = '\t';
    const backslash: u8 = '\\';

    std.debug.print("'A' = {}\n", .{char_a});
    std.debug.print("'z' = {}\n", .{char_z});
    std.debug.print("'0' = {}\n", .{char_0});
    std.debug.print("newline code = {}\n", .{newline});
    std.debug.print("tab code = {}\n", .{tab});
    std.debug.print("backslash code = {}\n", .{backslash});

    // การแปลง char เป็นตัวพิมพ์เล็ก/ใหญ่
    const upper: u8 = 'A';
    const lower: u8 = upper + 32;  // 'a' = 'A' + 32
    std.debug.print("'A' to lowercase: {c}\n", .{lower});

    // ตรวจสอบว่าเป็น digit หรือ letter
    const ch: u8 = '7';
    const is_digit = ch >= '0' and ch <= '9';
    const is_upper = ch >= 'A' and ch <= 'Z';
    const is_lower = ch >= 'a' and ch <= 'z';

    std.debug.print("'{c}' is digit: {}\n", .{ ch, is_digit });
    std.debug.print("'{c}' is uppercase: {}\n", .{ ch, is_upper });
    std.debug.print("'{c}' is lowercase: {}\n", .{ ch, is_lower });
}
```

### ตัวอย่างโค้ด: Unicode Characters

```zig
const std = @import("std");

pub fn main() void {
    // Unicode character literal ใช้ '\u{XXXX}'
    const thai_ko: u21 = 'ก';          // U+0E01 (ก ไก่)
    const emoji_smile: u21 = '😀';    // U+1F600
    const kanji: u21 = '漢';           // U+6F22

    std.debug.print("Thai 'ก' code point: U+{X:0>4}\n", .{thai_ko});
    std.debug.print("Emoji '😀' code point: U+{X:0>4}\n", .{emoji_smile});
    std.debug.print("Kanji '漢' code point: U+{X:0>4}\n", .{kanji});

    // String literal เป็น []const u8 (UTF-8 bytes)
    const hello_thai: []const u8 = "สวัสดี";
    const hello_eng: []const u8 = "Hello";

    // len คือจำนวน bytes ไม่ใช่จำนวนตัวอักษร!
    std.debug.print("Thai string bytes: {}\n", .{hello_thai.len});
    std.debug.print("English string bytes: {}\n", .{hello_eng.len});

    // แสดงแต่ละ byte ของสตริงไทย
    std.debug.print("Bytes of 'สวัสดี': ", .{});
    for (hello_thai) |byte| {
        std.debug.print("{X:0>2} ", .{byte});
    }
    std.debug.print("\n", .{});
}
```

### การนับ Unicode Characters อย่างถูกต้อง

```zig
const std = @import("std");

pub fn main() void {
    const text: []const u8 = "Hello สวัสดี 👋";

    // len = จำนวน bytes
    std.debug.print("byte length: {}\n", .{text.len});

    // นับ Unicode code points
    var char_count: usize = 0;
    var utf8_iter = std.unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    while (utf8_iter.nextCodepoint()) |_| {
        char_count += 1;
    }
    std.debug.print("character count: {}\n", .{char_count});
}
```

### Escape Sequences สำคัญ

| Escape | ความหมาย | ASCII Code |
|--------|---------|-----------|
| `\n`   | Newline | 10 |
| `\r`   | Carriage Return | 13 |
| `\t`   | Tab | 9 |
| `\\`   | Backslash | 92 |
| `\'`   | Single quote | 39 |
| `\"`   | Double quote | 34 |
| `\0`   | Null | 0 |
| `\xNN` | Hex byte | NN (hex) |
| `\u{NNNN}` | Unicode point | |

---

## Step 37: void และ noreturn — ชนิดพิเศษ

### void: ไม่มีค่า

`void` คือชนิดข้อมูลที่แทนความว่างเปล่า — ใช้เมื่อฟังก์ชันไม่ return ค่าใดๆ

```zig
const std = @import("std");

// ฟังก์ชันที่ return void (ไม่มีค่า return)
fn printMessage(msg: []const u8) void {
    std.debug.print("{s}\n", .{msg});
}

// void ยังใช้ใน error union ได้
fn doSomething() !void {
    // อาจ return error หรือไม่ return ค่า
    std.debug.print("doing something\n", .{});
}

pub fn main() void {
    // ขนาดของ void คือ 0 bytes!
    std.debug.print("sizeof void: {} bytes\n", .{@sizeOf(void)});

    // เรียกฟังก์ชันที่ return void
    printMessage("Hello from printMessage");

    // เรียกฟังก์ชันที่ return !void
    doSomething() catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };

    // void ใน HashMap เป็น Set pattern
    // (แสดงแนวคิด — ใช้ HashMap(Key, void) แทน Set)
    std.debug.print("void เป็นชนิด 0-byte ที่มีประโยชน์\n", .{});
}
```

### void ใน HashMap สร้าง Set

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // HashMap(K, void) ทำหน้าที่เป็น Set
    var set = std.HashMap([]const u8, void, std.hash_map.StringContext, 80).init(allocator);
    defer set.deinit();

    // เพิ่มสมาชิก
    try set.put("apple", {});
    try set.put("banana", {});
    try set.put("cherry", {});
    try set.put("apple", {});  // ซ้ำ — ไม่มีผล

    std.debug.print("set size: {}\n", .{set.count()});
    std.debug.print("has 'apple': {}\n", .{set.contains("apple")});
    std.debug.print("has 'grape': {}\n", .{set.contains("grape")});
}
```

### noreturn: ไม่มีทางกลับ

`noreturn` คือชนิดข้อมูลสำหรับฟังก์ชันที่ **ไม่เคย return** ไม่ว่าจะด้วยเหตุใด:
- ออกจากโปรแกรม (`std.process.exit`)
- throw error ตลอดเวลา
- loop ไม่มีสิ้นสุด

```zig
const std = @import("std");

// ฟังก์ชันที่ return noreturn — หยุดโปรแกรมเสมอ
fn fatalError(msg: []const u8) noreturn {
    std.debug.print("FATAL: {s}\n", .{msg});
    std.process.exit(1);
}

// ฟังก์ชันที่ทำงานต่อเนื่องไม่มีสิ้นสุด
fn infiniteLoop() noreturn {
    while (true) {
        // server event loop
    }
}

// panic ก็เป็น noreturn
fn panicExample(x: i32) i32 {
    if (x < 0) {
        @panic("x must be non-negative!");  // noreturn
    }
    return x * 2;
}

pub fn main() void {
    const value: i32 = 5;
    const result = panicExample(value);
    std.debug.print("result: {}\n", .{result});

    // ถ้า uncomment บรรทัดด้านล่าง โปรแกรมจะ crash
    // const bad_result = panicExample(-1);
    // _ = bad_result;

    // noreturn มีประโยชน์กับ if/switch เพราะ compiler รู้ว่า branch นั้นไม่ return
    const check: bool = false;
    const x: i32 = if (check) 42 else blk: {
        // ถ้าเราใส่ fatalError() ที่นี่ compiler จะไม่บ่นเรื่อง type
        break :blk 0;
    };
    std.debug.print("x = {}\n", .{x});
}
```

### ความแตกต่างระหว่าง void และ noreturn

| ลักษณะ | `void` | `noreturn` |
|--------|--------|-----------|
| ขนาด | 0 bytes | N/A |
| return value | ไม่มีค่า แต่ return | ไม่ return เลย |
| ใช้ใน function | return type ปกติ | return type ที่ไม่เคย return |
| ตัวอย่าง | void function | exit(), panic(), infinite loop |

---

## Step 38: Type Coercion และ Casting

### Type Coercion: การแปลงอัตโนมัติ (จำกัด)

Zig **ไม่มี** implicit type conversion กว้างๆ แบบ C แต่มี coercion แบบ "safe" บางอย่าง:

1. Integer ชนิดย่อยสามารถ coerce ไปยังชนิดใหญ่กว่า
2. `comptime_int` สามารถ coerce เป็น integer ชนิดใดก็ได้
3. `comptime_float` สามารถ coerce เป็น float ชนิดใดก็ได้

```zig
const std = @import("std");

pub fn main() void {
    // Coercion อัตโนมัติ: i8 -> i16 -> i32 (safe — ไม่มีข้อมูลหาย)
    const small: i8 = 100;
    const medium: i16 = small;  // OK: coercion อัตโนมัติ
    const large: i32 = medium;  // OK: coercion อัตโนมัติ

    std.debug.print("small={}, medium={}, large={}\n", .{ small, medium, large });

    // comptime integer literal
    const x: i32 = 42;    // 42 เป็น comptime_int, coerce เป็น i32
    const y: u64 = 1000;  // 1000 coerce เป็น u64

    std.debug.print("x={}, y={}\n", .{ x, y });
}
```

### @as: การ Cast ชัดเจน

`@as(T, value)` แปลงค่าเป็นชนิด T อย่างชัดเจน ใช้เมื่อ Zig ไม่ทำให้อัตโนมัติ

```zig
const std = @import("std");

pub fn main() void {
    // @as: explicit type annotation/cast
    const a: i32 = 300;

    // ต้องใช้ @as เพราะ i32 -> i8 อาจทำให้ข้อมูลหาย
    const b: i8 = @as(i8, @truncate(a));  // truncate: ตัดบิตที่เกิน

    // @as กับ float
    const pi_approx: i32 = @as(i32, @intFromFloat(3.14));

    std.debug.print("300 as i8 (truncated): {}\n", .{b});
    std.debug.print("3.14 as i32: {}\n", .{pi_approx});

    // @as ยังใช้เพื่อบอก compiler ว่าต้องการชนิดใด
    const x = @as(f64, 5);  // บอกว่า 5 เป็น f64 ไม่ใช่ integer
    std.debug.print("5 as f64: {d:.1}\n", .{x});
}
```

### @intCast: Cast ระหว่าง Integer Types

`@intCast(value)` แปลง integer เป็นอีกชนิดหนึ่ง ถ้าค่าเกินช่วง จะ panic ใน Debug mode

```zig
const std = @import("std");

pub fn main() void {
    const big: i32 = 1000;
    const small: i8 = @intCast(big);  // panic ถ้าค่าเกิน i8 range!

    std.debug.print("1000 as i8: {}\n", .{small});

    // ตัวอย่างที่ปลอดภัย
    const safe_val: i32 = 50;
    const safe_small: i8 = @intCast(safe_val);  // 50 อยู่ใน range ของ i8
    std.debug.print("50 as i8: {}\n", .{safe_small});

    // การตรวจสอบก่อน cast
    const value: i32 = 200;
    const min_i8 = std.math.minInt(i8);
    const max_i8 = std.math.maxInt(i8);

    if (value >= min_i8 and value <= max_i8) {
        const casted: i8 = @intCast(value);
        std.debug.print("safe cast: {}\n", .{casted});
    } else {
        std.debug.print("value {} is out of i8 range [{}, {}]\n", .{ value, min_i8, max_i8 });
    }
}
```

### @floatCast: Cast ระหว่าง Float Types

```zig
const std = @import("std");

pub fn main() void {
    const precise: f64 = 3.14159265358979;

    // @floatCast: แปลง float — อาจสูญเสียความแม่นยำ
    const less_precise: f32 = @floatCast(precise);

    std.debug.print("f64: {d:.14}\n", .{precise});
    std.debug.print("f32: {d:.14}\n", .{less_precise});
}
```

### @intFromFloat และ @floatFromInt

```zig
const std = @import("std");

pub fn main() void {
    // float -> integer: ตัดทศนิยม (truncate toward zero)
    const f: f64 = 3.9;
    const i: i32 = @intFromFloat(f);
    std.debug.print("{d:.1} -> {}\n", .{ f, i });  // 3.9 -> 3

    const f_neg: f64 = -3.9;
    const i_neg: i32 = @intFromFloat(f_neg);
    std.debug.print("{d:.1} -> {}\n", .{ f_neg, i_neg });  // -3.9 -> -3

    // integer -> float
    const n: i32 = 42;
    const float_n: f64 = @floatFromInt(n);
    std.debug.print("{} -> {d:.1}\n", .{ n, float_n });  // 42 -> 42.0

    // การปัดเศษที่ต้องการ
    const val: f64 = 3.7;
    const rounded_down = @as(i32, @intFromFloat(val));           // truncate: 3
    const rounded_up = @as(i32, @intFromFloat(val + 0.5));       // round: 4
    const floor_val = @as(i32, @intFromFloat(@floor(val)));      // floor: 3
    const ceil_val = @as(i32, @intFromFloat(@ceil(val)));        // ceil: 4

    std.debug.print("3.7 truncate: {}\n", .{rounded_down});
    std.debug.print("3.7 round: {}\n", .{rounded_up});
    std.debug.print("3.7 floor: {}\n", .{floor_val});
    std.debug.print("3.7 ceil: {}\n", .{ceil_val});
}
```

### @bitCast: ตีความ bits ใหม่

```zig
const std = @import("std");

pub fn main() void {
    // @bitCast: ตีความ bits ใหม่โดยไม่แปลงค่า (ต้องมีขนาดเท่ากัน)
    const float_val: f32 = 1.0;
    const int_bits: u32 = @bitCast(float_val);

    std.debug.print("f32 1.0 as bits: 0x{X:0>8}\n", .{int_bits});
    // IEEE 754: 1.0 = 0x3F800000

    // แปลงกลับ
    const back_to_float: f32 = @bitCast(int_bits);
    std.debug.print("bits back to f32: {d:.1}\n", .{back_to_float});
}
```

### ตารางสรุป Cast Functions

| Function | ทำอะไร | ความปลอดภัย |
|----------|--------|-----------|
| `@as(T, v)` | บอกชนิด / coerce | Safe (เฉพาะ safe coercion) |
| `@intCast(v)` | integer → integer | Panics ถ้า overflow |
| `@floatCast(v)` | float → float | อาจสูญเสียความแม่นยำ |
| `@intFromFloat(v)` | float → integer | Panics ถ้า NaN/Inf/overflow |
| `@floatFromInt(v)` | integer → float | อาจสูญเสียความแม่นยำ |
| `@truncate(v)` | ตัดบิตที่เกิน | ไม่ panic |
| `@bitCast(v)` | ตีความ bits ใหม่ | ต้องขนาดเท่ากัน |

---

## Step 39: Integer Overflow Behavior

### ปัญหา Integer Overflow

**Integer overflow** เกิดขึ้นเมื่อผลลัพธ์ของการคำนวณเกินช่วงค่าของชนิดข้อมูล ตัวอย่างเช่น:
- `i8` สูงสุด 127 ถ้าบวก 1 จะได้... อะไร?

### พฤติกรรมใน Zig

Zig แก้ปัญหานี้อย่างชัดเจนและปลอดภัย:

| Mode | พฤติกรรมเมื่อ overflow |
|------|----------------------|
| Debug build | **Panic** (crash พร้อม error message) |
| Release build | **Undefined behavior** (เหมือน C) — ไม่ควรเกิดขึ้น! |
| Wrapping operators | **Wrap around** (วนกลับ) — ตั้งใจ |
| Saturating operators | **Clamp** (ติดค่า max/min) — ตั้งใจ |

### ตัวอย่าง: Overflow ใน Debug Mode

```zig
const std = @import("std");

pub fn main() void {
    var x: u8 = 255;

    // บรรทัดนี้จะ PANIC ใน Debug mode!
    // x += 1;  // ห้ามทำใน production code!

    // วิธีที่ถูกต้อง: ตรวจสอบก่อน
    if (x < 255) {
        x += 1;
        std.debug.print("x = {}\n", .{x});
    } else {
        std.debug.print("x is already at max!\n", .{});
    }
}
```

### Wrapping Arithmetic: ตั้งใจให้ wrap

เมื่อต้องการให้ overflow วนกลับอย่างตั้งใจ ใช้ **wrapping operators**:

| Operator | ความหมาย |
|----------|---------|
| `+%` | Wrapping addition |
| `-%` | Wrapping subtraction |
| `*%` | Wrapping multiplication |
| `-% x` | Wrapping negation |
| `+%=` | Wrapping addition assignment |
| `-%=` | Wrapping subtraction assignment |
| `*%=` | Wrapping multiplication assignment |

```zig
const std = @import("std");

pub fn main() void {
    // Wrapping addition
    var a: u8 = 255;
    const b: u8 = a +% 1;    // 255 + 1 = 0 (wrap around)
    std.debug.print("255 +% 1 = {}\n", .{b});  // 0

    var c: u8 = 0;
    const d: u8 = c -% 1;    // 0 - 1 = 255 (wrap around)
    std.debug.print("0 -% 1 = {}\n", .{d});    // 255

    // Wrapping multiplication
    var e: u8 = 100;
    const f: u8 = e *% 3;    // 300 mod 256 = 44
    std.debug.print("100 *% 3 = {}\n", .{f});  // 44

    // signed wrapping
    var g: i8 = 127;
    const h: i8 = g +% 1;    // 127 + 1 = -128 (two's complement wrap)
    std.debug.print("127 +% 1 = {}\n", .{h});  // -128

    // ใช้กับ assignment
    a = 200;
    a +%= 100;  // 300 mod 256 = 44
    std.debug.print("200 +%= 100: a = {}\n", .{a});  // 44
}
```

### Saturating Arithmetic: ติดค่า max/min

เมื่อต้องการให้ overflow "ติด" ที่ค่าสูงสุดหรือต่ำสุด ใช้ **saturating operators**:

| Operator | ความหมาย |
|----------|---------|
| `+|` | Saturating addition |
| `-|` | Saturating subtraction |
| `*|` | Saturating multiplication |
| `<<|` | Saturating shift |
| `+|=` | Saturating addition assignment |
| `-|=` | Saturating subtraction assignment |
| `*|=` | Saturating multiplication assignment |
| `<<|=` | Saturating shift assignment |

```zig
const std = @import("std");

pub fn main() void {
    // Saturating addition: ติดที่ max
    var a: u8 = 250;
    const b: u8 = a +| 20;   // 250 + 20 = 270 แต่ติดที่ 255
    std.debug.print("250 +| 20 = {}\n", .{b});   // 255

    // Saturating subtraction: ติดที่ min
    var c: u8 = 5;
    const d: u8 = c -| 10;   // 5 - 10 = -5 แต่ติดที่ 0
    std.debug.print("5 -| 10 = {}\n", .{d});     // 0

    // Saturating สำหรับ signed
    var e: i8 = 100;
    const f: i8 = e +| 50;   // 100 + 50 = 150 แต่ติดที่ 127
    std.debug.print("100 +| 50 = {}\n", .{f});   // 127

    var g: i8 = -100;
    const h: i8 = g -| 50;   // -100 - 50 = -150 แต่ติดที่ -128
    std.debug.print("-100 -| 50 = {}\n", .{h});  // -128

    // กรณีใช้งานจริง: audio volume control
    var volume: u8 = 200;

    // เพิ่ม volume โดยไม่เกิน 255
    volume +|= 100;
    std.debug.print("volume (saturated): {}\n", .{volume});  // 255

    // ลด volume โดยไม่ต่ำกว่า 0
    volume -|= 255;
    std.debug.print("volume (saturated): {}\n", .{volume});  // 0
}
```

### Checked Arithmetic: ตรวจสอบ overflow

```zig
const std = @import("std");

pub fn main() void {
    // @addWithOverflow: return tuple (result, overflow_bit)
    const result = @addWithOverflow(@as(u8, 200), @as(u8, 100));
    const value = result[0];
    const overflowed = result[1];

    std.debug.print("200 + 100 (u8) = {}, overflowed = {}\n", .{ value, overflowed });

    // ใช้ในฟังก์ชันที่ต้องการ safe arithmetic
    const a: u8 = 250;
    const b: u8 = 10;

    const add_result = @addWithOverflow(a, b);
    if (add_result[1] == 1) {
        std.debug.print("Overflow detected! Cannot add {} + {}\n", .{ a, b });
    } else {
        std.debug.print("{} + {} = {}\n", .{ a, b, add_result[0] });
    }

    // @subWithOverflow, @mulWithOverflow ก็มีเช่นกัน
    const sub_result = @subWithOverflow(@as(u8, 0), @as(u8, 1));
    std.debug.print("0 - 1 (u8): result={}, overflow={}\n", .{ sub_result[0], sub_result[1] });
}
```

### กฎสรุป Integer Overflow

```
ต้องการพฤติกรรมอะไร?
├── ไม่ต้องการ overflow เลย → ตรวจสอบก่อนคำนวณ หรือใช้ @addWithOverflow
├── ยอมให้ wrap around (เช่น hash, crypto) → ใช้ +%, -%, *%
├── ยอมให้ clamp ที่ min/max (เช่น graphics, audio) → ใช้ +|, -|, *|
└── ไม่สน (risky!) → ปกติจะ panic ใน debug, UB ใน release
```

---

## Step 40: Bit Operations และ Bitwise Operators

### Bitwise Operators ทั้งหมด

| Operator | ชื่อ | ตัวอย่าง |
|----------|------|---------|
| `&` | Bitwise AND | `0b1100 & 0b1010 = 0b1000` |
| `\|` | Bitwise OR | `0b1100 \| 0b1010 = 0b1110` |
| `^` | Bitwise XOR | `0b1100 ^ 0b1010 = 0b0110` |
| `~` | Bitwise NOT | `~0b1100 = 0b0011...` |
| `<<` | Left shift | `0b0001 << 3 = 0b1000` |
| `>>` | Right shift | `0b1000 >> 2 = 0b0010` |

### ตัวอย่างโค้ด: Bitwise Operations พื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 0b11001010;  // 202
    const b: u8 = 0b10110101;  // 181

    // AND: 1 เฉพาะเมื่อทั้งคู่เป็น 1
    const and_result: u8 = a & b;
    std.debug.print("{b:0>8} AND {b:0>8} = {b:0>8} ({})\n", .{ a, b, and_result, and_result });

    // OR: 1 เมื่ออย่างน้อยหนึ่งตัวเป็น 1
    const or_result: u8 = a | b;
    std.debug.print("{b:0>8} OR  {b:0>8} = {b:0>8} ({})\n", .{ a, b, or_result, or_result });

    // XOR: 1 เมื่อต่างกัน
    const xor_result: u8 = a ^ b;
    std.debug.print("{b:0>8} XOR {b:0>8} = {b:0>8} ({})\n", .{ a, b, xor_result, xor_result });

    // NOT: กลับค่าทุก bit
    const not_result: u8 = ~a;
    std.debug.print("NOT {b:0>8} = {b:0>8} ({})\n", .{ a, not_result, not_result });

    // Left shift: คูณด้วย 2^n
    const val: u8 = 1;
    std.debug.print("{b:0>8} << 1 = {b:0>8} ({})\n", .{ val, val << 1, val << 1 });
    std.debug.print("{b:0>8} << 3 = {b:0>8} ({})\n", .{ val, val << 3, val << 3 });

    // Right shift: หารด้วย 2^n
    const val2: u8 = 0b10000000;  // 128
    std.debug.print("{b:0>8} >> 1 = {b:0>8} ({})\n", .{ val2, val2 >> 1, val2 >> 1 });
    std.debug.print("{b:0>8} >> 3 = {b:0>8} ({})\n", .{ val2, val2 >> 3, val2 >> 3 });
}
```

### การใช้ Bitwise สำหรับ Flags

```zig
const std = @import("std");

// Permission flags สำหรับ file system
const READ: u8 = 0b00000001;    // bit 0
const WRITE: u8 = 0b00000010;   // bit 1
const EXEC: u8 = 0b00000100;    // bit 2

// หรือใช้ shift
const READ2: u8 = 1 << 0;  // 0b00000001
const WRITE2: u8 = 1 << 1; // 0b00000010
const EXEC2: u8 = 1 << 2;  // 0b00000100

pub fn main() void {
    // สร้าง permission mask
    var perm: u8 = 0;

    // เพิ่ม permission ด้วย OR
    perm |= READ;
    perm |= WRITE;

    std.debug.print("permissions: {b:0>8}\n", .{perm});
    std.debug.print("has READ: {}\n", .{perm & READ != 0});
    std.debug.print("has WRITE: {}\n", .{perm & WRITE != 0});
    std.debug.print("has EXEC: {}\n", .{perm & EXEC != 0});

    // เพิ่ม EXEC
    perm |= EXEC;
    std.debug.print("after adding EXEC: {b:0>8}\n", .{perm});

    // ลบ WRITE ด้วย AND NOT
    perm &= ~WRITE;
    std.debug.print("after removing WRITE: {b:0>8}\n", .{perm});

    // Toggle READ ด้วย XOR
    perm ^= READ;
    std.debug.print("after toggling READ: {b:0>8}\n", .{perm});

    // แสดงชัดเจนขึ้น
    std.debug.print("\nFinal permissions:\n", .{});
    std.debug.print("  READ:  {}\n", .{perm & READ != 0});
    std.debug.print("  WRITE: {}\n", .{perm & WRITE != 0});
    std.debug.print("  EXEC:  {}\n", .{perm & EXEC != 0});
}
```

### เทคนิค Bit Manipulation ที่มีประโยชน์

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 42;

    // 1. ตรวจสอบว่าเลขคู่หรือคี่ (ดูที่ bit สุดท้าย)
    const is_even = (x & 1) == 0;
    const is_odd = (x & 1) == 1;
    std.debug.print("{} is even: {}, odd: {}\n", .{ x, is_even, is_odd });

    // 2. คูณด้วย 2 โดยใช้ left shift
    const doubled = x << 1;
    std.debug.print("{} * 2 = {}\n", .{ x, doubled });

    // 3. หารด้วย 2 โดยใช้ right shift
    const halved = x >> 1;
    std.debug.print("{} / 2 = {}\n", .{ x, halved });

    // 4. ตรวจสอบว่า bit ที่ n เปิดอยู่หรือไม่
    const flags: u32 = 0b10110100;
    for (0..8) |bit| {
        const set = (flags >> @intCast(bit)) & 1;
        std.debug.print("bit {}: {}\n", .{ bit, set });
    }

    // 5. นับ bit ที่เป็น 1 (popcount)
    const num: u32 = 0b10110110;
    const ones = @popCount(num);
    std.debug.print("popcount of {b}: {}\n", .{ num, ones });

    // 6. หา leading zeros
    const lz = @clz(num);
    std.debug.print("leading zeros: {}\n", .{lz});

    // 7. หา trailing zeros
    const tz = @ctz(num);
    std.debug.print("trailing zeros: {}\n", .{tz});
}
```

### RGB Color Manipulation

```zig
const std = @import("std");

// RGB color ที่เก็บในรูปแบบ 0xRRGGBB
const Color = u24;

fn makeColor(r: u8, g: u8, b: u8) Color {
    return (@as(Color, r) << 16) | (@as(Color, g) << 8) | @as(Color, b);
}

fn getRed(color: Color) u8 {
    return @truncate((color >> 16) & 0xFF);
}

fn getGreen(color: Color) u8 {
    return @truncate((color >> 8) & 0xFF);
}

fn getBlue(color: Color) u8 {
    return @truncate(color & 0xFF);
}

fn blendColors(c1: Color, c2: Color, t: u8) Color {
    // Linear interpolation ระหว่างสองสี
    const r1: u16 = getRed(c1);
    const g1: u16 = getGreen(c1);
    const b1: u16 = getBlue(c1);

    const r2: u16 = getRed(c2);
    const g2: u16 = getGreen(c2);
    const b2: u16 = getBlue(c2);

    const factor: u16 = t;

    const r: u8 = @truncate((r1 * (255 - factor) + r2 * factor) / 255);
    const g: u8 = @truncate((g1 * (255 - factor) + g2 * factor) / 255);
    const b: u8 = @truncate((b1 * (255 - factor) + b2 * factor) / 255);

    return makeColor(r, g, b);
}

pub fn main() void {
    const red = makeColor(255, 0, 0);
    const blue = makeColor(0, 0, 255);

    std.debug.print("Red: #{X:0>6}\n", .{red});
    std.debug.print("Blue: #{X:0>6}\n", .{blue});

    std.debug.print("Red   - R:{}, G:{}, B:{}\n", .{ getRed(red), getGreen(red), getBlue(red) });
    std.debug.print("Blue  - R:{}, G:{}, B:{}\n", .{ getRed(blue), getGreen(blue), getBlue(blue) });

    // Blend สองสี 50/50
    const purple = blendColors(red, blue, 128);
    std.debug.print("Purple (50% blend): #{X:0>6}\n", .{purple});
    std.debug.print("Purple - R:{}, G:{}, B:{}\n", .{ getRed(purple), getGreen(purple), getBlue(purple) });
}
```

### Bit Operations กับ @bitReverse และ @byteSwap

```zig
const std = @import("std");

pub fn main() void {
    const x: u8 = 0b11001010;

    // กลับ bit ทั้งหมด
    const reversed = @bitReverse(x);
    std.debug.print("original:  {b:0>8}\n", .{x});
    std.debug.print("reversed:  {b:0>8}\n", .{reversed});

    // สลับ byte order (endian conversion)
    const big_endian: u32 = 0x12345678;
    const little_endian = @byteSwap(big_endian);
    std.debug.print("big endian:    0x{X:0>8}\n", .{big_endian});
    std.debug.print("little endian: 0x{X:0>8}\n", .{little_endian});

    // ตรวจสอบ endianness ของระบบ
    const native_endian = @import("builtin").cpu.arch.endian();
    std.debug.print("system endian: {}\n", .{native_endian});
}
```

---

## ตารางเปรียบเทียบ Memory Size ทุกชนิด

| ชนิดข้อมูล | Bytes | Bits | หมายเหตุ |
|-----------|-------|------|---------|
| `bool`    | 1     | 8    | ใช้จริงแค่ 1 bit |
| `u8`/`i8`   | 1   | 8    | ASCII byte |
| `u16`/`i16` | 2   | 16   | ตัวอักษรบางภาษา |
| `f16`     | 2     | 16   | half precision |
| `u32`/`i32` | 4   | 32   | int ที่ใช้บ่อยที่สุด |
| `f32`     | 4     | 32   | single precision |
| `u64`/`i64` | 8   | 64   | timestamp, large numbers |
| `f64`     | 8     | 64   | double precision (default) |
| `usize`/`isize` | 4–8 | 32–64 | ขึ้นอยู่กับ platform |
| `u128`/`i128` | 16 | 128  | UUID, crypto |
| `f80`     | 10    | 80   | x86 extended |
| `f128`    | 16    | 128  | quad precision |

---

## แบบฝึกหัด (Exercises)

### ระดับ 1: พื้นฐาน

**แบบฝึกหัด 1.1**: เขียนโปรแกรมที่ประกาศตัวแปรทุกชนิดข้อมูล (i8 ถึง i128, u8 ถึง u128, f32, f64, bool) และพิมพ์ค่า min/max ของแต่ละชนิด

**แบบฝึกหัด 1.2**: เขียนฟังก์ชัน `celsius_to_fahrenheit(c: f64) f64` ที่แปลงอุณหภูมิและทดสอบกับ 0°C, 100°C, -40°C

**แบบฝึกหัด 1.3**: เขียนโปรแกรมที่รับ `u8` และพิมพ์ว่าเป็นตัวอักษร (letter), ตัวเลข (digit) หรืออักขระพิเศษ (special character)

### ระดับ 2: กลาง

**แบบฝึกหัด 2.1**: เขียนฟังก์ชัน `safe_add(a: i32, b: i32) ?i32` ที่คืนค่า null เมื่อเกิด overflow และคืนค่าผลรวมเมื่อปกติ

**แบบฝึกหัด 2.2**: สร้าง `packed struct` สำหรับเก็บ RGBA color (8 bit ต่อ channel) ในขนาด 32 bit พร้อม helper functions สำหรับ get/set แต่ละ channel

**แบบฝึกหัด 2.3**: เขียนฟังก์ชัน `is_power_of_two(n: u64) bool` โดยใช้ bitwise operations (ไม่ใช้ loop หรือ division)

### ระดับ 3: ท้าทาย

**แบบฝึกหัด 3.1**: Implement `compress_rgb(r: u8, g: u8, b: u8) u16` ที่บีบอัด RGB888 (24 bit) เป็น RGB565 (16 bit) โดยใช้ bit operations

**แบบฝึกหัด 3.2**: เขียน `count_set_bits(n: u32) u6` ที่นับจำนวน bit ที่เป็น 1 โดยไม่ใช้ `@popCount` (implement ด้วย bitwise ล้วนๆ)

**แบบฝึกหัด 3.3**: สร้าง fixed-point arithmetic library สำหรับตัวเลขทศนิยมที่มี 16 bit integer part และ 16 bit fractional part โดยใช้ `u32`

---

## เฉลยแบบฝึกหัดบางข้อ

### เฉลย 2.3: is_power_of_two

```zig
const std = @import("std");

fn is_power_of_two(n: u64) bool {
    // เลขที่เป็น power of 2 จะมี bit เดียวที่เป็น 1
    // เช่น 8 = 0b1000, 7 = 0b0111
    // 8 & 7 = 0b1000 & 0b0111 = 0
    if (n == 0) return false;
    return (n & (n - 1)) == 0;
}

pub fn main() void {
    const test_values = [_]u64{ 0, 1, 2, 3, 4, 5, 8, 16, 100, 128, 1024 };
    for (test_values) |v| {
        std.debug.print("{} is power of 2: {}\n", .{ v, is_power_of_two(v) });
    }
}
```

### เฉลย 3.1: RGB888 to RGB565

```zig
const std = @import("std");

// RGB888: RRRRRRRR GGGGGGGG BBBBBBBB (24 bit)
// RGB565: RRRRR GGGGGG BBBBB (16 bit)
fn compress_rgb(r: u8, g: u8, b: u8) u16 {
    const r5: u16 = @as(u16, r >> 3);      // ใช้ 5 บิตสูงสุดของ R
    const g6: u16 = @as(u16, g >> 2);      // ใช้ 6 บิตสูงสุดของ G
    const b5: u16 = @as(u16, b >> 3);      // ใช้ 5 บิตสูงสุดของ B

    return (r5 << 11) | (g6 << 5) | b5;
}

fn decompress_rgb(compressed: u16) struct { r: u8, g: u8, b: u8 } {
    const r5: u8 = @truncate(compressed >> 11);
    const g6: u8 = @truncate((compressed >> 5) & 0x3F);
    const b5: u8 = @truncate(compressed & 0x1F);

    // ขยายกลับโดยการ replicate bits
    return .{
        .r = (r5 << 3) | (r5 >> 2),
        .g = (g6 << 2) | (g6 >> 4),
        .b = (b5 << 3) | (b5 >> 2),
    };
}

pub fn main() void {
    const r: u8 = 255;
    const g: u8 = 128;
    const b: u8 = 64;

    const compressed = compress_rgb(r, g, b);
    const decompressed = decompress_rgb(compressed);

    std.debug.print("Original:     R={}, G={}, B={}\n", .{ r, g, b });
    std.debug.print("Compressed:   0x{X:0>4} ({} bits)\n", .{ compressed, @bitSizeOf(u16) });
    std.debug.print("Decompressed: R={}, G={}, B={}\n", .{
        decompressed.r, decompressed.g, decompressed.b,
    });
}
```

---

## สรุปภาค 04

ในภาคนี้เราได้เรียนรู้ชนิดข้อมูลพื้นฐานของ Zig ทั้งหมด:

### สิ่งที่ได้เรียน

| Step | หัวข้อ | สาระสำคัญ |
|------|--------|---------|
| 31 | Signed Integers | `i8`–`i128`, `isize`; ช่วงค่า -(2^(N-1)) ถึง 2^(N-1)-1 |
| 32 | Unsigned Integers | `u8`–`u128`, `usize`; ช่วงค่า 0 ถึง 2^N-1 |
| 33 | Arbitrary Width | `u3`, `i7`, `u24` — ทุกขนาดตั้งแต่ 0 ถึง 65535 บิต |
| 34 | Float Types | `f16`–`f128`; IEEE 754; เลือก f64 เป็น default |
| 35 | Boolean | `bool`; `true`/`false`; short-circuit evaluation |
| 36 | Characters & Unicode | `u8` สำหรับ ASCII, `u21` สำหรับ Unicode; UTF-8 encoding |
| 37 | void & noreturn | `void` = no value; `noreturn` = never returns |
| 38 | Type Casting | `@as`, `@intCast`, `@floatCast`, `@intFromFloat`, `@floatFromInt`, `@bitCast` |
| 39 | Integer Overflow | panic ใน debug; wrapping (`+%`); saturating (`+|`); `@addWithOverflow` |
| 40 | Bit Operations | `&`, `|`, `^`, `~`, `<<`, `>>`; flags pattern; RGB manipulation |

### หลักการสำคัญของ Zig Type System

1. **ทุกอย่างชัดเจน**: ไม่มี implicit conversion ที่ซ่อนเร้น
2. **Safety by default**: overflow panic ใน debug mode
3. **Expressiveness**: arbitrary-width integers สำหรับกรณีพิเศษ
4. **Performance**: saturating/wrapping arithmetic สำหรับกรณีที่รู้ว่าต้องการ
5. **Unicode-aware**: string = `[]const u8` (UTF-8 bytes), character = `u21` (code point)

### ภาคต่อไป

ภาค 05 จะครอบคลุม:
- Optional types (`?T`) — ค่าที่อาจเป็น null
- Error types และ error handling (`!T`)
- Enums และ tagged unions
- Structs และ methods

---

*เอกสารนี้เป็นส่วนหนึ่งของ Zig Programming Course ภาษาไทย*  
*ภาค 04 จาก N ภาค — Steps 31–40*
