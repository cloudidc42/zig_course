# ส่วนที่ 25: Interfaces ด้วย Comptime ใน Zig

## Steps 241-250: การสร้าง Interface Patterns ใน Zig

---

## Step 241: Interfaces ใน Zig - ไม่มี keyword interface

### ทำไม Zig ถึงไม่มี `interface` keyword?

Zig เป็นภาษาที่เน้นความเรียบง่ายและ explicit ในการทำงาน ภาษาอื่นอย่าง Go, Java, C# มี `interface` keyword แต่ Zig เลือกที่จะไม่มี เพราะเชื่อว่าโปรแกรมเมอร์ควรเลือก pattern ที่เหมาะสมกับ use case ของตัวเอง

ใน Zig เราสามารถสร้าง interface ได้หลายวิธี:
1. **Duck typing ด้วย `anytype`** - ตรวจสอบ interface ตอน compile time
2. **Comptime interface checking** - ตรวจสอบ type ว่ามี methods ที่ต้องการ
3. **vtable pattern** - dynamic dispatch เหมือน C++ virtual functions
4. **Fat pointer pattern** - pointer + vtable รวมกัน
5. **Tagged union** - polymorphism ด้วย union

### ตัวอย่างเปรียบเทียบกับ Go

ใน Go เราเขียน interface แบบนี้:
```go
// Go interface
type Animal interface {
    Sound() string
    Name() string
}

type Dog struct {
    name string
}

func (d Dog) Sound() string { return "Woof!" }
func (d Dog) Name() string { return d.name }

func makeNoise(a Animal) {
    fmt.Printf("%s says: %s\n", a.Name(), a.Sound())
}
```

ใน Zig เราทำสิ่งเดียวกันได้หลายวิธี ขึ้นอยู่กับความต้องการ:

```zig
const std = @import("std");

// วิธีที่ 1: Duck typing ด้วย anytype
const Dog = struct {
    name: []const u8,

    pub fn sound(self: Dog) []const u8 {
        return "Woof!";
    }

    pub fn name(self: Dog) []const u8 {
        return self.name;
    }
};

const Cat = struct {
    name: []const u8,

    pub fn sound(self: Cat) []const u8 {
        return "Meow!";
    }

    pub fn name(self: Cat) []const u8 {
        return self.name;
    }
};

// ฟังก์ชันนี้รับ anytype - ทำงานกับ type ใดก็ได้ที่มี method sound() และ name()
fn makeNoise(animal: anytype) void {
    std.debug.print("{s} says: {s}\n", .{ animal.name(), animal.sound() });
}

pub fn main() void {
    const dog = Dog{ .name = "Rex" };
    const cat = Cat{ .name = "Whiskers" };

    makeNoise(dog);
    makeNoise(cat);
}
```

### ความแตกต่างจาก OOP interfaces

| ภาษา | วิธี | ตรวจสอบตอน |
|------|------|-------------|
| Java | `interface` keyword + `implements` | Compile time (explicit) |
| Go | Structural typing | Compile time (implicit) |
| C++ | Virtual functions | Runtime |
| Zig `anytype` | Duck typing | Compile time |
| Zig vtable | Function pointers | Runtime |

---

## Step 242: Duck Typing ด้วย `anytype`

### แนวคิด Duck Typing

"ถ้ามันเดินแบบเป็ด และร้องแบบเป็ด มันก็คือเป็ด"

ใน Zig การใช้ `anytype` ทำให้ compiler ตรวจสอบ interface ตอน compile time โดยอัตโนมัติ ถ้า type ไม่มี method ที่ต้องการ จะเกิด compile error

```zig
const std = @import("std");

// Writer interface แบบ duck typing
fn writeMessage(writer: anytype, message: []const u8) !void {
    try writer.writeAll(message);
    try writer.writeAll("\n");
}

// Serializable interface แบบ duck typing
fn serialize(value: anytype, writer: anytype) !void {
    try value.serialize(writer);
}

// ตัวอย่าง: Point struct ที่ implement serialize
const Point = struct {
    x: f64,
    y: f64,

    pub fn serialize(self: Point, writer: anytype) !void {
        try writer.print("Point({d}, {d})", .{ self.x, self.y });
    }
};

// ตัวอย่าง: Circle struct ที่ implement serialize
const Circle = struct {
    center: Point,
    radius: f64,

    pub fn serialize(self: Circle, writer: anytype) !void {
        try writer.print("Circle(center=", .{});
        try self.center.serialize(writer);
        try writer.print(", radius={d})", .{self.radius});
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    const p = Point{ .x = 3.0, .y = 4.0 };
    const c = Circle{
        .center = Point{ .x = 0.0, .y = 0.0 },
        .radius = 5.0,
    };

    try serialize(p, stdout);
    try stdout.writeAll("\n");
    try serialize(c, stdout);
    try stdout.writeAll("\n");
}
```

### การทำงานของ anytype

เมื่อ compiler เจอ `anytype` มันจะสร้าง function ใหม่สำหรับแต่ละ type ที่ใช้ (monomorphization) นี่คือสิ่งที่เกิดขึ้นเบื้องหลัง:

```zig
// โค้ดที่เราเขียน
fn printValue(value: anytype) void {
    std.debug.print("{any}\n", .{value});
}

// Compiler สร้าง (conceptually):
fn printValue_i32(value: i32) void {
    std.debug.print("{any}\n", .{value});
}

fn printValue_f64(value: f64) void {
    std.debug.print("{any}\n", .{value});
}

fn printValue_Point(value: Point) void {
    std.debug.print("{any}\n", .{value});
}
```

### ข้อดีและข้อเสียของ anytype

**ข้อดี:**
- Zero runtime overhead - ไม่มี virtual function call
- Compiler error ที่ชัดเจน
- ยืดหยุ่นสูง ไม่ต้อง declare interface ล่วงหน้า

**ข้อเสีย:**
- ไม่สามารถเก็บ collection ของ types ต่างๆ ใน runtime
- Code bloat ถ้าใช้กับหลาย types
- Error messages อาจซับซ้อน

```zig
const std = @import("std");

// ตัวอย่างการใช้ anytype กับ constraints
fn add(a: anytype, b: anytype) @TypeOf(a) {
    // ตรวจสอบว่า a และ b เป็น numeric type
    comptime {
        const T = @TypeOf(a);
        const info = @typeInfo(T);
        switch (info) {
            .Int, .Float => {},
            else => @compileError("add() ต้องการ numeric type เท่านั้น"),
        }
    }
    return a + b;
}

// ตัวอย่างที่ดีกว่า: ใช้ function ที่ generic กว่า
fn sum(comptime T: type, items: []const T) T {
    var total: T = 0;
    for (items) |item| {
        total += item;
    }
    return total;
}

pub fn main() void {
    // add() ทำงานกับ int และ float
    const int_result = add(3, 4);
    const float_result = add(3.14, 2.86);

    std.debug.print("3 + 4 = {d}\n", .{int_result});
    std.debug.print("3.14 + 2.86 = {d}\n", .{float_result});

    // sum() กับ arrays
    const ints = [_]i32{ 1, 2, 3, 4, 5 };
    const floats = [_]f64{ 1.1, 2.2, 3.3 };

    std.debug.print("sum of ints: {d}\n", .{sum(i32, &ints)});
    std.debug.print("sum of floats: {d:.1}\n", .{sum(f64, &floats)});
}
```

---

## Step 243: Comptime Interface Checking

### การตรวจสอบ Interface ตอน Compile Time

เราสามารถเขียน comptime code เพื่อตรวจสอบว่า type มี methods ที่ต้องการ และให้ error message ที่ดีขึ้น

```zig
const std = @import("std");

// ฟังก์ชัน comptime สำหรับตรวจสอบ interface
fn checkInterface(comptime T: type, comptime Interface: type) void {
    _ = Interface; // suppress unused warning
    const interface_info = @typeInfo(Interface);
    _ = interface_info;
    // ในทางปฏิบัติ เราจะตรวจสอบ methods ที่ต้องการ
}

// ฟังก์ชัน helper สำหรับตรวจสอบว่า type มี method หรือไม่
fn hasMethod(comptime T: type, comptime method_name: []const u8) bool {
    return @hasDecl(T, method_name);
}

// ตรวจสอบ Animal interface
fn checkAnimalInterface(comptime T: type) void {
    if (!hasMethod(T, "sound")) {
        @compileError(@typeName(T) ++ " ต้องมี method sound() []const u8");
    }
    if (!hasMethod(T, "getName")) {
        @compileError(@typeName(T) ++ " ต้องมี method getName() []const u8");
    }
    if (!hasMethod(T, "move")) {
        @compileError(@typeName(T) ++ " ต้องมี method move() void");
    }
}

// ฟังก์ชัน interface สำหรับ Animal
fn makeAnimalNoise(animal: anytype) void {
    // ตรวจสอบ interface ตอน comptime
    comptime checkAnimalInterface(@TypeOf(animal));

    std.debug.print("{s} ({s}): {s}\n", .{
        animal.getName(),
        @typeName(@TypeOf(animal)),
        animal.sound(),
    });
    animal.move();
}

// Implementations
const Dog = struct {
    name: []const u8,

    pub fn sound(self: Dog) []const u8 {
        _ = self;
        return "Woof!";
    }

    pub fn getName(self: Dog) []const u8 {
        return self.name;
    }

    pub fn move(self: Dog) void {
        std.debug.print("{s} วิ่งไปข้างหน้า\n", .{self.name});
    }
};

const Bird = struct {
    name: []const u8,

    pub fn sound(self: Bird) []const u8 {
        _ = self;
        return "Tweet!";
    }

    pub fn getName(self: Bird) []const u8 {
        return self.name;
    }

    pub fn move(self: Bird) void {
        std.debug.print("{s} บินขึ้นไปบนฟ้า\n", .{self.name});
    }
};

// Type ที่ไม่ implement interface (จะเกิด compile error ถ้า uncomment)
// const Rock = struct {
//     pub fn getName(self: Rock) []const u8 { return "หิน"; }
//     // ไม่มี sound() และ move()
// };

pub fn main() void {
    const rex = Dog{ .name = "Rex" };
    const tweety = Bird{ .name = "Tweety" };

    makeAnimalNoise(rex);
    makeAnimalNoise(tweety);

    // makeAnimalNoise(Rock{}); // จะเกิด compile error
}
```

### Interface Checking แบบ Advanced

```zig
const std = @import("std");

// ตรวจสอบ type signature ด้วย
fn checkHashableInterface(comptime T: type) void {
    // ตรวจสอบว่ามี hash() method
    if (!@hasDecl(T, "hash")) {
        @compileError(@typeName(T) ++ " ต้องมี method hash(self: @This()) u64");
    }

    // ตรวจสอบ return type ของ hash()
    const hash_fn = @field(T, "hash");
    const hash_fn_info = @typeInfo(@TypeOf(hash_fn));

    switch (hash_fn_info) {
        .Fn => |fn_info| {
            if (fn_info.return_type) |ret| {
                if (ret != u64) {
                    @compileError(@typeName(T) ++ ".hash() ต้อง return u64");
                }
            }
        },
        else => @compileError(@typeName(T) ++ ".hash ต้องเป็น function"),
    }

    // ตรวจสอบว่ามี eql() method
    if (!@hasDecl(T, "eql")) {
        @compileError(@typeName(T) ++ " ต้องมี method eql(self: @This(), other: @This()) bool");
    }
}

// ใช้ interface checking
fn hashSet(comptime T: type) type {
    comptime checkHashableInterface(T);

    return struct {
        items: std.ArrayList(T),
        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{
                .items = std.ArrayList(T).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.items.deinit();
        }

        pub fn add(self: *Self, item: T) !bool {
            // ตรวจสอบว่ามีอยู่แล้วหรือไม่
            for (self.items.items) |existing| {
                if (existing.eql(item)) return false;
            }
            try self.items.append(item);
            return true;
        }

        pub fn contains(self: Self, item: T) bool {
            for (self.items.items) |existing| {
                if (existing.eql(item)) return true;
            }
            return false;
        }
    };
}

// Type ที่ implement Hashable interface
const StringKey = struct {
    value: []const u8,

    pub fn hash(self: StringKey) u64 {
        var h: u64 = 5381;
        for (self.value) |c| {
            h = ((h << 5) +% h) +% c;
        }
        return h;
    }

    pub fn eql(self: StringKey, other: StringKey) bool {
        return std.mem.eql(u8, self.value, other.value);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const StringSet = hashSet(StringKey);
    var set = StringSet.init(allocator);
    defer set.deinit();

    _ = try set.add(StringKey{ .value = "apple" });
    _ = try set.add(StringKey{ .value = "banana" });
    _ = try set.add(StringKey{ .value = "apple" }); // duplicate

    std.debug.print("มี 'apple': {}\n", .{set.contains(StringKey{ .value = "apple" })});
    std.debug.print("มี 'cherry': {}\n", .{set.contains(StringKey{ .value = "cherry" })});
    std.debug.print("จำนวน items: {d}\n", .{set.items.items.len});
}
```

---

## Step 244: VTable Pattern (Dynamic Dispatch)

### VTable คืออะไร?

VTable (Virtual Function Table) คือ pattern ที่ใช้ function pointers เพื่อทำ dynamic dispatch - เราสามารถเก็บ objects ของ types ต่างๆ ใน collection เดียวกัน และเรียก methods ผ่าน function pointers

เปรียบเทียบกับ C++:
```cpp
// C++ virtual functions
class Animal {
public:
    virtual std::string sound() = 0;
    virtual std::string name() = 0;
};
```

ใน Zig เราต้องสร้าง VTable เอง:

```zig
const std = @import("std");

// VTable สำหรับ Animal interface
const AnimalVTable = struct {
    soundFn: *const fn (ctx: *anyopaque) []const u8,
    nameFn: *const fn (ctx: *anyopaque) []const u8,
    moveFn: *const fn (ctx: *anyopaque) void,
    deinitFn: *const fn (ctx: *anyopaque, allocator: std.mem.Allocator) void,
};

// Animal interface (ptr + vtable)
const Animal = struct {
    ctx: *anyopaque,
    vtable: *const AnimalVTable,

    pub fn sound(self: Animal) []const u8 {
        return self.vtable.soundFn(self.ctx);
    }

    pub fn name(self: Animal) []const u8 {
        return self.vtable.nameFn(self.ctx);
    }

    pub fn move(self: Animal) void {
        self.vtable.moveFn(self.ctx);
    }

    pub fn deinit(self: Animal, allocator: std.mem.Allocator) void {
        self.vtable.deinitFn(self.ctx, allocator);
    }
};

// Helper function สำหรับสร้าง Animal จาก concrete type
fn makeAnimal(comptime T: type, allocator: std.mem.Allocator, value: T) !Animal {
    // สร้าง VTable สำหรับ type T
    const vtable = comptime &AnimalVTable{
        .soundFn = struct {
            fn sound(ctx: *anyopaque) []const u8 {
                const self: *T = @ptrCast(@alignCast(ctx));
                return self.sound();
            }
        }.sound,
        .nameFn = struct {
            fn getName(ctx: *anyopaque) []const u8 {
                const self: *T = @ptrCast(@alignCast(ctx));
                return self.getName();
            }
        }.getName,
        .moveFn = struct {
            fn move(ctx: *anyopaque) void {
                const self: *T = @ptrCast(@alignCast(ctx));
                self.move();
            }
        }.move,
        .deinitFn = struct {
            fn deinit(ctx: *anyopaque, alloc: std.mem.Allocator) void {
                const self: *T = @ptrCast(@alignCast(ctx));
                alloc.destroy(self);
            }
        }.deinit,
    };

    // Allocate และ copy value
    const ptr = try allocator.create(T);
    ptr.* = value;

    return Animal{
        .ctx = ptr,
        .vtable = vtable,
    };
}

// Concrete implementations
const Dog = struct {
    dog_name: []const u8,

    pub fn sound(self: *Dog) []const u8 {
        _ = self;
        return "Woof!";
    }

    pub fn getName(self: *Dog) []const u8 {
        return self.dog_name;
    }

    pub fn move(self: *Dog) void {
        std.debug.print("{s} วิ่งบนพื้น\n", .{self.dog_name});
    }
};

const Fish = struct {
    fish_name: []const u8,

    pub fn sound(self: *Fish) []const u8 {
        _ = self;
        return "...";
    }

    pub fn getName(self: *Fish) []const u8 {
        return self.fish_name;
    }

    pub fn move(self: *Fish) void {
        std.debug.print("{s} ว่ายน้ำ\n", .{self.fish_name});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง animals array (dynamic dispatch!)
    var animals = std.ArrayList(Animal).init(allocator);
    defer animals.deinit();

    // เพิ่ม different types ลงใน array เดียวกัน
    try animals.append(try makeAnimal(Dog, allocator, Dog{ .dog_name = "Rex" }));
    try animals.append(try makeAnimal(Fish, allocator, Fish{ .fish_name = "Nemo" }));
    try animals.append(try makeAnimal(Dog, allocator, Dog{ .dog_name = "Buddy" }));

    // เรียกใช้ผ่าน interface
    std.debug.print("=== สัตว์ทั้งหมด ===\n", .{});
    for (animals.items) |animal| {
        std.debug.print("{s}: {s}\n", .{ animal.name(), animal.sound() });
        animal.move();
        std.debug.print("\n", .{});
    }

    // Cleanup
    for (animals.items) |animal| {
        animal.deinit(allocator);
    }
}
```

### VTable Pattern ที่ใช้ใน Standard Library

Standard library ของ Zig ใช้ pattern คล้ายกันนี้สำหรับ `std.io.Writer` และ `std.io.Reader`:

```zig
// ดูโครงสร้างของ std.io.GenericWriter
// (simplified version)
pub fn GenericWriter(
    comptime Context: type,
    comptime WriteError: type,
    comptime writeFn: fn (context: Context, bytes: []const u8) WriteError!usize,
) type {
    return struct {
        context: Context,

        const Self = @This();
        pub const Error = WriteError;

        pub fn write(self: Self, bytes: []const u8) Error!usize {
            return writeFn(self.context, bytes);
        }

        pub fn writeAll(self: Self, bytes: []const u8) Error!void {
            var index: usize = 0;
            while (index != bytes.len) {
                index += try self.write(bytes[index..]);
            }
        }

        pub fn print(self: Self, comptime format: []const u8, args: anytype) Error!void {
            return std.fmt.format(self, format, args);
        }
    };
}
```

---

## Step 245: Fat Pointer Pattern (ptr + vtable)

### Fat Pointer คืออะไร?

Fat pointer คือการรวม pointer ไปยัง data กับ pointer ไปยัง vtable ไว้ด้วยกัน นี่คือ pattern ที่ Rust ใช้สำหรับ trait objects (`dyn Trait`)

```zig
const std = @import("std");

// Fat pointer สำหรับ Drawable interface
const Drawable = struct {
    // ptr ไปยัง data
    ptr: *anyopaque,
    // ptr ไปยัง vtable
    vtable: *const VTable,

    const VTable = struct {
        draw: *const fn (ptr: *anyopaque) void,
        boundingBox: *const fn (ptr: *anyopaque) BoundingBox,
        area: *const fn (ptr: *anyopaque) f64,
    };

    const BoundingBox = struct {
        x: f64,
        y: f64,
        width: f64,
        height: f64,
    };

    // Method dispatch ผ่าน vtable
    pub fn draw(self: Drawable) void {
        self.vtable.draw(self.ptr);
    }

    pub fn boundingBox(self: Drawable) BoundingBox {
        return self.vtable.boundingBox(self.ptr);
    }

    pub fn area(self: Drawable) f64 {
        return self.vtable.area(self.ptr);
    }

    // Type-safe constructor helper
    pub fn init(pointer: anytype) Drawable {
        const T = @TypeOf(pointer);
        const ptr_info = @typeInfo(T);

        if (ptr_info != .Pointer) @compileError("ต้องส่ง pointer");
        const Child = ptr_info.Pointer.child;

        // สร้าง vtable สำหรับ Child type
        const gen = struct {
            fn draw(p: *anyopaque) void {
                const self: *Child = @ptrCast(@alignCast(p));
                self.draw();
            }

            fn boundingBox(p: *anyopaque) BoundingBox {
                const self: *Child = @ptrCast(@alignCast(p));
                return self.boundingBox();
            }

            fn area(p: *anyopaque) f64 {
                const self: *Child = @ptrCast(@alignCast(p));
                return self.area();
            }

            const vtable = VTable{
                .draw = draw,
                .boundingBox = boundingBox,
                .area = area,
            };
        };

        return .{
            .ptr = pointer,
            .vtable = &gen.vtable,
        };
    }
};

// Concrete shapes
const Rectangle = struct {
    x: f64,
    y: f64,
    width: f64,
    height: f64,

    pub fn draw(self: *Rectangle) void {
        std.debug.print("วาด Rectangle ที่ ({d:.1}, {d:.1}) ขนาด {d:.1}x{d:.1}\n", .{
            self.x, self.y, self.width, self.height,
        });
    }

    pub fn boundingBox(self: *Rectangle) Drawable.BoundingBox {
        return .{
            .x = self.x,
            .y = self.y,
            .width = self.width,
            .height = self.height,
        };
    }

    pub fn area(self: *Rectangle) f64 {
        return self.width * self.height;
    }
};

const Circle = struct {
    cx: f64,
    cy: f64,
    radius: f64,

    pub fn draw(self: *Circle) void {
        std.debug.print("วาด Circle ที่ ({d:.1}, {d:.1}) รัศมี {d:.1}\n", .{
            self.cx, self.cy, self.radius,
        });
    }

    pub fn boundingBox(self: *Circle) Drawable.BoundingBox {
        return .{
            .x = self.cx - self.radius,
            .y = self.cy - self.radius,
            .width = self.radius * 2,
            .height = self.radius * 2,
        };
    }

    pub fn area(self: *Circle) f64 {
        return std.math.pi * self.radius * self.radius;
    }
};

const Triangle = struct {
    x1: f64,
    y1: f64,
    x2: f64,
    y2: f64,
    x3: f64,
    y3: f64,

    pub fn draw(self: *Triangle) void {
        std.debug.print("วาด Triangle ({d:.0},{d:.0})-({d:.0},{d:.0})-({d:.0},{d:.0})\n", .{
            self.x1, self.y1,
            self.x2, self.y2,
            self.x3, self.y3,
        });
    }

    pub fn boundingBox(self: *Triangle) Drawable.BoundingBox {
        const min_x = @min(self.x1, @min(self.x2, self.x3));
        const min_y = @min(self.y1, @min(self.y2, self.y3));
        const max_x = @max(self.x1, @max(self.x2, self.x3));
        const max_y = @max(self.y1, @max(self.y2, self.y3));
        return .{
            .x = min_x,
            .y = min_y,
            .width = max_x - min_x,
            .height = max_y - min_y,
        };
    }

    pub fn area(self: *Triangle) f64 {
        // Heron's formula
        const a = std.math.sqrt((self.x2 - self.x1) * (self.x2 - self.x1) +
            (self.y2 - self.y1) * (self.y2 - self.y1));
        const b = std.math.sqrt((self.x3 - self.x2) * (self.x3 - self.x2) +
            (self.y3 - self.y2) * (self.y3 - self.y2));
        const c = std.math.sqrt((self.x1 - self.x3) * (self.x1 - self.x3) +
            (self.y1 - self.y3) * (self.y1 - self.y3));
        const s = (a + b + c) / 2.0;
        return std.math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
};

// Drawing canvas ที่ใช้ Drawable interface
const Canvas = struct {
    shapes: std.ArrayList(Drawable),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Canvas {
        return Canvas{
            .shapes = std.ArrayList(Drawable).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Canvas) void {
        self.shapes.deinit();
    }

    pub fn addShape(self: *Canvas, shape: Drawable) !void {
        try self.shapes.append(shape);
    }

    pub fn drawAll(self: Canvas) void {
        std.debug.print("=== วาด Canvas ===\n", .{});
        for (self.shapes.items) |shape| {
            shape.draw();
            const bb = shape.boundingBox();
            std.debug.print("  Bounding box: ({d:.1}, {d:.1}) {d:.1}x{d:.1}\n", .{
                bb.x, bb.y, bb.width, bb.height,
            });
            std.debug.print("  พื้นที่: {d:.2}\n", .{shape.area()});
        }
    }

    pub fn totalArea(self: Canvas) f64 {
        var total: f64 = 0;
        for (self.shapes.items) |shape| {
            total += shape.area();
        }
        return total;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var canvas = Canvas.init(allocator);
    defer canvas.deinit();

    // สร้าง shapes บน stack
    var rect = Rectangle{ .x = 0, .y = 0, .width = 10, .height = 5 };
    var circle = Circle{ .cx = 5, .cy = 5, .radius = 3 };
    var triangle = Triangle{
        .x1 = 0, .y1 = 0,
        .x2 = 4, .y2 = 0,
        .x3 = 2, .y3 = 3,
    };

    // เพิ่ม shapes ผ่าน Drawable interface
    try canvas.addShape(Drawable.init(&rect));
    try canvas.addShape(Drawable.init(&circle));
    try canvas.addShape(Drawable.init(&triangle));

    canvas.drawAll();
    std.debug.print("\nพื้นที่รวมทั้งหมด: {d:.2}\n", .{canvas.totalArea()});
}
```

### เปรียบเทียบกับ Rust trait objects

```rust
// Rust trait object (conceptually similar)
trait Drawable {
    fn draw(&self);
    fn area(&self) -> f64;
}

fn draw_all(shapes: &[Box<dyn Drawable>]) {
    for shape in shapes {
        shape.draw();
        println!("Area: {}", shape.area());
    }
}
```

ใน Zig เราต้องสร้าง VTable เอง แต่ได้ความยืดหยุ่นมากกว่า และเห็นชัดว่าเกิดอะไรขึ้น

---

## Step 246: Interface ด้วย Tagged Union

### Tagged Union เป็น Interface

Tagged union เป็นอีกวิธีหนึ่งในการทำ polymorphism ใน Zig เหมาะเมื่อเรารู้ล่วงหน้าว่ามี types กี่อย่าง

```zig
const std = @import("std");

// Shape interface ด้วย tagged union
const Shape = union(enum) {
    rectangle: Rectangle,
    circle: Circle,
    triangle: Triangle,
    line: Line,

    const Rectangle = struct {
        x: f64,
        y: f64,
        width: f64,
        height: f64,
    };

    const Circle = struct {
        cx: f64,
        cy: f64,
        radius: f64,
    };

    const Triangle = struct {
        x1: f64,
        y1: f64,
        x2: f64,
        y2: f64,
        x3: f64,
        y3: f64,
    };

    const Line = struct {
        x1: f64,
        y1: f64,
        x2: f64,
        y2: f64,
    };

    // Methods ที่ทำงานกับทุก variants
    pub fn area(self: Shape) f64 {
        return switch (self) {
            .rectangle => |r| r.width * r.height,
            .circle => |c| std.math.pi * c.radius * c.radius,
            .triangle => |t| blk: {
                const a = std.math.sqrt((t.x2 - t.x1) * (t.x2 - t.x1) + (t.y2 - t.y1) * (t.y2 - t.y1));
                const b = std.math.sqrt((t.x3 - t.x2) * (t.x3 - t.x2) + (t.y3 - t.y2) * (t.y3 - t.y2));
                const c = std.math.sqrt((t.x1 - t.x3) * (t.x1 - t.x3) + (t.y1 - t.y3) * (t.y1 - t.y3));
                const s = (a + b + c) / 2.0;
                break :blk std.math.sqrt(s * (s - a) * (s - b) * (s - c));
            },
            .line => 0.0, // เส้นไม่มีพื้นที่
        };
    }

    pub fn perimeter(self: Shape) f64 {
        return switch (self) {
            .rectangle => |r| 2 * (r.width + r.height),
            .circle => |c| 2 * std.math.pi * c.radius,
            .triangle => |t| blk: {
                const a = std.math.sqrt((t.x2 - t.x1) * (t.x2 - t.x1) + (t.y2 - t.y1) * (t.y2 - t.y1));
                const b = std.math.sqrt((t.x3 - t.x2) * (t.x3 - t.x2) + (t.y3 - t.y2) * (t.y3 - t.y2));
                const c = std.math.sqrt((t.x1 - t.x3) * (t.x1 - t.x3) + (t.y1 - t.y3) * (t.y1 - t.y3));
                break :blk a + b + c;
            },
            .line => |l| std.math.sqrt((l.x2 - l.x1) * (l.x2 - l.x1) + (l.y2 - l.y1) * (l.y2 - l.y1)),
        };
    }

    pub fn describe(self: Shape) void {
        switch (self) {
            .rectangle => |r| std.debug.print("สี่เหลี่ยม: {d:.1}x{d:.1} ที่ ({d:.1},{d:.1})", .{
                r.width, r.height, r.x, r.y,
            }),
            .circle => |c| std.debug.print("วงกลม: รัศมี {d:.1} ที่ ({d:.1},{d:.1})", .{
                c.radius, c.cx, c.cy,
            }),
            .triangle => std.debug.print("สามเหลี่ยม", .{}),
            .line => std.debug.print("เส้นตรง", .{}),
        }
        std.debug.print(" | พื้นที่: {d:.2} | เส้นรอบ: {d:.2}\n", .{
            self.area(), self.perimeter(),
        });
    }

    // Scale shape
    pub fn scale(self: Shape, factor: f64) Shape {
        return switch (self) {
            .rectangle => |r| Shape{
                .rectangle = .{
                    .x = r.x,
                    .y = r.y,
                    .width = r.width * factor,
                    .height = r.height * factor,
                },
            },
            .circle => |c| Shape{
                .circle = .{
                    .cx = c.cx,
                    .cy = c.cy,
                    .radius = c.radius * factor,
                },
            },
            .triangle => |t| Shape{
                .triangle = .{
                    .x1 = t.x1 * factor,
                    .y1 = t.y1 * factor,
                    .x2 = t.x2 * factor,
                    .y2 = t.y2 * factor,
                    .x3 = t.x3 * factor,
                    .y3 = t.y3 * factor,
                },
            },
            .line => |l| Shape{
                .line = .{
                    .x1 = l.x1 * factor,
                    .y1 = l.y1 * factor,
                    .x2 = l.x2 * factor,
                    .y2 = l.y2 * factor,
                },
            },
        };
    }
};

// JSON Serializer ด้วย tagged union
const JsonValue = union(enum) {
    null_value,
    bool_value: bool,
    int_value: i64,
    float_value: f64,
    string_value: []const u8,
    array_value: []const JsonValue,
    object_value: []const KeyValue,

    const KeyValue = struct {
        key: []const u8,
        value: JsonValue,
    };

    pub fn serialize(self: JsonValue, writer: anytype) !void {
        switch (self) {
            .null_value => try writer.writeAll("null"),
            .bool_value => |b| try writer.writeAll(if (b) "true" else "false"),
            .int_value => |i| try writer.print("{d}", .{i}),
            .float_value => |f| try writer.print("{d}", .{f}),
            .string_value => |s| {
                try writer.writeByte('"');
                for (s) |c| {
                    switch (c) {
                        '"' => try writer.writeAll("\\\""),
                        '\\' => try writer.writeAll("\\\\"),
                        '\n' => try writer.writeAll("\\n"),
                        '\t' => try writer.writeAll("\\t"),
                        else => try writer.writeByte(c),
                    }
                }
                try writer.writeByte('"');
            },
            .array_value => |arr| {
                try writer.writeByte('[');
                for (arr, 0..) |item, i| {
                    if (i > 0) try writer.writeAll(", ");
                    try item.serialize(writer);
                }
                try writer.writeByte(']');
            },
            .object_value => |obj| {
                try writer.writeByte('{');
                for (obj, 0..) |kv, i| {
                    if (i > 0) try writer.writeAll(", ");
                    try writer.print("\"{s}\": ", .{kv.key});
                    try kv.value.serialize(writer);
                }
                try writer.writeByte('}');
            },
        }
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    // ทดสอบ Shape interface
    const shapes = [_]Shape{
        Shape{ .rectangle = .{ .x = 0, .y = 0, .width = 5, .height = 3 } },
        Shape{ .circle = .{ .cx = 2, .cy = 2, .radius = 4 } },
        Shape{ .triangle = .{ .x1 = 0, .y1 = 0, .x2 = 3, .y2 = 0, .x3 = 1.5, .y3 = 2.6 } },
        Shape{ .line = .{ .x1 = 0, .y1 = 0, .x2 = 5, .y2 = 5 } },
    };

    std.debug.print("=== รูปทรงต่างๆ ===\n", .{});
    for (shapes) |shape| {
        shape.describe();
    }

    // Scale ทุก shape เป็น 2 เท่า
    std.debug.print("\n=== หลังจาก scale 2x ===\n", .{});
    for (shapes) |shape| {
        shape.scale(2.0).describe();
    }

    // ทดสอบ JSON serializer
    std.debug.print("\n=== JSON Serialization ===\n", .{});
    const json_data = JsonValue{
        .object_value = &[_]JsonValue.KeyValue{
            .{ .key = "name", .value = .{ .string_value = "สมชาย" } },
            .{ .key = "age", .value = .{ .int_value = 25 } },
            .{ .key = "active", .value = .{ .bool_value = true } },
            .{ .key = "score", .value = .{ .float_value = 9.8 } },
            .{ .key = "tags", .value = .{
                .array_value = &[_]JsonValue{
                    .{ .string_value = "zig" },
                    .{ .string_value = "programming" },
                },
            } },
        },
    };

    try json_data.serialize(stdout);
    try stdout.writeByte('\n');
}
```

---

## Step 247: Writer Interface (std.io.Writer)

### std.io.Writer คืออะไร?

`std.io.Writer` เป็นหนึ่งใน interface patterns ที่ใช้ใน Zig standard library สำหรับการเขียนข้อมูล

```zig
const std = @import("std");

// Custom Writer: เขียนลงใน buffer พร้อม formatting
const PrettyWriter = struct {
    buffer: *std.ArrayList(u8),
    indent_level: u32,
    at_line_start: bool,

    const Self = @This();

    pub fn init(buffer: *std.ArrayList(u8)) Self {
        return Self{
            .buffer = buffer,
            .indent_level = 0,
            .at_line_start = true,
        };
    }

    pub fn writer(self: *Self) std.io.Writer(*Self, error{OutOfMemory}, write) {
        return .{ .context = self };
    }

    fn write(self: *Self, bytes: []const u8) error{OutOfMemory}!usize {
        for (bytes) |byte| {
            if (self.at_line_start and byte != '\n') {
                // เพิ่ม indentation
                var i: u32 = 0;
                while (i < self.indent_level * 2) : (i += 1) {
                    try self.buffer.append(' ');
                }
                self.at_line_start = false;
            }
            try self.buffer.append(byte);
            if (byte == '\n') {
                self.at_line_start = true;
            }
        }
        return bytes.len;
    }

    pub fn indent(self: *Self) void {
        self.indent_level += 1;
    }

    pub fn dedent(self: *Self) void {
        if (self.indent_level > 0) self.indent_level -= 1;
    }
};

// การใช้ Writer pattern ใน function
fn writeCode(w: anytype, code: []const u8, language: []const u8) !void {
    try w.print("```{s}\n", .{language});
    try w.writeAll(code);
    try w.writeAll("\n```\n");
}

fn writeMarkdownList(w: anytype, items: []const []const u8) !void {
    for (items) |item| {
        try w.print("- {s}\n", .{item});
    }
}

fn writeMarkdownTable(w: anytype, headers: []const []const u8, rows: []const []const []const u8) !void {
    // Header row
    try w.writeByte('|');
    for (headers) |header| {
        try w.print(" {s} |", .{header});
    }
    try w.writeByte('\n');

    // Separator row
    try w.writeByte('|');
    for (headers) |_| {
        try w.writeAll(" --- |");
    }
    try w.writeByte('\n');

    // Data rows
    for (rows) |row| {
        try w.writeByte('|');
        for (row) |cell| {
            try w.print(" {s} |", .{cell});
        }
        try w.writeByte('\n');
    }
}

// Counting writer - นับจำนวน bytes ที่เขียน
const CountingWriter = struct {
    inner: anytype,
    count: usize,

    const Self = @This();

    pub fn init(inner: anytype) Self {
        return Self{
            .inner = inner,
            .count = 0,
        };
    }

    pub fn write(self: *Self, bytes: []const u8) @TypeOf(self.inner).Error!usize {
        const written = try self.inner.write(bytes);
        self.count += written;
        return written;
    }

    pub fn writer(self: *Self) std.io.Writer(*Self, @TypeOf(self.inner).Error, write) {
        return .{ .context = self };
    }
};

// Buffered writer สำหรับ performance
fn demoBufferedWriter(allocator: std.mem.Allocator) !void {
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();

    // ใช้ buffered writer
    var buffered = std.io.bufferedWriter(output.writer());
    const w = buffered.writer();

    // เขียนข้อมูล
    try w.writeAll("Hello, ");
    try w.writeAll("World!\n");
    try w.print("Pi = {d:.5}\n", .{std.math.pi});

    // ต้อง flush เพื่อให้ข้อมูลถูกเขียนจริง
    try buffered.flush();

    std.debug.print("Buffered output: {s}", .{output.items});
}

// Custom writer ที่แปลง uppercase
const UppercaseWriter = struct {
    inner: anytype,

    pub fn write(self: @This(), bytes: []const u8) @TypeOf(self.inner).Error!usize {
        for (bytes) |byte| {
            const upper = if (byte >= 'a' and byte <= 'z') byte - 32 else byte;
            _ = try self.inner.write(&[_]u8{upper});
        }
        return bytes.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ทดสอบ PrettyWriter
    std.debug.print("=== PrettyWriter ===\n", .{});
    var buffer = std.ArrayList(u8).init(allocator);
    defer buffer.deinit();

    var pretty = PrettyWriter.init(&buffer);
    const pw = pretty.writer();

    try pw.writeAll("class Foo {\n");
    pretty.indent();
    try pw.writeAll("void method() {\n");
    pretty.indent();
    try pw.writeAll("doSomething();\n");
    pretty.dedent();
    try pw.writeAll("}\n");
    pretty.dedent();
    try pw.writeAll("}\n");

    std.debug.print("{s}", .{buffer.items});

    // ทดสอบ Markdown functions
    std.debug.print("\n=== Markdown Writer ===\n", .{});
    const stdout = std.io.getStdOut().writer();

    try writeCode(stdout, "const x = 42;", "zig");

    const items = [_][]const u8{ "Zig", "Rust", "Go", "C" };
    try writeMarkdownList(stdout, &items);

    const headers = [_][]const u8{ "ภาษา", "ปี", "ใช้งาน" };
    const row1 = [_][]const u8{ "Zig", "2016", "System" };
    const row2 = [_][]const u8{ "Rust", "2015", "System" };
    const row3 = [_][]const u8{ "Go", "2009", "Backend" };
    const rows = [_][]const []const u8{ &row1, &row2, &row3 };
    try writeMarkdownTable(stdout, &headers, &rows);

    // ทดสอบ buffered writer
    std.debug.print("\n=== Buffered Writer ===\n", .{});
    try demoBufferedWriter(allocator);
}
```

---

## Step 248: Reader Interface (std.io.Reader)

### std.io.Reader Pattern

เหมือนกับ Writer แต่สำหรับการอ่านข้อมูล

```zig
const std = @import("std");

// Custom Reader: อ่านจาก string
fn stringReader(data: []const u8) std.io.FixedBufferStream([]const u8) {
    return std.io.fixedBufferStream(data);
}

// Line reader - อ่านทีละบรรทัด
fn readLine(reader: anytype, buffer: []u8) !?[]u8 {
    var idx: usize = 0;
    while (idx < buffer.len) {
        const byte = reader.readByte() catch |err| {
            if (err == error.EndOfStream) {
                if (idx == 0) return null;
                return buffer[0..idx];
            }
            return err;
        };

        if (byte == '\n') {
            return buffer[0..idx];
        }
        if (byte == '\r') continue; // skip CR

        buffer[idx] = byte;
        idx += 1;
    }
    return buffer[0..idx];
}

// CSV Parser ด้วย Reader interface
const CsvParser = struct {
    reader: anytype,
    buffer: [4096]u8,

    const Self = @This();
    const ParseError = error{
        InvalidCsv,
        BufferTooSmall,
    };

    pub fn nextLine(self: *Self) !?[][]const u8 {
        _ = self;
        // simplified implementation
        return null;
    }
};

// ตัวอย่างการใช้ Reader สำหรับ protocol parsing
fn parseHttpRequest(reader: anytype, allocator: std.mem.Allocator) !struct {
    method: []u8,
    path: []u8,
    headers: std.StringHashMap([]u8),
} {
    var line_buf: [1024]u8 = undefined;

    // อ่าน request line
    const request_line = try reader.readUntilDelimiter(&line_buf, '\n');
    const trimmed = std.mem.trimRight(u8, request_line, "\r");

    // parse method และ path
    var parts = std.mem.splitScalar(u8, trimmed, ' ');
    const method = parts.next() orelse return error.InvalidRequest;
    const path = parts.next() orelse return error.InvalidRequest;

    var headers = std.StringHashMap([]u8).init(allocator);
    errdefer headers.deinit();

    // อ่าน headers
    while (true) {
        const header_line = try reader.readUntilDelimiter(&line_buf, '\n');
        const header_trimmed = std.mem.trimRight(u8, header_line, "\r");

        if (header_trimmed.len == 0) break; // end of headers

        const colon_pos = std.mem.indexOf(u8, header_trimmed, ": ") orelse continue;
        const header_name = header_trimmed[0..colon_pos];
        const header_value = header_trimmed[colon_pos + 2 ..];

        const name_dup = try allocator.dupe(u8, header_name);
        const value_dup = try allocator.dupe(u8, header_value);
        try headers.put(name_dup, value_dup);
    }

    return .{
        .method = try allocator.dupe(u8, method),
        .path = try allocator.dupe(u8, path),
        .headers = headers,
    };
}

// Counting reader - นับจำนวน bytes ที่อ่าน
const CountingReader = struct {
    inner: anytype,
    count: usize,

    const Self = @This();

    pub fn init(inner: anytype) Self {
        return .{ .inner = inner, .count = 0 };
    }

    pub fn read(self: *Self, buffer: []u8) @TypeOf(self.inner).Error!usize {
        const n = try self.inner.read(buffer);
        self.count += n;
        return n;
    }

    pub fn reader(self: *Self) std.io.Reader(*Self, @TypeOf(self.inner).Error, read) {
        return .{ .context = self };
    }
};

// Parser ที่ใช้ Reader interface
fn parseNumbers(reader: anytype, allocator: std.mem.Allocator) ![]i64 {
    var numbers = std.ArrayList(i64).init(allocator);
    errdefer numbers.deinit();

    var buf: [64]u8 = undefined;
    var buf_stream = std.io.fixedBufferStream(&buf);

    while (true) {
        // อ่านจนกว่าจะเจอ space หรือ newline
        buf_stream.reset();
        const bw = buf_stream.writer();

        inner: while (true) {
            const byte = reader.readByte() catch |err| {
                if (err == error.EndOfStream) break :inner;
                return err;
            };

            if (byte == ' ' or byte == '\n' or byte == '\t' or byte == ',') {
                break :inner;
            }
            try bw.writeByte(byte);
        }

        const token = buf_stream.getWritten();
        if (token.len == 0) break;

        const num = std.fmt.parseInt(i64, token, 10) catch continue;
        try numbers.append(num);
    }

    return numbers.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ทดสอบ string reader
    std.debug.print("=== String Reader ===\n", .{});
    const data = "บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3\n";
    var stream = std.io.fixedBufferStream(data);
    const reader = stream.reader();

    var line_buf: [256]u8 = undefined;
    while (try readLine(reader, &line_buf)) |line| {
        std.debug.print("อ่านได้: [{s}]\n", .{line});
    }

    // ทดสอบ number parser
    std.debug.print("\n=== Number Parser ===\n", .{});
    const number_data = "1 2 3 42 100 -5 99";
    var num_stream = std.io.fixedBufferStream(number_data);
    const num_reader = num_stream.reader();

    const numbers = try parseNumbers(num_reader, allocator);
    defer allocator.free(numbers);

    std.debug.print("ตัวเลขที่อ่านได้: ", .{});
    for (numbers, 0..) |n, i| {
        if (i > 0) std.debug.print(", ", .{});
        std.debug.print("{d}", .{n});
    }
    std.debug.print("\n", .{});

    // ทดสอบ HTTP request parsing
    std.debug.print("\n=== HTTP Request Parser ===\n", .{});
    const http_request =
        "GET /api/users HTTP/1.1\r\n" ++
        "Host: example.com\r\n" ++
        "Content-Type: application/json\r\n" ++
        "Authorization: Bearer token123\r\n" ++
        "\r\n";

    var http_stream = std.io.fixedBufferStream(http_request);
    const http_reader = http_stream.reader();

    const request = try parseHttpRequest(http_reader, allocator);
    defer {
        allocator.free(request.method);
        allocator.free(request.path);
        var it = request.headers.iterator();
        while (it.next()) |entry| {
            allocator.free(entry.key_ptr.*);
            allocator.free(entry.value_ptr.*);
        }
        var headers = request.headers;
        headers.deinit();
    }

    std.debug.print("Method: {s}\n", .{request.method});
    std.debug.print("Path: {s}\n", .{request.path});
    std.debug.print("Headers:\n", .{});
    var it = request.headers.iterator();
    while (it.next()) |entry| {
        std.debug.print("  {s}: {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
}
```

---

## Step 249: Iterator Interface Pattern

### Iterator Pattern ใน Zig

Zig ใช้ pattern ของ iterator อย่างแพร่หลาย โดย convention คือการมี `next()` method ที่ return `?T`

```zig
const std = @import("std");

// Iterator interface ด้วย comptime
fn isIterator(comptime T: type) bool {
    // ตรวจสอบว่ามี next() method
    if (!@hasDecl(T, "next")) return false;

    // ตรวจสอบ return type ของ next()
    const next_fn = @TypeOf(@field(T, "next"));
    const fn_info = @typeInfo(next_fn);

    if (fn_info != .Fn) return false;

    // next() ต้อง return optional type
    if (fn_info.Fn.return_type) |ret| {
        const ret_info = @typeInfo(ret);
        return ret_info == .Optional;
    }

    return false;
}

// Range iterator - สร้าง sequence ของตัวเลข
fn Range(comptime T: type) type {
    return struct {
        current: T,
        end: T,
        step: T,

        const Self = @This();

        pub fn init(start: T, end: T, step: T) Self {
            return Self{
                .current = start,
                .end = end,
                .step = step,
            };
        }

        pub fn next(self: *Self) ?T {
            if (self.step > 0 and self.current >= self.end) return null;
            if (self.step < 0 and self.current <= self.end) return null;

            const val = self.current;
            self.current += self.step;
            return val;
        }
    };
}

// Map iterator - แปลงค่าใน iterator
fn MapIterator(comptime Iter: type, comptime T: type, comptime U: type) type {
    return struct {
        iter: Iter,
        transform: *const fn (T) U,

        const Self = @This();

        pub fn next(self: *Self) ?U {
            if (self.iter.next()) |val| {
                return self.transform(val);
            }
            return null;
        }
    };
}

// Filter iterator - กรองค่าใน iterator
fn FilterIterator(comptime Iter: type, comptime T: type) type {
    return struct {
        iter: Iter,
        predicate: *const fn (T) bool,

        const Self = @This();

        pub fn next(self: *Self) ?T {
            while (self.iter.next()) |val| {
                if (self.predicate(val)) return val;
            }
            return null;
        }
    };
}

// Take iterator - เอาแค่ N ตัวแรก
fn TakeIterator(comptime Iter: type, comptime T: type) type {
    return struct {
        iter: Iter,
        remaining: usize,

        const Self = @This();

        pub fn next(self: *Self) ?T {
            if (self.remaining == 0) return null;
            self.remaining -= 1;
            return self.iter.next();
        }
    };
}

// Zip iterator - รวม 2 iterators
fn ZipIterator(comptime IterA: type, comptime A: type, comptime IterB: type, comptime B: type) type {
    return struct {
        iter_a: IterA,
        iter_b: IterB,

        const Self = @This();

        pub fn next(self: *Self) ?struct { a: A, b: B } {
            const a = self.iter_a.next() orelse return null;
            const b = self.iter_b.next() orelse return null;
            return .{ .a = a, .b = b };
        }
    };
}

// Collect iterator เป็น ArrayList
fn collect(iter: anytype, allocator: std.mem.Allocator) !std.ArrayList(@typeInfo(@TypeOf(iter.next())).Optional.child) {
    const T = @typeInfo(@TypeOf(iter.next())).Optional.child;
    var list = std.ArrayList(T).init(allocator);
    var it = iter;
    while (it.next()) |val| {
        try list.append(val);
    }
    return list;
}

// ตัวอย่าง Tree Iterator (depth-first)
const BinaryTree = struct {
    value: i32,
    left: ?*BinaryTree,
    right: ?*BinaryTree,

    const Self = @This();

    // In-order iterator (left, root, right)
    const Iterator = struct {
        stack: std.ArrayList(*BinaryTree),
        current: ?*BinaryTree,

        pub fn init(root: ?*BinaryTree, allocator: std.mem.Allocator) !Iterator {
            var iter = Iterator{
                .stack = std.ArrayList(*BinaryTree).init(allocator),
                .current = root,
            };
            return iter;
        }

        pub fn deinit(self: *Iterator) void {
            self.stack.deinit();
        }

        pub fn next(self: *Iterator) !?i32 {
            while (self.current != null or self.stack.items.len > 0) {
                // ไปทางซ้ายสุด
                while (self.current) |node| {
                    try self.stack.append(node);
                    self.current = node.left;
                }

                // pop และเยี่ยม
                if (self.stack.items.len > 0) {
                    const node = self.stack.pop();
                    self.current = node.right;
                    return node.value;
                }
            }
            return null;
        }
    };

    pub fn iterator(self: *BinaryTree, allocator: std.mem.Allocator) !Iterator {
        return Iterator.init(self, allocator);
    }
};

// String iterator - iterate characters
const CharIterator = struct {
    bytes: []const u8,
    index: usize,

    pub fn init(s: []const u8) CharIterator {
        return .{ .bytes = s, .index = 0 };
    }

    pub fn next(self: *CharIterator) ?u8 {
        if (self.index >= self.bytes.len) return null;
        defer self.index += 1;
        return self.bytes[self.index];
    }
};

// Word iterator
const WordIterator = struct {
    bytes: []const u8,
    index: usize,

    pub fn init(s: []const u8) WordIterator {
        return .{ .bytes = s, .index = 0 };
    }

    pub fn next(self: *WordIterator) ?[]const u8 {
        // ข้าม whitespace
        while (self.index < self.bytes.len and
            (self.bytes[self.index] == ' ' or
            self.bytes[self.index] == '\n' or
            self.bytes[self.index] == '\t'))
        {
            self.index += 1;
        }

        if (self.index >= self.bytes.len) return null;

        const start = self.index;
        // อ่านจนกว่าจะเจอ whitespace
        while (self.index < self.bytes.len and
            self.bytes[self.index] != ' ' and
            self.bytes[self.index] != '\n' and
            self.bytes[self.index] != '\t')
        {
            self.index += 1;
        }

        return self.bytes[start..self.index];
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ทดสอบ Range iterator
    std.debug.print("=== Range Iterator ===\n", .{});
    var range = Range(i32).init(1, 11, 1);
    std.debug.print("1 ถึง 10: ", .{});
    while (range.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // เลขคู่ 0-20
    var even_range = Range(i32).init(0, 21, 2);
    std.debug.print("เลขคู่ 0-20: ", .{});
    while (even_range.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // Countdown
    var countdown = Range(i32).init(10, 0, -1);
    std.debug.print("นับถอยหลัง: ", .{});
    while (countdown.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // ทดสอบ Word iterator
    std.debug.print("\n=== Word Iterator ===\n", .{});
    const sentence = "Zig เป็นภาษา programming ที่ดีมาก";
    var words = WordIterator.init(sentence);
    var word_count: usize = 0;
    while (words.next()) |word| {
        word_count += 1;
        std.debug.print("Word {d}: [{s}]\n", .{ word_count, word });
    }

    // ทดสอบ Char iterator
    std.debug.print("\n=== Char Iterator ===\n", .{});
    var chars = CharIterator.init("Hello");
    var char_count: usize = 0;
    while (chars.next()) |c| {
        char_count += 1;
        std.debug.print("Char {d}: '{c}' (ASCII: {d})\n", .{ char_count, c, c });
    }

    // ทดสอบ Binary Tree iterator
    std.debug.print("\n=== Binary Tree In-Order Iterator ===\n", .{});
    //        4
    //       / \
    //      2   6
    //     / \ / \
    //    1  3 5  7
    var node1 = BinaryTree{ .value = 1, .left = null, .right = null };
    var node3 = BinaryTree{ .value = 3, .left = null, .right = null };
    var node5 = BinaryTree{ .value = 5, .left = null, .right = null };
    var node7 = BinaryTree{ .value = 7, .left = null, .right = null };
    var node2 = BinaryTree{ .value = 2, .left = &node1, .right = &node3 };
    var node6 = BinaryTree{ .value = 6, .left = &node5, .right = &node7 };
    var root = BinaryTree{ .value = 4, .left = &node2, .right = &node6 };

    var tree_iter = try root.iterator(allocator);
    defer tree_iter.deinit();

    std.debug.print("In-order traversal: ", .{});
    while (try tree_iter.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // แสดง comptime interface checking
    std.debug.print("\n=== Interface Checking ===\n", .{});
    std.debug.print("Range(i32) เป็น iterator: {}\n", .{isIterator(Range(i32))});
    std.debug.print("CharIterator เป็น iterator: {}\n", .{isIterator(CharIterator)});
    std.debug.print("WordIterator เป็น iterator: {}\n", .{isIterator(WordIterator)});
    std.debug.print("i32 เป็น iterator: {}\n", .{isIterator(i32)});
    _ = allocator;
}
```

---

## Step 250: Interface เปรียบเทียบกับ OOP Interfaces

### ตาราง เปรียบเทียบ Interface Patterns

```zig
// ==========================================
// เปรียบเทียบ 5 วิธีทำ Interface ใน Zig
// ==========================================

const std = @import("std");

// ============================================
// วิธีที่ 1: anytype (Static dispatch)
// เหมาะสำหรับ: Generic algorithms, high performance
// ============================================

const Method1 = struct {
    pub fn describe() void {
        std.debug.print(
            \\วิธีที่ 1: anytype
            \\  - Compile time dispatch
            \\  - Zero overhead
            \\  - ไม่สามารถใช้ใน runtime collection
            \\  - เหมือน C++ templates / Rust generics
            \\
        , .{});
    }

    fn printArea(shape: anytype) void {
        std.debug.print("พื้นที่: {d:.2}\n", .{shape.area()});
    }
};

// ============================================
// วิธีที่ 2: Comptime checking (Static dispatch + error)
// เหมาะสำหรับ: Library APIs ที่ต้องการ error messages ดี
// ============================================

const Method2 = struct {
    pub fn describe() void {
        std.debug.print(
            \\วิธีที่ 2: Comptime checking
            \\  - Compile time dispatch พร้อม custom error messages
            \\  - Zero overhead
            \\  - Interface contract ที่ชัดเจน
            \\  - เหมาะสำหรับ library development
            \\
        , .{});
    }

    fn requiresShape(comptime T: type) void {
        if (!@hasDecl(T, "area")) {
            @compileError(@typeName(T) ++ " ต้องมี area() method");
        }
        if (!@hasDecl(T, "perimeter")) {
            @compileError(@typeName(T) ++ " ต้องมี perimeter() method");
        }
    }

    fn printShapeInfo(shape: anytype) void {
        comptime requiresShape(@TypeOf(shape));
        std.debug.print("พื้นที่: {d:.2}, เส้นรอบ: {d:.2}\n", .{
            shape.area(),
            shape.perimeter(),
        });
    }
};

// ============================================
// วิธีที่ 3: VTable / Fat pointer (Dynamic dispatch)
// เหมาะสำหรับ: Collections ของ types ต่างๆ, plugin systems
// ============================================

const Method3 = struct {
    pub fn describe() void {
        std.debug.print(
            \\วิธีที่ 3: VTable / Fat pointer
            \\  - Runtime dispatch
            \\  - มี overhead จาก pointer indirection
            \\  - สามารถเก็บใน collection ได้
            \\  - เหมือน Go interfaces / Rust dyn Trait
            \\
        , .{});
    }

    // (ดูตัวอย่างจาก Step 245)
};

// ============================================
// วิธีที่ 4: Tagged union (Closed set)
// เหมาะสำหรับ: AST nodes, JSON values, known types
// ============================================

const Method4 = struct {
    pub fn describe() void {
        std.debug.print(
            \\วิธีที่ 4: Tagged union
            \\  - Compile time known types
            \\  - Exhaustive switch - compiler บังคับให้ handle ทุก case
            \\  - มี overhead จาก tag field
            \\  - เหมาะสำหรับ closed polymorphism
            \\
        , .{});
    }

    // (ดูตัวอย่างจาก Step 246)
};

// ============================================
// วิธีที่ 5: Comptime type generation
// เหมาะสำหรับ: Generic data structures
// ============================================

const Method5 = struct {
    pub fn describe() void {
        std.debug.print(
            \\วิธีที่ 5: Comptime type generation
            \\  - สร้าง type ใหม่ตอน compile time
            \\  - Zero overhead
            \\  - ใช้สำหรับ generic data structures
            \\  - เหมือน Rust generic structs
            \\
        , .{});
    }

    // ตัวอย่าง generic Stack
    fn Stack(comptime T: type) type {
        return struct {
            items: std.ArrayList(T),
            allocator: std.mem.Allocator,

            const Self = @This();

            pub fn init(allocator: std.mem.Allocator) Self {
                return .{
                    .items = std.ArrayList(T).init(allocator),
                    .allocator = allocator,
                };
            }

            pub fn deinit(self: *Self) void {
                self.items.deinit();
            }

            pub fn push(self: *Self, item: T) !void {
                try self.items.append(item);
            }

            pub fn pop(self: *Self) ?T {
                return self.items.popOrNull();
            }

            pub fn peek(self: Self) ?T {
                if (self.items.items.len == 0) return null;
                return self.items.items[self.items.items.len - 1];
            }

            pub fn isEmpty(self: Self) bool {
                return self.items.items.len == 0;
            }
        };
    }
};

// ============================================
// Real-world example: Plugin system ด้วย VTable
// ============================================

// Plugin interface
const Plugin = struct {
    ctx: *anyopaque,
    vtable: *const VTable,

    const VTable = struct {
        name: *const fn (*anyopaque) []const u8,
        version: *const fn (*anyopaque) []const u8,
        execute: *const fn (*anyopaque, []const u8) anyerror!void,
        deinit: *const fn (*anyopaque, std.mem.Allocator) void,
    };

    pub fn getName(self: Plugin) []const u8 {
        return self.vtable.name(self.ctx);
    }

    pub fn getVersion(self: Plugin) []const u8 {
        return self.vtable.version(self.ctx);
    }

    pub fn execute(self: Plugin, input: []const u8) anyerror!void {
        return self.vtable.execute(self.ctx, input);
    }

    pub fn deinit(self: Plugin, allocator: std.mem.Allocator) void {
        self.vtable.deinit(self.ctx, allocator);
    }

    pub fn fromConcrete(comptime T: type, allocator: std.mem.Allocator, value: T) !Plugin {
        const gen = comptime struct {
            fn name(ctx: *anyopaque) []const u8 {
                return @as(*T, @ptrCast(@alignCast(ctx))).name();
            }
            fn version(ctx: *anyopaque) []const u8 {
                return @as(*T, @ptrCast(@alignCast(ctx))).version();
            }
            fn execute(ctx: *anyopaque, input: []const u8) anyerror!void {
                return @as(*T, @ptrCast(@alignCast(ctx))).execute(input);
            }
            fn deinit(ctx: *anyopaque, alloc: std.mem.Allocator) void {
                const ptr = @as(*T, @ptrCast(@alignCast(ctx)));
                ptr.deinit();
                alloc.destroy(ptr);
            }
            const vtable = VTable{
                .name = name,
                .version = version,
                .execute = execute,
                .deinit = deinit,
            };
        };

        const ptr = try allocator.create(T);
        ptr.* = value;
        return Plugin{ .ctx = ptr, .vtable = &gen.vtable };
    }
};

// Concrete plugin implementations
const UppercasePlugin = struct {
    pub fn name(self: *UppercasePlugin) []const u8 {
        _ = self;
        return "Uppercase Plugin";
    }

    pub fn version(self: *UppercasePlugin) []const u8 {
        _ = self;
        return "1.0.0";
    }

    pub fn execute(self: *UppercasePlugin, input: []const u8) !void {
        _ = self;
        std.debug.print("UPPERCASE: ", .{});
        for (input) |c| {
            const upper = if (c >= 'a' and c <= 'z') c - 32 else c;
            std.debug.print("{c}", .{upper});
        }
        std.debug.print("\n", .{});
    }

    pub fn deinit(self: *UppercasePlugin) void {
        _ = self;
    }
};

const ReversePlugin = struct {
    pub fn name(self: *ReversePlugin) []const u8 {
        _ = self;
        return "Reverse Plugin";
    }

    pub fn version(self: *ReversePlugin) []const u8 {
        _ = self;
        return "2.1.0";
    }

    pub fn execute(self: *ReversePlugin, input: []const u8) !void {
        _ = self;
        std.debug.print("REVERSED: ", .{});
        var i: usize = input.len;
        while (i > 0) {
            i -= 1;
            std.debug.print("{c}", .{input[i]});
        }
        std.debug.print("\n", .{});
    }

    pub fn deinit(self: *ReversePlugin) void {
        _ = self;
    }
};

const CountPlugin = struct {
    count: usize,

    pub fn name(self: *CountPlugin) []const u8 {
        _ = self;
        return "Count Plugin";
    }

    pub fn version(self: *CountPlugin) []const u8 {
        _ = self;
        return "1.5.2";
    }

    pub fn execute(self: *CountPlugin, input: []const u8) !void {
        self.count += 1;
        std.debug.print("COUNT: {d} chars (executed {d} times)\n", .{ input.len, self.count });
    }

    pub fn deinit(self: *CountPlugin) void {
        _ = self;
    }
};

// Plugin manager
const PluginManager = struct {
    plugins: std.ArrayList(Plugin),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) PluginManager {
        return .{
            .plugins = std.ArrayList(Plugin).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *PluginManager) void {
        for (self.plugins.items) |plugin| {
            plugin.deinit(self.allocator);
        }
        self.plugins.deinit();
    }

    pub fn register(self: *PluginManager, plugin: Plugin) !void {
        try self.plugins.append(plugin);
        std.debug.print("ลงทะเบียน plugin: {s} v{s}\n", .{
            plugin.getName(),
            plugin.getVersion(),
        });
    }

    pub fn executeAll(self: PluginManager, input: []const u8) !void {
        std.debug.print("\n--- Execute all plugins with: \"{s}\" ---\n", .{input});
        for (self.plugins.items) |plugin| {
            try plugin.execute(input);
        }
    }

    pub fn listPlugins(self: PluginManager) void {
        std.debug.print("\n=== Registered Plugins ===\n", .{});
        for (self.plugins.items, 1..) |plugin, i| {
            std.debug.print("{d}. {s} ({s})\n", .{ i, plugin.getName(), plugin.getVersion() });
        }
    }
};

// ============================================
// เปรียบเทียบ Interface Patterns ทั้งหมด
// ============================================

const InterfaceComparison = struct {
    pub fn printComparison() void {
        std.debug.print(
            \\
            \\╔════════════════════════════════════════════════════════════════╗
            \\║         เปรียบเทียบ Interface Patterns ใน Zig                 ║
            \\╠══════════════╦══════════════╦═══════════╦═════════════════════╣
            \\║ Pattern      ║ Dispatch     ║ Overhead  ║ Use Case            ║
            \\╠══════════════╬══════════════╬═══════════╬═════════════════════╣
            \\║ anytype      ║ Compile time ║ Zero      ║ Generic algorithms  ║
            \\║ comptime     ║ Compile time ║ Zero      ║ Library APIs        ║
            \\║ vtable       ║ Runtime      ║ ptr indir ║ Plugin systems      ║
            \\║ tagged union ║ Runtime      ║ tag field ║ AST, JSON values    ║
            \\║ comptime gen ║ Compile time ║ Zero      ║ Generic structures  ║
            \\╚══════════════╩══════════════╩═══════════╩═════════════════════╝
            \\
        , .{});
    }

    pub fn printVsOtherLanguages() void {
        std.debug.print(
            \\
            \\╔══════════════════════════════════════════════════════════════════════╗
            \\║          Zig vs ภาษาอื่น สำหรับ Interface                           ║
            \\╠══════════════╦════════════════════════════════════════════════════╣
            \\║ ภาษา        ║ Interface Mechanism                                 ║
            \\╠══════════════╬════════════════════════════════════════════════════╣
            \\║ Java         ║ interface keyword, explicit implements              ║
            \\║ Go           ║ implicit structural typing, interface keyword       ║
            \\║ Rust         ║ trait keyword, explicit impl Trait                  ║
            \\║ C++          ║ virtual functions, abstract classes                 ║
            \\║ Python       ║ duck typing, ABCs                                   ║
            \\║ Zig anytype  ║ duck typing, zero overhead, compile time            ║
            \\║ Zig vtable   ║ explicit fat pointer, runtime dispatch              ║
            \\║ Zig union    ║ tagged union, exhaustive switch                     ║
            \\╚══════════════╩════════════════════════════════════════════════════╝
            \\
        , .{});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // แสดงวิธีต่างๆ
    std.debug.print("=== Interface Methods ใน Zig ===\n\n", .{});
    Method1.describe();
    Method2.describe();
    Method3.describe();
    Method4.describe();
    Method5.describe();

    // ทดสอบ Generic Stack
    std.debug.print("\n=== Generic Stack ===\n", .{});
    const IntStack = Method5.Stack(i32);
    var stack = IntStack.init(allocator);
    defer stack.deinit();

    try stack.push(10);
    try stack.push(20);
    try stack.push(30);

    std.debug.print("peek: {?d}\n", .{stack.peek()});
    while (stack.pop()) |val| {
        std.debug.print("pop: {d}\n", .{val});
    }

    // ทดสอบ Plugin system
    std.debug.print("\n=== Plugin System ===\n", .{});
    var manager = PluginManager.init(allocator);
    defer manager.deinit();

    try manager.register(try Plugin.fromConcrete(UppercasePlugin, allocator, UppercasePlugin{}));
    try manager.register(try Plugin.fromConcrete(ReversePlugin, allocator, ReversePlugin{}));
    try manager.register(try Plugin.fromConcrete(CountPlugin, allocator, CountPlugin{ .count = 0 }));

    manager.listPlugins();
    try manager.executeAll("Hello, Zig!");
    try manager.executeAll("สวัสดี Zig");

    // แสดงตาราง เปรียบเทียบ
    InterfaceComparison.printComparison();
    InterfaceComparison.printVsOtherLanguages();
}
```

---

## สรุปบทเรียน: Interface Patterns ใน Zig

### เมื่อไหร่ควรใช้แต่ละ Pattern?

**1. `anytype` (Duck Typing)**
- เมื่อต้องการ generic algorithm ที่ทำงานกับหลาย types
- เมื่อ performance เป็นสิ่งสำคัญ
- เมื่อไม่จำเป็นต้องเก็บ types ต่างๆ ใน collection เดียวกัน
- ตัวอย่าง: `sort`, `print`, `serialize`

**2. Comptime Interface Checking**
- เมื่อต้องการ error messages ที่ดีสำหรับ library users
- เมื่อต้องการ document interface contract อย่างชัดเจน
- ตัวอย่าง: `std.hash_map`, custom container types

**3. VTable / Fat Pointer**
- เมื่อต้องการเก็บ types ต่างๆ ใน collection เดียวกัน (runtime polymorphism)
- เมื่อต้องการ plugin system หรือ callback
- ตัวอย่าง: `std.io.Writer`, `std.io.Reader`, plugin managers

**4. Tagged Union**
- เมื่อรู้ล่วงหน้าว่ามี types กี่อย่าง (closed set)
- เมื่อต้องการ exhaustive handling ของทุก cases
- ตัวอย่าง: AST nodes, JSON values, error types

**5. Comptime Type Generation**
- เมื่อสร้าง generic data structures
- เมื่อต้องการ type-safe containers
- ตัวอย่าง: `ArrayList(T)`, `HashMap(K, V)`

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Comparable Interface

```zig
// TODO: สร้าง interface สำหรับ comparable types
// โดยใช้ comptime interface checking
// ต้องมี methods:
//   - compareTo(other: @This()) std.math.Order
//   - eql(other: @This()) bool

// สร้าง sortable collection ที่ใช้ interface นี้
fn SortedList(comptime T: type) type {
    // TODO: implement
    _ = T;
    return struct {};
}

// Test ด้วย:
// - integers
// - strings
// - custom struct (เช่น Person โดย sort ด้วย age)
```

### แบบฝึกหัดที่ 2: สร้าง Event System ด้วย VTable

```zig
// TODO: สร้าง event system ที่รองรับ multiple event types
// และ multiple handlers

// EventHandler interface ด้วย VTable
const EventHandler = struct {
    // TODO: define vtable and dispatch methods
};

// Event types
const Event = union(enum) {
    click: struct { x: i32, y: i32 },
    keypress: struct { key: u8 },
    resize: struct { width: u32, height: u32 },
};

// Event dispatcher
const EventDispatcher = struct {
    handlers: std.ArrayList(EventHandler),
    // TODO: implement
};
```

### แบบฝึกหัดที่ 3: Lazy Iterator Pipeline

```zig
// TODO: สร้าง iterator pipeline ที่รองรับ:
// - map: แปลงค่าแต่ละ element
// - filter: กรอง elements
// - take: เอาแค่ N ตัวแรก
// - skip: ข้าม N ตัวแรก
// - reduce: รวมทุก elements เป็นค่าเดียว
// - collect: เก็บลง ArrayList

// ตัวอย่างการใช้:
// const result = try pipeline(Range(i32).init(1, 100, 1))
//     .filter(isEven)
//     .map(square)
//     .take(5)
//     .collect(allocator);
// ผลลัพธ์: [4, 16, 36, 64, 100]
```

### แบบฝึกหัดที่ 4: Serialization Framework

```zig
// TODO: สร้าง serialization framework ที่รองรับหลาย formats
// - JSON
// - CSV
// - Binary

// Serializer interface
fn Serializer(comptime WriterType: type) type {
    return struct {
        writer: WriterType,
        // TODO: implement serialize methods
    };
}

// ให้ struct สามารถ implement เอง
// หรือใช้ reflection เพื่อ auto-serialize
```

### แบบฝึกหัดที่ 5: Plugin Architecture

```zig
// TODO: สร้าง plugin architecture สำหรับ text processor
// Plugins:
// 1. SpellChecker - ตรวจสอบการสะกดคำ
// 2. WordCounter - นับคำ
// 3. SentimentAnalyzer - วิเคราะห์ sentiment
// 4. Formatter - จัด format ข้อความ

// แต่ละ plugin ต้อง implement:
//   - process(text: []const u8) ![]const u8
//   - name() []const u8
//   - configure(opts: anytype) !void

// สร้าง pipeline ที่รัน plugins ตามลำดับ
```

---

## สิ่งที่ได้เรียนรู้ใน Part 25

1. **Zig ไม่มี `interface` keyword** แต่มีหลายวิธีในการทำ interface behavior
2. **`anytype`** สำหรับ duck typing ที่ตรวจสอบตอน compile time
3. **Comptime interface checking** สำหรับ error messages ที่ดีขึ้น
4. **VTable pattern** สำหรับ dynamic dispatch เหมือน Go interfaces
5. **Fat pointer** สำหรับรวม data pointer กับ vtable
6. **Tagged union** สำหรับ closed-set polymorphism
7. **`std.io.Writer/Reader`** เป็นตัวอย่างที่ดีของ interface patterns ใน standard library
8. **Iterator pattern** ด้วย `next() ?T` convention
9. แต่ละ pattern มีข้อดีข้อเสียและ use cases ที่แตกต่างกัน

---

## อ่านเพิ่มเติม

- [Zig Documentation - comptime](https://ziglang.org/documentation/master/#comptime)
- [Zig std.io source](https://github.com/ziglang/zig/blob/master/lib/std/io.zig)
- [Zig Interfaces discussion](https://github.com/ziglang/zig/issues/130)
- [Andrew Kelley on Zig interfaces](https://andrewkelley.me/post/zig-programming-language-blurs-line-compile-time-run-time.html)

---

*ส่วนที่ 25 จบแล้ว - ต่อไปใน Part 26: Error Handling Advanced Patterns*
