# Part 41: Build System พื้นฐาน

> **หลักสูตร Zig ภาษาไทย** | Steps 401-410

---

## Step 401: build.zig คืออะไร?

`build.zig` คือไฟล์หัวใจของระบบ build ใน Zig โดยแทนที่จะใช้ Makefile หรือ CMakeLists.txt แบบเดิม Zig มีระบบ build ที่เขียนด้วย Zig เอง ทำให้นักพัฒนาสามารถใช้ภาษา Zig ที่คุ้นเคยในการกำหนดกระบวนการ build ได้

### ทำไม build.zig ถึงพิเศษ?

1. **ใช้ Zig ทั้งหมด** - ไม่มีภาษาพิเศษแยกต่างหาก
2. **Type-safe** - compiler ช่วยตรวจสอบข้อผิดพลาด
3. **Cross-compilation ในตัว** - build สำหรับ platform อื่นได้ง่าย
4. **Package management** - จัดการ dependencies ได้
5. **Reproducible builds** - ผลลัพธ์เหมือนกันทุกครั้ง

### โครงสร้างพื้นฐานของโปรเจค Zig

```
my-project/
├── build.zig          # Build script หลัก
├── build.zig.zon      # Package manifest (dependencies)
├── src/
│   ├── main.zig       # Source code หลัก
│   └── lib.zig        # Library code
└── tests/
    └── test_main.zig  # Test files
```

### ตัวอย่าง build.zig ง่ายที่สุด

```zig
// build.zig - ตัวอย่างพื้นฐาน
const std = @import("std");

pub fn build(b: *std.Build) void {
    // สร้าง executable จาก src/main.zig
    const exe = b.addExecutable(.{
        .name = "hello",
        .root_source_file = b.path("src/main.zig"),
        .target = b.standardTargetOptions(.{}),
        .optimize = b.standardOptimizeOption(.{}),
    });

    // ติดตั้ง executable ใน output directory
    b.installArtifact(exe);
}
```

```zig
// src/main.zig
const std = @import("std");

pub fn main() void {
    std.debug.print("สวัสดี Zig Build System!\n", .{});
}
```

### การรัน build

```bash
# Build โปรเจค
zig build

# รัน โปรแกรม
zig build run

# รัน tests
zig build test
```

---

## Step 402: โครงสร้าง build.zig

`build.zig` มีโครงสร้างที่กำหนดไว้ชัดเจน มาดูรายละเอียดแต่ละส่วน

### Function Signature หลัก

```zig
pub fn build(b: *std.Build) void {
    // เนื้อหาการ build ทั้งหมดอยู่ที่นี่
}
```

`b` คือ pointer ไปยัง `std.Build` ซึ่งเป็น build context ที่ให้เราเข้าถึง API ทั้งหมดในการกำหนดกระบวนการ build

### Standard Options

Zig มี options มาตรฐาน 2 อย่างที่ควรใช้เสมอ:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // 1. Target options - กำหนด platform ที่จะ compile ไป
    const target = b.standardTargetOptions(.{});

    // 2. Optimize options - กำหนด optimization level
    const optimize = b.standardOptimizeOption(.{});

    // ใช้กับ artifact
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);
}
```

### การกำหนด Target

```zig
// Target เริ่มต้น (native)
const target = b.standardTargetOptions(.{});

// Target เฉพาะ (cross-compile)
const cross_target = b.resolveTargetQuery(.{
    .cpu_arch = .x86_64,
    .os_tag = .linux,
    .abi = .gnu,
});
```

### Optimization Levels

```zig
// Optimize level เริ่มต้น
const optimize = b.standardOptimizeOption(.{});

// กำหนดเอง
const optimize_release = std.builtin.OptimizeMode.ReleaseFast;
```

### โครงสร้าง build.zig ที่สมบูรณ์

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // ========== Options ==========
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ========== Executable ==========
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // ========== Run Step ==========
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);

    // ========== Test Step ==========
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

## Step 403: addExecutable, addStaticLibrary, addSharedLibrary

Zig Build System รองรับการสร้าง artifact หลายประเภท

### addExecutable - สร้างโปรแกรม

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง executable
    const exe = b.addExecutable(.{
        .name = "my-program",                           // ชื่อไฟล์ output
        .root_source_file = b.path("src/main.zig"),    // ไฟล์ต้นทาง
        .target = target,
        .optimize = optimize,
        // options เพิ่มเติม:
        // .version = .{ .major = 1, .minor = 0, .patch = 0 },
        // .strip = false,  // ไม่ลบ debug symbols
        // .single_threaded = false,
    });

    b.installArtifact(exe);
}
```

### addStaticLibrary - สร้าง Static Library

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง static library
    const lib = b.addStaticLibrary(.{
        .name = "mylib",                               // จะสร้าง libmylib.a (Linux) หรือ mylib.lib (Windows)
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(lib);
}
```

```zig
// src/lib.zig - ไลบรารีที่จะถูก compile เป็น static library
const std = @import("std");

// ฟังก์ชันที่ export ออกไป
pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

pub const MathError = error{
    DivisionByZero,
    Overflow,
};

pub fn divide(a: i32, b: i32) MathError!i32 {
    if (b == 0) return MathError.DivisionByZero;
    return @divTrunc(a, b);
}
```

### addSharedLibrary - สร้าง Dynamic/Shared Library

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง shared library
    const shared_lib = b.addSharedLibrary(.{
        .name = "mysharedlib",   // จะสร้าง libmysharedlib.so (Linux) หรือ mysharedlib.dll (Windows)
        .root_source_file = b.path("src/shared_lib.zig"),
        .target = target,
        .optimize = optimize,
        .version = .{ .major = 1, .minor = 0, .patch = 0 },
    });

    b.installArtifact(shared_lib);
}
```

```zig
// src/shared_lib.zig
const std = @import("std");

// ต้องใช้ export สำหรับ C ABI
export fn zig_add(a: c_int, b: c_int) c_int {
    return a + b;
}

export fn zig_greet(name: [*:0]const u8) void {
    std.debug.print("สวัสดี, {s}!\n", .{name});
}
```

### ใช้ Library ใน Executable

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง library ก่อน
    const lib = b.addStaticLibrary(.{
        .name = "mathlib",
        .root_source_file = b.path("src/math.zig"),
        .target = target,
        .optimize = optimize,
    });

    // สร้าง executable และ link กับ library
    const exe = b.addExecutable(.{
        .name = "calculator",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Link กับ library
    exe.linkLibrary(lib);

    b.installArtifact(exe);
}
```

---

## Step 404: addRunArtifact - การ run

`addRunArtifact` สร้าง run step ที่รัน executable หลังจาก build

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // สร้าง run command
    const run_cmd = b.addRunArtifact(exe);

    // ให้ run หลัง install เสร็จ
    run_cmd.step.dependOn(b.getInstallStep());

    // รับ arguments จากบรรทัดคำสั่ง: zig build run -- arg1 arg2
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // สร้าง step ชื่อ "run"
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

### การส่ง Arguments ให้โปรแกรม

```bash
# ส่ง arguments หลัง --
zig build run -- --port 8080 --verbose
```

```zig
// ใน build.zig: เพิ่ม arguments คงที่
run_cmd.addArg("--config");
run_cmd.addArg("config.json");

// หรือเพิ่มหลาย args พร้อมกัน
run_cmd.addArgs(&[_][]const u8{ "--host", "localhost", "--port", "3000" });
```

### กำหนด Environment Variables

```zig
// กำหนด env vars สำหรับ run
run_cmd.setEnvironmentVariable("DEBUG", "1");
run_cmd.setEnvironmentVariable("LOG_LEVEL", "info");
```

### กำหนด Working Directory

```zig
// กำหนด directory ที่จะรัน
run_cmd.setCwd(b.path("data/"));
```

### ตัวอย่าง Run หลาย Programs

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Server
    const server = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("src/server.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(server);

    const run_server = b.addRunArtifact(server);
    run_server.step.dependOn(b.getInstallStep());

    const server_step = b.step("server", "Run the server");
    server_step.dependOn(&run_server.step);

    // Client
    const client = b.addExecutable(.{
        .name = "client",
        .root_source_file = b.path("src/client.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(client);

    const run_client = b.addRunArtifact(client);
    run_client.step.dependOn(b.getInstallStep());

    const client_step = b.step("client", "Run the client");
    client_step.dependOn(&run_client.step);
}
```

---

## Step 405: addInstallArtifact - การ install

การ install คือการคัดลอกไฟล์ output ไปยัง output directory (โดยปกติคือ `zig-out/`)

### b.installArtifact - วิธีง่าย

```zig
// ติดตั้งที่ default location
b.installArtifact(exe);  // ไปที่ zig-out/bin/my-app
b.installArtifact(lib);  // ไปที่ zig-out/lib/libmylib.a
```

### b.addInstallArtifact - กำหนด location เอง

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ติดตั้งไปยัง custom location
    const install_exe = b.addInstallArtifact(exe, .{
        .dest_dir = .{ .override = .{ .custom = "custom-bin" } },
    });

    b.getInstallStep().dependOn(&install_exe.step);
}
```

### ติดตั้ง Files เพิ่มเติม

```zig
// ติดตั้ง file เดี่ยว
b.installFile("config.json", "share/my-app/config.json");

// ติดตั้งทั้ง directory
b.installDirectory(.{
    .source_dir = b.path("assets"),
    .install_dir = .prefix,
    .install_subdir = "share/my-app/assets",
});
```

### โครงสร้าง Output Directory

```
zig-out/
├── bin/
│   └── my-app        # executables
├── lib/
│   ├── libmylib.a    # static libraries
│   └── libmylib.so   # shared libraries
├── include/
│   └── mylib.h       # header files
└── share/
    └── my-app/
        └── assets/   # data files
```

### ตัวอย่างการ install header file

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const lib = b.addSharedLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ติดตั้ง library
    b.installArtifact(lib);

    // ติดตั้ง header file ด้วย
    b.installFile("include/mylib.h", "include/mylib.h");
}
```

---

## Step 406: Build modes (Debug, ReleaseSafe, ReleaseFast, ReleaseSmall)

Zig มี 4 optimization modes ที่แต่ละอย่างมีจุดประสงค์ต่างกัน

### Debug Mode (ค่าเริ่มต้น)

```zig
const optimize = std.builtin.OptimizeMode.Debug;
```

**คุณสมบัติ:**
- ไม่ optimize code
- เก็บ debug information ทั้งหมด
- เปิด safety checks ทั้งหมด
- Compile เร็ว, รันช้า
- ไฟล์ขนาดใหญ่

**เหมาะสำหรับ:** การพัฒนาและ debugging

```zig
// ใน Debug mode สิ่งนี้จะ panic ถ้า overflow
var x: u8 = 200;
x += 100; // Panic: integer overflow!
```

### ReleaseSafe Mode

```zig
const optimize = std.builtin.OptimizeMode.ReleaseSafe;
```

**คุณสมบัติ:**
- Optimize code
- เก็บ safety checks (ป้องกัน undefined behavior)
- Panic แทนที่จะเป็น undefined behavior
- เร็วกว่า Debug แต่ช้ากว่า ReleaseFast

**เหมาะสำหรับ:** Production ที่ต้องการ safety

### ReleaseFast Mode

```zig
const optimize = std.builtin.OptimizeMode.ReleaseFast;
```

**คุณสมบัติ:**
- Optimize สูงสุด
- ปิด safety checks
- เร็วที่สุด
- Undefined behavior อาจเกิดขึ้นถ้า code มีข้อผิดพลาด

**เหมาะสำหรับ:** Performance-critical applications

### ReleaseSmall Mode

```zig
const optimize = std.builtin.OptimizeMode.ReleaseSmall;
```

**คุณสมบัติ:**
- Optimize เพื่อขนาดไฟล์เล็ก
- ปิด safety checks
- เหมาะสำหรับ embedded systems

**เหมาะสำหรับ:** Embedded, WebAssembly, ที่ขนาดไฟล์สำคัญ

### การเลือก Build Mode ใน build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // ให้ผู้ใช้เลือกเองจาก command line
    const optimize = b.standardOptimizeOption(.{});

    // หรือกำหนดค่าเริ่มต้น
    const optimize_with_default = b.standardOptimizeOption(.{
        .preferred_optimize_mode = .ReleaseSafe,
    });

    _ = optimize_with_default;

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = b.standardTargetOptions(.{}),
        .optimize = optimize,
    });

    b.installArtifact(exe);
}
```

### การเลือก Mode จาก Command Line

```bash
# Debug (ค่าเริ่มต้น)
zig build

# ReleaseSafe
zig build -Doptimize=ReleaseSafe

# ReleaseFast
zig build -Doptimize=ReleaseFast

# ReleaseSmall
zig build -Doptimize=ReleaseSmall
```

### เปรียบเทียบขนาดไฟล์

```zig
// โปรแกรม Hello World
const std = @import("std");
pub fn main() void {
    std.debug.print("Hello, World!\n", .{});
}
```

```bash
# Debug: ~800KB
zig build                    # zig-out/bin/hello = ~800KB

# ReleaseSafe: ~200KB
zig build -Doptimize=ReleaseSafe   # zig-out/bin/hello = ~200KB

# ReleaseFast: ~150KB
zig build -Doptimize=ReleaseFast   # zig-out/bin/hello = ~150KB

# ReleaseSmall: ~10KB
zig build -Doptimize=ReleaseSmall  # zig-out/bin/hello = ~10KB
```

### Safety Checks ที่แตกต่างกัน

```zig
const std = @import("std");

pub fn main() void {
    // Integer overflow
    var x: u8 = 255;
    x += 1;  // Debug/ReleaseSafe: panic!
             // ReleaseFast/ReleaseSmall: wraps to 0 (undefined behavior)

    // Array out of bounds
    var arr = [_]i32{ 1, 2, 3 };
    _ = arr[10];  // Debug/ReleaseSafe: panic!
                  // ReleaseFast/ReleaseSmall: undefined behavior

    // Null pointer dereference
    var ptr: ?*i32 = null;
    _ = ptr.?;  // Debug/ReleaseSafe: panic!
                // ReleaseFast/ReleaseSmall: crash (likely)
}
```

---

## Step 407: Dependencies ระหว่าง build steps

Build steps สามารถ depend กันได้ ทำให้การ build มีลำดับที่ถูกต้อง

### dependOn พื้นฐาน

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    // สร้าง run step
    const run_cmd = b.addRunArtifact(exe);

    // run step ต้องรอ install step เสร็จก่อน
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

### สร้าง Custom Step Dependencies

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Step 1: Generate code
    const generate_step = b.step("generate", "Generate source files");

    // Step 2: Build (ต้องรอ generate)
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // exe ต้องการ generate step ก่อน
    exe.step.dependOn(generate_step);

    b.installArtifact(exe);
}
```

### การ build แบบ Parallel

Zig Build System รัน steps แบบ parallel ถ้า steps ไม่ depend กัน

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สอง executables นี้จะ compile แบบ parallel
    const server = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("src/server.zig"),
        .target = target,
        .optimize = optimize,
    });

    const client = b.addExecutable(.{
        .name = "client",
        .root_source_file = b.path("src/client.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ทั้งสองจะ build พร้อมกัน
    b.installArtifact(server);
    b.installArtifact(client);
}
```

### Dependency Graph

```
install step
├── server (build แบบ parallel)
│   └── src/server.zig
└── client (build แบบ parallel)
    └── src/client.zig

run step
└── depends on install step
    └── server executable
```

---

## Step 408: Custom build steps

เราสร้าง build steps เองได้สำหรับงานที่ไม่ใช่แค่ compile

### สร้าง Custom Step ด้วย makeStep

```zig
const std = @import("std");

// Custom step function
fn cleanStep(step: *std.Build.Step, prog_node: std.Progress.Node) !void {
    _ = prog_node;
    const b = step.owner;

    // ลบ zig-out directory
    std.fs.cwd().deleteTree("zig-out") catch |err| {
        if (err != error.FileNotFound) return err;
    };

    // ลบ zig-cache directory
    std.fs.cwd().deleteTree(".zig-cache") catch |err| {
        if (err != error.FileNotFound) return err;
    };

    std.debug.print("Cleaned build artifacts\n", .{});
    _ = b;
}

pub fn build(b: *std.Build) void {
    // สร้าง custom clean step
    const clean = b.step("clean", "Remove build artifacts");
    clean.makeFn = cleanStep;

    // build ปกติ
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);
}
```

### สร้าง Custom Step แบบ OOP

```zig
const std = @import("std");

// Custom step struct
const FormatStep = struct {
    step: std.Build.Step,
    source_files: []const []const u8,

    pub fn create(b: *std.Build, source_files: []const []const u8) *FormatStep {
        const self = b.allocator.create(FormatStep) catch @panic("OOM");
        self.* = .{
            .step = std.Build.Step.init(.{
                .id = .custom,
                .name = "format",
                .owner = b,
                .makeFn = make,
            }),
            .source_files = source_files,
        };
        return self;
    }

    fn make(step: *std.Build.Step, prog_node: std.Progress.Node) !void {
        _ = prog_node;
        const self: *FormatStep = @fieldParentPtr("step", step);

        for (self.source_files) |file| {
            std.debug.print("Formatting: {s}\n", .{file});
            // ในการใช้จริง: รัน zig fmt
        }
    }
};

pub fn build(b: *std.Build) void {
    const source_files = &[_][]const u8{
        "src/main.zig",
        "src/lib.zig",
    };

    const fmt_step = FormatStep.create(b, source_files);

    const format = b.step("fmt", "Format source files");
    format.dependOn(&fmt_step.step);
}
```

### รัน External Commands

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // รัน external command (เช่น code generation)
    const gen_cmd = b.addSystemCommand(&[_][]const u8{
        "python3",
        "scripts/generate.py",
        "--output",
        "src/generated.zig",
    });

    // exe ต้องรอ gen_cmd เสร็จก่อน
    exe.step.dependOn(&gen_cmd.step);
}
```

---

## Step 409: zig build run, zig build test

### zig build - คำสั่งพื้นฐาน

```bash
# Build ทั้งหมด (default step)
zig build

# Build พร้อมเลือก options
zig build -Doptimize=ReleaseFast

# Build สำหรับ platform อื่น
zig build -Dtarget=x86_64-linux

# Build แบบ verbose (เห็น commands ที่รัน)
zig build --verbose

# Build พร้อม summary
zig build --summary all
```

### zig build run

```bash
# รัน default run step
zig build run

# ส่ง arguments
zig build run -- --help
zig build run -- --port 8080 --host localhost

# รัน specific run step
zig build server
zig build client
```

### zig build test

```bash
# รัน tests ทั้งหมด
zig build test

# รัน tests แบบ verbose
zig build test --summary all

# กำหนด filter สำหรับ test
zig build test -- --test-filter "math"
```

### การตั้งค่า Test ใน build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ========== Main executable ==========
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // ========== Unit Tests ==========
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);

    // ========== Integration Tests ==========
    const integration_tests = b.addTest(.{
        .root_source_file = b.path("tests/integration.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_integration_tests = b.addRunArtifact(integration_tests);

    // ========== Test Step ==========
    const test_step = b.step("test", "Run all tests");
    test_step.dependOn(&run_unit_tests.step);
    test_step.dependOn(&run_integration_tests.step);

    // หรือแยก test steps
    const unit_step = b.step("test-unit", "Run unit tests only");
    unit_step.dependOn(&run_unit_tests.step);

    const integration_step = b.step("test-integration", "Run integration tests only");
    integration_step.dependOn(&run_integration_tests.step);
}
```

```zig
// src/main.zig - with tests
const std = @import("std");

pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn factorial(n: u32) u32 {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

pub fn main() void {
    std.debug.print("5! = {}\n", .{factorial(5)});
    std.debug.print("3 + 4 = {}\n", .{add(3, 4)});
}

// Tests
test "add function" {
    try std.testing.expectEqual(@as(i32, 7), add(3, 4));
    try std.testing.expectEqual(@as(i32, 0), add(-5, 5));
    try std.testing.expectEqual(@as(i32, -10), add(-3, -7));
}

test "factorial" {
    try std.testing.expectEqual(@as(u32, 1), factorial(0));
    try std.testing.expectEqual(@as(u32, 1), factorial(1));
    try std.testing.expectEqual(@as(u32, 120), factorial(5));
    try std.testing.expectEqual(@as(u32, 720), factorial(6));
}
```

---

## Step 410: สรุปโครงสร้างโปรเจค Zig

### โครงสร้างโปรเจคที่แนะนำ

```
my-zig-project/
├── build.zig              # Build script
├── build.zig.zon          # Package manifest
├── src/
│   ├── main.zig           # Entry point
│   ├── root.zig           # Root module (สำหรับ library)
│   ├── config.zig         # Configuration
│   └── utils/
│       ├── string.zig
│       └── math.zig
├── tests/
│   ├── unit/
│   │   ├── test_math.zig
│   │   └── test_string.zig
│   └── integration/
│       └── test_app.zig
├── examples/
│   ├── basic.zig
│   └── advanced.zig
├── docs/
│   └── api.md
└── README.md
```

### build.zig สมบูรณ์สำหรับโปรเจคจริง

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // ==========================================
    // Standard Options
    // ==========================================
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ==========================================
    // Library Module
    // ==========================================
    const lib_mod = b.addModule("mylib", .{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ==========================================
    // Main Executable
    // ==========================================
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่ม module dependency
    exe.root_module.addImport("mylib", lib_mod);

    b.installArtifact(exe);

    // ==========================================
    // Run Step
    // ==========================================
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);

    // ==========================================
    // Tests
    // ==========================================
    const lib_tests = b.addTest(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_lib_tests = b.addRunArtifact(lib_tests);

    const exe_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe_tests.root_module.addImport("mylib", lib_mod);

    const run_exe_tests = b.addRunArtifact(exe_tests);

    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_lib_tests.step);
    test_step.dependOn(&run_exe_tests.step);
}
```

### build.zig.zon พื้นฐาน

```zig
// build.zig.zon
.{
    .name = "my-zig-project",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    .dependencies = .{
        // dependencies จะอยู่ที่นี่
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

### คำสั่ง build ทั้งหมดที่ควรรู้

```bash
# ===== Build Commands =====
zig build                    # Build ด้วย default settings
zig build -Doptimize=Debug   # Build ใน Debug mode
zig build -Doptimize=ReleaseSafe   # Build ใน ReleaseSafe
zig build -Doptimize=ReleaseFast   # Build ใน ReleaseFast
zig build -Doptimize=ReleaseSmall  # Build ใน ReleaseSmall

# ===== Target Commands =====
zig build -Dtarget=x86_64-linux-gnu    # Linux x86_64
zig build -Dtarget=aarch64-macos       # macOS ARM
zig build -Dtarget=x86_64-windows-gnu  # Windows x86_64
zig build -Dtarget=wasm32-wasi         # WebAssembly

# ===== Run Commands =====
zig build run                  # รัน default run step
zig build run -- arg1 arg2     # ส่ง arguments
zig build server               # รัน custom step

# ===== Test Commands =====
zig build test                 # รัน tests ทั้งหมด
zig build test-unit            # รัน unit tests เท่านั้น

# ===== Utility Commands =====
zig build --help               # แสดง help
zig build --list-steps         # แสดง steps ทั้งหมด
zig build --verbose            # verbose output
zig build --summary all        # สรุปผล build
zig build -j4                  # ใช้ 4 threads

# ===== Format Commands =====
zig fmt src/                   # format ทุกไฟล์ใน src/
zig fmt --check src/           # ตรวจสอบ format โดยไม่แก้ไข
```

### สรุป Best Practices

1. **ใช้ standardTargetOptions และ standardOptimizeOption เสมอ** - ให้ผู้ใช้เลือก options ได้
2. **แยก steps อย่างชัดเจน** - run, test, format, clean
3. **ใช้ modules สำหรับ code reuse** - addModule ดีกว่าการ copy code
4. **กำหนด dependencies ให้ถูกต้อง** - ป้องกัน race conditions
5. **เขียน tests คู่กับ code** - Zig รองรับ inline tests ได้ดี

### ตัวอย่างโปรเจคจริง: Calculator

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Math library
    const math_lib = b.addStaticLibrary(.{
        .name = "math",
        .root_source_file = b.path("src/math.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Calculator executable
    const calc = b.addExecutable(.{
        .name = "calculator",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    calc.linkLibrary(math_lib);
    b.installArtifact(calc);

    // Run
    const run = b.addRunArtifact(calc);
    run.step.dependOn(b.getInstallStep());
    if (b.args) |args| run.addArgs(args);
    b.step("run", "Run calculator").dependOn(&run.step);

    // Tests
    const tests = b.addTest(.{
        .root_source_file = b.path("src/math.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.step("test", "Run tests").dependOn(&b.addRunArtifact(tests).step);
}
```

```zig
// src/math.zig
const std = @import("std");

pub const MathError = error{ DivisionByZero, Overflow };

pub fn add(a: f64, b: f64) f64 { return a + b; }
pub fn subtract(a: f64, b: f64) f64 { return a - b; }
pub fn multiply(a: f64, b: f64) f64 { return a * b; }

pub fn divide(a: f64, b: f64) MathError!f64 {
    if (b == 0.0) return MathError.DivisionByZero;
    return a / b;
}

test "basic arithmetic" {
    try std.testing.expectEqual(@as(f64, 5.0), add(2.0, 3.0));
    try std.testing.expectEqual(@as(f64, -1.0), subtract(2.0, 3.0));
    try std.testing.expectEqual(@as(f64, 6.0), multiply(2.0, 3.0));
    try std.testing.expectEqual(@as(f64, 2.5), try divide(5.0, 2.0));
}

test "division by zero" {
    try std.testing.expectError(MathError.DivisionByZero, divide(1.0, 0.0));
}
```

```zig
// src/main.zig
const std = @import("std");
const math = @import("math.zig");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    try stdout.print("Calculator\n", .{});
    try stdout.print("5 + 3 = {d}\n", .{math.add(5, 3)});
    try stdout.print("10 - 4 = {d}\n", .{math.subtract(10, 4)});
    try stdout.print("6 * 7 = {d}\n", .{math.multiply(6, 7)});

    const result = math.divide(15, 3) catch |err| {
        try stdout.print("Error: {}\n", .{err});
        return;
    };
    try stdout.print("15 / 3 = {d}\n", .{result});
}
```

---

## สรุป Part 41: Build System พื้นฐาน

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 401 | build.zig คืออะไร | ระบบ build ของ Zig |
| 402 | โครงสร้าง build.zig | Function signature, options |
| 403 | addExecutable/Library | สร้าง artifacts ต่างๆ |
| 404 | addRunArtifact | การรัน programs |
| 405 | addInstallArtifact | การ install artifacts |
| 406 | Build modes | Debug, Release modes |
| 407 | Dependencies | Step dependencies |
| 408 | Custom steps | สร้าง steps เอง |
| 409 | zig build commands | คำสั่ง build ต่างๆ |
| 410 | สรุปโครงสร้าง | Best practices |

> **ต่อไป**: Part 42 - Build System ขั้นสูง (Package management, Cross-compilation)
