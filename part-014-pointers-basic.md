# Part 14: Pointers พื้นฐานใน Zig (Steps 131-140)

> ซีรีส์: เรียน Zig ทีละขั้นตอน  
> ระดับ: ปานกลาง  
> ก่อนหน้า: Part 13 - Slices และ Arrays  
> ถัดไป: Part 15 - Slices ขั้นสูง

---

## บทนำ

Pointers (ตัวชี้) เป็นหนึ่งในแนวคิดที่สำคัญที่สุดในการเขียนโปรแกรมระดับต่ำ และ Zig ออกแบบมาเพื่อให้ทำงานกับ pointers ได้อย่างปลอดภัยและชัดเจน ในบทนี้เราจะเรียนรู้ตั้งแต่พื้นฐานจนถึงการใช้งานขั้นสูง พร้อมเข้าใจความแตกต่างระหว่าง Zig และภาษา C ในการจัดการ pointers

---

## Step 131: Pointers คืออะไร? แนวคิดและหน่วยความจำ

### หน่วยความจำของโปรแกรม

ก่อนที่จะเข้าใจ pointer เราต้องเข้าใจว่าโปรแกรมจัดการหน่วยความจำอย่างไร

เมื่อโปรแกรมทำงาน ระบบปฏิบัติการจะจัดสรรหน่วยความจำให้ โดยหน่วยความจำนี้คือพื้นที่สำหรับเก็บข้อมูลต่าง ๆ

```
หน่วยความจำ (RAM) ของโปรแกรม:
┌─────────────────────────────────────┐
│  Address  │  ค่าที่เก็บอยู่          │
├─────────────────────────────────────┤
│  0x1000   │  42  (ตัวแปร x)         │
│  0x1004   │  100 (ตัวแปร y)         │
│  0x1008   │  0x1000 (pointer p)     │
│  0x100C   │  ...                    │
│  0x1010   │  ...                    │
└─────────────────────────────────────┘
```

- **Address (ที่อยู่)**: ตำแหน่งในหน่วยความจำ เปรียบเหมือนบ้านเลขที่
- **Value (ค่า)**: ข้อมูลที่เก็บในตำแหน่งนั้น
- **Pointer**: ตัวแปรที่เก็บ "ที่อยู่" ของตัวแปรอื่น

### แนวคิดพื้นฐาน

```
ตัวแปรธรรมดา:                    Pointer:
┌──────────┐                    ┌──────────┐
│  x = 42  │                    │  p = ──────────────────► ┌──────────┐
│ @0x1000  │                    │ @0x1008  │               │  x = 42  │
└──────────┘                    └──────────┘               │ @0x1000  │
                                                           └──────────┘
```

Pointer `p` เก็บค่า `0x1000` ซึ่งเป็น address ของตัวแปร `x`

### ทำไมต้องใช้ Pointer?

1. **ประหยัดหน่วยความจำ**: แทนที่จะคัดลอกข้อมูลทั้งก้อน ส่งแค่ที่อยู่
2. **แก้ไขข้อมูลต้นทาง**: ฟังก์ชันสามารถแก้ไขค่าของตัวแปรภายนอกได้
3. **โครงสร้างข้อมูลแบบ Dynamic**: Linked lists, trees ฯลฯ
4. **Interface กับ C libraries**: ภาษา C ใช้ pointers อย่างแพร่หลาย

### ตัวอย่างแรก: เหตุใด Pointer จึงสำคัญ

```zig
const std = @import("std");

// ฟังก์ชันที่ไม่ใช้ pointer - ไม่สามารถแก้ค่าต้นทางได้
fn doubleValue(x: i32) i32 {
    return x * 2;
}

// ฟังก์ชันที่ใช้ pointer - แก้ค่าต้นทางได้โดยตรง
fn doubleValueInPlace(x: *i32) void {
    x.* = x.* * 2;
}

pub fn main() void {
    var number: i32 = 5;
    
    // วิธีที่ 1: ไม่ใช้ pointer
    const result = doubleValue(number);
    std.debug.print("result = {d}, number ยังคงเป็น {d}\n", .{result, number});
    // ผลลัพธ์: result = 10, number ยังคงเป็น 5
    
    // วิธีที่ 2: ใช้ pointer
    doubleValueInPlace(&number);
    std.debug.print("number ตอนนี้ = {d}\n", .{number});
    // ผลลัพธ์: number ตอนนี้ = 10
}
```

### ความแตกต่างระหว่าง Zig และ C

| คุณสมบัติ | C | Zig |
|-----------|---|-----|
| Null pointer | อาจเกิดโดยไม่ได้ตั้งใจ | ต้องใช้ `?*T` อย่างชัดเจน |
| Pointer arithmetic | ทำได้เสมอ | ต้องใช้ `[*]T` อย่างชัดเจน |
| Dangling pointer | ตรวจจับได้ยาก | Compiler ช่วยป้องกันบางกรณี |
| Type safety | อ่อนแอ | เข้มงวดกว่า |

---

## Step 132: Single-item Pointer *T

### ไวยากรณ์พื้นฐาน

Single-item pointer ใน Zig เขียนว่า `*T` โดยที่ `T` คือชนิดข้อมูลที่ pointer ชี้ไป

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศตัวแปรธรรมดา
    var x: i32 = 42;
    
    // ประกาศ pointer ที่ชี้ไปยัง x
    // &x คือ "address ของ x"
    const ptr: *i32 = &x;
    
    std.debug.print("ค่าของ x = {d}\n", .{x});
    std.debug.print("address ของ x = {*}\n", .{ptr});
    std.debug.print("ค่าที่ ptr ชี้ไป = {d}\n", .{ptr.*});
}
```

### ชนิดข้อมูลของ Pointer

```zig
const std = @import("std");

pub fn main() void {
    var a: i32 = 10;
    var b: f64 = 3.14;
    var c: bool = true;
    
    // pointer ชนิดต่างๆ
    const ptr_a: *i32  = &a;  // pointer ชี้ไป i32
    const ptr_b: *f64  = &b;  // pointer ชี้ไป f64
    const ptr_c: *bool = &c;  // pointer ชี้ไป bool
    
    // แสดงค่าผ่าน pointer
    std.debug.print("a = {d}\n", .{ptr_a.*});
    std.debug.print("b = {d}\n", .{ptr_b.*});
    std.debug.print("c = {}\n", .{ptr_c.*});
    
    // แก้ค่าผ่าน pointer
    ptr_a.* = 99;
    std.debug.print("a หลังแก้ไข = {d}\n", .{a});
}
```

### Diagram: Single-item Pointer

```
ก่อนสร้าง pointer:
Stack Memory:
┌─────────────────────────────┐
│  Address   │  Variable      │
├─────────────────────────────┤
│  0x7fff0000│  x = 42 (i32)  │
└─────────────────────────────┘

หลังสร้าง pointer:
Stack Memory:
┌─────────────────────────────────────────┐
│  Address   │  Variable                  │
├─────────────────────────────────────────┤
│  0x7fff0000│  x = 42 (i32)              │
│  0x7fff0008│  ptr = 0x7fff0000 (*i32)   │
└─────────────────────────────────────────┘

ptr ──────────────────────────────► x (42)
(เก็บ 0x7fff0000)                   (อยู่ที่ 0x7fff0000)
```

### Pointer กับ Struct

```zig
const std = @import("std");

const Point = struct {
    x: f32,
    y: f32,
};

pub fn main() void {
    var p = Point{ .x = 3.0, .y = 4.0 };
    
    // สร้าง pointer ชี้ไป struct
    const ptr: *Point = &p;
    
    // เข้าถึง field ผ่าน pointer
    std.debug.print("x = {d}, y = {d}\n", .{ptr.x, ptr.y});
    // สังเกต: ใน Zig เขียน ptr.x แทน ptr->x ใน C
    
    // แก้ค่า field ผ่าน pointer
    ptr.x = 10.0;
    std.debug.print("x หลังแก้ไข = {d}\n", .{p.x});
}
```

**หมายเหตุสำคัญ**: ใน Zig เราใช้ `ptr.field` ไม่ใช่ `ptr->field` แบบใน C Zig จะ auto-dereference ให้อัตโนมัติ

### ขนาดของ Pointer

```zig
const std = @import("std");

pub fn main() void {
    // ทุก pointer มีขนาดเท่ากัน (ขึ้นอยู่กับสถาปัตยกรรม)
    // บน 64-bit: 8 bytes
    // บน 32-bit: 4 bytes
    
    std.debug.print("ขนาดของ *i32  = {d} bytes\n", .{@sizeOf(*i32)});
    std.debug.print("ขนาดของ *i64  = {d} bytes\n", .{@sizeOf(*i64)});
    std.debug.print("ขนาดของ *bool = {d} bytes\n", .{@sizeOf(*bool)});
    // ผลลัพธ์ทั้งหมดเหมือนกัน (8 bytes บน 64-bit)
}
```

---

## Step 133: & address-of Operator และ * Dereference

### & Address-of Operator

`&` ใช้เพื่อ "รับ address" ของตัวแปร

```zig
const std = @import("std");

pub fn main() void {
    var number: i32 = 100;
    
    // & ดึง address ของ number
    const addr: *i32 = &number;
    
    std.debug.print("ค่าของ number  = {d}\n", .{number});
    std.debug.print("address ของ number = {*}\n", .{addr});
    // ผลลัพธ์: address ของ number = <some_hex_address>
}
```

### * Dereference Operator

`.*` ใช้เพื่อ "อ่านค่า" จาก address ที่ pointer ชี้ไป

```zig
const std = @import("std");

pub fn main() void {
    var number: i32 = 100;
    const ptr = &number;
    
    // อ่านค่าผ่าน pointer (dereference)
    const value: i32 = ptr.*;
    std.debug.print("ค่าที่ ptr ชี้ไป = {d}\n", .{value});
    
    // เขียนค่าผ่าน pointer (dereference แล้วกำหนดค่า)
    ptr.* = 200;
    std.debug.print("number หลังแก้ไข = {d}\n", .{number});
}
```

### ลำดับการทำงาน

```
ตัวอย่าง: ptr.* = ptr.* * 2

1. ptr.*          → อ่านค่าจาก address ที่ ptr ชี้ไป  (เช่น ได้ 50)
2. ptr.* * 2      → คำนวณ 50 * 2 = 100
3. ptr.* = 100    → เขียน 100 ลงที่ address ที่ ptr ชี้ไป
```

### ตัวอย่างการใช้งานจริง

```zig
const std = @import("std");

// สลับค่าสองตัวแปร
fn swap(a: *i32, b: *i32) void {
    const temp = a.*;
    a.* = b.*;
    b.* = temp;
}

// บวกค่าเข้าไปในตัวแปรต้นทาง
fn addTo(target: *i32, amount: i32) void {
    target.* += amount;
}

// เพิ่มค่าทีละ 1
fn increment(value: *i32) void {
    value.* += 1;
}

pub fn main() void {
    var x: i32 = 10;
    var y: i32 = 20;
    
    std.debug.print("ก่อน swap: x={d}, y={d}\n", .{x, y});
    swap(&x, &y);
    std.debug.print("หลัง swap: x={d}, y={d}\n", .{x, y});
    
    addTo(&x, 5);
    std.debug.print("x หลัง addTo 5 = {d}\n", .{x});
    
    increment(&y);
    std.debug.print("y หลัง increment = {d}\n", .{y});
}
```

### ความสัมพันธ์ระหว่าง & และ *

```
& และ .* เป็น inverse ของกัน:

var x: i32 = 42;
const ptr = &x;    // ptr ชี้ไปที่ x
const val = ptr.*; // val = 42 (ค่าของ x)

// (&x).* = x  เสมอ
// &(x.*) = x (เมื่อ x เป็น pointer)
```

### Chaining Pointers (Pointer ซ้อน Pointer)

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 42;
    var ptr: *i32 = &x;
    var ptr_to_ptr: **i32 = &ptr;
    
    // เข้าถึงค่าผ่าน pointer ซ้อน
    std.debug.print("x = {d}\n", .{x});
    std.debug.print("ptr.* = {d}\n", .{ptr.*});
    std.debug.print("ptr_to_ptr.*.* = {d}\n", .{ptr_to_ptr.*.*});
    
    // แก้ค่าผ่าน double pointer
    ptr_to_ptr.*.* = 99;
    std.debug.print("x หลังแก้ผ่าน double pointer = {d}\n", .{x});
}
```

### ข้อผิดพลาดที่ควรระวัง

```zig
// ❌ ผิด: ไม่สามารถ dereference pointer ชั่วคราว (dangling reference)
fn badFunction() *i32 {
    var local: i32 = 42;
    return &local; // ERROR: local จะถูกทำลายเมื่อฟังก์ชันสิ้นสุด
}

// ✅ ถูก: ส่ง pointer มาจากภายนอก หรือใช้ allocator
fn goodFunction(value: *i32) void {
    value.* = 42;
}
```

---

## Step 134: const Pointer vs Pointer-to-const

### ความแตกต่างสำคัญ

ใน Zig มี pointer 4 แบบที่ต่างกัน:

| ประเภท | Syntax | เปลี่ยน pointer ได้? | เปลี่ยนค่าที่ชี้ได้? |
|--------|--------|----------------------|----------------------|
| Mutable pointer to mutable data | `*T` | ✅ (ถ้า pointer เป็น var) | ✅ |
| Const pointer to mutable data | `*T` (const ptr) | ❌ | ✅ |
| Mutable pointer to const data | `*const T` | ✅ (ถ้า pointer เป็น var) | ❌ |
| Const pointer to const data | `*const T` (const ptr) | ❌ | ❌ |

### ตัวอย่างแสดงความแตกต่าง

```zig
const std = @import("std");

pub fn main() void {
    var mutable_data: i32 = 42;
    var another_data: i32 = 100;
    
    // ─────────────────────────────────────────────────────
    // กรณีที่ 1: var pointer ชี้ไป mutable data
    // สามารถเปลี่ยนทั้ง pointer และค่า
    // ─────────────────────────────────────────────────────
    var ptr1: *i32 = &mutable_data;
    ptr1.* = 50;           // ✅ เปลี่ยนค่าได้
    ptr1 = &another_data;  // ✅ เปลี่ยน pointer ได้
    std.debug.print("ptr1.* = {d}\n", .{ptr1.*});
    
    // ─────────────────────────────────────────────────────
    // กรณีที่ 2: const pointer ชี้ไป mutable data
    // ไม่สามารถเปลี่ยน pointer แต่เปลี่ยนค่าได้
    // ─────────────────────────────────────────────────────
    const ptr2: *i32 = &mutable_data;
    ptr2.* = 99;           // ✅ เปลี่ยนค่าได้
    // ptr2 = &another_data; // ❌ ERROR: ไม่สามารถกำหนดค่าให้ const
    std.debug.print("ptr2.* = {d}\n", .{ptr2.*});
    
    // ─────────────────────────────────────────────────────
    // กรณีที่ 3: pointer ชี้ไป const data (*const T)
    // สามารถเปลี่ยน pointer แต่ไม่สามารถเปลี่ยนค่า
    // ─────────────────────────────────────────────────────
    var ptr3: *const i32 = &mutable_data;
    // ptr3.* = 77;          // ❌ ERROR: ไม่สามารถเปลี่ยนค่าผ่าน *const
    ptr3 = &another_data;   // ✅ เปลี่ยน pointer ได้
    std.debug.print("ptr3.* = {d}\n", .{ptr3.*});
    
    // ─────────────────────────────────────────────────────
    // กรณีที่ 4: const pointer ชี้ไป const data
    // ไม่สามารถเปลี่ยนทั้ง pointer และค่า
    // ─────────────────────────────────────────────────────
    const ptr4: *const i32 = &mutable_data;
    // ptr4.* = 55;           // ❌ ERROR
    // ptr4 = &another_data;  // ❌ ERROR
    std.debug.print("ptr4.* = {d}\n", .{ptr4.*});
}
```

### Diagram: const pointer vs pointer-to-const

```
var ptr: *i32 = &data          (ทั้ง pointer และ data เปลี่ยนได้)
┌─────────────┐                ┌─────────────┐
│  ptr        │────────────►   │  data = 42  │
│  (var)      │◄─ เปลี่ยนได้   │  (var)      │◄─ เปลี่ยนได้
└─────────────┘                └─────────────┘

const ptr: *i32 = &data        (pointer คงที่ แต่ data เปลี่ยนได้)
┌─────────────┐                ┌─────────────┐
│  ptr        │────────────►   │  data = 42  │
│  (const)    │  เปลี่ยนไม่ได้  │  (var)      │◄─ เปลี่ยนได้
└─────────────┘                └─────────────┘

var ptr: *const i32 = &data    (pointer เปลี่ยนได้ แต่ data เปลี่ยนไม่ได้)
┌─────────────┐                ┌─────────────┐
│  ptr        │────────────►   │  data = 42  │
│  (var)      │◄─ เปลี่ยนได้   │  (const)    │  เปลี่ยนไม่ได้
└─────────────┘                └─────────────┘
```

### การใช้งาน *const T ในฟังก์ชัน

```zig
const std = @import("std");

const Point = struct {
    x: f32,
    y: f32,
};

// รับ pointer-to-const เพราะเราแค่อ่านค่า ไม่แก้ไข
fn printPoint(p: *const Point) void {
    std.debug.print("Point({d}, {d})\n", .{p.x, p.y});
}

// ต้องการ *Point เพราะจะแก้ไขค่า
fn movePoint(p: *Point, dx: f32, dy: f32) void {
    p.x += dx;
    p.y += dy;
}

// ✅ Best Practice: ถ้าไม่ต้องแก้ไข ใช้ *const T เพื่อความชัดเจน
fn distanceFromOrigin(p: *const Point) f32 {
    return @sqrt(p.x * p.x + p.y * p.y);
}

pub fn main() void {
    var pt = Point{ .x = 3.0, .y = 4.0 };
    
    printPoint(&pt);
    
    const dist = distanceFromOrigin(&pt);
    std.debug.print("ระยะจากจุดกำเนิด = {d}\n", .{dist});
    
    movePoint(&pt, 1.0, 2.0);
    printPoint(&pt);
}
```

### Const ข้อมูลใน Global Scope

```zig
const std = @import("std");

// const ระดับ global สร้าง pointer-to-const โดยอัตโนมัติ
const GLOBAL_CONST: i32 = 100;

pub fn main() void {
    // &GLOBAL_CONST ได้ชนิด *const i32 เสมอ
    const ptr: *const i32 = &GLOBAL_CONST;
    std.debug.print("GLOBAL_CONST = {d}\n", .{ptr.*});
    // ptr.* = 200; // ❌ ERROR: ไม่สามารถแก้ค่า const ได้
}
```

---

## Step 135: Pointers กับ Structs (self: *Self)

### Pattern: self: *Self

นี่เป็น pattern ที่ใช้บ่อยมากใน Zig เพื่อให้ struct method สามารถแก้ไขตัวมันเองได้

```zig
const std = @import("std");

const Counter = struct {
    count: i32,
    max: i32,
    
    // Self คือ alias ของ Counter ใน context นี้
    const Self = @This();
    
    // Method ที่แก้ไข state → ต้องใช้ self: *Self
    pub fn increment(self: *Self) void {
        if (self.count < self.max) {
            self.count += 1;
        }
    }
    
    // Method ที่แค่อ่าน → ใช้ self: Self หรือ self: *const Self
    pub fn getValue(self: Self) i32 {
        return self.count;
    }
    
    // ใช้ *const Self เมื่อ struct ใหญ่ (ไม่ต้อง copy)
    pub fn isAtMax(self: *const Self) bool {
        return self.count >= self.max;
    }
    
    pub fn reset(self: *Self) void {
        self.count = 0;
    }
};

pub fn main() void {
    var counter = Counter{ .count = 0, .max = 5 };
    
    std.debug.print("เริ่มต้น count = {d}\n", .{counter.getValue()});
    
    counter.increment();
    counter.increment();
    counter.increment();
    
    std.debug.print("หลัง increment 3 ครั้ง = {d}\n", .{counter.getValue()});
    std.debug.print("ถึง max แล้ว? {}\n", .{counter.isAtMax()});
    
    counter.increment();
    counter.increment();
    counter.increment(); // ครั้งนี้จะไม่นับเพราะถึง max แล้ว
    
    std.debug.print("หลัง increment อีก 3 ครั้ง = {d}\n", .{counter.getValue()});
    std.debug.print("ถึง max แล้ว? {}\n", .{counter.isAtMax()});
    
    counter.reset();
    std.debug.print("หลัง reset = {d}\n", .{counter.getValue()});
}
```

### Diagram: self pointer

```
stack:
┌─────────────────────────────────────┐
│  counter (Counter struct)           │
│  ┌──────────────────────────────┐   │
│  │  count: 3                    │   │
│  │  max:   5                    │   │
│  └──────────────────────────────┘   │
│  address: 0x7fff1000                │
└─────────────────────────────────────┘

เมื่อเรียก counter.increment():
  ↓ Zig แปลงเป็น Counter.increment(&counter)
  ↓ self = pointer ที่ชี้ไปที่ counter
  ↓ self.count += 1 → แก้ไข counter.count โดยตรง
```

### ตัวอย่าง: Stack Data Structure

```zig
const std = @import("std");

const MAX_SIZE = 10;

const IntStack = struct {
    items: [MAX_SIZE]i32,
    top: usize,
    
    const Self = @This();
    
    pub fn init() Self {
        return Self{
            .items = undefined,
            .top = 0,
        };
    }
    
    pub fn push(self: *Self, value: i32) !void {
        if (self.top >= MAX_SIZE) {
            return error.StackOverflow;
        }
        self.items[self.top] = value;
        self.top += 1;
    }
    
    pub fn pop(self: *Self) !i32 {
        if (self.top == 0) {
            return error.StackUnderflow;
        }
        self.top -= 1;
        return self.items[self.top];
    }
    
    pub fn peek(self: *const Self) !i32 {
        if (self.top == 0) {
            return error.EmptyStack;
        }
        return self.items[self.top - 1];
    }
    
    pub fn isEmpty(self: *const Self) bool {
        return self.top == 0;
    }
    
    pub fn size(self: *const Self) usize {
        return self.top;
    }
};

pub fn main() !void {
    var stack = IntStack.init();
    
    try stack.push(10);
    try stack.push(20);
    try stack.push(30);
    
    std.debug.print("Stack size = {d}\n", .{stack.size()});
    std.debug.print("Top = {d}\n", .{try stack.peek()});
    
    while (!stack.isEmpty()) {
        const val = try stack.pop();
        std.debug.print("popped: {d}\n", .{val});
    }
}
```

### self: Self vs self: *Self vs self: *const Self

```zig
const Thing = struct {
    value: i32,
    
    const Self = @This();
    
    // self: Self → คัดลอก struct มาใช้ (ใช้กับ struct เล็กๆ ที่ไม่ต้องแก้ค่า)
    pub fn getValueCopy(self: Self) i32 {
        return self.value;
    }
    
    // self: *const Self → ส่ง pointer มาอ่าน (ดีกว่าสำหรับ struct ใหญ่)
    pub fn getValueRef(self: *const Self) i32 {
        return self.value;
    }
    
    // self: *Self → ส่ง pointer มาแก้ไข (จำเป็นต้องใช้เมื่อต้องการแก้ค่า)
    pub fn setValue(self: *Self, new_value: i32) void {
        self.value = new_value;
    }
};
```

---

## Step 136: Pointer Arithmetic

### หลักการ Pointer Arithmetic

Pointer arithmetic คือการบวก/ลบตัวเลขเข้ากับ pointer เพื่อเลื่อนไปยัง memory location อื่น

**ข้อสำคัญ**: ใน Zig ไม่สามารถทำ pointer arithmetic กับ `*T` (single-item pointer) ต้องใช้ `[*]T` (many-item pointer) แทน

### การทำงานของ Pointer Arithmetic

```
Memory layout สำหรับ array ของ i32:
┌─────────┬─────────┬─────────┬─────────┐
│  10     │  20     │  30     │  40     │
├─────────┼─────────┼─────────┼─────────┤
│  0x100  │  0x104  │  0x108  │  0x10C  │
└─────────┴─────────┴─────────┴─────────┘

ptr ชี้ที่ 0x100:
ptr + 0 → 0x100 (ค่า 10)
ptr + 1 → 0x104 (ค่า 20)  ← +1 = +4 bytes (ขนาดของ i32)
ptr + 2 → 0x108 (ค่า 30)
ptr + 3 → 0x10C (ค่า 40)
```

### ตัวอย่าง Pointer Arithmetic กับ Array

```zig
const std = @import("std");

pub fn main() void {
    var numbers = [_]i32{ 10, 20, 30, 40, 50 };
    
    // ใช้ [*]i32 สำหรับ pointer arithmetic
    const ptr: [*]i32 = &numbers;
    
    // เข้าถึงด้วย index
    std.debug.print("ptr[0] = {d}\n", .{ptr[0]});
    std.debug.print("ptr[1] = {d}\n", .{ptr[1]});
    std.debug.print("ptr[2] = {d}\n", .{ptr[2]});
    
    // เลื่อน pointer
    const ptr2 = ptr + 2;  // ชี้ไปที่ element ที่ 2
    std.debug.print("(ptr+2)[0] = {d}\n", .{ptr2[0]});  // 30
    std.debug.print("(ptr+2)[1] = {d}\n", .{ptr2[1]});  // 40
}
```

### แปลง [*]T เป็น Slice

```zig
const std = @import("std");

pub fn main() void {
    var data = [_]f32{ 1.0, 2.0, 3.0, 4.0, 5.0 };
    const ptr: [*]f32 = &data;
    
    // แปลง many-item pointer เป็น slice ด้วย [start..end]
    const slice: []f32 = ptr[0..5];
    
    for (slice, 0..) |val, i| {
        std.debug.print("slice[{d}] = {d}\n", .{i, val});
    }
    
    // slice ย่อยๆ
    const middle: []f32 = ptr[1..4];
    std.debug.print("middle slice:\n", .{});
    for (middle) |val| {
        std.debug.print("  {d}\n", .{val});
    }
}
```

### ตัวอย่าง: เขียน memcpy เอง

```zig
const std = @import("std");

// คัดลอกข้อมูลแบบ byte-by-byte
fn myMemcpy(dst: [*]u8, src: [*]const u8, len: usize) void {
    var i: usize = 0;
    while (i < len) : (i += 1) {
        dst[i] = src[i];
    }
}

pub fn main() void {
    const source = "Hello, Zig!";
    var dest: [20]u8 = undefined;
    
    myMemcpy(&dest, source.ptr, source.len);
    dest[source.len] = 0; // null terminator
    
    std.debug.print("copied: {s}\n", .{dest[0..source.len]});
}
```

### ความปลอดภัยของ Pointer Arithmetic ใน Zig vs C

```zig
// ใน C ไม่มีการตรวจสอบ bounds:
// int arr[5] = {1,2,3,4,5};
// int* ptr = arr;
// *(ptr + 10) = 99;  // ⚠️ Undefined behavior! ไม่มี error

// ใน Zig ด้วย [*]T ก็ยังต้องระวัง:
// ต้องรู้ขอบเขตเองและแปลงเป็น slice ก่อนใช้

// ✅ วิธีปลอดภัยกว่า: ใช้ slice แทน many-item pointer
fn sumArray(arr: []const i32) i32 {
    var total: i32 = 0;
    for (arr) |val| {
        total += val;
    }
    return total;
}
```

### @ptrCast กับ Alignment

```zig
const std = @import("std");

pub fn main() void {
    var bytes = [_]u8{ 0x01, 0x00, 0x00, 0x00 }; // little-endian: i32 = 1
    
    // แปลง *[4]u8 เป็น *i32 (ต้องระวัง alignment!)
    // ใช้ @alignCast เพื่อบอก compiler ว่า alignment ถูกต้อง
    const int_ptr: *i32 = @alignCast(@ptrCast(&bytes));
    std.debug.print("ค่าเป็น i32: {d}\n", .{int_ptr.*});
}
```

---

## Step 137: Many-item Pointer [*]T

### ความแตกต่างระหว่าง *T และ [*]T

| | `*T` | `[*]T` |
|--|------|--------|
| ชี้ไปที่ | 1 element | หลาย elements |
| pointer arithmetic | ❌ | ✅ |
| indexing | ❌ | ✅ |
| รู้ขนาด | รู้แน่นอน (1) | ไม่รู้ |
| ใช้งาน | ชี้ไป struct/variable | array, C strings |

### การสร้างและใช้ [*]T

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 10, 20, 30, 40, 50 };
    
    // [*]i32 จาก array
    const many_ptr: [*]i32 = &arr;
    
    // indexing ทำงานได้
    std.debug.print("many_ptr[0] = {d}\n", .{many_ptr[0]});
    std.debug.print("many_ptr[3] = {d}\n", .{many_ptr[3]});
    
    // pointer arithmetic
    const ptr2 = many_ptr + 1;
    std.debug.print("(ptr+1)[0] = {d}\n", .{ptr2[0]});  // 20
    
    // แก้ค่าผ่าน many-item pointer
    many_ptr[2] = 999;
    std.debug.print("arr[2] = {d}\n", .{arr[2]});  // 999
}
```

### Diagram: [*]T

```
var arr = [_]i32{10, 20, 30, 40, 50};
const ptr: [*]i32 = &arr;

Memory:
ptr ─────────────────────────────────────────────────────────►
      ┌───────┬───────┬───────┬───────┬───────┐
      │  10   │  20   │  30   │  40   │  50   │
      ├───────┼───────┼───────┼───────┼───────┤
      │ [0]   │ [1]   │ [2]   │ [3]   │ [4]   │
      └───────┴───────┴───────┴───────┴───────┘
ptr+0 ↑        ↑        ↑        ↑        ↑
              ptr+1    ptr+2    ptr+3    ptr+4

[*]T ไม่รู้ว่า array มีกี่ element!
ถ้าเข้าถึง ptr[5] ออกนอก bounds → ⚠️ undefined behavior
```

### [*]T กับ C Interop

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("string.h");
});

// C function ที่รับ char* จะถูก import เป็น [*c]u8 ใน Zig
// [*c]u8 คือ optional many-item pointer สำหรับ C compatibility

pub fn main() void {
    // ตัวอย่างการทำงานกับ C-style string
    var buffer: [100]u8 = undefined;
    const message = "Hello from Zig!";
    
    // คัดลอกข้อความ
    @memcpy(buffer[0..message.len], message);
    buffer[message.len] = 0;
    
    // ส่งให้ C function ผ่าน many-item pointer
    const len = c.strlen(&buffer);
    std.debug.print("ความยาว = {d}\n", .{len});
}
```

### แปลงระหว่าง *T, [*]T, และ []T

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    
    // *[5]i32 → pointer ไปยัง array ที่รู้ขนาด
    const array_ptr: *[5]i32 = &arr;
    
    // *[5]i32 → []i32 (slice)
    const slice: []i32 = array_ptr;
    
    // *[5]i32 → [*]i32 (many-item pointer)
    const many: [*]i32 = array_ptr;
    
    std.debug.print("array_ptr[2] = {d}\n", .{array_ptr[2]});
    std.debug.print("slice[2]     = {d}\n", .{slice[2]});
    std.debug.print("many[2]      = {d}\n", .{many[2]});
    
    // []i32 → [*]i32
    const many2: [*]i32 = slice.ptr;
    std.debug.print("many2[2]     = {d}\n", .{many2[2]});
}
```

### Sentinel-terminated Pointer [*:0]T

Pointer ชนิดพิเศษที่มี sentinel value (มักใช้กับ C strings)

```zig
const std = @import("std");

pub fn main() void {
    // [*:0]const u8 คือ null-terminated string (C string)
    const c_string: [*:0]const u8 = "Hello, World!";
    
    // นับความยาวด้วย sentinel
    var len: usize = 0;
    while (c_string[len] != 0) {
        len += 1;
    }
    std.debug.print("ความยาว = {d}\n", .{len});
    
    // Zig string literal คือ *const [N:0]u8
    // แปลงเป็น slice ได้โดยตรง
    const zig_str = "สวัสดี Zig";
    std.debug.print("{s}\n", .{zig_str});
}
```

---

## Step 138: Optional Pointer ?*T

### ปัญหา: Null Pointer ใน C

ใน C `NULL` pointer เป็นสาเหตุหลักของ crashes:

```c
// C code ที่อันตราย:
int* findValue(int* arr, int size, int target) {
    for (int i = 0; i < size; i++) {
        if (arr[i] == target) return &arr[i];
    }
    return NULL;  // ถ้าไม่เจอ
}

// อาจลืม check null:
int* result = findValue(arr, 5, 99);
*result = 100;  // ⚠️ Segfault ถ้า result เป็น NULL!
```

### วิธีแก้ใน Zig: ?*T

```zig
const std = @import("std");

// ค้นหาค่าใน array คืน optional pointer
fn findValue(arr: []i32, target: i32) ?*i32 {
    for (arr) |*val| {
        if (val.* == target) return val;
    }
    return null;
}

pub fn main() void {
    var numbers = [_]i32{ 10, 20, 30, 40, 50 };
    
    // ค้นหาค่าที่มีอยู่
    if (findValue(&numbers, 30)) |ptr| {
        std.debug.print("เจอค่า 30 ที่ address {*}\n", .{ptr});
        ptr.* = 999;  // แก้ค่าได้
        std.debug.print("แก้เป็น: {d}\n", .{numbers[2]});
    } else {
        std.debug.print("ไม่เจอค่า 30\n", .{});
    }
    
    // ค้นหาค่าที่ไม่มี
    if (findValue(&numbers, 99)) |ptr| {
        std.debug.print("เจอ: {d}\n", .{ptr.*});
    } else {
        std.debug.print("ไม่เจอค่า 99\n", .{});
    }
}
```

### Diagram: Optional Pointer

```
?*i32 มีสองสถานะ:

สถานะที่ 1: มีค่า (non-null)
┌──────────────────────────────────┐
│  ?*i32: some(ptr)                │
│  ┌──────────────────┐            │
│  │  ptr = 0x1000    │────────► ┌─────┐
│  └──────────────────┘          │  42 │
└──────────────────────────────── └─────┘

สถานะที่ 2: ไม่มีค่า (null)
┌──────────────────────────────────┐
│  ?*i32: null                     │
│  ┌──────────────────┐            │
│  │  (empty)         │            │
│  └──────────────────┘            │
└──────────────────────────────────┘
```

### วิธี unwrap Optional Pointer

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 42;
    var opt_ptr: ?*i32 = &x;
    
    // วิธีที่ 1: if-let pattern (แนะนำ)
    if (opt_ptr) |ptr| {
        std.debug.print("ค่า: {d}\n", .{ptr.*});
    }
    
    // วิธีที่ 2: orelse - ใช้ default value
    const ptr = opt_ptr orelse {
        std.debug.print("ไม่มี pointer\n", .{});
        return;
    };
    std.debug.print("ค่าผ่าน orelse: {d}\n", .{ptr.*});
    
    // วิธีที่ 3: .? - unwrap แบบ force (อาจ panic ถ้าเป็น null)
    const forced = opt_ptr.?;
    std.debug.print("force unwrap: {d}\n", .{forced.*});
    
    // ตั้งเป็น null
    opt_ptr = null;
    
    // วิธีที่ 4: ตรวจสอบ null ก่อนใช้
    if (opt_ptr != null) {
        std.debug.print("ไม่ใช่ null\n", .{});
    } else {
        std.debug.print("เป็น null\n", .{});
    }
}
```

### ตัวอย่าง: Linked List ด้วย Optional Pointer

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const Node = struct {
    value: i32,
    next: ?*Node,  // optional pointer ไป node ถัดไป
    
    const Self = @This();
    
    pub fn create(allocator: Allocator, val: i32) !*Self {
        const node = try allocator.create(Self);
        node.* = Self{ .value = val, .next = null };
        return node;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง linked list: 1 -> 2 -> 3 -> null
    const node1 = try Node.create(allocator, 1);
    defer allocator.destroy(node1);
    
    const node2 = try Node.create(allocator, 2);
    defer allocator.destroy(node2);
    
    const node3 = try Node.create(allocator, 3);
    defer allocator.destroy(node3);
    
    node1.next = node2;
    node2.next = node3;
    // node3.next ยังเป็น null
    
    // traverse list
    var current: ?*Node = node1;
    while (current) |node| {
        std.debug.print("{d}", .{node.value});
        current = node.next;
        if (current != null) std.debug.print(" -> ", .{});
    }
    std.debug.print("\n", .{});
}
```

---

## Step 139: Null Pointer Safety

### ทำไม Null Pointer Safety สำคัญ?

Null pointer dereference เป็นหนึ่งในบั๊กที่พบบ่อยที่สุดในโปรแกรม C/C++ Tony Hoare (ผู้คิดค้น null) เรียกมันว่า "The Billion Dollar Mistake"

```c
// C ที่อันตราย:
struct Node* head = NULL;
printf("%d\n", head->value);  // CRASH! Segmentation fault
```

### Zig บังคับให้ตรวจสอบ Null

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

pub fn main() void {
    var node = Node{ .value = 42, .next = null };
    var maybe_node: ?*Node = &node;
    
    // ❌ ไม่สามารถเข้าถึงโดยตรง → compiler error
    // std.debug.print("{d}\n", .{maybe_node.value});  // ERROR!
    
    // ✅ ต้อง unwrap ก่อน
    if (maybe_node) |n| {
        std.debug.print("value = {d}\n", .{n.value});
    }
    
    // ✅ หรือใช้ orelse
    const n = maybe_node orelse return;
    std.debug.print("value = {d}\n", .{n.value});
}
```

### Pattern ป้องกัน Null อย่างปลอดภัย

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16,
    timeout: ?u32,  // optional field
};

fn getTimeout(config: *const Config) u32 {
    // ใช้ orelse เพื่อ default value
    return config.timeout orelse 30_000;  // default 30 วินาที
}

fn processConfig(config: ?*const Config) void {
    // ตรวจสอบ null ก่อนใช้
    const cfg = config orelse {
        std.debug.print("ไม่มี config ใช้ default\n", .{});
        return;
    };
    
    std.debug.print("เชื่อมต่อไปยัง {s}:{d}\n", .{cfg.host, cfg.port});
    std.debug.print("timeout = {d}ms\n", .{getTimeout(cfg)});
}

pub fn main() void {
    const my_config = Config{
        .host = "localhost",
        .port = 8080,
        .timeout = null,
    };
    
    processConfig(&my_config);
    processConfig(null);
}
```

### Optional Chaining Pattern

```zig
const std = @import("std");

const Department = struct {
    name: []const u8,
    manager: ?*Employee,
};

const Employee = struct {
    name: []const u8,
    email: ?[]const u8,
    department: ?*Department,
};

fn getManagerEmail(emp: *const Employee) ?[]const u8 {
    // Optional chaining: ตรวจสอบทุกขั้น
    const dept = emp.department orelse return null;
    const manager = dept.manager orelse return null;
    return manager.email;
}

pub fn main() void {
    var manager = Employee{
        .name = "สมชาย",
        .email = "somchai@example.com",
        .department = null,
    };
    
    var dept = Department{
        .name = "วิศวกรรม",
        .manager = &manager,
    };
    
    var emp = Employee{
        .name = "สมหญิง",
        .email = null,
        .department = &dept,
    };
    
    if (getManagerEmail(&emp)) |email| {
        std.debug.print("Email ของผู้จัดการ: {s}\n", .{email});
    } else {
        std.debug.print("ไม่พบ email ของผู้จัดการ\n", .{});
    }
}
```

### การ Debug Null Pointer

```zig
const std = @import("std");

// Helper function สำหรับ debug
fn expectNotNull(ptr: anytype, comptime message: []const u8) @TypeOf(ptr.?) {
    return ptr orelse {
        std.debug.print("ERROR: {s}\n", .{message});
        @panic(message);
    };
}

pub fn main() void {
    var value: i32 = 100;
    var opt: ?*i32 = &value;
    
    const ptr = expectNotNull(opt, "ต้องการ pointer ที่ไม่ใช่ null");
    std.debug.print("ค่า: {d}\n", .{ptr.*});
    
    // การใช้ if ด้วย else panic
    const ptr2 = if (opt) |p| p else @panic("pointer เป็น null!");
    std.debug.print("ค่า: {d}\n", .{ptr2.*});
}
```

### Common Mistake: Dangling Pointer

```zig
const std = @import("std");

// ❌ ผิด: คืน pointer ไปยัง local variable
fn getBadPointer() *i32 {
    var local: i32 = 42;
    return &local;  // ⚠️ local ถูกทำลายเมื่อฟังก์ชันสิ้นสุด
    // compiler ใน Zig อาจตรวจจับได้ แต่ไม่ใช่ทุกกรณี
}

// ✅ ถูก: รับ pointer มาจากภายนอก
fn processValue(ptr: *i32) void {
    ptr.* *= 2;
}

// ✅ ถูก: ใช้ allocator
fn createValue(allocator: std.mem.Allocator, val: i32) !*i32 {
    const ptr = try allocator.create(i32);
    ptr.* = val;
    return ptr;  // caller ต้อง destroy เมื่อใช้เสร็จ
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var x: i32 = 10;
    processValue(&x);
    std.debug.print("x = {d}\n", .{x});
    
    const heap_ptr = try createValue(gpa.allocator(), 99);
    defer gpa.allocator().destroy(heap_ptr);
    std.debug.print("heap value = {d}\n", .{heap_ptr.*});
}
```

---

## Step 140: @ptrToInt และ @intToPtr

### แปลง Pointer เป็น Integer

`@intFromPtr` (เดิมชื่อ `@ptrToInt` ใน Zig เก่า) ใช้แปลง pointer เป็นตัวเลข

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 42;
    const ptr = &x;
    
    // แปลง pointer เป็น integer (address)
    const addr: usize = @intFromPtr(ptr);
    std.debug.print("address ของ x = 0x{X}\n", .{addr});
    
    // ตรวจสอบ alignment
    std.debug.print("i32 ต้อง align 4 bytes: {}\n", .{addr % 4 == 0});
}
```

**หมายเหตุ**: ใน Zig 0.11+ ใช้ `@intFromPtr` และ `@ptrFromInt` แทน `@ptrToInt` และ `@intToPtr` ที่ deprecated แล้ว

### แปลง Integer เป็น Pointer

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 42;
    
    // รับ address
    const addr: usize = @intFromPtr(&x);
    
    // แปลง address กลับเป็น pointer
    // ต้องระบุ alignment ด้วย @alignCast
    const ptr: *i32 = @ptrFromInt(addr);
    
    std.debug.print("ค่าที่ address นั้น = {d}\n", .{ptr.*});
    
    // แก้ค่าผ่าน reconstructed pointer
    ptr.* = 100;
    std.debug.print("x หลังแก้ไข = {d}\n", .{x});
}
```

### ใช้งานจริง: Pointer Comparison

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 10, 20, 30, 40, 50 };
    
    const start: usize = @intFromPtr(&arr[0]);
    const end: usize = @intFromPtr(&arr[4]);
    
    std.debug.print("start address = 0x{X}\n", .{start});
    std.debug.print("end address   = 0x{X}\n", .{end});
    std.debug.print("ระยะห่าง = {d} bytes\n", .{end - start});
    // ระยะห่าง = 16 bytes (4 elements × 4 bytes/element)
    
    // ตรวจสอบว่า pointer อยู่ในช่วง array หรือเปล่า
    const check_addr: usize = @intFromPtr(&arr[2]);
    if (check_addr >= start and check_addr <= end) {
        std.debug.print("pointer อยู่ใน array\n", .{});
    }
}
```

### ใช้งานจริง: Custom Allocator ตรวจสอบ Alignment

```zig
const std = @import("std");

fn checkAlignment(ptr: *const anyopaque, required_align: usize) bool {
    const addr = @intFromPtr(ptr);
    return addr % required_align == 0;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // allocate ด้วย alignment 16 bytes
    const aligned_mem = try allocator.alignedAlloc(u8, 16, 64);
    defer allocator.free(aligned_mem);
    
    std.debug.print("aligned_mem ptr = {*}\n", .{aligned_mem.ptr});
    std.debug.print("aligned ถูกต้อง? {}\n", .{
        checkAlignment(aligned_mem.ptr, 16)
    });
}
```

### @ptrCast: แปลงชนิดของ Pointer

```zig
const std = @import("std");

pub fn main() void {
    var float_val: f32 = 3.14;
    
    // แปลง *f32 เป็น *u32 เพื่อดู bit pattern
    // ต้องใช้ @alignCast ด้วยถ้า alignment ต่างกัน
    const bits_ptr: *u32 = @ptrCast(&float_val);
    
    std.debug.print("f32 value = {d}\n", .{float_val});
    std.debug.print("bit pattern = 0x{X}\n", .{bits_ptr.*});
    // IEEE 754: 3.14 ≈ 0x4048F5C3
    
    // @bitCast เป็นวิธีที่ปลอดภัยกว่าสำหรับ reinterpret
    const bits: u32 = @bitCast(float_val);
    std.debug.print("@bitCast = 0x{X}\n", .{bits});
}
```

### ตารางสรุป Built-in Functions สำหรับ Pointer

| Function | ใช้งาน | ตัวอย่าง |
|----------|--------|----------|
| `@intFromPtr(ptr)` | pointer → usize | `const addr = @intFromPtr(&x)` |
| `@ptrFromInt(addr)` | usize → pointer | `const ptr: *i32 = @ptrFromInt(addr)` |
| `@ptrCast(ptr)` | แปลงชนิด pointer | `const p: *u8 = @ptrCast(&x)` |
| `@alignCast(ptr)` | เพิ่ม alignment info | `const p: *align(4) T = @alignCast(ptr)` |
| `@bitCast(val)` | reinterpret bits | `const u: u32 = @bitCast(f_val)` |

### ตัวอย่างรวม: Memory Inspector

```zig
const std = @import("std");

fn printMemory(ptr: *const anyopaque, len: usize) void {
    const bytes: [*]const u8 = @ptrCast(ptr);
    const addr = @intFromPtr(ptr);
    
    var i: usize = 0;
    while (i < len) : (i += 1) {
        if (i % 8 == 0) {
            std.debug.print("\n0x{X:0>8}: ", .{addr + i});
        }
        std.debug.print("{X:0>2} ", .{bytes[i]});
    }
    std.debug.print("\n", .{});
}

pub fn main() void {
    const values = [_]i32{ 1, 2, 3, 4 };
    
    std.debug.print("Memory layout ของ array [1,2,3,4]:", .{});
    printMemory(&values, @sizeOf(@TypeOf(values)));
}
```

---

## สรุปภาพรวม: ชนิด Pointer ทั้งหมดใน Zig

```
Pointer Types ใน Zig:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

*T            → Single-item pointer (ชี้ไป 1 element)
              → ไม่รองรับ arithmetic
              → ไม่สามารถเป็น null

*const T      → Single-item pointer ไป const data
              → ไม่สามารถแก้ไขข้อมูลผ่าน pointer นี้

[*]T          → Many-item pointer (ชี้ไปหลาย element)
              → รองรับ arithmetic และ indexing
              → ไม่รู้ขนาด (ไม่มี bounds checking)
              → ต้องแปลงเป็น slice ก่อนใช้อย่างปลอดภัย

[*:s]T        → Sentinel-terminated many-item pointer
              → รู้จุดสิ้นสุดผ่าน sentinel value s

?*T           → Optional single-item pointer
              → สามารถเป็น null ได้
              → ต้อง unwrap ก่อนใช้

?[*]T         → Optional many-item pointer

[]T           → Slice (ptr + len): วิธีปลอดภัยที่สุด
[]const T     → Slice to const data

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Exercises (แบบฝึกหัด)

### Exercise 1: Swap โดยไม่ใช้ตัวแปรชั่วคราว

เขียนฟังก์ชัน `swapXOR` ที่สลับค่าสองตัวแปรโดยใช้ XOR operation (ไม่ใช้ตัวแปร temp)

```zig
fn swapXOR(a: *i32, b: *i32) void {
    // TODO: ใช้ XOR เพื่อสลับค่า
    // hint: a XOR a = 0, a XOR 0 = a
    // a.* = a.* ^ b.*;
    // b.* = a.* ^ b.*;
    // a.* = a.* ^ b.*;
}
```

### Exercise 2: Find Min/Max ด้วย Pointer

เขียนฟังก์ชันที่คืน pointer ไปยังค่า min และ max ใน array

```zig
const Result = struct {
    min: *i32,
    max: *i32,
};

fn findMinMax(arr: []i32) Result {
    // TODO: implement
}
```

### Exercise 3: Reverse Array In-Place

เขียนฟังก์ชันที่กลับค่าใน array โดยใช้ pointers

```zig
fn reverseArray(arr: []i32) void {
    // TODO: ใช้ pointer swap เพื่อกลับ array
}
```

### Exercise 4: Optional Pointer Chain

เขียน function `getFirstPositive` ที่รับ `?[]i32` และคืน `?*i32` ไปยัง element แรกที่เป็นบวก

```zig
fn getFirstPositive(maybe_arr: ?[]i32) ?*i32 {
    // TODO: implement
    // ถ้า maybe_arr เป็น null ให้คืน null
    // ถ้าไม่มี positive element ให้คืน null
    // ถ้าเจอ ให้คืน pointer ไปยัง element นั้น
}
```

### Exercise 5: Struct Counter ขั้นสูง

สร้าง `RateCounter` struct ที่:
- นับจำนวนครั้ง
- เก็บเวลาเริ่มต้น
- มี method `getRate()` คืนค่า rate ต่อวินาที

```zig
const RateCounter = struct {
    count: u64,
    start_ns: i128,
    
    const Self = @This();
    
    pub fn init() Self { ... }
    pub fn tick(self: *Self) void { ... }
    pub fn getRate(self: *const Self) f64 { ... }
    pub fn reset(self: *Self) void { ... }
};
```

---

## เฉลย Exercise

### เฉลย Exercise 1: Swap XOR

```zig
const std = @import("std");

fn swapXOR(a: *i32, b: *i32) void {
    if (a == b) return; // กัน edge case: a และ b ชี้ไปที่เดียวกัน
    a.* = a.* ^ b.*;
    b.* = a.* ^ b.*;
    a.* = a.* ^ b.*;
}

pub fn main() void {
    var x: i32 = 15;
    var y: i32 = 27;
    std.debug.print("ก่อน: x={d}, y={d}\n", .{x, y});
    swapXOR(&x, &y);
    std.debug.print("หลัง: x={d}, y={d}\n", .{x, y});
}
```

### เฉลย Exercise 2: Find Min/Max

```zig
const std = @import("std");

const MinMaxResult = struct {
    min: *i32,
    max: *i32,
};

fn findMinMax(arr: []i32) MinMaxResult {
    var min_ptr = &arr[0];
    var max_ptr = &arr[0];
    
    for (arr[1..]) |*val| {
        if (val.* < min_ptr.*) min_ptr = val;
        if (val.* > max_ptr.*) max_ptr = val;
    }
    
    return MinMaxResult{ .min = min_ptr, .max = max_ptr };
}

pub fn main() void {
    var numbers = [_]i32{ 5, 3, 8, 1, 9, 2, 7 };
    const result = findMinMax(&numbers);
    std.debug.print("min = {d}, max = {d}\n", .{result.min.*, result.max.*});
    
    // เปลี่ยนค่า min เป็น 0
    result.min.* = 0;
    std.debug.print("หลังเปลี่ยน min: {d}\n", .{numbers[3]});
}
```

### เฉลย Exercise 3: Reverse Array

```zig
const std = @import("std");

fn reverseArray(arr: []i32) void {
    var left: usize = 0;
    var right: usize = arr.len - 1;
    
    while (left < right) {
        // swap
        const temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left += 1;
        right -= 1;
    }
}

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("ก่อน: ", .{});
    for (arr) |v| std.debug.print("{d} ", .{v});
    
    reverseArray(&arr);
    
    std.debug.print("\nหลัง: ", .{});
    for (arr) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
}
```

### เฉลย Exercise 4: Optional Pointer Chain

```zig
const std = @import("std");

fn getFirstPositive(maybe_arr: ?[]i32) ?*i32 {
    const arr = maybe_arr orelse return null;
    for (arr) |*val| {
        if (val.* > 0) return val;
    }
    return null;
}

pub fn main() void {
    var numbers1 = [_]i32{ -3, -1, 0, 5, 2 };
    var numbers2 = [_]i32{ -1, -2, -3 };
    
    if (getFirstPositive(&numbers1)) |ptr| {
        std.debug.print("พบค่าบวกแรก: {d}\n", .{ptr.*});
        ptr.* = 100; // แก้ค่าได้
        std.debug.print("หลังแก้ไข: {d}\n", .{numbers1[3]});
    }
    
    if (getFirstPositive(&numbers2)) |ptr| {
        std.debug.print("พบค่าบวก: {d}\n", .{ptr.*});
    } else {
        std.debug.print("ไม่มีค่าบวก\n", .{});
    }
    
    if (getFirstPositive(null)) |ptr| {
        std.debug.print("{d}\n", .{ptr.*});
    } else {
        std.debug.print("array เป็น null\n", .{});
    }
}
```

---

## สรุปบทเรียน

### สิ่งที่เรียนในบทนี้

```
Step 131: Pointers คืออะไร
  ✓ หน่วยความจำและ addresses
  ✓ ทำไมต้องใช้ pointers
  ✓ ความแตกต่างจาก C

Step 132: Single-item Pointer *T
  ✓ การสร้าง pointer
  ✓ ขนาดของ pointer (8 bytes บน 64-bit)
  ✓ pointer กับ struct

Step 133: & และ * operators
  ✓ & สำหรับ address-of
  ✓ .* สำหรับ dereference
  ✓ double pointer **T

Step 134: const pointer vs *const T
  ✓ 4 combination ของ const/mutable
  ✓ *const T สำหรับ read-only parameter
  ✓ ความชัดเจนใน API design

Step 135: Pointers กับ Structs
  ✓ self: *Self สำหรับ mutating methods
  ✓ self: *const Self สำหรับ reading methods
  ✓ self: Self สำหรับ copying methods

Step 136: Pointer Arithmetic
  ✓ ต้องใช้ [*]T
  ✓ +N เลื่อน N elements
  ✓ แปลงเป็น slice เพื่อความปลอดภัย

Step 137: Many-item Pointer [*]T
  ✓ ความแตกต่างจาก *T
  ✓ การใช้กับ C interop
  ✓ sentinel-terminated pointer [*:0]T

Step 138: Optional Pointer ?*T
  ✓ แก้ปัญหา null pointer ใน C
  ✓ if-let pattern
  ✓ orelse, .? operators

Step 139: Null Pointer Safety
  ✓ Dangling pointer
  ✓ Optional chaining
  ✓ Debug helpers

Step 140: @intFromPtr และ @ptrFromInt
  ✓ แปลงระหว่าง pointer และ integer
  ✓ @ptrCast สำหรับ type conversion
  ✓ Memory inspection
```

### Best Practices สรุป

1. **ใช้ slice `[]T` แทน `[*]T`** เมื่อเป็นไปได้ เพราะมี bounds checking
2. **ใช้ `*const T`** เมื่อฟังก์ชันแค่อ่าน ไม่แก้ไข
3. **ใช้ `?*T`** แทนการส่ง raw null pointer
4. **ไม่ return pointer ไปยัง local variable**
5. **ใช้ `if (ptr) |p|` pattern** เพื่อ unwrap optional pointer อย่างปลอดภัย
6. **ชื่น`@alignCast`** เมื่อทำ pointer cast เพื่อ alignment safety

### ขั้นตอนถัดไป

ในบทต่อไป (Part 15) เราจะเรียนเรื่อง:
- Slices ขั้นสูง
- Slice operations
- String handling ด้วย slice
- Memory layout ของ slice

---

*Part 14 จบแล้ว | ไปต่อ: [Part 15: Slices ขั้นสูง →](./part-015-slices-advanced.md)*
