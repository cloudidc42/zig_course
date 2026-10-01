# Part 18: Error Handling พื้นฐานใน Zig (Steps 171-180)

Error handling เป็นหนึ่งในจุดแข็งที่โดดเด่นที่สุดของภาษา Zig ต่างจากภาษาอื่นๆ ที่ใช้ exceptions หรือ return codes แบบดั้งเดิม Zig มีระบบ error handling ที่ถูก build เข้าไปใน type system อย่างแน่นหนา ทำให้ compiler สามารถตรวจสอบได้ว่าเราจัดการ errors ครบถ้วนหรือไม่

---

## Step 171: Error Handling ใน Zig - Philosophy

### แนวคิดหลักของ Error Handling ใน Zig

Zig มีปรัชญาที่ชัดเจนเกี่ยวกับ error handling:

1. **Errors เป็น values** - errors ไม่ใช่ exceptions ที่ถูก throw แต่เป็น values ที่ return กลับมาเหมือนกัน
2. **Errors ต้องถูกจัดการ** - compiler จะ error ถ้าคุณไม่จัดการ error ที่อาจเกิดขึ้น
3. **ไม่มี hidden control flow** - ไม่มีการกระโดด stack แบบ exception ทำให้โค้ดอ่านง่ายและ predictable
4. **Performance** - ไม่มี overhead จาก exception handling mechanism
5. **Explicit** - ทุกจุดที่อาจเกิด error จะชัดเจนจากการอ่านโค้ด

### เปรียบเทียบกับภาษาอื่น

```
ภาษา Java/Python:
try {
    result = doSomething()  // อาจ throw exception ที่ไหนก็ได้
    result2 = doMore()      // ไม่รู้ว่า function นี้ throw อะไรบ้าง
} catch (Exception e) {
    // จัดการ error
}

ภาษา C:
int result = doSomething();
if (result < 0) {  // ต้องจำว่า -1 หมายถึงอะไร?
    // handle error
}

ภาษา Zig:
const result = try doSomething(); // ชัดเจนว่า function นี้อาจ error
const result2 = try doMore();     // ชัดเจนทุกจุดที่อาจ error
```

### ตัวอย่างแรก: Function ที่ return error

```zig
const std = @import("std");

// Function ที่อาจเกิด error
fn divide(a: i32, b: i32) !i32 {
    if (b == 0) {
        return error.DivisionByZero;
    }
    return a / b;
}

pub fn main() !void {
    // ต้องจัดการ error
    const result = divide(10, 2) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("Result: {}\n", .{result});
}
```

### ทำไม Zig ถึงเลือกแนวทางนี้?

Zig ถูกออกแบบมาสำหรับ systems programming ที่ต้องการ:
- **Predictability**: รู้ว่าโค้ดทำงานอย่างไรเมื่ออ่าน
- **Control**: ควบคุมได้ว่า error จะถูกจัดการอย่างไร
- **No surprises**: ไม่มี unexpected behavior จาก hidden exceptions

```zig
const std = @import("std");

// ตัวอย่างแสดงให้เห็นว่า error handling ใน Zig ชัดเจน
fn readConfig(path: []const u8) ![]const u8 {
    // ทุกจุดที่มี try คือจุดที่อาจ error
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    
    var buffer: [1024]u8 = undefined;
    const bytes_read = try file.readAll(&buffer);
    
    return buffer[0..bytes_read];
}

pub fn main() void {
    // เราเห็นชัดว่าต้องจัดการ error จาก readConfig
    const config = readConfig("config.txt") catch |err| switch (err) {
        error.FileNotFound => {
            std.debug.print("Config file not found, using defaults\n", .{});
            return;
        },
        error.AccessDenied => {
            std.debug.print("Permission denied\n", .{});
            return;
        },
        else => {
            std.debug.print("Unexpected error: {}\n", .{err});
            return;
        },
    };
    
    std.debug.print("Config: {s}\n", .{config});
}
```

---

## Step 172: Error Sets - การกำหนด Errors

### Error Set คืออะไร?

Error set เป็นกลุ่มของ error values ที่เราสามารถ return ได้จาก function มันคล้ายกับ enum แต่ถูกออกแบบมาสำหรับ errors โดยเฉพาะ

### การประกาศ Error Set

```zig
const std = @import("std");

// การประกาศ error set แบบพื้นฐาน
const MathError = error{
    DivisionByZero,
    Overflow,
    Underflow,
    NaN,
};

// การประกาศ error set สำหรับ file operations
const FileError = error{
    NotFound,
    AccessDenied,
    TooLarge,
    InvalidFormat,
    DiskFull,
};

// การประกาศ error set สำหรับ network operations
const NetworkError = error{
    ConnectionRefused,
    Timeout,
    HostNotFound,
    InvalidResponse,
    TooManyRedirects,
};
```

### การใช้ Error Sets ใน Functions

```zig
const std = @import("std");

const ParseError = error{
    InvalidCharacter,
    UnexpectedEof,
    NumberTooLarge,
    NumberTooSmall,
};

fn parsePositiveInt(str: []const u8) ParseError!u64 {
    if (str.len == 0) {
        return ParseError.UnexpectedEof;
    }
    
    var result: u64 = 0;
    for (str) |char| {
        if (char < '0' or char > '9') {
            return ParseError.InvalidCharacter;
        }
        
        // ตรวจสอบ overflow
        const digit = char - '0';
        if (result > (std.math.maxInt(u64) - digit) / 10) {
            return ParseError.NumberTooLarge;
        }
        
        result = result * 10 + digit;
    }
    
    return result;
}

pub fn main() void {
    const inputs = [_][]const u8{ "123", "456abc", "", "99999999999999999999" };
    
    for (inputs) |input| {
        const result = parsePositiveInt(input);
        if (result) |value| {
            std.debug.print("Parsed '{s}' = {}\n", .{ input, value });
        } else |err| {
            std.debug.print("Error parsing '{s}': {}\n", .{ input, err });
        }
    }
}
```

### Error Set Union - รวม Error Sets

```zig
const std = @import("std");

const ValidationError = error{
    TooShort,
    TooLong,
    InvalidCharacter,
};

const DatabaseError = error{
    ConnectionFailed,
    QueryFailed,
    RecordNotFound,
};

// รวม error sets เข้าด้วยกัน
const UserCreationError = ValidationError || DatabaseError;

fn validateUsername(name: []const u8) ValidationError!void {
    if (name.len < 3) return ValidationError.TooShort;
    if (name.len > 20) return ValidationError.TooLong;
    
    for (name) |char| {
        const is_valid = (char >= 'a' and char <= 'z') or
                         (char >= 'A' and char <= 'Z') or
                         (char >= '0' and char <= '9') or
                         char == '_';
        if (!is_valid) return ValidationError.InvalidCharacter;
    }
}

fn saveToDatabase(name: []const u8) DatabaseError!u64 {
    _ = name;
    // จำลองการ save ลง database
    // return DatabaseError.ConnectionFailed; // สมมติว่า connection ล้มเหลว
    return 12345; // return user ID
}

fn createUser(name: []const u8) UserCreationError!u64 {
    try validateUsername(name);
    const user_id = try saveToDatabase(name);
    return user_id;
}

pub fn main() void {
    const names = [_][]const u8{ "alice", "ab", "this_is_a_very_long_username_that_exceeds_limit", "hello world" };
    
    for (names) |name| {
        const result = createUser(name);
        if (result) |id| {
            std.debug.print("Created user '{s}' with ID: {}\n", .{ name, id });
        } else |err| {
            std.debug.print("Failed to create user '{s}': {}\n", .{ name, err });
        }
    }
}
```

### Inline Error Sets

```zig
const std = @import("std");

// สามารถใช้ inline error set ได้โดยตรง
fn riskyOperation() error{Failed, TimedOut}!void {
    // function body
    _ = undefined;
}

// หรือใช้แบบยาว
fn anotherOperation() (error{ NotReady, Busy })!i32 {
    return 42;
}

pub fn main() !void {
    try riskyOperation();
    const val = try anotherOperation();
    std.debug.print("Value: {}\n", .{val});
}
```

---

## Step 173: Error Union Type T!E

### Error Union คืออะไร?

`T!E` หรือ `E!T` (ใน Zig เขียนเป็น `E!T`) คือ type ที่อาจเป็น value ของ type T หรือ error ของ error set E

### รูปแบบการเขียน

```zig
const std = @import("std");

// รูปแบบต่างๆ ของ error union
// !T  -> anyerror!T (ใช้ global error set)
// E!T -> error set E กับ type T

fn getValueA() !i32 {
    return 42;  // anyerror!i32
}

fn getValueB() error{Failed}!i32 {
    return 42;  // error{Failed}!i32
}

fn getValueC() anyerror!i32 {
    return 42;  // explicit anyerror!i32
}
```

### การทำงานของ Error Union

```zig
const std = @import("std");

const MyError = error{
    SomethingWentWrong,
    AnotherProblem,
};

fn mayFail(succeed: bool) MyError!i32 {
    if (succeed) {
        return 100;
    } else {
        return MyError.SomethingWentWrong;
    }
}

pub fn main() void {
    // Error union เป็น type ที่สมบูรณ์
    const result1: MyError!i32 = mayFail(true);
    const result2: MyError!i32 = mayFail(false);
    
    // ตรวจสอบด้วย if
    if (result1) |value| {
        std.debug.print("Success: {}\n", .{value});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }
    
    if (result2) |value| {
        std.debug.print("Success: {}\n", .{value});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }
}
```

### ขนาดของ Error Union

```zig
const std = @import("std");

pub fn main() void {
    // ดูขนาดของ types ต่างๆ
    std.debug.print("Size of i32: {}\n", .{@sizeOf(i32)});
    std.debug.print("Size of anyerror: {}\n", .{@sizeOf(anyerror)});
    std.debug.print("Size of anyerror!i32: {}\n", .{@sizeOf(anyerror!i32)});
    
    // Error union มีขนาดเท่ากับ max(sizeof(T), sizeof(error)) + alignment
    // ซึ่งมักจะเป็น 8 bytes สำหรับ anyerror!i32
}
```

### การ Nest Error Unions

```zig
const std = @import("std");

const OuterError = error{OuterFail};
const InnerError = error{InnerFail};

// ไม่สามารถ nest error unions ได้โดยตรง
// fn badFunction() OuterError!(InnerError!i32) {} // Error!

// แต่สามารถรวม error sets ได้
fn goodFunction() (OuterError || InnerError)!i32 {
    return 42;
}

pub fn main() !void {
    const result = try goodFunction();
    std.debug.print("Result: {}\n", .{result});
}
```

### Error Union กับ Optional

```zig
const std = @import("std");

// ความแตกต่างระหว่าง optional และ error union
fn mayReturnNull() ?i32 {
    return null; // ไม่มีค่า แต่ไม่ใช่ error
}

fn mayFail() !i32 {
    return error.Failed; // มี error
}

fn mayReturnNullOrFail() !?i32 {
    return null; // ส่งคืน null (ไม่ใช่ error)
    // หรือ return error.Failed;
    // หรือ return 42;
}

pub fn main() !void {
    // จัดการ optional
    if (mayReturnNull()) |value| {
        std.debug.print("Got value: {}\n", .{value});
    } else {
        std.debug.print("Got null\n", .{});
    }
    
    // จัดการ error union
    const result = mayFail() catch |err| {
        std.debug.print("Got error: {}\n", .{err});
        return;
    };
    _ = result;
    
    // จัดการ error union ที่มี optional
    const complex = try mayReturnNullOrFail();
    if (complex) |value| {
        std.debug.print("Got value: {}\n", .{value});
    } else {
        std.debug.print("Got null (not an error)\n", .{});
    }
}
```

---

## Step 174: Try Operator - Propagate Error

### try คืออะไร?

`try` เป็น operator ที่:
1. ประเมินค่า expression ที่ return error union
2. ถ้าสำเร็จ: unwrap value และส่งคืน
3. ถ้า error: return error นั้นออกไปจาก current function

มันเหมือนกับ shorthand สำหรับ:
```zig
const result = expr catch |err| return err;
```

### การใช้ try พื้นฐาน

```zig
const std = @import("std");

fn step1() !i32 {
    return 10;
}

fn step2(n: i32) !i32 {
    if (n < 0) return error.NegativeInput;
    return n * 2;
}

fn step3(n: i32) ![]const u8 {
    if (n > 100) return error.TooBig;
    if (n == 20) return "twenty";
    return "other";
}

// ใช้ try เพื่อ propagate errors
fn doWork() !void {
    const a = try step1();        // ถ้า step1 error -> return error
    const b = try step2(a);       // ถ้า step2 error -> return error
    const c = try step3(b);       // ถ้า step3 error -> return error
    std.debug.print("Result: {s}\n", .{c});
}

pub fn main() !void {
    try doWork();
}
```

### try กับ Stack Trace

```zig
const std = @import("std");

fn deepFunction() !void {
    return error.DeepError;
}

fn middleFunction() !void {
    try deepFunction();  // ถ้า error จะ propagate ขึ้นมา
}

fn topFunction() !void {
    try middleFunction();
}

pub fn main() !void {
    topFunction() catch |err| {
        // เมื่อ error เกิดขึ้น Zig จะแสดง stack trace (ใน debug mode)
        std.debug.print("Caught error: {}\n", .{err});
        // stack trace จะแสดงว่า error เกิดที่ไหน
        return;
    };
}
```

### try ใน Real-World Example: File Processing

```zig
const std = @import("std");

const ProcessError = error{
    EmptyFile,
    InvalidHeader,
    ChecksumFailed,
};

fn readFileHeader(data: []const u8) ProcessError!u32 {
    if (data.len == 0) return ProcessError.EmptyFile;
    if (data.len < 4) return ProcessError.InvalidHeader;
    
    // อ่าน magic number จาก header
    const magic = std.mem.readInt(u32, data[0..4], .little);
    if (magic != 0xDEADBEEF) return ProcessError.InvalidHeader;
    
    return magic;
}

fn validateChecksum(data: []const u8) ProcessError!void {
    if (data.len < 8) return ProcessError.InvalidHeader;
    
    var sum: u32 = 0;
    for (data[0 .. data.len - 4]) |byte| {
        sum +%= byte;
    }
    
    const stored_checksum = std.mem.readInt(u32, data[data.len - 4 ..][0..4], .little);
    if (sum != stored_checksum) return ProcessError.ChecksumFailed;
}

fn processData(data: []const u8) ProcessError![]const u8 {
    // ใช้ try เพื่อ propagate errors อย่างสะดวก
    const magic = try readFileHeader(data);
    _ = magic;
    
    try validateChecksum(data);
    
    // Return actual data (ข้ามไป header และ checksum)
    return data[4 .. data.len - 4];
}

pub fn main() void {
    // สร้างข้อมูลทดสอบ
    var test_data = [_]u8{ 0xEF, 0xBE, 0xAD, 0xDE, 0x48, 0x65, 0x6C, 0x6C, 0x6F, 0x00, 0x00, 0x00, 0x00 };
    
    // คำนวณ checksum
    var sum: u32 = 0;
    for (test_data[0..9]) |byte| {
        sum +%= byte;
    }
    std.mem.writeInt(u32, test_data[9..13], sum, .little);
    
    const result = processData(&test_data);
    if (result) |payload| {
        std.debug.print("Data payload length: {}\n", .{payload.len});
    } else |err| {
        std.debug.print("Processing failed: {}\n", .{err});
    }
}
```

### try กับ Allocation

```zig
const std = @import("std");

fn createBuffer(allocator: std.mem.Allocator, size: usize) ![]u8 {
    // try จะ propagate error จาก allocator.alloc
    const buffer = try allocator.alloc(u8, size);
    // ถ้าถึงที่นี่แปลว่า allocation สำเร็จ
    @memset(buffer, 0);
    return buffer;
}

fn processMultipleBuffers(allocator: std.mem.Allocator) !void {
    // ใช้ try กับ multiple allocations
    const buf1 = try createBuffer(allocator, 100);
    defer allocator.free(buf1);
    
    const buf2 = try createBuffer(allocator, 200);
    defer allocator.free(buf2);
    
    const buf3 = try createBuffer(allocator, 300);
    defer allocator.free(buf3);
    
    std.debug.print("Allocated {} bytes total\n", .{buf1.len + buf2.len + buf3.len});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    try processMultipleBuffers(gpa.allocator());
}
```

---

## Step 175: Catch - Handle Error

### catch คืออะไร?

`catch` ใช้สำหรับจัดการ error ที่อาจเกิดขึ้น แทนที่จะ propagate ขึ้นไป

### รูปแบบต่างๆ ของ catch

```zig
const std = @import("std");

fn mayFail() !i32 {
    return error.SomeError;
}

pub fn main() void {
    // รูปแบบที่ 1: catch กับ default value
    const val1 = mayFail() catch 0;
    std.debug.print("val1: {}\n", .{val1});
    
    // รูปแบบที่ 2: catch กับ block
    const val2 = mayFail() catch {
        std.debug.print("Error occurred, using default\n", .{});
        break :catch 0;  // ไม่ต้องใช้ break ถ้าเป็น simple expression
    };
    _ = val2;
    
    // รูปแบบที่ 3: catch กับ error variable
    const val3 = mayFail() catch |err| blk: {
        std.debug.print("Caught error: {}\n", .{err});
        break :blk -1;
    };
    std.debug.print("val3: {}\n", .{val3});
    
    // รูปแบบที่ 4: catch กับ return
    const val4 = mayFail() catch |err| {
        std.debug.print("Fatal error: {}\n", .{err});
        return;
    };
    std.debug.print("val4: {}\n", .{val4});
}
```

### catch กับ Unreachable

```zig
const std = @import("std");

fn alwaysSucceeds() !i32 {
    return 42;  // เราแน่ใจว่าไม่เกิด error
}

pub fn main() void {
    // ถ้าเราแน่ใจ 100% ว่าไม่มี error สามารถใช้ unreachable ได้
    // แต่ต้องระวัง! ถ้าเกิด error ใน debug mode จะ panic
    const val = alwaysSucceeds() catch unreachable;
    std.debug.print("Value: {}\n", .{val});
}
```

### catch กับ Logging และ Recovery

```zig
const std = @import("std");

const AppError = error{
    DatabaseError,
    NetworkError,
    ValidationError,
};

fn fetchUserFromDB(user_id: u32) AppError![]const u8 {
    if (user_id == 0) return AppError.ValidationError;
    if (user_id == 999) return AppError.DatabaseError;
    return "John Doe";
}

fn fetchUserFromCache(user_id: u32) ?[]const u8 {
    if (user_id == 1) return "Cached John";
    return null;
}

pub fn main() void {
    const user_id: u32 = 999;
    
    // ใช้ catch สำหรับ fallback mechanism
    const user_name = fetchUserFromDB(user_id) catch |err| blk: {
        std.debug.print("DB Error: {}, trying cache...\n", .{err});
        
        // Fallback to cache
        if (fetchUserFromCache(user_id)) |cached| {
            break :blk cached;
        }
        
        // Default fallback
        std.debug.print("Cache miss, using guest\n", .{});
        break :blk "Guest";
    };
    
    std.debug.print("User: {s}\n", .{user_name});
}
```

### Real-World Example: Configuration Loading

```zig
const std = @import("std");

const ConfigError = error{
    FileNotFound,
    ParseError,
    MissingRequiredField,
};

const Config = struct {
    host: []const u8,
    port: u16,
    max_connections: u32,
};

fn loadConfigFromFile(path: []const u8) ConfigError!Config {
    _ = path;
    // จำลองการ load จาก file
    return ConfigError.FileNotFound;
}

fn loadConfigFromEnv() ConfigError!Config {
    // จำลองการ load จาก environment variables
    return ConfigError.MissingRequiredField;
}

fn getDefaultConfig() Config {
    return Config{
        .host = "localhost",
        .port = 8080,
        .max_connections = 100,
    };
}

pub fn main() void {
    // ลำดับการพยายาม load config
    const config = loadConfigFromFile("/etc/app/config.json") catch |file_err| blk: {
        std.debug.print("Failed to load from file: {}\n", .{file_err});
        
        break :blk loadConfigFromEnv() catch |env_err| {
            std.debug.print("Failed to load from env: {}\n", .{env_err});
            break :blk getDefaultConfig();
        };
    };
    
    std.debug.print("Config: host={s}, port={}, max_conn={}\n", .{
        config.host, config.port, config.max_connections,
    });
}
```

---

## Step 176: Catch กับ Switch

### การใช้ switch ใน catch block

```zig
const std = @import("std");

const HttpError = error{
    NotFound,
    Unauthorized,
    Forbidden,
    InternalServerError,
    ServiceUnavailable,
    Timeout,
    Unknown,
};

fn fetchResource(url: []const u8) HttpError![]const u8 {
    _ = url;
    return HttpError.NotFound;
}

pub fn main() void {
    const result = fetchResource("https://api.example.com/data") catch |err| switch (err) {
        HttpError.NotFound => {
            std.debug.print("Resource not found (404)\n", .{});
            return;
        },
        HttpError.Unauthorized => {
            std.debug.print("Please login first (401)\n", .{});
            return;
        },
        HttpError.Forbidden => {
            std.debug.print("Access denied (403)\n", .{});
            return;
        },
        HttpError.InternalServerError => {
            std.debug.print("Server error, please try again (500)\n", .{});
            return;
        },
        HttpError.ServiceUnavailable, HttpError.Timeout => {
            std.debug.print("Service temporarily unavailable\n", .{});
            return;
        },
        HttpError.Unknown => {
            std.debug.print("Unknown error occurred\n", .{});
            return;
        },
    };
    
    std.debug.print("Response: {s}\n", .{result});
}
```

### switch ด้วย else

```zig
const std = @import("std");

const DatabaseError = error{
    ConnectionFailed,
    QueryTimeout,
    ConstraintViolation,
    DuplicateKey,
    InvalidQuery,
    OutOfMemory,
};

fn executeQuery(query: []const u8) DatabaseError!u32 {
    _ = query;
    return DatabaseError.DuplicateKey;
}

pub fn main() void {
    const rows_affected = executeQuery("INSERT INTO users VALUES (1, 'Alice')") catch |err| switch (err) {
        DatabaseError.DuplicateKey => blk: {
            // จัดการ duplicate key อย่างพิเศษ
            std.debug.print("Record already exists, updating instead...\n", .{});
            break :blk @as(u32, 0); // ไม่มีแถวใหม่
        },
        DatabaseError.ConstraintViolation => blk: {
            std.debug.print("Data constraint violated\n", .{});
            break :blk @as(u32, 0);
        },
        else => blk: {
            // errors อื่นๆ ทั้งหมด
            std.debug.print("Unexpected database error: {}\n", .{err});
            break :blk @as(u32, 0);
        },
    };
    
    std.debug.print("Rows affected: {}\n", .{rows_affected});
}
```

### switch ที่ return value

```zig
const std = @import("std");

const ParseError = error{
    InvalidFormat,
    OutOfRange,
    UnknownType,
    EmptyInput,
};

fn parseColor(str: []const u8) ParseError!u32 {
    if (str.len == 0) return ParseError.EmptyInput;
    if (str.len != 7 or str[0] != '#') return ParseError.InvalidFormat;
    
    var result: u32 = 0;
    for (str[1..]) |char| {
        const nibble: u32 = switch (char) {
            '0'...'9' => char - '0',
            'a'...'f' => char - 'a' + 10,
            'A'...'F' => char - 'A' + 10,
            else => return ParseError.InvalidFormat,
        };
        result = (result << 4) | nibble;
    }
    
    return result;
}

pub fn main() void {
    const colors = [_][]const u8{ "#FF5733", "#invalid", "", "#GG0000" };
    
    for (colors) |color_str| {
        const color = parseColor(color_str) catch |err| switch (err) {
            ParseError.InvalidFormat => {
                std.debug.print("'{s}' is not a valid hex color\n", .{color_str});
                continue;
            },
            ParseError.EmptyInput => {
                std.debug.print("Empty string is not a valid color\n", .{});
                continue;
            },
            ParseError.OutOfRange, ParseError.UnknownType => {
                std.debug.print("Parse error for '{s}'\n", .{color_str});
                continue;
            },
        };
        
        std.debug.print("Color '{s}' = 0x{X:0>6}\n", .{ color_str, color });
    }
}
```

### Switch Pattern: Error Recovery Hierarchy

```zig
const std = @import("std");

const StorageError = error{
    PrimaryDown,
    SecondaryDown,
    AllReplicasFailed,
    DataCorrupted,
};

fn readFromPrimary(key: []const u8) StorageError![]const u8 {
    _ = key;
    return StorageError.PrimaryDown;
}

fn readFromSecondary(key: []const u8) StorageError![]const u8 {
    _ = key;
    return "data_from_secondary";
}

fn readFromBackup(key: []const u8) StorageError![]const u8 {
    _ = key;
    return "data_from_backup";
}

pub fn main() void {
    const key = "user_123";
    
    const data = readFromPrimary(key) catch |primary_err| switch (primary_err) {
        StorageError.PrimaryDown => blk: {
            std.debug.print("Primary down, trying secondary...\n", .{});
            break :blk readFromSecondary(key) catch |secondary_err| switch (secondary_err) {
                StorageError.SecondaryDown => {
                    std.debug.print("Secondary down, trying backup...\n", .{});
                    break :blk readFromBackup(key) catch {
                        std.debug.print("All replicas failed!\n", .{});
                        break :blk "DEFAULT_VALUE";
                    };
                },
                else => {
                    std.debug.print("Secondary error: {}\n", .{secondary_err});
                    break :blk "DEFAULT_VALUE";
                },
            };
        },
        StorageError.DataCorrupted => blk: {
            std.debug.print("Data corrupted! Initiating recovery...\n", .{});
            break :blk "RECOVERED_DATA";
        },
        else => blk: {
            std.debug.print("Unexpected error: {}\n", .{primary_err});
            break :blk "DEFAULT_VALUE";
        },
    };
    
    std.debug.print("Got data: {s}\n", .{data});
}
```

---

## Step 177: If กับ Error Union

### การใช้ if กับ error union

```zig
const std = @import("std");

fn divide(a: f64, b: f64) !f64 {
    if (b == 0.0) return error.DivisionByZero;
    return a / b;
}

pub fn main() void {
    // รูปแบบที่ 1: if-else กับ error union
    if (divide(10.0, 2.0)) |result| {
        std.debug.print("10 / 2 = {d}\n", .{result});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }
    
    // รูปแบบที่ 2: if กับ error union ที่ return error
    if (divide(10.0, 0.0)) |result| {
        std.debug.print("10 / 0 = {d}\n", .{result});
    } else |err| {
        std.debug.print("Cannot divide: {}\n", .{err});
    }
}
```

### if กับ Specific Error Matching

```zig
const std = @import("std");

const NetworkError = error{
    Timeout,
    ConnectionRefused,
    InvalidResponse,
    TooManyRequests,
};

fn httpGet(url: []const u8) NetworkError![]const u8 {
    _ = url;
    return NetworkError.Timeout;
}

pub fn main() void {
    if (httpGet("https://api.example.com")) |response| {
        std.debug.print("Response: {s}\n", .{response});
    } else |err| {
        // match กับ specific errors
        if (err == NetworkError.Timeout) {
            std.debug.print("Request timed out, please retry\n", .{});
        } else if (err == NetworkError.TooManyRequests) {
            std.debug.print("Rate limited, please wait\n", .{});
        } else {
            std.debug.print("Network error: {}\n", .{err});
        }
    }
}
```

### if กับ Multiple Operations

```zig
const std = @import("std");

const IOError = error{
    FileNotFound,
    PermissionDenied,
    DiskFull,
    InvalidData,
};

fn readConfig() IOError![]const u8 {
    return IOError.FileNotFound;
}

fn parseConfig(data: []const u8) IOError!u32 {
    _ = data;
    return 8080;
}

fn validatePort(port: u32) IOError!void {
    if (port < 1 or port > 65535) return IOError.InvalidData;
}

pub fn main() void {
    // chain ของ operations ด้วย if
    if (readConfig()) |config_data| {
        if (parseConfig(config_data)) |port| {
            if (validatePort(port)) {
                std.debug.print("Server will start on port {}\n", .{port});
            } else |err| {
                std.debug.print("Invalid port: {}\n", .{err});
            }
        } else |err| {
            std.debug.print("Failed to parse config: {}\n", .{err});
        }
    } else |err| {
        std.debug.print("Config not found: {}, using defaults\n", .{err});
    }
}
```

### if กับ Error Union และ Comptime

```zig
const std = @import("std");

fn processInput(input: []const u8) !i64 {
    if (input.len == 0) return error.EmptyInput;
    return std.fmt.parseInt(i64, input, 10);
}

pub fn main() void {
    const inputs = [_][]const u8{ "42", "-17", "abc", "", "9999999999999999999" };
    
    for (inputs) |input| {
        // ใช้ if กับ error union สำหรับแต่ละ input
        if (processInput(input)) |value| {
            std.debug.print("'{s}' -> {}\n", .{ input, value });
        } else |err| {
            std.debug.print("'{s}' -> Error: {s}\n", .{ input, @errorName(err) });
        }
    }
}
```

---

## Step 178: Error ใน Main Function

### Main Function กับ Error Handling

```zig
const std = @import("std");

// main สามารถ return !void ได้
pub fn main() !void {
    // ใช้ try ได้เลยใน main
    const result = try std.fmt.parseInt(i32, "42", 10);
    std.debug.print("Parsed: {}\n", .{result});
}
```

### Main กับ Error Reporting

```zig
const std = @import("std");

fn doActualWork() !void {
    return error.WorkFailed;
}

pub fn main() !void {
    // เมื่อ main return error, Zig จะ:
    // 1. Print error message ไปยัง stderr
    // 2. Return exit code ที่ไม่ใช่ 0
    try doActualWork();
}
```

เมื่อรัน program นี้จะเห็น:
```
error: WorkFailed
/path/to/file.zig:4:5: 0x... in doActualWork (main)
    return error.WorkFailed;
    ^
/path/to/file.zig:9:5: 0x... in main (main)
    try doActualWork();
    ^
```

### Main กับ Graceful Error Handling

```zig
const std = @import("std");

const AppError = error{
    InvalidArguments,
    ConfigurationError,
    RuntimeError,
};

fn parseArgs(args: []const []const u8) AppError!struct { verbose: bool, input: []const u8 } {
    if (args.len < 2) return AppError.InvalidArguments;
    
    var verbose = false;
    var input: ?[]const u8 = null;
    
    var i: usize = 1;
    while (i < args.len) : (i += 1) {
        if (std.mem.eql(u8, args[i], "--verbose") or std.mem.eql(u8, args[i], "-v")) {
            verbose = true;
        } else {
            input = args[i];
        }
    }
    
    if (input == null) return AppError.InvalidArguments;
    
    return .{ .verbose = verbose, .input = input.? };
}

pub fn main() void {
    // main ไม่ return error แต่จัดการ errors เอง
    const allocator = std.heap.page_allocator;
    
    const args = std.process.argsAlloc(allocator) catch |err| {
        std.io.getStdErr().writer().print("Failed to read arguments: {}\n", .{err}) catch {};
        std.process.exit(1);
    };
    defer std.process.argsFree(allocator, args);
    
    const parsed = parseArgs(args) catch |err| switch (err) {
        AppError.InvalidArguments => {
            std.io.getStdErr().writer().print("Usage: program [--verbose] <input>\n", .{}) catch {};
            std.process.exit(1);
        },
        else => {
            std.io.getStdErr().writer().print("Error: {}\n", .{err}) catch {};
            std.process.exit(1);
        },
    };
    
    if (parsed.verbose) {
        std.debug.print("Verbose mode enabled\n", .{});
    }
    std.debug.print("Input: {s}\n", .{parsed.input});
}
```

### Main กับ Error ที่ครบถ้วน

```zig
const std = @import("std");

pub fn main() !void {
    // ตัวอย่างการใช้งาน main กับ allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const leaked = gpa.deinit();
        if (leaked == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }
    
    const allocator = gpa.allocator();
    
    // ทำงานต่างๆ
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();
    
    try list.append(1);
    try list.append(2);
    try list.append(3);
    
    for (list.items) |item| {
        std.debug.print("{} ", .{item});
    }
    std.debug.print("\n", .{});
}
```

---

## Step 179: Anyerror - Global Error Set

### anyerror คืออะไร?

`anyerror` เป็น global error set ที่รวม errors ทั้งหมดในโปรแกรม มันเป็น superset ของ error sets ทั้งหมด

### เมื่อไหร่ควรใช้ anyerror?

```zig
const std = @import("std");

// ใช้ anyerror เมื่อไม่รู้ว่าจะเกิด error อะไร
fn callbackFunction(data: []const u8) anyerror!void {
    _ = data;
}

// Function ที่รับ callback ที่อาจเกิด error ใดก็ได้
fn processWithCallback(
    data: []const u8,
    callback: *const fn ([]const u8) anyerror!void,
) anyerror!void {
    try callback(data);
}

pub fn main() !void {
    try processWithCallback("hello", callbackFunction);
}
```

### ข้อดีและข้อเสียของ anyerror

```zig
const std = @import("std");

// ข้อเสีย: compiler ไม่สามารถตรวจสอบว่าเราจัดการ errors ครบหรือไม่
fn badPractice() anyerror!void {
    return error.SomeError;
}

// ข้อดี: flexible สำหรับ interfaces
const ErrorHandlerFn = *const fn (anyerror) void;

fn handleMyError(err: anyerror) void {
    std.debug.print("Got error: {s}\n", .{@errorName(err)});
}

fn doWorkWithHandler(handler: ErrorHandlerFn) void {
    const result = badPractice();
    if (result) |_| {
        std.debug.print("Success\n", .{});
    } else |err| {
        handler(err);
    }
}

pub fn main() void {
    doWorkWithHandler(handleMyError);
}
```

### anyerror กับ Error Coercion

```zig
const std = @import("std");

const MyError = error{
    Failed,
    NotReady,
};

fn specificError() MyError!void {
    return MyError.Failed;
}

fn anyErrorWrapper() anyerror!void {
    // MyError!void สามารถ coerce ไปเป็น anyerror!void ได้
    try specificError();
}

pub fn main() !void {
    // anyerror!void ก็สามารถใช้กับ try ได้
    anyErrorWrapper() catch |err| {
        std.debug.print("Error name: {s}\n", .{@errorName(err)});
    };
}
```

### @errorName และ @errorFromInt

```zig
const std = @import("std");

pub fn main() void {
    const err = error.MyCustomError;
    
    // ดูชื่อของ error
    std.debug.print("Error name: {s}\n", .{@errorName(err)});
    
    // ดู integer value ของ error (ใช้ใน advanced cases)
    const err_int = @intFromError(err);
    std.debug.print("Error int: {}\n", .{err_int});
    
    // แปลง integer กลับเป็น error
    const err_back = @errorFromInt(err_int);
    std.debug.print("Back to error: {s}\n", .{@errorName(err_back)});
}
```

### anyerror ใน Generic Functions

```zig
const std = @import("std");

// Generic function ที่ทำงานกับ error types ต่างๆ
fn retryN(
    comptime n: u32,
    func: *const fn () anyerror!void,
) anyerror!void {
    var last_err: anyerror = undefined;
    var i: u32 = 0;
    while (i < n) : (i += 1) {
        func() catch |err| {
            last_err = err;
            std.debug.print("Attempt {} failed: {s}\n", .{ i + 1, @errorName(err) });
            continue;
        };
        return; // success
    }
    return last_err;
}

var attempt_count: u32 = 0;

fn flakyOperation() anyerror!void {
    attempt_count += 1;
    if (attempt_count < 3) {
        return error.TemporaryFailure;
    }
    std.debug.print("Operation succeeded on attempt {}\n", .{attempt_count});
}

pub fn main() !void {
    attempt_count = 0;
    try retryN(5, flakyOperation);
}
```

---

## Step 180: Error เทียบกับ Exceptions ในภาษาอื่น

### เปรียบเทียบแนวคิดหลัก

| คุณสมบัติ | Zig Errors | Java Exceptions | Python Exceptions |
|-----------|-----------|-----------------|-------------------|
| Type safety | Compile-time checked | Runtime only | Runtime only |
| Performance | Zero overhead | Significant overhead | Significant overhead |
| Explicitness | Every error site marked | Hidden in call stacks | Hidden in call stacks |
| Control flow | No hidden jumps | Stack unwinding | Stack unwinding |
| Forcing handling | Yes, compiler error | Only checked exceptions | No enforcement |

### Java vs Zig

```java
// Java - Checked Exceptions
public class JavaExample {
    public static int divide(int a, int b) throws ArithmeticException {
        if (b == 0) throw new ArithmeticException("Division by zero");
        return a / b;
    }
    
    public static void main(String[] args) {
        try {
            int result = divide(10, 0);
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Error: " + e.getMessage());
        }
        // โปรแกรม continue ได้ต่อหลัง catch
    }
}
```

เทียบกับ Zig:
```zig
const std = @import("std");

// Zig - Error Values
fn divide(a: i32, b: i32) error{DivisionByZero}!i32 {
    if (b == 0) return error.DivisionByZero;
    return @divTrunc(a, b);
}

pub fn main() void {
    const result = divide(10, 0) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("Result: {}\n", .{result});
}
```

### Python vs Zig

```python
# Python - Exceptions
def read_file(path: str) -> str:
    try:
        with open(path, 'r') as f:
            return f.read()
    except FileNotFoundError:
        print("File not found")
        return ""
    except PermissionError:
        print("Permission denied")
        return ""
    except Exception as e:
        print(f"Unexpected error: {e}")
        return ""

# Problem: ไม่รู้จาก function signature ว่า function นี้อาจ throw อะไร
def process(path: str):
    data = read_file(path)  # ไม่ชัดเจนว่าอาจ fail
    return data.upper()
```

เทียบกับ Zig:
```zig
const std = @import("std");

const ReadError = error{
    FileNotFound,
    PermissionDenied,
    ReadError,
};

fn readFile(path: []const u8) ReadError![]const u8 {
    _ = path;
    return ReadError.FileNotFound;
}

// จาก function signature รู้ทันทีว่าอาจ fail ด้วย ReadError
fn process(path: []const u8) ReadError![]const u8 {
    const data = try readFile(path); // ชัดเจนว่า line นี้อาจ fail
    return data;
}

pub fn main() void {
    const result = process("data.txt") catch |err| switch (err) {
        ReadError.FileNotFound => {
            std.debug.print("File not found\n", .{});
            return;
        },
        ReadError.PermissionDenied => {
            std.debug.print("Permission denied\n", .{});
            return;
        },
        ReadError.ReadError => {
            std.debug.print("Read error\n", .{});
            return;
        },
    };
    std.debug.print("Data: {s}\n", .{result});
}
```

### Rust vs Zig

```rust
// Rust - Result type (คล้าย Zig)
use std::num::ParseIntError;

fn parse_and_double(s: &str) -> Result<i32, ParseIntError> {
    let n: i32 = s.parse()?;  // ? คล้าย try ใน Zig
    Ok(n * 2)
}

fn main() {
    match parse_and_double("21") {
        Ok(n) => println!("Result: {}", n),
        Err(e) => println!("Error: {}", e),
    }
}
```

เทียบกับ Zig:
```zig
const std = @import("std");

fn parseAndDouble(s: []const u8) !i32 {
    const n = try std.fmt.parseInt(i32, s, 10); // try คล้าย ? ใน Rust
    return n * 2;
}

pub fn main() void {
    if (parseAndDouble("21")) |result| {
        std.debug.print("Result: {}\n", .{result});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }
}
```

### ข้อดีของ Zig Error Handling

**1. Zero-cost Abstraction**

```zig
const std = @import("std");

// Zig error handling ไม่มี runtime overhead
// ไม่มีการสร้าง objects, ไม่มี stack unwinding
fn fastOperation() error{Failed}!i32 {
    return 42;
}

pub fn main() !void {
    // การ check error เทียบเท่ากับ C ที่ check return code
    const result = try fastOperation();
    std.debug.print("Result: {}\n", .{result});
}
```

**2. Exhaustive Error Handling**

```zig
const std = @import("std");

const StatusError = error{
    NotReady,
    Busy,
    Failed,
};

fn getStatus() StatusError!u32 {
    return StatusError.Busy;
}

pub fn main() void {
    // Compiler บังคับให้จัดการทุก error case
    const status = getStatus() catch |err| switch (err) {
        StatusError.NotReady => {
            std.debug.print("System not ready\n", .{});
            return;
        },
        StatusError.Busy => {
            std.debug.print("System busy\n", .{});
            return;
        },
        StatusError.Failed => {
            std.debug.print("System failed\n", .{});
            return;
        },
        // ถ้าลืม case ใด compiler จะ error
    };
    std.debug.print("Status: {}\n", .{status});
}
```

**3. No Hidden Control Flow**

```java
// Java - ไม่รู้ว่า exception จะ throw ที่ไหน
void processOrder(Order order) {
    validateOrder(order);      // อาจ throw ValidationException
    checkInventory(order);     // อาจ throw InventoryException  
    chargeCustomer(order);     // อาจ throw PaymentException
    shipOrder(order);          // อาจ throw ShippingException
    // ไม่ชัดเจนจากการอ่านโค้ด
}
```

```zig
// Zig - ชัดเจนทุกจุดที่อาจ error
fn processOrder(order: Order) !void {
    try validateOrder(order);   // ชัดว่า validate อาจ fail
    try checkInventory(order);  // ชัดว่า inventory check อาจ fail
    try chargeCustomer(order);  // ชัดว่า payment อาจ fail
    try shipOrder(order);       // ชัดว่า shipping อาจ fail
}
```

### Real-World Comparison: File Processing

```zig
const std = @import("std");

// ============================================================
// Zig approach - แบบ explicit และ exhaustive
// ============================================================

const FileProcessError = error{
    OpenFailed,
    ReadFailed,
    ParseFailed,
    InvalidData,
    WriteFailed,
};

fn processFile(
    allocator: std.mem.Allocator,
    input_path: []const u8,
    output_path: []const u8,
) FileProcessError!u64 {
    _ = allocator;
    _ = input_path;
    _ = output_path;
    
    // สมมติว่าทำงานสำเร็จ
    return 1024; // bytes processed
}

pub fn main() void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    const bytes = processFile(
        gpa.allocator(),
        "input.txt",
        "output.txt",
    ) catch |err| switch (err) {
        FileProcessError.OpenFailed => {
            std.debug.print("Cannot open file\n", .{});
            return;
        },
        FileProcessError.ReadFailed => {
            std.debug.print("Cannot read file\n", .{});
            return;
        },
        FileProcessError.ParseFailed => {
            std.debug.print("File format is invalid\n", .{});
            return;
        },
        FileProcessError.InvalidData => {
            std.debug.print("File contains invalid data\n", .{});
            return;
        },
        FileProcessError.WriteFailed => {
            std.debug.print("Cannot write output\n", .{});
            return;
        },
    };
    
    std.debug.print("Processed {} bytes successfully\n", .{bytes});
}
```

---

## Best Practices สำหรับ Error Handling ใน Zig

### 1. ตั้งชื่อ Error ให้มีความหมาย

```zig
// ไม่ดี
const Error = error{
    E1,
    E2,
    E3,
};

// ดี
const UserAuthError = error{
    InvalidCredentials,
    AccountLocked,
    SessionExpired,
    TwoFactorRequired,
};
```

### 2. ใช้ Error Sets ที่เฉพาะเจาะจง

```zig
// ไม่ดี - anyerror กว้างเกินไป
fn processData(data: []const u8) anyerror!void {
    _ = data;
}

// ดี - specific error set
const ProcessError = error{
    EmptyData,
    InvalidFormat,
    ProcessingFailed,
};

fn processData2(data: []const u8) ProcessError!void {
    if (data.len == 0) return ProcessError.EmptyData;
    _ = data;
}
```

### 3. Document Error Cases

```zig
const std = @import("std");

/// อ่าน configuration จาก file
/// 
/// Errors:
/// - `error.FileNotFound`: ถ้าไม่พบไฟล์ config
/// - `error.ParseError`: ถ้า format ของ config ไม่ถูกต้อง
/// - `error.MissingField`: ถ้าขาด required fields
fn loadConfig(path: []const u8) error{ FileNotFound, ParseError, MissingField }!void {
    _ = path;
}
```

### 4. Defer สำหรับ Cleanup

```zig
const std = @import("std");

fn processWithCleanup(allocator: std.mem.Allocator) !void {
    const buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer); // จะ free เสมอแม้เกิด error
    
    // ทำงานกับ buffer
    // ถ้าเกิด error ที่นี่ defer จะยังทำงาน
    try doSomethingWithBuffer(buffer);
}

fn doSomethingWithBuffer(buf: []u8) !void {
    _ = buf;
    return error.Failed;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    processWithCleanup(gpa.allocator()) catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### 5. Error Wrapping สำหรับ Context

```zig
const std = @import("std");

const AppError = error{
    DatabaseError,
    NetworkError,
    ConfigError,
};

fn connectToDatabase() error{ConnectionFailed}!void {
    return error.ConnectionFailed;
}

fn initializeApp() AppError!void {
    connectToDatabase() catch {
        // แปลง low-level error เป็น app-level error
        return AppError.DatabaseError;
    };
}

pub fn main() void {
    initializeApp() catch |err| {
        std.debug.print("App init failed: {}\n", .{err});
        return;
    };
    std.debug.print("App started successfully\n", .{});
}
```

---

## ตัวอย่างโปรเจกต์จริง: Simple JSON-like Parser

```zig
const std = @import("std");

const ParseError = error{
    UnexpectedCharacter,
    UnexpectedEof,
    NumberOverflow,
    InvalidEscape,
    StringTooLong,
    NestingTooDeep,
};

const ValueType = enum {
    null_value,
    boolean,
    integer,
    string,
};

const Value = union(ValueType) {
    null_value: void,
    boolean: bool,
    integer: i64,
    string: []const u8,
};

const Parser = struct {
    input: []const u8,
    pos: usize,
    depth: u32,
    
    const MAX_DEPTH = 64;
    const MAX_STRING = 4096;
    
    fn init(input: []const u8) Parser {
        return .{ .input = input, .pos = 0, .depth = 0 };
    }
    
    fn peek(self: *Parser) ParseError!u8 {
        if (self.pos >= self.input.len) return ParseError.UnexpectedEof;
        return self.input[self.pos];
    }
    
    fn advance(self: *Parser) ParseError!u8 {
        const c = try self.peek();
        self.pos += 1;
        return c;
    }
    
    fn skipWhitespace(self: *Parser) void {
        while (self.pos < self.input.len) {
            switch (self.input[self.pos]) {
                ' ', '\t', '\n', '\r' => self.pos += 1,
                else => break,
            }
        }
    }
    
    fn parseNull(self: *Parser) ParseError!Value {
        if (self.pos + 4 > self.input.len) return ParseError.UnexpectedEof;
        if (!std.mem.eql(u8, self.input[self.pos .. self.pos + 4], "null")) {
            return ParseError.UnexpectedCharacter;
        }
        self.pos += 4;
        return Value{ .null_value = {} };
    }
    
    fn parseBool(self: *Parser) ParseError!Value {
        if (self.pos + 4 <= self.input.len and 
            std.mem.eql(u8, self.input[self.pos .. self.pos + 4], "true")) {
            self.pos += 4;
            return Value{ .boolean = true };
        }
        if (self.pos + 5 <= self.input.len and
            std.mem.eql(u8, self.input[self.pos .. self.pos + 5], "false")) {
            self.pos += 5;
            return Value{ .boolean = false };
        }
        return ParseError.UnexpectedCharacter;
    }
    
    fn parseNumber(self: *Parser) ParseError!Value {
        const start = self.pos;
        var negative = false;
        
        if (self.pos < self.input.len and self.input[self.pos] == '-') {
            negative = true;
            self.pos += 1;
        }
        
        if (self.pos >= self.input.len or self.input[self.pos] < '0' or self.input[self.pos] > '9') {
            return ParseError.UnexpectedCharacter;
        }
        
        var result: i64 = 0;
        while (self.pos < self.input.len and self.input[self.pos] >= '0' and self.input[self.pos] <= '9') {
            const digit = self.input[self.pos] - '0';
            if (result > (std.math.maxInt(i64) - digit) / 10) {
                return ParseError.NumberOverflow;
            }
            result = result * 10 + digit;
            self.pos += 1;
        }
        
        _ = start;
        return Value{ .integer = if (negative) -result else result };
    }
    
    fn parseString(self: *Parser) ParseError!Value {
        // ข้าม opening quote
        const first = try self.advance();
        if (first != '"') return ParseError.UnexpectedCharacter;
        
        const start = self.pos;
        var len: usize = 0;
        
        while (true) {
            const c = try self.advance();
            if (c == '"') break;
            if (c == '\\') _ = try self.advance(); // escape sequence
            len += 1;
            if (len > MAX_STRING) return ParseError.StringTooLong;
        }
        
        return Value{ .string = self.input[start .. self.pos - 1] };
    }
    
    fn parseValue(self: *Parser) ParseError!Value {
        self.skipWhitespace();
        
        if (self.depth >= MAX_DEPTH) return ParseError.NestingTooDeep;
        
        const c = try self.peek();
        return switch (c) {
            'n' => try self.parseNull(),
            't', 'f' => try self.parseBool(),
            '0'...'9', '-' => try self.parseNumber(),
            '"' => try self.parseString(),
            else => ParseError.UnexpectedCharacter,
        };
    }
};

pub fn main() void {
    const test_inputs = [_][]const u8{
        "null",
        "true",
        "false",
        "42",
        "-17",
        "\"hello world\"",
        "xyz",
        "",
    };
    
    for (test_inputs) |input| {
        var parser = Parser.init(input);
        const result = parser.parseValue();
        
        if (result) |value| {
            switch (value) {
                .null_value => std.debug.print("'{s}' = null\n", .{input}),
                .boolean => |b| std.debug.print("'{s}' = bool:{}\n", .{ input, b }),
                .integer => |n| std.debug.print("'{s}' = int:{}\n", .{ input, n }),
                .string => |s| std.debug.print("'{s}' = string:\"{s}\"\n", .{ input, s }),
            }
        } else |err| {
            std.debug.print("'{s}' = Error: {s}\n", .{ input, @errorName(err) });
        }
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Error Set Design

ออกแบบ error sets สำหรับระบบ online shopping:
- `ProductError`: errors เกี่ยวกับสินค้า
- `CartError`: errors เกี่ยวกับตะกร้าสินค้า  
- `PaymentError`: errors เกี่ยวกับการชำระเงิน
- `OrderError`: errors เกี่ยวกับการสั่งซื้อ (รวมทั้งหมด)

```zig
// TODO: ออกแบบ error sets และ functions ต่อไปนี้
// fn getProduct(id: u32) ProductError!Product
// fn addToCart(cart: *Cart, product: Product) CartError!void
// fn checkout(cart: Cart, payment: PaymentInfo) PaymentError!OrderId
// fn createOrder(cart: Cart, payment: PaymentInfo) OrderError!OrderId
```

### แบบฝึกหัดที่ 2: Error Recovery

เขียน function ที่:
1. พยายามอ่าน config จาก file
2. ถ้าไม่ได้ ให้อ่านจาก environment variables
3. ถ้าไม่ได้ ให้ใช้ default values
4. Log แต่ละ step ที่ fail

```zig
// TODO: implement config loading with fallback
const Config = struct {
    db_host: []const u8,
    db_port: u16,
    log_level: []const u8,
};

fn loadConfig(allocator: std.mem.Allocator) !Config {
    // ใช้ catch สำหรับ fallback
    _ = allocator;
    return error.NotImplemented;
}
```

### แบบฝึกหัดที่ 3: CSV Parser

เขียน simple CSV parser ที่จัดการ errors:

```zig
const std = @import("std");

const CsvError = error{
    InvalidFormat,
    TooManyColumns,
    TooFewColumns,
    InvalidNumber,
    EmptyInput,
};

// TODO: implement CSV parser
fn parseCsvLine(line: []const u8, expected_cols: usize) CsvError![][]const u8 {
    _ = line;
    _ = expected_cols;
    return CsvError.InvalidFormat;
}
```

### แบบฝึกหัดที่ 4: Retry Logic

เขียน retry function ที่:
- ลองทำ operation สูงสุด N ครั้ง
- รอระหว่าง retries (จำลองด้วย counter)
- Return error ล่าสุดถ้าทุกครั้ง fail

```zig
// TODO: implement retry with exponential backoff
fn withRetry(
    comptime max_attempts: u32,
    comptime ReturnType: type,
    func: *const fn () anyerror!ReturnType,
) anyerror!ReturnType {
    _ = func;
    return error.NotImplemented;
}
```

### แบบฝึกหัดที่ 5: Error Context

เพิ่ม context ให้กับ errors โดยใช้ struct:

```zig
const std = @import("std");

// TODO: สร้าง ErrorWithContext struct
const ErrorWithContext = struct {
    err: anyerror,
    context: []const u8,
    file: []const u8,
    line: u32,
    
    fn format(
        self: ErrorWithContext,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{s}:{} - {s}: {s}", .{
            self.file, self.line, self.context, @errorName(self.err),
        });
    }
};
```

---

## สรุป

ใน Part 18 นี้เราได้เรียนรู้:

### สิ่งที่เราเรียนรู้

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|---------------|
| Philosophy | Errors เป็น values, explicit, no hidden control flow |
| Error Sets | การประกาศและใช้งาน error sets, union ของ error sets |
| Error Union (T!E) | รูปแบบ syntax, วิธีทำงาน, ความสัมพันธ์กับ optional |
| try operator | propagate errors อย่างสะดวก, เทียบเท่า catch + return |
| catch | จัดการ errors ด้วย default values, blocks, switches |
| if กับ error union | รูปแบบ if-else สำหรับ unwrap error union |
| Error ใน main | วิธีใช้ !void ใน main, graceful error handling |
| anyerror | global error set, เมื่อไหร่ควรใช้ |
| เทียบกับ exceptions | ข้อดีของ Zig approach เทียบกับ Java/Python/Rust |

### Key Takeaways

1. **Errors ใน Zig เป็น values** - ไม่ใช่ exceptions ที่ถูก throw
2. **try = catch + return** - shorthand สำหรับ error propagation
3. **Compiler enforces handling** - ไม่สามารถ ignore errors ได้
4. **No hidden control flow** - ทุก error site ชัดเจนจากโค้ด
5. **Zero runtime overhead** - ไม่มี cost เพิ่มเติมจาก mechanism
6. **anyerror ใช้เมื่อจำเป็น** - ควรใช้ specific error sets เมื่อทำได้

### Anti-patterns ที่ควรหลีกเลี่ยง

```zig
// 1. อย่าใช้ unreachable โดยไม่แน่ใจ
// const result = risky() catch unreachable; // อันตราย!

// 2. อย่า ignore errors ด้วย _ 
// _ = risky(); // Zig ไม่อนุญาตอยู่แล้ว

// 3. อย่าใช้ anyerror เมื่อไม่จำเป็น
// fn myFunc() anyerror!void {} // ควรใช้ specific error set

// 4. อย่า swallow errors โดยไม่ log
// const result = risky() catch 0; // อย่าทำแบบนี้ถ้าไม่ log
```

### ถัดไป: Part 19

ใน Part 19 เราจะเรียนรู้เรื่อง:
- Advanced error handling patterns
- Error handling กับ comptime
- Custom error types ที่มี payload
- Error handling ใน concurrent code
- Testing error conditions

---

*Part 18 เสร็จสมบูรณ์ - ยินดีด้วย! คุณเข้าใจ error handling พื้นฐานใน Zig แล้ว*
