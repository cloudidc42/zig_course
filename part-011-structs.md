# ภาค 11: Structs ใน Zig (Steps 101-110)

Structs คือรากฐานสำคัญของการเขียนโปรแกรมเชิงวัตถุ (OOP) ใน Zig แม้ว่า Zig จะไม่มี class แบบภาษา OOP ทั่วไป แต่ struct ใน Zig มีความสามารถเทียบเท่าและยิ่งกว่าในหลายด้าน บทนี้จะพาคุณเรียนรู้ struct ตั้งแต่พื้นฐานจนถึงขั้นสูง

---

## Step 101: การสร้าง Struct พื้นฐาน

### Struct คืออะไร?

Struct (Structure) คือชนิดข้อมูลแบบผสม (composite type) ที่ช่วยให้เราจัดกลุ่มข้อมูลที่เกี่ยวข้องกันไว้ด้วยกัน เหมือนกับการสร้าง "แม่แบบ" สำหรับวัตถุในโลกจริง

ตัวอย่างเช่น ถ้าเราต้องการเก็บข้อมูลจุดในระนาบ 2 มิติ เราต้องการทั้ง x และ y พร้อมกัน แทนที่จะใช้สองตัวแปรแยกกัน เราใช้ struct เพื่อห่อหุ้มทั้งสองไว้ด้วยกัน

### Syntax พื้นฐาน

```zig
const TypeName = struct {
    field1: Type1,
    field2: Type2,
    // ...
};
```

### ตัวอย่างแรก: Point

```zig
const std = @import("std");

// การสร้าง struct พื้นฐาน
const Point = struct {
    x: f64,
    y: f64,
};

pub fn main() void {
    // การสร้าง instance ของ struct
    const p = Point{ .x = 3.0, .y = 4.0 };
    std.debug.print("Point: ({d}, {d})\n", .{ p.x, p.y });
}
```

**ผลลัพธ์:**
```
Point: (3, 4)
```

### ตัวอย่างที่ 2: Person

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,
    height: f32,
};

pub fn main() void {
    const alice = Person{
        .name = "Alice",
        .age = 30,
        .height = 165.5,
    };

    std.debug.print("ชื่อ: {s}\n", .{alice.name});
    std.debug.print("อายุ: {d} ปี\n", .{alice.age});
    std.debug.print("ส่วนสูง: {d:.1} ซม.\n", .{alice.height});
}
```

**ผลลัพธ์:**
```
ชื่อ: Alice
อายุ: 30 ปี
ส่วนสูง: 165.5 ซม.
```

### ตัวอย่างที่ 3: Rectangle

```zig
const std = @import("std");

const Rectangle = struct {
    width: f64,
    height: f64,
};

pub fn main() void {
    const rect = Rectangle{
        .width = 10.0,
        .height = 5.0,
    };

    const area = rect.width * rect.height;
    const perimeter = 2.0 * (rect.width + rect.height);

    std.debug.print("กว้าง: {d}\n", .{rect.width});
    std.debug.print("สูง: {d}\n", .{rect.height});
    std.debug.print("พื้นที่: {d}\n", .{area});
    std.debug.print("เส้นรอบรูป: {d}\n", .{perimeter});
}
```

**ผลลัพธ์:**
```
กว้าง: 10
สูง: 5
พื้นที่: 50
เส้นรอบรูป: 30
```

### หลักการตั้งชื่อ Struct

ใน Zig เราใช้ **PascalCase** (ขึ้นต้นด้วยตัวพิมพ์ใหญ่) สำหรับชื่อ struct เสมอ เช่น `Point`, `Person`, `Rectangle`, `HttpClient` เป็นต้น

### Struct เป็น Type

สิ่งสำคัญที่ต้องเข้าใจคือ struct ใน Zig เป็น **type** เหมือนกับ `u32`, `f64`, `bool` ทุกประการ เราสามารถใช้มันเป็นชนิดของ parameter, return type, หรือ field ของ struct อื่นได้

```zig
const std = @import("std");

const Color = struct {
    r: u8,
    g: u8,
    b: u8,
};

// ใช้ Color เป็น parameter type
fn printColor(color: Color) void {
    std.debug.print("RGB({d}, {d}, {d})\n", .{ color.r, color.g, color.b });
}

// ใช้ Color เป็น return type
fn makeRed() Color {
    return Color{ .r = 255, .g = 0, .b = 0 };
}

pub fn main() void {
    const red = makeRed();
    printColor(red);

    const blue = Color{ .r = 0, .g = 0, .b = 255 };
    printColor(blue);
}
```

**ผลลัพธ์:**
```
RGB(255, 0, 0)
RGB(0, 0, 255)
```

---

## Step 102: Fields - Default Values, Types

### Field ใน Struct

Field คือตัวแปรที่อยู่ภายใน struct แต่ละ field มีชื่อและชนิดข้อมูล สามารถมีค่าเริ่มต้น (default value) ได้ด้วย

### Syntax ของ Field

```zig
const MyStruct = struct {
    // field ไม่มี default value
    required_field: u32,

    // field มี default value
    optional_field: u32 = 42,

    // field ที่เป็น optional type
    nullable_field: ?u32 = null,

    // field ที่เป็น string
    name: []const u8 = "default",
};
```

### ตัวอย่าง: Config Struct

```zig
const std = @import("std");

const ServerConfig = struct {
    // fields จำเป็นต้องระบุ (ไม่มี default)
    host: []const u8,
    port: u16,

    // fields ที่มีค่า default
    max_connections: u32 = 100,
    timeout_seconds: u32 = 30,
    debug_mode: bool = false,
    log_level: u8 = 2, // 0=error, 1=warn, 2=info, 3=debug
};

pub fn main() void {
    // ต้องระบุ host และ port เท่านั้น
    const config = ServerConfig{
        .host = "localhost",
        .port = 8080,
    };

    std.debug.print("Server: {s}:{d}\n", .{ config.host, config.port });
    std.debug.print("Max connections: {d}\n", .{config.max_connections});
    std.debug.print("Timeout: {d}s\n", .{config.timeout_seconds});
    std.debug.print("Debug: {}\n", .{config.debug_mode});

    // Override บาง default values
    const dev_config = ServerConfig{
        .host = "0.0.0.0",
        .port = 3000,
        .debug_mode = true,
        .log_level = 3,
    };

    std.debug.print("\nDev Server: {s}:{d}\n", .{ dev_config.host, dev_config.port });
    std.debug.print("Debug: {}\n", .{dev_config.debug_mode});
    std.debug.print("Log level: {d}\n", .{dev_config.log_level});
}
```

**ผลลัพธ์:**
```
Server: localhost:8080
Max connections: 100
Timeout: 30s
Debug: false

Dev Server: 0.0.0.0:3000
Debug: true
Log level: 3
```

### ชนิดข้อมูลต่างๆ สำหรับ Fields

```zig
const std = @import("std");

const DataTypes = struct {
    // Integer types
    small_int: i8 = 0,
    unsigned_int: u32 = 0,
    big_int: i64 = 0,
    pointer_size: usize = 0,

    // Float types
    single: f32 = 0.0,
    double: f64 = 0.0,

    // Boolean
    flag: bool = false,

    // Characters and strings
    letter: u8 = 'A',
    text: []const u8 = "",

    // Optional (nullable)
    maybe_number: ?i32 = null,

    // Comptime-known values
    pi: comptime_float = 3.14159265358979,
};

pub fn main() void {
    const data = DataTypes{};

    std.debug.print("small_int: {d}\n", .{data.small_int});
    std.debug.print("unsigned_int: {d}\n", .{data.unsigned_int});
    std.debug.print("single: {d}\n", .{data.single});
    std.debug.print("flag: {}\n", .{data.flag});
    std.debug.print("letter: {c}\n", .{data.letter});
    std.debug.print("maybe_number: {?d}\n", .{data.maybe_number});
    std.debug.print("pi: {d:.5}\n", .{data.pi});
}
```

### Fields ที่เป็น Arrays

```zig
const std = @import("std");

const Matrix2x2 = struct {
    data: [2][2]f64 = .{
        .{ 1.0, 0.0 },
        .{ 0.0, 1.0 },
    },
};

const Inventory = struct {
    items: [10][]const u8 = .{"", "", "", "", "", "", "", "", "", ""},
    count: usize = 0,
};

pub fn main() void {
    const identity = Matrix2x2{};
    std.debug.print("Matrix[0][0]: {d}\n", .{identity.data[0][0]});
    std.debug.print("Matrix[1][1]: {d}\n", .{identity.data[1][1]});

    var inv = Inventory{};
    inv.items[0] = "sword";
    inv.items[1] = "shield";
    inv.count = 2;
    std.debug.print("Items: {s}, {s} (total: {d})\n", .{
        inv.items[0],
        inv.items[1],
        inv.count,
    });
}
```

---

## Step 103: การสร้าง Instance และเข้าถึง Fields

### การสร้าง Instance

มีหลายวิธีในการสร้าง instance ของ struct:

#### 1. Named Field Initialization (แนะนำ)

```zig
const Point = struct {
    x: f64,
    y: f64,
};

// ใช้ชื่อ field อย่างชัดเจน
const p1 = Point{ .x = 1.0, .y = 2.0 };
```

#### 2. Default Initialization

```zig
const Config = struct {
    width: u32 = 800,
    height: u32 = 600,
};

// ใช้ทุก default value
const default_config = Config{};
```

#### 3. Partial Initialization พร้อม Defaults

```zig
const Config = struct {
    width: u32 = 800,
    height: u32 = 600,
    fullscreen: bool = false,
};

// ระบุแค่บางส่วน
const custom_config = Config{ .fullscreen = true };
```

### การเข้าถึง Fields

```zig
const std = @import("std");

const Student = struct {
    name: []const u8,
    student_id: u32,
    gpa: f32 = 0.0,
    is_active: bool = true,
};

pub fn main() void {
    const student = Student{
        .name = "สมชาย ใจดี",
        .student_id = 12345,
        .gpa = 3.75,
    };

    // เข้าถึง field ด้วย dot notation
    std.debug.print("ชื่อ: {s}\n", .{student.name});
    std.debug.print("รหัสนักศึกษา: {d}\n", .{student.student_id});
    std.debug.print("GPA: {d:.2}\n", .{student.gpa});
    std.debug.print("สถานะ: {s}\n", .{
        if (student.is_active) "ลงทะเบียนอยู่" else "ไม่ได้ลงทะเบียน",
    });
}
```

### Mutable vs Immutable Instances

```zig
const std = @import("std");

const Counter = struct {
    value: i32 = 0,
};

pub fn main() void {
    // const = immutable (ไม่สามารถแก้ไขได้)
    const immutable = Counter{ .value = 10 };
    // immutable.value = 20; // ERROR: cannot assign to constant

    // var = mutable (สามารถแก้ไขได้)
    var mutable = Counter{ .value = 10 };
    mutable.value = 20; // OK!
    mutable.value += 5; // OK!

    std.debug.print("Immutable: {d}\n", .{immutable.value});
    std.debug.print("Mutable: {d}\n", .{mutable.value});
}
```

### การส่ง Struct เป็น Parameter

```zig
const std = @import("std");

const Vector2 = struct {
    x: f64,
    y: f64,
};

// pass by value - function ได้รับ copy ของ struct
fn addVectors(a: Vector2, b: Vector2) Vector2 {
    return Vector2{
        .x = a.x + b.x,
        .y = a.y + b.y,
    };
}

// pass by pointer - function สามารถแก้ไข struct ต้นฉบับได้
fn scaleVector(v: *Vector2, factor: f64) void {
    v.x *= factor;
    v.y *= factor;
}

// pass by const pointer - อ่านอย่างเดียว
fn printVector(v: *const Vector2) void {
    std.debug.print("({d:.2}, {d:.2})\n", .{ v.x, v.y });
}

pub fn main() void {
    const v1 = Vector2{ .x = 1.0, .y = 2.0 };
    const v2 = Vector2{ .x = 3.0, .y = 4.0 };

    const sum = addVectors(v1, v2);
    std.debug.print("v1 + v2 = ", .{});
    printVector(&sum);

    var v3 = Vector2{ .x = 1.0, .y = 1.0 };
    scaleVector(&v3, 5.0);
    std.debug.print("v3 * 5 = ", .{});
    printVector(&v3);
}
```

**ผลลัพธ์:**
```
v1 + v2 = (4.00, 6.00)
v3 * 5 = (5.00, 5.00)
```

### Pointer Dereferencing

```zig
const std = @import("std");

const Box = struct {
    value: i32,
};

pub fn main() void {
    var box = Box{ .value = 42 };
    const ptr = &box;

    // เข้าถึง field ผ่าน pointer (Zig auto-dereference)
    std.debug.print("Value: {d}\n", .{ptr.value}); // เหมือน (*ptr).value

    // แก้ไขผ่าน pointer
    ptr.value = 100;
    std.debug.print("New value: {d}\n", .{box.value});
}
```

---

## Step 104: Methods ใน Struct (fn กับ self parameter)

### Methods คืออะไร?

Method คือ function ที่ถูกกำหนดไว้ภายใน struct ช่วยให้เราจัดกลุ่ม behavior (พฤติกรรม) ไว้กับข้อมูล เป็นหลักการ OOP พื้นฐาน

### Syntax ของ Method

```zig
const MyStruct = struct {
    field: u32,

    // Method ที่รับ self แบบ value (immutable copy)
    fn getValue(self: MyStruct) u32 {
        return self.field;
    }

    // Method ที่รับ self แบบ pointer (mutable)
    fn setValue(self: *MyStruct, value: u32) void {
        self.field = value;
    }

    // Method ที่รับ self แบบ const pointer (read-only)
    fn print(self: *const MyStruct) void {
        // ไม่สามารถแก้ไข self ได้
        _ = self;
    }
};
```

### ตัวอย่าง: Circle

```zig
const std = @import("std");

const Circle = struct {
    x: f64,
    y: f64,
    radius: f64,

    const pi = std.math.pi;

    // คำนวณพื้นที่
    fn area(self: Circle) f64 {
        return pi * self.radius * self.radius;
    }

    // คำนวณเส้นรอบวง
    fn circumference(self: Circle) f64 {
        return 2.0 * pi * self.radius;
    }

    // ตรวจสอบว่าจุดอยู่ภายในวงกลมหรือไม่
    fn containsPoint(self: Circle, px: f64, py: f64) bool {
        const dx = px - self.x;
        const dy = py - self.y;
        const dist_sq = dx * dx + dy * dy;
        return dist_sq <= self.radius * self.radius;
    }

    // เลื่อนวงกลม (แก้ไข self)
    fn move(self: *Circle, dx: f64, dy: f64) void {
        self.x += dx;
        self.y += dy;
    }

    // แสดงข้อมูล
    fn print(self: *const Circle) void {
        std.debug.print("Circle: center=({d:.2}, {d:.2}), r={d:.2}\n", .{
            self.x,
            self.y,
            self.radius,
        });
    }
};

pub fn main() void {
    var c = Circle{ .x = 0.0, .y = 0.0, .radius = 5.0 };

    c.print();
    std.debug.print("Area: {d:.4}\n", .{c.area()});
    std.debug.print("Circumference: {d:.4}\n", .{c.circumference()});
    std.debug.print("Contains (3,4): {}\n", .{c.containsPoint(3.0, 4.0)});
    std.debug.print("Contains (4,4): {}\n", .{c.containsPoint(4.0, 4.0)});

    c.move(10.0, 5.0);
    c.print();
}
```

**ผลลัพธ์:**
```
Circle: center=(0.00, 0.00), r=5.00
Area: 78.5398
Circumference: 31.4159
Contains (3,4): true
Contains (4,4): false
Circle: center=(10.00, 5.00), r=5.00
```

### ตัวอย่าง: BankAccount (OOP Pattern)

```zig
const std = @import("std");

const BankAccount = struct {
    owner: []const u8,
    balance: f64,
    account_number: u32,

    // Constructor pattern
    fn init(owner: []const u8, account_number: u32, initial_balance: f64) BankAccount {
        return BankAccount{
            .owner = owner,
            .balance = initial_balance,
            .account_number = account_number,
        };
    }

    // ฝากเงิน
    fn deposit(self: *BankAccount, amount: f64) void {
        if (amount <= 0) {
            std.debug.print("จำนวนเงินต้องมากกว่า 0\n", .{});
            return;
        }
        self.balance += amount;
        std.debug.print("ฝากเงิน {d:.2} บาท สำเร็จ\n", .{amount});
    }

    // ถอนเงิน
    fn withdraw(self: *BankAccount, amount: f64) bool {
        if (amount <= 0) {
            std.debug.print("จำนวนเงินต้องมากกว่า 0\n", .{});
            return false;
        }
        if (amount > self.balance) {
            std.debug.print("ยอดเงินไม่เพียงพอ\n", .{});
            return false;
        }
        self.balance -= amount;
        std.debug.print("ถอนเงิน {d:.2} บาท สำเร็จ\n", .{amount});
        return true;
    }

    // ดูยอดเงิน
    fn getBalance(self: *const BankAccount) f64 {
        return self.balance;
    }

    // แสดงข้อมูล
    fn printStatement(self: *const BankAccount) void {
        std.debug.print("=== บัญชีธนาคาร ===\n", .{});
        std.debug.print("เจ้าของ: {s}\n", .{self.owner});
        std.debug.print("เลขบัญชี: {d:0>10}\n", .{self.account_number});
        std.debug.print("ยอดคงเหลือ: {d:.2} บาท\n", .{self.balance});
        std.debug.print("==================\n", .{});
    }
};

pub fn main() void {
    var account = BankAccount.init("สมหมาย ดีใจ", 1234567890, 1000.0);

    account.printStatement();

    account.deposit(5000.0);
    account.deposit(2500.0);

    _ = account.withdraw(3000.0);
    _ = account.withdraw(10000.0); // ไม่พอ

    account.printStatement();
}
```

**ผลลัพธ์:**
```
=== บัญชีธนาคาร ===
เจ้าของ: สมหมาย ดีใจ
เลขบัญชี: 1234567890
ยอดคงเหลือ: 1000.00 บาท
==================
ฝากเงิน 5000.00 บาท สำเร็จ
ฝากเงิน 2500.00 บาท สำเร็จ
ถอนเงิน 3000.00 บาท สำเร็จ
ยอดเงินไม่เพียงพอ
=== บัญชีธนาคาร ===
เจ้าของ: สมหมาย ดีใจ
เลขบัญชี: 1234567890
ยอดคงเหลือ: 5500.00 บาท
==================
```

### Static Methods (ไม่มี self parameter)

Method ที่ไม่มี `self` parameter คือ static method หรือที่เรียกว่า "associated function"

```zig
const std = @import("std");

const Temperature = struct {
    celsius: f64,

    // Static method: ไม่มี self
    fn fromFahrenheit(f: f64) Temperature {
        return Temperature{ .celsius = (f - 32.0) * 5.0 / 9.0 };
    }

    fn fromKelvin(k: f64) Temperature {
        return Temperature{ .celsius = k - 273.15 };
    }

    // Instance method: มี self
    fn toFahrenheit(self: Temperature) f64 {
        return self.celsius * 9.0 / 5.0 + 32.0;
    }

    fn toKelvin(self: Temperature) f64 {
        return self.celsius + 273.15;
    }

    fn print(self: *const Temperature) void {
        std.debug.print("{d:.2}°C = {d:.2}°F = {d:.2}K\n", .{
            self.celsius,
            self.toFahrenheit(),
            self.toKelvin(),
        });
    }
};

pub fn main() void {
    const boiling = Temperature{ .celsius = 100.0 };
    boiling.print();

    const body_temp = Temperature.fromFahrenheit(98.6);
    body_temp.print();

    const absolute_zero = Temperature.fromKelvin(0.0);
    absolute_zero.print();
}
```

**ผลลัพธ์:**
```
100.00°C = 212.00°F = 373.15K
37.00°C = 98.60°F = 310.15K
-273.15°C = -459.67°F = 0.00K
```

---

## Step 105: pub Methods และ Encapsulation

### Visibility ใน Zig

ใน Zig การ encapsulate ทำได้ผ่าน `pub` keyword และ module system

- **ไม่มี `pub`**: เข้าถึงได้เฉพาะภายใน file เดียวกัน (private)
- **มี `pub`**: เข้าถึงได้จากภายนอก (public)

### ตัวอย่าง: File เดียว

```zig
const std = @import("std");

const Temperature2 = struct {
    // private field (ตามธรรมเนียม ใช้ underscore นำหน้า)
    _celsius: f64,

    // public constructor
    pub fn init(celsius: f64) Temperature2 {
        return Temperature2{ ._celsius = celsius };
    }

    // public getter
    pub fn getCelsius(self: Temperature2) f64 {
        return self._celsius;
    }

    // public setter พร้อม validation
    pub fn setCelsius(self: *Temperature2, celsius: f64) !void {
        if (celsius < -273.15) {
            return error.BelowAbsoluteZero;
        }
        self._celsius = celsius;
    }

    // public method
    pub fn toFahrenheit(self: Temperature2) f64 {
        return self._celsius * 9.0 / 5.0 + 32.0;
    }

    // private helper method (ไม่ต้องการ pub)
    fn validate(celsius: f64) bool {
        return celsius >= -273.15;
    }
};

pub fn main() !void {
    var temp = Temperature2.init(25.0);
    std.debug.print("Celsius: {d:.2}\n", .{temp.getCelsius()});
    std.debug.print("Fahrenheit: {d:.2}\n", .{temp.toFahrenheit()});

    try temp.setCelsius(100.0);
    std.debug.print("New temp: {d:.2}°C\n", .{temp.getCelsius()});

    // ลองตั้งค่าต่ำกว่า absolute zero
    temp.setCelsius(-300.0) catch |err| {
        std.debug.print("Error: {s}\n", .{@errorName(err)});
    };
}
```

### ตัวอย่าง: Stack with Encapsulation

```zig
const std = @import("std");

// Simple stack สำหรับ integers
const IntStack = struct {
    data: [100]i32,
    size: usize,

    pub fn init() IntStack {
        return IntStack{
            .data = undefined,
            .size = 0,
        };
    }

    pub fn push(self: *IntStack, value: i32) !void {
        if (self.size >= self.data.len) {
            return error.StackOverflow;
        }
        self.data[self.size] = value;
        self.size += 1;
    }

    pub fn pop(self: *IntStack) !i32 {
        if (self.size == 0) {
            return error.StackUnderflow;
        }
        self.size -= 1;
        return self.data[self.size];
    }

    pub fn peek(self: *const IntStack) !i32 {
        if (self.size == 0) {
            return error.EmptyStack;
        }
        return self.data[self.size - 1];
    }

    pub fn isEmpty(self: *const IntStack) bool {
        return self.size == 0;
    }

    pub fn count(self: *const IntStack) usize {
        return self.size;
    }

    pub fn print(self: *const IntStack) void {
        std.debug.print("Stack [{d}]: ", .{self.size});
        if (self.size == 0) {
            std.debug.print("(empty)\n", .{});
            return;
        }
        var i: usize = 0;
        while (i < self.size) : (i += 1) {
            if (i > 0) std.debug.print(", ", .{});
            std.debug.print("{d}", .{self.data[i]});
        }
        std.debug.print("\n", .{});
    }
};

pub fn main() !void {
    var stack = IntStack.init();

    try stack.push(10);
    try stack.push(20);
    try stack.push(30);
    stack.print();

    const top = try stack.peek();
    std.debug.print("Top: {d}\n", .{top});

    const popped = try stack.pop();
    std.debug.print("Popped: {d}\n", .{popped});
    stack.print();

    std.debug.print("Count: {d}\n", .{stack.count()});
    std.debug.print("Empty: {}\n", .{stack.isEmpty()});
}
```

**ผลลัพธ์:**
```
Stack [3]: 10, 20, 30
Top: 30
Popped: 30
Stack [2]: 10, 20
Count: 2
Empty: false
```

### Constants ใน Struct

```zig
const std = @import("std");

const MathConstants = struct {
    // Constants สามารถอยู่ใน struct namespace ได้
    pub const PI = 3.14159265358979323846;
    pub const E = 2.71828182845904523536;
    pub const GOLDEN_RATIO = 1.61803398874989484820;

    value: f64,

    pub fn init(v: f64) MathConstants {
        return MathConstants{ .value = v };
    }

    pub fn circleArea(self: MathConstants) f64 {
        return PI * self.value * self.value;
    }
};

pub fn main() void {
    std.debug.print("PI = {d:.10}\n", .{MathConstants.PI});
    std.debug.print("E = {d:.10}\n", .{MathConstants.E});

    const r = MathConstants.init(5.0);
    std.debug.print("Area of circle with r=5: {d:.4}\n", .{r.circleArea()});
}
```

---

## Step 106: Struct Initialization - Named Fields

### Named Fields Initialization

Zig บังคับใช้ named fields เสมอ (ยกเว้น anonymous struct) ซึ่งทำให้โค้ดอ่านง่ายและลด bug

```zig
const std = @import("std");

const Pixel = struct {
    x: u32,
    y: u32,
    color: u32,
    alpha: u8 = 255,
};

pub fn main() void {
    // Named fields: ชัดเจน อ่านง่าย
    const p1 = Pixel{
        .x = 100,
        .y = 200,
        .color = 0xFF0000,
    };

    // สามารถสลับลำดับได้
    const p2 = Pixel{
        .color = 0x00FF00,
        .alpha = 128,
        .x = 50,
        .y = 75,
    };

    std.debug.print("p1: ({d},{d}) color=#{x:0>6} alpha={d}\n", .{
        p1.x, p1.y, p1.color, p1.alpha,
    });
    std.debug.print("p2: ({d},{d}) color=#{x:0>6} alpha={d}\n", .{
        p2.x, p2.y, p2.color, p2.alpha,
    });
}
```

### Update Syntax

Zig มีวิธีสร้าง struct จาก struct เดิมพร้อมแก้ไขบาง field

```zig
const std = @import("std");

const WindowSettings = struct {
    width: u32 = 1280,
    height: u32 = 720,
    fullscreen: bool = false,
    vsync: bool = true,
    title: []const u8 = "My App",
};

pub fn main() void {
    const default_settings = WindowSettings{};

    // สร้าง setting ใหม่จาก default โดยแก้ไขแค่บางส่วน
    // (Zig ไม่มี spread operator แต่เราสร้าง helper function แทน)
    const fullscreen_settings = WindowSettings{
        .width = default_settings.width,
        .height = default_settings.height,
        .fullscreen = true, // เปลี่ยนเป็น fullscreen
        .vsync = default_settings.vsync,
        .title = default_settings.title,
    };

    std.debug.print("Default: {}x{} fullscreen={}\n", .{
        default_settings.width,
        default_settings.height,
        default_settings.fullscreen,
    });

    std.debug.print("Fullscreen: {}x{} fullscreen={}\n", .{
        fullscreen_settings.width,
        fullscreen_settings.height,
        fullscreen_settings.fullscreen,
    });
}
```

### Builder Pattern ใน Zig

```zig
const std = @import("std");

const HttpRequest = struct {
    method: []const u8,
    url: []const u8,
    body: ?[]const u8,
    content_type: []const u8,
    timeout_ms: u32,
    follow_redirects: bool,

    // Builder
    const Builder = struct {
        method: []const u8 = "GET",
        url: []const u8,
        body: ?[]const u8 = null,
        content_type: []const u8 = "application/json",
        timeout_ms: u32 = 30000,
        follow_redirects: bool = true,

        fn init(url: []const u8) Builder {
            return Builder{ .url = url };
        }

        fn withMethod(self: Builder, method: []const u8) Builder {
            var b = self;
            b.method = method;
            return b;
        }

        fn withBody(self: Builder, body: []const u8) Builder {
            var b = self;
            b.body = body;
            return b;
        }

        fn withTimeout(self: Builder, ms: u32) Builder {
            var b = self;
            b.timeout_ms = ms;
            return b;
        }

        fn build(self: Builder) HttpRequest {
            return HttpRequest{
                .method = self.method,
                .url = self.url,
                .body = self.body,
                .content_type = self.content_type,
                .timeout_ms = self.timeout_ms,
                .follow_redirects = self.follow_redirects,
            };
        }
    };

    fn print(self: *const HttpRequest) void {
        std.debug.print("{s} {s}\n", .{ self.method, self.url });
        std.debug.print("  Timeout: {d}ms\n", .{self.timeout_ms});
        if (self.body) |b| {
            std.debug.print("  Body: {s}\n", .{b});
        }
    }
};

pub fn main() void {
    const get_request = HttpRequest.Builder.init("https://api.example.com/users")
        .withTimeout(5000)
        .build();

    const post_request = HttpRequest.Builder.init("https://api.example.com/users")
        .withMethod("POST")
        .withBody("{\"name\": \"สมชาย\"}")
        .withTimeout(10000)
        .build();

    std.debug.print("=== GET Request ===\n", .{});
    get_request.print();

    std.debug.print("=== POST Request ===\n", .{});
    post_request.print();
}
```

**ผลลัพธ์:**
```
=== GET Request ===
GET https://api.example.com/users
  Timeout: 5000ms
=== POST Request ===
POST https://api.example.com/users
  Timeout: 10000ms
  Body: {"name": "สมชาย"}
```

---

## Step 107: Nested Structs

### Struct ซ้อน Struct

Struct สามารถมี field ที่เป็น struct อื่นได้ เรียกว่า nested structs

```zig
const std = @import("std");

const Point2D = struct {
    x: f64,
    y: f64,

    fn distance(self: Point2D, other: Point2D) f64 {
        const dx = self.x - other.x;
        const dy = self.y - other.y;
        return @sqrt(dx * dx + dy * dy);
    }
};

const Line = struct {
    start: Point2D,
    end: Point2D,

    fn length(self: Line) f64 {
        return self.start.distance(self.end);
    }

    fn midpoint(self: Line) Point2D {
        return Point2D{
            .x = (self.start.x + self.end.x) / 2.0,
            .y = (self.start.y + self.end.y) / 2.0,
        };
    }
};

pub fn main() void {
    const line = Line{
        .start = Point2D{ .x = 0.0, .y = 0.0 },
        .end = Point2D{ .x = 3.0, .y = 4.0 },
    };

    std.debug.print("Length: {d:.2}\n", .{line.length()});

    const mid = line.midpoint();
    std.debug.print("Midpoint: ({d:.2}, {d:.2})\n", .{ mid.x, mid.y });
}
```

### ตัวอย่าง: องค์กรบริษัท

```zig
const std = @import("std");

const Address = struct {
    street: []const u8,
    city: []const u8,
    country: []const u8,
    postal_code: []const u8,

    fn print(self: *const Address) void {
        std.debug.print("{s}, {s}, {s} {s}\n", .{
            self.street,
            self.city,
            self.country,
            self.postal_code,
        });
    }
};

const ContactInfo = struct {
    email: []const u8,
    phone: []const u8,
    address: Address,

    fn print(self: *const ContactInfo) void {
        std.debug.print("Email: {s}\n", .{self.email});
        std.debug.print("Phone: {s}\n", .{self.phone});
        std.debug.print("Address: ", .{});
        self.address.print();
    }
};

const Employee = struct {
    id: u32,
    name: []const u8,
    department: []const u8,
    salary: f64,
    contact: ContactInfo,

    fn print(self: *const Employee) void {
        std.debug.print("=== พนักงาน ID: {d} ===\n", .{self.id});
        std.debug.print("ชื่อ: {s}\n", .{self.name});
        std.debug.print("แผนก: {s}\n", .{self.department});
        std.debug.print("เงินเดือน: {d:.0} บาท\n", .{self.salary});
        std.debug.print("ข้อมูลติดต่อ:\n", .{});
        self.contact.print();
    }
};

pub fn main() void {
    const emp = Employee{
        .id = 1001,
        .name = "วิชัย มีสุข",
        .department = "วิศวกรรมซอฟต์แวร์",
        .salary = 75000.0,
        .contact = ContactInfo{
            .email = "wichai@company.com",
            .phone = "082-123-4567",
            .address = Address{
                .street = "123 ถนนสุขุมวิท",
                .city = "กรุงเทพมหานคร",
                .country = "ประเทศไทย",
                .postal_code = "10110",
            },
        },
    };

    emp.print();
}
```

**ผลลัพธ์:**
```
=== พนักงาน ID: 1001 ===
ชื่อ: วิชัย มีสุข
แผนก: วิศวกรรมซอฟต์แวร์
เงินเดือน: 75000 บาท
ข้อมูลติดต่อ:
Email: wichai@company.com
Phone: 082-123-4567
Address: 123 ถนนสุขุมวิท, กรุงเทพมหานคร, ประเทศไทย 10110
```

### Nested Struct Methods เรียก Outer Struct

```zig
const std = @import("std");

const Wheel = struct {
    diameter_cm: f64,
    pressure_psi: f64,

    fn isInflated(self: Wheel) bool {
        return self.pressure_psi >= 30.0 and self.pressure_psi <= 45.0;
    }
};

const Car = struct {
    make: []const u8,
    model: []const u8,
    year: u32,
    front_left: Wheel,
    front_right: Wheel,
    rear_left: Wheel,
    rear_right: Wheel,

    fn allWheelsInflated(self: *const Car) bool {
        return self.front_left.isInflated() and
            self.front_right.isInflated() and
            self.rear_left.isInflated() and
            self.rear_right.isInflated();
    }

    fn printStatus(self: *const Car) void {
        std.debug.print("{d} {s} {s}\n", .{ self.year, self.make, self.model });
        std.debug.print("FL: {d:.1} psi | FR: {d:.1} psi\n", .{
            self.front_left.pressure_psi,
            self.front_right.pressure_psi,
        });
        std.debug.print("RL: {d:.1} psi | RR: {d:.1} psi\n", .{
            self.rear_left.pressure_psi,
            self.rear_right.pressure_psi,
        });
        std.debug.print("Ready to drive: {}\n", .{self.allWheelsInflated()});
    }
};

pub fn main() void {
    const car = Car{
        .make = "Toyota",
        .model = "Camry",
        .year = 2024,
        .front_left = Wheel{ .diameter_cm = 50.8, .pressure_psi = 35.0 },
        .front_right = Wheel{ .diameter_cm = 50.8, .pressure_psi = 35.0 },
        .rear_left = Wheel{ .diameter_cm = 50.8, .pressure_psi = 35.0 },
        .rear_right = Wheel{ .diameter_cm = 50.8, .pressure_psi = 20.0 }, // ยางแฟบ!
    };

    car.printStatus();
}
```

---

## Step 108: Packed Structs (Bit-level Packing)

### Memory Layout ปกติ

ปกติ CPU จะ align ข้อมูลในหน่วยความจำตาม natural alignment:

```zig
const std = @import("std");

const NormalStruct = struct {
    a: u8,   // 1 byte + 3 bytes padding
    b: u32,  // 4 bytes
    c: u8,   // 1 byte + 7 bytes padding (เพราะต้องการ align 8)
    d: u64,  // 8 bytes
};
// Total: อาจมากถึง 24 bytes เนื่องจาก alignment padding

pub fn main() void {
    std.debug.print("NormalStruct size: {d} bytes\n", .{@sizeOf(NormalStruct)});
    std.debug.print("NormalStruct align: {d} bytes\n", .{@alignOf(NormalStruct)});
}
```

### Packed Struct

`packed struct` บีบอัดข้อมูลให้แน่นที่สุดโดยไม่มี padding:

```zig
const std = @import("std");

const NormalFlags = struct {
    flag_a: bool,
    flag_b: bool,
    flag_c: bool,
    flag_d: bool,
};
// อาจใช้ถึง 4 bytes (แต่ละ bool = 1 byte)

const PackedFlags = packed struct {
    flag_a: bool,
    flag_b: bool,
    flag_c: bool,
    flag_d: bool,
    // padding bits เพื่อให้ครบ 8 bits
    _padding: u4 = 0,
};
// ใช้แค่ 1 byte!

pub fn main() void {
    std.debug.print("NormalFlags size: {d} bytes\n", .{@sizeOf(NormalFlags)});
    std.debug.print("PackedFlags size: {d} bytes\n", .{@sizeOf(PackedFlags)});

    var flags = PackedFlags{
        .flag_a = true,
        .flag_b = false,
        .flag_c = true,
        .flag_d = true,
    };

    std.debug.print("a={} b={} c={} d={}\n", .{
        flags.flag_a,
        flags.flag_b,
        flags.flag_c,
        flags.flag_d,
    });

    flags.flag_b = true;
    std.debug.print("After setting b: b={}\n", .{flags.flag_b});
}
```

**ผลลัพธ์:**
```
NormalFlags size: 4 bytes
PackedFlags size: 1 bytes
a=true b=false c=true d=true
After setting b: b=true
```

### Bit Fields ใน Packed Struct

```zig
const std = @import("std");

// IPv4 Header (simplified)
const IPv4Header = packed struct {
    version: u4,        // 4 bits
    ihl: u4,            // 4 bits (Internet Header Length)
    dscp: u6,           // 6 bits (Differentiated Services)
    ecn: u2,            // 2 bits (Explicit Congestion Notification)
    total_length: u16,  // 16 bits
    identification: u16,// 16 bits
    flags: u3,          // 3 bits
    fragment_offset: u13, // 13 bits
    ttl: u8,            // 8 bits
    protocol: u8,       // 8 bits
    checksum: u16,      // 16 bits
    src_addr: u32,      // 32 bits
    dst_addr: u32,      // 32 bits
};

pub fn main() void {
    std.debug.print("IPv4Header size: {d} bytes (expected: 20)\n", .{
        @sizeOf(IPv4Header),
    });

    const header = IPv4Header{
        .version = 4,
        .ihl = 5,
        .dscp = 0,
        .ecn = 0,
        .total_length = 60,
        .identification = 0x1234,
        .flags = 2, // Don't Fragment
        .fragment_offset = 0,
        .ttl = 64,
        .protocol = 6, // TCP
        .checksum = 0,
        .src_addr = 0xC0A80001, // 192.168.0.1
        .dst_addr = 0xC0A80002, // 192.168.0.2
    };

    std.debug.print("Version: {d}\n", .{header.version});
    std.debug.print("IHL: {d}\n", .{header.ihl});
    std.debug.print("TTL: {d}\n", .{header.ttl});
    std.debug.print("Protocol: {d} (TCP=6)\n", .{header.protocol});
}
```

### Color Packing

```zig
const std = @import("std");

// ปกติ: 4 bytes ต่อสี
const ColorNormal = struct {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
};

// Packed: สามารถแปลงเป็น u32 ได้ง่าย
const ColorPacked = packed struct {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
};

pub fn main() void {
    std.debug.print("ColorNormal: {d} bytes\n", .{@sizeOf(ColorNormal)});
    std.debug.print("ColorPacked: {d} bytes\n", .{@sizeOf(ColorPacked)});

    const red = ColorPacked{ .r = 255, .g = 0, .b = 0, .a = 255 };
    std.debug.print("Red: r={d} g={d} b={d} a={d}\n", .{
        red.r, red.g, red.b, red.a,
    });

    // แปลง packed struct เป็น u32
    const as_u32 = @as(u32, @bitCast(red));
    std.debug.print("As u32: 0x{x:0>8}\n", .{as_u32});
}
```

### ข้อจำกัดของ Packed Struct

```zig
// Packed structs มีข้อจำกัด:
// 1. fields ต้องมีขนาดที่กำหนดชัดเจนใน bits (u1, u2, ..., u64, f32, f64, bool, enum)
// 2. ไม่รองรับ slices หรือ pointers
// 3. ไม่รองรับ nested normal structs

const ValidPacked = packed struct {
    x: u16,
    y: u16,
    z: u32,
};

// ประโยชน์:
// - ลดการใช้หน่วยความจำ
// - สื่อสารกับ hardware registers
// - Network protocols
// - File formats
```

---

## Step 109: Extern Structs (C-compatible Layout)

### ทำไมต้องมี Extern Struct?

เมื่อเราต้องการทำงานร่วมกับ C libraries หรือ system APIs เราต้องการให้ struct มี memory layout ที่เหมือนกับที่ C กำหนด

### ความแตกต่างระหว่าง Normal, Packed, Extern

```zig
const std = @import("std");

// Normal struct: Zig อาจ reorder fields เพื่อประสิทธิภาพ
const NormalPoint = struct {
    x: f32,
    y: f32,
    z: f32,
};

// Packed struct: ไม่มี padding, bits compact
const PackedPoint = packed struct {
    x: f32,
    y: f32,
    z: f32,
};

// Extern struct: C ABI compatible, fields ไม่ถูก reorder, มี C-style alignment
const ExternPoint = extern struct {
    x: f32,
    y: f32,
    z: f32,
};

pub fn main() void {
    std.debug.print("NormalPoint size: {d}\n", .{@sizeOf(NormalPoint)});
    std.debug.print("PackedPoint size: {d}\n", .{@sizeOf(PackedPoint)});
    std.debug.print("ExternPoint size: {d}\n", .{@sizeOf(ExternPoint)});

    std.debug.print("\nOffsets ใน ExternPoint:\n", .{});
    std.debug.print("  x: byte {d}\n", .{@offsetOf(ExternPoint, "x")});
    std.debug.print("  y: byte {d}\n", .{@offsetOf(ExternPoint, "y")});
    std.debug.print("  z: byte {d}\n", .{@offsetOf(ExternPoint, "z")});
}
```

### ตัวอย่าง: C Library Interop

```zig
const std = @import("std");

// ตัวอย่าง struct ที่ compatible กับ C library

// เหมือน timespec ใน C
const TimeSpec = extern struct {
    tv_sec: i64,   // seconds
    tv_nsec: i64,  // nanoseconds
};

// เหมือน sockaddr_in ใน C (สำหรับ networking)
const SockAddrIn = extern struct {
    sin_family: u16,
    sin_port: u16,
    sin_addr: u32,
    sin_zero: [8]u8,
};

pub fn main() void {
    std.debug.print("TimeSpec size: {d} (C expects: 16)\n", .{@sizeOf(TimeSpec)});
    std.debug.print("SockAddrIn size: {d} (C expects: 16)\n", .{@sizeOf(SockAddrIn)});

    const ts = TimeSpec{
        .tv_sec = 1700000000,
        .tv_nsec = 500000000,
    };
    std.debug.print("Time: {d}.{d:0>9}s\n", .{ ts.tv_sec, ts.tv_nsec });
}
```

### Memory Layout Comparison

```zig
const std = @import("std");

// ลอง struct ที่มี mixed types
const MixedNormal = struct {
    a: u8,
    b: u32,
    c: u8,
    d: u64,
};

const MixedExtern = extern struct {
    a: u8,
    b: u32,
    c: u8,
    d: u64,
};

pub fn main() void {
    std.debug.print("=== Memory Layout Comparison ===\n", .{});
    std.debug.print("\nNormal struct:\n", .{});
    std.debug.print("  Size: {d} bytes\n", .{@sizeOf(MixedNormal)});
    std.debug.print("  Align: {d} bytes\n", .{@alignOf(MixedNormal)});

    std.debug.print("\nExtern struct:\n", .{});
    std.debug.print("  Size: {d} bytes\n", .{@sizeOf(MixedExtern)});
    std.debug.print("  Align: {d} bytes\n", .{@alignOf(MixedExtern)});
    std.debug.print("  Offset a: {d}\n", .{@offsetOf(MixedExtern, "a")});
    std.debug.print("  Offset b: {d}\n", .{@offsetOf(MixedExtern, "b")});
    std.debug.print("  Offset c: {d}\n", .{@offsetOf(MixedExtern, "c")});
    std.debug.print("  Offset d: {d}\n", .{@offsetOf(MixedExtern, "d")});
}
```

### ตาราง Memory Layout สรุป

```
struct type  | field reorder | padding | C compatible
-------------|---------------|---------|-------------
normal       | อาจมี         | มี      | ไม่รับประกัน
packed       | ไม่มี         | ไม่มี   | ไม่
extern       | ไม่มี         | มี (C)  | ใช่
```

### Extern Struct กับ @cImport

```zig
// ในโลกจริง เราใช้ extern struct กับ C headers
// const c = @cImport({
//     @cInclude("stdio.h");
// });
//
// Zig จะสร้าง extern struct ให้อัตโนมัติสำหรับ C structs

// ตัวอย่าง manual extern struct
const FILE_like = extern struct {
    _IO_read_ptr: ?*u8 = null,
    _IO_read_end: ?*u8 = null,
    _IO_buf_base: ?*u8 = null,
    // ... (simplified)
};

// เมื่อใช้กับ callconv(.C)
fn some_c_callback(data: *ExternPoint, count: u32) callconv(.C) void {
    _ = data;
    _ = count;
}

const ExternPoint = extern struct {
    x: f32,
    y: f32,
    z: f32,
};
```

---

## Step 110: Anonymous Structs และ Tuples

### Anonymous Structs

Anonymous struct คือ struct ที่ไม่มีชื่อ type ใช้เพื่อส่งข้อมูลกลุ่มชั่วคราว

```zig
const std = @import("std");

pub fn main() void {
    // Anonymous struct literal
    const point = .{ .x = 1.0, .y = 2.0 };
    std.debug.print("x={d}, y={d}\n", .{ point.x, point.y });

    // ชนิดถูก infer โดย compiler
    // typeof(point) คือ struct { x: comptime_float, y: comptime_float }
    std.debug.print("Type: {s}\n", .{@typeName(@TypeOf(point))});
}
```

### Anonymous Struct เป็น Return Value

```zig
const std = @import("std");

// Function ที่ return anonymous struct (multi-return pattern)
fn minMax(values: []const i32) struct { min: i32, max: i32 } {
    var min = values[0];
    var max = values[0];
    for (values[1..]) |v| {
        if (v < min) min = v;
        if (v > max) max = v;
    }
    return .{ .min = min, .max = max };
}

fn divMod(a: i32, b: i32) struct { quotient: i32, remainder: i32 } {
    return .{
        .quotient = @divTrunc(a, b),
        .remainder = @rem(a, b),
    };
}

pub fn main() void {
    const numbers = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };
    const result = minMax(&numbers);
    std.debug.print("Min: {d}, Max: {d}\n", .{ result.min, result.max });

    const dm = divMod(17, 5);
    std.debug.print("17 / 5 = {d} remainder {d}\n", .{
        dm.quotient,
        dm.remainder,
    });
}
```

**ผลลัพธ์:**
```
Min: 1, Max: 9
17 / 5 = 3 remainder 2
```

### Tuples

Tuple ใน Zig คือ anonymous struct ที่ field เป็นตัวเลข (0, 1, 2, ...)

```zig
const std = @import("std");

pub fn main() void {
    // Tuple syntax: .{ value1, value2, ... }
    const t = .{ 1, "hello", 3.14, true };

    // เข้าถึงด้วย index
    std.debug.print("t[0] = {d}\n", .{t[0]});
    std.debug.print("t[1] = {s}\n", .{t[1]});
    std.debug.print("t[2] = {d}\n", .{t[2]});
    std.debug.print("t[3] = {}\n", .{t[3]});

    // ประเภทของ tuple
    std.debug.print("Type: {s}\n", .{@typeName(@TypeOf(t))});

    // len ของ tuple
    std.debug.print("Length: {d}\n", .{t.len});
}
```

### Tuple สำหรับ Multiple Return Values

```zig
const std = @import("std");

// Return หลายค่าด้วย tuple
fn swap(a: i32, b: i32) struct { i32, i32 } {
    return .{ b, a };
}

fn rgb_to_hsv(r: f32, g: f32, b: f32) struct { f32, f32, f32 } {
    const max = @max(r, @max(g, b));
    const min = @min(r, @min(g, b));
    const delta = max - min;

    const v = max;
    const s = if (max == 0) 0.0 else delta / max;
    const h = if (delta == 0) blk: {
        break :blk 0.0;
    } else if (max == r) blk: {
        break :blk 60.0 * @mod((g - b) / delta, 6.0);
    } else if (max == g) blk: {
        break :blk 60.0 * ((b - r) / delta + 2.0);
    } else blk: {
        break :blk 60.0 * ((r - g) / delta + 4.0);
    };

    return .{ h, s, v };
}

pub fn main() void {
    const swapped = swap(10, 20);
    std.debug.print("swap(10, 20) = ({d}, {d})\n", .{ swapped[0], swapped[1] });

    const hsv = rgb_to_hsv(1.0, 0.0, 0.0); // Red
    std.debug.print("Red HSV: h={d:.1} s={d:.2} v={d:.2}\n", .{
        hsv[0],
        hsv[1],
        hsv[2],
    });
}
```

### Anonymous Struct เป็น Function Parameter

```zig
const std = @import("std");

// รับ anonymous struct เป็น parameter
fn formatName(name: struct {
    first: []const u8,
    last: []const u8,
    middle: ?[]const u8 = null,
}) void {
    if (name.middle) |m| {
        std.debug.print("{s} {s} {s}\n", .{ name.first, m, name.last });
    } else {
        std.debug.print("{s} {s}\n", .{ name.first, name.last });
    }
}

// Configuration options pattern
fn connect(options: struct {
    host: []const u8,
    port: u16 = 80,
    tls: bool = false,
    timeout: u32 = 30,
}) void {
    const scheme = if (options.tls) "https" else "http";
    std.debug.print("Connecting to {s}://{s}:{d} (timeout: {d}s)\n", .{
        scheme,
        options.host,
        options.port,
        options.timeout,
    });
}

pub fn main() void {
    formatName(.{
        .first = "John",
        .last = "Doe",
    });

    formatName(.{
        .first = "สมชาย",
        .middle = "รัตน์",
        .last = "ใจดี",
    });

    connect(.{
        .host = "example.com",
        .port = 443,
        .tls = true,
    });

    connect(.{
        .host = "api.service.com",
        .timeout = 60,
    });
}
```

**ผลลัพธ์:**
```
John Doe
สมชาย รัตน์ ใจดี
Connecting to https://example.com:443 (timeout: 30s)
Connecting to http://api.service.com:80 (timeout: 60s)
```

### Comptime Anonymous Struct

```zig
const std = @import("std");

// ใช้ comptime สำหรับ generic-like behavior
fn createTypedPair(comptime T1: type, comptime T2: type) type {
    return struct {
        first: T1,
        second: T2,

        fn swap(self: @This()) struct { T2, T1 } {
            return .{ self.second, self.first };
        }
    };
}

pub fn main() void {
    const StringIntPair = createTypedPair([]const u8, i32);

    const pair = StringIntPair{
        .first = "answer",
        .second = 42,
    };

    std.debug.print("first={s}, second={d}\n", .{ pair.first, pair.second });

    const swapped = pair.swap();
    std.debug.print("swapped: {d}, {s}\n", .{ swapped[0], swapped[1] });
}
```

---

## OOP Patterns ใน Zig ด้วย Structs

### Pattern 1: Constructor/Destructor

```zig
const std = @import("std");

const ArrayList = struct {
    items: []i32,
    len: usize,
    allocator: std.mem.Allocator,

    // Constructor
    pub fn init(allocator: std.mem.Allocator) !ArrayList {
        const items = try allocator.alloc(i32, 16);
        return ArrayList{
            .items = items,
            .len = 0,
            .allocator = allocator,
        };
    }

    // Destructor
    pub fn deinit(self: *ArrayList) void {
        self.allocator.free(self.items);
    }

    pub fn append(self: *ArrayList, value: i32) !void {
        if (self.len >= self.items.len) {
            const new_cap = self.items.len * 2;
            self.items = try self.allocator.realloc(self.items, new_cap);
        }
        self.items[self.len] = value;
        self.len += 1;
    }

    pub fn get(self: *const ArrayList, index: usize) !i32 {
        if (index >= self.len) return error.IndexOutOfBounds;
        return self.items[index];
    }

    pub fn print(self: *const ArrayList) void {
        std.debug.print("[", .{});
        for (0..self.len) |i| {
            if (i > 0) std.debug.print(", ", .{});
            std.debug.print("{d}", .{self.items[i]});
        }
        std.debug.print("]\n", .{});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = try ArrayList.init(allocator);
    defer list.deinit();

    try list.append(10);
    try list.append(20);
    try list.append(30);

    list.print();

    const val = try list.get(1);
    std.debug.print("list[1] = {d}\n", .{val});
}
```

### Pattern 2: Inheritance-like ด้วย Embedding

Zig ไม่มี inheritance แต่เราสามารถใช้ composition แทน:

```zig
const std = @import("std");

// "Base" struct
const Shape = struct {
    color: u32,
    visible: bool,

    fn getColor(self: *const Shape) u32 {
        return self.color;
    }

    fn isVisible(self: *const Shape) bool {
        return self.visible;
    }
};

// "Derived" struct ที่ embed Shape
const ColoredCircle = struct {
    shape: Shape,    // embed Shape
    x: f64,
    y: f64,
    radius: f64,

    fn init(x: f64, y: f64, r: f64, color: u32) ColoredCircle {
        return ColoredCircle{
            .shape = Shape{ .color = color, .visible = true },
            .x = x,
            .y = y,
            .radius = r,
        };
    }

    fn area(self: *const ColoredCircle) f64 {
        return std.math.pi * self.radius * self.radius;
    }

    // Delegate to shape
    fn getColor(self: *const ColoredCircle) u32 {
        return self.shape.getColor();
    }

    fn print(self: *const ColoredCircle) void {
        std.debug.print("ColoredCircle at ({d:.1},{d:.1}) r={d:.1} color=#{x:0>6}\n", .{
            self.x, self.y, self.radius, self.getColor(),
        });
    }
};

pub fn main() void {
    const c = ColoredCircle.init(5.0, 5.0, 3.0, 0xFF5733);
    c.print();
    std.debug.print("Area: {d:.4}\n", .{c.area()});
    std.debug.print("Visible: {}\n", .{c.shape.isVisible()});
}
```

### Pattern 3: Interface-like ด้วย Function Pointers

```zig
const std = @import("std");

// "Interface" ด้วย struct of function pointers
const Drawable = struct {
    ptr: *anyopaque,
    drawFn: *const fn (*anyopaque) void,
    areaFn: *const fn (*anyopaque) f64,

    fn draw(self: Drawable) void {
        self.drawFn(self.ptr);
    }

    fn area(self: Drawable) f64 {
        return self.areaFn(self.ptr);
    }
};

const Rect = struct {
    w: f64,
    h: f64,

    fn draw(ptr: *anyopaque) void {
        const self: *Rect = @ptrCast(@alignCast(ptr));
        std.debug.print("Drawing rectangle {d}x{d}\n", .{ self.w, self.h });
    }

    fn area(ptr: *anyopaque) f64 {
        const self: *Rect = @ptrCast(@alignCast(ptr));
        return self.w * self.h;
    }

    fn drawable(self: *Rect) Drawable {
        return Drawable{
            .ptr = self,
            .drawFn = draw,
            .areaFn = area,
        };
    }
};

const Circ = struct {
    r: f64,

    fn draw(ptr: *anyopaque) void {
        const self: *Circ = @ptrCast(@alignCast(ptr));
        std.debug.print("Drawing circle r={d:.1}\n", .{self.r});
    }

    fn area(ptr: *anyopaque) f64 {
        const self: *Circ = @ptrCast(@alignCast(ptr));
        return std.math.pi * self.r * self.r;
    }

    fn drawable(self: *Circ) Drawable {
        return Drawable{
            .ptr = self,
            .drawFn = draw,
            .areaFn = area,
        };
    }
};

pub fn main() void {
    var rect = Rect{ .w = 10.0, .h = 5.0 };
    var circle = Circ{ .r = 3.0 };

    const shapes = [_]Drawable{
        rect.drawable(),
        circle.drawable(),
    };

    for (shapes) |shape| {
        shape.draw();
        std.debug.print("  Area: {d:.4}\n", .{shape.area()});
    }
}
```

**ผลลัพธ์:**
```
Drawing rectangle 10x5
  Area: 50.0000
Drawing circle r=3.0
  Area: 28.2743
```

---

## ตัวอย่างโลกจริง: Game Entity System

```zig
const std = @import("std");

const Vec2 = struct {
    x: f32,
    y: f32,

    fn add(self: Vec2, other: Vec2) Vec2 {
        return Vec2{ .x = self.x + other.x, .y = self.y + other.y };
    }

    fn scale(self: Vec2, factor: f32) Vec2 {
        return Vec2{ .x = self.x * factor, .y = self.y * factor };
    }

    fn length(self: Vec2) f32 {
        return @sqrt(self.x * self.x + self.y * self.y);
    }
};

const Transform = struct {
    position: Vec2,
    velocity: Vec2,
    rotation: f32,

    fn update(self: *Transform, dt: f32) void {
        self.position = self.position.add(self.velocity.scale(dt));
    }
};

const Health = struct {
    current: f32,
    max: f32,

    fn heal(self: *Health, amount: f32) void {
        self.current = @min(self.current + amount, self.max);
    }

    fn takeDamage(self: *Health, amount: f32) void {
        self.current = @max(self.current - amount, 0.0);
    }

    fn isAlive(self: *const Health) bool {
        return self.current > 0.0;
    }

    fn percentage(self: *const Health) f32 {
        return self.current / self.max * 100.0;
    }
};

const Player = struct {
    name: []const u8,
    transform: Transform,
    health: Health,
    score: u32,

    fn init(name: []const u8, x: f32, y: f32) Player {
        return Player{
            .name = name,
            .transform = Transform{
                .position = Vec2{ .x = x, .y = y },
                .velocity = Vec2{ .x = 0.0, .y = 0.0 },
                .rotation = 0.0,
            },
            .health = Health{ .current = 100.0, .max = 100.0 },
            .score = 0,
        };
    }

    fn update(self: *Player, dt: f32) void {
        self.transform.update(dt);
    }

    fn addScore(self: *Player, points: u32) void {
        self.score += points;
    }

    fn printStatus(self: *const Player) void {
        std.debug.print("Player: {s}\n", .{self.name});
        std.debug.print("  Position: ({d:.1}, {d:.1})\n", .{
            self.transform.position.x,
            self.transform.position.y,
        });
        std.debug.print("  Health: {d:.0}/{d:.0} ({d:.1}%)\n", .{
            self.health.current,
            self.health.max,
            self.health.percentage(),
        });
        std.debug.print("  Score: {d}\n", .{self.score});
        std.debug.print("  Alive: {}\n", .{self.health.isAlive()});
    }
};

pub fn main() void {
    var player = Player.init("Hero", 0.0, 0.0);

    // Move right
    player.transform.velocity = Vec2{ .x = 5.0, .y = 0.0 };
    player.update(1.0); // 1 second

    // Take damage
    player.health.takeDamage(30.0);
    player.addScore(100);

    player.printStatus();

    // More updates
    player.update(2.0); // 2 more seconds
    player.health.heal(10.0);
    player.addScore(250);

    std.debug.print("\nAfter more gameplay:\n", .{});
    player.printStatus();
}
```

**ผลลัพธ์:**
```
Player: Hero
  Position: (5.0, 0.0)
  Health: 70/100 (70.0%)
  Score: 100
  Alive: true

After more gameplay:
  Position: (15.0, 0.0)
  Health: 80/100 (80.0%)
  Score: 350
  Alive: true
```

---

## สรุปความแตกต่าง: Normal vs Packed vs Extern

### Normal Struct

```zig
const Normal = struct {
    a: u8,   // offset 0
    // 3 bytes padding
    b: u32,  // offset 4
    c: u16,  // offset 8
    // 6 bytes padding (เพื่อ align 8)
    d: u64,  // offset 16
};
// Total size: 24 bytes
// ข้อดี: เร็ว, flexible
// ข้อเสีย: ใช้ memory มากกว่า
```

### Packed Struct

```zig
const Packed = packed struct {
    a: u8,   // bits 0-7
    b: u32,  // bits 8-39
    c: u16,  // bits 40-55
    d: u64,  // bits 56-119
};
// Total size: 15 bytes (120 bits)
// ข้อดี: compact, bit manipulation
// ข้อเสีย: อาจช้ากว่า, C-incompatible
```

### Extern Struct

```zig
const Extern = extern struct {
    a: u8,   // offset 0
    // 3 bytes padding (C-style)
    b: u32,  // offset 4
    c: u16,  // offset 8
    // 6 bytes padding
    d: u64,  // offset 16
};
// Total size: 24 bytes (same as C)
// ข้อดี: C ABI compatible
// ข้อเสีย: layout controlled by C rules
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Complex Number

สร้าง struct `Complex` สำหรับจำนวนเชิงซ้อน พร้อม methods:
- `add(other: Complex) Complex`
- `multiply(other: Complex) Complex`
- `magnitude() f64`
- `conjugate() Complex`
- `print()`

```zig
// โครงสร้างเริ่มต้น
const Complex = struct {
    real: f64,
    imag: f64,

    // TODO: implement methods
};
```

### แบบฝึกหัดที่ 2: Linked List Node

สร้าง `Node` struct สำหรับ linked list:

```zig
const Node = struct {
    value: i32,
    next: ?*Node,

    // TODO: implement:
    // fn append(self: *Node, value: i32) void
    // fn count(self: *const Node) usize
    // fn print(self: *const Node) void
};
```

### แบบฝึกหัดที่ 3: Matrix 3x3

สร้าง `Matrix3x3` สำหรับ 3x3 matrix:

```zig
const Matrix3x3 = struct {
    data: [3][3]f64,

    // TODO: implement:
    // fn identity() Matrix3x3     (static)
    // fn multiply(self, other) Matrix3x3
    // fn transpose(self) Matrix3x3
    // fn determinant(self) f64
    // fn print(self)
};
```

### แบบฝึกหัดที่ 4: Bit Flags ด้วย Packed Struct

```zig
const FilePermissions = packed struct {
    owner_read: bool,
    owner_write: bool,
    owner_exec: bool,
    group_read: bool,
    group_write: bool,
    group_exec: bool,
    other_read: bool,
    other_write: bool,
    other_exec: bool,
    _padding: u7 = 0,

    // TODO: implement:
    // fn fromOctal(octal: u16) FilePermissions  (static)
    // fn toOctal(self) u16
    // fn print(self) void  (แสดงแบบ rwxrwxrwx)
};
```

### แบบฝึกหัดที่ 5: GPS Coordinate

```zig
const GPS = struct {
    latitude: f64,   // -90 to 90
    longitude: f64,  // -180 to 180

    // TODO: implement:
    // fn isValid(self) bool
    // fn distanceTo(self, other: GPS) f64  (Haversine formula)
    // fn format(self) string representation
};
```

---

## สรุปบทเรียน

### สิ่งที่เรียนรู้ในภาค 11

| Step | หัวข้อ | สาระสำคัญ |
|------|--------|-----------|
| 101  | Basic Structs | struct เป็น type ใน Zig, PascalCase naming |
| 102  | Fields | ชนิดข้อมูล, default values, ประเภท field |
| 103  | Instances | const vs var, dot access, pointer passing |
| 104  | Methods | self parameter, static vs instance methods |
| 105  | pub/Encapsulation | visibility, validation, constructor pattern |
| 106  | Named Init | named fields บังคับ, builder pattern |
| 107  | Nested | composition over inheritance, delegation |
| 108  | Packed | bit packing, ลด memory, hardware interface |
| 109  | Extern | C ABI compatible, FFI, memory layout |
| 110  | Anonymous/Tuples | multi-return, options pattern, comptime |

### Struct vs Class

| Feature | Zig Struct | OOP Class |
|---------|-----------|-----------|
| Fields | ใช่ | ใช่ |
| Methods | ใช่ | ใช่ |
| Inheritance | ไม่ (composition) | ใช่ |
| Polymorphism | Manual (vtable) | Built-in |
| Constructor | `init()` แบบ convention | Special syntax |
| Destructor | `deinit()` แบบ convention | Automatic |
| Access Control | pub/no-pub | public/private/protected |
| Memory | Manual/Explicit | GC หรือ manual |

### Best Practices

1. **ใช้ `init()` เป็น constructor**: ทำให้ชัดเจนว่าต้องทำอะไรก่อนใช้งาน
2. **ใช้ `deinit()` เป็น destructor**: ทำให้ชัดเจนว่าต้อง cleanup อะไร
3. **ใช้ `const` เมื่อไม่ต้องแก้ไข**: ป้องกัน mutation โดยไม่ตั้งใจ
4. **Pass by pointer สำหรับ mutable**: `fn foo(self: *MyStruct)`
5. **Pass by const pointer สำหรับ read-only**: `fn foo(self: *const MyStruct)`
6. **ใช้ packed struct เมื่อ**: ต้องการ bit-level control หรือ compact storage
7. **ใช้ extern struct เมื่อ**: ต้องการ interop กับ C code

### เปรียบเทียบ Memory Layout

```
Normal Struct:
[a][pad][pad][pad][b  b  b  b][c  c][pad pad pad pad pad pad][ d  d  d  d  d  d  d  d]
 0   1   2   3   4  5  6  7   8   9  10  11  12  13  14  15  16 17 18 19 20 21 22 23

Packed Struct:
[a][b  b  b  b][c  c][d  d  d  d  d  d  d  d]
 0  1  2  3  4   5  6   7  8  9 10 11 12 13 14

Extern Struct (C ABI):
เหมือน Normal แต่รับประกัน layout ตาม C standard
```

---

## ก้าวต่อไป

ในบทถัดไป (ภาค 12) เราจะเรียนรู้เกี่ยวกับ **Enums และ Tagged Unions** ซึ่งทำงานร่วมกับ struct ได้อย่างดีเยี่ยม:

- Enum พื้นฐานและ methods
- Tagged union (sum types)
- Pattern matching ด้วย switch
- Optional และ error handling ขั้นสูง

Struct ใน Zig ให้เราเขียนโค้ดที่มีโครงสร้างชัดเจน ปลอดภัย และมีประสิทธิภาพสูง โดยไม่ต้องพึ่งพา garbage collector หรือ runtime ที่ซับซ้อน!
