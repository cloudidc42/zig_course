# Part 39: std.testing - Unit Testing ใน Zig

## บทนำ

Zig มีระบบ testing แบบ built-in ที่ทรงพลัง ไม่ต้องติดตั้ง library เพิ่มเติม `std.testing` มี utilities ครบครันสำหรับการเขียน unit tests ที่ดี บทนี้จะครอบคลุมทุกแง่มุมของการทดสอบใน Zig

---

## Step 381: Unit Testing ใน Zig

### ทำไมต้องเขียน Test?

การเขียน test ช่วยให้:
- มั่นใจว่าโค้ดทำงานถูกต้อง
- ตรวจพบ regression เมื่อแก้ไขโค้ด
- เป็น documentation ที่ verify ได้
- ทำให้ refactor ได้มั่นใจกว่า

### Test Block พื้นฐาน

```zig
// math.zig
const std = @import("std");

pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

pub fn divide(a: f64, b: f64) !f64 {
    if (b == 0.0) return error.DivisionByZero;
    return a / b;
}

test "add basic" {
    try std.testing.expectEqual(@as(i32, 5), add(2, 3));
    try std.testing.expectEqual(@as(i32, 0), add(0, 0));
    try std.testing.expectEqual(@as(i32, -1), add(-3, 2));
}

test "multiply" {
    try std.testing.expectEqual(@as(i32, 6), multiply(2, 3));
    try std.testing.expectEqual(@as(i32, 0), multiply(5, 0));
    try std.testing.expectEqual(@as(i32, -12), multiply(-3, 4));
}

test "divide" {
    const result = try divide(10.0, 4.0);
    try std.testing.expectApproxEqAbs(@as(f64, 2.5), result, 0.0001);
}

test "divide by zero" {
    try std.testing.expectError(error.DivisionByZero, divide(1.0, 0.0));
}
```

### การ Run Tests

```bash
# Run tests ในไฟล์เดียว
zig test math.zig

# Run tests ทั้งโปรเจกต์
zig build test

# Run tests พร้อม verbose output
zig test math.zig --test-filter "add"

# Run tests แบบ release (optimized)
zig test math.zig -O ReleaseSafe
```

### Test Output Format

```
All 4 tests passed.

# ถ้ามี failure:
FAIL math.zig:test "add basic"
  expected: 5
  found: 6
  at math.zig:16
```

---

## Step 382: test blocks

### Test Block ขั้นสูง

```zig
const std = @import("std");

// โครงสร้างที่จะ test
const Stack = struct {
    items: std.ArrayList(i32),

    pub fn init(allocator: std.mem.Allocator) Stack {
        return .{ .items = std.ArrayList(i32).init(allocator) };
    }

    pub fn deinit(self: *Stack) void {
        self.items.deinit();
    }

    pub fn push(self: *Stack, val: i32) !void {
        try self.items.append(val);
    }

    pub fn pop(self: *Stack) ?i32 {
        return self.items.popOrNull();
    }

    pub fn peek(self: *const Stack) ?i32 {
        if (self.items.items.len == 0) return null;
        return self.items.items[self.items.items.len - 1];
    }

    pub fn isEmpty(self: *const Stack) bool {
        return self.items.items.len == 0;
    }

    pub fn size(self: *const Stack) usize {
        return self.items.items.len;
    }
};

test "Stack - empty initially" {
    var stack = Stack.init(std.testing.allocator);
    defer stack.deinit();

    try std.testing.expect(stack.isEmpty());
    try std.testing.expectEqual(@as(usize, 0), stack.size());
    try std.testing.expectEqual(@as(?i32, null), stack.peek());
    try std.testing.expectEqual(@as(?i32, null), stack.pop());
}

test "Stack - push and pop" {
    var stack = Stack.init(std.testing.allocator);
    defer stack.deinit();

    try stack.push(1);
    try stack.push(2);
    try stack.push(3);

    try std.testing.expectEqual(@as(usize, 3), stack.size());
    try std.testing.expectEqual(@as(?i32, 3), stack.peek());

    try std.testing.expectEqual(@as(?i32, 3), stack.pop());
    try std.testing.expectEqual(@as(?i32, 2), stack.pop());
    try std.testing.expectEqual(@as(?i32, 1), stack.pop());
    try std.testing.expect(stack.isEmpty());
}

test "Stack - LIFO order" {
    var stack = Stack.init(std.testing.allocator);
    defer stack.deinit();

    const values = [_]i32{ 10, 20, 30, 40, 50 };
    for (values) |v| {
        try stack.push(v);
    }

    // pop ต้องออกในลำดับย้อนกลับ
    var i: usize = values.len;
    while (i > 0) {
        i -= 1;
        const popped = stack.pop();
        try std.testing.expectEqual(@as(?i32, values[i]), popped);
    }
}
```

### Test ด้วย Setup/Teardown Pattern

```zig
const std = @import("std");

const Database = struct {
    data: std.StringHashMap([]const u8),

    pub fn init(allocator: std.mem.Allocator) Database {
        return .{ .data = std.StringHashMap([]const u8).init(allocator) };
    }

    pub fn deinit(self: *Database) void {
        self.data.deinit();
    }

    pub fn set(self: *Database, key: []const u8, value: []const u8) !void {
        try self.data.put(key, value);
    }

    pub fn get(self: *const Database, key: []const u8) ?[]const u8 {
        return self.data.get(key);
    }

    pub fn delete(self: *Database, key: []const u8) bool {
        return self.data.remove(key);
    }
};

fn setupDatabase(allocator: std.mem.Allocator) !Database {
    var db = Database.init(allocator);
    try db.set("name", "Alice");
    try db.set("email", "alice@example.com");
    try db.set("role", "admin");
    return db;
}

test "Database - get existing key" {
    var db = try setupDatabase(std.testing.allocator);
    defer db.deinit();

    const name = db.get("name");
    try std.testing.expectEqualStrings("Alice", name orelse "not found");
}

test "Database - get non-existing key" {
    var db = try setupDatabase(std.testing.allocator);
    defer db.deinit();

    try std.testing.expectEqual(@as(?[]const u8, null), db.get("nonexistent"));
}

test "Database - delete key" {
    var db = try setupDatabase(std.testing.allocator);
    defer db.deinit();

    try std.testing.expect(db.delete("email"));
    try std.testing.expectEqual(@as(?[]const u8, null), db.get("email"));
    try std.testing.expect(!db.delete("email")); // ลบซ้ำต้อง return false
}
```

---

## Step 383: std.testing.expect, expectEqual

### expect Functions ทั้งหมด

```zig
const std = @import("std");

test "expect - basic boolean check" {
    try std.testing.expect(true);
    try std.testing.expect(1 == 1);
    try std.testing.expect("hello".len == 5);
}

test "expectEqual - exact equality" {
    try std.testing.expectEqual(@as(i32, 42), @as(i32, 42));
    try std.testing.expectEqual(@as(u8, 255), @as(u8, 255));
    try std.testing.expectEqual(true, true);
    try std.testing.expectEqual(@as(?i32, null), @as(?i32, null));
}

test "expectEqualSlices" {
    const a = [_]i32{ 1, 2, 3, 4, 5 };
    const b = [_]i32{ 1, 2, 3, 4, 5 };
    try std.testing.expectEqualSlices(i32, &a, &b);
}

test "expectEqualStrings" {
    const s1 = "hello world";
    const s2 = "hello world";
    try std.testing.expectEqualStrings(s1, s2);
}

test "expectEqualSentinelSlices" {
    const s1: [*:0]const u8 = "hello";
    const s2: [*:0]const u8 = "hello";
    try std.testing.expectEqualSentinelSlices(u8, 0, s1, s2);
}

test "expectApproxEqAbs - float comparison" {
    const pi = 3.14159265;
    const approx_pi = 3.14159;
    try std.testing.expectApproxEqAbs(pi, approx_pi, 0.001);
}

test "expectApproxEqRel - relative float comparison" {
    const large_num: f64 = 1_000_000.0;
    const approx: f64 = 999_999.9;
    try std.testing.expectApproxEqRel(large_num, approx, 0.0001);
}
```

### ตัวอย่างที่แสดง error messages

```zig
const std = @import("std");

// ตัวอย่างที่ fail เพื่อดู error message
// (comment out เพื่อ run ปกติ)

// test "failure example - expectEqual" {
//     try std.testing.expectEqual(@as(i32, 5), @as(i32, 6));
// }
// Output: expected 5, found 6

// test "failure example - expectEqualStrings" {
//     try std.testing.expectEqualStrings("expected", "actual");
// }
// Output: expected "expected", found "actual"

// test "failure example - expect" {
//     try std.testing.expect(false);
// }
// Output: expectation failed

// ทดสอบ custom error message
test "custom assertion" {
    const value = 42;
    if (value != 42) {
        std.debug.print("Expected 42, got {}\n", .{value});
        return error.TestFailed;
    }
    try std.testing.expect(true);
}
```

---

## Step 384: std.testing.expectError

### การ Test Error Cases

```zig
const std = @import("std");

const ParseError = error{
    InvalidFormat,
    Overflow,
    EmptyInput,
};

pub fn parsePositiveInt(s: []const u8) ParseError!u32 {
    if (s.len == 0) return ParseError.EmptyInput;

    var result: u32 = 0;
    for (s) |c| {
        if (c < '0' or c > '9') return ParseError.InvalidFormat;
        const digit = c - '0';
        result = std.math.mul(u32, result, 10) catch return ParseError.Overflow;
        result = std.math.add(u32, result, digit) catch return ParseError.Overflow;
    }
    return result;
}

test "parsePositiveInt - valid numbers" {
    try std.testing.expectEqual(@as(u32, 0), try parsePositiveInt("0"));
    try std.testing.expectEqual(@as(u32, 42), try parsePositiveInt("42"));
    try std.testing.expectEqual(@as(u32, 4294967295), try parsePositiveInt("4294967295"));
}

test "parsePositiveInt - empty string" {
    try std.testing.expectError(ParseError.EmptyInput, parsePositiveInt(""));
}

test "parsePositiveInt - invalid format" {
    try std.testing.expectError(ParseError.InvalidFormat, parsePositiveInt("12a3"));
    try std.testing.expectError(ParseError.InvalidFormat, parsePositiveInt("-5"));
    try std.testing.expectError(ParseError.InvalidFormat, parsePositiveInt("3.14"));
}

test "parsePositiveInt - overflow" {
    try std.testing.expectError(ParseError.Overflow, parsePositiveInt("99999999999"));
}
```

### Error Union Testing

```zig
const std = @import("std");

const FileError = error{
    NotFound,
    PermissionDenied,
    TooLarge,
};

pub fn readFile(path: []const u8, max_size: usize) FileError![]const u8 {
    _ = max_size;
    if (std.mem.eql(u8, path, "/nonexistent")) return FileError.NotFound;
    if (std.mem.eql(u8, path, "/private")) return FileError.PermissionDenied;
    if (std.mem.eql(u8, path, "/huge_file")) return FileError.TooLarge;
    return "file content";
}

test "readFile - success" {
    const content = try readFile("/normal", 1024);
    try std.testing.expectEqualStrings("file content", content);
}

test "readFile - not found" {
    const result = readFile("/nonexistent", 1024);
    try std.testing.expectError(FileError.NotFound, result);
}

test "readFile - permission denied" {
    try std.testing.expectError(FileError.PermissionDenied, readFile("/private", 1024));
}

test "readFile - too large" {
    try std.testing.expectError(FileError.TooLarge, readFile("/huge_file", 100));
}

// Test error wrapping/unwrapping
test "error type checking" {
    const result = readFile("/nonexistent", 1024);

    // ตรวจสอบว่า error เป็น type ที่ถูกต้อง
    if (result) |_| {
        return error.ExpectedError;
    } else |err| {
        try std.testing.expect(err == FileError.NotFound);
    }
}
```

---

## Step 385: std.testing.allocator

### ใช้ Testing Allocator

```zig
const std = @import("std");

// std.testing.allocator จะตรวจสอบ memory leaks อัตโนมัติ

pub fn createList(allocator: std.mem.Allocator) !std.ArrayList(i32) {
    var list = std.ArrayList(i32).init(allocator);
    try list.append(1);
    try list.append(2);
    try list.append(3);
    return list;
}

test "createList - no memory leak" {
    // testing.allocator จะ fail ถ้ามี memory leak
    var list = try createList(std.testing.allocator);
    defer list.deinit(); // ถ้าไม่ deinit จะมี leak

    try std.testing.expectEqual(@as(usize, 3), list.items.len);
}

// ตัวอย่าง: function ที่ leak memory (จะ fail ถ้า test)
pub fn leakingFunction(allocator: std.mem.Allocator) ![]i32 {
    const slice = try allocator.alloc(i32, 10);
    // ลืม free!
    return slice;
}

// test "memory leak detection" {
//     _ = try leakingFunction(std.testing.allocator);
//     // Test จะ fail เพราะ memory leak!
// }
```

### Testing Complex Memory Patterns

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,

    pub fn create(allocator: std.mem.Allocator, value: i32) !*Node {
        const node = try allocator.create(Node);
        node.* = .{ .value = value, .next = null };
        return node;
    }
};

const LinkedList = struct {
    head: ?*Node,
    allocator: std.mem.Allocator,
    len: usize,

    pub fn init(allocator: std.mem.Allocator) LinkedList {
        return .{ .head = null, .allocator = allocator, .len = 0 };
    }

    pub fn deinit(self: *LinkedList) void {
        var current = self.head;
        while (current) |node| {
            const next = node.next;
            self.allocator.destroy(node);
            current = next;
        }
    }

    pub fn prepend(self: *LinkedList, value: i32) !void {
        const node = try Node.create(self.allocator, value);
        node.next = self.head;
        self.head = node;
        self.len += 1;
    }

    pub fn toSlice(self: *const LinkedList, allocator: std.mem.Allocator) ![]i32 {
        var result = try allocator.alloc(i32, self.len);
        var i: usize = 0;
        var current = self.head;
        while (current) |node| {
            result[i] = node.value;
            i += 1;
            current = node.next;
        }
        return result;
    }
};

test "LinkedList - prepend and iterate" {
    const allocator = std.testing.allocator;

    var list = LinkedList.init(allocator);
    defer list.deinit();

    try list.prepend(3);
    try list.prepend(2);
    try list.prepend(1);

    try std.testing.expectEqual(@as(usize, 3), list.len);

    const slice = try list.toSlice(allocator);
    defer allocator.free(slice);

    try std.testing.expectEqualSlices(i32, &[_]i32{ 1, 2, 3 }, slice);
}
```

---

## Step 386: Test Organization

### การจัดระเบียบ Test Files

```zig
// src/string_utils.zig
const std = @import("std");

pub fn capitalize(allocator: std.mem.Allocator, s: []const u8) ![]u8 {
    if (s.len == 0) return try allocator.dupe(u8, s);
    const result = try allocator.dupe(u8, s);
    result[0] = std.ascii.toUpper(result[0]);
    return result;
}

pub fn countOccurrences(haystack: []const u8, needle: []const u8) usize {
    if (needle.len == 0) return 0;
    var count: usize = 0;
    var i: usize = 0;
    while (i + needle.len <= haystack.len) : (i += 1) {
        if (std.mem.eql(u8, haystack[i .. i + needle.len], needle)) {
            count += 1;
        }
    }
    return count;
}

pub fn trim(s: []const u8) []const u8 {
    return std.mem.trim(u8, s, " \t\n\r");
}

// Tests ใน file เดียวกัน
test "capitalize - basic" {
    const result = try capitalize(std.testing.allocator, "hello");
    defer std.testing.allocator.free(result);
    try std.testing.expectEqualStrings("Hello", result);
}

test "capitalize - empty string" {
    const result = try capitalize(std.testing.allocator, "");
    defer std.testing.allocator.free(result);
    try std.testing.expectEqualStrings("", result);
}

test "countOccurrences" {
    try std.testing.expectEqual(@as(usize, 3), countOccurrences("hello hello hello", "hello"));
    try std.testing.expectEqual(@as(usize, 0), countOccurrences("hello", "world"));
    try std.testing.expectEqual(@as(usize, 2), countOccurrences("abab", "ab"));
}

test "trim" {
    try std.testing.expectEqualStrings("hello", trim("  hello  "));
    try std.testing.expectEqualStrings("hello", trim("\t\nhello\n"));
    try std.testing.expectEqualStrings("hello world", trim(" hello world "));
}
```

### แยก Test File

```zig
// src/string_utils_test.zig
const std = @import("std");
const string_utils = @import("string_utils.zig");

test "all string utils tests" {
    // รวม tests ทั้งหมดจาก module
    std.testing.refAllDecls(string_utils);
}

// หรือ test แยกแบบ explicit
test "integration - capitalize then count" {
    const allocator = std.testing.allocator;

    const words = [_][]const u8{ "hello", "world", "hello" };
    var result_list = std.ArrayList([]u8).init(allocator);
    defer {
        for (result_list.items) |item| {
            allocator.free(item);
        }
        result_list.deinit();
    }

    for (words) |word| {
        const cap = try string_utils.capitalize(allocator, word);
        try result_list.append(cap);
    }

    try std.testing.expectEqualStrings("Hello", result_list.items[0]);
    try std.testing.expectEqualStrings("World", result_list.items[1]);
    try std.testing.expectEqualStrings("Hello", result_list.items[2]);
}
```

### Test Namespace

```zig
const std = @import("std");

pub const MathModule = struct {
    pub fn square(x: i32) i32 {
        return x * x;
    }

    pub fn cube(x: i32) i32 {
        return x * x * x;
    }

    pub fn isPrime(n: u32) bool {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        var i: u32 = 3;
        while (i * i <= n) : (i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
};

// จัดกลุ่ม tests ใน namespace
const math_tests = struct {
    test "square" {
        try std.testing.expectEqual(@as(i32, 4), MathModule.square(2));
        try std.testing.expectEqual(@as(i32, 9), MathModule.square(3));
        try std.testing.expectEqual(@as(i32, 0), MathModule.square(0));
    }

    test "cube" {
        try std.testing.expectEqual(@as(i32, 8), MathModule.cube(2));
        try std.testing.expectEqual(@as(i32, 27), MathModule.cube(3));
    }

    test "isPrime" {
        try std.testing.expect(!MathModule.isPrime(0));
        try std.testing.expect(!MathModule.isPrime(1));
        try std.testing.expect(MathModule.isPrime(2));
        try std.testing.expect(MathModule.isPrime(3));
        try std.testing.expect(!MathModule.isPrime(4));
        try std.testing.expect(MathModule.isPrime(5));
        try std.testing.expect(!MathModule.isPrime(6));
        try std.testing.expect(MathModule.isPrime(7));
        try std.testing.expect(!MathModule.isPrime(100));
        try std.testing.expect(MathModule.isPrime(97));
    }
};

test "math module" {
    _ = math_tests;
}
```

---

## Step 387: Parameterized Tests

### Table-Driven Tests

```zig
const std = @import("std");

pub fn fibonacci(n: u32) u64 {
    if (n <= 1) return n;
    var a: u64 = 0;
    var b: u64 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}

test "fibonacci - table driven" {
    const TestCase = struct {
        input: u32,
        expected: u64,
    };

    const cases = [_]TestCase{
        .{ .input = 0, .expected = 0 },
        .{ .input = 1, .expected = 1 },
        .{ .input = 2, .expected = 1 },
        .{ .input = 3, .expected = 2 },
        .{ .input = 4, .expected = 3 },
        .{ .input = 5, .expected = 5 },
        .{ .input = 10, .expected = 55 },
        .{ .input = 20, .expected = 6765 },
        .{ .input = 30, .expected = 832040 },
    };

    for (cases) |tc| {
        const result = fibonacci(tc.input);
        if (result != tc.expected) {
            std.debug.print("fibonacci({}) = {}, expected {}\n", .{
                tc.input, result, tc.expected,
            });
        }
        try std.testing.expectEqual(tc.expected, result);
    }
}
```

### Inline Test Cases

```zig
const std = @import("std");

pub fn isValidEmail(email: []const u8) bool {
    // Simplified email validation
    const at_pos = std.mem.indexOf(u8, email, "@") orelse return false;
    if (at_pos == 0) return false;
    if (at_pos == email.len - 1) return false;

    const domain = email[at_pos + 1 ..];
    if (std.mem.indexOf(u8, domain, ".") == null) return false;
    if (domain[domain.len - 1] == '.') return false;

    return true;
}

test "email validation" {
    const valid_emails = [_][]const u8{
        "user@example.com",
        "alice@company.org",
        "test.email+tag@subdomain.domain.com",
    };

    for (valid_emails) |email| {
        if (!isValidEmail(email)) {
            std.debug.print("Expected valid email: {s}\n", .{email});
            try std.testing.expect(false);
        }
    }

    const invalid_emails = [_][]const u8{
        "",
        "notanemail",
        "@nodomain.com",
        "user@",
        "user@nodot",
        "user@domain.",
    };

    for (invalid_emails) |email| {
        if (isValidEmail(email)) {
            std.debug.print("Expected invalid email: {s}\n", .{email});
            try std.testing.expect(false);
        }
    }
}
```

### Comptime Test Generation

```zig
const std = @import("std");

fn testPow(base: comptime_int, exp: comptime_int, expected: comptime_int) !void {
    const result = std.math.pow(i64, base, exp);
    try std.testing.expectEqual(@as(i64, expected), result);
}

test "powers of 2" {
    comptime var i: comptime_int = 0;
    inline while (i < 10) : (i += 1) {
        const expected: comptime_int = 1 << i;
        try testPow(2, i, expected);
    }
}

test "powers of 3" {
    const cases = comptime blk: {
        var c: [6]struct { exp: u32, result: i64 } = undefined;
        var val: i64 = 1;
        for (0..6) |i| {
            c[i] = .{ .exp = @intCast(i), .result = val };
            val *= 3;
        }
        break :blk c;
    };

    for (cases) |tc| {
        const result = std.math.pow(i64, 3, tc.exp);
        try std.testing.expectEqual(tc.result, result);
    }
}
```

---

## Step 388: Test Coverage

### การวัด Test Coverage

```zig
// coverage_example.zig
const std = @import("std");

pub const Grade = enum {
    A, B, C, D, F,

    pub fn fromScore(score: u32) Grade {
        if (score >= 90) return .A;
        if (score >= 80) return .B;
        if (score >= 70) return .C;
        if (score >= 60) return .D;
        return .F;
    }

    pub fn toGPA(self: Grade) f32 {
        return switch (self) {
            .A => 4.0,
            .B => 3.0,
            .C => 2.0,
            .D => 1.0,
            .F => 0.0,
        };
    }

    pub fn isPassing(self: Grade) bool {
        return self != .F;
    }
};

// Tests ที่ cover ทุก branch
test "Grade.fromScore - all grades" {
    // Boundary testing
    try std.testing.expectEqual(Grade.A, Grade.fromScore(100));
    try std.testing.expectEqual(Grade.A, Grade.fromScore(90));
    try std.testing.expectEqual(Grade.B, Grade.fromScore(89));
    try std.testing.expectEqual(Grade.B, Grade.fromScore(80));
    try std.testing.expectEqual(Grade.C, Grade.fromScore(79));
    try std.testing.expectEqual(Grade.C, Grade.fromScore(70));
    try std.testing.expectEqual(Grade.D, Grade.fromScore(69));
    try std.testing.expectEqual(Grade.D, Grade.fromScore(60));
    try std.testing.expectEqual(Grade.F, Grade.fromScore(59));
    try std.testing.expectEqual(Grade.F, Grade.fromScore(0));
}

test "Grade.toGPA - all grades" {
    try std.testing.expectApproxEqAbs(@as(f32, 4.0), Grade.A.toGPA(), 0.001);
    try std.testing.expectApproxEqAbs(@as(f32, 3.0), Grade.B.toGPA(), 0.001);
    try std.testing.expectApproxEqAbs(@as(f32, 2.0), Grade.C.toGPA(), 0.001);
    try std.testing.expectApproxEqAbs(@as(f32, 1.0), Grade.D.toGPA(), 0.001);
    try std.testing.expectApproxEqAbs(@as(f32, 0.0), Grade.F.toGPA(), 0.001);
}

test "Grade.isPassing" {
    try std.testing.expect(Grade.A.isPassing());
    try std.testing.expect(Grade.B.isPassing());
    try std.testing.expect(Grade.C.isPassing());
    try std.testing.expect(Grade.D.isPassing());
    try std.testing.expect(!Grade.F.isPassing());
}
```

### Mutation Testing Mindset

```zig
const std = @import("std");

// Function ที่มี edge cases มากมาย
pub fn binarySearch(sorted: []const i32, target: i32) ?usize {
    if (sorted.len == 0) return null;

    var low: usize = 0;
    var high: usize = sorted.len - 1;

    while (low <= high) {
        const mid = low + (high - low) / 2;
        if (sorted[mid] == target) return mid;
        if (sorted[mid] < target) {
            low = mid + 1;
        } else {
            if (mid == 0) return null;
            high = mid - 1;
        }
    }
    return null;
}

test "binarySearch - comprehensive" {
    // Empty array
    try std.testing.expectEqual(@as(?usize, null), binarySearch(&[_]i32{}, 5));

    // Single element
    try std.testing.expectEqual(@as(?usize, 0), binarySearch(&[_]i32{5}, 5));
    try std.testing.expectEqual(@as(?usize, null), binarySearch(&[_]i32{5}, 3));

    // Target at boundaries
    const arr = [_]i32{ 1, 3, 5, 7, 9, 11 };
    try std.testing.expectEqual(@as(?usize, 0), binarySearch(&arr, 1));  // First
    try std.testing.expectEqual(@as(?usize, 5), binarySearch(&arr, 11)); // Last

    // Target in middle
    try std.testing.expectEqual(@as(?usize, 2), binarySearch(&arr, 5));
    try std.testing.expectEqual(@as(?usize, 3), binarySearch(&arr, 7));

    // Target not found
    try std.testing.expectEqual(@as(?usize, null), binarySearch(&arr, 0));
    try std.testing.expectEqual(@as(?usize, null), binarySearch(&arr, 2));
    try std.testing.expectEqual(@as(?usize, null), binarySearch(&arr, 12));

    // Duplicate handling (first occurrence)
    const dup_arr = [_]i32{ 1, 2, 2, 2, 3 };
    const dup_idx = binarySearch(&dup_arr, 2);
    try std.testing.expect(dup_idx != null);
    try std.testing.expectEqual(@as(i32, 2), dup_arr[dup_idx.?]);
}
```

---

## Step 389: Integration Tests

### Integration Test Pattern

```zig
const std = @import("std");

// ระบบ user management จำลอง
const UserStore = struct {
    users: std.StringHashMap(User),
    allocator: std.mem.Allocator,
    next_id: u64,

    const User = struct {
        id: u64,
        name: []const u8,
        email: []const u8,
        active: bool,
    };

    const UserStoreError = error{
        UserNotFound,
        EmailAlreadyExists,
        InvalidEmail,
    };

    pub fn init(allocator: std.mem.Allocator) UserStore {
        return .{
            .users = std.StringHashMap(User).init(allocator),
            .allocator = allocator,
            .next_id = 1,
        };
    }

    pub fn deinit(self: *UserStore) void {
        self.users.deinit();
    }

    pub fn createUser(self: *UserStore, name: []const u8, email: []const u8) UserStoreError!u64 {
        if (std.mem.indexOf(u8, email, "@") == null) return UserStoreError.InvalidEmail;
        if (self.users.contains(email)) return UserStoreError.EmailAlreadyExists;

        const id = self.next_id;
        self.next_id += 1;
        self.users.put(email, .{
            .id = id,
            .name = name,
            .email = email,
            .active = true,
        }) catch return UserStoreError.UserNotFound;

        return id;
    }

    pub fn findByEmail(self: *const UserStore, email: []const u8) UserStoreError!User {
        return self.users.get(email) orelse UserStoreError.UserNotFound;
    }

    pub fn deactivate(self: *UserStore, email: []const u8) UserStoreError!void {
        if (self.users.getPtr(email)) |user| {
            user.active = false;
        } else {
            return UserStoreError.UserNotFound;
        }
    }
};

test "UserStore - complete workflow" {
    const allocator = std.testing.allocator;

    var store = UserStore.init(allocator);
    defer store.deinit();

    // Create users
    const alice_id = try store.createUser("Alice", "alice@example.com");
    const bob_id = try store.createUser("Bob", "bob@example.com");

    try std.testing.expect(alice_id != bob_id);
    try std.testing.expect(alice_id > 0);

    // Find users
    const alice = try store.findByEmail("alice@example.com");
    try std.testing.expectEqualStrings("Alice", alice.name);
    try std.testing.expect(alice.active);

    // Duplicate email
    try std.testing.expectError(
        UserStore.UserStoreError.EmailAlreadyExists,
        store.createUser("Alice2", "alice@example.com"),
    );

    // Invalid email
    try std.testing.expectError(
        UserStore.UserStoreError.InvalidEmail,
        store.createUser("Test", "notvalidemail"),
    );

    // Deactivate
    try store.deactivate("alice@example.com");
    const alice_after = try store.findByEmail("alice@example.com");
    try std.testing.expect(!alice_after.active);

    // Bob should still be active
    const bob = try store.findByEmail("bob@example.com");
    try std.testing.expect(bob.active);
    _ = bob_id;
}
```

---

## Step 390: Test-driven Development ใน Zig

### TDD Workflow: Red-Green-Refactor

```zig
// Step 1: เขียน test ก่อน (Red - test จะ fail)
// Step 2: เขียน code ให้ผ่าน (Green)
// Step 3: refactor โค้ดโดยยังคง test ผ่าน

const std = @import("std");

// --- TDD Example: Calculator ---
// Step 1: Define interface ก่อน

const Calculator = struct {
    history: std.ArrayList(i64),

    pub fn init(allocator: std.mem.Allocator) Calculator {
        return .{ .history = std.ArrayList(i64).init(allocator) };
    }

    pub fn deinit(self: *Calculator) void {
        self.history.deinit();
    }

    pub fn add(self: *Calculator, a: i64, b: i64) !i64 {
        const result = a + b;
        try self.history.append(result);
        return result;
    }

    pub fn subtract(self: *Calculator, a: i64, b: i64) !i64 {
        const result = a - b;
        try self.history.append(result);
        return result;
    }

    pub fn multiply(self: *Calculator, a: i64, b: i64) !i64 {
        const result = a * b;
        try self.history.append(result);
        return result;
    }

    pub fn divide(self: *Calculator, a: i64, b: i64) !i64 {
        if (b == 0) return error.DivisionByZero;
        const result = @divTrunc(a, b);
        try self.history.append(result);
        return result;
    }

    pub fn lastResult(self: *const Calculator) ?i64 {
        if (self.history.items.len == 0) return null;
        return self.history.items[self.history.items.len - 1];
    }

    pub fn historyCount(self: *const Calculator) usize {
        return self.history.items.len;
    }

    pub fn clearHistory(self: *Calculator) void {
        self.history.clearRetainingCapacity();
    }
};

// Step 2: Tests (เขียนก่อน implement)
test "Calculator - basic operations" {
    var calc = Calculator.init(std.testing.allocator);
    defer calc.deinit();

    try std.testing.expectEqual(@as(i64, 5), try calc.add(2, 3));
    try std.testing.expectEqual(@as(i64, 1), try calc.subtract(3, 2));
    try std.testing.expectEqual(@as(i64, 6), try calc.multiply(2, 3));
    try std.testing.expectEqual(@as(i64, 3), try calc.divide(9, 3));
}

test "Calculator - history tracking" {
    var calc = Calculator.init(std.testing.allocator);
    defer calc.deinit();

    try std.testing.expectEqual(@as(?i64, null), calc.lastResult());

    _ = try calc.add(10, 20);
    try std.testing.expectEqual(@as(?i64, 30), calc.lastResult());
    try std.testing.expectEqual(@as(usize, 1), calc.historyCount());

    _ = try calc.multiply(5, 5);
    try std.testing.expectEqual(@as(?i64, 25), calc.lastResult());
    try std.testing.expectEqual(@as(usize, 2), calc.historyCount());

    calc.clearHistory();
    try std.testing.expectEqual(@as(usize, 0), calc.historyCount());
}

test "Calculator - error handling" {
    var calc = Calculator.init(std.testing.allocator);
    defer calc.deinit();

    try std.testing.expectError(error.DivisionByZero, calc.divide(10, 0));
    // History ไม่ควรเก็บ error results
    try std.testing.expectEqual(@as(usize, 0), calc.historyCount());
}

test "Calculator - chain operations" {
    var calc = Calculator.init(std.testing.allocator);
    defer calc.deinit();

    // (2 + 3) * 4 - 5 = 15
    const step1 = try calc.add(2, 3);       // 5
    const step2 = try calc.multiply(step1, 4); // 20
    const step3 = try calc.subtract(step2, 5); // 15

    try std.testing.expectEqual(@as(i64, 15), step3);
    try std.testing.expectEqual(@as(usize, 3), calc.historyCount());
}
```

### Property-Based Testing Pattern

```zig
const std = @import("std");

// Property: a + b == b + a (commutative)
test "addition is commutative" {
    const values = [_]i32{ -100, -1, 0, 1, 100, std.math.maxInt(i16) };

    for (values) |a| {
        for (values) |b| {
            const sum1 = @as(i64, a) + @as(i64, b);
            const sum2 = @as(i64, b) + @as(i64, a);
            try std.testing.expectEqual(sum1, sum2);
        }
    }
}

// Property: sort then check is sorted
pub fn bubbleSort(arr: []i32) void {
    var n = arr.len;
    while (n > 1) {
        var i: usize = 0;
        while (i + 1 < n) : (i += 1) {
            if (arr[i] > arr[i + 1]) {
                const tmp = arr[i];
                arr[i] = arr[i + 1];
                arr[i + 1] = tmp;
            }
        }
        n -= 1;
    }
}

fn isSorted(arr: []const i32) bool {
    for (0..arr.len - 1) |i| {
        if (arr[i] > arr[i + 1]) return false;
    }
    return true;
}

test "bubbleSort - result is always sorted" {
    const allocator = std.testing.allocator;

    const test_arrays = [_][]const i32{
        &[_]i32{ 3, 1, 4, 1, 5, 9, 2, 6 },
        &[_]i32{ 1, 2, 3, 4, 5 },
        &[_]i32{ 5, 4, 3, 2, 1 },
        &[_]i32{ 1 },
        &[_]i32{},
    };

    for (test_arrays) |original| {
        const arr = try allocator.dupe(i32, original);
        defer allocator.free(arr);

        bubbleSort(arr);

        if (arr.len > 1) {
            try std.testing.expect(isSorted(arr));
        }

        // Sum ต้องเท่ากับก่อน sort
        var sum_before: i64 = 0;
        var sum_after: i64 = 0;
        for (original) |v| sum_before += v;
        for (arr) |v| sum_after += v;
        try std.testing.expectEqual(sum_before, sum_after);
    }
}
```

---

## สรุปบทที่ 39

ในบทนี้เราได้เรียนรู้:

1. **Unit Testing** - ทำไมต้องเขียน test และโครงสร้างพื้นฐานของ test block
2. **test blocks** - การเขียน test ใน Zig รวมถึง setup/teardown pattern
3. **Expect functions** - expectEqual, expectEqualStrings, expectApproxEqAbs และอื่นๆ
4. **expectError** - การทดสอบ error cases อย่างถูกต้อง
5. **testing.allocator** - การตรวจจับ memory leaks อัตโนมัติ
6. **Test Organization** - การจัดระเบียบ test files และ namespaces
7. **Parameterized tests** - Table-driven tests และ inline test cases
8. **Test Coverage** - การครอบคลุม edge cases และ boundary conditions
9. **Integration tests** - การทดสอบ system ทั้งระบบ
10. **TDD** - Test-Driven Development workflow ใน Zig

Zig's built-in testing เป็นเครื่องมือที่ทรงพลังและใช้งานง่าย ช่วยให้เขียน code ที่มีคุณภาพสูงและไม่มี memory leaks
