# ส่วนที่ 13: Unions ใน Zig (Steps 121-130)

Union เป็นหนึ่งในโครงสร้างข้อมูลที่ทรงพลังที่สุดใน Zig ช่วยให้เราสามารถจัดเก็บข้อมูลหลายประเภทในพื้นที่หน่วยความจำเดียวกันได้ ในบทนี้เราจะเรียนรู้ตั้งแต่ bare union พื้นฐาน ไปจนถึงการใช้งานขั้นสูงเช่น tagged union, extern union, และการนำ union ไปประยุกต์ใช้งานจริง

---

## Step 121: Union พื้นฐาน (Bare Union)

### Union คืออะไร?

Union คือโครงสร้างข้อมูลที่ช่วยให้ตัวแปรหนึ่งตัวสามารถเก็บค่าได้หลายประเภท โดยใช้พื้นที่หน่วยความจำร่วมกัน ขนาดของ union จะเท่ากับขนาดของ field ที่ใหญ่ที่สุด

### โครงสร้างหน่วยความจำของ Union

```
Union { i32, f64, bool }

ตำแหน่งหน่วยความจำ:
+--------+--------+--------+--------+--------+--------+--------+--------+
| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 | byte 6 | byte 7 |
+--------+--------+--------+--------+--------+--------+--------+--------+
|           f64 (8 bytes)                                               |
|    i32 (4 bytes)          |
| bool (1 byte) |
+--------+--------+--------+--------+--------+--------+--------+--------+

ขนาดรวม = 8 bytes (ตามขนาดของ f64 ซึ่งใหญ่ที่สุด)
```

เปรียบเทียบกับ Struct:
```
Struct { i32, f64, bool }

+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 | byte 6 | byte 7 | byte 8 |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|     i32 (4 bytes)         |         (padding)         |   f64 (8 bytes)...     |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+

ขนาดรวม = 4 (i32) + 4 (padding) + 8 (f64) + 1 (bool) + 7 (padding) = 24 bytes
```

### Bare Union ใน Zig

```zig
const std = @import("std");

// การประกาศ bare union
const Value = union {
    integer: i32,
    float: f64,
    boolean: bool,
};

pub fn main() void {
    // สร้าง union ด้วย field integer
    var v = Value{ .integer = 42 };
    std.debug.print("integer: {}\n", .{v.integer});

    // เปลี่ยนไปใช้ field float
    v = Value{ .float = 3.14 };
    std.debug.print("float: {}\n", .{v.float});

    // เปลี่ยนไปใช้ field boolean
    v = Value{ .boolean = true };
    std.debug.print("boolean: {}\n", .{v.boolean});
}
```

### ข้อควรระวังกับ Bare Union

Bare union ไม่มีการติดตามว่าตอนนี้กำลังใช้ field ไหนอยู่ การเข้าถึง field ที่ไม่ถูกต้องจะเป็น undefined behavior:

```zig
const Value = union {
    integer: i32,
    float: f64,
};

pub fn main() void {
    var v = Value{ .integer = 100 };

    // อย่าทำแบบนี้! เป็น undefined behavior
    // std.debug.print("{}\n", .{v.float}); // ผิด!

    // ต้องทำแบบนี้เท่านั้น
    std.debug.print("{}\n", .{v.integer}); // ถูกต้อง
}
```

### ขนาดและ Alignment ของ Union

```zig
const std = @import("std");

const MyUnion = union {
    a: u8,
    b: u32,
    c: f64,
    d: bool,
};

pub fn main() void {
    std.debug.print("ขนาดของ MyUnion: {} bytes\n", .{@sizeOf(MyUnion)});
    std.debug.print("Alignment ของ MyUnion: {} bytes\n", .{@alignOf(MyUnion)});

    // ขนาดจะเท่ากับ field ที่ใหญ่ที่สุด (f64 = 8 bytes)
    // alignment จะเท่ากับ alignment ที่มากที่สุด (f64 = 8 bytes)
}
```

### เปรียบเทียบกับ C Union

ใน C:
```c
union Value {
    int integer;
    double floating;
    char boolean;
};

// ใน C เราสามารถอ่าน field ใดก็ได้โดยไม่มี error ที่ compile time
// แต่ผลลัพธ์อาจไม่ถูกต้อง (type punning)
union Value v;
v.integer = 42;
printf("%f\n", v.floating); // ถูกต้องใน C แต่ผลลัพธ์ไม่แน่นอน
```

ใน Zig (Bare Union):
```zig
// Zig ตรวจสอบที่ runtime ในโหมด Debug
// ถ้าพยายามอ่าน field ที่ไม่ได้ active จะเกิด panic ใน Debug mode
```

---

## Step 122: Tagged Union กับ Enum

### Tagged Union คืออะไร?

Tagged union (หรือ discriminated union) คือ union ที่มี enum tag ฝังอยู่ด้วย เพื่อติดตามว่าตอนนี้กำลังใช้ field ไหนอยู่ นี่คือวิธีที่ปลอดภัยในการใช้ union

### การประกาศ Tagged Union

```zig
const std = @import("std");

// Zig สร้าง enum tag ให้อัตโนมัติ
const Value = union(enum) {
    integer: i32,
    float: f64,
    text: []const u8,
    nothing: void,  // field ที่ไม่มีข้อมูล
};

pub fn main() void {
    const v1 = Value{ .integer = 42 };
    const v2 = Value{ .float = 3.14 };
    const v3 = Value{ .text = "สวัสดี" };
    const v4 = Value{ .nothing = {} };

    // ตรวจสอบ tag ปัจจุบัน
    std.debug.print("v1 tag: {s}\n", .{@tagName(v1)});
    std.debug.print("v2 tag: {s}\n", .{@tagName(v2)});
    std.debug.print("v3 tag: {s}\n", .{@tagName(v3)});
    std.debug.print("v4 tag: {s}\n", .{@tagName(v4)});
}
```

### การใช้ Enum ที่กำหนดเองกับ Tagged Union

```zig
const std = @import("std");

// กำหนด enum ก่อน
const ValueTag = enum {
    integer,
    float,
    text,
};

// ใช้ enum ที่กำหนดเองเป็น tag
const Value = union(ValueTag) {
    integer: i32,
    float: f64,
    text: []const u8,
};

pub fn main() void {
    const v = Value{ .integer = 100 };

    // เข้าถึง tag ได้โดยตรง
    const tag: ValueTag = v;
    std.debug.print("tag: {}\n", .{tag});
    std.debug.print("tag == .integer: {}\n", .{tag == .integer});
}
```

### โครงสร้างหน่วยความจำของ Tagged Union

```
Tagged Union { integer: i32, float: f64, text: []const u8 }

+----------+---------------------------+
| tag      | data                      |
| (enum)   | (ใหญ่สุดคือ []const u8)  |
+----------+---------------------------+
| 8 bytes  | 16 bytes                  |
+----------+---------------------------+
รวม = 24 bytes (+ alignment padding)

สำหรับ []const u8 (slice):
- pointer: 8 bytes
- length: 8 bytes
= 16 bytes รวม
```

### Active Field และ Type Safety

```zig
const std = @import("std");

const Shape = union(enum) {
    circle: f64,      // รัศมี
    rectangle: struct { width: f64, height: f64 },
    triangle: struct { base: f64, height: f64 },
};

pub fn area(shape: Shape) f64 {
    return switch (shape) {
        .circle => |r| std.math.pi * r * r,
        .rectangle => |rect| rect.width * rect.height,
        .triangle => |tri| 0.5 * tri.base * tri.height,
    };
}

pub fn main() void {
    const shapes = [_]Shape{
        .{ .circle = 5.0 },
        .{ .rectangle = .{ .width = 4.0, .height = 6.0 } },
        .{ .triangle = .{ .base = 3.0, .height = 8.0 } },
    };

    for (shapes) |shape| {
        std.debug.print("พื้นที่: {d:.2}\n", .{area(shape)});
    }
}
```

---

## Step 123: การใช้ switch กับ Tagged Union

### Pattern Matching ด้วย switch

switch เป็นวิธีหลักในการทำงานกับ tagged union ใน Zig และ compiler จะบังคับให้จัดการทุก case:

```zig
const std = @import("std");

const Expr = union(enum) {
    number: f64,
    add: struct { left: *Expr, right: *Expr },
    multiply: struct { left: *Expr, right: *Expr },
    negate: *Expr,
};

pub fn evaluate(expr: *const Expr) f64 {
    return switch (expr.*) {
        // จับค่าของ field ด้วย |value|
        .number => |n| n,
        .add => |op| evaluate(op.left) + evaluate(op.right),
        .multiply => |op| evaluate(op.left) * evaluate(op.right),
        .negate => |e| -evaluate(e),
    };
}

pub fn main() void {
    // สร้าง expression tree สำหรับ: -(3 + 4) * 2
    var three = Expr{ .number = 3 };
    var four = Expr{ .number = 4 };
    var two = Expr{ .number = 2 };

    var add_expr = Expr{ .add = .{ .left = &three, .right = &four } };
    var neg_expr = Expr{ .negate = &add_expr };
    var mul_expr = Expr{ .multiply = .{ .left = &neg_expr, .right = &two } };

    const result = evaluate(&mul_expr);
    std.debug.print("ผลลัพธ์: {d}\n", .{result}); // -14
}
```

### Exhaustive Switch

Zig บังคับให้ switch ครอบคลุมทุก tag ของ union:

```zig
const std = @import("std");

const Color = union(enum) {
    rgb: struct { r: u8, g: u8, b: u8 },
    hsv: struct { h: f32, s: f32, v: f32 },
    name: []const u8,
};

pub fn describe(color: Color) void {
    switch (color) {
        .rgb => |c| std.debug.print("RGB({}, {}, {})\n", .{ c.r, c.g, c.b }),
        .hsv => |c| std.debug.print("HSV({d:.0}, {d:.0}%, {d:.0}%)\n", .{ c.h, c.s * 100, c.v * 100 }),
        .name => |n| std.debug.print("ชื่อสี: {s}\n", .{n}),
        // ถ้าลืม case ใด compiler จะ error!
    }
}

pub fn main() void {
    describe(Color{ .rgb = .{ .r = 255, .g = 128, .b = 0 } });
    describe(Color{ .hsv = .{ .h = 30, .s = 1.0, .v = 1.0 } });
    describe(Color{ .name = "ส้ม" });
}
```

### การจับค่าด้วย Pointer (Mutable Access)

```zig
const std = @import("std");

const Counter = union(enum) {
    simple: u64,
    named: struct { name: []const u8, value: u64 },
};

pub fn increment(counter: *Counter) void {
    switch (counter.*) {
        // ใช้ |*ptr| เพื่อแก้ไขค่า
        .simple => |*n| n.* += 1,
        .named => |*nc| nc.value += 1,
    }
}

pub fn getValue(counter: Counter) u64 {
    return switch (counter) {
        .simple => |n| n,
        .named => |nc| nc.value,
    };
}

pub fn main() void {
    var c1 = Counter{ .simple = 0 };
    var c2 = Counter{ .named = .{ .name = "ตัวนับหลัก", .value = 10 } };

    increment(&c1);
    increment(&c1);
    increment(&c2);

    std.debug.print("c1 = {}\n", .{getValue(c1)}); // 2
    std.debug.print("c2 = {}\n", .{getValue(c2)}); // 11
}
```

### switch กับ else branch

```zig
const std = @import("std");

const Token = union(enum) {
    number: f64,
    plus,
    minus,
    multiply,
    divide,
    lparen,
    rparen,
    eof,
};

pub fn isOperator(token: Token) bool {
    return switch (token) {
        .plus, .minus, .multiply, .divide => true,
        else => false,  // ครอบคลุมที่เหลือทั้งหมด
    };
}

pub fn main() void {
    const tokens = [_]Token{
        .{ .number = 3.14 },
        .plus,
        .{ .number = 2.71 },
        .multiply,
        .lparen,
    };

    for (tokens) |tok| {
        if (isOperator(tok)) {
            std.debug.print("{s} เป็น operator\n", .{@tagName(tok)});
        } else {
            std.debug.print("{s} ไม่ใช่ operator\n", .{@tagName(tok)});
        }
    }
}
```

---

## Step 124: Union Methods

### การเพิ่ม Methods ให้ Union

Union ใน Zig สามารถมี methods ได้เหมือน struct:

```zig
const std = @import("std");

const Number = union(enum) {
    integer: i64,
    float: f64,
    rational: struct { numerator: i64, denominator: i64 },

    // Method สำหรับแปลงเป็น f64
    pub fn toFloat(self: Number) f64 {
        return switch (self) {
            .integer => |n| @floatFromInt(n),
            .float => |f| f,
            .rational => |r| @as(f64, @floatFromInt(r.numerator)) /
                             @as(f64, @floatFromInt(r.denominator)),
        };
    }

    // Method สำหรับตรวจสอบว่าเป็น zero
    pub fn isZero(self: Number) bool {
        return switch (self) {
            .integer => |n| n == 0,
            .float => |f| f == 0.0,
            .rational => |r| r.numerator == 0,
        };
    }

    // Method สำหรับแสดงผล
    pub fn print(self: Number) void {
        switch (self) {
            .integer => |n| std.debug.print("{}", .{n}),
            .float => |f| std.debug.print("{d}", .{f}),
            .rational => |r| std.debug.print("{}/{}", .{ r.numerator, r.denominator }),
        }
    }

    // Method แบบ mutable
    pub fn negate(self: *Number) void {
        switch (self.*) {
            .integer => |*n| n.* = -n.*,
            .float => |*f| f.* = -f.*,
            .rational => |*r| r.numerator = -r.numerator,
        }
    }
};

pub fn main() void {
    var n1 = Number{ .integer = 42 };
    var n2 = Number{ .float = 3.14 };
    var n3 = Number{ .rational = .{ .numerator = 1, .denominator = 3 } };

    std.debug.print("n1 = "); n1.print(); std.debug.print("\n", .{});
    std.debug.print("n2 = "); n2.print(); std.debug.print("\n", .{});
    std.debug.print("n3 = "); n3.print(); std.debug.print("\n", .{});

    std.debug.print("n1 เป็น float: {d}\n", .{n1.toFloat()});
    std.debug.print("n3 เป็น float: {d:.4}\n", .{n3.toFloat()});

    n1.negate();
    std.debug.print("n1 หลัง negate: "); n1.print(); std.debug.print("\n", .{});
}
```

### Union กับ Interface Pattern

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const Animal = union(enum) {
    dog: struct { name: []const u8, breed: []const u8 },
    cat: struct { name: []const u8, indoor: bool },
    bird: struct { name: []const u8, can_fly: bool },

    pub fn getName(self: Animal) []const u8 {
        return switch (self) {
            .dog => |d| d.name,
            .cat => |c| c.name,
            .bird => |b| b.name,
        };
    }

    pub fn speak(self: Animal) []const u8 {
        return switch (self) {
            .dog => "โฮ่ง!",
            .cat => "เมี้ยว!",
            .bird => "จิ๊บๆ!",
        };
    }

    pub fn describe(self: Animal) void {
        const name = self.getName();
        const sound = self.speak();
        switch (self) {
            .dog => |d| std.debug.print("{s} ({s}) พูดว่า: {s}\n", .{ name, d.breed, sound }),
            .cat => |c| std.debug.print("{s} ({s}) พูดว่า: {s}\n",
                .{ name, if (c.indoor) "แมวในบ้าน" else "แมวนอกบ้าน", sound }),
            .bird => |b| std.debug.print("{s} ({s}) พูดว่า: {s}\n",
                .{ name, if (b.can_fly) "บินได้" else "บินไม่ได้", sound }),
        }
    }
};

pub fn main() void {
    const animals = [_]Animal{
        .{ .dog = .{ .name = "บิ้ง", .breed = "โกลเดน รีทรีฟเวอร์" } },
        .{ .cat = .{ .name = "มิ้ว", .indoor = true } },
        .{ .bird = .{ .name = "แก้ว", .can_fly = true } },
        .{ .bird = .{ .name = "เพนกวิน", .can_fly = false } },
    };

    for (animals) |animal| {
        animal.describe();
    }
}
```

### Static Methods และ Constructors

```zig
const std = @import("std");

const Vector = union(enum) {
    vec2: struct { x: f32, y: f32 },
    vec3: struct { x: f32, y: f32, z: f32 },
    vec4: struct { x: f32, y: f32, z: f32, w: f32 },

    // Constructor functions (static methods)
    pub fn zero2() Vector {
        return .{ .vec2 = .{ .x = 0, .y = 0 } };
    }

    pub fn zero3() Vector {
        return .{ .vec3 = .{ .x = 0, .y = 0, .z = 0 } };
    }

    pub fn length(self: Vector) f32 {
        return switch (self) {
            .vec2 => |v| @sqrt(v.x * v.x + v.y * v.y),
            .vec3 => |v| @sqrt(v.x * v.x + v.y * v.y + v.z * v.z),
            .vec4 => |v| @sqrt(v.x * v.x + v.y * v.y + v.z * v.z + v.w * v.w),
        };
    }

    pub fn dimension(self: Vector) u8 {
        return switch (self) {
            .vec2 => 2,
            .vec3 => 3,
            .vec4 => 4,
        };
    }
};

pub fn main() void {
    const v1 = Vector{ .vec2 = .{ .x = 3, .y = 4 } };
    const v2 = Vector{ .vec3 = .{ .x = 1, .y = 2, .z = 2 } };
    const v0 = Vector.zero2();

    std.debug.print("v1 ความยาว: {d}\n", .{v1.length()}); // 5.0
    std.debug.print("v2 ความยาว: {d}\n", .{v2.length()}); // 3.0
    std.debug.print("v0 มิติ: {}\n", .{v0.dimension()});  // 2
}
```

---

## Step 125: Extern Union (C-Compatible)

### Extern Union คืออะไร?

`extern union` ใช้สำหรับการทำงานร่วมกับ C code โดยมี memory layout ที่เหมือนกับ C union:

```zig
const std = @import("std");

// extern union - C-compatible
const CValue = extern union {
    i: i32,
    u: u32,
    f: f32,
    bytes: [4]u8,
};

pub fn main() void {
    var v = CValue{ .i = -1 };

    std.debug.print("i32: {}\n", .{v.i});
    std.debug.print("u32: {}\n", .{v.u}); // จะเห็น bit pattern ของ -1
    std.debug.print("f32: {}\n", .{v.f}); // NaN หรือค่าแปลกๆ
    std.debug.print("bytes: {any}\n", .{v.bytes});

    // เหมาะสำหรับ type punning ที่ controlled
    v = CValue{ .f = 1.0 };
    std.debug.print("f32 = 1.0 มี bits: {b:0>32}\n", .{v.u});
}
```

### การใช้กับ C Interop

```zig
const std = @import("std");

// จำลอง C struct ที่มี union ข้างใน
const sockaddr_in = extern struct {
    sin_family: u16,
    sin_port: u16,
    sin_addr: extern union {
        s_addr: u32,
        s_bytes: [4]u8,
    },
    sin_zero: [8]u8,
};

pub fn main() void {
    var addr = sockaddr_in{
        .sin_family = 2, // AF_INET
        .sin_port = 0x5000, // port 80 in big-endian
        .sin_addr = .{ .s_bytes = .{ 127, 0, 0, 1 } }, // 127.0.0.1
        .sin_zero = [_]u8{0} ** 8,
    };

    std.debug.print("IP: {}.{}.{}.{}\n", .{
        addr.sin_addr.s_bytes[0],
        addr.sin_addr.s_bytes[1],
        addr.sin_addr.s_bytes[2],
        addr.sin_addr.s_bytes[3],
    });

    std.debug.print("s_addr: 0x{x:0>8}\n", .{addr.sin_addr.s_addr});
}
```

### Extern Union กับ Bitfields

```zig
// จำลองการทำงานกับ hardware registers
const ControlRegister = extern union {
    raw: u32,
    bits: packed struct {
        enable: u1,
        reset: u1,
        mode: u2,
        speed: u4,
        _reserved: u24,
    },
};

pub fn main() void {
    var reg = ControlRegister{ .raw = 0 };

    // ตั้งค่า bits ต่างๆ
    reg.bits.enable = 1;
    reg.bits.mode = 2;
    reg.bits.speed = 8;

    // อ่านค่าทั้งหมดในครั้งเดียว
    _ = reg.raw;
}
```

### ความแตกต่างระหว่าง Union ประเภทต่างๆ

```zig
// Bare union - ไม่มี tag, ไม่ C-compatible
const BareUnion = union {
    a: i32,
    b: f64,
};

// Tagged union - มี tag, ไม่ C-compatible
const TaggedUnion = union(enum) {
    a: i32,
    b: f64,
};

// Extern union - ไม่มี tag, C-compatible
const ExternUnion = extern union {
    a: i32,
    b: f64,
};

// Packed union - ไม่มี tag, packed layout
const PackedUnion = packed union {
    a: u8,
    b: u16,
};
```

---

## Step 126: Packed Union

### Packed Union คืออะไร?

`packed union` มี memory layout ที่แน่นอน (bit-for-bit) โดยไม่มี padding และ alignment requirements:

```zig
const std = @import("std");

// packed union ใช้ขนาดตาม field ที่ใหญ่ที่สุด
const PackedValue = packed union {
    a: u8,
    b: u16,
    c: u32,
};

pub fn main() void {
    std.debug.print("ขนาด PackedValue: {} bytes\n", .{@sizeOf(PackedValue)});
    // = 4 bytes (ขนาดของ u32)

    var v = PackedValue{ .c = 0xDEADBEEF };
    std.debug.print("c: 0x{x}\n", .{v.c});

    // สามารถอ่าน sub-fields ได้
    std.debug.print("a (low byte): 0x{x}\n", .{v.a});  // 0xEF
    std.debug.print("b (low word): 0x{x}\n", .{v.b});  // 0xBEEF
}
```

### ประโยชน์ของ Packed Union

```zig
const std = @import("std");

// ใช้สำหรับ bit manipulation
const IPv4Address = packed union {
    raw: u32,
    bytes: packed struct {
        d: u8,
        c: u8,
        b: u8,
        a: u8,
    },
};

pub fn main() void {
    var ip = IPv4Address{ .bytes = .{ .a = 192, .b = 168, .c = 1, .d = 100 } };

    std.debug.print("IP: {}.{}.{}.{}\n", .{
        ip.bytes.a, ip.bytes.b, ip.bytes.c, ip.bytes.d,
    });
    std.debug.print("Raw: 0x{x:0>8}\n", .{ip.raw});

    // เปรียบเทียบ IP addresses โดยตรง
    const ip2 = IPv4Address{ .bytes = .{ .a = 10, .b = 0, .c = 0, .d = 1 } };
    if (ip.raw == ip2.raw) {
        std.debug.print("IP addresses เหมือนกัน\n", .{});
    } else {
        std.debug.print("IP addresses ต่างกัน\n", .{});
    }
}
```

### Packed Union กับ Floating Point

```zig
const std = @import("std");

// IEEE 754 float breakdown
const Float32 = packed union {
    value: f32,
    bits: packed struct {
        mantissa: u23,
        exponent: u8,
        sign: u1,
    },
    raw: u32,
};

pub fn inspectFloat(f: f32) void {
    const v = Float32{ .value = f };
    std.debug.print("Float: {}\n", .{f});
    std.debug.print("  Sign: {}\n", .{v.bits.sign});
    std.debug.print("  Exponent: {} (actual: {})\n", .{ v.bits.exponent, @as(i32, v.bits.exponent) - 127 });
    std.debug.print("  Mantissa: 0x{x}\n", .{v.bits.mantissa});
    std.debug.print("  Raw bits: 0x{x:0>8}\n\n", .{v.raw});
}

pub fn main() void {
    inspectFloat(1.0);
    inspectFloat(-2.5);
    inspectFloat(0.1);
}
```

---

## Step 127: Union เป็น Type-Safe Variant

### Variant Type Pattern

Union เป็นเครื่องมือที่ดีในการสร้าง type-safe variant types:

```zig
const std = @import("std");

// Variant type สำหรับค่าที่อาจเป็นได้หลายประเภท
const Variant = union(enum) {
    null_value: void,
    boolean: bool,
    integer: i64,
    float: f64,
    string: []const u8,

    pub fn typeName(self: Variant) []const u8 {
        return switch (self) {
            .null_value => "null",
            .boolean => "boolean",
            .integer => "integer",
            .float => "float",
            .string => "string",
        };
    }

    pub fn isTruthy(self: Variant) bool {
        return switch (self) {
            .null_value => false,
            .boolean => |b| b,
            .integer => |n| n != 0,
            .float => |f| f != 0.0,
            .string => |s| s.len > 0,
        };
    }

    pub fn equals(self: Variant, other: Variant) bool {
        const tag_self = std.meta.activeTag(self);
        const tag_other = std.meta.activeTag(other);

        if (tag_self != tag_other) return false;

        return switch (self) {
            .null_value => true,
            .boolean => |b| b == other.boolean,
            .integer => |n| n == other.integer,
            .float => |f| f == other.float,
            .string => |s| std.mem.eql(u8, s, other.string),
        };
    }

    pub fn format(
        self: Variant,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        switch (self) {
            .null_value => try writer.writeAll("null"),
            .boolean => |b| try writer.print("{}", .{b}),
            .integer => |n| try writer.print("{}", .{n}),
            .float => |f| try writer.print("{d}", .{f}),
            .string => |s| try writer.print("\"{s}\"", .{s}),
        }
    }
};

pub fn main() void {
    const values = [_]Variant{
        .{ .null_value = {} },
        .{ .boolean = true },
        .{ .integer = 42 },
        .{ .float = 3.14 },
        .{ .string = "สวัสดี" },
        .{ .integer = 0 },
    };

    for (values) |v| {
        std.debug.print("ค่า: {} | ประเภท: {s} | truthy: {}\n",
            .{ v, v.typeName(), v.isTruthy() });
    }

    const a = Variant{ .integer = 42 };
    const b = Variant{ .integer = 42 };
    const c = Variant{ .float = 42.0 };

    std.debug.print("\na == b: {}\n", .{a.equals(b)});
    std.debug.print("a == c: {}\n", .{a.equals(c)});
}
```

### Optional-like Pattern

```zig
const std = @import("std");

// สร้าง Optional ของตัวเอง
fn Maybe(comptime T: type) type {
    return union(enum) {
        some: T,
        none: void,

        pub fn isSome(self: @This()) bool {
            return switch (self) {
                .some => true,
                .none => false,
            };
        }

        pub fn unwrap(self: @This()) T {
            return switch (self) {
                .some => |v| v,
                .none => @panic("unwrap called on none"),
            };
        }

        pub fn unwrapOr(self: @This(), default: T) T {
            return switch (self) {
                .some => |v| v,
                .none => default,
            };
        }

        pub fn map(self: @This(), comptime U: type, f: fn (T) U) Maybe(U) {
            return switch (self) {
                .some => |v| .{ .some = f(v) },
                .none => .{ .none = {} },
            };
        }
    };
}

fn parseInt(s: []const u8) Maybe(i64) {
    const n = std.fmt.parseInt(i64, s, 10) catch return .{ .none = {} };
    return .{ .some = n };
}

fn double(n: i64) i64 {
    return n * 2;
}

pub fn main() void {
    const MaybeInt = Maybe(i64);

    const a = MaybeInt{ .some = 42 };
    const b = MaybeInt{ .none = {} };

    std.debug.print("a.isSome(): {}\n", .{a.isSome()});
    std.debug.print("b.isSome(): {}\n", .{b.isSome()});
    std.debug.print("a.unwrapOr(0): {}\n", .{a.unwrapOr(0)});
    std.debug.print("b.unwrapOr(0): {}\n", .{b.unwrapOr(0)});

    const doubled = a.map(i64, double);
    std.debug.print("a doubled: {}\n", .{doubled.unwrap()});

    const r1 = parseInt("123");
    const r2 = parseInt("abc");

    std.debug.print("parseInt(\"123\"): {}\n", .{r1.unwrapOr(-1)});
    std.debug.print("parseInt(\"abc\"): {}\n", .{r2.unwrapOr(-1)});
}
```

---

## Step 128: JSON-like Data Structure ด้วย Union

### การสร้าง JSON Value Type

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;
const ArrayList = std.ArrayList;
const StringHashMap = std.StringHashMap;

// JSON Value type
pub const JsonValue = union(enum) {
    null_value: void,
    boolean: bool,
    integer: i64,
    float: f64,
    string: []const u8,
    array: ArrayList(JsonValue),
    object: StringHashMap(JsonValue),

    pub fn deinit(self: *JsonValue, allocator: Allocator) void {
        switch (self.*) {
            .array => |*arr| {
                for (arr.items) |*item| {
                    item.deinit(allocator);
                }
                arr.deinit();
            },
            .object => |*obj| {
                var iter = obj.iterator();
                while (iter.next()) |entry| {
                    var val = entry.value_ptr.*;
                    val.deinit(allocator);
                    allocator.free(entry.key_ptr.*);
                }
                obj.deinit();
            },
            .string => |s| allocator.free(s),
            else => {},
        }
    }

    pub fn isNull(self: JsonValue) bool {
        return switch (self) {
            .null_value => true,
            else => false,
        };
    }

    pub fn asString(self: JsonValue) ?[]const u8 {
        return switch (self) {
            .string => |s| s,
            else => null,
        };
    }

    pub fn asInteger(self: JsonValue) ?i64 {
        return switch (self) {
            .integer => |n| n,
            else => null,
        };
    }

    pub fn asFloat(self: JsonValue) ?f64 {
        return switch (self) {
            .float => |f| f,
            .integer => |n| @floatFromInt(n),
            else => null,
        };
    }

    pub fn print(self: JsonValue, indent: usize) void {
        const spaces = "                                "; // 32 spaces
        const pad = spaces[0..@min(indent * 2, 32)];

        switch (self) {
            .null_value => std.debug.print("null", .{}),
            .boolean => |b| std.debug.print("{}", .{b}),
            .integer => |n| std.debug.print("{}", .{n}),
            .float => |f| std.debug.print("{d}", .{f}),
            .string => |s| std.debug.print("\"{s}\"", .{s}),
            .array => |arr| {
                std.debug.print("[\n", .{});
                for (arr.items, 0..) |item, i| {
                    std.debug.print("{s}  ", .{pad});
                    item.print(indent + 1);
                    if (i < arr.items.len - 1) std.debug.print(",", .{});
                    std.debug.print("\n", .{});
                }
                std.debug.print("{s}]", .{pad});
            },
            .object => |obj| {
                std.debug.print("{{\n", .{});
                var iter = obj.iterator();
                while (iter.next()) |entry| {
                    std.debug.print("{s}  \"{s}\": ", .{ pad, entry.key_ptr.* });
                    entry.value_ptr.*.print(indent + 1);
                    std.debug.print("\n", .{});
                }
                std.debug.print("{s}}}", .{pad});
            },
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง JSON structure
    var person = JsonValue{ .object = StringHashMap(JsonValue).init(allocator) };

    // เพิ่มข้อมูล
    const name_key = try allocator.dupe(u8, "name");
    const age_key = try allocator.dupe(u8, "age");
    const active_key = try allocator.dupe(u8, "active");
    const scores_key = try allocator.dupe(u8, "scores");

    try person.object.put(name_key, JsonValue{ .string = try allocator.dupe(u8, "สมชาย") });
    try person.object.put(age_key, JsonValue{ .integer = 30 });
    try person.object.put(active_key, JsonValue{ .boolean = true });

    var scores = JsonValue{ .array = ArrayList(JsonValue).init(allocator) };
    try scores.array.append(JsonValue{ .integer = 95 });
    try scores.array.append(JsonValue{ .integer = 87 });
    try scores.array.append(JsonValue{ .float = 92.5 });
    try person.object.put(scores_key, scores);

    // แสดงผล
    std.debug.print("JSON Structure:\n", .{});
    person.print(0);
    std.debug.print("\n", .{});

    // เข้าถึงค่า
    if (person.object.get("name")) |name_val| {
        if (name_val.asString()) |name| {
            std.debug.print("ชื่อ: {s}\n", .{name});
        }
    }

    if (person.object.get("age")) |age_val| {
        if (age_val.asInteger()) |age| {
            std.debug.print("อายุ: {}\n", .{age});
        }
    }

    // Cleanup
    person.deinit(allocator);
}
```

### AST Node Pattern

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Abstract Syntax Tree nodes
const AstNode = union(enum) {
    number: f64,
    identifier: []const u8,
    binary_op: *BinaryOp,
    unary_op: *UnaryOp,
    call: *FunctionCall,

    const BinaryOp = struct {
        op: enum { add, sub, mul, div, mod },
        left: AstNode,
        right: AstNode,
    };

    const UnaryOp = struct {
        op: enum { neg, not },
        operand: AstNode,
    };

    const FunctionCall = struct {
        name: []const u8,
        args: []AstNode,
    };

    pub fn evaluate(self: AstNode) f64 {
        return switch (self) {
            .number => |n| n,
            .identifier => |_| 0, // ต้องการ symbol table จริงๆ
            .binary_op => |op| switch (op.op) {
                .add => op.left.evaluate() + op.right.evaluate(),
                .sub => op.left.evaluate() - op.right.evaluate(),
                .mul => op.left.evaluate() * op.right.evaluate(),
                .div => op.left.evaluate() / op.right.evaluate(),
                .mod => @mod(op.left.evaluate(), op.right.evaluate()),
            },
            .unary_op => |op| switch (op.op) {
                .neg => -op.operand.evaluate(),
                .not => if (op.operand.evaluate() == 0) 1 else 0,
            },
            .call => |_| 0, // ต้องการ function registry
        };
    }

    pub fn printTree(self: AstNode, depth: usize) void {
        const indent = "  ";
        var i: usize = 0;
        while (i < depth) : (i += 1) {
            std.debug.print("{s}", .{indent});
        }

        switch (self) {
            .number => |n| std.debug.print("Number({})\n", .{n}),
            .identifier => |id| std.debug.print("Identifier({s})\n", .{id}),
            .binary_op => |op| {
                std.debug.print("BinaryOp({s})\n", .{@tagName(op.op)});
                op.left.printTree(depth + 1);
                op.right.printTree(depth + 1);
            },
            .unary_op => |op| {
                std.debug.print("UnaryOp({s})\n", .{@tagName(op.op)});
                op.operand.printTree(depth + 1);
            },
            .call => |c| {
                std.debug.print("Call({s})\n", .{c.name});
                for (c.args) |arg| {
                    arg.printTree(depth + 1);
                }
            },
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง AST สำหรับ: (3 + 4) * -2
    const three = AstNode{ .number = 3 };
    const four = AstNode{ .number = 4 };
    const two = AstNode{ .number = 2 };

    var add_node = try allocator.create(AstNode.BinaryOp);
    defer allocator.destroy(add_node);
    add_node.* = .{ .op = .add, .left = three, .right = four };

    var neg_node = try allocator.create(AstNode.UnaryOp);
    defer allocator.destroy(neg_node);
    neg_node.* = .{ .op = .neg, .operand = two };

    var mul_node = try allocator.create(AstNode.BinaryOp);
    defer allocator.destroy(mul_node);
    mul_node.* = .{
        .op = .mul,
        .left = AstNode{ .binary_op = add_node },
        .right = AstNode{ .unary_op = neg_node },
    };

    const root = AstNode{ .binary_op = mul_node };

    std.debug.print("AST Tree:\n", .{});
    root.printTree(0);
    std.debug.print("\nผลลัพธ์: {d}\n", .{root.evaluate()});
}
```

---

## Step 129: Result Type Pattern ด้วย Union

### สร้าง Result Type

Result type เป็น pattern ที่ใช้แทน error handling ด้วย union:

```zig
const std = @import("std");

// Generic Result type
fn Result(comptime T: type, comptime E: type) type {
    return union(enum) {
        ok: T,
        err: E,

        const Self = @This();

        pub fn isOk(self: Self) bool {
            return switch (self) {
                .ok => true,
                .err => false,
            };
        }

        pub fn isErr(self: Self) bool {
            return !self.isOk();
        }

        pub fn unwrap(self: Self) T {
            return switch (self) {
                .ok => |v| v,
                .err => @panic("called unwrap on err"),
            };
        }

        pub fn unwrapErr(self: Self) E {
            return switch (self) {
                .ok => @panic("called unwrapErr on ok"),
                .err => |e| e,
            };
        }

        pub fn unwrapOr(self: Self, default: T) T {
            return switch (self) {
                .ok => |v| v,
                .err => default,
            };
        }

        pub fn map(self: Self, comptime U: type, f: fn (T) U) Result(U, E) {
            return switch (self) {
                .ok => |v| .{ .ok = f(v) },
                .err => |e| .{ .err = e },
            };
        }

        pub fn andThen(self: Self, comptime U: type, f: fn (T) Result(U, E)) Result(U, E) {
            return switch (self) {
                .ok => |v| f(v),
                .err => |e| .{ .err = e },
            };
        }
    };
}

// ตัวอย่างการใช้งาน
const ParseError = enum {
    empty_string,
    invalid_character,
    overflow,
};

const ParseResult = Result(i64, ParseError);

fn parseNumber(s: []const u8) ParseResult {
    if (s.len == 0) return .{ .err = .empty_string };

    var result: i64 = 0;
    var negative = false;
    var start: usize = 0;

    if (s[0] == '-') {
        negative = true;
        start = 1;
    }

    for (s[start..]) |c| {
        if (c < '0' or c > '9') return .{ .err = .invalid_character };
        result = result * 10 + @as(i64, c - '0');
    }

    if (negative) result = -result;
    return .{ .ok = result };
}

fn doubleValue(n: i64) i64 {
    return n * 2;
}

pub fn main() void {
    const tests = [_][]const u8{ "42", "-17", "", "12x3", "100" };

    for (tests) |input| {
        const result = parseNumber(input);
        switch (result) {
            .ok => |n| std.debug.print("parse(\"{s}\") = {}\n", .{ input, n }),
            .err => |e| std.debug.print("parse(\"{s}\") = Error({s})\n", .{ input, @tagName(e) }),
        }
    }

    std.debug.print("\n--- Method chaining ---\n", .{});
    const r = parseNumber("21").map(i64, doubleValue);
    std.debug.print("parse(\"21\").map(double) = {}\n", .{r.unwrap()});

    const r2 = parseNumber("abc").map(i64, doubleValue);
    std.debug.print("parse(\"abc\").map(double).isErr() = {}\n", .{r2.isErr()});
}
```

### Error Handling Pattern ที่ซับซ้อน

```zig
const std = @import("std");

// Error types ที่ละเอียด
const DatabaseError = union(enum) {
    connection_failed: struct { host: []const u8, port: u16 },
    query_error: struct { message: []const u8, code: i32 },
    not_found: struct { table: []const u8, id: u64 },
    permission_denied: struct { user: []const u8, operation: []const u8 },

    pub fn describe(self: DatabaseError) void {
        switch (self) {
            .connection_failed => |info| std.debug.print(
                "ไม่สามารถเชื่อมต่อกับ {}:{}\n", .{ info.host, info.port }),
            .query_error => |info| std.debug.print(
                "Query error (code {}): {s}\n", .{ info.code, info.message }),
            .not_found => |info| std.debug.print(
                "ไม่พบ record ในตาราง {s} (id: {})\n", .{ info.table, info.id }),
            .permission_denied => |info| std.debug.print(
                "ผู้ใช้ {s} ไม่มีสิทธิ์ {s}\n", .{ info.user, info.operation }),
        }
    }

    pub fn isRetryable(self: DatabaseError) bool {
        return switch (self) {
            .connection_failed => true,
            .query_error => false,
            .not_found => false,
            .permission_denied => false,
        };
    }
};

const DbResult = union(enum) {
    success: []const u8,
    failure: DatabaseError,
};

fn simulateQuery(query: []const u8) DbResult {
    // จำลองผลลัพธ์ต่างๆ
    if (std.mem.startsWith(u8, query, "SELECT")) {
        return .{ .success = "ผลลัพธ์จาก database" };
    } else if (std.mem.startsWith(u8, query, "DROP")) {
        return .{ .failure = .{ .permission_denied = .{
            .user = "guest",
            .operation = "DROP TABLE",
        } } };
    } else {
        return .{ .failure = .{ .query_error = .{
            .message = "syntax error",
            .code = 1064,
        } } };
    }
}

pub fn main() void {
    const queries = [_][]const u8{
        "SELECT * FROM users",
        "DROP TABLE users",
        "INVALID SYNTAX HERE",
    };

    for (queries) |q| {
        std.debug.print("Query: {s}\n", .{q});
        const result = simulateQuery(q);
        switch (result) {
            .success => |data| std.debug.print("  สำเร็จ: {s}\n", .{data}),
            .failure => |err| {
                std.debug.print("  ล้มเหลว: ", .{});
                err.describe();
                std.debug.print("  สามารถลองใหม่: {}\n", .{err.isRetryable()});
            },
        }
        std.debug.print("\n", .{});
    }
}
```

### Zig's Built-in Error Union

เปรียบเทียบ custom Result type กับ Zig's error union:

```zig
const std = @import("std");

// Custom Result (ดูสวยงามกว่าแต่ verbose กว่า)
const MathError = enum { division_by_zero, overflow };
const MathResult = union(enum) {
    ok: f64,
    err: MathError,
};

fn divCustom(a: f64, b: f64) MathResult {
    if (b == 0) return .{ .err = .division_by_zero };
    return .{ .ok = a / b };
}

// Zig's native error union (กระชับกว่า)
const NativeError = error{ DivisionByZero, Overflow };

fn divNative(a: f64, b: f64) NativeError!f64 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

pub fn main() void {
    // Custom Result
    switch (divCustom(10, 2)) {
        .ok => |v| std.debug.print("Custom: 10/2 = {}\n", .{v}),
        .err => |e| std.debug.print("Custom Error: {}\n", .{e}),
    }

    switch (divCustom(10, 0)) {
        .ok => |v| std.debug.print("Custom: {}\n", .{v}),
        .err => |e| std.debug.print("Custom Error: {}\n", .{e}),
    }

    // Native error union
    const r1 = divNative(10, 2) catch |e| {
        std.debug.print("Native Error: {}\n", .{e});
        return;
    };
    std.debug.print("Native: 10/2 = {}\n", .{r1});

    const r2 = divNative(10, 0) catch |e| blk: {
        std.debug.print("Native Error: {}\n", .{e});
        break :blk 0;
    };
    std.debug.print("Native (default): {}\n", .{r2});
}
```

---

## Step 130: Union กับ Comptime

### Comptime Union Operations

```zig
const std = @import("std");

// สร้าง union ที่ขึ้นอยู่กับ comptime parameter
fn TypedValue(comptime types: []const type) type {
    // สร้าง union fields จาก slice of types
    var fields: [types.len]std.builtin.Type.UnionField = undefined;
    var enum_fields: [types.len]std.builtin.Type.EnumField = undefined;

    inline for (types, 0..) |T, i| {
        fields[i] = .{
            .name = @typeName(T),
            .type = T,
            .alignment = @alignOf(T),
        };
        enum_fields[i] = .{
            .name = @typeName(T),
            .value = i,
        };
    }

    _ = fields;
    _ = enum_fields;

    // ในทางปฏิบัติ comptime union generation ต้องใช้ @Type
    // นี่เป็นตัวอย่างเพื่อแสดง concept
    return struct {
        pub const supported_types = types;
    };
}

// Comptime type checking ใน union operations
fn UnionUtils(comptime U: type) type {
    return struct {
        pub fn hasField(comptime field_name: []const u8) bool {
            const info = @typeInfo(U);
            if (info != .Union) return false;
            for (info.Union.fields) |field| {
                if (std.mem.eql(u8, field.name, field_name)) return true;
            }
            return false;
        }

        pub fn fieldCount() usize {
            const info = @typeInfo(U);
            if (info != .Union) return 0;
            return info.Union.fields.len;
        }

        pub fn fieldNames() [fieldCount()][]const u8 {
            const info = @typeInfo(U);
            var names: [fieldCount()][]const u8 = undefined;
            for (info.Union.fields, 0..) |field, i| {
                names[i] = field.name;
            }
            return names;
        }
    };
}

const MyUnion = union(enum) {
    integer: i32,
    float: f64,
    text: []const u8,
};

pub fn main() void {
    const Utils = UnionUtils(MyUnion);

    std.debug.print("จำนวน fields: {}\n", .{Utils.fieldCount()});
    std.debug.print("มี field 'integer': {}\n", .{Utils.hasField("integer")});
    std.debug.print("มี field 'complex': {}\n", .{Utils.hasField("complex")});

    const names = Utils.fieldNames();
    std.debug.print("Field names: ", .{});
    for (names, 0..) |name, i| {
        if (i > 0) std.debug.print(", ", .{});
        std.debug.print("{s}", .{name});
    }
    std.debug.print("\n", .{});
}
```

### Comptime Generic Union Functions

```zig
const std = @import("std");

// Function ที่ทำงานกับ union ประเภทใดก็ได้
fn tagName(value: anytype) []const u8 {
    const T = @TypeOf(value);
    const info = @typeInfo(T);

    if (info != .Union) {
        @compileError("ต้องการ union type แต่ได้รับ " ++ @typeName(T));
    }

    if (info.Union.tag_type == null) {
        @compileError("ต้องการ tagged union แต่ได้รับ bare union");
    }

    return @tagName(value);
}

// Serialize union เป็น string (comptime-driven)
fn serialize(value: anytype, writer: anytype) !void {
    const T = @TypeOf(value);
    const info = @typeInfo(T);

    if (info != .Union) {
        @compileError("serialize ต้องการ union type");
    }

    try writer.print("{{\"type\":\"{s}\",\"value\":", .{@tagName(value)});

    inline for (info.Union.fields) |field| {
        if (std.mem.eql(u8, field.name, @tagName(value))) {
            const field_val = @field(value, field.name);
            const field_info = @typeInfo(field.type);

            switch (field_info) {
                .Int, .ComptimeInt => try writer.print("{}", .{field_val}),
                .Float, .ComptimeFloat => try writer.print("{d}", .{field_val}),
                .Bool => try writer.print("{}", .{field_val}),
                .Pointer => |ptr| {
                    if (ptr.size == .Slice and ptr.child == u8) {
                        try writer.print("\"{s}\"", .{field_val});
                    } else {
                        try writer.print("\"<pointer>\"", .{});
                    }
                },
                .Void => try writer.writeAll("null"),
                else => try writer.writeAll("\"<complex>\""),
            }
            break;
        }
    }

    try writer.writeAll("}");
}

const Config = union(enum) {
    timeout_ms: u32,
    retry_count: u8,
    host: []const u8,
    debug_mode: bool,
    disabled: void,
};

pub fn main() !void {
    const configs = [_]Config{
        .{ .timeout_ms = 5000 },
        .{ .retry_count = 3 },
        .{ .host = "localhost" },
        .{ .debug_mode = true },
        .{ .disabled = {} },
    };

    var buf: [256]u8 = undefined;
    var stream = std.io.fixedBufferStream(&buf);
    const writer = stream.writer();

    for (configs) |cfg| {
        stream.reset();
        try serialize(cfg, writer);
        const written = stream.getWritten();
        std.debug.print("{s}\n", .{written});
    }

    std.debug.print("\ntag names: ", .{});
    for (configs) |cfg| {
        std.debug.print("{s} ", .{tagName(cfg)});
    }
    std.debug.print("\n", .{});
}
```

### Comptime Union Switching

```zig
const std = @import("std");

// สร้าง dispatch table ที่ compile time
fn makeDispatcher(comptime U: type, comptime ReturnType: type) type {
    return struct {
        const Handler = fn (U) ReturnType;

        pub fn dispatch(handlers: anytype, value: U) ReturnType {
            const info = @typeInfo(U).Union;
            inline for (info.fields) |field| {
                if (std.mem.eql(u8, field.name, @tagName(value))) {
                    if (@hasField(@TypeOf(handlers), field.name)) {
                        const handler = @field(handlers, field.name);
                        return handler(@field(value, field.name));
                    }
                }
            }
            unreachable;
        }
    };
}

const Event = union(enum) {
    click: struct { x: i32, y: i32 },
    keypress: u8,
    resize: struct { width: u32, height: u32 },
    close: void,
};

pub fn main() void {
    const Dispatcher = makeDispatcher(Event, []const u8);

    const handlers = struct {
        pub fn click(c: struct { x: i32, y: i32 }) []const u8 {
            _ = c;
            return "จัดการ click event";
        }
        pub fn keypress(k: u8) []const u8 {
            _ = k;
            return "จัดการ keypress event";
        }
        pub fn resize(r: struct { width: u32, height: u32 }) []const u8 {
            _ = r;
            return "จัดการ resize event";
        }
        pub fn close(_: void) []const u8 {
            return "จัดการ close event";
        }
    };

    const events = [_]Event{
        .{ .click = .{ .x = 100, .y = 200 } },
        .{ .keypress = 'A' },
        .{ .resize = .{ .width = 1920, .height = 1080 } },
        .{ .close = {} },
    };

    for (events) |event| {
        const result = Dispatcher.dispatch(handlers, event);
        std.debug.print("{s}: {s}\n", .{ @tagName(event), result });
    }
}
```

---

## บทสรุปและตาราง Union Types

### เปรียบเทียบ Union Types ทั้งหมด

| ประเภท | Tag | C-Compatible | Padding | ใช้งานหลัก |
|--------|-----|--------------|---------|------------|
| `union` | ไม่มี | ไม่ใช่ | มี | Type punning ภายใน |
| `union(enum)` | ใช่ | ไม่ใช่ | มี | Pattern matching ทั่วไป |
| `extern union` | ไม่มี | ใช่ | ไม่มี | C interop |
| `packed union` | ไม่มี | ไม่ใช่ | ไม่มี | Bit manipulation |

### เมื่อไหรควรใช้ Union ประเภทไหน

```
Bare Union (union):
  ✓ เมื่อต้องการ type-safe access แต่ไม่ต้องการ tag overhead
  ✓ สำหรับ low-level bit manipulation ภายใน
  ✗ ไม่เหมาะสำหรับการใช้งานทั่วไป (ไม่มี safety)

Tagged Union (union(enum)):
  ✓ การใช้งานทั่วไปส่วนใหญ่
  ✓ เมื่อต้องการ pattern matching
  ✓ สำหรับ algebraic data types
  ✓ Result/Option/Maybe types
  ✗ มี overhead จาก tag

Extern Union (extern union):
  ✓ การทำงานร่วมกับ C libraries
  ✓ Hardware register access
  ✓ เมื่อต้องการ exact C memory layout
  ✗ ไม่มี type safety

Packed Union (packed union):
  ✓ Bit-level manipulation
  ✓ การใช้งานกับ packed structs
  ✓ Protocol parsing
  ✗ ใช้งานยากกว่า
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Token Lexer

สร้าง lexer อย่างง่ายโดยใช้ union สำหรับ token types:

```zig
// TODO: implement these
const TokenType = union(enum) {
    // numbers
    integer: i64,
    float: f64,

    // operators
    plus,
    minus,
    star,
    slash,

    // delimiters
    lparen,
    rparen,
    semicolon,

    // identifiers and keywords
    identifier: []const u8,
    keyword_if,
    keyword_while,
    keyword_return,

    // end of file
    eof,

    pub fn isLiteral(self: TokenType) bool {
        // TODO: return true สำหรับ integer, float, identifier
        _ = self;
        return false;
    }

    pub fn precedence(self: TokenType) u8 {
        // TODO: คืน precedence ของ operators
        // plus, minus = 1
        // star, slash = 2
        // อื่นๆ = 0
        _ = self;
        return 0;
    }
};

// ลองสร้าง tokenize function ที่แปลง string เป็น []TokenType
```

### แบบฝึกหัดที่ 2: HTTP Response

สร้าง type สำหรับ HTTP response ที่แสดงผลต่างๆ:

```zig
// TODO: implement this
const HttpResponse = union(enum) {
    // success
    ok: struct {
        status: u16,
        body: []const u8,
        content_type: []const u8,
    },
    // redirects
    redirect: struct {
        status: u16,
        location: []const u8,
    },
    // client errors
    client_error: struct {
        status: u16,
        message: []const u8,
    },
    // server errors
    server_error: struct {
        status: u16,
        message: []const u8,
    },

    pub fn statusCode(self: HttpResponse) u16 {
        // TODO
        _ = self;
        return 0;
    }

    pub fn isSuccess(self: HttpResponse) bool {
        // TODO: return true สำหรับ ok response
        _ = self;
        return false;
    }

    pub fn bodyText(self: HttpResponse) ?[]const u8 {
        // TODO: return body ถ้ามี
        _ = self;
        return null;
    }
};
```

### แบบฝึกหัดที่ 3: Configuration System

สร้าง configuration system ที่รับค่าได้หลายประเภท:

```zig
// TODO: implement this
const ConfigValue = union(enum) {
    string: []const u8,
    integer: i64,
    float: f64,
    boolean: bool,
    list: []const ConfigValue,

    pub fn toString(self: ConfigValue, allocator: std.mem.Allocator) ![]u8 {
        // TODO: แปลงค่าเป็น string representation
        _ = self;
        _ = allocator;
        return error.NotImplemented;
    }

    pub fn coerceToFloat(self: ConfigValue) ?f64 {
        // TODO: พยายามแปลงค่าเป็น f64
        // integer -> float
        // float -> float
        // string ที่เป็นตัวเลข -> float
        // อื่นๆ -> null
        _ = self;
        return null;
    }
};

// สร้าง Config struct ที่ใช้ StringHashMap(ConfigValue)
// พร้อม get/set methods
```

### แบบฝึกหัดที่ 4: State Machine

สร้าง state machine สำหรับ traffic light โดยใช้ union:

```zig
// TODO: implement this
const TrafficLight = union(enum) {
    red: struct { remaining_ms: u32 },
    yellow: struct { transitioning_to: enum { green, red } },
    green: struct { remaining_ms: u32 },

    pub fn next(self: TrafficLight) TrafficLight {
        // TODO: คืน state ถัดไป
        // red -> green (3000ms)
        // green -> yellow (transitioning_to: .red)
        // yellow (to red) -> red (3000ms)
        // yellow (to green) -> green (2000ms)
        _ = self;
        return self;
    }

    pub fn canGo(self: TrafficLight) bool {
        // TODO: return true เมื่อรถสามารถวิ่งได้
        _ = self;
        return false;
    }

    pub fn color(self: TrafficLight) []const u8 {
        // TODO
        _ = self;
        return "unknown";
    }
};
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Union ใน Zig อย่างครอบคลุม:

### สิ่งที่เรียนรู้

**Step 121 - Bare Union:**
- Union พื้นฐานที่แชร์พื้นที่หน่วยความจำ
- ขนาดเท่ากับ field ที่ใหญ่ที่สุด
- ไม่มี type safety - ต้องติดตาม active field เอง

**Step 122 - Tagged Union:**
- Union ที่มี enum tag ติดตาม active field
- ปลอดภัยกว่า bare union มาก
- เป็นมาตรฐานในการใช้งาน union ทั่วไป

**Step 123 - switch กับ Tagged Union:**
- Pattern matching ที่ exhaustive และปลอดภัย
- การจับค่าด้วย `|value|` และ `|*ptr|`
- Compiler บังคับให้จัดการทุก case

**Step 124 - Union Methods:**
- Union มี methods ได้เหมือน struct
- Static methods และ constructor functions
- Interface pattern ผ่าน union

**Step 125 - Extern Union:**
- C-compatible memory layout
- ใช้สำหรับ C interop
- ไม่มี Zig-specific padding

**Step 126 - Packed Union:**
- Bit-level control ของ memory layout
- ไม่มี padding
- เหมาะสำหรับ hardware/protocol work

**Step 127 - Type-Safe Variant:**
- Union เป็น algebraic data type
- Maybe/Optional pattern
- Exhaustive handling

**Step 128 - JSON-like Structures:**
- JSON value representation
- Recursive data structures
- AST nodes

**Step 129 - Result Type Pattern:**
- Error handling ด้วย union
- Functional programming patterns (map, andThen)
- เปรียบเทียบกับ Zig's native error union

**Step 130 - Comptime Union:**
- Type introspection ด้วย `@typeInfo`
- Generic functions ที่ทำงานกับ union ใดก็ได้
- Compile-time dispatch tables

### Best Practices

1. **ใช้ Tagged Union เป็น default** - ปลอดภัยกว่าและ expressive กว่า bare union
2. **จัดการทุก case ใน switch** - อย่าใช้ `else` โดยไม่จำเป็น
3. **ใช้ extern union เฉพาะสำหรับ C interop** - เมื่อต้องการ C memory layout
4. **เพิ่ม methods ที่สื่อความหมาย** - ทำให้ union ใช้งานง่ายขึ้น
5. **Result type แทน null** - ใช้ union ที่ชัดเจนแทนการ return null

### การเปรียบเทียบกับภาษาอื่น

| Zig | Rust | Haskell | TypeScript |
|-----|------|---------|------------|
| `union(enum)` | `enum` | `data` | union types |
| switch exhaustive | match exhaustive | case exhaustive | narrowing |
| comptime generic | generic + trait | type class | generics |
| extern union | repr(C) enum | - | - |

Union ใน Zig เป็นเครื่องมือที่ทรงพลังและยืดหยุ่น ช่วยให้เขียนโค้ดที่ type-safe และมีประสิทธิภาพได้ในเวลาเดียวกัน

---

*ต่อไป: Part 14 - Error Handling ขั้นสูงใน Zig*
