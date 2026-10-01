# Part 03: ตัวแปรและค่าคงที่ใน Zig (Variables and Constants)
## Steps 21-30

---

## บทนำ (Introduction)

ใน Part นี้เราจะเรียนรู้เกี่ยวกับการจัดการตัวแปรและค่าคงที่ใน Zig ซึ่งเป็นพื้นฐานที่สำคัญมากของการเขียนโปรแกรม Zig มีปรัชญาที่แตกต่างจากภาษาอื่นในหลายด้าน โดยเฉพาะการเน้น **ความปลอดภัย (safety)** และ **ความชัดเจน (explicitness)** ในการจัดการข้อมูล

ใน Zig ทุกอย่างถูกออกแบบให้เป็น **immutable by default** เมื่อใช้ `const` และต้องประกาศอย่างชัดเจนเมื่อต้องการให้ตัวแปรเปลี่ยนแปลงได้

---

## Step 21: การประกาศตัวแปรด้วย `var`

### แนวคิดพื้นฐาน

`var` ใน Zig ใช้สำหรับประกาศตัวแปรที่ **สามารถเปลี่ยนแปลงค่าได้ (mutable)** หลังจากกำหนดค่าครั้งแรกแล้ว

### รูปแบบการประกาศ

```zig
var ชื่อตัวแปร: ชนิดข้อมูล = ค่าเริ่มต้น;
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศตัวแปรชนิด i32 (integer 32-bit แบบมีเครื่องหมาย)
    var age: i32 = 25;
    std.debug.print("อายุ: {d}\n", .{age});

    // เปลี่ยนค่าตัวแปร
    age = 26;
    std.debug.print("อายุปีหน้า: {d}\n", .{age});

    // ประกาศตัวแปรชนิด f64 (floating point 64-bit)
    var temperature: f64 = 36.5;
    std.debug.print("อุณหภูมิ: {d:.1}\n", .{temperature});

    temperature = 37.2;
    std.debug.print("อุณหภูมิหลังออกกำลังกาย: {d:.1}\n", .{temperature});
}
```

**ผลลัพธ์:**
```
อายุ: 25
อายุปีหน้า: 26
อุณหภูมิ: 36.5
อุณหภูมิหลังออกกำลังกาย: 37.2
```

### การประกาศตัวแปรชนิดต่างๆ

```zig
const std = @import("std");

pub fn main() void {
    // Integer types
    var small_num: i8 = 100;      // -128 ถึง 127
    var medium_num: i16 = 1000;   // -32768 ถึง 32767
    var large_num: i32 = 100000;  // -2^31 ถึง 2^31-1
    var huge_num: i64 = 9999999999; // -2^63 ถึง 2^63-1

    // Unsigned integer types (ไม่มีค่าลบ)
    var u_small: u8 = 255;        // 0 ถึง 255
    var u_medium: u16 = 65535;    // 0 ถึง 65535
    var u_large: u32 = 4294967295; // 0 ถึง 2^32-1

    // Float types
    var f32_num: f32 = 3.14;
    var f64_num: f64 = 3.141592653589793;

    // Boolean type
    var is_valid: bool = true;

    // Platform-dependent types
    var ptr_size: usize = 0;  // ขนาดของ pointer (32 หรือ 64 bit ขึ้นกับ platform)
    var signed_ptr: isize = -10;

    std.debug.print("small_num: {d}\n", .{small_num});
    std.debug.print("huge_num: {d}\n", .{huge_num});
    std.debug.print("u_small: {d}\n", .{u_small});
    std.debug.print("f64_num: {d:.15}\n", .{f64_num});
    std.debug.print("is_valid: {}\n", .{is_valid});

    _ = medium_num;
    _ = large_num;
    _ = u_medium;
    _ = u_large;
    _ = f32_num;
    _ = ptr_size;
    _ = signed_ptr;
}
```

### ข้อควรระวัง: ตัวแปรที่ไม่ได้ใช้งาน

Zig มีกฎเข้มงวดว่า **ต้องใช้งานตัวแปรที่ประกาศ** ถ้าประกาศแล้วไม่ใช้จะเกิด compile error:

```zig
// ไม่ถูกต้อง - จะเกิด error: unused variable 'x'
pub fn main() void {
    var x: i32 = 10;  // ERROR!
}
```

```zig
// วิธีแก้ไข - ใช้ _ เพื่อบอกว่าตั้งใจไม่ใช้ตัวแปรนี้
pub fn main() void {
    var x: i32 = 10;
    _ = x;  // OK - บอกว่าตั้งใจ discard ค่านี้
}
```

### การเปลี่ยนค่าตัวแปรซ้ำ

```zig
const std = @import("std");

pub fn main() void {
    var counter: u32 = 0;
    
    counter = counter + 1;  // เพิ่มทีละ 1
    counter += 1;           // syntax สั้นกว่า
    counter *= 2;           // คูณ 2
    counter -= 1;           // ลด 1
    
    std.debug.print("counter: {d}\n", .{counter});  // ผลลัพธ์: 3
    
    // การ loop และเปลี่ยนค่า
    var sum: i32 = 0;
    var i: i32 = 1;
    while (i <= 10) : (i += 1) {
        sum += i;
    }
    std.debug.print("ผลรวม 1-10: {d}\n", .{sum});  // ผลลัพธ์: 55
}
```

---

## Step 22: ค่าคงที่ด้วย `const`

### แนวคิดพื้นฐาน

`const` ใน Zig ใช้สำหรับประกาศ **ค่าคงที่ (constant)** ที่ไม่สามารถเปลี่ยนแปลงค่าได้หลังจากกำหนดครั้งแรก Zig แนะนำให้ใช้ `const` มากกว่า `var` เมื่อไม่จำเป็นต้องเปลี่ยนค่า เพราะช่วยให้โค้ดปลอดภัยและอ่านง่ายกว่า

### รูปแบบการประกาศ

```zig
const ชื่อค่าคงที่: ชนิดข้อมูล = ค่า;
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศค่าคงที่ทางคณิตศาสตร์
    const PI: f64 = 3.141592653589793;
    const E: f64 = 2.718281828459045;
    const GOLDEN_RATIO: f64 = 1.618033988749895;

    // ประกาศค่าคงที่สำหรับโปรแกรม
    const MAX_USERS: u32 = 1000;
    const APP_VERSION: []const u8 = "1.0.0";
    const DEBUG_MODE: bool = false;

    std.debug.print("PI = {d:.10}\n", .{PI});
    std.debug.print("E = {d:.10}\n", .{E});
    std.debug.print("จำนวนผู้ใช้สูงสุด: {d}\n", .{MAX_USERS});
    std.debug.print("เวอร์ชัน: {s}\n", .{APP_VERSION});
    std.debug.print("โหมด debug: {}\n", .{DEBUG_MODE});
    _ = E;
    _ = GOLDEN_RATIO;
}
```

### ทำไม `const` ถึงดีกว่า `var` ในหลายกรณี

```zig
const std = @import("std");

pub fn calculateArea(radius: f64) f64 {
    // PI ไม่ควรเปลี่ยนแปลง ใช้ const
    const PI: f64 = 3.141592653589793;
    
    // radius ที่รับมาเป็น parameter ก็เป็น immutable โดยอัตโนมัติ
    return PI * radius * radius;
}

pub fn main() void {
    const r: f64 = 5.0;  // รัศมีไม่เปลี่ยน
    const area = calculateArea(r);
    
    std.debug.print("รัศมี: {d:.1}\n", .{r});
    std.debug.print("พื้นที่วงกลม: {d:.4}\n", .{area});
}
```

### ความแตกต่างระหว่าง `const` ใน Zig กับ `const` ใน C/C++

```zig
// Zig: const หมายถึง immutable ทุกอย่าง
const std = @import("std");

pub fn main() void {
    // ใน Zig: const กับ pointer
    var data: i32 = 42;
    
    // pointer to const data - ไม่สามารถเปลี่ยนค่าที่ชี้ได้
    const ptr_to_const: *const i32 = &data;
    std.debug.print("ค่าที่ ptr ชี้: {d}\n", .{ptr_to_const.*});
    
    // const pointer to mutable data - ไม่สามารถเปลี่ยน pointer เองได้
    // แต่เปลี่ยนค่าที่ชี้ได้
    const const_ptr: *i32 = &data;
    const_ptr.* = 100;  // OK - เปลี่ยนค่าที่ชี้ได้
    std.debug.print("ค่าใหม่: {d}\n", .{data});
    
    _ = ptr_to_const;
}
```

### ตัวอย่างการใช้ `const` กับ String

```zig
const std = @import("std");

pub fn main() void {
    // String ใน Zig คือ slice of bytes
    const greeting: []const u8 = "สวัสดีโลก!";
    const name: []const u8 = "zig";
    
    std.debug.print("{s}\n", .{greeting});
    std.debug.print("ยินดีต้อนรับสู่ {s}\n", .{name});
    
    // ความยาวของ string
    std.debug.print("ความยาว greeting: {d} bytes\n", .{greeting.len});
    
    // String literal เป็น *const [N:0]u8 (null-terminated)
    const str_literal = "Hello, Zig!";
    std.debug.print("{s}\n", .{str_literal});
}
```

---

## Step 23: Type Inference (การอนุมานชนิดข้อมูล)

### แนวคิดพื้นฐาน

Zig รองรับ **type inference** ซึ่งหมายความว่า compiler สามารถ "อนุมาน" ชนิดข้อมูลจากค่าที่กำหนดให้โดยอัตโนมัติ ทำให้ไม่ต้องระบุ type ทุกครั้ง

### การใช้งาน Type Inference

```zig
const std = @import("std");

pub fn main() void {
    // โดยไม่ระบุ type - compiler จะอนุมานเอง
    const x = 42;          // อนุมานเป็น comptime_int
    const y = 3.14;        // อนุมานเป็น comptime_float
    const z = true;        // อนุมานเป็น bool
    const msg = "hello";   // อนุมานเป็น *const [5:0]u8

    std.debug.print("x = {d}\n", .{x});
    std.debug.print("y = {d:.2}\n", .{y});
    std.debug.print("z = {}\n", .{z});
    std.debug.print("msg = {s}\n", .{msg});
    
    // ตรวจสอบ type ด้วย @TypeOf
    std.debug.print("type of x: {}\n", .{@TypeOf(x)});
    std.debug.print("type of y: {}\n", .{@TypeOf(y)});
    std.debug.print("type of z: {}\n", .{@TypeOf(z)});
}
```

**ผลลัพธ์:**
```
x = 42
y = 3.14
z = true
msg = hello
type of x: comptime_int
type of y: comptime_float
type of z: bool
```

### comptime_int และ comptime_float

```zig
const std = @import("std");

pub fn main() void {
    // comptime_int มีความแม่นยำไม่จำกัด
    const big_num = 99999999999999999999999999;
    
    // เมื่อใช้ใน runtime context จะถูก coerce เป็น type ที่เหมาะสม
    var runtime_num: i64 = big_num;  // จะ error ถ้า big_num ใหญ่เกินไปสำหรับ i64
    
    // comptime_float มีความแม่นยำ f128
    const precise = 1.0 / 3.0;  // แม่นยำมากตอน comptime
    var runtime_float: f64 = precise;  // ลดความแม่นยำลงตอน runtime
    
    std.debug.print("runtime_num: {d}\n", .{runtime_num});
    std.debug.print("precise: {d:.20}\n", .{runtime_float});
    
    _ = big_num;
}
```

### Type Inference กับ Struct และ Array

```zig
const std = @import("std");

pub fn main() void {
    // Array - type inference
    const numbers = [_]i32{ 1, 2, 3, 4, 5 };
    // [_] หมายถึง ให้ compiler นับขนาดเอง
    
    std.debug.print("จำนวนสมาชิก: {d}\n", .{numbers.len});
    std.debug.print("สมาชิกแรก: {d}\n", .{numbers[0]});
    
    // ตรวจสอบ type
    std.debug.print("type: {}\n", .{@TypeOf(numbers)});
    // จะแสดง: [5]i32
    
    // Tuple-like initialization
    const coords = .{ 10, 20, 30 };
    std.debug.print("x={d}, y={d}, z={d}\n", .{ coords[0], coords[1], coords[2] });
}
```

### เมื่อใดควรระบุ type และเมื่อใดใช้ inference

```zig
const std = @import("std");

pub fn main() void {
    // กรณีที่ควรระบุ type ชัดเจน:
    // 1. เมื่อต้องการ type เฉพาะ
    var specific: u8 = 200;  // ถ้าไม่ระบุจะเป็น comptime_int
    
    // 2. เมื่อค่าเริ่มต้นไม่ชัดเจน
    var result: f64 = 10;  // 10 จะถูก coerce เป็น f64
    
    // กรณีที่ใช้ inference ได้:
    // 1. ค่าคงที่ที่ชัดเจน
    const PI = 3.14159;
    const IS_RELEASE = false;
    const APP_NAME = "MyApp";
    
    std.debug.print("specific: {d}\n", .{specific});
    std.debug.print("result: {d:.1}\n", .{result});
    std.debug.print("PI: {d:.5}\n", .{PI});
    std.debug.print("IS_RELEASE: {}\n", .{IS_RELEASE});
    std.debug.print("APP_NAME: {s}\n", .{APP_NAME});
    _ = specific;
    _ = result;
}
```

### ข้อแตกต่างกับภาษาอื่น

| ภาษา | Type Inference |
|------|---------------|
| Zig | `const x = 42;` (เป็น comptime_int) |
| Go | `x := 42` (เป็น int) |
| Rust | `let x = 42;` (เป็น i32 โดย default) |
| TypeScript | `const x = 42;` (เป็น number literal type `42`) |
| C++ | `auto x = 42;` (เป็น int) |

---

## Step 24: การประกาศตัวแปรแบบกำหนด Type ชัดเจน

### ทำไมต้องระบุ Type ชัดเจน

แม้ Zig จะรองรับ type inference แต่บางครั้งการระบุ type ชัดเจนมีประโยชน์มาก:
1. ทำให้โค้ดอ่านเข้าใจง่ายขึ้น
2. ป้องกัน bugs จาก type mismatch
3. ควบคุมขนาดหน่วยความจำที่ใช้

### Integer Types ทั้งหมด

```zig
const std = @import("std");

pub fn main() void {
    // Signed integers
    var i8_val: i8 = -100;      // 8-bit, ช่วง: -128 ถึง 127
    var i16_val: i16 = -1000;   // 16-bit, ช่วง: -32768 ถึง 32767
    var i32_val: i32 = -100000; // 32-bit, ช่วง: ~-2.1B ถึง ~2.1B
    var i64_val: i64 = -999999999999; // 64-bit
    var i128_val: i128 = -99999999999999999999; // 128-bit (หายาก)
    
    // Unsigned integers
    var u8_val: u8 = 255;       // 8-bit, ช่วง: 0 ถึง 255
    var u16_val: u16 = 65535;   // 16-bit, ช่วง: 0 ถึง 65535
    var u32_val: u32 = 4294967295; // 32-bit
    var u64_val: u64 = 18446744073709551615; // 64-bit
    
    // Platform-dependent
    var usize_val: usize = 1024; // ขนาดของ pointer
    var isize_val: isize = -512;
    
    std.debug.print("i8: {d}\n", .{i8_val});
    std.debug.print("u8: {d}\n", .{u8_val});
    std.debug.print("i64: {d}\n", .{i64_val});
    std.debug.print("u64: {d}\n", .{u64_val});
    
    _ = i16_val; _ = i32_val; _ = i128_val;
    _ = u16_val; _ = u32_val;
    _ = usize_val; _ = isize_val;
}
```

### Arbitrary-width Integers

Zig มีความสามารถพิเศษที่ภาษาอื่นส่วนใหญ่ไม่มี คือสามารถกำหนดความกว้างของ integer ได้ทุกขนาดตั้งแต่ `i1` ถึง `i65535`:

```zig
const std = @import("std");

pub fn main() void {
    // Integer ขนาดพิเศษ
    var three_bit: u3 = 7;   // 0 ถึง 7 (3 bits)
    var five_bit: i5 = -15;  // -16 ถึง 15 (5 bits)
    var ten_bit: u10 = 1023; // 0 ถึง 1023 (10 bits)
    
    std.debug.print("u3 max: {d}\n", .{three_bit});
    std.debug.print("i5: {d}\n", .{five_bit});
    std.debug.print("u10 max: {d}\n", .{ten_bit});
    
    // ใช้จริงในการทำ bit packing
    // เช่นในการจัดการ hardware registers
    const Hardware = packed struct {
        flag1: u1,
        flag2: u1,
        mode: u3,
        value: u11,
    };
    
    var hw = Hardware{
        .flag1 = 1,
        .flag2 = 0,
        .mode = 5,
        .value = 1024,
    };
    
    std.debug.print("flag1: {d}, mode: {d}, value: {d}\n", 
                    .{hw.flag1, hw.mode, hw.value});
    _ = hw;
}
```

### Float Types

```zig
const std = @import("std");

pub fn main() void {
    var f16_val: f16 = 3.14;    // 16-bit float (half precision)
    var f32_val: f32 = 3.14159; // 32-bit float (single precision)
    var f64_val: f64 = 3.141592653589793; // 64-bit float (double precision)
    var f80_val: f80 = 3.14159265358979323846; // 80-bit float (extended)
    var f128_val: f128 = 3.141592653589793238462643383279; // 128-bit float (quad)

    std.debug.print("f16: {d:.4}\n", .{f16_val});
    std.debug.print("f32: {d:.7}\n", .{f32_val});
    std.debug.print("f64: {d:.15}\n", .{f64_val});
    
    _ = f80_val;
    _ = f128_val;
}
```

### การแปลง Type อย่างชัดเจน

Zig ไม่อนุญาตให้แปลง type โดยอัตโนมัติ (implicit conversion) ต้องทำอย่างชัดเจนเสมอ:

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 100;
    
    // ผิด - ไม่สามารถกำหนด i32 ให้ i64 โดยตรง
    // var y: i64 = x;  // ERROR: type mismatch
    
    // ถูก - ต้อง cast ชัดเจน
    var y: i64 = @as(i64, x);  // ใช้ @as() สำหรับ safe cast
    var z: i64 = @intCast(x);  // ใช้ @intCast() สำหรับ integer cast
    
    std.debug.print("x: {d}\n", .{x});
    std.debug.print("y: {d}\n", .{y});
    std.debug.print("z: {d}\n", .{z});
    
    // Float to integer
    var f: f64 = 3.7;
    var i: i32 = @intFromFloat(f);  // truncates (ตัดทศนิยม)
    std.debug.print("f: {d:.1}, i: {d}\n", .{ f, i });  // 3.7 -> 3
    
    // Integer to float
    var n: i32 = 42;
    var float_n: f64 = @floatFromInt(n);
    std.debug.print("n: {d}, float_n: {d:.1}\n", .{ n, float_n });
}
```

---

## Step 25: `undefined` - ค่าเริ่มต้นที่ไม่ได้กำหนด

### แนวคิดพื้นฐาน

ใน Zig `undefined` เป็นค่าพิเศษที่บอกว่า **"ยังไม่ได้กำหนดค่า"** ใช้เมื่อต้องการประกาศตัวแปรแต่ยังไม่พร้อมกำหนดค่าในทันที

### รูปแบบการใช้งาน

```zig
var ชื่อตัวแปร: ชนิดข้อมูล = undefined;
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศตัวแปรโดยยังไม่กำหนดค่า
    var x: i32 = undefined;
    
    // ต้องกำหนดค่าก่อนใช้งาน!
    x = 42;
    std.debug.print("x = {d}\n", .{x});
    
    // Array ที่ยังไม่ได้กำหนดค่า
    var buffer: [10]u8 = undefined;
    
    // กำหนดค่าทีหลัง
    for (&buffer, 0..) |*byte, i| {
        byte.* = @intCast(i);
    }
    
    std.debug.print("buffer[5] = {d}\n", .{buffer[5]});
}
```

### ความเสี่ยงของ `undefined`

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = undefined;
    
    // ในโหมด Debug: Zig จะใส่ค่า 0xaa ทุก byte
    // เพื่อให้ตรวจพบ bug ได้ง่าย
    
    // ถ้า build แบบ Debug จะเห็นค่า garbage
    // ถ้า build แบบ ReleaseSafe ยังตรวจสอบ
    // ถ้า build แบบ ReleaseFast ไม่มีการตรวจสอบ - อันตราย!
    
    // ดีกว่า: กำหนดค่าก่อนใช้เสมอ
    x = 0;
    std.debug.print("x = {d}\n", .{x});
}
```

### Use Cases ที่เหมาะสมสำหรับ `undefined`

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Use case 1: Buffer ที่จะถูกเขียนทับทั้งหมด
    var read_buffer: [1024]u8 = undefined;
    // ตัวอย่าง: อ่านข้อมูลลง buffer (จำลอง)
    const data = "Hello, Zig!";
    @memcpy(read_buffer[0..data.len], data);
    try stdout.print("buffer: {s}\n", .{read_buffer[0..data.len]});
    
    // Use case 2: ตัวแปรที่จะถูกกำหนดใน branch ทุก branch
    var result: i32 = undefined;
    const condition = true;
    
    if (condition) {
        result = 100;
    } else {
        result = -100;
    }
    // ณ จุดนี้ result ถูกกำหนดแน่นอน
    try stdout.print("result = {d}\n", .{result});
    
    // Use case 3: Struct ที่จะถูกกำหนดทุก field ทันที
    const Point = struct { x: f32, y: f32 };
    var p: Point = undefined;
    p.x = 3.0;
    p.y = 4.0;
    try stdout.print("point: ({d:.1}, {d:.1})\n", .{p.x, p.y});
}
```

### `undefined` กับโหมด Build ต่างๆ

```
โหมด Debug:
  - Zig เติม 0xaa ใน undefined memory
  - ถ้าใช้ undefined value จะเห็นค่า 0xaaaa... ซึ่งน่าสังเกต

โหมด ReleaseSafe:
  - ยังมีการตรวจสอบ safety บางส่วน

โหมด ReleaseFast:
  - ไม่มีการตรวจสอบ
  - undefined หมายถึงไม่มีค่าจริงๆ (garbage from memory)
  - อันตรายถ้าไม่ระวัง

โหมด ReleaseSmall:
  - คล้าย ReleaseFast แต่ optimize ขนาด
```

---

## Step 26: ความแตกต่างระหว่าง `const` และ `var`

### ตารางเปรียบเทียบ

| คุณสมบัติ | `const` | `var` |
|-----------|---------|-------|
| เปลี่ยนค่าได้หรือไม่ | ไม่ได้ | ได้ |
| ใช้ใน comptime ได้ | ได้ (เสมอ) | ได้บางกรณี |
| ความปลอดภัย | สูงกว่า | ต่ำกว่า |
| Performance | ดีกว่า (compiler optimize) | ขึ้นอยู่กับการใช้งาน |
| แนะนำให้ใช้เมื่อ | ไม่ต้องเปลี่ยนค่า | ต้องเปลี่ยนค่า |

### ตัวอย่างเปรียบเทียบ

```zig
const std = @import("std");

pub fn main() void {
    // ==========================================
    // const - immutable
    // ==========================================
    const max_size: u32 = 100;
    
    // ต่อไปนี้จะเกิด compile error:
    // max_size = 200;  // ERROR: cannot assign to constant
    
    std.debug.print("max_size: {d}\n", .{max_size});
    
    // ==========================================
    // var - mutable
    // ==========================================
    var current_size: u32 = 0;
    
    current_size = 50;   // OK
    current_size += 10;  // OK
    
    std.debug.print("current_size: {d}\n", .{current_size});
    
    // ==========================================
    // ตัวอย่างการใช้ทั้งสอง
    // ==========================================
    const limit: i32 = 10;
    var count: i32 = 0;
    
    while (count < limit) : (count += 1) {
        std.debug.print("count: {d}\n", .{count});
        if (count >= 3) break;  // หยุดที่ 3 เพื่อประหยัดพื้นที่
    }
}
```

### `const` กับ Pointer

```zig
const std = @import("std");

pub fn main() void {
    var value: i32 = 10;
    
    // Pointer แบบต่างๆ:
    
    // 1. const pointer ที่ชี้ไปยัง const data
    const const_ptr_to_const: *const i32 = &value;
    // - ไม่สามารถเปลี่ยน pointer เอง
    // - ไม่สามารถเปลี่ยนค่าที่ชี้
    std.debug.print("const_ptr_to_const: {d}\n", .{const_ptr_to_const.*});
    
    // 2. const pointer ที่ชี้ไปยัง mutable data
    const const_ptr: *i32 = &value;
    // - ไม่สามารถเปลี่ยน pointer เอง
    // - สามารถเปลี่ยนค่าที่ชี้ได้
    const_ptr.* = 20;
    std.debug.print("after change via const_ptr: {d}\n", .{value});
    
    // 3. var pointer ที่ชี้ไปยัง mutable data
    var another: i32 = 30;
    var var_ptr: *i32 = &value;
    // - สามารถเปลี่ยน pointer เองได้
    // - สามารถเปลี่ยนค่าที่ชี้ได้
    var_ptr.* = 50;
    var_ptr = &another;  // เปลี่ยน pointer ให้ชี้ที่อื่น
    var_ptr.* = 60;
    std.debug.print("value: {d}, another: {d}\n", .{ value, another });
}
```

### `const` ไม่ได้หมายความว่า deeply immutable เสมอ

```zig
const std = @import("std");

pub fn main() void {
    var data: [5]i32 = .{ 1, 2, 3, 4, 5 };
    
    // const pointer ไปยัง array
    const slice: []i32 = &data;
    
    // slice เองเปลี่ยนไม่ได้ (จะ point ที่ไหนไม่ได้เปลี่ยน)
    // แต่ยังเปลี่ยนค่าใน array ได้!
    slice[0] = 100;
    slice[2] = 300;
    
    std.debug.print("data[0]: {d}\n", .{data[0]});  // 100
    std.debug.print("data[2]: {d}\n", .{data[2]});  // 300
    
    // ถ้าต้องการ immutable slice ต้องใช้ []const i32
    const immutable_slice: []const i32 = &data;
    // immutable_slice[0] = 999;  // ERROR: cannot assign to constant
    std.debug.print("immutable view[0]: {d}\n", .{immutable_slice[0]});
}
```

### Zig Style Guide: ควรใช้ `const` ก่อนเสมอ

```zig
const std = @import("std");

pub fn goodStyle() void {
    // ดี: เริ่มต้นด้วย const เสมอ
    const x = calculateValue();
    const y = x * 2;
    
    // ถ้าจำเป็นต้องเปลี่ยนแปลงค่า ค่อยใช้ var
    var counter: i32 = 0;
    counter += 1;
    
    std.debug.print("x={d}, y={d}, counter={d}\n", .{x, y, counter});
}

fn calculateValue() i32 {
    return 42;
}

pub fn main() void {
    goodStyle();
}
```

---

## Step 27: Scope ของตัวแปร (Block Scope)

### แนวคิดพื้นฐาน

ใน Zig ตัวแปรมี **block scope** หมายความว่าตัวแปรมีชีวิตอยู่เฉพาะใน block `{ }` ที่ประกาศ เมื่อออกจาก block แล้วตัวแปรนั้นจะไม่สามารถเข้าถึงได้อีก

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    var outer: i32 = 10;
    
    {   // เริ่ม block ใหม่
        var inner: i32 = 20;
        std.debug.print("inner ใน block: {d}\n", .{inner});
        std.debug.print("outer ใน block: {d}\n", .{outer});  // เข้าถึง outer ได้
        
        outer = 15;  // เปลี่ยนค่า outer ได้
    }   // inner ถูก destroy ที่นี่
    
    // std.debug.print("{d}", .{inner});  // ERROR: ไม่สามารถเข้าถึง inner
    std.debug.print("outer หลัง block: {d}\n", .{outer});
}
```

### Scope ใน If/While/For

```zig
const std = @import("std");

pub fn main() void {
    const value: i32 = 42;
    
    // ตัวแปรที่ประกาศใน if block
    if (value > 10) {
        const message = "ค่ามากกว่า 10";  // มีชีวิตแค่ใน if block
        std.debug.print("{s}: {d}\n", .{message, value});
    }
    // message ไม่มีอยู่ที่นี่แล้ว
    
    // ตัวแปรที่ประกาศใน while loop
    var sum: i32 = 0;
    var i: i32 = 0;
    while (i < 5) : (i += 1) {
        const current = i * i;  // ถูกสร้างและ destroy ทุก iteration
        sum += current;
    }
    // current ไม่มีอยู่ที่นี่
    std.debug.print("sum of squares: {d}\n", .{sum});
    
    // For loop scope
    const items = [_]i32{ 10, 20, 30, 40, 50 };
    var total: i32 = 0;
    for (items) |item| {
        const doubled = item * 2;  // มีชีวิตแค่ใน loop body
        total += doubled;
    }
    std.debug.print("total doubled: {d}\n", .{total});
}
```

### Block เป็น Expression

ใน Zig block สามารถ return ค่าได้ ทำให้ตัวแปรภายนอกรับค่าจาก block ได้:

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 10;
    
    // Block expression - block return ค่าด้วย break
    const result = blk: {
        const a = x * 2;
        const b = a + 5;
        break :blk b;  // return ค่าออกจาก block
    };
    
    std.debug.print("result: {d}\n", .{result});  // 25
    
    // ใช้ในการคำนวณที่ซับซ้อน
    const discount = 10;
    const price: f64 = blk: {
        const base_price: f64 = 100.0;
        const discount_amount = base_price * @as(f64, @floatFromInt(discount)) / 100.0;
        break :blk base_price - discount_amount;
    };
    
    std.debug.print("ราคาหลังลด {d}%: {d:.2}\n", .{discount, price});
}
```

### Nested Scope

```zig
const std = @import("std");

pub fn main() void {
    var level1: i32 = 1;
    std.debug.print("Level 1: {d}\n", .{level1});
    
    {
        var level2: i32 = 2;
        std.debug.print("Level 1 ใน Level 2: {d}\n", .{level1});
        std.debug.print("Level 2: {d}\n", .{level2});
        
        {
            var level3: i32 = 3;
            std.debug.print("Level 1 ใน Level 3: {d}\n", .{level1});
            std.debug.print("Level 2 ใน Level 3: {d}\n", .{level2});
            std.debug.print("Level 3: {d}\n", .{level3});
            _ = level3;
        }
        // level3 ไม่มีแล้ว
        _ = level2;
    }
    // level2 ไม่มีแล้ว
    _ = level1;
}
```

### Function Scope

Function ก็เป็น scope เช่นกัน:

```zig
const std = @import("std");

// global_counter ใช้ได้ทั้ง program
var global_counter: i32 = 0;

pub fn incrementCounter() void {
    var local_var: i32 = 100;  // ใช้ได้แค่ใน function นี้
    global_counter += 1;
    local_var += 1;
    std.debug.print("local_var: {d}\n", .{local_var});
}

pub fn main() void {
    incrementCounter();
    incrementCounter();
    std.debug.print("global_counter: {d}\n", .{global_counter});
    // local_var ไม่สามารถเข้าถึงได้ที่นี่
}
```

---

## Step 28: Shadowing ตัวแปร

### แนวคิดพื้นฐาน

**Shadowing** คือการประกาศตัวแปรในชื่อเดียวกันกับตัวแปรใน scope ที่ใหญ่กว่า ทำให้ตัวแปรใหม่ "บดบัง" ตัวแปรเดิมใน scope ที่เล็กกว่า

### Shadowing ใน Zig

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 10;
    std.debug.print("x นอก block: {d}\n", .{x});  // 10
    
    {
        // สร้าง x ใหม่ที่บดบัง x ข้างนอก
        var x: i32 = 20;  // ใน Zig นี้จะ compile แต่ต้องระวัง!
        std.debug.print("x ใน block: {d}\n", .{x});  // 20
    }
    
    std.debug.print("x หลัง block: {d}\n", .{x});  // 10 (x เดิมกลับมา)
}
```

**หมายเหตุ:** Zig จะแสดง warning สำหรับ shadowing ที่อาจทำให้สับสน

### Shadowing กับ Type ต่างกัน

```zig
const std = @import("std");

pub fn main() void {
    const value: i32 = 42;
    std.debug.print("value (i32): {d}\n", .{value});
    
    {
        // Shadow ด้วย type ต่างกัน
        const value: f64 = 3.14;
        std.debug.print("value (f64): {d:.2}\n", .{value});
        
        {
            const value: []const u8 = "zig";
            std.debug.print("value (string): {s}\n", .{value});
        }
    }
    
    std.debug.print("value กลับมา: {d}\n", .{value});
}
```

### Shadowing ใน Zig vs ภาษาอื่น

```
Rust: อนุญาต shadowing อย่างชัดเจน (let x = 5; let x = x + 1; เปลี่ยน type ได้)
Go: ไม่อนุญาต shadowing ใน same scope (ต้องใช้ block ใหม่)
C: อนุญาต shadowing (compiler warning)
Zig: อนุญาต แต่มี compiler warning ถ้า shadow อาจทำให้สับสน
```

### ตัวอย่างที่ Shadowing มีประโยชน์

```zig
const std = @import("std");

pub fn processInput(raw_input: []const u8) void {
    // Trim whitespace แล้ว shadow ด้วยชื่อเดิม
    const input = std.mem.trim(u8, raw_input, " \t\n\r");
    
    // ตอนนี้ input คือ trimmed version
    // ไม่ต้องตั้งชื่อว่า trimmed_input หรือ cleaned_input
    std.debug.print("Input: '{s}'\n", .{input});
    std.debug.print("ความยาว: {d}\n", .{input.len});
}

pub fn main() void {
    processInput("  Hello, Zig!  ");
}
```

### ข้อควรระวังกับ Shadowing

```zig
const std = @import("std");

pub fn main() void {
    var result: i32 = 0;
    
    for (0..5) |i| {
        // อย่า shadow loop variable โดยไม่จำเป็น
        // var i: i32 = 100;  // นี่จะทำให้สับสนมาก!
        result += @intCast(i);
    }
    
    std.debug.print("result: {d}\n", .{result});
    
    // ดีกว่า: ใช้ชื่อที่แตกต่างกันอย่างชัดเจน
    var total: i32 = 0;
    const limit: i32 = 5;
    var counter: i32 = 0;
    while (counter < limit) : (counter += 1) {
        total += counter;
    }
    std.debug.print("total: {d}\n", .{total});
}
```

---

## Step 29: Global Variables และ `threadlocal`

### Global Variables

ตัวแปร global ใน Zig ประกาศนอก function ใดๆ และสามารถเข้าถึงได้จากทุก function ในไฟล์:

```zig
const std = @import("std");

// Global constants
const APP_NAME: []const u8 = "MyApp";
const VERSION: u32 = 1;
const MAX_CONNECTIONS: usize = 100;

// Global variables (mutable)
var request_count: u64 = 0;
var is_running: bool = false;

pub fn startServer() void {
    is_running = true;
    std.debug.print("{s} v{d} เริ่มทำงาน\n", .{APP_NAME, VERSION});
}

pub fn handleRequest() void {
    if (!is_running) return;
    request_count += 1;
    std.debug.print("Request #{d} (max: {d})\n", .{request_count, MAX_CONNECTIONS});
}

pub fn stopServer() void {
    is_running = false;
    std.debug.print("หยุดทำงาน (ประมวลผล {d} requests)\n", .{request_count});
}

pub fn main() void {
    startServer();
    handleRequest();
    handleRequest();
    handleRequest();
    stopServer();
}
```

### Global Variables กับ Initialization

```zig
const std = @import("std");

// Global const สามารถใช้ comptime expression
const BUFFER_SIZE: usize = 4096;
const HALF_BUFFER: usize = BUFFER_SIZE / 2;
const KB: usize = 1024;
const MB: usize = KB * 1024;

// Global var ต้องมีค่าเริ่มต้นที่เป็น comptime value
var error_count: u32 = 0;
var total_bytes: u64 = 0;

// ไม่ได้! ค่า runtime ไม่สามารถใช้เป็น global initializer
// var timestamp: i64 = std.time.timestamp();  // ERROR

pub fn main() void {
    std.debug.print("BUFFER_SIZE: {d}\n", .{BUFFER_SIZE});
    std.debug.print("MB: {d}\n", .{MB});
    
    error_count += 1;
    total_bytes += 1024;
    
    std.debug.print("errors: {d}, bytes: {d}\n", .{error_count, total_bytes});
    _ = HALF_BUFFER;
}
```

### `threadlocal` - ตัวแปรแบบ Thread-local Storage

`threadlocal` ประกาศตัวแปรที่แต่ละ thread มี copy ของตัวเอง ทำให้ไม่ต้อง synchronize ระหว่าง threads:

```zig
const std = @import("std");

// threadlocal - แต่ละ thread มี copy ของตัวเอง
threadlocal var thread_id: u32 = 0;
threadlocal var thread_request_count: u64 = 0;

pub fn workerFunction(id: u32) void {
    thread_id = id;  // แต่ละ thread set ค่าของตัวเอง
    
    // ทำงานของ thread
    var i: u32 = 0;
    while (i < 5) : (i += 1) {
        thread_request_count += 1;
    }
    
    // ค่าที่เห็นเป็นของ thread นี้เท่านั้น
    std.debug.print("Thread {d}: processed {d} requests\n", 
                    .{thread_id, thread_request_count});
}

pub fn main() !void {
    // จำลองการใช้งาน threadlocal
    const worker1_id: u32 = 1;
    const worker2_id: u32 = 2;
    
    workerFunction(worker1_id);
    workerFunction(worker2_id);
    
    std.debug.print("Main thread_id: {d}\n", .{thread_id});
    _ = worker2_id;
    _ = worker1_id;
}
```

### ตัวอย่าง Thread-safe Counter

```zig
const std = @import("std");

// Global counter ที่ share ระหว่าง threads (ต้อง synchronize)
var shared_counter: u64 = 0;

// threadlocal counter ไม่ต้อง synchronize
threadlocal var local_counter: u64 = 0;

pub fn incrementShared() void {
    // ในการใช้งานจริงต้องใช้ atomic operations หรือ mutex
    shared_counter += 1;
}

pub fn incrementLocal() void {
    // ปลอดภัยไม่ต้อง synchronize
    local_counter += 1;
}

pub fn main() void {
    // จำลองการเรียกจาก thread เดียว
    for (0..10) |_| {
        incrementShared();
        incrementLocal();
    }
    
    std.debug.print("shared_counter: {d}\n", .{shared_counter});
    std.debug.print("local_counter (this thread): {d}\n", .{local_counter});
}
```

### ข้อแตกต่างระหว่าง global var และ threadlocal

```
Global var:
- มีแค่ copy เดียวของทั้ง program
- ทุก thread share กัน
- ต้อง synchronize เมื่อ write จาก multiple threads
- ตัวอย่าง: global configuration, shared state

threadlocal var:
- แต่ละ thread มี copy ของตัวเอง
- ไม่มี sharing ระหว่าง threads
- ไม่ต้อง synchronize
- ตัวอย่าง: error codes, per-thread buffers, thread-specific state
```

---

## Step 30: Comptime Variables

### แนวคิดพื้นฐาน

**Comptime** (Compile-time) คือหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ Zig ช่วยให้สามารถทำการคำนวณและตัดสินใจ ณ เวลา compile แทนที่จะรอถึง runtime

### `comptime` Keyword

```zig
const std = @import("std");

pub fn main() void {
    // comptime expression - คำนวณตอน compile time
    comptime var x: i32 = 0;
    comptime {
        x = 10;
        x *= 2;
        x += 5;
    }
    // ค่า x เป็นที่รู้จักตอน compile: 25
    
    std.debug.print("x (comptime calculated): {d}\n", .{x});
    
    // Array size กำหนดจาก comptime expression
    const SIZE = comptime blk: {
        var s: usize = 1;
        s *= 2;
        s *= 2;
        break :blk s;
    };
    
    const arr: [SIZE]i32 = .{ 1, 2, 3, 4 };
    std.debug.print("arr.len: {d}\n", .{arr.len});
}
```

### Comptime Functions

```zig
const std = @import("std");

// Function ที่รับ comptime parameter
fn createArray(comptime size: usize, comptime T: type) [size]T {
    var arr: [size]T = undefined;
    for (&arr, 0..) |*item, i| {
        item.* = @intCast(i);
    }
    return arr;
}

// Function ที่ return type ขึ้นกับ comptime parameter
fn Vec2(comptime T: type) type {
    return struct {
        x: T,
        y: T,
        
        pub fn add(self: @This(), other: @This()) @This() {
            return .{
                .x = self.x + other.x,
                .y = self.y + other.y,
            };
        }
    };
}

pub fn main() void {
    // สร้าง array ขนาดต่างๆ จาก function เดียวกัน
    const arr5 = createArray(5, i32);
    const arr3 = createArray(3, f32);
    
    std.debug.print("arr5: ", .{});
    for (arr5) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    std.debug.print("arr3: ", .{});
    for (arr3) |v| std.debug.print("{d:.0} ", .{v});
    std.debug.print("\n", .{});
    
    // ใช้ Vec2 type
    const Vec2i = Vec2(i32);
    const Vec2f = Vec2(f64);
    
    var v1 = Vec2i{ .x = 1, .y = 2 };
    var v2 = Vec2i{ .x = 3, .y = 4 };
    const v3 = v1.add(v2);
    
    std.debug.print("v3: ({d}, {d})\n", .{v3.x, v3.y});
    
    var vf1 = Vec2f{ .x = 1.5, .y = 2.5 };
    var vf2 = Vec2f{ .x = 0.5, .y = 0.5 };
    const vf3 = vf1.add(vf2);
    
    std.debug.print("vf3: ({d:.1}, {d:.1})\n", .{vf3.x, vf3.y});
    
    _ = v1; _ = v2; _ = vf1; _ = vf2;
}
```

### Comptime Conditionals

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // Conditional compilation ด้วย comptime
    if (comptime builtin.os.tag == .linux) {
        std.debug.print("กำลังรันบน Linux\n", .{});
    } else if (comptime builtin.os.tag == .windows) {
        std.debug.print("กำลังรันบน Windows\n", .{});
    } else if (comptime builtin.os.tag == .macos) {
        std.debug.print("กำลังรันบน macOS\n", .{});
    } else {
        std.debug.print("กำลังรันบน platform อื่น\n", .{});
    }
    
    // ตรวจสอบ build mode
    if (comptime builtin.mode == .Debug) {
        std.debug.print("โหมด Debug - มีการตรวจสอบเพิ่มเติม\n", .{});
    } else {
        std.debug.print("โหมด Release\n", .{});
    }
    
    // ตรวจสอบ CPU architecture
    std.debug.print("CPU: {}\n", .{builtin.cpu.arch});
}
```

### Comptime Type Reflection

```zig
const std = @import("std");

fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    
    switch (info) {
        .Int => |int_info| {
            std.debug.print("Integer: {d}-bit, signed={}\n", 
                          .{int_info.bits, int_info.signedness == .signed});
        },
        .Float => |float_info| {
            std.debug.print("Float: {d}-bit\n", .{float_info.bits});
        },
        .Bool => {
            std.debug.print("Boolean\n", .{});
        },
        .Struct => |struct_info| {
            std.debug.print("Struct with {d} fields\n", .{struct_info.fields.len});
        },
        else => {
            std.debug.print("Other type: {}\n", .{T});
        },
    }
}

pub fn main() void {
    printTypeInfo(i32);
    printTypeInfo(u8);
    printTypeInfo(f64);
    printTypeInfo(bool);
    
    const Point = struct { x: f32, y: f32 };
    printTypeInfo(Point);
}
```

### Comptime Variables ใน Loop

```zig
const std = @import("std");

pub fn main() void {
    // Unroll loop ด้วย comptime
    comptime var i: usize = 0;
    inline while (i < 5) : (i += 1) {
        // code นี้จะถูก unroll ตอน compile
        std.debug.print("Unrolled iteration {d}\n", .{i});
    }
    
    // สร้าง tuple ด้วย comptime
    const tuple = comptime blk: {
        var t: struct { a: i32, b: f64, c: bool } = undefined;
        t.a = 42;
        t.b = 3.14;
        t.c = true;
        break :blk t;
    };
    
    std.debug.print("tuple.a={d}, tuple.b={d:.2}, tuple.c={}\n",
                    .{tuple.a, tuple.b, tuple.c});
}
```

### Comptime vs Runtime

```zig
const std = @import("std");

// Fibonacci - comptime version
fn fibComptime(comptime n: u64) u64 {
    if (n <= 1) return n;
    return fibComptime(n - 1) + fibComptime(n - 2);
}

// Fibonacci - runtime version
fn fibRuntime(n: u64) u64 {
    if (n <= 1) return n;
    return fibRuntime(n - 1) + fibRuntime(n - 2);
}

pub fn main() void {
    // Comptime - คำนวณตอน compile ไม่มี overhead ตอน runtime
    const fib20 = comptime fibComptime(20);
    std.debug.print("fib(20) [comptime] = {d}\n", .{fib20});
    
    // Runtime - คำนวณตอน runtime
    const n: u64 = 20;
    const fib20_rt = fibRuntime(n);
    std.debug.print("fib(20) [runtime] = {d}\n", .{fib20_rt});
    
    // ทั้งสองให้ค่าเท่ากัน แต่ comptime เร็วกว่าตอน runtime
}
```

### ประโยชน์หลักของ Comptime

```zig
const std = @import("std");

// 1. Generic programming โดยไม่ต้องใช้ template หรือ generics ซับซ้อน
fn sum(comptime T: type, values: []const T) T {
    var total: T = 0;
    for (values) |v| {
        total += v;
    }
    return total;
}

// 2. สร้าง lookup table ตอน compile
const SIN_TABLE: [360]f64 = blk: {
    var table: [360]f64 = undefined;
    for (&table, 0..) |*v, i| {
        const radians = @as(f64, @floatFromInt(i)) * std.math.pi / 180.0;
        v.* = @sin(radians);
    }
    break :blk table;
};

pub fn main() void {
    const ints = [_]i32{ 1, 2, 3, 4, 5 };
    const floats = [_]f64{ 1.1, 2.2, 3.3 };
    
    std.debug.print("sum of ints: {d}\n", .{sum(i32, &ints)});
    std.debug.print("sum of floats: {d:.1}\n", .{sum(f64, &floats)});
    
    // ใช้ precomputed sin table
    std.debug.print("sin(30°) = {d:.4}\n", .{SIN_TABLE[30]});
    std.debug.print("sin(90°) = {d:.4}\n", .{SIN_TABLE[90]});
}
```

---

## ตัวอย่างโปรแกรมสมบูรณ์: ระบบคลังสินค้า

```zig
const std = @import("std");

// Constants สำหรับระบบ
const MAX_ITEMS: usize = 100;
const APP_VERSION: []const u8 = "1.0.0";
const TAX_RATE: f64 = 0.07;  // 7%

// Global state
var item_count: usize = 0;
var total_revenue: f64 = 0.0;

const Item = struct {
    name: []const u8,
    price: f64,
    quantity: u32,
    
    pub fn totalValue(self: Item) f64 {
        return self.price * @as(f64, @floatFromInt(self.quantity));
    }
};

// Buffer สำหรับเก็บสินค้า
var inventory: [MAX_ITEMS]Item = undefined;

pub fn addItem(name: []const u8, price: f64, qty: u32) bool {
    if (item_count >= MAX_ITEMS) return false;
    
    inventory[item_count] = Item{
        .name = name,
        .price = price,
        .quantity = qty,
    };
    item_count += 1;
    return true;
}

pub fn calculateTotal() f64 {
    var subtotal: f64 = 0;
    for (inventory[0..item_count]) |item| {
        subtotal += item.totalValue();
    }
    return subtotal;
}

pub fn printReport() void {
    std.debug.print("\n=== รายงานคลังสินค้า v{s} ===\n", .{APP_VERSION});
    std.debug.print("{s:<20} {s:>10} {s:>8} {s:>12}\n", 
                    .{"ชื่อสินค้า", "ราคา/หน่วย", "จำนวน", "มูลค่ารวม"});
    std.debug.print("{s}\n", .{"-" ** 52});
    
    for (inventory[0..item_count]) |item| {
        std.debug.print("{s:<20} {d:>10.2} {d:>8} {d:>12.2}\n",
                       .{item.name, item.price, item.quantity, item.totalValue()});
    }
    
    const subtotal = calculateTotal();
    const tax = subtotal * TAX_RATE;
    const total = subtotal + tax;
    
    std.debug.print("{s}\n", .{"-" ** 52});
    std.debug.print("ยอดรวม:              {d:>24.2}\n", .{subtotal});
    std.debug.print("ภาษี ({d:.0}%):          {d:>24.2}\n", .{TAX_RATE * 100, tax});
    std.debug.print("ยอดสุทธิ:            {d:>24.2}\n", .{total});
    
    total_revenue = total;
}

pub fn main() void {
    _ = addItem("แอปเปิ้ล", 25.0, 50);
    _ = addItem("กล้วย", 15.0, 100);
    _ = addItem("ส้ม", 35.0, 30);
    _ = addItem("องุ่น", 80.0, 20);
    
    printReport();
    
    std.debug.print("\nรายได้ทั้งหมด: {d:.2} บาท\n", .{total_revenue});
    std.debug.print("จำนวนประเภทสินค้า: {d}/{d}\n", .{item_count, MAX_ITEMS});
}
```

---

## Common Errors และการแก้ไข

### Error 1: Modifying const

```zig
// ผิด
const x: i32 = 10;
x = 20;  // ERROR: cannot assign to constant

// ถูก
var x: i32 = 10;
x = 20;
```

### Error 2: Unused Variable

```zig
// ผิด
var y: i32 = 5;  // ERROR: unused variable 'y'

// ถูก
var y: i32 = 5;
_ = y;  // หรือใช้งานตัวแปรจริงๆ
```

### Error 3: Type Mismatch

```zig
// ผิด
var a: i32 = 10;
var b: i64 = a;  // ERROR: expected type 'i64', found 'i32'

// ถูก
var a: i32 = 10;
var b: i64 = @as(i64, a);
// หรือ
var b2: i64 = @intCast(a);
```

### Error 4: Integer Overflow

```zig
// ผิด (runtime panic ใน Debug mode)
var x: u8 = 255;
x += 1;  // PANIC: integer overflow

// ถูก - ใช้ wrapping arithmetic
var x: u8 = 255;
x +%= 1;  // ใช้ +% สำหรับ wrapping add (ผลลัพธ์: 0)

// หรือตรวจสอบก่อน
if (x < 255) x += 1;
```

### Error 5: Accessing undefined

```zig
// อันตราย - อย่าทำ
var data: i32 = undefined;
std.debug.print("{d}", .{data});  // undefined behavior!

// ถูก - กำหนดค่าก่อนใช้เสมอ
var data: i32 = undefined;
data = 42;
std.debug.print("{d}", .{data});
```

### Error 6: Global variable with runtime initializer

```zig
// ผิด - ค่า runtime ไม่สามารถใช้เป็น global initializer
const std = @import("std");
var timestamp: i64 = std.time.timestamp();  // ERROR!

// ถูก - ใช้ undefined หรือ comptime value
var timestamp: i64 = 0;

pub fn main() void {
    // กำหนดค่าใน main
    timestamp = std.time.timestamp();
}
```

---

## เปรียบเทียบกับภาษาอื่น

### Python

```python
# Python
x = 10       # mutable โดย default
x = "hello"  # เปลี่ยน type ได้ด้วย!

# Python ไม่มี native const (ใช้ convention ตัวพิมพ์ใหญ่)
MAX_SIZE = 100  # แค่ convention, ยังเปลี่ยนได้
```

```zig
// Zig
var x: i32 = 10;    // mutable, type ชัดเจน
// x = "hello";     // ERROR: type เปลี่ยนไม่ได้
const MAX_SIZE: u32 = 100;  // truly immutable
```

### JavaScript/TypeScript

```typescript
// TypeScript
let x = 10;         // mutable
const y = 20;       // "immutable" แต่ object/array ยังเปลี่ยนได้
var z = 30;         // function-scoped (avoid)

// TypeScript ไม่มี block scope กับ var
if (true) {
    var leaked = 100;  // ยังเข้าถึงได้นอก if!
}
console.log(leaked);  // 100 (leaked!)
```

```zig
// Zig
var x: i32 = 10;
const y: i32 = 20;
// ไม่มี "var" แบบ function-scoped
// ทุกอย่างเป็น block-scoped อย่างเคร่งครัด
if (true) {
    var scoped: i32 = 100;
    _ = scoped;
}
// scoped ไม่มีที่นี่
```

### Rust

```rust
// Rust
let x = 10;           // immutable by default
let mut y = 20;       // mutable ต้องใส่ mut
let y = y + 1;        // shadowing ได้

const MAX: i32 = 100; // compile-time constant
static GLOBAL: i32 = 0; // static variable
```

```zig
// Zig - คล้าย Rust แต่ต่างกันบางจุด
const x: i32 = 10;    // immutable
var y: i32 = 20;      // mutable ใช้ var
const y2: i32 = y + 1; // shadowing (ใน inner scope)

const MAX: i32 = 100;  // compile-time constant
var GLOBAL: i32 = 0;   // global variable
```

### C/C++

```c
// C
int x = 10;           // mutable
const int y = 20;     // immutable
#define MAX 100       // preprocessor (ไม่มี type)

// C ไม่มี block scope ที่เข้มงวดสำหรับ if/while
// ตัวแปรใน loop สามารถ leak ออกมาใน C89
```

```zig
// Zig - ปลอดภัยกว่า C มาก
var x: i32 = 10;
const y: i32 = 20;
const MAX: u32 = 100;  // typed constant

// Block scope เข้มงวดกว่า C
// ไม่มี macro/preprocessor
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: พื้นฐาน var และ const

เขียนโปรแกรมที่:
1. ประกาศตัวแปร `temperature` แบบ `var` ชนิด `f64` กำหนดค่าเริ่มต้น 25.0
2. ประกาศค่าคงที่ `BOILING_POINT` ชนิด `f64` = 100.0
3. เพิ่มค่า temperature ทีละ 10 จนถึงจุดเดือด
4. นับว่าเพิ่มกี่ครั้ง

```zig
// โค้ดเฉลย
const std = @import("std");

pub fn main() void {
    var temperature: f64 = 25.0;
    const BOILING_POINT: f64 = 100.0;
    var steps: u32 = 0;
    
    while (temperature < BOILING_POINT) : (steps += 1) {
        temperature += 10.0;
        std.debug.print("อุณหภูมิ: {d:.1}°C\n", .{temperature});
    }
    
    std.debug.print("ถึงจุดเดือดใน {d} ขั้นตอน\n", .{steps});
}
```

### แบบฝึกหัดที่ 2: Type Inference

เขียนโปรแกรมที่ใช้ type inference และตรวจสอบ type ด้วย `@TypeOf()`:

```zig
// โค้ดเฉลย
const std = @import("std");

pub fn main() void {
    const a = 42;
    const b = 3.14;
    const c = "zig";
    const d = true;
    const e = [_]i32{1, 2, 3};
    
    std.debug.print("42 -> {}\n", .{@TypeOf(a)});
    std.debug.print("3.14 -> {}\n", .{@TypeOf(b)});
    std.debug.print("\"zig\" -> {}\n", .{@TypeOf(c)});
    std.debug.print("true -> {}\n", .{@TypeOf(d)});
    std.debug.print("[1,2,3] -> {}\n", .{@TypeOf(e)});
}
```

### แบบฝึกหัดที่ 3: Scope และ Block Expression

เขียน function ที่คำนวณ factorial โดยใช้ block expression:

```zig
// โค้ดเฉลย
const std = @import("std");

pub fn factorial(n: u64) u64 {
    return blk: {
        if (n <= 1) break :blk 1;
        var result: u64 = 1;
        var i: u64 = 2;
        while (i <= n) : (i += 1) {
            result *= i;
        }
        break :blk result;
    };
}

pub fn main() void {
    for (0..10) |i| {
        std.debug.print("{d}! = {d}\n", .{i, factorial(i)});
    }
}
```

### แบบฝึกหัดที่ 4: Comptime Generic

เขียน generic function `minValue` ที่หาค่าต่ำสุดใน array:

```zig
// โค้ดเฉลย
const std = @import("std");

fn minValue(comptime T: type, values: []const T) T {
    std.debug.assert(values.len > 0);
    var min = values[0];
    for (values[1..]) |v| {
        if (v < min) min = v;
    }
    return min;
}

pub fn main() void {
    const ints = [_]i32{5, 2, 8, 1, 9, 3};
    const floats = [_]f64{3.14, 2.71, 1.41, 1.73};
    
    std.debug.print("min of ints: {d}\n", .{minValue(i32, &ints)});
    std.debug.print("min of floats: {d:.2}\n", .{minValue(f64, &floats)});
}
```

### แบบฝึกหัดที่ 5: Global State Management

สร้างระบบ counter ง่ายๆ ด้วย global variables:

```zig
// โค้ดเฉลย
const std = @import("std");

const MAX_COUNT: u32 = 10;
var current_count: u32 = 0;
var is_at_max: bool = false;

pub fn increment() bool {
    if (current_count >= MAX_COUNT) {
        is_at_max = true;
        return false;
    }
    current_count += 1;
    return true;
}

pub fn reset() void {
    current_count = 0;
    is_at_max = false;
}

pub fn getStatus() void {
    std.debug.print("Count: {d}/{d}, AtMax: {}\n", 
                    .{current_count, MAX_COUNT, is_at_max});
}

pub fn main() void {
    // เพิ่มจน overflow
    for (0..12) |_| {
        const ok = increment();
        if (!ok) std.debug.print("ไม่สามารถเพิ่มได้ (ถึง max แล้ว)\n", .{});
    }
    getStatus();
    
    // reset
    reset();
    getStatus();
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 03

| Step | หัวข้อ | สิ่งสำคัญ |
|------|--------|-----------|
| 21 | `var` | ตัวแปรที่เปลี่ยนแปลงได้ ต้องระบุค่าเริ่มต้น |
| 22 | `const` | ค่าคงที่ immutable แนะนำให้ใช้มากกว่า var |
| 23 | Type Inference | compiler อนุมาน type ได้ comptime_int/float |
| 24 | Explicit Types | ระบุ type ชัดเจน cast ต้องทำอย่างชัดเจน |
| 25 | `undefined` | ยังไม่กำหนดค่า อันตรายถ้าใช้โดยไม่กำหนดก่อน |
| 26 | const vs var | const เป็น deeply immutable มักปลอดภัยกว่า |
| 27 | Block Scope | ตัวแปรมีชีวิตเฉพาะใน block ที่ประกาศ |
| 28 | Shadowing | ประกาศตัวแปรซ้ำใน inner scope ทำได้แต่ระวัง |
| 29 | Global/threadlocal | Global share ทุก thread, threadlocal แยกต่างหาก |
| 30 | Comptime | คำนวณตอน compile เพื่อ performance และ generics |

### หลักการสำคัญของ Zig ที่ได้เรียนรู้

1. **Explicit over implicit**: ทุกอย่างต้องชัดเจน ไม่มีการแปลง type อัตโนมัติ
2. **Immutable by default**: ใช้ `const` ก่อนเสมอ เปลี่ยนเป็น `var` เมื่อจำเป็น
3. **No hidden allocations**: ไม่มีการจอง memory แบบซ่อนเร้น
4. **Comptime power**: ใช้ comptime แทน generics/templates/macros
5. **Safety**: ตรวจสอบ bounds, overflow ใน Debug mode

### เตรียมพร้อมสำหรับ Part ถัดไป

ใน Part 04 เราจะเรียนเรื่อง **ชนิดข้อมูลและโครงสร้างข้อมูล** ซึ่งจะครอบคลุม:
- Arrays และ Slices
- Structs
- Enums
- Unions
- Optionals (?)
- Error unions (!)

---

*Part 03 จบแล้ว - ขอแสดงความยินดีที่เรียนรู้เรื่อง Variables และ Constants ใน Zig ครบทั้ง 10 steps!*
