# Part 42: Build System ขั้นสูง

> **หลักสูตร Zig ภาษาไทย** | Steps 411-420

---

## Step 411: build.zig.zon - package manifest

`build.zig.zon` คือไฟล์ manifest ของ package ใน Zig ใช้กำหนด metadata และ dependencies ของโปรเจค

### รูปแบบพื้นฐาน

```zig
// build.zig.zon
.{
    .name = .my_project,
    .version = "1.0.0",
    .minimum_zig_version = "0.14.0",

    .dependencies = .{
        .zig_clap = .{
            .url = "https://github.com/Hejsil/zig-clap/archive/refs/tags/0.9.1.tar.gz",
            .hash = "122048992e2c0c3d8c8697cf3e99ddd71cf77a19a5c4ef0b79f45b2a5e3b1f7e456",
        },
    },

    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "README.md",
        "LICENSE",
    },
}
```

### Fields ใน build.zig.zon

| Field | ความหมาย | จำเป็น? |
|-------|----------|---------|
| `.name` | ชื่อ package | ใช่ |
| `.version` | version ของ package (semantic versioning) | ใช่ |
| `.minimum_zig_version` | version Zig ขั้นต่ำที่รองรับ | ไม่ |
| `.dependencies` | dependencies ที่ต้องการ | ไม่ |
| `.paths` | files ที่รวมใน package | ไม่ |

### Dependency Types

```zig
.dependencies = .{
    // 1. URL dependency (tar.gz หรือ zip)
    .pkg_a = .{
        .url = "https://example.com/pkg.tar.gz",
        .hash = "1220...",  // SHA-256 hash
    },

    // 2. Local path dependency
    .pkg_b = .{
        .path = "../my-local-lib",
    },

    // 3. Git dependency (ผ่าน URL ของ archive)
    .pkg_c = .{
        .url = "https://github.com/user/repo/archive/abc123.tar.gz",
        .hash = "1220...",
    },
},
```

### การตรวจสอบ Hash

```bash
# ดาวน์โหลดและตรวจสอบ hash อัตโนมัติ
zig fetch --save https://github.com/user/repo/archive/v1.0.tar.gz

# ไฟล์ build.zig.zon จะถูกอัพเดตอัตโนมัติพร้อม hash
```

### ตัวอย่าง build.zig.zon สมบูรณ์

```zig
.{
    .name = .awesome_cli,
    .version = "2.1.0",
    .minimum_zig_version = "0.14.0",

    .dependencies = .{
        // CLI argument parser
        .clap = .{
            .url = "https://github.com/Hejsil/zig-clap/archive/refs/tags/0.9.1.tar.gz",
            .hash = "122048992e2c0c3d8c8697cf3e99ddd71cf77a19a5c4ef0b79f45b2a5e3b1f7e456",
        },

        // HTTP library
        .httpz = .{
            .url = "https://github.com/karlseguin/http.zig/archive/refs/heads/master.tar.gz",
            .hash = "12200af7e98b6b4e2fc8d3fef7f2ae18e09db0a2c3d5e1f7a8b2c4d6e0f1a3b5c7",
        },
    },

    // Files ที่รวมใน package เมื่อ publish
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "docs",
        "README.md",
        "LICENSE",
        "CHANGELOG.md",
    },
}
```

---

## Step 412: zig fetch - downloading dependencies

`zig fetch` ดาวน์โหลด dependencies และอัพเดต `build.zig.zon`

### การใช้งาน

```bash
# ดาวน์โหลด dependency และเพิ่มใน build.zig.zon
zig fetch --save https://github.com/user/repo/archive/v1.0.tar.gz

# ดาวน์โหลดพร้อมกำหนดชื่อ
zig fetch --save=clap https://github.com/Hejsil/zig-clap/archive/refs/tags/0.9.1.tar.gz

# ดูข้อมูลโดยไม่ save
zig fetch https://github.com/user/repo/archive/v1.0.tar.gz
```

### กระบวนการ fetch

```
1. ดาวน์โหลด archive จาก URL
2. ตรวจสอบ integrity
3. คำนวณ SHA-256 hash
4. เก็บ cache ใน ~/.cache/zig/p/
5. อัพเดต build.zig.zon พร้อม hash
```

### โครงสร้าง Cache

```
~/.cache/zig/
└── p/
    ├── 1220.../         # package hash
    │   ├── build.zig
    │   ├── build.zig.zon
    │   └── src/
    └── 1220.../         # อีก package
```

### ตัวอย่างการเพิ่ม dependency ทีละขั้นตอน

```bash
# ขั้นที่ 1: fetch dependency
zig fetch --save=clap \
  https://github.com/Hejsil/zig-clap/archive/refs/tags/0.9.1.tar.gz

# build.zig.zon จะถูกอัพเดตเป็น:
# .clap = .{
#     .url = "https://...",
#     .hash = "1220...",
# }

# ขั้นที่ 2: ใช้ใน build.zig
# const clap = b.dependency("clap", .{ ... });
```

### การจัดการ Dependencies ในทีม

```bash
# เมื่อ clone โปรเจคใหม่ dependencies จะถูกดาวน์โหลดอัตโนมัติเมื่อ build
zig build

# หรือ fetch เพียงอย่างเดียวโดยไม่ build
zig build --fetch
```

---

## Step 413: Module dependencies

Modules คือวิธีการ organize และ share code ใน Zig

### สร้าง Module

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง module
    const utils_mod = b.addModule("utils", .{
        .root_source_file = b.path("src/utils.zig"),
        .target = target,
        .optimize = optimize,
    });

    const math_mod = b.addModule("math", .{
        .root_source_file = b.path("src/math.zig"),
        .target = target,
        .optimize = optimize,
    });

    // math module ต้องการ utils module
    math_mod.addImport("utils", utils_mod);

    // executable ใช้ทั้งสอง modules
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe.root_module.addImport("utils", utils_mod);
    exe.root_module.addImport("math", math_mod);

    b.installArtifact(exe);
}
```

```zig
// src/utils.zig
pub fn clamp(value: f32, min: f32, max: f32) f32 {
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

pub fn lerp(a: f32, b: f32, t: f32) f32 {
    return a + (b - a) * clamp(t, 0, 1);
}
```

```zig
// src/math.zig
const utils = @import("utils");  // import module by name

pub fn smoothstep(edge0: f32, edge1: f32, x: f32) f32 {
    const t = utils.clamp((x - edge0) / (edge1 - edge0), 0, 1);
    return t * t * (3 - 2 * t);
}
```

```zig
// src/main.zig
const std = @import("std");
const utils = @import("utils");  // import module
const math = @import("math");

pub fn main() void {
    const val = utils.lerp(0, 100, 0.5);
    std.debug.print("lerp(0, 100, 0.5) = {d}\n", .{val});

    const smooth = math.smoothstep(0, 1, 0.75);
    std.debug.print("smoothstep = {d}\n", .{smooth});
}
```

### ใช้ External Package เป็น Module

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ดึง dependency
    const clap_dep = b.dependency("clap", .{
        .target = target,
        .optimize = optimize,
    });

    // ดึง module จาก dependency
    const clap_mod = clap_dep.module("clap");

    const exe = b.addExecutable(.{
        .name = "my-cli",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่ม clap module
    exe.root_module.addImport("clap", clap_mod);

    b.installArtifact(exe);
}
```

```zig
// src/main.zig - ใช้ clap library
const std = @import("std");
const clap = @import("clap");

pub fn main() !void {
    const params = comptime clap.parseParamsComptime(
        \\-h, --help    Display help
        \\-v, --verbose Be verbose
        \\<str>...      Input files
        \\
    );

    var diag = clap.Diagnostic{};
    var res = clap.parse(clap.Help, &params, clap.parsers.default, .{
        .diagnostic = &diag,
    }) catch |err| {
        diag.report(std.io.getStdErr().writer(), err) catch {};
        return err;
    };
    defer res.deinit();

    if (res.args.help != 0)
        return clap.help(std.io.getStdErr().writer(), clap.Help, &params, .{});
}
```

---

## Step 414: Cross-compilation targets

Zig มีความสามารถ cross-compile ได้โดยไม่ต้องติดตั้ง toolchain เพิ่มเติม

### รายการ Targets ที่รองรับ

```bash
# ดูรายการ targets ทั้งหมด
zig targets

# Targets สำคัญ:
# x86_64-linux-gnu      Linux 64-bit (Intel/AMD)
# x86_64-linux-musl     Linux 64-bit (musl libc - static)
# aarch64-linux-gnu     Linux ARM 64-bit
# x86_64-macos          macOS Intel
# aarch64-macos         macOS Apple Silicon
# x86_64-windows-gnu    Windows 64-bit
# x86_64-windows-msvc   Windows 64-bit (MSVC ABI)
# wasm32-wasi           WebAssembly WASI
# wasm32-freestanding   WebAssembly (no OS)
# arm-linux-gnueabihf   Raspberry Pi
```

### Cross-compile ด้วย command line

```bash
# Compile สำหรับ Linux AMD64
zig build -Dtarget=x86_64-linux-gnu

# Compile สำหรับ Windows
zig build -Dtarget=x86_64-windows-gnu

# Compile สำหรับ macOS ARM (Apple Silicon)
zig build -Dtarget=aarch64-macos

# Compile สำหรับ WebAssembly
zig build -Dtarget=wasm32-wasi

# Compile สำหรับ Raspberry Pi Zero 2W
zig build -Dtarget=aarch64-linux-musleabi -Dcpu=cortex_a53
```

### กำหนด Multiple Targets ใน build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const optimize = b.standardOptimizeOption(.{});

    // กำหนด targets ที่ต้องการ build
    const targets: []const std.Target.Query = &.{
        .{ .cpu_arch = .x86_64, .os_tag = .linux,   .abi = .gnu },
        .{ .cpu_arch = .x86_64, .os_tag = .windows, .abi = .gnu },
        .{ .cpu_arch = .aarch64, .os_tag = .macos },
        .{ .cpu_arch = .x86_64, .os_tag = .macos },
    };

    for (targets) |t| {
        const resolved_target = b.resolveTargetQuery(t);

        const exe = b.addExecutable(.{
            .name = "my-app",
            .root_source_file = b.path("src/main.zig"),
            .target = resolved_target,
            .optimize = optimize,
        });

        // ใส่ไว้ใน subdirectory ตาม target
        const install = b.addInstallArtifact(exe, .{
            .dest_dir = .{
                .override = .{
                    .custom = b.fmt("{s}-{s}", .{
                        @tagName(t.cpu_arch orelse .x86_64),
                        @tagName(t.os_tag orelse .linux),
                    }),
                },
            },
        });

        b.getInstallStep().dependOn(&install.step);
    }
}
```

### Build สำหรับ Embedded (no OS)

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // กำหนด target สำหรับ microcontroller
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .thumb,
        .os_tag = .freestanding,
        .abi = .eabi,
        .cpu_model = .{ .explicit = &std.Target.arm.cpu.cortex_m4 },
    });

    const exe = b.addExecutable(.{
        .name = "firmware",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseSmall,
        .strip = true,
        .single_threaded = true,
    });

    // link script สำหรับ linker
    exe.setLinkerScript(b.path("linker.ld"));

    b.installArtifact(exe);
}
```

### ตรวจสอบ Target ใน Code

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    // ตรวจสอบ OS
    switch (builtin.target.os.tag) {
        .linux => std.debug.print("Running on Linux\n", .{}),
        .windows => std.debug.print("Running on Windows\n", .{}),
        .macos => std.debug.print("Running on macOS\n", .{}),
        else => std.debug.print("Running on other OS\n", .{}),
    }

    // ตรวจสอบ CPU architecture
    switch (builtin.target.cpu.arch) {
        .x86_64 => std.debug.print("x86_64 CPU\n", .{}),
        .aarch64 => std.debug.print("ARM64 CPU\n", .{}),
        .wasm32 => std.debug.print("WebAssembly\n", .{}),
        else => std.debug.print("Other CPU\n", .{}),
    }
}
```

---

## Step 415: Build options (addOption, b.option)

Build options ช่วยให้ผู้ใช้กำหนดค่าตอน build ได้

### b.option - รับค่าจาก command line

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // รับ option จาก command line: -Dport=8080
    const port = b.option(u16, "port", "Server port") orelse 3000;

    // Boolean option: -Ddebug-mode
    const debug_mode = b.option(bool, "debug-mode", "Enable debug mode") orelse false;

    // String option: -Dlog-level=info
    const log_level = b.option([]const u8, "log-level", "Log level") orelse "warn";

    std.debug.print("Building with port={}, debug={}, log={s}\n", .{
        port, debug_mode, log_level,
    });

    const exe = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);
}
```

### addOptions - ส่งค่าไปยัง source code

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // รับ options
    const port = b.option(u16, "port", "Server port") orelse 3000;
    const debug_mode = b.option(bool, "debug-mode", "Enable debug mode") orelse false;
    const app_name = b.option([]const u8, "app-name", "Application name") orelse "MyApp";

    // สร้าง options object
    const options = b.addOptions();
    options.addOption(u16, "port", port);
    options.addOption(bool, "debug_mode", debug_mode);
    options.addOption([]const u8, "app_name", app_name);

    const exe = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่ม options เป็น module
    exe.root_module.addOptions("build_options", options);

    b.installArtifact(exe);
}
```

```zig
// src/main.zig - ใช้ build options
const std = @import("std");
const build_options = @import("build_options");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    try stdout.print("App: {s}\n", .{build_options.app_name});
    try stdout.print("Port: {}\n", .{build_options.port});
    try stdout.print("Debug: {}\n", .{build_options.debug_mode});

    if (build_options.debug_mode) {
        try stdout.print("Debug mode enabled!\n", .{});
    }
}
```

```bash
# Build พร้อม custom options
zig build -Dport=8080 -Ddebug-mode=true -Dapp-name=MyServer
```

### Enum Options

```zig
const std = @import("std");

const LogLevel = enum { debug, info, warn, err };

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Enum option
    const log_level = b.option(LogLevel, "log-level", "Log level") orelse .warn;

    const options = b.addOptions();
    options.addOption(LogLevel, "log_level", log_level);

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe.root_module.addOptions("build_options", options);
    b.installArtifact(exe);
}
```

```bash
zig build -Dlog-level=debug
zig build -Dlog-level=info
zig build -Dlog-level=err
```

---

## Step 416: Multiple executables ในโปรเจคเดียว

โปรเจคหนึ่งสามารถมีหลาย executables ที่ share code กันได้

### ตัวอย่าง: Client-Server Project

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ===== Shared Library =====
    const common_mod = b.addModule("common", .{
        .root_source_file = b.path("src/common/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ===== Server =====
    const server = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("src/server/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    server.root_module.addImport("common", common_mod);
    b.installArtifact(server);

    const run_server = b.addRunArtifact(server);
    run_server.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_server.addArgs(args);
    b.step("server", "Run the server").dependOn(&run_server.step);

    // ===== Client =====
    const client = b.addExecutable(.{
        .name = "client",
        .root_source_file = b.path("src/client/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    client.root_module.addImport("common", common_mod);
    b.installArtifact(client);

    const run_client = b.addRunArtifact(client);
    run_client.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_client.addArgs(args);
    b.step("client", "Run the client").dependOn(&run_client.step);

    // ===== Admin Tool =====
    const admin = b.addExecutable(.{
        .name = "admin",
        .root_source_file = b.path("src/admin/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    admin.root_module.addImport("common", common_mod);
    b.installArtifact(admin);

    // ===== Tests =====
    const tests = b.addTest(.{
        .root_source_file = b.path("src/common/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.step("test", "Run tests").dependOn(&b.addRunArtifact(tests).step);
}
```

### โครงสร้างไฟล์

```
project/
├── build.zig
├── build.zig.zon
└── src/
    ├── common/
    │   ├── lib.zig         # Shared code
    │   ├── protocol.zig    # Network protocol
    │   └── types.zig       # Shared types
    ├── server/
    │   ├── main.zig
    │   ├── handler.zig
    │   └── db.zig
    ├── client/
    │   ├── main.zig
    │   └── ui.zig
    └── admin/
        ├── main.zig
        └── commands.zig
```

### Build Targets ที่แตกต่างกัน

```bash
zig build          # Build ทั้งหมด
zig build server   # รัน server
zig build client   # รัน client
zig build test     # รัน tests
```

---

## Step 417: Test step configuration

การตั้งค่า test step ให้มีประสิทธิภาพ

### Test Filter

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Test filter option
    const test_filter = b.option(
        []const u8,
        "test-filter",
        "Filter tests by name",
    );

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .filter = test_filter,  // กรอง tests ตามชื่อ
    });

    const run_tests = b.addRunArtifact(unit_tests);
    b.step("test", "Run tests").dependOn(&run_tests.step);
}
```

```bash
# รัน tests ทั้งหมด
zig build test

# รันเฉพาะ tests ที่มีคำว่า "math" ในชื่อ
zig build test -Dtest-filter=math

# รันเฉพาะ tests ที่มีคำว่า "parse" ในชื่อ
zig build test -Dtest-filter=parse
```

### Test Timeout

```zig
const unit_tests = b.addTest(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

// กำหนด timeout (milliseconds)
const run_tests = b.addRunArtifact(unit_tests);
run_tests.addArg("--timeout");
run_tests.addArg("5000");  // 5 seconds
```

### Multiple Test Files

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const test_step = b.step("test", "Run all tests");

    // รายชื่อไฟล์ test
    const test_files = &[_][]const u8{
        "src/math.zig",
        "src/string.zig",
        "src/network.zig",
        "tests/integration.zig",
    };

    for (test_files) |test_file| {
        const tests = b.addTest(.{
            .root_source_file = b.path(test_file),
            .target = target,
            .optimize = optimize,
        });

        const run = b.addRunArtifact(tests);
        test_step.dependOn(&run.step);
    }
}
```

### Test ใน Code

```zig
// src/math.zig
const std = @import("std");

pub fn fibonacci(n: u32) u64 {
    if (n <= 1) return n;
    var prev: u64 = 0;
    var curr: u64 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

test "fibonacci sequence" {
    try std.testing.expectEqual(@as(u64, 0), fibonacci(0));
    try std.testing.expectEqual(@as(u64, 1), fibonacci(1));
    try std.testing.expectEqual(@as(u64, 1), fibonacci(2));
    try std.testing.expectEqual(@as(u64, 8), fibonacci(6));
    try std.testing.expectEqual(@as(u64, 55), fibonacci(10));
}

test "fibonacci large" {
    // ตรวจสอบว่าไม่ overflow
    const result = fibonacci(50);
    try std.testing.expect(result > 0);
}
```

---

## Step 418: Generating source files

บางครั้งต้องสร้างไฟล์ source code ระหว่าง build

### WriteFile Step

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้างไฟล์ Zig จาก build info
    const write_files = b.addWriteFiles();

    // เขียนข้อมูล build ลงไฟล์
    const build_info_source = b.fmt(
        \\pub const version = "{s}";
        \\pub const build_date = "{s}";
        \\pub const git_commit = "{s}";
        \\
    , .{
        "1.0.0",
        "2024-01-01",
        "abc123",
    });

    const build_info_file = write_files.add("build_info.zig", build_info_source);

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ทำให้ exe รู้จักกับ generated file
    exe.root_module.addAnonymousImport("build_info", .{
        .root_source_file = build_info_file,
    });

    b.installArtifact(exe);
}
```

```zig
// src/main.zig
const std = @import("std");
const build_info = @import("build_info");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Version: {s}\n", .{build_info.version});
    try stdout.print("Built: {s}\n", .{build_info.build_date});
    try stdout.print("Commit: {s}\n", .{build_info.git_commit});
}
```

### Code Generation จาก External Tool

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // รัน code generator
    const gen = b.addSystemCommand(&[_][]const u8{
        "python3",
        "tools/generate_bindings.py",
    });

    // กำหนด output file
    const generated = gen.addOutputFileArg("src/generated/bindings.zig");

    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ใช้ generated file
    exe.root_module.addAnonymousImport("bindings", .{
        .root_source_file = generated,
    });

    b.installArtifact(exe);
}
```

### Protocol Buffer / Schema Generation

```zig
// ตัวอย่างการสร้าง types จาก JSON schema
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Generate types จาก schema
    const schema_gen = b.addSystemCommand(&[_][]const u8{
        "node",
        "tools/schema-to-zig.js",
        "--schema",
        "schema/api.json",
    });

    const types_file = schema_gen.addOutputFileArg("src/api_types.zig");

    const exe = b.addExecutable(.{
        .name = "api-server",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    exe.root_module.addAnonymousImport("api_types", .{
        .root_source_file = types_file,
    });

    b.installArtifact(exe);
}
```

---

## Step 419: Embedding files (@embedFile)

`@embedFile` ฝัง file เป็น binary data ใน executable

### พื้นฐาน @embedFile

```zig
// src/main.zig
const std = @import("std");

// ฝัง HTML template ลงใน binary
const html_template = @embedFile("templates/index.html");

// ฝัง configuration file
const default_config = @embedFile("config/default.json");

// ฝัง image (binary)
const logo_png = @embedFile("assets/logo.png");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // ใช้ embedded data
    try stdout.print("HTML template ({} bytes):\n", .{html_template.len});
    try stdout.writeAll(html_template);

    // Parse embedded JSON
    std.debug.print("Config: {s}\n", .{default_config});
}
```

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>My App</title>
</head>
<body>
    <h1>Hello from embedded HTML!</h1>
    <p>This content is embedded in the binary.</p>
</body>
</html>
```

### Web Server ที่ฝัง Static Files

```zig
const std = @import("std");

// ฝัง static files ทั้งหมด
const index_html = @embedFile("web/index.html");
const style_css  = @embedFile("web/style.css");
const app_js     = @embedFile("web/app.js");
const logo_svg   = @embedFile("web/logo.svg");

const StaticFile = struct {
    content: []const u8,
    content_type: []const u8,
};

const static_files = std.StaticStringMap(StaticFile).initComptime(.{
    .{ "/", .{ .content = index_html, .content_type = "text/html" } },
    .{ "/style.css", .{ .content = style_css, .content_type = "text/css" } },
    .{ "/app.js", .{ .content = app_js, .content_type = "application/javascript" } },
    .{ "/logo.svg", .{ .content = logo_svg, .content_type = "image/svg+xml" } },
});

pub fn serveStaticFile(path: []const u8) ?StaticFile {
    return static_files.get(path);
}

pub fn main() !void {
    // เริ่ม web server ที่ serve embedded files
    std.debug.print("Serving {} static files\n", .{static_files.kvs.len});
}
```

### Embedded Database / Dictionary

```zig
const std = @import("std");

// ฝัง word list
const word_list = @embedFile("data/words.txt");

// ฝัง JSON data
const country_data = @embedFile("data/countries.json");

// ฝัง binary protocol buffer
const proto_schema = @embedFile("schemas/messages.pb");

pub fn main() !void {
    // นับจำนวนคำใน word list
    var line_count: usize = 0;
    var iter = std.mem.splitScalar(u8, word_list, '\n');
    while (iter.next()) |_| line_count += 1;

    std.debug.print("Loaded {} words from embedded data\n", .{line_count});
}
```

### ข้อควรระวังกับ @embedFile

```zig
// *** ข้อควรระวัง ***
// 1. @embedFile เพิ่มขนาด binary
// 2. ไฟล์ต้องมีอยู่ตอน compile time
// 3. เปลี่ยนไม่ได้ตอน runtime
// 4. ไม่เหมาะกับไฟล์ขนาดใหญ่มาก

// สำหรับไฟล์ขนาดใหญ่ ควรใช้วิธีอ่านจาก disk แทน
const config = blk: {
    // อ่านจาก file system ตอน runtime
    const file = std.fs.cwd().openFile("config.json", .{}) catch {
        // ถ้าไม่มีไฟล์ ใช้ embedded default
        break :blk @embedFile("config/default.json");
    };
    defer file.close();
    // อ่านและ return
    break :blk "{}"; // simplified
};
```

---

## Step 420: Production build configurations

การตั้งค่าสำหรับ production builds

### Production build.zig สมบูรณ์

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ===== Options =====
    const strip = b.option(bool, "strip", "Strip debug info") orelse
        (optimize != .Debug);

    const single_threaded = b.option(
        bool,
        "single-threaded",
        "Optimize for single thread",
    ) orelse false;

    const version = b.option(
        []const u8,
        "version",
        "Application version",
    ) orelse "dev";

    // ===== Build Info =====
    const options = b.addOptions();
    options.addOption([]const u8, "version", version);
    options.addOption(bool, "debug", optimize == .Debug);

    // ===== Library =====
    const lib = b.addStaticLibrary(.{
        .name = "core",
        .root_source_file = b.path("src/core/lib.zig"),
        .target = target,
        .optimize = optimize,
        .strip = strip,
        .single_threaded = single_threaded,
    });
    lib.root_module.addOptions("build_options", options);

    // ===== Executable =====
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
        .strip = strip,
        .single_threaded = single_threaded,
    });
    exe.linkLibrary(lib);
    exe.root_module.addOptions("build_options", options);
    b.installArtifact(exe);

    // ===== Run =====
    const run = b.addRunArtifact(exe);
    run.step.dependOn(b.getInstallStep());
    if (b.args) |args| run.addArgs(args);
    b.step("run", "Run the application").dependOn(&run.step);

    // ===== Tests =====
    const tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    tests.root_module.addOptions("build_options", options);
    b.step("test", "Run tests").dependOn(&b.addRunArtifact(tests).step);
}
```

### Production Build Scripts

```bash
#!/bin/bash
# scripts/build-release.sh

# Build สำหรับทุก platforms
PLATFORMS=(
    "x86_64-linux-gnu"
    "x86_64-linux-musl"
    "aarch64-linux-gnu"
    "x86_64-windows-gnu"
    "x86_64-macos"
    "aarch64-macos"
)

VERSION=$(git describe --tags --always)

for platform in "${PLATFORMS[@]}"; do
    echo "Building for $platform..."
    zig build \
        -Dtarget="$platform" \
        -Doptimize=ReleaseFast \
        -Dstrip=true \
        -Dversion="$VERSION" \
        --prefix "release/$platform"
    echo "Done: release/$platform"
done

echo "All builds complete!"
ls -la release/
```

### CI/CD Configuration

```yaml
# .github/workflows/build.yml
name: Build and Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Zig
        uses: goto-bus-stop/setup-zig@v2
        with:
          version: 0.14.0

      - name: Run Tests
        run: zig build test

      - name: Build Release
        run: zig build -Doptimize=ReleaseSafe

      - name: Build Cross-platform
        run: |
          zig build -Dtarget=x86_64-linux-gnu -Doptimize=ReleaseFast
          zig build -Dtarget=x86_64-windows-gnu -Doptimize=ReleaseFast
          zig build -Dtarget=aarch64-linux-gnu -Doptimize=ReleaseFast
```

### Docker สำหรับ Build Environment

```dockerfile
# Dockerfile.build
FROM ubuntu:22.04

# ติดตั้ง Zig
RUN apt-get update && apt-get install -y curl xz-utils
RUN curl -L https://ziglang.org/download/0.14.0/zig-linux-x86_64-0.14.0.tar.xz \
    | tar -xJ -C /opt/
ENV PATH="/opt/zig-linux-x86_64-0.14.0:$PATH"

WORKDIR /app
COPY . .

# Build
RUN zig build -Doptimize=ReleaseSmall -Dtarget=x86_64-linux-musl

# Runtime image (เล็กมาก - ใช้ musl libc)
FROM scratch
COPY --from=0 /app/zig-out/bin/my-app /my-app
ENTRYPOINT ["/my-app"]
```

### Optimization Tips

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "optimized-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // LTO (Link Time Optimization) - ทำให้ binary เล็กลงและเร็วขึ้น
    // (ใช้ได้เมื่อ optimize != Debug)
    // exe.want_lto = optimize != .Debug;

    // Stack size
    // exe.stack_size = 1024 * 1024;  // 1MB stack

    b.installArtifact(exe);
}
```

---

## สรุป Part 42: Build System ขั้นสูง

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 411 | build.zig.zon | Package manifest format |
| 412 | zig fetch | ดาวน์โหลด dependencies |
| 413 | Module dependencies | การ share code ด้วย modules |
| 414 | Cross-compilation | Build สำหรับ platform อื่น |
| 415 | Build options | รับค่าจาก command line |
| 416 | Multiple executables | หลาย programs ในโปรเจคเดียว |
| 417 | Test configuration | ตั้งค่า test step |
| 418 | Generating sources | สร้างไฟล์ระหว่าง build |
| 419 | @embedFile | ฝังไฟล์ใน binary |
| 420 | Production builds | Configuration สำหรับ production |

### คำสั่ง Build สำหรับ Production

```bash
# Development build
zig build

# Production build
zig build -Doptimize=ReleaseFast -Dstrip=true

# Cross-compile for Linux (static binary)
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseSafe

# Small binary for embedded
zig build -Doptimize=ReleaseSmall -Dstrip=true

# Check formatting
zig fmt --check src/

# Run tests before release
zig build test -Doptimize=ReleaseSafe
```

> **ต่อไป**: Part 43 - C Interoperability พื้นฐาน (การใช้ C libraries ใน Zig)
