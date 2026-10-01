# ส่วนที่ 29: std.io ใน Zig Standard Library

## Steps 281-290: การจัดการ Input/Output ใน Zig

---

## บทนำ

`std.io` เป็นโมดูลสำคัญในไลบรารีมาตรฐานของ Zig ที่จัดการเรื่องการรับส่งข้อมูล (Input/Output) ทั้งหมด ตั้งแต่การอ่านข้อมูลจาก stdin การเขียนข้อมูลไปยัง stdout และ stderr ไปจนถึงการสร้าง custom reader/writer สำหรับกรณีพิเศษ

ใน Part นี้เราจะเรียนรู้:
- การใช้งาน standard streams (stdin, stdout, stderr)
- Writer interface และ Reader interface
- การใช้ buffered I/O เพื่อประสิทธิภาพที่ดีขึ้น
- รูปแบบการอ่าน/เขียนข้อมูลต่างๆ
- การสร้าง custom reader/writer ของตัวเอง

---

## Step 281: std.io.getStdIn, getStdOut, getStdErr

### ทำความรู้จักกับ Standard Streams

ใน Zig, standard streams ทั้งสามแบบสามารถเข้าถึงได้ผ่านฟังก์ชันใน `std.io`:

- `std.io.getStdIn()` - รับ file handle สำหรับ standard input (keyboard input)
- `std.io.getStdOut()` - รับ file handle สำหรับ standard output (terminal output)
- `std.io.getStdErr()` - รับ file handle สำหรับ standard error (error messages)

ฟังก์ชันเหล่านี้คืนค่าเป็น `std.fs.File` ซึ่งเป็น struct ที่ห่อหุ้ม file descriptor ของระบบปฏิบัติการ

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    // รับ standard streams
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    const stderr = std.io.getStdErr();

    // ตรวจสอบว่าเป็น terminal หรือไม่
    if (stdout.isTty()) {
        try stdout.writeAll("กำลังทำงานใน terminal\n");
    } else {
        try stdout.writeAll("กำลังทำงานใน pipe หรือ redirection\n");
    }

    // เขียนไปยัง stderr
    try stderr.writeAll("นี่คือข้อความ error\n");

    // แสดง file descriptors
    _ = stdin;
    _ = stdout;
    _ = stderr;
}
```

### ความแตกต่างระหว่าง stdout และ stderr

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const stderr = std.io.getStdErr();

    // stdout ใช้สำหรับผลลัพธ์ปกติ
    try stdout.writeAll("ผลลัพธ์โปรแกรม: สำเร็จ\n");
    try stdout.writeAll("จำนวนรายการที่ประมวลผล: 42\n");

    // stderr ใช้สำหรับ error messages และ diagnostics
    try stderr.writeAll("คำเตือน: ไฟล์ไม่พบ\n");
    try stderr.writeAll("ข้อผิดพลาด: การเชื่อมต่อล้มเหลว\n");
}
```

### การใช้ writer() method

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const stderr = std.io.getStdErr();

    // writer() คืนค่า Writer struct สำหรับใช้กับ std.fmt.format
    const out = stdout.writer();
    const err = stderr.writer();

    // ใช้ print เพื่อ format ข้อความ
    try out.print("สวัสดี {s}! คุณอายุ {d} ปี\n", .{ "โลก", 25 });
    try err.print("ข้อผิดพลาด: {s}\n", .{"ไม่สามารถเปิดไฟล์ได้"});
}
```

### ข้อมูลเพิ่มเติมเกี่ยวกับ std.fs.File

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();

    // ตรวจสอบ metadata ของ file
    const stat = try stdout.stat();
    _ = stat;

    // ตรวจสอบว่าสามารถอ่าน/เขียนได้หรือไม่
    const stdout_writer = stdout.writer();
    try stdout_writer.print("stdout handle: {}\n", .{stdout.handle});

    // การใช้ writeAll และ write
    try stdout.writeAll("writeAll เขียนข้อมูลทั้งหมด\n");
    const bytes_written = try stdout.write("write คืนค่าจำนวน bytes ที่เขียน\n");
    _ = bytes_written;
}
```

---

## Step 282: Writer Interface

### ทำความเข้าใจ Writer Interface

Writer ใน Zig เป็น interface (comptime duck typing) ที่กำหนดว่าต้องมีฟังก์ชัน `write` ที่รับ `[]const u8` และคืนค่า `anyerror!usize`

```zig
const std = @import("std");

// Writer interface มีรูปแบบดังนี้:
// pub fn write(self: Self, bytes: []const u8) anyerror!usize

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const writer = stdout.writer();

    // วิธีต่างๆ ในการใช้ writer
    
    // 1. writeAll - เขียนข้อมูลทั้งหมด (retry ถ้าเขียนไม่ครบ)
    try writer.writeAll("สวัสดีชาวโลก\n");

    // 2. writeByte - เขียน 1 byte
    try writer.writeByte('A');
    try writer.writeByte('\n');

    // 3. writeByteNTimes - เขียน byte ซ้ำ n ครั้ง
    try writer.writeByteNTimes('=', 20);
    try writer.writeByte('\n');

    // 4. print - เขียนด้วย format string
    try writer.print("ค่า: {d}, ข้อความ: {s}\n", .{ 42, "hello" });

    // 5. writeBytesNTimes - เขียน slice ซ้ำ n ครั้ง
    try writer.writeBytesNTimes("ab", 5);
    try writer.writeByte('\n');
}
```

### การสร้าง Generic Function ที่รับ Writer

```zig
const std = @import("std");

// ฟังก์ชันที่รับ anytype writer - ทำงานกับ writer ใดก็ได้
fn writeGreeting(writer: anytype, name: []const u8) !void {
    try writer.print("สวัสดีคุณ {s}!\n", .{name});
    try writer.print("ยินดีต้อนรับสู่โปรแกรม Zig\n", .{});
}

// ฟังก์ชันที่แสดงตาราง
fn printTable(writer: anytype, data: []const []const u8) !void {
    try writer.writeAll("+--------------------+\n");
    try writer.writeAll("| รายการ             |\n");
    try writer.writeAll("+--------------------+\n");
    for (data) |item| {
        try writer.print("| {s:<18} |\n", .{item});
    }
    try writer.writeAll("+--------------------+\n");
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const writer = stdout.writer();

    try writeGreeting(writer, "สมชาย");
    try writer.writeByte('\n');

    const items = [_][]const u8{ "แอปเปิ้ล", "กล้วย", "ส้ม", "องุ่น" };
    try printTable(writer, &items);
}
```

### Writer สำหรับ ArrayList (Memory Buffer)

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง ArrayList เพื่อเก็บข้อมูลใน memory
    var buffer = std.ArrayList(u8).init(allocator);
    defer buffer.deinit();

    // ใช้ writer เขียนไปยัง buffer
    const writer = buffer.writer();
    try writer.writeAll("บรรทัดที่ 1\n");
    try writer.print("บรรทัดที่ {d}\n", .{2});
    try writer.writeAll("บรรทัดที่ 3\n");

    // แสดงผลลัพธ์
    const stdout = std.io.getStdOut();
    try stdout.print("ข้อมูลใน buffer ({d} bytes):\n", .{buffer.items.len});
    try stdout.writeAll(buffer.items);
}
```

### การนับ bytes ที่เขียน (CountingWriter)

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();

    // ห่อ writer ด้วย counting writer
    var counting_writer = std.io.countingWriter(stdout.writer());
    const writer = counting_writer.writer();

    try writer.writeAll("สวัสดี\n");
    try writer.print("นับ: {d}\n", .{42});
    try writer.writeAll("ลาก่อน\n");

    // ดูจำนวน bytes ที่เขียนทั้งหมด
    try stdout.print("\nเขียนทั้งหมด: {d} bytes\n", .{counting_writer.bytes_written});
}
```

### Multi-Writer (เขียนไปหลายที่พร้อมกัน)

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdout = std.io.getStdOut();
    
    // สร้าง buffer สำหรับเก็บ log
    var log_buffer = std.ArrayList(u8).init(allocator);
    defer log_buffer.deinit();

    // เขียนไปยัง stdout
    try stdout.writer().print("ข้อความสำคัญ\n", .{});
    
    // เขียนเดียวกันไปยัง log buffer
    try log_buffer.writer().print("ข้อความสำคัญ\n", .{});

    // แสดง log ที่เก็บไว้
    try stdout.writer().print("Log: {s}", .{log_buffer.items});
}
```

---

## Step 283: Reader Interface

### ทำความเข้าใจ Reader Interface

Reader interface กำหนดว่าต้องมีฟังก์ชัน `read` ที่รับ buffer `[]u8` และคืนค่า `anyerror!usize` (จำนวน bytes ที่อ่านได้)

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn();
    const reader = stdin.reader();

    // อ่านข้อมูลพื้นฐาน
    var buffer: [1024]u8 = undefined;
    
    // read - อ่านข้อมูลเท่าที่มี (อาจไม่เต็ม buffer)
    const bytes_read = try reader.read(&buffer);
    
    const stdout = std.io.getStdOut();
    try stdout.print("อ่านได้ {d} bytes\n", .{bytes_read});
    try stdout.print("ข้อมูล: {s}", .{buffer[0..bytes_read]});
}
```

### วิธีต่างๆ ในการอ่านข้อมูล

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง reader จาก string สำหรับทดสอบ
    const data = "สวัสดี\nโลก\nZig\n";
    var stream = std.io.fixedBufferStream(data);
    const reader = stream.reader();
    
    const stdout = std.io.getStdOut();
    
    // 1. readByte - อ่าน 1 byte
    const first_byte = try reader.readByte();
    try stdout.print("byte แรก: {c} (0x{x})\n", .{ first_byte, first_byte });

    // 2. readNoEof - อ่านข้อมูลเต็ม buffer
    var small_buf: [3]u8 = undefined;
    try reader.readNoEof(&small_buf);
    try stdout.print("3 bytes: {s}\n", .{&small_buf});

    // 3. readAll - อ่านข้อมูลทั้งหมดเท่าที่ buffer รองรับ
    var large_buf: [100]u8 = undefined;
    const n = try reader.readAll(&large_buf);
    try stdout.print("อ่านได้ {d} bytes ที่เหลือ: {s}", .{ n, large_buf[0..n] });
}
```

### การอ่านข้อมูลด้วย readUntilDelimiter

```zig
const std = @import("std");

pub fn main() !void {
    const data = "บรรทัด1\nบรรทัด2\nบรรทัด3\n";
    var stream = std.io.fixedBufferStream(data);
    const reader = stream.reader();
    
    const stdout = std.io.getStdOut();

    var line_buf: [256]u8 = undefined;
    var line_num: usize = 1;

    while (true) {
        // อ่านจนถึง delimiter '\n' หรือ eof
        const line = reader.readUntilDelimiterOrEof(&line_buf, '\n') catch |err| {
            if (err == error.StreamTooLong) {
                try stdout.writeAll("บรรทัดยาวเกินไป!\n");
                break;
            }
            return err;
        };

        if (line) |l| {
            try stdout.print("บรรทัด {d}: {s}\n", .{ line_num, l });
            line_num += 1;
        } else {
            break; // EOF
        }
    }
}
```

### Fixed Buffer Reader

```zig
const std = @import("std");

pub fn main() !void {
    const json_data =
        \\{"name": "สมชาย", "age": 30}
    ;
    
    var fbs = std.io.fixedBufferStream(json_data);
    const reader = fbs.reader();
    
    const stdout = std.io.getStdOut();
    
    // อ่านทีละ chunk
    var chunk: [8]u8 = undefined;
    var total: usize = 0;
    
    while (true) {
        const n = try reader.read(&chunk);
        if (n == 0) break;
        total += n;
        try stdout.print("chunk: {s}\n", .{chunk[0..n]});
    }
    
    try stdout.print("อ่านทั้งหมด: {d} bytes\n", .{total});
}
```

### Generic Reader Function

```zig
const std = @import("std");

// ฟังก์ชันที่รับ anytype reader
fn countLines(reader: anytype) !usize {
    var count: usize = 0;
    var buf: [4096]u8 = undefined;
    
    while (true) {
        const n = try reader.read(&buf);
        if (n == 0) break;
        
        for (buf[0..n]) |byte| {
            if (byte == '\n') count += 1;
        }
    }
    
    return count;
}

fn sumNumbers(reader: anytype, allocator: std.mem.Allocator) !i64 {
    var sum: i64 = 0;
    var line_buf = std.ArrayList(u8).init(allocator);
    defer line_buf.deinit();
    
    while (true) {
        line_buf.clearRetainingCapacity();
        
        reader.readUntilDelimiterArrayList(&line_buf, '\n', 1024) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };
        
        const trimmed = std.mem.trim(u8, line_buf.items, " \t\r\n");
        if (trimmed.len == 0) continue;
        
        const num = std.fmt.parseInt(i64, trimmed, 10) catch continue;
        sum += num;
    }
    
    return sum;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const data = "สวัสดี\nโลก\nZig\nProgram\n";
    var stream1 = std.io.fixedBufferStream(data);
    const lines = try countLines(stream1.reader());
    
    const stdout = std.io.getStdOut();
    try stdout.print("จำนวนบรรทัด: {d}\n", .{lines});
    
    const numbers = "10\n20\n30\n40\n50\n";
    var stream2 = std.io.fixedBufferStream(numbers);
    const total = try sumNumbers(stream2.reader(), allocator);
    try stdout.print("ผลรวม: {d}\n", .{total});
}
```

---

## Step 284: std.io.bufferedWriter, bufferedReader

### ทำไมต้องใช้ Buffered I/O?

การ I/O โดยตรงกับระบบปฏิบัติการทุกครั้งมีค่าใช้จ่ายสูง (system call overhead) Buffered I/O จะสะสมข้อมูลใน buffer ก่อนแล้วส่งครั้งเดียว ทำให้ประสิทธิภาพดีขึ้นมาก

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    // Unbuffered writer - ทุก write() เรียก system call
    const unbuffered_writer = stdout.writer();
    
    // Buffered writer - เก็บข้อมูลใน buffer ก่อน
    var bw = std.io.bufferedWriter(stdout.writer());
    const buffered_writer = bw.writer();
    
    // เขียนหลายบรรทัดด้วย buffered writer
    for (0..10) |i| {
        try buffered_writer.print("บรรทัดที่ {d}\n", .{i + 1});
    }
    
    // สำคัญมาก: ต้อง flush เพื่อส่งข้อมูลที่ค้างใน buffer
    try bw.flush();
    
    // เปรียบเทียบ
    try unbuffered_writer.writeAll("Unbuffered: ส่งทันที\n");
}
```

### ขนาด Buffer ใน bufferedWriter

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    // bufferedWriter มี buffer เริ่มต้น 4096 bytes
    var bw = std.io.bufferedWriter(stdout.writer());
    
    // หรือกำหนดขนาด buffer เอง
    var bw_custom = std.io.bufferedWriterSize(8192, stdout.writer());
    
    const writer1 = bw.writer();
    const writer2 = bw_custom.writer();
    
    try writer1.writeAll("ใช้ buffer 4096 bytes\n");
    try bw.flush();
    
    try writer2.writeAll("ใช้ buffer 8192 bytes\n");
    try bw_custom.flush();
}
```

### bufferedReader

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn();
    
    // สร้าง buffered reader
    var br = std.io.bufferedReader(stdin.reader());
    const reader = br.reader();
    
    const stdout = std.io.getStdOut();
    try stdout.writeAll("พิมพ์บางอย่างแล้วกด Enter: ");
    
    var buf: [256]u8 = undefined;
    if (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
        try stdout.print("คุณพิมพ์: {s}\n", .{line});
    }
}
```

### การใช้ Buffered I/O สำหรับการอ่าน CSV

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const csv_data =
        \\ชื่อ,อายุ,เมือง
        \\สมชาย,30,กรุงเทพ
        \\สมหญิง,25,เชียงใหม่
        \\สมศักดิ์,35,ภูเก็ต
    ;
    
    var fbs = std.io.fixedBufferStream(csv_data);
    
    // ใช้ buffered reader
    var br = std.io.bufferedReader(fbs.reader());
    const reader = br.reader();
    
    const stdout = std.io.getStdOut();
    
    // ใช้ buffered writer สำหรับ output
    var bw = std.io.bufferedWriter(stdout.writer());
    const writer = bw.writer();
    
    var line_buf = std.ArrayList(u8).init(allocator);
    defer line_buf.deinit();
    
    var row_num: usize = 0;
    
    while (true) {
        line_buf.clearRetainingCapacity();
        
        reader.readUntilDelimiterArrayList(&line_buf, '\n', 1024) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };
        
        if (row_num == 0) {
            try writer.print("--- {s} ---\n", .{line_buf.items});
        } else {
            // แยก fields
            var field_iter = std.mem.splitScalar(u8, line_buf.items, ',');
            const name = field_iter.next() orelse "";
            const age = field_iter.next() orelse "";
            const city = field_iter.next() orelse "";
            
            try writer.print("ชื่อ: {s} | อายุ: {s} | เมือง: {s}\n", .{ name, age, city });
        }
        
        row_num += 1;
    }
    
    // Flush buffer
    try bw.flush();
}
```

### Buffered I/O Performance Demo

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    // วิธีที่ 1: Buffered (แนะนำ)
    {
        var bw = std.io.bufferedWriter(stdout.writer());
        const writer = bw.writer();
        
        var i: usize = 0;
        while (i < 100) : (i += 1) {
            try writer.print("{d} ", .{i});
        }
        try writer.writeByte('\n');
        try bw.flush(); // ส่งข้อมูลทั้งหมดในคราวเดียว
    }
    
    // วิธีที่ 2: Unbuffered (ช้ากว่า)
    {
        const writer = stdout.writer();
        
        var i: usize = 0;
        while (i < 100) : (i += 1) {
            try writer.print("{d} ", .{i}); // แต่ละครั้งส่งทันที
        }
        try writer.writeByte('\n');
    }
}
```

---

## Step 285: readUntilDelimiterOrEof

### การทำงานของ readUntilDelimiterOrEof

`readUntilDelimiterOrEof` อ่านข้อมูลจนกว่าจะพบ delimiter ที่กำหนดหรือถึง EOF

```zig
const std = @import("std");

pub fn main() !void {
    const data = "บรรทัดที่หนึ่ง\nบรรทัดที่สอง\nบรรทัดที่สาม\n";
    var fbs = std.io.fixedBufferStream(data);
    const reader = fbs.reader();
    
    const stdout = std.io.getStdOut();
    var buf: [256]u8 = undefined;
    
    while (true) {
        // คืนค่า ?[]u8 - null เมื่อถึง EOF
        const line = try reader.readUntilDelimiterOrEof(&buf, '\n');
        
        if (line) |l| {
            try stdout.print(">> {s}\n", .{l});
        } else {
            try stdout.writeAll("ถึง EOF แล้ว\n");
            break;
        }
    }
}
```

### การจัดการ error StreamTooLong

```zig
const std = @import("std");

pub fn main() !void {
    const data = "บรรทัดสั้น\nบรรทัดที่ยาวมากๆ เกิน buffer ที่กำหนดไว้อย่างแน่นอน\nบรรทัดอื่น\n";
    var fbs = std.io.fixedBufferStream(data);
    const reader = fbs.reader();
    
    const stdout = std.io.getStdOut();
    var buf: [20]u8 = undefined; // buffer เล็กโดยเจตนา
    
    var line_num: usize = 1;
    
    while (true) {
        const result = reader.readUntilDelimiterOrEof(&buf, '\n');
        
        if (result) |maybe_line| {
            if (maybe_line) |line| {
                try stdout.print("บรรทัด {d}: {s}\n", .{ line_num, line });
            } else {
                break;
            }
        } else |err| {
            if (err == error.StreamTooLong) {
                try stdout.print("บรรทัด {d}: ยาวเกิน buffer!\n", .{line_num});
                // ต้องอ่านและทิ้งข้อมูลที่เหลือในบรรทัดนี้
                var skip_buf: [64]u8 = undefined;
                while (true) {
                    const skip = reader.readUntilDelimiterOrEof(&skip_buf, '\n') catch break;
                    if (skip) |_| break; // พบ newline แล้ว
                }
            } else {
                return err;
            }
        }
        
        line_num += 1;
    }
}
```

### readUntilDelimiterOrEof กับ delimiter ต่างๆ

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    // ตัวอย่าง 1: อ่านจนถึง comma
    {
        const data = "ฟิลด์1,ฟิลด์2,ฟิลด์3,ฟิลด์4";
        var fbs = std.io.fixedBufferStream(data);
        const reader = fbs.reader();
        var buf: [64]u8 = undefined;
        
        try stdout.writeAll("CSV Fields:\n");
        while (try reader.readUntilDelimiterOrEof(&buf, ',')) |field| {
            try stdout.print("  - {s}\n", .{field});
        }
    }
    
    // ตัวอย่าง 2: อ่านจนถึง null byte
    {
        const data = "string1\x00string2\x00string3\x00";
        var fbs = std.io.fixedBufferStream(data);
        const reader = fbs.reader();
        var buf: [64]u8 = undefined;
        
        try stdout.writeAll("\nNull-terminated strings:\n");
        while (try reader.readUntilDelimiterOrEof(&buf, 0)) |s| {
            try stdout.print("  - {s}\n", .{s});
        }
    }
    
    // ตัวอย่าง 3: อ่านจนถึง pipe
    {
        const data = "ชื่อ|อายุ|เมือง|อาชีพ";
        var fbs = std.io.fixedBufferStream(data);
        const reader = fbs.reader();
        var buf: [64]u8 = undefined;
        
        try stdout.writeAll("\nPipe-separated:\n");
        while (try reader.readUntilDelimiterOrEof(&buf, '|')) |field| {
            try stdout.print("  -> {s}\n", .{field});
        }
    }
}
```

### readUntilDelimiterArrayList

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const data = "บรรทัดที่ยาวมากๆ ที่ไม่รู้ว่าจะยาวแค่ไหน\nบรรทัดที่สอง\n";
    var fbs = std.io.fixedBufferStream(data);
    const reader = fbs.reader();
    
    const stdout = std.io.getStdOut();
    
    var line = std.ArrayList(u8).init(allocator);
    defer line.deinit();
    
    while (true) {
        line.clearRetainingCapacity();
        
        // ไม่มีการจำกัดขนาด buffer - ขยายตัวตาม ArrayList
        reader.readUntilDelimiterArrayList(&line, '\n', std.math.maxInt(usize)) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };
        
        try stdout.print("บรรทัด ({d} chars): {s}\n", .{ line.items.len, line.items });
    }
}
```

---

## Step 286: stdin Reading Patterns

### รูปแบบที่ 1: อ่านบรรทัดเดียว

```zig
const std = @import("std");

pub fn readLine(reader: anytype, buf: []u8) !?[]u8 {
    return reader.readUntilDelimiterOrEof(buf, '\n');
}

pub fn main() !void {
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    var br = std.io.bufferedReader(stdin.reader());
    const in = br.reader();
    
    try out.writeAll("ป้อนชื่อของคุณ: ");
    try bw.flush();
    
    var buf: [256]u8 = undefined;
    if (try readLine(in, &buf)) |name| {
        try out.print("สวัสดีคุณ {s}!\n", .{name});
    } else {
        try out.writeAll("ไม่ได้รับข้อมูล\n");
    }
    
    try bw.flush();
}
```

### รูปแบบที่ 2: อ่านตัวเลขจาก stdin

```zig
const std = @import("std");

pub fn readInt(reader: anytype, comptime T: type) !T {
    var buf: [64]u8 = undefined;
    const line = try reader.readUntilDelimiterOrEof(&buf, '\n') orelse
        return error.EndOfStream;
    
    const trimmed = std.mem.trim(u8, line, " \t\r");
    return std.fmt.parseInt(T, trimmed, 10);
}

pub fn main() !void {
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    var br = std.io.bufferedReader(stdin.reader());
    const in = br.reader();
    
    try out.writeAll("ป้อนตัวเลขสองตัว:\n");
    
    try out.writeAll("ตัวเลขที่ 1: ");
    try bw.flush();
    const a = readInt(in, i64) catch {
        try out.writeAll("ข้อมูลไม่ถูกต้อง\n");
        try bw.flush();
        return;
    };
    
    try out.writeAll("ตัวเลขที่ 2: ");
    try bw.flush();
    const b = readInt(in, i64) catch {
        try out.writeAll("ข้อมูลไม่ถูกต้อง\n");
        try bw.flush();
        return;
    };
    
    try out.print("ผลรวม: {d} + {d} = {d}\n", .{ a, b, a + b });
    try out.print("ผลต่าง: {d} - {d} = {d}\n", .{ a, b, a - b });
    try out.print("ผลคูณ: {d} × {d} = {d}\n", .{ a, b, a * b });
    
    if (b != 0) {
        try out.print("ผลหาร: {d} ÷ {d} = {d}\n", .{ a, b, @divTrunc(a, b) });
    }
    
    try bw.flush();
}
```

### รูปแบบที่ 3: โปรแกรม Interactive REPL

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    var br = std.io.bufferedReader(stdin.reader());
    const in = br.reader();
    
    try out.writeAll("Calculator REPL (พิมพ์ 'quit' เพื่อออก)\n");
    try out.writeAll("รูปแบบ: <ตัวเลข> <operator> <ตัวเลข>\n");
    try out.writeAll("Operators: +, -, *, /\n\n");
    try bw.flush();
    
    var input_buf = std.ArrayList(u8).init(allocator);
    defer input_buf.deinit();
    
    while (true) {
        try out.writeAll("calc> ");
        try bw.flush();
        
        input_buf.clearRetainingCapacity();
        in.readUntilDelimiterArrayList(&input_buf, '\n', 1024) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };
        
        const input = std.mem.trim(u8, input_buf.items, " \t\r\n");
        
        if (std.mem.eql(u8, input, "quit") or std.mem.eql(u8, input, "exit")) {
            try out.writeAll("ลาก่อน!\n");
            try bw.flush();
            break;
        }
        
        if (input.len == 0) continue;
        
        // แยก tokens
        var tokens = std.mem.splitScalar(u8, input, ' ');
        const a_str = tokens.next() orelse continue;
        const op_str = tokens.next() orelse continue;
        const b_str = tokens.next() orelse continue;
        
        const a = std.fmt.parseFloat(f64, a_str) catch {
            try out.print("ข้อผิดพลาด: '{s}' ไม่ใช่ตัวเลข\n", .{a_str});
            try bw.flush();
            continue;
        };
        
        const b = std.fmt.parseFloat(f64, b_str) catch {
            try out.print("ข้อผิดพลาด: '{s}' ไม่ใช่ตัวเลข\n", .{b_str});
            try bw.flush();
            continue;
        };
        
        const result: f64 = switch (op_str[0]) {
            '+' => a + b,
            '-' => a - b,
            '*' => a * b,
            '/' => blk: {
                if (b == 0) {
                    try out.writeAll("ข้อผิดพลาด: หารด้วยศูนย์ไม่ได้\n");
                    try bw.flush();
                    continue;
                }
                break :blk a / b;
            },
            else => {
                try out.print("ข้อผิดพลาด: operator '{s}' ไม่รู้จัก\n", .{op_str});
                try bw.flush();
                continue;
            },
        };
        
        try out.print("= {d:.4}\n", .{result});
        try bw.flush();
    }
}
```

### รูปแบบที่ 4: อ่านข้อมูล Binary จาก stdin

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // อ่าน header (4 bytes magic + 4 bytes version)
    var magic: [4]u8 = undefined;
    var version: [4]u8 = undefined;
    
    stdin.reader().readNoEof(&magic) catch {
        try out.writeAll("ไม่สามารถอ่าน magic bytes\n");
        try bw.flush();
        return;
    };
    
    stdin.reader().readNoEof(&version) catch {
        try out.writeAll("ไม่สามารถอ่าน version\n");
        try bw.flush();
        return;
    };
    
    try out.print("Magic: {x:0>2}{x:0>2}{x:0>2}{x:0>2}\n", .{
        magic[0], magic[1], magic[2], magic[3],
    });
    
    const ver = std.mem.readInt(u32, &version, .little);
    try out.print("Version: {d}\n", .{ver});
    
    try bw.flush();
}
```

### รูปแบบที่ 5: อ่านหลายบรรทัดด้วย Iterator

```zig
const std = @import("std");

// Iterator สำหรับอ่านบรรทัดจาก reader
fn LineIterator(comptime ReaderType: type) type {
    return struct {
        reader: ReaderType,
        buf: [4096]u8 = undefined,
        done: bool = false,
        
        const Self = @This();
        
        pub fn next(self: *Self) !?[]u8 {
            if (self.done) return null;
            
            const line = self.reader.readUntilDelimiterOrEof(&self.buf, '\n') catch |err| {
                if (err == error.StreamTooLong) {
                    // ทิ้งข้อมูลที่เหลือ
                    self.done = true;
                    return error.StreamTooLong;
                }
                return err;
            };
            
            if (line == null) {
                self.done = true;
            }
            
            return line;
        }
    };
}

pub fn main() !void {
    const data =
        \\รายการสินค้า:
        \\1. แอปเปิ้ล - 50 บาท
        \\2. กล้วย - 30 บาท
        \\3. ส้ม - 40 บาท
        \\รวมทั้งหมด: 120 บาท
    ;
    
    var fbs = std.io.fixedBufferStream(data);
    var iter = LineIterator(@TypeOf(fbs.reader())){ .reader = fbs.reader() };
    
    const stdout = std.io.getStdOut();
    var line_num: usize = 1;
    
    while (try iter.next()) |line| {
        try stdout.print("{d:3}: {s}\n", .{ line_num, line });
        line_num += 1;
    }
}
```

---

## Step 287: stdout Writing Patterns

### รูปแบบการเขียน stdout ต่างๆ

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    defer bw.flush() catch {};
    
    // 1. เขียน string ตรงๆ
    try out.writeAll("Hello, World!\n");
    
    // 2. เขียนด้วย format
    const name = "zig";
    const version = "0.14.0";
    try out.print("ภาษา: {s} เวอร์ชัน: {s}\n", .{ name, version });
    
    // 3. เขียนตัวเลขในรูปแบบต่างๆ
    const num: i32 = 42;
    try out.print("ทศนิยม: {d}\n", .{num});
    try out.print("เลขฐาน 16: {x}\n", .{num});
    try out.print("เลขฐาน 16 ตัวใหญ่: {X}\n", .{num});
    try out.print("เลขฐาน 2: {b}\n", .{num});
    try out.print("เลขฐาน 8: {o}\n", .{num});
    
    // 4. จัดรูปแบบความกว้าง
    try out.print("{d:10}\n", .{num});  // กว้าง 10, ชิดขวา
    try out.print("{d:<10}\n", .{num}); // กว้าง 10, ชิดซ้าย
    try out.print("{d:^10}\n", .{num}); // กว้าง 10, กึ่งกลาง
    try out.print("{d:0>10}\n", .{num}); // เติม 0 ด้านหน้า
    
    // 5. เขียน float
    const pi: f64 = std.math.pi;
    try out.print("Pi = {d}\n", .{pi});
    try out.print("Pi = {d:.4}\n", .{pi});     // ทศนิยม 4 ตำแหน่ง
    try out.print("Pi = {e}\n", .{pi});         // scientific notation
    try out.print("Pi = {e:.2}\n", .{pi});
}
```

### การแสดงตารางข้อมูล

```zig
const std = @import("std");

const Student = struct {
    name: []const u8,
    score: f64,
    grade: u8,
};

pub fn printTable(out: anytype, students: []const Student) !void {
    // header
    try out.writeAll("┌──────────────────┬───────┬───────┐\n");
    try out.writeAll("│ ชื่อ             │ คะแนน │ เกรด  │\n");
    try out.writeAll("├──────────────────┼───────┼───────┤\n");
    
    for (students) |s| {
        try out.print("│ {s:<16} │ {d:>5.1} │   {c}   │\n", .{
            s.name,
            s.score,
            s.grade,
        });
    }
    
    try out.writeAll("└──────────────────┴───────┴───────┘\n");
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    const students = [_]Student{
        .{ .name = "สมชาย", .score = 92.5, .grade = 'A' },
        .{ .name = "สมหญิง", .score = 85.0, .grade = 'B' },
        .{ .name = "สมศักดิ์", .score = 78.5, .grade = 'C' },
        .{ .name = "สมพงษ์", .score = 95.0, .grade = 'A' },
    };
    
    try printTable(out, &students);
    try bw.flush();
}
```

### การแสดง Progress Bar

```zig
const std = @import("std");

pub fn printProgress(out: anytype, current: usize, total: usize, width: usize) !void {
    const progress = if (total > 0) @as(f64, @floatFromInt(current)) / @as(f64, @floatFromInt(total)) else 0.0;
    const filled = @as(usize, @intFromFloat(progress * @as(f64, @floatFromInt(width))));
    
    try out.writeAll("\r[");
    
    var i: usize = 0;
    while (i < width) : (i += 1) {
        if (i < filled) {
            try out.writeByte('#');
        } else {
            try out.writeByte(' ');
        }
    }
    
    try out.print("] {d}%", .{@as(usize, @intFromFloat(progress * 100))});
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const total = 50;
    
    var i: usize = 0;
    while (i <= total) : (i += 1) {
        try printProgress(stdout.writer(), i, total, 40);
        
        // จำลองการทำงาน
        std.time.sleep(50 * std.time.ns_per_ms);
    }
    
    try stdout.writeAll("\nเสร็จแล้ว!\n");
}
```

### การใช้ ANSI Color Codes

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    yellow,
    blue,
    magenta,
    cyan,
    white,
    reset,
    
    pub fn code(self: Color) []const u8 {
        return switch (self) {
            .red => "\x1b[31m",
            .green => "\x1b[32m",
            .yellow => "\x1b[33m",
            .blue => "\x1b[34m",
            .magenta => "\x1b[35m",
            .cyan => "\x1b[36m",
            .white => "\x1b[37m",
            .reset => "\x1b[0m",
        };
    }
};

pub fn printColored(out: anytype, text: []const u8, color: Color) !void {
    if (std.io.getStdOut().isTty()) {
        try out.writeAll(color.code());
        try out.writeAll(text);
        try out.writeAll(Color.reset.code());
    } else {
        try out.writeAll(text);
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    try printColored(out, "สถานะ: ", .white);
    try printColored(out, "OK", .green);
    try out.writeByte('\n');
    
    try printColored(out, "คำเตือน: ", .yellow);
    try out.writeAll("ไฟล์ไม่พบ\n");
    
    try printColored(out, "ข้อผิดพลาด: ", .red);
    try out.writeAll("การเชื่อมต่อล้มเหลว\n");
    
    try printColored(out, "ข้อมูล: ", .cyan);
    try out.writeAll("กำลังประมวลผล...\n");
    
    try bw.flush();
}
```

### การเขียนข้อมูล JSON แบบง่าย

```zig
const std = @import("std");

pub fn writeJsonString(out: anytype, s: []const u8) !void {
    try out.writeByte('"');
    for (s) |c| {
        switch (c) {
            '"' => try out.writeAll("\\\""),
            '\\' => try out.writeAll("\\\\"),
            '\n' => try out.writeAll("\\n"),
            '\r' => try out.writeAll("\\r"),
            '\t' => try out.writeAll("\\t"),
            else => try out.writeByte(c),
        }
    }
    try out.writeByte('"');
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // เขียน JSON object
    try out.writeAll("{\n");
    try out.writeAll("  \"name\": ");
    try writeJsonString(out, "สมชาย \"นามสมมติ\" รักเรียน");
    try out.writeAll(",\n");
    try out.print("  \"age\": {d},\n", .{30});
    try out.writeAll("  \"active\": true,\n");
    try out.writeAll("  \"scores\": [92, 88, 95]\n");
    try out.writeAll("}\n");
    
    try bw.flush();
}
```

---

## Step 288: Pipe ระหว่าง Process

### ทำความเข้าใจ Pipe ใน Zig

Zig สามารถสร้าง pipe สำหรับการสื่อสารระหว่าง process ได้ผ่าน `std.os` และ `std.posix`

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // สร้าง pipe
    const pipe = try std.posix.pipe();
    const read_end = pipe[0];
    const write_end = pipe[1];
    
    // เขียนข้อมูลไปยัง write end
    const message = "สวัสดีจาก pipe!\n";
    _ = try std.posix.write(write_end, message);
    std.posix.close(write_end);
    
    // อ่านข้อมูลจาก read end
    var buf: [256]u8 = undefined;
    const n = try std.posix.read(read_end, &buf);
    std.posix.close(read_end);
    
    try out.print("ได้รับ: {s}", .{buf[0..n]});
    try bw.flush();
}
```

### การใช้ Child Process พร้อม Pipe

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // รัน child process และดัก stdout
    var child = std.process.Child.init(&[_][]const u8{ "echo", "สวัสดีจาก child process" }, allocator);
    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Pipe;
    
    try child.spawn();
    
    // อ่าน stdout ของ child
    var child_stdout = child.stdout.?;
    var buf: [4096]u8 = undefined;
    const n = try child_stdout.readAll(&buf);
    
    const term = try child.wait();
    
    try out.print("Child output: {s}", .{buf[0..n]});
    try out.print("Exit code: {d}\n", .{term.Exited});
    
    try bw.flush();
}
```

### การส่งข้อมูลไปยัง Child Process

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // รัน cat (อ่านจาก stdin และส่งไปยัง stdout)
    var child = std.process.Child.init(&[_][]const u8{"cat"}, allocator);
    child.stdin_behavior = .Pipe;
    child.stdout_behavior = .Pipe;
    
    try child.spawn();
    
    // เขียนข้อมูลไปยัง stdin ของ child
    const input = "บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3\n";
    try child.stdin.?.writeAll(input);
    child.stdin.?.close();
    child.stdin = null;
    
    // อ่าน stdout ของ child
    const output = try child.stdout.?.readToEndAlloc(allocator, 1024 * 1024);
    defer allocator.free(output);
    
    _ = try child.wait();
    
    try out.print("ผลลัพธ์จาก cat:\n{s}", .{output});
    try bw.flush();
}
```

### การอ่าน stdout จาก Shell Command

```zig
const std = @import("std");

pub fn runCommand(allocator: std.mem.Allocator, argv: []const []const u8) ![]u8 {
    var child = std.process.Child.init(argv, allocator);
    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Ignore;
    
    try child.spawn();
    
    const output = try child.stdout.?.readToEndAlloc(allocator, 10 * 1024 * 1024);
    errdefer allocator.free(output);
    
    const term = try child.wait();
    if (term != .Exited or term.Exited != 0) {
        allocator.free(output);
        return error.CommandFailed;
    }
    
    return output;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // รัน ls -la
    const output = runCommand(allocator, &[_][]const u8{ "ls", "-la" }) catch |err| {
        try out.print("ข้อผิดพลาด: {}\n", .{err});
        try bw.flush();
        return;
    };
    defer allocator.free(output);
    
    try out.print("รายการไฟล์:\n{s}\n", .{output});
    try bw.flush();
}
```

### Pipeline ระหว่าง Process

```zig
const std = @import("std");

// จำลอง pipeline: producer | consumer
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // สร้าง pipe
    const pipe_fds = try std.posix.pipe();
    
    // Producer: เขียนข้อมูลไปยัง pipe
    {
        const write_file = std.fs.File{ .handle = pipe_fds[1] };
        const writer = write_file.writer();
        
        for (1..6) |i| {
            try writer.print("ข้อมูล {d}\n", .{i});
        }
        
        write_file.close();
    }
    
    // Consumer: อ่านข้อมูลจาก pipe
    {
        const read_file = std.fs.File{ .handle = pipe_fds[0] };
        const reader = read_file.reader();
        
        var buf: [256]u8 = undefined;
        var total_lines: usize = 0;
        
        while (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
            try out.print("ประมวลผล: {s}\n", .{line});
            total_lines += 1;
        }
        
        read_file.close();
        
        try out.print("ประมวลผลทั้งหมด {d} บรรทัด\n", .{total_lines});
    }
    
    try bw.flush();
    _ = allocator;
}
```

---

## Step 289: std.io.StreamSource

### ทำความเข้าใจ StreamSource

`std.io.StreamSource` เป็น tagged union ที่รวม reader และ writer เข้าด้วยกันสำหรับแหล่งข้อมูลต่างๆ สามารถเป็น:
- `buffer`: อ่านจาก/เขียนไปยัง slice ใน memory
- `const_buffer`: อ่านจาก const slice
- `file`: อ่านจาก/เขียนไปยัง file

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // StreamSource จาก buffer
    var buf: [256]u8 = undefined;
    var source = std.io.StreamSource{ .buffer = std.io.fixedBufferStream(&buf) };
    
    // เขียนไปยัง source
    const writer = source.writer();
    try writer.writeAll("สวัสดี StreamSource\n");
    try writer.print("ค่า: {d}\n", .{42});
    
    // อ่านกลับจาก source
    source.buffer.pos = 0; // reset position
    const reader = source.reader();
    
    var read_buf: [256]u8 = undefined;
    const n = try reader.readAll(&read_buf);
    
    try out.print("อ่านได้: {s}", .{read_buf[0..n]});
    try bw.flush();
}
```

### StreamSource จาก const_buffer

```zig
const std = @import("std");

pub fn processSource(source: *std.io.StreamSource, out: anytype) !void {
    const reader = source.reader();
    var buf: [128]u8 = undefined;
    var line_num: usize = 1;
    
    while (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
        try out.print("{d:3}| {s}\n", .{ line_num, line });
        line_num += 1;
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // ใช้กับ const_buffer (string literal)
    const code =
        \\const std = @import("std");
        \\
        \\pub fn main() !void {
        \\    const out = std.io.getStdOut().writer();
        \\    try out.writeAll("Hello!\n");
        \\}
    ;
    
    var source = std.io.StreamSource{ .const_buffer = std.io.fixedBufferStream(code) };
    
    try out.writeAll("--- Zig Code ---\n");
    try processSource(&source, out);
    
    try bw.flush();
}
```

### การใช้ StreamSource กับ File

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    // สร้างไฟล์ชั่วคราว
    const tmp_path = "/tmp/zig_stream_test.txt";
    
    // เขียนไฟล์
    {
        const file = try std.fs.createFileAbsolute(tmp_path, .{});
        defer file.close();
        
        var source = std.io.StreamSource{ .file = file };
        const writer = source.writer();
        
        try writer.writeAll("บรรทัดที่ 1\n");
        try writer.writeAll("บรรทัดที่ 2\n");
        try writer.writeAll("บรรทัดที่ 3\n");
    }
    
    // อ่านไฟล์
    {
        const file = try std.fs.openFileAbsolute(tmp_path, .{});
        defer file.close();
        
        var source = std.io.StreamSource{ .file = file };
        const reader = source.reader();
        
        const content = try reader.readAllAlloc(allocator, 1024 * 1024);
        defer allocator.free(content);
        
        try out.print("เนื้อหาไฟล์:\n{s}", .{content});
    }
    
    // ลบไฟล์ชั่วคราว
    try std.fs.deleteFileAbsolute(tmp_path);
    
    try bw.flush();
}
```

### StreamSource เป็น Abstraction Layer

```zig
const std = @import("std");

// ฟังก์ชันที่ทำงานกับ StreamSource ใดก็ได้
fn parseSimpleConfig(source: *std.io.StreamSource, allocator: std.mem.Allocator) !std.StringHashMap([]const u8) {
    var map = std.StringHashMap([]const u8).init(allocator);
    errdefer {
        var it = map.iterator();
        while (it.next()) |entry| {
            allocator.free(entry.key_ptr.*);
            allocator.free(entry.value_ptr.*);
        }
        map.deinit();
    }
    
    const reader = source.reader();
    var line_buf: [512]u8 = undefined;
    
    while (try reader.readUntilDelimiterOrEof(&line_buf, '\n')) |line| {
        const trimmed = std.mem.trim(u8, line, " \t\r");
        
        // ข้าม comment และบรรทัดว่าง
        if (trimmed.len == 0 or trimmed[0] == '#') continue;
        
        // แยก key = value
        const eq_pos = std.mem.indexOfScalar(u8, trimmed, '=') orelse continue;
        
        const key = std.mem.trim(u8, trimmed[0..eq_pos], " \t");
        const value = std.mem.trim(u8, trimmed[eq_pos + 1 ..], " \t");
        
        const key_owned = try allocator.dupe(u8, key);
        const value_owned = try allocator.dupe(u8, value);
        
        try map.put(key_owned, value_owned);
    }
    
    return map;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const config_data =
        \\# Config file
        \\host = localhost
        \\port = 8080
        \\debug = true
        \\max_connections = 100
    ;
    
    var source = std.io.StreamSource{ .const_buffer = std.io.fixedBufferStream(config_data) };
    
    var config = try parseSimpleConfig(&source, allocator);
    defer {
        var it = config.iterator();
        while (it.next()) |entry| {
            allocator.free(entry.key_ptr.*);
            allocator.free(entry.value_ptr.*);
        }
        config.deinit();
    }
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    try out.writeAll("Config values:\n");
    var it = config.iterator();
    while (it.next()) |entry| {
        try out.print("  {s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
    
    try bw.flush();
}
```

---

## Step 290: Custom Reader/Writer Implementation

### การสร้าง Custom Writer

```zig
const std = @import("std");

// Custom writer ที่แปลง lowercase เป็น uppercase
const UppercaseWriter = struct {
    inner: std.fs.File.Writer,
    
    const Self = @This();
    const WriteError = std.fs.File.WriteError;
    
    pub fn write(self: Self, bytes: []const u8) WriteError!usize {
        // สร้าง buffer ชั่วคราว
        var buf: [4096]u8 = undefined;
        const len = @min(bytes.len, buf.len);
        
        for (bytes[0..len], 0..) |c, i| {
            buf[i] = std.ascii.toUpper(c);
        }
        
        return self.inner.write(buf[0..len]);
    }
    
    pub fn writer(self: Self) std.io.Writer(Self, WriteError, write) {
        return .{ .context = self };
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    const upper_writer = UppercaseWriter{ .inner = stdout.writer() };
    const uw = upper_writer.writer();
    
    try uw.writeAll("hello world, this is zig!\n");
    try uw.print("version {s}\n", .{"0.14.0"});
    
    _ = out;
    try bw.flush();
}
```

### Custom Writer ที่นับ Bytes และกรองข้อมูล

```zig
const std = @import("std");

// Writer ที่ตัด whitespace ซ้ำออก
const CompactWriter = struct {
    inner: std.fs.File.Writer,
    last_was_space: bool = false,
    bytes_written: usize = 0,
    
    const Self = @This();
    const WriteError = std.fs.File.WriteError;
    
    pub fn write(self: *Self, bytes: []const u8) WriteError!usize {
        for (bytes) |c| {
            const is_space = c == ' ' or c == '\t';
            
            if (is_space and self.last_was_space) {
                continue; // ข้ามถ้าเจอ space ต่อเนื่อง
            }
            
            _ = try self.inner.write(&[_]u8{c});
            self.bytes_written += 1;
            self.last_was_space = is_space;
        }
        
        return bytes.len;
    }
    
    pub fn writer(self: *Self) std.io.Writer(*Self, WriteError, write) {
        return .{ .context = self };
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    var compact = CompactWriter{ .inner = stdout.writer() };
    const cw = compact.writer();
    
    const text = "นี่คือ   ข้อความ    ที่มี    ช่องว่าง    เยอะมาก\n";
    try cw.writeAll(text);
    
    try stdout.print("Input bytes: {d}\n", .{text.len});
    try stdout.print("Output bytes: {d}\n", .{compact.bytes_written});
}
```

### Custom Reader Implementation

```zig
const std = @import("std");

// Reader ที่แปลง CR LF เป็น LF
const UnixLineReader = struct {
    inner: std.io.FixedBufferStream([]const u8).Reader,
    
    const Self = @This();
    pub const Error = std.io.FixedBufferStream([]const u8).ReadError;
    
    pub fn read(self: *Self, dest: []u8) Error!usize {
        var total: usize = 0;
        var i: usize = 0;
        
        // อ่านทีละ byte และกรอง CR
        while (i < dest.len) {
            var byte: [1]u8 = undefined;
            const n = try self.inner.read(&byte);
            if (n == 0) break;
            
            if (byte[0] == '\r') {
                // ข้าม CR (อาจเป็น CRLF)
                continue;
            }
            
            dest[i] = byte[0];
            i += 1;
            total += 1;
        }
        
        return total;
    }
    
    pub fn reader(self: *Self) std.io.Reader(*Self, Error, read) {
        return .{ .context = self };
    }
};

pub fn main() !void {
    const windows_text = "บรรทัด 1\r\nบรรทัด 2\r\nบรรทัด 3\r\n";
    var fbs = std.io.fixedBufferStream(windows_text);
    
    var unix_reader = UnixLineReader{ .inner = fbs.reader() };
    const reader = unix_reader.reader();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    var buf: [256]u8 = undefined;
    var line_num: usize = 1;
    
    while (try reader.readUntilDelimiterOrEof(&buf, '\n')) |line| {
        try out.print("บรรทัด {d}: '{s}'\n", .{ line_num, line });
        line_num += 1;
    }
    
    try bw.flush();
}
```

### Custom Writer สำหรับ Logging

```zig
const std = @import("std");

const LogLevel = enum {
    debug,
    info,
    warn,
    err,
    
    pub fn prefix(self: LogLevel) []const u8 {
        return switch (self) {
            .debug => "[DEBUG]",
            .info  => "[INFO] ",
            .warn  => "[WARN] ",
            .err   => "[ERROR]",
        };
    }
};

const Logger = struct {
    inner: std.fs.File.Writer,
    level: LogLevel,
    
    const Self = @This();
    
    pub fn log(self: Self, level: LogLevel, comptime fmt: []const u8, args: anytype) !void {
        if (@intFromEnum(level) < @intFromEnum(self.level)) return;
        
        // ดึงเวลา
        const epoch_secs = std.time.timestamp();
        
        try self.inner.print("{d} {s} ", .{ epoch_secs, level.prefix() });
        try self.inner.print(fmt, args);
        try self.inner.writeByte('\n');
    }
    
    pub fn debug(self: Self, comptime fmt: []const u8, args: anytype) !void {
        try self.log(.debug, fmt, args);
    }
    
    pub fn info(self: Self, comptime fmt: []const u8, args: anytype) !void {
        try self.log(.info, fmt, args);
    }
    
    pub fn warn(self: Self, comptime fmt: []const u8, args: anytype) !void {
        try self.log(.warn, fmt, args);
    }
    
    pub fn err(self: Self, comptime fmt: []const u8, args: anytype) !void {
        try self.log(.err, fmt, args);
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    var logger = Logger{
        .inner = stdout.writer(),
        .level = .info,
    };
    
    try logger.debug("Debug message (จะไม่แสดง)", .{});
    try logger.info("เริ่มต้นโปรแกรม", .{});
    try logger.info("กำลังโหลดข้อมูล จำนวน {d} รายการ", .{100});
    try logger.warn("หน่วยความจำใกล้เต็ม: {d}%", .{85});
    try logger.err("ไม่สามารถบันทึกข้อมูลได้", .{});
}
```

### Custom Reader ที่นับ Bytes

```zig
const std = @import("std");

// Generic counting reader ที่ใช้ได้กับ reader ใดก็ได้
fn CountingReader(comptime ReaderType: type) type {
    return struct {
        inner: ReaderType,
        bytes_read: u64 = 0,
        
        const Self = @This();
        pub const Error = ReaderType.Error;
        
        pub fn read(self: *Self, dest: []u8) Error!usize {
            const n = try self.inner.read(dest);
            self.bytes_read += n;
            return n;
        }
        
        pub fn reader(self: *Self) std.io.Reader(*Self, Error, read) {
            return .{ .context = self };
        }
    };
}

pub fn main() !void {
    const data = "นับจำนวน bytes ที่อ่านจาก stream นี้";
    var fbs = std.io.fixedBufferStream(data);
    
    const InnerReader = @TypeOf(fbs.reader());
    var counting = CountingReader(InnerReader){ .inner = fbs.reader() };
    const reader = counting.reader();
    
    var buf: [512]u8 = undefined;
    _ = try reader.readAll(&buf);
    
    const stdout = std.io.getStdOut();
    try stdout.print("อ่านได้ {d} bytes\n", .{counting.bytes_read});
    try stdout.print("ข้อมูล: {s}\n", .{data});
}
```

### Custom Reader/Writer สำหรับ Encryption (XOR)

```zig
const std = @import("std");

// Simple XOR cipher writer
fn XorWriter(comptime WriterType: type) type {
    return struct {
        inner: WriterType,
        key: u8,
        
        const Self = @This();
        pub const Error = WriterType.Error;
        
        pub fn write(self: *Self, bytes: []const u8) Error!usize {
            for (bytes) |byte| {
                const encrypted = byte ^ self.key;
                _ = try self.inner.write(&[_]u8{encrypted});
            }
            return bytes.len;
        }
        
        pub fn writer(self: *Self) std.io.Writer(*Self, Error, write) {
            return .{ .context = self };
        }
    };
}

// Simple XOR cipher reader
fn XorReader(comptime ReaderType: type) type {
    return struct {
        inner: ReaderType,
        key: u8,
        
        const Self = @This();
        pub const Error = ReaderType.Error;
        
        pub fn read(self: *Self, dest: []u8) Error!usize {
            const n = try self.inner.read(dest);
            for (dest[0..n]) |*byte| {
                byte.* = byte.* ^ self.key;
            }
            return n;
        }
        
        pub fn reader(self: *Self) std.io.Reader(*Self, Error, read) {
            return .{ .context = self };
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    
    const key: u8 = 0x42;
    const message = "Hello, Secret Message!";
    
    // Encrypt
    var encrypted_buf = std.ArrayList(u8).init(allocator);
    defer encrypted_buf.deinit();
    
    const ListWriter = @TypeOf(encrypted_buf.writer());
    var xor_writer = XorWriter(ListWriter){
        .inner = encrypted_buf.writer(),
        .key = key,
    };
    const enc_writer = xor_writer.writer();
    
    try enc_writer.writeAll(message);
    
    try out.print("Original: {s}\n", .{message});
    try out.print("Encrypted (hex): ", .{});
    for (encrypted_buf.items) |b| {
        try out.print("{x:0>2} ", .{b});
    }
    try out.writeByte('\n');
    
    // Decrypt
    var enc_stream = std.io.fixedBufferStream(encrypted_buf.items);
    const FbsReader = @TypeOf(enc_stream.reader());
    var xor_reader = XorReader(FbsReader){
        .inner = enc_stream.reader(),
        .key = key,
    };
    const dec_reader = xor_reader.reader();
    
    var decrypted_buf: [256]u8 = undefined;
    const n = try dec_reader.readAll(&decrypted_buf);
    
    try out.print("Decrypted: {s}\n", .{decrypted_buf[0..n]});
    
    try bw.flush();
}
```

### Custom Tee Writer (เขียนไปหลายที่พร้อมกัน)

```zig
const std = @import("std");

// Tee writer - เขียนไปสอง writer พร้อมกัน
fn TeeWriter(comptime Writer1Type: type, comptime Writer2Type: type) type {
    return struct {
        writer1: Writer1Type,
        writer2: Writer2Type,
        
        const Self = @This();
        pub const Error = Writer1Type.Error || Writer2Type.Error;
        
        pub fn write(self: *Self, bytes: []const u8) Error!usize {
            // เขียนไปยัง writer ทั้งสอง
            const n1 = try self.writer1.write(bytes);
            _ = try self.writer2.write(bytes[0..n1]);
            return n1;
        }
        
        pub fn writer(self: *Self) std.io.Writer(*Self, Error, write) {
            return .{ .context = self };
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    
    // สร้าง buffer สำหรับเก็บ log
    var log_buf = std.ArrayList(u8).init(allocator);
    defer log_buf.deinit();
    
    // สร้าง tee writer
    const StdoutWriter = std.fs.File.Writer;
    const ListWriter = @TypeOf(log_buf.writer());
    
    var tee = TeeWriter(StdoutWriter, ListWriter){
        .writer1 = stdout.writer(),
        .writer2 = log_buf.writer(),
    };
    const tee_writer = tee.writer();
    
    // เขียนไปทั้ง stdout และ log buffer พร้อมกัน
    try tee_writer.writeAll("=== โปรแกรมเริ่มต้น ===\n");
    try tee_writer.print("เวลา: {d}\n", .{std.time.timestamp()});
    try tee_writer.writeAll("กำลังประมวลผล...\n");
    try tee_writer.writeAll("=== โปรแกรมสิ้นสุด ===\n");
    
    // แสดง log ที่เก็บไว้
    try stdout.print("\n--- Log ({d} bytes) ---\n{s}", .{
        log_buf.items.len,
        log_buf.items,
    });
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Word Count

สร้างโปรแกรมที่อ่านข้อความจาก stdin และนับจำนวน:
- ตัวอักษร (characters)
- คำ (words)
- บรรทัด (lines)

```zig
const std = @import("std");

// TODO: สร้าง WordCounter struct พร้อม method:
// - countFromReader(reader: anytype) !Stats
// - Stats ประกอบด้วย: chars, words, lines

const Stats = struct {
    chars: usize,
    words: usize,
    lines: usize,
};

pub fn countFromReader(reader: anytype, allocator: std.mem.Allocator) !Stats {
    var stats = Stats{ .chars = 0, .words = 0, .lines = 0 };
    var in_word = false;
    
    var line_buf = std.ArrayList(u8).init(allocator);
    defer line_buf.deinit();
    
    while (true) {
        line_buf.clearRetainingCapacity();
        
        reader.readUntilDelimiterArrayList(&line_buf, '\n', std.math.maxInt(usize)) catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };
        
        stats.lines += 1;
        stats.chars += line_buf.items.len + 1; // +1 for newline
        
        for (line_buf.items) |c| {
            if (c == ' ' or c == '\t') {
                in_word = false;
            } else {
                if (!in_word) {
                    stats.words += 1;
                    in_word = true;
                }
            }
        }
        in_word = false;
    }
    
    return stats;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const text =
        \\นี่คือ บทความ ทดสอบ
        \\มีหลายบรรทัด
        \\และหลายคำ ในแต่ละบรรทัด
    ;
    
    var fbs = std.io.fixedBufferStream(text);
    const stats = try countFromReader(fbs.reader(), allocator);
    
    const stdout = std.io.getStdOut();
    try stdout.print("ตัวอักษร: {d}\n", .{stats.chars});
    try stdout.print("คำ: {d}\n", .{stats.words});
    try stdout.print("บรรทัด: {d}\n", .{stats.lines});
}
```

### แบบฝึกหัดที่ 2: File Copy with Progress

สร้างโปรแกรมที่คัดลอกไฟล์และแสดง progress:

```zig
const std = @import("std");

pub fn copyWithProgress(
    reader: anytype,
    writer: anytype,
    total_size: ?u64,
    out: anytype,
) !u64 {
    var buf: [65536]u8 = undefined; // 64KB buffer
    var copied: u64 = 0;
    
    while (true) {
        const n = try reader.read(&buf);
        if (n == 0) break;
        
        try writer.writeAll(buf[0..n]);
        copied += n;
        
        if (total_size) |total| {
            const percent = (copied * 100) / total;
            try out.print("\rคัดลอก: {d}% ({d}/{d} bytes)", .{ percent, copied, total });
        } else {
            try out.print("\rคัดลอก: {d} bytes", .{copied});
        }
    }
    
    try out.writeByte('\n');
    return copied;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut();
    
    // จำลองการคัดลอก
    const source_data = "a" ** 1000;
    var source = std.io.fixedBufferStream(source_data);
    
    var dest = std.ArrayList(u8).init(allocator);
    defer dest.deinit();
    
    const copied = try copyWithProgress(
        source.reader(),
        dest.writer(),
        @as(u64, source_data.len),
        stdout.writer(),
    );
    
    try stdout.print("คัดลอกสำเร็จ: {d} bytes\n", .{copied});
}
```

### แบบฝึกหัดที่ 3: Interactive Menu

```zig
const std = @import("std");

const MenuItem = struct {
    key: u8,
    label: []const u8,
};

pub fn showMenu(out: anytype, title: []const u8, items: []const MenuItem) !void {
    try out.print("\n=== {s} ===\n", .{title});
    for (items) |item| {
        try out.print("{c}. {s}\n", .{ item.key, item.label });
    }
    try out.writeAll("เลือก: ");
}

pub fn getChoice(reader: anytype, valid_keys: []const u8) !u8 {
    var buf: [10]u8 = undefined;
    while (true) {
        const input = try reader.readUntilDelimiterOrEof(&buf, '\n') orelse
            return error.EndOfStream;
        
        if (input.len == 0) continue;
        
        const choice = input[0];
        for (valid_keys) |k| {
            if (choice == k) return choice;
        }
    }
}

pub fn main() !void {
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    
    var bw = std.io.bufferedWriter(stdout.writer());
    const out = bw.writer();
    var br = std.io.bufferedReader(stdin.reader());
    const in = br.reader();
    
    const menu_items = [_]MenuItem{
        .{ .key = '1', .label = "เพิ่มรายการ" },
        .{ .key = '2', .label = "ดูรายการทั้งหมด" },
        .{ .key = '3', .label = "ลบรายการ" },
        .{ .key = 'q', .label = "ออกจากโปรแกรม" },
    };
    
    while (true) {
        try showMenu(out, "เมนูหลัก", &menu_items);
        try bw.flush();
        
        const choice = getChoice(in, "123q") catch break;
        
        switch (choice) {
            '1' => {
                try out.writeAll("เพิ่มรายการ...\n");
            },
            '2' => {
                try out.writeAll("แสดงรายการ...\n");
            },
            '3' => {
                try out.writeAll("ลบรายการ...\n");
            },
            'q' => {
                try out.writeAll("ออกจากโปรแกรม\n");
                try bw.flush();
                break;
            },
            else => {},
        }
        
        try bw.flush();
    }
}
```

### แบบฝึกหัดที่ 4: Custom Buffered Writer

ลองสร้าง buffered writer ของตัวเองเพื่อทำความเข้าใจการทำงาน:

```zig
const std = @import("std");

// Custom buffered writer
fn MyBufferedWriter(comptime buffer_size: usize, comptime WriterType: type) type {
    return struct {
        inner: WriterType,
        buf: [buffer_size]u8 = undefined,
        end: usize = 0,
        
        const Self = @This();
        pub const Error = WriterType.Error;
        
        pub fn flush(self: *Self) Error!void {
            if (self.end > 0) {
                try self.inner.writeAll(self.buf[0..self.end]);
                self.end = 0;
            }
        }
        
        pub fn write(self: *Self, bytes: []const u8) Error!usize {
            var remaining = bytes;
            
            while (remaining.len > 0) {
                const space = buffer_size - self.end;
                
                if (space == 0) {
                    try self.flush();
                    continue;
                }
                
                const to_copy = @min(space, remaining.len);
                @memcpy(self.buf[self.end..][0..to_copy], remaining[0..to_copy]);
                self.end += to_copy;
                remaining = remaining[to_copy..];
            }
            
            return bytes.len;
        }
        
        pub fn writer(self: *Self) std.io.Writer(*Self, Error, write) {
            return .{ .context = self };
        }
    };
}

pub fn main() !void {
    const stdout = std.io.getStdOut();
    
    var my_bw = MyBufferedWriter(16, std.fs.File.Writer){
        .inner = stdout.writer(),
    };
    const writer = my_bw.writer();
    
    // เขียนข้อมูล
    try writer.writeAll("Hello, ");
    try writer.writeAll("World! ");
    try writer.writeAll("This is my custom buffered writer.\n");
    
    // Flush ข้อมูลที่เหลือ
    try my_bw.flush();
}
```

---

## สรุป Part 29: std.io

### สิ่งที่เรียนรู้ในบทนี้

| หัวข้อ | สิ่งสำคัญ |
|--------|-----------|
| **Step 281** | `getStdIn()`, `getStdOut()`, `getStdErr()` คืนค่า `std.fs.File` |
| **Step 282** | Writer interface ใช้ `write([]const u8) !usize` |
| **Step 283** | Reader interface ใช้ `read([]u8) !usize` |
| **Step 284** | `bufferedWriter`/`bufferedReader` ต้อง `flush()` เสมอ |
| **Step 285** | `readUntilDelimiterOrEof` คืน `?[]u8`, null = EOF |
| **Step 286** | ใช้ buffered reader/writer กับ stdin/stdout เสมอ |
| **Step 287** | `print()`, `writeAll()`, format specifiers |
| **Step 288** | `std.posix.pipe()` และ `std.process.Child` |
| **Step 289** | `StreamSource` เป็น abstraction สำหรับหลาย source |
| **Step 290** | Custom reader/writer ด้วย comptime generic |

### Best Practices

1. **ใช้ buffered I/O เสมอ** สำหรับ stdout/stdin ในโปรแกรมที่มีการอ่าน/เขียนจำนวนมาก

2. **อย่าลืม flush()** เมื่อใช้ buffered writer โดยเฉพาะก่อนโปรแกรมจบ

3. **ใช้ `anytype` parameter** สำหรับฟังก์ชันที่รับ reader/writer ทำให้ยืดหยุ่นและทดสอบง่าย

4. **จัดการ EOF อย่างถูกต้อง** `readUntilDelimiterOrEof` คืน `null` เมื่อถึง EOF ไม่ใช่ error

5. **ระวัง StreamTooLong** เมื่อใช้ `readUntilDelimiterOrEof` กับ buffer ขนาดจำกัด

6. **ใช้ `fixedBufferStream`** สำหรับการทดสอบ reader/writer โดยไม่ต้องใช้ไฟล์จริง

### Pattern สำคัญที่ควรจำ

```zig
// Pattern 1: Standard I/O setup
const stdin = std.io.getStdIn();
const stdout = std.io.getStdOut();
var bw = std.io.bufferedWriter(stdout.writer());
var br = std.io.bufferedReader(stdin.reader());
defer bw.flush() catch {};
const out = bw.writer();
const in = br.reader();

// Pattern 2: อ่านบรรทัดจาก stdin
var buf: [1024]u8 = undefined;
while (try in.readUntilDelimiterOrEof(&buf, '\n')) |line| {
    // ประมวลผล line
}

// Pattern 3: Generic I/O function
fn processData(reader: anytype, writer: anytype) !void {
    // ทำงานกับ reader/writer ใดก็ได้
}

// Pattern 4: Buffer-based I/O
var mem_buf: [4096]u8 = undefined;
var fbs = std.io.fixedBufferStream(&mem_buf);
const writer = fbs.writer();
const reader = fbs.reader();
```

---

## บทถัดไป

ใน Part 30 เราจะเรียนรู้เกี่ยวกับ **std.fs - File System Operations** ซึ่งจะครอบคลุม:
- การสร้าง อ่าน เขียน ลบไฟล์
- การจัดการ directories
- การทำงานกับ paths
- File metadata และ permissions
- Directory walking

---

*จบ Part 29: std.io ใน Zig Standard Library*
