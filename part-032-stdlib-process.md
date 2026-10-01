# ส่วนที่ 32: std.process - การจัดการ Process

## ภาพรวมของบทเรียน

`std.process` ใน Zig ให้ high-level interface สำหรับการจัดการ process ทั้งการรับ command line arguments, จัดการ environment, สร้าง child processes, และ handle signals ครอบคลุมทุกสิ่งที่ต้องการสำหรับการสร้าง CLI applications และ process management

---

## Step 311: std.process.args() - Command Line Arguments

### การอ่าน Command Line Arguments พื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // วิธีที่ 1: ใช้ std.process.args()
    var args = try std.process.argsWithAllocator(std.heap.page_allocator);
    defer args.deinit();
    
    try stdout.print("Command line arguments:\n", .{});
    
    var index: usize = 0;
    while (args.next()) |arg| {
        try stdout.print("  [{d}] = {s}\n", .{ index, arg });
        index += 1;
    }
    
    try stdout.print("จำนวน arguments: {d}\n", .{index});
}
```

### การใช้ std.process.argsAlloc

```zig
const std = @import("std");

pub fn parseArguments(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ดึง arguments ทั้งหมดเป็น slice
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);
    
    // args[0] คือ ชื่อโปรแกรม
    if (args.len > 0) {
        try stdout.print("โปรแกรม: {s}\n", .{args[0]});
    }
    
    // args[1..] คือ arguments ที่ผู้ใช้ส่งมา
    if (args.len <= 1) {
        try stdout.print("ไม่มี arguments\n", .{});
        return;
    }
    
    try stdout.print("Arguments ที่ส่งมา:\n", .{});
    for (args[1..], 0..) |arg, i| {
        try stdout.print("  {d}: {s}\n", .{ i + 1, arg });
    }
}
```

### Simple Argument Parser

```zig
const std = @import("std");

const ArgParseResult = struct {
    verbose: bool = false,
    output_file: ?[]const u8 = null,
    input_files: std.ArrayList([]const u8),
    help: bool = false,
    
    pub fn deinit(self: *ArgParseResult) void {
        self.input_files.deinit();
    }
};

pub fn parseArgs(allocator: std.mem.Allocator) !ArgParseResult {
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);
    
    var result = ArgParseResult{
        .input_files = std.ArrayList([]const u8).init(allocator),
    };
    
    var i: usize = 1; // ข้าม args[0] (ชื่อโปรแกรม)
    while (i < args.len) : (i += 1) {
        const arg = args[i];
        
        if (std.mem.eql(u8, arg, "--verbose") or std.mem.eql(u8, arg, "-v")) {
            result.verbose = true;
        } else if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
            result.help = true;
        } else if (std.mem.eql(u8, arg, "--output") or std.mem.eql(u8, arg, "-o")) {
            i += 1;
            if (i < args.len) {
                result.output_file = try allocator.dupe(u8, args[i]);
            }
        } else if (!std.mem.startsWith(u8, arg, "-")) {
            // ถ้าไม่ใช่ flag ถือว่าเป็น input file
            try result.input_files.append(try allocator.dupe(u8, arg));
        }
    }
    
    return result;
}

pub fn demonstrateArgParser(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    var parsed = try parseArgs(allocator);
    defer parsed.deinit();
    
    if (parsed.help) {
        try stdout.print("Usage: program [options] [files...]\n", .{});
        try stdout.print("Options:\n", .{});
        try stdout.print("  -v, --verbose    แสดงข้อมูลเพิ่มเติม\n", .{});
        try stdout.print("  -o, --output     กำหนดไฟล์ output\n", .{});
        try stdout.print("  -h, --help       แสดง help\n", .{});
        return;
    }
    
    try stdout.print("Verbose: {}\n", .{parsed.verbose});
    if (parsed.output_file) |out| {
        try stdout.print("Output: {s}\n", .{out});
    }
    try stdout.print("Input files: {d}\n", .{parsed.input_files.items.len});
}
```

---

## Step 312: std.process.getEnvMap() - Environment

### การดึง Environment Map

```zig
const std = @import("std");

pub fn getEnvironmentMap(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ดึง environment variables ทั้งหมด
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();
    
    try stdout.print("Environment Variables:\n", .{});
    try stdout.print("=" .* 40 ++ "\n", .{});
    
    // Iterate ผ่าน all entries
    var iter = env_map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("{s}={s}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
    
    // ค้นหา specific variable
    if (env_map.get("PATH")) |path| {
        try stdout.print("\nPATH directories:\n", .{});
        var it = std.mem.splitScalar(u8, path, ':');
        while (it.next()) |dir| {
            try stdout.print("  {s}\n", .{dir});
        }
    }
}
```

### การใช้ Environment Map ในการ Configure Application

```zig
const std = @import("std");

const DatabaseConfig = struct {
    host: []const u8,
    port: u16,
    name: []const u8,
    username: []const u8,
    password: []const u8,
    pool_size: u32,
    
    pub fn fromEnvMap(env: std.process.EnvMap) DatabaseConfig {
        return DatabaseConfig{
            .host = env.get("DB_HOST") orelse "localhost",
            .port = blk: {
                const p = env.get("DB_PORT") orelse "5432";
                break :blk std.fmt.parseInt(u16, p, 10) catch 5432;
            },
            .name = env.get("DB_NAME") orelse "mydb",
            .username = env.get("DB_USER") orelse "postgres",
            .password = env.get("DB_PASSWORD") orelse "",
            .pool_size = blk: {
                const p = env.get("DB_POOL_SIZE") orelse "10";
                break :blk std.fmt.parseInt(u32, p, 10) catch 10;
            },
        };
    }
    
    pub fn print(self: DatabaseConfig, writer: anytype) !void {
        try writer.print("Database Configuration:\n", .{});
        try writer.print("  Host: {s}\n", .{self.host});
        try writer.print("  Port: {d}\n", .{self.port});
        try writer.print("  Name: {s}\n", .{self.name});
        try writer.print("  User: {s}\n", .{self.username});
        try writer.print("  Password: {s}\n", .{if (self.password.len > 0) "***" else "(empty)"});
        try writer.print("  Pool Size: {d}\n", .{self.pool_size});
    }
};

pub fn demonstrateDatabaseConfig(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();
    
    const config = DatabaseConfig.fromEnvMap(env_map);
    try config.print(stdout);
}
```

---

## Step 313: std.process.exit() - Exit Codes

### การใช้ Exit Codes

```zig
const std = @import("std");

// Exit code conventions:
// 0 = Success
// 1 = General error
// 2 = Misuse of command
// 126 = Command found but not executable
// 127 = Command not found
// 128+N = Fatal signal N received

pub const ExitCode = enum(u8) {
    success = 0,
    general_error = 1,
    misuse = 2,
    config_error = 3,
    runtime_error = 4,
    network_error = 5,
    
    pub fn value(self: ExitCode) u8 {
        return @intFromEnum(self);
    }
};

pub fn exitWithCode(code: ExitCode) noreturn {
    const stdout = std.io.getStdOut().writer();
    
    switch (code) {
        .success => {
            stdout.print("โปรแกรมจบการทำงานสำเร็จ\n", .{}) catch {};
        },
        .general_error => {
            const stderr = std.io.getStdErr().writer();
            stderr.print("เกิดข้อผิดพลาด\n", .{}) catch {};
        },
        else => {
            const stderr = std.io.getStdErr().writer();
            stderr.print("ออกจากโปรแกรมด้วย code {d}\n", .{code.value()}) catch {};
        },
    }
    
    std.process.exit(code.value());
}

pub fn demonstrateExitCodes() !void {
    const stdout = std.io.getStdOut().writer();
    
    // แสดง exit code ต่างๆ
    try stdout.print("Exit Codes:\n", .{});
    try stdout.print("  SUCCESS = {d}\n", .{ExitCode.success.value()});
    try stdout.print("  GENERAL_ERROR = {d}\n", .{ExitCode.general_error.value()});
    try stdout.print("  MISUSE = {d}\n", .{ExitCode.misuse.value()});
    
    // ตัวอย่างการใช้ - ไม่เรียกจริงๆ ในตัวอย่างนี้
    // exitWithCode(.success);
}
```

### การจัดการ Errors แล้ว Exit

```zig
const std = @import("std");

pub fn runApplication(allocator: std.mem.Allocator) !void {
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);
    
    if (args.len < 2) {
        const stderr = std.io.getStdErr().writer();
        try stderr.print("Usage: {s} <file>\n", .{args[0]});
        std.process.exit(2); // Misuse
    }
    
    const file_path = args[1];
    
    // ลองเปิดไฟล์
    const file = std.fs.cwd().openFile(file_path, .{}) catch |err| {
        const stderr = std.io.getStdErr().writer();
        try stderr.print("ไม่สามารถเปิดไฟล์ {s}: {}\n", .{ file_path, err });
        std.process.exit(1);
    };
    defer file.close();
    
    const stdout = std.io.getStdOut().writer();
    try stdout.print("เปิดไฟล์ {s} สำเร็จ\n", .{file_path});
}
```

---

## Step 314: std.process.Child - Child Process

### การสร้าง Child Process

```zig
const std = @import("std");

pub fn createChildProcess(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง Child process สำหรับรัน "ls -la"
    var child = std.process.Child.init(&[_][]const u8{ "ls", "-la" }, allocator);
    
    // กำหนด stdin/stdout/stderr behavior
    child.stdin_behavior = .Ignore;
    child.stdout_behavior = .Inherit; // output ไปที่ terminal โดยตรง
    child.stderr_behavior = .Inherit;
    
    // Spawn process
    try child.spawn();
    try stdout.print("Child process spawned, PID: {d}\n", .{child.id});
    
    // รอให้ child process จบ
    const term = try child.wait();
    
    switch (term) {
        .Exited => |code| {
            try stdout.print("Process จบด้วย exit code: {d}\n", .{code});
        },
        .Signal => |sig| {
            try stdout.print("Process ถูก kill ด้วย signal: {d}\n", .{sig});
        },
        .Stopped => |sig| {
            try stdout.print("Process หยุดด้วย signal: {d}\n", .{sig});
        },
        .Unknown => |code| {
            try stdout.print("Process จบด้วย code ที่ไม่รู้จัก: {d}\n", .{code});
        },
    }
}
```

### Child Process พร้อม Working Directory

```zig
const std = @import("std");

pub fn childProcessWithDir(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var child = std.process.Child.init(
        &[_][]const u8{ "pwd" },
        allocator,
    );
    
    // กำหนด working directory
    child.cwd = "/tmp";
    child.stdout_behavior = .Inherit;
    child.stderr_behavior = .Inherit;
    
    try child.spawn();
    try stdout.print("รัน pwd ใน /tmp\n", .{});
    
    const term = try child.wait();
    switch (term) {
        .Exited => |code| try stdout.print("Exit code: {d}\n", .{code}),
        else => {},
    }
}
```

### Child Process พร้อม Custom Environment

```zig
const std = @import("std");

pub fn childProcessWithEnv(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง environment สำหรับ child process
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();
    
    // เพิ่ม custom variables
    try env_map.put("MY_CUSTOM_VAR", "hello_from_parent");
    try env_map.put("APP_ENV", "testing");
    
    var child = std.process.Child.init(
        &[_][]const u8{ "env" }, // แสดง environment ทั้งหมด
        allocator,
    );
    
    child.env_map = &env_map;
    child.stdout_behavior = .Inherit;
    
    try child.spawn();
    try stdout.print("Child process with custom environment\n", .{});
    
    _ = try child.wait();
}
```

---

## Step 315: Child.run() - Capture Output

### การ Capture Output จาก Child Process

```zig
const std = @import("std");

pub fn captureProcessOutput(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ใช้ Child.run() เพื่อ capture stdout และ stderr
    const result = try std.process.Child.run(.{
        .allocator = allocator,
        .argv = &[_][]const u8{ "ls", "-la", "/tmp" },
    });
    defer allocator.free(result.stdout);
    defer allocator.free(result.stderr);
    
    try stdout.print("Exit term: {}\n", .{result.term});
    try stdout.print("\n--- stdout ---\n{s}\n", .{result.stdout});
    
    if (result.stderr.len > 0) {
        try stdout.print("--- stderr ---\n{s}\n", .{result.stderr});
    }
}
```

### Capture และ Parse Output

```zig
const std = @import("std");

pub fn runAndParseOutput(
    allocator: std.mem.Allocator,
    command: []const []const u8,
) !std.ArrayList([]u8) {
    var result = try std.process.Child.run(.{
        .allocator = allocator,
        .argv = command,
        .max_output_bytes = 1024 * 1024, // 1MB limit
    });
    defer allocator.free(result.stderr);
    
    // Check exit code
    switch (result.term) {
        .Exited => |code| {
            if (code != 0) {
                allocator.free(result.stdout);
                return error.CommandFailed;
            }
        },
        else => {
            allocator.free(result.stdout);
            return error.CommandFailed;
        },
    }
    
    // แยก output เป็นบรรทัดๆ
    var lines = std.ArrayList([]u8).init(allocator);
    errdefer {
        for (lines.items) |line| allocator.free(line);
        lines.deinit();
    }
    
    var it = std.mem.splitScalar(u8, result.stdout, '\n');
    while (it.next()) |line| {
        if (line.len > 0) {
            const line_copy = try allocator.dupe(u8, line);
            try lines.append(line_copy);
        }
    }
    
    allocator.free(result.stdout);
    return lines;
}

pub fn demonstrateCaptureAndParse(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // รัน `uname -a` และ parse output
    var lines = try runAndParseOutput(
        allocator,
        &[_][]const u8{ "uname", "-a" },
    );
    defer {
        for (lines.items) |line| allocator.free(line);
        lines.deinit();
    }
    
    try stdout.print("System info:\n", .{});
    for (lines.items) |line| {
        try stdout.print("  {s}\n", .{line});
    }
}
```

### Run with Timeout

```zig
const std = @import("std");

pub fn runWithTimeout(
    allocator: std.mem.Allocator,
    command: []const []const u8,
    timeout_ms: u64,
) !std.process.Child.RunResult {
    var child = std.process.Child.init(command, allocator);
    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Pipe;
    
    try child.spawn();
    
    // Simple timeout implementation
    var timer = try std.time.Timer.start();
    
    while (timer.read() < timeout_ms * std.time.ns_per_ms) {
        // ลองรอ process (non-blocking)
        // ใน production ควรใช้ poll/epoll
        std.time.sleep(10 * std.time.ns_per_ms);
        break; // Simplified version
    }
    
    // Collect output
    var stdout_data = std.ArrayList(u8).init(allocator);
    var stderr_data = std.ArrayList(u8).init(allocator);
    
    if (child.stdout) |stdout_pipe| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = stdout_pipe.read(&buf) catch break;
            if (n == 0) break;
            try stdout_data.appendSlice(buf[0..n]);
        }
    }
    
    if (child.stderr) |stderr_pipe| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = stderr_pipe.read(&buf) catch break;
            if (n == 0) break;
            try stderr_data.appendSlice(buf[0..n]);
        }
    }
    
    const term = try child.wait();
    
    return std.process.Child.RunResult{
        .term = term,
        .stdout = try stdout_data.toOwnedSlice(),
        .stderr = try stderr_data.toOwnedSlice(),
    };
}
```

---

## Step 316: Child.spawn() - Async Child Process

### Async Process Management

```zig
const std = @import("std");

pub fn asyncChildProcess(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // Spawn หลาย processes พร้อมกัน
    var children: [3]std.process.Child = undefined;
    
    // สร้าง 3 child processes
    for (&children, 0..) |*child, i| {
        const cmd = try std.fmt.allocPrint(allocator, "sleep {d}", .{i + 1});
        defer allocator.free(cmd);
        
        child.* = std.process.Child.init(
            &[_][]const u8{ "sleep", cmd[6..] },
            allocator,
        );
        child.stdin_behavior = .Ignore;
        child.stdout_behavior = .Ignore;
        child.stderr_behavior = .Ignore;
        
        try child.spawn();
        try stdout.print("Spawned process {d}, PID: {d}\n", .{ i, child.id });
    }
    
    // รอทุก processes จบ
    for (&children, 0..) |*child, i| {
        const term = try child.wait();
        try stdout.print("Process {d} จบ: {}\n", .{ i, term });
    }
    
    try stdout.print("ทุก processes จบแล้ว\n", .{});
}
```

### Process Pool

```zig
const std = @import("std");

const ProcessPool = struct {
    allocator: std.mem.Allocator,
    processes: std.ArrayList(std.process.Child),
    max_processes: usize,
    
    pub fn init(allocator: std.mem.Allocator, max: usize) ProcessPool {
        return ProcessPool{
            .allocator = allocator,
            .processes = std.ArrayList(std.process.Child).init(allocator),
            .max_processes = max,
        };
    }
    
    pub fn deinit(self: *ProcessPool) void {
        // Wait สำหรับ remaining processes
        for (self.processes.items) |*p| {
            _ = p.wait() catch {};
        }
        self.processes.deinit();
    }
    
    pub fn spawn(self: *ProcessPool, args: []const []const u8) !void {
        // รอถ้า pool เต็ม
        while (self.processes.items.len >= self.max_processes) {
            // รอ process แรกใน queue
            if (self.processes.items.len > 0) {
                var first = self.processes.orderedRemove(0);
                _ = try first.wait();
            }
        }
        
        // Spawn process ใหม่
        var child = std.process.Child.init(args, self.allocator);
        child.stdin_behavior = .Ignore;
        child.stdout_behavior = .Inherit;
        child.stderr_behavior = .Inherit;
        
        try child.spawn();
        try self.processes.append(child);
    }
    
    pub fn waitAll(self: *ProcessPool) !void {
        for (self.processes.items) |*p| {
            _ = try p.wait();
        }
        self.processes.clearRetainingCapacity();
    }
};

pub fn demonstrateProcessPool(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var pool = ProcessPool.init(allocator, 3);
    defer pool.deinit();
    
    // รัน หลาย commands ผ่าน pool
    const commands = [_][]const u8{
        "echo task 1",
        "echo task 2",
        "echo task 3",
        "echo task 4",
        "echo task 5",
    };
    
    for (commands) |cmd| {
        try stdout.print("Spawning: {s}\n", .{cmd});
        try pool.spawn(&[_][]const u8{ "sh", "-c", cmd });
    }
    
    try pool.waitAll();
    try stdout.print("ทุก tasks จบแล้ว\n", .{});
}
```

---

## Step 317: Process Pipes (stdin/stdout/stderr)

### การส่งข้อมูลผ่าน Stdin

```zig
const std = @import("std");

pub fn writeToChildStdin(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง child process ที่รับ input จาก stdin
    var child = std.process.Child.init(
        &[_][]const u8{ "cat" }, // cat อ่าน stdin แล้ว print ออกมา
        allocator,
    );
    
    child.stdin_behavior = .Pipe;   // เราจะเขียนเข้า stdin
    child.stdout_behavior = .Pipe;  // เราจะอ่าน stdout
    child.stderr_behavior = .Ignore;
    
    try child.spawn();
    
    // เขียนเข้า stdin ของ child
    if (child.stdin) |stdin_pipe| {
        const data = "สวัสดี จาก parent process!\nบรรทัดที่สอง\nบรรทัดที่สาม\n";
        try stdin_pipe.writeAll(data);
        stdin_pipe.close();
        child.stdin = null; // mark as closed
    }
    
    // อ่าน stdout ของ child
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    if (child.stdout) |stdout_pipe| {
        var buf: [1024]u8 = undefined;
        while (true) {
            const n = try stdout_pipe.read(&buf);
            if (n == 0) break;
            try output.appendSlice(buf[0..n]);
        }
    }
    
    _ = try child.wait();
    
    try stdout.print("Child output:\n{s}\n", .{output.items});
}
```

### Pipe Chain (Process Piping)

```zig
const std = @import("std");
const posix = std.posix;

// สร้าง pipeline: cmd1 | cmd2 | cmd3
pub fn createPipeline(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // เทียบเท่ากับ: echo "hello world" | tr ' ' '\n' | sort
    
    // Process 1: echo
    var p1 = std.process.Child.init(
        &[_][]const u8{ "echo", "banana apple cherry date elderberry" },
        allocator,
    );
    p1.stdin_behavior = .Ignore;
    p1.stdout_behavior = .Pipe;
    p1.stderr_behavior = .Ignore;
    try p1.spawn();
    
    // Process 2: tr (แยก words เป็นบรรทัดๆ)
    var p2 = std.process.Child.init(
        &[_][]const u8{ "tr", " ", "\n" },
        allocator,
    );
    p2.stdin_behavior = .Pipe;
    p2.stdout_behavior = .Pipe;
    p2.stderr_behavior = .Ignore;
    try p2.spawn();
    
    // Connect p1.stdout -> p2.stdin
    if (p1.stdout) |p1_out| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = try p1_out.read(&buf);
            if (n == 0) break;
            if (p2.stdin) |p2_in| {
                try p2_in.writeAll(buf[0..n]);
            }
        }
    }
    
    if (p2.stdin) |p2_in| {
        p2_in.close();
        p2.stdin = null;
    }
    
    // Process 3: sort
    var p3 = std.process.Child.init(
        &[_][]const u8{"sort"},
        allocator,
    );
    p3.stdin_behavior = .Pipe;
    p3.stdout_behavior = .Pipe;
    p3.stderr_behavior = .Ignore;
    try p3.spawn();
    
    // Connect p2.stdout -> p3.stdin
    if (p2.stdout) |p2_out| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = try p2_out.read(&buf);
            if (n == 0) break;
            if (p3.stdin) |p3_in| {
                try p3_in.writeAll(buf[0..n]);
            }
        }
    }
    
    if (p3.stdin) |p3_in| {
        p3_in.close();
        p3.stdin = null;
    }
    
    // อ่าน final output
    var final_output = std.ArrayList(u8).init(allocator);
    defer final_output.deinit();
    
    if (p3.stdout) |p3_out| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = try p3_out.read(&buf);
            if (n == 0) break;
            try final_output.appendSlice(buf[0..n]);
        }
    }
    
    _ = try p1.wait();
    _ = try p2.wait();
    _ = try p3.wait();
    
    try stdout.print("Pipeline result:\n{s}\n", .{final_output.items});
}
```

---

## Step 318: Signal Handling

### Signal Handler สำหรับ Graceful Shutdown

```zig
const std = @import("std");
const posix = std.posix;

const AppState = struct {
    running: std.atomic.Value(bool),
    cleanup_done: std.atomic.Value(bool),
    
    pub fn init() AppState {
        return AppState{
            .running = std.atomic.Value(bool).init(true),
            .cleanup_done = std.atomic.Value(bool).init(false),
        };
    }
};

var app_state = AppState.init();

fn handleShutdown(sig: i32) callconv(.C) void {
    _ = sig;
    app_state.running.store(false, .seq_cst);
    
    // เขียน message โดยตรง (async-signal-safe)
    const msg = "\nกำลัง shutdown อย่างปลอดภัย...\n";
    _ = posix.write(posix.STDOUT_FILENO, msg) catch {};
}

fn handleReload(sig: i32) callconv(.C) void {
    _ = sig;
    const msg = "\nกำลัง reload configuration...\n";
    _ = posix.write(posix.STDOUT_FILENO, msg) catch {};
}

pub fn setupSignalHandlers() !void {
    // SIGTERM - graceful shutdown
    const sa_term = posix.Sigaction{
        .handler = .{ .handler = handleShutdown },
        .mask = posix.empty_sigset,
        .flags = 0,
    };
    try posix.sigaction(posix.SIG.TERM, &sa_term, null);
    
    // SIGINT - Ctrl+C
    try posix.sigaction(posix.SIG.INT, &sa_term, null);
    
    // SIGHUP - reload config
    const sa_hup = posix.Sigaction{
        .handler = .{ .handler = handleReload },
        .mask = posix.empty_sigset,
        .flags = 0,
    };
    try posix.sigaction(posix.SIG.HUP, &sa_hup, null);
    
    // SIGPIPE - ignore (สำหรับ network servers)
    const sa_ignore = posix.Sigaction{
        .handler = .{ .handler = posix.SIG.IGN },
        .mask = posix.empty_sigset,
        .flags = 0,
    };
    try posix.sigaction(posix.SIG.PIPE, &sa_ignore, null);
}

pub fn runServer() !void {
    const stdout = std.io.getStdOut().writer();
    
    try setupSignalHandlers();
    try stdout.print("Server เริ่มทำงาน... (กด Ctrl+C เพื่อหยุด)\n", .{});
    
    var iteration: u32 = 0;
    while (app_state.running.load(.seq_cst)) {
        iteration += 1;
        try stdout.print("Processing... iteration {d}\n", .{iteration});
        std.time.sleep(500 * std.time.ns_per_ms);
        
        // หยุดหลัง 10 iterations ในตัวอย่าง
        if (iteration >= 10) break;
    }
    
    // Cleanup
    try stdout.print("กำลัง cleanup...\n", .{});
    std.time.sleep(100 * std.time.ns_per_ms);
    app_state.cleanup_done.store(true, .seq_cst);
    try stdout.print("Cleanup เสร็จสิ้น\n", .{});
}
```

---

## Step 319: Process Environment Management

### Environment Isolation สำหรับ Child Processes

```zig
const std = @import("std");

pub fn createIsolatedEnvironment(allocator: std.mem.Allocator) !std.process.EnvMap {
    var env = std.process.EnvMap.init(allocator);
    
    // สร้าง minimal environment
    try env.put("PATH", "/usr/bin:/bin");
    try env.put("HOME", "/tmp");
    try env.put("USER", "sandbox");
    try env.put("SHELL", "/bin/sh");
    try env.put("TERM", "xterm");
    
    return env;
}

pub fn runInIsolatedEnv(
    allocator: std.mem.Allocator,
    command: []const []const u8,
) !void {
    const stdout = std.io.getStdOut().writer();
    
    var env = try createIsolatedEnvironment(allocator);
    defer env.deinit();
    
    var child = std.process.Child.init(command, allocator);
    child.env_map = &env;
    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Pipe;
    
    try child.spawn();
    
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    if (child.stdout) |out| {
        var buf: [4096]u8 = undefined;
        while (true) {
            const n = out.read(&buf) catch break;
            if (n == 0) break;
            try output.appendSlice(buf[0..n]);
        }
    }
    
    _ = try child.wait();
    
    try stdout.print("Output:\n{s}\n", .{output.items});
}
```

### Environment Variable Inheritance

```zig
const std = @import("std");

pub fn inheritAndExtendEnv(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // เริ่มจาก parent environment
    var env = try std.process.getEnvMap(allocator);
    defer env.deinit();
    
    // เพิ่ม/แก้ไข variables
    try env.put("MY_APP_VERSION", "1.0.0");
    try env.put("MY_APP_ENV", "production");
    try env.put("MY_APP_LOG_LEVEL", "warn");
    
    // ลบ variables ที่ไม่ต้องการ
    env.remove("DISPLAY"); // ไม่ต้องการ display ใน child
    env.remove("DBUS_SESSION_BUS_ADDRESS");
    
    var child = std.process.Child.init(
        &[_][]const u8{ "env" },
        allocator,
    );
    child.env_map = &env;
    child.stdout_behavior = .Pipe;
    
    try child.spawn();
    
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    if (child.stdout) |out| {
        var buf: [8192]u8 = undefined;
        while (true) {
            const n = out.read(&buf) catch break;
            if (n == 0) break;
            try output.appendSlice(buf[0..n]);
        }
    }
    
    _ = try child.wait();
    
    // นับ variables
    var count: usize = 0;
    var it = std.mem.splitScalar(u8, output.items, '\n');
    while (it.next()) |line| {
        if (line.len > 0) count += 1;
    }
    
    try stdout.print("Environment มี {d} variables\n", .{count});
}
```

---

## Step 320: CLI Application Patterns

### สร้าง Professional CLI Application

```zig
const std = @import("std");

const CliApp = struct {
    name: []const u8,
    version: []const u8,
    description: []const u8,
    allocator: std.mem.Allocator,
    
    const Command = struct {
        name: []const u8,
        description: []const u8,
        handler: *const fn (args: [][]const u8, app: *const CliApp) anyerror!void,
    };
    
    commands: std.ArrayList(Command),
    
    pub fn init(
        allocator: std.mem.Allocator,
        name: []const u8,
        version: []const u8,
        description: []const u8,
    ) CliApp {
        return CliApp{
            .allocator = allocator,
            .name = name,
            .version = version,
            .description = description,
            .commands = std.ArrayList(Command).init(allocator),
        };
    }
    
    pub fn deinit(self: *CliApp) void {
        self.commands.deinit();
    }
    
    pub fn addCommand(self: *CliApp, cmd: Command) !void {
        try self.commands.append(cmd);
    }
    
    pub fn run(self: *const CliApp) !void {
        const args = try std.process.argsAlloc(self.allocator);
        defer std.process.argsFree(self.allocator, args);
        
        const stdout = std.io.getStdOut().writer();
        
        if (args.len < 2) {
            try self.printHelp(stdout);
            return;
        }
        
        const subcmd = args[1];
        
        // Handle built-in commands
        if (std.mem.eql(u8, subcmd, "--version") or 
            std.mem.eql(u8, subcmd, "-V")) {
            try stdout.print("{s} version {s}\n", .{ self.name, self.version });
            return;
        }
        
        if (std.mem.eql(u8, subcmd, "--help") or 
            std.mem.eql(u8, subcmd, "-h") or 
            std.mem.eql(u8, subcmd, "help")) {
            try self.printHelp(stdout);
            return;
        }
        
        // ค้นหา subcommand
        for (self.commands.items) |cmd| {
            if (std.mem.eql(u8, cmd.name, subcmd)) {
                try cmd.handler(args[2..], self);
                return;
            }
        }
        
        // ไม่พบ command
        const stderr = std.io.getStdErr().writer();
        try stderr.print("ไม่รู้จัก command: {s}\n", .{subcmd});
        try stderr.print("ลองใช้ '{s} help' สำหรับ help\n", .{self.name});
        std.process.exit(1);
    }
    
    fn printHelp(self: *const CliApp, writer: anytype) !void {
        try writer.print("{s} - {s}\n", .{ self.name, self.description });
        try writer.print("Version: {s}\n\n", .{self.version});
        try writer.print("Usage: {s} <command> [options]\n\n", .{self.name});
        try writer.print("Commands:\n", .{});
        
        for (self.commands.items) |cmd| {
            try writer.print("  {s:<20} {s}\n", .{ cmd.name, cmd.description });
        }
        
        try writer.print("\nOptions:\n", .{});
        try writer.print("  -h, --help      แสดง help\n", .{});
        try writer.print("  -V, --version   แสดง version\n", .{});
    }
};

// ตัวอย่าง command handlers
fn handleBuild(args: [][]const u8, app: *const CliApp) !void {
    _ = app;
    const stdout = std.io.getStdOut().writer();
    try stdout.print("กำลัง build...\n", .{});
    
    if (args.len > 0) {
        try stdout.print("Target: {s}\n", .{args[0]});
    }
    
    std.time.sleep(100 * std.time.ns_per_ms);
    try stdout.print("Build สำเร็จ!\n", .{});
}

fn handleTest(args: [][]const u8, app: *const CliApp) !void {
    _ = args;
    _ = app;
    const stdout = std.io.getStdOut().writer();
    try stdout.print("กำลังรัน tests...\n", .{});
    try stdout.print("All tests passed!\n", .{});
}

fn handleClean(args: [][]const u8, app: *const CliApp) !void {
    _ = args;
    _ = app;
    const stdout = std.io.getStdOut().writer();
    try stdout.print("กำลัง clean...\n", .{});
    try stdout.print("Clean เสร็จสิ้น\n", .{});
}

pub fn demonstrateCliApp(allocator: std.mem.Allocator) !void {
    var app = CliApp.init(
        allocator,
        "mybuild",
        "1.0.0",
        "Build system tool",
    );
    defer app.deinit();
    
    try app.addCommand(.{
        .name = "build",
        .description = "Build โปรเจกต์",
        .handler = handleBuild,
    });
    
    try app.addCommand(.{
        .name = "test",
        .description = "รัน tests",
        .handler = handleTest,
    });
    
    try app.addCommand(.{
        .name = "clean",
        .description = "ลบ build artifacts",
        .handler = handleClean,
    });
    
    try app.run();
}
```

### Progress Bar สำหรับ CLI

```zig
const std = @import("std");

pub const ProgressBar = struct {
    total: u64,
    current: u64,
    width: u32,
    label: []const u8,
    
    pub fn init(total: u64, width: u32, label: []const u8) ProgressBar {
        return ProgressBar{
            .total = total,
            .current = 0,
            .width = width,
            .label = label,
        };
    }
    
    pub fn update(self: *ProgressBar, current: u64) void {
        self.current = @min(current, self.total);
        self.render();
    }
    
    pub fn increment(self: *ProgressBar, amount: u64) void {
        self.update(self.current + amount);
    }
    
    fn render(self: *const ProgressBar) void {
        const stdout = std.io.getStdOut().writer();
        
        const percent = if (self.total > 0) 
            (self.current * 100) / self.total 
        else 
            0;
        
        const filled = (self.current * self.width) / @max(self.total, 1);
        
        stdout.print("\r{s}: [", .{self.label}) catch return;
        
        var i: u32 = 0;
        while (i < self.width) : (i += 1) {
            if (i < filled) {
                stdout.print("=", .{}) catch return;
            } else if (i == filled) {
                stdout.print(">", .{}) catch return;
            } else {
                stdout.print(" ", .{}) catch return;
            }
        }
        
        stdout.print("] {d}% ({d}/{d})", .{
            percent,
            self.current,
            self.total,
        }) catch return;
        
        if (self.current >= self.total) {
            stdout.print("\n", .{}) catch return;
        }
    }
};

pub fn demonstrateProgressBar() !void {
    var progress = ProgressBar.init(100, 40, "Downloading");
    
    var i: u64 = 0;
    while (i <= 100) : (i += 5) {
        progress.update(i);
        std.time.sleep(50 * std.time.ns_per_ms);
    }
}
```

### Terminal Colors

```zig
const std = @import("std");

pub const Color = enum {
    reset,
    black,
    red,
    green,
    yellow,
    blue,
    magenta,
    cyan,
    white,
    bold,
    dim,
    
    pub fn code(self: Color) []const u8 {
        return switch (self) {
            .reset => "\x1b[0m",
            .black => "\x1b[30m",
            .red => "\x1b[31m",
            .green => "\x1b[32m",
            .yellow => "\x1b[33m",
            .blue => "\x1b[34m",
            .magenta => "\x1b[35m",
            .cyan => "\x1b[36m",
            .white => "\x1b[37m",
            .bold => "\x1b[1m",
            .dim => "\x1b[2m",
        };
    }
};

pub fn colorPrint(
    writer: anytype, 
    color: Color, 
    comptime fmt: []const u8, 
    args: anytype,
) !void {
    // ตรวจสอบว่า terminal รองรับ colors หรือไม่
    const supports_color = std.posix.isatty(std.posix.STDOUT_FILENO);
    
    if (supports_color) {
        try writer.print("{s}", .{color.code()});
    }
    
    try writer.print(fmt, args);
    
    if (supports_color) {
        try writer.print("{s}", .{Color.reset.code()});
    }
}

pub fn demonstrateColors() !void {
    const stdout = std.io.getStdOut().writer();
    
    try colorPrint(stdout, .green, "สำเร็จ: {s}\n", .{"ทุกอย่างทำงานปกติ"});
    try colorPrint(stdout, .red, "ผิดพลาด: {s}\n", .{"มีปัญหาเกิดขึ้น"});
    try colorPrint(stdout, .yellow, "คำเตือน: {s}\n", .{"ควรตรวจสอบ"});
    try colorPrint(stdout, .blue, "ข้อมูล: {s}\n", .{"Version 1.0.0"});
    try colorPrint(stdout, .bold, "หัวข้อสำคัญ\n", .{});
}
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Build a 'which' Command Clone

```zig
const std = @import("std");

pub fn whichCommand(allocator: std.mem.Allocator, program: []const u8) !?[]u8 {
    // ดึง PATH
    const path = std.posix.getenv("PATH") orelse return null;
    
    // แยก PATH directories
    var it = std.mem.splitScalar(u8, path, ':');
    while (it.next()) |dir| {
        const full_path = try std.fmt.allocPrint(
            allocator,
            "{s}/{s}",
            .{ dir, program },
        );
        errdefer allocator.free(full_path);
        
        // ตรวจสอบว่าไฟล์มีอยู่และ executable
        std.fs.accessAbsolute(full_path, .{ .mode = .read_only }) catch {
            allocator.free(full_path);
            continue;
        };
        
        return full_path;
    }
    
    return null;
}
```

### แบบฝึกหัดที่ 2: Process Monitor

```zig
const std = @import("std");

pub fn monitorProcess(pid: std.posix.pid_t) !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Monitoring PID {d}...\n", .{pid});
    
    // อ่าน /proc/[pid]/status (Linux only)
    if (@import("builtin").os.tag == .linux) {
        const status_path = try std.fmt.allocPrint(
            std.heap.page_allocator,
            "/proc/{d}/status",
            .{pid},
        );
        defer std.heap.page_allocator.free(status_path);
        
        const file = std.fs.openFileAbsolute(status_path, .{}) catch |err| {
            try stdout.print("ไม่พบ process {d}: {}\n", .{ pid, err });
            return;
        };
        defer file.close();
        
        var buf: [4096]u8 = undefined;
        const n = try file.read(&buf);
        try stdout.print("{s}\n", .{buf[0..n]});
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **std.process.args()** - การอ่าน command line arguments
2. **std.process.getEnvMap()** - การจัดการ environment variables
3. **std.process.exit()** - การจบ process ด้วย exit codes
4. **std.process.Child** - การสร้างและจัดการ child processes
5. **Child.run()** - การ capture output จาก subprocess
6. **Child.spawn()** - การรัน async processes
7. **Process Pipes** - การส่งข้อมูลระหว่าง processes
8. **Signal Handling** - การ handle OS signals
9. **Environment Management** - การจัดการ environment ของ child processes
10. **CLI Patterns** - การสร้าง professional CLI applications

**Best Practices:**
- ใช้ `defer` เสมอเมื่อ spawn child process
- Handle exit codes อย่างเหมาะสม
- ระวัง deadlock เมื่อใช้ pipes (อ่านและเขียนพร้อมกัน)
- ใช้ atomic variables สำหรับ signal handlers
- ปิด pipe ends ที่ไม่ได้ใช้เสมอ

---

*ต่อไปในส่วนที่ 33: std.math - Mathematical Operations*
