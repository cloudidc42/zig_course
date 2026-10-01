# Part 92: Metaprogramming ใน Zig

## Steps 911-920

---

## Step 911: Metaprogramming Overview

### Metaprogramming คืออะไร?

Metaprogramming คือการเขียนโค้ดที่สร้าง หรือ transform โค้ดอื่น ใน Zig เราทำผ่าน `comptime` ซึ่งให้อำนาจในการทำงานกับ type system ณ compile time

### Metaprogramming ใน Zig vs ภาษาอื่น

```
C++ Templates:     ใช้ template metaprogramming ที่ซับซ้อน
Rust:              ใช้ procedural macros
Haskell:           Template Haskell
Zig:               comptime - เรียบง่าย แต่ทรงพลัง
```

### Core Concepts ของ Metaprogramming ใน Zig

```zig
const std = @import("std");

// 1. comptime variables
comptime var counter: u32 = 0;

// 2. comptime functions
pub fn comptime_add(comptime a: i32, comptime b: i32) i32 {
    return a + b;
}

// 3. Type functions - functions ที่ return types
pub fn Pair(comptime T: type, comptime U: type) type {
    return struct {
        first: T,
        second: U,
    };
}

// 4. comptime type inspection
pub fn typeNameOf(comptime T: type) []const u8 {
    return @typeName(T);
}

// 5. Generating code at comptime
pub fn generateGetterSetter(comptime T: type, comptime field_name: []const u8) type {
    const FieldType = @TypeOf(@field(@as(T, undefined), field_name));
    return struct {
        pub fn get(self: T) FieldType {
            return @field(self, field_name);
        }
        pub fn set(self: *T, value: FieldType) void {
            @field(self, field_name) = value;
        }
    };
}

// ตัวอย่างการใช้งาน
const IntPair = Pair(i32, i32);
const StringIntPair = Pair([]const u8, i32);

pub fn main() !void {
    const result = comptime_add(10, 20);
    std.debug.print("Compile-time result: {}\n", .{result});

    const p = IntPair{ .first = 1, .second = 2 };
    std.debug.print("Pair: ({}, {})\n", .{ p.first, p.second });

    std.debug.print("Type name of i32: {s}\n", .{typeNameOf(i32)});
    std.debug.print("Type name of f64: {s}\n", .{typeNameOf(f64)});
}
```

### Comptime Evaluation

```zig
// Comptime evaluation ทำให้ expressions ถูก evaluate ตั้งแต่ compile time
pub fn fibonacci(comptime n: u32) u32 {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// ผลลัพธ์ถูกคำนวณ ณ compile time
const fib10 = comptime fibonacci(10);  // = 55
const fib20 = comptime fibonacci(20);  // = 6765
```

---

## Step 912: Code Generation ด้วย Comptime

### Generating Struct Methods

```zig
const std = @import("std");

// Auto-generate toString method สำหรับ struct
pub fn WithToString(comptime T: type) type {
    return struct {
        data: T,

        const Self = @This();

        pub fn toString(self: Self, allocator: std.mem.Allocator) ![]const u8 {
            var buf = std.ArrayList(u8).init(allocator);
            errdefer buf.deinit();

            try buf.appendSlice(@typeName(T));
            try buf.appendSlice("{ ");

            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) try buf.appendSlice(", ");
                        try buf.appendSlice(field.name);
                        try buf.appendSlice(": ");
                        const val = @field(self.data, field.name);
                        try std.fmt.format(buf.writer(), "{any}", .{val});
                    }
                },
                else => try buf.appendSlice("..."),
            }

            try buf.appendSlice(" }");
            return buf.toOwnedSlice();
        }
    };
}

// Auto-generate comparison methods
pub fn WithComparison(comptime T: type) type {
    return struct {
        value: T,

        const Self = @This();

        pub fn eql(self: Self, other: Self) bool {
            return std.meta.eql(self.value, other.value);
        }

        pub fn lessThan(self: Self, other: Self) bool {
            return switch (@typeInfo(T)) {
                .Int, .Float => self.value < other.value,
                else => false,
            };
        }

        pub fn greaterThan(self: Self, other: Self) bool {
            return other.lessThan(self);
        }
    };
}

// Mixin pattern - combining multiple generated behaviors
pub fn Mixin(comptime T: type, comptime mixins: []const type) type {
    // Create a struct that combines all mixin types
    _ = mixins;
    return struct {
        inner: T,

        pub fn init(val: T) @This() {
            return @This(){ .inner = val };
        }
    };
}

// Code generation for validation
pub fn WithValidation(comptime T: type, comptime validators: anytype) type {
    return struct {
        data: T,

        const Self = @This();
        const Validators = @TypeOf(validators);

        pub fn validate(self: Self) !void {
            const info = @typeInfo(Validators);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        const validator = @field(validators, field.name);
                        try validator(self.data);
                    }
                },
                else => {},
            }
        }

        pub fn init(data: T) !Self {
            const instance = Self{ .data = data };
            try instance.validate();
            return instance;
        }
    };
}

// ตัวอย่างการ generate code
const Point = struct {
    x: f32,
    y: f32,
};

const PointWithString = WithToString(Point);

pub fn codeGenerationExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const p = PointWithString{ .data = Point{ .x = 3.14, .y = 2.72 } };
    const str = try p.toString(allocator);
    defer allocator.free(str);
    std.debug.print("Point as string: {s}\n", .{str});
}
```

### Generating Enums and Tagged Unions

```zig
// Generate enum from array of strings
pub fn EnumFromStrings(comptime strings: []const []const u8) type {
    comptime var fields: [strings.len]std.builtin.Type.EnumField = undefined;
    inline for (strings, 0..) |s, i| {
        fields[i] = .{
            .name = s,
            .value = i,
        };
    }

    return @Type(.{
        .Enum = .{
            .tag_type = u32,
            .fields = &fields,
            .decls = &.{},
            .is_exhaustive = true,
        },
    });
}

// ตัวอย่างการใช้งาน
const Colors = EnumFromStrings(&[_][]const u8{ "Red", "Green", "Blue", "Yellow" });

pub fn enumGenerationExample() void {
    const c = Colors.Red;
    std.debug.print("Color: {}\n", .{c});
    std.debug.print("Color value: {}\n", .{@intFromEnum(c)});
}
```

---

## Step 913: Type-Level Programming

### Type Algebra

```zig
const std = @import("std");

// Type-level boolean operations
pub fn TypeOr(comptime A: type, comptime B: type) type {
    return union(enum) {
        left: A,
        right: B,
    };
}

pub fn TypeAnd(comptime A: type, comptime B: type) type {
    return struct {
        left: A,
        right: B,
    };
}

// Type-level list operations
pub fn TypeList(comptime types: []const type) type {
    // Generate a tuple type from a list of types
    _ = types;
    return struct {};
}

// Higher-kinded types simulation
pub fn Functor(comptime F: fn (type) type) type {
    return struct {
        pub fn map(comptime A: type, comptime B: type, fa: F(A), f: fn (A) B) F(B) {
            _ = fa;
            _ = f;
            return undefined;
        }
    };
}

// Type constraints
pub fn Numeric(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => true,
        else => false,
    };
}

pub fn isContainer(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .Struct, .Union, .Enum => true,
        else => false,
    };
}

pub fn hasMethod(comptime T: type, comptime method_name: []const u8) bool {
    return @hasDecl(T, method_name);
}

// Type transformation
pub fn Optional(comptime T: type) type {
    return ?T;
}

pub fn Slice(comptime T: type) type {
    return []T;
}

pub fn Pointer(comptime T: type) type {
    return *T;
}

// Dependent types simulation
pub fn FixedArray(comptime T: type, comptime n: usize) type {
    return [n]T;
}

pub fn Matrix(comptime T: type, comptime rows: usize, comptime cols: usize) type {
    return [rows][cols]T;
}

// Type-level numbers
pub fn Vec(comptime T: type, comptime n: usize) type {
    return struct {
        data: [n]T,

        const Self = @This();

        pub fn add(self: Self, other: Self) Self {
            var result: Self = undefined;
            inline for (0..n) |i| {
                result.data[i] = self.data[i] + other.data[i];
            }
            return result;
        }

        pub fn dot(self: Self, other: Self) T {
            var sum: T = 0;
            inline for (0..n) |i| {
                sum += self.data[i] * other.data[i];
            }
            return sum;
        }

        pub fn scale(self: Self, factor: T) Self {
            var result: Self = undefined;
            inline for (0..n) |i| {
                result.data[i] = self.data[i] * factor;
            }
            return result;
        }

        pub fn magnitude(self: Self) f64 {
            var sum: f64 = 0;
            inline for (0..n) |i| {
                const val: f64 = @floatCast(self.data[i]);
                sum += val * val;
            }
            return std.math.sqrt(sum);
        }
    };
}

pub fn typeLevelExample() void {
    const Vec3f = Vec(f32, 3);
    const v1 = Vec3f{ .data = .{ 1.0, 2.0, 3.0 } };
    const v2 = Vec3f{ .data = .{ 4.0, 5.0, 6.0 } };

    const sum = v1.add(v2);
    const dot = v1.dot(v2);

    std.debug.print("Sum: {any}\n", .{sum.data});
    std.debug.print("Dot product: {}\n", .{dot});
    std.debug.print("Magnitude of v1: {d:.2}\n", .{v1.magnitude()});

    // Type-level matrix
    const Mat2x2 = Matrix(f64, 2, 2);
    var m: Mat2x2 = undefined;
    m[0][0] = 1.0;
    m[0][1] = 0.0;
    m[1][0] = 0.0;
    m[1][1] = 1.0;
    std.debug.print("Identity matrix: {any}\n", .{m});
}
```

---

## Step 914: Compile-Time Computation

### Complex Compile-Time Algorithms

```zig
const std = @import("std");

// Compile-time prime sieve
pub fn primeSieve(comptime limit: usize) [limit]bool {
    comptime var sieve = [_]bool{true} ** limit;
    sieve[0] = false;
    if (limit > 1) sieve[1] = false;

    comptime var i = 2;
    inline while (i * i < limit) : (i += 1) {
        if (sieve[i]) {
            comptime var j = i * i;
            inline while (j < limit) : (j += i) {
                sieve[j] = false;
            }
        }
    }
    return sieve;
}

// Compile-time matrix multiply
pub fn MatMul(comptime rows: usize, comptime inner: usize, comptime cols: usize) type {
    return struct {
        pub fn multiply(
            a: [rows][inner]f64,
            b: [inner][cols]f64,
        ) [rows][cols]f64 {
            var result = [_][cols]f64{[_]f64{0} ** cols} ** rows;
            inline for (0..rows) |i| {
                inline for (0..cols) |j| {
                    inline for (0..inner) |k| {
                        result[i][j] += a[i][k] * b[k][j];
                    }
                }
            }
            return result;
        }
    };
}

// Compile-time string processing
pub fn countOccurrences(comptime str: []const u8, comptime target: u8) comptime_int {
    comptime var count = 0;
    inline for (str) |c| {
        if (c == target) count += 1;
    }
    return count;
}

pub fn splitAt(comptime str: []const u8, comptime delimiter: u8) []const []const u8 {
    comptime var parts: [countOccurrences(str, delimiter) + 1][]const u8 = undefined;
    comptime var start = 0;
    comptime var part_idx = 0;

    inline for (str, 0..) |c, i| {
        if (c == delimiter) {
            parts[part_idx] = str[start..i];
            start = i + 1;
            part_idx += 1;
        }
    }
    parts[part_idx] = str[start..];

    return &parts;
}

// Compile-time hash map
pub fn ComptimeMap(comptime K: type, comptime V: type, comptime entries: anytype) type {
    return struct {
        pub fn get(key: K) ?V {
            inline for (entries) |entry| {
                if (std.meta.eql(entry[0], key)) {
                    return entry[1];
                }
            }
            return null;
        }

        pub fn has(key: K) bool {
            return get(key) != null;
        }
    };
}

// Compile-time sort
pub fn comptimeSort(comptime T: type, comptime arr: []const T) [arr.len]T {
    comptime var sorted = arr[0..arr.len].*;
    
    // Bubble sort at comptime
    comptime var i = 0;
    inline while (i < sorted.len) : (i += 1) {
        comptime var j = 0;
        inline while (j < sorted.len - i - 1) : (j += 1) {
            if (sorted[j] > sorted[j + 1]) {
                const tmp = sorted[j];
                sorted[j] = sorted[j + 1];
                sorted[j + 1] = tmp;
            }
        }
    }

    return sorted;
}

pub fn comptimeExample() void {
    // Prime sieve at compile time
    const primes = comptime primeSieve(50);
    std.debug.print("Primes up to 50: ", .{});
    inline for (primes, 0..) |is_prime, i| {
        if (is_prime) std.debug.print("{} ", .{i});
    }
    std.debug.print("\n", .{});

    // Compile-time string split
    const parts = comptime splitAt("hello,world,foo,bar", ',');
    std.debug.print("Parts: {}\n", .{parts.len});
    for (parts) |p| {
        std.debug.print("  '{s}'\n", .{p});
    }

    // Compile-time sort
    const unsorted = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };
    const sorted = comptime comptimeSort(i32, &unsorted);
    std.debug.print("Sorted: {any}\n", .{sorted});

    // Compile-time map
    const ColorMap = ComptimeMap([]const u8, u32, .{
        .{ "red", 0xFF0000 },
        .{ "green", 0x00FF00 },
        .{ "blue", 0x0000FF },
    });

    const red = ColorMap.get("red");
    std.debug.print("Red color: 0x{X}\n", .{red.?});
}
```

---

## Step 915: Generic Algorithms ขั้นสูง

### Advanced Generic Algorithms

```zig
const std = @import("std");

// Generic map function
pub fn map(
    comptime T: type,
    comptime U: type,
    items: []const T,
    f: fn (T) U,
    allocator: std.mem.Allocator,
) ![]U {
    var result = try allocator.alloc(U, items.len);
    for (items, 0..) |item, i| {
        result[i] = f(item);
    }
    return result;
}

// Generic filter function
pub fn filter(
    comptime T: type,
    items: []const T,
    predicate: fn (T) bool,
    allocator: std.mem.Allocator,
) ![]T {
    var result = std.ArrayList(T).init(allocator);
    errdefer result.deinit();
    for (items) |item| {
        if (predicate(item)) {
            try result.append(item);
        }
    }
    return result.toOwnedSlice();
}

// Generic reduce function
pub fn reduce(
    comptime T: type,
    comptime Acc: type,
    items: []const T,
    initial: Acc,
    f: fn (Acc, T) Acc,
) Acc {
    var acc = initial;
    for (items) |item| {
        acc = f(acc, item);
    }
    return acc;
}

// Generic zip function
pub fn zip(
    comptime A: type,
    comptime B: type,
    a: []const A,
    b: []const B,
    allocator: std.mem.Allocator,
) ![]struct { first: A, second: B } {
    const len = @min(a.len, b.len);
    var result = try allocator.alloc(struct { first: A, second: B }, len);
    for (0..len) |i| {
        result[i] = .{ .first = a[i], .second = b[i] };
    }
    return result;
}

// Generic pipeline
pub fn Pipeline(comptime T: type) type {
    return struct {
        value: T,
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(val: T, allocator: std.mem.Allocator) Self {
            return Self{ .value = val, .allocator = allocator };
        }

        pub fn pipe(self: Self, comptime U: type, f: fn (T, std.mem.Allocator) anyerror!U) !Pipeline(U) {
            const result = try f(self.value, self.allocator);
            return Pipeline(U).init(result, self.allocator);
        }

        pub fn unwrap(self: Self) T {
            return self.value;
        }
    };
}

// Generic sorting with comparator
pub fn sortBy(
    comptime T: type,
    items: []T,
    comptime less: fn (T, T) bool,
) void {
    // Insertion sort for simplicity
    var i: usize = 1;
    while (i < items.len) : (i += 1) {
        const key = items[i];
        var j: usize = i;
        while (j > 0 and less(key, items[j - 1])) : (j -= 1) {
            items[j] = items[j - 1];
        }
        items[j] = key;
    }
}

// Generic binary search
pub fn binarySearch(
    comptime T: type,
    items: []const T,
    target: T,
    comptime compare: fn (T, T) std.math.Order,
) ?usize {
    var lo: usize = 0;
    var hi = items.len;

    while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        switch (compare(items[mid], target)) {
            .lt => lo = mid + 1,
            .gt => hi = mid,
            .eq => return mid,
        }
    }
    return null;
}

// Generic merge
pub fn merge(
    comptime T: type,
    left: []const T,
    right: []const T,
    comptime less: fn (T, T) bool,
    allocator: std.mem.Allocator,
) ![]T {
    var result = try allocator.alloc(T, left.len + right.len);
    var i: usize = 0;
    var j: usize = 0;
    var k: usize = 0;

    while (i < left.len and j < right.len) {
        if (less(left[i], right[j])) {
            result[k] = left[i];
            i += 1;
        } else {
            result[k] = right[j];
            j += 1;
        }
        k += 1;
    }

    while (i < left.len) {
        result[k] = left[i];
        i += 1;
        k += 1;
    }

    while (j < right.len) {
        result[k] = right[j];
        j += 1;
        k += 1;
    }

    return result;
}

pub fn genericAlgorithmExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const numbers = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

    // Map
    const doubled = try map(i32, i32, &numbers, struct {
        fn f(x: i32) i32 { return x * 2; }
    }.f, allocator);
    defer allocator.free(doubled);
    std.debug.print("Doubled: {any}\n", .{doubled});

    // Filter
    const evens = try filter(i32, &numbers, struct {
        fn f(x: i32) bool { return @mod(x, 2) == 0; }
    }.f, allocator);
    defer allocator.free(evens);
    std.debug.print("Evens: {any}\n", .{evens});

    // Reduce
    const sum = reduce(i32, i32, &numbers, 0, struct {
        fn f(acc: i32, x: i32) i32 { return acc + x; }
    }.f);
    std.debug.print("Sum: {}\n", .{sum});

    // Binary search
    const sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13 };
    const idx = binarySearch(i32, &sorted, 7, struct {
        fn f(a: i32, b: i32) std.math.Order {
            return std.math.order(a, b);
        }
    }.f);
    std.debug.print("Index of 7: {?}\n", .{idx});
}
```

---

## Step 916: Type Transformation

### Transforming Types at Comptime

```zig
const std = @import("std");

// Make all fields optional
pub fn AllOptional(comptime T: type) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var new_fields: [s.fields.len]std.builtin.Type.StructField = undefined;
            inline for (s.fields, 0..) |field, i| {
                new_fields[i] = .{
                    .name = field.name,
                    .type = ?field.type,
                    .default_value = &@as(?field.type, null),
                    .is_comptime = false,
                    .alignment = @alignOf(?field.type),
                };
            }
            return @Type(.{
                .Struct = .{
                    .layout = .auto,
                    .fields = &new_fields,
                    .decls = &.{},
                    .is_tuple = false,
                },
            });
        },
        else => @compileError("AllOptional only works on structs"),
    }
}

// Make all fields pointers
pub fn AllPointers(comptime T: type) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var new_fields: [s.fields.len]std.builtin.Type.StructField = undefined;
            inline for (s.fields, 0..) |field, i| {
                new_fields[i] = .{
                    .name = field.name,
                    .type = *field.type,
                    .default_value = null,
                    .is_comptime = false,
                    .alignment = @alignOf(*field.type),
                };
            }
            return @Type(.{
                .Struct = .{
                    .layout = .auto,
                    .fields = &new_fields,
                    .decls = &.{},
                    .is_tuple = false,
                },
            });
        },
        else => @compileError("AllPointers only works on structs"),
    }
}

// Merge two struct types
pub fn MergeStructs(comptime A: type, comptime B: type) type {
    const a_info = @typeInfo(A);
    const b_info = @typeInfo(B);

    switch (a_info) {
        .Struct => |a_s| {
            switch (b_info) {
                .Struct => |b_s| {
                    comptime var fields: [a_s.fields.len + b_s.fields.len]std.builtin.Type.StructField = undefined;
                    inline for (a_s.fields, 0..) |f, i| {
                        fields[i] = f;
                    }
                    inline for (b_s.fields, 0..) |f, i| {
                        fields[a_s.fields.len + i] = f;
                    }
                    return @Type(.{
                        .Struct = .{
                            .layout = .auto,
                            .fields = &fields,
                            .decls = &.{},
                            .is_tuple = false,
                        },
                    });
                },
                else => @compileError("MergeStructs: B must be a struct"),
            }
        },
        else => @compileError("MergeStructs: A must be a struct"),
    }
}

// Pick specific fields
pub fn Pick(comptime T: type, comptime field_names: []const []const u8) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var picked_fields: [field_names.len]std.builtin.Type.StructField = undefined;
            comptime var picked_count: usize = 0;

            inline for (field_names) |name| {
                inline for (s.fields) |field| {
                    if (std.mem.eql(u8, field.name, name)) {
                        picked_fields[picked_count] = field;
                        picked_count += 1;
                    }
                }
            }

            return @Type(.{
                .Struct = .{
                    .layout = .auto,
                    .fields = picked_fields[0..picked_count],
                    .decls = &.{},
                    .is_tuple = false,
                },
            });
        },
        else => @compileError("Pick only works on structs"),
    }
}

// Omit specific fields
pub fn Omit(comptime T: type, comptime field_names: []const []const u8) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var kept_fields: [s.fields.len]std.builtin.Type.StructField = undefined;
            comptime var kept_count: usize = 0;

            inline for (s.fields) |field| {
                comptime var should_omit = false;
                inline for (field_names) |name| {
                    if (std.mem.eql(u8, field.name, name)) {
                        should_omit = true;
                    }
                }
                if (!should_omit) {
                    kept_fields[kept_count] = field;
                    kept_count += 1;
                }
            }

            return @Type(.{
                .Struct = .{
                    .layout = .auto,
                    .fields = kept_fields[0..kept_count],
                    .decls = &.{},
                    .is_tuple = false,
                },
            });
        },
        else => @compileError("Omit only works on structs"),
    }
}

const User = struct {
    id: u64,
    name: []const u8,
    email: []const u8,
    password_hash: []const u8,
    age: u32,
    created_at: i64,
};

pub fn typeTransformExample() void {
    const UserPublic = Omit(User, &[_][]const u8{ "password_hash" });
    const UserBasic = Pick(User, &[_][]const u8{ "id", "name", "email" });
    
    std.debug.print("UserPublic fields: {}\n", .{@typeInfo(UserPublic).Struct.fields.len});
    std.debug.print("UserBasic fields: {}\n", .{@typeInfo(UserBasic).Struct.fields.len});
    
    const OptionalUser = AllOptional(UserBasic);
    var ou: OptionalUser = .{};
    ou.id = 42;
    ou.name = "Alice";
    std.debug.print("Optional user id: {?}\n", .{ou.id});
}
```

---

## Step 917: Automatic Serialization

### Auto JSON Serialization

```zig
const std = @import("std");

// Auto serializer for any struct
pub fn JsonSerializer(comptime T: type) type {
    return struct {
        pub fn serialize(value: T, writer: anytype) !void {
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    try writer.writeByte('{');
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) try writer.writeByte(',');
                        try writer.writeByte('"');
                        try writer.writeAll(field.name);
                        try writer.writeAll("\":");
                        const val = @field(value, field.name);
                        try serializeValue(field.type, val, writer);
                    }
                    try writer.writeByte('}');
                },
                .Array => |a| {
                    try writer.writeByte('[');
                    for (value, 0..) |item, i| {
                        if (i > 0) try writer.writeByte(',');
                        try serializeValue(a.child, item, writer);
                    }
                    try writer.writeByte(']');
                },
                else => try serializeValue(T, value, writer),
            }
        }

        fn serializeValue(comptime VT: type, val: VT, writer: anytype) !void {
            const vinfo = @typeInfo(VT);
            switch (vinfo) {
                .Bool => try writer.writeAll(if (val) "true" else "false"),
                .Int => try std.fmt.format(writer, "{}", .{val}),
                .Float => try std.fmt.format(writer, "{d}", .{val}),
                .Pointer => |p| {
                    if (p.size == .Slice and p.child == u8) {
                        try writer.writeByte('"');
                        try writer.writeAll(val);
                        try writer.writeByte('"');
                    }
                },
                .Optional => {
                    if (val) |v| {
                        try serializeValue(p.child, v, writer);
                    } else {
                        try writer.writeAll("null");
                    }
                },
                .Struct => {
                    const SubSerializer = JsonSerializer(VT);
                    try SubSerializer.serialize(val, writer);
                },
                else => try writer.writeAll("null"),
            }
        }
    };
}

// Auto deserializer
pub fn JsonDeserializer(comptime T: type) type {
    return struct {
        pub fn deserialize(allocator: std.mem.Allocator, json_str: []const u8) !T {
            var scanner = std.json.Scanner.initCompleteInput(allocator, json_str);
            defer scanner.deinit();
            
            return std.json.parseFromTokenSource(T, allocator, &scanner, .{}) catch |e| {
                std.debug.print("Deserialization error: {}\n", .{e});
                return e;
            };
        }
    };
}

// Binary serialization
pub fn BinarySerializer(comptime T: type) type {
    return struct {
        pub fn serialize(value: T, writer: anytype) !void {
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        const val = @field(value, field.name);
                        try serializeField(field.type, val, writer);
                    }
                },
                else => try serializeField(T, value, writer),
            }
        }

        fn serializeField(comptime FT: type, val: FT, writer: anytype) !void {
            const finfo = @typeInfo(FT);
            switch (finfo) {
                .Int => |i| {
                    const bytes = i.bits / 8;
                    var buf: [8]u8 = undefined;
                    switch (bytes) {
                        1 => buf[0] = @intCast(val),
                        2 => std.mem.writeInt(u16, buf[0..2], @intCast(val), .little),
                        4 => std.mem.writeInt(u32, buf[0..4], @intCast(val), .little),
                        8 => std.mem.writeInt(u64, buf[0..8], @intCast(val), .little),
                        else => {},
                    }
                    try writer.writeAll(buf[0..bytes]);
                },
                .Float => {
                    const bytes = @sizeOf(FT);
                    const bits: u64 = @bitCast(val);
                    var buf: [8]u8 = undefined;
                    std.mem.writeInt(u64, &buf, bits, .little);
                    try writer.writeAll(buf[0..bytes]);
                },
                .Bool => {
                    try writer.writeByte(if (val) 1 else 0);
                },
                else => {},
            }
        }

        pub fn deserialize(reader: anytype) !T {
            var result: T = undefined;
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        @field(result, field.name) = try deserializeField(field.type, reader);
                    }
                },
                else => result = try deserializeField(T, reader),
            }
            return result;
        }

        fn deserializeField(comptime FT: type, reader: anytype) !FT {
            const finfo = @typeInfo(FT);
            switch (finfo) {
                .Int => |i| {
                    const bytes = i.bits / 8;
                    var buf: [8]u8 = undefined;
                    _ = try reader.readAll(buf[0..bytes]);
                    return switch (bytes) {
                        1 => @intCast(buf[0]),
                        2 => @intCast(std.mem.readInt(u16, buf[0..2], .little)),
                        4 => @intCast(std.mem.readInt(u32, buf[0..4], .little)),
                        8 => @intCast(std.mem.readInt(u64, buf[0..8], .little)),
                        else => 0,
                    };
                },
                .Bool => {
                    const byte = try reader.readByte();
                    return byte != 0;
                },
                else => return undefined,
            }
        }
    };
}

const Person = struct {
    id: u32,
    age: u8,
    is_active: bool,
};

pub fn serializationExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const person = Person{ .id = 42, .age = 30, .is_active = true };

    // Binary serialization
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    const PersonBinSerializer = BinarySerializer(Person);
    try PersonBinSerializer.serialize(person, buf.writer());

    std.debug.print("Serialized {} bytes\n", .{buf.items.len});

    // Deserialize
    var stream = std.io.fixedBufferStream(buf.items);
    const deserialized = try PersonBinSerializer.deserialize(stream.reader());
    
    std.debug.print("Deserialized: id={}, age={}, active={}\n", .{
        deserialized.id,
        deserialized.age,
        deserialized.is_active,
    });
}
```

---

## Step 918: ORM via Metaprogramming

### Object-Relational Mapping ด้วย Comptime

```zig
const std = @import("std");

// ORM annotations
pub const Column = struct {
    name: ?[]const u8 = null,
    primary_key: bool = false,
    nullable: bool = false,
    unique: bool = false,
    default: ?[]const u8 = null,
};

// Model definition
pub fn Model(comptime T: type) type {
    return struct {
        data: T,

        const Self = @This();
        const ModelType = T;

        pub fn tableName() []const u8 {
            // Default: use struct name in lowercase
            return @typeName(T);
        }

        pub fn fieldNames() []const []const u8 {
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    comptime var names: [s.fields.len][]const u8 = undefined;
                    inline for (s.fields, 0..) |field, i| {
                        names[i] = field.name;
                    }
                    return &names;
                },
                else => return &.{},
            }
        }

        pub fn generateInsert(self: Self, allocator: std.mem.Allocator) ![]const u8 {
            var sql = std.ArrayList(u8).init(allocator);
            errdefer sql.deinit();

            try sql.appendSlice("INSERT INTO ");
            try sql.appendSlice(tableName());
            try sql.appendSlice(" (");

            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) try sql.appendSlice(", ");
                        try sql.appendSlice(field.name);
                    }
                    try sql.appendSlice(") VALUES (");
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) try sql.appendSlice(", ");
                        const val = @field(self.data, field.name);
                        switch (@typeInfo(field.type)) {
                            .Int => {
                                var buf: [32]u8 = undefined;
                                const s2 = try std.fmt.bufPrint(&buf, "{}", .{val});
                                try sql.appendSlice(s2);
                            },
                            .Pointer => {
                                try sql.append('\'');
                                try sql.appendSlice(val);
                                try sql.append('\'');
                            },
                            .Bool => try sql.appendSlice(if (val) "1" else "0"),
                            else => try sql.appendSlice("NULL"),
                        }
                    }
                    try sql.append(')');
                },
                else => {},
            }

            return sql.toOwnedSlice();
        }

        pub fn generateSelect(comptime where: ?[]const u8, allocator: std.mem.Allocator) ![]const u8 {
            var sql = std.ArrayList(u8).init(allocator);
            errdefer sql.deinit();

            try sql.appendSlice("SELECT * FROM ");
            try sql.appendSlice(tableName());

            if (where) |w| {
                try sql.appendSlice(" WHERE ");
                try sql.appendSlice(w);
            }

            return sql.toOwnedSlice();
        }
    };
}

// Repository pattern
pub fn Repository(comptime T: type) type {
    return struct {
        allocator: std.mem.Allocator,

        const Self = @This();
        const ModelT = Model(T);

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{ .allocator = allocator };
        }

        pub fn findById(self: Self, id: u64) !?T {
            const sql = try std.fmt.allocPrint(self.allocator,
                "SELECT * FROM {s} WHERE id = {}",
                .{ @typeName(T), id });
            defer self.allocator.free(sql);
            std.debug.print("Query: {s}\n", .{sql});
            // In real implementation, execute SQL and return result
            return null;
        }

        pub fn findAll(self: Self) ![]T {
            const sql = try ModelT.generateSelect(null, self.allocator);
            defer self.allocator.free(sql);
            std.debug.print("Query: {s}\n", .{sql});
            return &.{};
        }

        pub fn save(self: Self, entity: T) !void {
            const model = ModelT{ .data = entity };
            const sql = try model.generateInsert(self.allocator);
            defer self.allocator.free(sql);
            std.debug.print("Insert: {s}\n", .{sql});
        }
    };
}

const User = struct {
    id: u64,
    name: []const u8,
    email: []const u8,
    age: u32,
};

pub fn ormExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const repo = Repository(User).init(allocator);

    const user = User{
        .id = 1,
        .name = "Alice",
        .email = "alice@example.com",
        .age = 30,
    };

    try repo.save(user);
    _ = try repo.findById(1);
    _ = try repo.findAll();
}
```

---

## Step 919: Test Generation

### Automatic Test Generation

```zig
const std = @import("std");

// Generate property tests
pub fn PropertyTest(comptime T: type) type {
    return struct {
        pub fn forAll(
            property: fn (T) bool,
            generator: fn (u64) T,
            n: usize,
        ) !void {
            var seed: u64 = 42;
            var i: usize = 0;
            while (i < n) : (i += 1) {
                const value = generator(seed);
                seed = seed *% 6364136223846793005 +% 1442695040888963407;
                if (!property(value)) {
                    std.debug.print("Property failed for value: {any}\n", .{value});
                    return error.PropertyViolated;
                }
            }
            std.debug.print("Property held for {} cases\n", .{n});
        }
    };
}

// Generate roundtrip tests
pub fn RoundtripTest(comptime T: type) type {
    return struct {
        pub fn test_roundtrip(
            serialize: fn (T, std.mem.Allocator) anyerror![]const u8,
            deserialize: fn ([]const u8, std.mem.Allocator) anyerror!T,
            value: T,
            allocator: std.mem.Allocator,
        ) !void {
            const serialized = try serialize(value, allocator);
            defer allocator.free(serialized);

            const deserialized = try deserialize(serialized, allocator);
            
            if (!std.meta.eql(value, deserialized)) {
                return error.RoundtripFailed;
            }
            std.debug.print("Roundtrip test passed\n", .{});
        }
    };
}

// Auto-generate tests for struct
pub fn generateTests(comptime T: type) void {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            std.debug.print("Generated tests for {s}:\n", .{@typeName(T)});
            inline for (s.fields) |field| {
                std.debug.print("  - test_{s}_field_{s}\n", .{ @typeName(T), field.name });
            }
        },
        else => {},
    }
}

// Table-driven test generator
pub fn TableTest(comptime Input: type, comptime Expected: type) type {
    return struct {
        pub const Case = struct {
            name: []const u8,
            input: Input,
            expected: Expected,
        };

        pub fn run(
            cases: []const Case,
            test_fn: fn (Input) Expected,
        ) !void {
            var passed: u32 = 0;
            var failed: u32 = 0;

            for (cases) |case| {
                const result = test_fn(case.input);
                if (std.meta.eql(result, case.expected)) {
                    std.debug.print("  PASS: {s}\n", .{case.name});
                    passed += 1;
                } else {
                    std.debug.print("  FAIL: {s}\n", .{case.name});
                    std.debug.print("    Expected: {any}\n", .{case.expected});
                    std.debug.print("    Got: {any}\n", .{result});
                    failed += 1;
                }
            }

            std.debug.print("Results: {}/{} passed\n", .{ passed, passed + failed });
            if (failed > 0) return error.TestFailed;
        }
    };
}

pub fn testGenerationExample() !void {
    // Table-driven test example
    const IntTest = TableTest(i32, i32);
    const cases = [_]IntTest.Case{
        .{ .name = "double zero", .input = 0, .expected = 0 },
        .{ .name = "double positive", .input = 5, .expected = 10 },
        .{ .name = "double negative", .input = -3, .expected = -6 },
    };

    try IntTest.run(&cases, struct {
        fn f(x: i32) i32 { return x * 2; }
    }.f);

    // Property test example
    const IntProp = PropertyTest(i32);
    try IntProp.forAll(
        struct {
            fn f(x: i32) bool {
                // Property: adding zero doesn't change value
                return x + 0 == x;
            }
        }.f,
        struct {
            fn f(seed: u64) i32 {
                return @intCast(seed % 1000);
            }
        }.f,
        100,
    );
}
```

---

## Step 920: Production Metaprogramming Patterns

### Real-World Metaprogramming

```zig
const std = @import("std");

// Event system via metaprogramming
pub fn EventSystem(comptime events: anytype) type {
    const EventsType = @TypeOf(events);
    const events_info = @typeInfo(EventsType);

    return struct {
        const Self = @This();

        // Generate handler maps for each event type
        allocator: std.mem.Allocator,

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{ .allocator = allocator };
        }

        pub fn emit(self: Self, comptime event_name: []const u8, data: anytype) !void {
            _ = self;
            // Look up event in events map
            switch (events_info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        if (std.mem.eql(u8, field.name, event_name)) {
                            std.debug.print("Event emitted: {s} with data: {any}\n", .{ event_name, data });
                            return;
                        }
                    }
                },
                else => {},
            }
            return error.UnknownEvent;
        }
    };
}

// Plugin system
pub fn PluginSystem(comptime PluginInterface: type) type {
    return struct {
        plugins: std.ArrayList(*PluginInterface),
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{
                .plugins = std.ArrayList(*PluginInterface).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.plugins.deinit();
        }

        pub fn register(self: *Self, plugin: *PluginInterface) !void {
            try self.plugins.append(plugin);
        }

        pub fn dispatch(self: Self, comptime method: []const u8, args: anytype) !void {
            for (self.plugins.items) |plugin| {
                if (@hasField(@TypeOf(plugin.*), method)) {
                    try @field(plugin.*, method)(args);
                }
            }
        }
    };
}

// Middleware pattern
pub fn Middleware(comptime Ctx: type) type {
    return struct {
        const HandlerFn = fn (*Ctx) anyerror!void;
        const MiddlewareFn = fn (*Ctx, HandlerFn) anyerror!void;

        chain: std.ArrayList(MiddlewareFn),
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{
                .chain = std.ArrayList(MiddlewareFn).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.chain.deinit();
        }

        pub fn use(self: *Self, m: MiddlewareFn) !void {
            try self.chain.append(m);
        }

        pub fn execute(self: Self, ctx: *Ctx, handler: HandlerFn) !void {
            if (self.chain.items.len == 0) {
                try handler(ctx);
                return;
            }
            try self.chain.items[0](ctx, handler);
        }
    };
}

// Command pattern with comptime dispatch
pub fn CommandDispatcher(comptime commands: anytype) type {
    return struct {
        pub fn dispatch(name: []const u8, args: []const []const u8) !void {
            const info = @typeInfo(@TypeOf(commands));
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        if (std.mem.eql(u8, field.name, name)) {
                            const cmd = @field(commands, field.name);
                            try cmd(args);
                            return;
                        }
                    }
                },
                else => {},
            }
            std.debug.print("Unknown command: {s}\n", .{name});
            return error.UnknownCommand;
        }
    };
}

pub fn productionPatternExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Command dispatcher
    const Cmds = CommandDispatcher(.{
        .hello = struct {
            fn run(args: []const []const u8) !void {
                if (args.len > 0) {
                    std.debug.print("Hello, {s}!\n", .{args[0]});
                } else {
                    std.debug.print("Hello, World!\n", .{});
                }
            }
        }.run,
        .greet = struct {
            fn run(args: []const []const u8) !void {
                std.debug.print("Greeting with {} args\n", .{args.len});
            }
        }.run,
    });

    const args = [_][]const u8{"Alice"};
    try Cmds.dispatch("hello", &args);
    _ = allocator;
}

pub fn main() !void {
    try codeGenerationExample();
    typeLevelExample();
    try genericAlgorithmExample();
    typeTransformExample();
    try serializationExample();
    try ormExample();
    try testGenerationExample();
    try productionPatternExample();
}
```

---

## สรุป Part 92

ในส่วนนี้เราได้เรียนรู้:

1. **Metaprogramming Overview** - แนวคิดและ core concepts
2. **Code Generation** - สร้าง methods และ types อัตโนมัติ
3. **Type-Level Programming** - type algebra และ higher-kinded types
4. **Compile-Time Computation** - algorithms ที่ทำงานตั้งแต่ compile time
5. **Generic Algorithms** - map, filter, reduce, sort ที่ทำงานกับทุก type
6. **Type Transformation** - AllOptional, Pick, Omit, Merge
7. **Automatic Serialization** - JSON และ binary serialization อัตโนมัติ
8. **ORM** - Object-relational mapping ด้วย comptime
9. **Test Generation** - สร้าง tests อัตโนมัติ
10. **Production Patterns** - Event system, Plugin system, Middleware

### Key Takeaways

- `comptime` ใน Zig ทรงพลังมากและเรียบง่ายกว่า C++ templates
- Type functions ช่วยสร้าง reusable abstractions
- Compile-time computation ทำให้โค้ด runtime เร็วขึ้น
- Metaprogramming ควรใช้เมื่อจำเป็นจริงๆ ไม่ใช่ทุกที่
