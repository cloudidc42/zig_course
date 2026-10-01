# Part 21: Comptime พื้นฐานใน Zig (Steps 201-210)

> **ระดับ:** ขั้นกลาง-สูง  
> **เวลาโดยประมาณ:** 3-4 ชั่วโมง  
> **ความรู้ที่ต้องมีก่อน:** Types, Functions, Structs, Interfaces พื้นฐาน

---

## บทนำ: ทำไม Comptime ถึงสำคัญ?

หนึ่งในฟีเจอร์ที่ทำให้ Zig โดดเด่นที่สุดคือระบบ **comptime** ซึ่งย่อมาจาก "compile-time" หรือ "เวลาคอมไพล์"

ใน C เราใช้ macro preprocessor เพื่อสร้าง code ที่ยืดหยุ่น แต่ macro นั้น fragile และ debug ยาก ใน C++ เราใช้ template ซึ่งดีกว่า แต่ syntax ซับซ้อนมาก และ error message อ่านแทบไม่ออก

Zig แก้ปัญหานี้ด้วยแนวคิดที่เรียบง่ายแต่ทรงพลัง: **ทำให้ Zig code สามารถรันได้ที่ compile-time โดยใช้ภาษา Zig เดียวกัน** ไม่มี DSL พิเศษ ไม่มี template syntax แปลกๆ

ผลลัพธ์คือ:
- **Zero-cost abstractions** — abstraction ที่ไม่มี runtime overhead
- **Generics ที่ใช้งานง่าย** — ผ่าน `comptime` parameters
- **Code generation ที่ type-safe** — ไม่เหมือน C macro
- **Error message ที่เข้าใจได้** — เพราะมันคือ Zig code ธรรมดา

---

## Step 201: comptime คืออะไร? Compile-Time Execution

### แนวคิดพื้นฐาน

ปกติโปรแกรมทำงานใน 2 ช่วง:
1. **Compile-time** — ตอนที่ compiler แปลง source code เป็น binary
2. **Runtime** — ตอนที่ผู้ใช้รันโปรแกรม

ใน Zig คำว่า `comptime` บอก compiler ว่า "code ส่วนนี้ต้องรู้ผลลัพธ์ตอน compile แล้ว"

```zig
const std = @import("std");

pub fn main() void {
    // นี่คือ runtime value — ไม่รู้ค่าจนกว่าจะรันโปรแกรม
    var x: i32 = 42;
    x += 1;
    std.debug.print("runtime: {}\n", .{x});

    // นี่คือ comptime value — compiler รู้ค่านี้ตอน compile
    const comptime_val = comptime blk: {
        var sum: i32 = 0;
        var i: i32 = 1;
        while (i <= 100) : (i += 1) {
            sum += i;
        }
        break :blk sum;
    };
    // compiler คำนวณ 1+2+...+100 = 5050 แล้วฝัง literal ไว้ใน binary
    std.debug.print("comptime: {}\n", .{comptime_val});
}
```

### ความแตกต่างที่สำคัญ

| ลักษณะ | Runtime | Comptime |
|--------|---------|---------|
| ค่าของตัวแปร | รู้ตอนรัน | รู้ตอน compile |
| Types | fixed | สามารถเป็น parameter ได้ |
| Loops | รันทุกครั้ง | unrolled โดย compiler |
| ผลกระทบต่อ binary | อยู่ใน binary | ฝังเป็น literal หรือหายไป |

### ค่าที่ต้องเป็น comptime

มีบางอย่างใน Zig ที่ **ต้อง** เป็น comptime:

```zig
const std = @import("std");

pub fn main() void {
    // 1. Array size ต้องเป็น comptime
    const size = 10;
    var arr: [size]i32 = undefined;
    // arr[0] = 0;  // (ใช้งานได้จริง แต่ละเว้นการกำหนดค่าในตัวอย่างนี้)
    _ = arr;

    // 2. Type ต้องเป็น comptime
    const T = i32;
    var val: T = 5;
    _ = val;

    // 3. Switch บน type ต้องเป็น comptime
    const MyType = i64;
    const info = @typeInfo(MyType);
    switch (info) {
        .Int => std.debug.print("เป็น integer\n", .{}),
        .Float => std.debug.print("เป็น float\n", .{}),
        else => std.debug.print("อื่นๆ\n", .{}),
    }
}
```

### เปรียบเทียบกับ C Macro

```c
// C: macro ไม่มี type safety
#define MAX(a, b) ((a) > (b) ? (a) : (b))
// MAX(1, 2.0) ทำงานได้แต่อาจมี implicit conversion
// MAX(i++, j++) มี side effects ซ้อน!
```

```zig
// Zig: comptime function มี type safety เต็มรูปแบบ
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
// max(i32, 1, 2) — type-safe, ไม่มี gotchas
```

---

## Step 202: Comptime Variables และ Expressions

### Comptime Variables

ตัวแปร `comptime` ต้องมีค่าที่รู้ได้ตอน compile:

```zig
const std = @import("std");

pub fn main() void {
    // comptime variable — ค่าต้องรู้ตอน compile
    comptime var count: usize = 0;
    comptime {
        count += 1;
        count += 1;
        count += 1;
    }
    // count ตอนนี้คือ 3 ที่ compile-time

    std.debug.print("count = {}\n", .{count});

    // const ที่ top-level เป็น comptime โดยอัตโนมัติ
    const PI = 3.14159265358979;
    const CIRCLE_AREA = PI * 5.0 * 5.0;  // คำนวณตอน compile
    std.debug.print("พื้นที่วงกลม r=5: {d:.4}\n", .{CIRCLE_AREA});
}
```

### Comptime Expressions

Expression ใดก็ตามสามารถถูก evaluate ที่ compile-time ถ้าข้อมูลทั้งหมดเป็น comptime:

```zig
const std = @import("std");

// Fibonacci ที่ compile-time
fn fibonacci(comptime n: u32) u64 {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Factorial ที่ compile-time
fn factorial(comptime n: u32) u64 {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}

pub fn main() void {
    // ทั้งหมดนี้คำนวณตอน compile ไม่ใช่ runtime!
    const fib10 = comptime fibonacci(10);
    const fact10 = comptime factorial(10);

    std.debug.print("Fibonacci(10) = {}\n", .{fib10});
    std.debug.print("Factorial(10) = {}\n", .{fact10});

    // แม้แต่ string operations
    const greeting = comptime blk: {
        const base = "Hello";
        const suffix = ", Zig!";
        // ใน real code จะใช้ std.fmt.comptimePrint
        _ = base;
        _ = suffix;
        break :blk "Hello, Zig!";
    };
    std.debug.print("{s}\n", .{greeting});
}
```

### ขอบเขตของ Comptime

ไม่ใช่ทุกอย่างทำได้ที่ comptime — มีข้อจำกัด:

```zig
// สิ่งที่ทำได้ที่ comptime:
// - การคำนวณทางคณิตศาสตร์
// - การจัดการ string literal
// - การสร้างและใช้ types
// - การเรียก comptime functions
// - Loops และ conditions
// - @typeInfo, @Type, @field

// สิ่งที่ทำไม่ได้ที่ comptime:
// - I/O (ไม่สามารถ print ออก terminal ได้)
// - System calls
// - การจัดการ pointer ที่ชี้ไปยัง runtime memory
// - ทุกอย่างที่ขึ้นกับ runtime state
```

---

## Step 203: Comptime Functions

### ฟังก์ชันที่รับ Comptime Parameters

```zig
const std = @import("std");

// T เป็น comptime parameter — เป็น type
fn printValue(comptime T: type, value: T) void {
    const info = @typeInfo(T);
    switch (info) {
        .Int => std.debug.print("Integer: {d}\n", .{value}),
        .Float => std.debug.print("Float: {d:.4}\n", .{value}),
        .Bool => std.debug.print("Bool: {}\n", .{value}),
        else => std.debug.print("Other: {any}\n", .{value}),
    }
}

// ฟังก์ชันที่ return type ขึ้นกับ comptime parameter
fn makeArray(comptime T: type, comptime size: usize) [size]T {
    var arr: [size]T = undefined;
    for (&arr, 0..) |*elem, i| {
        elem.* = @as(T, @intCast(i));
    }
    return arr;
}

pub fn main() void {
    printValue(i32, 42);
    printValue(f64, 3.14);
    printValue(bool, true);

    const intArr = makeArray(i32, 5);
    std.debug.print("Array: {any}\n", .{intArr});

    const floatArr = makeArray(f64, 3);
    std.debug.print("Float Array: {any}\n", .{floatArr});
}
```

### Comptime Type Inspection

```zig
const std = @import("std");

// ฟังก์ชันที่ตรวจสอบ type และทำงานต่างกัน
fn sum(comptime T: type, items: []const T) T {
    // ตรวจสอบว่า T เป็น numeric type หรือไม่
    const info = @typeInfo(T);
    switch (info) {
        .Int, .Float => {},  // OK
        else => @compileError("sum() ใช้ได้กับ numeric types เท่านั้น"),
    }

    var total: T = 0;
    for (items) |item| {
        total += item;
    }
    return total;
}

// ฟังก์ชันที่สร้าง type ใหม่ตาม parameter
fn Vector(comptime T: type, comptime N: usize) type {
    return struct {
        data: [N]T,

        const Self = @This();

        pub fn init(fill: T) Self {
            return .{ .data = [_]T{fill} ** N };
        }

        pub fn dot(a: Self, b: Self) T {
            var result: T = 0;
            for (a.data, b.data) |av, bv| {
                result += av * bv;
            }
            return result;
        }

        pub fn len(self: Self) usize {
            _ = self;
            return N;
        }
    };
}

pub fn main() void {
    const ints = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("Sum of ints: {}\n", .{sum(i32, &ints)});

    const floats = [_]f64{ 1.1, 2.2, 3.3 };
    std.debug.print("Sum of floats: {d:.1}\n", .{sum(f64, &floats)});

    // Vector type ที่สร้างตอน compile
    const Vec3f = Vector(f32, 3);
    const v1 = Vec3f{ .data = .{ 1.0, 2.0, 3.0 } };
    const v2 = Vec3f{ .data = .{ 4.0, 5.0, 6.0 } };
    std.debug.print("Dot product: {d:.1}\n", .{v1.dot(v2)});
    std.debug.print("Vector length: {}\n", .{v1.len()});
}
```

### เปรียบเทียบกับ C++ Templates

```cpp
// C++: template syntax ซับซ้อน
template<typename T>
T sum(const std::vector<T>& items) {
    T total = 0;
    for (const auto& item : items) total += item;
    return total;
}

// Error message เมื่อใช้ผิด type อ่านยากมาก
// "no matching function for call to 'sum'"
// "candidate template ignored: substitution failure..."
```

```zig
// Zig: ใช้ comptime parameter ธรรมดา
fn sum(comptime T: type, items: []const T) T {
    // ถ้า T ผิดจะ error ที่บรรทัดนี้พร้อม message ที่เข้าใจได้
    if (@typeInfo(T) != .Int and @typeInfo(T) != .Float) {
        @compileError("sum() ต้องการ numeric type");
    }
    var total: T = 0;
    for (items) |item| total += item;
    return total;
}
```

---

## Step 204: Comptime Blocks

### Syntax ของ Comptime Block

```zig
const std = @import("std");

pub fn main() void {
    // comptime block — ทุกอย่างภายในรันตอน compile
    const lookup_table = comptime blk: {
        var table: [256]u8 = undefined;
        for (&table, 0..) |*entry, i| {
            entry.* = if (i < 128) @as(u8, @intCast(i)) else 0;
        }
        break :blk table;
    };

    std.debug.print("lookup[65] = {}\n", .{lookup_table[65]});  // 'A' = 65
    std.debug.print("lookup[200] = {}\n", .{lookup_table[200]});  // 0

    // comptime block สามารถมีหลาย statements
    const magic_number = comptime blk: {
        const a = 6;
        const b = 7;
        const c = a * b;
        // สามารถใช้ loop ได้
        var sum: i32 = 0;
        var i: i32 = 1;
        while (i <= c) : (i += 1) {
            sum += i;
        }
        break :blk sum;
    };
    std.debug.print("Magic: {}\n", .{magic_number});
}
```

### Comptime Block สำหรับสร้าง Lookup Tables

หนึ่งในการใช้งาน comptime block ที่พบบ่อยที่สุดคือการสร้าง lookup table:

```zig
const std = @import("std");

// สร้าง sin lookup table ที่ compile-time
const SIN_TABLE_SIZE = 360;
const sin_table = blk: {
    @setEvalBranchQuota(100000);
    var table: [SIN_TABLE_SIZE]f32 = undefined;
    for (&table, 0..) |*v, i| {
        const angle = @as(f32, @floatFromInt(i)) * std.math.pi / 180.0;
        v.* = @sin(angle);
    }
    break :blk table;
};

// CRC32 lookup table
const crc32_table = blk: {
    var table: [256]u32 = undefined;
    var i: u32 = 0;
    while (i < 256) : (i += 1) {
        var crc = i;
        var j: u32 = 0;
        while (j < 8) : (j += 1) {
            if (crc & 1 != 0) {
                crc = (crc >> 1) ^ 0xEDB88320;
            } else {
                crc >>= 1;
            }
        }
        table[i] = crc;
    }
    break :blk table;
};

pub fn main() void {
    // ใช้ sin table — ไม่มี runtime computation
    std.debug.print("sin(90°) ≈ {d:.4}\n", .{sin_table[90]});
    std.debug.print("sin(180°) ≈ {d:.4}\n", .{sin_table[180]});

    // ทดสอบ CRC table
    std.debug.print("CRC32 table[1] = 0x{X:0>8}\n", .{crc32_table[1]});
}
```

### Nested Comptime Blocks

```zig
const std = @import("std");

// การ nest comptime blocks
fn generateMatrix(comptime rows: usize, comptime cols: usize) [rows][cols]u32 {
    return comptime blk: {
        var matrix: [rows][cols]u32 = undefined;
        var r: usize = 0;
        while (r < rows) : (r += 1) {
            var c: usize = 0;
            while (c < cols) : (c += 1) {
                matrix[r][c] = @as(u32, @intCast(r * cols + c));
            }
        }
        break :blk matrix;
    };
}

pub fn main() void {
    const mat = generateMatrix(3, 4);
    for (mat) |row| {
        for (row) |val| {
            std.debug.print("{:2} ", .{val});
        }
        std.debug.print("\n", .{});
    }
}
```

---

## Step 205: Comptime กับ Type Parameters

### Types เป็น First-Class Values

ใน Zig, types เป็น values ชนิดหนึ่ง (ชนิด `type`) ที่สามารถส่งผ่านฟังก์ชันได้:

```zig
const std = @import("std");

// ฟังก์ชันที่รับ type เป็น argument และ return type ใหม่
fn Optional(comptime T: type) type {
    return struct {
        value: T,
        has_value: bool,

        const Self = @This();

        pub fn some(val: T) Self {
            return .{ .value = val, .has_value = true };
        }

        pub fn none() Self {
            return .{ .value = undefined, .has_value = false };
        }

        pub fn unwrap(self: Self) T {
            if (!self.has_value) @panic("Unwrap on None!");
            return self.value;
        }

        pub fn unwrapOr(self: Self, default: T) T {
            if (self.has_value) return self.value;
            return default;
        }
    };
}

pub fn main() void {
    const MaybeInt = Optional(i32);
    const MaybeStr = Optional([]const u8);

    const val1 = MaybeInt.some(42);
    const val2 = MaybeInt.none();
    const val3 = MaybeStr.some("hello");

    std.debug.print("val1: {}\n", .{val1.unwrap()});
    std.debug.print("val2: {}\n", .{val2.unwrapOr(-1)});
    std.debug.print("val3: {s}\n", .{val3.unwrap()});
}
```

### Type Constraints

```zig
const std = @import("std");

// กำหนดข้อจำกัดว่า T ต้องมี method อะไรบ้าง
fn printable(comptime T: type) void {
    // ตรวจสอบว่า T มี method toString() หรือไม่
    if (!@hasDecl(T, "toString")) {
        @compileError("Type " ++ @typeName(T) ++ " ต้องมี method toString()");
    }
}

// ตรวจสอบว่า type เป็น integer หรือไม่
fn assertInteger(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int => {},
        else => @compileError(@typeName(T) ++ " ไม่ใช่ integer type"),
    }
}

// Generic min function พร้อม constraint
fn min(comptime T: type, a: T, b: T) T {
    assertInteger(T);
    return if (a < b) a else b;
}

// ดูข้อมูลของ type
fn describeType(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("Type: {s}\n", .{@typeName(T)});
    switch (info) {
        .Int => |int_info| {
            std.debug.print("  Signedness: {s}\n", .{
                if (int_info.signedness == .signed) "signed" else "unsigned"
            });
            std.debug.print("  Bits: {}\n", .{int_info.bits});
        },
        .Float => |float_info| {
            std.debug.print("  Bits: {}\n", .{float_info.bits});
        },
        .Struct => |struct_info| {
            std.debug.print("  Fields: {}\n", .{struct_info.fields.len});
        },
        else => std.debug.print("  (other)\n", .{}),
    }
}

pub fn main() void {
    describeType(i32);
    describeType(f64);
    describeType(struct { x: i32, y: i32 });

    std.debug.print("min(3, 7) = {}\n", .{min(i32, 3, 7)});
}
```

### การสร้าง Type Hierarchy ด้วย Comptime

```zig
const std = @import("std");

// สร้าง Stack type ที่รองรับ type ใดก็ได้
fn Stack(comptime T: type, comptime capacity: usize) type {
    return struct {
        items: [capacity]T,
        top: usize,

        const Self = @This();
        pub const Error = error{ StackFull, StackEmpty };

        pub fn init() Self {
            return .{ .items = undefined, .top = 0 };
        }

        pub fn push(self: *Self, item: T) Error!void {
            if (self.top >= capacity) return Error.StackFull;
            self.items[self.top] = item;
            self.top += 1;
        }

        pub fn pop(self: *Self) Error!T {
            if (self.top == 0) return Error.StackEmpty;
            self.top -= 1;
            return self.items[self.top];
        }

        pub fn peek(self: Self) Error!T {
            if (self.top == 0) return Error.StackEmpty;
            return self.items[self.top - 1];
        }

        pub fn isEmpty(self: Self) bool {
            return self.top == 0;
        }

        pub fn isFull(self: Self) bool {
            return self.top >= capacity;
        }

        pub fn size(self: Self) usize {
            return self.top;
        }
    };
}

pub fn main() void {
    // IntStack ขนาด 10
    var int_stack = Stack(i32, 10).init();
    int_stack.push(1) catch unreachable;
    int_stack.push(2) catch unreachable;
    int_stack.push(3) catch unreachable;

    while (!int_stack.isEmpty()) {
        const val = int_stack.pop() catch break;
        std.debug.print("popped: {}\n", .{val});
    }

    // FloatStack ขนาด 5
    var float_stack = Stack(f32, 5).init();
    float_stack.push(1.1) catch unreachable;
    float_stack.push(2.2) catch unreachable;

    std.debug.print("float top: {d:.1}\n", .{float_stack.peek() catch 0.0});
    std.debug.print("float size: {}\n", .{float_stack.size()});
}
```

---

## Step 206: comptime_int และ comptime_float

### ประเภทพิเศษของ Comptime Numbers

Zig มีประเภทพิเศษ 2 ชนิดที่ใช้สำหรับตัวเลขที่ compile-time:

```zig
const std = @import("std");

pub fn main() void {
    // comptime_int — integer ที่ไม่มีขนาดจำกัด
    const big: comptime_int = 1_000_000_000_000_000_000_000;
    std.debug.print("big number: {}\n", .{big});

    // comptime_float — floating point ความแม่นยำสูงมาก
    const precise: comptime_float = 3.14159265358979323846264338327950288;
    std.debug.print("precise pi: {d:.15}\n", .{precise});

    // comptime_int สามารถแปลงเป็น integer type ใดก็ได้
    const x: i8 = 100;    // OK
    const y: u64 = 100;   // OK — literal 100 เป็น comptime_int
    _ = x;
    _ = y;

    // Integer literals เป็น comptime_int
    const n = 42;  // type ของ n คือ comptime_int
    const m: i32 = n;  // แปลงเป็น i32 โดยอัตโนมัติ
    _ = m;
    std.debug.print("type of n: {s}\n", .{@typeName(@TypeOf(n))});

    // Float literals เป็น comptime_float
    const pi = 3.14;  // type คือ comptime_float
    const pi_f32: f32 = pi;
    const pi_f64: f64 = pi;
    std.debug.print("pi as f32: {d:.6}\n", .{pi_f32});
    std.debug.print("pi as f64: {d:.10}\n", .{pi_f64});
}
```

### ความพิเศษของ comptime_int

```zig
const std = @import("std");

pub fn main() void {
    // comptime_int ไม่มี overflow!
    const huge = comptime 1_000_000 * 1_000_000 * 1_000_000;
    std.debug.print("huge: {}\n", .{huge});

    // สามารถใช้กับ bit operations ได้
    const flags: comptime_int = 0b1010_1010;
    std.debug.print("flags: 0b{b:0>8}\n", .{flags});
    std.debug.print("flags >> 1: {}\n", .{flags >> 1});

    // Hex literals
    const color: comptime_int = 0xFF5733;
    std.debug.print("color: #{X:0>6}\n", .{color});

    // comptime_int เป็น compatible กับทุก integer type
    // ถ้าค่าพอดีกัน
    const small_val: i8 = 127;  // OK, 127 fits in i8
    _ = small_val;
    // const overflow_val: i8 = 200;  // ERROR: 200 ไม่พอดีใน i8

    // การใช้ comptime_int ใน calculations
    const BUFFER_SIZE = 4 * 1024;  // 4KB — comptime_int calculation
    var buffer: [BUFFER_SIZE]u8 = undefined;
    buffer[0] = 0xFF;
    std.debug.print("buffer size: {}\n", .{BUFFER_SIZE});
}
```

### comptime_float และความแม่นยำ

```zig
const std = @import("std");

pub fn main() void {
    // comptime_float มีความแม่นยำสูงกว่า f64
    const exact_pi: comptime_float = 3.14159265358979323846264338327950288;

    // เมื่อแปลงเป็น f32 จะสูญเสียความแม่นยำ
    const pi_f32: f32 = exact_pi;
    // เมื่อแปลงเป็น f64 จะสูญเสียบางส่วน
    const pi_f64: f64 = exact_pi;

    std.debug.print("f32 pi: {d:.10}\n", .{pi_f32});
    std.debug.print("f64 pi: {d:.15}\n", .{pi_f64});

    // comptime calculation ด้วย comptime_float
    const e: comptime_float = 2.71828182845904523536;
    const golden: comptime_float = 1.61803398874989484820;

    const result: f64 = exact_pi * e * golden;
    std.debug.print("π × e × φ ≈ {d:.6}\n", .{result});

    // ใช้ @as เพื่อระบุ type
    const precise_half = @as(f64, @as(comptime_float, 1.0) / 3.0);
    std.debug.print("1/3 ≈ {d:.15}\n", .{precise_half});
}
```

---

## Step 207: @compileError และ @compileLog

### @compileError — สร้าง Error ที่ Compile-Time

```zig
const std = @import("std");

// ตรวจสอบ type constraint
fn ensureNumeric(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {},
        else => @compileError(
            "ต้องการ numeric type แต่ได้รับ: " ++ @typeName(T)
        ),
    }
}

// ตรวจสอบว่า struct มี field ที่ต้องการ
fn ensureHasField(comptime T: type, comptime field_name: []const u8) void {
    if (!@hasField(T, field_name)) {
        @compileError(
            @typeName(T) ++ " ต้องมี field '" ++ field_name ++ "'"
        );
    }
}

// ตรวจสอบ version
const REQUIRED_ZIG_VERSION = "0.12.0";
fn checkVersion() void {
    // ตัวอย่างการ check version (simplified)
    _ = REQUIRED_ZIG_VERSION;
    // ใน real code จะใช้ builtin.zig_version
}

const Point = struct {
    x: f32,
    y: f32,
};

const Color = struct {
    r: u8,
    g: u8,
    b: u8,
};

pub fn main() void {
    // ถ้า uncomment บรรทัดต่อไปนี้จะ compile error:
    // ensureNumeric([]const u8);

    ensureNumeric(i32);
    ensureNumeric(f64);

    ensureHasField(Point, "x");
    ensureHasField(Color, "r");
    // ensureHasField(Point, "z");  // ERROR: Point ต้องมี field 'z'

    std.debug.print("All checks passed!\n", .{});
}
```

### @compileLog — Debug ที่ Compile-Time

```zig
const std = @import("std");

fn typeSize(comptime T: type) usize {
    // @compileLog พิมพ์ข้อมูลตอน compile (ไม่ใช่ runtime)
    @compileLog("Computing size of:", @typeName(T));
    return @sizeOf(T);
}

// ใช้ @compileLog เพื่อ debug comptime computation
fn computeAtComptime(comptime n: u32) u64 {
    @compileLog("Computing for n =", n);
    var result: u64 = 0;
    var i: u32 = 0;
    while (i < n) : (i += 1) {
        result += i;
    }
    @compileLog("Result =", result);
    return result;
}

pub fn main() void {
    // @compileLog จะพิมพ์ออกมาตอน compile ไม่ใช่ runtime
    // (uncomment เพื่อดูผล)
    // _ = typeSize(i32);
    // _ = computeAtComptime(5);

    std.debug.print("Size of i32: {}\n", .{@sizeOf(i32)});
    std.debug.print("Size of f64: {}\n", .{@sizeOf(f64)});
    std.debug.print("Size of bool: {}\n", .{@sizeOf(bool)});
}
```

### @compileError สำหรับ API Validation

```zig
const std = @import("std");

// Validated buffer type
fn Buffer(comptime T: type, comptime capacity: usize) type {
    // Validate parameters ที่ compile-time
    if (capacity == 0) {
        @compileError("Buffer capacity ต้องมากกว่า 0");
    }
    if (capacity > 1024 * 1024) {
        @compileError("Buffer capacity ไม่ควรเกิน 1MB — ใช้ heap แทน");
    }

    const allowed_types = [_]type{ u8, i8, u16, i16, u32, i32, f32, f64 };
    var type_ok = false;
    for (allowed_types) |allowed| {
        if (T == allowed) {
            type_ok = true;
            break;
        }
    }
    if (!type_ok) {
        @compileError("Buffer ไม่รองรับ type: " ++ @typeName(T));
    }

    return struct {
        data: [capacity]T,
        used: usize,

        const Self = @This();

        pub fn init() Self {
            return .{ .data = undefined, .used = 0 };
        }

        pub fn write(self: *Self, value: T) !void {
            if (self.used >= capacity) return error.BufferFull;
            self.data[self.used] = value;
            self.used += 1;
        }

        pub fn read(self: Self, index: usize) !T {
            if (index >= self.used) return error.IndexOutOfBounds;
            return self.data[index];
        }
    };
}

pub fn main() void {
    var buf = Buffer(u8, 256).init();

    buf.write(0x48) catch unreachable;  // 'H'
    buf.write(0x65) catch unreachable;  // 'e'
    buf.write(0x6C) catch unreachable;  // 'l'
    buf.write(0x6C) catch unreachable;  // 'l'
    buf.write(0x6F) catch unreachable;  // 'o'

    std.debug.print("First byte: 0x{X}\n", .{buf.read(0) catch 0});
    std.debug.print("Used: {}\n", .{buf.used});
}
```

---

## Step 208: Comptime Branching (if at Comptime)

### if ที่ Comptime

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // Comptime if — เลือก code path ตอน compile
    // สาขาที่ไม่ถูกเลือกจะไม่ถูก compile เลย
    if (comptime builtin.os.tag == .linux) {
        std.debug.print("Running on Linux!\n", .{});
    } else if (comptime builtin.os.tag == .windows) {
        std.debug.print("Running on Windows!\n", .{});
    } else if (comptime builtin.os.tag == .macos) {
        std.debug.print("Running on macOS!\n", .{});
    } else {
        std.debug.print("Running on other OS\n", .{});
    }

    // Comptime if สำหรับ type selection
    const is_debug = builtin.mode == .Debug;
    if (comptime is_debug) {
        std.debug.print("Debug mode: extra checks enabled\n", .{});
    }
}
```

### Comptime Branching ใน Generic Functions

```zig
const std = @import("std");

// ฟังก์ชัน serialize ที่ทำงานต่างกันตาม type
fn serialize(comptime T: type, value: T, writer: anytype) !void {
    const info = @typeInfo(T);

    // comptime if — แต่ละ branch อาจใช้ operations ที่แตกต่างกัน
    if (comptime info == .Int) {
        try writer.print("{d}", .{value});
    } else if (comptime info == .Float) {
        try writer.print("{d:.6}", .{value});
    } else if (comptime info == .Bool) {
        try writer.print("{s}", .{if (value) "true" else "false"});
    } else if (comptime info == .Pointer) {
        // String
        if (comptime info.Pointer.child == u8) {
            try writer.print("\"{s}\"", .{value});
        }
    } else if (comptime info == .Struct) {
        try writer.print("{{", .{});
        inline for (info.Struct.fields, 0..) |field, i| {
            if (i > 0) try writer.print(", ", .{});
            try writer.print("{s}: ", .{field.name});
            try serialize(field.type, @field(value, field.name), writer);
        }
        try writer.print("}}", .{});
    } else {
        try writer.print("<unknown>", .{});
    }
}

const Point = struct { x: f32, y: f32 };
const Person = struct { name: []const u8, age: u32 };

pub fn main() void {
    const stdout = std.io.getStdOut().writer();

    serialize(i32, 42, stdout) catch unreachable;
    stdout.print("\n", .{}) catch unreachable;

    serialize(f64, 3.14159, stdout) catch unreachable;
    stdout.print("\n", .{}) catch unreachable;

    serialize(bool, true, stdout) catch unreachable;
    stdout.print("\n", .{}) catch unreachable;

    serialize(Point, .{ .x = 1.0, .y = 2.0 }, stdout) catch unreachable;
    stdout.print("\n", .{}) catch unreachable;

    serialize(Person, .{ .name = "Alice", .age = 30 }, stdout) catch unreachable;
    stdout.print("\n", .{}) catch unreachable;
}
```

### Platform-Specific Code ด้วย Comptime If

```zig
const std = @import("std");
const builtin = @import("builtin");

// Type ที่เลือกตาม platform
const NativeInt = if (builtin.cpu.arch.ptrBitWidth() == 64) i64 else i32;
const NativeUInt = if (builtin.cpu.arch.ptrBitWidth() == 64) u64 else u32;

// ฟังก์ชันที่ใช้ platform-specific implementation
fn getPageSize() usize {
    return comptime if (builtin.os.tag == .windows) 4096 else 4096;
    // ใน real code จะมี different values สำหรับต่าง OS
}

// Endianness handling
fn byteSwap(comptime T: type, value: T) T {
    if (comptime @typeInfo(T).Int.bits <= 8) return value;
    return @byteSwap(value);
}

fn toNetworkByteOrder(comptime T: type, value: T) T {
    return if (comptime builtin.cpu.arch.endian() == .little)
        byteSwap(T, value)
    else
        value;
}

pub fn main() void {
    std.debug.print("NativeInt size: {} bits\n", .{@bitSizeOf(NativeInt)});
    std.debug.print("Page size: {} bytes\n", .{getPageSize()});

    const val: u32 = 0x12345678;
    const network_val = toNetworkByteOrder(u32, val);
    std.debug.print("Network byte order: 0x{X:0>8}\n", .{network_val});
}
```

---

## Step 209: Comptime Loop (inline for)

### inline for — Loop ที่ Unroll ตอน Compile

```zig
const std = @import("std");

pub fn main() void {
    // inline for กับ tuple/array ที่รู้ขนาดตอน compile
    const values = .{ 1, 2.5, "hello", true };

    std.debug.print("Tuple elements:\n", .{});
    inline for (values, 0..) |val, i| {
        std.debug.print("  [{}]: {any} (type: {s})\n", .{
            i, val, @typeName(@TypeOf(val))
        });
    }

    // inline for กับ types
    const types = .{ i8, i16, i32, i64 };
    std.debug.print("\nInteger sizes:\n", .{});
    inline for (types) |T| {
        std.debug.print("  {s}: {} bytes\n", .{ @typeName(T), @sizeOf(T) });
    }
}
```

### inline for สำหรับ Struct Iteration

```zig
const std = @import("std");

const Config = struct {
    host: []const u8 = "localhost",
    port: u16 = 8080,
    debug: bool = false,
    max_connections: u32 = 100,
    timeout_ms: u64 = 5000,
};

// ฟังก์ชันที่ iterate ผ่าน fields ของ struct ด้วย comptime
fn printConfig(config: Config) void {
    std.debug.print("Configuration:\n", .{});
    inline for (@typeInfo(Config).Struct.fields) |field| {
        const value = @field(config, field.name);
        std.debug.print("  {s} = {any}\n", .{ field.name, value });
    }
}

// Deep equality comparison ที่ generic
fn deepEqual(comptime T: type, a: T, b: T) bool {
    const info = @typeInfo(T);
    return switch (info) {
        .Int, .Float, .Bool => a == b,
        .Struct => blk: {
            inline for (info.Struct.fields) |field| {
                if (!deepEqual(
                    field.type,
                    @field(a, field.name),
                    @field(b, field.name),
                )) break :blk false;
            }
            break :blk true;
        },
        else => false,
    };
}

pub fn main() void {
    const cfg = Config{
        .host = "example.com",
        .port = 443,
        .debug = true,
    };
    printConfig(cfg);

    const c1 = Config{ .port = 9000 };
    const c2 = Config{ .port = 9000 };
    const c3 = Config{ .port = 8080 };

    std.debug.print("\nc1 == c2: {}\n", .{deepEqual(Config, c1, c2)});
    std.debug.print("c1 == c3: {}\n", .{deepEqual(Config, c1, c3)});
}
```

### inline while — Loop แบบอื่น

```zig
const std = @import("std");

// Unroll loop ด้วย inline for กับ range
fn unrolledSum(comptime N: usize, arr: [N]i32) i32 {
    var sum: i32 = 0;
    // inline for unrolls loop ทั้งหมดตอน compile
    inline for (0..N) |i| {
        sum += arr[i];
    }
    return sum;
}

// String interning ด้วย comptime
fn internString(comptime s: []const u8) *const [s.len]u8 {
    // สร้าง global constant สำหรับ string นี้
    return comptime &s.*;
}

pub fn main() void {
    const arr = [_]i32{ 1, 2, 3, 4, 5 };
    const total = unrolledSum(5, arr);
    std.debug.print("Sum (unrolled): {}\n", .{total});

    // ตัวอย่าง bit manipulation ด้วย inline for
    const flags: u8 = 0b10110101;
    var bit_count: u32 = 0;
    inline for (0..8) |i| {
        if (flags & (1 << i) != 0) bit_count += 1;
    }
    std.debug.print("Bit count of {b:0>8}: {}\n", .{ flags, bit_count });
}
```

### ตัวอย่าง: Tuple Processing

```zig
const std = @import("std");

// ฟังก์ชันที่ process tuple ที่มี mixed types
fn printAll(args: anytype) void {
    const ArgsType = @TypeOf(args);
    const fields = @typeInfo(ArgsType).Struct.fields;

    inline for (fields) |field| {
        const val = @field(args, field.name);
        const T = field.type;
        const info = @typeInfo(T);

        switch (info) {
            .Int => std.debug.print("{d} ", .{val}),
            .Float => std.debug.print("{d:.2} ", .{val}),
            .Bool => std.debug.print("{} ", .{val}),
            .Pointer => std.debug.print("{s} ", .{val}),
            else => std.debug.print("? ", .{}),
        }
    }
    std.debug.print("\n", .{});
}

// คำนวณ hash ของ struct fields ที่ compile-time
fn structHash(comptime T: type) u64 {
    var h: u64 = 0;
    inline for (@typeInfo(T).Struct.fields) |field| {
        const name = field.name;
        inline for (name) |c| {
            h = h *% 31 +% c;
        }
    }
    return h;
}

pub fn main() void {
    printAll(.{ 42, 3.14, true, "hello" });
    printAll(.{ 1, 2, 3 });
    printAll(.{ "foo", "bar", "baz" });

    const MyStruct = struct { x: i32, y: f32, name: []const u8 };
    std.debug.print("Struct hash: {}\n", .{structHash(MyStruct)});
}
```

---

## Step 210: Comptime กับ Generic Data Structures

### Generic Queue

```zig
const std = @import("std");

// Queue ที่ใช้ได้กับ type ใดก็ได้
fn Queue(comptime T: type) type {
    return struct {
        const Self = @This();
        const Node = struct {
            value: T,
            next: ?*Node,
        };

        head: ?*Node,
        tail: ?*Node,
        size: usize,
        allocator: std.mem.Allocator,

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .head = null,
                .tail = null,
                .size = 0,
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            while (self.dequeue()) |_| {}
        }

        pub fn enqueue(self: *Self, value: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .value = value, .next = null };

            if (self.tail) |tail| {
                tail.next = node;
            } else {
                self.head = node;
            }
            self.tail = node;
            self.size += 1;
        }

        pub fn dequeue(self: *Self) ?T {
            const head = self.head orelse return null;
            self.head = head.next;
            if (self.head == null) self.tail = null;
            const value = head.value;
            self.allocator.destroy(head);
            self.size -= 1;
            return value;
        }

        pub fn peek(self: Self) ?T {
            return if (self.head) |h| h.value else null;
        }

        pub fn isEmpty(self: Self) bool {
            return self.size == 0;
        }
    };
}

pub fn main() void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Queue สำหรับ integers
    var int_queue = Queue(i32).init(allocator);
    defer int_queue.deinit();

    int_queue.enqueue(1) catch unreachable;
    int_queue.enqueue(2) catch unreachable;
    int_queue.enqueue(3) catch unreachable;

    std.debug.print("Queue size: {}\n", .{int_queue.size});
    while (int_queue.dequeue()) |val| {
        std.debug.print("dequeued: {}\n", .{val});
    }

    // Queue สำหรับ strings
    var str_queue = Queue([]const u8).init(allocator);
    defer str_queue.deinit();

    str_queue.enqueue("first") catch unreachable;
    str_queue.enqueue("second") catch unreachable;
    str_queue.enqueue("third") catch unreachable;

    while (str_queue.dequeue()) |val| {
        std.debug.print("string: {s}\n", .{val});
    }
}
```

### Generic HashMap (Simplified)

```zig
const std = @import("std");

// HashMap อย่างง่ายที่ใช้ comptime
fn SimpleHashMap(comptime K: type, comptime V: type, comptime capacity: usize) type {
    // ตรวจสอบ constraints ที่ compile-time
    if (capacity == 0 or (capacity & (capacity - 1)) != 0) {
        @compileError("capacity ต้องเป็น power of 2");
    }

    return struct {
        const Self = @This();
        const Entry = struct {
            key: K,
            value: V,
            occupied: bool,
        };

        entries: [capacity]Entry,
        count: usize,

        pub fn init() Self {
            return .{
                .entries = [_]Entry{.{ .key = undefined, .value = undefined, .occupied = false }} ** capacity,
                .count = 0,
            };
        }

        fn hashKey(key: K) usize {
            // Simple hash based on type
            const info = @typeInfo(K);
            return switch (info) {
                .Int => @as(usize, @intCast(key)) & (capacity - 1),
                else => 0,
            };
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            if (self.count >= capacity / 2) return error.MapFull;
            var idx = hashKey(key);
            while (self.entries[idx].occupied and self.entries[idx].key != key) {
                idx = (idx + 1) & (capacity - 1);
            }
            if (!self.entries[idx].occupied) self.count += 1;
            self.entries[idx] = .{ .key = key, .value = value, .occupied = true };
        }

        pub fn get(self: Self, key: K) ?V {
            var idx = hashKey(key);
            var probe: usize = 0;
            while (probe < capacity) : (probe += 1) {
                const entry = self.entries[idx];
                if (!entry.occupied) return null;
                if (entry.key == key) return entry.value;
                idx = (idx + 1) & (capacity - 1);
            }
            return null;
        }
    };
}

pub fn main() void {
    var map = SimpleHashMap(u32, []const u8, 16).init();

    map.put(1, "one") catch unreachable;
    map.put(2, "two") catch unreachable;
    map.put(3, "three") catch unreachable;
    map.put(42, "forty-two") catch unreachable;

    if (map.get(1)) |v| std.debug.print("1 -> {s}\n", .{v});
    if (map.get(42)) |v| std.debug.print("42 -> {s}\n", .{v});
    if (map.get(99)) |v| std.debug.print("99 -> {s}\n", .{v})
    else std.debug.print("99 -> not found\n", .{});

    std.debug.print("count: {}\n", .{map.count});
}
```

### Real-World Example: Type-Safe Event System

```zig
const std = @import("std");

// Event system ที่ type-safe ด้วย comptime
fn EventSystem(comptime Events: type) type {
    // Events ต้องเป็น tagged union
    if (@typeInfo(Events) != .Union) {
        @compileError("Events ต้องเป็น union type");
    }

    return struct {
        const Self = @This();
        const Handler = *const fn (event: Events) void;
        const MAX_HANDLERS = 16;

        handlers: [MAX_HANDLERS]?Handler,
        handler_count: usize,

        pub fn init() Self {
            return .{
                .handlers = [_]?Handler{null} ** MAX_HANDLERS,
                .handler_count = 0,
            };
        }

        pub fn subscribe(self: *Self, handler: Handler) !void {
            if (self.handler_count >= MAX_HANDLERS) return error.TooManyHandlers;
            self.handlers[self.handler_count] = handler;
            self.handler_count += 1;
        }

        pub fn emit(self: Self, event: Events) void {
            for (self.handlers[0..self.handler_count]) |maybe_handler| {
                if (maybe_handler) |handler| {
                    handler(event);
                }
            }
        }
    };
}

// Define events สำหรับ game
const GameEvent = union(enum) {
    player_moved: struct { x: f32, y: f32 },
    enemy_spawned: struct { id: u32, hp: u32 },
    score_changed: u32,
    game_over: void,
};

fn onGameEvent(event: GameEvent) void {
    switch (event) {
        .player_moved => |pos| {
            std.debug.print("Player moved to ({d:.1}, {d:.1})\n", .{ pos.x, pos.y });
        },
        .enemy_spawned => |info| {
            std.debug.print("Enemy #{} spawned with {} HP\n", .{ info.id, info.hp });
        },
        .score_changed => |score| {
            std.debug.print("Score: {}\n", .{score});
        },
        .game_over => {
            std.debug.print("Game Over!\n", .{});
        },
    }
}

pub fn main() void {
    var game_events = EventSystem(GameEvent).init();

    game_events.subscribe(onGameEvent) catch unreachable;

    game_events.emit(.{ .player_moved = .{ .x = 10.0, .y = 20.0 } });
    game_events.emit(.{ .enemy_spawned = .{ .id = 1, .hp = 100 } });
    game_events.emit(.{ .score_changed = 500 });
    game_events.emit(.game_over);
}
```

---

## Performance Implications

### Comptime vs Runtime Performance

```zig
const std = @import("std");

// ตัวอย่างที่แสดงถึง zero-cost abstraction
fn isPrime(comptime n: u64) bool {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    var i: u64 = 3;
    while (i * i <= n) : (i += 2) {
        if (n % i == 0) return false;
    }
    return true;
}

// Primes ถูกคำนวณตอน compile — ไม่มี runtime cost!
const PRIME_TABLE = blk: {
    @setEvalBranchQuota(1000000);
    var primes: [50]u64 = undefined;
    var count: usize = 0;
    var n: u64 = 2;
    while (count < 50) : (n += 1) {
        if (isPrime(n)) {
            primes[count] = n;
            count += 1;
        }
    }
    break :blk primes;
};

pub fn main() void {
    std.debug.print("First 10 primes: ", .{});
    for (PRIME_TABLE[0..10]) |p| {
        std.debug.print("{} ", .{p});
    }
    std.debug.print("\n", .{});

    // PRIME_TABLE เป็นแค่ data ใน binary — ไม่มีการคำนวณตอน runtime
    std.debug.print("50th prime: {}\n", .{PRIME_TABLE[49]});
}
```

### Memory Layout Optimization

```zig
const std = @import("std");

// Struct ที่ optimize memory layout ด้วย comptime
fn SortedStruct(comptime T: type) type {
    const info = @typeInfo(T);
    if (info != .Struct) @compileError("ต้องเป็น struct type");

    // Sort fields ตาม size (descending) เพื่อลด padding
    const fields = info.Struct.fields;

    // ใน real implementation จะ sort fields
    // ที่นี่แสดงแนวคิดการ inspect fields
    std.debug.print("Struct: {s}\n", .{@typeName(T)});
    inline for (fields) |field| {
        std.debug.print("  {s}: {} bytes\n", .{ field.name, @sizeOf(field.type) });
    }
    return T;
}

// Bad layout: padding wastes space
const BadLayout = struct {
    a: u8,    // 1 byte + 7 padding
    b: u64,   // 8 bytes
    c: u8,    // 1 byte + 7 padding
    d: u64,   // 8 bytes
};

// Good layout: sorted by size, minimal padding
const GoodLayout = struct {
    b: u64,   // 8 bytes
    d: u64,   // 8 bytes
    a: u8,    // 1 byte
    c: u8,    // 1 byte + 6 padding (at end)
};

pub fn main() void {
    std.debug.print("BadLayout size: {} bytes\n", .{@sizeOf(BadLayout)});
    std.debug.print("GoodLayout size: {} bytes\n", .{@sizeOf(GoodLayout)});
    // BadLayout = 32 bytes, GoodLayout = 18 bytes
    // Comptime tools ช่วยให้เราวิเคราะห์และ optimize ได้

    _ = SortedStruct(GoodLayout);
}
```

---

## เปรียบเทียบกับ C Macros และ C++ Templates

### C Macros vs Zig Comptime

```c
// === C Macros — ปัญหาที่พบบ่อย ===

// 1. ไม่มี type safety
#define DOUBLE(x) ((x) * 2)
// DOUBLE("hello")  // compile แต่ undefined behavior!

// 2. Side effects
#define MAX(a, b) ((a) > (b) ? (a) : (b))
int x = 5, y = 3;
int m = MAX(x++, y++);  // x++ ถูกเรียก 2 ครั้ง!

// 3. Debug ยาก
#define ASSERT(cond) if (!(cond)) abort()
// Error message ชี้ไปที่ macro แทน call site

// 4. Scope ไม่ถูกต้อง
#define SWAP(a, b) { int tmp = a; a = b; b = tmp; }
// ต้องระวัง dangling else และ scope issues
```

```zig
// === Zig Comptime — แก้ปัญหาทั้งหมด ===

// 1. Type safety สมบูรณ์
fn double(comptime T: type, x: T) T {
    return x * 2;
    // double([]const u8, "hello")  // COMPILE ERROR ชัดเจน
}

// 2. ไม่มี side effects
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
    // max(i32, x, y)  — a และ b ถูกประเมินครั้งเดียว
}

// 3. Error message ชี้ที่ call site ได้
fn assert(condition: bool) void {
    if (!condition) @panic("assertion failed");
}

// 4. Scope ปกติ
fn swap(comptime T: type, a: *T, b: *T) void {
    const tmp = a.*;
    a.* = b.*;
    b.* = tmp;
}
```

### C++ Templates vs Zig Comptime

```cpp
// === C++ Templates ===

// 1. Syntax ซับซ้อน
template<typename T, size_t N>
class Array {
public:
    T data[N];
    // methods...
};

// 2. Error messages อ่านยาก (hundreds of lines)
// "In instantiation of 'Array<std::string, 5>'"
// "required from 'void process(Array<T, N>)'"
// "[with T = std::string; N = 5]"

// 3. Concept (C++20) ช่วยได้บางส่วนแต่ syntax ยังซับซ้อน
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T sum(std::vector<T> items);
```

```zig
// === Zig Comptime ===

// 1. Syntax เรียบง่าย — เป็น Zig ปกติ
fn Array(comptime T: type, comptime N: usize) type {
    return struct { data: [N]T };
}

// 2. Error messages ชัดเจน — ระบุบรรทัดที่ผิดพลาด
// error: sum() ต้องการ numeric type
//   note: called from here

// 3. Constraints ด้วย @compileError ที่อ่านเข้าใจได้
fn sum(comptime T: type, items: []const T) T {
    switch (@typeInfo(T)) {
        .Int, .Float => {},
        else => @compileError("ต้องการ numeric type"),
    }
    // ...
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Generic Min/Max

สร้าง generic `clamp` function ที่จำกัดค่าให้อยู่ระหว่าง `min` และ `max`:

```zig
// TODO: implement this
fn clamp(comptime T: type, value: T, min_val: T, max_val: T) T {
    // ตรวจสอบว่า T เป็น numeric type
    // return ค่าที่ถูก clamp แล้ว
    _ = .{ value, min_val, max_val };
    return undefined;
}

// ทดสอบ:
// clamp(i32, 5, 0, 10)  -> 5
// clamp(i32, -5, 0, 10) -> 0
// clamp(i32, 15, 0, 10) -> 10
```

### แบบฝึกหัดที่ 2: Compile-Time String Operations

สร้าง comptime function ที่นับจำนวนตัวอักษรในชนิดต่างๆ:

```zig
// TODO: implement this
fn countChar(comptime str: []const u8, comptime ch: u8) comptime_int {
    // นับจำนวนครั้งที่ ch ปรากฏใน str
    _ = .{ str, ch };
    return 0;
}

// ทดสอบ:
// countChar("hello world", 'l')  -> 3
// countChar("banana", 'a')       -> 3
```

### แบบฝึกหัดที่ 3: Generic Ring Buffer

สร้าง ring buffer (circular buffer) ที่ใช้ได้กับ type ใดก็ได้:

```zig
// TODO: implement this
fn RingBuffer(comptime T: type, comptime capacity: usize) type {
    // ต้องมี:
    // - write(value: T) !void
    // - read() ?T
    // - isFull() bool
    // - isEmpty() bool
    // - size() usize
    return struct {};
}
```

### แบบฝึกหัดที่ 4: Struct Printer

สร้าง comptime function ที่ print ทุก field ของ struct:

```zig
// TODO: implement this
fn printStruct(value: anytype) void {
    // ใช้ @typeInfo เพื่อ iterate fields
    // print แต่ละ field name และ value
    _ = value;
}

// ทดสอบ:
// printStruct(.{ .x = 1, .y = 2, .name = "test" })
// Output:
//   x: 1
//   y: 2
//   name: test
```

### แบบฝึกหัดที่ 5: Compile-Time State Machine

สร้าง type-safe state machine ที่ transition ถูกตรวจสอบตอน compile:

```zig
// TODO: implement this
// States: .idle, .running, .paused, .stopped
// Transitions:
//   idle -> running (start)
//   running -> paused (pause)
//   paused -> running (resume)
//   running -> stopped (stop)
//   paused -> stopped (stop)

fn StateMachine(comptime States: type, comptime Transitions: anytype) type {
    _ = .{ States, Transitions };
    return struct {};
}
```

---

## สรุป (Summary)

### สิ่งที่เราเรียนรู้ใน Part 21

| Step | หัวข้อ | สิ่งสำคัญที่ต้องจำ |
|------|--------|-------------------|
| 201 | comptime คืออะไร | Code ที่รันตอน compile ไม่ใช่ runtime |
| 202 | comptime variables/expressions | ตัวแปรที่รู้ค่าตอน compile |
| 203 | comptime functions | ฟังก์ชันที่รับ type เป็น argument |
| 204 | comptime blocks | `comptime { ... }` สำหรับ code ที่รันตอน compile |
| 205 | comptime type parameters | Types เป็น first-class values ใน Zig |
| 206 | comptime_int/float | Number types ที่ไม่มีขนาดจำกัดตอน compile |
| 207 | @compileError/@compileLog | Error handling และ debug ที่ compile-time |
| 208 | comptime branching | `if (comptime ...)` เลือก code path ตอน compile |
| 209 | inline for | Unrolled loops ที่ compile-time |
| 210 | generic data structures | Stack, Queue, HashMap ที่ใช้กับ type ใดก็ได้ |

### หลักการสำคัญของ Comptime

1. **ไม่มี special syntax** — comptime คือ Zig code ธรรมดาที่รันเร็วขึ้น
2. **Zero-cost abstraction** — abstraction ที่คุณเขียนจะหายไปใน binary
3. **Type safety** — compiler ตรวจสอบทุกอย่างให้คุณ
4. **Error messages ที่อ่านได้** — ต่างจาก C++ template errors

### เมื่อควรใช้ Comptime

- ✅ เมื่อต้องการ generic containers หรือ algorithms
- ✅ เมื่อ type ของข้อมูลรู้แน่นอนตอน compile
- ✅ เมื่อต้องการ lookup tables ที่คำนวณล่วงหน้า
- ✅ เมื่อต้องการ platform-specific code paths
- ✅ เมื่อต้องการตรวจสอบ invariants ตอน compile

- ❌ เมื่อข้อมูลมาจาก user input หรือ network
- ❌ เมื่อต้องการ I/O operations
- ❌ เมื่อ type ถูกกำหนดตอน runtime

### คำถามเพื่อทบทวน

1. อธิบายความแตกต่างระหว่าง `const` และ `comptime var`
2. ทำไม Zig comptime ถึงดีกว่า C macros?
3. `inline for` ต่างจาก `for` ปกติอย่างไร?
4. เมื่อใดควรใช้ `@compileError`?
5. อธิบาย zero-cost abstraction ด้วยตัวอย่างจาก comptime

---

## What's Next?

ในบทต่อไป (Part 22) เราจะเรียนรู้เรื่อง:
- **Advanced Comptime Patterns** — Type erasure, reflection
- **@typeInfo ขั้นสูง** — การสร้าง types แบบ dynamic
- **Comptime-time computation ที่ซับซ้อน** — Parsing, code generation
- **Metaprogramming patterns** — ที่ใช้ใน real-world libraries

---

*Part 21 จาก Zig Programming Course*  
*ผู้เขียน: Zig Course Team*  
*ปรับปรุงล่าสุด: 2024*
