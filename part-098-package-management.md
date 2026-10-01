# Part 98: Package Management ใน Zig
## Steps 971-980

---

## Step 971: Zig Package System Overview

### ภาพรวมของ Zig Package System

Zig 0.12+ มี package manager ในตัวที่ใช้ไฟล์ `build.zig.zon` สำหรับจัดการ dependencies

### โครงสร้าง Package ใน Zig

```
my-project/
├── build.zig          # Build script
├── build.zig.zon      # Package manifest
├── src/
│   ├── main.zig       # Main source file
│   └── lib.zig        # Library code
└── tests/
    └── test.zig       # Tests
```

### build.zig.zon พื้นฐาน

```zon
// build.zig.zon
.{
    .name = "my-project",
    .version = "0.1.0",
    .minimum_zig_version = "0.13.0",
    
    .dependencies = .{
        .zig_clap = .{
            .url = "https://github.com/Hejsil/zig-clap/archive/0.9.1.tar.gz",
            .hash = "122053da74bbc0d8d613e0cba36a7d6e3b1a2f0e9cf5d8cbce6de58b72539c23b9",
        },
    },
    
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "README.md",
    },
}
```

### build.zig สำหรับใช้ Dependencies

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // ดึง dependency
    const clap = b.dependency("zig_clap", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "my-app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // เพิ่ม dependency เข้า executable
    exe.root_module.addImport("clap", clap.module("clap"));
    
    b.installArtifact(exe);
    
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    
    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);
}
```

### การใช้ Package ใน Code

```zig
// main.zig - ใช้ dependency
const std = @import("std");
const clap = @import("clap");

pub fn main() !void {
    const params = comptime clap.parseParamsComptime(
        \\-h, --help    Display this help and exit.
        \\-n, --name <str>   Name to greet.
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
    
    if (res.args.help != 0) {
        return clap.help(std.io.getStdErr().writer(), clap.Help, &params, .{});
    }
    
    const name = res.args.name orelse "World";
    std.debug.print("Hello, {s}!\n", .{name});
}
```

---

## Step 972: build.zig.zon Anatomy

### โครงสร้างละเอียดของ build.zig.zon

```zon
// build.zig.zon - full example
.{
    // ชื่อ package (lowercase, dash-separated)
    .name = "my-awesome-lib",
    
    // Semantic version
    .version = "1.2.3",
    
    // Minimum Zig version ที่รองรับ
    .minimum_zig_version = "0.13.0",
    
    // Dependencies
    .dependencies = .{
        // Named dependency จาก URL
        .httpz = .{
            .url = "https://github.com/karlseguin/http.zig/archive/refs/tags/v0.5.0.tar.gz",
            .hash = "1220abc123...", // SHA256 hash
        },
        
        // Dependency จาก git commit
        .zap = .{
            .url = "https://github.com/zigzap/zap/archive/v0.7.0.tar.gz",
            .hash = "1220def456...",
            // Optional: lazy loading
            .lazy = true,
        },
        
        // Local path dependency
        .my_utils = .{
            .path = "../my-utils",
        },
    },
    
    // Files ที่ include เมื่อ publish
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
        "README.md",
        "LICENSE",
        "docs",
    },
}
```

### Hash Verification

```bash
# รับ hash สำหรับ URL
zig fetch https://github.com/user/repo/archive/v1.0.0.tar.gz

# Output จะแสดง hash:
# info: Fetching https://...
# 1220abc123def456...

# เพิ่มใน build.zig.zon
.my_dep = .{
    .url = "https://...",
    .hash = "1220abc123def456...",
},
```

### Lazy Dependencies

```zig
// build.zig - lazy dependency
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Lazy dependency จะถูก fetch เฉพาะเมื่อต้องการ
    const enable_crypto = b.option(bool, "crypto", "Enable cryptography support") orelse false;
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    if (enable_crypto) {
        // Lazy dependency ถูก fetch เฉพาะ -Dcrypto=true
        const crypto = b.lazyDependency("zcrypto", .{
            .target = target,
            .optimize = optimize,
        }) orelse return; // null ถ้ายังไม่ได้ fetch
        
        exe.root_module.addImport("zcrypto", crypto.module("zcrypto"));
    }
    
    b.installArtifact(exe);
}
```

### Fields ของ Dependency

```zig
// dependency_fields.zig
// build.zig ที่แสดงการใช้งาน dependency fields ต่างๆ

const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const dep = b.dependency("some_lib", .{
        .target = target,
        .optimize = optimize,
        // Options เพิ่มเติม ขึ้นกับ library
        .enable_feature = true,
        .log_level = .info,
    });
    
    // ดึง module จาก dependency
    const some_lib_module = dep.module("some_lib");
    
    // ดึง artifact (compiled library)
    const some_lib_artifact = dep.artifact("some_lib");
    
    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    exe.root_module.addImport("some_lib", some_lib_module);
    exe.linkLibrary(some_lib_artifact);
    
    b.installArtifact(exe);
}
```

---

## Step 973: zig fetch กับ URL

### การใช้ zig fetch

```bash
# Fetch และ print hash (ไม่บันทึก)
zig fetch https://github.com/user/repo/archive/v1.0.0.tar.gz

# Fetch และบันทึกใน build.zig.zon โดยอัตโนมัติ
zig fetch --save https://github.com/user/repo/archive/v1.0.0.tar.gz

# Fetch ด้วยชื่อ custom
zig fetch --save=mylib https://github.com/user/repo/archive/v1.0.0.tar.gz

# Fetch จาก git URL
zig fetch --save git+https://github.com/user/repo#main

# Fetch จาก specific commit
zig fetch --save git+https://github.com/user/repo#abc123def456
```

### Fetch จาก URL ต่างๆ

```bash
# GitHub Release
zig fetch --save https://github.com/user/repo/archive/refs/tags/v1.0.0.tar.gz

# GitHub Release (tarball)
zig fetch --save https://github.com/user/repo/releases/download/v1.0.0/mylib-v1.0.0.tar.gz

# Codeberg
zig fetch --save https://codeberg.org/user/repo/archive/v1.0.0.tar.gz

# GitLab
zig fetch --save https://gitlab.com/user/repo/-/archive/v1.0.0/repo-v1.0.0.tar.gz

# Direct tarball
zig fetch --save https://example.com/mylib-1.0.0.tar.gz
```

### การจัดการ Hash

```zig
// hash_management.zig
// ตัวอย่างการแสดง hash ที่ถูกต้องใน build.zig.zon

// Hash format: "1220" + 64 hex characters (SHA256)
// ตัวอย่าง:
// "122053da74bbc0d8d613e0cba36a7d6e3b1a2f0e9cf5d8cbce6de58b72539c23b9"
//   ^^^^                                                               
//   1220 = content hash prefix

// ถ้า hash ผิด Zig จะ error:
// error: expected .hash = "1220...", found ...

// สามารถ verify hash ด้วย:
// zig fetch <url> - จะ print hash ถ้าสำเร็จ
```

### Script สำหรับอัปเดต Dependencies

```bash
#!/bin/bash
# update_deps.sh - Script สำหรับอัปเดต dependencies

set -e

echo "Updating dependencies..."

# อัปเดต specific dependency
update_dep() {
    local name=$1
    local url=$2
    echo "Updating $name from $url..."
    zig fetch --save="$name" "$url"
}

# อัปเดต dependencies
update_dep "httpz" "https://github.com/karlseguin/http.zig/archive/refs/tags/v0.6.0.tar.gz"
update_dep "zap" "https://github.com/zigzap/zap/archive/v0.8.0.tar.gz"

echo "Done! Review build.zig.zon for changes."
```

---

## Step 974: Local Packages

### การสร้างและใช้ Local Packages

```zig
// local_package/build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // สร้าง module สำหรับ local package
    _ = b.addModule("myutils", .{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
}
```

```zig
// local_package/src/root.zig
const std = @import("std");

/// String utilities
pub const strings = @import("strings.zig");

/// Math utilities
pub const math = @import("math.zig");

/// Array utilities
pub const arrays = @import("arrays.zig");

test {
    std.testing.refAllDecls(@This());
}
```

```zon
// main_project/build.zig.zon
.{
    .name = "main-project",
    .version = "0.1.0",
    .dependencies = .{
        // Local package โดย path
        .myutils = .{
            .path = "../local_package",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

```zig
// main_project/build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // ใช้ local dependency
    const myutils = b.dependency("myutils", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "main",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    exe.root_module.addImport("myutils", myutils.module("myutils"));
    b.installArtifact(exe);
}
```

### Monorepo Structure

```
monorepo/
├── build.zig
├── build.zig.zon
├── packages/
│   ├── core/
│   │   ├── build.zig
│   │   └── src/
│   ├── http/
│   │   ├── build.zig
│   │   └── src/
│   └── cli/
│       ├── build.zig
│       └── src/
└── apps/
    ├── server/
    │   ├── build.zig
    │   └── src/
    └── client/
        ├── build.zig
        └── src/
```

```zig
// monorepo/build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // Build internal packages
    const core = buildPackage(b, "core", target, optimize);
    const http = buildPackage(b, "http", target, optimize);
    
    // Server app depends on core and http
    const server = b.addExecutable(.{
        .name = "server",
        .root_source_file = b.path("apps/server/src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    server.root_module.addImport("core", core);
    server.root_module.addImport("http", http);
    b.installArtifact(server);
}

fn buildPackage(
    b: *std.Build,
    name: []const u8,
    target: std.Build.ResolvedTarget,
    optimize: std.builtin.OptimizeMode,
) *std.Build.Module {
    return b.addModule(name, .{
        .root_source_file = b.path(b.fmt("packages/{s}/src/root.zig", .{name})),
        .target = target,
        .optimize = optimize,
    });
}
```

---

## Step 975: Git Packages

### การใช้ Git Packages

```bash
# Fetch จาก git repository (default branch)
zig fetch --save git+https://github.com/user/repo

# Fetch จาก specific branch
zig fetch --save git+https://github.com/user/repo#main

# Fetch จาก specific tag
zig fetch --save git+https://github.com/user/repo#v1.0.0

# Fetch จาก specific commit
zig fetch --save git+https://github.com/user/repo#abc123

# ใช้กับ private repository (ต้องมี SSH key หรือ token)
zig fetch --save git+git@github.com:user/private-repo.git#main
```

### build.zig.zon สำหรับ Git Dependencies

```zon
// build.zig.zon
.{
    .name = "my-app",
    .version = "0.1.0",
    .dependencies = .{
        // Git dependency พร้อม hash
        .zig_uuid = .{
            .url = "git+https://github.com/r4gus/uuid-zig#0.3.0",
            .hash = "1220abc...",
        },
        
        // Tarball อาจดีกว่า git clone เพราะ:
        // - เร็วกว่า (ไม่ต้อง clone ทั้ง history)
        // - Reproducible ง่ายกว่า
        // - ไม่ต้องการ git binary
        .zig_network = .{
            .url = "https://github.com/MasterQ32/zig-network/archive/refs/heads/master.tar.gz",
            .hash = "1220def...",
        },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

### Pinning vs Floating Dependencies

```zon
// pinned_dependencies.zig.zon - Reproducible builds
.{
    .dependencies = .{
        // Pinned to specific version - reproducible
        .lib_a = .{
            .url = "https://github.com/user/lib-a/archive/refs/tags/v2.1.3.tar.gz",
            .hash = "1220abc...",
        },
    },
}

// floating_dependencies.zig.zon - อาจไม่ reproducible
.{
    .dependencies = .{
        // Floating to HEAD - อาจเปลี่ยนแปลงได้
        .lib_b = .{
            .url = "git+https://github.com/user/lib-b#main",
            .hash = "1220def...", // hash จะเปลี่ยนเมื่อ main เปลี่ยน
        },
    },
}
```

---

## Step 976: Package Versioning

### Semantic Versioning ใน Zig

```zig
// versioning.zig
const std = @import("std");

pub const Version = struct {
    major: u32,
    minor: u32,
    patch: u32,
    pre: ?[]const u8 = null,
    build: ?[]const u8 = null,
    
    pub fn parse(s: []const u8) !Version {
        // ใช้ std.SemanticVersion
        const sv = try std.SemanticVersion.parse(s);
        return Version{
            .major = sv.major,
            .minor = sv.minor,
            .patch = sv.patch,
            .pre = sv.pre,
            .build = sv.build,
        };
    }
    
    pub fn isCompatible(self: Version, requirement: Version) bool {
        // Major version ต้องตรงกัน
        if (self.major != requirement.major) return false;
        
        // Minor version ต้อง >= requirement
        if (self.minor < requirement.minor) return false;
        
        // ถ้า minor เท่ากัน patch ต้อง >=
        if (self.minor == requirement.minor and self.patch < requirement.patch) return false;
        
        return true;
    }
    
    pub fn format(
        self: Version,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{d}.{d}.{d}", .{ self.major, self.minor, self.patch });
        if (self.pre) |pre| try writer.print("-{s}", .{pre});
        if (self.build) |build| try writer.print("+{s}", .{build});
    }
};

// Versioning rules:
// 1.0.0 -> ออก
// 1.1.0 -> เพิ่ม feature (backwards compatible)
// 1.1.1 -> bug fix
// 2.0.0 -> breaking change

pub fn checkMinimumVersion(
    current: std.SemanticVersion,
    minimum: std.SemanticVersion,
) !void {
    const order = current.order(minimum);
    if (order == .lt) {
        std.debug.print(
            "Error: ต้องการ Zig >= {}, แต่มี {}\n",
            .{ minimum, current },
        );
        return error.VersionTooOld;
    }
}
```

### Version Constraints

```zig
// version_constraints.zig
const std = @import("std");

pub const Constraint = struct {
    kind: Kind,
    version: std.SemanticVersion,
    
    pub const Kind = enum {
        exact,       // == 1.2.3
        compatible,  // ^1.2.3 (>= 1.2.3, < 2.0.0)
        minor,       // ~1.2.3 (>= 1.2.3, < 1.3.0)
        greater,     // > 1.2.3
        greater_eq,  // >= 1.2.3
        less,        // < 1.2.3
        less_eq,     // <= 1.2.3
    };
    
    pub fn parse(s: []const u8) !Constraint {
        if (std.mem.startsWith(u8, s, "^")) {
            return .{
                .kind = .compatible,
                .version = try std.SemanticVersion.parse(s[1..]),
            };
        } else if (std.mem.startsWith(u8, s, "~")) {
            return .{
                .kind = .minor,
                .version = try std.SemanticVersion.parse(s[1..]),
            };
        } else if (std.mem.startsWith(u8, s, ">=")) {
            return .{
                .kind = .greater_eq,
                .version = try std.SemanticVersion.parse(s[2..]),
            };
        } else if (std.mem.startsWith(u8, s, ">")) {
            return .{
                .kind = .greater,
                .version = try std.SemanticVersion.parse(s[1..]),
            };
        } else {
            return .{
                .kind = .exact,
                .version = try std.SemanticVersion.parse(s),
            };
        }
    }
    
    pub fn matches(self: Constraint, version: std.SemanticVersion) bool {
        return switch (self.kind) {
            .exact => version.order(self.version) == .eq,
            .compatible => version.order(self.version) != .lt and
                version.major == self.version.major,
            .minor => version.order(self.version) != .lt and
                version.major == self.version.major and
                version.minor == self.version.minor,
            .greater => version.order(self.version) == .gt,
            .greater_eq => version.order(self.version) != .lt,
            .less => version.order(self.version) == .lt,
            .less_eq => version.order(self.version) != .gt,
        };
    }
};
```

---

## Step 977: Package Namespacing

### การจัดการ Namespaces

```zig
// namespacing.zig
const std = @import("std");

// การ import และ namespace
const mylib = @import("mylib");
const strings = mylib.strings;
const math = mylib.math;

pub fn main() void {
    // ใช้ qualified names
    const result = math.add(1, 2);
    const upper = strings.toUpper("hello");
    
    std.debug.print("{d}\n", .{result});
    std.debug.print("{s}\n", .{upper});
}
```

### การ Structure Module สำหรับ Namespacing ที่ดี

```zig
// mylib/src/root.zig - Entry point ของ library
const std = @import("std");

// Re-export modules
pub const strings = @import("strings.zig");
pub const math = @import("math.zig");
pub const collections = @import("collections.zig");
pub const io = @import("io.zig");

// Re-export commonly used types ตรงๆ
pub const ArrayList = std.ArrayList;
pub const HashMap = std.AutoHashMap;

// Version info
pub const version = std.SemanticVersion{ .major = 1, .minor = 0, .patch = 0 };

test {
    // รัน tests ทั้งหมดใน sub-modules
    _ = strings;
    _ = math;
    _ = collections;
    _ = io;
}
```

```zig
// mylib/src/strings.zig
const std = @import("std");

/// แปลง string เป็น uppercase
pub fn toUpper(s: []const u8) []const u8 {
    // implementation
    _ = s;
    return "";
}

/// ตรวจสอบว่า string เริ่มต้นด้วย prefix
pub fn startsWith(s: []const u8, prefix: []const u8) bool {
    return std.mem.startsWith(u8, s, prefix);
}

/// Split string ด้วย delimiter
pub fn split(
    s: []const u8,
    delimiter: u8,
    allocator: std.mem.Allocator,
) ![][]const u8 {
    var result = std.ArrayList([]const u8).init(allocator);
    var iter = std.mem.splitScalar(u8, s, delimiter);
    while (iter.next()) |part| {
        try result.append(part);
    }
    return result.toOwnedSlice();
}
```

### Avoiding Name Conflicts

```zig
// avoid_conflicts.zig
const std = @import("std");

// ถ้า dependencies มีชื่อ module ซ้ำ ต้องใช้ aliasing
const lib_a = @import("lib_a");
const lib_b = @import("lib_b");

// ถ้าทั้งคู่มี .utils module
const utils_a = lib_a.utils;
const utils_b = lib_b.utils;

pub fn main() void {
    utils_a.doSomething();
    utils_b.doSomething();
}

// build.zig - กำหนดชื่อ import ที่ไม่ซ้ำกัน
const BuildFn = fn (*std.Build) void;
pub const build: BuildFn = struct {
    pub fn build(b: *std.Build) void {
        const target = b.standardTargetOptions(.{});
        const optimize = b.standardOptimizeOption(.{});
        
        const dep_a = b.dependency("lib_a", .{ .target = target, .optimize = optimize });
        const dep_b = b.dependency("lib_b", .{ .target = target, .optimize = optimize });
        
        const exe = b.addExecutable(.{
            .name = "app",
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        });
        
        // ใช้ชื่อ import ที่แตกต่างกัน
        exe.root_module.addImport("lib_a", dep_a.module("lib_a"));
        exe.root_module.addImport("lib_b", dep_b.module("lib_b"));
    }
}.build;
```

---

## Step 978: Publishing Packages

### การ Publish Package

```bash
# 1. สร้าง tag ใน git
git tag v1.0.0
git push origin v1.0.0

# 2. สร้าง GitHub Release (ถ้าใช้ GitHub)
gh release create v1.0.0 --title "v1.0.0" --notes "Initial release"

# 3. ตรวจสอบว่า package สามารถ fetch ได้
zig fetch https://github.com/user/mylib/archive/refs/tags/v1.0.0.tar.gz
```

### Package Manifest ที่ครบถ้วน

```zon
// complete_package.zig.zon
.{
    .name = "zig-awesome-lib",
    .version = "1.0.0",
    .minimum_zig_version = "0.13.0",
    
    .dependencies = .{
        // ไม่มี dependencies ภายนอก
    },
    
    // ต้องระบุ paths เพื่อให้ package size เล็ก
    .paths = .{
        // Build files
        "build.zig",
        "build.zig.zon",
        
        // Source code
        "src",
        
        // Documentation
        "README.md",
        "LICENSE",
        
        // Optional: examples
        "examples",
        
        // Optional: docs
        "docs",
    },
}
```

### build.zig สำหรับ Library Package

```zig
// library_build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // สร้าง module (สำหรับ package consumers)
    const mod = b.addModule("awesome_lib", .{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // สร้าง static library (สำหรับ C interop)
    const lib = b.addStaticLibrary(.{
        .name = "awesome_lib",
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(lib);
    
    // Install header สำหรับ C consumers
    lib.installHeader(b.path("include/awesome_lib.h"), "awesome_lib.h");
    
    // Tests
    const tests = b.addTest(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    const run_tests = b.addRunArtifact(tests);
    
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_tests.step);
    
    // Documentation
    const docs = b.addInstallDirectory(.{
        .source_dir = lib.getEmittedDocs(),
        .install_dir = .prefix,
        .install_subdir = "docs",
    });
    
    const docs_step = b.step("docs", "Generate documentation");
    docs_step.dependOn(&docs.step);
    
    // Unused module warning fix
    _ = mod;
}
```

---

## Step 979: Community Packages

### แหล่งหา Zig Packages

**1. ziglings ecosystem**: https://github.com/ratfactor/ziglings  
**2. astrolabe**: Package index (unofficial)  
**3. GitHub Topics**: github.com/topics/zig-package  
**4. Discord**: Zig programming Discord server

### Popular Packages

```zig
// popular_packages.zig - ตัวอย่างการใช้ popular packages

// zig-clap: Command line argument parsing
// https://github.com/Hejsil/zig-clap
const clap = @import("clap");

// httpz: HTTP server/client
// https://github.com/karlseguin/http.zig
const httpz = @import("httpz");

// zap: Fast HTTP server (wraps facil.io)
// https://github.com/zigzap/zap
const zap = @import("zap");

// sqlite: SQLite bindings
// https://github.com/vrischmann/zig-sqlite
const sqlite = @import("sqlite");

// mach: Game engine / multimedia
// https://github.com/hexops/mach
const mach = @import("mach");

// zzz: HTTP server
// https://github.com/tarkin000/zzz
const zzz = @import("zzz");

// Example: ใช้ zig-clap
pub fn clapExample() !void {
    const params = comptime clap.parseParamsComptime(
        \\-h, --help     Display help
        \\-v, --verbose  Enable verbose output
        \\<str>...       Input files
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
    
    if (res.args.help != 0) {
        try clap.help(std.io.getStdErr().writer(), clap.Help, &params, .{});
        return;
    }
    
    if (res.args.verbose != 0) {
        std.debug.print("Verbose mode enabled\n", .{});
    }
    
    for (res.positionals) |file| {
        std.debug.print("Processing: {s}\n", .{file});
    }
}

const std = @import("std");
```

### การ Evaluate Package Quality

```zig
// package_evaluation.zig
// Checklist สำหรับ evaluate package ก่อนใช้

// 1. Maintenance
//    - มี recent commits หรือไม่? (ภายใน 6 เดือน)
//    - มี open issues ที่ค้างนานหรือไม่?
//    - Owner ตอบสนอง issues หรือไม่?

// 2. Quality  
//    - มี tests หรือไม่?
//    - CI/CD pass หรือไม่?
//    - มี documentation หรือไม่?

// 3. Compatibility
//    - Compatible กับ Zig version ที่ใช้หรือไม่?
//    - มี build.zig.zon หรือไม่? (ถ้าไม่มีอาจเป็น package เก่า)

// 4. Security
//    - ตรวจสอบ source code ก่อน use
//    - มีแค่ dependencies ที่จำเป็นหรือไม่?

// 5. License
//    - Compatible กับ project ของเราหรือไม่?
//    - MIT, Apache 2.0, BSD - generally permissive
//    - GPL - copyleft, อาจ affect closed source projects
```

---

## Step 980: Dependency Management Best Practices

### Best Practices สำหรับ Dependency Management

```zig
// dep_best_practices.zig
const std = @import("std");

// Practice 1: Minimize dependencies
// ถามตัวเองก่อน: จำเป็นต้องใช้ external package หรือไม่?
// Zig standard library ครอบคลุมมาก

// ตัวอย่าง: แทนที่จะใช้ external JSON parser
// ใช้ std.json ที่มีให้อยู่แล้ว
pub fn parseJsonExample() !void {
    const json_str =
        \\{"name": "Alice", "age": 30}
    ;
    
    const parsed = try std.json.parseFromSlice(
        struct { name: []const u8, age: u32 },
        std.testing.allocator,
        json_str,
        .{},
    );
    defer parsed.deinit();
    
    std.debug.print("Name: {s}\n", .{parsed.value.name});
    std.debug.print("Age: {d}\n", .{parsed.value.age});
}

// Practice 2: Pin dependency versions สำหรับ production
// ใช้ exact versions เพื่อ reproducible builds
//
// .my_dep = .{
//     .url = "...v1.2.3.tar.gz",  // pinned version
//     .hash = "1220...",
// }

// Practice 3: Review dependency source
// ก่อน add dependency ควร:
// - อ่าน source code คร่าวๆ
// - ตรวจสอบว่าไม่มี malicious code
// - ดู history และ contributors

// Practice 4: Keep dependencies updated
// - ตรวจสอบ security advisories
// - อัปเดต เมื่อมี bug fixes สำคัญ
// - Test อย่างละเอียดก่อน update

// Practice 5: Consider vendoring สำหรับ critical projects
// Vendoring = เก็บ dependency source code ไว้ใน repository ของตัวเอง
```

### Vendoring Dependencies

```bash
# สร้าง vendor directory
mkdir -p vendor

# Clone dependencies เข้า vendor
git clone --depth 1 https://github.com/user/dep1 vendor/dep1
git clone --depth 1 https://github.com/user/dep2 vendor/dep2

# ใน build.zig.zon
# .dep1 = .{ .path = "vendor/dep1" }
```

```zig
// vendored_build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    // ใช้ vendored dependencies
    const dep1 = b.dependency("dep1", .{
        .target = target,
        .optimize = optimize,
    });
    
    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    exe.root_module.addImport("dep1", dep1.module("dep1"));
    b.installArtifact(exe);
}
```

### Dependency Audit Script

```bash
#!/bin/bash
# audit_deps.sh - ตรวจสอบ dependencies

echo "=== Dependency Audit ==="

# ตรวจสอบ build.zig.zon
if [ -f build.zig.zon ]; then
    echo "Found build.zig.zon"
    
    # แสดง dependencies
    echo ""
    echo "Dependencies:"
    grep -A 3 '\.url' build.zig.zon | grep -v '#'
    
    # ตรวจสอบว่า hashes ครบ
    echo ""
    if grep -q '\.url' build.zig.zon && ! grep -q '\.hash' build.zig.zon; then
        echo "WARNING: Some dependencies may be missing hash verification!"
    else
        echo "All URL dependencies have hash verification."
    fi
else
    echo "No build.zig.zon found"
fi
```

### Dependency Graph Visualization

```zig
// dep_graph.zig
const std = @import("std");

pub const DependencyGraph = struct {
    nodes: std.StringArrayHashMap(Node),
    allocator: std.mem.Allocator,
    
    const Node = struct {
        name: []const u8,
        version: []const u8,
        deps: std.ArrayList([]const u8),
    };
    
    pub fn init(allocator: std.mem.Allocator) DependencyGraph {
        return .{
            .nodes = std.StringArrayHashMap(Node).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *DependencyGraph) void {
        var it = self.nodes.iterator();
        while (it.next()) |entry| {
            entry.value_ptr.deps.deinit();
        }
        self.nodes.deinit();
    }
    
    pub fn addPackage(
        self: *DependencyGraph,
        name: []const u8,
        version: []const u8,
        deps: []const []const u8,
    ) !void {
        var dep_list = std.ArrayList([]const u8).init(self.allocator);
        for (deps) |d| try dep_list.append(d);
        
        try self.nodes.put(name, .{
            .name = name,
            .version = version,
            .deps = dep_list,
        });
    }
    
    pub fn printTree(self: *const DependencyGraph, root: []const u8, depth: u32) void {
        const indent = "  " ** @min(depth, 10);
        const node = self.nodes.get(root) orelse {
            std.debug.print("{s}[{s}] (not found)\n", .{ indent, root });
            return;
        };
        
        std.debug.print("{s}[{s}@{s}]\n", .{ indent, node.name, node.version });
        
        if (depth < 5) { // ป้องกัน infinite recursion
            for (node.deps.items) |dep| {
                self.printTree(dep, depth + 1);
            }
        }
    }
    
    pub fn detectCircular(self: *const DependencyGraph) !bool {
        var visited = std.StringHashMap(bool).init(self.allocator);
        defer visited.deinit();
        var in_stack = std.StringHashMap(bool).init(self.allocator);
        defer in_stack.deinit();
        
        var it = self.nodes.iterator();
        while (it.next()) |entry| {
            if (!visited.contains(entry.key_ptr.*)) {
                if (try self.dfs(entry.key_ptr.*, &visited, &in_stack)) {
                    return true;
                }
            }
        }
        return false;
    }
    
    fn dfs(
        self: *const DependencyGraph,
        name: []const u8,
        visited: *std.StringHashMap(bool),
        in_stack: *std.StringHashMap(bool),
    ) !bool {
        try visited.put(name, true);
        try in_stack.put(name, true);
        
        if (self.nodes.get(name)) |node| {
            for (node.deps.items) |dep| {
                if (!visited.contains(dep)) {
                    if (try self.dfs(dep, visited, in_stack)) return true;
                } else if (in_stack.get(dep) orelse false) {
                    return true; // Circular!
                }
            }
        }
        
        try in_stack.put(name, false);
        return false;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var graph = DependencyGraph.init(allocator);
    defer graph.deinit();
    
    try graph.addPackage("my-app", "1.0.0", &.{ "httpz", "sqlite" });
    try graph.addPackage("httpz", "0.5.0", &.{"zig-network"});
    try graph.addPackage("sqlite", "0.3.0", &.{});
    try graph.addPackage("zig-network", "1.0.0", &.{});
    
    std.debug.print("Dependency tree:\n", .{});
    graph.printTree("my-app", 0);
    
    const has_circular = try graph.detectCircular();
    std.debug.print("\nCircular dependencies: {}\n", .{has_circular});
}
```

---

## สรุป Part 98

ใน Part นี้เราได้เรียนรู้เกี่ยวกับ Package Management ใน Zig อย่างครบถ้วน:

- **Step 971**: ภาพรวมของ Zig package system และโครงสร้าง project
- **Step 972**: โครงสร้างละเอียดของ `build.zig.zon` รวมถึง lazy dependencies
- **Step 973**: การใช้ `zig fetch` เพื่อดาวน์โหลดและ pin dependencies
- **Step 974**: การสร้างและใช้ local packages รวมถึง monorepo structure
- **Step 975**: การใช้ git packages และการเลือกระหว่าง tarball vs git
- **Step 976**: Semantic versioning และ version constraints
- **Step 977**: Package namespacing เพื่อหลีกเลี่ยง name conflicts
- **Step 978**: การ publish packages ให้คนอื่นใช้
- **Step 979**: การค้นหาและ evaluate community packages
- **Step 980**: Best practices สำหรับ dependency management รวมถึง vendoring

Zig's package system เน้น reproducibility และ simplicity โดยไม่ต้องการ central registry เหมือน npm หรือ crates.io
