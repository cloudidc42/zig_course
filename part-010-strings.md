# ภาคที่ 10: Strings และการจัดการข้อความใน Zig
## Steps 91-100

---

ยินดีต้อนรับสู่ภาคที่ 10 ของคอร์ส Zig! ในภาคนี้เราจะเรียนรู้เรื่อง **Strings** หรือข้อความใน Zig ซึ่งเป็นหนึ่งในหัวข้อที่มีความเป็นเอกลักษณ์มากที่สุดของภาษานี้ Zig ไม่มี built-in string type เหมือนภาษาอื่น แต่ใช้ byte slice (`[]const u8`) แทน ซึ่งทำให้เข้าใจการทำงานกับหน่วยความจำได้อย่างลึกซึ้งยิ่งขึ้น

---

## Step 91: Strings ใน Zig คืออะไร? ([]const u8)

### ความเข้าใจพื้นฐาน

ใน Zig **ไม่มี** `string` type เหมือนใน Python, Java, หรือ Rust ในทางกลับกัน Zig ใช้ **slice of bytes** (`[]const u8`) ในการแทนข้อความ

- `u8` คือ unsigned 8-bit integer (ค่าตั้งแต่ 0 ถึง 255)
- `[]const u8` คือ slice ที่ชี้ไปยังชุดของ bytes ที่ไม่สามารถเปลี่ยนแปลงได้
- ทุก string literal ใน Zig เป็น null-terminated อยู่ใน memory แต่เมื่อใช้เป็น slice จะไม่รวม null byte

### ประเภทของ String ใน Zig

```zig
const std = @import("std");

pub fn main() void {
    // 1. String literal - type เป็น *const [N:0]u8 (null-terminated array pointer)
    const str_literal = "สวัสดีโลก";

    // 2. Slice - type เป็น []const u8
    const str_slice: []const u8 = "Hello, World!";

    // 3. Array of bytes
    const str_array: [5]u8 = .{ 'H', 'e', 'l', 'l', 'o' };

    // 4. Mutable string (ต้องใช้ []u8)
    var mutable_str: [10]u8 = undefined;
    mutable_str[0] = 'Z';
    mutable_str[1] = 'i';
    mutable_str[2] = 'g';

    std.debug.print("String literal: {s}\n", .{str_literal});
    std.debug.print("String slice: {s}\n", .{str_slice});
    std.debug.print("Array ส่วนแรก: {c}\n", .{str_array[0]});
    std.debug.print("Mutable string: {s}\n", .{mutable_str[0..3]});
}
```

### ความแตกต่างระหว่าง *const [N:0]u8 และ []const u8

```zig
const std = @import("std");

pub fn main() void {
    // String literal มี type เป็น *const [N:0]u8
    // N คือความยาวของ string
    // :0 หมายถึง null-terminated
    const literal = "Zig";
    
    // เราสามารถดู type ได้ด้วย @TypeOf
    std.debug.print("Type of literal: {}\n", .{@TypeOf(literal)});
    // จะได้: *const [3:0]u8

    // Slice ไม่รวม null terminator ในความยาว
    const slice: []const u8 = "Zig";
    std.debug.print("Type of slice: {}\n", .{@TypeOf(slice)});
    // จะได้: []const u8

    // ความยาวของ string
    std.debug.print("Literal length: {}\n", .{literal.len});  // 3
    std.debug.print("Slice length: {}\n", .{slice.len});      // 3

    // String literal สามารถ coerce เป็น slice ได้อัตโนมัติ
    const auto_slice: []const u8 = literal;
    std.debug.print("Auto slice: {s}\n", .{auto_slice});
}
```

### โครงสร้างของ String ใน Memory

```zig
const std = @import("std");

pub fn main() void {
    const greeting = "Hello";
    
    // String เก็บเป็น bytes ใน memory
    // 'H' = 72, 'e' = 101, 'l' = 108, 'l' = 108, 'o' = 111, '\0' = 0
    
    std.debug.print("String: {s}\n", .{greeting});
    std.debug.print("Length: {}\n", .{greeting.len});
    
    // ดู bytes แต่ละตัว
    std.debug.print("Bytes: ", .{});
    for (greeting) |byte| {
        std.debug.print("{} ", .{byte});
    }
    std.debug.print("\n", .{});
    
    // เข้าถึง byte แต่ละตัว
    std.debug.print("First byte: {} ({c})\n", .{ greeting[0], greeting[0] });
    std.debug.print("Last byte: {} ({c})\n", .{ greeting[greeting.len - 1], greeting[greeting.len - 1] });
    
    // ตรวจสอบ null terminator (ต้องใช้ pointer ไม่ใช่ slice)
    // null terminator อยู่ที่ index greeting.len
    std.debug.print("Null terminator: {}\n", .{greeting[greeting.len]});
}
```

### ทำไม Zig ถึงไม่มี String Type?

Zig เลือกแนวทางนี้เพราะ:

1. **ความโปร่งใส (Transparency)**: คุณรู้แน่ชัดว่า string คืออะไรในระดับ memory
2. **ประสิทธิภาพ**: ไม่มี overhead จาก abstraction
3. **ความยืดหยุ่น**: สามารถใช้กับ encoding ใดก็ได้ (ASCII, UTF-8, UTF-16)
4. **ปลอดภัย**: การจัดการ memory ชัดเจนและตรวจสอบได้

---

## Step 92: String Literals และ Escape Sequences

### String Literals พื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    // String literals ธรรมดา
    const name = "Zig Programming";
    const version = "0.13.0";
    const empty = "";
    
    std.debug.print("Name: {s}\n", .{name});
    std.debug.print("Version: {s}\n", .{version});
    std.debug.print("Empty string length: {}\n", .{empty.len});
    
    // Single character literal ใช้ single quote
    const char: u8 = 'A';
    std.debug.print("Char: {c} (value: {})\n", .{ char, char });
    
    // Unicode character literal
    const thai_char: u21 = 'ก'; // u21 เพราะ Unicode codepoint ต้องการ 21 bits
    std.debug.print("Thai char codepoint: U+{X:0>4}\n", .{thai_char});
}
```

### Escape Sequences ทั้งหมด

```zig
const std = @import("std");

pub fn main() void {
    // Escape sequences ใน Zig
    
    // \n - newline (line feed)
    std.debug.print("Line 1\nLine 2\n", .{});
    
    // \r - carriage return
    std.debug.print("With CR\r\n", .{});
    
    // \t - horizontal tab
    std.debug.print("Column1\tColumn2\tColumn3\n", .{});
    
    // \\ - backslash literal
    std.debug.print("Path: C:\\Users\\Username\\\n", .{});
    
    // \" - double quote
    std.debug.print("He said \"Hello!\"\n", .{});
    
    // \' - single quote (ใช้ใน character literals)
    const quote_char: u8 = '\'';
    std.debug.print("Quote char: {c}\n", .{quote_char});
    
    // \xNN - hexadecimal byte value (NN คือ hex digits)
    const newline_hex = "\x0A"; // 0x0A = 10 = '\n'
    std.debug.print("Hex newline:{s}", .{newline_hex});
    
    // \u{NNNNNN} - Unicode codepoint (ใน string literals)
    const unicode_str = "\u{0E2A}\u{0E27}\u{0E31}\u{0E2A}\u{0E14}\u{0E35}"; // สวัสดี
    std.debug.print("Unicode string: {s}\n", .{unicode_str});
    
    // \0 - null byte
    const with_null = "before\x00after";
    std.debug.print("String with null - length: {}\n", .{with_null.len}); // 12
    
    // \a - alert/bell
    // \b - backspace  
    // \f - form feed
    // \v - vertical tab
    const special = "\a\b\f\v";
    std.debug.print("Special chars length: {}\n", .{special.len}); // 4
}
```

### Character Escape ใน Character Literals

```zig
const std = @import("std");

pub fn main() void {
    const tab: u8 = '\t';
    const newline: u8 = '\n';
    const backslash: u8 = '\\';
    const null_byte: u8 = '\x00';
    
    std.debug.print("Tab value: {}\n", .{tab});       // 9
    std.debug.print("Newline value: {}\n", .{newline}); // 10
    std.debug.print("Backslash value: {}\n", .{backslash}); // 92
    std.debug.print("Null value: {}\n", .{null_byte}); // 0
    
    // เปรียบเทียบค่า ASCII
    const upper_a: u8 = 'A'; // 65
    const lower_a: u8 = 'a'; // 97
    const zero: u8 = '0';    // 48
    
    std.debug.print("'A' = {}, 'a' = {}, '0' = {}\n", .{ upper_a, lower_a, zero });
    
    // ความสัมพันธ์ระหว่าง upper และ lower case
    std.debug.print("Difference: {}\n", .{lower_a - upper_a}); // 32
}
```

### Format Specifiers สำหรับ String

```zig
const std = @import("std");

pub fn main() void {
    const greeting = "สวัสดี";
    const number: u8 = 65;
    
    // {s} - แสดง string
    std.debug.print("{s}\n", .{greeting});
    
    // {c} - แสดง character จาก byte value
    std.debug.print("{c}\n", .{number}); // A
    
    // {d} - แสดง integer เป็น decimal
    std.debug.print("{d}\n", .{number}); // 65
    
    // {x} - แสดง integer เป็น hex (lowercase)
    std.debug.print("{x}\n", .{number}); // 41
    
    // {X} - แสดง integer เป็น hex (uppercase)
    std.debug.print("{X}\n", .{number}); // 41
    
    // {b} - แสดง integer เป็น binary
    std.debug.print("{b}\n", .{number}); // 1000001
    
    // {o} - แสดง integer เป็น octal
    std.debug.print("{o}\n", .{number}); // 101
    
    // Width และ alignment
    std.debug.print("{s:>20}\n", .{greeting}); // right-align
    std.debug.print("{s:<20}\n", .{greeting}); // left-align
    std.debug.print("{s:^20}\n", .{greeting}); // center-align
    
    // Fill character
    std.debug.print("{s:*>20}\n", .{"Zig"}); // ***************Zig
    std.debug.print("{s:-<20}\n", .{"Zig"}); // Zig-----------------
}
```

---

## Step 93: Multiline Strings

### Multiline String Syntax

Zig มี syntax พิเศษสำหรับ multiline strings โดยใช้ `\\` ที่ต้นบรรทัด

```zig
const std = @import("std");

pub fn main() void {
    // Multiline string ใช้ \\ นำหน้าแต่ละบรรทัด
    const poem =
        \\ใน Zig เราเขียนโค้ดที่เข้าใจ
        \\ทุกบรรทัดมีความหมาย
        \\ไม่มีการซ่อนเร้น
        \\ทุกอย่างโปร่งใส
    ;
    
    std.debug.print("{s}\n", .{poem});
    
    // Multiline string ไม่รวม leading backslash-backslash
    // แต่รวม newlines ระหว่างบรรทัด
    const code_example =
        \\const x = 42;
        \\const y = x * 2;
        \\std.debug.print("{}", .{y});
    ;
    
    std.debug.print("Code example:\n{s}\n", .{code_example});
}
```

### การใช้ Multiline Strings สำหรับ JSON, HTML, SQL

```zig
const std = @import("std");

pub fn main() void {
    // JSON
    const json_template =
        \\{
        \\  "name": "Zig",
        \\  "version": "0.13.0",
        \\  "features": [
        \\    "fast",
        \\    "safe",
        \\    "explicit"
        \\  ]
        \\}
    ;
    
    std.debug.print("JSON:\n{s}\n", .{json_template});
    
    // HTML
    const html_template =
        \\<!DOCTYPE html>
        \\<html>
        \\<head>
        \\  <title>Zig Course</title>
        \\</head>
        \\<body>
        \\  <h1>สวัสดีจาก Zig!</h1>
        \\</body>
        \\</html>
    ;
    
    std.debug.print("\nHTML:\n{s}\n", .{html_template});
    
    // SQL
    const sql_query =
        \\SELECT u.name, u.email, o.total
        \\FROM users u
        \\JOIN orders o ON u.id = o.user_id
        \\WHERE o.status = 'completed'
        \\ORDER BY o.total DESC
        \\LIMIT 10
    ;
    
    std.debug.print("\nSQL:\n{s}\n", .{sql_query});
}
```

### ความแตกต่างระหว่าง Multiline String กับ Regular String

```zig
const std = @import("std");

pub fn main() void {
    // Regular string ที่มี \n
    const regular = "Line 1\nLine 2\nLine 3";
    
    // Multiline string
    const multiline =
        \\Line 1
        \\Line 2
        \\Line 3
    ;
    
    // ทั้งสองมีค่าเท่ากัน!
    std.debug.print("Regular length: {}\n", .{regular.len});
    std.debug.print("Multiline length: {}\n", .{multiline.len});
    
    // เปรียบเทียบ
    if (std.mem.eql(u8, regular, multiline)) {
        std.debug.print("Both strings are equal!\n", .{});
    }
    
    // Multiline string อ่านง่ายกว่าสำหรับ content ที่มีหลายบรรทัด
    const help_text =
        \\Usage: myapp [options] <input>
        \\
        \\Options:
        \\  -h, --help     Show this help message
        \\  -v, --verbose  Enable verbose output
        \\  -o, --output   Output file path
        \\
        \\Examples:
        \\  myapp input.txt
        \\  myapp -v -o output.txt input.txt
    ;
    
    std.debug.print("\n{s}\n", .{help_text});
}
```

### Multiline String กับ Indentation

```zig
const std = @import("std");

pub fn main() void {
    // Multiline string เริ่มต้นหลัง \\ เสมอ
    // ดังนั้น indentation ใน source code ไม่กระทบ content
    
    const indented_code =
        \\fn hello() void {
        \\    const msg = "สวัสดี";
        \\    std.debug.print("{s}\n", .{msg});
        \\}
    ;
    
    std.debug.print("{s}\n", .{indented_code});
    
    // ข้อควรระวัง: บรรทัดว่าง
    const with_empty_line =
        \\First paragraph
        \\
        \\Second paragraph after empty line
    ;
    
    std.debug.print("{s}\n", .{with_empty_line});
    
    // การนับบรรทัด
    var line_count: usize = 0;
    for (with_empty_line) |byte| {
        if (byte == '\n') line_count += 1;
    }
    std.debug.print("Number of newlines: {}\n", .{line_count});
}
```

---

## Step 94: String Comparison (std.mem.eql)

### การเปรียบเทียบ String

ใน Zig เราไม่สามารถใช้ `==` เปรียบเทียบ slices ได้โดยตรง ต้องใช้ `std.mem.eql`

```zig
const std = @import("std");

pub fn main() void {
    const str1 = "Hello";
    const str2 = "Hello";
    const str3 = "World";
    const str4 = "hello"; // lowercase
    
    // ใช้ std.mem.eql สำหรับ string comparison
    std.debug.print("str1 == str2: {}\n", .{std.mem.eql(u8, str1, str2)}); // true
    std.debug.print("str1 == str3: {}\n", .{std.mem.eql(u8, str1, str3)}); // false
    std.debug.print("str1 == str4: {}\n", .{std.mem.eql(u8, str1, str4)}); // false (case-sensitive)
    
    // การเปรียบเทียบ pointer (ไม่ใช่ content comparison!)
    // สองตัวแปรที่ชี้ไปยัง string literal เดียวกันอาจมี pointer เดียวกัน
    // แต่ไม่ควรพึ่งพา behavior นี้
    if (str1.ptr == str2.ptr) {
        std.debug.print("str1 and str2 point to same memory\n", .{});
    }
    
    // ตัวอย่างจริง: เปรียบเทียบ username
    const input_user = "admin";
    const valid_user = "admin";
    
    if (std.mem.eql(u8, input_user, valid_user)) {
        std.debug.print("User authenticated!\n", .{});
    } else {
        std.debug.print("Authentication failed!\n", .{});
    }
}
```

### Case-Insensitive Comparison

```zig
const std = @import("std");
const ascii = std.ascii;

pub fn main() void {
    const str1 = "Hello World";
    const str2 = "HELLO WORLD";
    const str3 = "hello world";
    
    // std.ascii.eqlIgnoreCase สำหรับ ASCII case-insensitive comparison
    std.debug.print("eqlIgnoreCase(str1, str2): {}\n", 
        .{ascii.eqlIgnoreCase(str1, str2)}); // true
    std.debug.print("eqlIgnoreCase(str1, str3): {}\n", 
        .{ascii.eqlIgnoreCase(str1, str3)}); // true
    
    // ภาษาไทยไม่มี case ดังนั้น eql ปกติก็พอ
    const thai1 = "สวัสดี";
    const thai2 = "สวัสดี";
    std.debug.print("Thai comparison: {}\n", .{std.mem.eql(u8, thai1, thai2)}); // true
}
```

### std.mem.order - การเปรียบเทียบเชิงลำดับ

```zig
const std = @import("std");

pub fn main() void {
    const a = "apple";
    const b = "banana";
    const c = "apple";
    
    // std.mem.order คืนค่า std.math.Order (.lt, .eq, .gt)
    const order_ab = std.mem.order(u8, a, b);
    const order_ba = std.mem.order(u8, b, a);
    const order_ac = std.mem.order(u8, a, c);
    
    switch (order_ab) {
        .lt => std.debug.print("'{s}' < '{s}'\n", .{ a, b }),
        .eq => std.debug.print("'{s}' == '{s}'\n", .{ a, b }),
        .gt => std.debug.print("'{s}' > '{s}'\n", .{ a, b }),
    }
    
    switch (order_ba) {
        .lt => std.debug.print("'{s}' < '{s}'\n", .{ b, a }),
        .eq => std.debug.print("'{s}' == '{s}'\n", .{ b, a }),
        .gt => std.debug.print("'{s}' > '{s}'\n", .{ b, a }),
    }
    
    switch (order_ac) {
        .lt => std.debug.print("'{s}' < '{s}'\n", .{ a, c }),
        .eq => std.debug.print("'{s}' == '{s}'\n", .{ a, c }),
        .gt => std.debug.print("'{s}' > '{s}'\n", .{ a, c }),
    }
    
    // Sorting strings (conceptual example)
    const fruits = [_][]const u8{ "mango", "apple", "banana", "cherry" };
    std.debug.print("\nOriginal order:\n", .{});
    for (fruits) |fruit| {
        std.debug.print("  {s}\n", .{fruit});
    }
}
```

### การเปรียบเทียบ String ส่วนหนึ่ง

```zig
const std = @import("std");

pub fn main() void {
    const full_path = "/home/user/documents/report.txt";
    const prefix = "/home/user";
    const suffix = ".txt";
    
    // เช็คว่าขึ้นต้นด้วย prefix หรือไม่
    const starts_with_home = std.mem.startsWith(u8, full_path, prefix);
    std.debug.print("Starts with '/home/user': {}\n", .{starts_with_home});
    
    // เช็คว่าลงท้ายด้วย suffix หรือไม่
    const ends_with_txt = std.mem.endsWith(u8, full_path, suffix);
    std.debug.print("Ends with '.txt': {}\n", .{ends_with_txt});
    
    // เปรียบเทียบ substring โดยใช้ slice
    const filename = full_path[full_path.len - 10..]; // "report.txt"
    std.debug.print("Filename: {s}\n", .{filename});
    std.debug.print("Is 'report.txt': {}\n", .{std.mem.eql(u8, filename, "report.txt")});
}
```

---

## Step 95: String Concatenation (++, std.mem.concat)

### Compile-time Concatenation ด้วย ++

```zig
const std = @import("std");

pub fn main() void {
    // ++ operator ทำงานเฉพาะ compile-time เท่านั้น
    const first = "Hello";
    const second = ", ";
    const third = "World!";
    
    // concatenate ณ compile time
    const greeting = first ++ second ++ third;
    std.debug.print("{s}\n", .{greeting}); // Hello, World!
    
    // ใช้กับ comptime string
    const base_url = "https://example.com";
    const api_path = "/api/v1";
    const users_endpoint = base_url ++ api_path ++ "/users";
    std.debug.print("Endpoint: {s}\n", .{users_endpoint});
    
    // ++ ยังใช้กับ array ได้
    const arr1 = [_]u8{ 1, 2, 3 };
    const arr2 = [_]u8{ 4, 5, 6 };
    const combined = arr1 ++ arr2;
    std.debug.print("Combined array length: {}\n", .{combined.len}); // 6
    
    // String กับ numeric literal
    const prefix = "Version: ";
    const version_num = std.fmt.comptimePrint("{d}", .{13}); // ต้องใช้ comptimePrint
    const version_string = prefix ++ version_num;
    std.debug.print("{s}\n", .{version_string}); // Version: 13
}
```

### Runtime Concatenation ด้วย std.mem.concat

```zig
const std = @import("std");

pub fn main() !void {
    // สำหรับ runtime concatenation ต้องใช้ allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const first_name = "สมชาย";
    const last_name = "ใจดี";
    
    // std.mem.concat รับ slice of slices และ concatenate
    const full_name = try std.mem.concat(allocator, u8, &.{ first_name, " ", last_name });
    defer allocator.free(full_name);
    
    std.debug.print("Full name: {s}\n", .{full_name});
    std.debug.print("Length: {}\n", .{full_name.len});
    
    // concatenate หลาย strings
    const parts = [_][]const u8{
        "Zig ", "is ", "a ", "great ", "programming ", "language!"
    };
    const sentence = try std.mem.concat(allocator, u8, &parts);
    defer allocator.free(sentence);
    
    std.debug.print("{s}\n", .{sentence});
}
```

### การใช้ ArrayList สำหรับ String Building

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ArrayList(u8) เป็นวิธีที่ยืดหยุ่นสำหรับการสร้าง string
    var builder = std.ArrayList(u8).init(allocator);
    defer builder.deinit();
    
    // เพิ่มข้อความทีละส่วน
    try builder.appendSlice("สวัสดี");
    try builder.appendSlice(", ");
    try builder.appendSlice("ยินดีต้อนรับ");
    try builder.appendSlice("สู่ Zig!");
    
    const result = builder.items;
    std.debug.print("{s}\n", .{result});
    
    // สร้าง string จาก loop
    var number_list = std.ArrayList(u8).init(allocator);
    defer number_list.deinit();
    
    var i: u32 = 1;
    while (i <= 5) : (i += 1) {
        if (i > 1) try number_list.appendSlice(", ");
        try std.fmt.format(number_list.writer(), "{}", .{i});
    }
    
    std.debug.print("Numbers: {s}\n", .{number_list.items}); // 1, 2, 3, 4, 5
    
    // ใช้ writer() กับ print
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    const writer = output.writer();
    try writer.print("Name: {s}, Age: {}, City: {s}", .{ "สมหญิง", 25, "กรุงเทพ" });
    
    std.debug.print("{s}\n", .{output.items});
}
```

### std.fmt.allocPrint สำหรับ Format และ Concatenate

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // allocPrint ทำ format และ allocate memory ในคราวเดียว
    const msg = try std.fmt.allocPrint(allocator, "Hello, {s}! You are {d} years old.", .{
        "สมบัติ", 30
    });
    defer allocator.free(msg);
    
    std.debug.print("{s}\n", .{msg});
    
    // ใช้สำหรับสร้าง path
    const dir = "/home/user";
    const filename = "config.json";
    const full_path = try std.fmt.allocPrint(allocator, "{s}/{s}", .{ dir, filename });
    defer allocator.free(full_path);
    
    std.debug.print("Path: {s}\n", .{full_path});
    
    // สร้าง HTML
    const title = "Zig Programming";
    const body = "เรียน Zig กันเถอะ!";
    const html = try std.fmt.allocPrint(allocator,
        \\<html>
        \\<head><title>{s}</title></head>
        \\<body><p>{s}</p></body>
        \\</html>
    , .{ title, body });
    defer allocator.free(html);
    
    std.debug.print("{s}\n", .{html});
}
```

---

## Step 96: String Slicing และ Substring

### การ Slice String

```zig
const std = @import("std");

pub fn main() void {
    const text = "Hello, World! How are you?";
    
    // Slicing ใช้ syntax: slice[start..end]
    // start คือ index เริ่มต้น (inclusive)
    // end คือ index สิ้นสุด (exclusive)
    
    const hello = text[0..5];
    const world = text[7..12];
    const question = text[14..];  // จนถึงท้าย
    const start = text[..7];       // จากต้นถึง index 7 (ไม่รวม)
    
    std.debug.print("Hello: {s}\n", .{hello});     // Hello
    std.debug.print("World: {s}\n", .{world});     // World
    std.debug.print("Question: {s}\n", .{question}); // How are you?
    std.debug.print("Start: {s}\n", .{start});     // Hello, 
    
    // ความยาว
    std.debug.print("Full length: {}\n", .{text.len});
    std.debug.print("Hello length: {}\n", .{hello.len}); // 5
    
    // Slice ยังคง reference ไปยัง original string
    // ไม่ได้ copy data
    std.debug.print("Same memory: {}\n", .{hello.ptr == text.ptr});
}
```

### การหา Substring

```zig
const std = @import("std");

pub fn main() void {
    const haystack = "The quick brown fox jumps over the lazy dog";
    const needle = "fox";
    
    // std.mem.indexOf หา position ของ substring
    if (std.mem.indexOf(u8, haystack, needle)) |pos| {
        std.debug.print("Found '{s}' at position {}\n", .{ needle, pos });
        
        // Extract substring
        const found = haystack[pos..pos + needle.len];
        std.debug.print("Extracted: {s}\n", .{found});
    } else {
        std.debug.print("'{s}' not found\n", .{needle});
    }
    
    // std.mem.lastIndexOf - หาจากท้าย
    const text = "cat and cat and cat";
    if (std.mem.lastIndexOf(u8, text, "cat")) |pos| {
        std.debug.print("Last 'cat' at position {}\n", .{pos});
    }
    
    // std.mem.indexOfScalar - หา single byte
    const csv = "name,age,city";
    if (std.mem.indexOfScalar(u8, csv, ',')) |pos| {
        const first_field = csv[0..pos];
        std.debug.print("First field: {s}\n", .{first_field});
    }
}
```

### การตัด String (Trim)

```zig
const std = @import("std");

pub fn main() void {
    const padded = "   Hello, World!   ";
    
    // ตัด whitespace ทั้งสองด้าน
    const trimmed = std.mem.trim(u8, padded, " \t\n\r");
    std.debug.print("Trimmed: '{s}'\n", .{trimmed});
    
    // ตัดซ้าย
    const left_trimmed = std.mem.trimLeft(u8, padded, " ");
    std.debug.print("Left trimmed: '{s}'\n", .{left_trimmed});
    
    // ตัดขวา
    const right_trimmed = std.mem.trimRight(u8, padded, " ");
    std.debug.print("Right trimmed: '{s}'\n", .{right_trimmed});
    
    // ตัด characters อื่นๆ
    const with_stars = "***important***";
    const no_stars = std.mem.trim(u8, with_stars, "*");
    std.debug.print("No stars: '{s}'\n", .{no_stars}); // important
    
    // ตัด newline จากท้าย (common use case)
    const with_newline = "Hello\n";
    const no_newline = std.mem.trimRight(u8, with_newline, "\n");
    std.debug.print("No newline: '{s}'\n", .{no_newline}); // Hello
}
```

### Substring Operations

```zig
const std = @import("std");

// ฟังก์ชันหา substring ระหว่างสอง markers
fn extractBetween(str: []const u8, start_marker: []const u8, end_marker: []const u8) ?[]const u8 {
    const start_pos = std.mem.indexOf(u8, str, start_marker) orelse return null;
    const content_start = start_pos + start_marker.len;
    
    const remaining = str[content_start..];
    const end_pos = std.mem.indexOf(u8, remaining, end_marker) orelse return null;
    
    return remaining[0..end_pos];
}

pub fn main() void {
    const html = "<title>Zig Programming Course</title>";
    
    if (extractBetween(html, "<title>", "</title>")) |title| {
        std.debug.print("Title: {s}\n", .{title});
    }
    
    const template = "Hello, [NAME]! Welcome to [PLACE].";
    
    if (extractBetween(template, "[", "]")) |placeholder| {
        std.debug.print("First placeholder: {s}\n", .{placeholder}); // NAME
    }
    
    // การแบ่ง string ด้วย index
    const date = "2024-01-15";
    const year = date[0..4];
    const month = date[5..7];
    const day = date[8..10];
    
    std.debug.print("Year: {s}, Month: {s}, Day: {s}\n", .{ year, month, day });
}
```

---

## Step 97: String to Int/Float Conversion

### การแปลง String เป็น Integer

```zig
const std = @import("std");

pub fn main() !void {
    // std.fmt.parseInt สำหรับแปลง string เป็น integer
    const num_str = "42";
    const num = try std.fmt.parseInt(i32, num_str, 10); // base 10
    std.debug.print("Parsed: {}\n", .{num}); // 42
    
    // แปลงเลขฐานต่างๆ
    const hex_str = "FF";
    const hex_num = try std.fmt.parseInt(u8, hex_str, 16); // base 16
    std.debug.print("Hex FF = {}\n", .{hex_num}); // 255
    
    const binary_str = "1010";
    const binary_num = try std.fmt.parseInt(u8, binary_str, 2); // base 2
    std.debug.print("Binary 1010 = {}\n", .{binary_num}); // 10
    
    const octal_str = "17";
    const octal_num = try std.fmt.parseInt(u8, octal_str, 8); // base 8
    std.debug.print("Octal 17 = {}\n", .{octal_num}); // 15
    
    // negative numbers
    const neg_str = "-100";
    const neg_num = try std.fmt.parseInt(i32, neg_str, 10);
    std.debug.print("Negative: {}\n", .{neg_num}); // -100
    
    // ใช้ base 0 เพื่อ auto-detect (0x = hex, 0b = binary, 0o = octal)
    const auto_hex = try std.fmt.parseInt(u32, "0xFF", 0);
    const auto_bin = try std.fmt.parseInt(u32, "0b1010", 0);
    std.debug.print("Auto hex: {}, Auto binary: {}\n", .{ auto_hex, auto_bin });
}
```

### การจัดการ Error ใน parseInt

```zig
const std = @import("std");

pub fn parseIntSafe(str: []const u8) i32 {
    return std.fmt.parseInt(i32, str, 10) catch |err| {
        switch (err) {
            error.InvalidCharacter => {
                std.debug.print("Error: Invalid character in '{s}'\n", .{str});
                return 0;
            },
            error.Overflow => {
                std.debug.print("Error: Number overflow for '{s}'\n", .{str});
                return 0;
            },
        }
    };
}

pub fn main() void {
    const valid = "123";
    const invalid = "abc";
    const overflow = "999999999999999999999";
    const mixed = "12abc";
    
    std.debug.print("Valid: {}\n", .{parseIntSafe(valid)});
    std.debug.print("Invalid: {}\n", .{parseIntSafe(invalid)});
    std.debug.print("Overflow: {}\n", .{parseIntSafe(overflow)});
    std.debug.print("Mixed: {}\n", .{parseIntSafe(mixed)});
    
    // อีกวิธีจัดการ error
    const result = std.fmt.parseInt(i32, "not_a_number", 10);
    if (result) |value| {
        std.debug.print("Parsed: {}\n", .{value});
    } else |err| {
        std.debug.print("Parse error: {}\n", .{err});
    }
}
```

### การแปลง String เป็น Float

```zig
const std = @import("std");

pub fn main() !void {
    // std.fmt.parseFloat สำหรับแปลง string เป็น float
    const pi_str = "3.14159";
    const pi = try std.fmt.parseFloat(f64, pi_str);
    std.debug.print("Pi: {d:.5}\n", .{pi});
    
    const sci_str = "1.5e10";
    const sci_num = try std.fmt.parseFloat(f64, sci_str);
    std.debug.print("Scientific: {e}\n", .{sci_num});
    
    const neg_float = "-2.718";
    const e_num = try std.fmt.parseFloat(f64, neg_float);
    std.debug.print("Negative float: {d:.3}\n", .{e_num});
    
    // f32 precision
    const price_str = "99.99";
    const price = try std.fmt.parseFloat(f32, price_str);
    std.debug.print("Price: {d:.2}\n", .{price});
    
    // ตัวอย่างใช้งานจริง: รับ input และคำนวณ
    const width_str = "10.5";
    const height_str = "20.3";
    
    const width = try std.fmt.parseFloat(f64, width_str);
    const height = try std.fmt.parseFloat(f64, height_str);
    const area = width * height;
    
    std.debug.print("Area: {d:.2} sq units\n", .{area});
}
```

### การแปลง String เป็น Bool

```zig
const std = @import("std");

fn parseBool(str: []const u8) ?bool {
    if (std.mem.eql(u8, str, "true") or 
        std.mem.eql(u8, str, "1") or
        std.ascii.eqlIgnoreCase(str, "yes") or
        std.ascii.eqlIgnoreCase(str, "on")) {
        return true;
    }
    if (std.mem.eql(u8, str, "false") or 
        std.mem.eql(u8, str, "0") or
        std.ascii.eqlIgnoreCase(str, "no") or
        std.ascii.eqlIgnoreCase(str, "off")) {
        return false;
    }
    return null; // ไม่รู้จัก
}

pub fn main() void {
    const test_values = [_][]const u8{ "true", "false", "1", "0", "YES", "NO", "on", "off", "maybe" };
    
    for (test_values) |val| {
        if (parseBool(val)) |b| {
            std.debug.print("'{s}' -> {}\n", .{ val, b });
        } else {
            std.debug.print("'{s}' -> unknown\n", .{val});
        }
    }
}
```

---

## Step 98: Int/Float to String (std.fmt.bufPrint, allocPrint)

### std.fmt.bufPrint - เขียนลง Buffer ที่มีอยู่แล้ว

```zig
const std = @import("std");

pub fn main() !void {
    // bufPrint เขียนลง buffer ที่เราจัดเตรียมไว้
    // ไม่ต้องใช้ allocator - เหมาะสำหรับ stack allocation
    
    var buf: [64]u8 = undefined;
    
    // แปลง integer เป็น string
    const num = 42;
    const num_str = try std.fmt.bufPrint(&buf, "{d}", .{num});
    std.debug.print("Number as string: '{s}'\n", .{num_str});
    std.debug.print("Length: {}\n", .{num_str.len});
    
    // แปลง float เป็น string
    const pi: f64 = 3.14159;
    const pi_str = try std.fmt.bufPrint(&buf, "{d:.2}", .{pi});
    std.debug.print("Pi as string: '{s}'\n", .{pi_str});
    
    // สร้าง formatted string
    const age = 25;
    const name = "สมชาย";
    const formatted = try std.fmt.bufPrint(&buf, "{s} is {d} years old", .{ name, age });
    std.debug.print("{s}\n", .{formatted});
    
    // Error: buffer too small
    var small_buf: [5]u8 = undefined;
    const result = std.fmt.bufPrint(&small_buf, "This is too long", .{});
    if (result) |s| {
        std.debug.print("Success: {s}\n", .{s});
    } else |err| {
        std.debug.print("Error: {}\n", .{err}); // error.NoSpaceLeft
    }
}
```

### std.fmt.allocPrint - Allocate และ Print

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // allocPrint allocate memory อัตโนมัติ
    // ต้อง free หลังใช้งาน
    const num = 12345;
    const num_str = try std.fmt.allocPrint(allocator, "{d}", .{num});
    defer allocator.free(num_str);
    std.debug.print("Number: {s}\n", .{num_str});
    
    // Float with precision
    const temp: f64 = 36.6;
    const temp_str = try std.fmt.allocPrint(allocator, "{d:.1}°C", .{temp});
    defer allocator.free(temp_str);
    std.debug.print("Temperature: {s}\n", .{temp_str});
    
    // Hex format
    const color: u32 = 0xFF5733;
    const hex_str = try std.fmt.allocPrint(allocator, "#{X:0>6}", .{color});
    defer allocator.free(hex_str);
    std.debug.print("Color: {s}\n", .{hex_str}); // #FF5733
    
    // Binary
    const flags: u8 = 0b10110101;
    const bin_str = try std.fmt.allocPrint(allocator, "0b{b:0>8}", .{flags});
    defer allocator.free(bin_str);
    std.debug.print("Flags: {s}\n", .{bin_str}); // 0b10110101
}
```

### Format Options ทั้งหมด

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const num: i64 = 1234567;
    const float_num: f64 = 3.14159265358979;
    
    // Integer formats
    const formats = [_]struct { fmt: []const u8, val: i64 }{
        .{ .fmt = "{d}", .val = num },         // decimal
        .{ .fmt = "{}", .val = num },          // default (same as d for integers)
        .{ .fmt = "{x}", .val = num },         // hex lowercase
        .{ .fmt = "{X}", .val = num },         // hex uppercase
        .{ .fmt = "{o}", .val = num },         // octal
        .{ .fmt = "{b}", .val = num },         // binary
    };
    _ = formats;
    
    // แสดงตัวอย่าง format แต่ละแบบ
    var s: []u8 = undefined;
    
    s = try std.fmt.allocPrint(allocator, "{d}", .{num});
    std.debug.print("Decimal: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{x}", .{num});
    std.debug.print("Hex lower: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{X}", .{num});
    std.debug.print("Hex upper: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{b}", .{num});
    std.debug.print("Binary: {s}\n", .{s});
    allocator.free(s);
    
    // Float formats
    s = try std.fmt.allocPrint(allocator, "{d}", .{float_num});
    std.debug.print("Float default: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{d:.2}", .{float_num});
    std.debug.print("Float 2 decimals: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{e}", .{float_num});
    std.debug.print("Scientific: {s}\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{e:.3}", .{float_num});
    std.debug.print("Scientific 3 decimals: {s}\n", .{s});
    allocator.free(s);
    
    // Width and padding
    s = try std.fmt.allocPrint(allocator, "{d:10}", .{num});
    std.debug.print("Width 10 right: '{s}'\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{d:<10}", .{num});
    std.debug.print("Width 10 left: '{s}'\n", .{s});
    allocator.free(s);
    
    s = try std.fmt.allocPrint(allocator, "{d:0>10}", .{num});
    std.debug.print("Zero padded: '{s}'\n", .{s});
    allocator.free(s);
}
```

### Writer Interface สำหรับ Custom Output

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ใช้ ArrayList เป็น string builder
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    const writer = output.writer();
    
    // เขียนตาราง
    try writer.writeAll("┌────────────┬─────────┐\n");
    try writer.writeAll("│ Name       │ Score   │\n");
    try writer.writeAll("├────────────┼─────────┤\n");
    
    const students = [_]struct { name: []const u8, score: f32 }{
        .{ .name = "สมชาย", .score = 95.5 },
        .{ .name = "สมหญิง", .score = 87.3 },
        .{ .name = "สมศักดิ์", .score = 92.1 },
    };
    
    for (students) |student| {
        try writer.print("│ {s:<10} │ {d:>7.1} │\n", .{ student.name, student.score });
    }
    
    try writer.writeAll("└────────────┴─────────┘\n");
    
    std.debug.print("{s}", .{output.items});
}
```

---

## Step 99: Unicode และ UTF-8 ใน Zig

### ทำความเข้าใจ UTF-8

UTF-8 เป็น encoding ที่ Zig ใช้สำหรับ string literals โดย default ข้อควรระวังคือ:
- 1 Thai character ใช้ **3 bytes** ใน UTF-8
- 1 ASCII character ใช้ **1 byte**
- `str.len` คืนค่า **จำนวน bytes** ไม่ใช่จำนวน characters

```zig
const std = @import("std");

pub fn main() void {
    const ascii = "Hello";
    const thai = "สวัสดี";
    const mixed = "Hello สวัสดี";
    
    // len คือจำนวน bytes
    std.debug.print("ASCII '{s}': {} bytes\n", .{ ascii, ascii.len }); // 5
    std.debug.print("Thai '{s}': {} bytes\n", .{ thai, thai.len });    // 18 (6 chars × 3 bytes)
    std.debug.print("Mixed '{s}': {} bytes\n", .{ mixed, mixed.len }); // 5 + 1 + 18 = 24
    
    // แสดง bytes ของ string ไทย
    std.debug.print("\nBytes of 'สวัสดี':\n", .{});
    for (thai, 0..) |byte, i| {
        std.debug.print("[{}] = 0x{X:0>2} ({d})\n", .{ i, byte, byte });
    }
}
```

### Unicode Codepoints และ UTF-8 Encoding

```zig
const std = @import("std");

pub fn main() !void {
    // ตัวอักษรไทยอยู่ใน Unicode block U+0E00-U+0E7F
    // เมื่อ encode เป็น UTF-8 จะใช้ 3 bytes ต่อตัวอักษร
    
    // ก = U+0E01 = 0xE0 0xB8 0x81 ใน UTF-8
    const kor_kai = "ก";
    std.debug.print("ก: {} bytes\n", .{kor_kai.len}); // 3
    std.debug.print("Bytes: 0x{X:0>2} 0x{X:0>2} 0x{X:0>2}\n", .{
        kor_kai[0], kor_kai[1], kor_kai[2]
    });
    
    // นับจำนวน Unicode codepoints (ตัวอักษรจริง)
    const unicode = std.unicode;
    
    const thai_text = "สวัสดีครับ";
    const byte_count = thai_text.len;
    
    // นับ codepoints ด้วย utf8CountCodepoints
    const codepoint_count = try unicode.utf8CountCodepoints(thai_text);
    
    std.debug.print("\nText: {s}\n", .{thai_text});
    std.debug.print("Bytes: {}\n", .{byte_count});
    std.debug.print("Codepoints (characters): {}\n", .{codepoint_count});
    
    // ข้อสังเกต: สระบางตัวใช้ 2 codepoints (base + combining)
    // เช่น 'สวัสดี' มี 'ั' (sara a ไม้หัน) ที่เป็น combining character
}
```

### UTF-8 Iterator - วน Loop ทีละ Codepoint

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    const text = "Hello, สวัสดี! 🌏";
    
    std.debug.print("Text: {s}\n", .{text});
    std.debug.print("Total bytes: {}\n\n", .{text.len});
    
    // ใช้ Utf8View เพื่อ iterate ทีละ codepoint
    var view = try unicode.Utf8View.init(text);
    var iter = view.iterator();
    
    var char_count: usize = 0;
    while (iter.nextCodepointSlice()) |char_slice| {
        const codepoint = try unicode.utf8Decode(char_slice);
        std.debug.print("  U+{X:0>4} ({} bytes): {s}\n", .{
            codepoint,
            char_slice.len,
            char_slice,
        });
        char_count += 1;
    }
    
    std.debug.print("\nTotal characters: {}\n", .{char_count});
    
    // ตรวจสอบ valid UTF-8
    const valid_utf8 = "Hello ไทย";
    const invalid_bytes = [_]u8{ 0xFF, 0xFE, 0x00 }; // invalid UTF-8
    
    std.debug.print("\nValid UTF-8 check:\n", .{});
    std.debug.print("'{s}': {}\n", .{ valid_utf8, unicode.utf8ValidateSlice(valid_utf8) });
    std.debug.print("Invalid bytes: {}\n", .{unicode.utf8ValidateSlice(&invalid_bytes)});
}
```

### การทำงานกับ Thai Text

```zig
const std = @import("std");
const unicode = std.unicode;

// ตรวจสอบว่าเป็น Thai character หรือไม่
fn isThaiChar(codepoint: u21) bool {
    return codepoint >= 0x0E00 and codepoint <= 0x0E7F;
}

// นับตัวอักษรไทยใน string
fn countThaiChars(text: []const u8) !usize {
    var count: usize = 0;
    var view = try unicode.Utf8View.init(text);
    var iter = view.iterator();
    
    while (iter.nextCodepoint()) |cp| {
        if (isThaiChar(cp)) {
            count += 1;
        }
    }
    return count;
}

pub fn main() !void {
    const text1 = "Hello สวัสดี World ไทย";
    const text2 = "Programming in ภาษาไทย is fun!";
    
    const thai_count1 = try countThaiChars(text1);
    const thai_count2 = try countThaiChars(text2);
    
    std.debug.print("Text: {s}\n", .{text1});
    std.debug.print("Thai characters: {}\n\n", .{thai_count1});
    
    std.debug.print("Text: {s}\n", .{text2});
    std.debug.print("Thai characters: {}\n", .{thai_count2});
    
    // ตัวอย่างนับคำในภาษาอังกฤษ (แบ่งด้วย space)
    const english = "The quick brown fox";
    var word_count: usize = 0;
    var in_word = false;
    
    for (english) |byte| {
        if (byte == ' ') {
            in_word = false;
        } else if (!in_word) {
            in_word = true;
            word_count += 1;
        }
    }
    std.debug.print("\nWord count in '{s}': {}\n", .{ english, word_count });
}
```

### UTF-8 Encoding/Decoding Manual

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    // Encode codepoint เป็น UTF-8 bytes
    var buf: [4]u8 = undefined;
    
    // ASCII (1 byte)
    const a_len = try unicode.utf8Encode('A', &buf);
    std.debug.print("'A' = {} byte(s): {any}\n", .{ a_len, buf[0..a_len] });
    
    // Thai (3 bytes) - ก = U+0E01
    const gor_len = try unicode.utf8Encode(0x0E01, &buf);
    std.debug.print("'ก' = {} byte(s): {any}\n", .{ gor_len, buf[0..gor_len] });
    
    // Emoji (4 bytes) - 🌏 = U+1F30F
    const earth_len = try unicode.utf8Encode(0x1F30F, &buf);
    std.debug.print("'🌏' = {} byte(s): {any}\n", .{ earth_len, buf[0..earth_len] });
    
    // Decode bytes กลับเป็น codepoint
    const thai_bytes = [_]u8{ 0xE0, 0xB8, 0x81 }; // ก
    const decoded_cp = try unicode.utf8Decode(&thai_bytes);
    std.debug.print("\nDecoded U+{X:0>4}\n", .{decoded_cp});
    
    // ตรวจสอบ byte sequence length
    const byte_len_1 = unicode.utf8ByteSequenceLength(0x41) catch 0;  // A
    const byte_len_3 = unicode.utf8ByteSequenceLength(0xE0) catch 0;  // start of Thai
    const byte_len_4 = unicode.utf8ByteSequenceLength(0xF0) catch 0;  // start of emoji
    
    std.debug.print("\nByte sequence lengths:\n", .{});
    std.debug.print("0x41 (ASCII start): {} bytes\n", .{byte_len_1});
    std.debug.print("0xE0 (Thai start): {} bytes\n", .{byte_len_3});
    std.debug.print("0xF0 (Emoji start): {} bytes\n", .{byte_len_4});
}
```

### ตัวอย่าง: ย้อนกลับ String ที่มี Unicode

```zig
const std = @import("std");
const unicode = std.unicode;

// ย้อนกลับ string โดยรักษา Unicode codepoints
fn reverseUtf8(allocator: std.mem.Allocator, text: []const u8) ![]u8 {
    // เก็บ codepoints ทั้งหมดก่อน
    var codepoints = std.ArrayList([]const u8).init(allocator);
    defer codepoints.deinit();
    
    var view = try unicode.Utf8View.init(text);
    var iter = view.iterator();
    
    while (iter.nextCodepointSlice()) |slice| {
        try codepoints.append(slice);
    }
    
    // คำนวณความยาวรวม
    const total_len = text.len;
    const result = try allocator.alloc(u8, total_len);
    
    // เขียน codepoints ย้อนหลัง
    var pos: usize = 0;
    var i: usize = codepoints.items.len;
    while (i > 0) {
        i -= 1;
        const slice = codepoints.items[i];
        @memcpy(result[pos..pos + slice.len], slice);
        pos += slice.len;
    }
    
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ย้อนกลับ ASCII
    const ascii = "Hello";
    const rev_ascii = try reverseUtf8(allocator, ascii);
    defer allocator.free(rev_ascii);
    std.debug.print("Original: {s}\n", .{ascii});
    std.debug.print("Reversed: {s}\n\n", .{rev_ascii});
    
    // ย้อนกลับภาษาไทย
    const thai = "สวัสดี";
    const rev_thai = try reverseUtf8(allocator, thai);
    defer allocator.free(rev_thai);
    std.debug.print("Original: {s}\n", .{thai});
    std.debug.print("Reversed: {s}\n", .{rev_thai});
}
```

---

## Step 100: String Utilities - contains, starts/endsWith, split

### std.mem.contains และการค้นหา

```zig
const std = @import("std");

pub fn main() void {
    const text = "The quick brown fox jumps over the lazy dog";
    
    // ตรวจสอบว่ามี substring หรือไม่
    // Zig ไม่มี contains() แต่ใช้ indexOf != null แทน
    const has_fox = std.mem.indexOf(u8, text, "fox") != null;
    const has_cat = std.mem.indexOf(u8, text, "cat") != null;
    
    std.debug.print("Contains 'fox': {}\n", .{has_fox}); // true
    std.debug.print("Contains 'cat': {}\n", .{has_cat}); // false
    
    // นับจำนวนครั้งที่พบ
    fn countOccurrences(haystack: []const u8, needle: []const u8) usize {
        _ = haystack;
        _ = needle;
        // implementation below
    };
    
    var count: usize = 0;
    var search_from: usize = 0;
    while (std.mem.indexOf(u8, text[search_from..], "the")) |pos| {
        count += 1;
        search_from += pos + "the".len;
    }
    std.debug.print("Occurrences of 'the': {}\n", .{count}); // depends on case
    
    // Case-insensitive search
    var ci_count: usize = 0;
    var ci_search: usize = 0;
    while (ci_search < text.len) {
        const remaining = text[ci_search..];
        // simple implementation - find each 'the' case insensitive
        if (remaining.len < 3) break;
        const chunk = remaining[0..3];
        if (std.ascii.eqlIgnoreCase(chunk, "the")) {
            ci_count += 1;
            ci_search += 3;
        } else {
            ci_search += 1;
        }
    }
    std.debug.print("Case-insensitive 'the' count: {}\n", .{ci_count});
}
```

### startsWith และ endsWith

```zig
const std = @import("std");

pub fn main() void {
    // startsWith
    const files = [_][]const u8{
        "main.zig",
        "test_utils.zig",
        "build.zig",
        "readme.md",
        "config.json",
        "test_main.zig",
    };
    
    std.debug.print("Files starting with 'test_':\n", .{});
    for (files) |file| {
        if (std.mem.startsWith(u8, file, "test_")) {
            std.debug.print("  {s}\n", .{file});
        }
    }
    
    // endsWith
    std.debug.print("\nZig source files (.zig):\n", .{});
    for (files) |file| {
        if (std.mem.endsWith(u8, file, ".zig")) {
            std.debug.print("  {s}\n", .{file});
        }
    }
    
    // ตัวอย่างใช้งานจริง: ตรวจสอบ URL
    const urls = [_][]const u8{
        "https://example.com/page",
        "http://insecure.com",
        "ftp://files.example.com",
        "https://secure.example.com/api",
    };
    
    std.debug.print("\nHTTPS URLs:\n", .{});
    for (urls) |url| {
        if (std.mem.startsWith(u8, url, "https://")) {
            std.debug.print("  Secure: {s}\n", .{url});
        } else {
            std.debug.print("  Insecure: {s}\n", .{url});
        }
    }
    
    // ตรวจสอบ email
    const email = "user@example.com";
    const is_email = std.mem.indexOf(u8, email, "@") != null and
        std.mem.endsWith(u8, email, ".com");
    std.debug.print("\nIs valid email format: {}\n", .{is_email});
}
```

### String Split

```zig
const std = @import("std");

pub fn main() void {
    // std.mem.splitScalar - แบ่งด้วย single byte
    const csv = "name,age,city,country";
    
    std.debug.print("CSV fields:\n", .{});
    var iter = std.mem.splitScalar(u8, csv, ',');
    var field_num: usize = 0;
    while (iter.next()) |field| {
        field_num += 1;
        std.debug.print("  Field {}: {s}\n", .{ field_num, field });
    }
    
    // std.mem.splitSequence - แบ่งด้วย multi-byte sequence
    const text = "one::two::three::four";
    std.debug.print("\nSplit by '::':\n", .{});
    var iter2 = std.mem.splitSequence(u8, text, "::");
    while (iter2.next()) |part| {
        std.debug.print("  {s}\n", .{part});
    }
    
    // แบ่ง lines
    const multiline =
        \\Line 1
        \\Line 2
        \\Line 3
    ;
    
    std.debug.print("\nLines:\n", .{});
    var line_iter = std.mem.splitScalar(u8, multiline, '\n');
    var line_num: usize = 0;
    while (line_iter.next()) |line| {
        line_num += 1;
        std.debug.print("  {}: {s}\n", .{ line_num, line });
    }
    
    // แบ่ง path
    const path = "/home/user/documents/file.txt";
    std.debug.print("\nPath components:\n", .{});
    var path_iter = std.mem.splitScalar(u8, path, '/');
    while (path_iter.next()) |component| {
        if (component.len > 0) { // ข้ามส่วนที่ว่าง
            std.debug.print("  {s}\n", .{component});
        }
    }
}
```

### tokenize - Split โดยข้าม Consecutive Separators

```zig
const std = @import("std");

pub fn main() void {
    // tokenize ต่างจาก split ตรงที่จะข้าม consecutive separators
    const spaced = "  hello   world  foo  ";
    
    std.debug.print("Tokenize by space:\n", .{});
    var token_iter = std.mem.tokenizeScalar(u8, spaced, ' ');
    while (token_iter.next()) |token| {
        std.debug.print("  '{s}'\n", .{token});
    }
    // จะได้: 'hello', 'world', 'foo'
    
    // เปรียบเทียบกับ split
    std.debug.print("\nSplit by space (รวม empty):\n", .{});
    var split_iter = std.mem.splitScalar(u8, spaced, ' ');
    var count: usize = 0;
    while (split_iter.next()) |part| {
        count += 1;
        std.debug.print("  [{}] '{s}'\n", .{ count, part });
    }
    
    // tokenize ด้วย multiple separators (any byte in set)
    const mixed_sep = "word1,word2;word3.word4";
    std.debug.print("\nTokenize by ',;.':\n", .{});
    var multi_iter = std.mem.tokenizeAny(u8, mixed_sep, ",;.");
    while (multi_iter.next()) |token| {
        std.debug.print("  {s}\n", .{token});
    }
}
```

### String Replace

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // std.mem.replace นับจำนวน replacements
    // std.mem.replaceOwned สร้าง string ใหม่
    
    const original = "Hello World! Hello Zig! Hello Everyone!";
    const old = "Hello";
    const new = "Sawadee";
    
    // นับ occurrences ก่อน
    const count = std.mem.count(u8, original, old);
    std.debug.print("Found {} occurrences of '{s}'\n", .{ count, old });
    
    // replaceOwned สร้าง string ใหม่ที่มีการ replace ทั้งหมด
    const replaced = try std.mem.replaceOwned(u8, allocator, original, old, new);
    defer allocator.free(replaced);
    
    std.debug.print("Original: {s}\n", .{original});
    std.debug.print("Replaced: {s}\n", .{replaced});
    
    // ตัวอย่าง: Replace template variables
    const template = "Dear {NAME}, your order #{ORDER_ID} has been shipped.";
    
    var result = try std.mem.replaceOwned(u8, allocator, template, "{NAME}", "สมชาย");
    defer allocator.free(result);
    
    const final = try std.mem.replaceOwned(u8, allocator, result, "{ORDER_ID}", "12345");
    defer allocator.free(final);
    
    std.debug.print("\nTemplate result: {s}\n", .{final});
}
```

### String Utilities รวม - ตัวอย่างโปรแกรม

```zig
const std = @import("std");
const unicode = std.unicode;

// String utilities module
const StringUtils = struct {
    // ตรวจสอบว่า string ว่างเปล่าหรือมีแต่ whitespace
    pub fn isBlank(str: []const u8) bool {
        return std.mem.trim(u8, str, " \t\n\r").len == 0;
    }
    
    // นับจำนวนครั้งที่พบ substring
    pub fn countOccurrences(haystack: []const u8, needle: []const u8) usize {
        if (needle.len == 0) return 0;
        var count: usize = 0;
        var start: usize = 0;
        while (std.mem.indexOf(u8, haystack[start..], needle)) |pos| {
            count += 1;
            start += pos + needle.len;
        }
        return count;
    }
    
    // แปลงเป็น UPPER CASE (ASCII only)
    pub fn toUpper(allocator: std.mem.Allocator, str: []const u8) ![]u8 {
        const result = try allocator.dupe(u8, str);
        for (result) |*byte| {
            byte.* = std.ascii.toUpper(byte.*);
        }
        return result;
    }
    
    // แปลงเป็น lower case (ASCII only)
    pub fn toLower(allocator: std.mem.Allocator, str: []const u8) ![]u8 {
        const result = try allocator.dupe(u8, str);
        for (result) |*byte| {
            byte.* = std.ascii.toLower(byte.*);
        }
        return result;
    }
    
    // Repeat string N times
    pub fn repeat(allocator: std.mem.Allocator, str: []const u8, n: usize) ![]u8 {
        if (n == 0 or str.len == 0) return try allocator.dupe(u8, "");
        const result = try allocator.alloc(u8, str.len * n);
        var i: usize = 0;
        while (i < n) : (i += 1) {
            @memcpy(result[i * str.len..][0..str.len], str);
        }
        return result;
    }
    
    // ตรวจสอบว่าเป็น palindrome หรือไม่ (ASCII)
    pub fn isPalindrome(str: []const u8) bool {
        if (str.len <= 1) return true;
        var left: usize = 0;
        var right: usize = str.len - 1;
        while (left < right) {
            if (std.ascii.toLower(str[left]) != std.ascii.toLower(str[right])) {
                return false;
            }
            left += 1;
            right -= 1;
        }
        return true;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ทดสอบ isBlank
    std.debug.print("=== isBlank ===\n", .{});
    std.debug.print("'': {}\n", .{StringUtils.isBlank("")});
    std.debug.print("'   ': {}\n", .{StringUtils.isBlank("   ")});
    std.debug.print("'hello': {}\n", .{StringUtils.isBlank("hello")});
    std.debug.print("'  hi  ': {}\n", .{StringUtils.isBlank("  hi  ")});
    
    // ทดสอบ countOccurrences
    std.debug.print("\n=== countOccurrences ===\n", .{});
    const text = "banana";
    std.debug.print("'a' in 'banana': {}\n", .{StringUtils.countOccurrences(text, "a")}); // 3
    std.debug.print("'an' in 'banana': {}\n", .{StringUtils.countOccurrences(text, "an")}); // 2
    
    // ทดสอบ toUpper/toLower
    std.debug.print("\n=== Case Conversion ===\n", .{});
    const mixed = "Hello World";
    const upper = try StringUtils.toUpper(allocator, mixed);
    defer allocator.free(upper);
    const lower = try StringUtils.toLower(allocator, mixed);
    defer allocator.free(lower);
    
    std.debug.print("Original: {s}\n", .{mixed});
    std.debug.print("Upper: {s}\n", .{upper});
    std.debug.print("Lower: {s}\n", .{lower});
    
    // ทดสอบ repeat
    std.debug.print("\n=== Repeat ===\n", .{});
    const sep = try StringUtils.repeat(allocator, "-", 20);
    defer allocator.free(sep);
    std.debug.print("{s}\n", .{sep}); // --------------------
    
    const star_row = try StringUtils.repeat(allocator, "* ", 5);
    defer allocator.free(star_row);
    std.debug.print("{s}\n", .{star_row}); // * * * * * 
    
    // ทดสอบ isPalindrome
    std.debug.print("\n=== Palindrome ===\n", .{});
    const words = [_][]const u8{ "racecar", "hello", "level", "world", "noon" };
    for (words) |word| {
        std.debug.print("'{s}': {}\n", .{ word, StringUtils.isPalindrome(word) });
    }
}
```

### การทำงานกับ CSV ด้วย String Functions

```zig
const std = @import("std");

const CsvParser = struct {
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) CsvParser {
        return .{ .allocator = allocator };
    }
    
    // Parse CSV line เป็น fields
    pub fn parseLine(self: CsvParser, line: []const u8) !std.ArrayList([]const u8) {
        var fields = std.ArrayList([]const u8).init(self.allocator);
        var iter = std.mem.splitScalar(u8, line, ',');
        while (iter.next()) |field| {
            const trimmed = std.mem.trim(u8, field, " ");
            try fields.append(trimmed);
        }
        return fields;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const csv_data =
        \\ชื่อ,อายุ,เมือง,อาชีพ
        \\สมชาย ใจดี,30,กรุงเทพ,วิศวกร
        \\สมหญิง รักสวย,25,เชียงใหม่,นักออกแบบ
        \\สมศักดิ์ มั่นคง,35,ขอนแก่น,แพทย์
    ;
    
    const parser = CsvParser.init(allocator);
    
    // Process each line
    var line_iter = std.mem.splitScalar(u8, csv_data, '\n');
    var is_header = true;
    
    while (line_iter.next()) |line| {
        if (line.len == 0) continue;
        
        var fields = try parser.parseLine(line);
        defer fields.deinit();
        
        if (is_header) {
            std.debug.print("=== Headers ===\n", .{});
            for (fields.items, 0..) |field, i| {
                std.debug.print("  [{}] {s}\n", .{ i, field });
            }
            std.debug.print("=== Data ===\n", .{});
            is_header = false;
        } else {
            if (fields.items.len >= 4) {
                std.debug.print("Name: {s}, Age: {s}, City: {s}, Job: {s}\n", .{
                    fields.items[0],
                    fields.items[1],
                    fields.items[2],
                    fields.items[3],
                });
            }
        }
    }
}
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: String Inspector

เขียนโปรแกรมที่รับ string และแสดงข้อมูลต่อไปนี้:
- จำนวน bytes
- จำนวน Unicode codepoints
- จำนวน ASCII characters
- จำนวน Non-ASCII characters (รวมภาษาไทย)
- แสดง bytes ในรูปแบบ hex dump

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn stringInspect(text: []const u8) !void {
    std.debug.print("=== String Inspector ===\n", .{});
    std.debug.print("String: {s}\n", .{text});
    std.debug.print("Bytes: {}\n", .{text.len});
    
    // TODO: นักเรียนเติม code ส่วนนี้
    // 1. นับ Unicode codepoints
    // 2. นับ ASCII chars
    // 3. นับ non-ASCII chars
    // 4. แสดง hex dump
    
    var view = try unicode.Utf8View.init(text);
    var iter = view.iterator();
    
    var total_codepoints: usize = 0;
    var ascii_count: usize = 0;
    var non_ascii_count: usize = 0;
    
    while (iter.nextCodepoint()) |cp| {
        total_codepoints += 1;
        if (cp < 128) {
            ascii_count += 1;
        } else {
            non_ascii_count += 1;
        }
    }
    
    std.debug.print("Codepoints: {}\n", .{total_codepoints});
    std.debug.print("ASCII chars: {}\n", .{ascii_count});
    std.debug.print("Non-ASCII chars: {}\n", .{non_ascii_count});
    
    // Hex dump
    std.debug.print("\nHex dump:\n", .{});
    for (text, 0..) |byte, i| {
        if (i % 16 == 0) {
            if (i > 0) std.debug.print("\n", .{});
            std.debug.print("{X:0>4}: ", .{i});
        }
        std.debug.print("{X:0>2} ", .{byte});
    }
    std.debug.print("\n", .{});
}

pub fn main() !void {
    try stringInspect("Hello, สวัสดี! 🌏");
    std.debug.print("\n", .{});
    try stringInspect("Zig Programming Language");
    std.debug.print("\n", .{});
    try stringInspect("ภาษาไทยใน Zig");
}
```

### แบบฝึกหัดที่ 2: Simple Template Engine

เขียน template engine อย่างง่ายที่ replace `{key}` ด้วยค่าจาก map

```zig
const std = @import("std");

// Template variables
const TemplateVars = struct {
    allocator: std.mem.Allocator,
    vars: std.StringHashMap([]const u8),
    
    pub fn init(allocator: std.mem.Allocator) TemplateVars {
        return .{
            .allocator = allocator,
            .vars = std.StringHashMap([]const u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *TemplateVars) void {
        self.vars.deinit();
    }
    
    pub fn set(self: *TemplateVars, key: []const u8, value: []const u8) !void {
        try self.vars.put(key, value);
    }
    
    pub fn render(self: TemplateVars, template: []const u8) ![]u8 {
        var result = std.ArrayList(u8).init(self.allocator);
        defer result.deinit();
        
        var i: usize = 0;
        while (i < template.len) {
            if (template[i] == '{') {
                // หา closing brace
                if (std.mem.indexOfScalarPos(u8, template, i + 1, '}')) |end| {
                    const key = template[i + 1..end];
                    if (self.vars.get(key)) |value| {
                        try result.appendSlice(value);
                    } else {
                        // Key not found, keep original
                        try result.appendSlice(template[i..end + 1]);
                    }
                    i = end + 1;
                    continue;
                }
            }
            try result.append(template[i]);
            i += 1;
        }
        
        return try self.allocator.dupe(u8, result.items);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var vars = TemplateVars.init(allocator);
    defer vars.deinit();
    
    try vars.set("name", "สมชาย");
    try vars.set("age", "30");
    try vars.set("city", "กรุงเทพ");
    try vars.set("lang", "Zig");
    
    const template = "สวัสดีครับ {name} อายุ {age} ปี จาก{city} กำลังเรียน {lang}!";
    const rendered = try vars.render(template);
    defer allocator.free(rendered);
    
    std.debug.print("Template: {s}\n", .{template});
    std.debug.print("Rendered: {s}\n", .{rendered});
}
```

### แบบฝึกหัดที่ 3: Word Frequency Counter

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const text =
        \\Zig is a general purpose programming language
        \\Zig is designed for robustness optimality and maintainability
        \\Zig competes with C and aims to be a better C
    ;
    
    // สร้าง hash map เก็บความถี่
    var freq = std.StringHashMap(usize).init(allocator);
    defer freq.deinit();
    
    // แบ่งคำ (ใช้ tokenize เพื่อข้าม multiple spaces)
    var iter = std.mem.tokenizeAny(u8, text, " \n\t\r,.!?");
    while (iter.next()) |word| {
        const lower_word = try std.ascii.allocLowerString(allocator, word);
        defer allocator.free(lower_word);
        
        const entry = try freq.getOrPut(lower_word);
        if (entry.found_existing) {
            entry.value_ptr.* += 1;
        } else {
            entry.value_ptr.* = 1;
            // ต้อง dup key เพราะ lower_word จะถูก free
            entry.key_ptr.* = try allocator.dupe(u8, lower_word);
        }
    }
    
    // แสดงผล
    std.debug.print("Word Frequencies:\n", .{});
    var freq_iter = freq.iterator();
    while (freq_iter.next()) |entry| {
        std.debug.print("  {s}: {}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
    
    // Cleanup keys
    var cleanup_iter = freq.iterator();
    while (cleanup_iter.next()) |entry| {
        allocator.free(entry.key_ptr.*);
    }
}
```

### แบบฝึกหัดที่ 4: String Formatter สำหรับตาราง

```zig
const std = @import("std");

pub fn printTable(
    allocator: std.mem.Allocator,
    headers: []const []const u8,
    rows: []const []const []const u8,
) !void {
    // คำนวณความกว้างของแต่ละ column
    var col_widths = try allocator.alloc(usize, headers.len);
    defer allocator.free(col_widths);
    
    // เริ่มจากความกว้างของ headers
    for (headers, 0..) |header, i| {
        col_widths[i] = header.len;
    }
    
    // อัพเดท ด้วยความกว้างของ data
    for (rows) |row| {
        for (row, 0..) |cell, i| {
            if (i < col_widths.len and cell.len > col_widths[i]) {
                col_widths[i] = cell.len;
            }
        }
    }
    
    // Print separator
    const printSep = struct {
        fn f(widths: []const usize) void {
            std.debug.print("+", .{});
            for (widths) |w| {
                var j: usize = 0;
                while (j < w + 2) : (j += 1) {
                    std.debug.print("-", .{});
                }
                std.debug.print("+", .{});
            }
            std.debug.print("\n", .{});
        }
    }.f;
    
    // Print row
    const printRow = struct {
        fn f(cells: []const []const u8, widths: []const usize) void {
            std.debug.print("|", .{});
            for (cells, 0..) |cell, i| {
                if (i < widths.len) {
                    std.debug.print(" {s}", .{cell});
                    var pad: usize = widths[i] - cell.len;
                    while (pad > 0) : (pad -= 1) {
                        std.debug.print(" ", .{});
                    }
                    std.debug.print(" |", .{});
                }
            }
            std.debug.print("\n", .{});
        }
    }.f;
    
    printSep(col_widths);
    printRow(headers, col_widths);
    printSep(col_widths);
    
    for (rows) |row| {
        printRow(row, col_widths);
    }
    
    printSep(col_widths);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const headers: []const []const u8 = &.{ "ชื่อ", "อายุ", "เมือง", "อาชีพ" };
    const rows: []const []const []const u8 = &.{
        &.{ "สมชาย", "30", "กรุงเทพ", "วิศวกร" },
        &.{ "สมหญิง", "25", "เชียงใหม่", "นักออกแบบ" },
        &.{ "สมศักดิ์", "35", "ขอนแก่น", "แพทย์" },
    };
    
    try printTable(allocator, headers, rows);
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ในภาคที่ 10

| Step | หัวข้อ | สิ่งสำคัญ |
|------|--------|-----------|
| 91 | Strings คืออะไร | `[]const u8` คือ byte slice ไม่ใช่ string type |
| 92 | Escape sequences | `\n`, `\t`, `\\`, `\"`, `\xNN`, `\u{NNNN}` |
| 93 | Multiline strings | ใช้ `\\` นำหน้าแต่ละบรรทัด |
| 94 | String comparison | `std.mem.eql`, `std.mem.order`, `std.ascii.eqlIgnoreCase` |
| 95 | String concatenation | `++` (compile-time), `std.mem.concat`, `ArrayList(u8)` |
| 96 | String slicing | `str[start..end]`, `std.mem.indexOf`, `std.mem.trim` |
| 97 | String to number | `std.fmt.parseInt`, `std.fmt.parseFloat` |
| 98 | Number to string | `std.fmt.bufPrint`, `std.fmt.allocPrint` |
| 99 | Unicode/UTF-8 | `str.len` = bytes, ใช้ `Utf8View` สำหรับ codepoints |
| 100 | String utilities | `startsWith`, `endsWith`, `split`, `tokenize`, `replace` |

### ข้อควรจำเรื่อง UTF-8 (สำคัญมาก!)

```
ASCII (English):
  'A' = 1 byte (0x41)
  
Thai Characters:
  'ก' = 3 bytes (0xE0 0xB8 0x81)
  'สวัสดี' = 18 bytes (6 chars × 3 bytes)
  
Emoji:
  '🌏' = 4 bytes (0xF0 0x9F 0x8C 0x8F)
  
ดังนั้น:
  "สวัสดี".len = 18 (bytes)
  utf8CountCodepoints("สวัสดี") = 6 (characters)
```

### Best Practices

1. **ใช้ `[]const u8`** สำหรับ strings ที่ไม่ต้องการแก้ไข
2. **ใช้ `[]u8`** เฉพาะเมื่อต้องการแก้ไข string
3. **ระวัง `str.len`** คืนค่า bytes ไม่ใช่ characters
4. **ใช้ `Utf8View`** เมื่อต้องการ iterate ทีละ character
5. **ใช้ `bufPrint`** เมื่อรู้ขนาด output ล่วงหน้า (ประหยัด allocation)
6. **ใช้ `allocPrint`** เมื่อไม่รู้ขนาด output
7. **ใช้ `ArrayList(u8)`** สำหรับ string building ที่ซับซ้อน
8. **อย่าลืม `defer allocator.free()`** สำหรับ allocated strings

### Pattern ที่ใช้บ่อย

```zig
// Pattern 1: String comparison
if (std.mem.eql(u8, str1, str2)) { ... }

// Pattern 2: Check prefix/suffix
if (std.mem.startsWith(u8, str, prefix)) { ... }
if (std.mem.endsWith(u8, str, suffix)) { ... }

// Pattern 3: Find substring
if (std.mem.indexOf(u8, haystack, needle)) |pos| {
    const found = haystack[pos..pos + needle.len];
    _ = found;
}

// Pattern 4: Split string
var iter = std.mem.splitScalar(u8, csv, ',');
while (iter.next()) |field| { ... }

// Pattern 5: Build string
var buf: [256]u8 = undefined;
const s = try std.fmt.bufPrint(&buf, "{s}: {d}", .{ name, value });

// Pattern 6: Allocate string
const s = try std.fmt.allocPrint(allocator, "{s}: {d}", .{ name, value });
defer allocator.free(s);

// Pattern 7: Unicode iteration
var view = try std.unicode.Utf8View.init(text);
var it = view.iterator();
while (it.nextCodepointSlice()) |cp_slice| { ... }

// Pattern 8: Trim whitespace
const trimmed = std.mem.trim(u8, input, " \t\n\r");
```

### ตัวอย่าง Quick Reference

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const al = gpa.allocator();
    
    // === BASIC ===
    const s: []const u8 = "Hello, ไทย!";
    _ = s.len;                                   // byte count
    _ = s[0];                                    // first byte
    _ = s[0..5];                                 // slice
    
    // === COMPARISON ===
    _ = std.mem.eql(u8, s, "Hello, ไทย!");       // true
    _ = std.mem.startsWith(u8, s, "Hello");      // true
    _ = std.mem.endsWith(u8, s, "!");            // true
    _ = std.mem.indexOf(u8, s, "ไทย");           // some position
    
    // === CONVERSION ===
    _ = try std.fmt.parseInt(i32, "42", 10);     // 42
    _ = try std.fmt.parseFloat(f64, "3.14");     // 3.14
    
    var buf: [64]u8 = undefined;
    _ = try std.fmt.bufPrint(&buf, "{}", .{42}); // "42"
    
    const allocated = try std.fmt.allocPrint(al, "{}", .{42});
    defer al.free(allocated);
    
    // === UNICODE ===
    _ = try unicode.utf8CountCodepoints(s);      // character count
    _ = unicode.utf8ValidateSlice(s);            // true if valid UTF-8
    
    // === SPLIT ===
    var it = std.mem.splitScalar(u8, "a,b,c", ',');
    while (it.next()) |part| { _ = part; }
    
    // === TRIM ===
    _ = std.mem.trim(u8, "  hello  ", " ");      // "hello"
    
    // === REPLACE ===
    const replaced = try std.mem.replaceOwned(u8, al, "foo bar", "foo", "baz");
    defer al.free(replaced);
    
    std.debug.print("All string operations demonstrated!\n", .{});
}
```

---

## ยินดีด้วย! 🎉

คุณจบ Part 10: Strings และการจัดการข้อความใน Zig แล้ว!

ในภาคนี้คุณได้เรียนรู้:
- ทำไม Zig ถึงไม่มี built-in string type และทำงานอย่างไรกับ `[]const u8`
- วิธีใช้ escape sequences และ multiline strings
- การเปรียบเทียบ, ต่อ, ตัด และค้นหา strings
- การแปลงระหว่าง strings กับ numbers
- ความสำคัญของ UTF-8 และการทำงานกับ Unicode (รวมถึงภาษาไทย)
- String utilities ต่างๆ ที่อยู่ใน `std.mem` และ `std.fmt`

ภาคถัดไปเราจะเรียนรู้เรื่อง **Structs และ Methods** ซึ่งจะทำให้เราสามารถสร้าง data structures ที่ซับซ้อนขึ้นได้!

---

*จบ Part 10 - Steps 91-100*
