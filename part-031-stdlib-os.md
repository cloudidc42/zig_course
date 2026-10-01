# ส่วนที่ 31: std.os - OS Interface

## ภาพรวมของบทเรียน

ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ `std.os` ซึ่งเป็น module ที่ให้การเข้าถึง operating system interface โดยตรง ครอบคลุมตั้งแต่การจัดการ environment variables, file descriptors, process signals, ไปจนถึงการเขียนโค้ดที่ทำงานได้ข้ามแพลตฟอร์ม

---

## Step 301: std.os overview - POSIX API

### แนวคิดพื้นฐาน

`std.os` ใน Zig เป็น low-level interface ที่ wraps system calls โดยตรง โดยออกแบบมาให้:
- ใกล้เคียงกับ POSIX API มากที่สุด
- Error handling ผ่าน Zig error union
- รองรับทั้ง Linux, macOS, Windows และแพลตฟอร์มอื่นๆ

```zig
const std = @import("std");
const os = std.os;
const posix = std.posix;

pub fn main() !void {
    // std.os มี namespace แยกตามระบบปฏิบัติการ
    // std.os.linux - Linux specific
    // std.os.windows - Windows specific
    // std.posix - POSIX compatible (ทำงานได้บน Linux/macOS)
    
    const stdout = std.io.getStdOut().writer();
    
    // ดู OS ที่กำลังรัน
    const target_os = @import("builtin").os.tag;
    try stdout.print("กำลังรันบน: {s}\n", .{@tagName(target_os)});
    
    // ดู CPU architecture
    const target_arch = @import("builtin").cpu.arch;
    try stdout.print("CPU Architecture: {s}\n", .{@tagName(target_arch)});
}
```

### โครงสร้างของ std.os

```zig
const std = @import("std");

pub fn exploreStdOs() void {
    // std.os.linux - ฟังก์ชันเฉพาะ Linux เช่น epoll, io_uring
    // std.os.darwin - ฟังก์ชันเฉพาะ macOS/iOS
    // std.os.windows - ฟังก์ชันเฉพาะ Windows
    // std.posix - API ที่ใช้ร่วมกันได้ข้ามแพลตฟอร์ม POSIX
    
    // Constants ที่สำคัญใน std.posix
    _ = std.posix.O; // Open flags
    _ = std.posix.S; // File permission bits
    _ = std.posix.STDIN_FILENO; // = 0
    _ = std.posix.STDOUT_FILENO; // = 1
    _ = std.posix.STDERR_FILENO; // = 2
}
```

### Error Types ใน std.os

```zig
const std = @import("std");
const posix = std.posix;

pub fn demonstrateErrors() !void {
    // Error types ที่พบบ่อยใน std.posix
    // posix.OpenError - error เมื่อเปิดไฟล์
    // posix.ReadError - error เมื่ออ่านไฟล์
    // posix.WriteError - error เมื่อเขียนไฟล์
    
    // ตัวอย่างการ handle error
    const result = posix.open("/nonexistent", .{}, 0) catch |err| {
        switch (err) {
            error.FileNotFound => {
                std.debug.print("ไม่พบไฟล์\n", .{});
                return;
            },
            error.AccessDenied => {
                std.debug.print("ไม่มีสิทธิ์เข้าถึง\n", .{});
                return;
            },
            else => return err,
        }
    };
    defer posix.close(result);
}
```

---

## Step 302: Environment Variables (getenv, setenv)

### การอ่าน Environment Variables

```zig
const std = @import("std");
const posix = std.posix;

pub fn readEnvironmentVariables() !void {
    const stdout = std.io.getStdOut().writer();
    
    // วิธีที่ 1: ใช้ std.posix.getenv (ง่ายที่สุด)
    // ส่งคืน ?[]const u8 (null ถ้าไม่มี)
    if (posix.getenv("HOME")) |home| {
        try stdout.print("HOME = {s}\n", .{home});
    } else {
        try stdout.print("ไม่พบ HOME environment variable\n", .{});
    }
    
    // อ่าน PATH
    if (posix.getenv("PATH")) |path| {
        try stdout.print("PATH = {s}\n", .{path});
    }
    
    // อ่าน USER
    if (posix.getenv("USER")) |user| {
        try stdout.print("USER = {s}\n", .{user});
    }
    
    // อ่าน LANG
    if (posix.getenv("LANG")) |lang| {
        try stdout.print("LANG = {s}\n", .{lang});
    }
    
    // ตรวจสอบว่ามี variable หรือไม่
    const debug_mode = posix.getenv("DEBUG") != null;
    if (debug_mode) {
        try stdout.print("โหมด DEBUG เปิดอยู่\n", .{});
    }
}
```

### การใช้ std.process.getEnvMap

```zig
const std = @import("std");

pub fn getAllEnvironmentVariables(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ดึง environment map ทั้งหมด
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();
    
    try stdout.print("Environment Variables ทั้งหมด:\n", .{});
    try stdout.print("================================\n", .{});
    
    var iter = env_map.iterator();
    var count: usize = 0;
    while (iter.next()) |entry| {
        try stdout.print("{s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        count += 1;
    }
    
    try stdout.print("================================\n", .{});
    try stdout.print("จำนวน variables: {d}\n", .{count});
    
    // ค้นหา variable เฉพาะ
    if (env_map.get("HOME")) |home| {
        try stdout.print("\nHOME directory: {s}\n", .{home});
    }
}
```

### การ Set Environment Variables

```zig
const std = @import("std");

pub fn setEnvironmentVariables() !void {
    const stdout = std.io.getStdOut().writer();
    
    // หมายเหตุ: setenv ใน POSIX จะ modify environment ของ process ปัจจุบัน
    // ใน Zig เราใช้ std.posix.setenv
    
    // ตรวจสอบ OS ก่อนใช้
    if (@import("builtin").os.tag != .windows) {
        // Linux/macOS
        try std.posix.setenv("MY_ZIG_VAR", "hello_from_zig", true);
        
        if (std.posix.getenv("MY_ZIG_VAR")) |val| {
            try stdout.print("MY_ZIG_VAR = {s}\n", .{val});
        }
        
        // ลบ environment variable
        try std.posix.unsetenv("MY_ZIG_VAR");
        
        if (std.posix.getenv("MY_ZIG_VAR")) |_| {
            try stdout.print("ยังมี MY_ZIG_VAR อยู่\n", .{});
        } else {
            try stdout.print("ลบ MY_ZIG_VAR แล้ว\n", .{});
        }
    }
}
```

### ตัวอย่างการใช้งานจริง: Application Configuration

```zig
const std = @import("std");
const posix = std.posix;

const AppConfig = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    log_level: []const u8,
    
    pub fn fromEnv() AppConfig {
        return AppConfig{
            .host = posix.getenv("APP_HOST") orelse "localhost",
            .port = blk: {
                const port_str = posix.getenv("APP_PORT") orelse "8080";
                break :blk std.fmt.parseInt(u16, port_str, 10) catch 8080;
            },
            .debug = posix.getenv("APP_DEBUG") != null,
            .log_level = posix.getenv("LOG_LEVEL") orelse "info",
        };
    }
};

pub fn demonstrateAppConfig() !void {
    const stdout = std.io.getStdOut().writer();
    const config = AppConfig.fromEnv();
    
    try stdout.print("Application Configuration:\n", .{});
    try stdout.print("  Host: {s}\n", .{config.host});
    try stdout.print("  Port: {d}\n", .{config.port});
    try stdout.print("  Debug: {}\n", .{config.debug});
    try stdout.print("  Log Level: {s}\n", .{config.log_level});
}
```

---

## Step 303: File Descriptors (open, close, read, write)

### File Descriptors พื้นฐาน

```zig
const std = @import("std");
const posix = std.posix;

pub fn fileDescriptorBasics() !void {
    const stdout = std.io.getStdOut().writer();
    
    // File descriptors พื้นฐาน
    // 0 = STDIN_FILENO  - standard input
    // 1 = STDOUT_FILENO - standard output
    // 2 = STDERR_FILENO - standard error
    
    try stdout.print("STDIN fd: {d}\n", .{posix.STDIN_FILENO});
    try stdout.print("STDOUT fd: {d}\n", .{posix.STDOUT_FILENO});
    try stdout.print("STDERR fd: {d}\n", .{posix.STDERR_FILENO});
    
    // เขียนตรงๆ ผ่าน file descriptor
    const message = "สวัสดีจาก Zig!\n";
    _ = try posix.write(posix.STDOUT_FILENO, message);
}
```

### การเปิดและอ่านไฟล์

```zig
const std = @import("std");
const posix = std.posix;

pub fn openAndReadFile(file_path: []const u8, allocator: std.mem.Allocator) ![]u8 {
    // เปิดไฟล์ด้วย posix.open
    const fd = try posix.open(file_path, .{ .ACCMODE = .RDONLY }, 0);
    defer posix.close(fd);
    
    // ดูขนาดไฟล์
    const stat = try posix.fstat(fd);
    const file_size = @as(usize, @intCast(stat.size));
    
    // จัดสรร memory
    const buffer = try allocator.alloc(u8, file_size);
    errdefer allocator.free(buffer);
    
    // อ่านไฟล์
    var total_read: usize = 0;
    while (total_read < file_size) {
        const bytes_read = try posix.read(fd, buffer[total_read..]);
        if (bytes_read == 0) break; // EOF
        total_read += bytes_read;
    }
    
    return buffer[0..total_read];
}

pub fn demonstrateReadFile() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    // สร้างไฟล์ทดสอบก่อน
    const test_file = "/tmp/zig_test.txt";
    {
        const fd = try posix.open(test_file, .{
            .ACCMODE = .WRONLY,
            .CREAT = true,
            .TRUNC = true,
        }, 0o644);
        defer posix.close(fd);
        
        const content = "สวัสดี Zig!\nนี่คือการทดสอบ file descriptor\n";
        _ = try posix.write(fd, content);
    }
    
    // อ่านไฟล์
    const content = try openAndReadFile(test_file, allocator);
    defer allocator.free(content);
    
    try stdout.print("เนื้อหาไฟล์:\n{s}\n", .{content});
    
    // ลบไฟล์ทดสอบ
    try posix.unlink(test_file);
}
```

### การเขียนไฟล์ด้วย File Descriptor

```zig
const std = @import("std");
const posix = std.posix;

pub fn writeFileWithFd(file_path: []const u8, content: []const u8) !void {
    // เปิดไฟล์สำหรับเขียน (สร้างถ้าไม่มี, truncate ถ้ามีอยู่แล้ว)
    const fd = try posix.open(file_path, .{
        .ACCMODE = .WRONLY,
        .CREAT = true,
        .TRUNC = true,
    }, 0o644); // permission: rw-r--r--
    defer posix.close(fd);
    
    // เขียนข้อมูล
    var written: usize = 0;
    while (written < content.len) {
        const bytes_written = try posix.write(fd, content[written..]);
        written += bytes_written;
    }
    
    std.debug.print("เขียน {d} bytes สำเร็จ\n", .{written});
}

pub fn appendToFile(file_path: []const u8, content: []const u8) !void {
    // เปิดไฟล์ในโหมด append
    const fd = try posix.open(file_path, .{
        .ACCMODE = .WRONLY,
        .CREAT = true,
        .APPEND = true,
    }, 0o644);
    defer posix.close(fd);
    
    _ = try posix.write(fd, content);
}
```

### การใช้ pread และ pwrite (Positional I/O)

```zig
const std = @import("std");
const posix = std.posix;

pub fn positionalIO() !void {
    const file_path = "/tmp/zig_positional.bin";
    
    // สร้างไฟล์พร้อมข้อมูล
    {
        const fd = try posix.open(file_path, .{
            .ACCMODE = .WRONLY,
            .CREAT = true,
            .TRUNC = true,
        }, 0o644);
        defer posix.close(fd);
        
        // เขียนข้อมูล 100 bytes
        var data: [100]u8 = undefined;
        for (&data, 0..) |*b, i| {
            b.* = @intCast(i);
        }
        _ = try posix.write(fd, &data);
    }
    
    // อ่านด้วย pread (อ่านจากตำแหน่งที่กำหนดโดยไม่เปลี่ยน file position)
    {
        const fd = try posix.open(file_path, .{ .ACCMODE = .RDONLY }, 0);
        defer posix.close(fd);
        
        var buf: [10]u8 = undefined;
        
        // อ่าน 10 bytes จากตำแหน่ง 50
        const bytes_read = try posix.pread(fd, &buf, 50);
        std.debug.print("อ่านจากตำแหน่ง 50: {any}\n", .{buf[0..bytes_read]});
        
        // อ่าน 10 bytes จากตำแหน่ง 0
        const bytes_read2 = try posix.pread(fd, &buf, 0);
        std.debug.print("อ่านจากตำแหน่ง 0: {any}\n", .{buf[0..bytes_read2]});
    }
    
    try posix.unlink(file_path);
}
```

---

## Step 304: Process Signals (SIGTERM, SIGINT)

### ความเข้าใจเรื่อง Signals

```zig
const std = @import("std");
const posix = std.posix;

// Signals ที่พบบ่อย
// SIGTERM (15) - ขอให้ process หยุดทำงาน (graceful shutdown)
// SIGINT  (2)  - Interrupt (Ctrl+C)
// SIGKILL (9)  - บังคับให้หยุดทำงานทันที (ห้าม catch)
// SIGHUP  (1)  - Hangup (terminal ถูกปิด)
// SIGUSR1 (10) - User defined signal 1
// SIGUSR2 (12) - User defined signal 2

pub fn signalBasics() void {
    std.debug.print("Signal numbers:\n", .{});
    std.debug.print("SIGTERM = {d}\n", .{posix.SIG.TERM});
    std.debug.print("SIGINT  = {d}\n", .{posix.SIG.INT});
    std.debug.print("SIGHUP  = {d}\n", .{posix.SIG.HUP});
}
```

### การ Handle Signals

```zig
const std = @import("std");
const posix = std.posix;

// Global variable สำหรับ signal handler
var should_exit = std.atomic.Value(bool).init(false);

fn signalHandler(sig: i32) callconv(.C) void {
    _ = sig;
    should_exit.store(true, .seq_cst);
    // หมายเหตุ: ใน signal handler ห้ามใช้ฟังก์ชันที่ไม่ async-signal-safe
    // เช่น malloc, printf (ใช้ write แทน)
    const msg = "ได้รับ signal, กำลัง shutdown...\n";
    _ = posix.write(posix.STDOUT_FILENO, msg) catch {};
}

pub fn setupSignalHandlers() !void {
    // ตั้ง signal handler สำหรับ SIGTERM
    const sa_term = posix.Sigaction{
        .handler = .{ .handler = signalHandler },
        .mask = posix.empty_sigset,
        .flags = 0,
    };
    try posix.sigaction(posix.SIG.TERM, &sa_term, null);
    
    // ตั้ง signal handler สำหรับ SIGINT
    const sa_int = posix.Sigaction{
        .handler = .{ .handler = signalHandler },
        .mask = posix.empty_sigset,
        .flags = 0,
    };
    try posix.sigaction(posix.SIG.INT, &sa_int, null);
    
    std.debug.print("Signal handlers ตั้งค่าแล้ว\n", .{});
    std.debug.print("กด Ctrl+C เพื่อหยุด\n", .{});
}

pub fn runWithSignalHandling() !void {
    try setupSignalHandlers();
    
    // Main loop
    var count: u32 = 0;
    while (!should_exit.load(.seq_cst)) {
        std.debug.print("ทำงาน... ({d})\n", .{count});
        count += 1;
        std.time.sleep(1 * std.time.ns_per_s);
        
        if (count >= 5) break; // หยุดหลัง 5 วินาทีในตัวอย่าง
    }
    
    std.debug.print("Program จบการทำงานแล้ว\n", .{});
}
```

### การส่ง Signal ไปยัง Process อื่น

```zig
const std = @import("std");
const posix = std.posix;

pub fn sendSignalToProcess(pid: posix.pid_t, sig: u8) !void {
    try posix.kill(pid, sig);
    std.debug.print("ส่ง signal {d} ไปยัง PID {d}\n", .{ sig, pid });
}

pub fn getMyPid() posix.pid_t {
    return posix.getpid();
}

pub fn demonstrateSignalSending() !void {
    const my_pid = getMyPid();
    std.debug.print("PID ของฉัน: {d}\n", .{my_pid});
    
    // ส่ง SIGUSR1 ไปยังตัวเอง (เพื่อทดสอบ)
    // try sendSignalToProcess(my_pid, posix.SIG.USR1);
}
```

---

## Step 305: std.os.linux specific functions

### Linux-specific APIs

```zig
const std = @import("std");
const linux = std.os.linux;

pub fn linuxSpecificFunctions() !void {
    // ตรวจสอบว่ารันบน Linux
    if (@import("builtin").os.tag != .linux) {
        std.debug.print("ฟังก์ชันนี้ใช้ได้เฉพาะบน Linux\n", .{});
        return;
    }
    
    const stdout = std.io.getStdOut().writer();
    
    // getpid - ดู process ID
    const pid = linux.getpid();
    try stdout.print("PID: {d}\n", .{pid});
    
    // getuid - ดู user ID
    const uid = linux.getuid();
    try stdout.print("UID: {d}\n", .{uid});
    
    // getgid - ดู group ID
    const gid = linux.getgid();
    try stdout.print("GID: {d}\n", .{gid});
    
    // gettid - ดู thread ID
    const tid = linux.gettid();
    try stdout.print("TID: {d}\n", .{tid});
}
```

### Linux epoll - I/O Event Notification

```zig
const std = @import("std");
const linux = std.os.linux;
const posix = std.posix;

pub fn demonstrateEpoll() !void {
    if (@import("builtin").os.tag != .linux) return;
    
    // สร้าง epoll instance
    const epoll_fd = try posix.epoll_create1(linux.EPOLL.CLOEXEC);
    defer posix.close(epoll_fd);
    
    // เพิ่ม stdin ให้ epoll monitor
    var event = linux.epoll_event{
        .events = linux.EPOLL.IN,
        .data = .{ .fd = posix.STDIN_FILENO },
    };
    
    try posix.epoll_ctl(epoll_fd, linux.EPOLL.CTL_ADD, posix.STDIN_FILENO, &event);
    
    std.debug.print("กำลัง monitor stdin ด้วย epoll...\n", .{});
    std.debug.print("พิมพ์บางอย่างแล้วกด Enter\n", .{});
    
    // รอ events (timeout 5000ms = 5 วินาที)
    var events: [10]linux.epoll_event = undefined;
    const n = try posix.epoll_wait(epoll_fd, &events, 5000);
    
    if (n > 0) {
        std.debug.print("มี {d} event(s) เกิดขึ้น\n", .{n});
        
        // อ่านข้อมูลจาก stdin
        var buf: [256]u8 = undefined;
        const bytes_read = try posix.read(posix.STDIN_FILENO, &buf);
        std.debug.print("อ่านได้: {s}", .{buf[0..bytes_read]});
    } else {
        std.debug.print("Timeout - ไม่มีข้อมูลใน 5 วินาที\n", .{});
    }
}
```

### Linux memfd - Memory-only Files

```zig
const std = @import("std");
const linux = std.os.linux;
const posix = std.posix;

pub fn demonstrateMemfd() !void {
    if (@import("builtin").os.tag != .linux) return;
    
    // สร้าง file ที่อยู่ใน memory เท่านั้น
    const fd = @as(posix.fd_t, @intCast(linux.memfd_create("zig_memfd", linux.MFD.CLOEXEC)));
    if (fd < 0) return error.MemfdCreateFailed;
    defer posix.close(fd);
    
    // เขียนข้อมูลลงใน memory file
    const data = "ข้อมูลใน memory file\n";
    _ = try posix.write(fd, data);
    
    // กลับไปที่ตำแหน่ง 0
    _ = try posix.lseek_SET(fd, 0);
    
    // อ่านข้อมูลกลับมา
    var buf: [100]u8 = undefined;
    const bytes_read = try posix.read(fd, &buf);
    
    std.debug.print("อ่านจาก memfd: {s}", .{buf[0..bytes_read]});
}
```

---

## Step 306: Clock และ Time (std.time)

### การใช้ std.time

```zig
const std = @import("std");

pub fn timeBasics() !void {
    const stdout = std.io.getStdOut().writer();
    
    // เวลาปัจจุบัน (nanoseconds since epoch)
    const now_ns = std.time.nanoTimestamp();
    try stdout.print("เวลาปัจจุบัน (ns): {d}\n", .{now_ns});
    
    // เวลาปัจจุบัน (milliseconds since epoch)
    const now_ms = std.time.milliTimestamp();
    try stdout.print("เวลาปัจจุบัน (ms): {d}\n", .{now_ms});
    
    // เวลาปัจจุบัน (seconds since epoch)
    const now_s = std.time.timestamp();
    try stdout.print("เวลาปัจจุบัน (s): {d}\n", .{now_s});
    
    // Constants สำหรับแปลงหน่วยเวลา
    try stdout.print("\nTime constants:\n", .{});
    try stdout.print("1 ns = {d} ns\n", .{std.time.ns_per_ns});
    try stdout.print("1 us = {d} ns\n", .{std.time.ns_per_us});
    try stdout.print("1 ms = {d} ns\n", .{std.time.ns_per_ms});
    try stdout.print("1 s  = {d} ns\n", .{std.time.ns_per_s});
    try stdout.print("1 m  = {d} ns\n", .{std.time.ns_per_min});
    try stdout.print("1 h  = {d} ns\n", .{std.time.ns_per_hour});
    try stdout.print("1 d  = {d} ns\n", .{std.time.ns_per_day});
    try stdout.print("1 wk = {d} ns\n", .{std.time.ns_per_week});
}
```

### การวัดเวลาการทำงาน (Benchmarking)

```zig
const std = @import("std");

pub fn measureExecutionTime() !void {
    const stdout = std.io.getStdOut().writer();
    
    // วัดเวลาด้วย Timer
    var timer = try std.time.Timer.start();
    
    // ทำงานบางอย่าง
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    
    const elapsed_ns = timer.read();
    const elapsed_ms = elapsed_ns / std.time.ns_per_ms;
    const elapsed_us = elapsed_ns / std.time.ns_per_us;
    
    try stdout.print("ผลลัพธ์: {d}\n", .{sum});
    try stdout.print("เวลาที่ใช้: {d} ns ({d} us, {d} ms)\n", .{
        elapsed_ns,
        elapsed_us,
        elapsed_ms,
    });
}
```

### High-Resolution Timer

```zig
const std = @import("std");

pub fn highResolutionTimer() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Monotonic clock - ไม่ได้รับผลจากการเปลี่ยน system clock
    var timer = try std.time.Timer.start();
    
    // Sleep 100 milliseconds
    std.time.sleep(100 * std.time.ns_per_ms);
    
    const elapsed = timer.read();
    try stdout.print("Sleep 100ms, จริงๆ ใช้เวลา: {d} us\n", .{
        elapsed / std.time.ns_per_us,
    });
    
    // Reset timer
    timer.reset();
    
    // ทำงานอีกครั้ง
    var sum: u64 = 0;
    for (0..100_000) |j| {
        sum += j;
    }
    
    try stdout.print("Loop 100k ครั้ง: {d} ns\n", .{timer.read()});
    _ = sum;
}
```

---

## Step 307: Sockets พื้นฐาน ด้วย std.posix

### TCP Socket Server

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;

pub fn createTCPServer(port: u16) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง socket
    const server_fd = try posix.socket(
        posix.AF.INET,      // IPv4
        posix.SOCK.STREAM,  // TCP
        posix.IPPROTO.TCP,
    );
    defer posix.close(server_fd);
    
    // ตั้ง SO_REUSEADDR เพื่อ reuse port ได้เร็วขึ้น
    try posix.setsockopt(
        server_fd,
        posix.SOL.SOCKET,
        posix.SO.REUSEADDR,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    // Bind ไปที่ address
    const addr = net.Address.initIp4(.{ 0, 0, 0, 0 }, port);
    try posix.bind(server_fd, &addr.any, addr.getOsSockLen());
    
    // Listen
    try posix.listen(server_fd, 128);
    
    try stdout.print("Server กำลัง listen บน port {d}\n", .{port});
    
    // Accept connection (รับแค่ 1 connection ในตัวอย่าง)
    var client_addr: posix.sockaddr = undefined;
    var client_addr_len: posix.socklen_t = @sizeOf(posix.sockaddr);
    
    const client_fd = try posix.accept(server_fd, &client_addr, &client_addr_len, 0);
    defer posix.close(client_fd);
    
    try stdout.print("มี client เชื่อมต่อมา!\n", .{});
    
    // ส่ง response
    const response = "HTTP/1.1 200 OK\r\nContent-Length: 12\r\n\r\nHello, Zig!";
    _ = try posix.write(client_fd, response);
}
```

### TCP Socket Client

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;

pub fn createTCPClient(host: []const u8, port: u16) !void {
    const stdout = std.io.getStdOut().writer();
    
    // Resolve hostname
    const address_list = try net.getAddressList(
        std.heap.page_allocator,
        host,
        port,
    );
    defer address_list.deinit();
    
    if (address_list.addrs.len == 0) {
        return error.UnknownHostName;
    }
    
    const addr = address_list.addrs[0];
    
    // สร้าง socket
    const sock_fd = try posix.socket(
        addr.any.family,
        posix.SOCK.STREAM,
        posix.IPPROTO.TCP,
    );
    defer posix.close(sock_fd);
    
    // Connect
    try posix.connect(sock_fd, &addr.any, addr.getOsSockLen());
    try stdout.print("เชื่อมต่อกับ {s}:{d} สำเร็จ\n", .{ host, port });
    
    // ส่ง HTTP GET request
    const request = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n";
    _ = try posix.write(sock_fd, request);
    
    // รับ response
    var buf: [4096]u8 = undefined;
    const bytes_read = try posix.read(sock_fd, &buf);
    try stdout.print("ได้รับ {d} bytes\n", .{bytes_read});
}
```

---

## Step 308: std.os.windows specific functions

### Windows API Basics

```zig
const std = @import("std");

pub fn windowsSpecific() !void {
    // ตรวจสอบว่ารันบน Windows
    const builtin = @import("builtin");
    if (builtin.os.tag != .windows) {
        std.debug.print("ฟังก์ชันนี้ใช้ได้เฉพาะบน Windows\n", .{});
        return;
    }
    
    const windows = std.os.windows;
    
    // ดู Windows version
    const version = windows.ntdll.RtlGetVersion;
    _ = version;
    
    // ดู System Info
    var sys_info: windows.SYSTEM_INFO = undefined;
    windows.kernel32.GetSystemInfo(&sys_info);
    
    std.debug.print("Number of processors: {d}\n", .{
        sys_info.dwNumberOfProcessors,
    });
    std.debug.print("Page size: {d}\n", .{sys_info.dwPageSize});
}
```

### Windows Registry (Conceptual)

```zig
const std = @import("std");

// หมายเหตุ: โค้ดนี้แสดงแนวคิดการใช้ Windows Registry
// ต้องรันบน Windows เท่านั้น
pub fn windowsRegistryExample() void {
    if (@import("builtin").os.tag != .windows) return;
    
    // Windows Registry key paths
    // HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion
    // HKEY_CURRENT_USER\Software\MyApp
    
    std.debug.print("Windows Registry operations:\n", .{});
    std.debug.print("- RegOpenKeyEx - เปิด registry key\n", .{});
    std.debug.print("- RegQueryValueEx - อ่าน registry value\n", .{});
    std.debug.print("- RegSetValueEx - เขียน registry value\n", .{});
    std.debug.print("- RegCloseKey - ปิด registry key\n", .{});
}
```

### Windows Path Handling

```zig
const std = @import("std");

pub fn windowsPathHandling(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    if (@import("builtin").os.tag == .windows) {
        // บน Windows ใช้ backslash เป็น path separator
        const win_path = "C:\\Users\\User\\Documents\\file.txt";
        try stdout.print("Windows path: {s}\n", .{win_path});
        
        // แปลง Windows path เป็น POSIX path
        const win_path_alloc = try allocator.dupe(u8, win_path);
        defer allocator.free(win_path_alloc);
        
        for (win_path_alloc) |*c| {
            if (c.* == '\\') c.* = '/';
        }
        try stdout.print("POSIX path: {s}\n", .{win_path_alloc});
    } else {
        try stdout.print("ฟังก์ชันนี้สำหรับ Windows เท่านั้น\n", .{});
    }
}
```

---

## Step 309: Platform Detection ด้วย builtin.os.tag

### Compile-time Platform Detection

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn detectPlatform() void {
    // ตรวจสอบ OS ณ compile time
    switch (builtin.os.tag) {
        .linux => std.debug.print("กำลังรันบน Linux\n", .{}),
        .macos => std.debug.print("กำลังรันบน macOS\n", .{}),
        .windows => std.debug.print("กำลังรันบน Windows\n", .{}),
        .freebsd => std.debug.print("กำลังรันบน FreeBSD\n", .{}),
        .openbsd => std.debug.print("กำลังรันบน OpenBSD\n", .{}),
        .netbsd => std.debug.print("กำลังรันบน NetBSD\n", .{}),
        .dragonfly => std.debug.print("กำลังรันบน DragonFly BSD\n", .{}),
        .wasi => std.debug.print("กำลังรันใน WASI\n", .{}),
        else => std.debug.print("OS อื่นๆ: {s}\n", .{@tagName(builtin.os.tag)}),
    }
    
    // ตรวจสอบ CPU architecture
    switch (builtin.cpu.arch) {
        .x86_64 => std.debug.print("CPU: x86_64 (64-bit Intel/AMD)\n", .{}),
        .aarch64 => std.debug.print("CPU: ARM64 (Apple Silicon, etc.)\n", .{}),
        .x86 => std.debug.print("CPU: x86 (32-bit Intel/AMD)\n", .{}),
        .arm => std.debug.print("CPU: ARM (32-bit)\n", .{}),
        .riscv64 => std.debug.print("CPU: RISC-V 64-bit\n", .{}),
        else => std.debug.print("CPU: {s}\n", .{@tagName(builtin.cpu.arch)}),
    }
    
    // ตรวจสอบ Endianness
    switch (builtin.cpu.arch.endian()) {
        .big => std.debug.print("Endian: Big Endian\n", .{}),
        .little => std.debug.print("Endian: Little Endian\n", .{}),
    }
}
```

### Platform-specific Behavior

```zig
const std = @import("std");
const builtin = @import("builtin");

// ฟังก์ชันที่ทำงานต่างกันตาม OS
pub fn getConfigDir(allocator: std.mem.Allocator) ![]u8 {
    return switch (builtin.os.tag) {
        .linux, .freebsd, .openbsd, .netbsd => blk: {
            const home = std.posix.getenv("HOME") orelse "/tmp";
            break :blk try std.fmt.allocPrint(allocator, "{s}/.config", .{home});
        },
        .macos => blk: {
            const home = std.posix.getenv("HOME") orelse "/tmp";
            break :blk try std.fmt.allocPrint(
                allocator, 
                "{s}/Library/Application Support", 
                .{home},
            );
        },
        .windows => blk: {
            const appdata = std.posix.getenv("APPDATA") orelse "C:\\Users\\Default\\AppData\\Roaming";
            break :blk try allocator.dupe(u8, appdata);
        },
        else => try allocator.dupe(u8, "/tmp"),
    };
}

pub fn demonstrateConfigDir(allocator: std.mem.Allocator) !void {
    const config_dir = try getConfigDir(allocator);
    defer allocator.free(config_dir);
    
    std.debug.print("Config directory: {s}\n", .{config_dir});
}
```

### Conditional Compilation

```zig
const std = @import("std");
const builtin = @import("builtin");

// ใช้ comptime if สำหรับ conditional compilation
const is_linux = builtin.os.tag == .linux;
const is_windows = builtin.os.tag == .windows;
const is_macos = builtin.os.tag == .macos;
const is_posix = is_linux or is_macos or 
                  builtin.os.tag == .freebsd or 
                  builtin.os.tag == .openbsd;

pub fn platformSpecificInit() !void {
    if (comptime is_linux) {
        std.debug.print("Linux initialization...\n", .{});
        // Linux-specific setup
    } else if (comptime is_windows) {
        std.debug.print("Windows initialization...\n", .{});
        // Windows-specific setup
    } else if (comptime is_macos) {
        std.debug.print("macOS initialization...\n", .{});
        // macOS-specific setup
    }
}
```

---

## Step 310: Cross-platform Code Patterns

### Platform Abstraction Layer

```zig
const std = @import("std");
const builtin = @import("builtin");

// Abstract interface สำหรับ platform-specific operations
pub const PlatformOps = struct {
    // Function pointers สำหรับ operations ที่ต่างกันตาม platform
    getHomeDir: *const fn (allocator: std.mem.Allocator) anyerror![]u8,
    getTempDir: *const fn (allocator: std.mem.Allocator) anyerror![]u8,
    clearScreen: *const fn () void,
    
    pub fn init() PlatformOps {
        return switch (builtin.os.tag) {
            .linux, .macos, .freebsd, .openbsd => PlatformOps{
                .getHomeDir = posixGetHomeDir,
                .getTempDir = posixGetTempDir,
                .clearScreen = posixClearScreen,
            },
            .windows => PlatformOps{
                .getHomeDir = windowsGetHomeDir,
                .getTempDir = windowsGetTempDir,
                .clearScreen = windowsClearScreen,
            },
            else => PlatformOps{
                .getHomeDir = fallbackGetHomeDir,
                .getTempDir = fallbackGetTempDir,
                .clearScreen = fallbackClearScreen,
            },
        };
    }
};

fn posixGetHomeDir(allocator: std.mem.Allocator) ![]u8 {
    const home = std.posix.getenv("HOME") orelse return error.NoHomeDir;
    return allocator.dupe(u8, home);
}

fn posixGetTempDir(allocator: std.mem.Allocator) ![]u8 {
    const tmp = std.posix.getenv("TMPDIR") orelse "/tmp";
    return allocator.dupe(u8, tmp);
}

fn posixClearScreen() void {
    // ANSI escape code สำหรับ clear screen
    std.debug.print("\x1b[2J\x1b[H", .{});
}

fn windowsGetHomeDir(allocator: std.mem.Allocator) ![]u8 {
    const home = std.posix.getenv("USERPROFILE") orelse 
                 std.posix.getenv("HOMEDRIVE") orelse 
                 "C:\\Users\\Default";
    return allocator.dupe(u8, home);
}

fn windowsGetTempDir(allocator: std.mem.Allocator) ![]u8 {
    const tmp = std.posix.getenv("TEMP") orelse 
               std.posix.getenv("TMP") orelse 
               "C:\\Windows\\Temp";
    return allocator.dupe(u8, tmp);
}

fn windowsClearScreen() void {
    std.debug.print("\x1b[2J\x1b[H", .{});
}

fn fallbackGetHomeDir(allocator: std.mem.Allocator) ![]u8 {
    return allocator.dupe(u8, "/home");
}

fn fallbackGetTempDir(allocator: std.mem.Allocator) ![]u8 {
    return allocator.dupe(u8, "/tmp");
}

fn fallbackClearScreen() void {
    std.debug.print("\n\n\n", .{});
}
```

### Cross-platform Path Operations

```zig
const std = @import("std");
const builtin = @import("builtin");

pub const path_separator: u8 = if (builtin.os.tag == .windows) '\\' else '/';
pub const path_list_separator: u8 = if (builtin.os.tag == .windows) ';' else ':';

pub fn joinPath(allocator: std.mem.Allocator, parts: []const []const u8) ![]u8 {
    if (parts.len == 0) return allocator.dupe(u8, "");
    
    // คำนวณขนาด buffer ที่ต้องการ
    var total_len: usize = 0;
    for (parts) |part| {
        total_len += part.len + 1; // +1 สำหรับ separator
    }
    
    var buffer = try allocator.alloc(u8, total_len);
    var pos: usize = 0;
    
    for (parts, 0..) |part, i| {
        if (i > 0 and pos > 0 and buffer[pos - 1] != path_separator) {
            buffer[pos] = path_separator;
            pos += 1;
        }
        @memcpy(buffer[pos..pos + part.len], part);
        pos += part.len;
    }
    
    return buffer[0..pos];
}

pub fn demonstrateCrossplatformPath(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const parts = [_][]const u8{ "home", "user", "documents", "file.txt" };
    const full_path = try joinPath(allocator, &parts);
    defer allocator.free(full_path);
    
    try stdout.print("Path: {s}\n", .{full_path});
    try stdout.print("Path separator: '{c}'\n", .{path_separator});
}
```

### Error Handling ที่ Cross-platform

```zig
const std = @import("std");
const builtin = @import("builtin");

pub const OSError = error{
    FileNotFound,
    PermissionDenied,
    OutOfMemory,
    NetworkUnreachable,
    Timeout,
    Unknown,
};

// แปลง OS-specific errors เป็น common errors
pub fn normalizeError(err: anyerror) OSError {
    return switch (err) {
        error.FileNotFound, error.NoEntry => .FileNotFound,
        error.AccessDenied, error.PermissionDenied => .PermissionDenied,
        error.OutOfMemory => .OutOfMemory,
        error.NetworkUnreachable, error.ConnectionRefused => .NetworkUnreachable,
        error.WouldBlock, error.TimedOut => .Timeout,
        else => .Unknown,
    };
}

pub fn crossPlatformFileOp(path: []const u8) OSError!void {
    _ = std.fs.cwd().openFile(path, .{}) catch |err| {
        return normalizeError(err);
    };
}
```

### สรุป Best Practices สำหรับ Cross-platform Code

```zig
const std = @import("std");
const builtin = @import("builtin");

// 1. ใช้ comptime สำหรับ compile-time platform checks
pub fn compiletimeCheck() void {
    comptime {
        if (builtin.os.tag == .wasi) {
            @compileError("ไม่รองรับ WASI platform");
        }
    }
}

// 2. ใช้ std.fs สำหรับ file operations แทน posix โดยตรง
pub fn useStdFs(allocator: std.mem.Allocator) !void {
    // std.fs รองรับ cross-platform โดยอัตโนมัติ
    const file = try std.fs.cwd().createFile("test.txt", .{});
    defer file.close();
    
    try file.writeAll("Cross-platform file writing!\n");
    _ = allocator;
}

// 3. ใช้ std.net สำหรับ networking
pub fn useStdNet() !void {
    // std.net รองรับ cross-platform
    const address = try std.net.Address.parseIp4("127.0.0.1", 8080);
    _ = address;
    std.debug.print("Address parsed successfully\n", .{});
}

// 4. Test บน multiple platforms ผ่าน cross-compilation
// zig build-exe main.zig -target x86_64-linux
// zig build-exe main.zig -target x86_64-windows
// zig build-exe main.zig -target aarch64-macos
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Environment Variable Manager

สร้างโปรแกรมที่:
1. อ่าน environment variables ทั้งหมด
2. แสดงผลในรูปแบบที่อ่านง่าย
3. รองรับการ filter ด้วย prefix

```zig
const std = @import("std");

pub fn envManager(allocator: std.mem.Allocator, prefix: ?[]const u8) !void {
    var env_map = try std.process.getEnvMap(allocator);
    defer env_map.deinit();
    
    const stdout = std.io.getStdOut().writer();
    
    var iter = env_map.iterator();
    while (iter.next()) |entry| {
        const key = entry.key_ptr.*;
        const value = entry.value_ptr.*;
        
        if (prefix) |p| {
            if (!std.mem.startsWith(u8, key, p)) continue;
        }
        
        try stdout.print("{s}={s}\n", .{ key, value });
    }
}
```

### แบบฝึกหัดที่ 2: Simple File Server

```zig
const std = @import("std");
const posix = std.posix;

pub fn simpleFileServer(port: u16, dir: []const u8) !void {
    _ = dir; // ใช้สำหรับ serve files จาก directory
    
    const server_fd = try posix.socket(
        posix.AF.INET,
        posix.SOCK.STREAM,
        posix.IPPROTO.TCP,
    );
    defer posix.close(server_fd);
    
    try posix.setsockopt(
        server_fd,
        posix.SOL.SOCKET,
        posix.SO.REUSEADDR,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    const addr = std.net.Address.initIp4(.{ 0, 0, 0, 0 }, port);
    try posix.bind(server_fd, &addr.any, addr.getOsSockLen());
    try posix.listen(server_fd, 10);
    
    std.debug.print("File server running on port {d}\n", .{port});
    
    // Accept loop (simplified)
    var client_addr: posix.sockaddr = undefined;
    var client_addr_len: posix.socklen_t = @sizeOf(posix.sockaddr);
    
    const client_fd = try posix.accept(
        server_fd, 
        &client_addr, 
        &client_addr_len, 
        0,
    );
    defer posix.close(client_fd);
    
    // Read request
    var buf: [1024]u8 = undefined;
    const n = try posix.read(client_fd, &buf);
    std.debug.print("Request: {s}\n", .{buf[0..n]});
    
    // Send simple response
    const response = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, World!";
    _ = try posix.write(client_fd, response);
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **std.os overview** - โครงสร้างของ std.os และ POSIX API
2. **Environment Variables** - การอ่าน/เขียน env vars ด้วย getenv/setenv
3. **File Descriptors** - การจัดการ file ด้วย low-level API
4. **Process Signals** - การรับและส่ง signals
5. **Linux-specific** - epoll, memfd และ APIs เฉพาะ Linux
6. **Clock/Time** - การวัดเวลาด้วย std.time
7. **Sockets** - TCP networking พื้นฐาน
8. **Windows-specific** - APIs เฉพาะ Windows
9. **Platform Detection** - การตรวจสอบ OS ณ compile time
10. **Cross-platform Patterns** - การเขียนโค้ดที่ทำงานได้ทุกแพลตฟอร์ม

**สิ่งที่ควรจำ:**
- ใช้ `std.posix` สำหรับ POSIX-compatible code
- ใช้ `builtin.os.tag` สำหรับ compile-time platform detection
- ใช้ `comptime if` สำหรับ conditional compilation
- ใช้ `std.fs` และ `std.net` เมื่อต้องการ cross-platform support
- จัดการ errors อย่างเหมาะสมกับแต่ละ OS

---

*ต่อไปในส่วนที่ 32: std.process - การจัดการ Process*
