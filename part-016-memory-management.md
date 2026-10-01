# Part 16: Memory Management ใน Zig (Steps 151-160)

Memory management เป็นหัวใจสำคัญของ Zig ที่แตกต่างจากภาษาอื่น Zig ให้ความสำคัญกับการควบคุมหน่วยความจำอย่างชัดเจนและปลอดภัย โดยไม่มี garbage collector แต่มีเครื่องมือที่ช่วยให้เขียนโค้ดที่ปลอดภัยได้

---

## Step 151: หน่วยความจำ Stack vs Heap

### แนวคิดพื้นฐาน

หน่วยความจำในโปรแกรมแบ่งออกเป็นสองส่วนหลัก ได้แก่ **Stack** และ **Heap** ซึ่งมีลักษณะการทำงานและการใช้งานที่แตกต่างกันอย่างสิ้นเชิง

```
แผนภาพ: โครงสร้างหน่วยความจำของโปรแกรม
┌─────────────────────────────────────────┐
│           ADDRESS SPACE                 │
│                                         │
│  สูง  ┌───────────────────────────┐     │
│       │         Stack             │     │
│       │  ┌─────────────────────┐  │     │
│       │  │  local variables    │  │     │
│       │  │  function frames    │  │     │
│       │  │  return addresses   │  │     │
│       │  └─────────────────────┘  │     │
│       │      ↓ เติบโตลง          │     │
│       │                           │     │
│       │         (ว่าง)            │     │
│       │                           │     │
│       │      ↑ เติบโตขึ้น         │     │
│       │  ┌─────────────────────┐  │     │
│       │  │  dynamic allocations│  │     │
│       │  │  malloc/free data   │  │     │
│       │  └─────────────────────┘  │     │
│       │         Heap              │     │
│  ต่ำ  └───────────────────────────┘     │
│                                         │
│       ┌───────────────────────────┐     │
│       │  BSS (uninitialized data) │     │
│       ├───────────────────────────┤     │
│       │  Data (initialized data)  │     │
│       ├───────────────────────────┤     │
│       │  Text (code segment)      │     │
│       └───────────────────────────┘     │
└─────────────────────────────────────────┘
```

### Stack คืออะไร?

Stack เป็นหน่วยความจำที่ทำงานแบบ LIFO (Last In, First Out) มีลักษณะดังนี้:

- **ขนาดจำกัด**: โดยทั่วไปประมาณ 1-8 MB ต่อ thread
- **เร็วมาก**: การ allocate/deallocate เป็นแค่การขยับ stack pointer
- **อัตโนมัติ**: หน่วยความจำถูกจัดการโดยอัตโนมัติเมื่อฟังก์ชัน return
- **ต่อเนื่อง**: ข้อมูลอยู่ติดกันใน memory

```
Stack Frame เมื่อเรียกฟังก์ชัน:
┌─────────────────────────┐
│   main() frame          │
│   ┌───────────────────┐ │
│   │ var x: i32 = 10   │ │
│   │ var y: i32 = 20   │ │
│   └───────────────────┘ │
│         │               │
│    เรียก foo()          │
│         ↓               │
│   ┌───────────────────┐ │
│   │ foo() frame       │ │
│   │ var a: i32 = 5    │ │
│   │ return address    │ │
│   └───────────────────┘ │
│         │               │
│    foo() return         │
│         ↓               │
│   foo() frame ถูกลบ    │
└─────────────────────────┘
```

### Heap คืออะไร?

Heap เป็นพื้นที่หน่วยความจำแบบ dynamic ที่:

- **ขนาดยืดหยุ่น**: จัดสรรได้มากตามที่ระบบมี
- **ช้ากว่า**: ต้องมีการค้นหาพื้นที่ว่างและจัดการ metadata
- **ต้องจัดการเอง**: โปรแกรมต้องขอ (alloc) และคืน (free) หน่วยความจำเอง
- **กระจาย**: ข้อมูลอาจไม่อยู่ติดกัน

```zig
const std = @import("std");

pub fn main() !void {
    // Stack allocation - อัตโนมัติ, เร็ว
    var stack_var: i32 = 42;
    var stack_array: [10]i32 = undefined;
    
    // ตัวแปรบน Stack จะถูกลบเมื่อออกจาก scope
    {
        var local: i32 = 100;
        _ = local; // ใช้เพื่อหลีกเลี่ยง unused variable error
    } // local ถูกลบที่นี่โดยอัตโนมัติ
    
    _ = stack_var;
    _ = stack_array;
    
    std.debug.print("Stack allocation สำเร็จ\n", .{});
}
```

### เปรียบเทียบ Stack vs Heap

```
คุณสมบัติ        Stack              Heap
─────────────────────────────────────────────
ขนาด            จำกัด (~8MB)       ใหญ่มาก (GB)
ความเร็ว         เร็วมาก            ช้ากว่า
การจัดการ        อัตโนมัติ          ต้องจัดการเอง
อายุการใช้งาน   ตาม scope          จนกว่าจะ free
การเข้าถึง       Sequential         Random
ความปลอดภัย     สูง               ต้องระวัง leaks
เหมาะสำหรับ     ตัวแปร local       ข้อมูลขนาดใหญ่
                 ขนาดคงที่         ขนาดไม่แน่นอน
                                    อายุยาวนาน
```

---

## Step 152: Stack Allocation ใน Zig

### การประกาศตัวแปรบน Stack

ใน Zig ตัวแปรทุกตัวที่ประกาศภายในฟังก์ชันโดยตรงจะอยู่บน Stack

```zig
const std = @import("std");

fn demonstrateStack() void {
    // ตัวแปรพื้นฐาน - อยู่บน Stack
    var integer: i32 = 42;
    var float_val: f64 = 3.14;
    var boolean: bool = true;
    
    // Array ขนาดคงที่ - อยู่บน Stack
    var numbers: [5]i32 = .{ 1, 2, 3, 4, 5 };
    
    // Struct - อยู่บน Stack
    const Point = struct {
        x: f32,
        y: f32,
    };
    var point = Point{ .x = 1.0, .y = 2.0 };
    
    std.debug.print("integer = {}\n", .{integer});
    std.debug.print("float = {d}\n", .{float_val});
    std.debug.print("boolean = {}\n", .{boolean});
    std.debug.print("numbers[0] = {}\n", .{numbers[0]});
    std.debug.print("point = ({d}, {d})\n", .{ point.x, point.y });
}

pub fn main() void {
    demonstrateStack();
}
```

### Stack Frame และ Scope

```zig
const std = @import("std");

fn innerFunction(x: i32) i32 {
    // สร้าง stack frame ใหม่
    var result: i32 = x * 2;  // อยู่ใน frame ของ innerFunction
    return result;
    // result ถูกลบที่นี่ (stack frame คืนค่า)
}

fn outerFunction() void {
    var a: i32 = 10;  // อยู่ใน frame ของ outerFunction
    
    {
        // Block scope ใหม่ - แต่ยังอยู่ใน stack frame เดิม
        var b: i32 = 20;
        var c: i32 = a + b;
        std.debug.print("c = {}\n", .{c});
        // b และ c สามารถใช้ได้ที่นี่
    }
    // b และ c ไม่สามารถใช้ได้แล้ว (ออกจาก scope)
    
    var result = innerFunction(a);  // สร้าง stack frame สำหรับ innerFunction
    std.debug.print("result = {}\n", .{result});
    // innerFunction's stack frame ถูกลบแล้ว
}

pub fn main() void {
    outerFunction();
}
```

### ข้อจำกัดของ Stack

```zig
const std = @import("std");

// อันตราย! Stack overflow ถ้าอาร์เรย์ใหญ่เกินไป
fn badExample() void {
    // อย่าทำแบบนี้สำหรับข้อมูลขนาดใหญ่!
    // var huge_array: [1_000_000]i32 = undefined;  // อาจ stack overflow!
    
    // ควรใช้ Heap แทน
    std.debug.print("ขนาด Stack จำกัด\n", .{});
}

// ถูกต้อง - ใช้ Stack สำหรับข้อมูลขนาดเล็ก
fn goodExample() void {
    var small_array: [100]i32 = undefined;  // ขนาดเล็ก ปลอดภัย
    
    for (&small_array, 0..) |*item, i| {
        item.* = @intCast(i);
    }
    
    std.debug.print("Stack array ขนาดเล็กใช้ได้\n", .{});
}

pub fn main() void {
    badExample();
    goodExample();
}
```

### Stack-based Structs

```zig
const std = @import("std");

const Rectangle = struct {
    width: f32,
    height: f32,
    
    pub fn area(self: Rectangle) f32 {
        return self.width * self.height;
    }
    
    pub fn perimeter(self: Rectangle) f32 {
        return 2 * (self.width + self.height);
    }
};

pub fn main() void {
    // Rectangle อยู่บน Stack
    var rect = Rectangle{
        .width = 10.0,
        .height = 5.0,
    };
    
    std.debug.print("พื้นที่ = {d}\n", .{rect.area()});
    std.debug.print("เส้นรอบวง = {d}\n", .{rect.perimeter()});
    
    // Stack allocation หลายตัว
    var rects: [3]Rectangle = .{
        .{ .width = 1.0, .height = 1.0 },
        .{ .width = 2.0, .height = 3.0 },
        .{ .width = 5.0, .height = 4.0 },
    };
    
    for (rects) |r| {
        std.debug.print("พื้นที่ = {d}\n", .{r.area()});
    }
}
```

---

## Step 153: Heap Allocation ด้วย Allocator

### ทำไม Zig ถึงต้องการ Explicit Allocator?

Zig ต่างจากภาษาอื่นตรงที่ไม่มี global allocator โดยค่าเริ่มต้น นี่เป็นการออกแบบโดยตั้งใจเพื่อ:

1. **ความโปร่งใส**: คุณเห็นชัดว่าโค้ดส่วนไหนทำ heap allocation
2. **ความยืดหยุ่น**: เลือก allocator ที่เหมาะสมกับงาน
3. **การทดสอบ**: ใช้ test allocator เพื่อตรวจหา memory leaks
4. **Embedded systems**: ควบคุมได้ว่าจะใช้ dynamic memory หรือไม่

```
Zig Allocator Interface:
┌──────────────────────────────────────────┐
│           std.mem.Allocator              │
├──────────────────────────────────────────┤
│  alloc(T, n)     → []T                  │
│  create(T)       → *T                   │
│  free(slice)     → void                 │
│  destroy(ptr)    → void                 │
│  realloc(old, n) → []T                  │
└──────────────────────────────────────────┘
         ↑ implements
┌──────────────────────────────────────────┐
│  Implementations:                        │
│  • GeneralPurposeAllocator               │
│  • ArenaAllocator                        │
│  • FixedBufferAllocator                  │
│  • PageAllocator                         │
│  • c_allocator                           │
│  • testing.allocator (for tests)         │
└──────────────────────────────────────────┘
```

### ประเภทของ Allocator ใน Zig

```zig
const std = @import("std");

pub fn main() !void {
    // 1. GeneralPurposeAllocator - สำหรับการพัฒนาทั่วไป
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const gpa_alloc = gpa.allocator();
    
    // 2. ArenaAllocator - สำหรับ allocate หลายครั้ง แล้ว free ทีเดียว
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const arena_alloc = arena.allocator();
    
    // 3. FixedBufferAllocator - สำหรับ embedded หรือ stack-based allocation
    var buffer: [1024]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const fba_alloc = fba.allocator();
    
    // 4. page_allocator - allocate ทีละ page (ประหยัดน้อย)
    const page_alloc = std.heap.page_allocator;
    
    // ใช้ gpa_alloc ทำ allocation
    const arr = try gpa_alloc.alloc(i32, 5);
    defer gpa_alloc.free(arr);
    
    // ใช้ arena_alloc - ไม่ต้อง free แต่ละอัน
    const str = try arena_alloc.alloc(u8, 10);
    _ = str; // arena จะ free ทั้งหมดเมื่อ deinit
    
    // ใช้ fba_alloc - จาก buffer ที่กำหนดไว้
    const small = try fba_alloc.alloc(u8, 50);
    defer fba_alloc.free(small);
    
    // ใช้ page_allocator
    const page_mem = try page_alloc.alloc(u8, 4096);
    defer page_alloc.free(page_mem);
    
    std.debug.print("ทุก allocator ทำงานได้\n", .{});
}
```

### การส่ง Allocator ผ่านฟังก์ชัน

```zig
const std = @import("std");

// รูปแบบมาตรฐาน: รับ allocator เป็น parameter
fn createBuffer(allocator: std.mem.Allocator, size: usize) ![]u8 {
    const buf = try allocator.alloc(u8, size);
    @memset(buf, 0);  // initialize ด้วย zero
    return buf;
}

fn processData(allocator: std.mem.Allocator, data: []const u8) ![]u8 {
    // สร้าง buffer ใหม่ขนาดสองเท่า
    const result = try allocator.alloc(u8, data.len * 2);
    @memcpy(result[0..data.len], data);
    @memcpy(result[data.len..], data);
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ส่ง allocator ไปยังฟังก์ชัน
    const buf = try createBuffer(allocator, 100);
    defer allocator.free(buf);
    
    const message = "Hello";
    const doubled = try processData(allocator, message);
    defer allocator.free(doubled);
    
    std.debug.print("doubled = {s}\n", .{doubled});
}
```

---

## Step 154: allocator.alloc และ allocator.free

### การใช้ alloc และ free

`alloc` ใช้สำหรับ allocate array (slice) ของ type ที่กำหนด

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // alloc(Type, count) -> ![]Type
    // Type = ชนิดข้อมูลที่ต้องการ
    // count = จำนวน element
    
    // Allocate array ของ integers
    const int_array = try allocator.alloc(i32, 5);
    defer allocator.free(int_array);  // ต้อง free!
    
    // Initialize ข้อมูล
    for (int_array, 0..) |*item, i| {
        item.* = @intCast(i * 10);
    }
    
    // ใช้งาน
    for (int_array) |val| {
        std.debug.print("{} ", .{val});
    }
    std.debug.print("\n", .{});
    
    // Allocate string (slice of u8)
    const str = try allocator.alloc(u8, 20);
    defer allocator.free(str);
    
    // copy string
    const message = "Hello, World!";
    @memcpy(str[0..message.len], message);
    str[message.len] = 0;  // null terminator
    
    std.debug.print("str = {s}\n", .{str[0..message.len]});
}
```

### รูปแบบการใช้งาน alloc/free อย่างปลอดภัย

```zig
const std = @import("std");

// Pattern 1: defer free ทันทีหลัง alloc
fn pattern1(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    defer allocator.free(data);  // จะ free เมื่อออกจาก scope นี้
    
    // ทำงานกับ data ได้อย่างปลอดภัย
    for (data, 0..) |*item, i| {
        item.* = @intCast(i);
    }
    
    std.debug.print("Pattern 1: OK\n", .{});
    // data จะถูก free อัตโนมัติที่นี่
}

// Pattern 2: free ใน error handling
fn pattern2(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    errdefer allocator.free(data);  // free ถ้าเกิด error
    
    // ทำงานที่อาจเกิด error
    const more_data = try allocator.alloc(i32, 20);
    defer allocator.free(more_data);
    
    // ถ้าถึงจุดนี้ ทุกอย่างสำเร็จ
    // data จะถูก free เมื่อออกจาก scope
    defer allocator.free(data);
    
    std.debug.print("Pattern 2: OK\n", .{});
}

// Pattern 3: ส่งคืนไปยัง caller
fn pattern3(allocator: std.mem.Allocator) ![]i32 {
    const data = try allocator.alloc(i32, 10);
    // ไม่ free ที่นี่ - caller รับผิดชอบ
    
    for (data, 0..) |*item, i| {
        item.* = @intCast(i * i);
    }
    
    return data;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    try pattern1(allocator);
    try pattern2(allocator);
    
    const result = try pattern3(allocator);
    defer allocator.free(result);  // caller รับผิดชอบ free
    
    for (result) |val| {
        std.debug.print("{} ", .{val});
    }
    std.debug.print("\n", .{});
}
```

### การ Allocate หลายครั้ง

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Allocate หลาย arrays
    const arr1 = try allocator.alloc(i32, 5);
    defer allocator.free(arr1);
    
    const arr2 = try allocator.alloc(f64, 3);
    defer allocator.free(arr2);
    
    const arr3 = try allocator.alloc(u8, 100);
    defer allocator.free(arr3);
    
    // Initialize
    for (arr1, 0..) |*item, i| item.* = @intCast(i);
    for (arr2, 0..) |*item, i| item.* = @as(f64, @floatFromInt(i)) * 1.5;
    @memset(arr3, 'A');
    
    // ใช้งาน
    std.debug.print("arr1 = ", .{});
    for (arr1) |v| std.debug.print("{} ", .{v});
    std.debug.print("\n", .{});
    
    std.debug.print("arr2 = ", .{});
    for (arr2) |v| std.debug.print("{d:.1} ", .{v});
    std.debug.print("\n", .{});
    
    std.debug.print("arr3[0] = {c}\n", .{arr3[0]});
    
    // Memory layout visualization
    std.debug.print("\nMemory addresses:\n", .{});
    std.debug.print("arr1 ptr = {*}\n", .{arr1.ptr});
    std.debug.print("arr2 ptr = {*}\n", .{arr2.ptr});
    std.debug.print("arr3 ptr = {*}\n", .{arr3.ptr});
}
```

### ขนาด Memory ที่ Allocate

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // คำนวณขนาด memory ที่จะ allocate
    const count: usize = 10;
    
    // i32 ใช้ 4 bytes ต่อ element
    const int_arr = try allocator.alloc(i32, count);
    defer allocator.free(int_arr);
    std.debug.print("i32 array: {} bytes\n", .{int_arr.len * @sizeOf(i32)});
    
    // f64 ใช้ 8 bytes ต่อ element
    const float_arr = try allocator.alloc(f64, count);
    defer allocator.free(float_arr);
    std.debug.print("f64 array: {} bytes\n", .{float_arr.len * @sizeOf(f64)});
    
    // u8 ใช้ 1 byte ต่อ element
    const byte_arr = try allocator.alloc(u8, count);
    defer allocator.free(byte_arr);
    std.debug.print("u8 array: {} bytes\n", .{byte_arr.len * @sizeOf(u8)});
}
```

---

## Step 155: allocator.create และ allocator.destroy

### ความแตกต่างระหว่าง alloc และ create

```
alloc vs create:
┌─────────────────────────────────────────────────────┐
│  alloc(T, n) → []T (slice)                          │
│  • Allocate array ของ T จำนวน n ตัว                 │
│  • ส่งคืน slice (pointer + length)                   │
│  • ใช้สำหรับ array ของหลายตัว                       │
│  • free ด้วย allocator.free(slice)                   │
│                                                      │
│  create(T) → *T (pointer)                           │
│  • Allocate T จำนวน 1 ตัว                           │
│  • ส่งคืน pointer ไปยัง T                           │
│  • ใช้สำหรับสิ่งของชิ้นเดียว (struct, scalar)       │
│  • free ด้วย allocator.destroy(ptr)                  │
└─────────────────────────────────────────────────────┘
```

### การใช้ create และ destroy

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // create allocate หน่วยความจำสำหรับ 1 Node
    const node1 = try allocator.create(Node);
    defer allocator.destroy(node1);  // ต้อง destroy!
    
    node1.* = Node{
        .value = 10,
        .next = null,
    };
    
    const node2 = try allocator.create(Node);
    defer allocator.destroy(node2);
    
    node2.* = Node{
        .value = 20,
        .next = null,
    };
    
    // เชื่อมต่อ nodes
    node1.next = node2;
    
    // traverse
    var current: ?*Node = node1;
    while (current) |n| {
        std.debug.print("value = {}\n", .{n.value});
        current = n.next;
    }
}
```

### Linked List ด้วย create/destroy

```zig
const std = @import("std");

const ListNode = struct {
    data: i32,
    next: ?*ListNode,
};

const LinkedList = struct {
    head: ?*ListNode,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) LinkedList {
        return .{
            .head = null,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *LinkedList) void {
        var current = self.head;
        while (current) |node| {
            const next = node.next;
            self.allocator.destroy(node);  // destroy แต่ละ node
            current = next;
        }
        self.head = null;
    }
    
    pub fn push(self: *LinkedList, value: i32) !void {
        const node = try self.allocator.create(ListNode);
        node.* = .{
            .data = value,
            .next = self.head,
        };
        self.head = node;
    }
    
    pub fn pop(self: *LinkedList) ?i32 {
        const node = self.head orelse return null;
        const value = node.data;
        self.head = node.next;
        self.allocator.destroy(node);  // destroy node ที่ pop
        return value;
    }
    
    pub fn print(self: LinkedList) void {
        var current = self.head;
        while (current) |node| {
            std.debug.print("{} -> ", .{node.data});
            current = node.next;
        }
        std.debug.print("null\n", .{});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var list = LinkedList.init(allocator);
    defer list.deinit();  // destroy ทั้งหมดเมื่อเสร็จ
    
    try list.push(1);
    try list.push(2);
    try list.push(3);
    
    list.print();  // 3 -> 2 -> 1 -> null
    
    _ = list.pop();  // ลบ 3
    list.print();    // 2 -> 1 -> null
}
```

### สร้าง Struct บน Heap

```zig
const std = @import("std");

const Config = struct {
    host: []u8,
    port: u16,
    max_connections: u32,
    
    pub fn init(allocator: std.mem.Allocator, host: []const u8, port: u16) !*Config {
        const config = try allocator.create(Config);
        config.host = try allocator.dupe(u8, host);
        config.port = port;
        config.max_connections = 100;
        return config;
    }
    
    pub fn deinit(self: *Config, allocator: std.mem.Allocator) void {
        allocator.free(self.host);
        allocator.destroy(self);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const config = try Config.init(allocator, "localhost", 8080);
    defer config.deinit(allocator);
    
    std.debug.print("host = {s}\n", .{config.host});
    std.debug.print("port = {}\n", .{config.port});
    std.debug.print("max_connections = {}\n", .{config.max_connections});
}
```

---

## Step 156: Memory Leaks และการหลีกเลี่ยง

### Memory Leak คืออะไร?

Memory leak เกิดขึ้นเมื่อโปรแกรม allocate หน่วยความจำแต่ไม่ได้ free กลับคืน ทำให้หน่วยความจำที่ไม่ได้ใช้แล้วยังคงถูก occupy อยู่

```
Memory Leak Visualization:
┌─────────────────────────────────────┐
│              Heap                   │
│                                     │
│  Time 1:                            │
│  ┌──────┐  ← alloc(10)             │
│  │ 10B  │ (ใช้งาน)                 │
│  └──────┘                           │
│                                     │
│  Time 2:                            │
│  ┌──────┐  ← alloc(10)             │
│  │ 10B  │ (ใช้งาน)                 │
│  └──────┘                           │
│  ┌──────┐  ← alloc(20) (leaked)    │
│  │ 20B  │ (ลืม free! ยังอยู่)     │
│  └──────┘                           │
│                                     │
│  Time 3:                            │
│  ┌──────┐  ← alloc(10)             │
│  │ 10B  │ (ใช้งาน)                 │
│  └──────┘                           │
│  ┌──────┐  (leaked - ยังอยู่)      │
│  │ 20B  │                           │
│  └──────┘                           │
│  ┌──────┐  (leaked - ยังอยู่)      │
│  │ 50B  │                           │
│  └──────┘                           │
│                                     │
│  → Heap เต็มขึ้นเรื่อยๆ!          │
└─────────────────────────────────────┘
```

### ตัวอย่าง Memory Leak

```zig
const std = @import("std");

// ผิด! Memory leak
fn badFunction(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 100);
    // ลืม free data!
    // data หายไปเมื่อออกจาก scope แต่ memory ยังถูก allocate
    _ = data;
}

// ผิดเช่นกัน - leak เมื่อเกิด error
fn alsoWrong(allocator: std.mem.Allocator) !void {
    const data1 = try allocator.alloc(i32, 10);
    const data2 = try allocator.alloc(i32, 20);  // ถ้า error ที่นี่ data1 leak!
    
    // ทำงาน...
    
    allocator.free(data1);
    allocator.free(data2);
}

// ถูกต้อง - ใช้ defer
fn correctFunction(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 100);
    defer allocator.free(data);  // จะ free ทุกกรณี
    
    // ทำงานกับ data
    for (data, 0..) |*item, i| {
        item.* = @intCast(i);
    }
}

// ถูกต้อง - ใช้ errdefer สำหรับหลาย allocations
fn correctMultiple(allocator: std.mem.Allocator) !void {
    const data1 = try allocator.alloc(i32, 10);
    errdefer allocator.free(data1);  // free data1 ถ้าเกิด error หลังจากนี้
    
    const data2 = try allocator.alloc(i32, 20);
    errdefer allocator.free(data2);
    
    // ทำงาน...
    
    // free ปกติเมื่อสำเร็จ
    defer allocator.free(data1);
    defer allocator.free(data2);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // badFunction จะ leak!
    // try badFunction(allocator);
    
    try correctFunction(allocator);
    try correctMultiple(allocator);
    
    std.debug.print("ไม่มี memory leak\n", .{});
}
```

### สาเหตุที่พบบ่อยของ Memory Leak

```zig
const std = @import("std");

// 1. ลืม free เมื่อมีหลาย early returns
fn withEarlyReturn(allocator: std.mem.Allocator, condition: bool) !void {
    const data = try allocator.alloc(i32, 10);
    
    if (condition) {
        // ผิด! ลืม free data ก่อน return
        // return;
        
        // ถูก: ใช้ defer ก่อนหน้านี้
        allocator.free(data);
        return;
    }
    
    defer allocator.free(data);
    // ทำงาน...
}

// 2. Double free - ก็เป็นปัญหาเช่นกัน
fn doubleFree(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    
    allocator.free(data);
    // allocator.free(data);  // ผิด! Double free - undefined behavior
}

// 3. Use after free
fn useAfterFree(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    data[0] = 42;
    
    allocator.free(data);
    // data[0] = 100;  // ผิด! Use after free - undefined behavior
}

// รูปแบบที่ถูกต้อง
fn correctPattern(allocator: std.mem.Allocator, condition: bool) !void {
    const data = try allocator.alloc(i32, 10);
    defer allocator.free(data);  // ใช้ defer เสมอ!
    
    if (condition) {
        // ทำงานกับ data
        for (data) |*item| item.* = 0;
        return;  // defer จะ free data อัตโนมัติ
    }
    
    // ทำงานปกติ...
    for (data, 0..) |*item, i| item.* = @intCast(i);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    try withEarlyReturn(allocator, true);
    try withEarlyReturn(allocator, false);
    try doubleFree(allocator);
    try useAfterFree(allocator);
    try correctPattern(allocator, true);
    
    std.debug.print("เสร็จสิ้น\n", .{});
}
```

---

## Step 157: defer กับการ Free Memory

### ทำความเข้าใจ defer

`defer` เป็น mechanism ใน Zig ที่ทำให้ statement ถูก execute เมื่อออกจาก scope ปัจจุบัน ไม่ว่าจะออกโดยปกติหรือโดย error

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("เริ่มต้น\n", .{});
    
    defer std.debug.print("defer 1\n", .{});
    defer std.debug.print("defer 2\n", .{});
    defer std.debug.print("defer 3\n", .{});
    
    std.debug.print("กลาง\n", .{});
    
    // output:
    // เริ่มต้น
    // กลาง
    // defer 3  (ลำดับกลับกัน - LIFO)
    // defer 2
    // defer 1
}
```

### defer สำหรับ Memory Management

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();  // clean up allocator เมื่อ main() exit
    const allocator = gpa.allocator();
    
    // Pattern: alloc ตามด้วย defer free ทันที
    const buffer = try allocator.alloc(u8, 256);
    defer allocator.free(buffer);  // จะ free เสมอ
    
    // ทำงาน
    @memset(buffer, 0);
    const msg = "Hello, Memory!";
    @memcpy(buffer[0..msg.len], msg);
    
    std.debug.print("{s}\n", .{buffer[0..msg.len]});
    
    // defer หลาย levels
    {
        const inner = try allocator.alloc(i32, 10);
        defer allocator.free(inner);  // free เมื่อออกจาก inner scope
        
        for (inner, 0..) |*item, i| item.* = @intCast(i);
        std.debug.print("inner[5] = {}\n", .{inner[5]});
    }  // inner freed here
    
    // buffer ยังใช้ได้ที่นี่
    std.debug.print("buffer ยังอยู่: {s}\n", .{buffer[0..msg.len]});
}  // buffer freed here (defer)
```

### errdefer สำหรับ Error Cases

```zig
const std = @import("std");

const Config = struct {
    name: []u8,
    values: []i32,
};

fn loadConfig(allocator: std.mem.Allocator) !Config {
    // alloc name
    const name = try allocator.alloc(u8, 20);
    errdefer allocator.free(name);  // free name ถ้าเกิด error ต่อไป
    
    @memcpy(name[0..6], "config");
    
    // alloc values - ถ้า error ที่นี่ name จะถูก free โดย errdefer
    const values = try allocator.alloc(i32, 10);
    errdefer allocator.free(values);
    
    // initialize
    for (values, 0..) |*v, i| v.* = @intCast(i);
    
    // สำเร็จ! ส่งคืน Config
    // ไม่ free ที่นี่ - caller รับผิดชอบ
    return Config{
        .name = name,
        .values = values,
    };
}

fn freeConfig(allocator: std.mem.Allocator, config: *Config) void {
    allocator.free(config.name);
    allocator.free(config.values);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var config = try loadConfig(allocator);
    defer freeConfig(allocator, &config);
    
    std.debug.print("name = {s}\n", .{config.name[0..6]});
    std.debug.print("values[0] = {}\n", .{config.values[0]});
}
```

### defer กับ Struct cleanup

```zig
const std = @import("std");

const Database = struct {
    allocator: std.mem.Allocator,
    connection_string: []u8,
    buffer: []u8,
    is_connected: bool,
    
    pub fn init(allocator: std.mem.Allocator, conn_str: []const u8) !Database {
        const conn = try allocator.dupe(u8, conn_str);
        errdefer allocator.free(conn);
        
        const buf = try allocator.alloc(u8, 4096);
        errdefer allocator.free(buf);
        
        return .{
            .allocator = allocator,
            .connection_string = conn,
            .buffer = buf,
            .is_connected = false,
        };
    }
    
    pub fn deinit(self: *Database) void {
        self.allocator.free(self.connection_string);
        self.allocator.free(self.buffer);
        self.is_connected = false;
    }
    
    pub fn connect(self: *Database) void {
        self.is_connected = true;
        std.debug.print("เชื่อมต่อ: {s}\n", .{self.connection_string});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var db = try Database.init(allocator, "postgresql://localhost/mydb");
    defer db.deinit();  // cleanup อัตโนมัติ
    
    db.connect();
    
    std.debug.print("database connected: {}\n", .{db.is_connected});
}  // db.deinit() ถูกเรียกที่นี่
```

---

## Step 158: Realloc - การปรับขนาด Memory

### ทำไมต้องใช้ Realloc?

บางครั้งเราไม่รู้ขนาด memory ที่ต้องการล่วงหน้า หรือต้องการปรับขนาด array ให้ใหญ่ขึ้นหรือเล็กลง

```
Realloc Process:
┌──────────────────────────────────────────────┐
│  Before realloc:                             │
│  ┌──────────────────┐                        │
│  │ old_data [5]     │  address: 0x1000       │
│  └──────────────────┘                        │
│                                              │
│  realloc(old_data, 10)                       │
│                                              │
│  Case 1: พื้นที่ว่างพอ (extend in-place)    │
│  ┌────────────────────────────────┐          │
│  │ new_data [10]                  │  address: 0x1000 (เดิม)
│  └────────────────────────────────┘          │
│                                              │
│  Case 2: พื้นที่ว่างไม่พอ (move)            │
│  ┌──────────────────┐                        │
│  │ (freed)          │  address: 0x1000       │
│  └──────────────────┘                        │
│  ┌────────────────────────────────┐          │
│  │ new_data [10]                  │  address: 0x2000 (ใหม่)
│  └────────────────────────────────┘          │
└──────────────────────────────────────────────┘
```

### การใช้ realloc ใน Zig

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // เริ่มต้น allocate
    var buffer = try allocator.alloc(u8, 5);
    std.debug.print("เริ่มต้น len={}\n", .{buffer.len});
    
    // เพิ่มขนาด
    buffer = try allocator.realloc(buffer, 10);
    std.debug.print("หลัง realloc len={}\n", .{buffer.len});
    
    // เพิ่มอีก
    buffer = try allocator.realloc(buffer, 20);
    std.debug.print("หลัง realloc 2 len={}\n", .{buffer.len});
    
    // ลดขนาด
    buffer = try allocator.realloc(buffer, 3);
    std.debug.print("หลัง shrink len={}\n", .{buffer.len});
    
    defer allocator.free(buffer);
}
```

### Dynamic Array Implementation

```zig
const std = @import("std");

// Dynamic array ที่เติบโตได้
fn DynamicArray(comptime T: type) type {
    return struct {
        items: []T,
        capacity: usize,
        len: usize,
        allocator: std.mem.Allocator,
        
        const Self = @This();
        const INITIAL_CAPACITY = 4;
        
        pub fn init(allocator: std.mem.Allocator) !Self {
            const items = try allocator.alloc(T, INITIAL_CAPACITY);
            return .{
                .items = items,
                .capacity = INITIAL_CAPACITY,
                .len = 0,
                .allocator = allocator,
            };
        }
        
        pub fn deinit(self: *Self) void {
            self.allocator.free(self.items);
        }
        
        pub fn append(self: *Self, value: T) !void {
            if (self.len == self.capacity) {
                // เพิ่มขนาดเป็น 2 เท่า
                const new_capacity = self.capacity * 2;
                self.items = try self.allocator.realloc(self.items, new_capacity);
                self.capacity = new_capacity;
                std.debug.print("  → resize to capacity={}\n", .{new_capacity});
            }
            self.items[self.len] = value;
            self.len += 1;
        }
        
        pub fn get(self: Self, index: usize) T {
            return self.items[index];
        }
        
        pub fn slice(self: Self) []T {
            return self.items[0..self.len];
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var arr = try DynamicArray(i32).init(allocator);
    defer arr.deinit();
    
    std.debug.print("เพิ่มข้อมูล:\n", .{});
    var i: i32 = 0;
    while (i < 10) : (i += 1) {
        try arr.append(i * 10);
    }
    
    std.debug.print("\nผลลัพธ์: ", .{});
    for (arr.slice()) |v| {
        std.debug.print("{} ", .{v});
    }
    std.debug.print("\n", .{});
    std.debug.print("len={} capacity={}\n", .{ arr.len, arr.capacity });
}
```

### ArrayList ของ std library

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // std.ArrayList ใช้ realloc อัตโนมัติ
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();
    
    // append จะ realloc เมื่อจำเป็น
    try list.append(10);
    try list.append(20);
    try list.append(30);
    
    // appendSlice
    try list.appendSlice(&[_]i32{ 40, 50, 60 });
    
    std.debug.print("items: ", .{});
    for (list.items) |v| {
        std.debug.print("{} ", .{v});
    }
    std.debug.print("\n", .{});
    
    // ensureTotalCapacity - pre-allocate
    try list.ensureTotalCapacity(100);
    std.debug.print("capacity after ensure: {}\n", .{list.capacity});
    
    // shrinkAndFree - คืน memory ส่วนเกิน
    list.shrinkAndFree(list.items.len);
    std.debug.print("capacity after shrink: {}\n", .{list.capacity});
}
```

---

## Step 159: Memory Safety ใน Zig

### Zig Safety Features

Zig มีหลาย mechanisms เพื่อป้องกัน memory safety issues:

```
Memory Safety Tools ใน Zig:
┌─────────────────────────────────────────────┐
│  1. Bounds Checking (Debug mode)            │
│     • array[index] ตรวจสอบว่า index valid  │
│     • ป้องกัน buffer overflow              │
│                                             │
│  2. Null Safety                             │
│     • Optional types (?T)                  │
│     • ต้อง handle null cases               │
│                                             │
│  3. No Hidden Allocation                    │
│     • ไม่มี implicit heap allocation       │
│     • ต้องผ่าน explicit allocator          │
│                                             │
│  4. GeneralPurposeAllocator                 │
│     • ตรวจหา use-after-free               │
│     • ตรวจหา double-free                  │
│     • ตรวจหา memory leaks                 │
│                                             │
│  5. Safety Build Mode                       │
│     • Debug: เปิดทุก checks               │
│     • ReleaseSafe: checks แต่ optimize     │
│     • ReleaseFast: ปิด checks              │
└─────────────────────────────────────────────┘
```

### Bounds Checking

```zig
const std = @import("std");

pub fn main() void {
    var arr = [_]i32{ 1, 2, 3, 4, 5 };
    
    // ปลอดภัย - index อยู่ใน range
    std.debug.print("arr[0] = {}\n", .{arr[0]});
    std.debug.print("arr[4] = {}\n", .{arr[4]});
    
    // ใน Debug mode นี้จะ panic ถ้า index ออกนอกขอบ
    // arr[5] = 10;  // Error: index out of bounds
    
    // ตรวจสอบก่อน access
    const index: usize = 3;
    if (index < arr.len) {
        std.debug.print("arr[{}] = {}\n", .{ index, arr[index] });
    }
    
    // Slice bounds ก็ตรวจสอบ
    const slice = arr[1..4];  // [2, 3, 4]
    std.debug.print("slice len = {}\n", .{slice.len});
    // slice[5] = 10;  // Error ใน Debug mode
}
```

### Optional Types เพื่อหลีกเลี่ยง Null Pointer

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,  // Optional pointer - อาจเป็น null
};

fn findValue(head: ?*Node, target: i32) ?*Node {
    var current = head;
    while (current) |node| {
        if (node.value == target) return node;
        current = node.next;
    }
    return null;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง nodes
    const n3 = try allocator.create(Node);
    defer allocator.destroy(n3);
    n3.* = .{ .value = 30, .next = null };
    
    const n2 = try allocator.create(Node);
    defer allocator.destroy(n2);
    n2.* = .{ .value = 20, .next = n3 };
    
    const n1 = try allocator.create(Node);
    defer allocator.destroy(n1);
    n1.* = .{ .value = 10, .next = n2 };
    
    // ค้นหาด้วย optional return
    const found = findValue(n1, 20);
    if (found) |node| {
        std.debug.print("พบ: {}\n", .{node.value});
    } else {
        std.debug.print("ไม่พบ\n", .{});
    }
    
    const not_found = findValue(n1, 99);
    if (not_found) |node| {
        std.debug.print("พบ: {}\n", .{node.value});
    } else {
        std.debug.print("ไม่พบ value 99\n", .{});
    }
}
```

### Pointer Safety

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Pointer ปกติ - ต้อง dereference ด้วย .*
    const ptr = try allocator.create(i32);
    defer allocator.destroy(ptr);
    
    ptr.* = 42;
    std.debug.print("value = {}\n", .{ptr.*});
    
    // Slice - pointer + length
    const slice = try allocator.alloc(i32, 5);
    defer allocator.free(slice);
    
    for (slice, 0..) |*item, i| item.* = @intCast(i * 10);
    
    // Safe slicing
    const sub_slice = slice[1..4];  // index 1, 2, 3
    std.debug.print("sub_slice = ", .{});
    for (sub_slice) |v| std.debug.print("{} ", .{v});
    std.debug.print("\n", .{});
    
    // Pointer arithmetic (unsafe ต้องระวัง)
    const first_ptr = &slice[0];
    std.debug.print("first element via ptr = {}\n", .{first_ptr.*});
}
```

### FixedBufferAllocator สำหรับ Memory Safety

```zig
const std = @import("std");

pub fn main() !void {
    // FixedBufferAllocator ใช้ stack memory
    // ไม่สามารถ allocate เกินขนาด buffer
    var stack_buffer: [512]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&stack_buffer);
    const allocator = fba.allocator();
    
    // Allocate ภายใน buffer
    const arr1 = try allocator.alloc(u8, 100);
    const arr2 = try allocator.alloc(u8, 200);
    
    @memset(arr1, 'A');
    @memset(arr2, 'B');
    
    std.debug.print("arr1[0] = {c}\n", .{arr1[0]});
    std.debug.print("arr2[0] = {c}\n", .{arr2[0]});
    
    // ถ้า allocate เกิน 512 bytes จะได้ error OutOfMemory
    const result = allocator.alloc(u8, 300);  // 100+200+300 > 512
    if (result) |_| {
        std.debug.print("สำเร็จ\n", .{});
    } else |err| {
        std.debug.print("Error: {}\n", .{err});  // OutOfMemory
    }
}
```

---

## Step 160: การ Detect Memory Leaks ด้วย GeneralPurposeAllocator

### GeneralPurposeAllocator (GPA)

GPA เป็น allocator ที่ทรงพลังที่สุดสำหรับการพัฒนาและ debug โดยมีความสามารถ:

- ตรวจหา memory leaks
- ตรวจหา double-free
- ตรวจหา use-after-free
- แสดง stack trace ของ leak

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง GPA ด้วย configuration
    var gpa = std.heap.GeneralPurposeAllocator(.{
        // .safety = true,         // เปิด safety checks (default ใน Debug)
        // .stack_trace_frames = 7, // จำนวน frames ใน stack trace
    }){};
    
    // defer deinit และตรวจสอบ leaks
    defer {
        const result = gpa.deinit();
        if (result == .leak) {
            std.debug.print("MEMORY LEAK DETECTED!\n", .{});
        } else {
            std.debug.print("No memory leaks detected\n", .{});
        }
    }
    
    const allocator = gpa.allocator();
    
    // Allocate และ free อย่างถูกต้อง
    const data = try allocator.alloc(i32, 10);
    defer allocator.free(data);
    
    for (data, 0..) |*item, i| item.* = @intCast(i);
    std.debug.print("data[5] = {}\n", .{data[5]});
}
```

### การตรวจจับ Memory Leak

```zig
const std = @import("std");

// ฟังก์ชันที่มี memory leak
fn functionWithLeak(allocator: std.mem.Allocator) !void {
    const leaked = try allocator.alloc(i32, 5);
    // ลืม free leaked!
    _ = leaked;
    
    std.debug.print("ฟังก์ชันเสร็จแล้ว (แต่มี leak)\n", .{});
}

// ฟังก์ชันที่ถูกต้อง
fn functionWithoutLeak(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 5);
    defer allocator.free(data);  // ใช้ defer เสมอ
    
    std.debug.print("ฟังก์ชันเสร็จแล้ว (ไม่มี leak)\n", .{});
}

pub fn main() !void {
    // Test 1: ไม่มี leak
    {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer {
            const check = gpa.deinit();
            if (check == .leak) {
                std.debug.print("Test 1: LEAK FOUND\n", .{});
            } else {
                std.debug.print("Test 1: No leak - OK\n", .{});
            }
        }
        try functionWithoutLeak(gpa.allocator());
    }
    
    // Test 2: มี leak
    {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer {
            const check = gpa.deinit();
            if (check == .leak) {
                std.debug.print("Test 2: LEAK FOUND (expected)\n", .{});
            } else {
                std.debug.print("Test 2: No leak\n", .{});
            }
        }
        try functionWithLeak(gpa.allocator());
    }
}
```

### GPA ใน Unit Tests

```zig
const std = @import("std");
const testing = std.testing;

fn createAndFillArray(allocator: std.mem.Allocator, size: usize) ![]i32 {
    const arr = try allocator.alloc(i32, size);
    for (arr, 0..) |*item, i| item.* = @intCast(i * 2);
    return arr;
}

test "no memory leak" {
    // testing.allocator ตรวจหา leaks อัตโนมัติ
    const arr = try createAndFillArray(testing.allocator, 5);
    defer testing.allocator.free(arr);  // ต้อง free!
    
    try testing.expectEqual(@as(i32, 0), arr[0]);
    try testing.expectEqual(@as(i32, 8), arr[4]);
}

test "array values" {
    const arr = try createAndFillArray(testing.allocator, 3);
    defer testing.allocator.free(arr);
    
    try testing.expectEqual(@as(usize, 3), arr.len);
    try testing.expectEqual(@as(i32, 0), arr[0]);
    try testing.expectEqual(@as(i32, 2), arr[1]);
    try testing.expectEqual(@as(i32, 4), arr[2]);
}
```

### ArenaAllocator - Allocate หลายครั้ง Free ครั้งเดียว

```zig
const std = @import("std");

const ASTNode = struct {
    tag: enum { literal, add, mul },
    value: ?i32,
    left: ?*ASTNode,
    right: ?*ASTNode,
};

fn buildAST(allocator: std.mem.Allocator) !*ASTNode {
    // สร้าง AST: (2 + 3) * 4
    
    const num2 = try allocator.create(ASTNode);
    num2.* = .{ .tag = .literal, .value = 2, .left = null, .right = null };
    
    const num3 = try allocator.create(ASTNode);
    num3.* = .{ .tag = .literal, .value = 3, .left = null, .right = null };
    
    const num4 = try allocator.create(ASTNode);
    num4.* = .{ .tag = .literal, .value = 4, .left = null, .right = null };
    
    const add_node = try allocator.create(ASTNode);
    add_node.* = .{ .tag = .add, .value = null, .left = num2, .right = num3 };
    
    const mul_node = try allocator.create(ASTNode);
    mul_node.* = .{ .tag = .mul, .value = null, .left = add_node, .right = num4 };
    
    return mul_node;
}

fn evalAST(node: *ASTNode) i32 {
    switch (node.tag) {
        .literal => return node.value.?,
        .add => return evalAST(node.left.?) + evalAST(node.right.?),
        .mul => return evalAST(node.left.?) * evalAST(node.right.?),
    }
}

pub fn main() !void {
    // ArenaAllocator: allocate เยอะๆ แล้ว free ทีเดียว
    // เหมาะสำหรับ parsing, AST building, request handling
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();  // free ทุก allocation ในครั้งเดียว
    
    const allocator = arena.allocator();
    
    const ast = try buildAST(allocator);
    const result = evalAST(ast);
    
    std.debug.print("(2 + 3) * 4 = {}\n", .{result});
    
    // ไม่ต้อง free แต่ละ node
    // arena.deinit() จะ free ทั้งหมด
}
```

### Memory Usage Tracking

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // วัดการใช้ memory
    const sizes = [_]usize{ 100, 200, 50, 400, 150 };
    var allocated: [5][]u8 = undefined;
    
    var total_bytes: usize = 0;
    
    for (sizes, 0..) |size, i| {
        allocated[i] = try allocator.alloc(u8, size);
        total_bytes += size;
        std.debug.print("Allocated {} bytes (total: {})\n", .{ size, total_bytes });
    }
    
    // Free ทั้งหมด
    for (allocated) |ptr| {
        allocator.free(ptr);
    }
    
    std.debug.print("ทั้งหมด freed\n", .{});
}
```

### ใช้งาน GPA กับ Stack Traces

```zig
const std = @import("std");

// GPA ด้วย stack traces เพื่อ debug
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{
        .stack_trace_frames = 10,  // เก็บ 10 frames สำหรับ debug
    }){};
    defer {
        const result = gpa.deinit();
        switch (result) {
            .ok => std.debug.print("Memory: Clean\n", .{}),
            .leak => std.debug.print("Memory: LEAK!\n", .{}),
        }
    }
    
    const allocator = gpa.allocator();
    
    // Safe usage
    const buf = try allocator.alloc(u8, 64);
    defer allocator.free(buf);
    
    @memset(buf, 0);
    const msg = "Testing GPA";
    @memcpy(buf[0..msg.len], msg);
    
    std.debug.print("{s}\n", .{buf[0..msg.len]});
}
```

### สรุป Memory Allocators

```zig
const std = @import("std");

// ตัวอย่างการเลือก Allocator ตามสถานการณ์
pub fn main() !void {
    
    // 1. Development/Debug - ใช้ GPA
    {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer _ = gpa.deinit();
        const alloc = gpa.allocator();
        
        const data = try alloc.alloc(i32, 5);
        defer alloc.free(data);
        std.debug.print("GPA: allocated {} items\n", .{data.len});
    }
    
    // 2. Short-lived many allocations - ใช้ Arena
    {
        var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
        defer arena.deinit();
        const alloc = arena.allocator();
        
        // allocate หลายครั้งโดยไม่ต้อง free แต่ละอัน
        const a = try alloc.alloc(u8, 100);
        const b = try alloc.alloc(i32, 50);
        const c = try alloc.alloc(f64, 25);
        _ = a; _ = b; _ = c;
        std.debug.print("Arena: all freed at once\n", .{});
    }
    
    // 3. Embedded/Stack - ใช้ FixedBuffer
    {
        var buf: [256]u8 = undefined;
        var fba = std.heap.FixedBufferAllocator.init(&buf);
        const alloc = fba.allocator();
        
        const data = try alloc.alloc(u8, 128);
        defer alloc.free(data);
        std.debug.print("FBA: from stack buffer\n", .{});
    }
    
    // 4. Tests - ใช้ testing.allocator (แพ็ค leak detection)
    // const alloc = std.testing.allocator;  // ใช้ใน tests เท่านั้น
}
```

---

## รูปแบบ Memory Management ที่ดี (Best Practices)

### Pattern 1: RAII-style ด้วย init/deinit

```zig
const std = @import("std");

const MyResource = struct {
    allocator: std.mem.Allocator,
    data: []u8,
    extra: []i32,
    
    pub fn init(allocator: std.mem.Allocator, size: usize) !MyResource {
        const data = try allocator.alloc(u8, size);
        errdefer allocator.free(data);
        
        const extra = try allocator.alloc(i32, size / 4 + 1);
        errdefer allocator.free(extra);
        
        return .{
            .allocator = allocator,
            .data = data,
            .extra = extra,
        };
    }
    
    pub fn deinit(self: *MyResource) void {
        self.allocator.free(self.data);
        self.allocator.free(self.extra);
    }
    
    pub fn process(self: *MyResource) void {
        @memset(self.data, 0);
        for (self.extra) |*v| v.* = 42;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var resource = try MyResource.init(gpa.allocator(), 100);
    defer resource.deinit();
    
    resource.process();
    std.debug.print("data[0] = {}\n", .{resource.data[0]});
    std.debug.print("extra[0] = {}\n", .{resource.extra[0]});
}
```

### Pattern 2: Ownership Transfer

```zig
const std = @import("std");

// ฟังก์ชันที่ "owns" memory และส่งต่อ ownership
fn createString(allocator: std.mem.Allocator, value: []const u8) ![]u8 {
    // Caller ต้องรับผิดชอบ free
    return allocator.dupe(u8, value);
}

// ฟังก์ชันที่รับ ownership และ free เมื่อเสร็จ
fn consumeString(allocator: std.mem.Allocator, str: []u8) void {
    defer allocator.free(str);  // เราเป็นเจ้าของ จึง free ได้
    
    std.debug.print("Processing: {s}\n", .{str});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง string - เราเป็นเจ้าของ
    const str = try createString(allocator, "Hello, World!");
    
    // ส่ง ownership ให้ consumeString
    consumeString(allocator, str);
    // ไม่ต้อง free ที่นี่ - consumeString รับผิดชอบแล้ว
    
    std.debug.print("เสร็จสิ้น\n", .{});
}
```

### Pattern 3: Borrowing (ไม่รับ ownership)

```zig
const std = @import("std");

// ฟังก์ชันนี้แค่ "borrow" data ไม่ได้เป็นเจ้าของ
fn analyzeData(data: []const i32) i64 {
    var sum: i64 = 0;
    for (data) |v| sum += v;
    return sum;
}

fn printData(data: []const u8) void {
    std.debug.print("{s}\n", .{data});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Owner: main()
    const numbers = try allocator.alloc(i32, 5);
    defer allocator.free(numbers);
    
    for (numbers, 0..) |*n, i| n.* = @intCast(i + 1);
    
    // Borrow to analyzeData - ไม่ต้อง free ใน analyzeData
    const sum = analyzeData(numbers);
    std.debug.print("sum = {}\n", .{sum});
    
    // numbers ยังอยู่
    std.debug.print("numbers[0] = {}\n", .{numbers[0]});
}
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: สร้าง Stack Data Structure

```zig
// TODO: implement Stack ที่ใช้ heap allocation
const std = @import("std");

fn Stack(comptime T: type) type {
    return struct {
        // TODO: เพิ่ม fields ที่จำเป็น
        
        pub fn init(allocator: std.mem.Allocator) @This() {
            _ = allocator;
            // TODO: implement
            return undefined;
        }
        
        pub fn deinit(self: *@This()) void {
            _ = self;
            // TODO: implement
        }
        
        pub fn push(self: *@This(), value: T) !void {
            _ = value;
            // TODO: implement
        }
        
        pub fn pop(self: *@This()) ?T {
            // TODO: implement
            return null;
        }
        
        pub fn peek(self: @This()) ?T {
            // TODO: implement
            return null;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var stack = Stack(i32).init(gpa.allocator());
    defer stack.deinit();
    
    // Test push/pop
    try stack.push(1);
    try stack.push(2);
    try stack.push(3);
    
    std.debug.print("peek = {?}\n", .{stack.peek()});  // 3
    std.debug.print("pop = {?}\n", .{stack.pop()});     // 3
    std.debug.print("pop = {?}\n", .{stack.pop()});     // 2
}
```

### Exercise 2: Memory Pool

```zig
// TODO: implement MemoryPool ที่ reuse memory blocks
const std = @import("std");

fn MemoryPool(comptime T: type, comptime pool_size: usize) type {
    return struct {
        // TODO: fields
        
        pub fn init(allocator: std.mem.Allocator) !@This() {
            _ = allocator;
            // TODO: pre-allocate pool_size items
            return undefined;
        }
        
        pub fn deinit(self: *@This()) void {
            _ = self;
            // TODO: free all
        }
        
        pub fn acquire(self: *@This()) ?*T {
            _ = self;
            // TODO: return available slot
            return null;
        }
        
        pub fn release(self: *@This(), item: *T) void {
            _ = self;
            _ = item;
            // TODO: return slot to pool
        }
    };
}
```

### Exercise 3: String Builder

```zig
// TODO: implement StringBuilder ที่ใช้ dynamic allocation
const std = @import("std");

const StringBuilder = struct {
    // TODO: fields
    
    pub fn init(allocator: std.mem.Allocator) StringBuilder {
        _ = allocator;
        return undefined;
    }
    
    pub fn deinit(self: *StringBuilder) void {
        _ = self;
    }
    
    pub fn append(self: *StringBuilder, str: []const u8) !void {
        _ = str;
    }
    
    pub fn appendChar(self: *StringBuilder, c: u8) !void {
        _ = c;
    }
    
    pub fn toString(self: StringBuilder) []const u8 {
        _ = self;
        return "";
    }
    
    pub fn clear(self: *StringBuilder) void {
        _ = self;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var sb = StringBuilder.init(gpa.allocator());
    defer sb.deinit();
    
    try sb.append("Hello");
    try sb.append(", ");
    try sb.append("World!");
    
    std.debug.print("{s}\n", .{sb.toString()});
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 16

```
Memory Management ใน Zig - สรุป:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Step 151: Stack vs Heap                                 │
│  • Stack: เร็ว, จำกัด, อัตโนมัติ                       │
│  • Heap: ช้า, ใหญ่, ต้องจัดการเอง                      │
│                                                          │
│  Step 152: Stack Allocation                              │
│  • ตัวแปร local อยู่บน stack                            │
│  • จัดการอัตโนมัติตาม scope                             │
│                                                          │
│  Step 153: Allocator Pattern                             │
│  • ไม่มี global allocator                               │
│  • ส่ง allocator ผ่าน parameters                        │
│  • เลือก allocator ตามสถานการณ์                         │
│                                                          │
│  Step 154: alloc/free                                    │
│  • alloc(T, n) → []T                                    │
│  • free(slice) คืน memory                               │
│  • ใช้ defer เสมอ                                       │
│                                                          │
│  Step 155: create/destroy                                │
│  • create(T) → *T สำหรับ 1 object                      │
│  • destroy(ptr) คืน memory                              │
│  • เหมาะสำหรับ struct บน heap                           │
│                                                          │
│  Step 156: Memory Leaks                                  │
│  • ลืม free → leak                                      │
│  • Double free → undefined behavior                     │
│  • Use after free → undefined behavior                  │
│                                                          │
│  Step 157: defer                                         │
│  • defer free ทันทีหลัง alloc                          │
│  • errdefer สำหรับ error cases                          │
│  • LIFO execution order                                  │
│                                                          │
│  Step 158: realloc                                       │
│  • ปรับขนาด slice ที่มีอยู่                             │
│  • ใช้กับ dynamic arrays                                │
│  • ArrayList ใช้ realloc อัตโนมัติ                     │
│                                                          │
│  Step 159: Memory Safety                                 │
│  • Bounds checking ใน Debug                             │
│  • Optional types ป้องกัน null                          │
│  • FixedBufferAllocator สำหรับ embedded                  │
│                                                          │
│  Step 160: GeneralPurposeAllocator                       │
│  • ตรวจหา leaks, double-free, use-after-free           │
│  • Stack traces เพื่อ debug                             │
│  • testing.allocator สำหรับ tests                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### หลักการที่สำคัญ

1. **Explicit is better than implicit**: Zig บังคับให้ explicit เกี่ยวกับ memory
2. **Always defer free**: ใช้ `defer allocator.free()` ทันทีหลัง `alloc`
3. **errdefer for safety**: ใช้ `errdefer` เพื่อป้องกัน leak ใน error cases
4. **Choose the right allocator**: GPA สำหรับ debug, Arena สำหรับ batch, Fixed สำหรับ embedded
5. **Test with GPA**: ใช้ GPA ใน tests เพื่อตรวจหา leaks
6. **Own your memory**: รู้ว่าใครเป็นเจ้าของ memory และใครรับผิดชอบ free

### Allocator Selection Guide

```
เลือก Allocator อย่างไร?

╔═══════════════════════════════════════════════╗
║  Development / Debug                          ║
║  → GeneralPurposeAllocator                   ║
║  → ตรวจหา bugs ได้ดีที่สุด                  ║
╠═══════════════════════════════════════════════╣
║  Request-scoped / Parser / AST               ║
║  → ArenaAllocator                            ║
║  → allocate เยอะ free ครั้งเดียว             ║
╠═══════════════════════════════════════════════╣
║  Embedded / No OS / Stack-only               ║
║  → FixedBufferAllocator                      ║
║  → ใช้ buffer ที่กำหนดไว้                   ║
╠═══════════════════════════════════════════════╣
║  Unit Tests                                   ║
║  → testing.allocator                         ║
║  → ตรวจหา leaks อัตโนมัติ                   ║
╠═══════════════════════════════════════════════╣
║  Large blocks / OS-level                     ║
║  → page_allocator                            ║
║  → allocate ทีละ page                        ║
╚═══════════════════════════════════════════════╝
```

### ขั้นตอนถัดไป

ใน Part 17 เราจะศึกษา:
- **Pointers ขั้นสูง**: pointer arithmetic, C interop pointers
- **Comptime Memory**: compile-time memory strategies
- **Custom Allocators**: เขียน allocator ของตัวเอง
- **Memory Mapped Files**: จัดการไฟล์ขนาดใหญ่ผ่าน memory mapping

---

*Part 16 ครอบคลุม Steps 151-160 เกี่ยวกับ Memory Management ใน Zig*
*เขียนโดยใช้ Zig 0.13.0+*
