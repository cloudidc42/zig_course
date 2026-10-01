# Part 40: Debugging ใน Zig

## บทนำ

การ debug โปรแกรม Zig มีเครื่องมือหลายอย่างทั้งจาก standard library และ tools ภายนอก บทนี้จะครอบคลุมเทคนิค debugging ตั้งแต่การใช้ `std.debug.print` อย่างง่ายจนถึงการใช้ GDB/LLDB และ production logging strategies

---

## Step 391: std.debug.print สำหรับ Debugging

### พื้นฐาน std.debug.print

```zig
const std = @import("std");

pub fn main() void {
    // print พื้นฐาน
    std.debug.print("Hello, Debug!\n", .{});

    // print ตัวแปร
    const x: i32 = 42;
    const pi: f64 = 3.14159;
    const name = "Alice";
    const active = true;

    std.debug.print("x = {}\n", .{x});
    std.debug.print("pi = {d:.2}\n", .{pi});
    std.debug.print("name = {s}\n", .{name});
    std.debug.print("active = {}\n", .{active});

    // print array/slice
    const nums = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("nums = {any}\n", .{nums});

    // print แบบ verbose (show type info)
    const point = struct { x: f32, y: f32 }{ .x = 1.0, .y = 2.0 };
    std.debug.print("point = {}\n", .{point});
}
```

### Debug Print Utilities

```zig
const std = @import("std");

// Custom debug helper functions
pub fn debugVar(comptime name: []const u8, value: anytype) void {
    std.debug.print("[DEBUG] {s} = {any} (type: {s})\n", .{
        name, value, @typeName(@TypeOf(value)),
    });
}

pub fn debugSlice(comptime name: []const u8, slice: anytype) void {
    std.debug.print("[DEBUG] {s} (len={}):\n", .{ name, slice.len });
    for (slice, 0..) |item, i| {
        std.debug.print("  [{}] = {any}\n", .{ i, item });
    }
}

pub fn debugSeparator(comptime label: []const u8) void {
    std.debug.print("\n=== {s} ===\n", .{label});
}

pub fn debugTrace(comptime file: []const u8, comptime line: u32, comptime msg: []const u8) void {
    std.debug.print("[TRACE] {s}:{d} - {s}\n", .{ file, line, msg });
}

pub fn main() void {
    debugSeparator("Variable Values");

    const count: u32 = 100;
    debugVar("count", count);

    const ratio: f64 = 0.75;
    debugVar("ratio", ratio);

    debugSeparator("Array Contents");

    const scores = [_]u8{ 95, 87, 92, 78, 88 };
    debugSlice("scores", &scores);

    debugSeparator("Trace");
    debugTrace(@src().file, @src().line, "reached this point");
}
```

### การใช้ @src() สำหรับ Location Info

```zig
const std = @import("std");

pub fn logWithLocation(comptime msg: []const u8) void {
    const src = @src();
    std.debug.print("[{s}:{d}:{d}] {s}\n", .{
        src.file,
        src.line,
        src.column,
        msg,
    });
}

pub fn assertWithMessage(condition: bool, comptime msg: []const u8) void {
    if (!condition) {
        const src = @src();
        std.debug.print("ASSERTION FAILED at {s}:{d}: {s}\n", .{
            src.file, src.line, msg,
        });
        @panic("assertion failed");
    }
}

pub fn main() void {
    logWithLocation("Starting main function");

    const x = 42;
    assertWithMessage(x > 0, "x must be positive");
    assertWithMessage(x < 100, "x must be less than 100");

    logWithLocation("All assertions passed");
}
```

### Conditional Debug Output

```zig
const std = @import("std");

// ควบคุม debug output ด้วย comptime flag
const DEBUG = true;

pub fn debugPrint(comptime fmt: []const u8, args: anytype) void {
    if (DEBUG) {
        std.debug.print("[DEBUG] " ++ fmt, args);
    }
}

pub fn verbosePrint(comptime level: u8, comptime fmt: []const u8, args: anytype) void {
    const VERBOSE_LEVEL: u8 = 2;
    if (level <= VERBOSE_LEVEL) {
        std.debug.print("[V{}] " ++ fmt, args);
    }
}

pub fn main() void {
    debugPrint("This is a debug message\n", .{});
    debugPrint("Value: {}\n", .{42});

    verbosePrint(1, "Level 1 verbose message\n", .{});
    verbosePrint(2, "Level 2 verbose message\n", .{});
    verbosePrint(3, "This should not appear\n", .{}); // VERBOSE_LEVEL = 2
}
```

---

## Step 392: std.log Module

### การใช้ std.log

```zig
const std = @import("std");

pub fn main() void {
    // Log levels ต่างๆ
    std.log.debug("This is a debug message\n", .{});
    std.log.info("Server started on port {}\n", .{8080});
    std.log.warn("Memory usage is {}%\n", .{85});
    std.log.err("Failed to connect to database\n", .{});
}
```

### Custom Log Handler

```zig
const std = @import("std");
const builtin = @import("builtin");

// Override log handler ด้วย custom implementation
pub const std_options = struct {
    pub const log_level = .debug; // กำหนด min log level

    pub fn logFn(
        comptime level: std.log.Level,
        comptime scope: @TypeOf(.enum_literal),
        comptime format: []const u8,
        args: anytype,
    ) void {
        const level_str = comptime switch (level) {
            .debug => "\x1b[36m[DEBUG]\x1b[0m", // Cyan
            .info => "\x1b[32m[INFO] \x1b[0m",  // Green
            .warn => "\x1b[33m[WARN] \x1b[0m",  // Yellow
            .err => "\x1b[31m[ERROR]\x1b[0m",   // Red
        };

        const scope_str = if (scope == .default) "" else "[" ++ @tagName(scope) ++ "] ";

        std.debug.print(level_str ++ " " ++ scope_str ++ format, args);
    }
};

pub fn main() void {
    std.log.debug("Debug info: {}\n", .{42});
    std.log.info("Application started\n", .{});
    std.log.warn("Deprecated function called\n", .{});
    std.log.err("Critical error occurred\n", .{});
}
```

### Scoped Logging

```zig
const std = @import("std");

// สร้าง logger สำหรับ module ต่างๆ
const db_log = std.log.scoped(.database);
const net_log = std.log.scoped(.network);
const auth_log = std.log.scoped(.auth);

pub const std_options = struct {
    pub const log_level = .debug;
};

fn simulateDatabase() void {
    db_log.info("Connecting to database...\n", .{});
    db_log.debug("Query executed in {}ms\n", .{15});
    db_log.warn("Slow query detected ({}ms)\n", .{500});
}

fn simulateNetwork() void {
    net_log.info("Listening on port {}\n", .{8080});
    net_log.debug("Received {} bytes\n", .{1024});
}

fn simulateAuth() void {
    auth_log.info("User logged in: {s}\n", .{"alice@example.com"});
    auth_log.warn("Failed login attempt from {s}\n", .{"192.168.1.100"});
    auth_log.err("Token expired for user {s}\n", .{"bob@example.com"});
}

pub fn main() void {
    simulateDatabase();
    simulateNetwork();
    simulateAuth();
}
```

### Log to File

```zig
const std = @import("std");

var log_file: ?std.fs.File = null;

pub const std_options = struct {
    pub const log_level = .info;

    pub fn logFn(
        comptime level: std.log.Level,
        comptime scope: @TypeOf(.enum_literal),
        comptime format: []const u8,
        args: anytype,
    ) void {
        _ = scope;

        const level_str = switch (level) {
            .debug => "DEBUG",
            .info => "INFO ",
            .warn => "WARN ",
            .err => "ERROR",
        };

        // Write to stderr
        std.debug.print("[{s}] " ++ format, .{level_str} ++ args);

        // Write to file if open
        if (log_file) |f| {
            f.writer().print("[{s}] " ++ format, .{level_str} ++ args) catch {};
        }
    }
};

pub fn main() !void {
    // Open log file
    log_file = try std.fs.cwd().createFile("app.log", .{});
    defer if (log_file) |f| f.close();

    std.log.info("Application started\n", .{});
    std.log.info("Processing {} items\n", .{100});
    std.log.warn("Warning: approaching memory limit\n", .{});
    std.log.err("Error: connection timeout\n", .{});

    std.log.info("Application stopped\n", .{});
}
```

---

## Step 393: @panic และ Stack Traces

### การใช้ @panic

```zig
const std = @import("std");

pub fn safeDivide(a: i32, b: i32) i32 {
    if (b == 0) {
        @panic("Division by zero!");
    }
    return @divTrunc(a, b);
}

pub fn getElement(slice: []const i32, index: usize) i32 {
    if (index >= slice.len) {
        std.debug.panic("Index {} out of bounds (len={})", .{ index, slice.len });
    }
    return slice[index];
}

pub fn unreachableCode(value: u32) []const u8 {
    return switch (value) {
        0 => "zero",
        1 => "one",
        2 => "two",
        else => unreachable, // จะ panic ใน Debug mode
    };
}

pub fn main() void {
    const nums = [_]i32{ 10, 20, 30 };

    std.debug.print("10 / 2 = {}\n", .{safeDivide(10, 2)});
    std.debug.print("nums[1] = {}\n", .{getElement(&nums, 1)});
    std.debug.print("0 = {s}\n", .{unreachableCode(0)});

    // ตัวอย่างที่จะ panic (comment out เพื่อ run):
    // _ = safeDivide(10, 0);
    // _ = getElement(&nums, 10);
}
```

### Stack Trace Analysis

```zig
const std = @import("std");

// ฟังก์ชันที่เรียกซ้อนกัน
fn level3() void {
    @panic("Error in level 3!");
}

fn level2() void {
    level3();
}

fn level1() void {
    level2();
}

// Stack trace จะแสดง:
// thread 10894 panic: Error in level 3!
// /path/to/file.zig:5:5: 0x... in level3 (debug_example)
// /path/to/file.zig:9:5: 0x... in level2 (debug_example)
// /path/to/file.zig:13:5: 0x... in level1 (debug_example)
// /path/to/file.zig:17:5: 0x... in main (debug_example)

pub fn captureStackTrace() void {
    var addresses: [32]usize = undefined;
    var trace = std.builtin.StackTrace{
        .instruction_addresses = &addresses,
        .index = 0,
    };
    std.debug.captureStackTrace(null, &trace);

    std.debug.print("Stack trace:\n", .{});
    std.debug.dumpStackTrace(trace);
}

pub fn main() void {
    captureStackTrace();

    // เรียก level1() จะทำให้ panic + stack trace
    // level1();
}
```

### Custom Panic Handler

```zig
const std = @import("std");
const builtin = @import("builtin");

// Custom panic handler
pub fn panic(msg: []const u8, error_return_trace: ?*std.builtin.StackTrace, ret_addr: ?usize) noreturn {
    std.debug.print("\n!!! PANIC !!!\n", .{});
    std.debug.print("Message: {s}\n", .{msg});
    std.debug.print("Return address: {?x}\n", .{ret_addr});

    if (error_return_trace) |trace| {
        std.debug.print("Error return trace:\n", .{});
        std.debug.dumpStackTrace(trace.*);
    }

    std.debug.print("Stack trace:\n", .{});
    std.debug.dumpCurrentStackTrace(ret_addr);

    std.process.exit(1);
}

pub fn main() void {
    std.debug.print("Before panic\n", .{});
    @panic("This is a custom panic message");
}
```

---

## Step 394: Compile-time Debugging (@compileLog)

### การใช้ @compileLog

```zig
const std = @import("std");

// @compileLog แสดงข้อมูลตอน compile time
pub fn genericFunction(comptime T: type) void {
    @compileLog("Type:", T);
    @compileLog("Size:", @sizeOf(T));
    @compileLog("Alignment:", @alignOf(T));
}

// ใช้สำหรับ debug comptime values
fn computeAtComptime(comptime n: u32) comptime_int {
    @compileLog("Computing for n =", n);
    return n * n + 2 * n + 1;
}

// @compileError - error ที่ compile time
fn mustBeUnsigned(comptime T: type) void {
    if (@typeInfo(T) != .Int or @typeInfo(T).Int.signedness != .unsigned) {
        @compileError("Expected unsigned integer, got: " ++ @typeName(T));
    }
}

pub fn main() void {
    // @compileLog จะแสดงใน stderr ตอน compile
    genericFunction(i32);
    genericFunction(f64);
    genericFunction([]const u8);

    const result = computeAtComptime(5);
    std.debug.print("Result: {}\n", .{result});

    mustBeUnsigned(u32); // OK
    // mustBeUnsigned(i32); // Compile error!
}
```

### Comptime Assertions

```zig
const std = @import("std");

fn assertSameSize(comptime A: type, comptime B: type) void {
    if (@sizeOf(A) != @sizeOf(B)) {
        @compileError(std.fmt.comptimePrint(
            "Size mismatch: {} ({} bytes) vs {} ({} bytes)",
            .{ @typeName(A), @sizeOf(A), @typeName(B), @sizeOf(B) },
        ));
    }
}

fn assertAligned(comptime T: type, comptime alignment: u29) void {
    if (@alignOf(T) < alignment) {
        @compileError(std.fmt.comptimePrint(
            "{} has alignment {} but required {}",
            .{ @typeName(T), @alignOf(T), alignment },
        ));
    }
}

const SmallStruct = packed struct {
    a: u8,
    b: u8,
    c: u8,
    d: u8,
};

const AnotherSmall = packed struct {
    x: u16,
    y: u16,
};

pub fn main() void {
    // These should compile fine
    assertSameSize(SmallStruct, AnotherSmall); // Both 4 bytes
    assertSameSize(u32, i32); // Both 4 bytes

    std.debug.print("All compile-time assertions passed\n", .{});
    std.debug.print("SmallStruct size: {} bytes\n", .{@sizeOf(SmallStruct)});
}
```

### Type Introspection สำหรับ Debugging

```zig
const std = @import("std");

pub fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("Type: {s}\n", .{@typeName(T)});
    std.debug.print("Size: {} bytes\n", .{@sizeOf(T)});
    std.debug.print("Alignment: {} bytes\n", .{@alignOf(T)});

    switch (info) {
        .Struct => |s| {
            std.debug.print("Kind: struct ({} fields)\n", .{s.fields.len});
            inline for (s.fields) |field| {
                std.debug.print("  .{s}: {s} (offset: {})\n", .{
                    field.name,
                    @typeName(field.type),
                    @offsetOf(T, field.name),
                });
            }
        },
        .Int => |i| {
            std.debug.print("Kind: {s} integer ({} bits)\n", .{
                @tagName(i.signedness), i.bits,
            });
        },
        .Float => |f| {
            std.debug.print("Kind: float ({} bits)\n", .{f.bits});
        },
        .Enum => |e| {
            std.debug.print("Kind: enum ({} variants)\n", .{e.fields.len});
        },
        else => std.debug.print("Kind: {s}\n", .{@tagName(info)}),
    }
}

const Config = struct {
    port: u16,
    debug: bool,
    name: [64]u8,
    timeout: u32,
};

pub fn main() void {
    printTypeInfo(Config);
    std.debug.print("\n", .{});
    printTypeInfo(u32);
    std.debug.print("\n", .{});
    printTypeInfo(f64);
}
```

---

## Step 395: Runtime Assertions

### std.debug.assert

```zig
const std = @import("std");

pub fn processData(data: []const u8) u32 {
    // Precondition
    std.debug.assert(data.len > 0); // Panic ใน Debug mode ถ้า false
    std.debug.assert(data.len <= 1024);

    var result: u32 = 0;
    for (data) |byte| {
        result +%= byte;
    }

    // Postcondition
    std.debug.assert(result >= data[0]);

    return result;
}

pub fn main() void {
    const data = [_]u8{ 10, 20, 30, 40, 50 };
    const sum = processData(&data);
    std.debug.print("Sum: {}\n", .{sum});

    // ตัวอย่างที่จะ panic ใน Debug mode:
    // _ = processData(&[_]u8{});  // empty data
}
```

### Safe wrappers

```zig
const std = @import("std");

// Safe array access
pub fn safeGet(comptime T: type, slice: []const T, index: usize) ?T {
    if (index >= slice.len) return null;
    return slice[index];
}

// Safe cast
pub fn safeCast(comptime To: type, value: anytype) ?To {
    const From = @TypeOf(value);
    const to_info = @typeInfo(To);
    const from_info = @typeInfo(From);

    if (to_info == .Int and from_info == .Int) {
        // Check bounds
        if (value < std.math.minInt(To) or value > std.math.maxInt(To)) {
            return null;
        }
        return @intCast(value);
    }
    return null;
}

pub fn checkedAdd(a: i32, b: i32) ?i32 {
    return std.math.add(i32, a, b) catch null;
}

pub fn checkedMul(a: i32, b: i32) ?i32 {
    return std.math.mul(i32, a, b) catch null;
}

pub fn main() void {
    const arr = [_]i32{ 10, 20, 30 };

    // Safe access
    std.debug.print("arr[1] = {?}\n", .{safeGet(i32, &arr, 1)});
    std.debug.print("arr[5] = {?}\n", .{safeGet(i32, &arr, 5)});

    // Safe cast
    std.debug.print("i64 100 -> i8: {?}\n", .{safeCast(i8, @as(i64, 100))});
    std.debug.print("i64 300 -> i8: {?}\n", .{safeCast(i8, @as(i64, 300))});

    // Checked arithmetic
    std.debug.print("2000000000 + 1000000000 = {?}\n", .{checkedAdd(2_000_000_000, 1_000_000_000)});
    std.debug.print("2 * 3 = {?}\n", .{checkedMul(2, 3)});
}
```

### Invariant Checking

```zig
const std = @import("std");

const SortedArray = struct {
    data: std.ArrayList(i32),

    pub fn init(allocator: std.mem.Allocator) SortedArray {
        return .{ .data = std.ArrayList(i32).init(allocator) };
    }

    pub fn deinit(self: *SortedArray) void {
        self.data.deinit();
    }

    fn checkInvariant(self: *const SortedArray) void {
        if (self.data.items.len < 2) return;
        for (0..self.data.items.len - 1) |i| {
            std.debug.assert(self.data.items[i] <= self.data.items[i + 1]);
        }
    }

    pub fn insert(self: *SortedArray, value: i32) !void {
        // Find insertion point
        var lo: usize = 0;
        var hi: usize = self.data.items.len;
        while (lo < hi) {
            const mid = lo + (hi - lo) / 2;
            if (self.data.items[mid] < value) {
                lo = mid + 1;
            } else {
                hi = mid;
            }
        }
        try self.data.insert(lo, value);

        // Verify invariant after mutation
        self.checkInvariant();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var arr = SortedArray.init(gpa.allocator());
    defer arr.deinit();

    try arr.insert(5);
    try arr.insert(2);
    try arr.insert(8);
    try arr.insert(1);
    try arr.insert(6);

    std.debug.print("Sorted: {any}\n", .{arr.data.items});
}
```

---

## Step 396: Valgrind กับ Zig

### การใช้ Valgrind

```bash
# Compile ด้วย debug info
zig build-exe -O Debug main.zig

# Run ด้วย valgrind
valgrind --tool=memcheck --leak-check=full ./main

# Valgrind กับ Callgrind (profiling)
valgrind --tool=callgrind ./main
callgrind_annotate callgrind.out.PID

# Valgrind กับ Helgrind (thread errors)
valgrind --tool=helgrind ./main
```

### ตัวอย่างที่มี Memory Issues

```zig
const std = @import("std");

// ตัวอย่าง: use after free (Valgrind จะตรวจพบ)
pub fn useAfterFreeExample() void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    const ptr = allocator.create(i32) catch unreachable;
    ptr.* = 42;
    allocator.destroy(ptr);
    // ptr.* = 100; // Use after free! Valgrind จะ report
    _ = gpa.deinit();
}

// ตัวอย่าง: memory leak (Valgrind จะตรวจพบ)
pub fn memoryLeakExample() void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    const data = allocator.alloc(u8, 1024) catch unreachable;
    _ = data;
    // allocator.free(data); // ลืม free! -> memory leak
    _ = gpa.deinit(); // GPA จะ report leak
}

pub fn main() void {
    std.debug.print("=== Memory Debugging Examples ===\n", .{});

    // สาธิต Zig's built-in memory checking
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const result = gpa.deinit();
        if (result == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }
    const allocator = gpa.allocator();

    // Correct usage
    const data = allocator.alloc(u8, 100) catch unreachable;
    defer allocator.free(data);

    @memset(data, 0);
    std.debug.print("Allocated and initialized {} bytes\n", .{data.len});
}
```

### AddressSanitizer ใน Zig

```bash
# Compile พร้อม AddressSanitizer
zig build-exe -O Debug -fsanitize-address main.zig

# หรือใน build.zig
# exe.addSanitize(.address);
```

```zig
// build.zig example
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my_app",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // Enable sanitizers ใน debug builds
    if (optimize == .Debug) {
        // exe.addSanitize(.address);
        // exe.addSanitize(.thread);
    }

    b.installArtifact(exe);
}
```

---

## Step 397: GDB/LLDB กับ Zig

### GDB Setup

```bash
# Compile ด้วย debug symbols
zig build-exe -O Debug main.zig

# Start GDB
gdb ./main

# GDB Commands ที่ใช้บ่อย:
# (gdb) run           - run program
# (gdb) break main    - set breakpoint ที่ function main
# (gdb) break file.zig:25  - set breakpoint ที่ line 25
# (gdb) next (n)      - step over
# (gdb) step (s)      - step into
# (gdb) continue (c)  - continue to next breakpoint
# (gdb) print x       - print value of x
# (gdb) backtrace (bt) - show call stack
# (gdb) info locals   - show local variables
# (gdb) quit (q)      - exit GDB
```

### LLDB Setup

```bash
# Start LLDB
lldb ./main

# LLDB Commands:
# (lldb) run                     - run program
# (lldb) breakpoint set -n main  - set breakpoint
# (lldb) thread step-over        - step over
# (lldb) thread step-in          - step into
# (lldb) thread continue         - continue
# (lldb) frame variable          - show local variables
# (lldb) print x                 - print value
# (lldb) thread backtrace        - show call stack
# (lldb) quit                    - exit
```

### ตัวอย่างโปรแกรมที่เหมาะสำหรับ Debug ด้วย GDB

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    left: ?*Node = null,
    right: ?*Node = null,
};

const BST = struct {
    root: ?*Node,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) BST {
        return .{ .root = null, .allocator = allocator };
    }

    pub fn deinit(self: *BST) void {
        self.destroyNode(self.root);
    }

    fn destroyNode(self: *BST, node: ?*Node) void {
        if (node) |n| {
            self.destroyNode(n.left);
            self.destroyNode(n.right);
            self.allocator.destroy(n);
        }
    }

    pub fn insert(self: *BST, value: i32) !void {
        self.root = try self.insertNode(self.root, value);
    }

    fn insertNode(self: *BST, node: ?*Node, value: i32) !?*Node {
        if (node == null) {
            const new_node = try self.allocator.create(Node);
            new_node.* = .{ .value = value };
            return new_node;
        }

        const n = node.?;
        if (value < n.value) {
            n.left = try self.insertNode(n.left, value);
        } else if (value > n.value) {
            n.right = try self.insertNode(n.right, value);
        }
        return n;
    }

    pub fn contains(self: *const BST, value: i32) bool {
        var current = self.root;
        while (current) |node| {
            if (value == node.value) return true;
            if (value < node.value) {
                current = node.left;
            } else {
                current = node.right;
            }
        }
        return false;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var tree = BST.init(gpa.allocator());
    defer tree.deinit();

    // ใส่ breakpoint ที่นี่เพื่อดู state
    const values = [_]i32{ 5, 3, 7, 1, 4, 6, 8 };
    for (values) |v| {
        try tree.insert(v);
    }

    // Debug point: ตรวจสอบ tree structure
    std.debug.print("Contains 4: {}\n", .{tree.contains(4)});
    std.debug.print("Contains 9: {}\n", .{tree.contains(9)});
}
```

---

## Step 398: Zig Debugger Support

### zig-lldb Extension

```bash
# ติดตั้ง extension สำหรับ VS Code
# 1. CodeLLDB extension
# 2. Zig Language extension (ziglang.vscode-zig)

# launch.json configuration
```

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug Zig",
            "program": "${workspaceFolder}/zig-out/bin/${workspaceFolderBasename}",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "zig build"
        }
    ]
}
```

### Debug Build Configuration

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่ม debug info เสมอ (แม้ใน ReleaseSafe)
    // exe.root_module.strip = false;

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## Step 399: Common Debugging Patterns

### Pattern 1: Trace Logging

```zig
const std = @import("std");

const Tracer = struct {
    name: []const u8,
    start_time: i128,

    pub fn begin(name: []const u8) Tracer {
        std.debug.print("[TRACE] -> {s}\n", .{name});
        return .{
            .name = name,
            .start_time = std.time.nanoTimestamp(),
        };
    }

    pub fn end(self: Tracer) void {
        const elapsed = std.time.nanoTimestamp() - self.start_time;
        std.debug.print("[TRACE] <- {s} ({}µs)\n", .{
            self.name,
            @divTrunc(elapsed, 1000),
        });
    }
};

pub fn expensiveOperation(n: u32) u64 {
    const tracer = Tracer.begin("expensiveOperation");
    defer tracer.end();

    var result: u64 = 0;
    var i: u32 = 0;
    while (i < n) : (i += 1) {
        result += i * i;
    }
    return result;
}

pub fn main() void {
    const tracer = Tracer.begin("main");
    defer tracer.end();

    const result = expensiveOperation(1000);
    std.debug.print("Result: {}\n", .{result});
}
```

### Pattern 2: State Dumper

```zig
const std = @import("std");

const AppState = struct {
    user_count: u32,
    request_count: u64,
    error_count: u32,
    memory_used: usize,
    uptime_seconds: u64,

    pub fn dump(self: AppState) void {
        std.debug.print(
            \\=== App State ===
            \\Users:    {}
            \\Requests: {}
            \\Errors:   {} ({d:.1}%)
            \\Memory:   {} KB
            \\Uptime:   {}s
            \\=================
            \\
        , .{
            self.user_count,
            self.request_count,
            self.error_count,
            if (self.request_count > 0)
                @as(f64, @floatFromInt(self.error_count)) / @as(f64, @floatFromInt(self.request_count)) * 100.0
            else
                0.0,
            self.memory_used / 1024,
            self.uptime_seconds,
        });
    }
};

pub fn main() void {
    var state = AppState{
        .user_count = 1234,
        .request_count = 98765,
        .error_count = 42,
        .memory_used = 45 * 1024 * 1024, // 45 MB
        .uptime_seconds = 3600 * 24, // 1 day
    };

    state.dump();

    // Simulate some activity
    state.request_count += 100;
    state.error_count += 3;

    state.dump();
}
```

### Pattern 3: Debug Mode สำหรับ Data Structures

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn HashMap(comptime K: type, comptime V: type) type {
    return struct {
        inner: std.HashMap(K, V, std.hash_map.AutoContext(K), std.hash_map.default_max_load_percentage),
        operation_count: u64,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .inner = std.HashMap(K, V, std.hash_map.AutoContext(K), std.hash_map.default_max_load_percentage).init(allocator),
                .operation_count = 0,
            };
        }

        pub fn deinit(self: *Self) void {
            if (builtin.mode == .Debug) {
                std.debug.print("[HashMap] deinit: {} operations performed\n", .{self.operation_count});
            }
            self.inner.deinit();
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            self.operation_count += 1;
            if (builtin.mode == .Debug and self.operation_count % 1000 == 0) {
                std.debug.print("[HashMap] {} ops, load={d:.1}%\n", .{
                    self.operation_count,
                    @as(f64, @floatFromInt(self.inner.count())) / @as(f64, @floatFromInt(self.inner.capacity())) * 100.0,
                });
            }
            try self.inner.put(key, value);
        }

        pub fn get(self: *const Self, key: K) ?V {
            return self.inner.get(key);
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var map = HashMap(u32, []const u8).init(gpa.allocator());
    defer map.deinit();

    try map.put(1, "one");
    try map.put(2, "two");
    try map.put(3, "three");

    if (map.get(2)) |v| {
        std.debug.print("map[2] = {s}\n", .{v});
    }
}
```

### Pattern 4: Hexdump

```zig
const std = @import("std");

pub fn hexdump(writer: anytype, data: []const u8) !void {
    var i: usize = 0;
    while (i < data.len) : (i += 16) {
        const chunk_end = @min(i + 16, data.len);
        const chunk = data[i..chunk_end];

        // Address
        try writer.print("{X:0>8}  ", .{i});

        // Hex bytes (first 8)
        for (0..8) |j| {
            if (j < chunk.len) {
                try writer.print("{X:0>2} ", .{chunk[j]});
            } else {
                try writer.print("   ", .{});
            }
        }
        try writer.print(" ", .{});

        // Hex bytes (second 8)
        for (8..16) |j| {
            if (j < chunk.len) {
                try writer.print("{X:0>2} ", .{chunk[j]});
            } else {
                try writer.print("   ", .{});
            }
        }

        try writer.print(" |", .{});

        // ASCII representation
        for (chunk) |byte| {
            if (byte >= 0x20 and byte < 0x7F) {
                try writer.print("{c}", .{byte});
            } else {
                try writer.print(".", .{});
            }
        }

        // Pad remaining
        for (chunk.len..16) |_| {
            try writer.print(" ", .{});
        }

        try writer.print("|\n", .{});
    }
}

pub fn main() !void {
    const data = "Hello, World! This is a hexdump example.\x00\x01\x02\xFF";

    std.debug.print("Hexdump of '{}' bytes:\n", .{data.len});
    try hexdump(std.io.getStdErr().writer(), data);
}
```

---

## Step 400: Production Logging Strategies

### Structured Logging

```zig
const std = @import("std");

const LogLevel = enum(u8) {
    trace = 0,
    debug = 1,
    info = 2,
    warn = 3,
    err = 4,
    fatal = 5,

    pub fn toString(self: LogLevel) []const u8 {
        return switch (self) {
            .trace => "TRACE",
            .debug => "DEBUG",
            .info => "INFO ",
            .warn => "WARN ",
            .err => "ERROR",
            .fatal => "FATAL",
        };
    }
};

const LogEntry = struct {
    level: LogLevel,
    timestamp: i64,
    message: []const u8,
    fields: std.StringHashMap([]const u8),
};

const Logger = struct {
    min_level: LogLevel,
    output: std.fs.File,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, min_level: LogLevel) Logger {
        return .{
            .min_level = min_level,
            .output = std.io.getStdErr(),
            .allocator = allocator,
        };
    }

    pub fn log(
        self: *const Logger,
        level: LogLevel,
        comptime format: []const u8,
        args: anytype,
    ) void {
        if (@intFromEnum(level) < @intFromEnum(self.min_level)) return;

        const timestamp = std.time.milliTimestamp();
        const writer = self.output.writer();

        writer.print("[{d}] [{s}] ", .{ timestamp, level.toString() }) catch return;
        writer.print(format ++ "\n", args) catch return;
    }

    pub fn info(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.info, fmt, args);
    }

    pub fn warn(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.warn, fmt, args);
    }

    pub fn err(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.err, fmt, args);
    }

    pub fn debug(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.debug, fmt, args);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const logger = Logger.init(gpa.allocator(), .info);

    logger.info("Application starting, version {s}", .{"1.2.3"});
    logger.info("Listening on port {}", .{8080});
    logger.debug("This won't show (below min_level)", .{});
    logger.warn("Memory usage at {}%", .{78});
    logger.err("Failed to connect to Redis: {s}", .{"connection refused"});
}
```

### Rotating Log Files

```zig
const std = @import("std");

const RotatingLogger = struct {
    base_path: []const u8,
    current_file: ?std.fs.File,
    current_size: usize,
    max_size: usize,
    max_files: u32,
    file_count: u32,
    allocator: std.mem.Allocator,

    pub fn init(
        allocator: std.mem.Allocator,
        base_path: []const u8,
        max_size: usize,
        max_files: u32,
    ) RotatingLogger {
        return .{
            .base_path = base_path,
            .current_file = null,
            .current_size = 0,
            .max_size = max_size,
            .max_files = max_files,
            .file_count = 0,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *RotatingLogger) void {
        if (self.current_file) |f| f.close();
    }

    fn openCurrentFile(self: *RotatingLogger) !void {
        const path = try std.fmt.allocPrint(
            self.allocator,
            "{s}.{}.log",
            .{ self.base_path, self.file_count },
        );
        defer self.allocator.free(path);

        self.current_file = try std.fs.cwd().createFile(path, .{ .truncate = false });
        // Seek to end
        try self.current_file.?.seekFromEnd(0);
    }

    fn rotate(self: *RotatingLogger) !void {
        if (self.current_file) |f| {
            f.close();
            self.current_file = null;
        }

        self.file_count = (self.file_count + 1) % self.max_files;
        self.current_size = 0;
        try self.openCurrentFile();
    }

    pub fn write(self: *RotatingLogger, message: []const u8) !void {
        if (self.current_file == null) {
            try self.openCurrentFile();
        }

        if (self.current_size + message.len > self.max_size) {
            try self.rotate();
        }

        try self.current_file.?.writeAll(message);
        self.current_size += message.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var logger = RotatingLogger.init(
        allocator,
        "/tmp/app",
        1024 * 100, // 100KB per file
        5,          // keep 5 files
    );
    defer logger.deinit();

    // Write some logs
    var i: usize = 0;
    while (i < 100) : (i += 1) {
        const msg = try std.fmt.allocPrint(allocator, "[INFO] Log message {}\n", .{i});
        defer allocator.free(msg);
        try logger.write(msg);
    }

    std.debug.print("Logged 100 messages successfully\n", .{});
}
```

### Metrics Collection

```zig
const std = @import("std");

const Counter = struct {
    name: []const u8,
    value: std.atomic.Value(u64),

    pub fn init(name: []const u8) Counter {
        return .{
            .name = name,
            .value = std.atomic.Value(u64).init(0),
        };
    }

    pub fn increment(self: *Counter) void {
        _ = self.value.fetchAdd(1, .monotonic);
    }

    pub fn add(self: *Counter, n: u64) void {
        _ = self.value.fetchAdd(n, .monotonic);
    }

    pub fn get(self: *const Counter) u64 {
        return self.value.load(.monotonic);
    }

    pub fn reset(self: *Counter) u64 {
        return self.value.swap(0, .seq_cst);
    }
};

const Metrics = struct {
    requests_total: Counter,
    errors_total: Counter,
    bytes_received: Counter,
    bytes_sent: Counter,

    pub fn init() Metrics {
        return .{
            .requests_total = Counter.init("requests_total"),
            .errors_total = Counter.init("errors_total"),
            .bytes_received = Counter.init("bytes_received"),
            .bytes_sent = Counter.init("bytes_sent"),
        };
    }

    pub fn dump(self: *const Metrics) void {
        std.debug.print("=== Metrics ===\n", .{});
        std.debug.print("requests_total: {}\n", .{self.requests_total.get()});
        std.debug.print("errors_total: {}\n", .{self.errors_total.get()});
        std.debug.print("bytes_received: {} KB\n", .{self.bytes_received.get() / 1024});
        std.debug.print("bytes_sent: {} KB\n", .{self.bytes_sent.get() / 1024});
        if (self.requests_total.get() > 0) {
            const error_rate = @as(f64, @floatFromInt(self.errors_total.get())) /
                @as(f64, @floatFromInt(self.requests_total.get())) * 100.0;
            std.debug.print("error_rate: {d:.2}%\n", .{error_rate});
        }
    }
};

pub fn main() void {
    var metrics = Metrics.init();

    // Simulate some activity
    var i: usize = 0;
    while (i < 1000) : (i += 1) {
        metrics.requests_total.increment();
        metrics.bytes_received.add(512);
        metrics.bytes_sent.add(256);

        if (i % 10 == 0) {
            metrics.errors_total.increment();
        }
    }

    metrics.dump();
}
```

---

## สรุปบทที่ 40

ในบทนี้เราได้เรียนรู้:

1. **std.debug.print** - การใช้สำหรับ quick debugging รวมถึง @src() และ conditional output
2. **std.log** - Structured logging พร้อม custom handlers และ scoped logging
3. **@panic และ Stack Traces** - การจัดการ panics และการอ่าน stack traces
4. **@compileLog** - Debugging ที่ compile time รวมถึง type introspection
5. **Runtime Assertions** - std.debug.assert, safe wrappers, invariant checking
6. **Valgrind** - การตรวจจับ memory issues กับ Zig programs
7. **GDB/LLDB** - Debugger commands และการ setup สำหรับ Zig
8. **Debugger Support** - VS Code configuration และ build.zig settings
9. **Common Patterns** - Tracer, state dumper, hexdump, debug data structures
10. **Production Logging** - Structured logging, rotating files, metrics collection

การ debug อย่างมีประสิทธิภาพเป็นทักษะที่สำคัญ Zig มีเครื่องมือที่ดีมากทั้งจาก built-in และ external tools ที่ช่วยให้ค้นหาและแก้ไข bugs ได้รวดเร็วขึ้น

---

## สรุปรวม Parts 36-40

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 36 | std.ArrayList | 351-360 |
| 37 | String Handling ขั้นสูง | 361-370 |
| 38 | std.json | 371-380 |
| 39 | std.testing | 381-390 |
| 40 | Debugging | 391-400 |

ทั้ง 5 บทนี้ครอบคลุม standard library ที่สำคัญของ Zig ซึ่งใช้งานบ่อยในการพัฒนาโปรแกรมจริงๆ ตั้งแต่การจัดการ dynamic data (ArrayList) การประมวลผล text (String Handling, JSON) การทดสอบ (testing) และการหาข้อผิดพลาด (debugging)
