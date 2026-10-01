# ส่วนที่ 12: Enums ใน Zig (Steps 111-120)

Enum (Enumeration) เป็นชนิดข้อมูลที่ให้เราระบุชุดของค่าที่เป็นไปได้อย่างชัดเจน
ใน Zig, enum มีความสามารถมากกว่าใน C อย่างมาก เราสามารถกำหนด method, ทำ exhaustive matching,
และผสมกับ tagged union เพื่อสร้าง algebraic data types ที่ทรงพลัง

---

## Step 111: Enum พื้นฐาน

### การประกาศ Enum อย่างง่าย

Enum พื้นฐานใน Zig ประกาศด้วย `enum` keyword:

```zig
const std = @import("std");

const Direction = enum {
    north,
    south,
    east,
    west,
};

pub fn main() void {
    const dir = Direction.north;
    std.debug.print("Direction: {}\n", .{dir});

    // การเปรียบเทียบ enum
    if (dir == Direction.north) {
        std.debug.print("Going north!\n", .{});
    }
}
```

### Enum และ Type Safety

ความแตกต่างสำคัญระหว่าง Zig enum กับ C enum คือ type safety:

```zig
const Color = enum {
    red,
    green,
    blue,
};

const Status = enum {
    active,
    inactive,
};

pub fn main() void {
    const c = Color.red;
    const s = Status.active;

    // บรรทัดนี้จะ compile error - ไม่สามารถเปรียบเทียบ enum ต่างชนิดกันได้
    // if (c == s) {} // Error!

    // ต้องระบุ type ชัดเจน
    _ = c;
    _ = s;
}
```

### การใช้ Enum เป็น Parameter

```zig
const std = @import("std");

const Season = enum {
    spring,
    summer,
    autumn,
    winter,
};

fn describeSeason(season: Season) []const u8 {
    return switch (season) {
        .spring => "อากาศอบอุ่น ดอกไม้บาน",
        .summer => "ร้อนจัด",
        .autumn => "ใบไม้ร่วง",
        .winter => "หนาวเย็น",
    };
}

pub fn main() void {
    const s = Season.summer;
    std.debug.print("ฤดูกาล: {s}\n", .{describeSeason(s)});
}
```

### Enum ใน Array และ Slice

```zig
const std = @import("std");

const Weekday = enum {
    monday,
    tuesday,
    wednesday,
    thursday,
    friday,
    saturday,
    sunday,
};

pub fn main() void {
    const workdays = [_]Weekday{
        .monday,
        .tuesday,
        .wednesday,
        .thursday,
        .friday,
    };

    std.debug.print("วันทำงาน: {} วัน\n", .{workdays.len});

    for (workdays) |day| {
        std.debug.print("  - {}\n", .{day});
    }
}
```

### เปรียบเทียบกับ C Enum

```c
// C enum - ไม่มี type safety, เป็นแค่ integer
typedef enum {
    RED = 0,
    GREEN = 1,
    BLUE = 2
} Color;

typedef enum {
    ACTIVE = 0,
    INACTIVE = 1
} Status;

// C อนุญาตให้ทำแบบนี้ได้ (bug ที่หลบซ่อน!)
Color c = RED;
Status s = ACTIVE;
if (c == s) { /* เป็น true เพราะทั้งคู่เป็น 0 */ }
```

ใน Zig การเปรียบเทียบแบบนี้จะเป็น compile error ทันที ทำให้ code ปลอดภัยกว่ามาก

---

## Step 112: Enum กับ Integer Values

### การกำหนด Integer Values

เราสามารถกำหนดค่า integer ให้กับ enum ได้:

```zig
const std = @import("std");

const HttpStatus = enum(u16) {
    ok = 200,
    created = 201,
    no_content = 204,
    bad_request = 400,
    unauthorized = 401,
    forbidden = 403,
    not_found = 404,
    internal_server_error = 500,
};

pub fn main() void {
    const status = HttpStatus.ok;

    // แปลง enum เป็น integer
    const code = @intFromEnum(status);
    std.debug.print("HTTP Status Code: {d}\n", .{code}); // 200

    // แปลง integer เป็น enum
    const s = @as(HttpStatus, @enumFromInt(201));
    std.debug.print("Status: {}\n", .{s}); // created
}
```

### Enum กับ Bit Flags

```zig
const std = @import("std");

const FilePermission = enum(u8) {
    none = 0,
    read = 1,
    write = 2,
    execute = 4,
    read_write = 3,      // read | write
    read_execute = 5,    // read | execute
    all = 7,             // read | write | execute
};

pub fn main() void {
    const perm = FilePermission.read_write;
    const val = @intFromEnum(perm);
    std.debug.print("Permission value: {d}\n", .{val}); // 3

    // ตรวจสอบ permission
    const has_read = (val & @intFromEnum(FilePermission.read)) != 0;
    const has_write = (val & @intFromEnum(FilePermission.write)) != 0;
    const has_execute = (val & @intFromEnum(FilePermission.execute)) != 0;

    std.debug.print("Read: {}, Write: {}, Execute: {}\n", .{
        has_read, has_write, has_execute,
    });
}
```

### Enum ที่มี Integer Type ต่างๆ

```zig
const std = @import("std");

// u8 enum สำหรับค่าน้อยๆ
const Priority = enum(u8) {
    low = 1,
    medium = 5,
    high = 10,
    critical = 100,
};

// i32 enum สำหรับค่าที่มี sign
const Temperature = enum(i32) {
    absolute_zero = -273,
    freezing = 0,
    body_temp = 37,
    boiling = 100,
};

pub fn main() void {
    const p = Priority.high;
    const t = Temperature.body_temp;

    std.debug.print("Priority: {d}\n", .{@intFromEnum(p)});
    std.debug.print("Temperature: {d}°C\n", .{@intFromEnum(t)});

    // เปรียบเทียบค่า integer
    if (@intFromEnum(p) > 5) {
        std.debug.print("High priority task!\n", .{});
    }
}
```

### Auto-increment Values

```zig
const std = @import("std");

// Zig จะกำหนดค่าเริ่มต้นจาก 0 อัตโนมัติ
const Cardinal = enum(u8) {
    north = 0,  // explicit
    south,      // = 1
    east,       // = 2
    west,       // = 3
};

// ผสมระหว่าง explicit และ auto
const ErrorCode = enum(i32) {
    none = 0,
    io_error = -1,
    memory_error = -2,
    network_error = -10,
    timeout = -11,  // ต่อจาก -10 ไม่ได้ ต้องระบุ
};

pub fn main() void {
    std.debug.print("North = {d}\n", .{@intFromEnum(Cardinal.north)});
    std.debug.print("West = {d}\n", .{@intFromEnum(Cardinal.west)});
}
```

---

## Step 113: Enum Methods

### การเพิ่ม Method ให้ Enum

ใน Zig, enum สามารถมี method ได้เหมือน struct:

```zig
const std = @import("std");

const Direction = enum {
    north,
    south,
    east,
    west,

    // Method ที่ return ทิศตรงข้าม
    pub fn opposite(self: Direction) Direction {
        return switch (self) {
            .north => .south,
            .south => .north,
            .east => .west,
            .west => .east,
        };
    }

    // Method ที่ return ชื่อภาษาไทย
    pub fn thaiName(self: Direction) []const u8 {
        return switch (self) {
            .north => "เหนือ",
            .south => "ใต้",
            .east => "ตะวันออก",
            .west => "ตะวันตก",
        };
    }

    // Method ที่ return มุม (degrees)
    pub fn toDegrees(self: Direction) f32 {
        return switch (self) {
            .north => 0.0,
            .east => 90.0,
            .south => 180.0,
            .west => 270.0,
        };
    }
};

pub fn main() void {
    const dir = Direction.north;

    std.debug.print("ทิศ: {s}\n", .{dir.thaiName()});
    std.debug.print("ทิศตรงข้าม: {s}\n", .{dir.opposite().thaiName()});
    std.debug.print("มุม: {d}°\n", .{dir.toDegrees()});
}
```

### Method กับ Integer Backing

```zig
const std = @import("std");

const Suit = enum(u8) {
    clubs = 0,
    diamonds = 1,
    hearts = 2,
    spades = 3,

    pub fn symbol(self: Suit) []const u8 {
        return switch (self) {
            .clubs => "♣",
            .diamonds => "♦",
            .hearts => "♥",
            .spades => "♠",
        };
    }

    pub fn color(self: Suit) []const u8 {
        return switch (self) {
            .clubs, .spades => "ดำ",
            .diamonds, .hearts => "แดง",
        };
    }

    pub fn value(self: Suit) u8 {
        return @intFromEnum(self);
    }
};

pub fn main() void {
    const suits = [_]Suit{ .clubs, .diamonds, .hearts, .spades };

    for (suits) |suit| {
        std.debug.print("{s} ({s}) = {d}\n", .{
            suit.symbol(),
            suit.color(),
            suit.value(),
        });
    }
}
```

### Static Method (ไม่รับ self)

```zig
const std = @import("std");

const Month = enum(u8) {
    january = 1,
    february = 2,
    march = 3,
    april = 4,
    may = 5,
    june = 6,
    july = 7,
    august = 8,
    september = 9,
    october = 10,
    november = 11,
    december = 12,

    // Instance method
    pub fn daysInMonth(self: Month, year: u32) u8 {
        return switch (self) {
            .january, .march, .may, .july, .august, .october, .december => 31,
            .april, .june, .september, .november => 30,
            .february => if (Month.isLeapYear(year)) 29 else 28,
        };
    }

    pub fn thaiName(self: Month) []const u8 {
        return switch (self) {
            .january => "มกราคม",
            .february => "กุมภาพันธ์",
            .march => "มีนาคม",
            .april => "เมษายน",
            .may => "พฤษภาคม",
            .june => "มิถุนายน",
            .july => "กรกฎาคม",
            .august => "สิงหาคม",
            .september => "กันยายน",
            .october => "ตุลาคม",
            .november => "พฤศจิกายน",
            .december => "ธันวาคม",
        };
    }

    // Static method (ไม่รับ self)
    pub fn isLeapYear(year: u32) bool {
        return (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0);
    }
};

pub fn main() void {
    const m = Month.february;
    const year: u32 = 2024;

    std.debug.print("{s} {d}: {d} วัน\n", .{
        m.thaiName(),
        year,
        m.daysInMonth(year),
    });

    std.debug.print("ปี {d} เป็นปีอธิกสุรทิน: {}\n", .{
        year,
        Month.isLeapYear(year),
    });
}
```

---

## Step 114: @tagName และ @enumFromInt

### @tagName - แปลง Enum เป็น String

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    blue,
    yellow,
};

pub fn main() void {
    const c = Color.green;

    // @tagName คืนค่า comptime string ของชื่อ enum value
    const name = @tagName(c);
    std.debug.print("Color name: {s}\n", .{name}); // "green"

    // ใช้ในการ format
    std.debug.print("The color is: {s}\n", .{@tagName(Color.blue)});

    // loop ผ่าน enum fields
    const fields = @typeInfo(Color).@"enum".fields;
    std.debug.print("\nEnum fields:\n", .{});
    for (fields) |field| {
        std.debug.print("  {s} = {d}\n", .{ field.name, field.value });
    }
}
```

### @enumFromInt - แปลง Integer เป็น Enum

```zig
const std = @import("std");

const Level = enum(u8) {
    debug = 0,
    info = 1,
    warning = 2,
    err = 3,
    critical = 4,
};

pub fn main() void {
    // แปลง integer เป็น enum (อาจ panic ถ้าค่าไม่ valid)
    const level: Level = @enumFromInt(2);
    std.debug.print("Level: {s}\n", .{@tagName(level)}); // "warning"

    // ใช้ @intFromEnum ในทิศทางตรงข้าม
    const val = @intFromEnum(Level.err);
    std.debug.print("Value: {d}\n", .{val}); // 3

    // อ่านค่าจาก input และแปลง
    const user_input: u8 = 1;
    const user_level: Level = @enumFromInt(user_input);
    std.debug.print("User level: {s}\n", .{@tagName(user_level)});
}
```

### การ Validate ก่อน enumFromInt

```zig
const std = @import("std");

const Command = enum(u8) {
    start = 1,
    stop = 2,
    pause = 3,
    resume = 4,
};

fn parseCommand(value: u8) ?Command {
    // ตรวจสอบว่าค่าอยู่ใน range ที่ valid หรือไม่
    return switch (value) {
        1...4 => @enumFromInt(value),
        else => null,
    };
}

pub fn main() void {
    const inputs = [_]u8{ 1, 2, 5, 3, 99 };

    for (inputs) |input| {
        if (parseCommand(input)) |cmd| {
            std.debug.print("Command {d}: {s}\n", .{ input, @tagName(cmd) });
        } else {
            std.debug.print("Command {d}: invalid\n", .{input});
        }
    }
}
```

### @typeInfo กับ Enum

```zig
const std = @import("std");

const Planet = enum {
    mercury,
    venus,
    earth,
    mars,
    jupiter,
    saturn,
    uranus,
    neptune,
};

pub fn printEnumInfo(comptime E: type) void {
    const info = @typeInfo(E).@"enum";
    std.debug.print("Enum: {s}\n", .{@typeName(E)});
    std.debug.print("Tag type: {s}\n", .{@typeName(info.tag_type)});
    std.debug.print("Is exhaustive: {}\n", .{info.is_exhaustive});
    std.debug.print("Fields ({d}):\n", .{info.fields.len});
    for (info.fields) |field| {
        std.debug.print("  .{s} = {d}\n", .{ field.name, field.value });
    }
}

pub fn main() void {
    printEnumInfo(Planet);
}
```

---

## Step 115: Enum ใน Switch (Exhaustive Matching)

### Exhaustive Switch - ต้อง Handle ทุก Case

```zig
const std = @import("std");

const TrafficLight = enum {
    red,
    yellow,
    green,
};

pub fn action(light: TrafficLight) []const u8 {
    // Zig บังคับให้ handle ทุก enum value
    // ถ้าลืม case ใด จะเป็น compile error
    return switch (light) {
        .red => "หยุด",
        .yellow => "ระวัง",
        .green => "ไป",
        // ถ้าลบ .green ออก จะได้ error: switch must handle all cases
    };
}

pub fn main() void {
    const lights = [_]TrafficLight{ .red, .yellow, .green };

    for (lights) |light| {
        std.debug.print("ไฟ {s}: {s}\n", .{ @tagName(light), action(light) });
    }
}
```

### Switch กับ Multiple Values

```zig
const std = @import("std");

const DayOfWeek = enum {
    monday,
    tuesday,
    wednesday,
    thursday,
    friday,
    saturday,
    sunday,
};

fn isWeekend(day: DayOfWeek) bool {
    return switch (day) {
        .saturday, .sunday => true,
        .monday, .tuesday, .wednesday, .thursday, .friday => false,
    };
}

fn workHours(day: DayOfWeek) u8 {
    return switch (day) {
        .monday...friday => 8,    // range ใน switch
        .saturday => 4,
        .sunday => 0,
    };
}

// หรือใช้ inline else
fn workHours2(day: DayOfWeek) u8 {
    return switch (day) {
        .saturday => 4,
        .sunday => 0,
        else => 8,  // ใช้ else สำหรับ cases ที่เหลือ
    };
}

pub fn main() void {
    const days = [_]DayOfWeek{
        .monday, .wednesday, .friday, .saturday, .sunday,
    };

    for (days) |day| {
        std.debug.print("{s}: weekend={}, hours={d}\n", .{
            @tagName(day),
            isWeekend(day),
            workHours(day),
        });
    }
}
```

### Switch ที่ Return Value

```zig
const std = @import("std");

const Operator = enum {
    add,
    subtract,
    multiply,
    divide,
};

fn calculate(op: Operator, a: f64, b: f64) f64 {
    return switch (op) {
        .add => a + b,
        .subtract => a - b,
        .multiply => a * b,
        .divide => if (b != 0) a / b else std.math.nan(f64),
    };
}

pub fn main() void {
    const ops = [_]Operator{ .add, .subtract, .multiply, .divide };
    const a: f64 = 10.0;
    const b: f64 = 3.0;

    for (ops) |op| {
        const result = calculate(op, a, b);
        std.debug.print("{s}({d}, {d}) = {d:.2}\n", .{
            @tagName(op), a, b, result,
        });
    }
}
```

### Compile-time Exhaustive Check

```zig
// ถ้าเพิ่ม enum value ใหม่ แต่ลืม update switch
// Zig จะ error ที่ compile time ทันที

const std = @import("std");

const AppState = enum {
    loading,
    ready,
    running,
    paused,
    stopped,
    // error_state,  // ถ้า uncomment บรรทัดนี้ จะต้อง update switch ด้วย
};

fn handleState(state: AppState) void {
    switch (state) {
        .loading => std.debug.print("กำลังโหลด...\n", .{}),
        .ready => std.debug.print("พร้อมแล้ว\n", .{}),
        .running => std.debug.print("กำลังทำงาน\n", .{}),
        .paused => std.debug.print("หยุดชั่วคราว\n", .{}),
        .stopped => std.debug.print("หยุดแล้ว\n", .{}),
    }
}

pub fn main() void {
    handleState(.running);
    handleState(.paused);
}
```

---

## Step 116: Non-exhaustive Enum

### การสร้าง Non-exhaustive Enum

Non-exhaustive enum ใช้เมื่อเราต้องการรองรับค่าที่อาจไม่รู้จัก:

```zig
const std = @import("std");

// Non-exhaustive enum ต้องมี integer backing type
// และต้องมี _ field หรือใช้ else ใน switch
const KeyCode = enum(u16) {
    // Common keys
    escape = 27,
    enter = 13,
    space = 32,
    backspace = 8,

    // Arrow keys
    arrow_up = 256,
    arrow_down = 257,
    arrow_left = 258,
    arrow_right = 259,

    // ทำให้เป็น non-exhaustive
    _,
};

fn handleKey(key: KeyCode) void {
    switch (key) {
        .escape => std.debug.print("ESC pressed\n", .{}),
        .enter => std.debug.print("ENTER pressed\n", .{}),
        .space => std.debug.print("SPACE pressed\n", .{}),
        .backspace => std.debug.print("BACKSPACE pressed\n", .{}),
        .arrow_up => std.debug.print("UP pressed\n", .{}),
        .arrow_down => std.debug.print("DOWN pressed\n", .{}),
        .arrow_left => std.debug.print("LEFT pressed\n", .{}),
        .arrow_right => std.debug.print("RIGHT pressed\n", .{}),
        // ต้องมี else สำหรับ non-exhaustive enum
        _ => std.debug.print("Unknown key: {d}\n", .{@intFromEnum(key)}),
    }
}

pub fn main() void {
    handleKey(.enter);
    handleKey(.arrow_up);

    // สร้างค่าที่ไม่ได้ define ไว้
    const unknown_key: KeyCode = @enumFromInt(65); // 'A' key
    handleKey(unknown_key);
}
```

### Non-exhaustive กับ Protocol/API Compatibility

```zig
const std = @import("std");

// HTTP Method - อาจมี method ใหม่ในอนาคต
const HttpMethod = enum(u8) {
    GET = 0,
    POST = 1,
    PUT = 2,
    DELETE = 3,
    PATCH = 4,
    HEAD = 5,
    OPTIONS = 6,
    _,
};

fn methodName(method: HttpMethod) []const u8 {
    return switch (method) {
        .GET => "GET",
        .POST => "POST",
        .PUT => "PUT",
        .DELETE => "DELETE",
        .PATCH => "PATCH",
        .HEAD => "HEAD",
        .OPTIONS => "OPTIONS",
        _ => "UNKNOWN",
    };
}

fn isSafe(method: HttpMethod) bool {
    return switch (method) {
        .GET, .HEAD, .OPTIONS => true,
        else => false,
    };
}

pub fn main() void {
    const methods = [_]HttpMethod{ .GET, .POST, .DELETE };
    for (methods) |m| {
        std.debug.print("{s}: safe={}\n", .{ methodName(m), isSafe(m) });
    }
}
```

### ความแตกต่างระหว่าง Exhaustive และ Non-exhaustive

```zig
const std = @import("std");

// Exhaustive - ต้อง handle ทุก case
const State = enum {
    on,
    off,
};

// Non-exhaustive - ใช้ _ เพื่อ ignore unknown values
const ExternalState = enum(u8) {
    on = 1,
    off = 0,
    _,
};

pub fn main() void {
    // Exhaustive switch
    const s = State.on;
    const msg1 = switch (s) {
        .on => "켜짐",
        .off => "꺼짐",
        // ถ้าลบ case ใดออก จะ compile error
    };

    // Non-exhaustive switch (ต้องมี else หรือ _)
    const e = ExternalState.on;
    const msg2 = switch (e) {
        .on => "켜짐",
        .off => "꺼짐",
        _ => "알 수 없음",  // handle unknown values
    };

    std.debug.print("{s}, {s}\n", .{ msg1, msg2 });
}
```

---

## Step 117: Enum กับ Comptime

### Comptime Enum Operations

```zig
const std = @import("std");

const Fruit = enum {
    apple,
    banana,
    cherry,
    date,
    elderberry,
};

// Comptime function ที่ทำงานกับ enum type
fn enumCount(comptime E: type) comptime_int {
    return @typeInfo(E).@"enum".fields.len;
}

fn firstField(comptime E: type) E {
    const fields = @typeInfo(E).@"enum".fields;
    return @enumFromInt(fields[0].value);
}

fn lastField(comptime E: type) E {
    const fields = @typeInfo(E).@"enum".fields;
    return @enumFromInt(fields[fields.len - 1].value);
}

pub fn main() void {
    comptime {
        const count = enumCount(Fruit);
        std.debug.assert(count == 5);
    }

    std.debug.print("Fruit count: {d}\n", .{enumCount(Fruit)});
    std.debug.print("First: {s}\n", .{@tagName(firstField(Fruit))});
    std.debug.print("Last: {s}\n", .{@tagName(lastField(Fruit))});
}
```

### Generic Function กับ Enum

```zig
const std = @import("std");

// Generic function ที่รับ enum type เป็น comptime parameter
fn printAllValues(comptime E: type) void {
    std.debug.print("Values of {s}:\n", .{@typeName(E)});
    const fields = @typeInfo(E).@"enum".fields;
    inline for (fields) |field| {
        std.debug.print("  .{s} = {d}\n", .{ field.name, field.value });
    }
}

// สร้าง lookup table จาก enum ที่ comptime
fn buildLookupTable(comptime E: type, comptime values: []const f64) [enumCount(E)]f64 {
    comptime std.debug.assert(values.len == enumCount(E));
    var table: [enumCount(E)]f64 = undefined;
    comptime {
        const fields = @typeInfo(E).@"enum".fields;
        for (fields, 0..) |field, i| {
            table[field.value] = values[i];
        }
    }
    return table;
}

fn enumCount(comptime E: type) comptime_int {
    return @typeInfo(E).@"enum".fields.len;
}

const Resistance = enum(u8) {
    very_low = 0,
    low = 1,
    medium = 2,
    high = 3,
    very_high = 4,
};

pub fn main() void {
    printAllValues(Resistance);

    // Lookup table สำหรับค่า ohm ที่ comptime
    const ohm_values = buildLookupTable(
        Resistance,
        &[_]f64{ 10.0, 100.0, 1000.0, 10000.0, 100000.0 },
    );

    std.debug.print("\nResistance values:\n", .{});
    for (ohm_values, 0..) |ohm, i| {
        const r: Resistance = @enumFromInt(i);
        std.debug.print("  {s}: {d} Ω\n", .{ @tagName(r), ohm });
    }
}
```

### Comptime String-to-Enum Conversion

```zig
const std = @import("std");

const Color = enum {
    red,
    green,
    blue,
    white,
    black,
};

// แปลง string เป็น enum ที่ comptime
fn colorFromString(comptime name: []const u8) Color {
    const fields = @typeInfo(Color).@"enum".fields;
    inline for (fields) |field| {
        if (std.mem.eql(u8, field.name, name)) {
            return @enumFromInt(field.value);
        }
    }
    @compileError("Unknown color: " ++ name);
}

// Runtime version
fn colorFromStringRuntime(name: []const u8) ?Color {
    const fields = @typeInfo(Color).@"enum".fields;
    inline for (fields) |field| {
        if (std.mem.eql(u8, field.name, name)) {
            return @enumFromInt(field.value);
        }
    }
    return null;
}

pub fn main() void {
    // Comptime lookup
    const c = comptime colorFromString("green");
    std.debug.print("Comptime color: {s}\n", .{@tagName(c)});

    // Runtime lookup
    const names = [_][]const u8{ "red", "blue", "purple", "white" };
    for (names) |name| {
        if (colorFromStringRuntime(name)) |color| {
            std.debug.print("{s} -> {s}\n", .{ name, @tagName(color) });
        } else {
            std.debug.print("{s} -> not found\n", .{name});
        }
    }
}
```

---

## Step 118: Enum Iteration

### การ Iterate ผ่าน Enum Values

```zig
const std = @import("std");

const Suit = enum {
    clubs,
    diamonds,
    hearts,
    spades,
};

pub fn main() void {
    // วิธีที่ 1: ใช้ std.enums.values
    const all_suits = std.enums.values(Suit);
    std.debug.print("All suits:\n", .{});
    for (all_suits) |suit| {
        std.debug.print("  {s}\n", .{@tagName(suit)});
    }

    // วิธีที่ 2: ใช้ @typeInfo
    std.debug.print("\nUsing typeInfo:\n", .{});
    const fields = @typeInfo(Suit).@"enum".fields;
    inline for (fields) |field| {
        const suit: Suit = @enumFromInt(field.value);
        std.debug.print("  {s} = {d}\n", .{ field.name, @intFromEnum(suit) });
    }
}
```

### Inline For กับ Enum

```zig
const std = @import("std");

const Operation = enum {
    add,
    subtract,
    multiply,
    divide,
};

pub fn main() void {
    // inline for กับ enum fields
    inline for (@typeInfo(Operation).@"enum".fields) |field| {
        // field.name เป็น comptime string
        // สามารถใช้ใน comptime context ได้
        std.debug.print("Operation: {s}\n", .{field.name});
    }

    // สร้าง array จาก enum values
    var count: usize = 0;
    inline for (@typeInfo(Operation).@"enum".fields) |_| {
        count += 1;
    }
    std.debug.print("Total operations: {d}\n", .{count});
}
```

### std.enums.values และ std.enums.tagName

```zig
const std = @import("std");

const Status = enum {
    pending,
    processing,
    completed,
    failed,
    cancelled,
};

pub fn main() void {
    // std.enums.values คืน []const Status
    const statuses = std.enums.values(Status);
    std.debug.print("Status values ({d} total):\n", .{statuses.len});

    for (statuses) |status| {
        std.debug.print("  [{d}] {s}\n", .{
            @intFromEnum(status),
            @tagName(status),
        });
    }

    // ใช้ std.enums.nameCast สำหรับ name lookup
    const name = "completed";
    if (std.enums.nameCast(Status, name)) |s| {
        std.debug.print("\nFound: {s}\n", .{@tagName(s)});
    }
}
```

### Enum Iterator Pattern

```zig
const std = @import("std");

const Weekday = enum(u8) {
    monday = 1,
    tuesday = 2,
    wednesday = 3,
    thursday = 4,
    friday = 5,
    saturday = 6,
    sunday = 7,

    pub fn next(self: Weekday) Weekday {
        const val = @intFromEnum(self);
        const next_val = if (val == 7) @as(u8, 1) else val + 1;
        return @enumFromInt(next_val);
    }

    pub fn isWeekend(self: Weekday) bool {
        return self == .saturday or self == .sunday;
    }
};

pub fn main() void {
    // Iterate ผ่านวันในสัปดาห์
    var day = Weekday.monday;
    var i: u8 = 0;
    while (i < 7) : (i += 1) {
        std.debug.print("{s}: {s}\n", .{
            @tagName(day),
            if (day.isWeekend()) "วันหยุด" else "วันทำงาน",
        });
        day = day.next();
    }
}
```

### Building Dispatch Tables จาก Enum

```zig
const std = @import("std");

const Command = enum {
    help,
    version,
    list,
    quit,
};

const Handler = *const fn () void;

fn handleHelp() void {
    std.debug.print("คำสั่งที่ใช้ได้: help, version, list, quit\n", .{});
}

fn handleVersion() void {
    std.debug.print("เวอร์ชัน: 1.0.0\n", .{});
}

fn handleList() void {
    std.debug.print("รายการ: item1, item2, item3\n", .{});
}

fn handleQuit() void {
    std.debug.print("ลาก่อน!\n", .{});
}

// สร้าง dispatch table จาก enum
const handlers = blk: {
    var table: [@typeInfo(Command).@"enum".fields.len]Handler = undefined;
    table[@intFromEnum(Command.help)] = handleHelp;
    table[@intFromEnum(Command.version)] = handleVersion;
    table[@intFromEnum(Command.list)] = handleList;
    table[@intFromEnum(Command.quit)] = handleQuit;
    break :blk table;
};

pub fn main() void {
    const cmds = [_]Command{ .help, .version, .list, .quit };
    for (cmds) |cmd| {
        std.debug.print("--- {s} ---\n", .{@tagName(cmd)});
        handlers[@intFromEnum(cmd)]();
    }
}
```

---

## Step 119: Tagged Union กับ Enum

### Tagged Union พื้นฐาน

Tagged union คือ union ที่ใช้ enum เป็น discriminant:

```zig
const std = @import("std");

// Tagged union: แต่ละ case มีข้อมูลที่ต่างกัน
const Shape = union(enum) {
    circle: f64,           // radius
    rectangle: struct { width: f64, height: f64 },
    triangle: struct { base: f64, height: f64 },
    point: void,           // ไม่มีข้อมูล
};

fn area(shape: Shape) f64 {
    return switch (shape) {
        .circle => |r| std.math.pi * r * r,
        .rectangle => |rect| rect.width * rect.height,
        .triangle => |tri| 0.5 * tri.base * tri.height,
        .point => 0.0,
    };
}

fn perimeter(shape: Shape) f64 {
    return switch (shape) {
        .circle => |r| 2.0 * std.math.pi * r,
        .rectangle => |rect| 2.0 * (rect.width + rect.height),
        .triangle => |tri| {
            // สมมติเป็น isosceles triangle
            const hypotenuse = std.math.sqrt(
                (tri.base / 2.0) * (tri.base / 2.0) + tri.height * tri.height,
            );
            return tri.base + 2.0 * hypotenuse;
        },
        .point => 0.0,
    };
}

pub fn main() void {
    const shapes = [_]Shape{
        .{ .circle = 5.0 },
        .{ .rectangle = .{ .width = 4.0, .height = 6.0 } },
        .{ .triangle = .{ .base = 3.0, .height = 4.0 } },
        .point,
    };

    for (shapes) |shape| {
        std.debug.print("{s}: area={d:.2}, perimeter={d:.2}\n", .{
            @tagName(shape),
            area(shape),
            perimeter(shape),
        });
    }
}
```

### Tagged Union กับ Named Enum

```zig
const std = @import("std");

// ประกาศ enum แยก แล้วใช้กับ union
const ValueKind = enum {
    integer,
    float,
    boolean,
    string,
    null_value,
};

const Value = union(ValueKind) {
    integer: i64,
    float: f64,
    boolean: bool,
    string: []const u8,
    null_value: void,
};

fn printValue(v: Value) void {
    switch (v) {
        .integer => |n| std.debug.print("int: {d}", .{n}),
        .float => |f| std.debug.print("float: {d:.3}", .{f}),
        .boolean => |b| std.debug.print("bool: {}", .{b}),
        .string => |s| std.debug.print("string: \"{s}\"", .{s}),
        .null_value => std.debug.print("null", .{}),
    }
}

fn typeOf(v: Value) ValueKind {
    return std.meta.activeTag(v);
}

pub fn main() void {
    const values = [_]Value{
        .{ .integer = 42 },
        .{ .float = 3.14159 },
        .{ .boolean = true },
        .{ .string = "สวัสดี" },
        .null_value,
    };

    for (values) |v| {
        std.debug.print("({s}) ", .{@tagName(typeOf(v))});
        printValue(v);
        std.debug.print("\n", .{});
    }
}
```

### Result Type ด้วย Tagged Union

```zig
const std = @import("std");

const ParseError = enum {
    empty_input,
    invalid_character,
    overflow,
    underflow,
};

const ParseResult = union(enum) {
    ok: i64,
    err: ParseError,
};

fn parseInteger(s: []const u8) ParseResult {
    if (s.len == 0) return .{ .err = .empty_input };

    var result: i64 = 0;
    var negative = false;
    var start: usize = 0;

    if (s[0] == '-') {
        negative = true;
        start = 1;
    } else if (s[0] == '+') {
        start = 1;
    }

    for (s[start..]) |c| {
        if (c < '0' or c > '9') {
            return .{ .err = .invalid_character };
        }
        const digit = @as(i64, c - '0');
        result = result * 10 + digit;
    }

    if (negative) result = -result;
    return .{ .ok = result };
}

pub fn main() void {
    const inputs = [_][]const u8{ "42", "-17", "abc", "", "9999999999999999999" };

    for (inputs) |input| {
        const result = parseInteger(input);
        switch (result) {
            .ok => |n| std.debug.print("\"{s}\" -> {d}\n", .{ input, n }),
            .err => |e| std.debug.print("\"{s}\" -> error: {s}\n", .{ input, @tagName(e) }),
        }
    }
}
```

### State Machine ด้วย Tagged Union

```zig
const std = @import("std");

// TCP Connection State Machine
const TcpState = union(enum) {
    closed: void,
    listening: struct { port: u16 },
    syn_received: struct { port: u16, client_seq: u32 },
    established: struct {
        port: u16,
        local_seq: u32,
        remote_seq: u32,
    },
    fin_wait: struct { port: u16 },
    time_wait: void,

    pub fn isActive(self: TcpState) bool {
        return switch (self) {
            .established => true,
            else => false,
        };
    }

    pub fn describe(self: TcpState) void {
        switch (self) {
            .closed => std.debug.print("CLOSED\n", .{}),
            .listening => |l| std.debug.print("LISTENING on port {d}\n", .{l.port}),
            .syn_received => |s| std.debug.print("SYN_RECEIVED (port={d}, seq={d})\n", .{ s.port, s.client_seq }),
            .established => |e| std.debug.print("ESTABLISHED (port={d})\n", .{e.port}),
            .fin_wait => |f| std.debug.print("FIN_WAIT (port={d})\n", .{f.port}),
            .time_wait => std.debug.print("TIME_WAIT\n", .{}),
        }
    }
};

pub fn main() void {
    var state = TcpState{ .closed = {} };
    state.describe();

    state = TcpState{ .listening = .{ .port = 8080 } };
    state.describe();

    state = TcpState{ .established = .{
        .port = 8080,
        .local_seq = 1000,
        .remote_seq = 2000,
    } };
    state.describe();
    std.debug.print("Active: {}\n", .{state.isActive()});

    state = TcpState{ .closed = {} };
    state.describe();
}
```

---

## Step 120: Enum Patterns ใน Real-World Code

### Pattern 1: State Machine

```zig
const std = @import("std");

const VendingMachineState = enum {
    idle,
    coin_inserted,
    selecting,
    dispensing,
    returning_change,
    error_state,
};

const VendingMachineEvent = enum {
    insert_coin,
    select_item,
    item_available,
    item_not_available,
    dispense_complete,
    cancel,
    reset,
};

const VendingMachine = struct {
    state: VendingMachineState,
    balance: u32,  // สตางค์

    const Self = @This();

    pub fn init() Self {
        return .{ .state = .idle, .balance = 0 };
    }

    pub fn transition(self: *Self, event: VendingMachineEvent) bool {
        const new_state: VendingMachineState = switch (self.state) {
            .idle => switch (event) {
                .insert_coin => blk: {
                    self.balance += 500; // เพิ่ม 5 บาท
                    break :blk .coin_inserted;
                },
                .reset => .idle,
                else => return false,
            },
            .coin_inserted => switch (event) {
                .insert_coin => blk: {
                    self.balance += 500;
                    break :blk .coin_inserted;
                },
                .select_item => .selecting,
                .cancel => blk: {
                    self.balance = 0;
                    break :blk .idle;
                },
                else => return false,
            },
            .selecting => switch (event) {
                .item_available => .dispensing,
                .item_not_available => .coin_inserted,
                .cancel => blk: {
                    self.balance = 0;
                    break :blk .idle;
                },
                else => return false,
            },
            .dispensing => switch (event) {
                .dispense_complete => if (self.balance > 0)
                    .returning_change
                else
                    .idle,
                else => return false,
            },
            .returning_change => switch (event) {
                .dispense_complete => blk: {
                    self.balance = 0;
                    break :blk .idle;
                },
                else => return false,
            },
            .error_state => switch (event) {
                .reset => .idle,
                else => return false,
            },
        };

        std.debug.print("  {s} --[{s}]--> {s}\n", .{
            @tagName(self.state),
            @tagName(event),
            @tagName(new_state),
        });
        self.state = new_state;
        return true;
    }
};

pub fn main() void {
    var machine = VendingMachine.init();
    std.debug.print("State Machine Demo:\n", .{});

    _ = machine.transition(.insert_coin);
    _ = machine.transition(.insert_coin);
    _ = machine.transition(.select_item);
    _ = machine.transition(.item_available);
    _ = machine.transition(.dispense_complete);
    _ = machine.transition(.dispense_complete);

    std.debug.print("Balance: {d} สตางค์\n", .{machine.balance});
}
```

### Pattern 2: Error Handling ด้วย Enum

```zig
const std = @import("std");

const DatabaseError = enum {
    connection_failed,
    query_timeout,
    constraint_violation,
    record_not_found,
    permission_denied,
    disk_full,

    pub fn isRetryable(self: DatabaseError) bool {
        return switch (self) {
            .connection_failed, .query_timeout => true,
            .constraint_violation, .record_not_found,
            .permission_denied, .disk_full => false,
        };
    }

    pub fn httpStatusCode(self: DatabaseError) u16 {
        return switch (self) {
            .connection_failed, .query_timeout => 503,
            .constraint_violation => 409,
            .record_not_found => 404,
            .permission_denied => 403,
            .disk_full => 507,
        };
    }

    pub fn message(self: DatabaseError) []const u8 {
        return switch (self) {
            .connection_failed => "ไม่สามารถเชื่อมต่อฐานข้อมูลได้",
            .query_timeout => "Query หมดเวลา",
            .constraint_violation => "ข้อมูลละเมิด constraint",
            .record_not_found => "ไม่พบข้อมูล",
            .permission_denied => "ไม่มีสิทธิ์เข้าถึง",
            .disk_full => "พื้นที่ดิสก์เต็ม",
        };
    }
};

fn queryDatabase(id: u32) !u32 {
    if (id == 0) return error.constraint_violation; // ใช้ Zig error
    if (id > 1000) return error.record_not_found;
    return id * 2;
}

pub fn main() void {
    const errors = [_]DatabaseError{
        .connection_failed,
        .record_not_found,
        .query_timeout,
        .permission_denied,
    };

    for (errors) |err| {
        std.debug.print("[{d}] {s} (retryable: {})\n", .{
            err.httpStatusCode(),
            err.message(),
            err.isRetryable(),
        });
    }
}
```

### Pattern 3: Configuration ด้วย Enum

```zig
const std = @import("std");

const LogLevel = enum(u8) {
    trace = 0,
    debug = 1,
    info = 2,
    warn = 3,
    err = 4,
    fatal = 5,

    pub fn shouldLog(self: LogLevel, min_level: LogLevel) bool {
        return @intFromEnum(self) >= @intFromEnum(min_level);
    }

    pub fn prefix(self: LogLevel) []const u8 {
        return switch (self) {
            .trace => "[TRACE]",
            .debug => "[DEBUG]",
            .info => "[INFO] ",
            .warn => "[WARN] ",
            .err => "[ERROR]",
            .fatal => "[FATAL]",
        };
    }
};

const Logger = struct {
    min_level: LogLevel,

    pub fn log(self: Logger, level: LogLevel, comptime fmt: []const u8, args: anytype) void {
        if (!level.shouldLog(self.min_level)) return;
        std.debug.print("{s} " ++ fmt ++ "\n", .{level.prefix()} ++ args);
    }

    pub fn trace(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.trace, fmt, args);
    }

    pub fn debug(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.debug, fmt, args);
    }

    pub fn info(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.info, fmt, args);
    }

    pub fn warn(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.warn, fmt, args);
    }

    pub fn err(self: Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.err, fmt, args);
    }
};

pub fn main() void {
    const logger = Logger{ .min_level = .info };

    logger.trace("This won't show", .{});
    logger.debug("Neither will this", .{});
    logger.info("Application started", .{});
    logger.warn("Memory usage high: {d}%", .{85});
    logger.err("Failed to connect to {s}", .{"db.example.com"});
}
```

### Pattern 4: Command Pattern

```zig
const std = @import("std");

const EditorAction = union(enum) {
    insert: struct {
        position: usize,
        text: []const u8,
    },
    delete: struct {
        position: usize,
        length: usize,
    },
    move_cursor: struct {
        line: i32,
        column: i32,
    },
    undo: void,
    redo: void,
    save: void,

    pub fn describe(self: EditorAction) void {
        switch (self) {
            .insert => |ins| std.debug.print(
                "Insert \"{s}\" at pos {d}\n",
                .{ ins.text, ins.position },
            ),
            .delete => |del| std.debug.print(
                "Delete {d} chars at pos {d}\n",
                .{ del.length, del.position },
            ),
            .move_cursor => |mv| std.debug.print(
                "Move cursor to line {d}, col {d}\n",
                .{ mv.line, mv.column },
            ),
            .undo => std.debug.print("Undo\n", .{}),
            .redo => std.debug.print("Redo\n", .{}),
            .save => std.debug.print("Save\n", .{}),
        }
    }
};

pub fn main() void {
    const actions = [_]EditorAction{
        .{ .move_cursor = .{ .line = 10, .column = 5 } },
        .{ .insert = .{ .position = 100, .text = "Hello, Zig!" } },
        .{ .move_cursor = .{ .line = 10, .column = 16 } },
        .{ .delete = .{ .position = 105, .length = 4 } },
        .undo,
        .save,
    };

    std.debug.print("Editor actions:\n", .{});
    for (actions) |action| {
        std.debug.print("  ", .{});
        action.describe();
    }
}
```

### Pattern 5: Builder Pattern ด้วย Enum

```zig
const std = @import("std");

const QueryOperator = enum {
    equals,
    not_equals,
    greater_than,
    less_than,
    contains,
    starts_with,
    ends_with,

    pub fn toSql(self: QueryOperator) []const u8 {
        return switch (self) {
            .equals => "=",
            .not_equals => "!=",
            .greater_than => ">",
            .less_than => "<",
            .contains => "LIKE",
            .starts_with => "LIKE",
            .ends_with => "LIKE",
        };
    }
};

const SortOrder = enum {
    ascending,
    descending,

    pub fn toSql(self: SortOrder) []const u8 {
        return switch (self) {
            .ascending => "ASC",
            .descending => "DESC",
        };
    }
};

const QueryCondition = struct {
    field: []const u8,
    operator: QueryOperator,
    value: []const u8,
};

const QueryBuilder = struct {
    table: []const u8,
    conditions: std.ArrayList(QueryCondition),
    sort_field: ?[]const u8,
    sort_order: SortOrder,
    limit: ?usize,

    pub fn init(allocator: std.mem.Allocator, table: []const u8) QueryBuilder {
        return .{
            .table = table,
            .conditions = std.ArrayList(QueryCondition).init(allocator),
            .sort_field = null,
            .sort_order = .ascending,
            .limit = null,
        };
    }

    pub fn where(self: *QueryBuilder, field: []const u8, op: QueryOperator, value: []const u8) !*QueryBuilder {
        try self.conditions.append(.{
            .field = field,
            .operator = op,
            .value = value,
        });
        return self;
    }

    pub fn orderBy(self: *QueryBuilder, field: []const u8, order: SortOrder) *QueryBuilder {
        self.sort_field = field;
        self.sort_order = order;
        return self;
    }

    pub fn take(self: *QueryBuilder, n: usize) *QueryBuilder {
        self.limit = n;
        return self;
    }

    pub fn deinit(self: *QueryBuilder) void {
        self.conditions.deinit();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var query = QueryBuilder.init(allocator, "users");
    defer query.deinit();

    _ = try query.where("age", .greater_than, "18");
    _ = try query.where("status", .equals, "active");
    _ = query.orderBy("name", .ascending);
    _ = query.take(10);

    std.debug.print("Query on table: {s}\n", .{query.table});
    std.debug.print("Conditions ({d}):\n", .{query.conditions.items.len});
    for (query.conditions.items) |cond| {
        std.debug.print("  {s} {s} '{s}'\n", .{
            cond.field,
            cond.operator.toSql(),
            cond.value,
        });
    }
    if (query.sort_field) |sf| {
        std.debug.print("ORDER BY {s} {s}\n", .{ sf, query.sort_order.toSql() });
    }
    if (query.limit) |l| {
        std.debug.print("LIMIT {d}\n", .{l});
    }
}
```

---

## สรุป: ข้อดีของ Zig Enum เทียบกับ C Enum

| คุณสมบัติ | C Enum | Zig Enum |
|-----------|--------|----------|
| Type Safety | ไม่มี (เป็นแค่ integer) | มี (compile error ถ้า type ไม่ตรง) |
| Methods | ไม่มี | มีได้ |
| Exhaustive Switch | ไม่บังคับ | บังคับโดย compiler |
| Integer Backing Type | ขึ้นอยู่กับ compiler | กำหนดได้ชัดเจน |
| Comptime Operations | ไม่มี | ทำได้เต็มที่ |
| Tagged Union | ไม่มี | ใช้ union(enum) ได้ |
| Iteration | ต้องเขียนเอง | std.enums.values() |
| @tagName | ไม่มี | มี built-in |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Enum พื้นฐาน
สร้าง enum `CardRank` สำหรับไพ่ตั้งแต่ Ace ถึง King พร้อม method:
- `value()` คืนค่าตัวเลข (Ace=1, 2-10=ตามหน้า, Jack=11, Queen=12, King=13)
- `isRoyalty()` คืน true สำหรับ Jack, Queen, King
- `symbol()` คืน string ตัวอักษรไพ่

```zig
const CardRank = enum(u8) {
    ace = 1,
    two = 2,
    // ... ต่อไปจนถึง king = 13
    // เพิ่ม methods ที่นี่
};
```

### แบบฝึกหัดที่ 2: State Machine
สร้าง state machine สำหรับ Traffic Light ที่:
- มี 3 state: red, yellow, green
- มี method `next()` ที่เปลี่ยน state ตามลำดับ (red -> green -> yellow -> red)
- มี method `duration()` ที่คืนเวลาเป็นวินาที (red=30, yellow=5, green=25)

```zig
const TrafficLight = enum {
    red,
    yellow,
    green,

    pub fn next(self: TrafficLight) TrafficLight {
        // TODO: implement
    }

    pub fn duration(self: TrafficLight) u32 {
        // TODO: implement
    }
};
```

### แบบฝึกหัดที่ 3: Tagged Union
สร้าง `JsonValue` tagged union ที่รองรับ:
- null
- boolean
- number (f64)
- string ([]const u8)
- array (ใช้ ArrayList หรือ slice)

พร้อม method:
- `typeStr()` คืนชื่อ type เป็น string
- `toBool()` แปลง value เป็น bool (null=false, 0=false, ""=false, อื่นๆ=true)

### แบบฝึกหัดที่ 4: Enum Iteration
เขียน function `printEnumTable` ที่รับ comptime enum type และ print ออกมาเป็นตาราง:
```
Name          | Value | Tag Name
-------------|-------|--------
monday       |     0 | "monday"
tuesday      |     1 | "tuesday"
...
```

### แบบฝึกหัดที่ 5: Real-world Pattern
สร้าง simple task queue ด้วย enum:
- `TaskStatus`: pending, running, completed, failed, cancelled
- `TaskPriority`: low, normal, high, urgent
- struct `Task` ที่มี status และ priority
- method `canTransitionTo(new_status)` ที่ validate การเปลี่ยน status

---

## เฉลยแบบฝึกหัด

### เฉลยที่ 1

```zig
const std = @import("std");

const CardRank = enum(u8) {
    ace = 1,
    two = 2,
    three = 3,
    four = 4,
    five = 5,
    six = 6,
    seven = 7,
    eight = 8,
    nine = 9,
    ten = 10,
    jack = 11,
    queen = 12,
    king = 13,

    pub fn value(self: CardRank) u8 {
        return @intFromEnum(self);
    }

    pub fn isRoyalty(self: CardRank) bool {
        return switch (self) {
            .jack, .queen, .king => true,
            else => false,
        };
    }

    pub fn symbol(self: CardRank) []const u8 {
        return switch (self) {
            .ace => "A",
            .two => "2",
            .three => "3",
            .four => "4",
            .five => "5",
            .six => "6",
            .seven => "7",
            .eight => "8",
            .nine => "9",
            .ten => "10",
            .jack => "J",
            .queen => "Q",
            .king => "K",
        };
    }
};

pub fn main() void {
    const cards = [_]CardRank{ .ace, .ten, .jack, .queen, .king };
    for (cards) |card| {
        std.debug.print("{s} (value={d}, royalty={})\n", .{
            card.symbol(),
            card.value(),
            card.isRoyalty(),
        });
    }
}
```

### เฉลยที่ 2

```zig
const std = @import("std");

const TrafficLight = enum {
    red,
    yellow,
    green,

    pub fn next(self: TrafficLight) TrafficLight {
        return switch (self) {
            .red => .green,
            .green => .yellow,
            .yellow => .red,
        };
    }

    pub fn duration(self: TrafficLight) u32 {
        return switch (self) {
            .red => 30,
            .yellow => 5,
            .green => 25,
        };
    }
};

pub fn main() void {
    var light = TrafficLight.red;
    var total_time: u32 = 0;

    std.debug.print("Traffic Light Simulation (2 cycles):\n", .{});
    var i: u8 = 0;
    while (i < 6) : (i += 1) {
        std.debug.print("  {s} for {d}s\n", .{ @tagName(light), light.duration() });
        total_time += light.duration();
        light = light.next();
    }
    std.debug.print("Total time: {d}s\n", .{total_time});
}
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Enum พื้นฐาน** - การประกาศและใช้งาน enum ใน Zig พร้อม type safety ที่เข้มงวดกว่า C

2. **Integer Backing** - การกำหนดชนิด integer ให้ enum และแปลงระหว่าง enum กับ integer ด้วย `@intFromEnum` และ `@enumFromInt`

3. **Enum Methods** - การเพิ่ม method ให้ enum ทั้ง instance method และ static method

4. **@tagName และ @enumFromInt** - builtins สำหรับแปลง enum เป็น string และ integer เป็น enum

5. **Exhaustive Switch** - Zig บังคับให้ handle ทุก case ใน switch ป้องกัน bug จากการลืม case

6. **Non-exhaustive Enum** - ใช้ `_` field เมื่อต้องการรองรับค่าที่ไม่รู้จัก

7. **Comptime Operations** - การใช้ `@typeInfo` และ `inline for` กับ enum ที่ compile time

8. **Enum Iteration** - ใช้ `std.enums.values()` และ `@typeInfo` เพื่อ iterate ผ่าน enum values

9. **Tagged Union** - ใช้ `union(enum)` สร้าง algebraic data types ที่ type-safe

10. **Real-world Patterns** - State machine, Error handling, Configuration, Command pattern, Builder pattern

Enum ใน Zig เป็นหนึ่งในเครื่องมือที่ทรงพลังที่สุด เมื่อใช้ร่วมกับ tagged union จะทำให้เขียน code ที่ปลอดภัย, ชัดเจน และ maintainable ได้ง่ายขึ้นมาก

---

*ไปยัง Part 13: Error Handling ใน Zig ->*
