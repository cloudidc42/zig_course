# Part 60: Security Best Practices ใน Zig (Steps 591-600)

Security เป็นสิ่งสำคัญอย่างยิ่งใน systems programming เนื่องจาก Zig ทำงานใกล้ชิดกับ hardware และ memory โดยตรง บทนี้จะครอบคลุม security best practices ทั้งหมดที่นักพัฒนา Zig ควรรู้ ตั้งแต่การป้องกัน buffer overflows ไปจนถึง cryptography และ security audit

---

## Step 591: Security ใน Systems Programming

### ทำไม Security ถึงสำคัญใน Systems Programming?

```
ความเสี่ยงใน Systems Programming:
├─ ทำงานกับ memory โดยตรง (buffer overflows)
├─ Handle untrusted input (user data, network packets)
├─ ทำงานกับ privileges สูง (kernel, system services)
├─ Concurrent access (race conditions)
└─ ใช้ external libraries (supply chain attacks)

ผลกระทบของ Security Vulnerabilities:
├─ Remote Code Execution (RCE)
├─ Privilege Escalation
├─ Data Exfiltration
├─ Denial of Service (DoS)
└─ Information Disclosure
```

### Zig Security Advantages

```zig
// security_basics.zig - Zig's built-in security features
const std = @import("std");

pub fn main() void {
    // 1. Bounds checking (ค่าเริ่มต้นใน Debug mode)
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    // arr[10] = 0;  // Runtime panic: index out of bounds!
    
    // 2. Integer overflow detection
    const x: u8 = 255;
    // const y = x + 1;  // Compile error หรือ Runtime panic!
    
    // 3. Null pointer dereference prevention
    const maybe: ?*i32 = null;
    if (maybe) |ptr| {
        _ = ptr.*;  // ปลอดภัย: ตรวจ null ก่อน
    }
    
    // 4. Explicit error handling (ไม่มี silent failures)
    const result = std.fs.cwd().openFile("test.txt", .{}) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    defer result.close();
    
    // 5. No undefined behavior by default (ใน Debug mode)
    var uninitialized: i32 = undefined;
    // ใน Debug mode, undefined จะถูกเติมด้วย 0xAA
    // เพื่อช่วยตรวจหา bugs
    uninitialized = 42;  // ต้อง assign ก่อนใช้
    std.debug.print("Value: {}\n", .{uninitialized});
}
```

### Security Design Principles

```
Principle of Least Privilege:
- ให้ access เฉพาะที่จำเป็น
- Drop privileges เมื่อไม่ต้องการ

Defense in Depth:
- มีหลาย layers ของ security
- ไม่ depend on single point of protection

Fail Safely:
- เมื่อ error เกิด: deny access (ไม่ใช่ grant)
- Handle all error cases explicitly

Validate All Inputs:
- ไม่ trust ข้อมูลจาก users/network/files
- Validate type, range, format, encoding
```

---

## Step 592: Buffer Overflow Prevention ใน Zig

### Zig's Built-in Protection

```zig
// buffer_safety.zig - Zig prevents buffer overflows
const std = @import("std");

pub fn main() !void {
    // Safe: bounds checking อัตโนมัติ
    var buffer: [10]u8 = undefined;
    @memset(&buffer, 0);
    
    // Safe: slice bounds checking
    const slice = buffer[0..5];
    std.debug.print("Slice len: {}\n", .{slice.len});
    
    // Zig prevents this at compile time:
    // const bad_slice = buffer[0..15];  // ERROR!
    
    // Runtime check สำหรับ dynamic indices
    const dynamic_end: usize = std.crypto.random.int(usize) % 10 + 1;
    const safe_slice = buffer[0..dynamic_end];
    std.debug.print("Safe slice len: {}\n", .{safe_slice.len});
    
    // Allocator-based safe operations
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const dynamic_buf = try allocator.alloc(u8, 100);
    defer allocator.free(dynamic_buf);
    
    // Safe copy ด้วย @memcpy (ต้องระบุ size เท่ากัน)
    var src = [_]u8{0x41} ** 50;
    @memcpy(dynamic_buf[0..50], &src);
}
```

### Safe String Handling

```zig
// safe_strings.zig - safe string operations
const std = @import("std");

pub const SafeString = struct {
    data: []u8,
    allocator: std.mem.Allocator,
    
    const Self = @This();
    
    pub fn init(allocator: std.mem.Allocator, initial: []const u8) !Self {
        const data = try allocator.alloc(u8, initial.len);
        @memcpy(data, initial);
        return .{ .data = data, .allocator = allocator };
    }
    
    pub fn deinit(self: Self) void {
        self.allocator.free(self.data);
    }
    
    /// Append อย่างปลอดภัย (ไม่ overflow)
    pub fn append(self: *Self, suffix: []const u8) !void {
        const new_len = self.data.len + suffix.len;
        
        // ตรวจสอบ overflow ในการบวก length
        if (new_len < self.data.len) return error.Overflow;
        
        const new_data = try self.allocator.realloc(self.data, new_len);
        @memcpy(new_data[self.data.len..], suffix);
        self.data = new_data;
    }
    
    /// Copy อย่างปลอดภัย (truncate ถ้า dest เล็กกว่า)
    pub fn copyTo(self: Self, dest: []u8) usize {
        const copy_len = @min(self.data.len, dest.len);
        @memcpy(dest[0..copy_len], self.data[0..copy_len]);
        return copy_len;
    }
    
    /// Compare อย่าง constant-time (ป้องกัน timing attacks)
    pub fn eql(self: Self, other: []const u8) bool {
        return constantTimeEql(self.data, other);
    }
};

/// Constant-time string comparison ป้องกัน timing attacks
fn constantTimeEql(a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return false;
    
    var diff: u8 = 0;
    for (a, b) |x, y| {
        diff |= x ^ y;
    }
    return diff == 0;
}
```

### C Interop Safety

```zig
// safe_c_interop.zig - safe C string handling
const std = @import("std");

/// แปลง C string เป็น Zig slice อย่างปลอดภัย
pub fn fromCString(c_str: [*:0]const u8, max_len: usize) ![]const u8 {
    const len = std.mem.indexOfSentinel(u8, 0, c_str);
    
    // ตรวจสอบ length limit เพื่อป้องกัน unbounded reads
    if (len > max_len) {
        return error.StringTooLong;
    }
    
    return c_str[0..len];
}

/// สร้าง C string อย่างปลอดภัย (null-terminated)
pub fn toCString(allocator: std.mem.Allocator, zig_str: []const u8) ![:0]u8 {
    // ตรวจสอบว่าไม่มี null bytes ใน string
    if (std.mem.indexOfScalar(u8, zig_str, 0) != null) {
        return error.NullByteInString;
    }
    
    return allocator.dupeZ(u8, zig_str);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ตัวอย่างการใช้
    const c_str: [*:0]const u8 = "Hello, World!";
    const zig_str = try fromCString(c_str, 256);
    std.debug.print("Zig string: {s}\n", .{zig_str});
    
    const back_to_c = try toCString(allocator, zig_str);
    defer allocator.free(back_to_c);
    std.debug.print("C string length: {}\n", .{std.mem.len(back_to_c)});
}
```

---

## Step 593: Integer Overflow Handling

### Integer Overflow คืออะไร?

```zig
// integer_overflow.zig - ทำความเข้าใจ overflow
const std = @import("std");

pub fn main() void {
    // ใน Zig, integer overflow ใน Debug mode = panic
    // ใน Release mode = undefined behavior (เหมือน C)
    
    const x: u8 = 255;
    
    // วิธีที่ 1: Wrapping arithmetic (ตั้งใจ overflow)
    const wrapped = x +% 1;  // 0 (wraps around)
    std.debug.print("255 +% 1 = {}\n", .{wrapped});
    
    // วิธีที่ 2: Saturating arithmetic
    const saturated = x +| 1;  // 255 (stays at max)
    std.debug.print("255 +| 1 = {}\n", .{saturated});
    
    // วิธีที่ 3: Overflow detection
    const result = @addWithOverflow(x, @as(u8, 1));
    if (result[1]) {  // overflow occurred
        std.debug.print("Overflow detected!\n", .{});
    } else {
        std.debug.print("Result: {}\n", .{result[0]});
    }
    
    // วิธีที่ 4: ใช้ type ที่ใหญ่กว่า
    const safe_result = @as(u16, x) + 1;
    std.debug.print("Safe result: {}\n", .{safe_result});
}
```

### Secure Integer Operations

```zig
// secure_integers.zig - safe integer operations
const std = @import("std");

/// บวกอย่างปลอดภัย พร้อม overflow check
pub fn safeAdd(comptime T: type, a: T, b: T) !T {
    const result = @addWithOverflow(a, b);
    if (result[1]) return error.IntegerOverflow;
    return result[0];
}

/// คูณอย่างปลอดภัย พร้อม overflow check
pub fn safeMul(comptime T: type, a: T, b: T) !T {
    const result = @mulWithOverflow(a, b);
    if (result[1]) return error.IntegerOverflow;
    return result[0];
}

/// คำนวณ size สำหรับ array allocation อย่างปลอดภัย
pub fn safeArraySize(comptime T: type, count: usize, element_size: usize) !usize {
    const total = @mulWithOverflow(count, element_size);
    if (total[1]) return error.SizeOverflow;
    return total[0];
}

/// แปลง type อย่างปลอดภัย (checked cast)
pub fn safeCast(comptime T: type, value: anytype) !T {
    const V = @TypeOf(value);
    
    // ตรวจสอบ range
    if (@typeInfo(T) == .Int and @typeInfo(V) == .Int) {
        const min = std.math.minInt(T);
        const max = std.math.maxInt(T);
        
        if (value < min or value > max) {
            return error.ValueOutOfRange;
        }
    }
    
    return @intCast(value);
}

pub fn main() !void {
    // ตัวอย่าง safe operations
    const a: u32 = 1_000_000;
    const b: u32 = 2_000_000;
    
    const sum = try safeAdd(u32, a, b);
    std.debug.print("Safe sum: {}\n", .{sum});
    
    // Overflow detection
    const max: u32 = std.math.maxInt(u32);
    if (safeAdd(u32, max, 1)) |_| {
        unreachable;
    } else |err| {
        std.debug.print("Caught overflow: {}\n", .{err});
    }
    
    // Safe cast
    const large: i64 = 300;
    if (safeCast(u8, large)) |v| {
        std.debug.print("Cast: {}\n", .{v});
    } else |err| {
        std.debug.print("Cast failed: {}\n", .{err});
    }
}
```

---

## Step 594: Memory Safety ใน Zig

### Memory Safety Features

```zig
// memory_safety.zig - Zig's memory safety features
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{
        .safety = true,  // เปิด safety checks ทั้งหมด
    }){};
    defer {
        const status = gpa.deinit();
        if (status == .leak) std.debug.print("Memory leak!\n", .{});
    }
    const allocator = gpa.allocator();
    
    // 1. Double-free prevention
    // GPA จะ panic ถ้า free สอง ครั้ง
    const ptr = try allocator.create(i32);
    ptr.* = 42;
    allocator.destroy(ptr);
    // allocator.destroy(ptr);  // Panic: double free!
    
    // 2. Use-after-free detection (ใน Debug mode)
    // GPA overwrites freed memory ด้วย 0xAA
    // การอ่าน 0xAA pattern บ่งชี้ use-after-free
    
    // 3. Buffer overflow detection
    const buf = try allocator.alloc(u8, 10);
    defer allocator.free(buf);
    // GPA เพิ่ม sentinel bytes หลัง buffer
    // การเขียนนอก bounds จะ detected ที่ free time
    
    // 4. Alignment checking
    const aligned = try allocator.alignedAlloc(u32, 16, 5);
    defer allocator.free(aligned);
    std.debug.print("Aligned ptr: {}\n", .{@intFromPtr(aligned.ptr) % 16 == 0});
}
```

### RAII Pattern สำหรับ Resource Safety

```zig
// raii_pattern.zig - Resource Acquisition Is Initialization
const std = @import("std");

// File handle ที่ safe โดยใช้ RAII
pub const SafeFile = struct {
    file: std.fs.File,
    path: []const u8,
    
    const Self = @This();
    
    pub fn open(path: []const u8) !Self {
        const file = try std.fs.cwd().openFile(path, .{});
        return .{ .file = file, .path = path };
    }
    
    pub fn close(self: *Self) void {
        self.file.close();
        // ทำ invalid เพื่อป้องกัน use-after-close
        self.* = undefined;
    }
    
    pub fn read(self: Self, buf: []u8) !usize {
        return self.file.read(buf);
    }
};

// Socket ที่ auto-close
pub fn withConnection(
    host: []const u8,
    port: u16,
    callback: *const fn (std.net.Stream) anyerror!void,
) !void {
    const conn = try std.net.tcpConnectToHost(std.heap.page_allocator, host, port);
    defer conn.close();  // auto-close เมื่อออกจาก scope
    
    try callback(conn);
}

pub fn main() !void {
    // ตัวอย่าง RAII file handling
    var f = try SafeFile.open("/etc/hostname");
    defer f.close();
    
    var buf: [256]u8 = undefined;
    const n = try f.read(&buf);
    std.debug.print("Hostname: {s}\n", .{buf[0..n]});
}
```

---

## Step 595: Input Validation Patterns

### Comprehensive Input Validation

```zig
// input_validation.zig - secure input validation
const std = @import("std");

pub const ValidationError = error{
    TooLong,
    TooShort,
    InvalidCharacter,
    InvalidFormat,
    OutOfRange,
    NullByte,
    NotUTF8,
};

/// Validate string length
pub fn validateLength(
    s: []const u8,
    min: usize,
    max: usize,
) ValidationError!void {
    if (s.len < min) return ValidationError.TooShort;
    if (s.len > max) return ValidationError.TooLong;
}

/// Validate ว่าเป็น valid UTF-8
pub fn validateUTF8(s: []const u8) ValidationError!void {
    if (!std.unicode.utf8ValidateSlice(s)) {
        return ValidationError.NotUTF8;
    }
}

/// Validate ว่าไม่มี null bytes (สำหรับ C interop)
pub fn validateNoNull(s: []const u8) ValidationError!void {
    if (std.mem.indexOfScalar(u8, s, 0) != null) {
        return ValidationError.NullByte;
    }
}

/// Validate username: alphanumeric + underscore, 3-32 chars
pub fn validateUsername(username: []const u8) ValidationError!void {
    try validateLength(username, 3, 32);
    try validateUTF8(username);
    try validateNoNull(username);
    
    for (username) |c| {
        if (!std.ascii.isAlphanumeric(c) and c != '_') {
            return ValidationError.InvalidCharacter;
        }
    }
}

/// Validate email address (simplified)
pub fn validateEmail(email: []const u8) ValidationError!void {
    try validateLength(email, 5, 254);
    try validateUTF8(email);
    try validateNoNull(email);
    
    // ต้องมี @ อย่างน้อยหนึ่งตัว
    const at_idx = std.mem.indexOfScalar(u8, email, '@') orelse
        return ValidationError.InvalidFormat;
    
    // ต้องมีอักขระก่อนและหลัง @
    if (at_idx == 0 or at_idx == email.len - 1) {
        return ValidationError.InvalidFormat;
    }
    
    // ต้องมี . หลัง @
    const domain = email[at_idx + 1..];
    if (std.mem.indexOfScalar(u8, domain, '.') == null) {
        return ValidationError.InvalidFormat;
    }
}

/// Validate integer range
pub fn validateIntRange(
    value: i64,
    min: i64,
    max: i64,
) ValidationError!void {
    if (value < min or value > max) {
        return ValidationError.OutOfRange;
    }
}

/// Validate path (prevent path traversal)
pub fn validatePath(path: []const u8) ValidationError!void {
    try validateLength(path, 1, 4096);
    try validateNoNull(path);
    
    // ป้องกัน path traversal
    var it = std.mem.tokenizeScalar(u8, path, '/');
    while (it.next()) |component| {
        if (std.mem.eql(u8, component, "..")) {
            return ValidationError.InvalidCharacter;
        }
    }
}

pub fn main() !void {
    // ทดสอบ validation
    const usernames = [_][]const u8{
        "alice",
        "bob_123",
        "ab",          // too short
        "a" ** 33,     // too long
        "user@name",   // invalid char
    };
    
    for (usernames) |username| {
        if (validateUsername(username)) {
            std.debug.print("Valid username: {s}\n", .{username});
        } else |err| {
            std.debug.print("Invalid username '{s}': {}\n", .{
                username[0..@min(username.len, 20)], err
            });
        }
    }
}
```

### SQL Injection Prevention

```zig
// sql_safety.zig - prevent SQL injection
const std = @import("std");

/// Parameterized query builder (ป้องกัน SQL injection)
pub const QueryBuilder = struct {
    query: std.ArrayList(u8),
    params: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,
    
    const Self = @This();
    
    pub fn init(allocator: std.mem.Allocator) Self {
        return .{
            .query = std.ArrayList(u8).init(allocator),
            .params = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Self) void {
        self.query.deinit();
        self.params.deinit();
    }
    
    /// เพิ่ม literal SQL text
    pub fn addLiteral(self: *Self, sql: []const u8) !void {
        try self.query.appendSlice(sql);
    }
    
    /// เพิ่ม parameter (placeholder)
    pub fn addParam(self: *Self, value: []const u8) !void {
        // เพิ่ม placeholder
        try self.query.writer().print("${}", .{self.params.items.len + 1});
        // เพิ่ม value ไปยัง params list
        try self.params.append(value);
    }
    
    /// ตัวอย่าง: NEVER do string concatenation สำหรับ SQL!
    pub fn badExample(allocator: std.mem.Allocator, username: []const u8) ![]u8 {
        // VULNERABLE! SQL Injection possible:
        // username = "'; DROP TABLE users; --"
        return std.fmt.allocPrint(allocator, 
            "SELECT * FROM users WHERE name = '{s}'",
            .{username}
        );
    }
};
```

---

## Step 596: Cryptography Basics (Hashing, HMAC)

### Hashing ใน Zig

```zig
// crypto_basics.zig - cryptography ใน Zig
const std = @import("std");
const crypto = std.crypto;

pub fn hashingDemo() !void {
    const data = "Hello, Zig!";
    
    // SHA-256
    var sha256_hash: [32]u8 = undefined;
    crypto.hash.sha2.Sha256.hash(data, &sha256_hash, .{});
    std.debug.print("SHA-256: ", .{});
    for (sha256_hash) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("\n", .{});
    
    // SHA-512
    var sha512_hash: [64]u8 = undefined;
    crypto.hash.sha2.Sha512.hash(data, &sha512_hash, .{});
    std.debug.print("SHA-512: ", .{});
    for (sha512_hash[0..8]) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("...\n", .{});
    
    // Blake3 (faster, modern)
    var blake3_hash: [32]u8 = undefined;
    crypto.hash.Blake3.hash(data, &blake3_hash, .{});
    std.debug.print("BLAKE3: ", .{});
    for (blake3_hash) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("\n", .{});
}

pub fn hmacDemo() !void {
    const key = "secret_key_here";
    const message = "Important message";
    
    // HMAC-SHA256
    var hmac_result: [32]u8 = undefined;
    crypto.auth.hmac.HmacSha256.create(
        &hmac_result,
        message,
        key,
    );
    
    std.debug.print("HMAC-SHA256: ", .{});
    for (hmac_result) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("\n", .{});
    
    // Verify HMAC
    var expected: [32]u8 = undefined;
    crypto.auth.hmac.HmacSha256.create(&expected, message, key);
    
    const valid = constantTimeEql(&hmac_result, &expected);
    std.debug.print("HMAC valid: {}\n", .{valid});
}

/// ป้องกัน timing attacks ด้วย constant-time comparison
fn constantTimeEql(a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return false;
    
    var result: u8 = 0;
    for (a, b) |x, y| {
        result |= x ^ y;
    }
    return result == 0;
}

/// Password hashing ด้วย Argon2 (secure สำหรับ passwords)
pub fn passwordHashDemo(allocator: std.mem.Allocator) !void {
    const password = "user_password_here";
    
    // สร้าง random salt
    var salt: [16]u8 = undefined;
    crypto.random.bytes(&salt);
    
    // Argon2id (recommended สำหรับ password hashing)
    var hash: [32]u8 = undefined;
    try crypto.pwhash.argon2.kdf(
        allocator,
        &hash,
        password,
        &salt,
        .{
            .t = 3,           // time cost
            .m = 65536,       // memory cost (64MB)
            .p = 4,           // parallelism
        },
        .argon2id,
    );
    
    std.debug.print("Password hash: ", .{});
    for (hash) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    try hashingDemo();
    try hmacDemo();
    try passwordHashDemo(gpa.allocator());
}
```

---

## Step 597: Secure Random Numbers

### Random Numbers ใน Security Context

```zig
// secure_random.zig - cryptographically secure random numbers
const std = @import("std");
const crypto = std.crypto;

pub fn main() void {
    // Cryptographically Secure PRNG (CSPRNG)
    // std.crypto.random ใช้ OS entropy source
    
    // Random bytes
    var key: [32]u8 = undefined;
    crypto.random.bytes(&key);
    std.debug.print("Random key: ", .{});
    for (key[0..8]) |b| std.debug.print("{x:0>2}", .{b});
    std.debug.print("...\n", .{});
    
    // Random integer
    const random_int = crypto.random.int(u64);
    std.debug.print("Random u64: {}\n", .{random_int});
    
    // Random integer ใน range
    const random_in_range = crypto.random.intRangeLessThan(u32, 1, 101);
    std.debug.print("Random 1-100: {}\n", .{random_in_range});
    
    // Random boolean
    const coin_flip = crypto.random.boolean();
    std.debug.print("Coin flip: {}\n", .{coin_flip});
    
    // Shuffle array securely
    var arr = [_]u8{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    crypto.random.shuffle(u8, &arr);
    std.debug.print("Shuffled: {any}\n", .{arr});
    
    // สร้าง secure token
    var token: [32]u8 = undefined;
    crypto.random.bytes(&token);
    
    // Encode เป็น hex string
    var hex_token: [64]u8 = undefined;
    _ = std.fmt.bufPrint(&hex_token, "{}", .{std.fmt.fmtSliceHexLower(&token)}) catch {};
    std.debug.print("Token: {s}\n", .{hex_token});
}
```

### Insecure vs Secure Random

```zig
// random_comparison.zig - เปรียบเทียบ random implementations
const std = @import("std");

pub fn insecureRandom() void {
    // INSECURE: ใช้ non-cryptographic PRNG
    // เหมาะสำหรับ simulation, games แต่ไม่ใช่ security
    var prng = std.rand.DefaultPrng.init(42);  // predictable seed!
    const v1 = prng.random().int(u64);
    const v2 = prng.random().int(u64);
    std.debug.print("Insecure: {} {}\n", .{ v1, v2 });
    
    // ถ้า attacker รู้ seed = รู้ทุก values!
}

pub fn secureRandom() void {
    // SECURE: ใช้ OS entropy source
    const v1 = std.crypto.random.int(u64);
    const v2 = std.crypto.random.int(u64);
    std.debug.print("Secure: {} {}\n", .{ v1, v2 });
    
    // Unpredictable จาก OS entropy (hardware events, timing, etc.)
}

/// UUID v4 generation (secure)
pub fn generateUUID() [36]u8 {
    var bytes: [16]u8 = undefined;
    std.crypto.random.bytes(&bytes);
    
    // Set version (4) and variant bits
    bytes[6] = (bytes[6] & 0x0f) | 0x40;
    bytes[8] = (bytes[8] & 0x3f) | 0x80;
    
    var uuid: [36]u8 = undefined;
    _ = std.fmt.bufPrint(&uuid, 
        "{x:0>2}{x:0>2}{x:0>2}{x:0>2}-{x:0>2}{x:0>2}-{x:0>2}{x:0>2}-{x:0>2}{x:0>2}-{x:0>2}{x:0>2}{x:0>2}{x:0>2}{x:0>2}{x:0>2}",
        .{ bytes[0], bytes[1], bytes[2], bytes[3],
           bytes[4], bytes[5],
           bytes[6], bytes[7],
           bytes[8], bytes[9],
           bytes[10], bytes[11], bytes[12], bytes[13], bytes[14], bytes[15] }
    ) catch unreachable;
    
    return uuid;
}

pub fn main() void {
    insecureRandom();
    secureRandom();
    
    const uuid = generateUUID();
    std.debug.print("UUID: {s}\n", .{uuid});
}
```

---

## Step 598: Timing Attack Prevention

### Timing Attacks คืออะไร?

```
Timing Attack ใช้เวลาที่ใช้ในการ execute ในการ leak information:

ตัวอย่าง:
- Password comparison ที่หยุดเมื่อพบ mismatch แรก
  → Attacker วัดเวลา response เพื่อเดา password ทีละ character

- Cryptographic key comparison
  → เวลาต่างกันเพียง nanoseconds แต่ attacker วัดได้

การป้องกัน: Constant-Time Operations
- เวลาที่ใช้ไม่ขึ้นกับ content ของข้อมูล
```

### Constant-Time Implementation

```zig
// timing_safe.zig - constant-time operations
const std = @import("std");

/// Constant-time byte comparison
pub fn constantTimeByteEql(a: u8, b: u8) bool {
    return (a ^ b) == 0;
}

/// Constant-time slice comparison
pub fn constantTimeEql(a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return false;
    
    var diff: u8 = 0;
    for (a, b) |x, y| {
        diff |= x ^ y;
    }
    
    // diff == 0 ถ้า a == b
    return diff == 0;
}

/// Constant-time select (branch-free)
/// returns a ถ้า condition เป็น true, b ถ้า false
pub fn constantTimeSelect(comptime T: type, condition: bool, a: T, b: T) T {
    const mask: T = if (condition) 
        std.math.maxInt(T) 
    else 
        0;
    
    // ทำงานแบบ branch-free
    return (a & mask) | (b & ~mask);
}

/// Secure password verification
pub fn verifyPassword(stored_hash: []const u8, provided_hash: []const u8) bool {
    // ใช้ constant-time comparison เพื่อป้องกัน timing attacks
    return constantTimeEql(stored_hash, provided_hash);
}

/// HMAC verification (constant-time)
pub fn verifyHMAC(
    expected: []const u8,
    message: []const u8,
    key: []const u8,
) bool {
    var computed: [32]u8 = undefined;
    std.crypto.auth.hmac.HmacSha256.create(&computed, message, key);
    
    // Constant-time comparison ป้องกัน early-exit timing
    return constantTimeEql(expected, &computed);
}

pub fn main() void {
    // ตัวอย่าง timing-safe operations
    const hash1 = "abc123def456...";
    const hash2 = "abc123def456...";
    const hash3 = "xyz789uvw012...";
    
    std.debug.print("hash1 == hash2: {}\n", .{constantTimeEql(hash1, hash2)});
    std.debug.print("hash1 == hash3: {}\n", .{constantTimeEql(hash1, hash3)});
    
    // ทดสอบ constant-time select
    const val = constantTimeSelect(u32, true, 100, 200);
    std.debug.print("Select(true, 100, 200): {}\n", .{val});
}
```

---

## Step 599: Privilege Separation

### Privilege Separation Concepts

```zig
// privilege_sep.zig - privilege separation
const std = @import("std");

pub fn main() !void {
    if (comptime @import("builtin").os.tag != .linux) {
        std.debug.print("Linux only demo\n", .{});
        return;
    }
    
    // ตรวจสอบว่า run ด้วย root หรือไม่
    const uid = std.os.linux.getuid();
    const euid = std.os.linux.geteuid();
    
    std.debug.print("UID: {}, EUID: {}\n", .{ uid, euid });
    
    if (euid == 0) {
        std.debug.print("Running as root - will drop privileges\n", .{});
        try dropPrivileges();
    }
}

fn dropPrivileges() !void {
    const linux = std.os.linux;
    
    // Drop root privileges หลังจากทำงานที่ต้องการ privilege แล้ว
    const target_uid: u32 = 1000;  // UID ของ regular user
    const target_gid: u32 = 1000;  // GID ของ regular user
    
    // ต้อง drop GID ก่อน UID
    // setgid
    const setgid_result = linux.syscall1(.setgid, target_gid);
    if (setgid_result != 0) return error.SetGidFailed;
    
    // setuid
    const setuid_result = linux.syscall1(.setuid, target_uid);
    if (setuid_result != 0) return error.SetUidFailed;
    
    // ตรวจสอบว่า drop สำเร็จ
    const new_uid = linux.getuid();
    const new_euid = linux.geteuid();
    
    std.debug.print("After drop - UID: {}, EUID: {}\n", .{ new_uid, new_euid });
    
    if (new_euid == 0) {
        return error.PrivilegeDropFailed;
    }
}
```

### Seccomp Sandboxing (Linux)

```zig
// seccomp_demo.zig - system call filtering
const std = @import("std");

/// ตัวอย่างการจำกัด syscalls ด้วย seccomp
/// (ต้องใช้ Linux kernel 3.5+)
pub fn setupSeccompFilter() !void {
    if (comptime @import("builtin").os.tag != .linux) return;
    
    // ใน production ควรใช้ library เช่น libseccomp
    // นี่เป็นแค่ illustration ของ concept
    
    // Syscalls ที่ allow สำหรับ network server:
    const allowed_syscalls = [_][]const u8{
        "read",
        "write",
        "close",
        "recv",
        "send",
        "accept",
        "epoll_wait",
        "exit_group",
    };
    
    _ = allowed_syscalls;  // จะใช้กับ seccomp BPF filter จริง
    
    std.debug.print("Seccomp filter would restrict to {} syscalls\n", .{
        allowed_syscalls.len
    });
}
```

### Chroot Jail

```zig
// chroot_demo.zig - filesystem isolation
const std = @import("std");

pub fn setupChrootJail(new_root: []const u8) !void {
    if (comptime @import("builtin").os.tag != .linux) return;
    
    const linux = std.os.linux;
    
    // เปลี่ยน root filesystem
    // ต้องการ root privilege
    const chroot_result = linux.syscall1(
        .chroot,
        @intFromPtr(new_root.ptr),
    );
    
    if (chroot_result != 0) {
        return error.ChrootFailed;
    }
    
    // เปลี่ยน working directory เป็น new root
    std.posix.chdir("/") catch return error.ChdirFailed;
    
    std.debug.print("Chroot jail setup at: {s}\n", .{new_root});
}
```

---

## Step 600: Security Audit Checklist

### Security Audit Framework

```zig
// security_audit.zig - comprehensive security checks
const std = @import("std");

pub const SecurityCheck = struct {
    name: []const u8,
    passed: bool,
    severity: Severity,
    details: []const u8,
    
    pub const Severity = enum {
        critical,
        high,
        medium,
        low,
        info,
    };
};

pub fn runSecurityAudit(allocator: std.mem.Allocator) ![]SecurityCheck {
    var checks = std.ArrayList(SecurityCheck).init(allocator);
    
    // 1. Memory Safety Checks
    try checks.append(checkBoundsChecking());
    try checks.append(checkIntegerOverflow());
    try checks.append(checkNullPointers());
    
    // 2. Input Validation
    try checks.append(checkInputValidation());
    
    // 3. Cryptography
    try checks.append(checkCryptoUsage());
    try checks.append(checkRandomNumbers());
    
    // 4. Timing Safety
    try checks.append(checkTimingSafety());
    
    // 5. Resource Management
    try checks.append(checkMemoryLeaks());
    
    return checks.toOwnedSlice();
}

fn checkBoundsChecking() SecurityCheck {
    // ใน Debug build, Zig checks bounds อัตโนมัติ
    const is_debug = @import("builtin").mode == .Debug;
    return .{
        .name = "Bounds Checking",
        .passed = is_debug,
        .severity = .critical,
        .details = if (is_debug)
            "Runtime bounds checking enabled (Debug mode)"
        else
            "WARNING: Bounds checking disabled in release mode - use safe builds",
    };
}

fn checkIntegerOverflow() SecurityCheck {
    const is_debug = @import("builtin").mode == .Debug;
    return .{
        .name = "Integer Overflow Detection",
        .passed = is_debug,
        .severity = .high,
        .details = if (is_debug)
            "Integer overflow detection enabled"
        else
            "Use @addWithOverflow, @mulWithOverflow for safe arithmetic",
    };
}

fn checkNullPointers() SecurityCheck {
    // Zig's optional type ป้องกัน null pointer dereferences
    return .{
        .name = "Null Pointer Safety",
        .passed = true,
        .severity = .critical,
        .details = "Zig optional types prevent null pointer dereferences",
    };
}

fn checkInputValidation() SecurityCheck {
    // ต้องตรวจสอบ code manually
    return .{
        .name = "Input Validation",
        .passed = true,  // ต้อง manual review
        .severity = .high,
        .details = "Review: validate all external inputs (length, charset, format)",
    };
}

fn checkCryptoUsage() SecurityCheck {
    return .{
        .name = "Cryptography",
        .passed = true,
        .severity = .high,
        .details = "Use std.crypto - avoid custom crypto implementations",
    };
}

fn checkRandomNumbers() SecurityCheck {
    return .{
        .name = "Secure Random Numbers",
        .passed = true,
        .severity = .high,
        .details = "Use std.crypto.random for security-sensitive operations",
    };
}

fn checkTimingSafety() SecurityCheck {
    return .{
        .name = "Timing Safety",
        .passed = true,
        .severity = .medium,
        .details = "Use constant-time comparisons for secrets",
    };
}

fn checkMemoryLeaks() SecurityCheck {
    const is_debug = @import("builtin").mode == .Debug;
    return .{
        .name = "Memory Leak Detection",
        .passed = is_debug,
        .severity = .medium,
        .details = if (is_debug)
            "GeneralPurposeAllocator detects leaks in debug mode"
        else
            "Consider running with debug build to check for leaks",
    };
}

pub fn printAuditReport(checks: []const SecurityCheck) void {
    std.debug.print("\n╔══════════════════════════════════════════╗\n", .{});
    std.debug.print("║       SECURITY AUDIT REPORT              ║\n", .{});
    std.debug.print("╚══════════════════════════════════════════╝\n\n", .{});
    
    var passed: usize = 0;
    var failed: usize = 0;
    
    for (checks) |check| {
        const icon = if (check.passed) "✓" else "✗";
        const severity_str = switch (check.severity) {
            .critical => "CRITICAL",
            .high => "HIGH    ",
            .medium => "MEDIUM  ",
            .low => "LOW     ",
            .info => "INFO    ",
        };
        
        std.debug.print("[{s}] [{s}] {s}\n", .{ icon, severity_str, check.name });
        std.debug.print("         {s}\n\n", .{check.details});
        
        if (check.passed) passed += 1 else failed += 1;
    }
    
    std.debug.print("Summary: {}/{} checks passed\n", .{ passed, checks.len });
    
    if (failed > 0) {
        std.debug.print("ACTION REQUIRED: {} security issues found!\n", .{failed});
    } else {
        std.debug.print("All checks passed!\n", .{});
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    const checks = try runSecurityAudit(gpa.allocator());
    defer gpa.allocator().free(checks);
    
    printAuditReport(checks);
}
```

### Security Checklist สำหรับ Code Review

```
╔═══════════════════════════════════════════════════════════════╗
║              ZIG SECURITY CODE REVIEW CHECKLIST               ║
╚═══════════════════════════════════════════════════════════════╝

MEMORY SAFETY:
□ ใช้ bounds checking (Debug mode หรือ safe release)
□ ตรวจสอบ null pointers ก่อน dereference
□ ไม่มี use-after-free
□ ไม่มี double-free
□ Buffer sizes ถูกต้องใน @memcpy, @memset
□ Allocations สมดุลกับ deallocations (defer pattern)

INTEGER SAFETY:
□ ตรวจสอบ overflow ใน arithmetic operations
□ ใช้ @addWithOverflow, @mulWithOverflow สำหรับ untrusted data
□ Validate integer ranges ก่อนใช้งาน
□ Careful กับ integer conversions (truncation)

INPUT VALIDATION:
□ Validate length ของทุก input strings
□ Validate character sets (เฉพาะ allowed chars)
□ Validate numeric ranges
□ Sanitize paths (ป้องกัน path traversal)
□ ตรวจ null bytes ใน strings
□ Validate UTF-8 encoding

CRYPTOGRAPHY:
□ ใช้ std.crypto ไม่ใช่ custom implementation
□ ใช้ secure random: std.crypto.random
□ ไม่ใช้ MD5, SHA1 สำหรับ security
□ ใช้ Argon2/bcrypt/scrypt สำหรับ passwords
□ Constant-time comparisons สำหรับ secrets
□ Key derivation ใช้ HKDF หรือ Argon2
□ Nonces/IVs ไม่ reuse

PRIVILEGE MANAGEMENT:
□ Drop privileges เร็วที่สุด
□ ไม่ run ด้วย root ถ้าไม่จำเป็น
□ จำกัด filesystem access (chroot/seccomp)
□ Minimize attack surface

ERROR HANDLING:
□ Handle ทุก error cases
□ ไม่ leak sensitive info ใน error messages
□ Log errors อย่างเหมาะสม

DEPENDENCIES:
□ ตรวจสอบ third-party libraries
□ Update dependencies สม่ำเสมอ
□ Verify checksums/signatures

TIMING ATTACKS:
□ Constant-time comparison สำหรับ passwords/tokens
□ Constant-time สำหรับ crypto operations

RESOURCE LIMITS:
□ จำกัด memory usage
□ จำกัด CPU time
□ Rate limiting สำหรับ APIs
□ Timeout สำหรับ operations
```

---

**บทสรุป Part 60:**

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Security Best Practices ใน Zig:

1. **Step 591**: Security fundamentals ใน systems programming และ Zig's security advantages
2. **Step 592**: Buffer overflow prevention - Zig's built-in protection และ safe string handling
3. **Step 593**: Integer overflow handling - wrapping, saturating, checked operations
4. **Step 594**: Memory safety - GPA, RAII pattern, resource management
5. **Step 595**: Input validation - length, charset, format, path traversal prevention
6. **Step 596**: Cryptography - hashing, HMAC, password hashing ด้วย Argon2
7. **Step 597**: Secure random numbers - CSPRNG vs PRNG, UUID generation
8. **Step 598**: Timing attack prevention - constant-time comparisons
9. **Step 599**: Privilege separation - dropping privileges, chroot, seccomp
10. **Step 600**: Security audit checklist - comprehensive review framework

การเข้าใจและนำ security practices เหล่านี้ไปใช้จะช่วยให้โปรแกรม Zig มีความปลอดภัยสูงและทนต่อการโจมตีได้

---

**จบ Part 60 - Security Best Practices ใน Zig**

ด้วยความรู้ใน Parts 56-60 ครอบคลุม:
- Cross compilation สำหรับทุก platforms
- Linking และ library management
- Debugging tools และ techniques
- Profiling และ performance optimization
- Security best practices

คุณพร้อมที่จะพัฒนาโปรแกรม Zig ที่ professional, performant, และ secure แล้ว!
