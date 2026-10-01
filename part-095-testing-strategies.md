# Part 95: Testing Strategies ใน Zig

## Steps 941-950

---

## Step 941: Testing Philosophy

### ปรัชญาการทดสอบ

การทดสอบที่ดีไม่ใช่แค่การทำให้ code ผ่าน tests แต่คือการออกแบบระบบให้ testable และเข้าใจ behavior ที่ต้องการ

### Testing Pyramid

```
          /\
         /  \
        / E2E\
       /------\
      / Integ. \
     /----------\
    /   Unit     \
   /--------------\

Unit Tests:        มาก, เร็ว, isolated
Integration Tests: ปานกลาง, ช้ากว่า, กว้างกว่า
E2E Tests:         น้อย, ช้า, ครอบคลุม
```

### Testing Principles ใน Zig

```zig
const std = @import("std");
const testing = std.testing;

// 1. Tests อยู่ใกล้กับ code ที่ test
pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

test "add basic" {
    try testing.expectEqual(@as(i32, 3), add(1, 2));
    try testing.expectEqual(@as(i32, 0), add(-1, 1));
    try testing.expectEqual(@as(i32, -2), add(-1, -1));
}

// 2. Tests ควร F.I.R.S.T.
// Fast:      ทดสอบได้เร็ว
// Isolated:  ไม่ขึ้นกับ test อื่น
// Repeatable: ได้ผลเหมือนกันทุกครั้ง
// Self-validating: pass/fail ชัดเจน
// Timely:    เขียน tests พร้อมกับ code

// 3. Arrange-Act-Assert pattern
test "AAA pattern" {
    // Arrange
    const input = [_]i32{ 1, 2, 3, 4, 5 };
    
    // Act
    var sum: i32 = 0;
    for (input) |n| sum += n;
    
    // Assert
    try testing.expectEqual(@as(i32, 15), sum);
}

// 4. Test naming convention
test "functionName - scenario - expected result" {
    // Clear test names help understand failures
}

// 5. Testing error cases
pub fn divide(a: f64, b: f64) !f64 {
    if (b == 0) return error.DivisionByZero;
    return a / b;
}

test "divide - by zero - returns error" {
    try testing.expectError(error.DivisionByZero, divide(10, 0));
}

test "divide - normal - returns quotient" {
    const result = try divide(10, 2);
    try testing.expectApproxEqAbs(@as(f64, 5.0), result, 0.001);
}
```

### Testing Mindset

```zig
// Test behavior, not implementation
pub const Stack = struct {
    items: std.ArrayList(i32),

    pub fn init(allocator: std.mem.Allocator) Stack {
        return Stack{ .items = std.ArrayList(i32).init(allocator) };
    }

    pub fn deinit(self: *Stack) void {
        self.items.deinit();
    }

    pub fn push(self: *Stack, item: i32) !void {
        try self.items.append(item);
    }

    pub fn pop(self: *Stack) ?i32 {
        if (self.items.items.len == 0) return null;
        return self.items.pop();
    }

    pub fn peek(self: Stack) ?i32 {
        if (self.items.items.len == 0) return null;
        return self.items.items[self.items.items.len - 1];
    }

    pub fn isEmpty(self: Stack) bool {
        return self.items.items.len == 0;
    }

    pub fn size(self: Stack) usize {
        return self.items.items.len;
    }
};

// Test behavior, not internal state
test "Stack - LIFO order" {
    const allocator = testing.allocator;
    var stack = Stack.init(allocator);
    defer stack.deinit();

    try stack.push(1);
    try stack.push(2);
    try stack.push(3);

    try testing.expectEqual(@as(?i32, 3), stack.pop());
    try testing.expectEqual(@as(?i32, 2), stack.pop());
    try testing.expectEqual(@as(?i32, 1), stack.pop());
    try testing.expectEqual(@as(?i32, null), stack.pop());
}

test "Stack - isEmpty reflects state" {
    const allocator = testing.allocator;
    var stack = Stack.init(allocator);
    defer stack.deinit();

    try testing.expect(stack.isEmpty());
    try stack.push(42);
    try testing.expect(!stack.isEmpty());
    _ = stack.pop();
    try testing.expect(stack.isEmpty());
}
```

---

## Step 942: Unit Testing Strategies

### Comprehensive Unit Testing

```zig
const std = @import("std");
const testing = std.testing;

// Test helpers and utilities
pub const TestFixture = struct {
    allocator: std.mem.Allocator,
    arena: std.heap.ArenaAllocator,

    pub fn init(base_allocator: std.mem.Allocator) TestFixture {
        return TestFixture{
            .allocator = base_allocator,
            .arena = std.heap.ArenaAllocator.init(base_allocator),
        };
    }

    pub fn deinit(self: *TestFixture) void {
        self.arena.deinit();
    }

    pub fn tempAllocator(self: *TestFixture) std.mem.Allocator {
        return self.arena.allocator();
    }
};

// Subject under test
pub const Calculator = struct {
    memory: f64,
    history: std.ArrayList(f64),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Calculator {
        return Calculator{
            .memory = 0,
            .history = std.ArrayList(f64).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Calculator) void {
        self.history.deinit();
    }

    pub fn add(self: *Calculator, a: f64, b: f64) !f64 {
        const result = a + b;
        try self.history.append(result);
        return result;
    }

    pub fn subtract(self: *Calculator, a: f64, b: f64) !f64 {
        const result = a - b;
        try self.history.append(result);
        return result;
    }

    pub fn multiply(self: *Calculator, a: f64, b: f64) !f64 {
        const result = a * b;
        try self.history.append(result);
        return result;
    }

    pub fn divide(self: *Calculator, a: f64, b: f64) !f64 {
        if (b == 0) return error.DivisionByZero;
        const result = a / b;
        try self.history.append(result);
        return result;
    }

    pub fn storeMemory(self: *Calculator, value: f64) void {
        self.memory = value;
    }

    pub fn recallMemory(self: Calculator) f64 {
        return self.memory;
    }

    pub fn clearMemory(self: *Calculator) void {
        self.memory = 0;
    }

    pub fn getHistory(self: Calculator) []const f64 {
        return self.history.items;
    }

    pub fn clearHistory(self: *Calculator) void {
        self.history.clearRetainingCapacity();
    }
};

// Unit tests
test "Calculator.add - positive numbers" {
    const allocator = testing.allocator;
    var calc = Calculator.init(allocator);
    defer calc.deinit();

    const result = try calc.add(3.0, 4.0);
    try testing.expectApproxEqAbs(@as(f64, 7.0), result, 1e-10);
}

test "Calculator.add - negative numbers" {
    const allocator = testing.allocator;
    var calc = Calculator.init(allocator);
    defer calc.deinit();

    const result = try calc.add(-3.0, -4.0);
    try testing.expectApproxEqAbs(@as(f64, -7.0), result, 1e-10);
}

test "Calculator.divide - by zero" {
    const allocator = testing.allocator;
    var calc = Calculator.init(allocator);
    defer calc.deinit();

    try testing.expectError(error.DivisionByZero, calc.divide(10.0, 0.0));
}

test "Calculator.memory - store and recall" {
    const allocator = testing.allocator;
    var calc = Calculator.init(allocator);
    defer calc.deinit();

    calc.storeMemory(42.0);
    try testing.expectApproxEqAbs(@as(f64, 42.0), calc.recallMemory(), 1e-10);
    calc.clearMemory();
    try testing.expectApproxEqAbs(@as(f64, 0.0), calc.recallMemory(), 1e-10);
}

test "Calculator.history - tracks operations" {
    const allocator = testing.allocator;
    var calc = Calculator.init(allocator);
    defer calc.deinit();

    _ = try calc.add(1, 2);
    _ = try calc.multiply(3, 4);
    _ = try calc.subtract(10, 5);

    const history = calc.getHistory();
    try testing.expectEqual(@as(usize, 3), history.len);
    try testing.expectApproxEqAbs(@as(f64, 3.0), history[0], 1e-10);
    try testing.expectApproxEqAbs(@as(f64, 12.0), history[1], 1e-10);
    try testing.expectApproxEqAbs(@as(f64, 5.0), history[2], 1e-10);
}

// Parameterized tests
const AddTestCase = struct {
    a: f64,
    b: f64,
    expected: f64,
    name: []const u8,
};

test "Calculator.add - parameterized" {
    const allocator = testing.allocator;
    
    const test_cases = [_]AddTestCase{
        .{ .a = 1, .b = 2, .expected = 3, .name = "1+2=3" },
        .{ .a = -1, .b = 1, .expected = 0, .name = "-1+1=0" },
        .{ .a = 0, .b = 0, .expected = 0, .name = "0+0=0" },
        .{ .a = 100, .b = -50, .expected = 50, .name = "100-50=50" },
        .{ .a = 1.5, .b = 2.5, .expected = 4.0, .name = "1.5+2.5=4" },
    };

    for (test_cases) |tc| {
        var calc = Calculator.init(allocator);
        defer calc.deinit();

        const result = try calc.add(tc.a, tc.b);
        testing.expectApproxEqAbs(tc.expected, result, 1e-10) catch |e| {
            std.debug.print("Failed case '{s}': {}\n", .{ tc.name, e });
            return e;
        };
    }
}
```

---

## Step 943: Integration Testing

### Integration Testing Strategies

```zig
const std = @import("std");
const testing = std.testing;

// Database interface for testing
pub const DatabaseInterface = struct {
    ptr: *anyopaque,
    vtable: *const VTable,

    pub const VTable = struct {
        query: *const fn (*anyopaque, sql: []const u8, allocator: std.mem.Allocator) anyerror![]const u8,
        execute: *const fn (*anyopaque, sql: []const u8) anyerror!void,
        begin: *const fn (*anyopaque) anyerror!void,
        commit: *const fn (*anyopaque) anyerror!void,
        rollback: *const fn (*anyopaque) anyerror!void,
    };

    pub fn query(self: DatabaseInterface, sql: []const u8, allocator: std.mem.Allocator) ![]const u8 {
        return self.vtable.query(self.ptr, sql, allocator);
    }

    pub fn execute(self: DatabaseInterface, sql: []const u8) !void {
        return self.vtable.execute(self.ptr, sql);
    }

    pub fn begin(self: DatabaseInterface) !void {
        return self.vtable.begin(self.ptr);
    }

    pub fn commit(self: DatabaseInterface) !void {
        return self.vtable.commit(self.ptr);
    }

    pub fn rollback(self: DatabaseInterface) !void {
        return self.vtable.rollback(self.ptr);
    }
};

// In-memory database mock
pub const InMemoryDb = struct {
    tables: std.StringHashMap(std.ArrayList(std.StringHashMap([]const u8))),
    allocator: std.mem.Allocator,
    in_transaction: bool,

    pub fn init(allocator: std.mem.Allocator) InMemoryDb {
        return InMemoryDb{
            .tables = std.StringHashMap(std.ArrayList(std.StringHashMap([]const u8))).init(allocator),
            .allocator = allocator,
            .in_transaction = false,
        };
    }

    pub fn deinit(self: *InMemoryDb) void {
        var it = self.tables.valueIterator();
        while (it.next()) |rows| {
            for (rows.items) |*row| {
                row.deinit();
            }
            rows.deinit();
        }
        self.tables.deinit();
    }

    pub fn asInterface(self: *InMemoryDb) DatabaseInterface {
        const vtable = struct {
            const vt = DatabaseInterface.VTable{
                .query = queryImpl,
                .execute = executeImpl,
                .begin = beginImpl,
                .commit = commitImpl,
                .rollback = rollbackImpl,
            };

            fn queryImpl(ptr: *anyopaque, sql: []const u8, allocator: std.mem.Allocator) ![]const u8 {
                _ = ptr;
                return std.fmt.allocPrint(allocator, "[]", .{});
                _ = sql;
            }

            fn executeImpl(ptr: *anyopaque, sql: []const u8) !void {
                _ = ptr;
                std.debug.print("Execute: {s}\n", .{sql});
            }

            fn beginImpl(ptr: *anyopaque) !void {
                const self: *InMemoryDb = @ptrCast(@alignCast(ptr));
                self.in_transaction = true;
            }

            fn commitImpl(ptr: *anyopaque) !void {
                const self: *InMemoryDb = @ptrCast(@alignCast(ptr));
                self.in_transaction = false;
            }

            fn rollbackImpl(ptr: *anyopaque) !void {
                const self: *InMemoryDb = @ptrCast(@alignCast(ptr));
                self.in_transaction = false;
            }
        };

        return DatabaseInterface{
            .ptr = self,
            .vtable = &vtable.vt,
        };
    }
};

// Service that uses database
pub const UserService = struct {
    db: DatabaseInterface,
    allocator: std.mem.Allocator,

    pub fn init(db: DatabaseInterface, allocator: std.mem.Allocator) UserService {
        return UserService{ .db = db, .allocator = allocator };
    }

    pub fn createUser(self: UserService, name: []const u8, email: []const u8) !void {
        try self.db.begin();
        errdefer self.db.rollback() catch {};

        const sql = try std.fmt.allocPrint(
            self.allocator,
            "INSERT INTO users (name, email) VALUES ('{s}', '{s}')",
            .{ name, email },
        );
        defer self.allocator.free(sql);

        try self.db.execute(sql);
        try self.db.commit();
    }

    pub fn getUser(self: UserService, id: u64) ![]const u8 {
        const sql = try std.fmt.allocPrint(
            self.allocator,
            "SELECT * FROM users WHERE id = {}",
            .{id},
        );
        defer self.allocator.free(sql);

        return self.db.query(sql, self.allocator);
    }
};

// Integration test
test "UserService.createUser - integration" {
    const allocator = testing.allocator;

    var db = InMemoryDb.init(allocator);
    defer db.deinit();

    const service = UserService.init(db.asInterface(), allocator);

    // Should not error
    try service.createUser("Alice", "alice@example.com");
    try service.createUser("Bob", "bob@example.com");
}

test "UserService.getUser - integration" {
    const allocator = testing.allocator;

    var db = InMemoryDb.init(allocator);
    defer db.deinit();

    const service = UserService.init(db.asInterface(), allocator);

    const result = try service.getUser(1);
    defer allocator.free(result);

    // In-memory db returns empty array
    try testing.expectEqualStrings("[]", result);
}
```

---

## Step 944: Property-Based Testing

### Property-Based Testing ใน Zig

```zig
const std = @import("std");
const testing = std.testing;

// Simple random generator for testing
pub const TestRandom = struct {
    state: u64,

    pub fn init(seed: u64) TestRandom {
        return TestRandom{ .state = seed };
    }

    pub fn nextU32(self: *TestRandom) u32 {
        self.state = self.state *% 6364136223846793005 +% 1442695040888963407;
        return @truncate(self.state >> 33);
    }

    pub fn nextI32(self: *TestRandom) i32 {
        return @bitCast(self.nextU32());
    }

    pub fn nextBool(self: *TestRandom) bool {
        return self.nextU32() % 2 == 0;
    }

    pub fn nextF64(self: *TestRandom) f64 {
        const u = self.nextU32();
        return @as(f64, @floatFromInt(u)) / @as(f64, std.math.maxInt(u32));
    }

    pub fn nextInRange(self: *TestRandom, min: i32, max: i32) i32 {
        const range = max - min + 1;
        return min + @as(i32, @intCast(self.nextU32() % @as(u32, @intCast(range))));
    }
};

// Property test runner
pub fn forAll(
    comptime T: type,
    generator: fn (*TestRandom) T,
    property: fn (T) bool,
    num_cases: u32,
    seed: u64,
) !void {
    var rng = TestRandom.init(seed);
    var i: u32 = 0;
    while (i < num_cases) : (i += 1) {
        const value = generator(&rng);
        if (!property(value)) {
            std.debug.print("Property failed for case {}: value = {any}\n", .{ i, value });
            return error.PropertyViolated;
        }
    }
}

// Property tests for math operations

// Property: a + b == b + a (commutativity)
fn addCommutative(args: struct { a: i32, b: i32 }) bool {
    return args.a +% args.b == args.b +% args.a;
}

fn generateIntPair(rng: *TestRandom) struct { a: i32, b: i32 } {
    return .{
        .a = rng.nextI32(),
        .b = rng.nextI32(),
    };
}

test "addition is commutative" {
    try forAll(
        struct { a: i32, b: i32 },
        generateIntPair,
        addCommutative,
        1000,
        42,
    );
}

// Property: sort produces ordered array
pub fn isSorted(arr: []const i32) bool {
    var i: usize = 1;
    while (i < arr.len) : (i += 1) {
        if (arr[i] < arr[i - 1]) return false;
    }
    return true;
}

fn bubbleSort(arr: []i32) void {
    var i: usize = 0;
    while (i < arr.len) : (i += 1) {
        var j: usize = 0;
        while (j < arr.len - i - 1) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                const tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}

test "bubbleSort produces sorted output" {
    const allocator = testing.allocator;
    var rng = TestRandom.init(12345);

    var i: u32 = 0;
    while (i < 100) : (i += 1) {
        const len = rng.nextU32() % 20;
        var arr = try allocator.alloc(i32, len);
        defer allocator.free(arr);

        for (arr) |*v| v.* = rng.nextI32();
        
        bubbleSort(arr);
        try testing.expect(isSorted(arr));
    }
}

// Property: reverse twice == identity
fn reverseSlice(arr: []i32) void {
    var left: usize = 0;
    var right = arr.len;
    while (left < right) {
        right -= 1;
        const tmp = arr[left];
        arr[left] = arr[right];
        arr[right] = tmp;
        left += 1;
    }
}

test "double reverse is identity" {
    const allocator = testing.allocator;
    var rng = TestRandom.init(54321);

    var trial: u32 = 0;
    while (trial < 100) : (trial += 1) {
        const len = rng.nextU32() % 15;
        var arr = try allocator.alloc(i32, len);
        defer allocator.free(arr);
        const original = try allocator.dupe(i32, arr);
        defer allocator.free(original);

        for (arr) |*v| v.* = rng.nextI32();
        @memcpy(original, arr);

        reverseSlice(arr);
        reverseSlice(arr);

        try testing.expectEqualSlices(i32, original, arr);
    }
}

// Shrinking: find minimal failing case
pub fn shrink(
    comptime T: type,
    failing_case: T,
    property: fn (T) bool,
    shrink_fn: fn (T, std.mem.Allocator) anyerror![]T,
    allocator: std.mem.Allocator,
) !T {
    var current = failing_case;
    
    // Keep trying to shrink
    var changed = true;
    while (changed) {
        changed = false;
        const candidates = shrink_fn(current, allocator) catch break;
        defer allocator.free(candidates);
        
        for (candidates) |candidate| {
            if (!property(candidate)) {
                current = candidate;
                changed = true;
                break;
            }
        }
    }
    
    return current;
}
```

---

## Step 945: Fuzz Testing

### Fuzz Testing ใน Zig

```zig
const std = @import("std");
const testing = std.testing;

// ใน Zig fuzz testing ทำด้วย `zig build fuzz`
// และฟังก์ชันที่ fuzz ต้องมี signature พิเศษ

// Simple fuzz harness simulation
pub const FuzzInput = struct {
    data: []const u8,
    allocator: std.mem.Allocator,

    pub fn init(data: []const u8, allocator: std.mem.Allocator) FuzzInput {
        return FuzzInput{ .data = data, .allocator = allocator };
    }

    pub fn readByte(self: *FuzzInput, offset: usize) ?u8 {
        if (offset >= self.data.len) return null;
        return self.data[offset];
    }

    pub fn readU32(self: *FuzzInput, offset: usize) ?u32 {
        if (offset + 4 > self.data.len) return null;
        return std.mem.readInt(u32, self.data[offset..][0..4], .little);
    }

    pub fn readSlice(self: *FuzzInput, offset: usize, len: usize) ?[]const u8 {
        if (offset + len > self.data.len) return null;
        return self.data[offset .. offset + len];
    }
};

// Function to fuzz: JSON parser
pub fn parseSimpleJson(input: []const u8, allocator: std.mem.Allocator) !void {
    // Simple JSON parsing - should never crash, only return errors
    if (input.len == 0) return error.EmptyInput;
    
    var i: usize = 0;
    while (i < input.len and (input[i] == ' ' or input[i] == '\t' or input[i] == '\n')) {
        i += 1;
    }
    
    if (i >= input.len) return error.EmptyInput;
    
    switch (input[i]) {
        '{' => {
            // Object parsing
            var depth: u32 = 1;
            i += 1;
            while (i < input.len and depth > 0) {
                if (input[i] == '{') depth += 1;
                if (input[i] == '}') depth -= 1;
                i += 1;
            }
            if (depth != 0) return error.UnclosedObject;
        },
        '[' => {
            // Array parsing
            var depth: u32 = 1;
            i += 1;
            while (i < input.len and depth > 0) {
                if (input[i] == '[') depth += 1;
                if (input[i] == ']') depth -= 1;
                i += 1;
            }
            if (depth != 0) return error.UnclosedArray;
        },
        '"' => {
            // String parsing
            i += 1;
            while (i < input.len and input[i] != '"') {
                if (input[i] == '\\') i += 1; // Skip escape
                i += 1;
            }
            if (i >= input.len) return error.UnclosedString;
        },
        '0'...'9', '-' => {
            // Number parsing
            while (i < input.len and (
                (input[i] >= '0' and input[i] <= '9') or
                input[i] == '.' or input[i] == 'e' or input[i] == 'E' or
                input[i] == '+' or input[i] == '-'
            )) {
                i += 1;
            }
        },
        't', 'f', 'n' => {
            // true, false, null
        },
        else => return error.InvalidJson,
    }
    
    _ = allocator;
}

// Fuzz-like test with random inputs
test "parseSimpleJson - random inputs should not crash" {
    const allocator = testing.allocator;
    var rng = std.rand.DefaultPrng.init(42);
    const random = rng.random();
    
    var i: u32 = 0;
    while (i < 1000) : (i += 1) {
        const len = random.uintLessThan(u32, 100);
        const input = try allocator.alloc(u8, len);
        defer allocator.free(input);
        
        for (input) |*b| {
            b.* = random.int(u8);
        }
        
        // Should not crash, only return errors
        parseSimpleJson(input, allocator) catch {};
    }
}

// Structured fuzzing
pub const FuzzConfig = struct {
    max_iterations: u32 = 1000,
    seed: u64 = 0,
    max_input_size: usize = 1024,
    save_failing_cases: bool = true,
};

pub fn fuzz(
    comptime target_fn: anytype,
    config: FuzzConfig,
    allocator: std.mem.Allocator,
) !void {
    var rng = std.rand.DefaultPrng.init(config.seed);
    const random = rng.random();
    var failures: u32 = 0;

    var i: u32 = 0;
    while (i < config.max_iterations) : (i += 1) {
        const size = random.uintLessThan(usize, config.max_input_size) + 1;
        const input = try allocator.alloc(u8, size);
        defer allocator.free(input);

        for (input) |*b| {
            b.* = random.int(u8);
        }

        target_fn(input, allocator) catch |e| {
            switch (e) {
                error.InvalidJson, error.EmptyInput,
                error.UnclosedObject, error.UnclosedArray,
                error.UnclosedString => {}, // Expected errors
                else => {
                    failures += 1;
                    std.debug.print("Unexpected error on iteration {}: {}\n", .{ i, e });
                    if (config.save_failing_cases) {
                        std.debug.print("Input: {x}\n", .{std.fmt.fmtSliceHexLower(input)});
                    }
                },
            }
        };
    }

    if (failures > 0) {
        std.debug.print("Fuzz found {} unexpected failures\n", .{failures});
        return error.FuzzFailures;
    }
    
    std.debug.print("Fuzz completed {} iterations with no unexpected failures\n", .{config.max_iterations});
}
```

---

## Step 946: Performance Testing

### Benchmark Testing

```zig
const std = @import("std");
const testing = std.testing;

// Simple benchmark framework
pub const Benchmark = struct {
    name: []const u8,
    allocator: std.mem.Allocator,
    iterations: u32,
    warmup_iterations: u32,

    pub fn init(name: []const u8, allocator: std.mem.Allocator) Benchmark {
        return Benchmark{
            .name = name,
            .allocator = allocator,
            .iterations = 1000,
            .warmup_iterations = 100,
        };
    }

    pub fn run(self: Benchmark, func: anytype, args: anytype) !BenchmarkResult {
        // Warmup
        var i: u32 = 0;
        while (i < self.warmup_iterations) : (i += 1) {
            _ = @call(.auto, func, args);
        }

        // Benchmark
        const start = std.time.nanoTimestamp();
        i = 0;
        while (i < self.iterations) : (i += 1) {
            _ = @call(.auto, func, args);
        }
        const end = std.time.nanoTimestamp();

        const total_ns = @as(u64, @intCast(end - start));
        const avg_ns = total_ns / self.iterations;

        return BenchmarkResult{
            .name = self.name,
            .iterations = self.iterations,
            .total_ns = total_ns,
            .avg_ns = avg_ns,
            .min_ns = avg_ns, // Simplified
            .max_ns = avg_ns,
        };
    }
};

pub const BenchmarkResult = struct {
    name: []const u8,
    iterations: u32,
    total_ns: u64,
    avg_ns: u64,
    min_ns: u64,
    max_ns: u64,

    pub fn print(self: BenchmarkResult) void {
        std.debug.print("Benchmark: {s}\n", .{self.name});
        std.debug.print("  Iterations: {}\n", .{self.iterations});
        std.debug.print("  Total: {}ns\n", .{self.total_ns});
        std.debug.print("  Average: {}ns ({d:.2}μs)\n", .{
            self.avg_ns,
            @as(f64, @floatFromInt(self.avg_ns)) / 1000.0,
        });
        std.debug.print("  Throughput: {d:.2} ops/sec\n", .{
            1e9 / @as(f64, @floatFromInt(self.avg_ns)),
        });
    }

    pub fn compareWith(self: BenchmarkResult, other: BenchmarkResult) void {
        const ratio = @as(f64, @floatFromInt(self.avg_ns)) /
            @as(f64, @floatFromInt(other.avg_ns));
        if (ratio < 1.0) {
            std.debug.print("{s} is {d:.2}x faster than {s}\n", .{
                self.name, 1.0 / ratio, other.name,
            });
        } else {
            std.debug.print("{s} is {d:.2}x slower than {s}\n", .{
                self.name, ratio, other.name,
            });
        }
    }
};

// Functions to benchmark
fn linearSearch(arr: []const i32, target: i32) ?usize {
    for (arr, 0..) |v, i| {
        if (v == target) return i;
    }
    return null;
}

fn binarySearch(arr: []const i32, target: i32) ?usize {
    var lo: usize = 0;
    var hi = arr.len;
    while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) lo = mid + 1 else hi = mid;
    }
    return null;
}

// Performance regression test
pub const PerformanceTest = struct {
    name: []const u8,
    max_avg_ns: u64,
    func: anytype,
    args: anytype,

    pub fn run(self: @This(), allocator: std.mem.Allocator) !void {
        const bench = Benchmark.init(self.name, allocator);
        const result = try bench.run(self.func, self.args);
        result.print();

        if (result.avg_ns > self.max_avg_ns) {
            std.debug.print("PERFORMANCE REGRESSION: {} > {} ns\n", .{
                result.avg_ns,
                self.max_avg_ns,
            });
            return error.PerformanceRegression;
        }
    }
};

// Memory usage benchmark
pub fn measureMemoryUsage(
    allocator: std.mem.Allocator,
    func: anytype,
    args: anytype,
) !usize {
    var counting = std.heap.GeneralPurposeAllocator(.{ .enable_memory_limit = true }){};
    defer _ = counting.deinit();
    _ = allocator;

    @call(.auto, func, .{ counting.allocator() } ++ args) catch {};
    return 0; // Simplified
}
```

---

## Step 947: Snapshot Testing

### Snapshot Testing Implementation

```zig
const std = @import("std");
const testing = std.testing;

// Snapshot store
pub const SnapshotManager = struct {
    allocator: std.mem.Allocator,
    snapshots: std.StringHashMap([]const u8),
    snapshot_file: []const u8,
    update_mode: bool,

    pub fn init(
        allocator: std.mem.Allocator,
        snapshot_file: []const u8,
        update_mode: bool,
    ) SnapshotManager {
        return SnapshotManager{
            .allocator = allocator,
            .snapshots = std.StringHashMap([]const u8).init(allocator),
            .snapshot_file = snapshot_file,
            .update_mode = update_mode,
        };
    }

    pub fn deinit(self: *SnapshotManager) void {
        var it = self.snapshots.valueIterator();
        while (it.next()) |v| self.allocator.free(v.*);
        self.snapshots.deinit();
    }

    pub fn load(self: *SnapshotManager) !void {
        const file = std.fs.cwd().openFile(self.snapshot_file, .{}) catch |e| {
            if (e == error.FileNotFound) return;
            return e;
        };
        defer file.close();

        const content = try file.readToEndAlloc(self.allocator, 10 * 1024 * 1024);
        defer self.allocator.free(content);

        // Parse simple key=value format
        var lines = std.mem.splitScalar(u8, content, '\n');
        var current_key: ?[]const u8 = null;
        var current_value = std.ArrayList(u8).init(self.allocator);
        defer current_value.deinit();

        while (lines.next()) |line| {
            if (std.mem.startsWith(u8, line, "---")) {
                if (current_key) |key| {
                    const value = try self.allocator.dupe(u8, current_value.items);
                    try self.snapshots.put(key, value);
                    current_value.clearRetainingCapacity();
                }
                current_key = line[3..];
            } else if (current_key != null) {
                try current_value.appendSlice(line);
                try current_value.append('\n');
            }
        }
    }

    pub fn save(self: *SnapshotManager) !void {
        const file = try std.fs.cwd().createFile(self.snapshot_file, .{});
        defer file.close();
        const w = file.writer();

        var it = self.snapshots.iterator();
        while (it.next()) |entry| {
            try w.print("---{s}\n", .{entry.key_ptr.*});
            try w.writeAll(entry.value_ptr.*);
        }
    }

    pub fn match(self: *SnapshotManager, name: []const u8, value: []const u8) !void {
        if (self.update_mode) {
            // Update snapshot
            if (self.snapshots.getPtr(name)) |existing| {
                self.allocator.free(existing.*);
                existing.* = try self.allocator.dupe(u8, value);
            } else {
                const v = try self.allocator.dupe(u8, value);
                try self.snapshots.put(name, v);
            }
            return;
        }

        if (self.snapshots.get(name)) |snapshot| {
            if (!std.mem.eql(u8, snapshot, value)) {
                std.debug.print("\nSnapshot mismatch for '{s}':\n", .{name});
                std.debug.print("Expected:\n{s}\n", .{snapshot});
                std.debug.print("Got:\n{s}\n", .{value});
                return error.SnapshotMismatch;
            }
        } else {
            // First run - create snapshot
            const v = try self.allocator.dupe(u8, value);
            try self.snapshots.put(name, v);
            std.debug.print("Created snapshot: {s}\n", .{name});
        }
    }
};

// Format helpers for snapshots
pub fn formatForSnapshot(value: anytype, allocator: std.mem.Allocator) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    try std.fmt.format(buf.writer(), "{any}", .{value});
    return buf.toOwnedSlice();
}

// HTML snapshot example
pub fn renderHtml(
    title: []const u8,
    items: []const []const u8,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("<html><head><title>{s}</title></head><body>\n", .{title});
    try w.writeAll("<ul>\n");
    for (items) |item| {
        try w.print("  <li>{s}</li>\n", .{item});
    }
    try w.writeAll("</ul>\n</body></html>\n");

    return buf.toOwnedSlice();
}

test "renderHtml - snapshot" {
    const allocator = testing.allocator;
    
    const items = [_][]const u8{ "Apple", "Banana", "Cherry" };
    const html = try renderHtml("Fruits", &items, allocator);
    defer allocator.free(html);

    // In real test, compare with snapshot file
    const expected =
        "<html><head><title>Fruits</title></head><body>\n" ++
        "<ul>\n" ++
        "  <li>Apple</li>\n" ++
        "  <li>Banana</li>\n" ++
        "  <li>Cherry</li>\n" ++
        "</ul>\n</body></html>\n";

    try testing.expectEqualStrings(expected, html);
}
```

---

## Step 948: Mutation Testing

### Mutation Testing Concepts

```zig
const std = @import("std");
const testing = std.testing;

// Mutation testing: test if our tests can catch bugs
// ทำโดยการใส่ bugs เข้าไปในโค้ด แล้วดูว่า tests ตรวจจับได้

// Original function (correct)
pub fn max(a: i32, b: i32) i32 {
    return if (a > b) a else b;
}

// Mutants (buggy versions)
pub fn max_mutant_1(a: i32, b: i32) i32 {
    return if (a >= b) a else b; // Mutation: > changed to >=
}

pub fn max_mutant_2(a: i32, b: i32) i32 {
    return if (a < b) a else b; // Mutation: > changed to <
}

pub fn max_mutant_3(a: i32, b: i32) i32 {
    return if (a > b) b else a; // Mutation: swapped return values
}

pub fn max_mutant_4(a: i32, b: i32) i32 {
    return a; // Mutation: always return a
}

// Test suite that should kill all mutants
test "max - kills all mutants" {
    // Test 1: a > b
    try testing.expectEqual(@as(i32, 5), max(5, 3));
    // Kills mutant_2 (would return 3), mutant_4 (returns a=5 correct here but...)

    // Test 2: b > a
    try testing.expectEqual(@as(i32, 5), max(3, 5));
    // Kills mutant_4 (would return 3)

    // Test 3: equal values
    try testing.expectEqual(@as(i32, 5), max(5, 5));
    // Kills mutant_3 (would still work here)

    // Test 4: negative numbers
    try testing.expectEqual(@as(i32, -1), max(-1, -5));
    // Additional coverage

    // Test 5: zero vs positive
    try testing.expectEqual(@as(i32, 0), max(0, -1));
}

// Mutation score calculator
pub const MutationResult = struct {
    total_mutants: u32,
    killed_mutants: u32,

    pub fn score(self: MutationResult) f64 {
        if (self.total_mutants == 0) return 1.0;
        return @as(f64, @floatFromInt(self.killed_mutants)) /
            @as(f64, @floatFromInt(self.total_mutants));
    }

    pub fn print(self: MutationResult) void {
        std.debug.print("Mutation Score: {d:.1}% ({}/{} mutants killed)\n", .{
            self.score() * 100,
            self.killed_mutants,
            self.total_mutants,
        });
    }
};

// Automatically run mutation tests
pub fn runMutationTests(
    comptime original: anytype,
    comptime mutants: []const @TypeOf(original),
    test_fn: fn (@TypeOf(original)) bool,
) MutationResult {
    var killed: u32 = 0;

    // Check original passes
    if (!test_fn(original)) {
        std.debug.print("WARNING: Original function fails tests!\n", .{});
    }

    // Check each mutant is killed
    inline for (mutants, 0..) |mutant, i| {
        if (!test_fn(mutant)) {
            killed += 1;
        } else {
            std.debug.print("Surviving mutant {}: test suite did not kill it!\n", .{i});
        }
    }

    return MutationResult{
        .total_mutants = mutants.len,
        .killed_mutants = killed,
    };
}
```

---

## Step 949: Code Coverage

### Code Coverage Analysis

```zig
const std = @import("std");

// Coverage tracking (simulation)
pub const CoverageTracker = struct {
    covered_lines: std.AutoHashMap(u32, bool),
    total_lines: u32,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, total: u32) CoverageTracker {
        return CoverageTracker{
            .covered_lines = std.AutoHashMap(u32, bool).init(allocator),
            .total_lines = total,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *CoverageTracker) void {
        self.covered_lines.deinit();
    }

    pub fn mark(self: *CoverageTracker, line: u32) !void {
        try self.covered_lines.put(line, true);
    }

    pub fn coverage(self: CoverageTracker) f64 {
        if (self.total_lines == 0) return 1.0;
        return @as(f64, @floatFromInt(self.covered_lines.count())) /
            @as(f64, @floatFromInt(self.total_lines));
    }

    pub fn report(self: CoverageTracker) void {
        std.debug.print("Coverage: {d:.1}% ({}/{} lines)\n", .{
            self.coverage() * 100,
            self.covered_lines.count(),
            self.total_lines,
        });

        // Find uncovered lines
        var line: u32 = 1;
        while (line <= self.total_lines) : (line += 1) {
            if (!self.covered_lines.contains(line)) {
                std.debug.print("  Uncovered line: {}\n", .{line});
            }
        }
    }
};

// Function with branches to test
pub fn categorize(n: i32) []const u8 {
    if (n < 0) {  // line 1
        return "negative";  // line 2
    } else if (n == 0) {  // line 3
        return "zero";  // line 4
    } else if (n < 10) {  // line 5
        return "small";  // line 6
    } else if (n < 100) {  // line 7
        return "medium";  // line 8
    } else {
        return "large";  // line 9
    }
}

// Branch coverage test
const std_testing = std.testing;

test "categorize - full branch coverage" {
    // Cover negative branch (lines 1-2)
    try std_testing.expectEqualStrings("negative", categorize(-5));
    
    // Cover zero branch (lines 3-4)
    try std_testing.expectEqualStrings("zero", categorize(0));
    
    // Cover small branch (lines 5-6)
    try std_testing.expectEqualStrings("small", categorize(5));
    
    // Cover medium branch (lines 7-8)
    try std_testing.expectEqualStrings("medium", categorize(50));
    
    // Cover large branch (line 9)
    try std_testing.expectEqualStrings("large", categorize(200));
}

// Line coverage vs branch coverage vs path coverage
pub fn complexFunction(a: bool, b: bool, c: bool) u32 {
    var result: u32 = 0;
    if (a) result += 1;
    if (b) result += 2;
    if (c) result += 4;
    return result;
}

test "complexFunction - line coverage (minimal)" {
    // Just one test achieves 100% line coverage
    _ = complexFunction(true, true, true);
}

test "complexFunction - branch coverage" {
    // Need to test both branches of each if
    _ = complexFunction(true, true, true);
    _ = complexFunction(false, false, false);
}

test "complexFunction - path coverage (complete)" {
    // Need all 8 combinations for full path coverage
    _ = complexFunction(false, false, false);
    _ = complexFunction(false, false, true);
    _ = complexFunction(false, true, false);
    _ = complexFunction(false, true, true);
    _ = complexFunction(true, false, false);
    _ = complexFunction(true, false, true);
    _ = complexFunction(true, true, false);
    _ = complexFunction(true, true, true);
}
```

---

## Step 950: CI/CD Integration สำหรับ Tests

### Continuous Integration สำหรับ Zig Projects

```zig
// test_runner.zig - Custom test runner สำหรับ CI
const std = @import("std");
const builtin = @import("builtin");

// Test result format สำหรับ CI systems
pub const TestResult = struct {
    name: []const u8,
    passed: bool,
    duration_ns: u64,
    error_message: ?[]const u8,
};

pub const TestReport = struct {
    suite_name: []const u8,
    results: std.ArrayList(TestResult),
    start_time: i64,
    end_time: i64,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, suite: []const u8) TestReport {
        return TestReport{
            .suite_name = suite,
            .results = std.ArrayList(TestResult).init(allocator),
            .start_time = std.time.timestamp(),
            .end_time = 0,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *TestReport) void {
        self.results.deinit();
    }

    pub fn finish(self: *TestReport) void {
        self.end_time = std.time.timestamp();
    }

    pub fn passCount(self: TestReport) u32 {
        var count: u32 = 0;
        for (self.results.items) |r| {
            if (r.passed) count += 1;
        }
        return count;
    }

    pub fn failCount(self: TestReport) u32 {
        return @intCast(self.results.items.len - self.passCount());
    }

    // Output JUnit XML for CI systems (GitHub Actions, Jenkins, etc.)
    pub fn toJunitXml(self: TestReport, allocator: std.mem.Allocator) ![]const u8 {
        var buf = std.ArrayList(u8).init(allocator);
        errdefer buf.deinit();
        const w = buf.writer();

        try w.writeAll("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
        try w.print("<testsuite name=\"{s}\" tests=\"{}\" failures=\"{}\" time=\"{}\">\n", .{
            self.suite_name,
            self.results.items.len,
            self.failCount(),
            self.end_time - self.start_time,
        });

        for (self.results.items) |result| {
            try w.print("  <testcase name=\"{s}\" time=\"{d:.6}\"", .{
                result.name,
                @as(f64, @floatFromInt(result.duration_ns)) / 1e9,
            });

            if (result.passed) {
                try w.writeAll("/>\n");
            } else {
                try w.writeAll(">\n");
                if (result.error_message) |msg| {
                    try w.print("    <failure message=\"{s}\"/>\n", .{msg});
                } else {
                    try w.writeAll("    <failure/>\n");
                }
                try w.writeAll("  </testcase>\n");
            }
        }

        try w.writeAll("</testsuite>\n");
        return buf.toOwnedSlice();
    }

    // Output GitHub Actions annotations
    pub fn toGithubAnnotations(self: TestReport, allocator: std.mem.Allocator) ![]const u8 {
        var buf = std.ArrayList(u8).init(allocator);
        errdefer buf.deinit();
        const w = buf.writer();

        for (self.results.items) |result| {
            if (!result.passed) {
                if (result.error_message) |msg| {
                    try w.print("::error title=Test Failed::{s} - {s}\n", .{
                        result.name,
                        msg,
                    });
                } else {
                    try w.print("::error title=Test Failed::{s}\n", .{result.name});
                }
            }
        }

        return buf.toOwnedSlice();
    }

    pub fn printSummary(self: TestReport) void {
        const total = self.results.items.len;
        const passed = self.passCount();
        const failed = self.failCount();

        std.debug.print("\n=== Test Summary ===\n", .{});
        std.debug.print("Suite: {s}\n", .{self.suite_name});
        std.debug.print("Total: {}\n", .{total});
        std.debug.print("Passed: {} ✓\n", .{passed});
        std.debug.print("Failed: {} ✗\n", .{failed});

        if (failed > 0) {
            std.debug.print("\nFailed tests:\n", .{});
            for (self.results.items) |r| {
                if (!r.passed) {
                    std.debug.print("  - {s}", .{r.name});
                    if (r.error_message) |msg| {
                        std.debug.print(": {s}", .{msg});
                    }
                    std.debug.print("\n", .{});
                }
            }
        }
    }
};

// CI configuration
pub const CiConfig = struct {
    fail_fast: bool = false,
    timeout_seconds: u32 = 300,
    parallel: bool = false,
    coverage_threshold: f64 = 0.8,
    report_format: ReportFormat = .junit,

    pub const ReportFormat = enum { junit, github, plain };
};

// Complete test pipeline
pub const TestPipeline = struct {
    allocator: std.mem.Allocator,
    config: CiConfig,
    report: TestReport,

    pub fn init(allocator: std.mem.Allocator, suite_name: []const u8, config: CiConfig) TestPipeline {
        return TestPipeline{
            .allocator = allocator,
            .config = config,
            .report = TestReport.init(allocator, suite_name),
        };
    }

    pub fn deinit(self: *TestPipeline) void {
        self.report.deinit();
    }

    pub fn runTest(self: *TestPipeline, name: []const u8, test_fn: anytype) !void {
        const start = std.time.nanoTimestamp();
        var passed = true;
        var error_msg: ?[]const u8 = null;

        test_fn() catch |e| {
            passed = false;
            error_msg = try std.fmt.allocPrint(self.allocator, "{}", .{e});
        };

        const duration = @as(u64, @intCast(std.time.nanoTimestamp() - start));

        try self.report.results.append(TestResult{
            .name = name,
            .passed = passed,
            .duration_ns = duration,
            .error_message = error_msg,
        });

        if (!passed and self.config.fail_fast) {
            return error.TestFailed;
        }
    }

    pub fn finish(self: *TestPipeline) !void {
        self.report.finish();
        self.report.printSummary();

        // Generate report based on format
        switch (self.config.report_format) {
            .junit => {
                const xml = try self.report.toJunitXml(self.allocator);
                defer self.allocator.free(xml);
                
                const f = try std.fs.cwd().createFile("test-results.xml", .{});
                defer f.close();
                try f.writeAll(xml);
                std.debug.print("JUnit report saved to test-results.xml\n", .{});
            },
            .github => {
                const annotations = try self.report.toGithubAnnotations(self.allocator);
                defer self.allocator.free(annotations);
                std.debug.print("{s}", .{annotations});
            },
            .plain => {
                // Already printed summary
            },
        }

        if (self.report.failCount() > 0) {
            return error.TestsFailed;
        }
    }
};

// Pre-configured test configurations
pub const CiPresets = struct {
    pub const github_actions = CiConfig{
        .fail_fast = false,
        .timeout_seconds = 300,
        .parallel = true,
        .coverage_threshold = 0.8,
        .report_format = .github,
    };

    pub const local = CiConfig{
        .fail_fast = true,
        .timeout_seconds = 60,
        .parallel = false,
        .coverage_threshold = 0.7,
        .report_format = .plain,
    };

    pub const strict = CiConfig{
        .fail_fast = true,
        .timeout_seconds = 600,
        .parallel = true,
        .coverage_threshold = 0.95,
        .report_format = .junit,
    };
};

// Example of complete test setup
pub fn runCiTests() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var pipeline = TestPipeline.init(
        allocator,
        "MyProject",
        CiPresets.local,
    );
    defer pipeline.deinit();

    try pipeline.runTest("test_add_positive", struct {
        fn f() !void {
            const result: i32 = 3 + 4;
            if (result != 7) return error.Unexpected;
        }
    }.f);

    try pipeline.runTest("test_add_negative", struct {
        fn f() !void {
            const result: i32 = -3 + (-4);
            if (result != -7) return error.Unexpected;
        }
    }.f);

    try pipeline.runTest("test_divide_by_zero", struct {
        fn f() !void {
            // This test verifies error handling
            const divisor: f64 = 0;
            if (divisor == 0) return; // Expected
            _ = 10.0 / divisor;
        }
    }.f);

    pipeline.finish() catch |e| {
        std.debug.print("Tests failed: {}\n", .{e});
        return e;
    };
}

// Testing utilities
pub const TestHelpers = struct {
    // Assert within tolerance for floats
    pub fn expectNearlyEqual(expected: f64, actual: f64, tolerance: f64) !void {
        const diff = @abs(expected - actual);
        if (diff > tolerance) {
            std.debug.print("Expected {} ± {}, got {}\n", .{ expected, tolerance, actual });
            return error.NotNearlyEqual;
        }
    }

    // Assert slice contains element
    pub fn expectContains(comptime T: type, slice: []const T, element: T) !void {
        for (slice) |item| {
            if (std.meta.eql(item, element)) return;
        }
        return error.ElementNotFound;
    }

    // Assert string matches regex-like pattern
    pub fn expectMatchesPattern(str: []const u8, pattern: []const u8) !void {
        if (!std.mem.containsAtLeast(u8, str, 1, pattern)) {
            std.debug.print("String '{s}' does not contain pattern '{s}'\n", .{ str, pattern });
            return error.PatternNotFound;
        }
    }

    // Assert duration is within bounds
    pub fn expectDuration(
        func: anytype,
        args: anytype,
        max_ns: u64,
    ) !void {
        const start = std.time.nanoTimestamp();
        _ = @call(.auto, func, args);
        const elapsed = @as(u64, @intCast(std.time.nanoTimestamp() - start));
        
        if (elapsed > max_ns) {
            std.debug.print("Expected < {}ns, took {}ns\n", .{ max_ns, elapsed });
            return error.TooSlow;
        }
    }

    // Verify no memory leaks
    pub fn expectNoLeaks(func: anytype, args: anytype) !void {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        @call(.auto, func, .{gpa.allocator()} ++ args) catch {};
        const leaked = gpa.deinit();
        if (leaked == .leak) return error.MemoryLeak;
    }
};

// GitHub Actions workflow generation
pub fn generateGithubActionsWorkflow(
    project_name: []const u8,
    zig_version: []const u8,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("name: Test {s}\n\n", .{project_name});
    try w.writeAll("on:\n  push:\n    branches: [main]\n  pull_request:\n    branches: [main]\n\n");
    try w.writeAll("jobs:\n  test:\n    runs-on: ubuntu-latest\n    steps:\n");
    try w.writeAll("      - uses: actions/checkout@v4\n");
    try w.writeAll("      - name: Setup Zig\n");
    try w.writeAll("        uses: goto-bus-stop/setup-zig@v2\n");
    try w.writeAll("        with:\n");
    try w.print("          version: {s}\n", .{zig_version});
    try w.writeAll("      - name: Run tests\n");
    try w.writeAll("        run: zig build test\n");
    try w.writeAll("      - name: Upload test results\n");
    try w.writeAll("        if: always()\n");
    try w.writeAll("        uses: actions/upload-artifact@v3\n");
    try w.writeAll("        with:\n");
    try w.writeAll("          name: test-results\n");
    try w.writeAll("          path: test-results.xml\n");

    return buf.toOwnedSlice();
}

pub fn main() !void {
    try runCiTests();

    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Generate GitHub Actions workflow
    const workflow = try generateGithubActionsWorkflow("MyZigProject", "0.13.0", allocator);
    defer allocator.free(workflow);
    std.debug.print("\nGitHub Actions Workflow:\n{s}\n", .{workflow});
}
```

---

## สรุป Part 95

ในส่วนนี้เราได้เรียนรู้:

1. **Testing Philosophy** - หลักการและ mindset ของการทดสอบที่ดี
2. **Unit Testing** - AAA pattern, parameterized tests, fixtures
3. **Integration Testing** - mock objects, dependency injection
4. **Property-Based Testing** - forAll, generators, shrinking
5. **Fuzz Testing** - random input testing, coverage-guided fuzzing
6. **Performance Testing** - benchmarks, regression detection
7. **Snapshot Testing** - comparing output over time
8. **Mutation Testing** - verifying test effectiveness
9. **Code Coverage** - line, branch, path coverage
10. **CI/CD Integration** - JUnit XML, GitHub Actions, pipelines

### Testing Best Practices สำหรับ Zig

```
1. เขียน tests ไปพร้อมกับ code (TDD หรือ BDD)
2. Tests ต้อง fast, isolated, repeatable
3. Test behavior ไม่ใช่ implementation
4. เขียน test cases ที่อ่านเข้าใจง่าย
5. ใช้ testing.allocator เพื่อตรวจจับ memory leaks
6. Integration tests ใช้ interfaces/mocks
7. ใช้ property tests สำหรับ math operations
8. Track code coverage ใน CI
9. บันทึก performance baselines
10. ใช้ fuzz testing สำหรับ parsers/protocols
```

### ตัวอย่าง Complete Test Setup

```zig
// ใน build.zig
const test_step = b.step("test", "Run all tests");
const unit_tests = b.addTest(.{ .root_source_file = b.path("src/main.zig") });
const run_tests = b.addRunArtifact(unit_tests);
test_step.dependOn(&run_tests.step);
```
