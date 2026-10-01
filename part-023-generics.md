# Part 23: Generics ใน Zig (Steps 221-230)

Generics คือความสามารถในการเขียนโค้ดที่ทำงานได้กับหลาย type โดยไม่ต้องเขียนซ้ำ ใน Zig นั้น Generics ถูก implement ผ่านกลไก `comptime` ซึ่งต่างจากภาษาอื่นอย่าง Go, Rust หรือ C++ ที่มี syntax พิเศษสำหรับ generics โดยเฉพาะ

---

## Step 221: Generics ใน Zig - ผ่าน comptime type: type

### แนวคิดพื้นฐาน

ใน Zig ไม่มี syntax พิเศษสำหรับ generics แบบ `<T>` เหมือน Rust หรือ Go แต่ใช้ `comptime` parameter ที่มี type เป็น `type` แทน

**ใน Go:**
```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}
```

**ใน Rust:**
```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

**ใน C++:**
```cpp
template<typename T>
T largest(std::vector<T>& list) {
    T result = list[0];
    for (auto& item : list) {
        if (item > result) result = item;
    }
    return result;
}
```

**ใน Zig:**
```zig
const std = @import("std");

// comptime T: type คือวิธีการทำ generics ใน Zig
fn largest(comptime T: type, list: []const T) T {
    var result = list[0];
    for (list[1..]) |item| {
        if (item > result) result = item;
    }
    return result;
}

pub fn main() void {
    const ints = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6 };
    const floats = [_]f64{ 3.14, 2.71, 1.41 };

    std.debug.print("Largest int: {}\n", .{largest(i32, &ints)});
    std.debug.print("Largest float: {}\n", .{largest(f64, &floats)});
}
```

### ทำความเข้าใจ comptime

`comptime` ใน Zig หมายถึง "ทราบค่าได้ตอน compile time" เมื่อเราส่ง `T: type` เป็น comptime parameter:

1. Compiler จะ evaluate ค่าของ `T` ตอน compile time
2. สร้าง specialization ของ function นั้นสำหรับ type นั้นๆ
3. ผลลัพธ์คือ zero-overhead abstraction - ไม่มี runtime overhead

```zig
const std = @import("std");

// ตัวอย่างแสดงว่า type เป็น first-class value ใน Zig
pub fn main() void {
    // type สามารถเก็บใน variable ได้
    const T = i32;
    const value: T = 42;
    std.debug.print("value: {}\n", .{value});

    // @TypeOf ให้ type ของ expression
    const x: f64 = 3.14;
    const XType = @TypeOf(x);
    std.debug.print("Type of x: {}\n", .{XType});

    // type info ผ่าน @typeInfo
    const info = @typeInfo(i32);
    std.debug.print("Is signed: {}\n", .{info.int.signedness == .signed});
}
```

### Monomorphization

Zig ทำ monomorphization เหมือน Rust และ C++ กล่าวคือ compiler สร้าง code แยกต่างหากสำหรับแต่ละ type:

```zig
const std = @import("std");

fn identity(comptime T: type, value: T) T {
    return value;
}

pub fn main() void {
    // Compiler สร้าง identity_i32 และ identity_f64 แยกกัน
    const a = identity(i32, 42);
    const b = identity(f64, 3.14);
    const c = identity(bool, true);

    std.debug.print("{} {} {}\n", .{ a, b, c });
}
```

---

## Step 222: Generic Functions

### รูปแบบพื้นฐานของ Generic Functions

```zig
const std = @import("std");

// Generic swap
fn swap(comptime T: type, a: *T, b: *T) void {
    const temp = a.*;
    a.* = b.*;
    b.* = temp;
}

// Generic min และ max
fn min(comptime T: type, a: T, b: T) T {
    return if (a < b) a else b;
}

fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// Generic clamp
fn clamp(comptime T: type, value: T, lo: T, hi: T) T {
    return min(T, max(T, value, lo), hi);
}

pub fn main() void {
    var x: i32 = 10;
    var y: i32 = 20;
    swap(i32, &x, &y);
    std.debug.print("After swap: x={}, y={}\n", .{ x, y });

    std.debug.print("min(3, 5) = {}\n", .{min(i32, 3, 5)});
    std.debug.print("max(3, 5) = {}\n", .{max(i32, 3, 5)});
    std.debug.print("clamp(15, 0, 10) = {}\n", .{clamp(i32, 15, 0, 10)});
}
```

### Generic Higher-Order Functions

```zig
const std = @import("std");

// Generic map function
fn map(
    comptime T: type,
    comptime U: type,
    items: []const T,
    f: fn (T) U,
    allocator: std.mem.Allocator,
) ![]U {
    const result = try allocator.alloc(U, items.len);
    for (items, 0..) |item, i| {
        result[i] = f(item);
    }
    return result;
}

// Generic filter function
fn filter(
    comptime T: type,
    items: []const T,
    predicate: fn (T) bool,
    allocator: std.mem.Allocator,
) ![]T {
    var result = std.ArrayList(T).init(allocator);
    defer result.deinit();
    for (items) |item| {
        if (predicate(item)) {
            try result.append(item);
        }
    }
    return result.toOwnedSlice();
}

// Generic reduce/fold
fn reduce(
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

fn double(x: i32) i32 {
    return x * 2;
}

fn isEven(x: i32) bool {
    return @rem(x, 2) == 0;
}

fn sum(acc: i64, x: i32) i64 {
    return acc + @as(i64, x);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const numbers = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

    const doubled = try map(i32, i32, &numbers, double, allocator);
    defer allocator.free(doubled);
    std.debug.print("Doubled: {any}\n", .{doubled});

    const evens = try filter(i32, &numbers, isEven, allocator);
    defer allocator.free(evens);
    std.debug.print("Evens: {any}\n", .{evens});

    const total = reduce(i32, i64, &numbers, 0, sum);
    std.debug.print("Sum: {}\n", .{total});
}
```

### Generic Functions ด้วย Multiple Type Parameters

```zig
const std = @import("std");

// Generic zip สำหรับสองชนิด
fn zip(
    comptime A: type,
    comptime B: type,
    a: []const A,
    b: []const B,
    allocator: std.mem.Allocator,
) ![]struct { first: A, second: B } {
    const len = @min(a.len, b.len);
    const Pair = struct { first: A, second: B };
    const result = try allocator.alloc(Pair, len);
    for (0..len) |i| {
        result[i] = .{ .first = a[i], .second = b[i] };
    }
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const names = [_][]const u8{ "Alice", "Bob", "Charlie" };
    const scores = [_]i32{ 95, 87, 92 };

    const zipped = try zip([]const u8, i32, &names, &scores, allocator);
    defer allocator.free(zipped);

    for (zipped) |pair| {
        std.debug.print("{s}: {}\n", .{ pair.first, pair.second });
    }
}
```

---

## Step 223: Generic Structs (fn returning type)

### Pattern: Function ที่ return Type

ใน Zig การสร้าง generic struct ทำผ่านการเขียน function ที่รับ `comptime` parameters และ return `type`:

```zig
const std = @import("std");

// Generic Pair struct
fn Pair(comptime First: type, comptime Second: type) type {
    return struct {
        first: First,
        second: Second,

        const Self = @This();

        pub fn init(first: First, second: Second) Self {
            return .{ .first = first, .second = second };
        }

        pub fn swap(self: Self) Pair(Second, First) {
            return .{ .first = self.second, .second = self.first };
        }
    };
}

// Generic Optional (เหมือน Rust's Option)
fn Maybe(comptime T: type) type {
    return union(enum) {
        some: T,
        none,

        const Self = @This();

        pub fn some_val(value: T) Self {
            return .{ .some = value };
        }

        pub fn none_val() Self {
            return .none;
        }

        pub fn unwrap(self: Self) T {
            return switch (self) {
                .some => |v| v,
                .none => @panic("unwrap on none"),
            };
        }

        pub fn unwrap_or(self: Self, default: T) T {
            return switch (self) {
                .some => |v| v,
                .none => default,
            };
        }

        pub fn is_some(self: Self) bool {
            return self == .some;
        }
    };
}

pub fn main() void {
    // ใช้ Pair
    const p = Pair(i32, []const u8).init(42, "hello");
    std.debug.print("Pair: ({}, {s})\n", .{ p.first, p.second });

    const swapped = p.swap();
    std.debug.print("Swapped: ({s}, {})\n", .{ swapped.first, swapped.second });

    // ใช้ Maybe
    const MaybeInt = Maybe(i32);
    const some_value = MaybeInt.some_val(42);
    const no_value = MaybeInt.none_val();

    std.debug.print("Has value: {}\n", .{some_value.is_some()});
    std.debug.print("Value: {}\n", .{some_value.unwrap()});
    std.debug.print("Default: {}\n", .{no_value.unwrap_or(0)});
}
```

### Generic Wrapper Types

```zig
const std = @import("std");

// Generic Box - ownership wrapper
fn Box(comptime T: type) type {
    return struct {
        value: T,

        const Self = @This();

        pub fn init(value: T) Self {
            return .{ .value = value };
        }

        pub fn get(self: *const Self) *const T {
            return &self.value;
        }

        pub fn get_mut(self: *Self) *T {
            return &self.value;
        }

        pub fn map(self: Self, comptime U: type, f: fn (T) U) Box(U) {
            return Box(U).init(f(self.value));
        }
    };
}

// Generic Lazy - ค่าที่ evaluate แบบ lazy
fn Lazy(comptime T: type) type {
    return struct {
        value: ?T = null,
        compute: *const fn () T,

        const Self = @This();

        pub fn init(compute: *const fn () T) Self {
            return .{ .compute = compute };
        }

        pub fn get(self: *Self) T {
            if (self.value == null) {
                self.value = self.compute();
            }
            return self.value.?;
        }
    };
}

fn expensiveComputation() i32 {
    std.debug.print("Computing...\n", .{});
    return 42;
}

pub fn main() void {
    // Box
    var boxed = Box(i32).init(10);
    boxed.get_mut().* += 5;
    std.debug.print("Boxed value: {}\n", .{boxed.get().*});

    // Lazy
    var lazy = Lazy(i32).init(&expensiveComputation);
    std.debug.print("Before get\n", .{});
    std.debug.print("Value: {}\n", .{lazy.get()}); // "Computing..." จะปรากฏตอนนี้
    std.debug.print("Value again: {}\n", .{lazy.get()}); // ไม่มี "Computing..." อีก
}
```

---

## Step 224: Generic Stack, Queue, Linked List

### Generic Stack

```zig
const std = @import("std");

fn Stack(comptime T: type) type {
    return struct {
        items: std.ArrayList(T),

        const Self = @This();
        pub const Error = error{ StackEmpty, OutOfMemory };

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{ .items = std.ArrayList(T).init(allocator) };
        }

        pub fn deinit(self: *Self) void {
            self.items.deinit();
        }

        pub fn push(self: *Self, item: T) !void {
            try self.items.append(item);
        }

        pub fn pop(self: *Self) Error!T {
            if (self.items.items.len == 0) return Error.StackEmpty;
            return self.items.pop();
        }

        pub fn peek(self: *const Self) Error!T {
            if (self.items.items.len == 0) return Error.StackEmpty;
            return self.items.items[self.items.items.len - 1];
        }

        pub fn is_empty(self: *const Self) bool {
            return self.items.items.len == 0;
        }

        pub fn size(self: *const Self) usize {
            return self.items.items.len;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var stack = Stack(i32).init(allocator);
    defer stack.deinit();

    try stack.push(1);
    try stack.push(2);
    try stack.push(3);

    std.debug.print("Size: {}\n", .{stack.size()});
    std.debug.print("Peek: {}\n", .{try stack.peek()});

    while (!stack.is_empty()) {
        std.debug.print("Pop: {}\n", .{try stack.pop()});
    }
}
```

### Generic Queue (FIFO)

```zig
const std = @import("std");

fn Queue(comptime T: type) type {
    return struct {
        items: std.TailQueue(T),
        allocator: std.mem.Allocator,
        len: usize,

        const Self = @This();
        const Node = std.TailQueue(T).Node;

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .items = .{},
                .allocator = allocator,
                .len = 0,
            };
        }

        pub fn deinit(self: *Self) void {
            while (self.items.popFirst()) |node| {
                self.allocator.destroy(node);
            }
        }

        pub fn enqueue(self: *Self, value: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .data = value };
            self.items.append(node);
            self.len += 1;
        }

        pub fn dequeue(self: *Self) ?T {
            const node = self.items.popFirst() orelse return null;
            defer self.allocator.destroy(node);
            self.len -= 1;
            return node.data;
        }

        pub fn front(self: *const Self) ?T {
            const node = self.items.first orelse return null;
            return node.data;
        }

        pub fn is_empty(self: *const Self) bool {
            return self.len == 0;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var queue = Queue([]const u8).init(allocator);
    defer queue.deinit();

    try queue.enqueue("first");
    try queue.enqueue("second");
    try queue.enqueue("third");

    std.debug.print("Front: {s}\n", .{queue.front().?});

    while (queue.dequeue()) |item| {
        std.debug.print("Dequeue: {s}\n", .{item});
    }
}
```

### Generic Doubly Linked List

```zig
const std = @import("std");

fn LinkedList(comptime T: type) type {
    return struct {
        const Node = struct {
            value: T,
            prev: ?*Node,
            next: ?*Node,
        };

        head: ?*Node,
        tail: ?*Node,
        length: usize,
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .head = null,
                .tail = null,
                .length = 0,
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            var current = self.head;
            while (current) |node| {
                const next = node.next;
                self.allocator.destroy(node);
                current = next;
            }
        }

        pub fn push_front(self: *Self, value: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .value = value, .prev = null, .next = self.head };
            if (self.head) |head| {
                head.prev = node;
            } else {
                self.tail = node;
            }
            self.head = node;
            self.length += 1;
        }

        pub fn push_back(self: *Self, value: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .value = value, .prev = self.tail, .next = null };
            if (self.tail) |tail| {
                tail.next = node;
            } else {
                self.head = node;
            }
            self.tail = node;
            self.length += 1;
        }

        pub fn pop_front(self: *Self) ?T {
            const node = self.head orelse return null;
            defer self.allocator.destroy(node);

            self.head = node.next;
            if (self.head) |new_head| {
                new_head.prev = null;
            } else {
                self.tail = null;
            }
            self.length -= 1;
            return node.value;
        }

        pub fn pop_back(self: *Self) ?T {
            const node = self.tail orelse return null;
            defer self.allocator.destroy(node);

            self.tail = node.prev;
            if (self.tail) |new_tail| {
                new_tail.next = null;
            } else {
                self.head = null;
            }
            self.length -= 1;
            return node.value;
        }

        pub fn iter(self: *const Self) Iterator {
            return .{ .current = self.head };
        }

        const Iterator = struct {
            current: ?*Node,

            pub fn next(self: *Iterator) ?T {
                const node = self.current orelse return null;
                self.current = node.next;
                return node.value;
            }
        };
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = LinkedList(i32).init(allocator);
    defer list.deinit();

    try list.push_back(1);
    try list.push_back(2);
    try list.push_back(3);
    try list.push_front(0);

    std.debug.print("Length: {}\n", .{list.length});

    var iter = list.iter();
    while (iter.next()) |value| {
        std.debug.print("{} ", .{value});
    }
    std.debug.print("\n", .{});

    std.debug.print("Pop front: {}\n", .{list.pop_front().?});
    std.debug.print("Pop back: {}\n", .{list.pop_back().?});
}
```

---

## Step 225: Generic Sort Algorithm

### Generic Comparison-Based Sort

```zig
const std = @import("std");

// Generic insertion sort
fn insertionSort(comptime T: type, items: []T, lessThan: fn (T, T) bool) void {
    var i: usize = 1;
    while (i < items.len) : (i += 1) {
        const key = items[i];
        var j: usize = i;
        while (j > 0 and lessThan(key, items[j - 1])) : (j -= 1) {
            items[j] = items[j - 1];
        }
        items[j] = key;
    }
}

// Generic merge sort
fn mergeSort(
    comptime T: type,
    items: []T,
    lessThan: fn (T, T) bool,
    allocator: std.mem.Allocator,
) !void {
    if (items.len <= 1) return;

    const mid = items.len / 2;
    const left = items[0..mid];
    const right = items[mid..];

    try mergeSort(T, left, lessThan, allocator);
    try mergeSort(T, right, lessThan, allocator);

    const temp = try allocator.alloc(T, items.len);
    defer allocator.free(temp);

    var i: usize = 0;
    var j: usize = 0;
    var k: usize = 0;

    while (i < left.len and j < right.len) {
        if (lessThan(left[i], right[j])) {
            temp[k] = left[i];
            i += 1;
        } else {
            temp[k] = right[j];
            j += 1;
        }
        k += 1;
    }

    while (i < left.len) : (i += 1) {
        temp[k] = left[i];
        k += 1;
    }

    while (j < right.len) : (j += 1) {
        temp[k] = right[j];
        k += 1;
    }

    @memcpy(items, temp);
}

// Generic quicksort
fn quickSort(comptime T: type, items: []T, lessThan: fn (T, T) bool) void {
    if (items.len <= 1) return;

    const pivot_idx = partition(T, items, lessThan);
    quickSort(T, items[0..pivot_idx], lessThan);
    quickSort(T, items[pivot_idx + 1 ..], lessThan);
}

fn partition(comptime T: type, items: []T, lessThan: fn (T, T) bool) usize {
    const pivot = items[items.len - 1];
    var i: usize = 0;

    for (0..items.len - 1) |j| {
        if (lessThan(items[j], pivot)) {
            const temp = items[i];
            items[i] = items[j];
            items[j] = temp;
            i += 1;
        }
    }

    const temp = items[i];
    items[i] = items[items.len - 1];
    items[items.len - 1] = temp;

    return i;
}

fn intLessThan(a: i32, b: i32) bool {
    return a < b;
}

fn strLessThan(a: []const u8, b: []const u8) bool {
    return std.mem.lessThan(u8, a, b);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Sort integers
    var ints = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    insertionSort(i32, &ints, intLessThan);
    std.debug.print("Insertion sorted: {any}\n", .{ints});

    var ints2 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    try mergeSort(i32, &ints2, intLessThan, allocator);
    std.debug.print("Merge sorted: {any}\n", .{ints2});

    var ints3 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    quickSort(i32, &ints3, intLessThan);
    std.debug.print("Quick sorted: {any}\n", .{ints3});

    // Sort strings
    var strs = [_][]const u8{ "banana", "apple", "cherry", "date" };
    insertionSort([]const u8, &strs, strLessThan);
    for (strs) |s| std.debug.print("{s} ", .{s});
    std.debug.print("\n", .{});
}
```

### Generic Binary Search

```zig
const std = @import("std");

// Generic binary search
fn binarySearch(
    comptime T: type,
    items: []const T,
    target: T,
    compareFn: fn (T, T) std.math.Order,
) ?usize {
    var low: usize = 0;
    var high: usize = items.len;

    while (low < high) {
        const mid = low + (high - low) / 2;
        switch (compareFn(items[mid], target)) {
            .eq => return mid,
            .lt => low = mid + 1,
            .gt => high = mid,
        }
    }

    return null;
}

fn compareI32(a: i32, b: i32) std.math.Order {
    if (a < b) return .lt;
    if (a > b) return .gt;
    return .eq;
}

pub fn main() void {
    const sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15 };

    if (binarySearch(i32, &sorted, 7, compareI32)) |idx| {
        std.debug.print("Found 7 at index {}\n", .{idx});
    }

    if (binarySearch(i32, &sorted, 6, compareI32)) |_| {
        std.debug.print("Found 6\n", .{});
    } else {
        std.debug.print("6 not found\n", .{});
    }
}
```

---

## Step 226: Type Constraints ด้วย comptime

### การตรวจสอบ Type ด้วย @typeInfo

```zig
const std = @import("std");

// ตรวจสอบว่า type เป็น numeric
fn isNumeric(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int, .float, .comptime_int, .comptime_float => true,
        else => false,
    };
}

// ตรวจสอบว่า type เป็น integer
fn isInteger(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int, .comptime_int => true,
        else => false,
    };
}

// Generic sum ที่ต้องการ numeric type เท่านั้น
fn sum(comptime T: type, items: []const T) T {
    // Constraint: T ต้องเป็น numeric
    comptime {
        if (!isNumeric(T)) {
            @compileError("sum requires a numeric type, got: " ++ @typeName(T));
        }
    }

    var total: T = 0;
    for (items) |item| {
        total += item;
    }
    return total;
}

// Generic average
fn average(comptime T: type, items: []const T) f64 {
    comptime {
        if (!isNumeric(T)) {
            @compileError("average requires a numeric type");
        }
        if (items.len == 0) {
            @compileError("average requires non-empty slice");
        }
    }
    var total: f64 = 0;
    for (items) |item| {
        total += @as(f64, @floatFromInt(item));
    }
    return total / @as(f64, @floatFromInt(items.len));
}

pub fn main() void {
    const ints = [_]i32{ 1, 2, 3, 4, 5 };
    const floats = [_]f32{ 1.1, 2.2, 3.3 };

    std.debug.print("Sum of ints: {}\n", .{sum(i32, &ints)});
    std.debug.print("Sum of floats: {d:.1}\n", .{sum(f32, &floats)});

    // นี้จะ compile error:
    // sum([]const u8, &.{"hello"})
}
```

### Structural Type Constraints

```zig
const std = @import("std");

// ตรวจสอบว่า struct มี method ที่ต้องการ
fn hasMethod(comptime T: type, comptime method_name: []const u8) bool {
    return @hasDecl(T, method_name);
}

// ตรวจสอบว่า type implement interface หนึ่งๆ
fn implementsComparable(comptime T: type) bool {
    return hasMethod(T, "compare");
}

// Generic sort ที่ต้องการ Comparable interface
fn genericSort(comptime T: type, items: []T) void {
    comptime {
        if (!implementsComparable(T)) {
            @compileError(@typeName(T) ++ " must implement compare() method");
        }
    }

    // Bubble sort for simplicity
    var i: usize = 0;
    while (i < items.len) : (i += 1) {
        var j: usize = 0;
        while (j < items.len - i - 1) : (j += 1) {
            if (items[j].compare(items[j + 1]) == .gt) {
                const temp = items[j];
                items[j] = items[j + 1];
                items[j + 1] = temp;
            }
        }
    }
}

const Point = struct {
    x: i32,
    y: i32,

    pub fn compare(self: Point, other: Point) std.math.Order {
        const dist_self = self.x * self.x + self.y * self.y;
        const dist_other = other.x * other.x + other.y * other.y;
        if (dist_self < dist_other) return .lt;
        if (dist_self > dist_other) return .gt;
        return .eq;
    }

    pub fn format(
        self: Point,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("({}, {})", .{ self.x, self.y });
    }
};

pub fn main() void {
    var points = [_]Point{
        .{ .x = 3, .y = 4 },
        .{ .x = 1, .y = 1 },
        .{ .x = 5, .y = 0 },
        .{ .x = 2, .y = 2 },
    };

    genericSort(Point, &points);

    for (points) |p| {
        std.debug.print("{}\n", .{p});
    }
}
```

### comptime Assertions

```zig
const std = @import("std");

// ตรวจสอบ type properties ด้วย comptime
fn assertSameSize(comptime A: type, comptime B: type) void {
    comptime {
        if (@sizeOf(A) != @sizeOf(B)) {
            @compileError(std.fmt.comptimePrint(
                "Types must have same size: {} has size {}, {} has size {}",
                .{ @typeName(A), @sizeOf(A), @typeName(B), @sizeOf(B) },
            ));
        }
    }
}

// Generic bitcast
fn bitcast(comptime To: type, comptime From: type, value: From) To {
    assertSameSize(To, From);
    return @bitCast(value);
}

// Type-safe numeric cast
fn safeCast(comptime To: type, comptime From: type, value: From) ?To {
    comptime {
        if (!isNumericType(From) or !isNumericType(To)) {
            @compileError("safeCast requires numeric types");
        }
    }

    const result = @as(To, @intCast(value));
    // ตรวจสอบว่าไม่มี overflow
    if (@as(From, @intCast(result)) != value) return null;
    return result;
}

fn isNumericType(comptime T: type) bool {
    return switch (@typeInfo(T)) {
        .int, .float => true,
        else => false,
    };
}

pub fn main() void {
    const f: f32 = 1.5;
    const i: u32 = bitcast(u32, f32, f);
    std.debug.print("f32 bits: 0x{X}\n", .{i});

    // Safe cast
    const big: i64 = 1000;
    if (safeCast(i8, i64, big)) |v| {
        std.debug.print("Cast success: {}\n", .{v});
    } else {
        std.debug.print("Cast would overflow\n", .{});
    }

    const small: i64 = 100;
    if (safeCast(i8, i64, small)) |v| {
        std.debug.print("Cast success: {}\n", .{v});
    } else {
        std.debug.print("Cast would overflow\n", .{});
    }
}
```

---

## Step 227: Generic Interfaces (Duck Typing)

### Interface Pattern ด้วย comptime

Zig ใช้ "duck typing" ที่ compile time - ถ้า type มี methods ที่ต้องการ ก็ถือว่า implement interface นั้น

```zig
const std = @import("std");

// Writer interface
fn Writer(comptime T: type) type {
    // ตรวจสอบว่า T มี write method
    comptime {
        if (!@hasDecl(T, "write")) {
            @compileError(@typeName(T) ++ " must have write([]const u8) anyerror!usize");
        }
    }

    return struct {
        pub fn writeAll(impl: T, data: []const u8) !void {
            var remaining = data;
            while (remaining.len > 0) {
                const written = try impl.write(remaining);
                remaining = remaining[written..];
            }
        }

        pub fn writeFmt(impl: T, comptime fmt: []const u8, args: anytype) !void {
            var buf: [4096]u8 = undefined;
            const result = try std.fmt.bufPrint(&buf, fmt, args);
            try writeAll(impl, result);
        }
    };
}

// Concrete implementations
const StdoutWriter = struct {
    pub fn write(self: StdoutWriter, data: []const u8) !usize {
        _ = self;
        return std.io.getStdOut().write(data);
    }
};

const BufferWriter = struct {
    buffer: *std.ArrayList(u8),

    pub fn write(self: BufferWriter, data: []const u8) !usize {
        try self.buffer.appendSlice(data);
        return data.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Stdout writer
    const stdout = StdoutWriter{};
    const StdoutWriterInterface = Writer(StdoutWriter);
    try StdoutWriterInterface.writeFmt(stdout, "Hello, {}!\n", .{"World"});

    // Buffer writer
    var buffer = std.ArrayList(u8).init(allocator);
    defer buffer.deinit();

    const buf_writer = BufferWriter{ .buffer = &buffer };
    const BufferWriterInterface = Writer(BufferWriter);
    try BufferWriterInterface.writeFmt(buf_writer, "Buffered: {}\n", .{42});

    std.debug.print("Buffer contents: {s}", .{buffer.items});
}
```

### anytype Parameter

```zig
const std = @import("std");

// anytype ให้ compiler infer type ตอน compile time
fn printValue(value: anytype) void {
    const T = @TypeOf(value);
    switch (@typeInfo(T)) {
        .int => std.debug.print("int: {}\n", .{value}),
        .float => std.debug.print("float: {d}\n", .{value}),
        .bool => std.debug.print("bool: {}\n", .{value}),
        .pointer => |ptr| {
            if (ptr.child == u8) {
                std.debug.print("string: {s}\n", .{value});
            } else {
                std.debug.print("pointer: {any}\n", .{value});
            }
        },
        .array => std.debug.print("array: {any}\n", .{value}),
        else => std.debug.print("unknown type: {}\n", .{T}),
    }
}

// Generic equality check
fn equal(a: anytype, b: @TypeOf(a)) bool {
    const T = @TypeOf(a);
    return switch (@typeInfo(T)) {
        .int, .float, .bool => a == b,
        .array => |arr| blk: {
            _ = arr;
            break :blk std.mem.eql(@TypeOf(a[0]), &a, &b);
        },
        .pointer => |ptr| blk: {
            if (ptr.child == u8) {
                break :blk std.mem.eql(u8, a, b);
            }
            break :blk a == b;
        },
        else => a == b,
    };
}

pub fn main() void {
    printValue(42);
    printValue(3.14);
    printValue(true);
    printValue("hello");
    printValue([_]i32{ 1, 2, 3 });

    std.debug.print("equal(1, 1): {}\n", .{equal(@as(i32, 1), 1)});
    std.debug.print("equal(1, 2): {}\n", .{equal(@as(i32, 1), 2)});
    std.debug.print("equal(\"abc\", \"abc\"): {}\n", .{equal("abc", "abc")});
}
```

### Tagged Union สำหรับ Dynamic Dispatch

```zig
const std = @import("std");

// Simulation ของ polymorphism ผ่าน tagged union
const Shape = union(enum) {
    circle: Circle,
    rectangle: Rectangle,
    triangle: Triangle,

    pub fn area(self: Shape) f64 {
        return switch (self) {
            .circle => |c| c.area(),
            .rectangle => |r| r.area(),
            .triangle => |t| t.area(),
        };
    }

    pub fn perimeter(self: Shape) f64 {
        return switch (self) {
            .circle => |c| c.perimeter(),
            .rectangle => |r| r.perimeter(),
            .triangle => |t| t.perimeter(),
        };
    }
};

const Circle = struct {
    radius: f64,

    pub fn area(self: Circle) f64 {
        return std.math.pi * self.radius * self.radius;
    }

    pub fn perimeter(self: Circle) f64 {
        return 2.0 * std.math.pi * self.radius;
    }
};

const Rectangle = struct {
    width: f64,
    height: f64,

    pub fn area(self: Rectangle) f64 {
        return self.width * self.height;
    }

    pub fn perimeter(self: Rectangle) f64 {
        return 2.0 * (self.width + self.height);
    }
};

const Triangle = struct {
    base: f64,
    height: f64,
    a: f64,
    b: f64,
    c: f64,

    pub fn area(self: Triangle) f64 {
        return 0.5 * self.base * self.height;
    }

    pub fn perimeter(self: Triangle) f64 {
        return self.a + self.b + self.c;
    }
};

// Generic function ที่ทำงานกับ any Shape-like type
fn totalArea(shapes: []const Shape) f64 {
    var total: f64 = 0;
    for (shapes) |shape| {
        total += shape.area();
    }
    return total;
}

pub fn main() void {
    const shapes = [_]Shape{
        .{ .circle = .{ .radius = 5.0 } },
        .{ .rectangle = .{ .width = 4.0, .height = 6.0 } },
        .{ .triangle = .{ .base = 3.0, .height = 4.0, .a = 3.0, .b = 4.0, .c = 5.0 } },
    };

    for (shapes) |shape| {
        std.debug.print("Area: {d:.2}, Perimeter: {d:.2}\n", .{ shape.area(), shape.perimeter() });
    }

    std.debug.print("Total area: {d:.2}\n", .{totalArea(&shapes)});
}
```

---

## Step 228: Generic HashMap

### Generic HashMap Implementation

```zig
const std = @import("std");

fn HashMap(comptime K: type, comptime V: type) type {
    return struct {
        const Entry = struct {
            key: K,
            value: V,
            occupied: bool,
        };

        buckets: []Entry,
        count: usize,
        allocator: std.mem.Allocator,
        hash_fn: *const fn (K) u64,
        eq_fn: *const fn (K, K) bool,

        const Self = @This();
        const INITIAL_CAPACITY = 16;
        const LOAD_FACTOR = 0.75;

        pub fn init(
            allocator: std.mem.Allocator,
            hash_fn: *const fn (K) u64,
            eq_fn: *const fn (K, K) bool,
        ) !Self {
            const buckets = try allocator.alloc(Entry, INITIAL_CAPACITY);
            for (buckets) |*bucket| {
                bucket.occupied = false;
            }
            return .{
                .buckets = buckets,
                .count = 0,
                .allocator = allocator,
                .hash_fn = hash_fn,
                .eq_fn = eq_fn,
            };
        }

        pub fn deinit(self: *Self) void {
            self.allocator.free(self.buckets);
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            if (@as(f64, @floatFromInt(self.count + 1)) /
                @as(f64, @floatFromInt(self.buckets.len)) > LOAD_FACTOR)
            {
                try self.resize();
            }

            const idx = self.findSlot(key);
            if (!self.buckets[idx].occupied) {
                self.count += 1;
            }
            self.buckets[idx] = .{ .key = key, .value = value, .occupied = true };
        }

        pub fn get(self: *const Self, key: K) ?V {
            const idx = self.findSlotConst(key) orelse return null;
            return self.buckets[idx].value;
        }

        pub fn remove(self: *Self, key: K) bool {
            var hash = self.hash_fn(key) % self.buckets.len;
            while (self.buckets[hash].occupied) {
                if (self.eq_fn(self.buckets[hash].key, key)) {
                    self.buckets[hash].occupied = false;
                    self.count -= 1;
                    return true;
                }
                hash = (hash + 1) % self.buckets.len;
            }
            return false;
        }

        fn findSlot(self: *Self, key: K) usize {
            var hash = self.hash_fn(key) % self.buckets.len;
            while (self.buckets[hash].occupied and !self.eq_fn(self.buckets[hash].key, key)) {
                hash = (hash + 1) % self.buckets.len;
            }
            return hash;
        }

        fn findSlotConst(self: *const Self, key: K) ?usize {
            var hash = self.hash_fn(key) % self.buckets.len;
            var probe: usize = 0;
            while (probe < self.buckets.len) : (probe += 1) {
                if (!self.buckets[hash].occupied) return null;
                if (self.eq_fn(self.buckets[hash].key, key)) return hash;
                hash = (hash + 1) % self.buckets.len;
            }
            return null;
        }

        fn resize(self: *Self) !void {
            const old_buckets = self.buckets;
            defer self.allocator.free(old_buckets);

            const new_cap = old_buckets.len * 2;
            self.buckets = try self.allocator.alloc(Entry, new_cap);
            for (self.buckets) |*b| b.occupied = false;
            self.count = 0;

            for (old_buckets) |entry| {
                if (entry.occupied) {
                    try self.put(entry.key, entry.value);
                }
            }
        }
    };
}

fn hashString(s: []const u8) u64 {
    return std.hash.Fnv1a_64.hash(s);
}

fn eqString(a: []const u8, b: []const u8) bool {
    return std.mem.eql(u8, a, b);
}

fn hashInt(n: i32) u64 {
    return @as(u64, @bitCast(@as(i64, n)));
}

fn eqInt(a: i32, b: i32) bool {
    return a == b;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // String -> int map
    var word_count = try HashMap([]const u8, i32).init(allocator, hashString, eqString);
    defer word_count.deinit();

    const words = [_][]const u8{ "hello", "world", "hello", "zig", "world", "hello" };
    for (words) |word| {
        const current = word_count.get(word) orelse 0;
        try word_count.put(word, current + 1);
    }

    std.debug.print("hello: {?}\n", .{word_count.get("hello")});
    std.debug.print("world: {?}\n", .{word_count.get("world")});
    std.debug.print("zig: {?}\n", .{word_count.get("zig")});

    // Int -> string map
    var id_map = try HashMap(i32, []const u8).init(allocator, hashInt, eqInt);
    defer id_map.deinit();

    try id_map.put(1, "Alice");
    try id_map.put(2, "Bob");
    try id_map.put(3, "Charlie");

    std.debug.print("ID 2: {?s}\n", .{id_map.get(2)});
}
```

---

## Step 229: Generic Result Type

### Generic Error Handling ด้วย Result Type

```zig
const std = @import("std");

// Generic Result type (เหมือน Rust's Result<T, E>)
fn Result(comptime T: type, comptime E: type) type {
    return union(enum) {
        ok: T,
        err: E,

        const Self = @This();

        pub fn success(value: T) Self {
            return .{ .ok = value };
        }

        pub fn failure(error_val: E) Self {
            return .{ .err = error_val };
        }

        pub fn is_ok(self: Self) bool {
            return self == .ok;
        }

        pub fn is_err(self: Self) bool {
            return self == .err;
        }

        pub fn unwrap(self: Self) T {
            return switch (self) {
                .ok => |v| v,
                .err => @panic("Called unwrap on Err"),
            };
        }

        pub fn unwrap_or(self: Self, default: T) T {
            return switch (self) {
                .ok => |v| v,
                .err => default,
            };
        }

        pub fn unwrap_err(self: Self) E {
            return switch (self) {
                .ok => @panic("Called unwrap_err on Ok"),
                .err => |e| e,
            };
        }

        // map transforms Ok value
        pub fn map(self: Self, comptime U: type, f: fn (T) U) Result(U, E) {
            return switch (self) {
                .ok => |v| Result(U, E).success(f(v)),
                .err => |e| Result(U, E).failure(e),
            };
        }

        // map_err transforms Err value
        pub fn map_err(self: Self, comptime F: type, f: fn (E) F) Result(T, F) {
            return switch (self) {
                .ok => |v| Result(T, F).success(v),
                .err => |e| Result(T, F).failure(f(e)),
            };
        }

        // and_then chains operations
        pub fn and_then(self: Self, comptime U: type, f: fn (T) Result(U, E)) Result(U, E) {
            return switch (self) {
                .ok => |v| f(v),
                .err => |e| Result(U, E).failure(e),
            };
        }
    };
}

// ตัวอย่างการใช้งาน
const ParseError = enum {
    empty_input,
    invalid_number,
    out_of_range,
};

fn parseInt(s: []const u8) Result(i64, ParseError) {
    if (s.len == 0) return Result(i64, ParseError).failure(.empty_input);

    const value = std.fmt.parseInt(i64, s, 10) catch {
        return Result(i64, ParseError).failure(.invalid_number);
    };

    if (value < -1000 or value > 1000) {
        return Result(i64, ParseError).failure(.out_of_range);
    }

    return Result(i64, ParseError).success(value);
}

fn doubleValue(x: i64) i64 {
    return x * 2;
}

pub fn main() void {
    const inputs = [_][]const u8{ "42", "", "abc", "9999", "-50" };

    for (inputs) |input| {
        const result = parseInt(input);
        if (result.is_ok()) {
            const doubled = result.map(i64, doubleValue);
            std.debug.print("'{s}' -> {} (doubled: {})\n", .{ input, result.unwrap(), doubled.unwrap() });
        } else {
            std.debug.print("'{s}' -> Error: {}\n", .{ input, result.unwrap_err() });
        }
    }
}
```

### Generic Accumulator Pattern

```zig
const std = @import("std");

// Collect results, stop on first error
fn collectResults(
    comptime T: type,
    comptime E: type,
    items: []const T,
    process: fn (T) anyerror!T,
    allocator: std.mem.Allocator,
) ![]T {
    var results = std.ArrayList(T).init(allocator);
    errdefer results.deinit();

    for (items) |item| {
        const result = try process(item);
        try results.append(result);
    }

    return results.toOwnedSlice();
}

// Generic retry logic
fn withRetry(
    comptime T: type,
    max_attempts: u32,
    operation: fn () anyerror!T,
) anyerror!T {
    var last_err: anyerror = error.Unknown;
    var attempt: u32 = 0;
    while (attempt < max_attempts) : (attempt += 1) {
        return operation() catch |err| {
            last_err = err;
            std.debug.print("Attempt {} failed: {}\n", .{ attempt + 1, err });
            continue;
        };
    }
    return last_err;
}

var attempt_count: u32 = 0;

fn flakyOperation() !i32 {
    attempt_count += 1;
    if (attempt_count < 3) return error.TemporaryFailure;
    return 42;
}

pub fn main() !void {
    attempt_count = 0;
    const result = try withRetry(i32, 5, flakyOperation);
    std.debug.print("Got: {} after {} attempts\n", .{ result, attempt_count });
}
```

---

## Step 230: Generic Patterns ใน Standard Library

### std.ArrayList - Generic Dynamic Array

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // std.ArrayList(T) คือ generic struct ที่ return โดย function
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // เพิ่มข้อมูล
    for (0..10) |i| {
        try list.append(@intCast(i * i));
    }

    std.debug.print("Items: {any}\n", .{list.items});

    // sort ด้วย generic comparator
    std.sort.pdq(i32, list.items, {}, std.sort.desc(i32));
    std.debug.print("Sorted desc: {any}\n", .{list.items});

    // filter ด้วย iteration
    var filtered = std.ArrayList(i32).init(allocator);
    defer filtered.deinit();

    for (list.items) |item| {
        if (item > 20) try filtered.append(item);
    }
    std.debug.print("Filtered (>20): {any}\n", .{filtered.items});
}
```

### std.HashMap - Generic Hash Table

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // AutoHashMap ใช้ default hash function
    var map = std.AutoHashMap(i32, []const u8).init(allocator);
    defer map.deinit();

    try map.put(1, "one");
    try map.put(2, "two");
    try map.put(3, "three");

    // get ส่งคืน optional
    if (map.get(2)) |value| {
        std.debug.print("Key 2: {s}\n", .{value});
    }

    // iterate
    var iter = map.iterator();
    while (iter.next()) |entry| {
        std.debug.print("{}: {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }

    // StringHashMap สำหรับ string keys
    var str_map = std.StringHashMap(f64).init(allocator);
    defer str_map.deinit();

    try str_map.put("pi", 3.14159);
    try str_map.put("e", 2.71828);
    try str_map.put("phi", 1.61803);

    std.debug.print("pi = {d:.5}\n", .{str_map.get("pi").?});
}
```

### std.mem.Allocator - Generic Interface Pattern

```zig
const std = @import("std");

// Allocator เป็นตัวอย่าง interface ใน Zig standard library
// ทุก allocator implement interface เดียวกัน

fn createArray(
    comptime T: type,
    allocator: std.mem.Allocator,
    len: usize,
    fill_value: T,
) ![]T {
    const arr = try allocator.alloc(T, len);
    @memset(arr, fill_value);
    return arr;
}

pub fn main() !void {
    // ลอง allocator ต่างๆ แต่ใช้ function เดิม

    // General Purpose Allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const arr1 = try createArray(i32, gpa.allocator(), 5, 42);
    defer gpa.allocator().free(arr1);
    std.debug.print("GPA array: {any}\n", .{arr1});

    // Arena Allocator
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();

    const arr2 = try createArray(f64, arena.allocator(), 3, 3.14);
    // ไม่ต้อง free เพราะ arena จะ free ทุกอย่างพร้อมกัน
    std.debug.print("Arena array: {any}\n", .{arr2});
}
```

### comptime Metaprogramming Patterns

```zig
const std = @import("std");

// สร้าง enum จาก string array ตอน compile time
fn makeEnum(comptime names: []const []const u8) type {
    var fields: [names.len]std.builtin.Type.EnumField = undefined;
    for (names, 0..) |name, i| {
        fields[i] = .{
            .name = name,
            .value = i,
        };
    }
    return @Type(.{
        .@"enum" = .{
            .tag_type = u32,
            .fields = &fields,
            .decls = &.{},
            .is_exhaustive = true,
        },
    });
}

// สร้าง struct จาก field definitions
fn makeStruct(comptime field_defs: anytype) type {
    const defs = @typeInfo(@TypeOf(field_defs)).@"struct".fields;
    var fields: [defs.len]std.builtin.Type.StructField = undefined;

    for (defs, 0..) |def, i| {
        const FieldType = @field(field_defs, def.name);
        fields[i] = .{
            .name = def.name,
            .type = FieldType,
            .default_value_ptr = null,
            .is_comptime = false,
            .alignment = @alignOf(FieldType),
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

const Direction = makeEnum(&.{ "north", "south", "east", "west" });

pub fn main() void {
    const dir: Direction = .north;
    std.debug.print("Direction: {}\n", .{dir});

    // ตรวจสอบ enum value
    const dir_int = @intFromEnum(dir);
    std.debug.print("As int: {}\n", .{dir_int});
}
```

### Generic Iterator Pattern

```zig
const std = @import("std");

// Generic Range iterator
fn Range(comptime T: type) type {
    return struct {
        current: T,
        end: T,
        step: T,

        const Self = @This();

        pub fn init(start: T, end: T, step: T) Self {
            return .{ .current = start, .end = end, .step = step };
        }

        pub fn next(self: *Self) ?T {
            if (self.current >= self.end) return null;
            const value = self.current;
            self.current += self.step;
            return value;
        }

        pub fn collect(self: *Self, allocator: std.mem.Allocator) ![]T {
            var result = std.ArrayList(T).init(allocator);
            while (self.next()) |value| {
                try result.append(value);
            }
            return result.toOwnedSlice();
        }
    };
}

// Generic Enumerate iterator
fn Enumerate(comptime T: type) type {
    return struct {
        items: []const T,
        index: usize,

        const Self = @This();
        const Item = struct { index: usize, value: T };

        pub fn init(items: []const T) Self {
            return .{ .items = items, .index = 0 };
        }

        pub fn next(self: *Self) ?Item {
            if (self.index >= self.items.len) return null;
            const item = Item{ .index = self.index, .value = self.items[self.index] };
            self.index += 1;
            return item;
        }
    };
}

// Generic Zip iterator
fn Zip(comptime A: type, comptime B: type) type {
    return struct {
        a_items: []const A,
        b_items: []const B,
        index: usize,

        const Self = @This();
        const Item = struct { first: A, second: B };

        pub fn init(a: []const A, b: []const B) Self {
            return .{ .a_items = a, .b_items = b, .index = 0 };
        }

        pub fn next(self: *Self) ?Item {
            if (self.index >= self.a_items.len or self.index >= self.b_items.len) return null;
            const item = Item{
                .first = self.a_items[self.index],
                .second = self.b_items[self.index],
            };
            self.index += 1;
            return item;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Range iterator
    var range = Range(i32).init(0, 10, 2);
    const collected = try range.collect(allocator);
    defer allocator.free(collected);
    std.debug.print("Range(0,10,2): {any}\n", .{collected});

    // Enumerate
    const fruits = [_][]const u8{ "apple", "banana", "cherry" };
    var enum_iter = Enumerate([]const u8).init(&fruits);
    while (enum_iter.next()) |item| {
        std.debug.print("[{}] {s}\n", .{ item.index, item.value });
    }

    // Zip
    const keys = [_][]const u8{ "a", "b", "c" };
    const values = [_]i32{ 1, 2, 3 };
    var zip_iter = Zip([]const u8, i32).init(&keys, &values);
    while (zip_iter.next()) |pair| {
        std.debug.print("{s} -> {}\n", .{ pair.first, pair.second });
    }
}
```

---

## เปรียบเทียบ Generics ในภาษาต่างๆ

### ตารางเปรียบเทียบ

| Feature | Zig | Go | Rust | C++ |
|---------|-----|-----|------|-----|
| Syntax | `comptime T: type` | `[T any]` | `<T>` | `template<typename T>` |
| Type constraints | `@typeInfo` + comptime | `interface{}` | Traits | `concepts` (C++20) |
| Monomorphization | ใช่ | ไม่ (boxing) | ใช่ | ใช่ |
| Runtime overhead | ไม่มี | บางส่วน | ไม่มี | ไม่มี |
| Compile time | นานขึ้น | เร็ว | นานขึ้น | นานมาก |
| Error messages | ดี | ดีมาก | ดีมาก | ซับซ้อน |
| Specialization | ไม่มี | ไม่มี | มี | มี |
| SFINAE/Enable_if | comptime if | ไม่มี | where clause | มี |

### ข้อดีของ Zig Generics

1. **Simple Concept**: ใช้ comptime แทน syntax พิเศษ ทำให้ concept น้อยลง
2. **First-class Types**: `type` เป็น value จริงๆ ไม่ใช่แค่ parameter
3. **Compile-time Reflection**: `@typeInfo`, `@hasDecl`, `@hasField` ทรงพลังมาก
4. **Duck typing**: ไม่ต้องประกาศ interface ล่วงหน้า
5. **Zero overhead**: Monomorphization ทุกครั้ง

### ข้อเสียของ Zig Generics

1. **Verbose**: ต้องส่ง type เป็น parameter ทุกครั้ง (เช่น `max(i32, a, b)`)
2. **Error messages**: อาจซับซ้อนเมื่อ constraint ซับซ้อน
3. **No specialization**: ไม่สามารถ override behavior สำหรับ type พิเศษ
4. **Learning curve**: ต้องเข้าใจ comptime evaluation

---

## Performance Characteristics

### Compile Time vs Runtime Trade-offs

```zig
const std = @import("std");

// Compile-time computation ด้วย generics
fn factorial(comptime n: u64) u64 {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}

// เปรียบเทียบ generic vs non-generic performance
fn sumGeneric(comptime T: type, items: []const T) T {
    var total: T = 0;
    for (items) |item| total += item;
    return total;
}

fn sumI32(items: []const i32) i32 {
    var total: i32 = 0;
    for (items) |item| total += item;
    return total;
}

pub fn main() void {
    // factorial computed at compile time - zero runtime cost
    const f10 = comptime factorial(10);
    std.debug.print("10! = {}\n", .{f10});

    // Generic version compiled to same machine code as specific version
    const data = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("Generic sum: {}\n", .{sumGeneric(i32, &data)});
    std.debug.print("Specific sum: {}\n", .{sumI32(&data)});
    // ทั้งสองให้ assembly เหมือนกัน
}
```

### Code Size Considerations

```zig
// Monomorphization อาจทำให้ binary ใหญ่ขึ้น
// เพราะ compiler สร้าง code แยกสำหรับแต่ละ type

// ถ้าต้องการลด code size ใช้ type erasure:
const std = @import("std");

// Type-erased interface (vtable approach)
const Printable = struct {
    ptr: *anyopaque,
    print_fn: *const fn (*anyopaque) void,

    pub fn print(self: Printable) void {
        self.print_fn(self.ptr);
    }
};

fn makePrintable(value: anytype) Printable {
    const T = @TypeOf(value.*);
    const Impl = struct {
        fn print(ptr: *anyopaque) void {
            const typed = @as(*T, @ptrCast(@alignCast(ptr)));
            std.debug.print("{any}\n", .{typed.*});
        }
    };
    return .{
        .ptr = value,
        .print_fn = Impl.print,
    };
}

pub fn main() void {
    var x: i32 = 42;
    var y: f64 = 3.14;
    var z: bool = true;

    const printables = [_]Printable{
        makePrintable(&x),
        makePrintable(&y),
        makePrintable(&z),
    };

    for (printables) |p| {
        p.print();
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Generic MinHeap

สร้าง generic min-heap data structure ที่รองรับ:
- `push(value: T)` - เพิ่ม element
- `pop() ?T` - ลบและส่งคืน minimum
- `peek() ?T` - ดู minimum โดยไม่ลบ
- รองรับ custom comparator

```zig
// Template:
fn MinHeap(comptime T: type) type {
    return struct {
        data: std.ArrayList(T),
        compare: *const fn(T, T) bool,

        // TODO: implement push, pop, peek
        // Hint: ใช้ binary heap algorithm
        // Parent of i: (i - 1) / 2
        // Left child: 2*i + 1
        // Right child: 2*i + 2
    };
}
```

### แบบฝึกหัดที่ 2: Generic Graph

สร้าง directed graph generic:
```zig
fn Graph(comptime V: type, comptime E: type) type {
    // V = vertex data type
    // E = edge weight type
    return struct {
        // TODO: implement:
        // - add_vertex(data: V) -> VertexId
        // - add_edge(from: VertexId, to: VertexId, weight: E) -> void
        // - neighbors(v: VertexId) -> []const VertexId
        // - bfs(start: VertexId) -> Iterator
        // - dfs(start: VertexId) -> Iterator
    };
}
```

### แบบฝึกหัดที่ 3: Generic Pipeline

สร้าง data processing pipeline:
```zig
fn Pipeline(comptime T: type) type {
    return struct {
        // TODO: implement:
        // - source(data: []const T) -> Self
        // - filter(pred: fn(T) bool) -> Self  (chainable)
        // - map(comptime U: type, f: fn(T) U) -> Pipeline(U)
        // - reduce(comptime Acc: type, init: Acc, f: fn(Acc, T) Acc) -> Acc
        // - collect(allocator) -> ![]T
    };
}

// Expected usage:
// const result = try Pipeline(i32)
//     .source(&numbers)
//     .filter(isEven)
//     .map(i64, double)
//     .collect(allocator);
```

### แบบฝึกหัดที่ 4: Generic LRU Cache

```zig
fn LRUCache(comptime K: type, comptime V: type, capacity: usize) type {
    return struct {
        // TODO: implement Least Recently Used cache
        // - get(key: K) -> ?V
        // - put(key: K, value: V) -> void
        // เมื่อ cache เต็ม ลบ item ที่ใช้งานนานที่สุด

        // Hint: ใช้ doubly linked list + hash map
    };
}
```

### แบบฝึกหัดที่ 5: Type-Level Computation

```zig
// ใช้ comptime สร้าง tuple type
fn Tuple(comptime types: []const type) type {
    // TODO: สร้าง struct ที่มี fields named "field_0", "field_1", etc.
    // แต่ละ field มี type ตาม types[i]
    // เช่น Tuple(&.{i32, f64, []const u8}) ควรเป็น:
    // struct {
    //     field_0: i32,
    //     field_1: f64,
    //     field_2: []const u8,
    // }
}
```

---

## สรุป

### Key Takeaways

1. **Zig generics ทำงานผ่าน `comptime`**: ไม่มี syntax พิเศษ แต่ใช้ `comptime T: type` parameter

2. **Types เป็น first-class values**: สามารถเก็บใน variable, ส่งเป็น argument, return จาก function

3. **Generic structs ใช้ function returning type**: `fn Stack(comptime T: type) type { return struct { ... }; }`

4. **Type constraints ผ่าน `@typeInfo` และ `comptime if`**: สามารถตรวจสอบ properties ของ type ตอน compile time

5. **Duck typing**: ตรวจสอบ method/field ด้วย `@hasDecl`, `@hasField` - ไม่ต้องประกาศ interface ล่วงหน้า

6. **Zero overhead abstraction**: Monomorphization ทำให้ไม่มี runtime cost

7. **`anytype`**: ให้ compiler infer type โดยอัตโนมัติ - ง่ายกว่าการส่ง explicit type parameter

### Pattern Quick Reference

```zig
// Generic function
fn foo(comptime T: type, value: T) T { ... }

// Generic struct
fn MyStruct(comptime T: type) type {
    return struct { data: T, ... };
}

// Constraint check
comptime {
    if (@typeInfo(T) != .int) @compileError("need integer");
}

// Duck typing check
comptime {
    if (!@hasDecl(T, "len")) @compileError("need len method");
}

// anytype parameter
fn bar(value: anytype) void {
    const T = @TypeOf(value);
    // ...
}

// Compile-time type manipulation
const MyType = @Type(.{ .int = .{ .bits = 32, .signedness = .signed } });
```

### ขั้นตอนต่อไป

- **Part 24**: Comptime Programming เพิ่มเติม
- **Part 25**: Meta-programming และ Reflection
- **Part 26**: SIMD และ Vector Types
- **Part 27**: Build System และ Package Management

---

*Part 23 เสร็จสมบูรณ์ - ครอบคลุม Steps 221-230: Generics ใน Zig*
