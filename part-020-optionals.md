# Part 20: Optionals ใน Zig (Steps 191-200)

## ภาพรวม

ในส่วนนี้เราจะเรียนรู้เรื่อง **Optionals** ซึ่งเป็นหนึ่งในฟีเจอร์ที่สำคัญที่สุดของ Zig สำหรับการจัดการกับค่าที่อาจเป็น `null` ได้อย่างปลอดภัย

Zig แก้ปัญหา "Billion Dollar Mistake" (การใช้ null pointer ใน C/Java) ด้วยระบบ Optional ที่ทำให้ compiler บังคับให้โปรแกรมเมอร์จัดการกับกรณี null ก่อนที่จะใช้ค่า

---

## Step 191: Optional คืออะไร? (?T)

### แนวคิดพื้นฐาน

**Optional** คือ type ที่สามารถมีค่าได้สองแบบ:
1. ค่าจริงของ type `T`
2. ค่า `null` (ไม่มีค่า)

ใน Zig เราเขียน Optional type ด้วย `?T` เช่น `?i32`, `?[]const u8`, `?*Node`

### ทำไมต้องมี Optional?

ใน C เวลาที่ function คืนค่า pointer ที่อาจเป็น `NULL` เราต้องจำเองว่าต้อง check null ก่อน:

```c
// C - ไม่มีอะไรบังคับให้ check null
char* getName() {
    return NULL; // อาจคืน NULL
}

int main() {
    char* name = getName();
    printf("%s\n", name); // CRASH! ลืม check NULL
    return 0;
}
```

ใน Zig compiler บังคับให้เราจัดการกับ null:

```zig
const std = @import("std");

fn getName() ?[]const u8 {
    return null; // อาจคืน null
}

pub fn main() void {
    const name = getName();
    // std.debug.print("{s}\n", .{name}); // ERROR! compiler บอกว่าต้อง unwrap ก่อน

    // ต้อง unwrap อย่างใดอย่างหนึ่ง:
    if (name) |n| {
        std.debug.print("{s}\n", .{n});
    } else {
        std.debug.print("ไม่มีชื่อ\n", .{});
    }
}
```

### โครงสร้างของ Optional

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศ Optional
    var a: ?i32 = 42;      // Optional i32 ที่มีค่า 42
    var b: ?i32 = null;    // Optional i32 ที่ไม่มีค่า
    var c: ?[]const u8 = "สวัสดี"; // Optional string
    var d: ?[]const u8 = null;      // Optional string ที่ว่างเปล่า

    _ = a;
    _ = b;
    _ = c;
    _ = d;

    // ขนาดของ Optional
    std.debug.print("ขนาด i32: {}\n", .{@sizeOf(i32)});
    std.debug.print("ขนาด ?i32: {}\n", .{@sizeOf(?i32)});
    std.debug.print("ขนาด *i32: {}\n", .{@sizeOf(*i32)});
    std.debug.print("ขนาด ?*i32: {}\n", .{@sizeOf(?*i32)});
    // ?*i32 มีขนาดเท่ากับ *i32 เพราะ Zig ใช้ null pointer แทน
}
```

### Type ของ Optional

```zig
const std = @import("std");

pub fn main() void {
    const x: ?i32 = 5;

    // ตรวจสอบ type
    std.debug.print("type ของ x: {}\n", .{@TypeOf(x)});
    // ผลลัพธ์: type ของ x: ?i32

    // Optional สามารถเป็น type อะไรก็ได้
    const opt_bool: ?bool = true;
    const opt_float: ?f64 = 3.14;
    const opt_struct: ?struct { x: i32, y: i32 } = .{ .x = 1, .y = 2 };

    _ = opt_bool;
    _ = opt_float;
    _ = opt_struct;
}
```

---

## Step 192: null และการกำหนดค่า Optional

### การกำหนดค่า null

```zig
const std = @import("std");

pub fn main() void {
    // วิธีที่ 1: กำหนดตรงๆ
    var x: ?i32 = null;
    std.debug.print("x = {?}\n", .{x}); // x = null

    // วิธีที่ 2: กำหนดค่าใหม่
    x = 42;
    std.debug.print("x = {?}\n", .{x}); // x = 42

    // วิธีที่ 3: คืนค่าเป็น null กลับไปได้
    x = null;
    std.debug.print("x = {?}\n", .{x}); // x = null

    // การประกาศแบบ infer type
    const y: ?i32 = 100;
    std.debug.print("y = {?}\n", .{y}); // y = 100
}
```

### Optional กับ Struct

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,
    email: ?[]const u8, // email อาจไม่มีก็ได้
    phone: ?[]const u8, // phone อาจไม่มีก็ได้
};

pub fn main() void {
    const alice = Person{
        .name = "Alice",
        .age = 30,
        .email = "alice@example.com",
        .phone = null, // ไม่มีเบอร์โทร
    };

    const bob = Person{
        .name = "Bob",
        .age = 25,
        .email = null, // ไม่มี email
        .phone = "081-234-5678",
    };

    printPerson(alice);
    printPerson(bob);
}

fn printPerson(p: Person) void {
    std.debug.print("ชื่อ: {s}, อายุ: {}\n", .{ p.name, p.age });

    if (p.email) |email| {
        std.debug.print("  Email: {s}\n", .{email});
    } else {
        std.debug.print("  Email: ไม่มี\n", .{});
    }

    if (p.phone) |phone| {
        std.debug.print("  โทรศัพท์: {s}\n", .{phone});
    } else {
        std.debug.print("  โทรศัพท์: ไม่มี\n", .{});
    }
}
```

### Optional กับ Function Parameters

```zig
const std = @import("std");

// Function ที่รับ Optional parameter
fn greet(name: ?[]const u8) void {
    if (name) |n| {
        std.debug.print("สวัสดี, {s}!\n", .{n});
    } else {
        std.debug.print("สวัสดี, แขกไม่ทราบชื่อ!\n", .{});
    }
}

// Function ที่คืน Optional
fn findUser(id: u32) ?[]const u8 {
    return switch (id) {
        1 => "Alice",
        2 => "Bob",
        3 => "Charlie",
        else => null,
    };
}

pub fn main() void {
    greet("สมชาย");
    greet(null);

    const user1 = findUser(1);
    const user2 = findUser(999);

    std.debug.print("User 1: {?s}\n", .{user1}); // User 1: Alice
    std.debug.print("User 2: {?s}\n", .{user2}); // User 2: null
}
```

### ตรวจสอบว่า Optional มีค่าหรือไม่

```zig
const std = @import("std");

pub fn main() void {
    const x: ?i32 = 42;
    const y: ?i32 = null;

    // วิธีที่ 1: เปรียบเทียบกับ null
    if (x != null) {
        std.debug.print("x มีค่า\n", .{});
    }
    if (y == null) {
        std.debug.print("y ไม่มีค่า\n", .{});
    }

    // วิธีที่ 2: ใช้ if capture (แนะนำมากกว่า)
    if (x) |_| {
        std.debug.print("x มีค่า\n", .{});
    }
}
```

---

## Step 193: Optional unwrapping ด้วย orelse

### orelse คืออะไร?

`orelse` เป็น operator ที่ใช้ unwrap Optional โดยถ้า Optional เป็น null จะให้ค่า default แทน:

```zig
const value = optional orelse default_value;
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const a: ?i32 = 42;
    const b: ?i32 = null;

    // orelse ให้ค่า default
    const val_a = a orelse 0; // 42 (ใช้ค่าจาก a)
    const val_b = b orelse 0; // 0 (ใช้ค่า default)

    std.debug.print("val_a = {}\n", .{val_a}); // val_a = 42
    std.debug.print("val_b = {}\n", .{val_b}); // val_b = 0

    // ใช้กับ string
    const name: ?[]const u8 = null;
    const display_name = name orelse "ไม่ทราบชื่อ";
    std.debug.print("ชื่อ: {s}\n", .{display_name}); // ชื่อ: ไม่ทราบชื่อ
}
```

### orelse กับ return

```zig
const std = @import("std");

fn getAge() ?u32 {
    return null;
}

fn processAge() u32 {
    // ถ้า getAge() คืน null ให้ return จาก function นี้ทันที
    const age = getAge() orelse return 0;
    return age * 2;
}

pub fn main() void {
    const result = processAge();
    std.debug.print("ผลลัพธ์: {}\n", .{result}); // ผลลัพธ์: 0
}
```

### orelse กับ unreachable

```zig
const std = @import("std");

pub fn main() void {
    // ถ้าเราแน่ใจ 100% ว่าไม่เป็น null (ถ้าผิดจะ panic ที่ runtime)
    const x: ?i32 = 42;
    const val = x orelse unreachable; // เราแน่ใจว่า x ไม่เป็น null
    std.debug.print("val = {}\n", .{val}); // val = 42
}
```

### orelse chain

```zig
const std = @import("std");

fn getFromCache() ?[]const u8 {
    return null; // ไม่มีใน cache
}

fn getFromDatabase() ?[]const u8 {
    return null; // ไม่มีใน database
}

fn getDefault() []const u8 {
    return "ค่าเริ่มต้น";
}

pub fn main() void {
    // ลองหาค่าจากหลายที่
    const value = getFromCache() orelse getFromDatabase() orelse getDefault();
    std.debug.print("ค่า: {s}\n", .{value}); // ค่า: ค่าเริ่มต้น
}
```

### .? shorthand

```zig
const std = @import("std");

pub fn main() void {
    const x: ?i32 = 42;

    // .? เทียบเท่ากับ orelse unreachable
    const val = x.?;
    std.debug.print("val = {}\n", .{val}); // val = 42

    // ระวัง! ถ้า x เป็น null จะ panic
    // const y: ?i32 = null;
    // const bad = y.?; // PANIC: attempt to use null value
}
```

---

## Step 194: if กับ Optional (if (opt)) |value|

### Syntax พื้นฐาน

```zig
if (optional_value) |captured_value| {
    // ใช้ captured_value ที่นี่ (ไม่ใช่ Optional แล้ว)
} else {
    // จัดการกรณี null
}
```

### ตัวอย่างพื้นฐาน

```zig
const std = @import("std");

pub fn main() void {
    const x: ?i32 = 42;
    const y: ?i32 = null;

    // กรณีมีค่า
    if (x) |value| {
        std.debug.print("x มีค่า: {}\n", .{value});
    } else {
        std.debug.print("x เป็น null\n", .{});
    }

    // กรณีไม่มีค่า
    if (y) |value| {
        std.debug.print("y มีค่า: {}\n", .{value});
    } else {
        std.debug.print("y เป็น null\n", .{});
    }
}
```

### Capture ด้วย mutable reference

```zig
const std = @import("std");

pub fn main() void {
    var x: ?i32 = 42;

    // Capture เป็น pointer เพื่อแก้ไขค่า
    if (x) |*ptr| {
        ptr.* += 8; // แก้ไขค่าใน Optional
    }

    std.debug.print("x = {?}\n", .{x}); // x = 50
}
```

### Nested Optional

```zig
const std = @import("std");

fn getCity() ?[]const u8 {
    return "กรุงเทพฯ";
}

fn getZipCode(city: []const u8) ?[]const u8 {
    if (std.mem.eql(u8, city, "กรุงเทพฯ")) {
        return "10100";
    }
    return null;
}

pub fn main() void {
    // Nested if สำหรับ Optional chain
    if (getCity()) |city| {
        if (getZipCode(city)) |zip| {
            std.debug.print("เมือง: {s}, รหัสไปรษณีย์: {s}\n", .{ city, zip });
        } else {
            std.debug.print("เมือง: {s}, ไม่มีรหัสไปรษณีย์\n", .{city});
        }
    } else {
        std.debug.print("ไม่มีข้อมูลเมือง\n", .{});
    }
}
```

### if กับ Optional ใน Struct

```zig
const std = @import("std");

const Config = struct {
    max_connections: ?u32,
    timeout_ms: ?u64,
    host: ?[]const u8,
};

fn applyConfig(config: Config) void {
    if (config.max_connections) |max| {
        std.debug.print("ตั้งค่า max connections: {}\n", .{max});
    }

    if (config.timeout_ms) |timeout| {
        std.debug.print("ตั้งค่า timeout: {} ms\n", .{timeout});
    }

    if (config.host) |host| {
        std.debug.print("ตั้งค่า host: {s}\n", .{host});
    } else {
        std.debug.print("ใช้ host เริ่มต้น: localhost\n", .{});
    }
}

pub fn main() void {
    const config = Config{
        .max_connections = 100,
        .timeout_ms = null,
        .host = "api.example.com",
    };

    applyConfig(config);
}
```

### if-else chain กับ Optional

```zig
const std = @import("std");

fn classify(value: ?i32) []const u8 {
    if (value) |v| {
        if (v > 0) return "บวก";
        if (v < 0) return "ลบ";
        return "ศูนย์";
    } else {
        return "ไม่มีค่า";
    }
}

pub fn main() void {
    std.debug.print("{s}\n", .{classify(42)});   // บวก
    std.debug.print("{s}\n", .{classify(-5)});   // ลบ
    std.debug.print("{s}\n", .{classify(0)});    // ศูนย์
    std.debug.print("{s}\n", .{classify(null)}); // ไม่มีค่า
}
```

---

## Step 195: while กับ Optional iterator

### แนวคิด Optional Iterator

ใน Zig สามารถใช้ `while` ร่วมกับ Optional เพื่อวน loop จนกว่าจะได้ค่า null:

```zig
while (optional_expression) |value| {
    // ทำงานกับ value
}
```

### ตัวอย่าง Iterator พื้นฐาน

```zig
const std = @import("std");

// Iterator ง่ายๆ ที่คืน Optional
const Counter = struct {
    count: u32,
    max: u32,

    pub fn next(self: *Counter) ?u32 {
        if (self.count >= self.max) return null;
        const val = self.count;
        self.count += 1;
        return val;
    }
};

pub fn main() void {
    var counter = Counter{ .count = 0, .max = 5 };

    while (counter.next()) |value| {
        std.debug.print("ค่า: {}\n", .{value});
    }
    // ผลลัพธ์: 0, 1, 2, 3, 4
}
```

### while กับ Optional และ else

```zig
const std = @import("std");

const NumberIterator = struct {
    data: []const i32,
    index: usize,

    pub fn next(self: *NumberIterator) ?i32 {
        if (self.index >= self.data.len) return null;
        const val = self.data[self.index];
        self.index += 1;
        return val;
    }
};

pub fn main() void {
    const numbers = [_]i32{ 10, 20, 30, 40, 50 };
    var iter = NumberIterator{ .data = &numbers, .index = 0 };

    var sum: i32 = 0;
    while (iter.next()) |num| {
        sum += num;
    } else {
        std.debug.print("วน loop เสร็จแล้ว\n", .{});
    }

    std.debug.print("ผลรวม: {}\n", .{sum}); // ผลรวม: 150
}
```

### while Optional กับ condition

```zig
const std = @import("std");

const FilterIterator = struct {
    data: []const i32,
    index: usize,
    min_val: i32,

    pub fn next(self: *FilterIterator) ?i32 {
        while (self.index < self.data.len) {
            const val = self.data[self.index];
            self.index += 1;
            if (val >= self.min_val) return val;
        }
        return null;
    }
};

pub fn main() void {
    const numbers = [_]i32{ 5, 15, 3, 25, 8, 30, 2 };
    var iter = FilterIterator{
        .data = &numbers,
        .index = 0,
        .min_val = 10,
    };

    std.debug.print("ตัวเลขที่ >= 10: ", .{});
    while (iter.next()) |num| {
        std.debug.print("{} ", .{num});
    }
    std.debug.print("\n", .{});
    // ผลลัพธ์: 15 25 30
}
```

### SplitIterator (ตัวอย่างจริง)

```zig
const std = @import("std");

pub fn main() void {
    const text = "สวัสดี,โลก,Zig,Programming";
    var iter = std.mem.splitScalar(u8, text, ',');

    var index: u32 = 1;
    while (iter.next()) |part| {
        std.debug.print("ส่วนที่ {}: {s}\n", .{ index, part });
        index += 1;
    }
}
```

### while Optional กับ Iterator Pattern ที่ซับซ้อน

```zig
const std = @import("std");

// Iterator ที่แปลงข้อมูลระหว่างทาง
const TransformIterator = struct {
    source: []const u8,
    index: usize,

    pub fn next(self: *TransformIterator) ?u8 {
        if (self.index >= self.source.len) return null;
        const ch = self.source[self.index];
        self.index += 1;
        // แปลง lowercase เป็น uppercase
        return std.ascii.toUpper(ch);
    }
};

pub fn main() void {
    const text = "hello zig";
    var iter = TransformIterator{ .source = text, .index = 0 };

    var buf: [100]u8 = undefined;
    var len: usize = 0;

    while (iter.next()) |ch| {
        buf[len] = ch;
        len += 1;
    }

    std.debug.print("ผลลัพธ์: {s}\n", .{buf[0..len]});
    // ผลลัพธ์: HELLO ZIG
}
```

---

## Step 196: Optional pointer ?*T

### ความพิเศษของ Optional Pointer

ใน Zig `?*T` มีขนาดเท่ากับ `*T` เพราะ Zig ใช้ null pointer (0x0) เพื่อแทน null ทำให้ไม่มี overhead พิเศษ:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("ขนาด *i32: {}\n", .{@sizeOf(*i32)});
    std.debug.print("ขนาด ?*i32: {}\n", .{@sizeOf(?*i32)});
    // ทั้งสองมีขนาดเท่ากัน (8 bytes บน 64-bit system)

    std.debug.print("ขนาด i32: {}\n", .{@sizeOf(i32)});
    std.debug.print("ขนาด ?i32: {}\n", .{@sizeOf(?i32)});
    // ?i32 ใหญ่กว่า i32 เล็กน้อย (ต้องการ bit พิเศษ)
}
```

### การใช้ Optional Pointer

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 42;
    var y: i32 = 100;

    var ptr: ?*i32 = &x;

    if (ptr) |p| {
        std.debug.print("ค่าที่ ptr ชี้: {}\n", .{p.*});
    }

    // เปลี่ยน pointer ให้ชี้ที่อื่น
    ptr = &y;
    if (ptr) |p| {
        std.debug.print("ค่าใหม่: {}\n", .{p.*});
    }

    // ตั้งเป็น null
    ptr = null;
    if (ptr) |_| {
        std.debug.print("มี pointer\n", .{});
    } else {
        std.debug.print("pointer เป็น null\n", .{});
    }
}
```

### Optional Pointer ใน Linked List

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node, // Optional pointer ไปยัง node ถัดไป

    pub fn init(value: i32) Node {
        return Node{
            .value = value,
            .next = null,
        };
    }
};

const LinkedList = struct {
    head: ?*Node,

    pub fn init() LinkedList {
        return LinkedList{ .head = null };
    }

    pub fn printAll(self: *const LinkedList) void {
        var current = self.head;
        while (current) |node| {
            std.debug.print("{} -> ", .{node.value});
            current = node.next;
        }
        std.debug.print("null\n", .{});
    }
};

pub fn main() void {
    var node1 = Node.init(1);
    var node2 = Node.init(2);
    var node3 = Node.init(3);

    node1.next = &node2;
    node2.next = &node3;
    // node3.next = null (ค่าเริ่มต้น)

    var list = LinkedList{ .head = &node1 };
    list.printAll(); // 1 -> 2 -> 3 -> null
}
```

### Optional Pointer กับ Tree

```zig
const std = @import("std");

const TreeNode = struct {
    value: i32,
    left: ?*TreeNode,  // ลูกซ้าย (อาจไม่มี)
    right: ?*TreeNode, // ลูกขวา (อาจไม่มี)

    pub fn init(value: i32) TreeNode {
        return TreeNode{
            .value = value,
            .left = null,
            .right = null,
        };
    }
};

fn sumTree(node: ?*const TreeNode) i32 {
    // Base case: ถ้า node เป็น null คืน 0
    const n = node orelse return 0;
    return n.value + sumTree(n.left) + sumTree(n.right);
}

pub fn main() void {
    //      5
    //     / \
    //    3   8
    //   / \
    //  1   4

    var n1 = TreeNode.init(1);
    var n4 = TreeNode.init(4);
    var n3 = TreeNode.init(3);
    var n8 = TreeNode.init(8);
    var root = TreeNode.init(5);

    n3.left = &n1;
    n3.right = &n4;
    root.left = &n3;
    root.right = &n8;

    const total = sumTree(&root);
    std.debug.print("ผลรวมของ tree: {}\n", .{total}); // 21
}
```

### การแก้ไขค่าผ่าน Optional Pointer

```zig
const std = @import("std");

fn doubleIfExists(ptr: ?*i32) void {
    if (ptr) |p| {
        p.* *= 2;
    }
}

pub fn main() void {
    var x: i32 = 10;
    var y: i32 = 20;

    doubleIfExists(&x);
    doubleIfExists(null);
    doubleIfExists(&y);

    std.debug.print("x = {}\n", .{x}); // x = 20
    std.debug.print("y = {}\n", .{y}); // y = 40
}
```

---

## Step 197: Optional method chaining

### ปัญหาของ Optional Chain

เมื่อเราต้องการ access ข้อมูลที่อาจเป็น null หลายชั้น:

```zig
// สิ่งที่เราอยากเขียน (แต่ไม่ compile):
// const zip = user?.address?.city?.zipcode

// สิ่งที่ต้องเขียนใน Zig:
if (user) |u| {
    if (u.address) |addr| {
        if (addr.city) |city| {
            // ใช้ city
        }
    }
}
```

### Optional Chain Pattern

```zig
const std = @import("std");

const ZipCode = struct {
    code: []const u8,
};

const City = struct {
    name: []const u8,
    zip: ?ZipCode,
};

const Address = struct {
    street: []const u8,
    city: ?City,
};

const User = struct {
    name: []const u8,
    address: ?Address,
};

fn getUserZip(user: ?User) ?[]const u8 {
    const u = user orelse return null;
    const addr = u.address orelse return null;
    const city = addr.city orelse return null;
    const zip = city.zip orelse return null;
    return zip.code;
}

pub fn main() void {
    const user1 = User{
        .name = "Alice",
        .address = Address{
            .street = "123 Main St",
            .city = City{
                .name = "กรุงเทพฯ",
                .zip = ZipCode{ .code = "10100" },
            },
        },
    };

    const user2 = User{
        .name = "Bob",
        .address = null, // ไม่มีที่อยู่
    };

    const zip1 = getUserZip(user1);
    const zip2 = getUserZip(user2);
    const zip3 = getUserZip(null);

    std.debug.print("zip1: {?s}\n", .{zip1}); // zip1: 10100
    std.debug.print("zip2: {?s}\n", .{zip2}); // zip2: null
    std.debug.print("zip3: {?s}\n", .{zip3}); // zip3: null
}
```

### Optional Method Call Pattern

```zig
const std = @import("std");

const Database = struct {
    connected: bool,
    data: []const u8,

    pub fn query(self: *const Database, key: []const u8) ?[]const u8 {
        if (!self.connected) return null;
        _ = key;
        return self.data;
    }
};

fn getConnection() ?*Database {
    return null; // จำลองว่าเชื่อมต่อไม่ได้
}

pub fn main() void {
    // Pattern: optional method call
    const db = getConnection();

    // ต้อง unwrap ก่อน call method
    const result = if (db) |d| d.query("user_name") else null;

    if (result) |r| {
        std.debug.print("ผลลัพธ์: {s}\n", .{r});
    } else {
        std.debug.print("ไม่สามารถดึงข้อมูลได้\n", .{});
    }
}
```

### Helper Function Pattern

```zig
const std = @import("std");

const Config = struct {
    settings: ?Settings,

    pub fn getSetting(self: *const Config, key: []const u8) ?[]const u8 {
        const s = self.settings orelse return null;
        return s.get(key);
    }
};

const Settings = struct {
    items: [3]struct { key: []const u8, value: []const u8 },
    count: usize,

    pub fn get(self: *const Settings, key: []const u8) ?[]const u8 {
        for (self.items[0..self.count]) |item| {
            if (std.mem.eql(u8, item.key, key)) {
                return item.value;
            }
        }
        return null;
    }
};

pub fn main() void {
    var settings = Settings{
        .items = .{
            .{ .key = "theme", .value = "dark" },
            .{ .key = "lang", .value = "th" },
            .{ .key = "size", .value = "16" },
        },
        .count = 3,
    };

    const config = Config{ .settings = settings };

    const theme = config.getSetting("theme") orelse "light";
    const missing = config.getSetting("missing") orelse "default";

    std.debug.print("theme: {s}\n", .{theme});     // theme: dark
    std.debug.print("missing: {s}\n", .{missing}); // missing: default
}
```

---

## Step 198: เปรียบเทียบ Optional กับ null ใน C

### ปัญหาของ null ใน C

```c
// C - Billion Dollar Mistake
#include <stdio.h>
#include <string.h>

// ไม่มีอะไรบังคับให้ check NULL
char* getUsername(int id) {
    if (id == 1) return "Alice";
    return NULL; // อาจ return NULL
}

int getNameLength(int id) {
    char* name = getUsername(id);
    // ลืม check NULL -> CRASH
    return strlen(name); // Segmentation fault ถ้า name == NULL
}

// คืน -1 เป็น sentinel value
int findIndex(int* arr, int len, int target) {
    for (int i = 0; i < len; i++) {
        if (arr[i] == target) return i;
    }
    return -1; // ต้องจำว่า -1 หมายถึงไม่พบ
}
```

### ปัญหาของ null ใน Java

```java
// Java - NullPointerException
public class Example {
    // Caller ไม่รู้ว่า return value อาจเป็น null
    public static String getUsername(int id) {
        if (id == 1) return "Alice";
        return null;
    }

    public static void main(String[] args) {
        String name = getUsername(999);
        System.out.println(name.length()); // NullPointerException!
    }
}
```

### วิธีแก้ใน Zig

```zig
const std = @import("std");

// Zig บังคับให้ handle null ทุกครั้ง
fn getUsername(id: u32) ?[]const u8 {
    return switch (id) {
        1 => "Alice",
        2 => "Bob",
        else => null,
    };
}

fn getNameLength(id: u32) usize {
    // Compiler บังคับให้ unwrap ก่อน
    const name = getUsername(id) orelse return 0;
    return name.len;
}

pub fn main() void {
    // ต้อง handle ทุกกรณี
    if (getUsername(999)) |name| {
        std.debug.print("ชื่อ: {s}\n", .{name});
    } else {
        std.debug.print("ไม่พบผู้ใช้\n", .{});
    }

    const len = getNameLength(1);
    std.debug.print("ความยาวชื่อ: {}\n", .{len}); // 5
}
```

### เปรียบเทียบแบบตาราง

```zig
const std = @import("std");

// C style (ใน Zig แต่ไม่แนะนำ):
// ใช้ sentinel value
fn findIndexBadStyle(arr: []const i32, target: i32) i32 {
    for (arr, 0..) |val, i| {
        if (val == target) return @intCast(i);
    }
    return -1; // sentinel value - ไม่ชัดเจน
}

// Zig style (แนะนำ):
// ใช้ Optional
fn findIndex(arr: []const i32, target: i32) ?usize {
    for (arr, 0..) |val, i| {
        if (val == target) return i;
    }
    return null; // ชัดเจนว่าไม่พบ
}

pub fn main() void {
    const numbers = [_]i32{ 10, 20, 30, 40, 50 };

    // C style - ต้องรู้ว่า -1 หมายถึงอะไร
    const bad_result = findIndexBadStyle(&numbers, 30);
    if (bad_result != -1) {
        std.debug.print("พบที่ index: {}\n", .{bad_result});
    }

    // Zig style - ชัดเจน
    if (findIndex(&numbers, 30)) |index| {
        std.debug.print("พบที่ index: {}\n", .{index});
    } else {
        std.debug.print("ไม่พบ\n", .{});
    }

    // ไม่พบ
    if (findIndex(&numbers, 99)) |index| {
        std.debug.print("พบที่ index: {}\n", .{index});
    } else {
        std.debug.print("ไม่พบ 99 ในรายการ\n", .{});
    }
}
```

### ข้อได้เปรียบของ Optional ใน Zig

1. **Compile-time safety**: Compiler บังคับให้ handle null
2. **Zero overhead**: `?*T` มีขนาดเท่ากับ `*T`
3. **Clarity**: ชัดเจนว่า function ไหนอาจคืน null
4. **No sentinel values**: ไม่ต้องใช้ -1, "", หรือค่าพิเศษอื่นๆ

```zig
const std = @import("std");

// เปรียบเทียบ memory layout

pub fn main() void {
    // ใน C: NULL pointer = 0x0
    // ใน Zig: ?*T ใช้ null pointer เหมือนกัน แต่ compiler บังคับ check

    const ptr_size = @sizeOf(*i32);
    const opt_ptr_size = @sizeOf(?*i32);
    const int_size = @sizeOf(i32);
    const opt_int_size = @sizeOf(?i32);

    std.debug.print("*i32 = {} bytes\n", .{ptr_size});
    std.debug.print("?*i32 = {} bytes (เท่ากัน!)\n", .{opt_ptr_size});
    std.debug.print("i32 = {} bytes\n", .{int_size});
    std.debug.print("?i32 = {} bytes (มี overhead นิดหน่อย)\n", .{opt_int_size});
}
```

### Rust vs Zig vs C เปรียบเทียบ

```zig
// Rust ใช้ Option<T>:
// match x {
//     Some(val) => use val,
//     None => handle None,
// }

// Zig ใช้ ?T:
// if (x) |val| { use val } else { handle null }

// C ใช้ NULL pointer:
// if (x != NULL) { use x } // ลืมได้!

// ทั้ง Rust และ Zig บังคับ handle
// C ไม่บังคับ -> bug ง่าย
```

---

## Step 199: Optional กับ error union

### ความสัมพันธ์ระหว่าง Optional และ Error Union

ใน Zig มีสองวิธีในการบอกว่า "อาจไม่มีผลลัพธ์":
1. `?T` - ไม่มีค่า (null) โดยไม่มีข้อมูลเพิ่มเติม
2. `!T` หรือ `anyerror!T` - ล้มเหลวพร้อมบอกสาเหตุ

### เมื่อไหรควรใช้อะไร?

```zig
const std = @import("std");

// ใช้ Optional ?T เมื่อ:
// - ไม่มีข้อมูลเป็นเรื่องปกติ (ไม่ใช่ error)
fn findUser(id: u32) ?[]const u8 {
    return switch (id) {
        1 => "Alice",
        else => null, // ไม่พบก็ปกติ
    };
}

// ใช้ Error Union !T เมื่อ:
// - บางอย่างผิดพลาด และต้องการบอกสาเหตุ
const UserError = error{
    NotFound,
    DatabaseError,
    PermissionDenied,
};

fn getUserSecure(id: u32, caller_id: u32) UserError![]const u8 {
    if (caller_id != 0) return UserError.PermissionDenied;
    return switch (id) {
        1 => "Alice",
        2 => "Bob",
        else => UserError.NotFound,
    };
}

pub fn main() void {
    // Optional
    if (findUser(1)) |name| {
        std.debug.print("พบ: {s}\n", .{name});
    }

    // Error Union
    if (getUserSecure(1, 0)) |name| {
        std.debug.print("พบ (secure): {s}\n", .{name});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});
    }

    if (getUserSecure(1, 5)) |name| {
        std.debug.print("พบ (secure): {s}\n", .{name});
    } else |err| {
        std.debug.print("Error: {}\n", .{err}); // PermissionDenied
    }
}
```

### แปลง Optional เป็น Error Union

```zig
const std = @import("std");

const LookupError = error{NotFound};

fn optionalToError(opt: ?i32) LookupError!i32 {
    return opt orelse LookupError.NotFound;
}

pub fn main() void {
    const result1 = optionalToError(42) catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("ผลลัพธ์: {}\n", .{result1}); // 42

    const result2 = optionalToError(null) catch |err| blk: {
        std.debug.print("Error: {}\n", .{err}); // NotFound
        break :blk -1;
    };
    std.debug.print("ผลลัพธ์: {}\n", .{result2}); // -1
}
```

### แปลง Error Union เป็น Optional

```zig
const std = @import("std");

fn riskyOperation(x: i32) !i32 {
    if (x < 0) return error.NegativeNumber;
    return x * 2;
}

pub fn main() void {
    // ใช้ catch null เพื่อแปลง error เป็น null
    const result1: ?i32 = riskyOperation(5) catch null;
    const result2: ?i32 = riskyOperation(-1) catch null;

    std.debug.print("result1: {?}\n", .{result1}); // result1: 10
    std.debug.print("result2: {?}\n", .{result2}); // result2: null
}
```

### Optional กับ Error Union ใน Pipeline

```zig
const std = @import("std");

fn parseNumber(s: []const u8) !i32 {
    return std.fmt.parseInt(i32, s, 10);
}

fn validatePositive(n: i32) ?i32 {
    if (n > 0) return n;
    return null;
}

fn processInput(input: []const u8) ?i32 {
    // แปลง string -> number (อาจ error)
    const num = parseNumber(input) catch return null;
    // ตรวจสอบว่าบวก (อาจ null)
    return validatePositive(num);
}

pub fn main() void {
    const inputs = [_][]const u8{ "42", "-5", "abc", "100" };

    for (inputs) |input| {
        if (processInput(input)) |result| {
            std.debug.print("'{s}' -> {}\n", .{ input, result });
        } else {
            std.debug.print("'{s}' -> ไม่ valid\n", .{input});
        }
    }
    // '42' -> 42
    // '-5' -> ไม่ valid
    // 'abc' -> ไม่ valid
    // '100' -> 100
}
```

### ใช้ทั้งคู่พร้อมกัน

```zig
const std = @import("std");

const ConfigError = error{
    ParseError,
    InvalidValue,
};

const AppConfig = struct {
    port: u16,
    host: []const u8,
    max_conn: ?u32, // Optional - อาจไม่ระบุ
};

fn loadConfig(port_str: []const u8) ConfigError!AppConfig {
    const port = std.fmt.parseInt(u16, port_str, 10) catch
        return ConfigError.ParseError;

    if (port < 1024) return ConfigError.InvalidValue;

    return AppConfig{
        .port = port,
        .host = "localhost",
        .max_conn = null, // ไม่ระบุ max connections
    };
}

pub fn main() void {
    const config = loadConfig("8080") catch |err| {
        std.debug.print("ไม่สามารถโหลด config: {}\n", .{err});
        return;
    };

    std.debug.print("Port: {}\n", .{config.port});
    std.debug.print("Host: {s}\n", .{config.host});

    const max = config.max_conn orelse 100; // default 100
    std.debug.print("Max connections: {}\n", .{max});
}
```

---

## Step 200: Patterns การใช้ Optional ที่ดี

### Pattern 1: Early Return Pattern

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    email: ?[]const u8,
    age: ?u32,
};

// Bad: Pyramid of doom
fn processUserBad(user: ?User) void {
    if (user) |u| {
        if (u.email) |email| {
            if (u.age) |age| {
                if (age >= 18) {
                    std.debug.print("ส่ง email ถึง {s} ที่ {s}\n", .{ u.name, email });
                }
            }
        }
    }
}

// Good: Early return pattern
fn processUserGood(user: ?User) void {
    const u = user orelse return;
    const email = u.email orelse return;
    const age = u.age orelse return;
    if (age < 18) return;

    std.debug.print("ส่ง email ถึง {s} ที่ {s}\n", .{ u.name, email });
}

pub fn main() void {
    const user1: ?User = User{
        .name = "Alice",
        .email = "alice@example.com",
        .age = 25,
    };

    const user2: ?User = User{
        .name = "Bob",
        .email = null, // ไม่มี email
        .age = 30,
    };

    processUserBad(user1);
    processUserGood(user1);
    processUserGood(user2); // ออกจาก function เพราะไม่มี email
    processUserGood(null);  // ออกจาก function เพราะ null
}
```

### Pattern 2: Default Value Pattern

```zig
const std = @import("std");

const ServerConfig = struct {
    host: ?[]const u8,
    port: ?u16,
    workers: ?u32,
    timeout_ms: ?u64,
};

fn applyDefaults(config: ServerConfig) struct {
    host: []const u8,
    port: u16,
    workers: u32,
    timeout_ms: u64,
} {
    return .{
        .host = config.host orelse "localhost",
        .port = config.port orelse 8080,
        .workers = config.workers orelse 4,
        .timeout_ms = config.timeout_ms orelse 30_000,
    };
}

pub fn main() void {
    const config = ServerConfig{
        .host = "api.example.com",
        .port = null,
        .workers = null,
        .timeout_ms = 60_000,
    };

    const final = applyDefaults(config);
    std.debug.print("host: {s}\n", .{final.host});         // api.example.com
    std.debug.print("port: {}\n", .{final.port});          // 8080 (default)
    std.debug.print("workers: {}\n", .{final.workers});    // 4 (default)
    std.debug.print("timeout: {}\n", .{final.timeout_ms}); // 60000 (custom)
}
```

### Pattern 3: Optional กับ HashMap

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var map = std.StringHashMap(i32).init(allocator);
    defer map.deinit();

    try map.put("apple", 5);
    try map.put("banana", 3);
    try map.put("cherry", 8);

    // HashMap.get คืน Optional
    const names = [_][]const u8{ "apple", "grape", "banana" };

    for (names) |name| {
        if (map.get(name)) |count| {
            std.debug.print("{s}: {} อัน\n", .{ name, count });
        } else {
            std.debug.print("{s}: ไม่มีในรายการ\n", .{name});
        }
    }

    // ใช้ orelse สำหรับ default value
    const mango_count = map.get("mango") orelse 0;
    std.debug.print("mango: {} อัน\n", .{mango_count}); // 0
}
```

### Pattern 4: Optional ใน State Machine

```zig
const std = @import("std");

const State = enum {
    idle,
    loading,
    ready,
    error_state,
};

const AppState = struct {
    state: State,
    data: ?[]const u8,     // มีข้อมูลเมื่อ state = ready
    error_msg: ?[]const u8, // มี error message เมื่อ state = error_state

    pub fn init() AppState {
        return .{
            .state = .idle,
            .data = null,
            .error_msg = null,
        };
    }

    pub fn startLoading(self: *AppState) void {
        self.state = .loading;
        self.data = null;
        self.error_msg = null;
    }

    pub fn setData(self: *AppState, data: []const u8) void {
        self.state = .ready;
        self.data = data;
        self.error_msg = null;
    }

    pub fn setError(self: *AppState, msg: []const u8) void {
        self.state = .error_state;
        self.data = null;
        self.error_msg = msg;
    }

    pub fn display(self: *const AppState) void {
        switch (self.state) {
            .idle => std.debug.print("สถานะ: พักอยู่\n", .{}),
            .loading => std.debug.print("สถานะ: กำลังโหลด...\n", .{}),
            .ready => {
                const d = self.data orelse "(ว่าง)";
                std.debug.print("สถานะ: พร้อม - ข้อมูล: {s}\n", .{d});
            },
            .error_state => {
                const msg = self.error_msg orelse "Unknown error";
                std.debug.print("สถานะ: Error - {s}\n", .{msg});
            },
        }
    }
};

pub fn main() void {
    var app = AppState.init();
    app.display();        // สถานะ: พักอยู่

    app.startLoading();
    app.display();        // สถานะ: กำลังโหลด...

    app.setData("ข้อมูลจาก API");
    app.display();        // สถานะ: พร้อม - ข้อมูล: ข้อมูลจาก API

    app.setError("Network timeout");
    app.display();        // สถานะ: Error - Network timeout
}
```

### Pattern 5: Optional ใน Builder Pattern

```zig
const std = @import("std");

const HttpRequest = struct {
    url: []const u8,
    method: []const u8,
    body: ?[]const u8,
    auth_token: ?[]const u8,
    timeout: ?u64,

    const Builder = struct {
        url: []const u8,
        method: []const u8 = "GET",
        body: ?[]const u8 = null,
        auth_token: ?[]const u8 = null,
        timeout: ?u64 = null,

        pub fn withMethod(self: Builder, method: []const u8) Builder {
            var b = self;
            b.method = method;
            return b;
        }

        pub fn withBody(self: Builder, body: []const u8) Builder {
            var b = self;
            b.body = body;
            return b;
        }

        pub fn withAuth(self: Builder, token: []const u8) Builder {
            var b = self;
            b.auth_token = token;
            return b;
        }

        pub fn withTimeout(self: Builder, ms: u64) Builder {
            var b = self;
            b.timeout = ms;
            return b;
        }

        pub fn build(self: Builder) HttpRequest {
            return HttpRequest{
                .url = self.url,
                .method = self.method,
                .body = self.body,
                .auth_token = self.auth_token,
                .timeout = self.timeout,
            };
        }
    };

    pub fn builder(url: []const u8) Builder {
        return Builder{ .url = url };
    }
};

pub fn main() void {
    const req1 = HttpRequest.builder("https://api.example.com/users")
        .build();

    const req2 = HttpRequest.builder("https://api.example.com/posts")
        .withMethod("POST")
        .withBody("{\"title\":\"Hello\"}")
        .withAuth("Bearer token123")
        .withTimeout(5000)
        .build();

    std.debug.print("req1: {s} {s}\n", .{ req1.method, req1.url });
    std.debug.print("req2 method: {s}\n", .{req2.method});
    std.debug.print("req2 body: {?s}\n", .{req2.body});
    std.debug.print("req2 auth: {?s}\n", .{req2.auth_token});
    std.debug.print("req2 timeout: {?} ms\n", .{req2.timeout});
}
```

### Pattern 6: Optional Lazy Initialization

```zig
const std = @import("std");

const Cache = struct {
    data: ?[]const u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Cache {
        return Cache{
            .data = null,
            .allocator = allocator,
        };
    }

    // Lazy initialization - คำนวณเมื่อจำเป็น
    pub fn getData(self: *Cache) ![]const u8 {
        if (self.data) |cached| {
            std.debug.print("  (ใช้ cache)\n", .{});
            return cached;
        }

        // คำนวณครั้งแรก
        std.debug.print("  (คำนวณใหม่)\n", .{});
        const computed = try std.fmt.allocPrint(
            self.allocator,
            "computed_value_{d}",
            .{std.time.milliTimestamp()},
        );
        self.data = computed;
        return computed;
    }

    pub fn deinit(self: *Cache) void {
        if (self.data) |d| {
            self.allocator.free(d);
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var cache = Cache.init(allocator);
    defer cache.deinit();

    std.debug.print("ครั้งที่ 1: ", .{});
    const v1 = try cache.getData();
    std.debug.print("ค่า: {s}\n", .{v1});

    std.debug.print("ครั้งที่ 2: ", .{});
    const v2 = try cache.getData();
    std.debug.print("ค่า: {s}\n", .{v2});
    // ครั้งที่ 2 ใช้ cache ไม่คำนวณใหม่
}
```

### Pattern 7: Optional กับ Recursive Data Structures

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Binary Search Tree
const BST = struct {
    const Node = struct {
        value: i32,
        left: ?*Node,
        right: ?*Node,
    };

    root: ?*Node,
    allocator: Allocator,

    pub fn init(allocator: Allocator) BST {
        return BST{ .root = null, .allocator = allocator };
    }

    pub fn insert(self: *BST, value: i32) !void {
        self.root = try insertNode(self.allocator, self.root, value);
    }

    pub fn contains(self: *const BST, value: i32) bool {
        return searchNode(self.root, value);
    }

    fn insertNode(allocator: Allocator, node: ?*Node, value: i32) !?*Node {
        if (node) |n| {
            if (value < n.value) {
                n.left = try insertNode(allocator, n.left, value);
            } else if (value > n.value) {
                n.right = try insertNode(allocator, n.right, value);
            }
            return n;
        } else {
            const new_node = try allocator.create(Node);
            new_node.* = Node{
                .value = value,
                .left = null,
                .right = null,
            };
            return new_node;
        }
    }

    fn searchNode(node: ?*const Node, value: i32) bool {
        const n = node orelse return false;
        if (value == n.value) return true;
        if (value < n.value) return searchNode(n.left, value);
        return searchNode(n.right, value);
    }

    fn freeNode(allocator: Allocator, node: ?*Node) void {
        const n = node orelse return;
        freeNode(allocator, n.left);
        freeNode(allocator, n.right);
        allocator.destroy(n);
    }

    pub fn deinit(self: *BST) void {
        freeNode(self.allocator, self.root);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var tree = BST.init(gpa.allocator());
    defer tree.deinit();

    try tree.insert(5);
    try tree.insert(3);
    try tree.insert(8);
    try tree.insert(1);
    try tree.insert(4);

    const values = [_]i32{ 1, 4, 5, 6, 8, 9 };
    for (values) |v| {
        if (tree.contains(v)) {
            std.debug.print("{} อยู่ใน tree\n", .{v});
        } else {
            std.debug.print("{} ไม่อยู่ใน tree\n", .{v});
        }
    }
}
```

---

## สรุป Optional ใน Zig

### สิ่งที่ควรจำ

```zig
const std = @import("std");

pub fn main() void {
    // 1. ประกาศ Optional ด้วย ?T
    const opt: ?i32 = 42;
    const none: ?i32 = null;

    _ = none;

    // 2. Unwrap ด้วย if capture
    if (opt) |value| {
        std.debug.print("มีค่า: {}\n", .{value});
    }

    // 3. Default value ด้วย orelse
    const val = opt orelse 0;
    std.debug.print("ค่า: {}\n", .{val});

    // 4. Force unwrap ด้วย .? (ระวัง panic!)
    const forced = opt.?;
    std.debug.print("forced: {}\n", .{forced});

    // 5. ส่ง Optional ไปยัง function
    _ = maybeDouble(opt);
    _ = maybeDouble(null);

    // 6. ?*T มีขนาดเท่า *T
    std.debug.print("*i32 = {}, ?*i32 = {}\n", .{
        @sizeOf(*i32),
        @sizeOf(?*i32),
    });
}

fn maybeDouble(x: ?i32) ?i32 {
    const val = x orelse return null;
    return val * 2;
}
```

### Quick Reference

| การใช้งาน | Syntax |
|-----------|--------|
| ประกาศ Optional | `var x: ?i32 = 42;` |
| กำหนด null | `var x: ?i32 = null;` |
| Capture ด้วย if | `if (x) \|v\| { ... }` |
| Default value | `x orelse default` |
| Force unwrap | `x.?` |
| Early return | `const v = x orelse return;` |
| Optional pointer | `?*T` |
| Capture mutable | `if (x) \|*ptr\| { ptr.* = ... }` |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Optional String Processing

เขียนโปรแกรมที่:
1. มี function `parseAge(s: []const u8) ?u32` ที่ parse string เป็น age
2. ถ้า string ว่างหรือ parse ไม่ได้ ให้คืน null
3. ถ้า age น้อยกว่า 0 หรือมากกว่า 150 ให้คืน null

```zig
const std = @import("std");

fn parseAge(s: []const u8) ?u32 {
    // TODO: implement
    if (s.len == 0) return null;
    const age = std.fmt.parseInt(u32, s, 10) catch return null;
    if (age > 150) return null;
    return age;
}

pub fn main() void {
    const inputs = [_][]const u8{ "25", "abc", "", "200", "0", "100" };

    for (inputs) |input| {
        if (parseAge(input)) |age| {
            std.debug.print("'{s}' -> อายุ {}\n", .{ input, age });
        } else {
            std.debug.print("'{s}' -> ไม่ valid\n", .{input});
        }
    }
}
```

### แบบฝึกหัดที่ 2: Optional Chain

สร้าง struct hierarchy และ function ที่ traverse ผ่าน Optional fields:

```zig
const std = @import("std");

const Country = struct {
    name: []const u8,
    code: []const u8,
};

const City = struct {
    name: []const u8,
    country: ?Country,
};

const Address = struct {
    street: []const u8,
    city: ?City,
};

const Profile = struct {
    username: []const u8,
    address: ?Address,
};

// TODO: เขียน function getCountryCode(profile: ?Profile) ?[]const u8
// ที่คืน country code ถ้ามี หรือ null ถ้าไม่มี

fn getCountryCode(profile: ?Profile) ?[]const u8 {
    const p = profile orelse return null;
    const addr = p.address orelse return null;
    const city = addr.city orelse return null;
    const country = city.country orelse return null;
    return country.code;
}

pub fn main() void {
    const profile1: ?Profile = Profile{
        .username = "alice",
        .address = Address{
            .street = "123 Main St",
            .city = City{
                .name = "กรุงเทพฯ",
                .country = Country{ .name = "Thailand", .code = "TH" },
            },
        },
    };

    const profile2: ?Profile = Profile{
        .username = "bob",
        .address = null,
    };

    std.debug.print("alice country: {?s}\n", .{getCountryCode(profile1)});
    std.debug.print("bob country: {?s}\n", .{getCountryCode(profile2)});
    std.debug.print("null country: {?s}\n", .{getCountryCode(null)});
}
```

### แบบฝึกหัดที่ 3: Optional Iterator

สร้าง Iterator ที่กรองเฉพาะเลขคี่:

```zig
const std = @import("std");

const OddIterator = struct {
    data: []const i32,
    index: usize,

    pub fn init(data: []const i32) OddIterator {
        return OddIterator{ .data = data, .index = 0 };
    }

    // TODO: implement next() ที่คืน Optional
    // คืนเฉพาะเลขคี่ ข้ามเลขคู่
    pub fn next(self: *OddIterator) ?i32 {
        while (self.index < self.data.len) {
            const val = self.data[self.index];
            self.index += 1;
            if (@rem(val, 2) != 0) return val;
        }
        return null;
    }
};

pub fn main() void {
    const numbers = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    var iter = OddIterator.init(&numbers);

    std.debug.print("เลขคี่: ", .{});
    while (iter.next()) |num| {
        std.debug.print("{} ", .{num});
    }
    std.debug.print("\n", .{});
    // เลขคี่: 1 3 5 7 9
}
```

### แบบฝึกหัดที่ 4: Optional กับ Linked List

เขียน Linked List ที่:
1. เพิ่มและลบ node ได้
2. ค้นหา node ที่มีค่าที่ต้องการ
3. ทุก pointer ระหว่าง node เป็น Optional

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const IntList = struct {
    const Node = struct {
        value: i32,
        next: ?*Node,
    };

    head: ?*Node,
    allocator: Allocator,

    pub fn init(allocator: Allocator) IntList {
        return IntList{ .head = null, .allocator = allocator };
    }

    pub fn prepend(self: *IntList, value: i32) !void {
        const node = try self.allocator.create(Node);
        node.* = Node{ .value = value, .next = self.head };
        self.head = node;
    }

    pub fn find(self: *const IntList, value: i32) ?*Node {
        var current = self.head;
        while (current) |node| {
            if (node.value == value) return node;
            current = node.next;
        }
        return null;
    }

    pub fn print(self: *const IntList) void {
        var current = self.head;
        while (current) |node| {
            std.debug.print("{} ", .{node.value});
            current = node.next;
        }
        std.debug.print("\n", .{});
    }

    pub fn deinit(self: *IntList) void {
        var current = self.head;
        while (current) |node| {
            const next = node.next;
            self.allocator.destroy(node);
            current = next;
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var list = IntList.init(gpa.allocator());
    defer list.deinit();

    try list.prepend(3);
    try list.prepend(2);
    try list.prepend(1);

    list.print(); // 1 2 3

    if (list.find(2)) |node| {
        std.debug.print("พบ node ที่มีค่า: {}\n", .{node.value});
    }

    if (list.find(99)) |_| {
        std.debug.print("พบ 99\n", .{});
    } else {
        std.debug.print("ไม่พบ 99\n", .{});
    }
}
```

### แบบฝึกหัดที่ 5: Optional กับ Error Handling

สร้าง function ที่รวม Optional และ Error Union:

```zig
const std = @import("std");

const ParseError = error{
    EmptyInput,
    InvalidFormat,
    OutOfRange,
};

// แปลง "x,y" เป็น struct {x: f32, y: f32}
// คืน null ถ้า input ว่าง
// คืน error ถ้า format ผิด
fn parsePoint(input: []const u8) ParseError!?struct { x: f32, y: f32 } {
    if (input.len == 0) return null;

    const comma = std.mem.indexOfScalar(u8, input, ',') orelse
        return ParseError.InvalidFormat;

    const x = std.fmt.parseFloat(f32, input[0..comma]) catch
        return ParseError.InvalidFormat;
    const y = std.fmt.parseFloat(f32, input[comma + 1 ..]) catch
        return ParseError.InvalidFormat;

    if (x < -1000 or x > 1000 or y < -1000 or y > 1000)
        return ParseError.OutOfRange;

    return .{ .x = x, .y = y };
}

pub fn main() void {
    const inputs = [_][]const u8{ "3.14,2.72", "", "invalid", "1,2", "9999,1" };

    for (inputs) |input| {
        if (parsePoint(input)) |maybe_point| {
            if (maybe_point) |point| {
                std.debug.print("'{s}' -> ({d:.2}, {d:.2})\n", .{ input, point.x, point.y });
            } else {
                std.debug.print("'{s}' -> (ว่าง)\n", .{input});
            }
        } else |err| {
            std.debug.print("'{s}' -> Error: {}\n", .{ input, err });
        }
    }
}
```

---

## Milestone: จบส่วนพื้นฐานของ Zig

ยินดีด้วย! คุณได้เรียนรู้ครบทุกพื้นฐานของ Zig แล้ว:

### สิ่งที่เรียนรู้ใน Part 1-20:

| Part | หัวข้อ |
|------|--------|
| 1-2  | ติดตั้งและ Hello World |
| 3-5  | Variables, Types, Operators |
| 6-8  | Control Flow (if, switch, while, for) |
| 9-11 | Functions, Recursion |
| 12-14 | Arrays, Slices, Strings |
| 15-16 | Structs, Methods |
| 17   | Enums |
| 18   | Unions |
| 19   | Error Handling |
| **20** | **Optionals** |

### ก้าวต่อไป (Advanced Topics):

1. **Memory Management** - Allocators, Arena, heap allocation
2. **Pointers** - pointer arithmetic, multi-pointer
3. **Comptime** - compile-time computation
4. **Generics** - generic functions and structs
5. **Async/Await** - concurrent programming
6. **C Interop** - เรียกใช้ library จาก C
7. **Build System** - `build.zig`
8. **Testing** - unit tests ใน Zig
9. **Standard Library** - std.mem, std.fs, std.net

### สรุปหลักการสำคัญของ Optional:

```zig
// 1. ใช้ ?T แทน sentinel values (-1, "", NULL)
// 2. Compiler บังคับให้ handle null
// 3. orelse สำหรับ default value
// 4. if capture สำหรับการใช้ค่า
// 5. while capture สำหรับ iterators
// 6. ?*T มีขนาดเท่า *T (zero-cost)
// 7. ใช้ Optional เมื่อ "ไม่มีค่า" เป็นเรื่องปกติ
// 8. ใช้ Error Union เมื่อต้องการบอกสาเหตุของความล้มเหลว
```

---

## บันทึกเพิ่มเติม

### ข้อควรระวัง

1. **อย่าใช้ .? โดยไม่แน่ใจ** - ถ้า Optional เป็น null จะ panic ที่ runtime
2. **Pyramid of doom** - ระวัง nested if สำหรับ Optional หลายชั้น ใช้ early return แทน
3. **?*T vs *?T** - แตกต่างกัน: `?*T` คือ Optional pointer, `*?T` คือ pointer ไปยัง Optional

### ความแตกต่างระหว่าง Optional และ Error Union

```zig
// Optional: ไม่มีค่าเป็นเรื่องปกติ
fn findUser(id: u32) ?User { ... }

// Error Union: บางอย่างผิดพลาด
fn loadUser(id: u32) !User { ... }

// ใช้ทั้งคู่: บางอย่างผิดพลาด และอาจไม่มีค่า
fn queryUser(id: u32) !?User { ... }
```

### Performance Tips

- `?*T` มี overhead เป็นศูนย์ (same size as `*T`)
- `?T` สำหรับ value types มี overhead เล็กน้อย (ต้องการ bit พิเศษ)
- ในกรณีที่ performance สำคัญมาก อาจใช้ sentinel value แทน แต่ต้องระวังความปลอดภัย

---

*จบ Part 20: Optionals ใน Zig*

*ต่อไป: Part 21 - Memory Management และ Allocators*
