# Part 06: การควบคุมการทำงานใน Zig (Control Flow)
## Steps 51-60

---

## บทนำ

การควบคุมการทำงาน (Control Flow) คือหัวใจสำคัญของการเขียนโปรแกรม มันกำหนดว่าโค้ดจะทำงานในลำดับใด เงื่อนไขใดที่ต้องตรวจสอบ และจะจัดการกับสถานการณ์พิเศษอย่างไร

Zig มีโครงสร้างการควบคุมการทำงานที่ทรงพลังและปลอดภัย หลายอย่างที่ดูคุ้นเคยจากภาษาอื่น แต่มีรายละเอียดพิเศษที่ทำให้ Zig โดดเด่น เช่น `if` ที่เป็น expression ได้ `switch` ที่ต้องครอบคลุมทุกกรณี และ `defer`/`errdefer` ที่ช่วยจัดการทรัพยากรได้อย่างสะอาด

ในส่วนนี้เราจะครอบคลุม:
- **Steps 51-54**: การใช้ `if/else` ในรูปแบบต่างๆ รวมถึงการทำงานกับ Optional และ Error Union
- **Steps 55-58**: `switch` statement และ expression ที่ครอบคลุมทุกกรณี
- **Steps 59-60**: `defer` และ `errdefer` สำหรับการจัดการทรัพยากร

---

## Step 51: if/else Expression พื้นฐาน

### แนวคิดพื้นฐาน

`if/else` ใน Zig ทำงานคล้ายกับภาษาอื่นๆ แต่มีความแตกต่างสำคัญคือ ใน Zig เงื่อนไขต้องเป็น `bool` เท่านั้น ไม่สามารถใช้ตัวเลขหรือ pointer แทน `true`/`false` ได้เหมือน C

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 10;

    // รูปแบบพื้นฐาน
    if (x > 5) {
        std.debug.print("x มากกว่า 5\n", .{});
    }

    // if กับ else
    if (x > 20) {
        std.debug.print("x มากกว่า 20\n", .{});
    } else {
        std.debug.print("x ไม่เกิน 20\n", .{});
    }

    // if-else if-else
    if (x < 0) {
        std.debug.print("x เป็นลบ\n", .{});
    } else if (x == 0) {
        std.debug.print("x เป็นศูนย์\n", .{});
    } else if (x < 10) {
        std.debug.print("x เป็นเลขหลักเดียว\n", .{});
    } else {
        std.debug.print("x มากกว่าหรือเท่ากับ 10\n", .{});
    }
}
```

### ความแตกต่างจาก C/C++

ใน C คุณสามารถเขียนได้:
```c
// C code - ใช้ได้ใน C แต่ไม่ได้ใน Zig
int x = 1;
if (x) {  // ใน C ค่าที่ไม่ใช่ 0 ถือว่าเป็น true
    printf("true\n");
}
```

ใน Zig คุณต้องเขียนชัดเจน:
```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 1;

    // ต้องระบุเงื่อนไขให้ชัดเจน
    if (x != 0) {
        std.debug.print("x ไม่ใช่ศูนย์\n", .{});
    }

    // หรือใช้การเปรียบเทียบ
    if (x > 0) {
        std.debug.print("x เป็นบวก\n", .{});
    }
}
```

### เงื่อนไขซับซ้อนด้วย and/or

```zig
const std = @import("std");

pub fn main() void {
    const age: u32 = 25;
    const hasLicense: bool = true;
    const hasCar: bool = false;

    // ใช้ and (&&) และ or (||)
    if (age >= 18 and hasLicense) {
        std.debug.print("สามารถขับรถได้\n", .{});
    }

    if (hasLicense or hasCar) {
        std.debug.print("มีใบขับขี่หรือมีรถ\n", .{});
    }

    // ใช้ not (!)
    if (!hasCar) {
        std.debug.print("ไม่มีรถ\n", .{});
    }

    // ซ้อนกัน
    if (age >= 18 and (hasLicense or hasCar)) {
        std.debug.print("เงื่อนไขซับซ้อน\n", .{});
    }
}
```

### Block ใน if

```zig
const std = @import("std");

pub fn main() void {
    const score: u32 = 85;

    if (score >= 80) {
        // สามารถมีหลาย statement ใน block
        const grade = "B";
        const message = "ผ่านด้วยเกรดที่ดี";
        std.debug.print("เกรด: {s} - {s}\n", .{ grade, message });
    } else {
        const grade = "C หรือต่ำกว่า";
        std.debug.print("เกรด: {s}\n", .{grade});
    }
}
```

---

## Step 52: if ใน Zig เป็น Expression (ไม่ใช่ Statement)

### ความหมายของ Expression

ความแตกต่างสำคัญระหว่าง Zig กับ C/Java คือ `if` ใน Zig สามารถเป็น **expression** ได้ หมายความว่ามันมีค่า (value) และสามารถใช้ตรงที่ต้องการค่าได้

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 10;

    // if เป็น expression - ใช้แทนค่าตรงๆ ได้
    const result = if (x > 5) "มาก" else "น้อย";
    std.debug.print("ผลลัพธ์: {s}\n", .{result});

    // เทียบกับ C ที่ต้องใช้ ternary operator: x > 5 ? "มาก" : "น้อย"
    // Zig ใช้ if-else แทน ternary ได้เลย
}
```

### การใช้ if-expression กับตัวเลข

```zig
const std = @import("std");

pub fn main() void {
    const temperature: i32 = 30;

    // กำหนดค่าโดยใช้ if expression
    const clothing = if (temperature > 28) "เสื้อแขนสั้น" else "เสื้อแขนยาว";
    std.debug.print("ควรใส่: {s}\n", .{clothing});

    // ใช้ใน expression คำนวณ
    const bonus: i32 = if (temperature > 35) 1000 else 0;
    const salary: i32 = 50000 + bonus;
    std.debug.print("เงินเดือน: {d}\n", .{salary});
}
```

### if-expression ต้องมี else เสมอเมื่อใช้เป็น value

```zig
const std = @import("std");

pub fn main() void {
    const flag = true;

    // ถูกต้อง: มีทั้ง if และ else
    const value: i32 = if (flag) 1 else 0;
    std.debug.print("value: {d}\n", .{value});

    // ข้อผิดพลาดที่พบบ่อย:
    // const bad = if (flag) 1; // ERROR! ต้องมี else เมื่อใช้เป็น expression
}
```

### if-expression กับ block

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 42;

    // block ใน if expression สามารถ return ค่าด้วย break
    const result = if (x > 0) blk: {
        const doubled = x * 2;
        const message = "บวก";
        _ = message; // ไม่ใช้ message จริงๆ แต่แสดงให้เห็น
        break :blk doubled;
    } else blk: {
        const abs = -x;
        break :blk abs;
    };

    std.debug.print("ผลลัพธ์: {d}\n", .{result});
}
```

### การใช้ if-expression แบบ inline ใน function call

```zig
const std = @import("std");

fn getLabel(value: i32) []const u8 {
    return if (value > 0) "บวก" else if (value < 0) "ลบ" else "ศูนย์";
}

pub fn main() void {
    std.debug.print("{s}\n", .{getLabel(5)});
    std.debug.print("{s}\n", .{getLabel(-3)});
    std.debug.print("{s}\n", .{getLabel(0)});
}
```

### ตัวอย่างการประยุกต์ใช้

```zig
const std = @import("std");

pub fn main() void {
    const items = [_]i32{ 1, -2, 3, -4, 5 };

    var positive_sum: i32 = 0;
    var negative_sum: i32 = 0;

    for (items) |item| {
        // ใช้ if expression เพื่อแยกบวกและลบ
        positive_sum += if (item > 0) item else 0;
        negative_sum += if (item < 0) item else 0;
    }

    std.debug.print("ผลรวมบวก: {d}\n", .{positive_sum});
    std.debug.print("ผลรวมลบ: {d}\n", .{negative_sum});

    // ใช้ if expression ใน format string
    const is_positive = positive_sum > @abs(negative_sum);
    std.debug.print("ผลรวมรวม{s}บวก\n", .{if (is_positive) "เป็น" else "ไม่เป็น"});
}
```

---

## Step 53: if กับ Optional (if (optional)) |value|

### Optional Type ใน Zig

ใน Zig `?T` หมายถึง Optional ซึ่งอาจมีค่าเป็น `T` หรือ `null` ก็ได้ การใช้ `if` กับ Optional ช่วยให้เราแกะค่าออกมาได้อย่างปลอดภัย

```zig
const std = @import("std");

pub fn main() void {
    // Optional variable
    const maybe_number: ?i32 = 42;
    const no_number: ?i32 = null;

    // การแกะค่าจาก Optional ด้วย if
    if (maybe_number) |num| {
        std.debug.print("มีตัวเลข: {d}\n", .{num});
    }

    if (no_number) |num| {
        std.debug.print("มีตัวเลข: {d}\n", .{num});
    } else {
        std.debug.print("ไม่มีตัวเลข\n", .{});
    }
}
```

### รูปแบบ if (optional) |value|

```zig
const std = @import("std");

// ฟังก์ชันที่อาจ return null
fn findUser(id: u32) ?[]const u8 {
    return switch (id) {
        1 => "สมชาย",
        2 => "สมหญิง",
        3 => "สมศรี",
        else => null, // ไม่พบผู้ใช้
    };
}

pub fn main() void {
    // ตัวอย่างที่ 1: พบผู้ใช้
    if (findUser(1)) |name| {
        std.debug.print("พบผู้ใช้: {s}\n", .{name});
    } else {
        std.debug.print("ไม่พบผู้ใช้\n", .{});
    }

    // ตัวอย่างที่ 2: ไม่พบผู้ใช้
    if (findUser(99)) |name| {
        std.debug.print("พบผู้ใช้: {s}\n", .{name});
    } else {
        std.debug.print("ไม่พบผู้ใช้ ID 99\n", .{});
    }
}
```

### Optional Pointer

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

pub fn main() void {
    var node2 = Node{ .value = 20, .next = null };
    var node1 = Node{ .value = 10, .next = &node2 };

    // เดิน linked list ด้วย Optional pointer
    var current: ?*Node = &node1;
    while (current) |node| {
        std.debug.print("ค่า: {d}\n", .{node.value});
        current = node.next;
    }
}
```

### การแก้ไขค่าใน Optional

```zig
const std = @import("std");

pub fn main() void {
    var maybe_value: ?i32 = 10;

    // ใช้ |*ptr| เพื่อแก้ไขค่าใน Optional
    if (maybe_value) |*ptr| {
        ptr.* = ptr.* * 2; // คูณสองค่าเดิม
    }

    if (maybe_value) |val| {
        std.debug.print("ค่าหลังแก้ไข: {d}\n", .{val});
    }
}
```

### การซ้อน Optional

```zig
const std = @import("std");

const Config = struct {
    database: ?DatabaseConfig,
};

const DatabaseConfig = struct {
    host: []const u8,
    port: ?u16,
};

pub fn main() void {
    const config = Config{
        .database = DatabaseConfig{
            .host = "localhost",
            .port = 5432,
        },
    };

    // ซ้อน if สำหรับ Optional ที่ซ้อนกัน
    if (config.database) |db| {
        std.debug.print("Host: {s}\n", .{db.host});
        if (db.port) |port| {
            std.debug.print("Port: {d}\n", .{port});
        } else {
            std.debug.print("ใช้ port default\n", .{});
        }
    }
}
```

### เปรียบเทียบกับภาษาอื่น

```zig
// Zig - ปลอดภัยจาก null pointer
if (findUser(1)) |user| {
    // user ถูกรับประกันว่าไม่ใช่ null ในที่นี้
    _ = user;
}

// JavaScript - ต้องระวัง undefined/null
// const user = findUser(1);
// if (user !== null && user !== undefined) {
//     // ใช้ user
// }

// Rust - คล้ายกับ Zig
// if let Some(user) = find_user(1) {
//     // ใช้ user
// }
```

---

## Step 54: if กับ Error Union (if (result)) |value| else |err|

### Error Union ใน Zig

`!T` หรือ `ErrorSet!T` คือ Error Union ซึ่งอาจเป็น error หรือค่า `T` ก็ได้

```zig
const std = @import("std");

// ฟังก์ชันที่อาจเกิด error
fn divide(a: i32, b: i32) !i32 {
    if (b == 0) {
        return error.DivisionByZero;
    }
    return @divTrunc(a, b);
}

pub fn main() void {
    // การจัดการ error ด้วย if
    if (divide(10, 2)) |result| {
        std.debug.print("ผลหาร: {d}\n", .{result});
    } else |err| {
        std.debug.print("เกิด error: {}\n", .{err});
    }

    // กรณีเกิด error
    if (divide(10, 0)) |result| {
        std.debug.print("ผลหาร: {d}\n", .{result});
    } else |err| {
        std.debug.print("เกิด error: {}\n", .{err});
    }
}
```

### จัดการ error หลายประเภท

```zig
const std = @import("std");

const FileError = error{
    NotFound,
    PermissionDenied,
    TooLarge,
};

fn openFile(path: []const u8) FileError![]const u8 {
    if (std.mem.eql(u8, path, "/secret")) {
        return FileError.PermissionDenied;
    }
    if (std.mem.eql(u8, path, "/missing")) {
        return FileError.NotFound;
    }
    return "เนื้อหาไฟล์";
}

pub fn main() void {
    const paths = [_][]const u8{ "/normal", "/secret", "/missing" };

    for (paths) |path| {
        if (openFile(path)) |content| {
            std.debug.print("ไฟล์ {s}: {s}\n", .{ path, content });
        } else |err| {
            // สามารถตรวจสอบประเภท error ได้
            switch (err) {
                FileError.NotFound => std.debug.print("ไม่พบไฟล์: {s}\n", .{path}),
                FileError.PermissionDenied => std.debug.print("ไม่มีสิทธิ์เข้าถึง: {s}\n", .{path}),
                FileError.TooLarge => std.debug.print("ไฟล์ใหญ่เกินไป: {s}\n", .{path}),
            }
        }
    }
}
```

### Error Union กับ Optional พร้อมกัน

```zig
const std = @import("std");

fn parseNumber(str: []const u8) !?i32 {
    if (str.len == 0) return null; // ไม่มีค่า
    return try std.fmt.parseInt(i32, str, 10);
}

pub fn main() void {
    const inputs = [_][]const u8{ "42", "", "abc", "-10" };

    for (inputs) |input| {
        if (parseNumber(input)) |maybe_num| {
            if (maybe_num) |num| {
                std.debug.print("'{s}' -> {d}\n", .{ input, num });
            } else {
                std.debug.print("'{s}' -> (ว่าง)\n", .{input});
            }
        } else |err| {
            std.debug.print("'{s}' -> error: {}\n", .{ input, err });
        }
    }
}
```

### การใช้งานจริงกับ stdlib

```zig
const std = @import("std");

pub fn main() void {
    // std.fmt.parseInt คือ error union
    const input = "12345";

    if (std.fmt.parseInt(u32, input, 10)) |number| {
        std.debug.print("แปลงสำเร็จ: {d}\n", .{number});
    } else |err| {
        std.debug.print("แปลงไม่สำเร็จ: {}\n", .{err});
    }
}
```

### เปรียบเทียบกับ try/catch ในภาษาอื่น

```zig
// Zig - ชัดเจนและไม่มี exception ที่ไม่คาดหมาย
if (riskyOperation()) |value| {
    // ทำงานกับ value
    _ = value;
} else |err| {
    // จัดการ error
    _ = err;
}

// Go - คล้ายกัน แต่ใช้ค่า return สองตัว
// result, err := riskyOperation()
// if err != nil {
//     // จัดการ error
// }

// Rust - ใช้ match กับ Result
// match risky_operation() {
//     Ok(value) => { /* ใช้ value */ },
//     Err(err) => { /* จัดการ error */ },
// }
```

---

## Step 55: switch Statement พื้นฐาน

### แนวคิดพื้นฐาน

`switch` ใน Zig คล้ายกับภาษาอื่น แต่มีคุณสมบัติพิเศษสำคัญ: **ต้องครอบคลุมทุกกรณีที่เป็นไปได้** (exhaustive) ซึ่งหมายความว่า compiler จะตรวจสอบให้แน่ใจว่าไม่มีกรณีที่ตกหล่น

```zig
const std = @import("std");

pub fn main() void {
    const day: u8 = 3;

    switch (day) {
        1 => std.debug.print("วันจันทร์\n", .{}),
        2 => std.debug.print("วันอังคาร\n", .{}),
        3 => std.debug.print("วันพุธ\n", .{}),
        4 => std.debug.print("วันพฤหัสบดี\n", .{}),
        5 => std.debug.print("วันศุกร์\n", .{}),
        6 => std.debug.print("วันเสาร์\n", .{}),
        7 => std.debug.print("วันอาทิตย์\n", .{}),
        else => std.debug.print("วันที่ไม่รู้จัก\n", .{}),
    }
}
```

### switch กับ Block

```zig
const std = @import("std");

pub fn main() void {
    const grade: u8 = 'B';

    switch (grade) {
        'A' => {
            std.debug.print("ดีเยี่ยม!\n", .{});
            std.debug.print("คะแนน 90-100\n", .{});
        },
        'B' => {
            std.debug.print("ดี\n", .{});
            std.debug.print("คะแนน 80-89\n", .{});
        },
        'C' => {
            std.debug.print("พอใช้\n", .{});
            std.debug.print("คะแนน 70-79\n", .{});
        },
        'D' => {
            std.debug.print("ผ่าน\n", .{});
            std.debug.print("คะแนน 60-69\n", .{});
        },
        'F' => {
            std.debug.print("ตก\n", .{});
            std.debug.print("คะแนนต่ำกว่า 60\n", .{});
        },
        else => {
            std.debug.print("เกรดไม่ถูกต้อง\n", .{});
        },
    }
}
```

### หลายค่าใน Case เดียว

```zig
const std = @import("std");

pub fn main() void {
    const month: u8 = 4;

    // หลายค่าใน arm เดียวด้วย comma
    const days_in_month: u8 = switch (month) {
        1, 3, 5, 7, 8, 10, 12 => 31,
        4, 6, 9, 11 => 30,
        2 => 28, // ละเว้น leap year
        else => unreachable,
    };

    std.debug.print("เดือน {d} มี {d} วัน\n", .{ month, days_in_month });
}
```

### ความแตกต่างจาก C

ใน C `switch` มี "fall-through" คือถ้าไม่มี `break` จะทำงานต่อไปใน case ถัดไป แต่ใน Zig ไม่มี fall-through:

```c
// C - มี fall-through (อันตราย!)
switch (x) {
    case 1:
        printf("หนึ่ง\n");
        // ไม่มี break จะไหลลงไป case 2!
    case 2:
        printf("หนึ่งหรือสอง\n");
        break;
}
```

```zig
// Zig - ไม่มี fall-through เลย
switch (x) {
    1 => std.debug.print("หนึ่ง\n", .{}),
    2 => std.debug.print("สอง\n", .{}),
    else => {},
}
```

### else arm

```zig
const std = @import("std");

pub fn main() void {
    const code: u16 = 404;

    switch (code) {
        200 => std.debug.print("OK\n", .{}),
        201 => std.debug.print("Created\n", .{}),
        400 => std.debug.print("Bad Request\n", .{}),
        401 => std.debug.print("Unauthorized\n", .{}),
        403 => std.debug.print("Forbidden\n", .{}),
        404 => std.debug.print("Not Found\n", .{}),
        500 => std.debug.print("Internal Server Error\n", .{}),
        else => std.debug.print("HTTP Status: {d}\n", .{code}),
    }
}
```

---

## Step 56: switch กับ Enum, Integer Ranges

### switch กับ Enum

เมื่อใช้ `switch` กับ `enum` ใน Zig จะต้องครอบคลุมทุก variant ไม่งั้น compiler จะแจ้ง error ทำให้มั่นใจได้ว่าไม่มีกรณีที่ลืม

```zig
const std = @import("std");

const Direction = enum {
    North,
    South,
    East,
    West,
};

pub fn main() void {
    const dir = Direction.North;

    // switch กับ enum - ต้องครอบคลุมทุก variant
    switch (dir) {
        .North => std.debug.print("มุ่งหน้าเหนือ\n", .{}),
        .South => std.debug.print("มุ่งหน้าใต้\n", .{}),
        .East => std.debug.print("มุ่งหน้าตะวันออก\n", .{}),
        .West => std.debug.print("มุ่งหน้าตะวันตก\n", .{}),
    }
    // ไม่ต้องมี else เพราะครอบคลุมหมดแล้ว
}
```

### ประโยชน์ของ Exhaustive switch

```zig
const std = @import("std");

const Season = enum {
    Spring,
    Summer,
    Autumn,
    Winter,
};

fn getActivity(season: Season) []const u8 {
    return switch (season) {
        .Spring => "ปลูกดอกไม้",
        .Summer => "ว่ายน้ำ",
        .Autumn => "เก็บผลไม้",
        .Winter => "ทำหิมะ",
        // ถ้าเพิ่ม variant ใหม่ใน enum แล้วไม่อัพเดท switch
        // Compiler จะแจ้ง error ทันที!
    };
}

pub fn main() void {
    const seasons = [_]Season{ .Spring, .Summer, .Autumn, .Winter };
    for (seasons) |season| {
        std.debug.print("{s}: {s}\n", .{ @tagName(season), getActivity(season) });
    }
}
```

### Integer Ranges ใน switch

```zig
const std = @import("std");

pub fn main() void {
    const score: u32 = 75;

    // ใช้ range ด้วย syntax a...b (inclusive)
    const grade = switch (score) {
        90...100 => 'A',
        80...89 => 'B',
        70...79 => 'C',
        60...69 => 'D',
        0...59 => 'F',
        else => unreachable, // ป้องกัน compiler error
    };

    std.debug.print("คะแนน {d} ได้เกรด {c}\n", .{ score, grade });
}
```

### ผสม Ranges และค่าเดี่ยว

```zig
const std = @import("std");

pub fn main() void {
    const age: u32 = 15;

    const category = switch (age) {
        0...2 => "ทารก",
        3...12 => "เด็ก",
        13...17 => "วัยรุ่น",
        18...59 => "ผู้ใหญ่",
        60...79 => "ผู้สูงอายุ",
        80, 81, 82 => "สูงอายุมาก", // ค่าเดี่ยวหลายค่า
        83...120 => "สูงอายุมากๆ",
        else => "อายุไม่ถูกต้อง",
    };

    std.debug.print("อายุ {d} ปี: {s}\n", .{ age, category });
}
```

### switch กับ Packed Enum

```zig
const std = @import("std");

const Color = enum(u8) {
    Red = 0,
    Green = 1,
    Blue = 2,
    Alpha = 255,
};

pub fn main() void {
    const color = Color.Green;

    switch (color) {
        .Red => std.debug.print("สีแดง (R)\n", .{}),
        .Green => std.debug.print("สีเขียว (G)\n", .{}),
        .Blue => std.debug.print("สีน้ำเงิน (B)\n", .{}),
        .Alpha => std.debug.print("ความโปร่งใส (A)\n", .{}),
    }

    // เข้าถึงค่า integer ของ enum
    std.debug.print("ค่า: {d}\n", .{@intFromEnum(color)});
}
```

### ตัวอย่างการประยุกต์ใช้: State Machine

```zig
const std = @import("std");

const TrafficLight = enum {
    Red,
    Yellow,
    Green,
};

fn nextState(current: TrafficLight) TrafficLight {
    return switch (current) {
        .Red => .Green,
        .Green => .Yellow,
        .Yellow => .Red,
    };
}

fn getWaitTime(light: TrafficLight) u32 {
    return switch (light) {
        .Red => 30,
        .Yellow => 5,
        .Green => 25,
    };
}

pub fn main() void {
    var light = TrafficLight.Red;

    var i: u32 = 0;
    while (i < 6) : (i += 1) {
        std.debug.print("สัญญาณ: {s} (รอ {d} วินาที)\n", .{
            @tagName(light),
            getWaitTime(light),
        });
        light = nextState(light);
    }
}
```

---

## Step 57: switch เป็น Expression

### switch-expression ส่งค่ากลับ

เหมือนกับ `if` ที่เป็น expression ได้ `switch` ก็เป็น expression ได้เช่นกัน

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 3;

    // switch เป็น expression
    const name = switch (x) {
        1 => "หนึ่ง",
        2 => "สอง",
        3 => "สาม",
        4 => "สี่",
        5 => "ห้า",
        else => "อื่นๆ",
    };

    std.debug.print("{d} = {s}\n", .{ x, name });
}
```

### ใช้ switch-expression ใน variable declaration

```zig
const std = @import("std");

const OS = enum { Windows, Linux, MacOS };

pub fn main() void {
    const current_os = OS.Linux;

    // switch expression กำหนดค่าตัวแปร
    const shell = switch (current_os) {
        .Windows => "cmd.exe",
        .Linux => "/bin/bash",
        .MacOS => "/bin/zsh",
    };

    const path_sep = switch (current_os) {
        .Windows => '\\',
        .Linux, .MacOS => '/',
    };

    std.debug.print("Shell: {s}\n", .{shell});
    std.debug.print("Path separator: {c}\n", .{path_sep});
}
```

### switch-expression กับ block

```zig
const std = @import("std");

const Priority = enum { Low, Medium, High, Critical };

pub fn main() void {
    const priority = Priority.High;

    const action = switch (priority) {
        .Low => blk: {
            const delay = 7 * 24; // 7 วัน เป็นชั่วโมง
            _ = delay;
            break :blk "จัดการภายใน 1 สัปดาห์";
        },
        .Medium => blk: {
            const delay = 24; // 1 วัน เป็นชั่วโมง
            _ = delay;
            break :blk "จัดการภายใน 1 วัน";
        },
        .High => blk: {
            const delay = 4; // 4 ชั่วโมง
            _ = delay;
            break :blk "จัดการภายใน 4 ชั่วโมง";
        },
        .Critical => blk: {
            break :blk "จัดการทันที!";
        },
    };

    std.debug.print("ระดับ {s}: {s}\n", .{ @tagName(priority), action });
}
```

### switch-expression ใน function return

```zig
const std = @import("std");

const DayType = enum { Weekday, Weekend };

fn getDayType(day: u8) DayType {
    return switch (day) {
        1...5 => .Weekday,
        6, 7 => .Weekend,
        else => unreachable,
    };
}

fn getWorkHours(day_type: DayType) u8 {
    return switch (day_type) {
        .Weekday => 8,
        .Weekend => 0,
    };
}

pub fn main() void {
    var day: u8 = 1;
    while (day <= 7) : (day += 1) {
        const day_type = getDayType(day);
        const hours = getWorkHours(day_type);
        std.debug.print("วัน {d}: {s} - ทำงาน {d} ชั่วโมง\n", .{
            day,
            @tagName(day_type),
            hours,
        });
    }
}
```

### Inline switch กับ comptime

```zig
const std = @import("std");

fn typeDescription(comptime T: type) []const u8 {
    return switch (T) {
        i32 => "จำนวนเต็ม 32-bit",
        u32 => "จำนวนเต็มบวก 32-bit",
        f32 => "ทศนิยม 32-bit",
        f64 => "ทศนิยม 64-bit",
        bool => "ตรรกะ",
        []const u8 => "ข้อความ",
        else => "ชนิดอื่น",
    };
}

pub fn main() void {
    std.debug.print("i32: {s}\n", .{typeDescription(i32)});
    std.debug.print("f64: {s}\n", .{typeDescription(f64)});
    std.debug.print("bool: {s}\n", .{typeDescription(bool)});
}
```

---

## Step 58: switch กับ Tagged Union

### Tagged Union คืออะไร

Tagged Union (หรือเรียกว่า discriminated union) คือชนิดข้อมูลที่สามารถเก็บค่าได้หลายชนิด แต่ในเวลาหนึ่งจะเก็บได้แค่ชนิดเดียว พร้อมกับ tag บอกว่าตอนนี้เก็บชนิดใดอยู่

```zig
const std = @import("std");

// Tagged Union - เก็บได้หลายชนิด แต่ทีละชนิด
const Value = union(enum) {
    integer: i64,
    float: f64,
    boolean: bool,
    text: []const u8,
};

pub fn main() void {
    const values = [_]Value{
        .{ .integer = 42 },
        .{ .float = 3.14 },
        .{ .boolean = true },
        .{ .text = "สวัสดี" },
    };

    for (values) |val| {
        switch (val) {
            .integer => |n| std.debug.print("จำนวนเต็ม: {d}\n", .{n}),
            .float => |f| std.debug.print("ทศนิยม: {d:.2}\n", .{f}),
            .boolean => |b| std.debug.print("ตรรกะ: {}\n", .{b}),
            .text => |s| std.debug.print("ข้อความ: {s}\n", .{s}),
        }
    }
}
```

### Tagged Union สำหรับ JSON-like data

```zig
const std = @import("std");

const JsonValue = union(enum) {
    null_value,
    bool_value: bool,
    number: f64,
    string: []const u8,

    pub fn print(self: JsonValue) void {
        switch (self) {
            .null_value => std.debug.print("null", .{}),
            .bool_value => |b| std.debug.print("{}", .{b}),
            .number => |n| std.debug.print("{d}", .{n}),
            .string => |s| std.debug.print("\"{s}\"", .{s}),
        }
    }
};

pub fn main() void {
    const values = [_]JsonValue{
        .null_value,
        .{ .bool_value = true },
        .{ .number = 42.5 },
        .{ .string = "hello" },
    };

    std.debug.print("[", .{});
    for (values, 0..) |val, i| {
        if (i > 0) std.debug.print(", ", .{});
        val.print();
    }
    std.debug.print("]\n", .{});
}
```

### Tagged Union กับ Error Handling

```zig
const std = @import("std");

const ParseResult = union(enum) {
    success: i32,
    invalid_format,
    overflow,
    empty_input,
};

fn parseInteger(input: []const u8) ParseResult {
    if (input.len == 0) return .empty_input;

    const result = std.fmt.parseInt(i32, input, 10) catch |err| {
        return switch (err) {
            error.InvalidCharacter => .invalid_format,
            error.Overflow => .overflow,
        };
    };

    return .{ .success = result };
}

pub fn main() void {
    const inputs = [_][]const u8{ "123", "-456", "", "abc", "99999999999" };

    for (inputs) |input| {
        const result = parseInteger(input);
        switch (result) {
            .success => |n| std.debug.print("'{s}' -> {d}\n", .{ input, n }),
            .invalid_format => std.debug.print("'{s}' -> รูปแบบผิด\n", .{input}),
            .overflow => std.debug.print("'{s}' -> ตัวเลขใหญ่เกินไป\n", .{input}),
            .empty_input => std.debug.print("'{s}' -> ว่างเปล่า\n", .{input}),
        }
    }
}
```

### แก้ไขค่าใน Tagged Union

```zig
const std = @import("std");

const Shape = union(enum) {
    circle: f64, // radius
    rectangle: struct { width: f64, height: f64 },
    triangle: struct { base: f64, height: f64 },
};

fn area(shape: Shape) f64 {
    return switch (shape) {
        .circle => |r| std.math.pi * r * r,
        .rectangle => |rect| rect.width * rect.height,
        .triangle => |tri| 0.5 * tri.base * tri.height,
    };
}

fn perimeter(shape: Shape) f64 {
    return switch (shape) {
        .circle => |r| 2 * std.math.pi * r,
        .rectangle => |rect| 2 * (rect.width + rect.height),
        .triangle => |tri| {
            // ประมาณด้านทั้งสาม (ใช้สูตรง่ายๆ)
            const hyp = std.math.sqrt(tri.base * tri.base + tri.height * tri.height);
            return tri.base + tri.height + hyp;
        },
    };
}

pub fn main() void {
    const shapes = [_]Shape{
        .{ .circle = 5.0 },
        .{ .rectangle = .{ .width = 4.0, .height = 6.0 } },
        .{ .triangle = .{ .base = 3.0, .height = 4.0 } },
    };

    for (shapes) |shape| {
        const name = switch (shape) {
            .circle => "วงกลม",
            .rectangle => "สี่เหลี่ยม",
            .triangle => "สามเหลี่ยม",
        };
        std.debug.print("{s}: พื้นที่ = {d:.2}, เส้นรอบรูป = {d:.2}\n", .{
            name,
            area(shape),
            perimeter(shape),
        });
    }
}
```

### Tagged Union เปรียบเทียบกับ OOP

```zig
// Zig approach - Tagged Union + switch (data-oriented)
const std = @import("std");

const Animal = union(enum) {
    dog: struct { name: []const u8, breed: []const u8 },
    cat: struct { name: []const u8, indoor: bool },
    fish: struct { name: []const u8, species: []const u8 },
};

fn makeSound(animal: Animal) []const u8 {
    return switch (animal) {
        .dog => "โฮ่ง!",
        .cat => "เมี้ยว~",
        .fish => "...(ไม่มีเสียง)",
    };
}

pub fn main() void {
    const animals = [_]Animal{
        .{ .dog = .{ .name = "บัดดี้", .breed = "Golden Retriever" } },
        .{ .cat = .{ .name = "วิสกี้", .indoor = true } },
        .{ .fish = .{ .name = "นีโม่", .species = "Clownfish" } },
    };

    for (animals) |animal| {
        const name = switch (animal) {
            .dog => |d| d.name,
            .cat => |c| c.name,
            .fish => |f| f.name,
        };
        std.debug.print("{s}: {s}\n", .{ name, makeSound(animal) });
    }
}
```

---

## Step 59: defer Statement

### ความหมายของ defer

`defer` คือ statement ที่บอกให้รันโค้ดนั้น **เมื่อออกจาก scope ปัจจุบัน** ไม่ว่าจะออกแบบปกติหรือผ่าน error ก็ตาม มีประโยชน์มากสำหรับการทำความสะอาด (cleanup) ทรัพยากร

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("เริ่ม\n", .{});

    defer std.debug.print("defer รันตอนจบ\n", .{});

    std.debug.print("ตรงกลาง\n", .{});

    std.debug.print("สิ้นสุด\n", .{});
    // defer จะรันที่นี่ก่อนออกจาก main
}
// Output:
// เริ่ม
// ตรงกลาง
// สิ้นสุด
// defer รันตอนจบ
```

### defer หลายตัว - LIFO Order

```zig
const std = @import("std");

pub fn main() void {
    defer std.debug.print("defer 1\n", .{});
    defer std.debug.print("defer 2\n", .{});
    defer std.debug.print("defer 3\n", .{});

    std.debug.print("ทำงานปกติ\n", .{});

    // defer รันในลำดับย้อนกลับ (LIFO: Last In First Out)
    // จะแสดง:
    // ทำงานปกติ
    // defer 3
    // defer 2
    // defer 1
}
```

### defer สำหรับ Resource Cleanup

```zig
const std = @import("std");

const Resource = struct {
    name: []const u8,
    id: u32,

    pub fn open(name: []const u8) Resource {
        std.debug.print("เปิด resource: {s}\n", .{name});
        return .{ .name = name, .id = 1 };
    }

    pub fn close(self: Resource) void {
        std.debug.print("ปิด resource: {s}\n", .{self.name});
    }

    pub fn process(self: Resource) void {
        std.debug.print("ประมวลผล resource: {s} (id={d})\n", .{ self.name, self.id });
    }
};

pub fn doWork() void {
    const res = Resource.open("database");
    defer res.close(); // รับประกันว่าจะปิดเสมอ!

    res.process();
    // ถ้ามี return หลายจุด ก็ไม่ต้องกังวลว่าจะลืมปิด
    // defer จะจัดการให้
}

pub fn main() void {
    doWork();
    std.debug.print("จบการทำงาน\n", .{});
}
```

### defer กับ Memory Allocation

```zig
const std = @import("std");

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    // จัดสรร memory
    const buffer = try allocator.alloc(u8, 100);
    defer allocator.free(buffer); // รับประกันว่าจะ free เสมอ

    // ใช้งาน buffer
    @memset(buffer, 0);
    buffer[0] = 'H';
    buffer[1] = 'i';

    std.debug.print("buffer[0..2]: {s}\n", .{buffer[0..2]});
    // allocator.free จะรันที่นี่โดยอัตโนมัติ
}
```

### defer ใน Loop

```zig
const std = @import("std");

fn processItems(items: []const u32) void {
    for (items) |item| {
        // defer นี้รันทุกครั้งที่ loop body สิ้นสุด
        defer std.debug.print("  จบการประมวลผล item {d}\n", .{item});

        std.debug.print("ประมวลผล item {d}...\n", .{item});

        if (item == 2) {
            std.debug.print("  item 2 ต้องใช้เวลาพิเศษ\n", .{});
        }
    }
}

pub fn main() void {
    const items = [_]u32{ 1, 2, 3 };
    processItems(&items);
}
```

### defer กับ early return

```zig
const std = @import("std");

fn findAndProcess(data: []const i32, target: i32) bool {
    std.debug.print("เริ่มค้นหา {d}\n", .{target});
    defer std.debug.print("สิ้นสุดการค้นหา {d}\n", .{target});

    for (data) |item| {
        if (item == target) {
            std.debug.print("พบ {d}!\n", .{target});
            return true; // defer จะรันแม้จะ return ที่นี่
        }
    }

    std.debug.print("ไม่พบ {d}\n", .{target});
    return false; // defer จะรันที่นี่ด้วย
}

pub fn main() void {
    const data = [_]i32{ 1, 2, 3, 4, 5 };

    _ = findAndProcess(&data, 3);
    std.debug.print("---\n", .{});
    _ = findAndProcess(&data, 10);
}
```

### defer กับ Scope ย่อย

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("ก่อน scope\n", .{});

    {
        // scope ย่อย
        defer std.debug.print("defer ใน scope ย่อย\n", .{});
        std.debug.print("ใน scope ย่อย\n", .{});
    } // defer รันที่นี่ เมื่อออกจาก scope ย่อย

    std.debug.print("หลัง scope\n", .{});
}
```

### ตัวอย่างการประยุกต์ใช้จริง: Mutex Lock

```zig
const std = @import("std");

// จำลอง mutex
const Mutex = struct {
    locked: bool = false,

    pub fn lock(self: *Mutex) void {
        self.locked = true;
        std.debug.print("Mutex locked\n", .{});
    }

    pub fn unlock(self: *Mutex) void {
        self.locked = false;
        std.debug.print("Mutex unlocked\n", .{});
    }
};

var global_mutex = Mutex{};

fn criticalSection(value: i32) void {
    global_mutex.lock();
    defer global_mutex.unlock(); // รับประกันว่าจะ unlock เสมอ

    std.debug.print("ทำงานใน critical section: {d}\n", .{value});

    if (value < 0) {
        std.debug.print("ค่าลบ - ออกเร็ว\n", .{});
        return; // unlock จะถูกเรียกโดยอัตโนมัติ!
    }

    std.debug.print("ทำงานต่อ...\n", .{});
}

pub fn main() void {
    criticalSection(10);
    std.debug.print("---\n", .{});
    criticalSection(-5);
}
```

---

## Step 60: errdefer Statement

### ความหมายของ errdefer

`errdefer` คล้ายกับ `defer` แต่รันเฉพาะเมื่อออกจาก scope **ด้วย error เท่านั้น** ถ้าออกแบบปกติ (return ค่า) จะไม่รัน

```zig
const std = @import("std");

fn riskyOperation(succeed: bool) !void {
    std.debug.print("เริ่มทำงาน\n", .{});

    // errdefer รันเฉพาะเมื่อเกิด error
    errdefer std.debug.print("เกิด error! ทำ cleanup\n", .{});

    // defer รันเสมอ
    defer std.debug.print("defer ปกติ\n", .{});

    if (!succeed) {
        return error.OperationFailed;
    }

    std.debug.print("สำเร็จ!\n", .{});
}

pub fn main() void {
    std.debug.print("=== กรณีสำเร็จ ===\n", .{});
    riskyOperation(true) catch {};

    std.debug.print("\n=== กรณีล้มเหลว ===\n", .{});
    riskyOperation(false) catch {};
}
```

### errdefer กับ Memory Management

```zig
const std = @import("std");

const Resource = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, size: usize) !Resource {
        const data = try allocator.alloc(u8, size);
        errdefer allocator.free(data); // free ถ้า init ล้มเหลวหลังจากนี้

        // จำลองการเริ่มต้น resource ที่อาจล้มเหลว
        if (size > 1000) {
            return error.SizeTooLarge; // data จะถูก free อัตโนมัติ
        }

        @memset(data, 0);
        return .{ .data = data, .allocator = allocator };
    }

    pub fn deinit(self: Resource) void {
        self.allocator.free(self.data);
    }
};

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    // กรณีสำเร็จ
    if (Resource.init(allocator, 100)) |res| {
        defer res.deinit();
        std.debug.print("สร้าง resource สำเร็จ ขนาด {d} bytes\n", .{res.data.len});
    } else |err| {
        std.debug.print("ล้มเหลว: {}\n", .{err});
    }

    // กรณีล้มเหลว - memory จะไม่รั่ว
    if (Resource.init(allocator, 2000)) |res| {
        defer res.deinit();
        std.debug.print("สร้าง resource ขนาดใหญ่\n", .{});
    } else |err| {
        std.debug.print("ล้มเหลวตามคาด: {}\n", .{err});
    }
}
```

### ความแตกต่างระหว่าง defer และ errdefer

```zig
const std = @import("std");

fn demonstrateDifference(fail: bool) !void {
    defer std.debug.print("defer: รันเสมอ\n", .{});
    errdefer std.debug.print("errdefer: รันเมื่อเกิด error\n", .{});

    if (fail) {
        return error.Failure;
    }

    std.debug.print("สำเร็จ\n", .{});
}

pub fn main() void {
    std.debug.print("--- ไม่มี error ---\n", .{});
    demonstrateDifference(false) catch {};
    // Output:
    // สำเร็จ
    // defer: รันเสมอ

    std.debug.print("\n--- มี error ---\n", .{});
    demonstrateDifference(true) catch {};
    // Output:
    // errdefer: รันเมื่อเกิด error
    // defer: รันเสมอ
}
```

### errdefer กับการ Rollback

```zig
const std = @import("std");

// จำลอง database transaction
const DB = struct {
    pub fn beginTransaction() void {
        std.debug.print("เริ่ม transaction\n", .{});
    }

    pub fn commit() void {
        std.debug.print("Commit transaction\n", .{});
    }

    pub fn rollback() void {
        std.debug.print("Rollback transaction!\n", .{});
    }

    pub fn insert(record: []const u8) !void {
        std.debug.print("Insert: {s}\n", .{record});
        if (std.mem.eql(u8, record, "bad_data")) {
            return error.InvalidData;
        }
    }
};

fn saveData(records: []const []const u8) !void {
    DB.beginTransaction();
    errdefer DB.rollback(); // rollback ถ้าเกิด error

    for (records) |record| {
        try DB.insert(record);
    }

    DB.commit();
}

pub fn main() void {
    std.debug.print("=== ข้อมูลปกติ ===\n", .{});
    const good_data = [_][]const u8{ "user1", "user2", "user3" };
    saveData(&good_data) catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };

    std.debug.print("\n=== ข้อมูลผิดพลาด ===\n", .{});
    const bad_data = [_][]const u8{ "user1", "bad_data", "user3" };
    saveData(&bad_data) catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### errdefer กับ Stack Unwinding

```zig
const std = @import("std");

fn step1() !void {
    errdefer std.debug.print("step1 cleanup\n", .{});
    std.debug.print("step1 done\n", .{});
}

fn step2() !void {
    errdefer std.debug.print("step2 cleanup\n", .{});
    std.debug.print("step2 start\n", .{});
    return error.Step2Failed;
}

fn step3() !void {
    errdefer std.debug.print("step3 cleanup\n", .{});
    std.debug.print("step3 done\n", .{});
}

fn doMultiSteps() !void {
    try step1();
    errdefer std.debug.print("doMultiSteps cleanup after step1\n", .{});

    try step2(); // จะเกิด error ที่นี่

    try step3(); // ไม่ถึงที่นี่
}

pub fn main() void {
    doMultiSteps() catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### Pattern ที่ดีที่สุดของ errdefer

```zig
const std = @import("std");

const Server = struct {
    socket: u32,
    buffer: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) !Server {
        // ทำทีละขั้น โดยมี errdefer สำหรับแต่ละขั้น
        const socket = try openSocket();
        errdefer closeSocket(socket);

        const buffer = try allocator.alloc(u8, 4096);
        errdefer allocator.free(buffer);

        // ถ้าขั้นต่อไปล้มเหลว socket และ buffer จะถูก cleanup อัตโนมัติ
        try bindSocket(socket);

        return .{
            .socket = socket,
            .buffer = buffer,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: Server) void {
        closeSocket(self.socket);
        self.allocator.free(self.buffer);
    }
};

fn openSocket() !u32 {
    std.debug.print("เปิด socket\n", .{});
    return 42; // จำลอง socket fd
}

fn closeSocket(fd: u32) void {
    std.debug.print("ปิด socket {d}\n", .{fd});
}

fn bindSocket(fd: u32) !void {
    std.debug.print("Bind socket {d}\n", .{fd});
    // จำลอง: ล้มเหลวในบางกรณี
    if (fd > 100) return error.BindFailed;
}

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    if (Server.init(allocator)) |server| {
        defer server.deinit();
        std.debug.print("Server เริ่มทำงาน!\n", .{});
    } else |err| {
        std.debug.print("ไม่สามารถเริ่ม server: {}\n", .{err});
    }
}
```

---

## สรุปภาพรวม

### ตาราง: การควบคุมการทำงานใน Zig

| Construct | ใช้เมื่อ | ข้อสังเกต |
|-----------|----------|-----------|
| `if/else` | ตรวจสอบเงื่อนไข | เงื่อนไขต้องเป็น `bool` เท่านั้น |
| `if expression` | กำหนดค่าตามเงื่อนไข | ต้องมี `else` เสมอเมื่อใช้เป็น value |
| `if (opt) \|val\|` | แกะค่าจาก Optional | ปลอดภัยกว่าการใช้ `orelse` |
| `if (err) \|val\| else \|e\|` | จัดการ Error Union | ทางเลือกของ `try`/`catch` |
| `switch` | เลือกตามค่า | ต้อง exhaustive, ไม่มี fall-through |
| `switch expression` | กำหนดค่าตาม switch | คล้าย if expression |
| `switch` กับ enum | Pattern matching | Compiler บังคับครอบคลุมทุก variant |
| `switch` กับ union | แกะค่าจาก union | พร้อมแกะ payload ด้วย `\|val\|` |
| `defer` | Cleanup ที่รันเสมอ | LIFO order, รันเมื่อออก scope |
| `errdefer` | Cleanup เฉพาะ error | Rollback, undo allocations |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: FizzBuzz ด้วย if/else

เขียนโปรแกรม FizzBuzz โดยใช้ if-else expression:
- ถ้าหารด้วย 15 ลงตัว: "FizzBuzz"
- ถ้าหารด้วย 3 ลงตัว: "Fizz"
- ถ้าหารด้วย 5 ลงตัว: "Buzz"
- อื่นๆ: แสดงตัวเลข

```zig
const std = @import("std");

pub fn main() void {
    var i: u32 = 1;
    while (i <= 20) : (i += 1) {
        const output = if (i % 15 == 0) "FizzBuzz"
            else if (i % 3 == 0) "Fizz"
            else if (i % 5 == 0) "Buzz"
            else "";
        
        if (output.len > 0) {
            std.debug.print("{s}\n", .{output});
        } else {
            std.debug.print("{d}\n", .{i});
        }
    }
}
```

### แบบฝึกหัดที่ 2: Calculator ด้วย switch

สร้าง calculator ด้วย switch:

```zig
const std = @import("std");

const Operation = enum { Add, Sub, Mul, Div };

fn calculate(a: f64, b: f64, op: Operation) !f64 {
    return switch (op) {
        .Add => a + b,
        .Sub => a - b,
        .Mul => a * b,
        .Div => if (b == 0) error.DivisionByZero else a / b,
    };
}

pub fn main() void {
    const ops = [_]struct { a: f64, b: f64, op: Operation }{
        .{ .a = 10, .b = 5, .op = .Add },
        .{ .a = 10, .b = 3, .op = .Sub },
        .{ .a = 4, .b = 7, .op = .Mul },
        .{ .a = 15, .b = 4, .op = .Div },
        .{ .a = 10, .b = 0, .op = .Div },
    };

    for (ops) |op| {
        if (calculate(op.a, op.b, op.op)) |result| {
            std.debug.print("{d:.0} {s} {d:.0} = {d:.2}\n", .{
                op.a,
                switch (op.op) {
                    .Add => "+",
                    .Sub => "-",
                    .Mul => "*",
                    .Div => "/",
                },
                op.b,
                result,
            });
        } else |err| {
            std.debug.print("Error: {}\n", .{err});
        }
    }
}
```

### แบบฝึกหัดที่ 3: Resource Manager ด้วย defer

สร้าง resource manager ที่ใช้ defer:

```zig
const std = @import("std");

const Connection = struct {
    url: []const u8,
    open: bool,

    pub fn connect(url: []const u8) !Connection {
        std.debug.print("เชื่อมต่อ {s}...\n", .{url});
        if (std.mem.startsWith(u8, url, "bad://")) {
            return error.ConnectionFailed;
        }
        return .{ .url = url, .open = true };
    }

    pub fn disconnect(self: *Connection) void {
        if (self.open) {
            std.debug.print("ตัดการเชื่อมต่อ {s}\n", .{self.url});
            self.open = false;
        }
    }

    pub fn query(self: Connection, sql: []const u8) !void {
        if (!self.open) return error.NotConnected;
        std.debug.print("Query: {s}\n", .{sql});
    }
};

fn doDatabase(url: []const u8) !void {
    var conn = try Connection.connect(url);
    defer conn.disconnect();

    try conn.query("SELECT * FROM users");
    try conn.query("SELECT * FROM products");
}

pub fn main() void {
    std.debug.print("=== เชื่อมต่อปกติ ===\n", .{});
    doDatabase("db://localhost/mydb") catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };

    std.debug.print("\n=== เชื่อมต่อล้มเหลว ===\n", .{});
    doDatabase("bad://invalid") catch |err| {
        std.debug.print("Error: {}\n", .{err});
    };
}
```

### แบบฝึกหัดที่ 4: Type-safe Message System ด้วย Tagged Union

สร้างระบบ message ด้วย tagged union:

```zig
const std = @import("std");

const Message = union(enum) {
    text: []const u8,
    number: i64,
    command: enum { Start, Stop, Pause, Resume },
    error_msg: struct { code: u32, description: []const u8 },
};

fn handleMessage(msg: Message) void {
    switch (msg) {
        .text => |t| std.debug.print("ข้อความ: {s}\n", .{t}),
        .number => |n| std.debug.print("ตัวเลข: {d}\n", .{n}),
        .command => |cmd| std.debug.print("คำสั่ง: {s}\n", .{@tagName(cmd)}),
        .error_msg => |e| std.debug.print("Error {d}: {s}\n", .{ e.code, e.description }),
    }
}

pub fn main() void {
    const messages = [_]Message{
        .{ .text = "สวัสดีชาวโลก" },
        .{ .number = 42 },
        .{ .command = .Start },
        .{ .command = .Stop },
        .{ .error_msg = .{ .code = 404, .description = "ไม่พบข้อมูล" } },
    };

    for (messages) |msg| {
        handleMessage(msg);
    }
}
```

### แบบฝึกหัดท้าทาย: Inventory System

สร้างระบบ inventory ที่ใช้ทุก concept ที่เรียนมา:

```zig
const std = @import("std");

const ItemCategory = enum { Food, Electronics, Clothing, Books };

const Item = struct {
    id: u32,
    name: []const u8,
    price: f64,
    quantity: u32,
    category: ItemCategory,
};

const InventoryError = error{
    ItemNotFound,
    InsufficientStock,
    InvalidPrice,
};

fn findItem(inventory: []const Item, id: u32) ?Item {
    for (inventory) |item| {
        if (item.id == id) return item;
    }
    return null;
}

fn purchase(inventory: []Item, id: u32, qty: u32) InventoryError!f64 {
    for (inventory) |*item| {
        if (item.id == id) {
            if (item.quantity < qty) return InventoryError.InsufficientStock;
            item.quantity -= qty;
            return item.price * @as(f64, @floatFromInt(qty));
        }
    }
    return InventoryError.ItemNotFound;
}

fn categoryDiscount(cat: ItemCategory) f64 {
    return switch (cat) {
        .Food => 0.05,
        .Electronics => 0.10,
        .Clothing => 0.15,
        .Books => 0.20,
    };
}

pub fn main() void {
    var inventory = [_]Item{
        .{ .id = 1, .name = "แอปเปิ้ล", .price = 30.0, .quantity = 100, .category = .Food },
        .{ .id = 2, .name = "โน้ตบุ๊ค", .price = 25000.0, .quantity = 10, .category = .Electronics },
        .{ .id = 3, .name = "เสื้อยืด", .price = 299.0, .quantity = 50, .category = .Clothing },
    };

    // แสดงรายการสินค้า
    std.debug.print("=== รายการสินค้า ===\n", .{});
    for (inventory) |item| {
        const discount = categoryDiscount(item.category);
        const discounted_price = item.price * (1.0 - discount);
        std.debug.print("ID {d}: {s} - ราคา {d:.2} (ลด {d:.0}% เหลือ {d:.2}) จำนวน {d}\n", .{
            item.id,
            item.name,
            item.price,
            discount * 100,
            discounted_price,
            item.quantity,
        });
    }

    // ทดสอบการซื้อ
    std.debug.print("\n=== ทดสอบการซื้อ ===\n", .{});
    const orders = [_]struct { id: u32, qty: u32 }{
        .{ .id = 1, .qty = 5 },  // ปกติ
        .{ .id = 2, .qty = 15 }, // เกิน stock
        .{ .id = 99, .qty = 1 }, // ไม่มีสินค้า
    };

    for (orders) |order| {
        if (purchase(&inventory, order.id, order.qty)) |total| {
            std.debug.print("ซื้อ ID {d} จำนวน {d}: {d:.2} บาท\n", .{
                order.id, order.qty, total,
            });
        } else |err| {
            switch (err) {
                InventoryError.ItemNotFound => std.debug.print("ไม่พบสินค้า ID {d}\n", .{order.id}),
                InventoryError.InsufficientStock => std.debug.print("สินค้า ID {d} ไม่เพียงพอ\n", .{order.id}),
                InventoryError.InvalidPrice => std.debug.print("ราคาไม่ถูกต้อง\n", .{}),
            }
        }
    }

    // ค้นหาสินค้า
    std.debug.print("\n=== ค้นหาสินค้า ===\n", .{});
    if (findItem(&inventory, 2)) |item| {
        std.debug.print("พบ: {s} เหลือ {d} ชิ้น\n", .{ item.name, item.quantity });
    } else {
        std.debug.print("ไม่พบสินค้า\n", .{});
    }
}
```

---

## สรุป Key Points

### 1. if/else
- เงื่อนไขต้องเป็น `bool` เสมอ
- `if` เป็น expression ได้ (มีค่า)
- ต้องมี `else` เมื่อใช้เป็น expression

### 2. if กับ Optional
- รูปแบบ `if (opt) |val| { ... }` ปลอดภัยและชัดเจน
- ใช้ `|*ptr|` เมื่อต้องการแก้ไขค่า

### 3. if กับ Error Union
- รูปแบบ `if (result) |val| { ... } else |err| { ... }`
- จัดการ error ได้ชัดเจนและไม่มี exception ที่ไม่คาดหมาย

### 4. switch
- **Exhaustive**: ต้องครอบคลุมทุกกรณี
- **No fall-through**: แต่ละ arm เป็นอิสระ
- รองรับ range (`a...b`) และหลายค่า (`a, b, c`)
- เป็น expression ได้

### 5. switch กับ enum และ union
- Compiler บังคับให้จัดการทุก variant
- Tagged union ทำ pattern matching ได้อย่างปลอดภัย

### 6. defer
- รันเมื่อออกจาก scope เสมอ (ทั้ง return ปกติและ error)
- ลำดับ LIFO (Last In, First Out)
- เหมาะสำหรับ cleanup, close, free

### 7. errdefer
- รันเฉพาะเมื่อออกจาก scope **ด้วย error**
- เหมาะสำหรับ rollback และ undo allocation
- ใช้คู่กับ `defer` เพื่อจัดการ happy/sad path

---

## สิ่งที่ควรจำ

```zig
// Pattern ที่ดีสำหรับ resource management
fn doWork(allocator: std.mem.Allocator) !void {
    const resource1 = try initResource1();
    errdefer cleanupResource1(resource1); // undo ถ้า error

    const buffer = try allocator.alloc(u8, 100);
    errdefer allocator.free(buffer); // free ถ้า error

    // ทำงาน...
    try doSomething(resource1, buffer);

    // ถ้าถึงที่นี่ได้ = สำเร็จ
    // defer ที่ caller จะจัดการ cleanup
    _ = resource1;
    _ = buffer;
}
```

Control flow ที่ดีใน Zig คือการเขียนโค้ดที่ **ชัดเจน**, **ปลอดภัย** และ **ไม่มี resource leak** ด้วยการใช้ `defer`/`errdefer` อย่างเหมาะสม

---

*ต่อไป: Part 07 - Loops และ Iterations ใน Zig*
