# ส่วนที่ 30: std.fs ใน Zig Standard Library

## บทนำ

`std.fs` เป็นโมดูลสำคัญใน Zig Standard Library ที่ให้เครื่องมือสำหรับการทำงานกับระบบไฟล์ (File System) อย่างครบถ้วน ตั้งแต่การอ่าน-เขียนไฟล์ การสร้างและลบไดเรกทอรี การดูข้อมูล metadata ของไฟล์ ไปจนถึงการทำงานกับ path ต่าง ๆ

โมดูลนี้ออกแบบมาให้ทำงานข้ามแพลตฟอร์ม (cross-platform) โดย Zig จะจัดการความแตกต่างระหว่าง Linux, macOS, และ Windows ให้โดยอัตโนมัติ ผู้เขียนโปรแกรมสามารถเขียนโค้ดเดียวที่ทำงานได้บนทุกแพลตฟอร์ม

### โครงสร้างของ std.fs

```
std.fs
├── cwd()              - current working directory
├── openFileAbsolute() - เปิดไฟล์ด้วย absolute path
├── createFileAbsolute() - สร้างไฟล์ด้วย absolute path
├── makeDirAbsolute()  - สร้างไดเรกทอรีด้วย absolute path
├── deleteFileAbsolute() - ลบไฟล์ด้วย absolute path
├── Dir               - struct สำหรับจัดการไดเรกทอรี
├── File              - struct สำหรับจัดการไฟล์
└── path              - namespace สำหรับ path utilities
```

---

## Step 291: std.fs.cwd() - Current Working Directory

### ความหมายและการใช้งาน

`std.fs.cwd()` คืนค่า `Dir` ที่แทน current working directory (ไดเรกทอรีปัจจุบัน) ของโปรแกรม ซึ่งเป็นจุดเริ่มต้นสำหรับการทำงานกับไฟล์และไดเรกทอรีส่วนใหญ่

```zig
const std = @import("std");

pub fn main() !void {
    // รับ current working directory
    const cwd = std.fs.cwd();

    // cwd เป็น std.fs.Dir ที่ใช้งานได้ทันที
    // ไม่ต้อง close เพราะ Zig จัดการให้
    _ = cwd;

    std.debug.print("ได้รับ cwd สำเร็จ\n", .{});
}
```

### การใช้ cwd() เปิดไฟล์แบบ relative path

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // เปิดไฟล์ด้วย relative path จาก cwd
    const file = try cwd.openFile("readme.txt", .{});
    defer file.close();

    std.debug.print("เปิดไฟล์สำเร็จ\n", .{});
}
```

### การรับ path string ของ cwd

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // รับ path เป็น string
    const cwd_path = try std.fs.cwd().realpathAlloc(allocator, ".");
    defer allocator.free(cwd_path);

    std.debug.print("Current directory: {s}\n", .{cwd_path});
}
```

### ตัวอย่างการตรวจสอบว่าไฟล์มีอยู่ใน cwd

```zig
const std = @import("std");

fn fileExistsInCwd(filename: []const u8) bool {
    const cwd = std.fs.cwd();
    // ลองเปิดไฟล์ ถ้าเปิดได้แสดงว่ามีไฟล์นั้นอยู่
    const file = cwd.openFile(filename, .{}) catch return false;
    file.close();
    return true;
}

pub fn main() void {
    const files_to_check = [_][]const u8{
        "build.zig",
        "main.zig",
        "README.md",
        "nonexistent.txt",
    };

    for (files_to_check) |filename| {
        if (fileExistsInCwd(filename)) {
            std.debug.print("✓ {s} มีอยู่\n", .{filename});
        } else {
            std.debug.print("✗ {s} ไม่มีอยู่\n", .{filename});
        }
    }
}
```

### การเปิด subdirectory จาก cwd

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // เปิด subdirectory
    var src_dir = try cwd.openDir("src", .{});
    defer src_dir.close();

    std.debug.print("เปิด src directory สำเร็จ\n", .{});

    // เปิดไฟล์ภายใน subdirectory
    const main_file = try src_dir.openFile("main.zig", .{});
    defer main_file.close();

    std.debug.print("เปิด src/main.zig สำเร็จ\n", .{});
}
```

---

## Step 292: การอ่านไฟล์ (readFile, openFile)

### openFile - เปิดไฟล์สำหรับอ่าน

`openFile` เป็น method หลักสำหรับเปิดไฟล์ที่มีอยู่แล้ว

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // เปิดไฟล์สำหรับอ่านอย่างเดียว
    const file = try cwd.openFile("example.txt", .{
        .mode = .read_only,
    });
    defer file.close();

    // อ่านข้อมูลลงใน buffer
    var buffer: [1024]u8 = undefined;
    const bytes_read = try file.read(&buffer);

    std.debug.print("อ่านได้ {d} bytes\n", .{bytes_read});
    std.debug.print("เนื้อหา: {s}\n", .{buffer[0..bytes_read]});
}
```

### การอ่านไฟล์ทั้งหมดด้วย readToEndAlloc

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const cwd = std.fs.cwd();
    const file = try cwd.openFile("example.txt", .{});
    defer file.close();

    // อ่านไฟล์ทั้งหมด - max_bytes กำหนด upper bound
    const content = try file.readToEndAlloc(allocator, 10 * 1024 * 1024); // 10MB max
    defer allocator.free(content);

    std.debug.print("ขนาดไฟล์: {d} bytes\n", .{content.len});
    std.debug.print("เนื้อหา:\n{s}\n", .{content});
}
```

### การใช้ Dir.readFileAlloc สำหรับความสะดวก

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // shortcut: อ่านไฟล์โดยตรงจาก Dir
    const content = try std.fs.cwd().readFileAlloc(
        allocator,
        "config.json",
        1024 * 1024, // 1MB max
    );
    defer allocator.free(content);

    std.debug.print("Config content:\n{s}\n", .{content});
}
```

### การอ่านไฟล์ทีละบรรทัด

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const file = try std.fs.cwd().openFile("data.csv", .{});
    defer file.close();

    // สร้าง buffered reader เพื่อประสิทธิภาพที่ดีขึ้น
    var buf_reader = std.io.bufferedReader(file.reader());
    const reader = buf_reader.reader();

    var line_number: u32 = 0;
    var line_buf = std.ArrayList(u8).init(allocator);
    defer line_buf.deinit();

    while (true) {
        // อ่านทีละบรรทัด
        reader.streamUntilDelimiter(line_buf.writer(), '\n', null) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };

        line_number += 1;
        std.debug.print("บรรทัด {d}: {s}\n", .{ line_number, line_buf.items });
        line_buf.clearRetainingCapacity();
    }

    std.debug.print("อ่านทั้งหมด {d} บรรทัด\n", .{line_number});
}
```

### การอ่านไฟล์ด้วย fixed buffer

```zig
const std = @import("std");

pub fn main() !void {
    const file = try std.fs.cwd().openFile("binary.dat", .{});
    defer file.close();

    var buffer: [4096]u8 = undefined;
    var total_bytes: u64 = 0;

    // อ่านทีละ chunk
    while (true) {
        const bytes_read = try file.read(&buffer);
        if (bytes_read == 0) break; // ถึง EOF แล้ว

        total_bytes += bytes_read;

        // ประมวลผลข้อมูลใน buffer[0..bytes_read]
        std.debug.print("อ่าน chunk: {d} bytes (รวม: {d})\n", .{ bytes_read, total_bytes });
    }

    std.debug.print("อ่านไฟล์เสร็จ ทั้งหมด {d} bytes\n", .{total_bytes});
}
```

### การอ่านพร้อม error handling ที่ครบถ้วน

```zig
const std = @import("std");

const ReadFileError = error{
    FileNotFound,
    FileTooLarge,
    PermissionDenied,
    UnexpectedError,
};

fn readConfigFile(allocator: std.mem.Allocator, path: []const u8) ReadFileError![]u8 {
    const max_size = 64 * 1024; // 64KB

    const content = std.fs.cwd().readFileAlloc(allocator, path, max_size) catch |err| {
        return switch (err) {
            error.FileNotFound => ReadFileError.FileNotFound,
            error.AccessDenied => ReadFileError.PermissionDenied,
            error.FileTooBig => ReadFileError.FileTooLarge,
            else => ReadFileError.UnexpectedError,
        };
    };

    return content;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const content = readConfigFile(allocator, "app.config") catch |err| {
        switch (err) {
            ReadFileError.FileNotFound => std.debug.print("ไม่พบไฟล์ config\n", .{}),
            ReadFileError.PermissionDenied => std.debug.print("ไม่มีสิทธิ์อ่านไฟล์\n", .{}),
            ReadFileError.FileTooLarge => std.debug.print("ไฟล์ใหญ่เกินไป\n", .{}),
            ReadFileError.UnexpectedError => std.debug.print("เกิดข้อผิดพลาดที่ไม่คาดคิด\n", .{}),
        }
        return;
    };
    defer allocator.free(content);

    std.debug.print("Config loaded: {d} bytes\n", .{content.len});
}
```

---

## Step 293: การเขียนไฟล์ (createFile, writeFile)

### createFile - สร้างและเขียนไฟล์ใหม่

```zig
const std = @import("std");

pub fn main() !void {
    // สร้างไฟล์ใหม่ (ถ้ามีอยู่แล้วจะถูก truncate)
    const file = try std.fs.cwd().createFile("output.txt", .{
        .truncate = true,  // ลบเนื้อหาเดิมถ้ามีอยู่
    });
    defer file.close();

    // เขียนข้อมูล
    try file.writeAll("สวัสดี ชาวโลก!\n");
    try file.writeAll("นี่คือการทดสอบการเขียนไฟล์\n");

    std.debug.print("เขียนไฟล์สำเร็จ\n", .{});
}
```

### การเขียนด้วย writer interface

```zig
const std = @import("std");

pub fn main() !void {
    const file = try std.fs.cwd().createFile("log.txt", .{});
    defer file.close();

    const writer = file.writer();

    // ใช้ print เหมือน stdout
    try writer.print("เวลา: {d}\n", .{std.time.timestamp()});
    try writer.print("ข้อความ: {s}\n", .{"Hello from Zig"});
    try writer.print("จำนวน: {d}\n", .{42});
}
```

### การเขียนไฟล์แบบ buffered (ประสิทธิภาพดีกว่า)

```zig
const std = @import("std");

pub fn main() !void {
    const file = try std.fs.cwd().createFile("large_output.txt", .{});
    defer file.close();

    // Buffered writer ลด system call จำนวนมาก
    var buf_writer = std.io.bufferedWriter(file.writer());
    const writer = buf_writer.writer();

    var i: u32 = 0;
    while (i < 10000) : (i += 1) {
        try writer.print("บรรทัดที่ {d}: ข้อมูลตัวอย่าง\n", .{i});
    }

    // สำคัญ: ต้อง flush ก่อน close
    try buf_writer.flush();

    std.debug.print("เขียน 10000 บรรทัดสำเร็จ\n", .{});
}
```

### การต่อท้ายไฟล์ที่มีอยู่ (append)

```zig
const std = @import("std");

pub fn appendToLog(message: []const u8) !void {
    // เปิดไฟล์แบบ read_write เพื่อต่อท้าย
    const file = try std.fs.cwd().openFile("app.log", .{
        .mode = .read_write,
    });
    defer file.close();

    // ย้าย cursor ไปที่ท้ายไฟล์
    try file.seekFromEnd(0);

    const writer = file.writer();
    try writer.print("[LOG] {s}\n", .{message});
}

pub fn main() !void {
    // สร้างไฟล์ log ถ้ายังไม่มี
    const file = try std.fs.cwd().createFile("app.log", .{
        .truncate = false, // ไม่ลบเนื้อหาเดิม
        .exclusive = false, // ไม่ error ถ้ามีอยู่แล้ว
    });
    file.close();

    // ต่อท้าย log entries
    try appendToLog("โปรแกรมเริ่มทำงาน");
    try appendToLog("กำลังประมวลผล...");
    try appendToLog("โปรแกรมสิ้นสุดการทำงาน");

    std.debug.print("เขียน log เสร็จแล้ว\n", .{});
}
```

### การใช้ writeFileAll สำหรับการเขียนง่าย ๆ

```zig
const std = @import("std");

pub fn main() !void {
    // เขียนไฟล์ทั้งหมดในคราวเดียว
    try std.fs.cwd().writeFile(.{
        .sub_path = "simple.txt",
        .data = "เนื้อหาไฟล์ทั้งหมด\nบรรทัดที่สอง\n",
    });

    std.debug.print("เขียนไฟล์สำเร็จ\n", .{});
}
```

### ตัวอย่างการเขียน JSON config file

```zig
const std = @import("std");

const AppConfig = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    max_connections: u32,
};

fn writeConfig(config: AppConfig) !void {
    const file = try std.fs.cwd().createFile("config.json", .{});
    defer file.close();

    var buf_writer = std.io.bufferedWriter(file.writer());
    const writer = buf_writer.writer();

    try writer.writeAll("{\n");
    try writer.print("  \"host\": \"{s}\",\n", .{config.host});
    try writer.print("  \"port\": {d},\n", .{config.port});
    try writer.print("  \"debug\": {s},\n", .{if (config.debug) "true" else "false"});
    try writer.print("  \"max_connections\": {d}\n", .{config.max_connections});
    try writer.writeAll("}\n");

    try buf_writer.flush();
}

pub fn main() !void {
    const config = AppConfig{
        .host = "localhost",
        .port = 8080,
        .debug = true,
        .max_connections = 100,
    };

    try writeConfig(config);
    std.debug.print("บันทึก config.json สำเร็จ\n", .{});
}
```

### การเขียนพร้อม atomic operation (ป้องกันไฟล์เสียหาย)

```zig
const std = @import("std");

/// เขียนไฟล์แบบ atomic - เขียน temp file แล้ว rename
fn writeFileAtomic(dir: std.fs.Dir, filename: []const u8, content: []const u8) !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้างชื่อ temp file
    const temp_name = try std.fmt.allocPrint(allocator, "{s}.tmp", .{filename});
    defer allocator.free(temp_name);

    // เขียนลง temp file ก่อน
    {
        const temp_file = try dir.createFile(temp_name, .{});
        defer temp_file.close();
        try temp_file.writeAll(content);
    }

    // rename temp file เป็นชื่อจริง (atomic operation)
    try dir.rename(temp_name, filename);
}

pub fn main() !void {
    const important_data = "ข้อมูลสำคัญที่ต้องเขียนอย่างปลอดภัย\n";

    try writeFileAtomic(std.fs.cwd(), "important.txt", important_data);
    std.debug.print("เขียนไฟล์แบบ atomic สำเร็จ\n", .{});
}
```

---

## Step 294: Directory Operations (makeDir, openDir, deleteDir)

### makeDir - สร้างไดเรกทอรีใหม่

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไดเรกทอรีเดียว
    try cwd.makeDir("new_folder");
    std.debug.print("สร้าง new_folder สำเร็จ\n", .{});

    // สร้างหลายระดับ (recursive)
    try cwd.makePath("parent/child/grandchild");
    std.debug.print("สร้าง nested directories สำเร็จ\n", .{});
}
```

### openDir - เปิดไดเรกทอรี

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // เปิดไดเรกทอรี
    var dir = try cwd.openDir("src", .{
        .iterate = true, // อนุญาตให้ list ไฟล์ใน directory
    });
    defer dir.close();

    std.debug.print("เปิด src directory สำเร็จ\n", .{});
}
```

### deleteDir และ deleteTree

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้าง directory structure สำหรับทดสอบ
    try cwd.makePath("test_dir/sub1");
    try cwd.makePath("test_dir/sub2");

    // สร้างไฟล์ใน directories
    const f1 = try cwd.createFile("test_dir/file1.txt", .{});
    f1.close();
    const f2 = try cwd.createFile("test_dir/sub1/file2.txt", .{});
    f2.close();

    // ลบ directory เปล่า
    try cwd.deleteDir("test_dir/sub2");
    std.debug.print("ลบ sub2 สำเร็จ\n", .{});

    // ลบ directory ที่มีเนื้อหา (recursive delete)
    try cwd.deleteTree("test_dir");
    std.debug.print("ลบ test_dir ทั้งหมดสำเร็จ\n", .{});
}
```

### การจัดการ directory แบบมี error handling

```zig
const std = @import("std");

fn ensureDirectoryExists(dir: std.fs.Dir, path: []const u8) !void {
    dir.access(path, .{}) catch |err| {
        if (err == error.FileNotFound) {
            // ไดเรกทอรียังไม่มี สร้างใหม่
            try dir.makePath(path);
            std.debug.print("สร้างไดเรกทอรี: {s}\n", .{path});
        } else {
            return err;
        }
    };
}

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้าง directory structure สำหรับ application
    const dirs = [_][]const u8{
        "data",
        "data/logs",
        "data/cache",
        "data/uploads",
        "config",
    };

    for (dirs) |dir_path| {
        try ensureDirectoryExists(cwd, dir_path);
    }

    std.debug.print("เตรียม directory structure เสร็จแล้ว\n", .{});
}
```

### การย้ายไฟล์และไดเรกทอรี

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ทดสอบ
    const f = try cwd.createFile("original.txt", .{});
    try f.writeAll("เนื้อหาไฟล์ต้นฉบับ");
    f.close();

    // ย้ายไฟล์ (rename)
    try cwd.rename("original.txt", "renamed.txt");
    std.debug.print("ย้ายไฟล์สำเร็จ\n", .{});

    // ย้ายข้าม directory
    try cwd.makeDir("archive");
    try cwd.rename("renamed.txt", "archive/archived.txt");
    std.debug.print("ย้ายไฟล์ไปยัง archive สำเร็จ\n", .{});
}
```

### การลบไฟล์

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ทดสอบ
    const f = try cwd.createFile("temp.txt", .{});
    f.close();

    // ลบไฟล์
    try cwd.deleteFile("temp.txt");
    std.debug.print("ลบไฟล์สำเร็จ\n", .{});

    // ลบไฟล์แบบ safe (ไม่ error ถ้าไม่มี)
    cwd.deleteFile("maybe_exists.txt") catch |err| {
        if (err != error.FileNotFound) return err;
        // ไม่มีไฟล์ก็ OK
    };
}
```

---

## Step 295: File Metadata (stat, StatFile)

### การดู stat ของไฟล์

```zig
const std = @import("std");

pub fn main() !void {
    const file = try std.fs.cwd().openFile("example.txt", .{});
    defer file.close();

    // รับ stat ของไฟล์
    const stat = try file.stat();

    std.debug.print("ข้อมูลไฟล์:\n", .{});
    std.debug.print("  ขนาด: {d} bytes\n", .{stat.size});
    std.debug.print("  kind: {s}\n", .{@tagName(stat.kind)});
    std.debug.print("  mtime: {d}\n", .{stat.mtime});
    std.debug.print("  atime: {d}\n", .{stat.atime});
    std.debug.print("  ctime: {d}\n", .{stat.ctime});
}
```

### การใช้ statFile จาก Dir

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // statFile ไม่ต้องเปิดไฟล์ก่อน
    const stat = try cwd.statFile("example.txt");

    std.debug.print("ขนาดไฟล์: {d} bytes\n", .{stat.size});

    // ตรวจสอบประเภทของ entry
    switch (stat.kind) {
        .file => std.debug.print("เป็นไฟล์ปกติ\n", .{}),
        .directory => std.debug.print("เป็นไดเรกทอรี\n", .{}),
        .sym_link => std.debug.print("เป็น symbolic link\n", .{}),
        else => std.debug.print("ประเภทอื่น: {s}\n", .{@tagName(stat.kind)}),
    }
}
```

### การแปลง timestamp เป็นวันที่

```zig
const std = @import("std");

fn formatTimestamp(ns: i128) void {
    // แปลง nanoseconds เป็น seconds
    const seconds = @divTrunc(ns, std.time.ns_per_s);
    const epoch = std.time.epoch.EpochSeconds{ .secs = @intCast(seconds) };

    const epoch_day = epoch.getEpochDay();
    const year_day = epoch_day.calculateYearDay();
    const month_day = year_day.calculateMonthDay();

    std.debug.print("{d}/{d}/{d}", .{
        month_day.day_index + 1,
        month_day.month.numeric(),
        year_day.year,
    });
}

pub fn main() !void {
    const stat = try std.fs.cwd().statFile("build.zig");

    std.debug.print("วันที่แก้ไขล่าสุด: ", .{});
    formatTimestamp(stat.mtime);
    std.debug.print("\n", .{});

    std.debug.print("ขนาด: {d} bytes\n", .{stat.size});
}
```

### การเปรียบเทียบ modification time

```zig
const std = @import("std");

fn isFileNewer(path1: []const u8, path2: []const u8) !bool {
    const cwd = std.fs.cwd();
    const stat1 = try cwd.statFile(path1);
    const stat2 = try cwd.statFile(path2);
    return stat1.mtime > stat2.mtime;
}

pub fn main() !void {
    const file1 = "source.zig";
    const file2 = "compiled.out";

    if (try isFileNewer(file1, file2)) {
        std.debug.print("{s} ใหม่กว่า {s} - ต้อง recompile\n", .{ file1, file2 });
    } else {
        std.debug.print("{s} ยังเป็นปัจจุบัน\n", .{file2});
    }
}
```

### ตัวอย่าง file info utility

```zig
const std = @import("std");

fn printFileInfo(dir: std.fs.Dir, path: []const u8) !void {
    const stat = dir.statFile(path) catch |err| {
        std.debug.print("ไม่สามารถอ่าน stat ของ {s}: {s}\n", .{ path, @errorName(err) });
        return;
    };

    std.debug.print("=== {s} ===\n", .{path});
    std.debug.print("  ประเภท: {s}\n", .{@tagName(stat.kind)});
    std.debug.print("  ขนาด: {d} bytes ({d:.2} KB)\n", .{
        stat.size,
        @as(f64, @floatFromInt(stat.size)) / 1024.0,
    });

    // แสดง permissions (Unix-style)
    if (@hasField(@TypeOf(stat), "mode")) {
        std.debug.print("  Permissions: {o:0>4}\n", .{stat.mode & 0o7777});
    }
}

pub fn main() !void {
    const cwd = std.fs.cwd();
    const entries = [_][]const u8{
        "build.zig",
        "src",
        "README.md",
    };

    for (entries) |entry| {
        try printFileInfo(cwd, entry);
    }
}
```

---

## Step 296: Directory Listing (openDir, iterate)

### การ list ไฟล์ใน directory

```zig
const std = @import("std");

pub fn main() !void {
    var dir = try std.fs.cwd().openDir(".", .{
        .iterate = true, // จำเป็นสำหรับการ iterate
    });
    defer dir.close();

    var iter = dir.iterate();
    std.debug.print("ไฟล์และโฟลเดอร์ใน current directory:\n", .{});

    while (try iter.next()) |entry| {
        const type_char: u8 = switch (entry.kind) {
            .file => 'F',
            .directory => 'D',
            .sym_link => 'L',
            else => '?',
        };
        std.debug.print("  [{c}] {s}\n", .{ type_char, entry.name });
    }
}
```

### การแยก files และ directories

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var dir = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer dir.close();

    var files = std.ArrayList([]const u8).init(allocator);
    defer {
        for (files.items) |f| allocator.free(f);
        files.deinit();
    }

    var directories = std.ArrayList([]const u8).init(allocator);
    defer {
        for (directories.items) |d| allocator.free(d);
        directories.deinit();
    }

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        const name_copy = try allocator.dupe(u8, entry.name);
        switch (entry.kind) {
            .file => try files.append(name_copy),
            .directory => try directories.append(name_copy),
            else => allocator.free(name_copy),
        }
    }

    std.debug.print("ไฟล์ ({d}):\n", .{files.items.len});
    for (files.items) |f| {
        std.debug.print("  {s}\n", .{f});
    }

    std.debug.print("\nไดเรกทอรี ({d}):\n", .{directories.items.len});
    for (directories.items) |d| {
        std.debug.print("  {s}/\n", .{d});
    }
}
```

### การ filter ไฟล์ตาม extension

```zig
const std = @import("std");

fn hasExtension(filename: []const u8, ext: []const u8) bool {
    if (filename.len < ext.len + 1) return false;
    return std.mem.endsWith(u8, filename, ext);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var dir = try std.fs.cwd().openDir("src", .{ .iterate = true });
    defer dir.close();

    std.debug.print("ไฟล์ .zig ใน src/:\n", .{});
    var count: u32 = 0;

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.kind != .file) continue;
        if (!hasExtension(entry.name, ".zig")) continue;

        // รับ stat ของไฟล์
        const stat = try dir.statFile(entry.name);
        std.debug.print("  {s} ({d} bytes)\n", .{ entry.name, stat.size });
        count += 1;

        _ = allocator;
    }

    std.debug.print("รวม {d} ไฟล์\n", .{count});
}
```

### การเรียง directory listing

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var dir = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer dir.close();

    // เก็บชื่อทั้งหมดก่อน
    var names = std.ArrayList([]const u8).init(allocator);
    defer {
        for (names.items) |n| allocator.free(n);
        names.deinit();
    }

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.name[0] == '.') continue; // ข้าม hidden files
        const name_copy = try allocator.dupe(u8, entry.name);
        try names.append(name_copy);
    }

    // เรียงตามชื่อ
    std.mem.sort([]const u8, names.items, {}, struct {
        fn lessThan(_: void, a: []const u8, b: []const u8) bool {
            return std.mem.lessThan(u8, a, b);
        }
    }.lessThan);

    std.debug.print("ไฟล์เรียงตามชื่อ:\n", .{});
    for (names.items) |name| {
        std.debug.print("  {s}\n", .{name});
    }
}
```

### การนับไฟล์ในแต่ละ subdirectory

```zig
const std = @import("std");

fn countFilesInDir(dir: std.fs.Dir) !u32 {
    var count: u32 = 0;
    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.kind == .file) count += 1;
    }
    return count;
}

pub fn main() !void {
    var root = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer root.close();

    var iter = root.iterate();
    while (try iter.next()) |entry| {
        if (entry.kind != .directory) continue;
        if (entry.name[0] == '.') continue;

        var sub_dir = root.openDir(entry.name, .{ .iterate = true }) catch continue;
        defer sub_dir.close();

        const count = try countFilesInDir(sub_dir);
        std.debug.print("{s}/: {d} files\n", .{ entry.name, count });
    }
}
```

---

## Step 297: Path Operations (std.fs.path)

### การ join paths

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // join หลาย path components
    const full_path = try std.fs.path.join(allocator, &[_][]const u8{
        "/home",
        "user",
        "documents",
        "file.txt",
    });
    defer allocator.free(full_path);

    std.debug.print("Full path: {s}\n", .{full_path});
    // Output: /home/user/documents/file.txt
}
```

### การรับ dirname และ basename

```zig
const std = @import("std");

pub fn main() void {
    const path = "/home/user/documents/report.pdf";

    // dirname - ส่วนของ path ที่ไม่รวมชื่อไฟล์
    const dir = std.fs.path.dirname(path) orelse ".";
    std.debug.print("Directory: {s}\n", .{dir});
    // Output: /home/user/documents

    // basename - ชื่อไฟล์อย่างเดียว
    const base = std.fs.path.basename(path);
    std.debug.print("Filename: {s}\n", .{base});
    // Output: report.pdf

    // extension - นามสกุลไฟล์
    const ext = std.fs.path.extension(path);
    std.debug.print("Extension: {s}\n", .{ext});
    // Output: .pdf

    // stem - ชื่อไฟล์ไม่รวมนามสกุล
    const stem = std.fs.path.stem(path);
    std.debug.print("Stem: {s}\n", .{stem});
    // Output: report
}
```

### การ resolve relative path

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // resolve path ให้เป็น absolute path
    // (ต้องใช้กับ path ที่มีอยู่จริง)
    const resolved = try std.fs.cwd().realpathAlloc(allocator, ".");
    defer allocator.free(resolved);

    std.debug.print("Resolved cwd: {s}\n", .{resolved});

    // สร้าง absolute path โดย join กับ cwd
    const filename = "data.txt";
    const abs_path = try std.fs.path.join(allocator, &[_][]const u8{
        resolved,
        filename,
    });
    defer allocator.free(abs_path);

    std.debug.print("Absolute path: {s}\n", .{abs_path});
}
```

### การตรวจสอบ path

```zig
const std = @import("std");

pub fn main() void {
    const paths = [_][]const u8{
        "/usr/local/bin/zig",
        "relative/path",
        ".",
        "..",
        "./src/main.zig",
    };

    for (paths) |path| {
        const is_abs = std.fs.path.isAbsolute(path);
        std.debug.print("{s}: {s}\n", .{
            path,
            if (is_abs) "absolute" else "relative",
        });
    }
}
```

### Cross-platform path handling

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // std.fs.path จัดการ separator ให้ตาม platform อัตโนมัติ
    const path = try std.fs.path.join(allocator, &[_][]const u8{
        "src",
        "modules",
        "auth.zig",
    });
    defer allocator.free(path);

    // บน Linux/macOS: src/modules/auth.zig
    // บน Windows: src\modules\auth.zig
    std.debug.print("Path: {s}\n", .{path});
    std.debug.print("Separator: {c}\n", .{std.fs.path.sep});
}
```

### ตัวอย่าง path utility functions

```zig
const std = @import("std");

/// เปลี่ยน extension ของ path
fn changeExtension(
    allocator: std.mem.Allocator,
    path: []const u8,
    new_ext: []const u8,
) ![]u8 {
    const stem = std.fs.path.stem(path);
    const dir = std.fs.path.dirname(path);

    if (dir) |d| {
        return std.fs.path.join(allocator, &[_][]const u8{
            d,
            try std.fmt.allocPrint(allocator, "{s}{s}", .{ stem, new_ext }),
        });
    } else {
        return std.fmt.allocPrint(allocator, "{s}{s}", .{ stem, new_ext });
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const source = "src/module/parser.zig";
    const output = try changeExtension(allocator, source, ".o");
    defer allocator.free(output);

    std.debug.print("Source: {s}\n", .{source});
    std.debug.print("Object: {s}\n", .{output});
}
```

---

## Step 298: std.fs.Dir Methods

### ภาพรวม methods ของ std.fs.Dir

`std.fs.Dir` มี methods มากมายสำหรับทำงานกับไดเรกทอรี:

```zig
const std = @import("std");

pub fn main() !void {
    var dir = try std.fs.cwd().openDir(".", .{
        .iterate = true,
        .no_follow = false, // follow symlinks
    });
    defer dir.close();

    // --- File operations ---

    // openFile - เปิดไฟล์
    const f1 = try dir.openFile("readme.txt", .{});
    defer f1.close();

    // createFile - สร้างไฟล์
    const f2 = try dir.createFile("new.txt", .{});
    defer f2.close();

    // deleteFile - ลบไฟล์
    // try dir.deleteFile("old.txt");

    // readFileAlloc - อ่านไฟล์ทั้งหมด
    // const content = try dir.readFileAlloc(allocator, "data.txt", 1024);

    // writeFile - เขียนไฟล์
    try dir.writeFile(.{
        .sub_path = "output.txt",
        .data = "Hello",
    });

    // --- Directory operations ---

    // makeDir - สร้าง directory
    dir.makeDir("new_subdir") catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };

    // makePath - สร้าง nested directories
    try dir.makePath("a/b/c");

    // deleteDir - ลบ directory เปล่า
    // try dir.deleteDir("empty_dir");

    // deleteTree - ลบ directory และเนื้อหา
    // try dir.deleteTree("dir_with_content");

    // openDir - เปิด subdirectory
    var sub = try dir.openDir("src", .{});
    defer sub.close();

    std.debug.print("Dir methods demo สำเร็จ\n", .{});
}
```

### การใช้ Dir.access ตรวจสอบการมีอยู่

```zig
const std = @import("std");

fn pathExists(dir: std.fs.Dir, path: []const u8) bool {
    dir.access(path, .{}) catch return false;
    return true;
}

pub fn main() void {
    const cwd = std.fs.cwd();

    const paths = [_][]const u8{
        "build.zig",
        "src",
        "nonexistent.txt",
        ".gitignore",
    };

    for (paths) |path| {
        std.debug.print("{s}: {s}\n", .{
            path,
            if (pathExists(cwd, path)) "มีอยู่" else "ไม่มี",
        });
    }
}
```

### การ copy file ระหว่าง directories

```zig
const std = @import("std");

fn copyFile(
    src_dir: std.fs.Dir,
    src_path: []const u8,
    dst_dir: std.fs.Dir,
    dst_path: []const u8,
) !void {
    try src_dir.copyFile(src_path, dst_dir, dst_path, .{});
}

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ต้นฉบับ
    try cwd.writeFile(.{
        .sub_path = "original.txt",
        .data = "เนื้อหาต้นฉบับ\n",
    });

    // สร้าง destination directory
    cwd.makeDir("backup") catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };

    var backup_dir = try cwd.openDir("backup", .{});
    defer backup_dir.close();

    // copy ไฟล์
    try copyFile(cwd, "original.txt", backup_dir, "original.txt.bak");
    std.debug.print("Copy file สำเร็จ\n", .{});
}
```

### การใช้ Dir.symLink

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ต้นฉบับ
    try cwd.writeFile(.{
        .sub_path = "target.txt",
        .data = "Target file content\n",
    });

    // สร้าง symbolic link (Unix เท่านั้น)
    // target คือไฟล์ที่ link ชี้ไป
    // sym_link_path คือชื่อของ link เอง
    try cwd.symLink("target.txt", "link.txt", .{});

    std.debug.print("สร้าง symlink สำเร็จ\n", .{});

    // อ่านจาก link (จะอ่านจาก target)
    const content = try cwd.readFileAlloc(
        std.heap.page_allocator,
        "link.txt",
        1024,
    );
    defer std.heap.page_allocator.free(content);

    std.debug.print("เนื้อหา (จาก link): {s}", .{content});
}
```

### การใช้ Dir.rename และ Dir.renamePath

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ทดสอบ
    try cwd.writeFile(.{ .sub_path = "old_name.txt", .data = "content" });

    // rename ภายใน directory เดียวกัน
    try cwd.rename("old_name.txt", "new_name.txt");
    std.debug.print("Rename สำเร็จ\n", .{});

    // rename ข้าม directories
    try cwd.makeDir("dest");
    var dest = try cwd.openDir("dest", .{});
    defer dest.close();

    // ย้ายไฟล์จาก cwd ไปยัง dest
    try std.fs.rename(cwd, "new_name.txt", dest, "moved.txt");
    std.debug.print("ย้ายไฟล์ข้าม directory สำเร็จ\n", .{});
}
```

---

## Step 299: File Permissions และ Modes

### การสร้างไฟล์ด้วย permissions เฉพาะ

```zig
const std = @import("std");

pub fn main() !void {
    // สร้างไฟล์พร้อมกำหนด permissions
    const file = try std.fs.cwd().createFile("script.sh", .{
        // mode: ใช้ Unix permission bits
        // 0o755 = rwxr-xr-x (owner: read/write/execute, group/other: read/execute)
        .mode = 0o755,
    });
    defer file.close();

    try file.writeAll("#!/bin/bash\necho 'Hello'\n");

    std.debug.print("สร้าง script.sh ด้วย permission 755 สำเร็จ\n", .{});
}
```

### การอ่านและแก้ไข permissions

```zig
const std = @import("std");

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้างไฟล์ทดสอบ
    const f = try cwd.createFile("test_perms.txt", .{});
    try f.writeAll("test");
    f.close();

    // อ่าน stat เพื่อดู mode
    const stat = try cwd.statFile("test_perms.txt");
    std.debug.print("Mode เดิม: {o:0>4}\n", .{stat.mode & 0o7777});

    // เปลี่ยน permissions
    // ใช้ chmod ผ่าน file handle
    const file2 = try cwd.openFile("test_perms.txt", .{});
    defer file2.close();

    try file2.chmod(0o644);
    std.debug.print("เปลี่ยน permissions เป็น 644 สำเร็จ\n", .{});

    // ตรวจสอบ permissions ใหม่
    const new_stat = try cwd.statFile("test_perms.txt");
    std.debug.print("Mode ใหม่: {o:0>4}\n", .{new_stat.mode & 0o7777});
}
```

### ความหมายของ permission bits

```zig
const std = @import("std");

/// แสดง permission string แบบ Unix (rwxrwxrwx)
fn formatPermissions(mode: u32) [10]u8 {
    var result: [10]u8 = [_]u8{'-'} ** 10;

    // file type
    const kind = mode & 0o170000;
    if (kind == 0o040000) result[0] = 'd'; // directory
    if (kind == 0o120000) result[0] = 'l'; // symlink

    // owner permissions (bits 8-6)
    if (mode & 0o400 != 0) result[1] = 'r';
    if (mode & 0o200 != 0) result[2] = 'w';
    if (mode & 0o100 != 0) result[3] = 'x';

    // group permissions (bits 5-3)
    if (mode & 0o040 != 0) result[4] = 'r';
    if (mode & 0o020 != 0) result[5] = 'w';
    if (mode & 0o010 != 0) result[6] = 'x';

    // other permissions (bits 2-0)
    if (mode & 0o004 != 0) result[7] = 'r';
    if (mode & 0o002 != 0) result[8] = 'w';
    if (mode & 0o001 != 0) result[9] = 'x';

    return result;
}

pub fn main() !void {
    var dir = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer dir.close();

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        const stat = dir.statFile(entry.name) catch continue;
        const perms = formatPermissions(stat.mode);
        std.debug.print("{s} {s:>8} {s}\n", .{
            &perms,
            formatSize(stat.size),
            entry.name,
        });
    }
}

fn formatSize(size: u64) [8]u8 {
    var buf: [8]u8 = undefined;
    const result = std.fmt.bufPrint(&buf, "{d}", .{size}) catch "???";
    var out = [_]u8{' '} ** 8;
    const start = 8 - result.len;
    @memcpy(out[start..], result);
    return out;
}
```

### การตรวจสอบสิทธิ์การเข้าถึง

```zig
const std = @import("std");

fn checkAccess(dir: std.fs.Dir, path: []const u8) void {
    // ตรวจสอบ read access
    const read_ok = blk: {
        const f = dir.openFile(path, .{ .mode = .read_only }) catch break :blk false;
        f.close();
        break :blk true;
    };

    // ตรวจสอบ write access
    const write_ok = blk: {
        const f = dir.openFile(path, .{ .mode = .read_write }) catch break :blk false;
        f.close();
        break :blk true;
    };

    std.debug.print("{s}: read={s}, write={s}\n", .{
        path,
        if (read_ok) "yes" else "no",
        if (write_ok) "yes" else "no",
    });
}

pub fn main() void {
    const cwd = std.fs.cwd();
    checkAccess(cwd, "build.zig");
    checkAccess(cwd, "/etc/passwd");
    checkAccess(cwd, "/etc/shadow"); // ปกติจะไม่มีสิทธิ์อ่าน
}
```

### การทำงานกับ umask

```zig
const std = @import("std");

pub fn main() !void {
    // ใน Zig ไม่มี built-in umask API
    // แต่สามารถใช้ syscall ได้โดยตรง (Unix เท่านั้น)

    if (std.Target.current.os.tag == .linux or
        std.Target.current.os.tag == .macos)
    {
        // ตั้ง umask (Unix เท่านั้น)
        const old_umask = std.os.umask(0o022);
        defer _ = std.os.umask(old_umask);

        // ไฟล์ที่สร้างจะได้ permission = requested & ~umask
        // ถ้า request 0o666 และ umask 0o022 = 0o644
        const f = try std.fs.cwd().createFile("umask_test.txt", .{
            .mode = 0o666,
        });
        defer f.close();

        const stat = try f.stat();
        std.debug.print("Permission หลัง umask: {o:0>4}\n", .{stat.mode & 0o7777});
    }
}
```

---

## Step 300: Recursive File Operations

### Recursive directory listing (tree)

```zig
const std = @import("std");

fn printTree(dir: std.fs.Dir, prefix: []const u8, depth: u32, max_depth: u32) !void {
    if (depth > max_depth) return;

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        // ข้าม hidden files
        if (entry.name[0] == '.') continue;

        std.debug.print("{s}├── {s}", .{ prefix, entry.name });

        if (entry.kind == .directory) {
            std.debug.print("/\n", .{});

            var sub_dir = dir.openDir(entry.name, .{ .iterate = true }) catch continue;
            defer sub_dir.close();

            var new_prefix_buf: [256]u8 = undefined;
            const new_prefix = std.fmt.bufPrint(
                &new_prefix_buf,
                "{s}│   ",
                .{prefix},
            ) catch continue;

            try printTree(sub_dir, new_prefix, depth + 1, max_depth);
        } else {
            std.debug.print("\n", .{});
        }
    }
}

pub fn main() !void {
    std.debug.print(".\n", .{});

    var cwd = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer cwd.close();

    try printTree(cwd, "", 0, 3);
}
```

### Recursive directory copy

```zig
const std = @import("std");

fn copyDirRecursive(
    src_dir: std.fs.Dir,
    dst_parent: std.fs.Dir,
    dir_name: []const u8,
) !void {
    // สร้าง destination directory
    dst_parent.makeDir(dir_name) catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };

    var dst_dir = try dst_parent.openDir(dir_name, .{});
    defer dst_dir.close();

    var src_sub = try src_dir.openDir(dir_name, .{ .iterate = true });
    defer src_sub.close();

    var iter = src_sub.iterate();
    while (try iter.next()) |entry| {
        switch (entry.kind) {
            .file => {
                // copy ไฟล์
                try src_sub.copyFile(entry.name, dst_dir, entry.name, .{});
                std.debug.print("  copied file: {s}/{s}\n", .{ dir_name, entry.name });
            },
            .directory => {
                // recursive copy subdirectory
                try copyDirRecursive(src_sub, dst_dir, entry.name);
            },
            else => {}, // ข้าม special files
        }
    }
}

pub fn main() !void {
    const cwd = std.fs.cwd();

    // สร้าง test structure
    try cwd.makePath("source/subdir");
    try cwd.writeFile(.{ .sub_path = "source/file1.txt", .data = "content1" });
    try cwd.writeFile(.{ .sub_path = "source/subdir/file2.txt", .data = "content2" });

    // สร้าง destination
    cwd.makeDir("destination") catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };

    var dst = try cwd.openDir("destination", .{});
    defer dst.close();

    try copyDirRecursive(cwd, dst, "source");
    std.debug.print("Copy สำเร็จ\n", .{});
}
```

### Recursive file search

```zig
const std = @import("std");

const SearchResult = struct {
    path: []const u8,
};

fn searchFiles(
    allocator: std.mem.Allocator,
    dir: std.fs.Dir,
    current_path: []const u8,
    pattern: []const u8,
    results: *std.ArrayList([]const u8),
) !void {
    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.name[0] == '.') continue;

        const entry_path = try std.fs.path.join(
            allocator,
            &[_][]const u8{ current_path, entry.name },
        );
        // จะ free ใน caller หรือเก็บใน results

        switch (entry.kind) {
            .file => {
                // ตรวจสอบว่าชื่อไฟล์ match pattern
                if (std.mem.indexOf(u8, entry.name, pattern) != null) {
                    try results.append(entry_path);
                } else {
                    allocator.free(entry_path);
                }
            },
            .directory => {
                defer allocator.free(entry_path);
                var sub_dir = dir.openDir(entry.name, .{ .iterate = true }) catch continue;
                defer sub_dir.close();

                // recursive search
                try searchFiles(allocator, sub_dir, entry_path, pattern, results);
            },
            else => allocator.free(entry_path),
        }
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var results = std.ArrayList([]const u8).init(allocator);
    defer {
        for (results.items) |path| allocator.free(path);
        results.deinit();
    }

    var cwd = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer cwd.close();

    std.debug.print("ค้นหาไฟล์ .zig:\n", .{});
    try searchFiles(allocator, cwd, ".", ".zig", &results);

    for (results.items) |path| {
        std.debug.print("  {s}\n", .{path});
    }
    std.debug.print("พบ {d} ไฟล์\n", .{results.items.len});
}
```

### Recursive delete พร้อม error handling

```zig
const std = @import("std");

fn deleteRecursive(dir: std.fs.Dir, name: []const u8) !void {
    var sub_dir = dir.openDir(name, .{ .iterate = true }) catch |err| {
        if (err == error.NotDir) {
            // เป็นไฟล์ ลบตรง ๆ
            return dir.deleteFile(name);
        }
        return err;
    };

    // เป็น directory
    {
        defer sub_dir.close();
        var iter = sub_dir.iterate();
        while (try iter.next()) |entry| {
            try deleteRecursive(sub_dir, entry.name);
        }
    }

    // ลบ directory เปล่า
    try dir.deleteDir(name);
    std.debug.print("ลบ: {s}\n", .{name});
}

pub fn main() !void {
    // สร้าง test structure
    const cwd = std.fs.cwd();
    try cwd.makePath("to_delete/a/b");
    try cwd.makePath("to_delete/c");
    try cwd.writeFile(.{ .sub_path = "to_delete/file.txt", .data = "x" });
    try cwd.writeFile(.{ .sub_path = "to_delete/a/file2.txt", .data = "y" });

    std.debug.print("ลบ directory tree:\n", .{});
    try deleteRecursive(cwd, "to_delete");
    std.debug.print("ลบสำเร็จ\n", .{});
}
```

### Recursive size calculation

```zig
const std = @import("std");

fn calculateDirSize(dir: std.fs.Dir) !u64 {
    var total: u64 = 0;
    var iter = dir.iterate();

    while (try iter.next()) |entry| {
        switch (entry.kind) {
            .file => {
                const stat = dir.statFile(entry.name) catch continue;
                total += stat.size;
            },
            .directory => {
                if (entry.name[0] == '.') continue;
                var sub_dir = dir.openDir(entry.name, .{ .iterate = true }) catch continue;
                defer sub_dir.close();

                total += try calculateDirSize(sub_dir);
            },
            else => {},
        }
    }

    return total;
}

fn formatBytes(bytes: u64) struct { value: f64, unit: []const u8 } {
    if (bytes < 1024) return .{ .value = @floatFromInt(bytes), .unit = "B" };
    if (bytes < 1024 * 1024) return .{ .value = @as(f64, @floatFromInt(bytes)) / 1024.0, .unit = "KB" };
    if (bytes < 1024 * 1024 * 1024) return .{ .value = @as(f64, @floatFromInt(bytes)) / (1024.0 * 1024.0), .unit = "MB" };
    return .{ .value = @as(f64, @floatFromInt(bytes)) / (1024.0 * 1024.0 * 1024.0), .unit = "GB" };
}

pub fn main() !void {
    var cwd = try std.fs.cwd().openDir(".", .{ .iterate = true });
    defer cwd.close();

    const total_size = try calculateDirSize(cwd);
    const formatted = formatBytes(total_size);

    std.debug.print("ขนาดรวมของ directory: {d:.2} {s} ({d} bytes)\n", .{
        formatted.value,
        formatted.unit,
        total_size,
    });
}
```

---

## Real-World Patterns

### Pattern 1: Config File Manager

```zig
const std = @import("std");

const ConfigManager = struct {
    allocator: std.mem.Allocator,
    config_path: []const u8,
    data: std.StringHashMap([]const u8),

    pub fn init(allocator: std.mem.Allocator, config_path: []const u8) ConfigManager {
        return .{
            .allocator = allocator,
            .config_path = config_path,
            .data = std.StringHashMap([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *ConfigManager) void {
        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.data.deinit();
    }

    pub fn load(self: *ConfigManager) !void {
        const content = std.fs.cwd().readFileAlloc(
            self.allocator,
            self.config_path,
            64 * 1024,
        ) catch |err| {
            if (err == error.FileNotFound) return; // ไม่มีไฟล์ = ใช้ default
            return err;
        };
        defer self.allocator.free(content);

        // parse key=value format
        var lines = std.mem.splitScalar(u8, content, '\n');
        while (lines.next()) |line| {
            const trimmed = std.mem.trim(u8, line, " \t\r");
            if (trimmed.len == 0 or trimmed[0] == '#') continue;

            const eq_pos = std.mem.indexOf(u8, trimmed, "=") orelse continue;
            const key = std.mem.trim(u8, trimmed[0..eq_pos], " ");
            const value = std.mem.trim(u8, trimmed[eq_pos + 1 ..], " ");

            const key_copy = try self.allocator.dupe(u8, key);
            errdefer self.allocator.free(key_copy);
            const value_copy = try self.allocator.dupe(u8, value);
            errdefer self.allocator.free(value_copy);

            try self.data.put(key_copy, value_copy);
        }
    }

    pub fn save(self: *ConfigManager) !void {
        const file = try std.fs.cwd().createFile(self.config_path, .{});
        defer file.close();

        var buf_writer = std.io.bufferedWriter(file.writer());
        const writer = buf_writer.writer();

        try writer.writeAll("# Application Configuration\n");
        try writer.writeAll("# Generated automatically\n\n");

        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            try writer.print("{s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        }

        try buf_writer.flush();
    }

    pub fn get(self: *ConfigManager, key: []const u8) ?[]const u8 {
        return self.data.get(key);
    }

    pub fn set(self: *ConfigManager, key: []const u8, value: []const u8) !void {
        const key_copy = try self.allocator.dupe(u8, key);
        errdefer self.allocator.free(key_copy);
        const value_copy = try self.allocator.dupe(u8, value);
        errdefer self.allocator.free(value_copy);

        if (try self.data.fetchPut(key_copy, value_copy)) |old| {
            self.allocator.free(old.key);
            self.allocator.free(old.value);
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var config = ConfigManager.init(allocator, "app.cfg");
    defer config.deinit();

    // โหลด config (ถ้ามี)
    try config.load();

    // ตั้งค่า default
    if (config.get("host") == null) try config.set("host", "localhost");
    if (config.get("port") == null) try config.set("port", "8080");
    if (config.get("debug") == null) try config.set("debug", "false");

    std.debug.print("Host: {s}\n", .{config.get("host") orelse "unknown"});
    std.debug.print("Port: {s}\n", .{config.get("port") orelse "unknown"});

    // บันทึก config
    try config.save();
    std.debug.print("บันทึก config สำเร็จ\n", .{});
}
```

### Pattern 2: Log File Rotator

```zig
const std = @import("std");

const Logger = struct {
    allocator: std.mem.Allocator,
    log_path: []const u8,
    max_size: u64,
    max_files: u32,

    pub fn init(
        allocator: std.mem.Allocator,
        log_path: []const u8,
        max_size: u64,
        max_files: u32,
    ) Logger {
        return .{
            .allocator = allocator,
            .log_path = log_path,
            .max_size = max_size,
            .max_files = max_files,
        };
    }

    pub fn log(self: *Logger, level: []const u8, message: []const u8) !void {
        // ตรวจสอบขนาดไฟล์
        const need_rotate = blk: {
            const stat = std.fs.cwd().statFile(self.log_path) catch break :blk false;
            break :blk stat.size >= self.max_size;
        };

        if (need_rotate) try self.rotate();

        // เปิดหรือสร้างไฟล์ log
        const file = try std.fs.cwd().createFile(self.log_path, .{
            .truncate = false,
            .exclusive = false,
        });
        defer file.close();

        try file.seekFromEnd(0);

        const timestamp = std.time.timestamp();
        try file.writer().print("[{d}] [{s}] {s}\n", .{ timestamp, level, message });
    }

    fn rotate(self: *Logger) !void {
        // ลบไฟล์เก่าที่สุด
        var i: u32 = self.max_files - 1;
        while (i > 0) : (i -= 1) {
            const old_name = try std.fmt.allocPrint(
                self.allocator,
                "{s}.{d}",
                .{ self.log_path, i },
            );
            defer self.allocator.free(old_name);

            const new_name = try std.fmt.allocPrint(
                self.allocator,
                "{s}.{d}",
                .{ self.log_path, i + 1 },
            );
            defer self.allocator.free(new_name);

            std.fs.cwd().rename(old_name, new_name) catch {};
        }

        // ย้ายไฟล์ปัจจุบัน
        const rotated_name = try std.fmt.allocPrint(
            self.allocator,
            "{s}.1",
            .{self.log_path},
        );
        defer self.allocator.free(rotated_name);

        try std.fs.cwd().rename(self.log_path, rotated_name);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var logger = Logger.init(allocator, "app.log", 1024 * 1024, 5); // 1MB, max 5 files

    try logger.log("INFO", "โปรแกรมเริ่มทำงาน");
    try logger.log("DEBUG", "เชื่อมต่อ database แล้ว");
    try logger.log("INFO", "รับ request จาก client");
    try logger.log("ERROR", "พบข้อผิดพลาด: timeout");

    std.debug.print("เขียน log สำเร็จ\n", .{});
}
```

### Pattern 3: File Watcher (Polling)

```zig
const std = @import("std");

const FileWatcher = struct {
    path: []const u8,
    last_mtime: i128,

    pub fn init(path: []const u8) !FileWatcher {
        const stat = try std.fs.cwd().statFile(path);
        return .{
            .path = path,
            .last_mtime = stat.mtime,
        };
    }

    pub fn hasChanged(self: *FileWatcher) !bool {
        const stat = try std.fs.cwd().statFile(self.path);
        if (stat.mtime != self.last_mtime) {
            self.last_mtime = stat.mtime;
            return true;
        }
        return false;
    }
};

pub fn main() !void {
    var watcher = FileWatcher.init("config.json") catch {
        std.debug.print("ไม่พบไฟล์ config.json\n", .{});
        return;
    };

    std.debug.print("กำลัง watch config.json...\n", .{});
    std.debug.print("(จำลอง 3 รอบ)\n", .{});

    var rounds: u32 = 0;
    while (rounds < 3) : (rounds += 1) {
        std.time.sleep(std.time.ns_per_s); // รอ 1 วินาที

        if (try watcher.hasChanged()) {
            std.debug.print("ไฟล์เปลี่ยนแปลงแล้ว! กำลัง reload...\n", .{});
        } else {
            std.debug.print("ไม่มีการเปลี่ยนแปลง\n", .{});
        }
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: File Statistics Reporter

เขียนโปรแกรมที่รับ path เป็น argument แล้วแสดงสถิติ:
- จำนวนไฟล์ทั้งหมด
- จำนวน directory ทั้งหมด
- ขนาดรวม
- ไฟล์ที่ใหญ่ที่สุด 5 อันดับ
- นามสกุลที่พบมากที่สุด

```zig
// โครงสร้างที่ควรเขียน:
const std = @import("std");

const Stats = struct {
    file_count: u64 = 0,
    dir_count: u64 = 0,
    total_size: u64 = 0,
    // เพิ่ม fields เพิ่มเติม...
};

fn scanDirectory(stats: *Stats, dir: std.fs.Dir, path: []const u8, allocator: std.mem.Allocator) !void {
    // TODO: implement recursive scan
    _ = stats;
    _ = dir;
    _ = path;
    _ = allocator;
}

pub fn main() !void {
    // TODO: รับ argument และเรียก scanDirectory
}
```

### แบบฝึกหัดที่ 2: Simple Backup Tool

เขียน backup tool ที่:
- รับ source directory และ destination directory
- Copy ไฟล์ทั้งหมดแบบ recursive
- ข้าม hidden files (ที่ขึ้นต้นด้วย '.')
- แสดง progress
- สร้าง backup manifest file (list of backed up files)

```zig
// โครงสร้างที่ควรเขียน:
const std = @import("std");

fn backup(
    src: std.fs.Dir,
    dst: std.fs.Dir,
    manifest: anytype,
    prefix: []const u8,
) !struct { files: u32, size: u64 } {
    // TODO: implement backup logic
    _ = src;
    _ = dst;
    _ = manifest;
    _ = prefix;
    return .{ .files = 0, .size = 0 };
}
```

### แบบฝึกหัดที่ 3: Config File Parser

เขียน parser สำหรับ INI-style config file:

```ini
[database]
host = localhost
port = 5432
name = myapp

[server]
host = 0.0.0.0
port = 8080
workers = 4
```

ต้องทำได้:
- Parse sections (`[section]`)
- Parse key-value pairs
- Comment lines (ขึ้นต้นด้วย `;` หรือ `#`)
- Get value by `section.key`
- Save back to file

### แบบฝึกหัดที่ 4: Log Analyzer

เขียนโปรแกรมวิเคราะห์ log file:
- อ่าน log ทีละบรรทัด
- นับจำนวน ERROR, WARNING, INFO
- หา unique IP addresses (ถ้า log มี IP)
- แสดง top 10 error messages
- Export summary เป็น JSON file

### แบบฝึกหัดที่ 5: Directory Synchronizer

เขียน tool ที่ sync สอง directory:
- เปรียบเทียบไฟล์ทั้งสองฝั่ง
- Copy ไฟล์ใหม่หรือที่ถูกแก้ไขจาก source ไป destination
- Option: ลบไฟล์ใน destination ที่ไม่มีใน source
- แสดง diff ก่อน sync

---

## สรุปบทเรียน

### สิ่งที่ได้เรียนรู้ใน Part 30

| Step | หัวข้อ | สิ่งสำคัญ |
|------|--------|-----------|
| 291 | std.fs.cwd() | จุดเริ่มต้นสำหรับ file operations, คืน Dir struct |
| 292 | การอ่านไฟล์ | openFile, readToEndAlloc, buffered reading |
| 293 | การเขียนไฟล์ | createFile, buffered writing, atomic write |
| 294 | Directory ops | makeDir, makePath, deleteDir, deleteTree |
| 295 | File metadata | stat(), statFile(), mtime, size, kind |
| 296 | Directory listing | iterate(), filter by kind/extension |
| 297 | Path operations | join, dirname, basename, extension, isAbsolute |
| 298 | Dir methods | ครอบคลุม API ทั้งหมดของ std.fs.Dir |
| 299 | Permissions | mode bits, chmod, access checking |
| 300 | Recursive ops | tree listing, recursive copy/delete/search |

### Best Practices

1. **ใช้ `defer` เสมอ** สำหรับ file.close() และ dir.close()
2. **ใช้ buffered I/O** เมื่อ read/write จำนวนมาก
3. **จัดการ error ทุก case** โดยเฉพาะ FileNotFound, AccessDenied
4. **ใช้ atomic write** สำหรับไฟล์สำคัญ (เขียน temp แล้ว rename)
5. **ระวัง path separator** ใช้ std.fs.path.join แทนการ concatenate string
6. **ตรวจสอบก่อนลบ** โดยเฉพาะ deleteTree ซึ่งไม่สามารถ undo ได้

### Common Errors และวิธีแก้

```
error.FileNotFound     - ไฟล์/directory ไม่มีอยู่
error.PathAlreadyExists - ไฟล์/directory มีอยู่แล้ว (ใช้กับ createFile exclusive mode)
error.AccessDenied     - ไม่มีสิทธิ์เข้าถึง
error.IsDir            - พยายาม openFile บน directory
error.NotDir           - พยายาม openDir บนไฟล์
error.DirNotEmpty      - พยายาม deleteDir บน directory ที่ไม่ว่าง
error.NoSpaceLeft      - disk เต็ม
error.FileTooBig       - ไฟล์ใหญ่เกิน max_bytes ที่กำหนด
```

### Memory Management ที่สำคัญ

```zig
// Pattern 1: readFileAlloc - ต้อง free เอง
const content = try dir.readFileAlloc(allocator, "file.txt", max_size);
defer allocator.free(content);

// Pattern 2: path.join - ต้อง free เอง
const full_path = try std.fs.path.join(allocator, &parts);
defer allocator.free(full_path);

// Pattern 3: realpathAlloc - ต้อง free เอง
const abs_path = try dir.realpathAlloc(allocator, ".");
defer allocator.free(abs_path);

// Pattern 4: File และ Dir - ต้อง close
const file = try dir.openFile("data", .{});
defer file.close(); // ต้อง close เสมอ
```

### การเลือกใช้ API ที่เหมาะสม

```
ไฟล์เล็ก (< 1MB):   readFileAlloc หรือ writeFile
ไฟล์ใหญ่:           openFile + buffered reader/writer
เขียนครั้งเดียว:     createFile + writeAll
ต่อท้ายไฟล์:        openFile(read_write) + seekFromEnd(0)
ข้อมูลสำคัญ:        atomic write (temp + rename)
```

---

## ขั้นตอนต่อไป

หลังจากเรียนรู้ `std.fs` แล้ว คุณสามารถศึกษาต่อใน:

- **Part 31**: `std.json` - การทำงานกับ JSON format
- **Part 32**: `std.net` - Network programming
- **Part 33**: `std.process` - การรันโปรแกรมอื่น
- **Part 34**: `std.Thread` - Concurrent file operations

การเข้าใจ `std.fs` เป็นพื้นฐานสำคัญสำหรับการเขียนโปรแกรมที่ทำงานกับไฟล์จริง ๆ ไม่ว่าจะเป็น config management, logging, data processing หรือ build tools ต่าง ๆ

---

*จบ Part 30: std.fs ใน Zig Standard Library*
*Steps 291-300*
