# ส่วนที่ 05: ตัวดำเนินการใน Zig (Operators)
## Steps 41-50

---

ตัวดำเนินการ (Operators) คือสัญลักษณ์หรือคำสำคัญที่ใช้ในการดำเนินการกับค่าต่าง ๆ ใน Zig มีตัวดำเนินการหลากหลายประเภทที่ครอบคลุมตั้งแต่การคำนวณทางคณิตศาสตร์พื้นฐาน ไปจนถึงการดำเนินการระดับบิตที่ซับซ้อน หนึ่งในจุดเด่นของ Zig คือการมีตัวดำเนินการแบบ Wrapping และ Saturating ที่ไม่มีในภาษา C ทั่วไป

---

## Step 41: ตัวดำเนินการทางคณิตศาสตร์ (Arithmetic Operators)

ตัวดำเนินการทางคณิตศาสตร์พื้นฐานใน Zig ใช้งานเหมือนกับภาษา C ส่วนใหญ่ แต่มีความแตกต่างสำคัญคือ Zig จะตรวจสอบ overflow โดยค่าเริ่มต้นในโหมด Debug

### ตารางตัวดำเนินการทางคณิตศาสตร์

| ตัวดำเนินการ | ชื่อ | ตัวอย่าง | ผลลัพธ์ |
|:---:|:---:|:---:|:---:|
| `+` | บวก | `5 + 3` | `8` |
| `-` | ลบ | `5 - 3` | `2` |
| `*` | คูณ | `5 * 3` | `15` |
| `/` | หาร | `10 / 3` | `3` (จำนวนเต็ม) |
| `%` | เศษจากการหาร | `10 % 3` | `1` |
| `-` (unary) | ลบเอกภาค | `-5` | `-5` |

### ตัวอย่างการใช้งาน

```zig
const std = @import("std");

pub fn main() void {
    const a: i32 = 10;
    const b: i32 = 3;

    // การบวก
    const sum = a + b;
    std.debug.print("10 + 3 = {}\n", .{sum}); // 13

    // การลบ
    const diff = a - b;
    std.debug.print("10 - 3 = {}\n", .{diff}); // 7

    // การคูณ
    const product = a * b;
    std.debug.print("10 * 3 = {}\n", .{product}); // 30

    // การหาร (จำนวนเต็ม)
    const quotient = a / b;
    std.debug.print("10 / 3 = {}\n", .{quotient}); // 3

    // เศษจากการหาร
    const remainder = a % b;
    std.debug.print("10 %% 3 = {}\n", .{remainder}); // 1
}
```

### การหารจำนวนจริง (Float Division)

```zig
const std = @import("std");

pub fn main() void {
    const x: f64 = 10.0;
    const y: f64 = 3.0;

    const result = x / y;
    std.debug.print("10.0 / 3.0 = {d:.4}\n", .{result}); // 3.3333

    // การหารแบบจำนวนเต็มแล้วแปลงเป็น float
    const a: i32 = 10;
    const b: i32 = 3;
    const float_result = @as(f64, @floatFromInt(a)) / @as(f64, @floatFromInt(b));
    std.debug.print("10/3 as float = {d:.4}\n", .{float_result}); // 3.3333
}
```

### ข้อควรระวัง: Overflow ในโหมด Debug

```zig
const std = @import("std");

pub fn main() void {
    const max: u8 = 255;
    // บรรทัดต่อไปนี้จะทำให้เกิด panic ในโหมด Debug!
    // const overflow = max + 1; // ERROR: integer overflow

    // วิธีที่ถูกต้อง: ใช้ชนิดข้อมูลที่ใหญ่กว่า
    const bigger: u16 = @as(u16, max) + 1;
    std.debug.print("255 + 1 as u16 = {}\n", .{bigger}); // 256
}
```

### ความแตกต่างจาก C

ใน C การ overflow ของจำนวนเต็มแบบไม่มีเครื่องหมาย (unsigned) เป็นพฤติกรรมที่กำหนดไว้ (wraps around) แต่สำหรับจำนวนเต็มแบบมีเครื่องหมาย (signed) การ overflow เป็น Undefined Behavior

ใน Zig โหมด Debug จะตรวจจับ overflow ทั้งสองแบบและทำให้เกิด runtime panic ทำให้ debug ง่ายขึ้นมาก

---

## Step 42: Wrapping Arithmetic (+%, -%, *%)

Wrapping Arithmetic คือตัวดำเนินการพิเศษของ Zig ที่ทำการคำนวณแบบ "วนรอบ" เมื่อค่าเกิน range ของชนิดข้อมูล โดยไม่ทำให้เกิด panic แม้ในโหมด Debug

### ตารางตัวดำเนินการ Wrapping

| ตัวดำเนินการ | ชื่อ | คำอธิบาย |
|:---:|:---:|:---|
| `+%` | Wrapping add | บวกแบบ wrap around |
| `-%` | Wrapping subtract | ลบแบบ wrap around |
| `*%` | Wrapping multiply | คูณแบบ wrap around |
| `+%=` | Wrapping add assign | บวกแบบ wrap แล้วกำหนดค่า |
| `-%=` | Wrapping subtract assign | ลบแบบ wrap แล้วกำหนดค่า |
| `*%=` | Wrapping multiply assign | คูณแบบ wrap แล้วกำหนดค่า |

### ตัวอย่างการใช้งาน Wrapping Arithmetic

```zig
const std = @import("std");

pub fn main() void {
    // Wrapping Add สำหรับ u8
    const max_u8: u8 = 255;
    const wrapped_add = max_u8 +% 1;
    std.debug.print("255 +%% 1 = {} (u8)\n", .{wrapped_add}); // 0

    const wrapped_add2 = max_u8 +% 10;
    std.debug.print("255 +%% 10 = {} (u8)\n", .{wrapped_add2}); // 9

    // Wrapping Subtract
    const min_u8: u8 = 0;
    const wrapped_sub = min_u8 -% 1;
    std.debug.print("0 -%% 1 = {} (u8)\n", .{wrapped_sub}); // 255

    // Wrapping Multiply
    const large: u8 = 200;
    const wrapped_mul = large *% 2;
    std.debug.print("200 *%% 2 = {} (u8)\n", .{wrapped_mul}); // 144 (400 mod 256)

    // กับจำนวนเต็มแบบมีเครื่องหมาย
    const max_i8: i8 = 127;
    const wrapped_signed = max_i8 +% 1;
    std.debug.print("127 +%% 1 = {} (i8)\n", .{wrapped_signed}); // -128

    const min_i8: i8 = -128;
    const wrapped_signed2 = min_i8 -% 1;
    std.debug.print("-128 -%% 1 = {} (i8)\n", .{wrapped_signed2}); // 127
}
```

### การคำนวณ Wrapping แบบละเอียด

```zig
const std = @import("std");

pub fn main() void {
    // u8 มีค่าตั้งแต่ 0 ถึง 255 (256 ค่า)
    // เมื่อ overflow: (value + overflow) mod 256

    const a: u8 = 250;
    const b: u8 = 10;

    // ปกติจะเกิด overflow และ panic ใน debug mode
    // const normal = a + b; // PANIC!

    // ด้วย wrapping: (250 + 10) = 260, 260 mod 256 = 4
    const wrapped = a +% b;
    std.debug.print("{} +%% {} = {}\n", .{ a, b, wrapped }); // 4

    // การพิสูจน์
    const expected: u8 = (250 + 10) % 256;
    std.debug.print("Expected: {}\n", .{expected}); // 4
}
```

### กรณีการใช้งานจริง: Hash Functions และ Cryptography

Wrapping arithmetic มีประโยชน์มากในการเขียน hash functions และ cryptographic algorithms ที่ต้องการพฤติกรรม overflow แบบกำหนดไว้

```zig
const std = @import("std");

// ตัวอย่าง: Simple hash function ที่ใช้ wrapping arithmetic
fn simpleHash(data: []const u8) u32 {
    var hash: u32 = 5381;
    for (data) |byte| {
        // hash * 33 + byte (wrapping)
        hash = hash *% 33 +% byte;
    }
    return hash;
}

pub fn main() void {
    const text = "Hello, Zig!";
    const hash = simpleHash(text);
    std.debug.print("Hash of \"{s}\": {}\n", .{ text, hash });

    const text2 = "Hello, World!";
    const hash2 = simpleHash(text2);
    std.debug.print("Hash of \"{s}\": {}\n", .{ text2, hash2 });
}
```

### ทำไมถึงต้องมี Wrapping Arithmetic แยกจาก Regular Arithmetic?

การออกแบบนี้ทำให้โค้ด Zig มีความปลอดภัยมากขึ้น:
- **ชัดเจนในเจตนา**: เมื่อเห็น `+%` ผู้อ่านรู้ทันทีว่าโปรแกรมเมอร์ตั้งใจให้ wrap around
- **ตรวจจับ bug**: overflow ที่ไม่ตั้งใจจะถูกตรวจพบใน debug mode
- **ไม่มี undefined behavior**: ต่างจาก C ที่ signed overflow เป็น UB

---

## Step 43: Saturating Arithmetic (+|, -|, *|)

Saturating Arithmetic หรือการคำนวณแบบ "อิ่มตัว" คือการคำนวณที่เมื่อค่าเกิน range จะหยุดที่ค่าสูงสุดหรือต่ำสุดของชนิดข้อมูล แทนที่จะ wrap around

### ตารางตัวดำเนินการ Saturating

| ตัวดำเนินการ | ชื่อ | คำอธิบาย |
|:---:|:---:|:---|
| `+\|` | Saturating add | บวกแบบ saturate ที่ค่าสูงสุด |
| `-\|` | Saturating subtract | ลบแบบ saturate ที่ค่าต่ำสุด |
| `*\|` | Saturating multiply | คูณแบบ saturate |
| `<<\|` | Saturating shift left | เลื่อนบิตซ้ายแบบ saturate |
| `+\|=` | Saturating add assign | บวกแบบ saturate แล้วกำหนดค่า |
| `-\|=` | Saturating subtract assign | ลบแบบ saturate แล้วกำหนดค่า |
| `*\|=` | Saturating multiply assign | คูณแบบ saturate แล้วกำหนดค่า |
| `<<\|=` | Saturating shift left assign | เลื่อนบิตซ้ายแบบ saturate แล้วกำหนดค่า |

### ตัวอย่างการใช้งาน Saturating Arithmetic

```zig
const std = @import("std");

pub fn main() void {
    // Saturating Add สำหรับ u8
    const max_u8: u8 = 255;
    const sat_add = max_u8 +| 1;
    std.debug.print("255 +| 1 = {} (u8)\n", .{sat_add}); // 255 (ไม่ overflow!)

    const sat_add2 = max_u8 +| 100;
    std.debug.print("255 +| 100 = {} (u8)\n", .{sat_add2}); // 255

    // Saturating Subtract
    const min_u8: u8 = 0;
    const sat_sub = min_u8 -| 1;
    std.debug.print("0 -| 1 = {} (u8)\n", .{sat_sub}); // 0 (ไม่ underflow!)

    const sat_sub2 = min_u8 -| 100;
    std.debug.print("0 -| 100 = {} (u8)\n", .{sat_sub2}); // 0

    // กรณีปกติที่ไม่ overflow
    const a: u8 = 100;
    const b: u8 = 50;
    const sat_normal = a +| b;
    std.debug.print("100 +| 50 = {} (u8)\n", .{sat_normal}); // 150

    // กับจำนวนเต็มแบบมีเครื่องหมาย
    const max_i8: i8 = 127;
    const sat_signed = max_i8 +| 10;
    std.debug.print("127 +| 10 = {} (i8)\n", .{sat_signed}); // 127

    const min_i8: i8 = -128;
    const sat_signed2 = min_i8 -| 10;
    std.debug.print("-128 -| 10 = {} (i8)\n", .{sat_signed2}); // -128
}
```

### กรณีการใช้งานจริง: การประมวลผลสัญญาณเสียง/ภาพ

```zig
const std = @import("std");

// จำลองการปรับระดับเสียง (audio mixing)
// ค่า sample เสียงคือ i16 (ตั้งแต่ -32768 ถึง 32767)
fn mixAudio(sample1: i16, sample2: i16) i16 {
    // ใช้ saturating add เพื่อป้องกัน clipping
    return sample1 +| sample2;
}

// จำลองการปรับความสว่างภาพ
fn addBrightness(pixel: u8, brightness: u8) u8 {
    // ใช้ saturating add เพื่อให้ค่าไม่เกิน 255
    return pixel +| brightness;
}

pub fn main() void {
    // Audio mixing
    const loud1: i16 = 30000;
    const loud2: i16 = 30000;
    const mixed = mixAudio(loud1, loud2);
    std.debug.print("Mixed audio: {} (saturated at 32767)\n", .{mixed}); // 32767

    // Image brightness
    const pixel: u8 = 200;
    const brightness: u8 = 100;
    const bright_pixel = addBrightness(pixel, brightness);
    std.debug.print("Bright pixel: {} (saturated at 255)\n", .{bright_pixel}); // 255

    // กรณีปกติ
    const dark_pixel: u8 = 50;
    const normal_bright = addBrightness(dark_pixel, 100);
    std.debug.print("Normal pixel: {} (no saturation)\n", .{normal_bright}); // 150
}
```

### เปรียบเทียบ Regular, Wrapping, Saturating

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 250;
    const b: u8 = 10;

    // Regular: panic ใน debug mode
    // const reg = a + b; // PANIC in debug!

    // Wrapping: 260 mod 256 = 4
    const wrap = a +% b;

    // Saturating: หยุดที่ 255
    const sat = a +| b;

    std.debug.print("Wrapping:   250 +%% 10 = {}\n", .{wrap}); // 4
    std.debug.print("Saturating: 250 +|  10 = {}\n", .{sat});  // 255
}
```

---

## Step 44: ตัวดำเนินการเปรียบเทียบ (Comparison Operators)

ตัวดำเนินการเปรียบเทียบใช้สำหรับเปรียบเทียบค่าสองค่า และให้ผลลัพธ์เป็น `bool` (true หรือ false)

### ตารางตัวดำเนินการเปรียบเทียบ

| ตัวดำเนินการ | ชื่อ | ตัวอย่าง | ผลลัพธ์ |
|:---:|:---:|:---:|:---:|
| `==` | เท่ากับ | `5 == 5` | `true` |
| `!=` | ไม่เท่ากับ | `5 != 3` | `true` |
| `<` | น้อยกว่า | `3 < 5` | `true` |
| `>` | มากกว่า | `5 > 3` | `true` |
| `<=` | น้อยกว่าหรือเท่ากับ | `5 <= 5` | `true` |
| `>=` | มากกว่าหรือเท่ากับ | `5 >= 6` | `false` |

### ตัวอย่างการใช้งาน

```zig
const std = @import("std");

pub fn main() void {
    const a: i32 = 10;
    const b: i32 = 20;
    const c: i32 = 10;

    // เท่ากับ
    std.debug.print("10 == 20: {}\n", .{a == b}); // false
    std.debug.print("10 == 10: {}\n", .{a == c}); // true

    // ไม่เท่ากับ
    std.debug.print("10 != 20: {}\n", .{a != b}); // true
    std.debug.print("10 != 10: {}\n", .{a != c}); // false

    // น้อยกว่า
    std.debug.print("10 < 20: {}\n", .{a < b}); // true
    std.debug.print("20 < 10: {}\n", .{b < a}); // false

    // มากกว่า
    std.debug.print("20 > 10: {}\n", .{b > a}); // true

    // น้อยกว่าหรือเท่ากับ
    std.debug.print("10 <= 10: {}\n", .{a <= c}); // true
    std.debug.print("10 <= 20: {}\n", .{a <= b}); // true

    // มากกว่าหรือเท่ากับ
    std.debug.print("20 >= 10: {}\n", .{b >= a}); // true
}
```

### การเปรียบเทียบชนิดต่าง ๆ

```zig
const std = @import("std");

pub fn main() void {
    // การเปรียบเทียบจำนวนจริง
    const x: f64 = 3.14;
    const y: f64 = 2.71;
    std.debug.print("3.14 > 2.71: {}\n", .{x > y}); // true

    // การเปรียบเทียบ bool
    const t: bool = true;
    const f: bool = false;
    std.debug.print("true == false: {}\n", .{t == f}); // false
    std.debug.print("true != false: {}\n", .{t != f}); // true

    // ข้อควรระวัง: ไม่สามารถเปรียบเทียบต่างชนิดกันตรง ๆ
    const int_val: i32 = 5;
    const float_val: f32 = 5.0;
    // const result = int_val == float_val; // ERROR: ต้องแปลงชนิดก่อน

    const result = @as(f32, @floatFromInt(int_val)) == float_val;
    std.debug.print("5 == 5.0 (after cast): {}\n", .{result}); // true
}
```

### การเปรียบเทียบ Optional Values

```zig
const std = @import("std");

pub fn main() void {
    const maybe: ?i32 = 42;
    const nothing: ?i32 = null;

    // เปรียบเทียบกับ null
    std.debug.print("maybe == null: {}\n", .{maybe == null}); // false
    std.debug.print("nothing == null: {}\n", .{nothing == null}); // true

    // เปรียบเทียบ optional กับ optional
    const another: ?i32 = 42;
    std.debug.print("maybe == another: {}\n", .{maybe == another}); // true

    const different: ?i32 = 100;
    std.debug.print("maybe == different: {}\n", .{maybe == different}); // false
}
```

---

## Step 45: ตัวดำเนินการตรรกศาสตร์ (Logical Operators)

ตัวดำเนินการตรรกศาสตร์ใช้สำหรับการดำเนินการกับค่า boolean

### ตารางตัวดำเนินการตรรกศาสตร์

| ตัวดำเนินการ | ชื่อ | คำอธิบาย |
|:---:|:---:|:---|
| `and` | AND ตรรกะ | true ก็ต่อเมื่อทั้งสองข้าง true |
| `or` | OR ตรรกะ | true ถ้าอย่างน้อยหนึ่งข้าง true |
| `!` | NOT ตรรกะ | กลับค่า boolean |

### ตารางความจริง (Truth Tables)

**AND:**
| A | B | A and B |
|:---:|:---:|:---:|
| true | true | true |
| true | false | false |
| false | true | false |
| false | false | false |

**OR:**
| A | B | A or B |
|:---:|:---:|:---:|
| true | true | true |
| true | false | true |
| false | true | true |
| false | false | false |

**NOT:**
| A | !A |
|:---:|:---:|
| true | false |
| false | true |

### ตัวอย่างการใช้งาน

```zig
const std = @import("std");

pub fn main() void {
    const a: bool = true;
    const b: bool = false;

    // AND
    std.debug.print("true and true: {}\n", .{a and a});   // true
    std.debug.print("true and false: {}\n", .{a and b});  // false
    std.debug.print("false and false: {}\n", .{b and b}); // false

    // OR
    std.debug.print("true or false: {}\n", .{a or b});   // true
    std.debug.print("false or false: {}\n", .{b or b}); // false

    // NOT
    std.debug.print("!true: {}\n", .{!a});  // false
    std.debug.print("!false: {}\n", .{!b}); // true

    // การรวมกัน
    const x: i32 = 5;
    const y: i32 = 10;
    const z: i32 = 15;

    const in_range = x > 0 and x < 10;
    std.debug.print("5 is between 0 and 10: {}\n", .{in_range}); // true

    const is_extreme = y < 5 or y > 12;
    std.debug.print("10 is extreme: {}\n", .{is_extreme}); // false

    const complex = (x < y) and (y < z) and !(z > 20);
    std.debug.print("5 < 10 < 15 and 15 <= 20: {}\n", .{complex}); // true
}
```

### Short-Circuit Evaluation (การประเมินแบบลัดวงจร)

Zig ใช้ short-circuit evaluation เหมือน C คือถ้าผลลัพธ์ของนิพจน์แรกกำหนดผลลัพธ์สุดท้ายได้แล้ว จะไม่ประเมินนิพจน์ที่สอง

```zig
const std = @import("std");

fn expensiveCheck() bool {
    std.debug.print("expensive check called!\n", .{});
    return true;
}

fn mayFail() bool {
    std.debug.print("mayFail called!\n", .{});
    return false;
}

pub fn main() void {
    // Short-circuit AND: ถ้าฝั่งซ้ายเป็น false จะไม่ประเมินฝั่งขวา
    const result1 = mayFail() and expensiveCheck();
    std.debug.print("Result 1: {}\n", .{result1});
    // พิมพ์: "mayFail called!" แต่ไม่พิมพ์ "expensive check called!"

    std.debug.print("---\n", .{});

    // Short-circuit OR: ถ้าฝั่งซ้ายเป็น true จะไม่ประเมินฝั่งขวา
    const result2 = expensiveCheck() or mayFail();
    std.debug.print("Result 2: {}\n", .{result2});
    // พิมพ์: "expensive check called!" แต่ไม่พิมพ์ "mayFail called!"
}
```

### การใช้ในเงื่อนไข if

```zig
const std = @import("std");

pub fn checkAge(age: u8) void {
    if (age >= 18 and age <= 65) {
        std.debug.print("อายุ {} ปี: อยู่ในวัยทำงาน\n", .{age});
    } else if (age < 18) {
        std.debug.print("อายุ {} ปี: ยังเด็กเกินไป\n", .{age});
    } else {
        std.debug.print("อายุ {} ปี: วัยผู้สูงอายุ\n", .{age});
    }
}

pub fn main() void {
    checkAge(15);
    checkAge(30);
    checkAge(70);
}
```

---

## Step 46: ตัวดำเนินการบิต (Bitwise Operators)

ตัวดำเนินการบิตทำงานในระดับบิตของค่าจำนวนเต็ม มีประโยชน์มากในการเขียนโปรแกรมระดับต่ำ, การจัดการ flags, และการเพิ่มประสิทธิภาพ

### ตารางตัวดำเนินการบิต

| ตัวดำเนินการ | ชื่อ | คำอธิบาย |
|:---:|:---:|:---|
| `&` | Bitwise AND | AND ทีละบิต |
| `\|` | Bitwise OR | OR ทีละบิต |
| `^` | Bitwise XOR | XOR ทีละบิต |
| `~` | Bitwise NOT | กลับบิตทั้งหมด |
| `<<` | Shift left | เลื่อนบิตไปทางซ้าย |
| `>>` | Shift right | เลื่อนบิตไปทางขวา |

### ตารางความจริงระดับบิต

**Bitwise AND (&):**
| A | B | A & B |
|:---:|:---:|:---:|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

**Bitwise OR (|):**
| A | B | A \| B |
|:---:|:---:|:---:|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

**Bitwise XOR (^):**
| A | B | A ^ B |
|:---:|:---:|:---:|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

### ตัวอย่างการใช้งาน

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 0b1100_1010; // 202
    const b: u8 = 0b1010_0101; // 165

    std.debug.print("a = {b:0>8} ({d})\n", .{ a, a });
    std.debug.print("b = {b:0>8} ({d})\n", .{ b, b });
    std.debug.print("\n", .{});

    // Bitwise AND
    const and_result = a & b;
    std.debug.print("a & b  = {b:0>8} ({d})\n", .{ and_result, and_result }); // 0b10000000 = 128

    // Bitwise OR
    const or_result = a | b;
    std.debug.print("a | b  = {b:0>8} ({d})\n", .{ or_result, or_result }); // 0b11101111 = 239

    // Bitwise XOR
    const xor_result = a ^ b;
    std.debug.print("a ^ b  = {b:0>8} ({d})\n", .{ xor_result, xor_result }); // 0b01101111 = 111

    // Bitwise NOT
    const not_result = ~a;
    std.debug.print("~a     = {b:0>8} ({d})\n", .{ not_result, not_result }); // 0b00110101 = 53
}
```

### การใช้ Bitwise สำหรับ Bit Flags

```zig
const std = @import("std");

// Permission flags
const PERM_READ: u8 = 0b0000_0001;    // bit 0
const PERM_WRITE: u8 = 0b0000_0010;   // bit 1
const PERM_EXECUTE: u8 = 0b0000_0100; // bit 2
const PERM_DELETE: u8 = 0b0000_1000;  // bit 3

pub fn checkPermission(perms: u8, flag: u8) bool {
    return (perms & flag) != 0;
}

pub fn addPermission(perms: u8, flag: u8) u8 {
    return perms | flag;
}

pub fn removePermission(perms: u8, flag: u8) u8 {
    return perms & ~flag;
}

pub fn togglePermission(perms: u8, flag: u8) u8 {
    return perms ^ flag;
}

pub fn main() void {
    var user_perms: u8 = 0;

    // เพิ่ม permission
    user_perms = addPermission(user_perms, PERM_READ);
    user_perms = addPermission(user_perms, PERM_WRITE);
    std.debug.print("After adding READ and WRITE: {b:0>8}\n", .{user_perms});

    // ตรวจสอบ permission
    std.debug.print("Can read: {}\n", .{checkPermission(user_perms, PERM_READ)});    // true
    std.debug.print("Can write: {}\n", .{checkPermission(user_perms, PERM_WRITE)});   // true
    std.debug.print("Can execute: {}\n", .{checkPermission(user_perms, PERM_EXECUTE)}); // false

    // ลบ permission
    user_perms = removePermission(user_perms, PERM_WRITE);
    std.debug.print("After removing WRITE: {b:0>8}\n", .{user_perms});
    std.debug.print("Can write now: {}\n", .{checkPermission(user_perms, PERM_WRITE)}); // false

    // Toggle permission
    user_perms = togglePermission(user_perms, PERM_EXECUTE);
    std.debug.print("After toggling EXECUTE: {b:0>8}\n", .{user_perms});
}
```

### Bitwise สำหรับการเพิ่มประสิทธิภาพ

```zig
const std = @import("std");

pub fn main() void {
    const x: u32 = 100;

    // การคูณด้วย 2 โดยใช้ shift
    const doubled = x << 1;
    std.debug.print("{} * 2 = {} (using <<)\n", .{ x, doubled }); // 200

    // การหารด้วย 2 โดยใช้ shift
    const halved = x >> 1;
    std.debug.print("{} / 2 = {} (using >>)\n", .{ x, halved }); // 50

    // ตรวจสอบว่าเป็นเลขคู่หรือเลขคี่
    const is_even = (x & 1) == 0;
    std.debug.print("{} is even: {}\n", .{ x, is_even }); // true

    const y: u32 = 7;
    const y_is_even = (y & 1) == 0;
    std.debug.print("{} is even: {}\n", .{ y, y_is_even }); // false

    // การคูณด้วย powers of 2
    const val: u32 = 5;
    std.debug.print("5 * 4  = {} (5 << 2)\n", .{val << 2});   // 20
    std.debug.print("5 * 8  = {} (5 << 3)\n", .{val << 3});   // 40
    std.debug.print("5 * 16 = {} (5 << 4)\n", .{val << 4});   // 80
}
```

---

## Step 47: Shift Operators และ Bit Rotation

### Shift Operators อย่างละเอียด

```zig
const std = @import("std");

pub fn main() void {
    const val: u8 = 0b0000_0001; // 1

    std.debug.print("Original: {b:0>8} = {}\n", .{ val, val });

    // Shift left: คูณด้วย 2 ทุกครั้งที่เลื่อน 1 บิต
    std.debug.print("<< 1:     {b:0>8} = {}\n", .{ val << 1, val << 1 }); // 2
    std.debug.print("<< 2:     {b:0>8} = {}\n", .{ val << 2, val << 2 }); // 4
    std.debug.print("<< 3:     {b:0>8} = {}\n", .{ val << 3, val << 3 }); // 8
    std.debug.print("<< 7:     {b:0>8} = {}\n", .{ val << 7, val << 7 }); // 128

    std.debug.print("\n", .{});

    const val2: u8 = 0b1000_0000; // 128

    // Shift right: หารด้วย 2 ทุกครั้ง
    std.debug.print("Original: {b:0>8} = {}\n", .{ val2, val2 });
    std.debug.print(">> 1:     {b:0>8} = {}\n", .{ val2 >> 1, val2 >> 1 }); // 64
    std.debug.print(">> 2:     {b:0>8} = {}\n", .{ val2 >> 2, val2 >> 2 }); // 32
    std.debug.print(">> 7:     {b:0>8} = {}\n", .{ val2 >> 7, val2 >> 7 }); // 1
}
```

### Arithmetic vs Logical Shift Right

```zig
const std = @import("std");

pub fn main() void {
    // สำหรับ unsigned (logical shift): เติม 0 ทางซ้าย
    const unsigned: u8 = 0b1000_0000; // -128 เป็น signed
    const u_shifted = unsigned >> 1;
    std.debug.print("u8: {b:0>8} >> 1 = {b:0>8}\n", .{ unsigned, u_shifted });
    // 0b0100_0000 (เติม 0 ทางซ้าย)

    // สำหรับ signed (arithmetic shift): เติม sign bit ทางซ้าย
    const signed: i8 = -128; // 0b1000_0000
    const s_shifted = signed >> 1;
    std.debug.print("i8: {} >> 1 = {}\n", .{ signed, s_shifted });
    // -64 (เติม 1 ทางซ้าย เพื่อรักษาเครื่องหมาย)

    const positive: i8 = 64;
    const p_shifted = positive >> 1;
    std.debug.print("i8: {} >> 1 = {}\n", .{ positive, p_shifted });
    // 32 (เติม 0 ทางซ้าย เพราะเป็นบวก)
}
```

### Bit Rotation ด้วย @rotl และ @rotr

Zig มี built-in functions สำหรับ bit rotation:

```zig
const std = @import("std");

pub fn main() void {
    const val: u8 = 0b1010_0001; // 161

    std.debug.print("Original:  {b:0>8} = {}\n", .{ val, val });

    // Rotate left: บิตที่หลุดออกทางซ้ายจะเข้าทางขวา
    const rotl1 = @bitReverse(val); // note: using available function
    _ = rotl1;

    // ใช้ @rotl และ @rotr โดยตรง
    const rot_left_1 = std.math.rotl(u8, val, 1);
    const rot_left_4 = std.math.rotl(u8, val, 4);
    const rot_right_1 = std.math.rotr(u8, val, 1);
    const rot_right_4 = std.math.rotr(u8, val, 4);

    std.debug.print("rotl 1:    {b:0>8} = {}\n", .{ rot_left_1, rot_left_1 });
    std.debug.print("rotl 4:    {b:0>8} = {}\n", .{ rot_left_4, rot_left_4 });
    std.debug.print("rotr 1:    {b:0>8} = {}\n", .{ rot_right_1, rot_right_1 });
    std.debug.print("rotr 4:    {b:0>8} = {}\n", .{ rot_right_4, rot_right_4 });
}
```

### การนับบิต

```zig
const std = @import("std");

pub fn main() void {
    const val: u8 = 0b1010_1010; // 170

    // นับจำนวนบิตที่เป็น 1 (@popCount)
    const ones = @popCount(val);
    std.debug.print("Number of 1 bits in {b:0>8}: {}\n", .{ val, ones }); // 4

    // หาตำแหน่งบิตที่เป็น 1 แรกนับจากซ้าย (@clz = count leading zeros)
    const leading = @clz(val);
    std.debug.print("Leading zeros in {b:0>8}: {}\n", .{ val, leading }); // 0

    // หาตำแหน่งบิตที่เป็น 1 แรกนับจากขวา (@ctz = count trailing zeros)
    const trailing = @ctz(val);
    std.debug.print("Trailing zeros in {b:0>8}: {}\n", .{ val, trailing }); // 1

    const power_of_2: u8 = 0b0001_0000; // 16
    const p2_trailing = @ctz(power_of_2);
    std.debug.print("log2(16) = {} (trailing zeros)\n", .{p2_trailing}); // 4
}
```

---

## Step 48: ตัวดำเนินการกำหนดค่า (Assignment Operators)

ตัวดำเนินการกำหนดค่าใช้สำหรับกำหนดค่าให้กับตัวแปร Zig มีตัวดำเนินการแบบ compound ที่รวมการคำนวณและการกำหนดค่าเข้าด้วยกัน

### ตารางตัวดำเนินการกำหนดค่า

| ตัวดำเนินการ | ชื่อ | เทียบเท่า |
|:---:|:---:|:---:|
| `=` | กำหนดค่า | `x = value` |
| `+=` | บวกแล้วกำหนด | `x = x + value` |
| `-=` | ลบแล้วกำหนด | `x = x - value` |
| `*=` | คูณแล้วกำหนด | `x = x * value` |
| `/=` | หารแล้วกำหนด | `x = x / value` |
| `%=` | หารเอาเศษแล้วกำหนด | `x = x % value` |
| `&=` | bitwise AND แล้วกำหนด | `x = x & value` |
| `\|=` | bitwise OR แล้วกำหนด | `x = x \| value` |
| `^=` | bitwise XOR แล้วกำหนด | `x = x ^ value` |
| `<<=` | shift left แล้วกำหนด | `x = x << value` |
| `>>=` | shift right แล้วกำหนด | `x = x >> value` |
| `+%=` | wrapping add แล้วกำหนด | `x = x +% value` |
| `-%=` | wrapping sub แล้วกำหนด | `x = x -% value` |
| `*%=` | wrapping mul แล้วกำหนด | `x = x *% value` |
| `+\|=` | saturating add แล้วกำหนด | `x = x +\| value` |
| `-\|=` | saturating sub แล้วกำหนด | `x = x -\| value` |

### ตัวอย่างการใช้งาน

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 10;
    std.debug.print("Initial x = {}\n", .{x});

    // ตัวดำเนินการทางคณิตศาสตร์แบบกำหนดค่า
    x += 5;
    std.debug.print("After x += 5: {}\n", .{x}); // 15

    x -= 3;
    std.debug.print("After x -= 3: {}\n", .{x}); // 12

    x *= 2;
    std.debug.print("After x *= 2: {}\n", .{x}); // 24

    x /= 4;
    std.debug.print("After x /= 4: {}\n", .{x}); // 6

    x %= 4;
    std.debug.print("After x %%= 4: {}\n", .{x}); // 2

    // ตัวดำเนินการบิตแบบกำหนดค่า
    var flags: u8 = 0b1111_0000;
    std.debug.print("\nInitial flags: {b:0>8}\n", .{flags});

    flags &= 0b1010_1010;
    std.debug.print("After &= 0b10101010: {b:0>8}\n", .{flags}); // 0b10100000

    flags |= 0b0000_1111;
    std.debug.print("After |= 0b00001111: {b:0>8}\n", .{flags}); // 0b10101111

    flags ^= 0b1111_1111;
    std.debug.print("After ^= 0b11111111: {b:0>8}\n", .{flags}); // 0b01010000

    // Shift
    var bits: u8 = 0b0000_0001;
    bits <<= 3;
    std.debug.print("\nAfter <<= 3: {b:0>8}\n", .{bits}); // 0b00001000

    bits >>= 1;
    std.debug.print("After >>= 1: {b:0>8}\n", .{bits}); // 0b00000100
}
```

### การใช้ Compound Assignment ในลูป

```zig
const std = @import("std");

pub fn main() void {
    // การสะสมผลรวม
    var sum: i32 = 0;
    var i: i32 = 1;
    while (i <= 10) : (i += 1) {
        sum += i;
    }
    std.debug.print("Sum 1 to 10: {}\n", .{sum}); // 55

    // การสะสมผลคูณ (factorial)
    var factorial: u64 = 1;
    var n: u8 = 1;
    while (n <= 10) : (n += 1) {
        factorial *= n;
    }
    std.debug.print("10! = {}\n", .{factorial}); // 3628800

    // การสร้าง bitmask
    var mask: u32 = 0;
    var bit: u5 = 0;
    while (bit < 8) : (bit += 1) {
        mask |= @as(u32, 1) << bit;
    }
    std.debug.print("8-bit mask: {b:0>32}\n", .{mask}); // ล่าง 8 บิตเป็น 1
}
```

---

## Step 49: ลำดับความสำคัญของตัวดำเนินการ (Operator Precedence)

ลำดับความสำคัญกำหนดว่าตัวดำเนินการใดจะถูกประเมินก่อนในนิพจน์ที่ซับซ้อน

### ตารางลำดับความสำคัญ (จากสูงไปต่ำ)

| ระดับ | ตัวดำเนินการ | ชื่อ |
|:---:|:---:|:---|
| 1 (สูงสุด) | `x.y`, `x.?`, `x.*`, `x[i]` | Member access, unwrap, deref, index |
| 2 | `!x`, `-x`, `~x`, `&x`, `try`, `await` | Unary |
| 3 | `*`, `/`, `%`, `**`, `*%`, `*\|` | Multiplication |
| 4 | `+`, `-`, `++`, `+%`, `-%`, `+\|`, `-\|` | Addition |
| 5 | `<<`, `>>`, `<<\|` | Shift |
| 6 | `&` | Bitwise AND |
| 7 | `^` | Bitwise XOR |
| 8 | `\|` | Bitwise OR |
| 9 | `==`, `!=`, `<`, `>`, `<=`, `>=` | Comparison |
| 10 | `and` | Logical AND |
| 11 | `or` | Logical OR |
| 12 (ต่ำสุด) | `=`, `+=`, `-=`, ... | Assignment |

### ตัวอย่างลำดับความสำคัญ

```zig
const std = @import("std");

pub fn main() void {
    // คูณก่อนบวก
    const a = 2 + 3 * 4;
    std.debug.print("2 + 3 * 4 = {} (expected: 14)\n", .{a}); // 14

    // วงเล็บเปลี่ยนลำดับ
    const b = (2 + 3) * 4;
    std.debug.print("(2 + 3) * 4 = {} (expected: 20)\n", .{b}); // 20

    // Shift ก่อน Bitwise
    const c = 1 | 2 << 2;
    // = 1 | (2 << 2) = 1 | 8 = 9
    std.debug.print("1 | 2 << 2 = {} (expected: 9)\n", .{c}); // 9

    // Comparison ก่อน Logical
    const x: i32 = 5;
    const d = x > 3 and x < 10;
    // = (x > 3) and (x < 10) = true and true = true
    std.debug.print("5 > 3 and 5 < 10 = {} (expected: true)\n", .{d}); // true

    // ซับซ้อนมากขึ้น
    const e = 2 + 3 * 4 - 1;
    // = 2 + (3 * 4) - 1 = 2 + 12 - 1 = 13
    std.debug.print("2 + 3 * 4 - 1 = {} (expected: 13)\n", .{e}); // 13

    // Bitwise AND ก่อน XOR ก่อน OR
    const f = 0b1111 & 0b1010 | 0b0101 ^ 0b0011;
    // = (0b1111 & 0b1010) | (0b0101 ^ 0b0011)
    // = 0b1010 | 0b0110
    // = 0b1110 = 14
    std.debug.print("0b1111 & 0b1010 | 0b0101 ^ 0b0011 = {b} (expected: 1110)\n", .{f});
}
```

### คำแนะนำ: ใช้วงเล็บเสมอเมื่อไม่แน่ใจ

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 5;
    const y: i32 = 3;
    const z: i32 = 2;

    // ยากต่อการอ่าน
    const unclear = x + y * z - x / z;

    // ชัดเจนด้วยวงเล็บ
    const clear = x + (y * z) - (x / z);

    std.debug.print("unclear = {}\n", .{unclear}); // 10
    std.debug.print("clear = {}\n", .{clear});     // 10 (เหมือนกัน แต่อ่านง่ายกว่า)

    // ตัวอย่างที่วงเล็บสำคัญมาก
    const dangerous = !false and true;
    // = (!false) and true = true and true = true
    const intended = !(false and true);
    // = !(false) = true

    std.debug.print("!false and true = {}\n", .{dangerous}); // true
    std.debug.print("!(false and true) = {}\n", .{intended}); // true
    // ในกรณีนี้ผลเหมือนกัน แต่อาจต่างกันในกรณีอื่น

    const case2 = !true and false;
    // = (!true) and false = false and false = false
    const case2_paren = !(true and false);
    // = !(false) = true

    std.debug.print("!true and false = {}\n", .{case2});        // false
    std.debug.print("!(true and false) = {}\n", .{case2_paren}); // true ต่างกัน!
}
```

### Associativity (การเชื่อมโยง)

ตัวดำเนินการส่วนใหญ่มี left-to-right associativity:

```zig
const std = @import("std");

pub fn main() void {
    // Left-to-right: คำนวณจากซ้ายไปขวา
    const a = 10 - 3 - 2;
    // = (10 - 3) - 2 = 7 - 2 = 5
    std.debug.print("10 - 3 - 2 = {} (left-to-right: 5)\n", .{a}); // 5

    const b = 8 / 4 / 2;
    // = (8 / 4) / 2 = 2 / 2 = 1
    std.debug.print("8 / 4 / 2 = {} (left-to-right: 1)\n", .{b}); // 1

    // Unary operators เป็น right-to-left
    var val: bool = false;
    const c = !!val; // !!false = !(!false) = !(true) = false
    std.debug.print("!!false = {} (right-to-left: false)\n", .{c}); // false

    val = true;
    const d = !!val; // !!true = !(!true) = !(false) = true
    std.debug.print("!!true = {} (right-to-left: true)\n", .{d}); // true
}
```

---

## Step 50: ตัวดำเนินการพิเศษ (@sizeOf, @typeOf, @alignOf)

Zig มี built-in functions ที่ขึ้นต้นด้วย `@` หลายอย่างที่ทำหน้าที่เหมือนตัวดำเนินการ

### ตารางตัวดำเนินการ @ ที่สำคัญ

| ตัวดำเนินการ | คำอธิบาย |
|:---:|:---|
| `@sizeOf(T)` | ขนาดของชนิดข้อมูล T ในหน่วย bytes |
| `@bitSizeOf(T)` | ขนาดของชนิดข้อมูล T ในหน่วย bits |
| `@alignOf(T)` | alignment requirement ของชนิดข้อมูล T |
| `@TypeOf(expr)` | ชนิดข้อมูลของ expression |
| `@as(T, expr)` | แปลงชนิดข้อมูลของ expression เป็น T |
| `@intFromFloat(f)` | แปลง float เป็น integer |
| `@floatFromInt(i)` | แปลง integer เป็น float |
| `@intCast(x)` | แปลง integer ระหว่างขนาด |
| `@truncate(x)` | ตัดบิตออกโดยไม่ตรวจสอบ |
| `@bitCast(x)` | แปลงบิตโดยตรง |
| `@ptrToInt(p)` | แปลง pointer เป็น integer |
| `@intToPtr(T, i)` | แปลง integer เป็น pointer |
| `@fieldParentPtr(T, f, p)` | หา parent struct จาก field pointer |
| `@offsetOf(T, f)` | offset ของ field ใน struct |
| `@compileError(msg)` | สร้าง compile error |
| `@compileLog(args)` | log ตอน compile |

### @sizeOf - ขนาดของชนิดข้อมูล

```zig
const std = @import("std");

pub fn main() void {
    // ขนาดของชนิดข้อมูลพื้นฐาน
    std.debug.print("bool:    {} bytes\n", .{@sizeOf(bool)});   // 1
    std.debug.print("u8:      {} bytes\n", .{@sizeOf(u8)});     // 1
    std.debug.print("i8:      {} bytes\n", .{@sizeOf(i8)});     // 1
    std.debug.print("u16:     {} bytes\n", .{@sizeOf(u16)});    // 2
    std.debug.print("i16:     {} bytes\n", .{@sizeOf(i16)});    // 2
    std.debug.print("u32:     {} bytes\n", .{@sizeOf(u32)});    // 4
    std.debug.print("i32:     {} bytes\n", .{@sizeOf(i32)});    // 4
    std.debug.print("u64:     {} bytes\n", .{@sizeOf(u64)});    // 8
    std.debug.print("i64:     {} bytes\n", .{@sizeOf(i64)});    // 8
    std.debug.print("f32:     {} bytes\n", .{@sizeOf(f32)});    // 4
    std.debug.print("f64:     {} bytes\n", .{@sizeOf(f64)});    // 8
    std.debug.print("usize:   {} bytes\n", .{@sizeOf(usize)});  // 4 หรือ 8 (ขึ้นกับ platform)
    std.debug.print("isize:   {} bytes\n", .{@sizeOf(isize)});  // 4 หรือ 8

    // ขนาดของ struct
    const Point = struct {
        x: f32,
        y: f32,
    };
    std.debug.print("\nPoint struct: {} bytes\n", .{@sizeOf(Point)}); // 8

    // ขนาดของ struct ที่ซับซ้อน
    const Person = struct {
        age: u8,
        height: f32,
        weight: f64,
    };
    std.debug.print("Person struct: {} bytes\n", .{@sizeOf(Person)}); // อาจเป็น 16 (ขึ้นกับ padding)
}
```

### @bitSizeOf - ขนาดในหน่วยบิต

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("bit sizes:\n", .{});
    std.debug.print("bool:  {} bits\n", .{@bitSizeOf(bool)});  // 1
    std.debug.print("u8:    {} bits\n", .{@bitSizeOf(u8)});    // 8
    std.debug.print("u16:   {} bits\n", .{@bitSizeOf(u16)});   // 16
    std.debug.print("u32:   {} bits\n", .{@bitSizeOf(u32)});   // 32
    std.debug.print("u64:   {} bits\n", .{@bitSizeOf(u64)});   // 64
    std.debug.print("f32:   {} bits\n", .{@bitSizeOf(f32)});   // 32
    std.debug.print("f64:   {} bits\n", .{@bitSizeOf(f64)});   // 64

    // ชนิดที่มีขนาดแปลก ๆ
    std.debug.print("u3:    {} bits\n", .{@bitSizeOf(u3)});    // 3
    std.debug.print("u7:    {} bits\n", .{@bitSizeOf(u7)});    // 7
    std.debug.print("i13:   {} bits\n", .{@bitSizeOf(i13)});   // 13
}
```

### @alignOf - การ Alignment

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("alignments:\n", .{});
    std.debug.print("u8:      {} bytes\n", .{@alignOf(u8)});   // 1
    std.debug.print("u16:     {} bytes\n", .{@alignOf(u16)});  // 2
    std.debug.print("u32:     {} bytes\n", .{@alignOf(u32)});  // 4
    std.debug.print("u64:     {} bytes\n", .{@alignOf(u64)});  // 8
    std.debug.print("f32:     {} bytes\n", .{@alignOf(f32)});  // 4
    std.debug.print("f64:     {} bytes\n", .{@alignOf(f64)});  // 8

    // การ alignment ของ struct
    const Aligned = struct {
        x: u8,
        y: u32, // ต้องการ alignment 4
        z: u8,
    };
    std.debug.print("\nAligned struct align: {}\n", .{@alignOf(Aligned)}); // 4

    // ขนาดของ struct อาจใหญ่กว่าที่คิด เพราะ padding
    std.debug.print("Aligned struct size:  {}\n", .{@sizeOf(Aligned)}); // อาจเป็น 12 (ไม่ใช่ 6)
}
```

### @TypeOf - การหาชนิดข้อมูล

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 42;
    const y: f64 = 3.14;
    const z = "hello";

    // @TypeOf ใช้ตอน comptime
    const TypeOfX = @TypeOf(x);
    const TypeOfY = @TypeOf(y);
    const TypeOfZ = @TypeOf(z);

    _ = TypeOfX; // i32
    _ = TypeOfY; // f64
    _ = TypeOfZ; // *const [5:0]u8

    std.debug.print("sizeof @TypeOf(x): {}\n", .{@sizeOf(@TypeOf(x))}); // 4
    std.debug.print("sizeof @TypeOf(y): {}\n", .{@sizeOf(@TypeOf(y))}); // 8
}
```

### @as - การแปลงชนิดข้อมูล (Type Casting)

```zig
const std = @import("std");

pub fn main() void {
    // แปลงจาก i32 เป็น u8
    const big: i32 = 255;
    const small: u8 = @as(u8, @intCast(big));
    std.debug.print("i32({}) as u8: {}\n", .{ big, small }); // 255

    // แปลงจาก u8 เป็น u32 (widening)
    const byte: u8 = 200;
    const word: u32 = @as(u32, byte); // ไม่ต้องใช้ @intCast เพราะเป็น widening
    std.debug.print("u8({}) as u32: {}\n", .{ byte, word }); // 200

    // แปลงจาก float เป็น int
    const pi: f64 = 3.14159;
    const pi_int: i32 = @as(i32, @intFromFloat(pi));
    std.debug.print("f64({d:.5}) as i32: {}\n", .{ pi, pi_int }); // 3

    // แปลงจาก int เป็น float
    const count: i32 = 7;
    const count_f: f64 = @as(f64, @floatFromInt(count));
    std.debug.print("i32({}) as f64: {d:.1}\n", .{ count, count_f }); // 7.0
}
```

### @offsetOf - ตำแหน่งของ field ใน Struct

```zig
const std = @import("std");

const Point3D = struct {
    x: f32,
    y: f32,
    z: f32,
};

const Person = struct {
    name: [32]u8,
    age: u8,
    height: f32,
};

pub fn main() void {
    std.debug.print("Point3D offsets:\n", .{});
    std.debug.print("  x: {} bytes\n", .{@offsetOf(Point3D, "x")}); // 0
    std.debug.print("  y: {} bytes\n", .{@offsetOf(Point3D, "y")}); // 4
    std.debug.print("  z: {} bytes\n", .{@offsetOf(Point3D, "z")}); // 8

    std.debug.print("\nPerson offsets:\n", .{});
    std.debug.print("  name:   {} bytes\n", .{@offsetOf(Person, "name")});   // 0
    std.debug.print("  age:    {} bytes\n", .{@offsetOf(Person, "age")});    // 32
    std.debug.print("  height: {} bytes\n", .{@offsetOf(Person, "height")}); // 36 (aligned)
    std.debug.print("  total size: {} bytes\n", .{@sizeOf(Person)});         // 40
}
```

### การใช้ @ ในการ Generic Programming

```zig
const std = @import("std");

// Generic function ที่ทำงานกับชนิดข้อมูลใด ๆ
fn printTypeInfo(comptime T: type) void {
    std.debug.print("Type: {} | Size: {} bytes | Align: {} bytes | Bits: {}\n", .{
        @typeName(T),
        @sizeOf(T),
        @alignOf(T),
        @bitSizeOf(T),
    });
}

pub fn main() void {
    printTypeInfo(u8);
    printTypeInfo(i32);
    printTypeInfo(f64);
    printTypeInfo(bool);
    printTypeInfo(usize);
}
```

---

## ตารางสรุปตัวดำเนินการทั้งหมด

### ตารางครบถ้วน

| หมวดหมู่ | ตัวดำเนินการ | คำอธิบาย |
|:---|:---:|:---|
| **Arithmetic** | `+`, `-`, `*`, `/`, `%` | คณิตศาสตร์พื้นฐาน |
| **Wrapping** | `+%`, `-%`, `*%` | คณิตศาสตร์แบบ wrap around |
| **Saturating** | `+\|`, `-\|`, `*\|`, `<<\|` | คณิตศาสตร์แบบ saturate |
| **Comparison** | `==`, `!=`, `<`, `>`, `<=`, `>=` | เปรียบเทียบ |
| **Logical** | `and`, `or`, `!` | ตรรกศาสตร์ |
| **Bitwise** | `&`, `\|`, `^`, `~` | ระดับบิต |
| **Shift** | `<<`, `>>` | เลื่อนบิต |
| **Assignment** | `=`, `+=`, `-=`, `*=`, `/=`, `%=` | กำหนดค่า |
| **Bit Assignment** | `&=`, `\|=`, `^=`, `<<=`, `>>=` | bitwise กำหนดค่า |
| **Special** | `@sizeOf`, `@TypeOf`, `@alignOf` | พิเศษ |

---

## การเปรียบเทียบกับ C

### ตัวดำเนินการที่มีใน Zig แต่ไม่มีใน C

| ตัวดำเนินการ Zig | คำอธิบาย | เทียบเท่าใน C |
|:---:|:---|:---|
| `+%`, `-%`, `*%` | Wrapping arithmetic | ไม่มี (ต้องเขียน logic เอง) |
| `+\|`, `-\|`, `*\|` | Saturating arithmetic | ไม่มี (ต้องเขียน logic เอง) |
| `<<\|` | Saturating shift | ไม่มี |
| `orelse` | Optional unwrap | ไม่มี (ใช้ conditional) |
| `catch` | Error unwrap | ไม่มี (ใช้ goto/return) |
| `try` | Error propagation | ไม่มี (ต้องตรวจ return value) |

### ตัวดำเนินการที่มีใน C แต่ไม่มีใน Zig

| ตัวดำเนินการ C | คำอธิบาย | Zig ใช้อะไรแทน |
|:---:|:---|:---|
| `++`, `--` | Increment/Decrement | `+= 1`, `-= 1` |
| `?:` | Ternary | `if/else` expression |
| `,` | Comma operator | ไม่มี (ออกแบบให้ไม่ต้องใช้) |

```zig
// Zig ไม่มี ++ และ -- แต่ใช้ += แทน
var counter: i32 = 0;
counter += 1; // แทน counter++
counter -= 1; // แทน counter--

// Zig ไม่มี ternary operator
// แต่ใช้ if expression แทน
const x: i32 = 10;
const abs_x = if (x >= 0) x else -x; // เหมือน x >= 0 ? x : -x ใน C
```

---

## ข้อควรระวังและ Pitfalls

### 1. Integer Overflow ในโหมด Release

```zig
// ในโหมด Debug: overflow จะ panic
// ในโหมด Release: overflow จะ wrap around (ไม่ panic แต่ผลลัพธ์ผิด!)
// ดังนั้นควรใช้ +% เมื่อต้องการ wrapping และ regular + เมื่อต้องการ safety
```

### 2. การหารด้วย 0

```zig
const std = @import("std");

pub fn main() void {
    const a: i32 = 10;
    const b: i32 = 0;
    // const result = a / b; // PANIC: division by zero!

    // วิธีที่ถูกต้อง: ตรวจสอบก่อน
    if (b != 0) {
        const safe_result = a / b;
        std.debug.print("Result: {}\n", .{safe_result});
    } else {
        std.debug.print("Cannot divide by zero!\n", .{});
    }
}
```

### 3. Integer Type Mismatch

```zig
const std = @import("std");

pub fn main() void {
    const a: i32 = 5;
    const b: i64 = 10;

    // ไม่สามารถบวกต่างชนิดกันตรง ๆ
    // const result = a + b; // ERROR: type mismatch

    // ต้องแปลงชนิดก่อน
    const result = @as(i64, a) + b;
    std.debug.print("Result: {}\n", .{result}); // 15
}
```

### 4. Shift Amount ต้องถูกชนิด

```zig
const std = @import("std");

pub fn main() void {
    const val: u32 = 1;

    // Zig กำหนดให้ shift amount มีชนิดที่ถูกต้อง
    const shift_amount: u5 = 3; // สำหรับ u32, shift amount ต้องเป็น u5 (0-31)
    const shifted = val << shift_amount;
    std.debug.print("{} << {} = {}\n", .{ val, shift_amount, shifted }); // 8

    // ไม่สามารถ shift เกิน bit width
    // const too_much: u5 = 32; // ERROR: ค่า 32 ไม่อยู่ใน range ของ u5 (0-31)
}
```

### 5. Bitwise NOT กับ Signed Integer

```zig
const std = @import("std");

pub fn main() void {
    const a: u8 = 0b1010_1010; // 170
    const not_a = ~a;
    std.debug.print("~0b10101010 (u8) = {b:0>8} = {}\n", .{ not_a, not_a }); // 01010101 = 85

    // กับ signed: Two's complement
    const b: i8 = 10;
    const not_b = ~b;
    std.debug.print("~10 (i8) = {}\n", .{not_b}); // -11 (เท่ากับ -x - 1)
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: เครื่องคิดเลข
เขียนโปรแกรมเครื่องคิดเลขที่รับ operand สองตัวและ operator แล้วคำนวณผล ครอบคลุม `+`, `-`, `*`, `/`, `%`

```zig
const std = @import("std");

fn calculate(a: f64, op: u8, b: f64) !f64 {
    return switch (op) {
        '+' => a + b,
        '-' => a - b,
        '*' => a * b,
        '/' => blk: {
            if (b == 0.0) return error.DivisionByZero;
            break :blk a / b;
        },
        '%' => blk: {
            if (b == 0.0) return error.DivisionByZero;
            break :blk @mod(a, b);
        },
        else => error.UnknownOperator,
    };
}

pub fn main() !void {
    // ทดสอบ
    std.debug.print("10 + 3 = {d}\n", .{try calculate(10, '+', 3)});
    std.debug.print("10 - 3 = {d}\n", .{try calculate(10, '-', 3)});
    std.debug.print("10 * 3 = {d}\n", .{try calculate(10, '*', 3)});
    std.debug.print("10 / 3 = {d:.4}\n", .{try calculate(10, '/', 3)});
    std.debug.print("10 %% 3 = {d}\n", .{try calculate(10, '%', 3)});
}
```

### แบบฝึกหัดที่ 2: Bit Manipulation
เขียนฟังก์ชันสำหรับ set, clear, toggle และ check บิตที่ตำแหน่งที่กำหนด

```zig
const std = @import("std");

fn setBit(value: u32, pos: u5) u32 {
    return value | (@as(u32, 1) << pos);
}

fn clearBit(value: u32, pos: u5) u32 {
    return value & ~(@as(u32, 1) << pos);
}

fn toggleBit(value: u32, pos: u5) u32 {
    return value ^ (@as(u32, 1) << pos);
}

fn checkBit(value: u32, pos: u5) bool {
    return (value >> pos & 1) == 1;
}

pub fn main() void {
    var flags: u32 = 0;
    std.debug.print("Initial: {b:0>8}\n", .{flags});

    flags = setBit(flags, 3);
    std.debug.print("Set bit 3: {b:0>8}\n", .{flags});

    flags = setBit(flags, 7);
    std.debug.print("Set bit 7: {b:0>8}\n", .{flags});

    std.debug.print("Bit 3 is set: {}\n", .{checkBit(flags, 3)});

    flags = clearBit(flags, 3);
    std.debug.print("Clear bit 3: {b:0>8}\n", .{flags});

    std.debug.print("Bit 3 is set: {}\n", .{checkBit(flags, 3)});

    flags = toggleBit(flags, 7);
    std.debug.print("Toggle bit 7: {b:0>8}\n", .{flags});
}
```

### แบบฝึกหัดที่ 3: Saturating Color Mixing
เขียนโปรแกรมผสมสีแบบ RGB โดยใช้ saturating arithmetic

```zig
const std = @import("std");

const Color = struct {
    r: u8,
    g: u8,
    b: u8,
};

fn blendColors(c1: Color, c2: Color, ratio: u8) Color {
    // ratio: 0 = ทั้งหมดเป็น c1, 255 = ทั้งหมดเป็น c2
    const inv_ratio: u16 = 255 - @as(u16, ratio);
    const r = @as(u8, @truncate((@as(u16, c1.r) * inv_ratio + @as(u16, c2.r) * ratio) / 255));
    const g = @as(u8, @truncate((@as(u16, c1.g) * inv_ratio + @as(u16, c2.g) * ratio) / 255));
    const b = @as(u8, @truncate((@as(u16, c1.b) * inv_ratio + @as(u16, c2.b) * ratio) / 255));
    return Color{ .r = r, .g = g, .b = b };
}

fn brighten(color: Color, amount: u8) Color {
    // ใช้ saturating arithmetic เพื่อไม่ให้เกิน 255
    return Color{
        .r = color.r +| amount,
        .g = color.g +| amount,
        .b = color.b +| amount,
    };
}

pub fn main() void {
    const red = Color{ .r = 255, .g = 0, .b = 0 };
    const blue = Color{ .r = 0, .g = 0, .b = 255 };

    const purple = blendColors(red, blue, 128);
    std.debug.print("Purple: rgb({}, {}, {})\n", .{ purple.r, purple.g, purple.b });

    const dark_red = Color{ .r = 100, .g = 0, .b = 0 };
    const bright_red = brighten(dark_red, 200);
    std.debug.print("Bright red: rgb({}, {}, {})\n", .{ bright_red.r, bright_red.g, bright_red.b });
    // r = 100 +| 200 = 255 (saturated), ไม่ overflow!
}
```

### แบบฝึกหัดที่ 4: Type Information
เขียนโปรแกรมที่แสดง type information ของ struct ต่าง ๆ

```zig
const std = @import("std");

fn printStructInfo(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("Struct: {s}\n", .{@typeName(T)});
    std.debug.print("  Size: {} bytes\n", .{@sizeOf(T)});
    std.debug.print("  Alignment: {} bytes\n", .{@alignOf(T)});
    std.debug.print("  Fields: {}\n", .{info.@"struct".fields.len});
    inline for (info.@"struct".fields) |field| {
        std.debug.print("    {s}: {s} (offset: {})\n", .{
            field.name,
            @typeName(field.type),
            @offsetOf(T, field.name),
        });
    }
    std.debug.print("\n", .{});
}

const Vector2 = struct { x: f32, y: f32 };
const Vector3 = struct { x: f32, y: f32, z: f32 };
const RGBA = struct { r: u8, g: u8, b: u8, a: u8 };
const Mixed = struct { id: u32, flag: bool, value: f64 };

pub fn main() void {
    printStructInfo(Vector2);
    printStructInfo(Vector3);
    printStructInfo(RGBA);
    printStructInfo(Mixed);
}
```

---

## สรุป (Summary)

ในส่วนนี้เราได้เรียนรู้ตัวดำเนินการของ Zig ทั้งหมด ซึ่งมีความครบถ้วนและน่าสนใจมากเป็นพิเศษ:

### ประเด็นสำคัญที่ควรจำ

1. **Arithmetic Operators พื้นฐาน** (`+`, `-`, `*`, `/`, `%`) ทำงานเหมือน C แต่ตรวจสอบ overflow ใน Debug mode

2. **Wrapping Arithmetic** (`+%`, `-%`, `*%`) คือจุดเด่นของ Zig ที่ทำให้การ overflow แบบตั้งใจชัดเจนในโค้ด

3. **Saturating Arithmetic** (`+|`, `-|`, `*|`) สำหรับงานที่ต้องการหยุดที่ค่าสูงสุด/ต่ำสุด เช่น การประมวลผลสัญญาณ

4. **Comparison Operators** ให้ผลลัพธ์เป็น `bool` และไม่อนุญาตให้เปรียบเทียบต่างชนิดโดยตรง

5. **Logical Operators** (`and`, `or`, `!`) รองรับ short-circuit evaluation

6. **Bitwise Operators** สำหรับการจัดการบิตระดับต่ำ มีประโยชน์มากสำหรับ flags และ optimization

7. **Assignment Operators** มีครบทั้ง compound forms รวมถึง wrapping และ saturating variants

8. **Operator Precedence** ใช้วงเล็บเสมอเมื่อไม่แน่ใจเพื่อความชัดเจน

9. **Special `@` Operators** ให้ข้อมูล type ณ compile time มีประโยชน์มากสำหรับ generic programming

### เปรียบเทียบกับ C

Zig ได้รับแรงบันดาลใจจาก C แต่ปรับปรุงหลายด้าน:
- ไม่มี `++` และ `--` (ใช้ `+= 1` แทน เพื่อความชัดเจน)
- ไม่มี ternary `?:` (ใช้ `if` expression แทน)
- เพิ่ม wrapping และ saturating arithmetic
- ตรวจสอบ overflow ใน Debug mode
- ไม่มี implicit type conversion

### ขั้นตอนถัดไป

ในส่วนถัดไปเราจะเรียนรู้เกี่ยวกับ Control Flow (การควบคุมการทำงาน) ซึ่งจะใช้ตัวดำเนินการที่เรียนรู้ในส่วนนี้เป็นพื้นฐานสำหรับการตัดสินใจและวนซ้ำในโปรแกรม

---

*ส่วนที่ 05 จาก N | Steps 41-50 | ตัวดำเนินการใน Zig*
