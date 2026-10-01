# ภาค 8: ฟังก์ชันใน Zig (Steps 71-80)

ฟังก์ชันเป็นหัวใจสำคัญของการเขียนโปรแกรม ใน Zig ฟังก์ชันมีคุณสมบัติที่น่าสนใจหลายอย่าง ทั้งความเรียบง่ายในการประกาศ ความสามารถในการคืนค่าหลายค่า การจัดการข้อผิดพลาด และฟีเจอร์พิเศษอย่าง comptime functions ที่ทำให้ Zig แตกต่างจากภาษาอื่น

---

## Step 71: การประกาศและเรียกใช้ฟังก์ชัน

### แนวคิดพื้นฐาน

ใน Zig การประกาศฟังก์ชันใช้คีย์เวิร์ด `fn` ตามด้วยชื่อฟังก์ชัน รายการพารามิเตอร์ในวงเล็บ และชนิดข้อมูลที่คืนค่า

รูปแบบทั่วไป:
```
fn ชื่อฟังก์ชัน(พารามิเตอร์: ชนิดข้อมูล) ชนิดที่คืนค่า {
    // เนื้อหาฟังก์ชัน
    return ค่าที่คืน;
}
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

// ฟังก์ชันที่ไม่รับพารามิเตอร์และไม่คืนค่า
fn sayHello() void {
    std.debug.print("สวัสดีจากฟังก์ชัน!\n", .{});
}

// ฟังก์ชันที่รับพารามิเตอร์
fn greet(name: []const u8) void {
    std.debug.print("สวัสดี, {s}!\n", .{name});
}

// ฟังก์ชันที่คืนค่า
fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn main() void {
    // เรียกใช้ฟังก์ชัน
    sayHello();
    greet("สมชาย");
    
    const result = add(10, 20);
    std.debug.print("10 + 20 = {d}\n", .{result});
}
```

ผลลัพธ์:
```
สวัสดีจากฟังก์ชัน!
สวัสดี, สมชาย!
10 + 20 = 30
```

### Access Modifiers

ใน Zig ฟังก์ชันสามารถมี access modifier ได้:

- **ไม่มี modifier**: ฟังก์ชัน private ใช้ได้เฉพาะในไฟล์เดียวกัน
- **`pub`**: ฟังก์ชัน public ที่สามารถเรียกใช้จากไฟล์อื่นได้

```zig
const std = @import("std");

// ฟังก์ชัน private (ใช้ได้เฉพาะในไฟล์นี้)
fn privateHelper(x: i32) i32 {
    return x * 2;
}

// ฟังก์ชัน public (เรียกใช้จากภายนอกได้)
pub fn publicFunction(x: i32) i32 {
    return privateHelper(x) + 1;
}

pub fn main() void {
    const result = publicFunction(5);
    std.debug.print("ผลลัพธ์: {d}\n", .{result}); // 11
}
```

### ลำดับการประกาศฟังก์ชัน

ใน Zig ไม่จำเป็นต้องประกาศฟังก์ชันก่อนใช้งาน คอมไพเลอร์จะค้นหาฟังก์ชันทั้งหมดในไฟล์ก่อน:

```zig
const std = @import("std");

pub fn main() void {
    // เรียกใช้ฟังก์ชันที่ประกาศหลังจากนี้ได้เลย
    const result = multiply(3, 4);
    std.debug.print("3 × 4 = {d}\n", .{result});
}

// ประกาศหลังจาก main ก็ได้
fn multiply(a: i32, b: i32) i32 {
    return a * b;
}
```

### ฟังก์ชันที่ไม่คืนค่า (void)

```zig
const std = @import("std");

fn printSeparator() void {
    std.debug.print("━━━━━━━━━━━━━━━━━━━━\n", .{});
}

fn printHeader(title: []const u8) void {
    printSeparator();
    std.debug.print("  {s}\n", .{title});
    printSeparator();
}

pub fn main() void {
    printHeader("รายการสินค้า");
    std.debug.print("1. แอปเปิ้ล\n", .{});
    std.debug.print("2. กล้วย\n", .{});
    std.debug.print("3. ส้ม\n", .{});
    printSeparator();
}
```

### ฟังก์ชันกับ noreturn

บางฟังก์ชันไม่เคยคืนค่า เช่น ฟังก์ชันที่ออกจากโปรแกรม:

```zig
const std = @import("std");

fn fatalError(message: []const u8) noreturn {
    std.debug.print("ข้อผิดพลาดร้ายแรง: {s}\n", .{message});
    std.process.exit(1);
}

pub fn main() void {
    const value: i32 = -1;
    if (value < 0) {
        fatalError("ค่าต้องไม่เป็นลบ");
    }
    std.debug.print("ค่า: {d}\n", .{value});
}
```

---

## Step 72: Parameters และ Return Values

### การส่งพารามิเตอร์หลายตัว

```zig
const std = @import("std");

// ฟังก์ชันคำนวณพื้นที่สามเหลี่ยม
fn triangleArea(base: f64, height: f64) f64 {
    return 0.5 * base * height;
}

// ฟังก์ชันคำนวณระยะทาง
fn distance(x1: f64, y1: f64, x2: f64, y2: f64) f64 {
    const dx = x2 - x1;
    const dy = y2 - y1;
    return @sqrt(dx * dx + dy * dy);
}

pub fn main() void {
    const area = triangleArea(10.0, 5.0);
    std.debug.print("พื้นที่สามเหลี่ยม: {d:.2}\n", .{area});
    
    const dist = distance(0.0, 0.0, 3.0, 4.0);
    std.debug.print("ระยะทาง: {d:.2}\n", .{dist});
}
```

### ชนิดข้อมูลของพารามิเตอร์

```zig
const std = @import("std");

// พารามิเตอร์ชนิด integer ต่างๆ
fn sumIntegers(a: u8, b: u16, c: u32) u64 {
    return @as(u64, a) + @as(u64, b) + @as(u64, c);
}

// พารามิเตอร์ชนิด float
fn average(values: []const f64) f64 {
    if (values.len == 0) return 0.0;
    var sum: f64 = 0.0;
    for (values) |v| {
        sum += v;
    }
    return sum / @as(f64, @floatFromInt(values.len));
}

// พารามิเตอร์ชนิด boolean
fn printConditional(condition: bool, message: []const u8) void {
    if (condition) {
        std.debug.print("[✓] {s}\n", .{message});
    } else {
        std.debug.print("[✗] {s}\n", .{message});
    }
}

pub fn main() void {
    const total = sumIntegers(10, 200, 3000);
    std.debug.print("ผลรวม: {d}\n", .{total});
    
    const scores = [_]f64{ 85.5, 90.0, 78.5, 92.0, 88.0 };
    const avg = average(&scores);
    std.debug.print("ค่าเฉลี่ย: {d:.2}\n", .{avg});
    
    printConditional(true, "ผ่านการทดสอบ");
    printConditional(false, "ไม่ผ่านการทดสอบ");
}
```

### การส่งผ่านด้วย value และ pointer

ใน Zig ทุกอย่างส่งผ่านด้วย value ตามค่าเริ่มต้น แต่สามารถส่งผ่าน pointer เพื่อแก้ไขได้:

```zig
const std = @import("std");

// ส่งผ่านค่า (copy) - ไม่แก้ไขต้นฉบับ
fn doubleValue(x: i32) i32 {
    return x * 2;
}

// ส่งผ่าน pointer - แก้ไขต้นฉบับได้
fn doubleInPlace(x: *i32) void {
    x.* = x.* * 2;
}

// ส่งผ่าน slice (อ้างอิงไปยัง array)
fn sumSlice(numbers: []const i32) i32 {
    var sum: i32 = 0;
    for (numbers) |n| {
        sum += n;
    }
    return sum;
}

pub fn main() void {
    var value: i32 = 10;
    
    // การส่งแบบ copy
    const doubled = doubleValue(value);
    std.debug.print("ค่าเดิม: {d}, ค่าสองเท่า: {d}\n", .{ value, doubled });
    
    // การส่งแบบ pointer
    doubleInPlace(&value);
    std.debug.print("หลังแก้ไข: {d}\n", .{value}); // 20
    
    // การส่ง slice
    const numbers = [_]i32{ 1, 2, 3, 4, 5 };
    const total = sumSlice(&numbers);
    std.debug.print("ผลรวม: {d}\n", .{total});
}
```

### Return Values

```zig
const std = @import("std");

// คืนค่าชนิดต่างๆ
fn getPI() f64 {
    return 3.14159265358979;
}

fn getGreeting() []const u8 {
    return "สวัสดี!";
}

fn isEven(n: i32) bool {
    return @mod(n, 2) == 0;
}

// คืนค่าจากเงื่อนไข
fn absoluteValue(n: i32) i32 {
    if (n < 0) return -n;
    return n;
}

// คืนค่าจาก switch
fn dayName(day: u8) []const u8 {
    return switch (day) {
        1 => "จันทร์",
        2 => "อังคาร",
        3 => "พุธ",
        4 => "พฤหัสบดี",
        5 => "ศุกร์",
        6 => "เสาร์",
        7 => "อาทิตย์",
        else => "ไม่ทราบ",
    };
}

pub fn main() void {
    std.debug.print("PI = {d:.5}\n", .{getPI()});
    std.debug.print("{s}\n", .{getGreeting()});
    std.debug.print("4 เป็นเลขคู่: {}\n", .{isEven(4)});
    std.debug.print("|−15| = {d}\n", .{absoluteValue(-15)});
    std.debug.print("วันที่ 3 คือ: {s}\n", .{dayName(3)});
}
```

### ตัวอย่างจริง: ฟังก์ชันคำนวณภาษี

```zig
const std = @import("std");

fn calculateTax(income: f64) f64 {
    // อัตราภาษีเงินได้บุคคลธรรมดา (ตัวอย่าง)
    if (income <= 150_000.0) return 0.0;
    if (income <= 300_000.0) return (income - 150_000.0) * 0.05;
    if (income <= 500_000.0) return 7_500.0 + (income - 300_000.0) * 0.10;
    if (income <= 750_000.0) return 27_500.0 + (income - 500_000.0) * 0.15;
    if (income <= 1_000_000.0) return 65_000.0 + (income - 750_000.0) * 0.20;
    return 115_000.0 + (income - 1_000_000.0) * 0.25;
}

fn formatCurrency(amount: f64) void {
    std.debug.print("฿{d:,.2}\n", .{amount});
}

pub fn main() void {
    const incomes = [_]f64{ 100_000.0, 250_000.0, 450_000.0, 800_000.0 };
    
    std.debug.print("รายได้\t\t\tภาษีที่ต้องจ่าย\n", .{});
    std.debug.print("────────────────────────────────\n", .{});
    
    for (incomes) |income| {
        const tax = calculateTax(income);
        std.debug.print("฿{d:>12,.0}\t\t", .{income});
        formatCurrency(tax);
    }
}
```

---

## Step 73: Multiple Return Values ด้วย Struct

### ปัญหาเมื่อต้องการคืนค่าหลายค่า

ใน C หากต้องการคืนค่าหลายค่า ต้องใช้ pointer เป็น output parameter ซึ่งอ่านยาก ใน Zig เราใช้ struct เป็นวิธีที่ชัดเจนและปลอดภัยกว่า

### แนวทางที่ 1: Anonymous Struct

```zig
const std = @import("std");

// คืนค่าหลายค่าด้วย anonymous struct
fn divmod(a: i32, b: i32) struct { quotient: i32, remainder: i32 } {
    return .{
        .quotient = @divTrunc(a, b),
        .remainder = @mod(a, b),
    };
}

pub fn main() void {
    const result = divmod(17, 5);
    std.debug.print("17 ÷ 5 = {d} เศษ {d}\n", .{
        result.quotient,
        result.remainder,
    });
}
```

### แนวทางที่ 2: Named Struct

```zig
const std = @import("std");

// กำหนด struct สำหรับผลลัพธ์
const MinMax = struct {
    min: f64,
    max: f64,
    range: f64,
};

fn findMinMax(values: []const f64) MinMax {
    if (values.len == 0) {
        return .{ .min = 0.0, .max = 0.0, .range = 0.0 };
    }
    
    var min = values[0];
    var max = values[0];
    
    for (values[1..]) |v| {
        if (v < min) min = v;
        if (v > max) max = v;
    }
    
    return .{
        .min = min,
        .max = max,
        .range = max - min,
    };
}

pub fn main() void {
    const data = [_]f64{ 3.5, 7.2, 1.8, 9.4, 5.1, 2.3 };
    const stats = findMinMax(&data);
    
    std.debug.print("ค่าต่ำสุด: {d:.1}\n", .{stats.min});
    std.debug.print("ค่าสูงสุด: {d:.1}\n", .{stats.max});
    std.debug.print("ช่วง: {d:.1}\n", .{stats.range});
}
```

### ตัวอย่างจริง: Parser ผลลัพธ์

```zig
const std = @import("std");

const ParseResult = struct {
    value: i64,
    bytes_consumed: usize,
    success: bool,
};

fn parseInteger(input: []const u8) ParseResult {
    var i: usize = 0;
    var negative = false;
    var value: i64 = 0;
    
    if (i < input.len and input[i] == '-') {
        negative = true;
        i += 1;
    }
    
    if (i >= input.len or input[i] < '0' or input[i] > '9') {
        return .{ .value = 0, .bytes_consumed = 0, .success = false };
    }
    
    while (i < input.len and input[i] >= '0' and input[i] <= '9') {
        value = value * 10 + @as(i64, input[i] - '0');
        i += 1;
    }
    
    if (negative) value = -value;
    
    return .{
        .value = value,
        .bytes_consumed = i,
        .success = true,
    };
}

pub fn main() void {
    const inputs = [_][]const u8{ "123abc", "-456xyz", "hello", "0" };
    
    for (inputs) |input| {
        const result = parseInteger(input);
        if (result.success) {
            std.debug.print("'{s}' → ค่า: {d}, อ่านไป {d} ตัวอักษร\n", .{
                input,
                result.value,
                result.bytes_consumed,
            });
        } else {
            std.debug.print("'{s}' → ไม่สามารถ parse ได้\n", .{input});
        }
    }
}
```

### ตัวอย่างจริง: สถิติ

```zig
const std = @import("std");

const Statistics = struct {
    count: usize,
    sum: f64,
    mean: f64,
    variance: f64,
    std_dev: f64,
};

fn calculateStats(data: []const f64) Statistics {
    if (data.len == 0) {
        return .{
            .count = 0,
            .sum = 0.0,
            .mean = 0.0,
            .variance = 0.0,
            .std_dev = 0.0,
        };
    }
    
    // คำนวณผลรวมและค่าเฉลี่ย
    var sum: f64 = 0.0;
    for (data) |x| {
        sum += x;
    }
    const mean = sum / @as(f64, @floatFromInt(data.len));
    
    // คำนวณ variance
    var variance_sum: f64 = 0.0;
    for (data) |x| {
        const diff = x - mean;
        variance_sum += diff * diff;
    }
    const variance = variance_sum / @as(f64, @floatFromInt(data.len));
    
    return .{
        .count = data.len,
        .sum = sum,
        .mean = mean,
        .variance = variance,
        .std_dev = @sqrt(variance),
    };
}

pub fn main() void {
    const scores = [_]f64{ 75.0, 82.0, 90.0, 68.0, 95.0, 77.0, 88.0, 72.0 };
    const stats = calculateStats(&scores);
    
    std.debug.print("จำนวนข้อมูล: {d}\n", .{stats.count});
    std.debug.print("ผลรวม: {d:.2}\n", .{stats.sum});
    std.debug.print("ค่าเฉลี่ย: {d:.2}\n", .{stats.mean});
    std.debug.print("Variance: {d:.2}\n", .{stats.variance});
    std.debug.print("ส่วนเบี่ยงเบนมาตรฐาน: {d:.2}\n", .{stats.std_dev});
}
```

---

## Step 74: ฟังก์ชันที่ Return Error

### Error Handling ใน Zig

Zig มีระบบจัดการข้อผิดพลาดที่แข็งแกร่งด้วย error union type (`!T`) ที่บังคับให้โปรแกรมเมอร์จัดการข้อผิดพลาดทุกกรณี

### Error Set พื้นฐาน

```zig
const std = @import("std");

// กำหนด error set
const MathError = error{
    DivisionByZero,
    NegativeSquareRoot,
    Overflow,
};

// ฟังก์ชันที่อาจ return error
fn safeDivide(a: f64, b: f64) MathError!f64 {
    if (b == 0.0) return MathError.DivisionByZero;
    return a / b;
}

fn safeSqrt(x: f64) MathError!f64 {
    if (x < 0.0) return MathError.NegativeSquareRoot;
    return @sqrt(x);
}

pub fn main() void {
    // การจัดการด้วย catch
    const result1 = safeDivide(10.0, 3.0) catch |err| {
        std.debug.print("ข้อผิดพลาด: {}\n", .{err});
        return;
    };
    std.debug.print("10 / 3 = {d:.4}\n", .{result1});
    
    // การจัดการด้วย if/else
    if (safeDivide(5.0, 0.0)) |value| {
        std.debug.print("ผลลัพธ์: {d}\n", .{value});
    } else |err| {
        std.debug.print("เกิดข้อผิดพลาด: {s}\n", .{@errorName(err)});
    }
    
    // การจัดการ sqrt
    const root = safeSqrt(-4.0) catch 0.0;
    std.debug.print("sqrt(-4) catch 0 = {d}\n", .{root});
}
```

### Propagating Errors ด้วย try

```zig
const std = @import("std");

const FileError = error{
    NotFound,
    PermissionDenied,
    InvalidFormat,
};

fn readConfig(filename: []const u8) FileError![]const u8 {
    // จำลองการอ่านไฟล์
    if (std.mem.eql(u8, filename, "missing.txt")) {
        return FileError.NotFound;
    }
    if (std.mem.eql(u8, filename, "secret.txt")) {
        return FileError.PermissionDenied;
    }
    return "key=value\nport=8080\n";
}

fn parseConfig(content: []const u8) FileError!u16 {
    // จำลองการ parse config
    _ = content;
    return 8080;
}

// ฟังก์ชันที่ propagate error ด้วย try
fn loadServerConfig(filename: []const u8) FileError!u16 {
    const content = try readConfig(filename);  // ถ้า error จะ return ทันที
    const port = try parseConfig(content);     // เช่นเดียวกัน
    return port;
}

pub fn main() void {
    const filenames = [_][]const u8{ "config.txt", "missing.txt", "secret.txt" };
    
    for (filenames) |filename| {
        if (loadServerConfig(filename)) |port| {
            std.debug.print("ไฟล์ '{s}': port = {d}\n", .{ filename, port });
        } else |err| {
            std.debug.print("ไฟล์ '{s}': ข้อผิดพลาด - {s}\n", .{
                filename,
                @errorName(err),
            });
        }
    }
}
```

### Anyerror และ Error Union

```zig
const std = @import("std");

const ValidationError = error{
    TooShort,
    TooLong,
    InvalidCharacter,
    Empty,
};

fn validateUsername(username: []const u8) ValidationError!void {
    if (username.len == 0) return ValidationError.Empty;
    if (username.len < 3) return ValidationError.TooShort;
    if (username.len > 20) return ValidationError.TooLong;
    
    for (username) |c| {
        const valid = (c >= 'a' and c <= 'z') or
                      (c >= 'A' and c <= 'Z') or
                      (c >= '0' and c <= '9') or
                      c == '_';
        if (!valid) return ValidationError.InvalidCharacter;
    }
    // ถ้าผ่านทุกเงื่อนไข ไม่ต้อง return อะไร (void)
}

fn validatePassword(password: []const u8) ValidationError!void {
    if (password.len == 0) return ValidationError.Empty;
    if (password.len < 8) return ValidationError.TooShort;
    if (password.len > 64) return ValidationError.TooLong;
}

pub fn main() void {
    const usernames = [_][]const u8{
        "john_doe",
        "ab",
        "",
        "this_username_is_way_too_long_for_our_system",
        "invalid user!",
    };
    
    for (usernames) |username| {
        if (validateUsername(username)) {
            std.debug.print("'{s}' - ชื่อผู้ใช้ถูกต้อง\n", .{username});
        } else |err| {
            const msg = switch (err) {
                ValidationError.Empty => "ไม่สามารถเว้นว่างได้",
                ValidationError.TooShort => "สั้นเกินไป (ต้องการอย่างน้อย 3 ตัวอักษร)",
                ValidationError.TooLong => "ยาวเกินไป (ไม่เกิน 20 ตัวอักษร)",
                ValidationError.InvalidCharacter => "มีตัวอักษรที่ไม่อนุญาต",
            };
            std.debug.print("'{s}' - ไม่ถูกต้อง: {s}\n", .{ username, msg });
        }
    }
}
```

### ตัวอย่างจริง: การ Parse และ Validate ข้อมูล

```zig
const std = @import("std");

const ParseError = error{
    InvalidFormat,
    OutOfRange,
    UnexpectedEnd,
};

const Date = struct {
    year: u16,
    month: u8,
    day: u8,
};

fn parseDate(input: []const u8) ParseError!Date {
    // คาดว่า format: YYYY-MM-DD
    if (input.len != 10) return ParseError.InvalidFormat;
    if (input[4] != '-' or input[7] != '-') return ParseError.InvalidFormat;
    
    // Parse year
    var year: u16 = 0;
    for (input[0..4]) |c| {
        if (c < '0' or c > '9') return ParseError.InvalidFormat;
        year = year * 10 + @as(u16, c - '0');
    }
    
    // Parse month
    var month: u8 = 0;
    for (input[5..7]) |c| {
        if (c < '0' or c > '9') return ParseError.InvalidFormat;
        month = month * 10 + (c - '0');
    }
    if (month < 1 or month > 12) return ParseError.OutOfRange;
    
    // Parse day
    var day: u8 = 0;
    for (input[8..10]) |c| {
        if (c < '0' or c > '9') return ParseError.InvalidFormat;
        day = day * 10 + (c - '0');
    }
    if (day < 1 or day > 31) return ParseError.OutOfRange;
    
    return .{ .year = year, .month = month, .day = day };
}

pub fn main() void {
    const dates = [_][]const u8{
        "2567-10-15",
        "2567-13-01",
        "invalid",
        "2568-01-01",
    };
    
    for (dates) |dateStr| {
        if (parseDate(dateStr)) |date| {
            std.debug.print("'{s}' → ปี {d}, เดือน {d}, วัน {d}\n", .{
                dateStr, date.year, date.month, date.day,
            });
        } else |err| {
            std.debug.print("'{s}' → ข้อผิดพลาด: {s}\n", .{
                dateStr, @errorName(err),
            });
        }
    }
}
```

---

## Step 75: Recursive Functions

### แนวคิดของ Recursion

Recursion คือการที่ฟังก์ชันเรียกตัวเองซ้ำ จนกว่าจะถึงเงื่อนไขหยุด (base case)

### Factorial

```zig
const std = @import("std");

fn factorial(n: u64) u64 {
    // Base case: หยุดที่ 0 หรือ 1
    if (n <= 1) return 1;
    // Recursive case: n! = n × (n-1)!
    return n * factorial(n - 1);
}

pub fn main() void {
    var i: u64 = 0;
    while (i <= 10) : (i += 1) {
        std.debug.print("{d}! = {d}\n", .{ i, factorial(i) });
    }
}
```

ผลลัพธ์:
```
0! = 1
1! = 1
2! = 2
3! = 6
4! = 24
...
10! = 3628800
```

### Fibonacci

```zig
const std = @import("std");

// Fibonacci แบบ Naive (ช้ามาก)
fn fibonacci(n: u32) u64 {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Fibonacci แบบ Tail Recursion (มีประสิทธิภาพกว่า)
fn fibTailRecursive(n: u32, a: u64, b: u64) u64 {
    if (n == 0) return a;
    return fibTailRecursive(n - 1, b, a + b);
}

fn fib(n: u32) u64 {
    return fibTailRecursive(n, 0, 1);
}

pub fn main() void {
    std.debug.print("Fibonacci (naive):\n", .{});
    var i: u32 = 0;
    while (i <= 15) : (i += 1) {
        std.debug.print("fib({d}) = {d}\n", .{ i, fibonacci(i) });
    }
    
    std.debug.print("\nFibonacci (tail recursive):\n", .{});
    i = 0;
    while (i <= 20) : (i += 1) {
        std.debug.print("fib({d}) = {d}\n", .{ i, fib(i) });
    }
}
```

### Tower of Hanoi

```zig
const std = @import("std");

var move_count: u32 = 0;

fn hanoi(n: u32, from: []const u8, to: []const u8, via: []const u8) void {
    if (n == 1) {
        move_count += 1;
        std.debug.print("ย้ายแผ่น {d}: จาก {s} ไป {s}\n", .{ n, from, to });
        return;
    }
    hanoi(n - 1, from, via, to);
    move_count += 1;
    std.debug.print("ย้ายแผ่น {d}: จาก {s} ไป {s}\n", .{ n, from, to });
    hanoi(n - 1, via, to, from);
}

pub fn main() void {
    const disks: u32 = 3;
    std.debug.print("Tower of Hanoi ({d} แผ่น):\n", .{disks});
    hanoi(disks, "A", "C", "B");
    std.debug.print("\nจำนวนการย้ายทั้งหมด: {d} ครั้ง\n", .{move_count});
    std.debug.print("(ต้องการ 2^{d} - 1 = {d} ครั้ง)\n", .{
        disks,
        std.math.pow(u32, 2, disks) - 1,
    });
}
```

### Binary Search แบบ Recursive

```zig
const std = @import("std");

fn binarySearch(arr: []const i32, target: i32, low: usize, high: usize) ?usize {
    // Base case: ไม่พบ
    if (low > high) return null;
    
    const mid = low + (high - low) / 2;
    
    if (arr[mid] == target) return mid;
    if (arr[mid] < target) {
        if (mid + 1 > high) return null;
        return binarySearch(arr, target, mid + 1, high);
    } else {
        if (mid == 0) return null;
        return binarySearch(arr, target, low, mid - 1);
    }
}

pub fn main() void {
    const sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 };
    const targets = [_]i32{ 7, 1, 19, 6, 15 };
    
    for (targets) |target| {
        if (binarySearch(&sorted, target, 0, sorted.len - 1)) |index| {
            std.debug.print("พบ {d} ที่ index {d}\n", .{ target, index });
        } else {
            std.debug.print("ไม่พบ {d} ในอาร์เรย์\n", .{target});
        }
    }
}
```

### Merge Sort แบบ Recursive

```zig
const std = @import("std");

fn merge(arr: []i32, left: []const i32, right: []const i32) void {
    var i: usize = 0;
    var j: usize = 0;
    var k: usize = 0;
    
    while (i < left.len and j < right.len) {
        if (left[i] <= right[j]) {
            arr[k] = left[i];
            i += 1;
        } else {
            arr[k] = right[j];
            j += 1;
        }
        k += 1;
    }
    
    while (i < left.len) {
        arr[k] = left[i];
        i += 1;
        k += 1;
    }
    
    while (j < right.len) {
        arr[k] = right[j];
        j += 1;
        k += 1;
    }
}

fn mergeSort(allocator: std.mem.Allocator, arr: []i32) !void {
    if (arr.len <= 1) return;
    
    const mid = arr.len / 2;
    
    // คัดลอกส่วนซ้ายและขวา
    const left = try allocator.dupe(i32, arr[0..mid]);
    defer allocator.free(left);
    const right = try allocator.dupe(i32, arr[mid..]);
    defer allocator.free(right);
    
    // Recursively sort
    try mergeSort(allocator, left);
    try mergeSort(allocator, right);
    
    // Merge กลับ
    merge(arr, left, right);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var data = [_]i32{ 38, 27, 43, 3, 9, 82, 10 };
    
    std.debug.print("ก่อนเรียง: ", .{});
    for (data) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    try mergeSort(allocator, &data);
    
    std.debug.print("หลังเรียง: ", .{});
    for (data) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
}
```

---

## Step 76: Function Pointers

### แนวคิด Function Pointers

Function pointer คือตัวแปรที่เก็บ address ของฟังก์ชัน ทำให้สามารถส่งฟังก์ชันเป็น parameter หรือเก็บในโครงสร้างข้อมูลได้

### ไวยากรณ์ Function Pointer ใน Zig

```zig
// ประกาศ type ของ function pointer
// *const fn(พารามิเตอร์) ชนิดคืนค่า

const MathFn = *const fn(i32, i32) i32;
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

fn add(a: i32, b: i32) i32 { return a + b; }
fn subtract(a: i32, b: i32) i32 { return a - b; }
fn multiply(a: i32, b: i32) i32 { return a * b; }

// ฟังก์ชันที่รับ function pointer เป็น parameter
fn applyOperation(
    a: i32,
    b: i32,
    operation: *const fn(i32, i32) i32,
) i32 {
    return operation(a, b);
}

pub fn main() void {
    // เก็บ function pointer ในตัวแปร
    const ops = [_]*const fn(i32, i32) i32{ add, subtract, multiply };
    const names = [_][]const u8{ "บวก", "ลบ", "คูณ" };
    
    const a: i32 = 10;
    const b: i32 = 3;
    
    for (ops, 0..) |op, i| {
        const result = applyOperation(a, b, op);
        std.debug.print("{d} {s} {d} = {d}\n", .{ a, names[i], b, result });
    }
}
```

### Use Case 1: Strategy Pattern

```zig
const std = @import("std");

// Strategy: วิธีการเรียงข้อมูล
const CompareFunc = *const fn(a: i32, b: i32) bool;

fn lessThan(a: i32, b: i32) bool { return a < b; }
fn greaterThan(a: i32, b: i32) bool { return a > b; }

fn bubbleSort(arr: []i32, compare: CompareFunc) void {
    const n = arr.len;
    var i: usize = 0;
    while (i < n - 1) : (i += 1) {
        var j: usize = 0;
        while (j < n - 1 - i) : (j += 1) {
            if (compare(arr[j + 1], arr[j])) {
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

pub fn main() void {
    var data1 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    var data2 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    
    bubbleSort(&data1, lessThan);
    std.debug.print("เรียงน้อยไปมาก: ", .{});
    for (data1) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    bubbleSort(&data2, greaterThan);
    std.debug.print("เรียงมากไปน้อย: ", .{});
    for (data2) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
}
```

### Use Case 2: Event System

```zig
const std = @import("std");

const EventType = enum { click, hover, keypress };
const EventHandler = *const fn(event: EventType, data: []const u8) void;

const MAX_HANDLERS = 10;

const EventSystem = struct {
    handlers: [MAX_HANDLERS]EventHandler,
    count: usize,
    
    fn init() EventSystem {
        return .{
            .handlers = undefined,
            .count = 0,
        };
    }
    
    fn register(self: *EventSystem, handler: EventHandler) void {
        if (self.count < MAX_HANDLERS) {
            self.handlers[self.count] = handler;
            self.count += 1;
        }
    }
    
    fn dispatch(self: *const EventSystem, event: EventType, data: []const u8) void {
        for (self.handlers[0..self.count]) |handler| {
            handler(event, data);
        }
    }
};

fn logHandler(event: EventType, data: []const u8) void {
    std.debug.print("[LOG] เหตุการณ์: {s}, ข้อมูล: {s}\n", .{
        @tagName(event), data,
    });
}

fn analyticsHandler(event: EventType, data: []const u8) void {
    std.debug.print("[ANALYTICS] ติดตาม: {s} - {s}\n", .{
        @tagName(event), data,
    });
}

fn securityHandler(event: EventType, data: []const u8) void {
    if (event == .keypress) {
        std.debug.print("[SECURITY] ตรวจสอบการกดคีย์: {s}\n", .{data});
    }
}

pub fn main() void {
    var events = EventSystem.init();
    events.register(logHandler);
    events.register(analyticsHandler);
    events.register(securityHandler);
    
    events.dispatch(.click, "button#submit");
    events.dispatch(.keypress, "Enter");
    events.dispatch(.hover, "menu-item");
}
```

### Use Case 3: Plugin System

```zig
const std = @import("std");

const Plugin = struct {
    name: []const u8,
    process: *const fn(input: []const u8) []const u8,
    cleanup: ?*const fn() void,
};

fn uppercasePlugin(input: []const u8) []const u8 {
    // ในการใช้จริงต้องจัดการ memory แต่นี่แค่ตัวอย่าง
    _ = input;
    return "PROCESSED_UPPERCASE";
}

fn trimPlugin(input: []const u8) []const u8 {
    // Trim whitespace จากต้นและท้าย
    var start: usize = 0;
    var end: usize = input.len;
    
    while (start < end and (input[start] == ' ' or input[start] == '\t')) {
        start += 1;
    }
    while (end > start and (input[end - 1] == ' ' or input[end - 1] == '\t')) {
        end -= 1;
    }
    
    return input[start..end];
}

fn cleanupUppercase() void {
    std.debug.print("[Cleanup] uppercase plugin\n", .{});
}

pub fn main() void {
    const plugins = [_]Plugin{
        .{
            .name = "uppercase",
            .process = uppercasePlugin,
            .cleanup = cleanupUppercase,
        },
        .{
            .name = "trim",
            .process = trimPlugin,
            .cleanup = null,
        },
    };
    
    const input = "  hello world  ";
    
    for (plugins) |plugin| {
        const output = plugin.process(input);
        std.debug.print("Plugin '{s}': '{s}' → '{s}'\n", .{
            plugin.name, input, output,
        });
    }
    
    // Cleanup
    for (plugins) |plugin| {
        if (plugin.cleanup) |cleanup_fn| {
            cleanup_fn();
        }
    }
}
```

### Function Pointers ใน Array

```zig
const std = @import("std");

// Dispatch table สำหรับ calculator
const Operation = enum { add, sub, mul, div };

fn calc_add(a: f64, b: f64) f64 { return a + b; }
fn calc_sub(a: f64, b: f64) f64 { return a - b; }
fn calc_mul(a: f64, b: f64) f64 { return a * b; }
fn calc_div(a: f64, b: f64) f64 { 
    if (b == 0.0) return 0.0;
    return a / b; 
}

// Dispatch table: แทนที่ switch statement ยาวๆ
const dispatch_table = std.EnumArray(Operation, *const fn(f64, f64) f64).init(.{
    .add = calc_add,
    .sub = calc_sub,
    .mul = calc_mul,
    .div = calc_div,
});

pub fn main() void {
    const a: f64 = 15.0;
    const b: f64 = 4.0;
    const ops = [_]Operation{ .add, .sub, .mul, .div };
    const symbols = [_][]const u8{ "+", "-", "×", "÷" };
    
    for (ops, 0..) |op, i| {
        const func = dispatch_table.get(op);
        const result = func(a, b);
        std.debug.print("{d} {s} {d} = {d:.2}\n", .{ a, symbols[i], b, result });
    }
}
```

---

## Step 77: Anonymous Functions (Lambda)

### Anonymous Functions ใน Zig

Zig ไม่มี lambda syntax แบบ `=>` เหมือน JavaScript แต่มีวิธีสร้างฟังก์ชันแบบ anonymous ผ่าน struct ที่มี method และการใช้ inline functions

### Anonymous Struct Function

```zig
const std = @import("std");

pub fn main() void {
    // Anonymous function โดยใช้ struct
    const double = struct {
        fn call(x: i32) i32 {
            return x * 2;
        }
    }.call;
    
    const add5 = struct {
        fn call(x: i32) i32 {
            return x + 5;
        }
    }.call;
    
    std.debug.print("double(7) = {d}\n", .{double(7)});
    std.debug.print("add5(7) = {d}\n", .{add5(7)});
    
    // ใช้เป็น parameter
    const values = [_]i32{ 1, 2, 3, 4, 5 };
    std.debug.print("ค่าต่างๆ: ", .{});
    for (values) |v| {
        std.debug.print("{d} ", .{double(v)});
    }
    std.debug.print("\n", .{});
}
```

### Closure-like Pattern

Zig ไม่รองรับ closure ที่ capture ตัวแปรจาก scope ภายนอกโดยตรง แต่สามารถจำลองได้ด้วย struct:

```zig
const std = @import("std");

// Closure-like: adder ที่มี captured value
fn makeAdder(n: i32) type {
    return struct {
        captured: i32,
        
        const Self = @This();
        
        fn init(value: i32) Self {
            return .{ .captured = value };
        }
        
        fn call(self: Self, x: i32) i32 {
            return self.captured + x;
        }
    };
}

pub fn main() void {
    const Adder5 = makeAdder(5);
    const add5 = Adder5.init(5);
    
    const Adder10 = makeAdder(10);
    const add10 = Adder10.init(10);
    
    std.debug.print("add5(3) = {d}\n", .{add5.call(3)});
    std.debug.print("add5(7) = {d}\n", .{add5.call(7)});
    std.debug.print("add10(3) = {d}\n", .{add10.call(3)});
}
```

### Higher-Order Functions

```zig
const std = @import("std");

// map: ใช้ function กับทุกสมาชิกของ array
fn map(
    comptime T: type,
    comptime U: type,
    input: []const T,
    output: []U,
    transform: *const fn(T) U,
) void {
    for (input, 0..) |item, i| {
        output[i] = transform(item);
    }
}

// filter: กรองสมาชิกตามเงื่อนไข
fn filter(
    comptime T: type,
    input: []const T,
    output: []T,
    predicate: *const fn(T) bool,
) usize {
    var count: usize = 0;
    for (input) |item| {
        if (predicate(item)) {
            output[count] = item;
            count += 1;
        }
    }
    return count;
}

// reduce: รวมค่าทั้งหมด
fn reduce(
    comptime T: type,
    input: []const T,
    initial: T,
    combine: *const fn(T, T) T,
) T {
    var result = initial;
    for (input) |item| {
        result = combine(result, item);
    }
    return result;
}

fn square(x: i32) i32 { return x * x; }
fn isEven(x: i32) bool { return @mod(x, 2) == 0; }
fn sum(a: i32, b: i32) i32 { return a + b; }
fn product(a: i32, b: i32) i32 { return a * b; }

pub fn main() void {
    const numbers = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    
    // Map: ยกกำลังสอง
    var squares: [10]i32 = undefined;
    map(i32, i32, &numbers, &squares, square);
    std.debug.print("กำลังสอง: ", .{});
    for (squares) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    // Filter: เลขคู่
    var evens: [10]i32 = undefined;
    const even_count = filter(i32, &numbers, &evens, isEven);
    std.debug.print("เลขคู่: ", .{});
    for (evens[0..even_count]) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    // Reduce: ผลรวม
    const total = reduce(i32, &numbers, 0, sum);
    std.debug.print("ผลรวม 1-10: {d}\n", .{total});
    
    // Reduce: ผลคูณ
    const small_numbers = [_]i32{ 1, 2, 3, 4, 5 };
    const prod = reduce(i32, &small_numbers, 1, product);
    std.debug.print("5! = {d}\n", .{prod});
}
```

### Callback Pattern

```zig
const std = @import("std");

const ProgressCallback = *const fn(current: usize, total: usize) void;

fn processItems(
    items: []const []const u8,
    on_progress: ?ProgressCallback,
    on_complete: ?*const fn(count: usize) void,
) void {
    for (items, 0..) |item, i| {
        // Process item
        std.debug.print("  กำลังประมวลผล: {s}\n", .{item});
        
        // แจ้ง progress
        if (on_progress) |callback| {
            callback(i + 1, items.len);
        }
    }
    
    // แจ้ง complete
    if (on_complete) |callback| {
        callback(items.len);
    }
}

fn progressBar(current: usize, total: usize) void {
    const percentage = (current * 100) / total;
    std.debug.print("  ความคืบหน้า: {d}%\n", .{percentage});
}

fn onDone(count: usize) void {
    std.debug.print("เสร็จสิ้น! ประมวลผล {d} รายการ\n", .{count});
}

pub fn main() void {
    const items = [_][]const u8{
        "ไฟล์ A",
        "ไฟล์ B",
        "ไฟล์ C",
    };
    
    std.debug.print("เริ่มประมวลผล...\n", .{});
    processItems(&items, progressBar, onDone);
}
```

---

## Step 78: Comptime Functions

### แนวคิด Comptime

`comptime` คือการคำนวณที่เกิดขึ้นในเวลา compile ไม่ใช่ runtime ทำให้โปรแกรมเร็วขึ้นและสามารถสร้าง generic code ได้

### Comptime Functions พื้นฐาน

```zig
const std = @import("std");

// ฟังก์ชันที่ทำงาน comptime
fn maxValue(comptime T: type) T {
    return switch (T) {
        u8 => 255,
        u16 => 65535,
        u32 => 4294967295,
        i8 => 127,
        i16 => 32767,
        i32 => 2147483647,
        else => @compileError("ไม่รองรับชนิด " ++ @typeName(T)),
    };
}

pub fn main() void {
    std.debug.print("ค่าสูงสุดของ u8: {d}\n", .{maxValue(u8)});
    std.debug.print("ค่าสูงสุดของ u16: {d}\n", .{maxValue(u16)});
    std.debug.print("ค่าสูงสุดของ i32: {d}\n", .{maxValue(i32)});
    // maxValue(f32) จะทำให้ compile error
}
```

### Generic Functions ด้วย Comptime

```zig
const std = @import("std");

// Generic function ที่รับชนิดใดก็ได้ที่เป็นตัวเลข
fn min(comptime T: type, a: T, b: T) T {
    return if (a < b) a else b;
}

fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

fn clamp(comptime T: type, value: T, lo: T, hi: T) T {
    return max(T, lo, min(T, value, hi));
}

pub fn main() void {
    // ใช้กับ integer
    std.debug.print("min(i32, 3, 7) = {d}\n", .{min(i32, 3, 7)});
    std.debug.print("max(u8, 200, 100) = {d}\n", .{max(u8, 200, 100)});
    
    // ใช้กับ float
    std.debug.print("min(f64, 3.14, 2.71) = {d:.2}\n", .{min(f64, 3.14, 2.71)});
    
    // clamp
    std.debug.print("clamp(i32, 15, 0, 10) = {d}\n", .{clamp(i32, 15, 0, 10)});
    std.debug.print("clamp(i32, -5, 0, 10) = {d}\n", .{clamp(i32, -5, 0, 10)});
    std.debug.print("clamp(i32, 5, 0, 10) = {d}\n", .{clamp(i32, 5, 0, 10)});
}
```

### Generic Data Structures

```zig
const std = @import("std");

// Generic Stack
fn Stack(comptime T: type) type {
    return struct {
        items: [100]T,
        count: usize,
        
        const Self = @This();
        
        fn init() Self {
            return .{ .items = undefined, .count = 0 };
        }
        
        fn push(self: *Self, item: T) !void {
            if (self.count >= 100) return error.StackOverflow;
            self.items[self.count] = item;
            self.count += 1;
        }
        
        fn pop(self: *Self) !T {
            if (self.count == 0) return error.StackUnderflow;
            self.count -= 1;
            return self.items[self.count];
        }
        
        fn peek(self: *const Self) !T {
            if (self.count == 0) return error.Empty;
            return self.items[self.count - 1];
        }
        
        fn isEmpty(self: *const Self) bool {
            return self.count == 0;
        }
        
        fn size(self: *const Self) usize {
            return self.count;
        }
    };
}

pub fn main() !void {
    // Stack สำหรับ integer
    var int_stack = Stack(i32).init();
    try int_stack.push(10);
    try int_stack.push(20);
    try int_stack.push(30);
    
    std.debug.print("Stack integers:\n", .{});
    while (!int_stack.isEmpty()) {
        const value = try int_stack.pop();
        std.debug.print("  pop: {d}\n", .{value});
    }
    
    // Stack สำหรับ string
    var str_stack = Stack([]const u8).init();
    try str_stack.push("แอปเปิ้ล");
    try str_stack.push("กล้วย");
    try str_stack.push("ส้ม");
    
    std.debug.print("\nStack strings:\n", .{});
    while (!str_stack.isEmpty()) {
        const value = try str_stack.pop();
        std.debug.print("  pop: {s}\n", .{value});
    }
}
```

### Comptime Type Introspection

```zig
const std = @import("std");

// ฟังก์ชันที่ตรวจสอบ type ใน comptime
fn printTypeInfo(comptime T: type) void {
    const info = @typeInfo(T);
    std.debug.print("Type: {s}\n", .{@typeName(T)});
    
    switch (info) {
        .Int => |int_info| {
            std.debug.print("  Integer: {d} bits, signed: {}\n", .{
                int_info.bits,
                int_info.signedness == .signed,
            });
        },
        .Float => |float_info| {
            std.debug.print("  Float: {d} bits\n", .{float_info.bits});
        },
        .Bool => {
            std.debug.print("  Boolean type\n", .{});
        },
        .Struct => |struct_info| {
            std.debug.print("  Struct with {d} fields:\n", .{struct_info.fields.len});
            inline for (struct_info.fields) |field| {
                std.debug.print("    - {s}: {s}\n", .{
                    field.name,
                    @typeName(field.type),
                });
            }
        },
        else => {
            std.debug.print("  (ชนิดอื่นๆ)\n", .{});
        },
    }
}

const Point = struct {
    x: f32,
    y: f32,
    z: f32,
};

const Person = struct {
    name: []const u8,
    age: u8,
    active: bool,
};

pub fn main() void {
    printTypeInfo(i32);
    printTypeInfo(f64);
    printTypeInfo(bool);
    printTypeInfo(Point);
    printTypeInfo(Person);
}
```

### Comptime Assertions

```zig
const std = @import("std");

// ตรวจสอบ type ใน comptime
fn onlyNumbers(comptime T: type, value: T) T {
    // compile error ถ้าไม่ใช่ตัวเลข
    switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {},
        else => @compileError("ต้องการชนิดตัวเลขเท่านั้น แต่ได้รับ: " ++ @typeName(T)),
    }
    return value * value;
}

// ตรวจสอบ struct fields ใน comptime
fn hasField(comptime T: type, comptime field_name: []const u8) bool {
    inline for (@typeInfo(T).Struct.fields) |field| {
        if (std.mem.eql(u8, field.name, field_name)) return true;
    }
    return false;
}

const User = struct {
    id: u32,
    name: []const u8,
    email: []const u8,
};

pub fn main() void {
    std.debug.print("square(5) = {d}\n", .{onlyNumbers(i32, 5)});
    std.debug.print("square(3.14) = {d:.4}\n", .{onlyNumbers(f64, 3.14)});
    
    std.debug.print("\nUser struct fields:\n", .{});
    std.debug.print("  มี 'id': {}\n", .{hasField(User, "id")});
    std.debug.print("  มี 'name': {}\n", .{hasField(User, "name")});
    std.debug.print("  มี 'password': {}\n", .{hasField(User, "password")});
}
```

### Generic Algorithms

```zig
const std = @import("std");

// Generic sort ที่ใช้ได้กับ type ใดก็ได้ที่เปรียบเทียบได้
fn sortAsc(comptime T: type, arr: []T) void {
    const n = arr.len;
    var i: usize = 0;
    while (i < n) : (i += 1) {
        var j: usize = i + 1;
        while (j < n) : (j += 1) {
            if (arr[j] < arr[i]) {
                const temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
    }
}

// Generic sum
fn sum(comptime T: type, values: []const T) T {
    var total: T = 0;
    for (values) |v| {
        total += v;
    }
    return total;
}

pub fn main() void {
    var ints = [_]i32{ 5, 2, 8, 1, 9, 3 };
    sortAsc(i32, &ints);
    std.debug.print("เรียง int: ", .{});
    for (ints) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
    
    var floats = [_]f64{ 3.14, 1.41, 2.71, 1.73 };
    sortAsc(f64, &floats);
    std.debug.print("เรียง float: ", .{});
    for (floats) |v| std.debug.print("{d:.2} ", .{v});
    std.debug.print("\n", .{});
    
    const int_sum = sum(i32, &ints);
    std.debug.print("ผลรวม int: {d}\n", .{int_sum});
}
```

---

## Step 79: Inline Functions

### แนวคิด Inline Functions

`inline fn` บอก compiler ให้ขยายโค้ดของฟังก์ชันตรงที่เรียกใช้ แทนที่จะ call ไปยัง function จริงๆ ช่วยเพิ่มประสิทธิภาพลดการ overhead ของ function call

### Syntax

```zig
inline fn functionName(params) ReturnType {
    // body
}
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

// ฟังก์ชัน inline สำหรับการคำนวณที่เรียกบ่อย
inline fn square(x: f32) f32 {
    return x * x;
}

inline fn cube(x: f32) f32 {
    return x * x * x;
}

inline fn lerp(a: f32, b: f32, t: f32) f32 {
    return a + (b - a) * t;
}

pub fn main() void {
    const x: f32 = 3.0;
    std.debug.print("{d}² = {d}\n", .{ x, square(x) });
    std.debug.print("{d}³ = {d}\n", .{ x, cube(x) });
    
    const start: f32 = 0.0;
    const end: f32 = 100.0;
    std.debug.print("lerp(0, 100, 0.25) = {d}\n", .{lerp(start, end, 0.25)});
    std.debug.print("lerp(0, 100, 0.75) = {d}\n", .{lerp(start, end, 0.75)});
}
```

### Inline vs Regular Functions

```zig
const std = @import("std");

// Regular function (มี function call overhead)
fn regularAdd(a: i32, b: i32) i32 {
    return a + b;
}

// Inline function (ขยายโค้ดตรงที่เรียก ไม่มี overhead)
inline fn inlineAdd(a: i32, b: i32) i32 {
    return a + b;
}

// Inline for ทำงานใน loop ที่ทราบจำนวนครั้งตอน compile time
pub fn main() void {
    var sum_regular: i32 = 0;
    var sum_inline: i32 = 0;
    
    // Loop ปกติ
    var i: i32 = 1;
    while (i <= 100) : (i += 1) {
        sum_regular = regularAdd(sum_regular, i);
    }
    
    // Loop ด้วย inline function
    i = 1;
    while (i <= 100) : (i += 1) {
        sum_inline = inlineAdd(sum_inline, i);
    }
    
    std.debug.print("ผลรวม (regular): {d}\n", .{sum_regular});
    std.debug.print("ผลรวม (inline): {d}\n", .{sum_inline});
}
```

### Inline for Loop

```zig
const std = @import("std");

// inline for ทำงานกับ comptime-known tuples/arrays
pub fn main() void {
    // inline for กับ tuple ชนิดต่างกัน
    const values = .{ 42, 3.14, true, "hello" };
    
    std.debug.print("ข้อมูลต่างชนิด:\n", .{});
    inline for (values) |v| {
        std.debug.print("  {any} (ชนิด: {s})\n", .{ v, @typeName(@TypeOf(v)) });
    }
    
    // inline for กับ type list
    const types = [_]type{ u8, u16, u32, u64 };
    std.debug.print("\nขนาด integer types:\n", .{});
    inline for (types) |T| {
        std.debug.print("  {s}: {d} bytes\n", .{ @typeName(T), @sizeOf(T) });
    }
}
```

### Inline for กับ Struct Fields

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    max_connections: u32,
};

fn printStruct(comptime T: type, value: T) void {
    std.debug.print("--- {s} ---\n", .{@typeName(T)});
    inline for (@typeInfo(T).Struct.fields) |field| {
        const field_value = @field(value, field.name);
        std.debug.print("  {s}: {any}\n", .{ field.name, field_value });
    }
}

pub fn main() void {
    const config = Config{
        .host = "localhost",
        .port = 8080,
        .debug = true,
        .max_connections = 100,
    };
    
    printStruct(Config, config);
}
```

### Performance-Critical Code

```zig
const std = @import("std");

// การ process pixel ที่ต้องการความเร็วสูง
inline fn clampByte(value: i32) u8 {
    if (value < 0) return 0;
    if (value > 255) return 255;
    return @intCast(value);
}

inline fn rgbToGrayscale(r: u8, g: u8, b: u8) u8 {
    // สูตร luminance
    const gray: i32 = @as(i32, r) * 299 + @as(i32, g) * 587 + @as(i32, b) * 114;
    return clampByte(@divTrunc(gray, 1000));
}

// จำลอง image processing
pub fn main() void {
    const pixels = [_][3]u8{
        .{ 255, 0, 0 },     // แดง
        .{ 0, 255, 0 },     // เขียว
        .{ 0, 0, 255 },     // น้ำเงิน
        .{ 255, 255, 0 },   // เหลือง
        .{ 255, 255, 255 }, // ขาว
        .{ 0, 0, 0 },       // ดำ
    };
    
    std.debug.print("RGB to Grayscale:\n", .{});
    for (pixels) |pixel| {
        const gray = rgbToGrayscale(pixel[0], pixel[1], pixel[2]);
        std.debug.print("  RGB({d},{d},{d}) → Gray({d})\n", .{
            pixel[0], pixel[1], pixel[2], gray,
        });
    }
}
```

---

## Step 80: Varargs ด้วย anytype

### แนวคิด anytype

ใน Zig `anytype` ใช้สำหรับสร้าง generic functions ที่รับ parameter ชนิดใดก็ได้ คอมไพเลอร์จะสร้างเวอร์ชันของฟังก์ชันสำหรับแต่ละชนิดที่เรียกใช้

### anytype พื้นฐาน

```zig
const std = @import("std");

// ฟังก์ชันที่รับ parameter ชนิดใดก็ได้
fn printValue(value: anytype) void {
    const T = @TypeOf(value);
    switch (@typeInfo(T)) {
        .Int => std.debug.print("Integer: {d}\n", .{value}),
        .Float => std.debug.print("Float: {d:.4}\n", .{value}),
        .Bool => std.debug.print("Boolean: {}\n", .{value}),
        .Pointer => {
            if (T == []const u8 or T == []u8) {
                std.debug.print("String: {s}\n", .{value});
            } else {
                std.debug.print("Pointer: {*}\n", .{value});
            }
        },
        else => std.debug.print("Value: {any}\n", .{value}),
    }
}

pub fn main() void {
    printValue(42);
    printValue(3.14159);
    printValue(true);
    printValue("สวัสดี Zig");
    printValue(@as(u8, 255));
}
```

### Generic Math Functions

```zig
const std = @import("std");

fn add(a: anytype, b: anytype) @TypeOf(a, b) {
    return a + b;
}

fn multiply(a: anytype, b: anytype) @TypeOf(a, b) {
    return a * b;
}

fn average(values: anytype) f64 {
    // values ต้องเป็น slice
    if (values.len == 0) return 0.0;
    
    var sum: f64 = 0.0;
    for (values) |v| {
        sum += @as(f64, @floatCast(@as(f64, @floatFromInt(@as(i64, @intCast(v))))));
    }
    return sum / @as(f64, @floatFromInt(values.len));
}

pub fn main() void {
    // ใช้กับชนิดต่างๆ
    std.debug.print("add(3, 4) = {d}\n", .{add(3, 4)});
    std.debug.print("add(3.14, 2.71) = {d:.4}\n", .{add(3.14, 2.71)});
    std.debug.print("multiply(6, 7) = {d}\n", .{multiply(6, 7)});
    std.debug.print("multiply(2.5, 4.0) = {d}\n", .{multiply(2.5, 4.0)});
    
    const scores = [_]i32{ 80, 90, 75, 95, 85 };
    const avg = average(&scores);
    std.debug.print("ค่าเฉลี่ย: {d:.2}\n", .{avg});
}
```

### Varargs จำลองด้วย anytype และ tuple

```zig
const std = @import("std");

// จำลอง variadic function ด้วย comptime tuple
fn printAll(values: anytype) void {
    const T = @TypeOf(values);
    const fields = @typeInfo(T).Struct.fields;
    
    inline for (fields, 0..) |field, i| {
        const value = @field(values, field.name);
        if (i > 0) std.debug.print(", ", .{});
        std.debug.print("{any}", .{value});
    }
    std.debug.print("\n", .{});
}

fn sumAll(values: anytype) i64 {
    var total: i64 = 0;
    inline for (@typeInfo(@TypeOf(values)).Struct.fields) |field| {
        total += @as(i64, @intCast(@field(values, field.name)));
    }
    return total;
}

pub fn main() void {
    // เรียกใช้ด้วยจำนวน argument ต่างกัน
    std.debug.print("printAll: ", .{});
    printAll(.{ 1, 2, 3 });
    
    std.debug.print("printAll: ", .{});
    printAll(.{ "hello", 42, true });
    
    std.debug.print("printAll: ", .{});
    printAll(.{ 10, 20, 30, 40, 50 });
    
    const total = sumAll(.{ 1, 2, 3, 4, 5 });
    std.debug.print("sumAll(1,2,3,4,5) = {d}\n", .{total});
}
```

### ตัวอย่างจริง: Logger

```zig
const std = @import("std");

const LogLevel = enum { debug, info, warning, err, fatal };

const Logger = struct {
    level: LogLevel,
    prefix: []const u8,
    
    fn init(level: LogLevel, prefix: []const u8) Logger {
        return .{ .level = level, .prefix = prefix };
    }
    
    fn log(self: *const Logger, level: LogLevel, comptime fmt: []const u8, args: anytype) void {
        // ตรวจสอบ level
        if (@intFromEnum(level) < @intFromEnum(self.level)) return;
        
        const level_str = switch (level) {
            .debug => "DEBUG",
            .info => "INFO",
            .warning => "WARN",
            .err => "ERROR",
            .fatal => "FATAL",
        };
        
        std.debug.print("[{s}][{s}] ", .{ level_str, self.prefix });
        std.debug.print(fmt ++ "\n", args);
    }
    
    fn debug(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.debug, fmt, args);
    }
    
    fn info(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.info, fmt, args);
    }
    
    fn warn(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.warning, fmt, args);
    }
    
    fn err(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.err, fmt, args);
    }
};

pub fn main() void {
    var logger = Logger.init(.debug, "MyApp");
    
    logger.debug("เริ่มต้นโปรแกรม", .{});
    logger.info("เวอร์ชัน: {d}.{d}.{d}", .{ 1, 0, 0 });
    logger.warn("หน่วยความจำเหลือน้อย: {d}%", .{15});
    logger.err("ไม่สามารถเชื่อมต่อ: {s}", .{"database"});
    
    std.debug.print("\n--- เปลี่ยน level เป็น INFO ---\n", .{});
    logger.level = .info;
    logger.debug("ข้อความนี้ไม่แสดง", .{});
    logger.info("ข้อความนี้แสดง", .{});
}
```

### anytype สำหรับ Iterator Pattern

```zig
const std = @import("std");

fn forEach(items: anytype, action: anytype) void {
    for (items) |item| {
        action(item);
    }
}

fn printItem(item: anytype) void {
    std.debug.print("  → {any}\n", .{item});
}

fn doubleAndPrint(item: anytype) void {
    std.debug.print("  → {any} × 2 = {any}\n", .{ item, item * 2 });
}

pub fn main() void {
    const ints = [_]i32{ 10, 20, 30, 40, 50 };
    std.debug.print("ค่าต่างๆ:\n", .{});
    forEach(&ints, printItem);
    
    std.debug.print("\nค่าสองเท่า:\n", .{});
    forEach(&ints, doubleAndPrint);
    
    const floats = [_]f64{ 1.1, 2.2, 3.3 };
    std.debug.print("\nFloats:\n", .{});
    forEach(&floats, printItem);
}
```

### Type-Safe Serialization

```zig
const std = @import("std");

// Serialize struct เป็น JSON-like string
fn serialize(writer: anytype, value: anytype) !void {
    const T = @TypeOf(value);
    const info = @typeInfo(T);
    
    switch (info) {
        .Struct => {
            try writer.writeAll("{");
            const fields = info.Struct.fields;
            inline for (fields, 0..) |field, i| {
                if (i > 0) try writer.writeAll(", ");
                try writer.print("\"{s}\": ", .{field.name});
                try serialize(writer, @field(value, field.name));
            }
            try writer.writeAll("}");
        },
        .Int => {
            try writer.print("{d}", .{value});
        },
        .Float => {
            try writer.print("{d:.2}", .{value});
        },
        .Bool => {
            try writer.print("{}", .{value});
        },
        .Pointer => {
            if (T == []const u8) {
                try writer.print("\"{s}\"", .{value});
            } else {
                try writer.print("\"<pointer>\"", .{});
            }
        },
        else => {
            try writer.print("\"<unknown>\"", .{});
        },
    }
}

const Product = struct {
    name: []const u8,
    price: f64,
    in_stock: bool,
    quantity: u32,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const product = Product{
        .name = "แล็ปท็อป",
        .price = 29999.99,
        .in_stock = true,
        .quantity = 5,
    };
    
    try stdout.print("JSON: ", .{});
    try serialize(stdout, product);
    try stdout.print("\n", .{});
}
```

---

## ตัวอย่างโปรแกรมจริง: Function Composition

```zig
const std = @import("std");

// Function pipeline
const Pipeline = struct {
    const Transform = *const fn(i32) i32;
    
    transforms: [10]Transform,
    count: usize,
    
    fn init() Pipeline {
        return .{ .transforms = undefined, .count = 0 };
    }
    
    fn pipe(self: *Pipeline, transform: Transform) *Pipeline {
        if (self.count < 10) {
            self.transforms[self.count] = transform;
            self.count += 1;
        }
        return self;
    }
    
    fn execute(self: *const Pipeline, input: i32) i32 {
        var result = input;
        for (self.transforms[0..self.count]) |transform| {
            result = transform(result);
        }
        return result;
    }
};

fn double(x: i32) i32 { return x * 2; }
fn addTen(x: i32) i32 { return x + 10; }
fn square(x: i32) i32 { return x * x; }
fn negate(x: i32) i32 { return -x; }
fn absVal(x: i32) i32 { return if (x < 0) -x else x; }

pub fn main() void {
    var pipeline = Pipeline.init();
    _ = pipeline.pipe(double).pipe(addTen).pipe(square);
    
    const values = [_]i32{ 1, 2, 3, 4, 5 };
    
    std.debug.print("Pipeline: double → addTen → square\n", .{});
    for (values) |v| {
        const result = pipeline.execute(v);
        std.debug.print("  f({d}) = {d}\n", .{ v, result });
    }
    
    // Pipeline อื่น
    var pipeline2 = Pipeline.init();
    _ = pipeline2.pipe(negate).pipe(absVal).pipe(double);
    
    std.debug.print("\nPipeline: negate → abs → double\n", .{});
    for (values) |v| {
        const result = pipeline2.execute(v);
        std.debug.print("  f({d}) = {d}\n", .{ v, result });
    }
}
```

---

## ตัวอย่างโปรแกรมจริง: ระบบ Middleware

```zig
const std = @import("std");

// จำลอง HTTP middleware system
const Request = struct {
    method: []const u8,
    path: []const u8,
    body: []const u8,
};

const Response = struct {
    status: u16,
    body: []const u8,
};

const Handler = *const fn(req: Request) Response;
const Middleware = *const fn(req: Request, next: Handler) Response;

// Logger middleware
fn loggerMiddleware(req: Request, next: Handler) Response {
    std.debug.print("[LOG] {s} {s}\n", .{ req.method, req.path });
    const response = next(req);
    std.debug.print("[LOG] Response: {d}\n", .{response.status});
    return response;
}

// Auth middleware
fn authMiddleware(req: Request, next: Handler) Response {
    if (std.mem.eql(u8, req.path, "/admin")) {
        std.debug.print("[AUTH] ต้องการ authentication\n", .{});
        return .{ .status = 401, .body = "Unauthorized" };
    }
    return next(req);
}

// Route handlers
fn homeHandler(req: Request) Response {
    _ = req;
    return .{ .status = 200, .body = "Welcome to Home!" };
}

fn adminHandler(req: Request) Response {
    _ = req;
    return .{ .status = 200, .body = "Admin Panel" };
}

fn notFoundHandler(req: Request) Response {
    _ = req;
    return .{ .status = 404, .body = "Not Found" };
}

fn routeRequest(req: Request) Response {
    if (std.mem.eql(u8, req.path, "/")) return homeHandler(req);
    if (std.mem.eql(u8, req.path, "/admin")) return adminHandler(req);
    return notFoundHandler(req);
}

// ใช้ middleware chain
fn applyMiddlewares(req: Request, middlewares: []const Middleware, handler: Handler) Response {
    if (middlewares.len == 0) return handler(req);
    
    // สร้าง next function แบบ recursive
    const middleware = middlewares[0];
    const remaining = middlewares[1..];
    
    // สร้าง closure-like structure
    const Context = struct {
        req: Request,
        remaining: []const Middleware,
        handler: Handler,
        
        fn next(ctx: @This(), r: Request) Response {
            return applyMiddlewares(r, ctx.remaining, ctx.handler);
        }
    };
    
    _ = Context;
    _ = remaining;
    
    // เรียก middleware แรก
    return middleware(req, handler);
}

pub fn main() void {
    const requests = [_]Request{
        .{ .method = "GET", .path = "/", .body = "" },
        .{ .method = "GET", .path = "/admin", .body = "" },
        .{ .method = "GET", .path = "/unknown", .body = "" },
    };
    
    std.debug.print("=== Middleware Demo ===\n\n", .{});
    
    for (requests) |req| {
        std.debug.print("Request: {s} {s}\n", .{ req.method, req.path });
        
        // Logger → Auth → Route
        const after_log = loggerMiddleware(req, routeRequest);
        _ = after_log;
        std.debug.print("\n", .{});
    }
}
```

---

## ตัวอย่างโปรแกรมจริง: Expression Evaluator

```zig
const std = @import("std");

const TokenType = enum { number, plus, minus, multiply, divide, lparen, rparen, eof };

const Token = struct {
    kind: TokenType,
    value: f64,
};

const Lexer = struct {
    input: []const u8,
    pos: usize,
    
    fn init(input: []const u8) Lexer {
        return .{ .input = input, .pos = 0 };
    }
    
    fn skipWhitespace(self: *Lexer) void {
        while (self.pos < self.input.len and self.input[self.pos] == ' ') {
            self.pos += 1;
        }
    }
    
    fn nextToken(self: *Lexer) Token {
        self.skipWhitespace();
        
        if (self.pos >= self.input.len) {
            return .{ .kind = .eof, .value = 0 };
        }
        
        const ch = self.input[self.pos];
        
        if (ch >= '0' and ch <= '9') {
            var value: f64 = 0;
            while (self.pos < self.input.len and 
                   self.input[self.pos] >= '0' and 
                   self.input[self.pos] <= '9') {
                value = value * 10 + @as(f64, @floatFromInt(self.input[self.pos] - '0'));
                self.pos += 1;
            }
            return .{ .kind = .number, .value = value };
        }
        
        self.pos += 1;
        return switch (ch) {
            '+' => .{ .kind = .plus, .value = 0 },
            '-' => .{ .kind = .minus, .value = 0 },
            '*' => .{ .kind = .multiply, .value = 0 },
            '/' => .{ .kind = .divide, .value = 0 },
            '(' => .{ .kind = .lparen, .value = 0 },
            ')' => .{ .kind = .rparen, .value = 0 },
            else => .{ .kind = .eof, .value = 0 },
        };
    }
};

const Parser = struct {
    lexer: Lexer,
    current: Token,
    
    fn init(input: []const u8) Parser {
        var p = Parser{
            .lexer = Lexer.init(input),
            .current = undefined,
        };
        p.current = p.lexer.nextToken();
        return p;
    }
    
    fn eat(self: *Parser, kind: TokenType) void {
        if (self.current.kind == kind) {
            self.current = self.lexer.nextToken();
        }
    }
    
    fn factor(self: *Parser) f64 {
        const token = self.current;
        
        if (token.kind == .number) {
            self.eat(.number);
            return token.value;
        }
        
        if (token.kind == .lparen) {
            self.eat(.lparen);
            const result = self.expr();
            self.eat(.rparen);
            return result;
        }
        
        if (token.kind == .minus) {
            self.eat(.minus);
            return -self.factor();
        }
        
        return 0;
    }
    
    fn term(self: *Parser) f64 {
        var result = self.factor();
        
        while (self.current.kind == .multiply or self.current.kind == .divide) {
            if (self.current.kind == .multiply) {
                self.eat(.multiply);
                result *= self.factor();
            } else {
                self.eat(.divide);
                const divisor = self.factor();
                if (divisor != 0) result /= divisor;
            }
        }
        
        return result;
    }
    
    fn expr(self: *Parser) f64 {
        var result = self.term();
        
        while (self.current.kind == .plus or self.current.kind == .minus) {
            if (self.current.kind == .plus) {
                self.eat(.plus);
                result += self.term();
            } else {
                self.eat(.minus);
                result -= self.term();
            }
        }
        
        return result;
    }
};

fn evaluate(expression: []const u8) f64 {
    var parser = Parser.init(expression);
    return parser.expr();
}

pub fn main() void {
    const expressions = [_][]const u8{
        "2 + 3",
        "10 - 4 * 2",
        "(10 - 4) * 2",
        "100 / 4 + 5",
        "3 * (2 + 4) - 1",
    };
    
    std.debug.print("Expression Evaluator:\n", .{});
    for (expressions) |expr_str| {
        const result = evaluate(expr_str);
        std.debug.print("  {s} = {d}\n", .{ expr_str, result });
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Fibonacci ด้วย Memoization

เขียนฟังก์ชัน Fibonacci ที่ใช้ memoization เพื่อเพิ่มประสิทธิภาพ:

```zig
const std = @import("std");

// TODO: เขียนฟังก์ชัน fibonacci ที่ใช้ cache
fn fibonacci(n: u32, cache: []u64) u64 {
    if (n <= 1) return n;
    if (cache[n] != 0) return cache[n];
    
    const result = fibonacci(n - 1, cache) + fibonacci(n - 2, cache);
    cache[n] = result;
    return result;
}

pub fn main() void {
    var cache = [_]u64{0} ** 100;
    
    var i: u32 = 0;
    while (i <= 30) : (i += 1) {
        std.debug.print("fib({d}) = {d}\n", .{ i, fibonacci(i, &cache) });
    }
}
```

### แบบฝึกหัดที่ 2: Generic Queue

เขียน generic queue ด้วย comptime type:

```zig
const std = @import("std");

// TODO: สร้าง Queue(T) ที่ใช้ circular buffer
fn Queue(comptime T: type, comptime capacity: usize) type {
    return struct {
        buffer: [capacity]T,
        head: usize,
        tail: usize,
        count: usize,
        
        const Self = @This();
        
        // TODO: implement enqueue, dequeue, peek, isEmpty, isFull
        fn init() Self {
            return .{ .buffer = undefined, .head = 0, .tail = 0, .count = 0 };
        }
        
        fn enqueue(self: *Self, item: T) !void {
            if (self.count >= capacity) return error.QueueFull;
            self.buffer[self.tail] = item;
            self.tail = (self.tail + 1) % capacity;
            self.count += 1;
        }
        
        fn dequeue(self: *Self) !T {
            if (self.count == 0) return error.QueueEmpty;
            const item = self.buffer[self.head];
            self.head = (self.head + 1) % capacity;
            self.count -= 1;
            return item;
        }
        
        fn isEmpty(self: *const Self) bool {
            return self.count == 0;
        }
    };
}

pub fn main() !void {
    var q = Queue(i32, 5).init();
    
    try q.enqueue(10);
    try q.enqueue(20);
    try q.enqueue(30);
    
    while (!q.isEmpty()) {
        const val = try q.dequeue();
        std.debug.print("dequeue: {d}\n", .{val});
    }
}
```

### แบบฝึกหัดที่ 3: Calculator ด้วย Function Pointers

```zig
const std = @import("std");

// TODO: สร้าง calculator ที่รองรับ operations:
// +, -, *, /, %, pow, sqrt

const Operation = struct {
    name: []const u8,
    func: *const fn(f64, f64) f64,
    unary: bool, // true ถ้าใช้แค่ argument เดียว
};

fn op_add(a: f64, b: f64) f64 { return a + b; }
fn op_sub(a: f64, b: f64) f64 { return a - b; }
fn op_mul(a: f64, b: f64) f64 { return a * b; }
fn op_div(a: f64, b: f64) f64 { 
    if (b == 0.0) return 0.0;
    return a / b; 
}
fn op_mod(a: f64, b: f64) f64 { return @mod(a, b); }
fn op_pow(a: f64, b: f64) f64 { return std.math.pow(f64, a, b); }
fn op_sqrt(a: f64, _: f64) f64 { return @sqrt(a); }

pub fn main() void {
    const operations = [_]Operation{
        .{ .name = "+", .func = op_add, .unary = false },
        .{ .name = "-", .func = op_sub, .unary = false },
        .{ .name = "*", .func = op_mul, .unary = false },
        .{ .name = "/", .func = op_div, .unary = false },
        .{ .name = "%", .func = op_mod, .unary = false },
        .{ .name = "pow", .func = op_pow, .unary = false },
        .{ .name = "sqrt", .func = op_sqrt, .unary = true },
    };
    
    const a: f64 = 16.0;
    const b: f64 = 4.0;
    
    std.debug.print("a = {d}, b = {d}\n\n", .{ a, b });
    
    for (operations) |op| {
        if (op.unary) {
            std.debug.print("{s}({d}) = {d:.4}\n", .{ op.name, a, op.func(a, 0) });
        } else {
            std.debug.print("{d} {s} {d} = {d:.4}\n", .{ a, op.name, b, op.func(a, b) });
        }
    }
}
```

### แบบฝึกหัดที่ 4: Decorator Pattern ด้วย Comptime

```zig
const std = @import("std");

// TODO: สร้าง decorator สำหรับ memoize ฟังก์ชัน
fn memoize(comptime T: type, comptime U: type, comptime func: fn(T) U) type {
    return struct {
        cache: std.AutoHashMap(T, U),
        
        const Self = @This();
        
        fn init(allocator: std.mem.Allocator) Self {
            return .{ .cache = std.AutoHashMap(T, U).init(allocator) };
        }
        
        fn deinit(self: *Self) void {
            self.cache.deinit();
        }
        
        fn call(self: *Self, input: T) !U {
            if (self.cache.get(input)) |cached| {
                return cached;
            }
            const result = func(input);
            try self.cache.put(input, result);
            return result;
        }
    };
}

fn expensiveComputation(n: u32) u64 {
    // จำลองการคำนวณที่ใช้เวลานาน
    var result: u64 = 0;
    var i: u32 = 0;
    while (i <= n) : (i += 1) {
        result += i;
    }
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const MemoizedFn = memoize(u32, u64, expensiveComputation);
    var memo = MemoizedFn.init(allocator);
    defer memo.deinit();
    
    // เรียกหลายครั้ง - ครั้งที่ 2+ จะใช้ cache
    const inputs = [_]u32{ 100, 200, 100, 300, 200 };
    for (inputs) |n| {
        const result = try memo.call(n);
        std.debug.print("computation({d}) = {d}\n", .{ n, result });
    }
}
```

---

## สรุป

ในภาคนี้เราได้เรียนรู้เกี่ยวกับฟังก์ชันใน Zig ครอบคลุมทุกแง่มุมสำคัญ:

### สิ่งที่ได้เรียนรู้

| Step | หัวข้อ | สาระสำคัญ |
|------|--------|-----------|
| 71 | การประกาศและเรียกใช้ฟังก์ชัน | `fn`, `pub fn`, `noreturn`, ลำดับการประกาศ |
| 72 | Parameters และ Return Values | pass-by-value, pass-by-pointer, return types ต่างๆ |
| 73 | Multiple Return Values | anonymous struct, named struct สำหรับ return หลายค่า |
| 74 | Error Handling | error set, `!T`, `try`, `catch`, error propagation |
| 75 | Recursive Functions | base case, recursive case, tail recursion |
| 76 | Function Pointers | `*const fn(...)`, strategy pattern, event system |
| 77 | Anonymous Functions | anonymous struct functions, higher-order functions |
| 78 | Comptime Functions | generic code, type introspection, compile-time checks |
| 79 | Inline Functions | `inline fn`, performance optimization, `inline for` |
| 80 | anytype | generic parameters, varargs simulation, type-safe code |

### หลักการสำคัญ

**1. Functions เป็น First-Class Citizens**
ใน Zig ฟังก์ชันสามารถเก็บในตัวแปร ส่งเป็น parameter และคืนเป็นค่าได้ผ่าน function pointers

**2. Comptime สำหรับ Zero-Cost Abstractions**
`comptime` ช่วยสร้าง generic code ที่ compile-time ทำให้ไม่มี runtime overhead

**3. Error Safety**
ระบบ error union ของ Zig บังคับให้จัดการ error ทุกกรณี ป้องกัน unhandled errors

**4. anytype สำหรับ Generic Code**
`anytype` ช่วยสร้างฟังก์ชันที่ทำงานได้กับหลายชนิดข้อมูล โดย compiler สร้าง specialized version ให้อัตโนมัติ

**5. Inline สำหรับ Performance**
`inline fn` ช่วยลด function call overhead สำหรับฟังก์ชันเล็กที่เรียกบ่อย

### รูปแบบที่ควรจำ

```zig
// 1. Error handling
fn riskyOp() !ResultType {
    const value = try someCall();
    return value;
}

// 2. Generic function
fn genericFn(comptime T: type, value: T) T { ... }

// 3. Function pointer
const FnPtr = *const fn(i32) i32;
fn apply(f: FnPtr, x: i32) i32 { return f(x); }

// 4. anytype parameter
fn log(comptime fmt: []const u8, args: anytype) void {
    std.debug.print(fmt, args);
}

// 5. Multiple return
fn twoValues() struct { a: i32, b: i32 } {
    return .{ .a = 1, .b = 2 };
}
```

### ภาคต่อไป

ในภาคที่ 9 เราจะเรียนรู้เกี่ยวกับ **Structs และ Methods** ซึ่งจะนำความรู้เรื่องฟังก์ชันมาประยุกต์ใช้กับ Object-Oriented-like programming ใน Zig รวมถึง:
- การกำหนด method บน struct
- Interface ผ่าน vtable
- Composition แทน Inheritance
- Generic structs ด้วย comptime
