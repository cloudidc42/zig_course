# ส่วนที่ 24: Type System ใน Zig (Steps 231-240)

> **ระดับ:** กลาง-สูง  
> **เวลาเรียน:** ประมาณ 3-4 ชั่วโมง  
> **ความรู้ที่ต้องการ:** พื้นฐาน Zig, comptime, generic programming

---

## บทนำ: ทำไม Type System ถึงสำคัญ?

Type System เป็นหัวใจสำคัญของภาษาโปรแกรมสมัยใหม่ ใน Zig นั้น type system ถูกออกแบบมาเพื่อ:

1. **ความปลอดภัย** - ตรวจจับ bug ตั้งแต่ compile time
2. **ประสิทธิภาพ** - ไม่มี runtime overhead จาก type checking
3. **ความยืดหยุ่น** - ใช้ comptime เพื่อสร้าง generic code
4. **ความชัดเจน** - โค้ดที่อ่านแล้วเข้าใจพฤติกรรมได้ทันที

ใน Zig type เป็น **first-class value** ที่สามารถส่งผ่านเป็น parameter, คืนค่าจาก function, และใช้ใน comptime ได้อย่างอิสระ

---

## Step 231: Type System Overview ใน Zig

### ภาพรวมของ Type System

Zig มี type categories หลักดังนี้:

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // 1. Primitive Types (ชนิดข้อมูลพื้นฐาน)
    // ============================================

    // Integer types - มีขนาดและ sign ที่กำหนดได้
    const i8_val: i8 = -128;          // signed 8-bit
    const u8_val: u8 = 255;           // unsigned 8-bit
    const i16_val: i16 = -32768;      // signed 16-bit
    const u16_val: u16 = 65535;       // unsigned 16-bit
    const i32_val: i32 = -2147483648; // signed 32-bit
    const u32_val: u32 = 4294967295;  // unsigned 32-bit
    const i64_val: i64 = -9223372036854775808;
    const u64_val: u64 = 18446744073709551615;
    const i128_val: i128 = 0;
    const u128_val: u128 = 0;

    // Platform-dependent integer types
    const isize_val: isize = -1;  // signed pointer-sized integer
    const usize_val: usize = 0;   // unsigned pointer-sized integer (ใช้บ่อยมาก!)

    // Arbitrary-width integers (Zig's unique feature!)
    const u1_val: u1 = 1;          // 1-bit integer
    const i3_val: i3 = -4;         // 3-bit signed integer
    const u7_val: u7 = 127;        // 7-bit unsigned integer
    const u256_val: u256 = 0;      // 256-bit unsigned integer

    // ============================================
    // 2. Floating Point Types
    // ============================================
    const f16_val: f16 = 1.0;   // IEEE 754 half precision
    const f32_val: f32 = 3.14;  // IEEE 754 single precision
    const f64_val: f64 = 3.14159265358979;  // IEEE 754 double precision
    const f80_val: f80 = 0.0;   // x87 extended precision
    const f128_val: f128 = 0.0; // IEEE 754 quad precision

    // ============================================
    // 3. Boolean Type
    // ============================================
    const bool_val: bool = true;

    // ============================================
    // 4. Character/String Types
    // ============================================
    const char: u8 = 'A';           // ASCII character (u8)
    const unicode_char: u21 = '🦎'; // Unicode codepoint (u21)
    const string: []const u8 = "สวัสดี Zig!";  // string slice

    // ============================================
    // 5. Optional Types (Nullable)
    // ============================================
    const maybe_int: ?i32 = null;         // Optional integer
    const some_int: ?i32 = 42;            // Optional with value
    const unwrapped = some_int orelse 0;  // Unwrap with default

    // ============================================
    // 6. Error Union Types
    // ============================================
    const MyError = error{
        FileNotFound,
        PermissionDenied,
    };
    // error!i32 หมายความว่า ค่าที่คืนอาจเป็น i32 หรือ error
    const result: MyError!i32 = 42;
    const err_result: MyError!i32 = MyError.FileNotFound;

    // ============================================
    // 7. Pointer Types
    // ============================================
    var num: i32 = 42;
    const single_ptr: *i32 = &num;           // Single-item pointer
    const const_ptr: *const i32 = &num;      // Pointer to const
    const many_ptr: [*]i32 = @ptrCast(&num); // Many-item pointer

    // ============================================
    // 8. Array & Slice Types
    // ============================================
    const arr: [5]i32 = [5]i32{ 1, 2, 3, 4, 5 };
    const slice: []const i32 = &arr;

    // ============================================
    // 9. Struct, Enum, Union Types
    // ============================================
    const Point = struct { x: f32, y: f32 };
    const Direction = enum { north, south, east, west };
    const Value = union(enum) {
        int: i64,
        float: f64,
        boolean: bool,
    };

    // ============================================
    // 10. Function Types
    // ============================================
    const fn_type: fn (i32) i32 = double;

    // ============================================
    // 11. Type Type (comptime only)
    // ============================================
    const MyType: type = i32;  // type เป็น first-class value!
    var typed_var: MyType = 100;

    // ============================================
    // 12. void, noreturn, anyopaque
    // ============================================
    // void   - ไม่มีค่า (ใช้เป็น return type)
    // noreturn - function ที่ไม่มีวัน return (เช่น @panic)
    // anyopaque - pointer to unknown type (เหมือน void* ใน C)

    _ = i8_val;
    _ = u8_val;
    _ = i16_val;
    _ = u16_val;
    _ = i32_val;
    _ = u32_val;
    _ = i64_val;
    _ = u64_val;
    _ = i128_val;
    _ = u128_val;
    _ = isize_val;
    _ = usize_val;
    _ = u1_val;
    _ = i3_val;
    _ = u7_val;
    _ = u256_val;
    _ = f16_val;
    _ = f32_val;
    _ = f64_val;
    _ = f80_val;
    _ = f128_val;
    _ = bool_val;
    _ = char;
    _ = unicode_char;
    _ = string;
    _ = maybe_int;
    _ = unwrapped;
    _ = result;
    _ = err_result;
    _ = single_ptr;
    _ = const_ptr;
    _ = many_ptr;
    _ = slice;
    _ = fn_type;
    _ = typed_var;

    std.debug.print("Type system overview complete!\n", .{});
}

fn double(x: i32) i32 {
    return x * 2;
}
```

### ความสัมพันธ์ระหว่าง Types

```
Type Hierarchy ใน Zig:
───────────────────────────────────────────────────
type                    (the meta-type)
├── comptime_int        (integer literals)
├── comptime_float      (float literals)
├── bool
├── void
├── noreturn
├── anyopaque
├── Integer types       (i8, u8, i16, u16, ..., usize)
├── Float types         (f16, f32, f64, f80, f128)
├── Pointer types       (*T, [*]T, *const T, ...)
├── Array types         ([N]T)
├── Slice types         ([]T, []const T)
├── Optional types      (?T)
├── Error types         (error{...}, anyerror)
├── Error union types   (E!T)
├── Struct types        (struct { ... })
├── Enum types          (enum { ... })
├── Union types         (union(E) { ... })
└── Function types      (fn(T) R)
```

### Type ที่น่าสนใจเป็นพิเศษใน Zig

```zig
const std = @import("std");

pub fn main() void {
    // comptime_int - integer literal ที่ยังไม่ถูก coerce
    const x = 42;          // type คือ comptime_int
    const y: i32 = 42;     // type คือ i32 (ถูก coerce)

    // comptime_float - float literal ที่ยังไม่ถูก coerce
    const pi = 3.14159;       // type คือ comptime_float
    const pi_f32: f32 = 3.14; // type คือ f32

    // comptime_int มีความแม่นยำไม่จำกัด!
    const big_num = 123456789012345678901234567890;
    std.debug.print("Type of 42: {}\n", .{@TypeOf(x)});
    std.debug.print("Type of y: {}\n", .{@TypeOf(y)});
    std.debug.print("Type of pi: {}\n", .{@TypeOf(pi)});
    std.debug.print("Type of big_num: {}\n", .{@TypeOf(big_num)});
}
```

---

## Step 232: @TypeOf - Runtime Type Checking

### @TypeOf คืออะไร?

`@TypeOf` เป็น builtin function ที่คืนค่า **type** ของ expression ที่ให้ โดย evaluate ใน comptime เสมอ

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // การใช้ @TypeOf พื้นฐาน
    // ============================================
    const a: i32 = 42;
    const b: f64 = 3.14;
    const c = "hello";
    const d = true;

    std.debug.print("Type of a: {}\n", .{@TypeOf(a)});        // i32
    std.debug.print("Type of b: {}\n", .{@TypeOf(b)});        // f64
    std.debug.print("Type of c: {}\n", .{@TypeOf(c)});        // *const [5:0]u8
    std.debug.print("Type of d: {}\n", .{@TypeOf(d)});        // bool

    // ============================================
    // @TypeOf กับ multiple arguments
    // ============================================
    // เมื่อส่งหลาย argument จะหา "peer type" ที่เหมาะสม
    const peer1 = @TypeOf(1, 2, 3);          // comptime_int
    const peer2 = @TypeOf(1, @as(i32, 2));   // i32
    const peer3 = @TypeOf(1.0, 2);           // comptime_float

    std.debug.print("Peer type of 1,2,3: {}\n", .{peer1});
    std.debug.print("Peer type of 1,i32(2): {}\n", .{peer2});
    std.debug.print("Peer type of 1.0,2: {}\n", .{peer3});

    // ============================================
    // ใช้ @TypeOf ใน generic functions
    // ============================================
    const result = add(10, 20);
    std.debug.print("add result: {}\n", .{result});

    const result2 = add(10.5, 20.5);
    std.debug.print("add result2: {}\n", .{result2});
}

// Generic function ที่ใช้ @TypeOf เพื่อ preserve type
fn add(a: anytype, b: anytype) @TypeOf(a, b) {
    return a + b;
}
```

### @TypeOf ในการ introspection

```zig
const std = @import("std");

const Point = struct {
    x: f32,
    y: f32,
};

const Color = enum {
    red,
    green,
    blue,
};

pub fn main() void {
    const p = Point{ .x = 1.0, .y = 2.0 };
    const c = Color.red;
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    const slice = arr[0..];

    std.debug.print("Type of struct: {}\n", .{@TypeOf(p)});
    std.debug.print("Type of enum: {}\n", .{@TypeOf(c)});
    std.debug.print("Type of array: {}\n", .{@TypeOf(arr)});
    std.debug.print("Type of slice: {}\n", .{@TypeOf(slice)});

    // ============================================
    // ใช้ร่วมกับ @typeInfo เพื่อ inspect type structure
    // ============================================
    const type_info = @typeInfo(@TypeOf(p));
    switch (type_info) {
        .Struct => |s| {
            std.debug.print("It's a struct with {} fields\n", .{s.fields.len});
            for (s.fields) |field| {
                std.debug.print("  field: {s} of type {}\n", .{ field.name, field.type });
            }
        },
        else => std.debug.print("Not a struct\n", .{}),
    }
}
```

### ใช้ @TypeOf เพื่อเขียน Type-Safe Functions

```zig
const std = @import("std");

// ฟังก์ชันที่ใช้ @TypeOf เพื่อคืน type ที่ถูกต้อง
fn max(a: anytype, b: anytype) @TypeOf(a, b) {
    return if (a > b) a else b;
}

// ฟังก์ชันที่ตรวจสอบ type ก่อนดำเนินการ
fn printTypedValue(val: anytype) void {
    const T = @TypeOf(val);
    const info = @typeInfo(T);

    switch (info) {
        .Int => std.debug.print("Integer: {}\n", .{val}),
        .Float => std.debug.print("Float: {d}\n", .{val}),
        .Bool => std.debug.print("Boolean: {}\n", .{val}),
        .Pointer => |ptr| {
            if (ptr.child == u8 and ptr.size == .Slice) {
                std.debug.print("String: {s}\n", .{val});
            } else {
                std.debug.print("Pointer to {}\n", .{ptr.child});
            }
        },
        else => std.debug.print("Other type: {}\n", .{T}),
    }
}

pub fn main() void {
    std.debug.print("max(3, 7) = {}\n", .{max(3, 7)});
    std.debug.print("max(3.14, 2.71) = {d}\n", .{max(3.14, 2.71)});

    printTypedValue(@as(i32, 42));
    printTypedValue(@as(f64, 3.14));
    printTypedValue(true);
    printTypedValue("Hello, World!");
}
```

---

## Step 233: Type Coercion Rules

### Type Coercion คืออะไร?

Type coercion คือการแปลง type โดยอัตโนมัติโดย compiler เมื่อ type ต้นทางและปลายทาง "compatible" กัน

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // Rule 1: comptime_int -> integer types
    // ============================================
    // Integer literals (comptime_int) coerce เป็น integer type ใดก็ได้
    // ตราบที่ค่าอยู่ใน range
    const a: i32 = 42;      // comptime_int 42 -> i32
    const b: u8 = 200;      // comptime_int 200 -> u8
    const c: i64 = -1000;   // comptime_int -1000 -> i64

    // ============================================
    // Rule 2: comptime_float -> float types
    // ============================================
    const pi: f32 = 3.14159;   // comptime_float -> f32
    const e: f64 = 2.71828;    // comptime_float -> f64

    // ============================================
    // Rule 3: Smaller int -> larger int (same signedness)
    // ============================================
    const small: u8 = 100;
    const large: u16 = small;    // u8 -> u16 (widening, safe!)
    const larger: u32 = small;   // u8 -> u32 (widening, safe!)
    // const smaller: u8 = large; // ERROR! ไม่สามารถ narrow ได้

    // ============================================
    // Rule 4: T -> ?T (optional)
    // ============================================
    const num: i32 = 42;
    const maybe: ?i32 = num;     // i32 -> ?i32 (implicit wrap in optional)

    // ============================================
    // Rule 5: T -> anyerror!T
    // ============================================
    const val: i32 = 42;
    const err_union: anyerror!i32 = val;   // i32 -> anyerror!i32

    // ============================================
    // Rule 6: E!T -> anyerror!T
    // ============================================
    const MyError = error{Fail};
    const specific: MyError!i32 = 42;
    const general: anyerror!i32 = specific;  // MyError!i32 -> anyerror!i32

    // ============================================
    // Rule 7: *[N]T -> []T (array -> slice)
    // ============================================
    const arr: [5]i32 = [_]i32{ 1, 2, 3, 4, 5 };
    const slice: []const i32 = &arr;   // *[5]i32 -> []const i32

    // ============================================
    // Rule 8: *[N]T -> [*]T (array -> many-pointer)
    // ============================================
    const many_ptr: [*]const i32 = &arr;  // *[5]i32 -> [*]const i32

    // ============================================
    // Rule 9: *T -> *const T (adding const)
    // ============================================
    var mutable: i32 = 42;
    const ptr: *i32 = &mutable;
    const const_ptr: *const i32 = ptr;  // *i32 -> *const i32

    // ============================================
    // Rule 10: null -> ?T
    // ============================================
    const null_val: ?i32 = null;  // null coerces to any optional

    // ============================================
    // Rule 11: error -> E!T
    // ============================================
    const MyErr = error{NotFound};
    const err: MyErr!i32 = MyErr.NotFound;  // error -> error union

    std.debug.print("Type coercion examples:\n", .{});
    std.debug.print("a={}, b={}, c={}\n", .{a, b, c});
    std.debug.print("pi={d}, e={d}\n", .{pi, e});
    std.debug.print("small={}, large={}, larger={}\n", .{small, large, larger});
    std.debug.print("num={}, maybe={?}\n", .{num, maybe});

    _ = val;
    _ = err_union;
    _ = specific;
    _ = general;
    _ = slice;
    _ = many_ptr;
    _ = const_ptr;
    _ = null_val;
    _ = err;
}
```

### Coercion ที่ไม่ทำงาน (และวิธีแก้)

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // DOES NOT work: signed -> unsigned
    // ============================================
    const signed: i32 = 42;
    // const unsigned: u32 = signed; // ERROR: i32 ไม่ coerce เป็น u32 โดยอัตโนมัติ
    const unsigned: u32 = @intCast(signed);  // ต้องใช้ @intCast

    // ============================================
    // DOES NOT work: larger -> smaller (narrowing)
    // ============================================
    const big: u32 = 1000;
    // const small: u8 = big; // ERROR: u32 ไม่ coerce เป็น u8
    const small: u8 = @intCast(big);  // ต้องใช้ @intCast

    // ============================================
    // DOES NOT work: float -> int
    // ============================================
    const f: f64 = 3.14;
    // const i: i32 = f; // ERROR: f64 ไม่ coerce เป็น i32
    const i: i32 = @intFromFloat(f);  // ต้องใช้ @intFromFloat

    // ============================================
    // DOES NOT work: int -> float
    // ============================================
    const n: i32 = 42;
    // const fl: f64 = n; // ERROR: i32 ไม่ coerce เป็น f64
    const fl: f64 = @floatFromInt(n);  // ต้องใช้ @floatFromInt

    // ============================================
    // DOES NOT work: *T -> *U (different types)
    // ============================================
    var x: i32 = 42;
    const ptr_i32: *i32 = &x;
    // const ptr_u32: *u32 = ptr_i32; // ERROR
    const ptr_u32: *u32 = @ptrCast(ptr_i32);  // ต้องใช้ @ptrCast

    std.debug.print("unsigned={}, small={}, i={}, fl={d}\n", .{unsigned, small, i, fl});
    std.debug.print("ptr_u32={}\n", .{ptr_u32.*});
}
```

### Coercion ใน Struct/Function Context

```zig
const std = @import("std");

const Container = struct {
    value: i64,

    fn init(v: anytype) Container {
        // comptime_int coerce -> i64
        return Container{ .value = v };
    }
};

fn processOptional(val: ?i32) void {
    // i32 coerce -> ?i32 เมื่อส่งเป็น argument
    if (val) |v| {
        std.debug.print("Got value: {}\n", .{v});
    } else {
        std.debug.print("Got null\n", .{});
    }
}

pub fn main() void {
    const c = Container.init(42);     // 42 (comptime_int) -> i64
    std.debug.print("value: {}\n", .{c.value});

    processOptional(100);   // i32 100 -> ?i32 อัตโนมัติ
    processOptional(null);  // null -> ?i32 อัตโนมัติ

    // Array literal coerce to slice
    const numbers = [_]i32{ 1, 2, 3, 4, 5 };
    sum_slice(&numbers);  // *[5]i32 -> []const i32
}

fn sum_slice(nums: []const i32) void {
    var total: i32 = 0;
    for (nums) |n| total += n;
    std.debug.print("Sum: {}\n", .{total});
}
```

---

## Step 234: Peer Type Resolution

### Peer Type Resolution คืออะไร?

เมื่อมีหลาย expression ที่ต้องมี type เดียวกัน (เช่นใน if-else, switch, หรือ array literal) Zig จะหา **peer type** ที่ทุก expression สามารถ coerce เข้าหาได้

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // ตัวอย่างที่ 1: if-else expression
    // ============================================
    const condition = true;

    // u8 และ u16 -> peer type คือ u16
    const result1 = if (condition) @as(u8, 10) else @as(u16, 300);
    std.debug.print("result1 type: {}, value: {}\n", .{@TypeOf(result1), result1});

    // i32 และ u8 -> peer type คือ i32
    // (u8 สามารถ coerce เป็น i32 ได้ แต่ i32 ไม่ coerce เป็น u8)
    const result2 = if (condition) @as(i32, -100) else @as(u8, 50);
    std.debug.print("result2 type: {}, value: {}\n", .{@TypeOf(result2), result2});

    // comptime_int และ i32 -> peer type คือ i32
    const result3 = if (condition) @as(i32, 5) else 10;
    std.debug.print("result3 type: {}\n", .{@TypeOf(result3)});

    // ============================================
    // ตัวอย่างที่ 2: switch expression
    // ============================================
    const x: u8 = 2;
    const result4 = switch (x) {
        1 => @as(u8, 100),
        2 => @as(u16, 1000),  // กำหนด peer type เป็น u16
        else => @as(u16, 0),
    };
    std.debug.print("result4 type: {}, value: {}\n", .{@TypeOf(result4), result4});

    // ============================================
    // ตัวอย่างที่ 3: ?T และ null
    // ============================================
    const opt_result = if (condition) @as(i32, 42) else null;
    std.debug.print("opt_result type: {}\n", .{@TypeOf(opt_result)}); // ?i32

    // ============================================
    // ตัวอย่างที่ 4: Error union peer type
    // ============================================
    const E = error{Fail};
    const err_result = if (condition) @as(E!i32, 42) else E.Fail;
    std.debug.print("err_result type: {}\n", .{@TypeOf(err_result)});

    // ============================================
    // ตัวอย่างที่ 5: Pointer peer types
    // ============================================
    var arr1 = [3]u8{ 1, 2, 3 };
    var arr2 = [5]u8{ 4, 5, 6, 7, 8 };

    // *[3]u8 และ *[5]u8 -> peer type คือ []u8
    const ptr_result = if (condition) &arr1 else &arr2;
    std.debug.print("ptr_result type: {}\n", .{@TypeOf(ptr_result)});
}
```

### Peer Type Resolution Rules

```zig
const std = @import("std");

// Rule: Integer peer types
// - ถ้าทั้งคู่เป็น signed -> ใช้ type ที่ใหญ่กว่า
// - ถ้าทั้งคู่เป็น unsigned -> ใช้ type ที่ใหญ่กว่า
// - ถ้าหนึ่ง signed หนึ่ง unsigned และ unsigned ใหญ่กว่า -> error (ambiguous)
// - comptime_int -> coerce เป็น runtime type

fn demonstratePeerTypes() void {
    // Same sign, different size -> larger type wins
    const a: i16 = 100;
    const b: i32 = 200;
    const c = if (true) a else b;  // peer type: i32
    std.debug.print("i16, i32 peer: {}\n", .{@TypeOf(c)});

    // Both unsigned -> larger wins
    const d: u8 = 10;
    const e: u32 = 10000;
    const f = if (true) d else e;  // peer type: u32
    std.debug.print("u8, u32 peer: {}\n", .{@TypeOf(f)});

    // comptime_int adapts to runtime type
    const g: i32 = 42;
    const h = if (true) g else 100;  // 100 is comptime_int, adapts to i32
    std.debug.print("i32, comptime_int peer: {}\n", .{@TypeOf(h)});

    // Optional peer types
    const i: ?i32 = 42;
    const j = if (true) i else null;  // ?i32 and null -> ?i32
    std.debug.print("?i32, null peer: {}\n", .{@TypeOf(j)});
}

pub fn main() void {
    demonstratePeerTypes();
}
```

### Peer Type Resolution ใน Ternary-style operations

```zig
const std = @import("std");

const MathError = error{
    DivisionByZero,
    Overflow,
};

fn safeDivide(a: i32, b: i32) MathError!f64 {
    if (b == 0) return MathError.DivisionByZero;
    // i32 coerce to f64 via @floatFromInt ไม่ได้ coerce โดยตรง
    // แต่เราต้องทำให้ explicit
    return @as(f64, @floatFromInt(a)) / @as(f64, @floatFromInt(b));
}

pub fn main() void {
    // Peer type ใน while loop
    var i: u8 = 0;
    const limit: u16 = 256;  // u16 เพื่อให้ i < limit ทำงานได้

    // u8 และ u16 -> peer type resolution ใน comparison
    while (i < limit) : (i +%= 1) {
        if (i == 10) break;
    }
    std.debug.print("i = {}\n", .{i});

    // Peer type ใน try/catch-style
    const result = safeDivide(10, 3) catch |err| blk: {
        std.debug.print("Error: {}\n", .{err});
        break :blk 0.0;  // f64 peer กับ f64 return ของ safeDivide
    };
    std.debug.print("Result: {d:.4}\n", .{result});
}
```

---

## Step 235: Type Casting Hierarchy

### ภาพรวมของ Cast Functions ใน Zig

Zig มี cast functions ที่ชัดเจนว่าทำอะไร และแต่ละอันมีระดับ "ความอันตราย" ต่างกัน:

```
ระดับความปลอดภัย (ปลอดภัยมาก -> อันตราย):
───────────────────────────────────────────
@as           - ปลอดภัยสุด, เฉพาะ valid coercions
@intCast      - checked at runtime (ใน Debug mode)
@floatCast    - may lose precision
@intFromFloat - truncates fractional part
@floatFromInt - exact for small integers
@ptrCast      - reinterpret pointer type
@bitCast      - reinterpret bits completely
@truncate     - explicitly drop high bits
```

### @as - Safe Coercion

```zig
const std = @import("std");

pub fn main() void {
    // @as(T, expr) - coerce expr เป็น T
    // ใช้ได้เฉพาะเมื่อ coercion valid (compiler ตรวจ)

    // comptime_int -> runtime integer
    const a = @as(i32, 42);
    const b = @as(u64, 1000);

    // Widening
    const small: u8 = 100;
    const wide = @as(u32, small);  // u8 -> u32

    // Adding const
    var mutable: i32 = 42;
    const as_const = @as(*const i32, &mutable);

    // Optional wrapping
    const opt = @as(?i32, 42);

    // Error union wrapping
    const err_union = @as(anyerror!i32, 42);

    std.debug.print("a={}, b={}, wide={}\n", .{a, b, wide});
    std.debug.print("as_const={}\n", .{as_const.*});
    std.debug.print("opt={?}\n", .{opt});
    _ = err_union;

    // @as ไม่สามารถ:
    // - Narrowing integer (u32 -> u8) - ต้องใช้ @intCast
    // - signed <-> unsigned - ต้องใช้ @intCast หรือ @bitCast
    // - float <-> int - ต้องใช้ @floatFromInt หรือ @intFromFloat
}
```

### @intCast - Integer Casting

```zig
const std = @import("std");

pub fn main() void {
    // @intCast(expr) - cast integer เป็น integer type อื่น
    // ใน Debug mode: ตรวจสอบ overflow -> panic ถ้าค่าไม่พอดี
    // ใน ReleaseFast: ไม่ตรวจสอบ (ระวัง!)

    // Narrowing
    const big: u32 = 200;
    const small: u8 = @intCast(big);   // u32 -> u8 (200 อยู่ใน range ของ u8)
    std.debug.print("small={}\n", .{small});

    // Signed <-> unsigned
    const signed: i32 = 42;
    const unsigned: u32 = @intCast(signed);   // i32 -> u32
    std.debug.print("unsigned={}\n", .{unsigned});

    // อันตราย! ค่าไม่พอดี -> panic ใน Debug mode
    // const overflow: i32 = 300;
    // const small2: u8 = @intCast(overflow);  // 300 > 255 -> PANIC!

    // วิธีปลอดภัย: ตรวจสอบก่อน
    const value: i32 = 300;
    if (value >= 0 and value <= 255) {
        const safe: u8 = @intCast(value);
        std.debug.print("safe={}\n", .{safe});
    } else {
        std.debug.print("value {} does not fit in u8\n", .{value});
    }

    // @intCast กับ negative -> unsigned
    // ใน Debug mode: panic (negative ไม่สามารถแทนด้วย unsigned)
    // const neg: i32 = -1;
    // const to_u32: u32 = @intCast(neg);  // PANIC!

    // ถ้าต้องการ reinterpret bits ให้ใช้ @bitCast
    const neg: i32 = -1;
    const reinterpreted: u32 = @bitCast(neg);  // -1 -> 0xFFFFFFFF
    std.debug.print("reinterpreted={}\n", .{reinterpreted});
}
```

### @floatCast - Float Casting

```zig
const std = @import("std");

pub fn main() void {
    // @floatCast(expr) - cast float เป็น float type อื่น

    // Widening (ไม่สูญเสียข้อมูล)
    const f32_val: f32 = 3.14;
    const f64_val: f64 = @floatCast(f32_val);   // f32 -> f64
    std.debug.print("f64_val={d}\n", .{f64_val});

    // Narrowing (อาจสูญเสีย precision)
    const precise: f64 = 3.141592653589793;
    const less_precise: f32 = @floatCast(precise);  // f64 -> f32
    std.debug.print("precise={d}\n", .{precise});
    std.debug.print("less_precise={d}\n", .{less_precise});
    // สังเกต: ค่าไม่เหมือนกันเพราะ f32 มี precision น้อยกว่า

    // f64 -> f16 (อาจเกิด loss ของ precision มาก)
    const big_float: f64 = 12345.6789;
    const f16_val: f16 = @floatCast(big_float);
    std.debug.print("big_float={d}, f16_val={d}\n", .{big_float, @as(f32, @floatCast(f16_val))});
}
```

### @intFromFloat และ @floatFromInt

```zig
const std = @import("std");

pub fn main() void {
    // @intFromFloat(expr) - แปลง float เป็น integer
    // truncates (ตัดทิ้ง) fractional part
    const f: f64 = 3.99;
    const i: i32 = @intFromFloat(f);   // -> 3 (ไม่ใช่ 4!)
    std.debug.print("3.99 -> int: {}\n", .{i});

    const neg_f: f64 = -3.99;
    const neg_i: i32 = @intFromFloat(neg_f);  // -> -3 (truncate toward zero)
    std.debug.print("-3.99 -> int: {}\n", .{neg_i});

    // อันตราย: ถ้า float ไม่พอดีกับ int range -> undefined behavior!
    // ควรตรวจสอบก่อน
    const safe_convert = blk: {
        const value: f64 = 256.7;
        if (value >= 0 and value <= 255) {
            break :blk @as(u8, @intFromFloat(value));
        } else {
            break :blk @as(u8, 0);
        }
    };
    std.debug.print("safe_convert={}\n", .{safe_convert});

    // @floatFromInt(expr) - แปลง integer เป็น float
    const n: i32 = 42;
    const f_from_i: f64 = @floatFromInt(n);
    std.debug.print("42 as f64: {d}\n", .{f_from_i});

    // อาจสูญเสีย precision สำหรับ integer ขนาดใหญ่
    const big_int: i64 = 9007199254740993;  // 2^53 + 1
    const f64_big: f64 = @floatFromInt(big_int);
    std.debug.print("9007199254740993 as f64: {d}\n", .{f64_big});
    // อาจได้ 9007199254740992.0 เพราะ f64 มี 53 bits mantissa
}
```

### @ptrCast - Pointer Casting

```zig
const std = @import("std");

pub fn main() void {
    // @ptrCast(ptr) - reinterpret pointer type
    // อันตรายมาก: ไม่มีการตรวจสอบ type safety

    // Cast pointer ไปยัง *anyopaque (เหมือน void* ใน C)
    var x: i32 = 42;
    const void_ptr: *anyopaque = @ptrCast(&x);
    std.debug.print("void_ptr obtained\n", .{});

    // Cast กลับมา (ต้องรู้ type จริงๆ)
    const back_ptr: *i32 = @ptrCast(@alignCast(void_ptr));
    std.debug.print("value through cast: {}\n", .{back_ptr.*});

    // Cast struct pointer เป็น u8 pointer (เพื่อ byte-level access)
    const Point = struct { x: i32, y: i32 };
    var p = Point{ .x = 1, .y = 2 };
    const byte_ptr: [*]u8 = @ptrCast(&p);

    // อ่าน bytes ของ struct
    std.debug.print("Bytes of Point{1,2}: ", .{});
    for (0..@sizeOf(Point)) |idx| {
        std.debug.print("{x:02} ", .{byte_ptr[idx]});
    }
    std.debug.print("\n", .{});

    // @ptrCast กับ alignment
    var arr: [8]u8 align(4) = [_]u8{0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08};
    const u32_ptr: *u32 = @ptrCast(&arr);
    std.debug.print("u32 view of bytes: 0x{x}\n", .{u32_ptr.*});
}
```

### @bitCast - Bit Reinterpretation

```zig
const std = @import("std");

pub fn main() void {
    // @bitCast(val) - reinterpret bits เป็น type อื่น
    // source และ destination ต้องมีขนาด (bit count) เท่ากัน

    // i32 <-> u32
    const neg: i32 = -1;
    const as_u32: u32 = @bitCast(neg);
    std.debug.print("-1 as u32: {}\n", .{as_u32});  // 4294967295 (0xFFFFFFFF)

    const max_u32: u32 = 0xFFFFFFFF;
    const as_i32: i32 = @bitCast(max_u32);
    std.debug.print("0xFFFFFFFF as i32: {}\n", .{as_i32});  // -1

    // f32 <-> u32 (IEEE 754 bit pattern)
    const f_val: f32 = 1.0;
    const f_bits: u32 = @bitCast(f_val);
    std.debug.print("1.0 as bits: 0x{x}\n", .{f_bits});  // 0x3F800000

    const pi_approx: f32 = 3.14;
    const pi_bits: u32 = @bitCast(pi_approx);
    std.debug.print("3.14 as bits: 0x{x}\n", .{pi_bits});

    // ใช้กับ struct และ integer (ขนาดต้องเท่ากัน!)
    const PackedColor = packed struct {
        r: u8,
        g: u8,
        b: u8,
        a: u8,
    };
    const color = PackedColor{ .r = 255, .g = 128, .b = 0, .a = 255 };
    const as_u32_2: u32 = @bitCast(color);
    std.debug.print("Color as u32: 0x{x}\n", .{as_u32_2});

    const back: PackedColor = @bitCast(as_u32_2);
    std.debug.print("Back to color: r={} g={} b={} a={}\n", .{back.r, back.g, back.b, back.a});
}
```

### @truncate - Explicit Bit Truncation

```zig
const std = @import("std");

pub fn main() void {
    // @truncate(val) - ตัด bits ส่วนสูงออก (explicit!)
    // ไม่มี panic, ตัดข้อมูลทิ้งโดยเจตนา

    const big: u32 = 0xDEADBEEF;
    const truncated: u8 = @truncate(big);  // เอาแค่ low 8 bits
    std.debug.print("0xDEADBEEF truncated to u8: 0x{x}\n", .{truncated});  // 0xEF

    const big2: u32 = 300;  // 0x0000012C
    const truncated2: u8 = @truncate(big2);  // 300 & 0xFF = 44
    std.debug.print("300 truncated to u8: {}\n", .{truncated2});  // 44

    // ต่างจาก @intCast ตรงที่:
    // @intCast(300 -> u8) = PANIC (ใน Debug mode) เพราะ 300 > 255
    // @truncate(300 -> u8) = 44 (ไม่ panic, ตัด bits ทิ้ง)

    // ใช้กับ signed integer
    const big_signed: i32 = -1;  // 0xFFFFFFFF
    const low_bits: u8 = @truncate(@as(u32, @bitCast(big_signed)));
    std.debug.print("-1 as i32, low u8: {}\n", .{low_bits});  // 255
}
```

### Summary: เมื่อไรใช้ Cast อะไร?

```zig
const std = @import("std");

// ตารางสรุปการเลือกใช้ Cast
//
// ต้องการทำ                      | ใช้
// ─────────────────────────────────────────────────────────────
// Integer widening (u8->u32)    | coercion อัตโนมัติ หรือ @as
// Integer narrowing (u32->u8)   | @intCast (panic ถ้า overflow)
// Integer truncate (u32->u8)    | @truncate (ตัด bits ทิ้ง)
// Signed<->Unsigned same bits   | @bitCast
// Signed<->Unsigned check val   | @intCast
// Float widening (f32->f64)     | coercion อัตโนมัติ หรือ @floatCast
// Float narrowing (f64->f32)    | @floatCast (อาจ lose precision)
// Int -> Float                  | @floatFromInt
// Float -> Int (truncate)       | @intFromFloat
// Pointer reinterpret           | @ptrCast
// Bit reinterpret               | @bitCast

pub fn main() void {
    // Best practices:
    std.debug.print("Cast best practices:\n", .{});

    // 1. ใช้ @intCast เมื่อต้องการ runtime safety
    const value: i32 = 100;
    const as_u8: u8 = @intCast(value);  // safe: 100 fits in u8
    std.debug.print("1. @intCast: {}\n", .{as_u8});

    // 2. ใช้ @truncate เมื่อต้องการ masking behavior
    const mask_target: u32 = 0xABCDEF01;
    const low_byte: u8 = @truncate(mask_target);
    std.debug.print("2. @truncate: 0x{x}\n", .{low_byte});

    // 3. ใช้ @bitCast เมื่อ bits ต้องเหมือนกัน
    const f: f32 = std.math.pi;
    const raw: u32 = @bitCast(f);
    std.debug.print("3. @bitCast pi bits: 0x{x}\n", .{raw});

    // 4. ใช้ @floatFromInt/@intFromFloat เมื่อ convert numeric types
    const int_val: i32 = 42;
    const float_val: f64 = @floatFromInt(int_val);
    std.debug.print("4. @floatFromInt: {d}\n", .{float_val});
}
```

---

## Step 236: Comptime Types (Type เป็น First-Class Value)

### Types ใน Zig เป็น Comptime Values

ใน Zig `type` เป็น type พิเศษที่ใช้ได้เฉพาะใน comptime context เท่านั้น แต่ทำให้เขียน generic code ได้อย่างทรงพลัง

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // Type เป็น value ที่สามารถ assign ได้
    // ============================================
    const MyInt: type = i32;
    const MyFloat: type = f64;
    const MyString: type = []const u8;

    var x: MyInt = 42;
    var y: MyFloat = 3.14;
    var z: MyString = "hello";

    std.debug.print("x={}, y={d}, z={s}\n", .{x, y, z});

    // ============================================
    // Type เป็น parameter ของ function (generic)
    // ============================================
    std.debug.print("Size of i32: {}\n", .{typeSize(i32)});
    std.debug.print("Size of f64: {}\n", .{typeSize(f64)});
    std.debug.print("Size of bool: {}\n", .{typeSize(bool)});

    // ============================================
    // Function ที่คืน type
    // ============================================
    const ArrayOf5 = makeArrayType(i32, 5);
    var arr: ArrayOf5 = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("arr[2] = {}\n", .{arr[2]});

    // ============================================
    // Type ใน comptime variable
    // ============================================
    comptime {
        const IntOrFloat = if (true) type else type;
        const T: IntOrFloat = i32;
        _ = T;
    }

    _ = x;
    _ = y;
    _ = z;
}

// Function รับ type เป็น parameter
fn typeSize(comptime T: type) usize {
    return @sizeOf(T);
}

// Function คืน type
fn makeArrayType(comptime T: type, comptime N: usize) type {
    return [N]T;
}
```

### Generic Data Structures ด้วย Comptime Types

```zig
const std = @import("std");

// Generic Stack implementation
fn Stack(comptime T: type) type {
    return struct {
        items: []T,
        len: usize,
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{
                .items = &[_]T{},
                .len = 0,
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            if (self.items.len > 0) {
                self.allocator.free(self.items);
            }
        }

        pub fn push(self: *Self, item: T) !void {
            const new_items = try self.allocator.realloc(
                if (self.items.len == 0) @as([]T, &[_]T{}) else self.items,
                self.len + 1
            );
            self.items = new_items;
            self.items[self.len] = item;
            self.len += 1;
        }

        pub fn pop(self: *Self) ?T {
            if (self.len == 0) return null;
            self.len -= 1;
            return self.items[self.len];
        }

        pub fn peek(self: *const Self) ?T {
            if (self.len == 0) return null;
            return self.items[self.len - 1];
        }

        pub fn isEmpty(self: *const Self) bool {
            return self.len == 0;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Stack ของ integers
    var int_stack = Stack(i32).init(allocator);
    defer int_stack.deinit();

    try int_stack.push(1);
    try int_stack.push(2);
    try int_stack.push(3);

    while (int_stack.pop()) |val| {
        std.debug.print("Popped: {}\n", .{val});
    }

    // Stack ของ strings
    var str_stack = Stack([]const u8).init(allocator);
    defer str_stack.deinit();

    try str_stack.push("hello");
    try str_stack.push("world");

    while (str_stack.pop()) |val| {
        std.debug.print("Popped: {s}\n", .{val});
    }
}
```

### Comptime Type Computation

```zig
const std = @import("std");

// สร้าง type ตาม condition ใน comptime
fn TypeForSize(comptime bits: u16) type {
    return switch (bits) {
        8 => i8,
        16 => i16,
        32 => i32,
        64 => i64,
        else => @compileError("Unsupported bit size"),
    };
}

// สร้าง optional wrapper
fn Optional(comptime T: type) type {
    return struct {
        value: T,
        has_value: bool,

        const Self = @This();

        pub fn some(val: T) Self {
            return Self{ .value = val, .has_value = true };
        }

        pub fn none() Self {
            return Self{ .value = undefined, .has_value = false };
        }

        pub fn get(self: Self) ?T {
            if (self.has_value) return self.value;
            return null;
        }
    };
}

// สร้าง tuple type
fn Pair(comptime A: type, comptime B: type) type {
    return struct {
        first: A,
        second: B,
    };
}

pub fn main() void {
    // ใช้ TypeForSize
    const Int8 = TypeForSize(8);
    const Int32 = TypeForSize(32);
    var small: Int8 = 42;
    var large: Int32 = 1000000;
    std.debug.print("Int8: {}, Int32: {}\n", .{small, large});

    // ใช้ Optional
    const maybe_int = Optional(i32).some(42);
    const no_int = Optional(i32).none();
    std.debug.print("maybe_int: {?}\n", .{maybe_int.get()});
    std.debug.print("no_int: {?}\n", .{no_int.get()});

    // ใช้ Pair
    const IntStringPair = Pair(i32, []const u8);
    const pair = IntStringPair{ .first = 42, .second = "hello" };
    std.debug.print("pair: ({}, {s})\n", .{pair.first, pair.second});

    _ = small;
    _ = large;
}
```

### @compileError สำหรับ Compile-time Validation

```zig
const std = @import("std");

// ตรวจสอบ type ใน comptime และให้ error ที่ชัดเจน
fn requireInteger(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int => {},  // OK
        else => @compileError("Expected integer type, got: " ++ @typeName(T)),
    }
}

fn requireNumeric(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {},
        else => @compileError("Expected numeric type, got: " ++ @typeName(T)),
    }
}

fn sum(comptime T: type, items: []const T) T {
    comptime requireNumeric(T);
    var total: T = 0;
    for (items) |item| {
        total += item;
    }
    return total;
}

pub fn main() void {
    const ints = [_]i32{ 1, 2, 3, 4, 5 };
    const floats = [_]f64{ 1.1, 2.2, 3.3 };

    std.debug.print("Sum of ints: {}\n", .{sum(i32, &ints)});
    std.debug.print("Sum of floats: {d}\n", .{sum(f64, &floats)});

    // ถ้า uncomment บรรทัดนี้จะได้ compile error:
    // const strings = [_][]const u8{ "a", "b" };
    // _ = sum([]const u8, &strings);
    // Error: Expected numeric type, got: []const u8
}
```

---

## Step 237: anytype Parameter

### anytype คืออะไร?

`anytype` เป็น parameter type พิเศษที่บอก Zig ว่า "รับ type อะไรก็ได้" และ type จะถูก infer ใน comptime

```zig
const std = @import("std");

// ============================================
// anytype พื้นฐาน
// ============================================
fn printValue(val: anytype) void {
    std.debug.print("Value: {any}\n", .{val});
}

// anytype ทำให้ function เป็น generic โดยไม่ต้องระบุ type
fn identity(x: anytype) @TypeOf(x) {
    return x;
}

// ============================================
// anytype กับหลาย parameter
// ============================================
fn add(a: anytype, b: anytype) @TypeOf(a + b) {
    return a + b;
}

pub fn main() void {
    printValue(@as(i32, 42));
    printValue(@as(f64, 3.14));
    printValue(true);
    printValue("hello");

    const x = identity(@as(i32, 42));
    const y = identity(@as(f64, 3.14));
    std.debug.print("identity(42): {}\n", .{x});
    std.debug.print("identity(3.14): {d}\n", .{y});

    std.debug.print("add(1, 2): {}\n", .{add(1, 2)});
    std.debug.print("add(1.5, 2.5): {d}\n", .{add(1.5, 2.5)});
}
```

### anytype กับ @TypeOf และ @typeInfo

```zig
const std = @import("std");

// Function ที่ทำงานต่างกันตาม type
fn process(val: anytype) void {
    const T = @TypeOf(val);
    const info = @typeInfo(T);

    switch (info) {
        .Int => {
            std.debug.print("Processing integer: {}\n", .{val});
            std.debug.print("  Range: {} to {}\n", .{
                std.math.minInt(T),
                std.math.maxInt(T),
            });
        },
        .Float => {
            std.debug.print("Processing float: {d}\n", .{val});
        },
        .Bool => {
            std.debug.print("Processing bool: {}\n", .{val});
        },
        .Pointer => |ptr| {
            if (ptr.size == .Slice and ptr.child == u8) {
                std.debug.print("Processing string: {s}\n", .{val});
            }
        },
        .Struct => {
            std.debug.print("Processing struct of type {}\n", .{T});
        },
        else => {
            std.debug.print("Processing unknown type: {}\n", .{T});
        },
    }
}

// anytype กับ pointer
fn increment(val: anytype) void {
    // val ต้องเป็น pointer เพื่อ modify
    val.* += 1;
}

pub fn main() void {
    process(@as(i32, 42));
    process(@as(f64, 3.14));
    process(true);
    process("Hello, Zig!");

    var x: i32 = 10;
    var y: f64 = 1.5;
    increment(&x);
    increment(&y);
    std.debug.print("x after increment: {}\n", .{x});
    std.debug.print("y after increment: {d}\n", .{y});
}
```

### anytype Limitations และ Best Practices

```zig
const std = @import("std");

// ============================================
// anytype limitations
// ============================================

// 1. ไม่สามารถใช้ anytype เป็น struct field ได้
// const Bad = struct {
//     value: anytype,  // COMPILE ERROR!
// };

// 2. ไม่สามารถใช้ anytype เป็น return type ได้โดยตรง
// fn bad() anytype { ... }  // COMPILE ERROR!
// แต่สามารถใช้ @TypeOf ได้:
fn good(val: anytype) @TypeOf(val) {
    return val;
}

// 3. anytype ทุก instantiation สร้าง function ใหม่ (code bloat)
fn printN(val: anytype, n: usize) void {
    for (0..n) |_| {
        std.debug.print("{any} ", .{val});
    }
    std.debug.print("\n", .{});
}

// ============================================
// Best practice: ใช้ @typeInfo เพื่อ validate
// ============================================
fn safeAdd(a: anytype, b: anytype) @TypeOf(a, b) {
    const T = @TypeOf(a, b);
    // ตรวจสอบว่าเป็น numeric type
    switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {},
        else => @compileError("safeAdd requires numeric types"),
    }
    return a + b;
}

// ============================================
// anytype กับ comptime parameter
// ============================================
fn createArray(comptime n: usize, val: anytype) [@TypeOf(n)]@TypeOf(val) {
    _ = n;
    const T = @TypeOf(val);
    var result: [n]T = undefined;
    for (&result) |*item| {
        item.* = val;
    }
    return result;
}

pub fn main() void {
    std.debug.print("good(42): {}\n", .{good(@as(i32, 42))});

    printN(@as(i32, 42), 3);
    printN("hello", 2);

    std.debug.print("safeAdd(1, 2): {}\n", .{safeAdd(1, 2)});
    std.debug.print("safeAdd(1.5, 2.5): {d}\n", .{safeAdd(1.5, 2.5)});

    const arr = createArray(5, @as(i32, 0));
    std.debug.print("createArray: {any}\n", .{arr});
}
```

---

## Step 238: Type Aliases

### Type Aliases ใน Zig

```zig
const std = @import("std");

// ============================================
// Type Alias พื้นฐาน
// ============================================
const Integer = i32;     // Integer เป็นชื่ออื่นของ i32
const Text = []const u8; // Text เป็นชื่ออื่นของ []const u8
const Byte = u8;         // Byte เป็นชื่ออื่นของ u8

pub fn main() void {
    const x: Integer = 42;
    const greeting: Text = "สวัสดี";
    const b: Byte = 0xFF;

    std.debug.print("x={}, greeting={s}, b={x}\n", .{x, greeting, b});

    // Type alias ไม่สร้าง "distinct type" - เหมือนกันทุกอย่าง
    std.debug.print("Integer == i32: {}\n", .{Integer == i32});      // true
    std.debug.print("Text == []const u8: {}\n", .{Text == []const u8}); // true
}
```

### Type Aliases สำหรับ Readability

```zig
const std = @import("std");

// ============================================
// ใช้ Type Alias เพื่อ readability
// ============================================

// แทนที่จะเขียน fn(i32, i32) i32 ทุกที่
const BinaryIntFn = fn(i32, i32) i32;

// แทนที่จะเขียน *const fn(anytype) void ทุกที่
const PrintFn = *const fn([]const u8) void;

// Type aliases สำหรับ complex types
const Matrix4x4 = [4][4]f32;
const Vec3 = struct { x: f32, y: f32, z: f32 };
const Color32 = packed struct { r: u8, g: u8, b: u8, a: u8 };

// ============================================
// Type aliases ใน namespace
// ============================================
const Sizes = struct {
    const Small = i8;
    const Medium = i32;
    const Large = i64;
    const Huge = i128;
};

// ============================================
// Type aliases กับ Generic types
// ============================================
fn TypedList(comptime T: type) type {
    return std.ArrayList(T);
}
const IntList = TypedList(i32);
const StringList = TypedList([]const u8);

pub fn main() !void {
    // ใช้ BinaryIntFn
    const add: BinaryIntFn = struct {
        fn f(a: i32, b: i32) i32 {
            return a + b;
        }
    }.f;
    std.debug.print("add(3,4) = {}\n", .{add(3, 4)});

    // ใช้ Matrix4x4
    var identity: Matrix4x4 = undefined;
    for (0..4) |i| {
        for (0..4) |j| {
            identity[i][j] = if (i == j) 1.0 else 0.0;
        }
    }
    std.debug.print("Identity[0][0] = {d}\n", .{identity[0][0]});

    // ใช้ Sizes
    const small: Sizes.Small = 42;
    const medium: Sizes.Medium = 100000;
    std.debug.print("small={}, medium={}\n", .{small, medium});

    // ใช้ IntList
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var list = IntList.init(gpa.allocator());
    defer list.deinit();
    try list.append(1);
    try list.append(2);
    try list.append(3);
    std.debug.print("IntList: {any}\n", .{list.items});
}
```

### Type Aliases กับ Self-referential Types

```zig
const std = @import("std");

// Linked List ที่ใช้ type alias
const Node = struct {
    value: i32,
    next: ?*Node = null,  // self-referential ผ่าน pointer

    const Self = @This();  // Self เป็น alias สำหรับ Node

    pub fn append(self: *Self, allocator: std.mem.Allocator, val: i32) !*Self {
        const new_node = try allocator.create(Self);
        new_node.* = Self{ .value = val, .next = null };

        var current = self;
        while (current.next) |next| {
            current = next;
        }
        current.next = new_node;
        return new_node;
    }

    pub fn print(self: *const Self) void {
        var current: ?*const Self = self;
        while (current) |node| {
            std.debug.print("{} ", .{node.value});
            current = node.next;
        }
        std.debug.print("\n", .{});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var head = Node{ .value = 1 };
    _ = try head.append(allocator, 2);
    _ = try head.append(allocator, 3);
    _ = try head.append(allocator, 4);

    head.print();
}
```

---

## Step 239: Opaque Types

### Opaque Types คืออะไร?

Opaque type คือ type ที่ซ่อน implementation detail ออกจาก user ของ type นั้น เหมาะสำหรับ:
1. C interop (เชื่อมกับ C library)
2. Encapsulation ที่แข็งแกร่ง
3. ABI stability

```zig
const std = @import("std");

// ============================================
// Opaque type พื้นฐาน
// ============================================
const OpaqueHandle = opaque {};

// OpaqueHandle มีขนาด = 0 bytes ใน Zig
// แต่ pointer ไปหามันมีขนาดปกติ

pub fn main() void {
    std.debug.print("Size of OpaqueHandle: {}\n", .{@sizeOf(OpaqueHandle)});
    std.debug.print("Size of *OpaqueHandle: {}\n", .{@sizeOf(*OpaqueHandle)});
}
```

### Opaque Types สำหรับ C Interop

```zig
// ตัวอย่าง C interop ด้วย opaque type

// สมมุติว่ามี C library ที่มี:
// typedef struct SDL_Window SDL_Window;
// SDL_Window* SDL_CreateWindow(...);
// void SDL_DestroyWindow(SDL_Window*);

// ใน Zig เราสร้าง opaque type สำหรับ C struct:
const SDL_Window = opaque {};
const SDL_Renderer = opaque {};

// ประกาศ C functions (extern)
// extern fn SDL_CreateWindow(title: [*:0]const u8, ...) ?*SDL_Window;
// extern fn SDL_DestroyWindow(window: *SDL_Window) void;

// Wrapper ที่ปลอดภัยกว่า
const Window = struct {
    handle: *SDL_Window,

    // ใช้งานผ่าน handle โดยไม่ expose internals
    pub fn getTitle(self: *const Window) []const u8 {
        _ = self;
        return "Fake Window";  // ใน reality จะ call SDL function
    }
};
```

### Opaque Types สำหรับ Encapsulation

```zig
const std = @import("std");

// ============================================
// ใช้ opaque เพื่อซ่อน implementation
// ============================================

// Public interface (ใน header/pub API)
pub const DatabaseHandle = opaque {};

// Private implementation
const DatabaseImpl = struct {
    connection_string: []const u8,
    is_connected: bool,
    query_count: u64,
};

// "Constructor" ที่คืน pointer ไป opaque type
pub fn openDatabase(connection_str: []const u8) !*DatabaseHandle {
    const allocator = std.heap.page_allocator;
    const impl = try allocator.create(DatabaseImpl);
    impl.* = .{
        .connection_string = connection_str,
        .is_connected = true,
        .query_count = 0,
    };
    // ใช้ @ptrCast เพื่อแปลงจาก *DatabaseImpl เป็น *DatabaseHandle
    return @ptrCast(impl);
}

pub fn closeDatabase(handle: *DatabaseHandle) void {
    const impl: *DatabaseImpl = @ptrCast(@alignCast(handle));
    std.heap.page_allocator.destroy(impl);
}

pub fn query(handle: *DatabaseHandle, q: []const u8) void {
    const impl: *DatabaseImpl = @ptrCast(@alignCast(handle));
    impl.query_count += 1;
    std.debug.print("Query #{}: {s} (on {})\n", .{
        impl.query_count,
        q,
        impl.connection_string,
    });
}

pub fn main() !void {
    const db = try openDatabase("postgresql://localhost/mydb");
    defer closeDatabase(db);

    query(db, "SELECT * FROM users");
    query(db, "INSERT INTO logs VALUES (...)");

    // User ของ DatabaseHandle ไม่เห็น implementation details
    // ไม่สามารถ access .connection_string หรือ .query_count โดยตรง
}
```

### anyopaque - Zig's void*

```zig
const std = @import("std");

// anyopaque คือ type ที่ใช้แทน void* ใน C

// ============================================
// ใช้ anyopaque เพื่อ generic callback
// ============================================
const Callback = struct {
    ctx: *anyopaque,
    func: *const fn(*anyopaque) void,

    pub fn call(self: Callback) void {
        self.func(self.ctx);
    }
};

const PrintContext = struct {
    message: []const u8,
};

fn printCallback(ctx: *anyopaque) void {
    const print_ctx: *PrintContext = @ptrCast(@alignCast(ctx));
    std.debug.print("{s}\n", .{print_ctx.message});
}

const CountContext = struct {
    count: u32,
};

fn countCallback(ctx: *anyopaque) void {
    const count_ctx: *CountContext = @ptrCast(@alignCast(ctx));
    count_ctx.count += 1;
    std.debug.print("Count: {}\n", .{count_ctx.count});
}

pub fn main() void {
    var print_ctx = PrintContext{ .message = "Hello from callback!" };
    var count_ctx = CountContext{ .count = 0 };

    const callbacks = [_]Callback{
        Callback{ .ctx = &print_ctx, .func = printCallback },
        Callback{ .ctx = &count_ctx, .func = countCallback },
        Callback{ .ctx = &count_ctx, .func = countCallback },
        Callback{ .ctx = &print_ctx, .func = printCallback },
    };

    for (callbacks) |cb| {
        cb.call();
    }
}
```

---

## Step 240: Type System เปรียบเทียบกับภาษาอื่น

### Zig vs C

```zig
// ============================================
// C Type System ปัญหา และ Zig แก้อย่างไร
// ============================================

// ปัญหาที่ 1: Implicit conversions ที่อันตรายใน C
// C code: int x = 256; char c = x;  // undefined behavior!
// Zig: ต้องใช้ @intCast หรือ @truncate - explicit!

// ปัญหาที่ 2: NULL pointer ใน C
// C code: int* ptr = NULL; *ptr = 42;  // crash!
// Zig: ใช้ ?*i32 - compiler บังคับให้ unwrap ก่อน

// ปัญหาที่ 3: Array decay ใน C
// C code: void foo(int arr[]) -> arr เป็น pointer!
// Zig: []i32 เป็น slice ที่มี length - ปลอดภัยกว่า

// ปัญหาที่ 4: Signed overflow ใน C
// C code: int x = INT_MAX + 1;  // undefined behavior!
// Zig: +% (wrapping), +| (saturating), หรือ @addWithOverflow

// ปัญหาที่ 5: void* ไม่มี type information
// C code: void* ptr = ...;  // ไม่รู้ว่า point ไปอะไร
// Zig: anyopaque แต่ต้องใช้ @ptrCast อย่าง explicit
```

### Zig vs Rust

```
การเปรียบเทียบกับ Rust:
──────────────────────────────────────────────────────────
Feature              | Rust              | Zig
──────────────────────────────────────────────────────────
Generic             | <T> + Traits       | comptime T: type
Type inference      | สมบูรณ์            | partial (local vars)
Null safety         | Option<T>          | ?T
Error handling      | Result<T, E>       | E!T
Type aliases        | type Foo = Bar;    | const Foo = Bar;
Opaque types        | Box<dyn Trait>     | opaque {}
Integer overflow    | panic (debug)      | @addWithOverflow
Implicit cast       | ไม่มีเลย           | มีบางส่วน
Compile-time eval   | const fn           | comptime
Dependent types     | ไม่มี             | comptime (partial)
──────────────────────────────────────────────────────────
```

### Zig vs TypeScript

```
การเปรียบเทียบกับ TypeScript:
──────────────────────────────────────────────────────────
Feature              | TypeScript              | Zig
──────────────────────────────────────────────────────────
Type system          | Structural (duck typing)| Nominal
Runtime types        | มี (typeof, instanceof) | ไม่มี (comptime only)
Union types          | string | number         | tagged union
Generics             | <T extends Interface>   | comptime T: type
Optional             | T | null | undefined    | ?T
Error types          | throw/catch             | error!T
Type narrowing       | if (typeof x === 'string')| switch (@typeInfo)
Template types       | `Record<K, V>`          | comptime hash maps
Type erasure         | ที่ runtime             | ที่ comptime
──────────────────────────────────────────────────────────
```

### Zig vs Go

```zig
// การเปรียบเทียบกับ Go:
//
// Go: interface{} / any
// Zig: anytype (comptime), anyopaque (runtime pointer)
//
// Go: generics ใช้ [T any] หรือ [T Constraint]
// Zig: comptime T: type (ยืดหยุ่นกว่า แต่ไม่มี constraint system)
//
// Go: nil pointer
// Zig: ?*T (compiler บังคับ handle nil)
//
// Go: int overflow silently wraps
// Zig: panic ใน debug, ต้องใช้ +% สำหรับ explicit wrap
//
// Go: reflect package สำหรับ runtime type info
// Zig: @typeInfo ใน comptime (เร็วกว่า, ไม่มี overhead)
```

### Unique Features ของ Zig's Type System

```zig
const std = @import("std");

pub fn main() void {
    // ============================================
    // Feature 1: Arbitrary-width integers
    // ============================================
    const u3_val: u3 = 7;   // 3-bit integer (0-7)
    const i5_val: i5 = -16; // 5-bit signed (-16 to 15)
    std.debug.print("u3: {}, i5: {}\n", .{u3_val, i5_val});

    // ============================================
    // Feature 2: Comptime-known type parameters
    // ============================================
    // ทำให้ code generation แม่นยำกว่า C++ templates
    // และ checked ที่ comptime ทั้งหมด

    // ============================================
    // Feature 3: Error unions เป็น first-class type
    // ============================================
    // ไม่ใช่แค่ convention (เหมือน Go)
    // และไม่ต้องใช้ exceptions (เหมือน C++)

    // ============================================
    // Feature 4: Optional types ที่ explicit
    // ============================================
    // ต่างจาก Java/C# ที่ null เป็น implicit behavior ทุกที่

    // ============================================
    // Feature 5: Type เป็น first-class value
    // ============================================
    const types = [_]type{ i32, f64, bool, u8 };
    for (types) |T| {
        std.debug.print("Size of {}: {}\n", .{T, @sizeOf(T)});
    }

    // ============================================
    // Feature 6: Peer type resolution
    // ============================================
    // ไม่ต้องใช้ explicit cast ในทุก if/switch

    // ============================================
    // Feature 7: Packed structs และ integer layout
    // ============================================
    const Packed = packed struct {
        a: u4,
        b: u4,
    };
    std.debug.print("Packed size: {}\n", .{@sizeOf(Packed)});  // 1 byte
}
```

### Type System Design Principles

```
Zig Type System Design Principles:
═══════════════════════════════════════════════════════════

1. NO HIDDEN CONTROL FLOW
   - ไม่มี implicit exceptions
   - ไม่มี implicit allocations
   - Cast ทุกอย่าง explicit

2. NO HIDDEN MEMORY ALLOCATIONS
   - ต้องส่ง allocator เข้าไปใน function เสมอ
   - Type system ไม่ซ่อน heap allocation

3. COMPILE-TIME OVER RUNTIME
   - Generic ทำงานที่ comptime
   - Type information อยู่ที่ comptime
   - Zero runtime overhead สำหรับ generic code

4. EXPLICIT OVER IMPLICIT
   - Implicit coercions มีเฉพาะเมื่อ "safe" เท่านั้น
   - Type narrowing ต้อง explicit cast
   - Error handling ต้องจัดการทุก error path

5. ZERO-COST ABSTRACTIONS
   - Generic code compile เหมือน hand-written code
   - Optional type ไม่มี overhead (เป็น tagged union ที่ optimized)
   - Error union optimized เป็น integer ใน simple cases
```

---

## Type System Gotchas และข้อควรระวัง

### Gotcha 1: @intCast กับ Negative Numbers

```zig
const std = @import("std");

pub fn main() void {
    // ระวัง: @intCast จาก signed ไป unsigned กับ negative number
    const neg: i32 = -1;

    // ใน Debug mode: panic
    // ใน ReleaseFast: undefined behavior
    // const wrong: u32 = @intCast(neg);

    // วิธีที่ถูกต้อง: ตรวจสอบก่อน
    if (neg >= 0) {
        const safe: u32 = @intCast(neg);
        std.debug.print("safe: {}\n", .{safe});
    }

    // หรือใช้ @bitCast ถ้าต้องการ reinterpret bits
    const reinterpreted: u32 = @bitCast(neg);
    std.debug.print("reinterpreted: {} (0xFFFFFFFF = {})\n", .{reinterpreted, reinterpreted == 0xFFFFFFFF});
}
```

### Gotcha 2: @intFromFloat กับ Infinity/NaN

```zig
const std = @import("std");

pub fn main() void {
    // ระวัง: @intFromFloat กับ special float values

    const inf = std.math.inf(f64);
    const nan = std.math.nan(f64);
    const huge: f64 = 1e308;

    // ตรวจสอบก่อนแปลง
    if (std.math.isFinite(huge) and huge >= std.math.minInt(i64) and huge <= std.math.maxInt(i64)) {
        const as_int: i64 = @intFromFloat(huge);
        std.debug.print("huge as i64: {}\n", .{as_int});
    } else {
        std.debug.print("Cannot convert {} to i64\n", .{huge});
    }

    // ตรวจสอบ NaN และ Infinity
    std.debug.print("inf is finite: {}\n", .{std.math.isFinite(inf)});
    std.debug.print("nan is nan: {}\n", .{std.math.isNan(nan)});
}
```

### Gotcha 3: Type Coercion ใน Array Literals

```zig
const std = @import("std");

pub fn main() void {
    // Array literal กับ peer type resolution

    // ทุก element ต้องมี type เดียวกัน (หรือ coerce ได้)
    const arr1 = [_]i32{ 1, 2, 3 };  // OK: comptime_int -> i32

    // ระวัง: นี้จะไม่ compile ถ้า types ไม่ compatible
    // const arr2 = [_]i32{ 1, 2.0, 3 };  // ERROR: f64 ไม่ coerce เป็น i32

    // ต้องใช้ @intFromFloat
    const arr3 = [_]i32{ 1, @intFromFloat(2.0), 3 };

    std.debug.print("arr1: {any}\n", .{arr1});
    std.debug.print("arr3: {any}\n", .{arr3});
}
```

### Gotcha 4: @ptrCast กับ Alignment

```zig
const std = @import("std");

pub fn main() void {
    // ระวัง: @ptrCast ไม่ตรวจสอบ alignment
    // ถ้า alignment ไม่ถูกต้องจะได้ undefined behavior

    var bytes: [8]u8 align(8) = [_]u8{ 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08 };

    // ต้องใช้ @alignCast เมื่อ cast ไปยัง type ที่ต้องการ alignment สูงกว่า
    const u64_ptr: *u64 = @ptrCast(@alignCast(&bytes));
    std.debug.print("u64 view: 0x{x}\n", .{u64_ptr.*});

    // ถ้าลืม @alignCast:
    // const bad_ptr: *u64 = @ptrCast(&bytes);  // Might work by luck, but could crash
}
```

### Gotcha 5: comptime_int Overflow

```zig
const std = @import("std");

pub fn main() void {
    // comptime_int มีความแม่นยำไม่จำกัด
    // แต่เมื่อ coerce ไปยัง runtime type อาจ overflow

    const big = 1000000;

    // comptime_int 1000000 ไม่พอดีกับ u8 (max 255)
    // const too_small: u8 = big;  // COMPILE ERROR: value 1000000 cannot be stored in type u8

    // ถ้าต้องการ: ใช้ type ที่ใหญ่พอ
    const just_right: u32 = big;
    std.debug.print("big as u32: {}\n", .{just_right});
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Type Inspector

เขียน function `inspectType` ที่รับ `anytype` และพิมพ์ข้อมูล type อย่างละเอียด:

```zig
const std = @import("std");

fn inspectType(val: anytype) void {
    const T = @TypeOf(val);
    const info = @typeInfo(T);

    std.debug.print("=== Type Inspection ===\n", .{});
    std.debug.print("Type name: {s}\n", .{@typeName(T)});
    std.debug.print("Size: {} bytes\n", .{@sizeOf(T)});
    std.debug.print("Alignment: {} bytes\n", .{@alignOf(T)});

    switch (info) {
        .Int => |int_info| {
            std.debug.print("Kind: Integer\n", .{});
            std.debug.print("Bits: {}\n", .{int_info.bits});
            std.debug.print("Signed: {}\n", .{int_info.signedness == .signed});
        },
        .Float => |float_info| {
            std.debug.print("Kind: Float\n", .{});
            std.debug.print("Bits: {}\n", .{float_info.bits});
        },
        .Bool => {
            std.debug.print("Kind: Boolean\n", .{});
        },
        .Struct => |struct_info| {
            std.debug.print("Kind: Struct\n", .{});
            std.debug.print("Fields: {}\n", .{struct_info.fields.len});
            for (struct_info.fields) |field| {
                std.debug.print("  - {s}: {s}\n", .{field.name, @typeName(field.type)});
            }
        },
        .Enum => |enum_info| {
            std.debug.print("Kind: Enum\n", .{});
            std.debug.print("Fields: {}\n", .{enum_info.fields.len});
        },
        .Optional => |opt_info| {
            std.debug.print("Kind: Optional of {s}\n", .{@typeName(opt_info.child)});
        },
        .Pointer => |ptr_info| {
            std.debug.print("Kind: Pointer to {s}\n", .{@typeName(ptr_info.child)});
        },
        else => {
            std.debug.print("Kind: Other\n", .{});
        },
    }
    std.debug.print("\n", .{});
}

const TestStruct = struct {
    x: i32,
    y: f64,
    name: []const u8,
};

const TestEnum = enum { a, b, c };

pub fn main() void {
    inspectType(@as(i32, 42));
    inspectType(@as(f64, 3.14));
    inspectType(TestStruct{ .x = 1, .y = 2.0, .name = "test" });
    inspectType(TestEnum.a);
    inspectType(@as(?i32, null));
}
```

### แบบฝึกหัดที่ 2: Safe Integer Conversion

เขียน function `safeCast` ที่แปลง integer type โดยปลอดภัย:

```zig
const std = @import("std");

const CastError = error{
    ValueTooLarge,
    ValueTooSmall,
    NegativeToUnsigned,
};

fn safeCast(comptime To: type, val: anytype) CastError!To {
    const From = @TypeOf(val);
    const from_info = @typeInfo(From);
    const to_info = @typeInfo(To);

    // ตรวจสอบว่าทั้งคู่เป็น integer
    if (from_info != .Int or to_info != .Int) {
        @compileError("safeCast only works with integer types");
    }

    const to_int = to_info.Int;

    // ตรวจสอบ negative ไปยัง unsigned
    if (to_int.signedness == .unsigned and val < 0) {
        return CastError.NegativeToUnsigned;
    }

    // ตรวจสอบ range
    const min_val = std.math.minInt(To);
    const max_val = std.math.maxInt(To);

    if (val < min_val) return CastError.ValueTooSmall;
    if (val > max_val) return CastError.ValueTooLarge;

    return @intCast(val);
}

pub fn main() void {
    // Test cases
    const cases = .{
        .{ i32, @as(i64, 42) },
        .{ u8, @as(i32, 100) },
        .{ u8, @as(i32, 300) },    // too large
        .{ u8, @as(i32, -1) },     // negative to unsigned
        .{ i8, @as(i32, -100) },   // fits in i8
        .{ i8, @as(i32, 200) },    // too large for i8
    };

    inline for (cases) |c| {
        const To = c[0];
        const from = c[1];
        const result = safeCast(To, from);
        if (result) |val| {
            std.debug.print("safeCast({s}, {}) = {}\n", .{@typeName(To), from, val});
        } else |err| {
            std.debug.print("safeCast({s}, {}) = Error: {}\n", .{@typeName(To), from, err});
        }
    }
}
```

### แบบฝึกหัดที่ 3: Generic Min/Max

เขียน generic `min` และ `max` ที่รองรับ integers, floats, และ strings:

```zig
const std = @import("std");

fn minVal(a: anytype, b: anytype) @TypeOf(a, b) {
    const T = @TypeOf(a, b);
    const info = @typeInfo(T);
    switch (info) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {
            return if (a < b) a else b;
        },
        .Pointer => |ptr| {
            if (ptr.size == .Slice and ptr.child == u8) {
                // String comparison
                const order = std.mem.order(u8, a, b);
                return if (order == .lt) a else b;
            }
            @compileError("minVal: unsupported pointer type");
        },
        else => @compileError("minVal: unsupported type " ++ @typeName(T)),
    }
}

fn maxVal(a: anytype, b: anytype) @TypeOf(a, b) {
    const T = @TypeOf(a, b);
    const info = @typeInfo(T);
    switch (info) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {
            return if (a > b) a else b;
        },
        .Pointer => |ptr| {
            if (ptr.size == .Slice and ptr.child == u8) {
                const order = std.mem.order(u8, a, b);
                return if (order == .gt) a else b;
            }
            @compileError("maxVal: unsupported pointer type");
        },
        else => @compileError("maxVal: unsupported type " ++ @typeName(T)),
    }
}

pub fn main() void {
    std.debug.print("min(3, 7) = {}\n", .{minVal(3, 7)});
    std.debug.print("min(3.14, 2.71) = {d}\n", .{minVal(3.14, 2.71)});
    std.debug.print("min(\"apple\", \"banana\") = {s}\n", .{minVal("apple", "banana")});

    std.debug.print("max(3, 7) = {}\n", .{maxVal(3, 7)});
    std.debug.print("max(3.14, 2.71) = {d}\n", .{maxVal(3.14, 2.71)});
    std.debug.print("max(\"apple\", \"banana\") = {s}\n", .{maxVal("apple", "banana")});
}
```

### แบบฝึกหัดที่ 4: Type-Safe Builder Pattern

```zig
const std = @import("std");

// Builder pattern ที่ใช้ comptime type tracking
fn Builder(comptime T: type) type {
    return struct {
        data: T,

        const Self = @This();

        pub fn init() Self {
            return Self{ .data = std.mem.zeroes(T) };
        }

        pub fn set(self: Self, comptime field: []const u8, value: anytype) Self {
            var new_self = self;
            @field(new_self.data, field) = value;
            return new_self;
        }

        pub fn build(self: Self) T {
            return self.data;
        }
    };
}

const Person = struct {
    name: []const u8,
    age: u32,
    email: []const u8,
};

pub fn main() void {
    const person = Builder(Person).init()
        .set("name", "สมชาย")
        .set("age", @as(u32, 30))
        .set("email", "somchai@example.com")
        .build();

    std.debug.print("Person: name={s}, age={}, email={s}\n", .{
        person.name,
        person.age,
        person.email,
    });
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 24

| Step | หัวข้อ | Key Takeaway |
|------|--------|--------------|
| 231 | Type System Overview | Zig มี type categories หลากหลาย ทั้ง primitives, composites, และ special types |
| 232 | @TypeOf | ใช้ดึง type ของ expression ที่ comptime สำหรับ generic programming |
| 233 | Type Coercion | Zig มี implicit coercion น้อย แต่ที่มีนั้น safe เสมอ |
| 234 | Peer Type Resolution | Compiler หา common type สำหรับ if/switch branches |
| 235 | Type Casting | มี cast function หลายระดับ ตั้งแต่ safe ไปถึงอันตราย |
| 236 | Comptime Types | type เป็น first-class value ใช้สร้าง generic code |
| 237 | anytype | รับ type อะไรก็ได้ใน comptime, ใช้กับ @TypeOf |
| 238 | Type Aliases | const Alias = T; ทำให้โค้ดอ่านง่ายขึ้น |
| 239 | Opaque Types | ซ่อน implementation, ใช้กับ C interop |
| 240 | Type System Comparison | Zig เด่นที่ comptime, arbitrary-width ints, explicit casting |

### Cheat Sheet: Cast Functions

```
Cast Function    | เมื่อใช้                           | Safety
─────────────────────────────────────────────────────────────────
@as(T, v)        | Valid coercions                    | Compile-time safe
@intCast(v)      | Int -> narrower/different int      | Runtime check (Debug)
@floatCast(v)    | Float -> different float           | May lose precision
@floatFromInt(v) | Int -> Float                       | May lose precision
@intFromFloat(v) | Float -> Int (truncate)            | May panic/UB
@ptrCast(v)      | Pointer type change                | No check!
@bitCast(v)      | Reinterpret bits                   | No check!
@truncate(v)     | Explicit bit truncation            | Explicit UB intent
@alignCast(v)    | Assert/adjust pointer alignment    | Runtime check (Debug)
```

### Best Practices สรุป

```
1. ใช้ implicit coercion เมื่อทำได้ (widening, optional wrapping)
2. ใช้ @as เมื่อต้องการ explicit แต่ safe coercion
3. ใช้ @intCast เมื่อ narrow integers (ได้ runtime check ฟรี)
4. ใช้ @truncate เมื่อตั้งใจ drop high bits (explicit)
5. ใช้ @bitCast เมื่อต้องการ reinterpret bit pattern
6. ระวัง @ptrCast - ไม่มี type safety
7. ใช้ anytype กับ @typeInfo เพื่อ type-safe generics
8. ใช้ type เป็น first-class value แทน macros
9. ใช้ opaque types สำหรับ C interop และ encapsulation
```

---

## ก้าวต่อไป

หลังจากเข้าใจ Type System แล้ว บทเรียนต่อไปที่ควรศึกษา:

- **Part 25: Memory Management** - การจัดการ memory อย่างชัดเจน
- **Part 26: Allocators** - Allocator interface และ implementation
- **Part 27: Error Handling Advanced** - Error sets, error traces
- **Part 28: Comptime Deep Dive** - comptime programming แบบลึก
- **Part 29: Build System** - build.zig และ build system

---

*จบ Part 24: Type System ใน Zig (Steps 231-240)*

> **หมายเหตุ:** Zig ยังอยู่ในระหว่างการพัฒนา บาง API อาจเปลี่ยนแปลงได้ในเวอร์ชันอนาคต เอกสารนี้เขียนสำหรับ Zig 0.12.x/0.13.x
