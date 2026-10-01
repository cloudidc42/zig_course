# Part 22: Comptime ขั้นสูงใน Zig (Steps 211-220)

> **ระดับ:** ขั้นสูง (Advanced)  
> **ความต้องการ:** ควรเข้าใจ comptime พื้นฐาน, types, และ generics มาก่อน  
> **เป้าหมาย:** เรียนรู้ metaprogramming, reflection, และ code generation ด้วย comptime

---

## บทนำ: ทำไม Comptime ขั้นสูงถึงสำคัญ?

Zig ไม่มี runtime reflection เหมือน Java หรือ Python แต่มีสิ่งที่ดีกว่า: **comptime reflection** ซึ่งทำงานตอน compile time ทำให้ไม่มี overhead ตอน runtime เลย

ในบทนี้เราจะสร้าง:
- ระบบ serialization/deserialization อัตโนมัติ (คล้าย `encoding/json` ใน Go)
- Type validators ที่ตรวจสอบตอน compile time
- Code generators ที่สร้าง struct และ function จาก type information

```
Comptime Execution Flow:
┌─────────────────────────────────────────────┐
│  Source Code                                │
│    ↓                                        │
│  Parser → AST                               │
│    ↓                                        │
│  Semantic Analysis + Comptime Evaluation    │
│    ↓                                        │
│  IR Generation (with comptime resolved)     │
│    ↓                                        │
│  Machine Code (zero comptime overhead)      │
└─────────────────────────────────────────────┘
```

---

## Step 211: @typeInfo - Introspection ชนิดข้อมูล

### @typeInfo คืออะไร?

`@typeInfo` เป็น builtin function ที่รับ type แล้วคืน `std.builtin.Type` ซึ่งเป็น tagged union ที่บอกรายละเอียดของ type นั้น

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // ดู typeInfo ของ type พื้นฐาน
    const int_info = @typeInfo(i32);
    const float_info = @typeInfo(f64);
    const bool_info = @typeInfo(bool);
    const ptr_info = @typeInfo(*i32);

    // int_info มีชนิดเป็น .int
    switch (int_info) {
        .int => |info| {
            std.debug.print("i32: signedness={}, bits={}\n", .{
                info.signedness,
                info.bits,
            });
        },
        else => unreachable,
    }

    // float_info มีชนิดเป็น .float
    switch (float_info) {
        .float => |info| {
            std.debug.print("f64: bits={}\n", .{info.bits});
        },
        else => unreachable,
    }

    // bool ไม่มี extra info
    switch (bool_info) {
        .bool => std.debug.print("bool: เป็น bool!\n", .{}),
        else => unreachable,
    }

    // pointer มี info เยอะมาก
    switch (ptr_info) {
        .pointer => |info| {
            std.debug.print("*i32: size={s}, is_const={}, child={}\n", .{
                @tagName(info.size),
                info.is_const,
                info.child,
            });
        },
        else => unreachable,
    }
}
```

### TypeInfo Tags ทั้งหมด

`std.builtin.Type` มี variants เหล่านี้:

```zig
// std.builtin.Type - union ทั้งหมด
pub const Type = union(enum) {
    type,           // type ของ types เอง
    void,           // void
    bool,           // bool
    noreturn,       // noreturn
    int,            // i8, u8, i32, u64, ...
    float,          // f16, f32, f64, f128
    pointer,        // *T, []T, [*]T, *const T
    array,          // [N]T
    @"struct",      // struct { ... }
    comptime_float, // comptime_float
    comptime_int,   // comptime_int
    undefined,      // undefined type
    null,           // null type
    optional,       // ?T
    error_union,    // E!T
    error_set,      // error{...}
    @"enum",        // enum { ... }
    @"union",       // union { ... }
    @"fn",          // fn(args) Return
    @"opaque",      // opaque {}
    frame,          // @Frame(func)
    @"anyframe",    // anyframe->T
    vector,         // @Vector(N, T)
    enum_literal,   // .tag_name
};
```

### ฟังก์ชัน typeCheck อัตโนมัติ

```zig
const std = @import("std");

/// ตรวจสอบว่า type เป็น numeric หรือไม่
pub fn isNumeric(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int, .float, .comptime_int, .comptime_float => true,
        else => false,
    };
}

/// ตรวจสอบว่า type เป็น integer หรือไม่
pub fn isInteger(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int, .comptime_int => true,
        else => false,
    };
}

/// ตรวจสอบว่า type เป็น signed integer หรือไม่
pub fn isSignedInt(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int => |info| info.signedness == .signed,
        .comptime_int => true,
        else => false,
    };
}

/// ตรวจสอบว่า type เป็น pointer หรือไม่
pub fn isPointer(comptime T: type) bool {
    return @typeInfo(T) == .pointer;
}

/// ตรวจสอบว่า type เป็น optional หรือไม่
pub fn isOptional(comptime T: type) bool {
    return @typeInfo(T) == .optional;
}

/// ดึง child type ของ optional
pub fn OptionalChild(comptime T: type) type {
    return switch (@typeInfo(T)) {
        .optional => |opt| opt.child,
        else => @compileError("ไม่ใช่ optional type: " ++ @typeName(T)),
    };
}

/// ดึง child type ของ pointer
pub fn PointerChild(comptime T: type) type {
    return switch (@typeInfo(T)) {
        .pointer => |ptr| ptr.child,
        else => @compileError("ไม่ใช่ pointer type: " ++ @typeName(T)),
    };
}

pub fn main() void {
    std.debug.print("isNumeric(i32) = {}\n", .{isNumeric(i32)});
    std.debug.print("isNumeric(f64) = {}\n", .{isNumeric(f64)});
    std.debug.print("isNumeric(bool) = {}\n", .{isNumeric(bool)});
    std.debug.print("isInteger(i32) = {}\n", .{isInteger(i32)});
    std.debug.print("isSignedInt(i32) = {}\n", .{isSignedInt(i32)});
    std.debug.print("isSignedInt(u32) = {}\n", .{isSignedInt(u32)});
    std.debug.print("isPointer(*i32) = {}\n", .{isPointer(*i32)});
    std.debug.print("isOptional(?i32) = {}\n", .{isOptional(?i32)});

    const Child = OptionalChild(?i32);
    std.debug.print("OptionalChild(?i32) = {}\n", .{Child});
}
```

### การใช้ @typeInfo กับ Arrays

```zig
const std = @import("std");

/// คืน element type ของ array หรือ slice
pub fn ElementType(comptime T: type) type {
    return switch (@typeInfo(T)) {
        .array => |arr| arr.child,
        .pointer => |ptr| switch (ptr.size) {
            .Slice => ptr.child,
            else => @compileError("ต้องการ slice ไม่ใช่ pointer"),
        },
        else => @compileError("ไม่ใช่ array หรือ slice: " ++ @typeName(T)),
    };
}

/// คืนความยาวของ array (ใช้กับ array เท่านั้น)
pub fn arrayLen(comptime T: type) usize {
    return switch (@typeInfo(T)) {
        .array => |arr| arr.len,
        else => @compileError("ไม่ใช่ array: " ++ @typeName(T)),
    };
}

pub fn main() void {
    const arr_type = [10]u8;
    const slice_type = []const i32;

    std.debug.print("ElementType([10]u8) = {}\n", .{ElementType(arr_type)});
    std.debug.print("ElementType([]const i32) = {}\n", .{ElementType(slice_type)});
    std.debug.print("arrayLen([10]u8) = {}\n", .{arrayLen(arr_type)});
}
```

---

## Step 212: @typeInfo กับ Struct Fields

### การอ่าน Struct Fields ตอน Comptime

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,
    email: ?[]const u8,
    score: f64 = 0.0,
};

/// พิมพ์ข้อมูล fields ของ struct
pub fn printStructInfo(comptime T: type) void {
    const info = @typeInfo(T);
    if (info != .@"struct") {
        @compileError("ต้องการ struct type เท่านั้น");
    }

    const struct_info = info.@"struct";
    std.debug.print("Struct: {s}\n", .{@typeName(T)});
    std.debug.print("Fields ({d}):\n", .{struct_info.fields.len});

    inline for (struct_info.fields) |field| {
        std.debug.print("  - {s}: {s}", .{ field.name, @typeName(field.type) });
        if (field.default_value) |_| {
            std.debug.print(" (has default)", .{});
        }
        std.debug.print("\n", .{});
    }

    std.debug.print("Decls ({d}):\n", .{struct_info.decls.len});
    inline for (struct_info.decls) |decl| {
        std.debug.print("  - {s}\n", .{decl.name});
    }
}

pub fn main() void {
    printStructInfo(Person);
}
```

### การนับ Fields และ Validate Struct

```zig
const std = @import("std");

/// นับจำนวน fields ของ struct
pub fn fieldCount(comptime T: type) usize {
    return @typeInfo(T).@"struct".fields.len;
}

/// ตรวจสอบว่า struct มี field ชื่อนี้หรือไม่
pub fn hasField(comptime T: type, comptime name: []const u8) bool {
    inline for (@typeInfo(T).@"struct".fields) |field| {
        if (std.mem.eql(u8, field.name, name)) return true;
    }
    return false;
}

/// คืน type ของ field (หรือ error ถ้าไม่มี)
pub fn FieldType(comptime T: type, comptime name: []const u8) type {
    inline for (@typeInfo(T).@"struct".fields) |field| {
        if (std.mem.eql(u8, field.name, name)) return field.type;
    }
    @compileError("ไม่มี field ชื่อ '" ++ name ++ "' ใน " ++ @typeName(T));
}

const Config = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    timeout_ms: u64 = 5000,
};

pub fn main() void {
    std.debug.print("Config fields: {d}\n", .{fieldCount(Config)});
    std.debug.print("has 'host': {}\n", .{hasField(Config, "host")});
    std.debug.print("has 'password': {}\n", .{hasField(Config, "password")});

    const HostType = FieldType(Config, "host");
    std.debug.print("Type of 'host': {s}\n", .{@typeName(HostType)});

    const PortType = FieldType(Config, "port");
    std.debug.print("Type of 'port': {s}\n", .{@typeName(PortType)});
}
```

### IterateFields - วน loop ผ่าน fields

```zig
const std = @import("std");

/// clone struct โดยใช้ comptime reflection
pub fn cloneStruct(comptime T: type, src: T) T {
    var dst: T = undefined;
    inline for (@typeInfo(T).@"struct".fields) |field| {
        @field(dst, field.name) = @field(src, field.name);
    }
    return dst;
}

/// เปรียบเทียบ struct สองตัว field by field
pub fn structEqual(comptime T: type, a: T, b: T) bool {
    inline for (@typeInfo(T).@"struct".fields) |field| {
        const a_val = @field(a, field.name);
        const b_val = @field(b, field.name);
        // ใช้ std.meta.eql สำหรับ deep comparison
        if (!std.meta.eql(a_val, b_val)) return false;
    }
    return true;
}

/// รวม struct สองตัว (override ด้วย non-default values)
pub fn mergeStructs(comptime T: type, base: T, override: T) T {
    var result = base;
    inline for (@typeInfo(T).@"struct".fields) |field| {
        // ถ้า field มี default value ให้ใช้ค่าจาก override เสมอ
        if (field.default_value != null) {
            @field(result, field.name) = @field(override, field.name);
        }
    }
    return result;
}

const Point = struct {
    x: f32,
    y: f32,
    z: f32 = 0.0,
};

pub fn main() void {
    const p1 = Point{ .x = 1.0, .y = 2.0, .z = 3.0 };
    const p2 = cloneStruct(Point, p1);

    std.debug.print("p1 == p2: {}\n", .{structEqual(Point, p1, p2)});

    const p3 = Point{ .x = 1.0, .y = 2.0 }; // z = 0.0 (default)
    const p4 = Point{ .x = 5.0, .y = 6.0, .z = 7.0 };
    const merged = mergeStructs(Point, p3, p4);
    std.debug.print("merged.z: {}\n", .{merged.z}); // 7.0 (from override)
}
```

### @field builtin - การ Access Field ด้วย String

```zig
const std = @import("std");

const Settings = struct {
    width: u32 = 800,
    height: u32 = 600,
    fullscreen: bool = false,
    title: []const u8 = "My App",
};

/// รับค่า field จาก string name (comptime)
pub fn getField(comptime T: type, obj: T, comptime name: []const u8) FieldType(T, name) {
    return @field(obj, name);
}

/// ชื่อทุก fields เป็น array
pub fn fieldNames(comptime T: type) [fieldCount(T)][]const u8 {
    const fields = @typeInfo(T).@"struct".fields;
    var names: [fields.len][]const u8 = undefined;
    inline for (fields, 0..) |field, i| {
        names[i] = field.name;
    }
    return names;
}

fn fieldCount(comptime T: type) usize {
    return @typeInfo(T).@"struct".fields.len;
}

fn FieldType(comptime T: type, comptime name: []const u8) type {
    inline for (@typeInfo(T).@"struct".fields) |field| {
        if (std.mem.eql(u8, field.name, name)) return field.type;
    }
    @compileError("field not found: " ++ name);
}

pub fn main() void {
    const s = Settings{
        .width = 1920,
        .height = 1080,
        .fullscreen = true,
        .title = "Game",
    };

    const names = fieldNames(Settings);
    std.debug.print("Settings fields: ", .{});
    for (names) |name| {
        std.debug.print("{s} ", .{name});
    }
    std.debug.print("\n", .{});

    std.debug.print("width = {}\n", .{getField(Settings, s, "width")});
    std.debug.print("title = {s}\n", .{getField(Settings, s, "title")});
}
```

---

## Step 213: @typeInfo กับ Enum Values

### การ Introspect Enum

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    blue,
    yellow,
    purple,
};

const Direction = enum(u8) {
    north = 0,
    south = 180,
    east = 90,
    west = 270,
};

/// พิมพ์ข้อมูล enum
pub fn printEnumInfo(comptime T: type) void {
    const info = @typeInfo(T);
    if (info != .@"enum") {
        @compileError("ต้องการ enum type");
    }

    const enum_info = info.@"enum";
    std.debug.print("Enum: {s} (tag_type: {s})\n", .{
        @typeName(T),
        @typeName(enum_info.tag_type),
    });
    std.debug.print("is_exhaustive: {}\n", .{enum_info.is_exhaustive});
    std.debug.print("Fields ({d}):\n", .{enum_info.fields.len});

    inline for (enum_info.fields) |field| {
        std.debug.print("  .{s} = {d}\n", .{ field.name, field.value });
    }
}

pub fn main() void {
    printEnumInfo(Color);
    std.debug.print("\n", .{});
    printEnumInfo(Direction);
}
```

### Enum Utilities

```zig
const std = @import("std");

/// คืนจำนวน variants ของ enum
pub fn enumLen(comptime T: type) usize {
    return @typeInfo(T).@"enum".fields.len;
}

/// คืน array ของ enum values ทั้งหมด
pub fn enumValues(comptime T: type) [enumLen(T)]T {
    const fields = @typeInfo(T).@"enum".fields;
    var values: [fields.len]T = undefined;
    inline for (fields, 0..) |field, i| {
        values[i] = @enumFromInt(field.value);
    }
    return values;
}

/// คืน array ของ enum names ทั้งหมด
pub fn enumNames(comptime T: type) [enumLen(T)][]const u8 {
    const fields = @typeInfo(T).@"enum".fields;
    var names: [fields.len][]const u8 = undefined;
    inline for (fields, 0..) |field, i| {
        names[i] = field.name;
    }
    return names;
}

/// แปลง string เป็น enum value
pub fn enumFromString(comptime T: type, name: []const u8) ?T {
    inline for (@typeInfo(T).@"enum".fields) |field| {
        if (std.mem.eql(u8, field.name, name)) {
            return @enumFromInt(field.value);
        }
    }
    return null;
}

/// แปลง enum value เป็น string
pub fn enumToString(value: anytype) []const u8 {
    return @tagName(value);
}

const Season = enum { spring, summer, autumn, winter };

pub fn main() void {
    std.debug.print("Season count: {d}\n", .{enumLen(Season)});

    const values = enumValues(Season);
    std.debug.print("Values: ", .{});
    for (values) |v| {
        std.debug.print("{s} ", .{@tagName(v)});
    }
    std.debug.print("\n", .{});

    const names = enumNames(Season);
    std.debug.print("Names: ", .{});
    for (names) |n| {
        std.debug.print("{s} ", .{n});
    }
    std.debug.print("\n", .{});

    const parsed = enumFromString(Season, "summer");
    if (parsed) |s| {
        std.debug.print("Parsed 'summer': {s}\n", .{enumToString(s)});
    }

    const not_found = enumFromString(Season, "monsoon");
    std.debug.print("Parsed 'monsoon': {}\n", .{not_found});
}
```

### Enum-based Dispatch

```zig
const std = @import("std");

const Op = enum { add, sub, mul, div };

/// สร้าง dispatch table จาก enum ตอน comptime
pub fn makeDispatch(
    comptime E: type,
    comptime Fn: type,
    comptime handlers: anytype,
) [enumLen(E)]Fn {
    const fields = @typeInfo(E).@"enum".fields;
    var table: [fields.len]Fn = undefined;
    inline for (fields, 0..) |field, i| {
        const handler_name = field.name;
        table[i] = @field(handlers, handler_name);
    }
    return table;
}

fn enumLen(comptime T: type) usize {
    return @typeInfo(T).@"enum".fields.len;
}

fn doAdd(a: i32, b: i32) i32 { return a + b; }
fn doSub(a: i32, b: i32) i32 { return a - b; }
fn doMul(a: i32, b: i32) i32 { return a * b; }
fn doDiv(a: i32, b: i32) i32 { return @divTrunc(a, b); }

pub fn main() void {
    const dispatch = makeDispatch(
        Op,
        *const fn (i32, i32) i32,
        .{ .add = &doAdd, .sub = &doSub, .mul = &doMul, .div = &doDiv },
    );

    const ops = [_]Op{ .add, .mul, .sub, .div };
    for (ops) |op| {
        const result = dispatch[@intFromEnum(op)](10, 3);
        std.debug.print("{s}(10, 3) = {d}\n", .{ @tagName(op), result });
    }
}
```

---

## Step 214: std.meta - Utilities สำหรับ Type Manipulation

### std.meta พื้นฐาน

```zig
const std = @import("std");
const meta = std.meta;

pub fn main() void {
    // meta.activeTag - ดู active tag ของ union
    const Value = union(enum) {
        int: i32,
        float: f64,
        bool: bool,
    };

    const v = Value{ .int = 42 };
    std.debug.print("active tag: {s}\n", .{@tagName(meta.activeTag(v))});

    // meta.eql - เปรียบเทียบ deep equality
    const Point = struct { x: i32, y: i32 };
    const p1 = Point{ .x = 1, .y = 2 };
    const p2 = Point{ .x = 1, .y = 2 };
    const p3 = Point{ .x = 3, .y = 4 };
    std.debug.print("p1 == p2: {}\n", .{meta.eql(p1, p2)});
    std.debug.print("p1 == p3: {}\n", .{meta.eql(p1, p3)});

    // meta.fieldNames - คืน tuple ของ field names
    const names = meta.fieldNames(Point);
    std.debug.print("Point field names: {s}, {s}\n", .{ names[0], names[1] });

    // meta.fields - คืน tuple ของ field infos
    const fields = meta.fields(Point);
    std.debug.print("Point fields count: {d}\n", .{fields.len});
}
```

### meta.Tag และ meta.TagType

```zig
const std = @import("std");
const meta = std.meta;

const Status = union(enum) {
    ok: void,
    err: []const u8,
    pending: u64,
};

pub fn main() void {
    // meta.Tag - คืน enum type ของ tagged union
    const StatusTag = meta.Tag(Status);
    std.debug.print("Tag type: {s}\n", .{@typeName(StatusTag)});

    // ใช้กับ switch
    const s = Status{ .pending = 12345 };
    const tag = meta.activeTag(s);
    std.debug.print("Status tag: {s}\n", .{@tagName(tag)});

    // สามารถเปรียบเทียบ tag ได้
    if (tag == .pending) {
        std.debug.print("กำลังรอ: {d}\n", .{s.pending});
    }
}
```

### meta.IntType - สร้าง Integer Type ตามขนาด

```zig
const std = @import("std");
const meta = std.meta;

pub fn main() void {
    // สร้าง integer type ตาม signedness และ bit size
    const U8 = meta.Int(.unsigned, 8);
    const I16 = meta.Int(.signed, 16);
    const U64 = meta.Int(.unsigned, 64);

    std.debug.print("U8: {s}\n", .{@typeName(U8)});
    std.debug.print("I16: {s}\n", .{@typeName(I16)});
    std.debug.print("U64: {s}\n", .{@typeName(U64)});

    // ใช้งานจริง: สร้าง integer ที่ขนาดพอดีกับ enum
    const Color = enum { red, green, blue };
    const ColorInt = meta.Int(.unsigned, @bitSizeOf(Color));
    std.debug.print("ColorInt: {s}\n", .{@typeName(ColorInt)});
}
```

### meta.Elem และ Container Types

```zig
const std = @import("std");
const meta = std.meta;

pub fn main() void {
    // meta.Elem - ดึง element type
    const arr_elem = meta.Elem([5]i32);
    const slice_elem = meta.Elem([]const u8);
    const vec_elem = meta.Elem(@Vector(4, f32));

    std.debug.print("Elem([5]i32): {s}\n", .{@typeName(arr_elem)});
    std.debug.print("Elem([]const u8): {s}\n", .{@typeName(slice_elem)});
    std.debug.print("Elem(@Vector(4,f32)): {s}\n", .{@typeName(vec_elem)});
}
```

### std.meta สำหรับ Struct Manipulation

```zig
const std = @import("std");
const meta = std.meta;

/// สร้าง struct ใหม่โดยเพิ่ม field ใหม่ (comptime)
pub fn AddField(
    comptime Base: type,
    comptime name: []const u8,
    comptime FieldT: type,
    comptime default: ?FieldT,
) type {
    const base_fields = @typeInfo(Base).@"struct".fields;
    var new_fields: [base_fields.len + 1]std.builtin.Type.StructField = undefined;

    // copy fields เดิม
    inline for (base_fields, 0..) |field, i| {
        new_fields[i] = field;
    }

    // เพิ่ม field ใหม่
    const default_val: ?*const anyopaque = if (default) |d| blk: {
        const d_ref = d;
        break :blk @ptrCast(&d_ref);
    } else null;

    new_fields[base_fields.len] = .{
        .name = name,
        .type = FieldT,
        .default_value = default_val,
        .is_comptime = false,
        .alignment = @alignOf(FieldT),
    };

    return @Type(.{
        .@"struct" = .{
            .layout = .auto,
            .fields = &new_fields,
            .decls = &.{},
            .is_tuple = false,
        },
    });
}

const Base = struct {
    id: u32,
    name: []const u8,
};

pub fn main() void {
    const Extended = AddField(Base, "created_at", i64, null);
    const obj = Extended{
        .id = 1,
        .name = "test",
        .created_at = 1700000000,
    };
    std.debug.print("Extended: id={d}, name={s}, created_at={d}\n", .{
        obj.id, obj.name, obj.created_at,
    });
}
```

---

## Step 215: Comptime String Operations

### String Manipulation ตอน Comptime

```zig
const std = @import("std");

/// แปลง snake_case เป็น camelCase ตอน comptime
pub fn snakeToCamel(comptime s: []const u8) []const u8 {
    comptime {
        var result: [s.len]u8 = undefined;
        var i: usize = 0;
        var j: usize = 0;
        var capitalize_next = false;

        while (i < s.len) : (i += 1) {
            if (s[i] == '_') {
                capitalize_next = true;
            } else if (capitalize_next) {
                result[j] = std.ascii.toUpper(s[i]);
                j += 1;
                capitalize_next = false;
            } else {
                result[j] = s[i];
                j += 1;
            }
        }

        return result[0..j];
    }
}

/// แปลง camelCase เป็น snake_case ตอน comptime
pub fn camelToSnake(comptime s: []const u8) []const u8 {
    comptime {
        // นับขนาดก่อน
        var size: usize = s.len;
        for (s) |c| {
            if (std.ascii.isUpper(c)) size += 1;
        }

        var result: [size]u8 = undefined;
        var j: usize = 0;

        for (s) |c| {
            if (std.ascii.isUpper(c)) {
                if (j > 0) {
                    result[j] = '_';
                    j += 1;
                }
                result[j] = std.ascii.toLower(c);
            } else {
                result[j] = c;
            }
            j += 1;
        }

        return result[0..j];
    }
}

/// Compile-time string concat
pub fn concat(comptime a: []const u8, comptime b: []const u8) []const u8 {
    return a ++ b;
}

/// Compile-time string repeat
pub fn repeat(comptime s: []const u8, comptime n: usize) []const u8 {
    comptime {
        var result: [s.len * n]u8 = undefined;
        var i: usize = 0;
        while (i < n) : (i += 1) {
            @memcpy(result[i * s.len ..][0..s.len], s);
        }
        return &result;
    }
}

pub fn main() void {
    const camel = snakeToCamel("hello_world_example");
    std.debug.print("snake->camel: {s}\n", .{camel});

    const snake = camelToSnake("helloWorldExample");
    std.debug.print("camel->snake: {s}\n", .{snake});

    const repeated = repeat("abc", 3);
    std.debug.print("repeat('abc', 3): {s}\n", .{repeated});

    const joined = concat("Hello, ", "World!");
    std.debug.print("concat: {s}\n", .{joined});
}
```

### Comptime Format Strings

```zig
const std = @import("std");

/// สร้าง format string สำหรับ struct ตอน comptime
pub fn makeFormatString(comptime T: type) []const u8 {
    comptime {
        const fields = @typeInfo(T).@"struct".fields;
        var fmt: []const u8 = @typeName(T) ++ "{ ";

        inline for (fields, 0..) |field, i| {
            fmt = fmt ++ field.name ++ "=";

            // เลือก format specifier ตาม type
            fmt = fmt ++ switch (@typeInfo(field.type)) {
                .int, .comptime_int => "{d}",
                .float, .comptime_float => "{d:.2}",
                .bool => "{}",
                .pointer => |ptr| if (ptr.child == u8) "{s}" else "{any}",
                else => "{any}",
            };

            if (i < fields.len - 1) {
                fmt = fmt ++ ", ";
            }
        }

        return fmt ++ " }";
    }
}

const Point3D = struct {
    x: f64,
    y: f64,
    z: f64,
};

const User = struct {
    id: u32,
    name: []const u8,
    active: bool,
};

pub fn main() void {
    const point_fmt = makeFormatString(Point3D);
    std.debug.print("Point3D format: {s}\n", .{point_fmt});

    const user_fmt = makeFormatString(User);
    std.debug.print("User format: {s}\n", .{user_fmt});
}
```

### Comptime Tokenizer

```zig
const std = @import("std");

/// ตัดคำ comptime string ด้วย delimiter
pub fn split(comptime s: []const u8, comptime delim: u8) []const []const u8 {
    comptime {
        // นับจำนวน tokens
        var count: usize = 1;
        for (s) |c| {
            if (c == delim) count += 1;
        }

        var tokens: [count][]const u8 = undefined;
        var token_idx: usize = 0;
        var start: usize = 0;

        for (s, 0..) |c, i| {
            if (c == delim) {
                tokens[token_idx] = s[start..i];
                token_idx += 1;
                start = i + 1;
            }
        }
        tokens[token_idx] = s[start..];

        return &tokens;
    }
}

pub fn main() void {
    const parts = split("a,b,c,d,e", ',');
    std.debug.print("Split result ({d} parts):\n", .{parts.len});
    for (parts) |part| {
        std.debug.print("  '{s}'\n", .{part});
    }
}
```

---

## Step 216: Type Generation ด้วย Comptime

### @Type builtin - สร้าง Type ใหม่

```zig
const std = @import("std");

/// สร้าง struct type จาก field definitions ตอน comptime
pub fn makeStruct(comptime fields_def: anytype) type {
    comptime {
        const fields_info = @typeInfo(@TypeOf(fields_def)).@"struct".fields;
        var fields: [fields_info.len]std.builtin.Type.StructField = undefined;

        inline for (fields_info, 0..) |f, i| {
            const field_type = @field(fields_def, f.name);
            fields[i] = .{
                .name = f.name,
                .type = field_type,
                .default_value = null,
                .is_comptime = false,
                .alignment = @alignOf(field_type),
            };
        }

        return @Type(.{
            .@"struct" = .{
                .layout = .auto,
                .fields = &fields,
                .decls = &.{},
                .is_tuple = false,
            },
        });
    }
}

pub fn main() void {
    // สร้าง struct ชื่อ "dynamic" ตอน comptime
    const DynamicPoint = makeStruct(.{
        .x = f32,
        .y = f32,
        .z = f32,
    });

    const p = DynamicPoint{ .x = 1.0, .y = 2.0, .z = 3.0 };
    std.debug.print("DynamicPoint: ({d}, {d}, {d})\n", .{ p.x, p.y, p.z });

    const DynamicUser = makeStruct(.{
        .id = u64,
        .name = []const u8,
        .score = f64,
    });

    const u = DynamicUser{ .id = 42, .name = "Alice", .score = 99.5 };
    std.debug.print("DynamicUser: id={d}, name={s}, score={d:.1}\n", .{
        u.id, u.name, u.score,
    });
}
```

### Tuple Type Generation

```zig
const std = @import("std");

/// สร้าง tuple type จาก array ของ types
pub fn Tuple(comptime types: []const type) type {
    comptime {
        var fields: [types.len]std.builtin.Type.StructField = undefined;
        inline for (types, 0..) |T, i| {
            // ชื่อ field ของ tuple เป็น "0", "1", "2", ...
            const name = std.fmt.comptimePrint("{d}", .{i});
            fields[i] = .{
                .name = name,
                .type = T,
                .default_value = null,
                .is_comptime = false,
                .alignment = @alignOf(T),
            };
        }

        return @Type(.{
            .@"struct" = .{
                .layout = .auto,
                .fields = &fields,
                .decls = &.{},
                .is_tuple = true,
            },
        });
    }
}

pub fn main() void {
    const T = Tuple(&.{ i32, f64, []const u8 });
    const t = T{ 42, 3.14, "hello" };
    std.debug.print("tuple[0]={d}, [1]={d:.2}, [2]={s}\n", .{
        t.@"0", t.@"1", t.@"2",
    });
}
```

### Enum Generation จาก String Array

```zig
const std = @import("std");

/// สร้าง enum type จาก array ของ string names
pub fn makeEnum(comptime names: []const []const u8) type {
    comptime {
        var fields: [names.len]std.builtin.Type.EnumField = undefined;
        inline for (names, 0..) |name, i| {
            fields[i] = .{
                .name = name,
                .value = i,
            };
        }

        return @Type(.{
            .@"enum" = .{
                .tag_type = std.math.IntFittingRange(0, names.len - 1),
                .fields = &fields,
                .decls = &.{},
                .is_exhaustive = true,
            },
        });
    }
}

pub fn main() void {
    const HttpMethod = makeEnum(&.{
        "GET", "POST", "PUT", "DELETE", "PATCH", "HEAD", "OPTIONS",
    });

    const method: HttpMethod = .GET;
    std.debug.print("method: {s}\n", .{@tagName(method)});
    std.debug.print("POST value: {d}\n", .{@intFromEnum(HttpMethod.POST)});

    const DbOp = makeEnum(&.{ "create", "read", "update", "delete" });
    const op: DbOp = .read;
    std.debug.print("op: {s}\n", .{@tagName(op)});
}
```

### SoA (Structure of Arrays) Generator

```zig
const std = @import("std");

/// แปลง AoS (Array of Structs) เป็น SoA (Struct of Arrays)
pub fn SoA(comptime T: type, comptime capacity: usize) type {
    comptime {
        const fields = @typeInfo(T).@"struct".fields;
        var soa_fields: [fields.len + 1]std.builtin.Type.StructField = undefined;

        // สร้าง field สำหรับแต่ละ column
        inline for (fields, 0..) |field, i| {
            const ArrayType = [capacity]field.type;
            soa_fields[i] = .{
                .name = field.name,
                .type = ArrayType,
                .default_value = null,
                .is_comptime = false,
                .alignment = @alignOf(ArrayType),
            };
        }

        // เพิ่ม field len
        soa_fields[fields.len] = .{
            .name = "len",
            .type = usize,
            .default_value = &@as(usize, 0),
            .is_comptime = false,
            .alignment = @alignOf(usize),
        };

        return @Type(.{
            .@"struct" = .{
                .layout = .auto,
                .fields = &soa_fields,
                .decls = &.{},
                .is_tuple = false,
            },
        });
    }
}

const Particle = struct {
    x: f32,
    y: f32,
    vx: f32,
    vy: f32,
};

pub fn main() void {
    var particles = SoA(Particle, 1000){};

    // เพิ่ม particle
    particles.x[particles.len] = 1.0;
    particles.y[particles.len] = 2.0;
    particles.vx[particles.len] = 0.1;
    particles.vy[particles.len] = 0.2;
    particles.len += 1;

    particles.x[particles.len] = 3.0;
    particles.y[particles.len] = 4.0;
    particles.vx[particles.len] = -0.1;
    particles.vy[particles.len] = 0.5;
    particles.len += 1;

    std.debug.print("Particles: {d}\n", .{particles.len});

    // update ทุก particle (cache friendly!)
    for (0..particles.len) |i| {
        particles.x[i] += particles.vx[i];
        particles.y[i] += particles.vy[i];
    }

    for (0..particles.len) |i| {
        std.debug.print("Particle {d}: ({d:.2}, {d:.2})\n", .{
            i, particles.x[i], particles.y[i],
        });
    }
}
```

---

## Step 217: Comptime Reflection เพื่อ Serialization

### JSON Serializer อัตโนมัติ

```zig
const std = @import("std");

/// Serialize struct เป็น JSON string ด้วย comptime reflection
pub fn toJson(
    value: anytype,
    writer: anytype,
) !void {
    const T = @TypeOf(value);
    try toJsonValue(T, value, writer);
}

fn toJsonValue(
    comptime T: type,
    value: T,
    writer: anytype,
) !void {
    switch (@typeInfo(T)) {
        .bool => try writer.print("{s}", .{if (value) "true" else "false"}),

        .int, .comptime_int => try writer.print("{d}", .{value}),

        .float, .comptime_float => try writer.print("{d}", .{value}),

        .optional => {
            if (value) |v| {
                try toJsonValue(@TypeOf(v), v, writer);
            } else {
                try writer.writeAll("null");
            }
        },

        .pointer => |ptr| {
            if (ptr.size == .Slice and ptr.child == u8) {
                // String
                try writer.writeByte('"');
                for (value) |c| {
                    switch (c) {
                        '"' => try writer.writeAll("\\\""),
                        '\\' => try writer.writeAll("\\\\"),
                        '\n' => try writer.writeAll("\\n"),
                        '\r' => try writer.writeAll("\\r"),
                        '\t' => try writer.writeAll("\\t"),
                        else => try writer.writeByte(c),
                    }
                }
                try writer.writeByte('"');
            } else if (ptr.size == .Slice) {
                // Array/Slice
                try writer.writeByte('[');
                for (value, 0..) |item, i| {
                    if (i > 0) try writer.writeAll(", ");
                    try toJsonValue(ptr.child, item, writer);
                }
                try writer.writeByte(']');
            } else {
                try toJsonValue(ptr.child, value.*, writer);
            }
        },

        .array => |arr| {
            try writer.writeByte('[');
            for (value, 0..) |item, i| {
                if (i > 0) try writer.writeAll(", ");
                try toJsonValue(arr.child, item, writer);
            }
            try writer.writeByte(']');
        },

        .@"struct" => |struct_info| {
            try writer.writeByte('{');
            inline for (struct_info.fields, 0..) |field, i| {
                if (i > 0) try writer.writeAll(", ");
                try writer.writeByte('"');
                try writer.writeAll(field.name);
                try writer.writeAll("\": ");
                try toJsonValue(field.type, @field(value, field.name), writer);
            }
            try writer.writeByte('}');
        },

        .@"enum" => {
            try writer.writeByte('"');
            try writer.writeAll(@tagName(value));
            try writer.writeByte('"');
        },

        else => {
            try writer.writeAll("null");
        },
    }
}

// Test structs
const Address = struct {
    street: []const u8,
    city: []const u8,
    zip: u32,
};

const Person = struct {
    name: []const u8,
    age: u32,
    email: ?[]const u8,
    scores: []const f64,
    address: Address,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    const allocator = std.heap.page_allocator;

    var buffer = std.ArrayList(u8).init(allocator);
    defer buffer.deinit();

    const person = Person{
        .name = "สมชาย",
        .age = 30,
        .email = "somchai@example.com",
        .scores = &.{ 85.5, 92.0, 78.3 },
        .address = .{
            .street = "123 ถ.สุขุมวิท",
            .city = "กรุงเทพ",
            .zip = 10110,
        },
    };

    try toJson(person, buffer.writer());
    try stdout.print("{s}\n", .{buffer.items});

    // ทดสอบ optional null
    buffer.clearRetainingCapacity();
    const person2 = Person{
        .name = "สมหญิง",
        .age = 25,
        .email = null,
        .scores = &.{},
        .address = .{
            .street = "456 ถ.พระราม 4",
            .city = "กรุงเทพ",
            .zip = 10120,
        },
    };
    try toJson(person2, buffer.writer());
    try stdout.print("{s}\n", .{buffer.items});
}
```

### Pretty Print JSON

```zig
const std = @import("std");

pub fn toJsonPretty(
    value: anytype,
    writer: anytype,
    indent: usize,
) !void {
    const T = @TypeOf(value);
    try toJsonValuePretty(T, value, writer, indent);
}

fn writeIndent(writer: anytype, level: usize) !void {
    var i: usize = 0;
    while (i < level) : (i += 1) {
        try writer.writeAll("  ");
    }
}

fn toJsonValuePretty(
    comptime T: type,
    value: T,
    writer: anytype,
    indent: usize,
) !void {
    switch (@typeInfo(T)) {
        .bool => try writer.print("{s}", .{if (value) "true" else "false"}),
        .int, .comptime_int => try writer.print("{d}", .{value}),
        .float, .comptime_float => try writer.print("{d:.4}", .{value}),

        .optional => {
            if (value) |v| {
                try toJsonValuePretty(@TypeOf(v), v, writer, indent);
            } else {
                try writer.writeAll("null");
            }
        },

        .pointer => |ptr| {
            if (ptr.size == .Slice and ptr.child == u8) {
                try writer.writeByte('"');
                try writer.writeAll(value);
                try writer.writeByte('"');
            } else if (ptr.size == .Slice) {
                if (value.len == 0) {
                    try writer.writeAll("[]");
                    return;
                }
                try writer.writeAll("[\n");
                for (value, 0..) |item, i| {
                    try writeIndent(writer, indent + 1);
                    try toJsonValuePretty(ptr.child, item, writer, indent + 1);
                    if (i < value.len - 1) try writer.writeByte(',');
                    try writer.writeByte('\n');
                }
                try writeIndent(writer, indent);
                try writer.writeByte(']');
            } else {
                try toJsonValuePretty(ptr.child, value.*, writer, indent);
            }
        },

        .@"struct" => |struct_info| {
            try writer.writeAll("{\n");
            inline for (struct_info.fields, 0..) |field, i| {
                try writeIndent(writer, indent + 1);
                try writer.writeByte('"');
                try writer.writeAll(field.name);
                try writer.writeAll("\": ");
                try toJsonValuePretty(
                    field.type,
                    @field(value, field.name),
                    writer,
                    indent + 1,
                );
                if (i < struct_info.fields.len - 1) try writer.writeByte(',');
                try writer.writeByte('\n');
            }
            try writeIndent(writer, indent);
            try writer.writeByte('}');
        },

        .@"enum" => {
            try writer.writeByte('"');
            try writer.writeAll(@tagName(value));
            try writer.writeByte('"');
        },

        else => try writer.writeAll("null"),
    }
}

const Config = struct {
    name: []const u8,
    version: u32,
    debug: bool,
    tags: []const []const u8,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const config = Config{
        .name = "my_app",
        .version = 1,
        .debug = true,
        .tags = &.{ "production", "v1", "stable" },
    };

    try toJsonPretty(config, stdout, 0);
    try stdout.writeByte('\n');
}
```

---

## Step 218: Comptime Reflection เพื่อ Deserialization

### JSON Parser พื้นฐาน

```zig
const std = @import("std");

pub const ParseError = error{
    UnexpectedToken,
    MissingField,
    InvalidValue,
    OutOfMemory,
};

/// State machine สำหรับ JSON parsing
pub const JsonParser = struct {
    input: []const u8,
    pos: usize = 0,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, input: []const u8) JsonParser {
        return .{ .input = input, .allocator = allocator };
    }

    fn skipWhitespace(self: *JsonParser) void {
        while (self.pos < self.input.len and
            (self.input[self.pos] == ' ' or
            self.input[self.pos] == '\t' or
            self.input[self.pos] == '\n' or
            self.input[self.pos] == '\r'))
        {
            self.pos += 1;
        }
    }

    fn peek(self: *JsonParser) ?u8 {
        self.skipWhitespace();
        if (self.pos >= self.input.len) return null;
        return self.input[self.pos];
    }

    fn consume(self: *JsonParser) ?u8 {
        self.skipWhitespace();
        if (self.pos >= self.input.len) return null;
        defer self.pos += 1;
        return self.input[self.pos];
    }

    fn expect(self: *JsonParser, c: u8) ParseError!void {
        const got = self.consume() orelse return ParseError.UnexpectedToken;
        if (got != c) return ParseError.UnexpectedToken;
    }

    pub fn parseString(self: *JsonParser) ParseError![]const u8 {
        try self.expect('"');
        const start = self.pos;
        while (self.pos < self.input.len) {
            if (self.input[self.pos] == '\\') {
                self.pos += 2; // skip escape
            } else if (self.input[self.pos] == '"') {
                const result = self.input[start..self.pos];
                self.pos += 1;
                return result;
            } else {
                self.pos += 1;
            }
        }
        return ParseError.UnexpectedToken;
    }

    pub fn parseInt(self: *JsonParser, comptime T: type) ParseError!T {
        self.skipWhitespace();
        const start = self.pos;
        if (self.pos < self.input.len and self.input[self.pos] == '-') {
            self.pos += 1;
        }
        while (self.pos < self.input.len and
            self.input[self.pos] >= '0' and
            self.input[self.pos] <= '9')
        {
            self.pos += 1;
        }
        const num_str = self.input[start..self.pos];
        return std.fmt.parseInt(T, num_str, 10) catch ParseError.InvalidValue;
    }

    pub fn parseFloat(self: *JsonParser, comptime T: type) ParseError!T {
        self.skipWhitespace();
        const start = self.pos;
        if (self.pos < self.input.len and self.input[self.pos] == '-') {
            self.pos += 1;
        }
        while (self.pos < self.input.len and
            (self.input[self.pos] >= '0' and self.input[self.pos] <= '9' or
            self.input[self.pos] == '.' or
            self.input[self.pos] == 'e' or
            self.input[self.pos] == 'E' or
            self.input[self.pos] == '-' or
            self.input[self.pos] == '+'))
        {
            self.pos += 1;
        }
        const num_str = self.input[start..self.pos];
        return std.fmt.parseFloat(T, num_str) catch ParseError.InvalidValue;
    }

    pub fn parseBool(self: *JsonParser) ParseError!bool {
        self.skipWhitespace();
        if (std.mem.startsWith(u8, self.input[self.pos..], "true")) {
            self.pos += 4;
            return true;
        } else if (std.mem.startsWith(u8, self.input[self.pos..], "false")) {
            self.pos += 5;
            return false;
        }
        return ParseError.InvalidValue;
    }

    pub fn parseNull(self: *JsonParser) ParseError!void {
        self.skipWhitespace();
        if (std.mem.startsWith(u8, self.input[self.pos..], "null")) {
            self.pos += 4;
        } else {
            return ParseError.InvalidValue;
        }
    }

    /// Parse struct โดยใช้ comptime reflection
    pub fn parseStruct(self: *JsonParser, comptime T: type) ParseError!T {
        try self.expect('{');

        var result: T = undefined;
        var fields_set = std.StaticBitSet(
            @typeInfo(T).@"struct".fields.len
        ).initEmpty();

        while (true) {
            self.skipWhitespace();
            const c = self.peek() orelse return ParseError.UnexpectedToken;
            if (c == '}') {
                _ = self.consume();
                break;
            }

            const key = try self.parseString();
            self.skipWhitespace();
            try self.expect(':');

            // ค้นหา field ที่ตรงกับ key
            var found = false;
            inline for (@typeInfo(T).@"struct".fields, 0..) |field, i| {
                if (std.mem.eql(u8, field.name, key)) {
                    @field(result, field.name) = try self.parseValue(field.type);
                    fields_set.set(i);
                    found = true;
                }
            }

            if (!found) {
                // skip unknown field
                try self.skipValue();
            }

            self.skipWhitespace();
            if (self.peek() == ',') {
                _ = self.consume();
            }
        }

        // ตรวจสอบว่า required fields ครบหรือไม่
        inline for (@typeInfo(T).@"struct".fields, 0..) |field, i| {
            if (!fields_set.isSet(i)) {
                if (field.default_value) |dv| {
                    const dv_ptr: *const field.type = @ptrCast(@alignCast(dv));
                    @field(result, field.name) = dv_ptr.*;
                } else if (@typeInfo(field.type) == .optional) {
                    @field(result, field.name) = null;
                } else {
                    return ParseError.MissingField;
                }
            }
        }

        return result;
    }

    pub fn parseValue(self: *JsonParser, comptime T: type) ParseError!T {
        return switch (@typeInfo(T)) {
            .bool => self.parseBool(),
            .int => self.parseInt(T),
            .float => self.parseFloat(T),
            .optional => |opt| blk: {
                self.skipWhitespace();
                if (std.mem.startsWith(u8, self.input[self.pos..], "null")) {
                    self.pos += 4;
                    break :blk null;
                }
                break :blk try self.parseValue(opt.child);
            },
            .pointer => |ptr| blk: {
                if (ptr.size == .Slice and ptr.child == u8) {
                    break :blk try self.parseString();
                }
                break :blk ParseError.InvalidValue;
            },
            .@"struct" => self.parseStruct(T),
            else => ParseError.InvalidValue,
        };
    }

    fn skipValue(self: *JsonParser) ParseError!void {
        self.skipWhitespace();
        const c = self.peek() orelse return ParseError.UnexpectedToken;
        switch (c) {
            '"' => _ = try self.parseString(),
            '{' => {
                _ = self.consume();
                var depth: u32 = 1;
                while (depth > 0 and self.pos < self.input.len) {
                    if (self.input[self.pos] == '{') depth += 1
                    else if (self.input[self.pos] == '}') depth -= 1;
                    self.pos += 1;
                }
            },
            '[' => {
                _ = self.consume();
                var depth: u32 = 1;
                while (depth > 0 and self.pos < self.input.len) {
                    if (self.input[self.pos] == '[') depth += 1
                    else if (self.input[self.pos] == ']') depth -= 1;
                    self.pos += 1;
                }
            },
            else => {
                while (self.pos < self.input.len and
                    self.input[self.pos] != ',' and
                    self.input[self.pos] != '}' and
                    self.input[self.pos] != ']')
                {
                    self.pos += 1;
                }
            },
        }
    }
};

/// ฟังก์ชันหลักสำหรับ deserialization
pub fn fromJson(
    comptime T: type,
    allocator: std.mem.Allocator,
    input: []const u8,
) ParseError!T {
    var parser = JsonParser.init(allocator, input);
    return parser.parseValue(T);
}

const ServerConfig = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    timeout: u32 = 30,
    name: ?[]const u8,
};

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    const json1 =
        \\{"host": "localhost", "port": 8080, "debug": true, "name": "test-server"}
    ;

    const config1 = try fromJson(ServerConfig, allocator, json1);
    std.debug.print("host={s}, port={d}, debug={}, timeout={d}, name={s}\n", .{
        config1.host,
        config1.port,
        config1.debug,
        config1.timeout,
        config1.name orelse "null",
    });

    const json2 =
        \\{"host": "prod.example.com", "port": 443, "debug": false, "name": null}
    ;

    const config2 = try fromJson(ServerConfig, allocator, json2);
    std.debug.print("host={s}, port={d}, debug={}, name={}\n", .{
        config2.host,
        config2.port,
        config2.debug,
        config2.name,
    });
}
```

---

## Step 219: Comptime Assertions และ Validation

### @compileError และ @compileLog

```zig
const std = @import("std");

/// Compile-time assertion
pub fn comptimeAssert(comptime condition: bool, comptime msg: []const u8) void {
    if (!condition) {
        @compileError("Assertion failed: " ++ msg);
    }
}

/// ตรวจสอบว่า type เป็น numeric
pub fn requireNumeric(comptime T: type) void {
    const ok = switch (@typeInfo(T)) {
        .int, .float, .comptime_int, .comptime_float => true,
        else => false,
    };
    if (!ok) {
        @compileError(
            "ต้องการ numeric type แต่ได้รับ: " ++ @typeName(T)
        );
    }
}

/// ตรวจสอบว่า struct มี field ที่ต้องการ
pub fn requireFields(comptime T: type, comptime required: []const []const u8) void {
    inline for (required) |name| {
        var found = false;
        inline for (@typeInfo(T).@"struct".fields) |field| {
            if (std.mem.eql(u8, field.name, name)) {
                found = true;
                break;
            }
        }
        if (!found) {
            @compileError(
                @typeName(T) ++ " ต้องมี field '" ++ name ++ "'"
            );
        }
    }
}

/// ตรวจสอบว่า struct implement interface (มี functions ที่ต้องการ)
pub fn requireInterface(comptime T: type, comptime iface: type) void {
    const iface_decls = @typeInfo(iface).@"struct".decls;
    inline for (iface_decls) |decl| {
        if (!@hasDecl(T, decl.name)) {
            @compileError(
                @typeName(T) ++ " ต้อง implement '" ++ decl.name ++ "'"
            );
        }
    }
}

/// ตรวจสอบขนาด struct ไม่เกิน limit
pub fn requireSizeAtMost(comptime T: type, comptime max_bytes: usize) void {
    if (@sizeOf(T) > max_bytes) {
        @compileError(std.fmt.comptimePrint(
            "{s} มีขนาด {} bytes ซึ่งเกิน limit {d} bytes",
            .{ @typeName(T), @sizeOf(T), max_bytes },
        ));
    }
}

// ทดสอบ assertions
pub fn clampedAdd(comptime T: type, a: T, b: T) T {
    comptime requireNumeric(T);
    const max = std.math.maxInt(T);
    const result = @addWithOverflow(a, b);
    return if (result[1] != 0) max else result[0];
}

// Interface definition
const Drawable = struct {
    pub fn draw(self: anytype) void { _ = self; }
    pub fn getBounds(self: anytype) [4]f32 { _ = self; return .{0,0,0,0}; }
};

const Circle = struct {
    x: f32,
    y: f32,
    radius: f32,

    pub fn draw(self: Circle) void {
        std.debug.print("Drawing circle at ({d:.1},{d:.1}) r={d:.1}\n", .{
            self.x, self.y, self.radius,
        });
    }

    pub fn getBounds(self: Circle) [4]f32 {
        return .{
            self.x - self.radius,
            self.y - self.radius,
            self.x + self.radius,
            self.y + self.radius,
        };
    }
};

pub fn main() void {
    // Test requireNumeric
    const result = clampedAdd(u8, 200, 100);
    std.debug.print("clampedAdd(u8, 200, 100) = {d}\n", .{result});

    // Test requireInterface
    comptime requireInterface(Circle, Drawable);
    std.debug.print("Circle implements Drawable!\n", .{});

    const c = Circle{ .x = 5.0, .y = 5.0, .radius = 3.0 };
    c.draw();

    const bounds = c.getBounds();
    std.debug.print("Bounds: ({d:.1},{d:.1},{d:.1},{d:.1})\n", .{
        bounds[0], bounds[1], bounds[2], bounds[3],
    });
}
```

### Type-level Constraints

```zig
const std = @import("std");

/// Constraint: type ต้องมี alignment อย่างน้อย N bytes
pub fn AlignedType(comptime T: type, comptime min_align: u29) type {
    if (@alignOf(T) < min_align) {
        @compileError(std.fmt.comptimePrint(
            "{s} มี alignment {} แต่ต้องการอย่างน้อย {}",
            .{ @typeName(T), @alignOf(T), min_align },
        ));
    }
    return T;
}

/// Constraint: integer type ต้องมีขนาดอย่างน้อย N bits
pub fn MinBitsInt(comptime T: type, comptime min_bits: u8) type {
    const info = @typeInfo(T);
    if (info != .int) {
        @compileError("ต้องการ integer type");
    }
    if (info.int.bits < min_bits) {
        @compileError(std.fmt.comptimePrint(
            "{s} มีแค่ {d} bits แต่ต้องการอย่างน้อย {d} bits",
            .{ @typeName(T), info.int.bits, min_bits },
        ));
    }
    return T;
}

/// Validated generic vector
pub fn Vec(comptime T: type, comptime N: usize) type {
    // Validate ตอน comptime
    comptime {
        switch (@typeInfo(T)) {
            .int, .float => {},
            else => @compileError("Vec ต้องการ numeric type, ได้รับ: " ++ @typeName(T)),
        }
        if (N == 0) @compileError("Vec ต้องมี N > 0");
        if (N > 16) @compileError("Vec ขนาดใหญ่เกิน (max 16)");
    }

    return struct {
        data: [N]T,

        const Self = @This();

        pub fn init(values: [N]T) Self {
            return .{ .data = values };
        }

        pub fn dot(a: Self, b: Self) T {
            var sum: T = 0;
            inline for (0..N) |i| {
                sum += a.data[i] * b.data[i];
            }
            return sum;
        }

        pub fn add(a: Self, b: Self) Self {
            var result: Self = undefined;
            inline for (0..N) |i| {
                result.data[i] = a.data[i] + b.data[i];
            }
            return result;
        }

        pub fn scale(v: Self, s: T) Self {
            var result: Self = undefined;
            inline for (0..N) |i| {
                result.data[i] = v.data[i] * s;
            }
            return result;
        }
    };
}

pub fn main() void {
    const Vec3f = Vec(f32, 3);
    const a = Vec3f.init(.{ 1.0, 2.0, 3.0 });
    const b = Vec3f.init(.{ 4.0, 5.0, 6.0 });

    const dot_product = a.dot(b);
    std.debug.print("dot product: {d}\n", .{dot_product}); // 32

    const sum = a.add(b);
    std.debug.print("sum: [{d},{d},{d}]\n", .{
        sum.data[0], sum.data[1], sum.data[2],
    });

    const scaled = a.scale(2.0);
    std.debug.print("scaled: [{d},{d},{d}]\n", .{
        scaled.data[0], scaled.data[1], scaled.data[2],
    });
}
```

### Compile-time Unit Testing Pattern

```zig
const std = @import("std");

/// Compile-time test runner
pub fn comptimeTest(comptime tests: anytype) void {
    comptime {
        const fields = @typeInfo(@TypeOf(tests)).@"struct".fields;
        var passed: usize = 0;
        var failed: usize = 0;

        inline for (fields) |field| {
            const test_fn = @field(tests, field.name);
            const result = test_fn();
            if (result) {
                passed += 1;
            } else {
                failed += 1;
                // ไม่สามารถ print ตอน comptime แต่สามารถ compile error ได้
                @compileError("Comptime test failed: " ++ field.name);
            }
        }
    }
}

// ทดสอบ math functions ตอน comptime
fn testAdd() bool {
    const result = 2 + 2;
    return result == 4;
}

fn testFibonacci() bool {
    // Fibonacci ตอน comptime
    const fib = struct {
        fn calc(n: u32) u32 {
            if (n <= 1) return n;
            return calc(n - 1) + calc(n - 2);
        }
    };

    return fib.calc(10) == 55;
}

fn testStringEqual() bool {
    const a = "hello";
    const b = "hello";
    const c = "world";
    return std.mem.eql(u8, a, b) and !std.mem.eql(u8, a, c);
}

// รัน tests ตอน comptime
comptime {
    comptimeTest(.{
        .test_add = testAdd,
        .test_fibonacci = testFibonacci,
        .test_string_equal = testStringEqual,
    });
}

pub fn main() void {
    std.debug.print("All comptime tests passed!\n", .{});
}
```

---

## Step 220: Comptime ใน Real-World (JSON Encoder/Decoder)

### Full-featured JSON Library

```zig
const std = @import("std");

// ======== JSON Value Type ========

pub const JsonValue = union(enum) {
    null_value: void,
    bool_value: bool,
    int_value: i64,
    float_value: f64,
    string_value: []const u8,
    array_value: std.ArrayList(JsonValue),
    object_value: std.StringHashMap(JsonValue),

    pub fn deinit(self: *JsonValue, allocator: std.mem.Allocator) void {
        switch (self.*) {
            .array_value => |*arr| {
                for (arr.items) |*item| item.deinit(allocator);
                arr.deinit();
            },
            .object_value => |*obj| {
                var iter = obj.iterator();
                while (iter.next()) |entry| {
                    var val = entry.value_ptr.*;
                    val.deinit(allocator);
                }
                obj.deinit();
            },
            else => {},
        }
    }
};

// ======== Encoder ========

pub const JsonEncoder = struct {
    writer: std.ArrayList(u8).Writer,
    pretty: bool,
    indent_level: usize,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, pretty: bool) !JsonEncoder {
        var buf = std.ArrayList(u8).init(allocator);
        return .{
            .writer = buf.writer(),
            .pretty = pretty,
            .indent_level = 0,
            .allocator = allocator,
        };
    }

    fn writeIndent(self: *JsonEncoder) !void {
        if (!self.pretty) return;
        var i: usize = 0;
        while (i < self.indent_level) : (i += 1) {
            try self.writer.writeAll("  ");
        }
    }

    fn writeNewline(self: *JsonEncoder) !void {
        if (self.pretty) try self.writer.writeByte('\n');
    }

    fn writeSpace(self: *JsonEncoder) !void {
        if (self.pretty) try self.writer.writeByte(' ');
    }

    /// Encode Zig value เป็น JSON ด้วย comptime reflection
    pub fn encode(self: *JsonEncoder, value: anytype) !void {
        const T = @TypeOf(value);
        try self.encodeTyped(T, value);
    }

    fn encodeTyped(self: *JsonEncoder, comptime T: type, value: T) !void {
        switch (@typeInfo(T)) {
            .void => try self.writer.writeAll("null"),
            .null => try self.writer.writeAll("null"),
            .bool => try self.writer.writeAll(if (value) "true" else "false"),

            .int, .comptime_int => {
                try self.writer.print("{d}", .{value});
            },

            .float, .comptime_float => {
                if (std.math.isNan(value) or std.math.isInf(value)) {
                    try self.writer.writeAll("null");
                } else {
                    try self.writer.print("{d}", .{value});
                }
            },

            .optional => {
                if (value) |v| {
                    try self.encodeTyped(@TypeOf(v), v);
                } else {
                    try self.writer.writeAll("null");
                }
            },

            .pointer => |ptr| {
                if (ptr.size == .Slice and ptr.child == u8) {
                    try self.encodeString(value);
                } else if (ptr.size == .Slice) {
                    try self.encodeSlice(ptr.child, value);
                } else if (ptr.size == .One) {
                    try self.encodeTyped(ptr.child, value.*);
                } else {
                    try self.writer.writeAll("null");
                }
            },

            .array => |arr| {
                try self.writer.writeByte('[');
                self.indent_level += 1;
                for (value, 0..) |item, i| {
                    if (i > 0) {
                        try self.writer.writeByte(',');
                    }
                    try self.writeNewline();
                    try self.writeIndent();
                    try self.encodeTyped(arr.child, item);
                }
                self.indent_level -= 1;
                if (value.len > 0) {
                    try self.writeNewline();
                    try self.writeIndent();
                }
                try self.writer.writeByte(']');
            },

            .@"struct" => |struct_info| {
                if (struct_info.is_tuple) {
                    // Tuple -> JSON array
                    try self.writer.writeByte('[');
                    self.indent_level += 1;
                    inline for (struct_info.fields, 0..) |field, i| {
                        if (i > 0) try self.writer.writeByte(',');
                        try self.writeNewline();
                        try self.writeIndent();
                        try self.encodeTyped(field.type, @field(value, field.name));
                    }
                    self.indent_level -= 1;
                    try self.writeNewline();
                    try self.writeIndent();
                    try self.writer.writeByte(']');
                } else {
                    // Struct -> JSON object
                    try self.writer.writeByte('{');
                    self.indent_level += 1;
                    inline for (struct_info.fields, 0..) |field, i| {
                        if (i > 0) try self.writer.writeByte(',');
                        try self.writeNewline();
                        try self.writeIndent();
                        try self.encodeString(field.name);
                        try self.writer.writeByte(':');
                        try self.writeSpace();
                        try self.encodeTyped(field.type, @field(value, field.name));
                    }
                    self.indent_level -= 1;
                    if (struct_info.fields.len > 0) {
                        try self.writeNewline();
                        try self.writeIndent();
                    }
                    try self.writer.writeByte('}');
                }
            },

            .@"enum" => {
                try self.encodeString(@tagName(value));
            },

            .@"union" => |union_info| {
                if (union_info.tag_type != null) {
                    // Tagged union -> JSON object with "__type" key
                    try self.writer.writeByte('{');
                    self.indent_level += 1;
                    try self.writeNewline();
                    try self.writeIndent();
                    try self.encodeString("__type");
                    try self.writer.writeByte(':');
                    try self.writeSpace();
                    try self.encodeString(@tagName(value));

                    inline for (union_info.fields) |field| {
                        if (std.mem.eql(u8, field.name, @tagName(value))) {
                            if (field.type != void) {
                                try self.writer.writeByte(',');
                                try self.writeNewline();
                                try self.writeIndent();
                                try self.encodeString("value");
                                try self.writer.writeByte(':');
                                try self.writeSpace();
                                try self.encodeTyped(field.type, @field(value, field.name));
                            }
                        }
                    }

                    self.indent_level -= 1;
                    try self.writeNewline();
                    try self.writeIndent();
                    try self.writer.writeByte('}');
                } else {
                    try self.writer.writeAll("null");
                }
            },

            else => try self.writer.writeAll("null"),
        }
    }

    fn encodeString(self: *JsonEncoder, s: []const u8) !void {
        try self.writer.writeByte('"');
        for (s) |c| {
            switch (c) {
                '"' => try self.writer.writeAll("\\\""),
                '\\' => try self.writer.writeAll("\\\\"),
                '\n' => try self.writer.writeAll("\\n"),
                '\r' => try self.writer.writeAll("\\r"),
                '\t' => try self.writer.writeAll("\\t"),
                0x00...0x1f => try self.writer.print("\\u{x:0>4}", .{c}),
                else => try self.writer.writeByte(c),
            }
        }
        try self.writer.writeByte('"');
    }

    fn encodeSlice(self: *JsonEncoder, comptime Child: type, slice: []const Child) !void {
        try self.writer.writeByte('[');
        self.indent_level += 1;
        for (slice, 0..) |item, i| {
            if (i > 0) try self.writer.writeByte(',');
            try self.writeNewline();
            try self.writeIndent();
            try self.encodeTyped(Child, item);
        }
        self.indent_level -= 1;
        if (slice.len > 0) {
            try self.writeNewline();
            try self.writeIndent();
        }
        try self.writer.writeByte(']');
    }
};

// ======== Test Data Structures ========

const Role = enum { admin, user, moderator };

const Address = struct {
    street: []const u8,
    city: []const u8,
    country: []const u8,
    zip: ?[]const u8,
};

const User = struct {
    id: u64,
    username: []const u8,
    email: []const u8,
    role: Role,
    age: ?u8,
    score: f64,
    active: bool,
    address: Address,
    tags: []const []const u8,
};

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    var output_buf = std.ArrayList(u8).init(allocator);
    defer output_buf.deinit();

    // สร้าง encoder
    var encoder = JsonEncoder{
        .writer = output_buf.writer(),
        .pretty = true,
        .indent_level = 0,
        .allocator = allocator,
    };

    const users = [_]User{
        .{
            .id = 1001,
            .username = "somchai_dev",
            .email = "somchai@example.com",
            .role = .admin,
            .age = 30,
            .score = 98.5,
            .active = true,
            .address = .{
                .street = "123 Sukhumvit Rd",
                .city = "Bangkok",
                .country = "Thailand",
                .zip = "10110",
            },
            .tags = &.{ "backend", "zig", "senior" },
        },
        .{
            .id = 1002,
            .username = "somying_ui",
            .email = "somying@example.com",
            .role = .user,
            .age = null,
            .score = 75.0,
            .active = false,
            .address = .{
                .street = "456 Rama 4 Rd",
                .city = "Bangkok",
                .country = "Thailand",
                .zip = null,
            },
            .tags = &.{ "frontend", "react" },
        },
    };

    try encoder.encode(users);

    const stdout = std.io.getStdOut().writer();
    try stdout.print("{s}\n", .{output_buf.items});
    try stdout.print("\n--- Size: {d} bytes ---\n", .{output_buf.items.len});
}
```

### Schema Validation ด้วย Comptime

```zig
const std = @import("std");

/// Schema descriptor สำหรับ validation
pub const SchemaField = struct {
    required: bool = true,
    min_len: ?usize = null,
    max_len: ?usize = null,
    min_val: ?i64 = null,
    max_val: ?i64 = null,
};

/// สร้าง validator function สำหรับ struct ตาม schema ตอน comptime
pub fn makeValidator(
    comptime T: type,
    comptime schema: anytype,
) type {
    return struct {
        pub const ValidationError = error{
            RequiredFieldMissing,
            ValueTooShort,
            ValueTooLong,
            ValueTooSmall,
            ValueTooBig,
        };

        pub const Error = struct {
            field: []const u8,
            message: []const u8,
        };

        pub fn validate(
            value: T,
            errors: *std.ArrayList(Error),
        ) !bool {
            var valid = true;

            inline for (@typeInfo(@TypeOf(schema)).@"struct".fields) |schema_field| {
                const field_schema: SchemaField = @field(schema, schema_field.name);
                const field_value = @field(value, schema_field.name);
                const FieldT = @TypeOf(field_value);

                // ตรวจสอบ required
                if (field_schema.required) {
                    switch (@typeInfo(FieldT)) {
                        .optional => {
                            if (field_value == null) {
                                try errors.append(.{
                                    .field = schema_field.name,
                                    .message = "required field is null",
                                });
                                valid = false;
                                continue;
                            }
                        },
                        else => {},
                    }
                }

                // ตรวจสอบ string length
                const actual_value = switch (@typeInfo(FieldT)) {
                    .optional => field_value orelse continue,
                    else => field_value,
                };

                switch (@typeInfo(@TypeOf(actual_value))) {
                    .pointer => |ptr| {
                        if (ptr.size == .Slice and ptr.child == u8) {
                            if (field_schema.min_len) |min| {
                                if (actual_value.len < min) {
                                    try errors.append(.{
                                        .field = schema_field.name,
                                        .message = "value too short",
                                    });
                                    valid = false;
                                }
                            }
                            if (field_schema.max_len) |max| {
                                if (actual_value.len > max) {
                                    try errors.append(.{
                                        .field = schema_field.name,
                                        .message = "value too long",
                                    });
                                    valid = false;
                                }
                            }
                        }
                    },
                    .int => {
                        if (field_schema.min_val) |min| {
                            if (@as(i64, actual_value) < min) {
                                try errors.append(.{
                                    .field = schema_field.name,
                                    .message = "value too small",
                                });
                                valid = false;
                            }
                        }
                        if (field_schema.max_val) |max| {
                            if (@as(i64, actual_value) > max) {
                                try errors.append(.{
                                    .field = schema_field.name,
                                    .message = "value too big",
                                });
                                valid = false;
                            }
                        }
                    },
                    else => {},
                }
            }

            return valid;
        }
    };
}

const UserInput = struct {
    username: []const u8,
    email: []const u8,
    age: u8,
    bio: ?[]const u8,
};

const UserSchema = makeValidator(UserInput, .{
    .username = SchemaField{ .required = true, .min_len = 3, .max_len = 32 },
    .email = SchemaField{ .required = true, .min_len = 5, .max_len = 100 },
    .age = SchemaField{ .required = true, .min_val = 13, .max_val = 120 },
    .bio = SchemaField{ .required = false, .max_len = 500 },
});

pub fn main() !void {
    const allocator = std.heap.page_allocator;
    var errors = std.ArrayList(UserSchema.Error).init(allocator);
    defer errors.deinit();

    // Valid input
    const valid_user = UserInput{
        .username = "somchai",
        .email = "somchai@example.com",
        .age = 25,
        .bio = "นักพัฒนา Zig",
    };

    const is_valid = try UserSchema.validate(valid_user, &errors);
    std.debug.print("Valid user: {}\n", .{is_valid});
    std.debug.print("Errors: {d}\n", .{errors.items.len});

    // Invalid input
    errors.clearRetainingCapacity();
    const invalid_user = UserInput{
        .username = "ab", // too short (< 3)
        .email = "x",    // too short (< 5)
        .age = 10,       // too small (< 13)
        .bio = null,
    };

    const is_invalid = try UserSchema.validate(invalid_user, &errors);
    std.debug.print("\nInvalid user: {}\n", .{is_invalid});
    std.debug.print("Errors ({d}):\n", .{errors.items.len});
    for (errors.items) |err| {
        std.debug.print("  [{s}]: {s}\n", .{ err.field, err.message });
    }
}
```

---

## เมื่อไรควรใช้ Comptime vs Runtime

```
┌─────────────────────────────────────────────────────────────────┐
│                    Comptime vs Runtime Decision                  │
├─────────────────────┬───────────────────────────────────────────┤
│       Comptime      │              Runtime                       │
├─────────────────────┼───────────────────────────────────────────┤
│ Type information    │ User input data                           │
│ Struct field names  │ File contents                             │
│ Enum variants       │ Network responses                         │
│ Code generation     │ Dynamic collections                       │
│ Validation rules    │ Database results                          │
│ Constants           │ Config files (usually)                    │
│ Algorithms on types │ User preferences                          │
│ Format strings      │ Error messages with context               │
└─────────────────────┴───────────────────────────────────────────┘
```

### ตัวอย่าง: เลือก Comptime vs Runtime

```zig
const std = @import("std");

// ✅ COMPTIME: สร้าง lookup table ขนาดคงที่
fn makeAsciiTable() [256]u8 {
    var table: [256]u8 = undefined;
    for (&table, 0..) |*entry, i| {
        entry.* = std.ascii.toUpper(@intCast(i));
    }
    return table;
}

const ASCII_UPPER_TABLE = makeAsciiTable();

// ✅ RUNTIME: อ่านข้อมูลจากผู้ใช้
fn processInput(allocator: std.mem.Allocator, input: []const u8) ![]u8 {
    const result = try allocator.alloc(u8, input.len);
    for (input, 0..) |c, i| {
        result[i] = ASCII_UPPER_TABLE[c]; // ใช้ comptime table ตอน runtime
    }
    return result;
}

// ✅ COMPTIME: กำหนด protocol ของ serialization
fn fieldOrder(comptime T: type) []const []const u8 {
    comptime {
        const fields = @typeInfo(T).@"struct".fields;
        var order: [fields.len][]const u8 = undefined;
        // เรียงตาม size (ใหญ่ก่อน เพื่อ alignment)
        inline for (fields, 0..) |field, i| {
            order[i] = field.name;
        }
        return &order;
    }
}

// ❌ ไม่ควร: ใช้ comptime สำหรับ runtime data
// fn processJson(comptime json_str: []const u8) SomeType { ... }
// ^ นี่ compile ไม่ผ่านถ้า json_str มาจาก runtime

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    const input = "hello, world!";
    const upper = try processInput(allocator, input);
    defer allocator.free(upper);

    std.debug.print("Upper: {s}\n", .{upper});

    const Point = struct { x: f64, y: f64, z: f64 };
    const order = fieldOrder(Point);
    std.debug.print("Field order: ", .{});
    for (order) |name| {
        std.debug.print("{s} ", .{name});
    }
    std.debug.print("\n", .{});
}
```

---

## สรุปเทคนิค Comptime ขั้นสูง

```zig
const std = @import("std");

// ======== Quick Reference ========

// 1. @typeInfo - ดู type information
// const info = @typeInfo(T);
// switch (info) { .int => ..., .struct => ..., ... }

// 2. @Type - สร้าง type ใหม่
// return @Type(.{ .struct = .{ .fields = &fields, ... } });

// 3. @field - access field ด้วย string
// const val = @field(obj, "field_name");

// 4. @hasField, @hasDecl - ตรวจสอบ field/declaration
// if (@hasField(T, "name")) { ... }

// 5. @compileError, @compileLog - compile-time output
// @compileError("error message " ++ @typeName(T));

// 6. comptime - force evaluation ตอน compile time
// const result = comptime computeTable();

// 7. inline for - loop ที่ unroll ตอน compile time
// inline for (fields) |field| { ... }

// 8. std.fmt.comptimePrint - format string ตอน compile time
// const s = std.fmt.comptimePrint("val={d}", .{42});

/// สรุป pattern ทั้งหมดในไฟล์เดียว
pub fn comptimePatterns() void {
    comptime {
        // Pattern 1: Type introspection
        const T = struct { x: i32, y: i32 };
        const info = @typeInfo(T);
        _ = info;

        // Pattern 2: Conditional compilation
        const is_debug = std.builtin.mode == .Debug;
        _ = is_debug;

        // Pattern 3: Field iteration
        inline for (@typeInfo(T).@"struct".fields) |field| {
            _ = field.name;
        }

        // Pattern 4: Type generation
        const GenType = @Type(.{
            .@"struct" = .{
                .layout = .auto,
                .fields = &[_]std.builtin.Type.StructField{},
                .decls = &.{},
                .is_tuple = false,
            },
        });
        _ = GenType;

        // Pattern 5: Compile-time string operations
        const s = "hello" ++ " world";
        _ = s;
    }
}

pub fn main() void {
    comptimePatterns();
    std.debug.print("Comptime patterns loaded!\n", .{});
}
```

---

## แบบฝึกหัด (Exercises)

### ระดับ 1: พื้นฐาน

**Exercise 1.1** - Type Inspector

สร้างฟังก์ชัน `inspectType(comptime T: type) void` ที่พิมพ์ข้อมูลเกี่ยวกับ type:
- ถ้าเป็น integer: บอก signed/unsigned และ bit width
- ถ้าเป็น struct: บอก field names และ types  
- ถ้าเป็น enum: บอก variant names และ values
- ถ้าเป็น optional: บอก child type
- ถ้าเป็นอื่น: บอกว่าเป็น type อะไร

```zig
// เริ่มต้นที่นี่:
pub fn inspectType(comptime T: type) void {
    // TODO: implement
}

pub fn main() void {
    inspectType(i32);
    inspectType(struct { name: []const u8, age: u8 });
    inspectType(enum { red, green, blue });
    inspectType(?[]const u8);
}
```

**Exercise 1.2** - Enum Parser

สร้างฟังก์ชัน generic `parseEnum` ที่รับ string แล้วแปลงเป็น enum value:

```zig
pub fn parseEnum(comptime E: type, s: []const u8) ?E {
    // TODO: ใช้ @typeInfo เพื่อ iterate over enum fields
    // คืน null ถ้าไม่เจอ
}

const Color = enum { red, green, blue };
const dir = parseEnum(Color, "green"); // ?Color.green
```

### ระดับ 2: กลาง

**Exercise 2.1** - Struct Differ

สร้างฟังก์ชัน `diff(comptime T: type, old: T, new: T) []Change` ที่คืนรายการ fields ที่เปลี่ยนแปลง:

```zig
pub const Change = struct {
    field: []const u8,
    // TODO: เพิ่ม old_value และ new_value
};

pub fn diff(comptime T: type, old: T, new: T, allocator: std.mem.Allocator) ![]Change {
    // TODO: implement using @typeInfo
}
```

**Exercise 2.2** - CSV Serializer

สร้าง `toCsv` ที่ serialize slice ของ struct เป็น CSV:

```zig
pub fn toCsv(
    comptime T: type,
    rows: []const T,
    writer: anytype,
) !void {
    // Header row: field names คั่นด้วย comma
    // Data rows: field values คั่นด้วย comma
    // TODO: implement
}

const Product = struct {
    id: u32,
    name: []const u8,
    price: f64,
    in_stock: bool,
};
// Output:
// id,name,price,in_stock
// 1,Widget,9.99,true
// 2,Gadget,24.50,false
```

### ระดับ 3: ขั้นสูง

**Exercise 3.1** - Generic Builder Pattern

สร้าง `Builder(T)` type ที่ generate builder pattern สำหรับ struct ใดๆ:

```zig
pub fn Builder(comptime T: type) type {
    // สร้าง struct ใหม่ที่มี:
    // - fields เหมือนกับ T แต่เป็น optional ทั้งหมด
    // - method set_{field_name} สำหรับแต่ละ field
    // - method build() !T
    return struct {
        // TODO: generate fields and methods
    };
}

// Usage:
const PersonBuilder = Builder(struct { name: []const u8, age: u32 });
var builder = PersonBuilder{};
const person = try builder
    .set_name("สมชาย")
    .set_age(30)
    .build();
```

**Exercise 3.2** - Type-Safe Event System

สร้าง event system ที่ type-safe ด้วย comptime:

```zig
/// สร้าง event system จาก event definitions
pub fn EventSystem(comptime EventDef: type) type {
    // EventDef เป็น struct ที่ fields เป็น event types
    // เช่น: struct { click: struct { x: i32, y: i32 }, keypress: struct { key: u8 } }
    return struct {
        // TODO: สร้าง:
        // - on(comptime event_name, handler) - register handler
        // - emit(event_name, event_data) - emit event
    };
}

const Events = EventSystem(.{
    .click = struct { x: i32, y: i32 },
    .keypress = struct { key: u8, modifiers: u8 },
    .resize = struct { width: u32, height: u32 },
});
```

**Exercise 3.3** - BSON Serializer

สร้าง BSON (Binary JSON) serializer ที่ใช้ comptime reflection:

```zig
pub fn toBson(
    value: anytype,
    writer: anytype,
) !void {
    // BSON format:
    // \x01 double
    // \x02 string (int32 length + bytes + \x00)
    // \x08 bool
    // \x10 int32
    // \x12 int64
    // \x03 document (nested struct)
    // TODO: implement
}
```

---

## สรุป Part 22

ในบทนี้เราได้เรียนรู้:

| Topic | Builtin/Tool | Use Case |
|-------|-------------|----------|
| Type introspection | `@typeInfo` | ดู metadata ของ type |
| Field access | `@field`, `@hasField` | dynamic field access |
| Type creation | `@Type` | สร้าง type ใหม่ตอน comptime |
| Compile errors | `@compileError` | validation ตอน compile |
| Meta utilities | `std.meta.*` | type manipulation helpers |
| String operations | `++`, `comptime` block | compile-time strings |
| Serialization | comptime + `inline for` | auto JSON/CSV encoder |
| Validation | comptime assertions | type-safe constraints |

### Key Insights

1. **Comptime = Zero Runtime Cost**: ทุกอย่างที่คำนวณตอน comptime ไม่มี overhead ตอน runtime เลย

2. **`inline for` คือหัวใจ**: การ iterate fields/variants ต้องใช้ `inline for` เสมอ เพราะ type ต้องรู้ตอน compile

3. **`@Type` เป็น escape hatch**: เมื่อต้องการสร้าง type แบบ dynamic สามารถใช้ `@Type` ได้

4. **Comptime ≠ Macros**: ใน Zig ไม่มี macros แต่ comptime ทรงพลังกว่าในหลายด้าน เพราะเป็น Zig จริงๆ

5. **Error ตอน Compile ดีกว่าตอน Runtime**: ใช้ `@compileError` เพื่อ fail early

### What's Next

- **Part 23**: Async/Await และ Event Loop ใน Zig
- **Part 24**: SIMD และ Vector Operations
- **Part 25**: Zig Build System ขั้นสูง

---

*Part 22 จาก Zig Programming Course*  
*Steps 211-220: Comptime ขั้นสูง*
