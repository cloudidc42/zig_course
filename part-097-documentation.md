# Part 97: Documentation ใน Zig
## Steps 961-970

---

## Step 961: Documentation Philosophy

### ปรัชญาของการทำ Documentation

Documentation ที่ดีไม่ใช่แค่การอธิบายว่าโค้ดทำอะไร แต่ต้องอธิบาย **ทำไม** ถึงทำแบบนั้น และ **เมื่อไหร่** ที่ควรใช้

### หลักการ Documentation ของ Zig

Zig มีปรัชญาที่เน้น:
1. **ชัดเจน** กว่า **กระชับ** - โค้ดควรอ่านง่ายเหมือนอ่าน English
2. **Self-documenting code** ลด noise ใน comments
3. **Documentation ที่ accurate** ดีกว่า documentation ที่มากแต่เก่า

```zig
// documentation_philosophy.zig
const std = @import("std");

// ❌ Bad: Comment ที่ไม่เพิ่มข้อมูล (เรียกว่า "noise comment")
// Increment i by 1
i += 1;

// ❌ Bad: Comment ที่ขัดแย้งกับโค้ด
// Returns the sum of a and b
fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

// ✅ Good: Comment ที่อธิบาย "ทำไม"
// ใช้ bitwise AND แทน modulo เพราะเร็วกว่า 3x บน modern hardware
// (ใช้ได้เฉพาะเมื่อ divisor เป็น power of 2)
const result = value & (bucket_count - 1);

// ✅ Good: Comment ที่อธิบาย algorithm ที่ไม่ obvious
// Knuth's multiplicative hash: กระจาย keys อย่างสม่ำเสมอ
// โดยใช้ golden ratio approximation (2^32 / phi ≈ 2654435769)
fn hashU32(key: u32) u32 {
    return key *% 2654435769;
}

// ✅ Good: Warning comment
// NOTE: ฟังก์ชันนี้ไม่ thread-safe ต้องใช้ mutex ป้องกัน
// ดู concurrent_access.zig สำหรับ thread-safe version
fn unsafeUpdate(state: *State, value: u32) void {
    state.value = value;
}

const State = struct { value: u32 };
```

### Documentation Debt

```zig
// documentation_debt.zig
const std = @import("std");

// TODO: อธิบาย algorithm นี้ให้ชัดขึ้น (documentation debt)
// ตอนนี้ทำงานถูกต้อง แต่คนอื่นอาจไม่เข้าใจ
fn mysteriousAlgorithm(n: u64) u64 {
    // FIXME: ยังไม่ handle edge case เมื่อ n = 0
    var result: u64 = 1;
    var base: u64 = n;
    var exp: u64 = 32;
    
    while (exp > 0) : (exp >>= 1) {
        if (exp & 1 != 0) result *%= base;
        base *%= base;
    }
    
    return result;
}

// เวอร์ชันที่ document แล้ว
/// คำนวณ n^32 mod 2^64 ด้วย fast exponentiation
/// 
/// ใช้ repeated squaring algorithm:
/// n^32 = (n^16)^2 = ((n^8)^2)^2 = ...
/// 
/// Time complexity: O(log exponent) = O(5) สำหรับ exponent=32
/// 
/// Parameters:
///   n: ค่าฐาน (base)
/// Returns: n^32 โดยใช้ wrapping arithmetic
fn power32(n: u64) u64 {
    var result: u64 = 1;
    var base: u64 = n;
    var exp: u64 = 32;
    
    while (exp > 0) : (exp >>= 1) {
        if (exp & 1 != 0) result *%= base;
        base *%= base;
    }
    
    return result;
}
```

### สิ่งที่ควร Document

```zig
// what_to_document.zig
const std = @import("std");

// 1. Public API - ต้อง document ทุกอย่าง
/// Queue ที่ thread-safe สำหรับ single producer, single consumer
/// 
/// ใช้ lock-free algorithm ที่อ้างอิงจาก:
/// "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms"
/// โดย Maged M. Michael และ Michael L. Scott (1996)
pub const SPSCQueue = struct {
    // ...
};

// 2. Non-obvious behavior
/// แปลง UTF-8 string เป็น uppercase
/// 
/// CAVEAT: ฟังก์ชันนี้ทำงานกับ ASCII เท่านั้น
/// สำหรับ Unicode-aware uppercase ดู unicode.toUpper()
pub fn asciiUpper(str: []const u8, out: []u8) void {
    std.debug.assert(str.len <= out.len);
    for (str, 0..) |c, i| {
        out[i] = std.ascii.toUpper(c);
    }
}

// 3. Error conditions
/// เปิด database connection
/// 
/// Errors:
///   error.FileNotFound - ถ้าไม่พบ database file
///   error.PermissionDenied - ถ้าไม่มีสิทธิ์อ่าน file
///   error.DatabaseCorrupt - ถ้า file เสียหาย
///   error.OutOfMemory - ถ้า memory ไม่พอ
pub fn openDatabase(path: []const u8, allocator: std.mem.Allocator) !*anyopaque {
    _ = path;
    _ = allocator;
    return error.FileNotFound;
}

// 4. Performance characteristics
/// ค้นหา element ใน sorted slice
/// 
/// Time: O(log n)
/// Space: O(1)
/// 
/// Precondition: slice ต้องเรียง ascending order แล้ว
pub fn binarySearch(T: type, slice: []const T, target: T) ?usize {
    var low: usize = 0;
    var high: usize = slice.len;
    while (low < high) {
        const mid = low + (high - low) / 2;
        if (slice[mid] == target) return mid;
        if (slice[mid] < target) low = mid + 1 else high = mid;
    }
    return null;
}
```

---

## Step 962: Code Comments และ Doc Comments

### ประเภทของ Comments ใน Zig

```zig
// comment_types.zig
const std = @import("std");

// Line comment - ใช้สำหรับ implementation notes

/// Doc comment - ใช้สำหรับ public API documentation
/// จะถูก zig doc รวมเข้าใน documentation

//! Top-level doc comment - อธิบาย module ทั้งหมด
//! ใส่ไว้ที่ต้น file หรือ container

/// คำนวณ Fibonacci number
/// 
/// ใช้ recursive algorithm (ไม่แนะนำสำหรับ n > 30)
/// สำหรับ performance ดี ดู `fibIterative` แทน
/// 
/// # Examples
/// ```
/// const result = fib(10);
/// assert(result == 55);
/// ```
pub fn fib(n: u32) u32 {
    return switch (n) {
        0 => 0,
        1 => 1,
        else => fib(n - 1) + fib(n - 2),
    };
}

/// คำนวณ Fibonacci number (iterative version)
/// 
/// Time: O(n), Space: O(1)
/// 
/// สามารถ handle n ได้ถึง ~93 ก่อน u64 overflow
pub fn fibIterative(n: u32) u64 {
    if (n <= 1) return n;
    var a: u64 = 0;
    var b: u64 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

### Doc Comments สำหรับ Struct

```zig
// struct_documentation.zig
const std = @import("std");

/// Configuration สำหรับ HTTP server
/// 
/// # Example
/// ```zig
/// const config = ServerConfig{
///     .port = 8080,
///     .max_connections = 100,
///     .timeout_ms = 5000,
/// };
/// ```
pub const ServerConfig = struct {
    /// Port ที่ server จะ listen (1-65535)
    port: u16 = 8080,
    
    /// จำนวน concurrent connections สูงสุด
    /// ค่า 0 หมายถึงไม่จำกัด
    max_connections: u32 = 1000,
    
    /// Timeout ของ connection ในหน่วย milliseconds
    /// ค่า 0 หมายถึงไม่มี timeout
    timeout_ms: u32 = 30_000,
    
    /// Path ของ TLS certificate file (optional)
    /// ถ้าเป็น null จะใช้ plain HTTP
    tls_cert_path: ?[]const u8 = null,
    
    /// Path ของ TLS private key file (optional)
    tls_key_path: ?[]const u8 = null,
    
    /// ตรวจสอบว่า config valid
    /// 
    /// Returns error ถ้า:
    ///   - port เป็น 0
    ///   - มี cert แต่ไม่มี key หรือกลับกัน
    pub fn validate(self: ServerConfig) !void {
        if (self.port == 0) return error.InvalidPort;
        
        const has_cert = self.tls_cert_path != null;
        const has_key = self.tls_key_path != null;
        
        if (has_cert != has_key) {
            return error.IncompleteТlsConfig;
        }
    }
    
    /// ตรวจสอบว่า TLS เปิดใช้งานอยู่
    pub fn isTls(self: ServerConfig) bool {
        return self.tls_cert_path != null;
    }
};

/// ผลลัพธ์จากการ parse HTTP request
pub const ParseResult = union(enum) {
    /// Parse สำเร็จ
    success: Request,
    /// Parse ล้มเหลวพร้อม error message
    failure: struct {
        /// HTTP status code ที่ควร return
        status: u16,
        /// Error message สำหรับ client
        message: []const u8,
    },
    
    const Request = struct {
        method: []const u8,
        path: []const u8,
        body: []const u8,
    };
};
```

### Best Practices สำหรับ Comments

```zig
// comment_best_practices.zig
const std = @import("std");

// ✅ อธิบาย "ทำไม" ไม่ใช่ "อะไร"
// หลีกเลี่ยงกรณีที่ slice ยาว 0 เพราะ ptr อาจ undefined
fn safeSlice(s: []const u8) []const u8 {
    if (s.len == 0) return s;
    return s;
}

// ✅ ใส่ reference ไปยัง algorithm หรือ source
// Robert Jenkins' 32-bit integer hash function
// http://burtleburtle.net/bob/hash/integer.html
fn jenkinsHash(a_: u32) u32 {
    var a = a_;
    a = (a +% 0x7ed55d16) +% (a << 12);
    a = (a ^ 0xc761c23c) ^ (a >> 19);
    a = (a +% 0x165667b1) +% (a << 5);
    a = (a +% 0xd3a2646c) ^ (a << 9);
    a = (a +% 0xfd7046c5) +% (a << 3);
    a = (a ^ 0xb55a4f09) ^ (a >> 16);
    return a;
}

// ✅ อธิบาย invariants
// Invariant: items[0..len] เป็น valid data เสมอ
// items[len..] อาจมีขยะจากการ allocate ก่อนหน้า
const Buffer = struct {
    items: []u8,
    len: usize,
};

// ✅ TODO/FIXME ต้องมีรายละเอียด
// TODO(username, 2024-01): เพิ่ม SIMD optimization สำหรับ x86_64
// ตอนนี้ใช้ scalar code ที่ช้ากว่า 4x
fn vectorAdd(a: []f32, b: []f32, out: []f32) void {
    std.debug.assert(a.len == b.len and b.len == out.len);
    for (out, a, b) |*o, x, y| o.* = x + y;
}

// ✅ HACK/WORKAROUND ต้องมี issue number
// HACK: Workaround สำหรับ bug ใน std library issue #1234
// ลบออกเมื่อ Zig 0.13 release
fn workaroundBug(buf: []u8) void {
    // ทำ workaround ที่นี่
    _ = buf;
}
```

---

## Step 963: zig doc Generation

### การสร้าง Documentation ด้วย zig doc

```bash
# สร้าง documentation
zig doc src/main.zig

# สร้าง documentation และ output ไปที่ directory เฉพาะ
zig build-lib src/mylib.zig -femit-docs=docs/

# สร้าง documentation จาก build.zig
zig build docs
```

### การตั้งค่า build.zig สำหรับ Documentation

```zig
// build.zig - documentation generation
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Library
    const lib = b.addStaticLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Documentation step
    const docs_step = b.step("docs", "Generate documentation");
    
    const docs = b.addInstallDirectory(.{
        .source_dir = lib.getEmittedDocs(),
        .install_dir = .prefix,
        .install_subdir = "docs",
    });
    
    docs_step.dependOn(&docs.step);
}
```

### การเขียน Module Doc Comments

```zig
//! # MyLib - Zig Library สำหรับ Data Processing
//!
//! Library นี้ให้ tools สำหรับการประมวลผลข้อมูลประสิทธิภาพสูง
//!
//! ## Features
//! - การ sort และ search อย่างมีประสิทธิภาพ
//! - Memory-efficient data structures
//! - Thread-safe collections
//!
//! ## Quick Start
//! ```zig
//! const mylib = @import("mylib");
//!
//! var list = mylib.ArrayList(u32).init(allocator);
//! defer list.deinit();
//!
//! try list.append(42);
//! ```
//!
//! ## สำหรับข้อมูลเพิ่มเติม
//! ดู README.md หรือเว็บไซต์ https://example.com/mylib

const std = @import("std");

/// Version ของ library
pub const version = std.SemanticVersion{ .major = 1, .minor = 2, .patch = 0 };

/// Type aliases สำหรับ convenience
pub const Allocator = std.mem.Allocator;
```

### การ Format Doc Comments ให้สวยงาม

```zig
// formatted_docs.zig
const std = @import("std");

/// HTTP Method ที่รองรับ
///
/// ใช้ใน `Request.method` และ `Router.route()`
pub const Method = enum {
    get,
    post,
    put,
    patch,
    delete,
    head,
    options,
    
    /// แปลง string เป็น Method
    ///
    /// Case-insensitive: "GET", "Get", "get" ทั้งหมด return .get
    ///
    /// Returns `null` ถ้า string ไม่ใช่ valid HTTP method
    ///
    /// # Examples
    /// ```zig
    /// const m = Method.parse("POST") orelse return error.InvalidMethod;
    /// assert(m == .post);
    /// ```
    pub fn parse(s: []const u8) ?Method {
        const methods = [_]struct { str: []const u8, method: Method }{
            .{ .str = "GET", .method = .get },
            .{ .str = "POST", .method = .post },
            .{ .str = "PUT", .method = .put },
            .{ .str = "PATCH", .method = .patch },
            .{ .str = "DELETE", .method = .delete },
            .{ .str = "HEAD", .method = .head },
            .{ .str = "OPTIONS", .method = .options },
        };
        
        for (methods) |entry| {
            if (std.ascii.eqlIgnoreCase(s, entry.str)) return entry.method;
        }
        return null;
    }
    
    /// แปลง Method เป็น string
    ///
    /// Returns uppercase string: .get -> "GET"
    pub fn toString(self: Method) []const u8 {
        return switch (self) {
            .get => "GET",
            .post => "POST",
            .put => "PUT",
            .patch => "PATCH",
            .delete => "DELETE",
            .head => "HEAD",
            .options => "OPTIONS",
        };
    }
};
```

---

## Step 964: README Best Practices

### การเขียน README ที่ดี

```markdown
# MyLib

> Short tagline describing what this library does in one sentence.

[![Build Status](https://github.com/user/mylib/workflows/CI/badge.svg)](https://github.com/user/mylib/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Zig Version](https://img.shields.io/badge/zig-0.13.0-orange.svg)](https://ziglang.org/)

## What is this?

ย่อหน้า 2-3 บรรทัดที่อธิบายว่า library ทำอะไร แก้ปัญหาอะไร และใครควรใช้

## Features

- Feature 1 พร้อม benefit
- Feature 2 พร้อม benefit  
- Feature 3 พร้อม benefit

## Installation

```bash
# เพิ่มใน build.zig.zon
zig fetch --save https://github.com/user/mylib/archive/v1.0.0.tar.gz
```

```zig
// ใน build.zig
const mylib = b.dependency("mylib", .{
    .target = target,
    .optimize = optimize,
});
exe.root_module.addImport("mylib", mylib.module("mylib"));
```

## Quick Start

```zig
const mylib = @import("mylib");
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();
    
    var list = try mylib.createList(allocator);
    defer list.deinit();
    
    try list.append("hello");
    std.debug.print("{s}\n", .{list.items[0]});
}
```

## Documentation

Full documentation is available at [docs.example.com/mylib](https://docs.example.com/mylib)

## Contributing

ดู [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT License - ดู [LICENSE](LICENSE)
```

### README สำหรับ Example Application

```zig
// readme_example.zig
// ตัวอย่างโค้ดสำหรับใน README

const std = @import("std");

/// Simple key-value store สำหรับ demo ใน README
pub const KVStore = struct {
    data: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) KVStore {
        return .{
            .data = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *KVStore) void {
        self.data.deinit();
    }
    
    /// เก็บ key-value pair
    /// 
    /// ถ้า key มีอยู่แล้ว จะ overwrite value เดิม
    pub fn set(self: *KVStore, key: []const u8, value: []const u8) !void {
        try self.data.put(key, value);
    }
    
    /// อ่าน value จาก key
    /// 
    /// Returns null ถ้าไม่พบ key
    pub fn get(self: *const KVStore, key: []const u8) ?[]const u8 {
        return self.data.get(key);
    }
    
    /// ลบ key
    /// 
    /// Returns true ถ้าลบสำเร็จ, false ถ้าไม่พบ key
    pub fn delete(self: *KVStore, key: []const u8) bool {
        return self.data.remove(key);
    }
    
    /// จำนวน entries ทั้งหมด
    pub fn count(self: *const KVStore) usize {
        return self.data.count();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var store = KVStore.init(allocator);
    defer store.deinit();
    
    try store.set("name", "Alice");
    try store.set("city", "Bangkok");
    
    if (store.get("name")) |name| {
        std.debug.print("Hello, {s}!\n", .{name});
    }
    
    std.debug.print("Total entries: {d}\n", .{store.count()});
}
```

---

## Step 965: API Documentation

### การเขียน API Documentation

```zig
// api_documentation.zig
const std = @import("std");

//! # HTTP Client API
//!
//! Module นี้ให้ HTTP client สำหรับทำ HTTP requests
//!
//! ## Usage Pattern
//!
//! ```zig
//! var client = try HttpClient.init(allocator);
//! defer client.deinit();
//!
//! const response = try client.get("https://api.example.com/data");
//! defer response.deinit();
//!
//! std.debug.print("Status: {d}\n", .{response.status});
//! ```

/// HTTP Client สำหรับทำ HTTP requests
///
/// Thread-safe: instance เดียวสามารถใช้ใน multiple threads ได้
///
/// Memory management:
/// - ต้อง call `deinit()` เมื่อไม่ใช้แล้ว
/// - Response objects ต้อง `deinit()` แยกต่างหาก
pub const HttpClient = struct {
    allocator: std.mem.Allocator,
    options: Options,
    
    /// Options สำหรับ HttpClient
    pub const Options = struct {
        /// Maximum time รอ connection (milliseconds)
        /// Default: 10,000 (10 seconds)
        connect_timeout_ms: u32 = 10_000,
        
        /// Maximum time รอ response (milliseconds)  
        /// Default: 30,000 (30 seconds)
        read_timeout_ms: u32 = 30_000,
        
        /// Maximum จำนวน redirects ที่จะ follow
        /// ค่า 0 หมายถึงไม่ follow redirects
        max_redirects: u8 = 10,
        
        /// User-Agent header
        user_agent: []const u8 = "MyLib/1.0 (Zig)",
    };
    
    /// HTTP Response
    pub const Response = struct {
        /// HTTP status code (200, 404, 500, etc.)
        status: u16,
        
        /// Response headers
        headers: std.StringHashMap([]const u8),
        
        /// Response body
        body: []const u8,
        
        allocator: std.mem.Allocator,
        
        pub fn deinit(self: *Response) void {
            self.headers.deinit();
            self.allocator.free(self.body);
        }
        
        /// ตรวจสอบว่า request สำเร็จ (status 2xx)
        pub fn isSuccess(self: Response) bool {
            return self.status >= 200 and self.status < 300;
        }
        
        /// อ่าน header value
        pub fn header(self: *const Response, name: []const u8) ?[]const u8 {
            return self.headers.get(name);
        }
    };
    
    /// Error types ที่ HttpClient อาจ return
    pub const Error = error{
        /// ไม่สามารถ connect ได้
        ConnectionFailed,
        /// Request timeout
        Timeout,
        /// Response ไม่ valid
        InvalidResponse,
        /// Too many redirects
        TooManyRedirects,
        /// Memory error
        OutOfMemory,
    };
    
    /// สร้าง HttpClient instance ใหม่
    ///
    /// Caller ต้อง call `deinit()` เมื่อเสร็จสิ้น
    pub fn init(allocator: std.mem.Allocator) !HttpClient {
        return HttpClient{
            .allocator = allocator,
            .options = .{},
        };
    }
    
    /// สร้าง HttpClient ด้วย custom options
    pub fn initWithOptions(allocator: std.mem.Allocator, options: Options) !HttpClient {
        return HttpClient{
            .allocator = allocator,
            .options = options,
        };
    }
    
    pub fn deinit(self: *HttpClient) void {
        _ = self;
    }
    
    /// ทำ GET request
    ///
    /// Parameters:
    ///   url: URL ที่จะ request (ต้องเริ่มต้นด้วย http:// หรือ https://)
    ///
    /// Returns:
    ///   Response ที่ caller ต้อง `deinit()`
    ///
    /// Errors:
    ///   error.ConnectionFailed - ไม่สามารถ connect ได้
    ///   error.Timeout - request หมดเวลา
    ///   error.InvalidResponse - response ไม่ valid
    ///
    /// # Example
    /// ```zig
    /// const resp = try client.get("https://api.example.com/users");
    /// defer resp.deinit();
    /// if (resp.isSuccess()) {
    ///     std.debug.print("{s}\n", .{resp.body});
    /// }
    /// ```
    pub fn get(self: *HttpClient, url: []const u8) Error!Response {
        _ = url;
        // Mock implementation
        var headers = std.StringHashMap([]const u8).init(self.allocator);
        const body = try self.allocator.dupe(u8, "{}");
        return Response{
            .status = 200,
            .headers = headers,
            .body = body,
            .allocator = self.allocator,
        };
    }
    
    /// ทำ POST request พร้อม JSON body
    ///
    /// Parameters:
    ///   url: URL ที่จะ request
    ///   body: JSON string ที่จะส่ง
    ///
    /// Content-Type header จะถูก set เป็น "application/json" อัตโนมัติ
    pub fn postJson(
        self: *HttpClient,
        url: []const u8,
        body: []const u8,
    ) Error!Response {
        _ = url;
        _ = body;
        var headers = std.StringHashMap([]const u8).init(self.allocator);
        const resp_body = try self.allocator.dupe(u8, "{}");
        return Response{
            .status = 201,
            .headers = headers,
            .body = resp_body,
            .allocator = self.allocator,
        };
    }
};
```

---

## Step 966: Examples ใน Documentation

### การเขียน Examples ที่ดี

```zig
// examples_in_docs.zig
const std = @import("std");

/// แปลง string เป็น integer
///
/// # Examples
///
/// Parse decimal integer:
/// ```zig
/// const n = try parseInt(i32, "42", 10);
/// try std.testing.expectEqual(42, n);
/// ```
///
/// Parse hexadecimal:
/// ```zig
/// const n = try parseInt(u32, "FF", 16);
/// try std.testing.expectEqual(255, n);
/// ```
///
/// Error handling:
/// ```zig
/// const result = parseInt(i32, "not-a-number", 10);
/// try std.testing.expectError(error.InvalidCharacter, result);
/// ```
pub fn parseInt(T: type, str: []const u8, radix: u8) !T {
    return std.fmt.parseInt(T, str, radix);
}

/// Stack data structure
///
/// # Overview
///
/// Stack ทำงานแบบ LIFO (Last In, First Out):
/// ```
/// push(1) -> [1]
/// push(2) -> [1, 2]
/// push(3) -> [1, 2, 3]
/// pop()   -> 3, stack: [1, 2]
/// pop()   -> 2, stack: [1]
/// ```
///
/// # Example: ใช้ Stack เพื่อ reverse string
/// ```zig
/// var stack = Stack(u8).init(allocator);
/// defer stack.deinit();
///
/// for ("hello") |c| try stack.push(c);
///
/// var result: [5]u8 = undefined;
/// var i: usize = 0;
/// while (stack.pop()) |c| : (i += 1) {
///     result[i] = c;
/// }
/// // result == "olleh"
/// ```
pub fn Stack(T: type) type {
    return struct {
        items: std.ArrayList(T),
        
        const Self = @This();
        
        pub fn init(allocator: std.mem.Allocator) Self {
            return .{ .items = std.ArrayList(T).init(allocator) };
        }
        
        pub fn deinit(self: *Self) void {
            self.items.deinit();
        }
        
        /// เพิ่ม element เข้า stack
        pub fn push(self: *Self, value: T) !void {
            try self.items.append(value);
        }
        
        /// เอา element ออกจาก stack
        /// Returns null ถ้า stack ว่าง
        pub fn pop(self: *Self) ?T {
            if (self.items.items.len == 0) return null;
            return self.items.pop();
        }
        
        /// ดู element บนสุดโดยไม่เอาออก
        pub fn peek(self: *const Self) ?T {
            if (self.items.items.len == 0) return null;
            return self.items.items[self.items.items.len - 1];
        }
        
        /// จำนวน elements ใน stack
        pub fn len(self: *const Self) usize {
            return self.items.items.len;
        }
        
        /// ตรวจสอบว่า stack ว่างหรือไม่
        pub fn isEmpty(self: *const Self) bool {
            return self.items.items.len == 0;
        }
    };
}

// Tests ที่ทำงานเป็น examples ด้วย
test "Stack: basic operations" {
    var stack = Stack(i32).init(std.testing.allocator);
    defer stack.deinit();
    
    try std.testing.expect(stack.isEmpty());
    
    try stack.push(1);
    try stack.push(2);
    try stack.push(3);
    
    try std.testing.expectEqual(3, stack.len());
    try std.testing.expectEqual(3, stack.peek().?);
    
    try std.testing.expectEqual(3, stack.pop().?);
    try std.testing.expectEqual(2, stack.pop().?);
    try std.testing.expectEqual(1, stack.pop().?);
    try std.testing.expectEqual(null, stack.pop());
}

test "Stack: reverse string" {
    var stack = Stack(u8).init(std.testing.allocator);
    defer stack.deinit();
    
    for ("hello") |c| try stack.push(c);
    
    var result: [5]u8 = undefined;
    var i: usize = 0;
    while (stack.pop()) |c| : (i += 1) {
        result[i] = c;
    }
    
    try std.testing.expectEqualStrings("olleh", &result);
}
```

---

## Step 967: Changelog Maintenance

### การเขียน CHANGELOG ที่ดี

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- เพิ่ม support สำหรับ IPv6 addresses ใน HttpClient

## [2.1.0] - 2024-01-15

### Added
- `HttpClient.postJson()` method สำหรับส่ง JSON data
- `Response.isSuccess()` helper method
- Support สำหรับ custom headers

### Changed
- `HttpClient.Options.connect_timeout_ms` default เปลี่ยนจาก 5000 เป็น 10000
- Error messages ชัดเจนขึ้น

### Deprecated
- `HttpClient.post()` ถูก deprecate แล้ว ใช้ `HttpClient.postJson()` แทน

### Fixed
- Fixed memory leak ใน `Response.deinit()` เมื่อ headers ว่าง
- Fixed crash เมื่อ URL ไม่มี path component

## [2.0.0] - 2023-12-01

### Breaking Changes
- `HttpClient.init()` ตอนนี้ return `!HttpClient` แทน `HttpClient`
- ลบ `HttpClient.setOption()` ออก ใช้ `Options` struct แทน
- `Response.body` ตอนนี้เป็น `[]const u8` แทน `[]u8`

### Added
- เพิ่ม `Options` struct สำหรับ configuration
- เพิ่ม `Response.header()` method

### Migration Guide

```zig
// Before (v1.x)
var client = HttpClient.init(allocator);
client.setOption(.timeout, 5000);

// After (v2.0)
var client = try HttpClient.initWithOptions(allocator, .{
    .connect_timeout_ms = 5000,
});
```

## [1.0.0] - 2023-10-01

### Added
- Initial release
- Basic HTTP GET/POST support
- Redirect following
- TLS support
```

### Changelog Automation

```zig
// changelog_helper.zig
// ช่วย generate changelog entries จาก git commits

const std = @import("std");

const CommitType = enum {
    feat,   // New feature
    fix,    // Bug fix
    docs,   // Documentation
    style,  // Formatting
    refactor, // Code refactoring
    perf,   // Performance improvement
    test,   // Tests
    chore,  // Maintenance
    
    fn fromPrefix(s: []const u8) ?CommitType {
        const types = [_]struct { prefix: []const u8, kind: CommitType }{
            .{ .prefix = "feat", .kind = .feat },
            .{ .prefix = "fix", .kind = .fix },
            .{ .prefix = "docs", .kind = .docs },
            .{ .prefix = "style", .kind = .style },
            .{ .prefix = "refactor", .kind = .refactor },
            .{ .prefix = "perf", .kind = .perf },
            .{ .prefix = "test", .kind = .test },
            .{ .prefix = "chore", .kind = .chore },
        };
        
        for (types) |t| {
            if (std.mem.startsWith(u8, s, t.prefix)) return t.kind;
        }
        return null;
    }
    
    fn toChangelog(self: CommitType) []const u8 {
        return switch (self) {
            .feat => "Added",
            .fix => "Fixed",
            .docs => "Documentation",
            .style => "Changed",
            .refactor => "Changed",
            .perf => "Performance",
            .test => "Testing",
            .chore => "Maintenance",
        };
    }
};

const Commit = struct {
    hash: []const u8,
    kind: CommitType,
    message: []const u8,
    breaking: bool,
};

fn generateChangelogEntry(commits: []const Commit, version: []const u8, date: []const u8) void {
    std.debug.print("\n## [{s}] - {s}\n\n", .{ version, date });
    
    // Group by type
    const sections = [_]CommitType{ .feat, .fix, .perf, .docs };
    
    for (sections) |section_type| {
        var has_items = false;
        
        for (commits) |commit| {
            if (commit.kind == section_type) {
                if (!has_items) {
                    std.debug.print("### {s}\n", .{section_type.toChangelog()});
                    has_items = true;
                }
                if (commit.breaking) {
                    std.debug.print("- **BREAKING**: {s}\n", .{commit.message});
                } else {
                    std.debug.print("- {s}\n", .{commit.message});
                }
            }
        }
        
        if (has_items) std.debug.print("\n", .{});
    }
}
```

---

## Step 968: Documentation Testing

### การทดสอบ Documentation

```zig
// documentation_testing.zig
const std = @import("std");

/// คูณ integer สองตัว
///
/// # Example
/// ```zig
/// const result = multiply(3, 4);
/// // result == 12
/// ```
///
/// # Example: overflow
/// ```zig
/// // สำหรับ large numbers ใช้ wrapping multiply
/// const result = multiply(std.math.maxInt(i32), 2); // undefined behavior!
/// // ใช้ multiplyWrapping แทน
/// ```
pub fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

/// คูณ integer แบบ wrapping (ไม่ overflow)
pub fn multiplyWrapping(a: i32, b: i32) i32 {
    return a *% b;
}

// Tests ที่สอดคล้องกับ examples ใน doc comments
test "multiply: basic" {
    try std.testing.expectEqual(12, multiply(3, 4));
    try std.testing.expectEqual(-12, multiply(3, -4));
    try std.testing.expectEqual(0, multiply(0, 100));
}

test "multiply: documentation examples" {
    // ตรวจสอบว่า examples ใน doc comment ถูกต้อง
    const result = multiply(3, 4);
    try std.testing.expectEqual(12, result);
}

test "multiplyWrapping: overflow handling" {
    const max_i32 = std.math.maxInt(i32);
    // ไม่ crash เมื่อ overflow
    const result = multiplyWrapping(max_i32, 2);
    _ = result; // ค่าที่ได้ขึ้นกับ hardware
}

/// สร้าง range iterator
///
/// # Example: Print 0 to 9
/// ```zig
/// var iter = range(0, 10);
/// while (iter.next()) |n| {
///     std.debug.print("{d}\n", .{n});
/// }
/// ```
///
/// # Example: Sum 1 to 100
/// ```zig
/// var iter = range(1, 101);
/// var sum: u32 = 0;
/// while (iter.next()) |n| sum += n;
/// // sum == 5050
/// ```
pub fn range(start: u32, end: u32) RangeIterator {
    return .{ .current = start, .end = end };
}

const RangeIterator = struct {
    current: u32,
    end: u32,
    
    pub fn next(self: *RangeIterator) ?u32 {
        if (self.current >= self.end) return null;
        const val = self.current;
        self.current += 1;
        return val;
    }
};

test "range: sum 1 to 100" {
    var iter = range(1, 101);
    var sum: u32 = 0;
    while (iter.next()) |n| sum += n;
    try std.testing.expectEqual(5050, sum);
}

test "range: print 0 to 9" {
    var iter = range(0, 10);
    var count: u32 = 0;
    while (iter.next()) |_| count += 1;
    try std.testing.expectEqual(10, count);
}
```

---

## Step 969: Documentation Automation

### การทำ Automated Documentation

```zig
// doc_automation.zig
const std = @import("std");

// build.zig สำหรับ automated documentation

pub fn buildDocs(b: *std.Build, lib: *std.Build.Step.Compile) void {
    const docs_step = b.step("docs", "Generate and serve documentation");
    
    // สร้าง documentation
    const docs = b.addInstallDirectory(.{
        .source_dir = lib.getEmittedDocs(),
        .install_dir = .prefix,
        .install_subdir = "docs",
    });
    docs_step.dependOn(&docs.step);
    
    // Optional: รัน local server เพื่อดู documentation
    const serve = b.addSystemCommand(&.{
        "python3", "-m", "http.server", "8080",
        "--directory", "zig-out/docs",
    });
    serve.step.dependOn(&docs.step);
    
    const serve_step = b.step("docs-serve", "Generate and serve documentation");
    serve_step.dependOn(&serve.step);
}
```

### Documentation Linting

```zig
// doc_linter.zig
const std = @import("std");

/// ตรวจสอบว่า function มี doc comment หรือไม่
const DocLinter = struct {
    issues: std.ArrayList(Issue),
    allocator: std.mem.Allocator,
    
    const Issue = struct {
        file: []const u8,
        line: u32,
        kind: IssueKind,
        name: []const u8,
    };
    
    const IssueKind = enum {
        missing_doc_comment,
        missing_return_description,
        missing_error_description,
        example_not_tested,
    };
    
    pub fn init(allocator: std.mem.Allocator) DocLinter {
        return .{
            .issues = std.ArrayList(Issue).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *DocLinter) void {
        self.issues.deinit();
    }
    
    pub fn checkFile(self: *DocLinter, path: []const u8) !void {
        const file = try std.fs.cwd().openFile(path, .{});
        defer file.close();
        
        const content = try file.readToEndAlloc(self.allocator, 10 * 1024 * 1024);
        defer self.allocator.free(content);
        
        var lines = std.mem.splitScalar(u8, content, '\n');
        var line_num: u32 = 0;
        var has_doc_comment = false;
        
        while (lines.next()) |line| {
            line_num += 1;
            const trimmed = std.mem.trim(u8, line, " \t");
            
            if (std.mem.startsWith(u8, trimmed, "///")) {
                has_doc_comment = true;
            } else if (std.mem.startsWith(u8, trimmed, "pub fn ")) {
                if (!has_doc_comment) {
                    // Extract function name
                    const name_start = "pub fn ".len;
                    var name_end = name_start;
                    while (name_end < trimmed.len and trimmed[name_end] != '(') {
                        name_end += 1;
                    }
                    
                    try self.issues.append(.{
                        .file = path,
                        .line = line_num,
                        .kind = .missing_doc_comment,
                        .name = trimmed[name_start..name_end],
                    });
                }
                has_doc_comment = false;
            } else if (!std.mem.startsWith(u8, trimmed, "//")) {
                has_doc_comment = false;
            }
        }
    }
    
    pub fn report(self: *const DocLinter) void {
        if (self.issues.items.len == 0) {
            std.debug.print("No documentation issues found!\n", .{});
            return;
        }
        
        std.debug.print("Documentation Issues:\n", .{});
        for (self.issues.items) |issue| {
            std.debug.print("  {s}:{d}: {s} - {s}\n", .{
                issue.file,
                issue.line,
                @tagName(issue.kind),
                issue.name,
            });
        }
        std.debug.print("\nTotal: {d} issues\n", .{self.issues.items.len});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var linter = DocLinter.init(allocator);
    defer linter.deinit();
    
    // ตรวจสอบ source files
    try linter.checkFile("src/main.zig");
    linter.report();
}
```

---

## Step 970: Complete Documentation Strategy

### Strategy ครบถ้วนสำหรับ Documentation

```zig
// complete_docs_strategy.zig
//! # Complete Documentation Strategy สำหรับ Zig Project
//!
//! Documentation ที่ดีต้องครอบคลุมหลายระดับ:
//!
//! 1. **Code Level**: Doc comments สำหรับ functions/types
//! 2. **Module Level**: Module doc comments อธิบาย purpose
//! 3. **Project Level**: README, CONTRIBUTING, CHANGELOG
//! 4. **Tutorial Level**: Getting started guides
//! 5. **Reference Level**: API reference จาก zig doc

const std = @import("std");

/// ============ Level 1: Type Documentation ============

/// HTTP Status Code
///
/// ครอบคลุม status codes ที่ใช้บ่อยตาม RFC 7231
pub const StatusCode = enum(u16) {
    // 2xx Success
    ok = 200,
    created = 201,
    no_content = 204,
    
    // 3xx Redirection
    moved_permanently = 301,
    not_modified = 304,
    
    // 4xx Client Errors
    bad_request = 400,
    unauthorized = 401,
    forbidden = 403,
    not_found = 404,
    method_not_allowed = 405,
    conflict = 409,
    unprocessable_entity = 422,
    too_many_requests = 429,
    
    // 5xx Server Errors
    internal_server_error = 500,
    not_implemented = 501,
    service_unavailable = 503,
    
    /// ตรวจสอบ status class
    pub fn isSuccess(self: StatusCode) bool {
        const v = @intFromEnum(self);
        return v >= 200 and v < 300;
    }
    
    pub fn isClientError(self: StatusCode) bool {
        const v = @intFromEnum(self);
        return v >= 400 and v < 500;
    }
    
    pub fn isServerError(self: StatusCode) bool {
        const v = @intFromEnum(self);
        return v >= 500 and v < 600;
    }
    
    /// Human-readable description
    pub fn phrase(self: StatusCode) []const u8 {
        return switch (self) {
            .ok => "OK",
            .created => "Created",
            .no_content => "No Content",
            .moved_permanently => "Moved Permanently",
            .not_modified => "Not Modified",
            .bad_request => "Bad Request",
            .unauthorized => "Unauthorized",
            .forbidden => "Forbidden",
            .not_found => "Not Found",
            .method_not_allowed => "Method Not Allowed",
            .conflict => "Conflict",
            .unprocessable_entity => "Unprocessable Entity",
            .too_many_requests => "Too Many Requests",
            .internal_server_error => "Internal Server Error",
            .not_implemented => "Not Implemented",
            .service_unavailable => "Service Unavailable",
        };
    }
};

/// ============ Level 2: Example-driven Documentation ============

/// Parse JSON value จาก string
///
/// # Supported Types
/// - `null`: JSON null
/// - `bool`: true/false  
/// - `number`: integers และ floats
/// - `string`: quoted strings
/// - `array`: [value, ...]
/// - `object`: {"key": value, ...}
///
/// # Example: Simple values
/// ```zig
/// const null_val = try parse("null", allocator);
/// const bool_val = try parse("true", allocator);
/// const num_val = try parse("42", allocator);
/// const str_val = try parse("\"hello\"", allocator);
/// ```
///
/// # Example: Complex structures
/// ```zig
/// const obj = try parse(
///     \\{"name": "Alice", "age": 30}
/// , allocator);
/// defer obj.deinit();
/// ```
///
/// # Error Handling
/// ```zig
/// const result = parse("invalid json", allocator) catch |err| switch (err) {
///     error.UnexpectedToken => {
///         std.debug.print("Invalid JSON!\n", .{});
///         return;
///     },
///     else => return err,
/// };
/// ```
pub fn parse(input: []const u8, allocator: std.mem.Allocator) !JsonValue {
    _ = allocator;
    // Simplified implementation for documentation purposes
    if (std.mem.eql(u8, input, "null")) return .null;
    if (std.mem.eql(u8, input, "true")) return .{ .bool = true };
    if (std.mem.eql(u8, input, "false")) return .{ .bool = false };
    
    if (std.fmt.parseInt(i64, input, 10)) |n| {
        return .{ .number = @floatFromInt(n) };
    } else |_| {}
    
    return error.UnexpectedToken;
}

pub const JsonValue = union(enum) {
    null,
    bool: bool,
    number: f64,
    string: []const u8,
    
    pub fn deinit(self: *JsonValue) void {
        _ = self;
    }
};

/// ============ Level 3: Documentation Tests ============

test "StatusCode: isSuccess" {
    try std.testing.expect(StatusCode.ok.isSuccess());
    try std.testing.expect(StatusCode.created.isSuccess());
    try std.testing.expect(!StatusCode.not_found.isSuccess());
    try std.testing.expect(!StatusCode.internal_server_error.isSuccess());
}

test "StatusCode: phrases" {
    try std.testing.expectEqualStrings("OK", StatusCode.ok.phrase());
    try std.testing.expectEqualStrings("Not Found", StatusCode.not_found.phrase());
}

test "parse: null" {
    const val = try parse("null", std.testing.allocator);
    try std.testing.expect(val == .null);
}

test "parse: bool" {
    const true_val = try parse("true", std.testing.allocator);
    try std.testing.expect(true_val.bool == true);
    
    const false_val = try parse("false", std.testing.allocator);
    try std.testing.expect(false_val.bool == false);
}

test "parse: invalid JSON" {
    try std.testing.expectError(error.UnexpectedToken, parse("not-json", std.testing.allocator));
}
```

### Documentation Checklist

สรุป checklist สำหรับ documentation ที่ครบถ้วน:

```
Project Documentation Checklist:

Code Level:
[ ] Public functions มี doc comments
[ ] Structs และ fields มีคำอธิบาย  
[ ] Error conditions ถูก document
[ ] Examples ใน doc comments ทำงานได้จริง
[ ] Non-obvious code มี inline comments

Module Level:
[ ] แต่ละ module มี top-level doc comment
[ ] Dependencies ถูก document
[ ] Module purpose ชัดเจน

Project Level:
[ ] README.md ครบถ้วน
[ ] CHANGELOG.md อัปเดตอยู่เสมอ
[ ] CONTRIBUTING.md มีสำหรับ open source projects
[ ] LICENSE file มีอยู่

Automation:
[ ] zig doc generate ใน CI/CD pipeline
[ ] Documentation tests รันในทุก CI run
[ ] Documentation linting มี (ถ้าต้องการ)
```

---

## สรุป Part 97

ใน Part นี้เราได้เรียนรู้เกี่ยวกับ Documentation ใน Zig ครบถ้วน:

- **Step 961**: ปรัชญาของ documentation - อธิบาย "ทำไม" ไม่ใช่ "อะไร"
- **Step 962**: ประเภทของ comments และวิธีเขียน doc comments ที่มีประสิทธิภาพ
- **Step 963**: การใช้ `zig doc` เพื่อ generate API documentation
- **Step 964**: การเขียน README ที่ดีและครบถ้วน
- **Step 965**: การเขียน API documentation ที่ละเอียดและมีประโยชน์
- **Step 966**: การเขียน examples ที่ทดสอบได้และอธิบายการใช้งานจริง
- **Step 967**: การ maintain CHANGELOG อย่างเป็นระบบ
- **Step 968**: การทดสอบ documentation ให้ตรงกับโค้ดจริง
- **Step 969**: การทำ automation สำหรับ documentation workflow
- **Step 970**: Complete documentation strategy ที่ครอบคลุมทุกระดับ

Documentation ที่ดีเป็นการลงทุนระยะยาวที่ช่วยให้ทีมและ community ใช้ library ของเราได้อย่างมีประสิทธิภาพ
