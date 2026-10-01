# ส่วนที่ 34: std.sort - การเรียงลำดับข้อมูล

## ภาพรวมของบทเรียน

`std.sort` ใน Zig ให้ sorting algorithms หลากหลายประเภท ตั้งแต่ stable sort ที่รักษาลำดับของ equal elements, fast unstable sort สำหรับประสิทธิภาพสูง, binary search, และ pattern ต่างๆ สำหรับการ sort ในโลกจริง

---

## Step 331: std.sort.block() - Stable Sort

### Stable Sort พื้นฐาน

```zig
const std = @import("std");

pub fn stableSortBasics(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // block sort เป็น stable sort - รักษาลำดับของ equal elements
    var numbers = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6, 0 };
    
    try stdout.print("ก่อน sort: {any}\n", .{numbers});
    
    // std.sort.block เป็น stable sort
    std.sort.block(i32, &numbers, {}, std.sort.asc(i32));
    
    try stdout.print("หลัง sort (ascending): {any}\n", .{numbers});
    
    // Sort descending
    std.sort.block(i32, &numbers, {}, std.sort.desc(i32));
    
    try stdout.print("หลัง sort (descending): {any}\n", .{numbers});
    
    // Sort strings
    var fruits = [_][]const u8{
        "banana",
        "apple",
        "cherry",
        "date",
        "elderberry",
    };
    
    std.sort.block([]const u8, &fruits, {}, struct {
        fn lessThan(_: void, a: []const u8, b: []const u8) bool {
            return std.mem.lessThan(u8, a, b);
        }
    }.lessThan);
    
    try stdout.print("\nFruits sorted: ", .{});
    for (fruits, 0..) |fruit, i| {
        if (i > 0) try stdout.print(", ", .{});
        try stdout.print("{s}", .{fruit});
    }
    try stdout.print("\n", .{});
    _ = allocator;
}
```

### การใช้ std.sort.block กับ Slices

```zig
const std = @import("std");

pub fn sortSlice() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Sort slice of floats
    var prices = [_]f64{ 19.99, 5.50, 99.99, 3.25, 45.00, 12.75 };
    
    try stdout.print("ราคาก่อน sort: ", .{});
    for (prices, 0..) |p, i| {
        if (i > 0) try stdout.print(", ", .{});
        try stdout.print("{d:.2}", .{p});
    }
    try stdout.print("\n", .{});
    
    std.sort.block(f64, &prices, {}, std.sort.asc(f64));
    
    try stdout.print("ราคาหลัง sort: ", .{});
    for (prices, 0..) |p, i| {
        if (i > 0) try stdout.print(", ", .{});
        try stdout.print("{d:.2}", .{p});
    }
    try stdout.print("\n", .{});
    
    // Sort partial slice
    var data = [_]u32{ 10, 9, 8, 7, 6, 5, 4, 3, 2, 1 };
    
    // Sort เฉพาะ elements 2-7 (index 2 ถึง 6)
    std.sort.block(u32, data[2..7], {}, std.sort.asc(u32));
    
    try stdout.print("Sort partial [2..7]: {any}\n", .{data});
}
```

### Stable Sort กับ Equal Elements

```zig
const std = @import("std");

const Student = struct {
    name: []const u8,
    grade: u32,
    original_order: usize,
};

pub fn demonstrateStability(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var students = std.ArrayList(Student).init(allocator);
    defer students.deinit();
    
    try students.append(.{ .name = "Alice", .grade = 85, .original_order = 0 });
    try students.append(.{ .name = "Bob", .grade = 92, .original_order = 1 });
    try students.append(.{ .name = "Charlie", .grade = 85, .original_order = 2 });
    try students.append(.{ .name = "David", .grade = 78, .original_order = 3 });
    try students.append(.{ .name = "Eve", .grade = 92, .original_order = 4 });
    try students.append(.{ .name = "Frank", .grade = 85, .original_order = 5 });
    
    // Stable sort by grade
    std.sort.block(Student, students.items, {}, struct {
        fn lessThan(_: void, a: Student, b: Student) bool {
            return a.grade < b.grade;
        }
    }.lessThan);
    
    try stdout.print("หลัง stable sort by grade:\n", .{});
    for (students.items) |s| {
        try stdout.print("  {s}: grade={d}, original_pos={d}\n", .{
            s.name,
            s.grade,
            s.original_order,
        });
    }
    
    // หมายเหตุ: ใน stable sort, students ที่มี grade เท่ากัน
    // จะยังคงเรียงตาม original order ของพวกเขา
}
```

---

## Step 332: std.sort.pdq() - Fast Sort

### Pattern-Defeating Quicksort

```zig
const std = @import("std");

pub fn fastSortBasics() !void {
    const stdout = std.io.getStdOut().writer();
    
    // pdq sort = Pattern-Defeating Quicksort
    // เร็วกว่า block sort แต่ไม่ stable
    
    var numbers = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6, 0 };
    
    try stdout.print("ก่อน sort: {any}\n", .{numbers});
    
    std.sort.pdq(i32, &numbers, {}, std.sort.asc(i32));
    
    try stdout.print("หลัง pdq sort: {any}\n", .{numbers});
    
    // pdq sort กับ large arrays
    var large_array: [10000]u32 = undefined;
    var prng = std.Random.DefaultPrng.init(42);
    const rand = prng.random();
    
    for (&large_array) |*val| {
        val.* = rand.int(u32);
    }
    
    var timer = try std.time.Timer.start();
    std.sort.pdq(u32, &large_array, {}, std.sort.asc(u32));
    const elapsed = timer.read();
    
    try stdout.print("Sort 10,000 elements: {d} us\n", .{
        elapsed / std.time.ns_per_us,
    });
    
    // ตรวจสอบว่า sorted
    try stdout.print("Is sorted: {}\n", .{std.sort.isSorted(u32, &large_array, {}, std.sort.asc(u32))});
}
```

### เปรียบเทียบ pdq vs block

```zig
const std = @import("std");

pub fn compareSortAlgorithms(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const sizes = [_]usize{ 100, 1000, 10000, 100000 };
    
    try stdout.print("{s:>10} | {s:>15} | {s:>15}\n", .{
        "Size",
        "block (us)",
        "pdq (us)",
    });
    try stdout.print("-" .* 44 ++ "\n", .{});
    
    for (sizes) |size| {
        // สร้าง random data
        var data1 = try allocator.alloc(u32, size);
        defer allocator.free(data1);
        var data2 = try allocator.alloc(u32, size);
        defer allocator.free(data2);
        
        var prng = std.Random.DefaultPrng.init(12345);
        const rand = prng.random();
        
        for (data1, 0..) |*v, i| {
            v.* = rand.int(u32);
            data2[i] = v.*;
        }
        
        // วัด block sort
        var timer = try std.time.Timer.start();
        std.sort.block(u32, data1, {}, std.sort.asc(u32));
        const block_time = timer.read();
        
        // วัด pdq sort
        timer.reset();
        std.sort.pdq(u32, data2, {}, std.sort.asc(u32));
        const pdq_time = timer.read();
        
        try stdout.print("{d:>10} | {d:>15} | {d:>15}\n", .{
            size,
            block_time / std.time.ns_per_us,
            pdq_time / std.time.ns_per_us,
        });
    }
}
```

---

## Step 333: Custom Comparators

### Custom Comparison Functions

```zig
const std = @import("std");

pub fn customComparators() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Sort integers ตาม absolute value
    var numbers = [_]i32{ -5, 3, -8, 1, -9, 2, 7, -4, 6, 0 };
    
    std.sort.block(i32, &numbers, {}, struct {
        fn lessThan(_: void, a: i32, b: i32) bool {
            return @abs(a) < @abs(b);
        }
    }.lessThan);
    
    try stdout.print("Sort by absolute value: {any}\n", .{numbers});
    
    // Sort strings by length, then alphabetically
    var words = [_][]const u8{
        "banana",
        "fig",
        "apple",
        "kiwi",
        "grape",
        "pear",
        "plum",
    };
    
    std.sort.block([]const u8, &words, {}, struct {
        fn lessThan(_: void, a: []const u8, b: []const u8) bool {
            if (a.len != b.len) return a.len < b.len;
            return std.mem.lessThan(u8, a, b);
        }
    }.lessThan);
    
    try stdout.print("Sort by length then alpha: ", .{});
    for (words, 0..) |w, i| {
        if (i > 0) try stdout.print(", ", .{});
        try stdout.print("{s}", .{w});
    }
    try stdout.print("\n", .{});
    
    // Sort ด้วย reverse order
    var desc_nums = [_]i32{ 1, 5, 3, 7, 2, 9, 4, 8, 6 };
    
    std.sort.block(i32, &desc_nums, {}, struct {
        fn greaterThan(_: void, a: i32, b: i32) bool {
            return a > b;
        }
    }.greaterThan);
    
    try stdout.print("Descending sort: {any}\n", .{desc_nums});
}
```

### Context-based Comparator

```zig
const std = @import("std");

// Comparator ที่ใช้ context (เช่น sort key index)
const MultiKeySort = struct {
    // Context เก็บ sort key priorities
    keys: []const usize,
    
    fn lessThan(ctx: MultiKeySort, a: []const i32, b: []const i32) bool {
        for (ctx.keys) |key| {
            if (key >= a.len or key >= b.len) continue;
            if (a[key] != b[key]) return a[key] < b[key];
        }
        return false;
    }
};

pub fn contextBasedSort(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // Table: [age, salary, years_experience]
    var table = [_][3]i32{
        .{ 30, 50000, 5 },
        .{ 25, 60000, 3 },
        .{ 30, 45000, 7 },
        .{ 25, 55000, 4 },
        .{ 35, 70000, 10 },
    };
    
    // Sort by age first, then by salary
    const keys = [_]usize{ 0, 1 }; // age, then salary
    const ctx = MultiKeySort{ .keys = &keys };
    
    _ = ctx;
    _ = allocator;
    
    // Sort using wrapper
    std.sort.block([3]i32, &table, {}, struct {
        fn lessThan(_: void, a: [3]i32, b: [3]i32) bool {
            // Sort by age first
            if (a[0] != b[0]) return a[0] < b[0];
            // Then by salary
            return a[1] < b[1];
        }
    }.lessThan);
    
    try stdout.print("Sorted by age, then salary:\n", .{});
    try stdout.print("  Age | Salary | Experience\n", .{});
    try stdout.print("-" .* 30 ++ "\n", .{});
    for (table) |row| {
        try stdout.print("  {d:3} | {d:6} | {d:10}\n", .{ row[0], row[1], row[2] });
    }
}
```

### Generic Sort Function

```zig
const std = @import("std");

// Generic sort wrapper ที่ใช้งานได้กับ any comparable type
pub fn sortAscending(
    comptime T: type,
    slice: []T,
) void {
    std.sort.pdq(T, slice, {}, std.sort.asc(T));
}

pub fn sortDescending(
    comptime T: type,
    slice: []T,
) void {
    std.sort.pdq(T, slice, {}, std.sort.desc(T));
}

pub fn sortByKey(
    comptime T: type,
    comptime K: type,
    slice: []T,
    key_fn: *const fn (T) K,
) void {
    const Ctx = struct {
        key_fn: *const fn (T) K,
        
        fn lessThan(self: @This(), a: T, b: T) bool {
            return self.key_fn(a) < self.key_fn(b);
        }
    };
    
    const ctx = Ctx{ .key_fn = key_fn };
    std.sort.pdq(T, slice, ctx, Ctx.lessThan);
}

pub fn demonstrateGenericSort() !void {
    const stdout = std.io.getStdOut().writer();
    
    var ints = [_]i32{ 5, 2, 8, 1, 9 };
    sortAscending(i32, &ints);
    try stdout.print("Sorted ints: {any}\n", .{ints});
    
    sortDescending(i32, &ints);
    try stdout.print("Reversed ints: {any}\n", .{ints});
    
    var floats = [_]f64{ 3.14, 1.41, 2.71, 0.57, 1.73 };
    sortAscending(f64, &floats);
    try stdout.print("Sorted floats: {any}\n", .{floats});
}
```

---

## Step 334: Sorting Structs

### Sort Structs ด้วยหลาย Fields

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,
    salary: f64,
    
    pub fn format(
        self: Person,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{s}(age={d}, salary={d:.0})", .{
            self.name,
            self.age,
            self.salary,
        });
    }
};

pub fn sortStructs(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var people = std.ArrayList(Person).init(allocator);
    defer people.deinit();
    
    try people.append(.{ .name = "Alice", .age = 30, .salary = 75000.0 });
    try people.append(.{ .name = "Bob", .age = 25, .salary = 60000.0 });
    try people.append(.{ .name = "Charlie", .age = 35, .salary = 90000.0 });
    try people.append(.{ .name = "Diana", .age = 28, .salary = 80000.0 });
    try people.append(.{ .name = "Eve", .age = 30, .salary = 70000.0 });
    
    // Sort by age
    std.sort.block(Person, people.items, {}, struct {
        fn lessThan(_: void, a: Person, b: Person) bool {
            return a.age < b.age;
        }
    }.lessThan);
    
    try stdout.print("Sort by age:\n", .{});
    for (people.items) |p| {
        try stdout.print("  {}\n", .{p});
    }
    
    // Sort by salary (descending)
    std.sort.block(Person, people.items, {}, struct {
        fn lessThan(_: void, a: Person, b: Person) bool {
            return a.salary > b.salary; // descending
        }
    }.lessThan);
    
    try stdout.print("\nSort by salary (desc):\n", .{});
    for (people.items) |p| {
        try stdout.print("  {}\n", .{p});
    }
    
    // Sort by age, then by name
    std.sort.block(Person, people.items, {}, struct {
        fn lessThan(_: void, a: Person, b: Person) bool {
            if (a.age != b.age) return a.age < b.age;
            return std.mem.lessThan(u8, a.name, b.name);
        }
    }.lessThan);
    
    try stdout.print("\nSort by age then name:\n", .{});
    for (people.items) |p| {
        try stdout.print("  {}\n", .{p});
    }
}
```

### Sort กับ Pointer/Index Arrays

```zig
const std = @import("std");

pub fn indirectSort(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ข้อมูลต้นฉบับ
    const data = [_]i32{ 50, 20, 80, 10, 90, 30, 70, 40, 60 };
    
    // สร้าง index array
    var indices = try allocator.alloc(usize, data.len);
    defer allocator.free(indices);
    
    for (indices, 0..) |*idx, i| {
        idx.* = i;
    }
    
    // Sort indices ตาม data values
    std.sort.pdq(usize, indices, data, struct {
        fn lessThan(ctx: [9]i32, a: usize, b: usize) bool {
            return ctx[a] < ctx[b];
        }
    }.lessThan);
    
    try stdout.print("Original data: {any}\n", .{data});
    try stdout.print("Sorted indices: {any}\n", .{indices});
    
    try stdout.print("Sorted values: ", .{});
    for (indices, 0..) |idx, i| {
        if (i > 0) try stdout.print(", ", .{});
        try stdout.print("{d}", .{data[idx]});
    }
    try stdout.print("\n", .{});
}
```

---

## Step 335: std.sort.isSorted()

### การตรวจสอบว่า Array ถูก Sort แล้ว

```zig
const std = @import("std");

pub fn checkIfSorted() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ตรวจสอบ ascending
    const sorted_asc = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    const not_sorted = [_]i32{ 1, 3, 2, 4, 5, 6, 7, 8, 9, 10 };
    const sorted_desc = [_]i32{ 10, 9, 8, 7, 6, 5, 4, 3, 2, 1 };
    
    try stdout.print("isSorted tests:\n", .{});
    
    try stdout.print("  {any}: asc={}, desc={}\n", .{
        sorted_asc,
        std.sort.isSorted(i32, &sorted_asc, {}, std.sort.asc(i32)),
        std.sort.isSorted(i32, &sorted_asc, {}, std.sort.desc(i32)),
    });
    
    try stdout.print("  {any}: asc={}, desc={}\n", .{
        not_sorted,
        std.sort.isSorted(i32, &not_sorted, {}, std.sort.asc(i32)),
        std.sort.isSorted(i32, &not_sorted, {}, std.sort.desc(i32)),
    });
    
    try stdout.print("  {any}: asc={}, desc={}\n", .{
        sorted_desc,
        std.sort.isSorted(i32, &sorted_desc, {}, std.sort.asc(i32)),
        std.sort.isSorted(i32, &sorted_desc, {}, std.sort.desc(i32)),
    });
    
    // ตรวจสอบกับ custom comparator
    const by_abs = [_]i32{ 0, -1, 2, -3, 4, -5, 6 };
    const is_sorted_by_abs = std.sort.isSorted(i32, &by_abs, {}, struct {
        fn lessThan(_: void, a: i32, b: i32) bool {
            return @abs(a) < @abs(b);
        }
    }.lessThan);
    
    try stdout.print("\nSorted by |value|: {any} -> {}\n", .{ by_abs, is_sorted_by_abs });
}
```

### Assert Sorted

```zig
const std = @import("std");

pub fn assertSorted(
    comptime T: type,
    slice: []const T,
    ctx: anytype,
    comptime lessThan: fn (@TypeOf(ctx), T, T) bool,
) void {
    if (std.debug.runtime_safety) {
        if (!std.sort.isSorted(T, slice, ctx, lessThan)) {
            @panic("Slice is not sorted!");
        }
    }
}

pub fn binarySearchRequiresSorted() !void {
    const stdout = std.io.getStdOut().writer();
    
    var data = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 };
    
    // ตรวจสอบก่อน binary search
    std.debug.assert(std.sort.isSorted(i32, &data, {}, std.sort.asc(i32)));
    
    try stdout.print("Data is sorted, proceeding with binary search\n", .{});
    try stdout.print("Data: {any}\n", .{data});
}
```

---

## Step 336: Binary Search - std.sort.binarySearch

### Binary Search พื้นฐาน

```zig
const std = @import("std");

pub fn binarySearchBasics() !void {
    const stdout = std.io.getStdOut().writer();
    
    var sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21 };
    
    try stdout.print("Sorted array: {any}\n\n", .{sorted});
    
    // Search for elements
    const targets = [_]i32{ 1, 7, 13, 21, 2, 8, 20 };
    
    for (targets) |target| {
        // std.sort.binarySearch ส่งคืน index ถ้าพบ, null ถ้าไม่พบ
        const result = std.sort.binarySearch(
            i32,
            target,
            &sorted,
            {},
            struct {
                fn order(_: void, key: i32, item: i32) std.math.Order {
                    return std.math.order(key, item);
                }
            }.order,
        );
        
        if (result) |idx| {
            try stdout.print("พบ {d:3} ที่ index {d}\n", .{ target, idx });
        } else {
            try stdout.print("ไม่พบ {d:3}\n", .{target});
        }
    }
}
```

### Lower Bound และ Upper Bound

```zig
const std = @import("std");

// Lower bound: index แรกที่ >= target
pub fn lowerBound(
    comptime T: type,
    sorted: []const T,
    target: T,
) usize {
    var lo: usize = 0;
    var hi: usize = sorted.len;
    
    while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        if (sorted[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    
    return lo;
}

// Upper bound: index แรกที่ > target
pub fn upperBound(
    comptime T: type,
    sorted: []const T,
    target: T,
) usize {
    var lo: usize = 0;
    var hi: usize = sorted.len;
    
    while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        if (sorted[mid] <= target) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    
    return lo;
}

pub fn demonstrateBounds() !void {
    const stdout = std.io.getStdOut().writer();
    
    const data = [_]i32{ 1, 3, 3, 3, 5, 7, 9, 9, 11 };
    
    try stdout.print("Data: {any}\n\n", .{data});
    
    const targets = [_]i32{ 0, 1, 3, 5, 9, 11, 12 };
    
    try stdout.print("{s:>6} | {s:>11} | {s:>11}\n", .{
        "Target",
        "lower_bound",
        "upper_bound",
    });
    try stdout.print("-" .* 33 ++ "\n", .{});
    
    for (targets) |t| {
        const lb = lowerBound(i32, &data, t);
        const ub = upperBound(i32, &data, t);
        try stdout.print("{d:>6} | {d:>11} | {d:>11}\n", .{ t, lb, ub });
    }
    
    // ตัวอย่าง: หาจำนวน occurrences ของ 3
    const target: i32 = 3;
    const lb = lowerBound(i32, &data, target);
    const ub = upperBound(i32, &data, target);
    try stdout.print("\nจำนวน {d} ใน array = {d}\n", .{ target, ub - lb });
}
```

### Binary Search กับ Structs

```zig
const std = @import("std");

const Record = struct {
    id: u32,
    value: []const u8,
};

pub fn binarySearchStructs() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Sorted array of records
    var records = [_]Record{
        .{ .id = 1, .value = "apple" },
        .{ .id = 5, .value = "banana" },
        .{ .id = 10, .value = "cherry" },
        .{ .id = 15, .value = "date" },
        .{ .id = 20, .value = "elderberry" },
        .{ .id = 25, .value = "fig" },
    };
    
    // Search by ID
    const search_id: u32 = 10;
    const result = std.sort.binarySearch(
        Record,
        search_id,
        &records,
        {},
        struct {
            fn order(_: void, key: u32, item: Record) std.math.Order {
                return std.math.order(key, item.id);
            }
        }.order,
    );
    
    if (result) |idx| {
        try stdout.print("พบ ID {d}: value = {s}\n", .{
            search_id,
            records[idx].value,
        });
    } else {
        try stdout.print("ไม่พบ ID {d}\n", .{search_id});
    }
    
    // Search ที่ไม่มีใน array
    const not_found_id: u32 = 12;
    if (std.sort.binarySearch(
        Record,
        not_found_id,
        &records,
        {},
        struct {
            fn order(_: void, key: u32, item: Record) std.math.Order {
                return std.math.order(key, item.id);
            }
        }.order,
    )) |_| {
        try stdout.print("พบ ID {d} (unexpected)\n", .{not_found_id});
    } else {
        try stdout.print("ไม่พบ ID {d} (expected)\n", .{not_found_id});
    }
}
```

---

## Step 337: Partial Sorting

### nth_element (การเลือก K ที่เล็กที่สุด/ใหญ่ที่สุด)

```zig
const std = @import("std");

// หา k-th smallest element (1-indexed)
// ใช้ quickselect algorithm
pub fn nthSmallest(
    comptime T: type,
    data: []T,
    k: usize,
) T {
    std.debug.assert(k > 0 and k <= data.len);
    
    // Partial sort โดยใช้ pdq sort แล้วเอา element ที่ k
    // (วิธีง่าย - สำหรับ production ควรใช้ quickselect)
    var copy = data;
    std.sort.pdq(T, copy, {}, std.sort.asc(T));
    return copy[k - 1];
}

// หา top-K elements
pub fn topK(
    comptime T: type,
    data: []T,
    k: usize,
    allocator: std.mem.Allocator,
) ![]T {
    const actual_k = @min(k, data.len);
    
    // Copy และ sort
    var sorted = try allocator.dupe(T, data);
    errdefer allocator.free(sorted);
    
    // Sort descending เพื่อให้ top-K อยู่ข้างหน้า
    std.sort.pdq(T, sorted, {}, std.sort.desc(T));
    
    // Return first k elements
    const result = try allocator.dupe(T, sorted[0..actual_k]);
    allocator.free(sorted);
    
    return result;
}

pub fn demonstratePartialSort(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var data = [_]i32{ 42, 17, 89, 5, 73, 31, 96, 12, 58, 67 };
    
    try stdout.print("Data: {any}\n", .{data});
    
    // หา 3rd smallest
    var copy = data;
    try stdout.print("3rd smallest = {d}\n", .{nthSmallest(i32, &copy, 3)});
    
    // หา top-5
    const top5 = try topK(i32, &data, 5, allocator);
    defer allocator.free(top5);
    
    try stdout.print("Top 5: {any}\n", .{top5});
    
    // Median (middle element)
    var median_copy = data;
    const median_idx = data.len / 2;
    std.sort.pdq(i32, &median_copy, {}, std.sort.asc(i32));
    try stdout.print("Median = {d}\n", .{median_copy[median_idx]});
}
```

### Stable Partition

```zig
const std = @import("std");

// แบ่ง array เป็น 2 ส่วน: ส่วนที่ผ่าน predicate และส่วนที่ไม่ผ่าน
pub fn stablePartition(
    comptime T: type,
    data: []T,
    predicate: *const fn (T) bool,
    allocator: std.mem.Allocator,
) !usize {
    var true_items = std.ArrayList(T).init(allocator);
    defer true_items.deinit();
    var false_items = std.ArrayList(T).init(allocator);
    defer false_items.deinit();
    
    for (data) |item| {
        if (predicate(item)) {
            try true_items.append(item);
        } else {
            try false_items.append(item);
        }
    }
    
    // Put back
    var i: usize = 0;
    for (true_items.items) |item| {
        data[i] = item;
        i += 1;
    }
    for (false_items.items) |item| {
        data[i] = item;
        i += 1;
    }
    
    return true_items.items.len;
}

fn isEven(x: i32) bool {
    return @mod(x, 2) == 0;
}

fn isPositive(x: i32) bool {
    return x > 0;
}

pub fn demonstratePartition(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var numbers = [_]i32{ 5, 2, 8, -3, 1, -7, 4, -9, 6, 3 };
    try stdout.print("Original: {any}\n", .{numbers});
    
    // Partition by even/odd
    var even_nums = numbers;
    const even_count = try stablePartition(i32, &even_nums, isEven, allocator);
    try stdout.print("Evens first (count={d}): {any}\n", .{ even_count, even_nums });
    
    // Partition by positive/negative
    var pos_nums = numbers;
    const pos_count = try stablePartition(i32, &pos_nums, isPositive, allocator);
    try stdout.print("Positives first (count={d}): {any}\n", .{ pos_count, pos_nums });
}
```

---

## Step 338: Sort Stability

### ความสำคัญของ Stable Sort

```zig
const std = @import("std");

const Task = struct {
    priority: u32,  // 1=high, 2=medium, 3=low
    name: []const u8,
    arrival_order: usize,
    
    pub fn format(
        self: Task,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("P{d}:{s}(#{d})", .{
            self.priority,
            self.name,
            self.arrival_order,
        });
    }
};

pub fn demonstrateStabilityImportance(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var tasks = std.ArrayList(Task).init(allocator);
    defer tasks.deinit();
    
    // Tasks มาถึงในลำดับนี้
    try tasks.append(.{ .priority = 2, .name = "Task_A", .arrival_order = 0 });
    try tasks.append(.{ .priority = 1, .name = "Task_B", .arrival_order = 1 });
    try tasks.append(.{ .priority = 3, .name = "Task_C", .arrival_order = 2 });
    try tasks.append(.{ .priority = 2, .name = "Task_D", .arrival_order = 3 });
    try tasks.append(.{ .priority = 1, .name = "Task_E", .arrival_order = 4 });
    try tasks.append(.{ .priority = 2, .name = "Task_F", .arrival_order = 5 });
    
    // Stable sort by priority: tasks ที่มี priority เดียวกัน
    // จะยังคงเรียงตาม arrival order
    std.sort.block(Task, tasks.items, {}, struct {
        fn lessThan(_: void, a: Task, b: Task) bool {
            return a.priority < b.priority;
        }
    }.lessThan);
    
    try stdout.print("Stable sort by priority:\n", .{});
    for (tasks.items) |task| {
        try stdout.print("  {}\n", .{task});
    }
    
    try stdout.print("\nหมายเหตุ: tasks ที่มี priority เดียวกัน\n", .{});
    try stdout.print("ยังเรียงตาม arrival order (A->D->F สำหรับ P2)\n", .{});
    
    // Unstable sort (pdq) อาจให้ผลต่าง
    var tasks2 = try std.ArrayList(Task).initCapacity(allocator, tasks.items.len);
    defer tasks2.deinit();
    try tasks2.appendSlice(tasks.items);
    
    // Reset to original order
    for (tasks2.items, 0..) |*t, i| {
        t.arrival_order = i;
    }
    
    std.sort.pdq(Task, tasks2.items, {}, struct {
        fn lessThan(_: void, a: Task, b: Task) bool {
            return a.priority < b.priority;
        }
    }.lessThan);
    
    try stdout.print("\npdq sort by priority (อาจไม่ stable):\n", .{});
    for (tasks2.items) |task| {
        try stdout.print("  {}\n", .{task});
    }
}
```

---

## Step 339: Performance Comparison of Sort Algorithms

### Benchmark Sort Algorithms

```zig
const std = @import("std");

const SortResult = struct {
    name: []const u8,
    time_us: u64,
};

pub fn benchmarkSorts(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const SIZES = [_]usize{ 1000, 10000, 100000 };
    
    for (SIZES) |size| {
        try stdout.print("\n=== Size: {d} ===\n", .{size});
        
        // Generate random data
        var original = try allocator.alloc(u32, size);
        defer allocator.free(original);
        
        var prng = std.Random.DefaultPrng.init(42);
        const rand = prng.random();
        
        for (original) |*v| {
            v.* = rand.int(u32);
        }
        
        // Test pdq sort
        {
            var data = try allocator.dupe(u32, original);
            defer allocator.free(data);
            
            var timer = try std.time.Timer.start();
            std.sort.pdq(u32, data, {}, std.sort.asc(u32));
            const elapsed = timer.read();
            
            try stdout.print("  pdq:   {d:8} us\n", .{elapsed / std.time.ns_per_us});
        }
        
        // Test block sort
        {
            var data = try allocator.dupe(u32, original);
            defer allocator.free(data);
            
            var timer = try std.time.Timer.start();
            std.sort.block(u32, data, {}, std.sort.asc(u32));
            const elapsed = timer.read();
            
            try stdout.print("  block: {d:8} us\n", .{elapsed / std.time.ns_per_us});
        }
    }
}
```

### Best/Worst Case Analysis

```zig
const std = @import("std");

pub fn analyzeInputPatterns(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const SIZE: usize = 100000;
    
    // ประเภท input ต่างๆ
    const patterns = [_]struct {
        name: []const u8,
        generate: *const fn ([]u32) void,
    }{
        .{ .name = "Random", .generate = generateRandom },
        .{ .name = "Sorted", .generate = generateSorted },
        .{ .name = "Reverse sorted", .generate = generateReverseSorted },
        .{ .name = "All same", .generate = generateAllSame },
        .{ .name = "Nearly sorted", .generate = generateNearlySorted },
    };
    
    try stdout.print("{s:>16} | {s:>12} | {s:>12}\n", .{
        "Pattern",
        "pdq (us)",
        "block (us)",
    });
    try stdout.print("-" .* 44 ++ "\n", .{});
    
    for (patterns) |pattern| {
        var data = try allocator.alloc(u32, SIZE);
        defer allocator.free(data);
        var data2 = try allocator.alloc(u32, SIZE);
        defer allocator.free(data2);
        
        pattern.generate(data);
        @memcpy(data2, data);
        
        var timer = try std.time.Timer.start();
        std.sort.pdq(u32, data, {}, std.sort.asc(u32));
        const pdq_time = timer.read();
        
        timer.reset();
        std.sort.block(u32, data2, {}, std.sort.asc(u32));
        const block_time = timer.read();
        
        try stdout.print("{s:>16} | {d:>12} | {d:>12}\n", .{
            pattern.name,
            pdq_time / std.time.ns_per_us,
            block_time / std.time.ns_per_us,
        });
    }
}

fn generateRandom(data: []u32) void {
    var prng = std.Random.DefaultPrng.init(42);
    const rand = prng.random();
    for (data) |*v| v.* = rand.int(u32);
}

fn generateSorted(data: []u32) void {
    for (data, 0..) |*v, i| v.* = @intCast(i);
}

fn generateReverseSorted(data: []u32) void {
    const n = data.len;
    for (data, 0..) |*v, i| v.* = @intCast(n - i - 1);
}

fn generateAllSame(data: []u32) void {
    for (data) |*v| v.* = 42;
}

fn generateNearlySorted(data: []u32) void {
    var prng = std.Random.DefaultPrng.init(42);
    const rand = prng.random();
    for (data, 0..) |*v, i| v.* = @intCast(i);
    
    // สลับ 1% ของ elements
    const swaps = data.len / 100;
    var i: usize = 0;
    while (i < swaps) : (i += 1) {
        const a = rand.uintLessThan(usize, data.len);
        const b = rand.uintLessThan(usize, data.len);
        const tmp = data[a];
        data[a] = data[b];
        data[b] = tmp;
    }
}
```

---

## Step 340: Real-world Sorting Patterns

### Leaderboard System

```zig
const std = @import("std");

const Player = struct {
    id: u32,
    name: []const u8,
    score: u64,
    wins: u32,
    losses: u32,
    
    pub fn winRate(self: Player) f64 {
        const total = self.wins + self.losses;
        if (total == 0) return 0.0;
        return @as(f64, @floatFromInt(self.wins)) / @as(f64, @floatFromInt(total));
    }
};

pub fn demonstrateLeaderboard(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var players = std.ArrayList(Player).init(allocator);
    defer players.deinit();
    
    try players.append(.{ .id = 1, .name = "DragonSlayer", .score = 95000, .wins = 145, .losses = 30 });
    try players.append(.{ .id = 2, .name = "NightOwl", .score = 87000, .wins = 120, .losses = 45 });
    try players.append(.{ .id = 3, .name = "StormBringer", .score = 95000, .wins = 138, .losses = 25 });
    try players.append(.{ .id = 4, .name = "IronFist", .score = 72000, .wins = 98, .losses = 60 });
    try players.append(.{ .id = 5, .name = "SilverArrow", .score = 88500, .wins = 125, .losses = 40 });
    
    // Sort by score (desc), then by wins (desc) สำหรับ tie-breaking
    std.sort.block(Player, players.items, {}, struct {
        fn lessThan(_: void, a: Player, b: Player) bool {
            if (a.score != b.score) return a.score > b.score;
            return a.wins > b.wins;
        }
    }.lessThan);
    
    try stdout.print("=== Leaderboard ===\n", .{});
    try stdout.print("{s:>4} {s:<15} {s:>8} {s:>6} {s:>6} {s:>8}\n", .{
        "Rank",
        "Player",
        "Score",
        "Wins",
        "Loss",
        "Win%",
    });
    try stdout.print("-" .* 50 ++ "\n", .{});
    
    for (players.items, 0..) |player, rank| {
        try stdout.print("{d:>4} {s:<15} {d:>8} {d:>6} {d:>6} {d:>7.1}%\n", .{
            rank + 1,
            player.name,
            player.score,
            player.wins,
            player.losses,
            player.winRate() * 100.0,
        });
    }
}
```

### File Explorer (Sort Files)

```zig
const std = @import("std");

const FileEntry = struct {
    name: []const u8,
    size: u64,
    is_dir: bool,
    modified_time: i64,
};

pub fn sortFileEntries(
    allocator: std.mem.Allocator,
    entries: []FileEntry,
    sort_by: enum { name, size, type, modified },
    ascending: bool,
) !void {
    const stdout = std.io.getStdOut().writer();
    
    switch (sort_by) {
        .name => {
            if (ascending) {
                std.sort.block(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        // Directories first
                        if (a.is_dir != b.is_dir) return a.is_dir;
                        return std.mem.lessThan(u8, a.name, b.name);
                    }
                }.lt);
            } else {
                std.sort.block(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        if (a.is_dir != b.is_dir) return !a.is_dir;
                        return std.mem.lessThan(u8, b.name, a.name);
                    }
                }.lt);
            }
        },
        .size => {
            if (ascending) {
                std.sort.pdq(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        return a.size < b.size;
                    }
                }.lt);
            } else {
                std.sort.pdq(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        return a.size > b.size;
                    }
                }.lt);
            }
        },
        .type => {
            std.sort.block(FileEntry, entries, {}, struct {
                fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                    // Dirs before files, then alphabetical
                    if (a.is_dir != b.is_dir) return a.is_dir;
                    return std.mem.lessThan(u8, a.name, b.name);
                }
            }.lt);
        },
        .modified => {
            if (ascending) {
                std.sort.pdq(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        return a.modified_time < b.modified_time;
                    }
                }.lt);
            } else {
                std.sort.pdq(FileEntry, entries, {}, struct {
                    fn lt(_: void, a: FileEntry, b: FileEntry) bool {
                        return a.modified_time > b.modified_time;
                    }
                }.lt);
            }
        },
    }
    
    try stdout.print("\nFiles sorted by {s} ({s}):\n", .{
        @tagName(sort_by),
        if (ascending) "asc" else "desc",
    });
    
    for (entries) |entry| {
        const type_str: []const u8 = if (entry.is_dir) "[DIR]" else "     ";
        try stdout.print("  {s} {s:<20} {d:>10} bytes\n", .{
            type_str,
            entry.name,
            entry.size,
        });
    }
    
    _ = allocator;
}

pub fn demonstrateFileSorting(allocator: std.mem.Allocator) !void {
    var entries = [_]FileEntry{
        .{ .name = "documents", .size = 0, .is_dir = true, .modified_time = 1000 },
        .{ .name = "photo.jpg", .size = 2048000, .is_dir = false, .modified_time = 3000 },
        .{ .name = "readme.txt", .size = 1024, .is_dir = false, .modified_time = 2000 },
        .{ .name = "projects", .size = 0, .is_dir = true, .modified_time = 4000 },
        .{ .name = "video.mp4", .size = 1024000000, .is_dir = false, .modified_time = 1500 },
        .{ .name = "config.json", .size = 512, .is_dir = false, .modified_time = 5000 },
    };
    
    // Sort by name
    try sortFileEntries(allocator, &entries, .name, true);
    
    // Sort by size descending
    try sortFileEntries(allocator, &entries, .size, false);
}
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Sort Validator

```zig
const std = @import("std");

pub fn validateSort(
    comptime T: type,
    sorted: []const T,
    ctx: anytype,
    comptime lessThan: fn (@TypeOf(ctx), T, T) bool,
) struct { is_sorted: bool, first_violation: ?usize } {
    var i: usize = 1;
    while (i < sorted.len) : (i += 1) {
        if (lessThan(ctx, sorted[i], sorted[i - 1])) {
            return .{ .is_sorted = false, .first_violation = i };
        }
    }
    return .{ .is_sorted = true, .first_violation = null };
}
```

### แบบฝึกหัดที่ 2: Priority Queue using Sort

```zig
const std = @import("std");

pub fn PriorityQueue(comptime T: type) type {
    return struct {
        data: std.ArrayList(T),
        less_fn: *const fn (T, T) bool,
        
        const Self = @This();
        
        pub fn init(
            allocator: std.mem.Allocator,
            less_fn: *const fn (T, T) bool,
        ) Self {
            return Self{
                .data = std.ArrayList(T).init(allocator),
                .less_fn = less_fn,
            };
        }
        
        pub fn deinit(self: *Self) void {
            self.data.deinit();
        }
        
        pub fn push(self: *Self, item: T) !void {
            try self.data.append(item);
            // Re-sort (inefficient but clear)
            const less = self.less_fn;
            std.sort.block(T, self.data.items, less, struct {
                fn lt(ctx: *const fn (T, T) bool, a: T, b: T) bool {
                    return ctx(a, b);
                }
            }.lt);
        }
        
        pub fn pop(self: *Self) ?T {
            if (self.data.items.len == 0) return null;
            return self.data.orderedRemove(0);
        }
        
        pub fn peek(self: *const Self) ?T {
            if (self.data.items.len == 0) return null;
            return self.data.items[0];
        }
        
        pub fn len(self: *const Self) usize {
            return self.data.items.len;
        }
    };
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **std.sort.block()** - Stable sort ที่รักษาลำดับ equal elements
2. **std.sort.pdq()** - Fast unstable sort (Pattern-Defeating Quicksort)
3. **Custom Comparators** - การเขียน comparison functions เอง
4. **Sorting Structs** - การ sort complex data structures
5. **std.sort.isSorted()** - การตรวจสอบว่า sorted แล้ว
6. **Binary Search** - การค้นหาใน sorted array
7. **Lower/Upper Bound** - การหา insertion points
8. **Partial Sorting** - Top-K, partition operations
9. **Sort Stability** - ความสำคัญของ stable sort
10. **Performance** - เปรียบเทียบและ benchmark sorting algorithms

**เมื่อไหร่ควรใช้อะไร:**
- **std.sort.block** - เมื่อต้องการ stable sort (เช่น multi-key sort)
- **std.sort.pdq** - เมื่อต้องการความเร็วและไม่สนใจ stability
- **Binary Search** - ค้นหาใน sorted array O(log n) แทน O(n)
- **Custom Comparator** - เมื่อ sort logic ซับซ้อน

---

*ต่อไปในส่วนที่ 35: std.hash_map - Hash Map Data Structure*
