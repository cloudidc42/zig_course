# ส่วนที่ 19: Error Handling ขั้นสูงใน Zig (Steps 181-190)

## บทนำ

การจัดการข้อผิดพลาด (Error Handling) เป็นหัวใจสำคัญของการเขียนโปรแกรมที่มีคุณภาพ Zig มีระบบ error handling ที่ทรงพลังและปลอดภัยอย่างยิ่ง โดยทำให้ผู้เขียนโปรแกรมต้องจัดการกับข้อผิดพลาดอย่างชัดเจน ไม่ใช่การซ่อนหรือเพิกเฉย

ในส่วนนี้เราจะเจาะลึกเทคนิคขั้นสูงของ error handling ใน Zig ตั้งแต่การรวม error sets การอ้างอิง error traces การสร้าง error messages ที่มีความหมาย ไปจนถึง patterns ที่ใช้จริงในโปรแกรมระดับ production

---

## Step 181: Error Sets - Merging และ Subsets

### ทำความเข้าใจ Error Sets

ใน Zig ข้อผิดพลาดถูกจัดกลุ่มเป็น "error sets" ซึ่งเป็นชุดของค่า error ที่กำหนดไว้ล่วงหน้า Error sets ทำงานคล้าย enum แต่มีวัตถุประสงค์เฉพาะสำหรับการรายงานข้อผิดพลาด

```zig
const std = @import("std");

// Error set พื้นฐาน
const FileError = error{
    NotFound,
    PermissionDenied,
    IsDirectory,
    TooLarge,
};

const NetworkError = error{
    ConnectionRefused,
    Timeout,
    DNSFailed,
    TooManyConnections,
};

pub fn main() void {
    // สร้าง error value
    const err: FileError = FileError.NotFound;
    std.debug.print("Error: {}\n", .{err});
}
```

### การรวม Error Sets (Merging)

Zig ให้ผสาน error sets หลายชุดเข้าด้วยกันโดยใช้ operator `||`

```zig
const std = @import("std");

const FileError = error{
    NotFound,
    PermissionDenied,
    IsDirectory,
};

const NetworkError = error{
    ConnectionRefused,
    Timeout,
    DNSFailed,
};

const ParseError = error{
    InvalidFormat,
    UnexpectedEOF,
    TooLarge,
};

// รวม error sets หลายชุด
const AppError = FileError || NetworkError || ParseError;

// ฟังก์ชันที่อาจเกิด error จากหลายแหล่ง
fn downloadAndParseConfig(url: []const u8) AppError![]const u8 {
    _ = url;
    // จำลองข้อผิดพลาดจากเครือข่าย
    return error.Timeout;
}

pub fn main() void {
    const result = downloadAndParseConfig("https://example.com/config.json");
    if (result) |config| {
        std.debug.print("Config: {s}\n", .{config});
    } else |err| {
        std.debug.print("Failed: {}\n", .{err});
    }
}
```

### Error Set Subsets

Error set หนึ่งสามารถเป็น subset ของอีก error set ได้ Zig จะตรวจสอบความสัมพันธ์นี้ตอน compile time

```zig
const std = @import("std");

const FullError = error{
    OutOfMemory,
    InvalidInput,
    NotFound,
    PermissionDenied,
    Timeout,
    IOError,
};

// Subset - มีเพียงบางส่วนของ FullError
const SimpleError = error{
    OutOfMemory,
    InvalidInput,
};

// ฟังก์ชันที่ใช้ error set เต็ม
fn complexOperation() FullError!void {
    return error.NotFound;
}

// ฟังก์ชันที่ใช้เฉพาะ error บางชนิด
fn simpleOperation() SimpleError!void {
    return error.OutOfMemory;
}

// Coerce จาก subset ไป superset ได้อัตโนมัติ
fn wrapSimple() FullError!void {
    try simpleOperation(); // SimpleError coerces to FullError
}

pub fn main() void {
    wrapSimple() catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### Error Set เป็น First-Class Types

```zig
const std = @import("std");

const DatabaseError = error{
    ConnectionFailed,
    QueryFailed,
    TransactionRollback,
    DeadlockDetected,
    DataCorruption,
};

const CacheError = error{
    KeyNotFound,
    SerializationFailed,
    CacheEviction,
};

// รวม error sets และสร้าง type ใหม่
const StorageError = DatabaseError || CacheError;

// ฟังก์ชันที่รับ error set type
fn handleStorageError(err: StorageError) void {
    switch (err) {
        error.ConnectionFailed => std.debug.print("ไม่สามารถเชื่อมต่อฐานข้อมูลได้\n", .{}),
        error.QueryFailed => std.debug.print("คำสั่ง SQL ผิดพลาด\n", .{}),
        error.TransactionRollback => std.debug.print("Transaction ถูก rollback\n", .{}),
        error.DeadlockDetected => std.debug.print("พบ Deadlock\n", .{}),
        error.DataCorruption => std.debug.print("ข้อมูลเสียหาย\n", .{}),
        error.KeyNotFound => std.debug.print("ไม่พบ key ใน cache\n", .{}),
        error.SerializationFailed => std.debug.print("ไม่สามารถแปลงข้อมูลได้\n", .{}),
        error.CacheEviction => std.debug.print("ข้อมูลถูกลบออกจาก cache\n", .{}),
    }
}

pub fn main() void {
    const err: StorageError = error.DeadlockDetected;
    handleStorageError(err);
}
```

### ข้อควรระวังเกี่ยวกับ Error Sets

```zig
const std = @import("std");

// Error set ที่ซ้อนกัน - Zig รองรับ
const LevelOneError = error{Alpha, Beta};
const LevelTwoError = error{Gamma, Delta} || LevelOneError;
const LevelThreeError = error{Epsilon} || LevelTwoError;

fn deepFunction() LevelThreeError!void {
    return error.Alpha; // Alpha มาจาก LevelOneError
}

fn midFunction() LevelThreeError!void {
    try deepFunction();
}

fn topFunction() LevelThreeError!void {
    try midFunction();
}

pub fn main() void {
    topFunction() catch |err| {
        std.debug.print("ได้รับ error: {s}\n", .{@errorName(err)});
    };
}
```

---

## Step 182: Inferred Error Sets (เขียน Error Set อัตโนมัติ)

### ทำความเข้าใจ Inferred Error Sets

แทนที่จะระบุ error set อย่างชัดเจน Zig สามารถ "อนุมาน" error set ได้จากเนื้อหาของฟังก์ชัน โดยใช้ `!` โดยไม่ระบุชื่อ error set

```zig
const std = @import("std");

// แบบไม่ระบุ error set (inferred)
fn openFile(path: []const u8) ![]const u8 {
    _ = path;
    // Zig จะอนุมาน error set จากการใช้งาน
    return error.NotFound;
}

// แบบระบุ error set อย่างชัดเจน (explicit)
const OpenError = error{ NotFound, PermissionDenied };
fn openFileExplicit(path: []const u8) OpenError![]const u8 {
    _ = path;
    return error.NotFound;
}

pub fn main() void {
    // ทั้งสองแบบใช้งานเหมือนกัน
    const r1 = openFile("test.txt");
    const r2 = openFileExplicit("test.txt");
    
    _ = r1 catch {};
    _ = r2 catch {};
}
```

### ข้อดีและข้อเสียของ Inferred Error Sets

```zig
const std = @import("std");

// ข้อดี: ไม่ต้องระบุ error set ซ้ำๆ
// ฟังก์ชันที่ใช้ allocator มักมี error sets ซับซ้อน
fn processData(allocator: std.mem.Allocator, data: []const u8) ![]u8 {
    // Zig อนุมานว่าฟังก์ชันนี้อาจเกิด OutOfMemory จาก allocate
    const result = try allocator.alloc(u8, data.len * 2);
    
    for (data, 0..) |byte, i| {
        result[i * 2] = byte;
        result[i * 2 + 1] = byte;
    }
    
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const input = "Hello";
    const output = try processData(allocator, input);
    defer allocator.free(output);
    
    std.debug.print("ผลลัพธ์: {s}\n", .{output});
}
```

### ข้อจำกัดของ Inferred Error Sets

```zig
const std = @import("std");

// ข้อเสีย: ไม่สามารถใช้ inferred error set กับ recursive functions ได้
// โค้ดนี้จะ compile ไม่ผ่าน:
// fn countDown(n: u32) !void {
//     if (n == 0) return error.Done;
//     try countDown(n - 1); // error: recursive function cannot use inferred error set
// }

// วิธีแก้: ระบุ error set อย่างชัดเจน
const CountError = error{Done};
fn countDown(n: u32) CountError!void {
    if (n == 0) return error.Done;
    try countDown(n - 1);
}

// ข้อเสีย: inferred error set ไม่สามารถใช้กับ function pointers ได้โดยตรง
const ExplicitError = error{ InvalidInput, OutOfRange };
const FnWithError = *const fn (x: i32) ExplicitError!i32;

fn doublePositive(x: i32) ExplicitError!i32 {
    if (x < 0) return error.InvalidInput;
    if (x > 1000) return error.OutOfRange;
    return x * 2;
}

pub fn main() void {
    // ใช้ function pointer กับ explicit error set
    const fn_ptr: FnWithError = &doublePositive;
    const result = fn_ptr(42) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("ผลลัพธ์: {}\n", .{result});
    
    // ทดสอบ recursive function
    countDown(3) catch |err| {
        std.debug.print("Count done: {}\n", .{err});
    };
}
```

### Inferred vs Explicit Error Sets - เมื่อควรใช้อะไร

```zig
const std = @import("std");

// กรณีที่ควรใช้ Inferred Error Sets:
// 1. ฟังก์ชันใน implementation (ไม่ใช่ public API)
// 2. ฟังก์ชันที่ call หลาย operations ที่มี error sets ซับซ้อน
fn internalProcess(allocator: std.mem.Allocator, n: usize) ![]u8 {
    const buf = try allocator.alloc(u8, n);
    @memset(buf, 0);
    return buf;
}

// กรณีที่ควรใช้ Explicit Error Sets:
// 1. Public API ที่ต้องการ documentation ชัดเจน
// 2. ฟังก์ชันที่ recursive
// 3. ฟังก์ชันที่ใช้กับ function pointers
// 4. เมื่อต้องการจำกัดว่าจะ propagate error อะไรบ้าง
const PublicError = error{
    InvalidInput,
    OutOfMemory,
    NotFound,
};

pub fn publicApiFunction(allocator: std.mem.Allocator, key: []const u8) PublicError![]u8 {
    if (key.len == 0) return error.InvalidInput;
    
    const result = allocator.alloc(u8, key.len) catch return error.OutOfMemory;
    @memcpy(result, key);
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ใช้งาน public API
    const result = try publicApiFunction(allocator, "hello");
    defer allocator.free(result);
    std.debug.print("ผลลัพธ์: {s}\n", .{result});
}
```

---

## Step 183: Error Return Traces

### ทำความเข้าใจ Error Return Traces

Error Return Trace เป็นฟีเจอร์พิเศษของ Zig ที่ช่วยติดตามว่า error เกิดขึ้นที่ไหนและถูก propagate ผ่านฟังก์ชันอะไรบ้าง

```zig
const std = @import("std");

fn level3() !void {
    return error.SomethingWentWrong;
}

fn level2() !void {
    try level3(); // error ถูก propagate จาก level3
}

fn level1() !void {
    try level2(); // error ถูก propagate จาก level2
}

pub fn main() !void {
    // เมื่อเรียก level1() และเกิด error
    // Zig จะแสดง error return trace ที่แสดง stack ของการ propagation
    try level1();
}
```

เมื่อรันโปรแกรมนี้ จะเห็น output ประมาณนี้:
```
error: SomethingWentWrong
/path/to/file.zig:4:5: 0x... in level3 (file)
/path/to/file.zig:8:5: 0x... in level2 (file)
/path/to/file.zig:12:5: 0x... in level1 (file)
/path/to/file.zig:16:5: 0x... in main (file)
```

### การใช้ Error Return Traces เพื่อ Debug

```zig
const std = @import("std");

const ConfigError = error{
    FileNotFound,
    ParseError,
    InvalidValue,
    MissingRequired,
};

fn validatePort(port: i64) ConfigError!void {
    if (port < 1 or port > 65535) {
        return error.InvalidValue; // Error เกิดที่นี่
    }
}

fn parsePort(value: []const u8) ConfigError!i64 {
    // จำลองการ parse
    _ = value;
    const port: i64 = 99999; // ค่าผิดพลาดสำหรับทดสอบ
    try validatePort(port); // Error ถูก propagate
    return port;
}

fn loadConfig(path: []const u8) ConfigError!void {
    _ = path;
    const port_str = "99999";
    _ = try parsePort(port_str); // Error ถูก propagate อีกครั้ง
}

fn initApplication() ConfigError!void {
    try loadConfig("config.json"); // Error ถูก propagate ไปถึงนี่
}

pub fn main() void {
    initApplication() catch |err| {
        std.debug.print("ไม่สามารถเริ่มต้นแอปพลิเคชันได้: {}\n", .{err});
        // Error return trace จะแสดงใน debug mode
    };
}
```

### การควบคุม Error Traces

```zig
const std = @import("std");

// ใน release mode error traces ถูกปิดโดยอัตโนมัติ
// ใน debug mode error traces เปิดอยู่เสมอ

fn riskyOperation() !i32 {
    // @errorReturnTrace() คืน ?*std.builtin.StackTrace
    // เพื่อดู trace ปัจจุบัน
    if (@errorReturnTrace()) |trace| {
        _ = trace;
        // สามารถใช้ trace สำหรับ logging ขั้นสูง
    }
    return error.Unexpected;
}

pub fn main() void {
    const result = riskyOperation();
    if (result) |val| {
        std.debug.print("ได้รับค่า: {}\n", .{val});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }
}
```

### Error Traces ในระบบจริง

```zig
const std = @import("std");

// ในระบบจริง เราอาจต้องการจัดการ error trace
fn processRequest(request_id: u64) !void {
    _ = request_id;
    return error.ServiceUnavailable;
}

fn handleRequest(request_id: u64) void {
    processRequest(request_id) catch |err| {
        // ใน debug mode จะมี error return trace อัตโนมัติ
        std.debug.print("[Request {}] Error: {s}\n", .{
            request_id,
            @errorName(err),
        });
        
        // สามารถดู stack trace ได้ใน debug mode
        if (@errorReturnTrace()) |trace| {
            std.debug.dumpStackTrace(trace.*);
        }
    };
}

pub fn main() void {
    handleRequest(12345);
    handleRequest(99999);
}
```

---

## Step 184: @errorName และ @errorCast

### @errorName - แปลง Error เป็น String

`@errorName` เป็น builtin function ที่แปลง error value เป็น string ชื่อของ error นั้น

```zig
const std = @import("std");

const DatabaseError = error{
    ConnectionTimeout,
    InvalidQuery,
    TransactionFailed,
    DataNotFound,
};

pub fn main() void {
    const err: DatabaseError = error.ConnectionTimeout;
    
    // @errorName คืนค่า []const u8
    const name: []const u8 = @errorName(err);
    std.debug.print("ชื่อ error: {s}\n", .{name});
    // Output: ชื่อ error: ConnectionTimeout
    
    // ใช้ใน logging
    const errors = [_]DatabaseError{
        error.ConnectionTimeout,
        error.InvalidQuery,
        error.TransactionFailed,
        error.DataNotFound,
    };
    
    for (errors) |e| {
        std.debug.print("Error: {s}\n", .{@errorName(e)});
    }
}
```

### ใช้ @errorName สำหรับ Error Logging

```zig
const std = @import("std");

const AppError = error{
    OutOfMemory,
    InvalidInput,
    NetworkFailure,
    ParseError,
    AuthenticationFailed,
};

// Logger อย่างง่าย
const Logger = struct {
    fn logError(context: []const u8, err: anyerror) void {
        // anyerror คือ global error set ที่รวมทุก error
        std.debug.print("[ERROR] {s}: {s}\n", .{
            context,
            @errorName(err),
        });
    }
    
    fn logErrorWithCode(context: []const u8, err: anyerror, code: u32) void {
        std.debug.print("[ERROR] {s} (code={d}): {s}\n", .{
            context,
            code,
            @errorName(err),
        });
    }
};

fn authenticate(username: []const u8, password: []const u8) AppError!void {
    if (username.len == 0) return error.InvalidInput;
    if (password.len < 8) return error.AuthenticationFailed;
}

pub fn main() void {
    authenticate("", "pass") catch |err| {
        Logger.logError("authenticate", err);
    };
    
    authenticate("user", "short") catch |err| {
        Logger.logErrorWithCode("authenticate", err, 401);
    };
    
    // ทดสอบกับ anyerror
    const any_err: anyerror = error.NetworkFailure;
    Logger.logError("network", any_err);
}
```

### @errorCast - แปลงระหว่าง Error Sets

`@errorCast` ใช้เมื่อต้องการแปลง error จาก error set ที่ใหญ่กว่าไปยัง error set ที่เล็กกว่า

```zig
const std = @import("std");

const SmallErrorSet = error{
    NotFound,
    OutOfMemory,
};

const LargeErrorSet = error{
    NotFound,
    OutOfMemory,
    NetworkError,
    ParseError,
    PermissionDenied,
};

fn getLargeError() LargeErrorSet!void {
    return error.NotFound;
}

pub fn main() void {
    // แปลงจาก large ไป small (ต้องรู้ว่า error นั้นมีใน small set)
    const result = getLargeError();
    if (result) |_| {
        // ไม่มีค่า
    } else |err| {
        // @errorCast ทำการ cast error จาก large set ไป small set
        // หาก error ไม่มีใน small set จะเกิด panic ใน debug mode
        const small_err = @errorCast(err); // แปลงเป็น SmallErrorSet
        _ = small_err;
        std.debug.print("Error ถูก cast สำเร็จ: {s}\n", .{@errorName(err)});
    }
}
```

### anyerror - Global Error Type

```zig
const std = @import("std");

// anyerror คือ error set ที่รวมทุก error ในโปรแกรม
// มีประโยชน์เมื่อต้องการรับ error ทุกชนิด

fn processAnyError(err: anyerror) void {
    // เปรียบเทียบกับ specific errors
    if (err == error.OutOfMemory) {
        std.debug.print("หน่วยความจำไม่พอ!\n", .{});
    } else if (err == error.FileNotFound) {
        std.debug.print("ไม่พบไฟล์!\n", .{});
    } else {
        std.debug.print("Error ที่ไม่รู้จัก: {s}\n", .{@errorName(err)});
    }
}

// ฟังก์ชันที่รับ error handler
fn runWithErrorHandler(
    comptime f: fn () anyerror!void,
    handler: *const fn (anyerror) void,
) void {
    f() catch |err| handler(err);
}

fn riskyTask() anyerror!void {
    return error.SomethingUnexpected;
}

pub fn main() void {
    // ใช้ anyerror เพื่อจัดการ error ทุกชนิด
    runWithErrorHandler(riskyTask, processAnyError);
    
    // แปลงระหว่าง specific error set และ anyerror
    const specific: error{Specific} = error.Specific;
    const any: anyerror = specific; // implicit coercion
    processAnyError(any);
}
```

---

## Step 185: errdefer สำหรับ Cleanup

### ทำความเข้าใจ errdefer

`errdefer` เป็น statement คล้ายกับ `defer` แต่ทำงานเฉพาะเมื่อฟังก์ชันออกด้วย error เท่านั้น ใช้สำหรับ cleanup เมื่อเกิดข้อผิดพลาด

```zig
const std = @import("std");

pub fn main() void {
    // defer: ทำงานเมื่อออกจาก scope เสมอ
    // errdefer: ทำงานเฉพาะเมื่อออกจาก scope ด้วย error
    
    exampleWithErrdefer() catch |err| {
        std.debug.print("เกิด error: {}\n", .{err});
    };
    
    std.debug.print("--- ทดสอบแบบสำเร็จ ---\n", .{});
    exampleSuccess() catch |err| {
        std.debug.print("ไม่ควรเกิด: {}\n", .{err});
    };
}

fn exampleWithErrdefer() !void {
    std.debug.print("เริ่มต้น\n", .{});
    
    defer std.debug.print("defer: ทำงานเสมอ\n", .{});
    errdefer std.debug.print("errdefer: ทำงานเมื่อเกิด error\n", .{});
    
    return error.SomeError; // จะ trigger errdefer
}

fn exampleSuccess() !void {
    std.debug.print("เริ่มต้น\n", .{});
    
    defer std.debug.print("defer: ทำงานเสมอ\n", .{});
    errdefer std.debug.print("errdefer: ทำงานเมื่อเกิด error\n", .{});
    
    // ไม่มี error - errdefer จะไม่ทำงาน
}
```

### errdefer สำหรับ Resource Management

```zig
const std = @import("std");

const Resource = struct {
    id: u32,
    name: []const u8,
    
    fn acquire(id: u32) !Resource {
        std.debug.print("ได้รับ resource {d}\n", .{id});
        if (id == 0) return error.InvalidResourceId;
        return Resource{ .id = id, .name = "resource" };
    }
    
    fn release(self: Resource) void {
        std.debug.print("คืน resource {d}\n", .{self.id});
    }
};

fn processWithMultipleResources(allocator: std.mem.Allocator) !void {
    // ได้รับ resource แรก
    const r1 = try Resource.acquire(1);
    errdefer r1.release(); // คืนถ้าเกิด error
    
    // จองหน่วยความจำ
    const buffer = try allocator.alloc(u8, 1024);
    errdefer allocator.free(buffer); // คืนถ้าเกิด error
    
    // ได้รับ resource ที่สอง
    const r2 = try Resource.acquire(2);
    errdefer r2.release(); // คืนถ้าเกิด error
    
    // ถ้าถึงนี่แล้วเกิด error
    return error.ProcessingFailed;
    
    // ถ้าสำเร็จ - errdefer จะไม่ทำงาน
    // (ในกรณีนี้ต้องจัดการ cleanup เองหรือใช้ defer)
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    processWithMultipleResources(allocator) catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### Pattern: errdefer กับ Allocation

```zig
const std = @import("std");

const Config = struct {
    host: []u8,
    port: u16,
    database: []u8,
    
    fn parse(allocator: std.mem.Allocator, raw: []const u8) !Config {
        _ = raw;
        
        // จองหน่วยความจำสำหรับ host
        const host = try allocator.dupe(u8, "localhost");
        errdefer allocator.free(host); // คืนถ้าขั้นตอนต่อไปผิดพลาด
        
        // จองหน่วยความจำสำหรับ database
        const database = try allocator.dupe(u8, "myapp");
        errdefer allocator.free(database); // คืนถ้าขั้นตอนต่อไปผิดพลาด
        
        // ตรวจสอบ port (อาจเกิด error)
        const port: u16 = 5432;
        if (port == 0) return error.InvalidPort; // จะ trigger errdefer
        
        // สำเร็จ - ต้องจัดการ free เองตอนใช้งานเสร็จ
        return Config{
            .host = host,
            .port = port,
            .database = database,
        };
    }
    
    fn deinit(self: *Config, allocator: std.mem.Allocator) void {
        allocator.free(self.host);
        allocator.free(self.database);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var config = try Config.parse(allocator, "raw config string");
    defer config.deinit(allocator);
    
    std.debug.print("Host: {s}, Port: {d}, DB: {s}\n", .{
        config.host,
        config.port,
        config.database,
    });
}
```

### errdefer กับ Parameter

```zig
const std = @import("std");

fn createItems(allocator: std.mem.Allocator, count: usize) ![][]u8 {
    const items = try allocator.alloc([]u8, count);
    errdefer allocator.free(items);
    
    var created: usize = 0;
    errdefer {
        // คืน items ที่สร้างสำเร็จแล้วทั้งหมด
        for (items[0..created]) |item| {
            allocator.free(item);
        }
    }
    
    for (0..count) |i| {
        if (i == 3) return error.LimitReached; // จำลอง error
        
        items[i] = try allocator.alloc(u8, 16);
        @memset(items[i], @as(u8, @intCast(i + 65)));
        created += 1;
    }
    
    return items;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const result = createItems(allocator, 5);
    if (result) |items| {
        defer {
            for (items) |item| allocator.free(item);
            allocator.free(items);
        }
        std.debug.print("สร้างสำเร็จ {} items\n", .{items.len});
    } else |err| {
        std.debug.print("สร้างล้มเหลว: {}\n", .{err});
        // errdefer ใน createItems ทำการ cleanup แล้ว
    }
}
```

---

## Step 186: Error กับ switch - Exhaustive Handling

### การใช้ switch กับ Errors

```zig
const std = @import("std");

const HttpError = error{
    BadRequest,
    Unauthorized,
    Forbidden,
    NotFound,
    MethodNotAllowed,
    InternalServerError,
    ServiceUnavailable,
    GatewayTimeout,
};

fn handleHttpError(err: HttpError) void {
    // switch บน error set ต้อง exhaustive (ครบทุก error)
    switch (err) {
        error.BadRequest => std.debug.print("400: คำขอไม่ถูกต้อง\n", .{}),
        error.Unauthorized => std.debug.print("401: ต้องยืนยันตัวตน\n", .{}),
        error.Forbidden => std.debug.print("403: ไม่มีสิทธิ์\n", .{}),
        error.NotFound => std.debug.print("404: ไม่พบทรัพยากร\n", .{}),
        error.MethodNotAllowed => std.debug.print("405: Method ไม่ได้รับอนุญาต\n", .{}),
        error.InternalServerError => std.debug.print("500: เซิร์ฟเวอร์มีปัญหา\n", .{}),
        error.ServiceUnavailable => std.debug.print("503: บริการไม่พร้อมใช้งาน\n", .{}),
        error.GatewayTimeout => std.debug.print("504: Gateway หมดเวลา\n", .{}),
    }
}

pub fn main() void {
    const err: HttpError = error.NotFound;
    handleHttpError(err);
}
```

### Exhaustive vs Non-Exhaustive Switch

```zig
const std = @import("std");

const DatabaseError = error{
    ConnectionFailed,
    QuerySyntaxError,
    ConstraintViolation,
    DeadlockDetected,
    DiskFull,
    CorruptData,
};

fn handleDbError(err: DatabaseError) []const u8 {
    return switch (err) {
        // จัดการเฉพาะบาง error
        error.ConnectionFailed => "ไม่สามารถเชื่อมต่อฐานข้อมูลได้",
        error.QuerySyntaxError => "คำสั่ง SQL มีข้อผิดพลาด",
        error.ConstraintViolation => "ข้อมูลละเมิด constraint",
        error.DeadlockDetected => "พบ deadlock ลองใหม่อีกครั้ง",
        // ใช้ else สำหรับ error ที่เหลือ
        else => "เกิดข้อผิดพลาดที่ไม่คาดคิด",
    };
}

// แต่ถ้าต้องการ exhaustive handling อย่างเข้มงวด
fn handleDbErrorStrict(err: DatabaseError) []const u8 {
    return switch (err) {
        error.ConnectionFailed => "ไม่สามารถเชื่อมต่อฐานข้อมูลได้",
        error.QuerySyntaxError => "คำสั่ง SQL มีข้อผิดพลาด",
        error.ConstraintViolation => "ข้อมูลละเมิด constraint",
        error.DeadlockDetected => "พบ deadlock ลองใหม่อีกครั้ง",
        error.DiskFull => "ดิสก์เต็ม ล้าง log ก่อน",
        error.CorruptData => "ข้อมูลเสียหาย ต้องกู้คืนจาก backup",
        // ไม่มี else - compiler จะแจ้งเตือนถ้ามี error ที่ไม่ได้จัดการ
    };
}

pub fn main() void {
    const errors = [_]DatabaseError{
        error.ConnectionFailed,
        error.DeadlockDetected,
        error.DiskFull,
    };
    
    for (errors) |err| {
        std.debug.print("{s}: {s}\n", .{
            @errorName(err),
            handleDbErrorStrict(err),
        });
    }
}
```

### Switch กับ Error Union

```zig
const std = @import("std");

const ValidationError = error{
    TooShort,
    TooLong,
    InvalidCharacters,
    AlreadyExists,
};

fn validateUsername(username: []const u8) ValidationError!void {
    if (username.len < 3) return error.TooShort;
    if (username.len > 20) return error.TooLong;
    
    for (username) |c| {
        if (!std.ascii.isAlphanumeric(c) and c != '_') {
            return error.InvalidCharacters;
        }
    }
}

fn registerUser(username: []const u8) void {
    validateUsername(username) catch |err| switch (err) {
        error.TooShort => std.debug.print(
            "'{s}': ชื่อผู้ใช้สั้นเกินไป (ต้องอย่างน้อย 3 ตัวอักษร)\n",
            .{username},
        ),
        error.TooLong => std.debug.print(
            "'{s}': ชื่อผู้ใช้ยาวเกินไป (ไม่เกิน 20 ตัวอักษร)\n",
            .{username},
        ),
        error.InvalidCharacters => std.debug.print(
            "'{s}': ชื่อผู้ใช้มีอักขระที่ไม่ได้รับอนุญาต\n",
            .{username},
        ),
        error.AlreadyExists => std.debug.print(
            "'{s}': ชื่อผู้ใช้นี้ถูกใช้งานแล้ว\n",
            .{username},
        ),
    };
}

pub fn main() void {
    registerUser("ab");          // TooShort
    registerUser("validuser");   // สำเร็จ
    registerUser("this_is_a_very_long_username_indeed"); // TooLong
    registerUser("user@name");   // InvalidCharacters
}
```

### Switch กับ anyerror

```zig
const std = @import("std");

// เมื่อใช้ anyerror ต้องใช้ else เพราะไม่รู้ว่ามี error อะไรบ้าง
fn handleAnyError(err: anyerror) void {
    switch (err) {
        error.OutOfMemory => std.debug.print("หน่วยความจำไม่พอ\n", .{}),
        error.InvalidInput => std.debug.print("ข้อมูลนำเข้าไม่ถูกต้อง\n", .{}),
        else => std.debug.print("Error อื่นๆ: {s}\n", .{@errorName(err)}),
    }
}

pub fn main() void {
    handleAnyError(error.OutOfMemory);
    handleAnyError(error.SomeRandomError);
}
```

---

## Step 187: Custom Error Messages ด้วย Format

### ทำไมต้องมี Custom Error Messages

Error ใน Zig มีเพียงชื่อ แต่ในระบบจริงเราต้องการข้อความที่อธิบายรายละเอียดมากกว่านั้น

```zig
const std = @import("std");

// วิธีที่ 1: ใช้ struct เพื่อเก็บ context ของ error
const DetailedError = struct {
    err: anyerror,
    message: []const u8,
    file: []const u8,
    line: u32,
    
    pub fn format(
        self: DetailedError,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{s}:{d}: {s}: {s}", .{
            self.file,
            self.line,
            @errorName(self.err),
            self.message,
        });
    }
};

fn createError(err: anyerror, msg: []const u8) DetailedError {
    return DetailedError{
        .err = err,
        .message = msg,
        .file = @src().file,
        .line = @src().line,
    };
}

pub fn main() void {
    const err = createError(
        error.ValidationFailed,
        "อีเมลไม่ถูกรูปแบบ",
    );
    std.debug.print("{}\n", .{err});
}
```

### Error กับ Payload Pattern

```zig
const std = @import("std");

// Pattern: เก็บข้อมูลเพิ่มเติมไว้ใน global state หรือ thread-local
// เนื่องจาก error ใน Zig ไม่รองรับ payload โดยตรง

// วิธีที่ 1: ใช้ global buffer สำหรับ error message
var error_message_buf: [512]u8 = undefined;
var error_message_len: usize = 0;

fn setErrorMessage(comptime fmt: []const u8, args: anytype) void {
    const result = std.fmt.bufPrint(&error_message_buf, fmt, args) catch "error message too long";
    error_message_len = result.len;
}

fn getErrorMessage() []const u8 {
    return error_message_buf[0..error_message_len];
}

const ParseError = error{
    InvalidNumber,
    OutOfRange,
    UnexpectedCharacter,
};

fn parseInteger(input: []const u8) ParseError!i64 {
    if (input.len == 0) {
        setErrorMessage("ข้อมูลว่างเปล่า", .{});
        return error.InvalidNumber;
    }
    
    // ตรวจสอบอักขระแต่ละตัว
    for (input, 0..) |c, i| {
        if (!std.ascii.isDigit(c) and !(i == 0 and c == '-')) {
            setErrorMessage(
                "อักขระที่ไม่ถูกต้องที่ตำแหน่ง {d}: '{c}'",
                .{i, c},
            );
            return error.UnexpectedCharacter;
        }
    }
    
    const value = std.fmt.parseInt(i64, input, 10) catch {
        setErrorMessage("ตัวเลข '{s}' เกินขอบเขต i64", .{input});
        return error.OutOfRange;
    };
    
    return value;
}

pub fn main() void {
    const test_inputs = [_][]const u8{
        "42",
        "",
        "12a3",
        "99999999999999999999",
    };
    
    for (test_inputs) |input| {
        const result = parseInteger(input);
        if (result) |value| {
            std.debug.print("'{s}' = {d}\n", .{input, value});
        } else |err| {
            std.debug.print("'{s}' -> Error ({s}): {s}\n", .{
                input,
                @errorName(err),
                getErrorMessage(),
            });
        }
    }
}
```

### Error Context กับ Allocator

```zig
const std = @import("std");

// Pattern ขั้นสูง: ใช้ allocator เพื่อสร้าง error message แบบ dynamic
const ErrorContext = struct {
    allocator: std.mem.Allocator,
    err: anyerror,
    message: []u8,
    
    pub fn init(
        allocator: std.mem.Allocator,
        err: anyerror,
        comptime fmt: []const u8,
        args: anytype,
    ) !ErrorContext {
        const message = try std.fmt.allocPrint(allocator, fmt, args);
        return ErrorContext{
            .allocator = allocator,
            .err = err,
            .message = message,
        };
    }
    
    pub fn deinit(self: *ErrorContext) void {
        self.allocator.free(self.message);
    }
    
    pub fn format(
        self: ErrorContext,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("[{s}] {s}", .{
            @errorName(self.err),
            self.message,
        });
    }
};

const FileProcessError = error{
    ReadFailed,
    ParseFailed,
    ValidationFailed,
};

fn processFile(
    allocator: std.mem.Allocator,
    filename: []const u8,
    line: u32,
) (FileProcessError || std.mem.Allocator.Error)!void {
    if (line > 1000) {
        var ctx = try ErrorContext.init(
            allocator,
            error.ValidationFailed,
            "ไฟล์ '{s}' มี {d} บรรทัด เกินขีดจำกัด 1000 บรรทัด",
            .{filename, line},
        );
        defer ctx.deinit();
        std.debug.print("Validation: {}\n", .{ctx});
        return error.ValidationFailed;
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    processFile(allocator, "data.csv", 1500) catch |err| {
        std.debug.print("ไม่สามารถประมวลผลไฟล์: {s}\n", .{@errorName(err)});
    };
}
```

### การสร้าง Human-Readable Error Messages

```zig
const std = @import("std");

const AppError = error{
    DatabaseConnectionFailed,
    UserNotFound,
    InsufficientPermissions,
    InvalidEmailFormat,
    PasswordTooWeak,
    TokenExpired,
    RateLimitExceeded,
};

// Map error ไปเป็น human-readable message
fn getErrorMessage(err: AppError) []const u8 {
    return switch (err) {
        error.DatabaseConnectionFailed => 
            "ไม่สามารถเชื่อมต่อกับฐานข้อมูลได้ กรุณาลองใหม่อีกครั้งในภายหลัง",
        error.UserNotFound => 
            "ไม่พบผู้ใช้งาน กรุณาตรวจสอบอีเมลและลองใหม่",
        error.InsufficientPermissions => 
            "คุณไม่มีสิทธิ์ดำเนินการนี้ กรุณาติดต่อผู้ดูแลระบบ",
        error.InvalidEmailFormat => 
            "รูปแบบอีเมลไม่ถูกต้อง กรุณาตรวจสอบและลองใหม่",
        error.PasswordTooWeak => 
            "รหัสผ่านไม่ปลอดภัยพอ ต้องมีอย่างน้อย 8 ตัวอักษร ตัวพิมพ์ใหญ่ และตัวเลข",
        error.TokenExpired => 
            "เซสชันหมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่",
        error.RateLimitExceeded => 
            "มีการส่งคำขอมากเกินไป กรุณารอสักครู่แล้วลองใหม่",
    };
}

// Map error ไปเป็น HTTP status code
fn getHttpStatusCode(err: AppError) u16 {
    return switch (err) {
        error.DatabaseConnectionFailed => 503,
        error.UserNotFound => 404,
        error.InsufficientPermissions => 403,
        error.InvalidEmailFormat => 400,
        error.PasswordTooWeak => 400,
        error.TokenExpired => 401,
        error.RateLimitExceeded => 429,
    };
}

pub fn main() void {
    const test_errors = [_]AppError{
        error.UserNotFound,
        error.InvalidEmailFormat,
        error.TokenExpired,
        error.RateLimitExceeded,
    };
    
    for (test_errors) |err| {
        std.debug.print("HTTP {d}: {s}\n", .{
            getHttpStatusCode(err),
            getErrorMessage(err),
        });
    }
}
```

---

## Step 188: Error Propagation Patterns

### Pattern 1: Direct Propagation ด้วย try

```zig
const std = @import("std");

const IoError = error{
    ReadFailed,
    WriteFailed,
    SeekFailed,
    FlushFailed,
};

fn readData(buf: []u8) IoError!usize {
    _ = buf;
    return 42;
}

fn writeData(data: []const u8) IoError!void {
    _ = data;
}

// Pattern ง่ายที่สุด: ปล่อยให้ error ลอยขึ้นไปเอง
fn copyData(source: []u8, dest: []const u8) IoError!void {
    const n = try readData(source);  // ถ้าล้มเหลว propagate ทันที
    _ = n;
    try writeData(dest);             // ถ้าล้มเหลว propagate ทันที
}

pub fn main() void {
    var buf: [100]u8 = undefined;
    copyData(&buf, "destination") catch |err| {
        std.debug.print("Copy failed: {}\n", .{err});
    };
}
```

### Pattern 2: Error Transformation

```zig
const std = @import("std");

// Low-level errors
const LowLevelError = error{
    ErrnoAccess,
    ErrnoNoEntry,
    ErrnoAgain,
};

// High-level errors
const HighLevelError = error{
    PermissionDenied,
    ResourceNotFound,
    TemporaryFailure,
};

// แปลง error จาก low-level เป็น high-level
fn transformError(err: LowLevelError) HighLevelError {
    return switch (err) {
        error.ErrnoAccess => error.PermissionDenied,
        error.ErrnoNoEntry => error.ResourceNotFound,
        error.ErrnoAgain => error.TemporaryFailure,
    };
}

fn lowLevelOperation(path: []const u8) LowLevelError!void {
    _ = path;
    return error.ErrnoNoEntry;
}

fn highLevelOperation(path: []const u8) HighLevelError!void {
    lowLevelOperation(path) catch |err| {
        return transformError(err);
    };
}

pub fn main() void {
    highLevelOperation("/missing/file") catch |err| {
        std.debug.print("High-level error: {}\n", .{err});
    };
}
```

### Pattern 3: Error Wrapping ด้วย Union Types

```zig
const std = @import("std");

// สร้าง Result type คล้าย Rust
fn Result(comptime T: type, comptime E: type) type {
    return union(enum) {
        ok: T,
        err: E,
        
        pub fn isOk(self: @This()) bool {
            return self == .ok;
        }
        
        pub fn unwrap(self: @This()) T {
            return switch (self) {
                .ok => |val| val,
                .err => unreachable,
            };
        }
        
        pub fn unwrapOr(self: @This(), default: T) T {
            return switch (self) {
                .ok => |val| val,
                .err => default,
            };
        }
    };
}

const MathError = error{ DivisionByZero, Overflow };

fn safeDivide(a: i64, b: i64) Result(i64, MathError) {
    if (b == 0) return .{ .err = error.DivisionByZero };
    return .{ .ok = @divTrunc(a, b) };
}

pub fn main() void {
    const r1 = safeDivide(10, 2);
    const r2 = safeDivide(10, 0);
    
    std.debug.print("10 / 2 = {d}\n", .{r1.unwrap()});
    std.debug.print("10 / 0 = {d} (with default)\n", .{r2.unwrapOr(-1)});
}
```

### Pattern 4: Error Accumulation

```zig
const std = @import("std");

const ValidationError = error{
    InvalidName,
    InvalidEmail,
    InvalidAge,
    InvalidPhone,
};

const UserData = struct {
    name: []const u8,
    email: []const u8,
    age: i32,
    phone: []const u8,
};

// เก็บ errors หลายๆ อันพร้อมกัน
const ValidationErrors = struct {
    errors: std.ArrayList(ValidationError),
    
    fn init(allocator: std.mem.Allocator) ValidationErrors {
        return .{ .errors = std.ArrayList(ValidationError).init(allocator) };
    }
    
    fn deinit(self: *ValidationErrors) void {
        self.errors.deinit();
    }
    
    fn add(self: *ValidationErrors, err: ValidationError) !void {
        try self.errors.append(err);
    }
    
    fn hasErrors(self: ValidationErrors) bool {
        return self.errors.items.len > 0;
    }
    
    fn print(self: ValidationErrors) void {
        for (self.errors.items) |err| {
            std.debug.print("  - {s}\n", .{@errorName(err)});
        }
    }
};

fn validateUser(allocator: std.mem.Allocator, user: UserData) !void {
    var errors = ValidationErrors.init(allocator);
    defer errors.deinit();
    
    if (user.name.len == 0) try errors.add(error.InvalidName);
    if (user.age < 0 or user.age > 150) try errors.add(error.InvalidAge);
    if (user.phone.len < 10) try errors.add(error.InvalidPhone);
    
    // ตรวจสอบอีเมลอย่างง่าย
    var has_at = false;
    for (user.email) |c| {
        if (c == '@') { has_at = true; break; }
    }
    if (!has_at) try errors.add(error.InvalidEmail);
    
    if (errors.hasErrors()) {
        std.debug.print("Validation errors:\n", .{});
        errors.print();
        return error.InvalidName; // return error หนึ่งอัน
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const bad_user = UserData{
        .name = "",
        .email = "not-valid",
        .age = -5,
        .phone = "123",
    };
    
    validateUser(allocator, bad_user) catch |err| {
        std.debug.print("การตรวจสอบล้มเหลว: {s}\n", .{@errorName(err)});
    };
}
```

### Pattern 5: Retry Pattern

```zig
const std = @import("std");

const NetworkError = error{
    Timeout,
    ConnectionRefused,
    ServiceUnavailable,
};

var attempt_count: u32 = 0;

fn makeRequest(url: []const u8) NetworkError![]const u8 {
    _ = url;
    attempt_count += 1;
    
    // จำลอง: ล้มเหลว 2 ครั้งแรก แล้วสำเร็จ
    if (attempt_count < 3) {
        return error.Timeout;
    }
    return "response data";
}

fn makeRequestWithRetry(url: []const u8, max_retries: u32) NetworkError![]const u8 {
    var retries: u32 = 0;
    
    while (retries <= max_retries) : (retries += 1) {
        const result = makeRequest(url);
        if (result) |data| {
            if (retries > 0) {
                std.debug.print("สำเร็จหลังจากพยายาม {} ครั้ง\n", .{retries + 1});
            }
            return data;
        } else |err| {
            // ตรวจสอบว่า error นี้ควร retry หรือไม่
            const should_retry = switch (err) {
                error.Timeout => true,
                error.ServiceUnavailable => true,
                error.ConnectionRefused => false, // ไม่ควร retry
            };
            
            if (!should_retry or retries == max_retries) {
                return err;
            }
            
            std.debug.print("พยายามครั้งที่ {d} ล้มเหลว ({s}), กำลัง retry...\n", .{
                retries + 1,
                @errorName(err),
            });
        }
    }
    
    return error.Timeout;
}

pub fn main() void {
    const response = makeRequestWithRetry("https://api.example.com/data", 5);
    if (response) |data| {
        std.debug.print("ได้รับ response: {s}\n", .{data});
    } else |err| {
        std.debug.print("ล้มเหลวทั้งหมด: {}\n", .{err});
    }
}
```

---

## Step 189: Testing Errors ด้วย std.testing.expectError

### พื้นฐานการทดสอบ Errors

```zig
const std = @import("std");
const testing = std.testing;

const MathError = error{
    DivisionByZero,
    Overflow,
    Underflow,
};

fn divide(a: i64, b: i64) MathError!i64 {
    if (b == 0) return error.DivisionByZero;
    return @divTrunc(a, b);
}

// การทดสอบ error ที่ควรเกิด
test "divide by zero returns DivisionByZero error" {
    try testing.expectError(error.DivisionByZero, divide(10, 0));
}

test "normal division works" {
    const result = try divide(10, 2);
    try testing.expectEqual(@as(i64, 5), result);
}

test "negative division" {
    const result = try divide(-10, 2);
    try testing.expectEqual(@as(i64, -5), result);
}
```

### ทดสอบ Error กับ String Validation

```zig
const std = @import("std");
const testing = std.testing;

const ValidationError = error{
    TooShort,
    TooLong,
    InvalidCharacters,
    Empty,
};

fn validatePassword(password: []const u8) ValidationError!void {
    if (password.len == 0) return error.Empty;
    if (password.len < 8) return error.TooShort;
    if (password.len > 128) return error.TooLong;
    
    for (password) |c| {
        if (c < 32 or c > 126) return error.InvalidCharacters;
    }
}

test "empty password" {
    try testing.expectError(error.Empty, validatePassword(""));
}

test "password too short" {
    try testing.expectError(error.TooShort, validatePassword("abc"));
}

test "password too long" {
    const long_password = "a" ** 200;
    try testing.expectError(error.TooLong, validatePassword(long_password));
}

test "password with invalid characters" {
    try testing.expectError(error.InvalidCharacters, validatePassword("pass\x00word"));
}

test "valid password passes" {
    try validatePassword("SecurePass123!");
}
```

### ทดสอบ Error กับ File Operations (จำลอง)

```zig
const std = @import("std");
const testing = std.testing;

const FileError = error{
    NotFound,
    PermissionDenied,
    TooLarge,
    InvalidPath,
};

// ระบบไฟล์จำลองสำหรับทดสอบ
const MockFileSystem = struct {
    const max_size: usize = 1024 * 1024; // 1MB
    
    fn readFile(path: []const u8) FileError![]const u8 {
        if (path.len == 0) return error.InvalidPath;
        if (std.mem.startsWith(u8, path, "/restricted/")) {
            return error.PermissionDenied;
        }
        if (std.mem.eql(u8, path, "/missing.txt")) {
            return error.NotFound;
        }
        if (std.mem.eql(u8, path, "/huge.dat")) {
            return error.TooLarge;
        }
        return "file contents";
    }
};

test "read missing file returns NotFound" {
    try testing.expectError(
        error.NotFound,
        MockFileSystem.readFile("/missing.txt"),
    );
}

test "read restricted file returns PermissionDenied" {
    try testing.expectError(
        error.PermissionDenied,
        MockFileSystem.readFile("/restricted/secret.txt"),
    );
}

test "read huge file returns TooLarge" {
    try testing.expectError(
        error.TooLarge,
        MockFileSystem.readFile("/huge.dat"),
    );
}

test "read empty path returns InvalidPath" {
    try testing.expectError(
        error.InvalidPath,
        MockFileSystem.readFile(""),
    );
}

test "read valid file succeeds" {
    const contents = try MockFileSystem.readFile("/valid.txt");
    try testing.expectEqualStrings("file contents", contents);
}
```

### ทดสอบ Error Chains และ Propagation

```zig
const std = @import("std");
const testing = std.testing;

const ConfigError = error{
    FileNotFound,
    ParseError,
    MissingField,
    InvalidValue,
};

fn parsePort(value: []const u8) ConfigError!u16 {
    const port = std.fmt.parseInt(u16, value, 10) catch {
        return error.InvalidValue;
    };
    if (port == 0) return error.InvalidValue;
    return port;
}

fn loadConfig(path: []const u8) ConfigError!u16 {
    if (std.mem.eql(u8, path, "missing.cfg")) {
        return error.FileNotFound;
    }
    if (std.mem.eql(u8, path, "invalid.cfg")) {
        return error.ParseError;
    }
    // จำลองการ load config
    return try parsePort("8080");
}

test "invalid port value" {
    try testing.expectError(error.InvalidValue, parsePort("abc"));
    try testing.expectError(error.InvalidValue, parsePort("0"));
    try testing.expectError(error.InvalidValue, parsePort("99999")); // เกิน u16
}

test "valid port" {
    try testing.expectEqual(@as(u16, 8080), try parsePort("8080"));
    try testing.expectEqual(@as(u16, 443), try parsePort("443"));
}

test "missing config file" {
    try testing.expectError(
        error.FileNotFound,
        loadConfig("missing.cfg"),
    );
}

test "invalid config file" {
    try testing.expectError(
        error.ParseError,
        loadConfig("invalid.cfg"),
    );
}

test "valid config loads successfully" {
    const port = try loadConfig("valid.cfg");
    try testing.expect(port > 0);
}
```

### Pattern การทดสอบขั้นสูง

```zig
const std = @import("std");
const testing = std.testing;

// Helper function สำหรับทดสอบ error หลายชุด
fn expectAnyError(comptime errors: anytype, result: anytype) !void {
    if (result) |_| {
        return error.ExpectedError;
    } else |err| {
        inline for (errors) |expected| {
            if (err == expected) return;
        }
        return error.WrongError;
    }
}

const ProcessError = error{
    InvalidInput,
    OutOfMemory,
    InternalError,
    NetworkError,
};

fn processRequest(input: []const u8) ProcessError![]const u8 {
    if (input.len == 0) return error.InvalidInput;
    if (std.mem.eql(u8, input, "crash")) return error.InternalError;
    return input;
}

test "empty input is rejected" {
    try testing.expectError(error.InvalidInput, processRequest(""));
}

test "crash input causes internal error" {
    try testing.expectError(error.InternalError, processRequest("crash"));
}

test "valid input passes through" {
    const result = try processRequest("hello");
    try testing.expectEqualStrings("hello", result);
}

// ทดสอบว่า error เกิดหรือไม่เกิดตามเงื่อนไข
test "error behavior under various inputs" {
    const inputs = [_]struct {
        input: []const u8,
        expected_err: ?ProcessError,
    }{
        .{ .input = "", .expected_err = error.InvalidInput },
        .{ .input = "crash", .expected_err = error.InternalError },
        .{ .input = "valid", .expected_err = null },
    };
    
    for (inputs) |tc| {
        if (tc.expected_err) |expected| {
            try testing.expectError(expected, processRequest(tc.input));
        } else {
            _ = try processRequest(tc.input);
        }
    }
}
```

---

## Step 190: Production Error Handling Patterns

### Pattern 1: Error Logging System

```zig
const std = @import("std");

// ระดับความรุนแรงของ error
const LogLevel = enum {
    Debug,
    Info,
    Warning,
    Error,
    Critical,
    
    fn toString(self: LogLevel) []const u8 {
        return switch (self) {
            .Debug => "DEBUG",
            .Info => "INFO",
            .Warning => "WARN",
            .Error => "ERROR",
            .Critical => "CRITICAL",
        };
    }
};

// Error Logger สำหรับระบบจริง
const ErrorLogger = struct {
    writer: std.io.AnyWriter,
    min_level: LogLevel,
    
    fn init(writer: std.io.AnyWriter, min_level: LogLevel) ErrorLogger {
        return .{
            .writer = writer,
            .min_level = min_level,
        };
    }
    
    fn log(
        self: ErrorLogger,
        level: LogLevel,
        err: anyerror,
        context: []const u8,
    ) void {
        if (@intFromEnum(level) < @intFromEnum(self.min_level)) return;
        
        self.writer.print(
            "[{s}] error.{s} in {s}\n",
            .{level.toString(), @errorName(err), context},
        ) catch {};
    }
    
    fn logWithMessage(
        self: ErrorLogger,
        level: LogLevel,
        err: anyerror,
        context: []const u8,
        message: []const u8,
    ) void {
        if (@intFromEnum(level) < @intFromEnum(self.min_level)) return;
        
        self.writer.print(
            "[{s}] error.{s} in {s}: {s}\n",
            .{level.toString(), @errorName(err), context, message},
        ) catch {};
    }
};

const AppError = error{
    DatabaseError,
    NetworkError,
    ValidationError,
    AuthError,
};

pub fn main() void {
    const stderr = std.io.getStdErr().writer().any();
    const logger = ErrorLogger.init(stderr, .Warning);
    
    const err: AppError = error.DatabaseError;
    logger.log(.Error, err, "UserService.getUser");
    logger.logWithMessage(
        .Critical,
        error.NetworkError,
        "PaymentService.charge",
        "ไม่สามารถเชื่อมต่อ payment gateway ได้",
    );
}
```

### Pattern 2: Circuit Breaker

```zig
const std = @import("std");

// Circuit Breaker pattern สำหรับป้องกัน cascading failures
const CircuitState = enum {
    Closed,   // ปกติ
    Open,     // เปิด - ปฏิเสธ requests ทั้งหมด
    HalfOpen, // กำลังทดสอบว่ากลับมาทำงานได้หรือยัง
};

const CircuitBreakerError = error{
    CircuitOpen,
    ServiceFailed,
};

const CircuitBreaker = struct {
    state: CircuitState,
    failure_count: u32,
    failure_threshold: u32,
    success_count: u32,
    
    fn init(threshold: u32) CircuitBreaker {
        return .{
            .state = .Closed,
            .failure_count = 0,
            .failure_threshold = threshold,
            .success_count = 0,
        };
    }
    
    fn call(self: *CircuitBreaker, operation: anytype) CircuitBreakerError!@typeInfo(@TypeOf(operation)).Fn.return_type.? {
        _ = operation;
        switch (self.state) {
            .Open => return error.CircuitOpen,
            .Closed, .HalfOpen => {},
        }
        return error.ServiceFailed;
    }
    
    fn recordSuccess(self: *CircuitBreaker) void {
        self.failure_count = 0;
        if (self.state == .HalfOpen) {
            self.success_count += 1;
            if (self.success_count >= 3) {
                self.state = .Closed;
                self.success_count = 0;
                std.debug.print("Circuit กลับสู่สถานะ Closed\n", .{});
            }
        }
    }
    
    fn recordFailure(self: *CircuitBreaker) void {
        self.failure_count += 1;
        if (self.failure_count >= self.failure_threshold) {
            self.state = .Open;
            std.debug.print("Circuit เปิด! ปฏิเสธ requests\n", .{});
        }
    }
    
    fn getState(self: CircuitBreaker) CircuitState {
        return self.state;
    }
};

var request_count: u32 = 0;

fn callExternalService() CircuitBreakerError![]const u8 {
    request_count += 1;
    if (request_count <= 3) {
        return error.ServiceFailed; // จำลองการล้มเหลว
    }
    return "service response";
}

pub fn main() void {
    var breaker = CircuitBreaker.init(3);
    
    // จำลองการเรียก service หลายครั้ง
    var i: u32 = 0;
    while (i < 8) : (i += 1) {
        if (breaker.getState() == .Open) {
            std.debug.print("Request {d}: Circuit เปิด, ข้ามไป\n", .{i + 1});
            continue;
        }
        
        const result = callExternalService();
        if (result) |_| {
            breaker.recordSuccess();
            std.debug.print("Request {d}: สำเร็จ\n", .{i + 1});
        } else |err| {
            breaker.recordFailure();
            std.debug.print("Request {d}: ล้มเหลว ({s})\n", .{i + 1, @errorName(err)});
        }
    }
}
```

### Pattern 3: Error Recovery Strategy

```zig
const std = @import("std");

const DataError = error{
    NotFound,
    Corrupted,
    Unavailable,
};

// กลยุทธ์การกู้คืนจาก error
const RecoveryStrategy = enum {
    ReturnDefault,
    RetryOnce,
    UseCache,
    Fail,
};

fn getRecoveryStrategy(err: DataError) RecoveryStrategy {
    return switch (err) {
        error.NotFound => .ReturnDefault,
        error.Unavailable => .RetryOnce,
        error.Corrupted => .Fail, // ไม่ควร recover จากข้อมูลเสีย
    };
}

var cache: ?[]const u8 = null;
var primary_call_count: u32 = 0;

fn fetchFromPrimary() DataError![]const u8 {
    primary_call_count += 1;
    if (primary_call_count == 1) return error.Unavailable;
    return "primary data";
}

fn fetchFromCache() ?[]const u8 {
    return cache;
}

fn fetchWithRecovery() DataError![]const u8 {
    const result = fetchFromPrimary();
    if (result) |data| {
        cache = data;
        return data;
    } else |err| {
        const strategy = getRecoveryStrategy(err);
        
        return switch (strategy) {
            .ReturnDefault => "default value",
            .RetryOnce => fetchFromPrimary() catch {
                // ถ้า retry ก็ยังล้มเหลว ลอง cache
                return fetchFromCache() orelse error.Unavailable;
            },
            .UseCache => fetchFromCache() orelse err,
            .Fail => err,
        };
    }
}

pub fn main() void {
    // ครั้งแรก: Unavailable → retry → สำเร็จ
    const r1 = fetchWithRecovery();
    if (r1) |data| {
        std.debug.print("ครั้งที่ 1: {s}\n", .{data});
    } else |err| {
        std.debug.print("ครั้งที่ 1 ล้มเหลว: {}\n", .{err});
    }
}
```

### Pattern 4: Structured Error Response (สำหรับ API)

```zig
const std = @import("std");

// Error response ที่มีโครงสร้างชัดเจน สำหรับ REST API
const ErrorCode = enum(u32) {
    OK = 0,
    InvalidRequest = 1001,
    Unauthorized = 1002,
    Forbidden = 1003,
    NotFound = 1004,
    Conflict = 1005,
    InternalError = 5000,
    ServiceUnavailable = 5003,
};

const ApiError = struct {
    code: ErrorCode,
    message: []const u8,
    details: ?[]const u8,
    request_id: []const u8,
    
    fn toJson(self: ApiError, writer: anytype) !void {
        try writer.print(
            \\{{
            \\  "error": {{
            \\    "code": {d},
            \\    "message": "{s}",
            \\    "request_id": "{s}"
            \\  }}
            \\}}
        ,
            .{
                @intFromEnum(self.code),
                self.message,
                self.request_id,
            },
        );
        if (self.details) |details| {
            _ = details;
            // เพิ่ม details ในกรณีจำเป็น
        }
    }
};

const RequestError = error{
    InvalidParam,
    NotAuthenticated,
    NotAuthorized,
    ResourceNotFound,
    DuplicateResource,
    ServerError,
};

fn mapToApiError(err: RequestError, request_id: []const u8) ApiError {
    return switch (err) {
        error.InvalidParam => ApiError{
            .code = .InvalidRequest,
            .message = "คำขอมีพารามิเตอร์ที่ไม่ถูกต้อง",
            .details = null,
            .request_id = request_id,
        },
        error.NotAuthenticated => ApiError{
            .code = .Unauthorized,
            .message = "กรุณาเข้าสู่ระบบก่อนใช้งาน",
            .details = null,
            .request_id = request_id,
        },
        error.NotAuthorized => ApiError{
            .code = .Forbidden,
            .message = "คุณไม่มีสิทธิ์ดำเนินการนี้",
            .details = null,
            .request_id = request_id,
        },
        error.ResourceNotFound => ApiError{
            .code = .NotFound,
            .message = "ไม่พบทรัพยากรที่ต้องการ",
            .details = null,
            .request_id = request_id,
        },
        error.DuplicateResource => ApiError{
            .code = .Conflict,
            .message = "ทรัพยากรนี้มีอยู่แล้ว",
            .details = null,
            .request_id = request_id,
        },
        error.ServerError => ApiError{
            .code = .InternalError,
            .message = "เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์",
            .details = null,
            .request_id = request_id,
        },
    };
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const test_errors = [_]RequestError{
        error.InvalidParam,
        error.NotAuthenticated,
        error.ResourceNotFound,
    };
    
    for (test_errors, 0..) |err, i| {
        var request_id_buf: [32]u8 = undefined;
        const request_id = std.fmt.bufPrint(
            &request_id_buf,
            "req-{d:0>5}",
            .{i + 1},
        ) catch unreachable;
        
        const api_err = mapToApiError(err, request_id);
        try api_err.toJson(stdout);
        try stdout.print("\n---\n", .{});
    }
}
```

### Pattern 5: Production-Ready Error Handler

```zig
const std = @import("std");

// Error handler ที่พร้อมใช้งานจริง
const ErrorSeverity = enum {
    Low,    // ปัญหาเล็กน้อย แจ้งผู้ใช้
    Medium, // ปัญหาปานกลาง บันทึก log
    High,   // ปัญหาร้ายแรง แจ้งเตือนทีม
    Fatal,  // ระบบล้มเหลว restart จำเป็น
};

const ProductionError = struct {
    original_error: anyerror,
    severity: ErrorSeverity,
    component: []const u8,
    user_message: []const u8,
    internal_message: []const u8,
    should_notify_team: bool,
    should_retry: bool,
    
    fn fromError(
        err: anyerror,
        component: []const u8,
    ) ProductionError {
        // กำหนด severity และ metadata ตาม error type
        if (err == error.OutOfMemory) {
            return .{
                .original_error = err,
                .severity = .Fatal,
                .component = component,
                .user_message = "ระบบกำลังมีปัญหา กรุณาลองใหม่ภายหลัง",
                .internal_message = "Out of memory - system may need restart",
                .should_notify_team = true,
                .should_retry = false,
            };
        }
        
        // default สำหรับ error ที่ไม่รู้จัก
        return .{
            .original_error = err,
            .severity = .Medium,
            .component = component,
            .user_message = "เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง",
            .internal_message = "Unexpected error",
            .should_notify_team = false,
            .should_retry = true,
        };
    }
    
    fn handle(self: ProductionError) void {
        // บันทึก log
        std.debug.print("[{s}] [{s}] {s}: {s}\n", .{
            @tagName(self.severity),
            self.component,
            @errorName(self.original_error),
            self.internal_message,
        });
        
        // แจ้งทีมถ้าจำเป็น
        if (self.should_notify_team) {
            std.debug.print("*** แจ้งทีม: {s} - {s} ***\n", .{
                self.component,
                self.internal_message,
            });
        }
        
        // แจ้งผู้ใช้
        std.debug.print("ข้อความสำหรับผู้ใช้: {s}\n", .{self.user_message});
        
        if (self.should_retry) {
            std.debug.print("สามารถลองใหม่ได้\n", .{});
        }
    }
};

pub fn main() void {
    // จำลอง error ต่างๆ
    const errors = [_]struct {
        err: anyerror,
        component: []const u8,
    }{
        .{ .err = error.OutOfMemory, .component = "UserService" },
        .{ .err = error.NetworkTimeout, .component = "PaymentGateway" },
        .{ .err = error.InvalidInput, .component = "ValidationLayer" },
    };
    
    for (errors) |item| {
        const prod_err = ProductionError.fromError(item.err, item.component);
        prod_err.handle();
        std.debug.print("\n", .{});
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Error Set Design
ออกแบบ error sets สำหรับระบบจัดการสินค้าคงคลัง ที่มี:
- Error จากการค้นหาสินค้า
- Error จากการอัปเดตสต็อก
- Error จากการตรวจสอบความถูกต้อง
- รวม error sets และสร้างฟังก์ชัน handler

```zig
// TODO: ออกแบบและ implement ระบบ error จัดการสินค้า
const InventorySearchError = error{
    // เพิ่ม errors ที่เหมาะสม
};

const InventoryUpdateError = error{
    // เพิ่ม errors ที่เหมาะสม
};

const InventoryValidationError = error{
    // เพิ่ม errors ที่เหมาะสม
};

const InventoryError = InventorySearchError || InventoryUpdateError || InventoryValidationError;

// Implement ฟังก์ชันเหล่านี้:
// fn findProduct(id: u64) InventoryError!Product
// fn updateStock(id: u64, delta: i32) InventoryError!void
// fn validateProduct(product: Product) InventoryError!void
```

### แบบฝึกหัดที่ 2: errdefer Cleanup Chain
เขียนฟังก์ชันที่เปิดไฟล์ 3 ไฟล์พร้อมกัน และถ้ามีขั้นตอนใดล้มเหลว ต้องปิดไฟล์ที่เปิดไปแล้วทั้งหมด

```zig
// TODO: Implement ฟังก์ชันนี้ด้วย errdefer
fn openMultipleFiles(
    path1: []const u8,
    path2: []const u8,
    path3: []const u8,
) !struct { f1: std.fs.File, f2: std.fs.File, f3: std.fs.File } {
    // ใช้ errdefer เพื่อปิดไฟล์เมื่อเกิด error
}
```

### แบบฝึกหัดที่ 3: Custom Error Logger
สร้าง error logger ที่:
- รองรับหลาย log levels
- เขียนไปยัง file และ stderr พร้อมกัน
- เพิ่ม timestamp ใน log message
- รองรับ context (component name)

```zig
// TODO: Implement ErrorLogger struct ที่มีฟีเจอร์ครบถ้วน
const ErrorLogger = struct {
    // ...
};
```

### แบบฝึกหัดที่ 4: Comprehensive Error Testing
เขียน test suite สำหรับฟังก์ชัน parser ที่รับ JSON-like string

```zig
// TODO: เขียน test ที่ครอบคลุม error cases ทุกชนิด
const ParserError = error{
    UnexpectedToken,
    UnexpectedEOF,
    InvalidNumber,
    StringNotTerminated,
    DuplicateKey,
    NestingTooDeep,
};

fn parseValue(input: []const u8) ParserError!void {
    // ...
}

// เขียน tests:
// test "empty input"
// test "invalid number format"
// test "unterminated string"
// test "nesting too deep"
// test "valid simple values"
```

### แบบฝึกหัดที่ 5: Error Recovery System
สร้างระบบ recovery ที่:
- ลอง primary service
- ถ้าล้มเหลว ลอง backup service
- ถ้ายังล้มเหลว ใช้ cached value
- ถ้าไม่มี cache ส่ง error กลับพร้อม context

```zig
// TODO: Implement ระบบ fallback เต็มรูปแบบ
fn fetchDataWithFallback(
    key: []const u8,
    allocator: std.mem.Allocator,
) ![]const u8 {
    // Primary service
    // Backup service
    // Cache lookup
    // Error with context
}
```

---

## สรุปบทเรียน

### ความรู้ที่ได้จากส่วนนี้

**Step 181 - Error Sets Merging:**
- Error sets สามารถรวมกันด้วย `||` operator
- Subset coercion เกิดขึ้นอัตโนมัติ
- Error sets เป็น first-class types ใน Zig

**Step 182 - Inferred Error Sets:**
- `!` โดยไม่ระบุชื่อ = inferred error set
- ดีสำหรับ internal functions แต่ไม่ดีสำหรับ public API
- ไม่ใช้กับ recursive functions หรือ function pointers

**Step 183 - Error Return Traces:**
- Zig ติดตาม error propagation path โดยอัตโนมัติ
- เปิดใช้งานใน debug mode, ปิดใน release mode
- ช่วย debug ได้มากเมื่อ error เกิดในชั้นลึก

**Step 184 - @errorName และ @errorCast:**
- `@errorName` แปลง error เป็น string
- `@errorCast` แปลงระหว่าง error sets (ระวัง panic!)
- `anyerror` รับ error ทุกชนิด

**Step 185 - errdefer:**
- ทำงานเฉพาะเมื่อออกจาก scope ด้วย error
- สำคัญมากสำหรับ resource management
- ใช้ร่วมกับ allocator เพื่อป้องกัน memory leak

**Step 186 - Switch กับ Errors:**
- Switch บน error set ต้อง exhaustive
- ใช้ `else` เมื่อต้องการ catch-all
- Switch กับ `anyerror` ต้องมี `else` เสมอ

**Step 187 - Custom Error Messages:**
- Zig errors ไม่มี payload - ต้องใช้ pattern อื่น
- Global buffer pattern สำหรับ simple cases
- Struct pattern สำหรับ complex error context

**Step 188 - Error Propagation Patterns:**
- Direct propagation ด้วย `try`
- Error transformation จาก low-level สู่ high-level
- Retry pattern กับ configurable backoff
- Error accumulation สำหรับ validation

**Step 189 - Testing Errors:**
- `testing.expectError` สำหรับ test error cases
- ทดสอบทั้ง happy path และ error paths
- Table-driven tests สำหรับ error scenarios หลายๆ อัน

**Step 190 - Production Patterns:**
- Structured error logging ด้วย severity levels
- Circuit breaker pattern
- Error recovery strategies
- Structured API error responses

### แนวทางปฏิบัติที่ดี (Best Practices)

1. **ใช้ explicit error sets สำหรับ public API** - ทำให้ผู้ใช้รู้ว่าต้องจัดการ error อะไรบ้าง

2. **ใช้ exhaustive switch** - อย่า depend on `else` สำหรับ error ที่รู้จัก ให้จัดการทุก error อย่างชัดเจน

3. **errdefer ก่อนที่จะ allocate** - วาง errdefer ทันทีหลังจากได้รับ resource

4. **แยก error levels** - ไม่ใช่ทุก error มีความร้ายแรงเท่ากัน ออกแบบให้ handle ต่างกัน

5. **Transform errors ที่ boundary** - แปลง low-level errors เป็น high-level errors ที่ layer boundary

6. **Test error paths** - เขียน tests สำหรับ error cases เท่ากับ happy paths

7. **Context ใน error messages** - ให้ข้อมูลเพียงพอสำหรับ debugging โดยไม่ expose sensitive data

8. **ใช้ anyerror ระมัดระวัง** - anyerror ทำให้สูญเสีย type safety ควรใช้เฉพาะเมื่อจำเป็น

### เปรียบเทียบกับภาษาอื่น

| ภาษา | วิธีจัดการ Error |
|------|----------------|
| Zig | Error sets, explicit handling, no exceptions |
| Go | Multiple return values `(T, error)` |
| Rust | `Result<T, E>` type |
| C++ | Exceptions (try/catch) |
| Java | Checked + Unchecked exceptions |

Zig เลือกวิธีที่ทำให้ code ชัดเจนที่สุด: compiler บังคับให้จัดการ error ทุกชนิด ไม่มี hidden control flow จาก exceptions และ error trace ช่วย debug ได้โดยไม่ต้องเพิ่ม overhead ใน production

---

*จบส่วนที่ 19: Error Handling ขั้นสูงใน Zig*

*ส่วนถัดไป: Part 20 - Comptime และ Meta-programming ขั้นสูง*
