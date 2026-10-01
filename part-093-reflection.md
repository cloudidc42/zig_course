# Part 93: Reflection ใน Zig

## Steps 921-930

---

## Step 921: Reflection ใน Zig ผ่าน Comptime

### Reflection คืออะไร?

Reflection คือความสามารถของโปรแกรมในการตรวจสอบและแก้ไขโครงสร้างของตัวเองขณะทำงาน ใน Zig เราทำผ่าน `comptime` และ `@typeInfo`

### ความแตกต่างจากภาษาอื่น

```
Java/C#:  Runtime reflection - ช้า, มี overhead
Python:   Runtime reflection - dynamic typing
Zig:      Compile-time reflection - ไม่มี runtime overhead
```

### เครื่องมือหลักของ Reflection ใน Zig

```zig
const std = @import("std");

pub fn main() void {
    // 1. @typeInfo - ดู type information
    const IntInfo = @typeInfo(i32);
    std.debug.print("i32 bits: {}\n", .{IntInfo.Int.bits});

    // 2. @typeName - ชื่อของ type
    std.debug.print("Type name: {s}\n", .{@typeName(i32)});

    // 3. @hasField - ตรวจสอบว่า struct มี field
    const Point = struct { x: i32, y: i32 };
    std.debug.print("Has x: {}\n", .{@hasField(Point, "x")});
    std.debug.print("Has z: {}\n", .{@hasField(Point, "z")});

    // 4. @hasDecl - ตรวจสอบว่า type มี declaration
    std.debug.print("Has format: {}\n", .{@hasDecl(std.fmt, "format")});

    // 5. @field - access field by name
    var p = Point{ .x = 10, .y = 20 };
    const x_val = @field(p, "x");
    std.debug.print("x = {}\n", .{x_val});
    @field(p, "y") = 30;
    std.debug.print("y = {}\n", .{p.y});

    // 6. @TypeOf - get type of expression
    const val: i32 = 42;
    std.debug.print("Type of val: {s}\n", .{@typeName(@TypeOf(val))});

    // 7. @sizeOf - size in bytes
    std.debug.print("Size of i64: {} bytes\n", .{@sizeOf(i64)});

    // 8. @alignOf - alignment
    std.debug.print("Align of i64: {} bytes\n", .{@alignOf(i64)});
}
```

### Reflection Use Cases

```zig
const std = @import("std");

// Use case 1: Universal printer
pub fn print(value: anytype) void {
    const T = @TypeOf(value);
    const info = @typeInfo(T);

    switch (info) {
        .Int => std.debug.print("{}", .{value}),
        .Float => std.debug.print("{d}", .{value}),
        .Bool => std.debug.print("{}", .{value}),
        .Pointer => |p| {
            if (p.size == .Slice and p.child == u8) {
                std.debug.print("{s}", .{value});
            } else {
                std.debug.print("{*}", .{value});
            }
        },
        .Struct => {
            std.debug.print("{s}{{", .{@typeName(T)});
            const struct_info = @typeInfo(T);
            switch (struct_info) {
                .Struct => |s| {
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) std.debug.print(", ", .{});
                        std.debug.print(".{s} = ", .{field.name});
                        print(@field(value, field.name));
                    }
                },
                else => {},
            }
            std.debug.print("}}", .{});
        },
        .Optional => {
            if (value) |v| {
                print(v);
            } else {
                std.debug.print("null", .{});
            }
        },
        .Enum => {
            std.debug.print("{s}", .{@tagName(value)});
        },
        else => std.debug.print("<{s}>", .{@typeName(T)}),
    }
    std.debug.print("\n", .{});
}

// Use case 2: Deep equality check
pub fn deepEqual(a: anytype, b: @TypeOf(a)) bool {
    const T = @TypeOf(a);
    const info = @typeInfo(T);

    return switch (info) {
        .Int, .Float, .Bool, .Enum => a == b,
        .Pointer => |p| {
            if (p.size == .Slice) {
                if (a.len != b.len) return false;
                for (a, b) |ai, bi| {
                    if (!deepEqual(ai, bi)) return false;
                }
                return true;
            }
            return a == b;
        },
        .Struct => |s| {
            inline for (s.fields) |field| {
                if (!deepEqual(@field(a, field.name), @field(b, field.name))) {
                    return false;
                }
            }
            return true;
        },
        .Optional => {
            if (a == null and b == null) return true;
            if (a == null or b == null) return false;
            return deepEqual(a.?, b.?);
        },
        else => false,
    };
}
```

---

## Step 922: @typeInfo Deep Dive

### TypeInfo Union

```zig
const std = @import("std");
const builtin = @import("builtin");

// @typeInfo returns std.builtin.Type which is a tagged union
pub fn exploreTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("\n=== Type: {s} ===\n", .{@typeName(T)});

    switch (info) {
        .Type => std.debug.print("  Kind: Type\n", .{}),
        .Void => std.debug.print("  Kind: Void\n", .{}),
        .Bool => std.debug.print("  Kind: Bool\n", .{}),
        .NoReturn => std.debug.print("  Kind: NoReturn\n", .{}),
        .Int => |i| {
            std.debug.print("  Kind: Int\n", .{});
            std.debug.print("  Bits: {}\n", .{i.bits});
            std.debug.print("  Signedness: {}\n", .{i.signedness});
        },
        .Float => |f| {
            std.debug.print("  Kind: Float\n", .{});
            std.debug.print("  Bits: {}\n", .{f.bits});
        },
        .Pointer => |p| {
            std.debug.print("  Kind: Pointer\n", .{});
            std.debug.print("  Size: {}\n", .{p.size});
            std.debug.print("  Is const: {}\n", .{p.is_const});
            std.debug.print("  Is volatile: {}\n", .{p.is_volatile});
            std.debug.print("  Child type: {s}\n", .{@typeName(p.child)});
        },
        .Array => |a| {
            std.debug.print("  Kind: Array\n", .{});
            std.debug.print("  Length: {}\n", .{a.len});
            std.debug.print("  Child type: {s}\n", .{@typeName(a.child)});
        },
        .Struct => |s| {
            std.debug.print("  Kind: Struct\n", .{});
            std.debug.print("  Fields ({}):\n", .{s.fields.len});
            inline for (s.fields) |field| {
                std.debug.print("    .{s}: {s}\n", .{ field.name, @typeName(field.type) });
            }
            std.debug.print("  Decls ({}):\n", .{s.decls.len});
            inline for (s.decls) |decl| {
                std.debug.print("    {s}\n", .{decl.name});
            }
        },
        .Enum => |e| {
            std.debug.print("  Kind: Enum\n", .{});
            std.debug.print("  Tag type: {s}\n", .{@typeName(e.tag_type)});
            std.debug.print("  Fields ({}):\n", .{e.fields.len});
            inline for (e.fields) |field| {
                std.debug.print("    .{s} = {}\n", .{ field.name, field.value });
            }
        },
        .Union => |u| {
            std.debug.print("  Kind: Union\n", .{});
            if (u.tag_type) |tt| {
                std.debug.print("  Tag type: {s}\n", .{@typeName(tt)});
            }
            std.debug.print("  Fields ({}):\n", .{u.fields.len});
            inline for (u.fields) |field| {
                std.debug.print("    .{s}: {s}\n", .{ field.name, @typeName(field.type) });
            }
        },
        .Fn => |f| {
            std.debug.print("  Kind: Function\n", .{});
            std.debug.print("  Params ({}):\n", .{f.params.len});
            inline for (f.params) |param| {
                if (param.type) |pt| {
                    std.debug.print("    {s}\n", .{@typeName(pt)});
                }
            }
            if (f.return_type) |rt| {
                std.debug.print("  Return type: {s}\n", .{@typeName(rt)});
            }
        },
        .Optional => |o| {
            std.debug.print("  Kind: Optional\n", .{});
            std.debug.print("  Child type: {s}\n", .{@typeName(o.child)});
        },
        .ErrorUnion => |eu| {
            std.debug.print("  Kind: ErrorUnion\n", .{});
            std.debug.print("  Error type: {s}\n", .{@typeName(eu.error_set)});
            std.debug.print("  Payload type: {s}\n", .{@typeName(eu.payload)});
        },
        .ErrorSet => {
            std.debug.print("  Kind: ErrorSet\n", .{});
        },
        .Vector => |v| {
            std.debug.print("  Kind: Vector\n", .{});
            std.debug.print("  Length: {}\n", .{v.len});
        },
        else => std.debug.print("  Kind: Other\n", .{}),
    }
}

pub fn typeInfoDemo() void {
    exploreTypeInfo(i32);
    exploreTypeInfo(f64);
    exploreTypeInfo([]const u8);
    exploreTypeInfo([5]u8);

    const MyStruct = struct {
        id: u64,
        name: []const u8,
        active: bool,

        pub fn greet(self: @This()) void {
            _ = self;
        }
    };
    exploreTypeInfo(MyStruct);

    const MyEnum = enum(u8) { Alpha, Beta, Gamma };
    exploreTypeInfo(MyEnum);

    const MyUnion = union(enum) {
        int_val: i32,
        float_val: f64,
        str_val: []const u8,
    };
    exploreTypeInfo(MyUnion);
}
```

---

## Step 923: Struct Field Reflection

### Struct Introspection

```zig
const std = @import("std");

// Get all field names of a struct
pub fn fieldNames(comptime T: type) []const []const u8 {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var names: [s.fields.len][]const u8 = undefined;
            inline for (s.fields, 0..) |field, i| {
                names[i] = field.name;
            }
            return &names;
        },
        else => @compileError("fieldNames requires a struct type"),
    }
}

// Get field types
pub fn fieldTypes(comptime T: type) []const type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var types: [s.fields.len]type = undefined;
            inline for (s.fields, 0..) |field, i| {
                types[i] = field.type;
            }
            return &types;
        },
        else => @compileError("fieldTypes requires a struct type"),
    }
}

// Get field count
pub fn fieldCount(comptime T: type) comptime_int {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| return s.fields.len,
        else => @compileError("fieldCount requires a struct type"),
    }
}

// Check if field exists
pub fn hasField(comptime T: type, comptime name: []const u8) bool {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                if (std.mem.eql(u8, field.name, name)) return true;
            }
            return false;
        },
        else => return false,
    }
}

// Get field type by name
pub fn FieldType(comptime T: type, comptime field_name: []const u8) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                if (std.mem.eql(u8, field.name, field_name)) {
                    return field.type;
                }
            }
            @compileError("Field not found: " ++ field_name);
        },
        else => @compileError("FieldType requires a struct type"),
    }
}

// Iterate over fields dynamically
pub fn iterateFields(comptime T: type, value: T, callback: fn ([]const u8, anytype) void) void {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                callback(field.name, @field(value, field.name));
            }
        },
        else => {},
    }
}

// Convert struct to map
pub fn structToMap(
    comptime T: type,
    value: T,
    allocator: std.mem.Allocator,
) !std.StringHashMap([]const u8) {
    var map = std.StringHashMap([]const u8).init(allocator);
    errdefer map.deinit();

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                const val = @field(value, field.name);
                const str = try std.fmt.allocPrint(allocator, "{any}", .{val});
                try map.put(field.name, str);
            }
        },
        else => return error.NotAStruct,
    }

    return map;
}

// Clone a struct
pub fn cloneStruct(comptime T: type, value: T) T {
    var result: T = undefined;
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                @field(result, field.name) = @field(value, field.name);
            }
        },
        else => result = value,
    }
    return result;
}

// Merge two structs (second overrides first)
pub fn mergeStructs(comptime T: type, base: T, override: T) T {
    return override; // Simple version - just use override
    _ = base;
}

// Diff two structs - find changed fields
pub fn diffStructs(comptime T: type, a: T, b: T, allocator: std.mem.Allocator) ![][]const u8 {
    var changed = std.ArrayList([]const u8).init(allocator);
    errdefer changed.deinit();

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                const va = @field(a, field.name);
                const vb = @field(b, field.name);
                if (!std.meta.eql(va, vb)) {
                    try changed.append(field.name);
                }
            }
        },
        else => {},
    }

    return changed.toOwnedSlice();
}

const Config = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    max_retries: u32,
    timeout_ms: u64,
};

pub fn structReflectionExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Get field names
    const names = comptime fieldNames(Config);
    std.debug.print("Config fields: ", .{});
    for (names) |name| std.debug.print("{s} ", .{name});
    std.debug.print("\n", .{});

    // Field count
    std.debug.print("Field count: {}\n", .{comptime fieldCount(Config)});

    // Check field existence
    std.debug.print("Has 'host': {}\n", .{comptime hasField(Config, "host")});
    std.debug.print("Has 'username': {}\n", .{comptime hasField(Config, "username")});

    // Convert to map
    const config = Config{
        .host = "localhost",
        .port = 8080,
        .debug = true,
        .max_retries = 3,
        .timeout_ms = 30000,
    };

    var map = try structToMap(Config, config, allocator);
    defer {
        var it = map.valueIterator();
        while (it.next()) |v| allocator.free(v.*);
        map.deinit();
    }

    var iter = map.iterator();
    while (iter.next()) |entry| {
        std.debug.print("{s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }

    // Diff structs
    const config2 = Config{
        .host = "0.0.0.0",
        .port = 9090,
        .debug = false,
        .max_retries = 5,
        .timeout_ms = 30000,
    };

    const changed = try diffStructs(Config, config, config2, allocator);
    defer allocator.free(changed);
    std.debug.print("Changed fields: ", .{});
    for (changed) |f| std.debug.print("{s} ", .{f});
    std.debug.print("\n", .{});
}
```

---

## Step 924: Enum Reflection

### Enum Introspection

```zig
const std = @import("std");

// Get enum field count
pub fn enumCount(comptime E: type) comptime_int {
    return @typeInfo(E).Enum.fields.len;
}

// Get all enum values
pub fn enumValues(comptime E: type) []const E {
    const info = @typeInfo(E).Enum;
    comptime var values: [info.fields.len]E = undefined;
    inline for (info.fields, 0..) |field, i| {
        values[i] = @enumFromInt(field.value);
    }
    return &values;
}

// Get all enum names
pub fn enumNames(comptime E: type) []const []const u8 {
    const info = @typeInfo(E).Enum;
    comptime var names: [info.fields.len][]const u8 = undefined;
    inline for (info.fields, 0..) |field, i| {
        names[i] = field.name;
    }
    return &names;
}

// Convert string to enum
pub fn enumFromString(comptime E: type, name: []const u8) ?E {
    const info = @typeInfo(E).Enum;
    inline for (info.fields) |field| {
        if (std.mem.eql(u8, field.name, name)) {
            return @enumFromInt(field.value);
        }
    }
    return null;
}

// Get next enum value
pub fn enumNext(comptime E: type, current: E) ?E {
    const info = @typeInfo(E).Enum;
    const current_val = @intFromEnum(current);
    var found = false;

    inline for (info.fields) |field| {
        if (found) {
            return @enumFromInt(field.value);
        }
        if (field.value == current_val) {
            found = true;
        }
    }
    return null;
}

// Check if enum value is valid
pub fn isValidEnum(comptime E: type, value: @typeInfo(E).Enum.tag_type) bool {
    const info = @typeInfo(E).Enum;
    inline for (info.fields) |field| {
        if (field.value == value) return true;
    }
    return false;
}

// Enum to map
pub fn enumToMap(
    comptime E: type,
    comptime V: type,
    values: []const V,
) !void {
    const info = @typeInfo(E).Enum;
    if (values.len != info.fields.len) {
        return error.LengthMismatch;
    }
}

// Generate switch cases at comptime
pub fn enumDispatch(
    comptime E: type,
    comptime handlers: anytype,
    value: E,
) !void {
    const info = @typeInfo(E).Enum;
    inline for (info.fields) |field| {
        if (@intFromEnum(value) == field.value) {
            const handler_name = field.name;
            if (@hasField(@TypeOf(handlers), handler_name)) {
                const handler = @field(handlers, handler_name);
                try handler();
                return;
            }
        }
    }
    return error.NoHandler;
}

// Enum set using a bitmask
pub fn EnumSet(comptime E: type) type {
    const count = enumCount(E);
    const MaskType = std.meta.Int(.unsigned, count);

    return struct {
        mask: MaskType = 0,

        const Self = @This();

        pub fn add(self: *Self, value: E) void {
            const idx = @intFromEnum(value);
            self.mask |= @as(MaskType, 1) << @intCast(idx);
        }

        pub fn remove(self: *Self, value: E) void {
            const idx = @intFromEnum(value);
            self.mask &= ~(@as(MaskType, 1) << @intCast(idx));
        }

        pub fn contains(self: Self, value: E) bool {
            const idx = @intFromEnum(value);
            return (self.mask >> @intCast(idx)) & 1 == 1;
        }

        pub fn count(self: Self) u32 {
            return @popCount(self.mask);
        }
    };
}

const Status = enum(u8) { Active, Inactive, Pending, Banned };
const Day = enum { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday };

pub fn enumReflectionExample() !void {
    // Enum count
    std.debug.print("Status count: {}\n", .{enumCount(Status)});

    // Enum values
    const statuses = enumValues(Status);
    std.debug.print("Status values: ", .{});
    for (statuses) |s| std.debug.print("{} ", .{s});
    std.debug.print("\n", .{});

    // Enum names
    const names = enumNames(Status);
    std.debug.print("Status names: ", .{});
    for (names) |n| std.debug.print("{s} ", .{n});
    std.debug.print("\n", .{});

    // String to enum
    const s = enumFromString(Status, "Active");
    std.debug.print("Active from string: {?}\n", .{s});

    // Next enum value
    const next = enumNext(Status, .Active);
    std.debug.print("Next after Active: {?}\n", .{next});

    // Validity check
    std.debug.print("Is 0 valid: {}\n", .{isValidEnum(Status, 0)});
    std.debug.print("Is 10 valid: {}\n", .{isValidEnum(Status, 10)});

    // Enum dispatch
    try enumDispatch(Status, .{
        .Active = struct {
            fn f() !void { std.debug.print("Handling Active\n", .{}); }
        }.f,
        .Inactive = struct {
            fn f() !void { std.debug.print("Handling Inactive\n", .{}); }
        }.f,
        .Pending = struct {
            fn f() !void { std.debug.print("Handling Pending\n", .{}); }
        }.f,
        .Banned = struct {
            fn f() !void { std.debug.print("Handling Banned\n", .{}); }
        }.f,
    }, .Active);

    // EnumSet
    var workdays = EnumSet(Day){};
    workdays.add(.Monday);
    workdays.add(.Tuesday);
    workdays.add(.Wednesday);
    workdays.add(.Thursday);
    workdays.add(.Friday);
    std.debug.print("Workdays count: {}\n", .{workdays.count()});
    std.debug.print("Contains Monday: {}\n", .{workdays.contains(.Monday)});
    std.debug.print("Contains Saturday: {}\n", .{workdays.contains(.Saturday)});
}
```

---

## Step 925: Function Reflection

### Function Introspection

```zig
const std = @import("std");

// Get function parameter types
pub fn paramTypes(comptime F: type) []const type {
    const info = @typeInfo(F);
    switch (info) {
        .Fn => |f| {
            comptime var types: [f.params.len]type = undefined;
            inline for (f.params, 0..) |param, i| {
                types[i] = param.type orelse @compileError("Param has no type");
            }
            return &types;
        },
        else => @compileError("paramTypes requires a function type"),
    }
}

// Get return type
pub fn returnType(comptime F: type) type {
    const info = @typeInfo(F);
    switch (info) {
        .Fn => |f| return f.return_type orelse @compileError("Function has no return type"),
        else => @compileError("returnType requires a function type"),
    }
}

// Get param count
pub fn paramCount(comptime F: type) comptime_int {
    const info = @typeInfo(F);
    switch (info) {
        .Fn => |f| return f.params.len,
        else => @compileError("paramCount requires a function type"),
    }
}

// Check if function is generic
pub fn isGenericFn(comptime F: type) bool {
    const info = @typeInfo(F);
    switch (info) {
        .Fn => |f| return f.is_generic,
        else => return false,
    }
}

// Function memoization via comptime
pub fn Memoize(comptime F: anytype) type {
    const FnType = @TypeOf(F);
    const info = @typeInfo(FnType);
    switch (info) {
        .Fn => |f| {
            if (f.params.len != 1) @compileError("Memoize supports single-param functions");
            const Input = f.params[0].type orelse @compileError("Param has no type");
            const Output = f.return_type orelse @compileError("No return type");

            return struct {
                cache: std.AutoHashMap(Input, Output),
                allocator: std.mem.Allocator,

                const Self = @This();

                pub fn init(allocator: std.mem.Allocator) Self {
                    return Self{
                        .cache = std.AutoHashMap(Input, Output).init(allocator),
                        .allocator = allocator,
                    };
                }

                pub fn deinit(self: *Self) void {
                    self.cache.deinit();
                }

                pub fn call(self: *Self, input: Input) !Output {
                    if (self.cache.get(input)) |cached| {
                        return cached;
                    }
                    const result = F(input);
                    try self.cache.put(input, result);
                    return result;
                }
            };
        },
        else => @compileError("Memoize requires a function"),
    }
}

// Partial application
pub fn partial(comptime F: anytype, comptime first_arg: anytype) fn (anytype) @TypeOf(F(first_arg, @as(anytype, undefined))) {
    _ = F;
    _ = first_arg;
    // Simplified - return a closure-like function
    return struct {
        fn call(rest: anytype) @TypeOf(F(first_arg, rest)) {
            return F(first_arg, rest);
        }
    }.call;
}

// Function composition
pub fn compose(comptime f: anytype, comptime g: anytype) fn (anytype) anytype {
    _ = f;
    _ = g;
    return struct {
        fn call(x: anytype) @TypeOf(f(g(x))) {
            return f(g(x));
        }
    }.call;
}

// Retry wrapper
pub fn withRetry(
    comptime max_retries: u32,
    f: anytype,
    args: anytype,
) !@TypeOf(@call(.auto, f, args)) {
    var attempts: u32 = 0;
    while (attempts < max_retries) : (attempts += 1) {
        return @call(.auto, f, args) catch |e| {
            if (attempts + 1 >= max_retries) return e;
            std.debug.print("Attempt {} failed: {}\n", .{ attempts + 1, e });
            continue;
        };
    }
    return error.MaxRetriesExceeded;
}

// Timing wrapper
pub fn timed(f: anytype, args: anytype) !struct {
    result: @TypeOf(@call(.auto, f, args)),
    elapsed_ns: u64,
} {
    const start = std.time.nanoTimestamp();
    const result = try @call(.auto, f, args);
    const elapsed = @as(u64, @intCast(std.time.nanoTimestamp() - start));
    return .{ .result = result, .elapsed_ns = elapsed };
}

fn slowFibonacci(n: u32) u64 {
    if (n <= 1) return n;
    return slowFibonacci(n - 1) + slowFibonacci(n - 2);
}

pub fn functionReflectionExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Function param count
    const FnType = @TypeOf(slowFibonacci);
    std.debug.print("Param count: {}\n", .{paramCount(FnType)});

    // Memoized fibonacci
    const MemoFib = Memoize(slowFibonacci);
    var memo_fib = MemoFib.init(allocator);
    defer memo_fib.deinit();

    const result1 = try memo_fib.call(10);
    const result2 = try memo_fib.call(10); // From cache
    std.debug.print("Fib(10) = {} (computed)\n", .{result1});
    std.debug.print("Fib(10) = {} (cached)\n", .{result2});

    // Timed execution
    const timed_result = try timed(slowFibonacci, .{30});
    std.debug.print("Fib(30) = {} in {}ns\n", .{ timed_result.result, timed_result.elapsed_ns });
}
```

---

## Step 926: Type Traits Implementation

### Type Traits ใน Zig

```zig
const std = @import("std");

// Type trait: isInteger
pub fn isInteger(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Int, .ComptimeInt => true,
        else => false,
    };
}

// Type trait: isFloat
pub fn isFloat(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Float, .ComptimeFloat => true,
        else => false,
    };
}

// Type trait: isNumeric
pub fn isNumeric(comptime T: type) bool {
    return isInteger(T) or isFloat(T);
}

// Type trait: isString
pub fn isString(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Pointer => |p| p.size == .Slice and p.child == u8,
        .Array => |a| a.child == u8,
        else => false,
    };
}

// Type trait: isStruct
pub fn isStruct(comptime T: type) bool {
    return @typeInfo(T) == .Struct;
}

// Type trait: isEnum
pub fn isEnum(comptime T: type) bool {
    return @typeInfo(T) == .Enum;
}

// Type trait: isUnion
pub fn isUnion(comptime T: type) bool {
    return @typeInfo(T) == .Union;
}

// Type trait: isPointer
pub fn isPointer(comptime T: type) bool {
    return @typeInfo(T) == .Pointer;
}

// Type trait: isSlice
pub fn isSlice(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Pointer => |p| p.size == .Slice,
        else => false,
    };
}

// Type trait: isOptional
pub fn isOptional(comptime T: type) bool {
    return @typeInfo(T) == .Optional;
}

// Type trait: isErrorUnion
pub fn isErrorUnion(comptime T: type) bool {
    return @typeInfo(T) == .ErrorUnion;
}

// Trait: Comparable - has == and < operations
pub fn isComparable(comptime T: type) bool {
    return isNumeric(T) or isString(T) or T == bool;
}

// Trait: Ordered - has total ordering
pub fn isOrdered(comptime T: type) bool {
    return isNumeric(T);
}

// Trait: Hashable - can be used as hash map key
pub fn isHashable(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Int, .Float, .Bool, .Enum, .Pointer => true,
        .Struct => |s| blk: {
            inline for (s.fields) |field| {
                if (!isHashable(field.type)) break :blk false;
            }
            break :blk true;
        },
        else => false,
    };
}

// Trait: Serializable - can be converted to JSON
pub fn isSerializable(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Int, .Float, .Bool => true,
        .Pointer => |p| p.size == .Slice and p.child == u8,
        .Struct => |s| blk: {
            inline for (s.fields) |field| {
                if (!isSerializable(field.type)) break :blk false;
            }
            break :blk true;
        },
        .Optional => |o| isSerializable(o.child),
        .Array => |a| isSerializable(a.child),
        else => false,
    };
}

// Trait: Default - has a default value
pub fn hasDefault(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Int, .Float, .Bool => true,
        .Optional => true,
        .Struct => |s| blk: {
            inline for (s.fields) |field| {
                if (field.default_value == null) break :blk false;
            }
            break :blk true;
        },
        else => false,
    };
}

// Enforce trait at compile time
pub fn requireNumeric(comptime T: type) void {
    if (!isNumeric(T)) {
        @compileError("Type " ++ @typeName(T) ++ " is not numeric");
    }
}

// Constraint check
pub fn constrainedFn(comptime T: type, val: T) T {
    comptime requireNumeric(T);
    return val * 2;
}

pub fn typeTraitsExample() void {
    std.debug.print("isInteger(i32): {}\n", .{isInteger(i32)});
    std.debug.print("isInteger(f32): {}\n", .{isInteger(f32)});
    std.debug.print("isFloat(f64): {}\n", .{isFloat(f64)});
    std.debug.print("isNumeric(u8): {}\n", .{isNumeric(u8)});
    std.debug.print("isString([]const u8): {}\n", .{isString([]const u8)});
    std.debug.print("isStruct(std.ArrayList(i32)): {}\n", .{isStruct(std.ArrayList(i32))});
    std.debug.print("isOptional(?i32): {}\n", .{isOptional(?i32)});
    std.debug.print("isSlice([]i32): {}\n", .{isSlice([]i32)});

    const result = constrainedFn(i32, 21);
    std.debug.print("constrainedFn(21) = {}\n", .{result});
}
```

---

## Step 927: Runtime Type Info

### Runtime Type Information

```zig
const std = @import("std");

// TypeId สำหรับ runtime type checking
pub const TypeId = struct {
    hash: u64,
    name: []const u8,

    pub fn of(comptime T: type) TypeId {
        return TypeId{
            .hash = comptime hashTypeName(@typeName(T)),
            .name = @typeName(T),
        };
    }

    pub fn eql(self: TypeId, other: TypeId) bool {
        return self.hash == other.hash;
    }

    fn hashTypeName(name: []const u8) u64 {
        var hash: u64 = 5381;
        for (name) |c| {
            hash = ((hash << 5) +% hash) +% c;
        }
        return hash;
    }
};

// Any type container
pub const Any = struct {
    type_id: TypeId,
    ptr: *anyopaque,
    vtable: *const VTable,

    pub const VTable = struct {
        clone: *const fn (*anyopaque, std.mem.Allocator) anyerror!*anyopaque,
        destroy: *const fn (*anyopaque, std.mem.Allocator) void,
        format: *const fn (*anyopaque, []u8) []u8,
    };

    pub fn wrap(comptime T: type, value: T, allocator: std.mem.Allocator) !Any {
        const ptr = try allocator.create(T);
        ptr.* = value;

        return Any{
            .type_id = TypeId.of(T),
            .ptr = ptr,
            .vtable = &comptime makeVTable(T),
        };
    }

    pub fn unwrap(self: Any, comptime T: type) ?*T {
        if (self.type_id.eql(TypeId.of(T))) {
            return @ptrCast(@alignCast(self.ptr));
        }
        return null;
    }

    pub fn destroy(self: Any, allocator: std.mem.Allocator) void {
        self.vtable.destroy(self.ptr, allocator);
    }

    fn makeVTable(comptime T: type) VTable {
        return VTable{
            .clone = struct {
                fn clone(ptr: *anyopaque, allocator: std.mem.Allocator) !*anyopaque {
                    const typed: *T = @ptrCast(@alignCast(ptr));
                    const new_ptr = try allocator.create(T);
                    new_ptr.* = typed.*;
                    return new_ptr;
                }
            }.clone,
            .destroy = struct {
                fn destroy(ptr: *anyopaque, allocator: std.mem.Allocator) void {
                    const typed: *T = @ptrCast(@alignCast(ptr));
                    allocator.destroy(typed);
                }
            }.destroy,
            .format = struct {
                fn format(ptr: *anyopaque, buf: []u8) []u8 {
                    const typed: *T = @ptrCast(@alignCast(ptr));
                    const s = std.fmt.bufPrint(buf, "{any}", .{typed.*}) catch buf[0..0];
                    return s;
                }
            }.format,
        };
    }
};

pub fn runtimeTypeExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // TypeId comparison
    const id1 = TypeId.of(i32);
    const id2 = TypeId.of(i32);
    const id3 = TypeId.of(f64);

    std.debug.print("i32 == i32: {}\n", .{id1.eql(id2)});
    std.debug.print("i32 == f64: {}\n", .{id1.eql(id3)});

    // Any type
    const any_int = try Any.wrap(i32, 42, allocator);
    defer any_int.destroy(allocator);

    const any_str = try Any.wrap([]const u8, "hello", allocator);
    defer any_str.destroy(allocator);

    // Type-safe unwrap
    if (any_int.unwrap(i32)) |val| {
        std.debug.print("Unwrapped int: {}\n", .{val.*});
    }

    if (any_int.unwrap(f64)) |_| {
        std.debug.print("This should not happen\n", .{});
    } else {
        std.debug.print("Correct: i32 != f64\n", .{});
    }

    var fmt_buf: [64]u8 = undefined;
    const formatted = any_int.vtable.format(any_int.ptr, &fmt_buf);
    std.debug.print("Formatted: {s}\n", .{formatted});
}
```

---

## Step 928: Dynamic Dispatch

### Virtual Dispatch ผ่าน Interface

```zig
const std = @import("std");

// Interface definition using vtable
pub fn Interface(comptime methods: type) type {
    return struct {
        ptr: *anyopaque,
        vtable: *const methods,

        const Self = @This();

        pub fn init(comptime T: type, obj: *T) !Self {
            // Verify T implements the interface
            const vtable = comptime buildVTable(T, methods);
            return Self{
                .ptr = obj,
                .vtable = &vtable,
            };
        }

        fn buildVTable(comptime T: type, comptime VTable: type) VTable {
            var vtable: VTable = undefined;
            const vt_info = @typeInfo(VTable);
            switch (vt_info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        if (@hasDecl(T, field.name)) {
                            @field(vtable, field.name) = @field(T, field.name);
                        } else {
                            @compileError("Type " ++ @typeName(T) ++ " missing method: " ++ field.name);
                        }
                    }
                },
                else => @compileError("VTable must be a struct"),
            }
            return vtable;
        }
    };
}

// Animal interface
const AnimalVTable = struct {
    speak: *const fn (*anyopaque) void,
    getName: *const fn (*anyopaque) []const u8,
    getLegs: *const fn (*anyopaque) u32,
};

const Animal = struct {
    ptr: *anyopaque,
    vtable: *const AnimalVTable,

    pub fn speak(self: Animal) void {
        self.vtable.speak(self.ptr);
    }

    pub fn getName(self: Animal) []const u8 {
        return self.vtable.getName(self.ptr);
    }

    pub fn getLegs(self: Animal) u32 {
        return self.vtable.getLegs(self.ptr);
    }

    pub fn from(comptime T: type, obj: *T) Animal {
        const vtable = comptime struct {
            const speak_fn: *const fn (*anyopaque) void = struct {
                fn f(ptr: *anyopaque) void {
                    const self: *T = @ptrCast(@alignCast(ptr));
                    self.speak();
                }
            }.f;
            const getName_fn: *const fn (*anyopaque) []const u8 = struct {
                fn f(ptr: *anyopaque) []const u8 {
                    const self: *T = @ptrCast(@alignCast(ptr));
                    return self.getName();
                }
            }.f;
            const getLegs_fn: *const fn (*anyopaque) u32 = struct {
                fn f(ptr: *anyopaque) u32 {
                    const self: *T = @ptrCast(@alignCast(ptr));
                    return self.getLegs();
                }
            }.f;

            const vtable = AnimalVTable{
                .speak = speak_fn,
                .getName = getName_fn,
                .getLegs = getLegs_fn,
            };
        };

        return Animal{
            .ptr = obj,
            .vtable = &vtable.vtable,
        };
    }
};

const Dog = struct {
    name: []const u8,

    pub fn speak(self: *Dog) void {
        std.debug.print("{s}: Woof!\n", .{self.name});
    }

    pub fn getName(self: *Dog) []const u8 {
        return self.name;
    }

    pub fn getLegs(self: *Dog) u32 {
        _ = self;
        return 4;
    }
};

const Cat = struct {
    name: []const u8,

    pub fn speak(self: *Cat) void {
        std.debug.print("{s}: Meow!\n", .{self.name});
    }

    pub fn getName(self: *Cat) []const u8 {
        return self.name;
    }

    pub fn getLegs(self: *Cat) u32 {
        _ = self;
        return 4;
    }
};

const Bird = struct {
    name: []const u8,

    pub fn speak(self: *Bird) void {
        std.debug.print("{s}: Tweet!\n", .{self.name});
    }

    pub fn getName(self: *Bird) []const u8 {
        return self.name;
    }

    pub fn getLegs(self: *Bird) u32 {
        _ = self;
        return 2;
    }
};

pub fn dynamicDispatchExample() void {
    var dog = Dog{ .name = "Rex" };
    var cat = Cat{ .name = "Whiskers" };
    var bird = Bird{ .name = "Tweety" };

    const animals = [_]Animal{
        Animal.from(Dog, &dog),
        Animal.from(Cat, &cat),
        Animal.from(Bird, &bird),
    };

    std.debug.print("=== Animal sounds ===\n", .{});
    for (animals) |animal| {
        animal.speak();
        std.debug.print("  {s} has {} legs\n", .{ animal.getName(), animal.getLegs() });
    }
}
```

---

## Step 929: Automatic Code Generation

### Code Generation ผ่าน Reflection

```zig
const std = @import("std");

// Auto-generate validator
pub fn Validator(comptime T: type) type {
    return struct {
        pub fn validate(value: T) !void {
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        const val = @field(value, field.name);
                        switch (@typeInfo(field.type)) {
                            .Pointer => |p| {
                                if (p.size == .Slice and p.child == u8) {
                                    if (val.len == 0) {
                                        std.debug.print("Field '{s}' is empty\n", .{field.name});
                                        return error.EmptyField;
                                    }
                                }
                            },
                            .Optional => {
                                // Optional fields pass validation
                            },
                            else => {},
                        }
                    }
                },
                else => {},
            }
        }
    };
}

// Auto-generate hash
pub fn AutoHash(comptime T: type) type {
    return struct {
        pub fn hash(value: T) u64 {
            var hasher = std.hash.Wyhash.init(0);
            hashValue(T, value, &hasher);
            return hasher.final();
        }

        fn hashValue(comptime VT: type, val: VT, hasher: anytype) void {
            const vinfo = @typeInfo(VT);
            switch (vinfo) {
                .Int => hasher.update(std.mem.asBytes(&val)),
                .Float => hasher.update(std.mem.asBytes(&val)),
                .Bool => hasher.update(if (val) &[_]u8{1} else &[_]u8{0}),
                .Pointer => |p| {
                    if (p.size == .Slice and p.child == u8) {
                        hasher.update(val);
                    }
                },
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        hashValue(field.type, @field(val, field.name), hasher);
                    }
                },
                .Enum => hasher.update(std.mem.asBytes(&@intFromEnum(val))),
                else => {},
            }
        }
    };
}

// Auto-generate copy constructor
pub fn AutoCopy(comptime T: type) type {
    return struct {
        pub fn copy(value: T, allocator: std.mem.Allocator) !T {
            var result: T = undefined;
            try copyFields(T, value, &result, allocator);
            return result;
        }

        fn copyFields(comptime CT: type, src: CT, dst: *CT, allocator: std.mem.Allocator) !void {
            const cinfo = @typeInfo(CT);
            switch (cinfo) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        const src_val = @field(src, field.name);
                        switch (@typeInfo(field.type)) {
                            .Pointer => |p| {
                                if (p.size == .Slice) {
                                    const new_slice = try allocator.dupe(p.child, src_val);
                                    @field(dst.*, field.name) = new_slice;
                                } else {
                                    @field(dst.*, field.name) = src_val;
                                }
                            },
                            else => @field(dst.*, field.name) = src_val,
                        }
                    }
                },
                else => dst.* = src,
            }
        }
    };
}

const Product = struct {
    id: u64,
    name: []const u8,
    description: []const u8,
    price: f64,
    stock: u32,
};

pub fn autoCodegenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const product = Product{
        .id = 1,
        .name = "Widget",
        .description = "A useful widget",
        .price = 9.99,
        .stock = 100,
    };

    // Auto validator
    const ProductValidator = Validator(Product);
    ProductValidator.validate(product) catch |e| {
        std.debug.print("Validation error: {}\n", .{e});
    };
    std.debug.print("Product is valid\n", .{});

    // Auto hash
    const ProductHash = AutoHash(Product);
    const hash = ProductHash.hash(product);
    std.debug.print("Product hash: {x}\n", .{hash});

    // Auto copy
    const ProductCopy = AutoCopy(Product);
    const copied = try ProductCopy.copy(product, allocator);
    defer {
        allocator.free(copied.name);
        allocator.free(copied.description);
    }
    std.debug.print("Copied product: {s}\n", .{copied.name});
}
```

---

## Step 930: Reflection Patterns ใน Practice

### Practical Reflection Patterns

```zig
const std = @import("std");

// Observer pattern ด้วย reflection
pub fn Observable(comptime T: type) type {
    return struct {
        value: T,
        listeners: std.ArrayList(*const fn (old: T, new: T) void),
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator, initial: T) Self {
            return Self{
                .value = initial,
                .listeners = std.ArrayList(*const fn (T, T) void).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.listeners.deinit();
        }

        pub fn get(self: Self) T {
            return self.value;
        }

        pub fn set(self: *Self, new_value: T) !void {
            const old = self.value;
            self.value = new_value;
            for (self.listeners.items) |listener| {
                listener(old, new_value);
            }
        }

        pub fn onChange(self: *Self, listener: *const fn (T, T) void) !void {
            try self.listeners.append(listener);
        }
    };
}

// Dependency injection ด้วย reflection
pub fn Container(comptime bindings: anytype) type {
    return struct {
        pub fn resolve(comptime T: type) T {
            const info = @typeInfo(@TypeOf(bindings));
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        if (field.type == T) {
                            return @field(bindings, field.name);
                        }
                    }
                },
                else => {},
            }
            @compileError("No binding found for type: " ++ @typeName(T));
        }
    };
}

// Aspect-oriented programming simulation
pub fn withLogging(comptime f: anytype, name: []const u8) @TypeOf(f) {
    _ = name;
    return f; // Simplified - real implementation would wrap the function
}

pub fn withTiming(comptime f: anytype) @TypeOf(f) {
    return f; // Simplified
}

// Event sourcing pattern
pub fn EventSourced(comptime State: type, comptime Event: type) type {
    return struct {
        state: State,
        events: std.ArrayList(Event),
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator, initial_state: State) Self {
            return Self{
                .state = initial_state,
                .events = std.ArrayList(Event).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.events.deinit();
        }

        pub fn apply(self: *Self, event: Event, reducer: fn (State, Event) State) !void {
            self.state = reducer(self.state, event);
            try self.events.append(event);
        }

        pub fn replay(events: []const Event, initial: State, reducer: fn (State, Event) State) State {
            var state = initial;
            for (events) |event| {
                state = reducer(state, event);
            }
            return state;
        }

        pub fn snapshot(self: Self) State {
            return self.state;
        }
    };
}

// Counter state machine example
const CounterState = struct { count: i32 };
const CounterEvent = union(enum) {
    increment: i32,
    decrement: i32,
    reset: void,
};

fn counterReducer(state: CounterState, event: CounterEvent) CounterState {
    return switch (event) {
        .increment => |n| CounterState{ .count = state.count + n },
        .decrement => |n| CounterState{ .count = state.count - n },
        .reset => CounterState{ .count = 0 },
    };
}

pub fn reflectionPracticeExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Observable value
    var obs = Observable(i32).init(allocator, 0);
    defer obs.deinit();

    try obs.onChange(struct {
        fn onChange(old: i32, new: i32) void {
            std.debug.print("Value changed: {} -> {}\n", .{ old, new });
        }
    }.onChange);

    try obs.set(10);
    try obs.set(20);
    try obs.set(30);

    // Event sourcing
    const CounterES = EventSourced(CounterState, CounterEvent);
    var counter = CounterES.init(allocator, CounterState{ .count = 0 });
    defer counter.deinit();

    try counter.apply(.{ .increment = 5 }, counterReducer);
    try counter.apply(.{ .increment = 3 }, counterReducer);
    try counter.apply(.{ .decrement = 2 }, counterReducer);

    std.debug.print("Counter state: {}\n", .{counter.snapshot().count});

    // Replay from events
    const replayed = CounterES.replay(
        counter.events.items,
        CounterState{ .count = 0 },
        counterReducer,
    );
    std.debug.print("Replayed state: {}\n", .{replayed.count});
}

pub fn main() !void {
    try structReflectionExample();
    try enumReflectionExample();
    try functionReflectionExample();
    typeTraitsExample();
    try runtimeTypeExample();
    dynamicDispatchExample();
    try autoCodegenExample();
    try reflectionPracticeExample();
}
```

---

## สรุป Part 93

ในส่วนนี้เราได้เรียนรู้:

1. **Reflection Concepts** - compile-time reflection ใน Zig
2. **@typeInfo Deep Dive** - TypeInfo union และการใช้งาน
3. **Struct Field Reflection** - introspection และ manipulation
4. **Enum Reflection** - การสำรวจ enum values และ names
5. **Function Reflection** - function introspection และ higher-order functions
6. **Type Traits** - การสร้าง type constraints
7. **Runtime Type Info** - TypeId และ Any type container
8. **Dynamic Dispatch** - vtable-based dispatch
9. **Automatic Code Generation** - validator, hash, copy
10. **Practical Patterns** - Observable, Event Sourcing, DI

### Key Points

- Reflection ใน Zig ทำงานที่ compile time ไม่ใช่ runtime
- `@typeInfo` คือ core API สำหรับ type introspection
- ใช้ `inline for` เพื่อ iterate fields ที่ compile time
- Type traits ช่วยสร้าง generic code ที่ safe
