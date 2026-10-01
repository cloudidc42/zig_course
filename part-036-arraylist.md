# Part 36: std.ArrayList - Dynamic Arrays ใน Zig

## บทนำ

`std.ArrayList` คือโครงสร้างข้อมูลพื้นฐานที่สำคัญมากใน Zig สำหรับการจัดการ array ที่มีขนาดเปลี่ยนแปลงได้ (dynamic array) ซึ่งคล้ายกับ `Vec<T>` ใน Rust หรือ `ArrayList<T>` ใน Java บทนี้จะครอบคลุมการใช้งาน ArrayList อย่างละเอียดตั้งแต่พื้นฐานจนถึงการใช้งานจริง

---

## Step 351: std.ArrayList(T) พื้นฐาน

### ArrayList คืออะไร?

`std.ArrayList(T)` เป็น generic type ที่จัดการ dynamic array สำหรับ element ประเภท `T` โดยใช้ allocator จัดการหน่วยความจำ

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง ArrayList สำหรับ i32
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit(); // ต้องเรียก deinit เพื่อคืนหน่วยความจำ

    // เพิ่ม element
    try list.append(10);
    try list.append(20);
    try list.append(30);

    // แสดงผล
    std.debug.print("Length: {}\n", .{list.items.len});
    std.debug.print("Items: {any}\n", .{list.items});
}
```

### โครงสร้างของ ArrayList

```zig
// โครงสร้างภายในของ ArrayList (simplified)
pub fn ArrayList(comptime T: type) type {
    return struct {
        items: []T,           // slice ของ items ที่มีอยู่
        capacity: usize,      // ขนาดหน่วยความจำที่จองไว้
        allocator: Allocator, // allocator ที่ใช้

        pub fn init(allocator: Allocator) Self { ... }
        pub fn deinit(self: Self) void { ... }
        pub fn append(self: *Self, item: T) !void { ... }
        // ... และ methods อื่นๆ
    };
}
```

### การสร้าง ArrayList ด้วย capacity เริ่มต้น

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // สร้าง ArrayList พร้อม capacity เริ่มต้น 10 elements
    var list = try std.ArrayList(i32).initCapacity(allocator, 10);
    defer list.deinit();

    std.debug.print("Initial capacity: {}\n", .{list.capacity});
    std.debug.print("Initial length: {}\n", .{list.items.len});

    // เพิ่ม elements
    var i: i32 = 0;
    while (i < 5) : (i += 1) {
        try list.append(i * 10);
    }

    std.debug.print("After appending 5 items:\n", .{});
    std.debug.print("  Length: {}\n", .{list.items.len});
    std.debug.print("  Capacity: {}\n", .{list.capacity});
    std.debug.print("  Items: {any}\n", .{list.items});
}
```

### ประเภทของ ArrayList

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ArrayList สำหรับ types ต่างๆ
    var int_list = std.ArrayList(i32).init(allocator);
    defer int_list.deinit();

    var float_list = std.ArrayList(f64).init(allocator);
    defer float_list.deinit();

    var string_list = std.ArrayList([]const u8).init(allocator);
    defer string_list.deinit();

    // ArrayList สำหรับ struct
    const Point = struct { x: f32, y: f32 };
    var point_list = std.ArrayList(Point).init(allocator);
    defer point_list.deinit();

    try int_list.append(42);
    try float_list.append(3.14);
    try string_list.append("hello");
    try point_list.append(.{ .x = 1.0, .y = 2.0 });

    std.debug.print("Int: {}\n", .{int_list.items[0]});
    std.debug.print("Float: {d:.2}\n", .{float_list.items[0]});
    std.debug.print("String: {s}\n", .{string_list.items[0]});
    std.debug.print("Point: ({d:.1}, {d:.1})\n", .{
        point_list.items[0].x,
        point_list.items[0].y,
    });
}
```

---

## Step 352: append, appendSlice, insert

### การใช้ append

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // append เพิ่ม element เดียวที่ท้าย list
    try list.append(1);
    try list.append(2);
    try list.append(3);

    std.debug.print("After append: {any}\n", .{list.items});

    // appendAssumeCapacity - append โดยไม่ตรวจสอบ capacity (faster but unsafe)
    try list.ensureUnusedCapacity(3);
    list.appendAssumeCapacity(4);
    list.appendAssumeCapacity(5);
    list.appendAssumeCapacity(6);

    std.debug.print("After appendAssumeCapacity: {any}\n", .{list.items});
}
```

### การใช้ appendSlice

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // appendSlice เพิ่มหลาย elements จาก slice
    const slice1 = [_]i32{ 1, 2, 3 };
    try list.appendSlice(&slice1);

    const slice2 = [_]i32{ 4, 5, 6 };
    try list.appendSlice(&slice2);

    std.debug.print("After appendSlice: {any}\n", .{list.items});

    // appendNTimes - เพิ่ม element เดิมหลายครั้ง
    try list.appendNTimes(0, 3);
    std.debug.print("After appendNTimes(0, 3): {any}\n", .{list.items});
}
```

### การใช้ insert

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 1, 2, 4, 5 });
    std.debug.print("Before insert: {any}\n", .{list.items});

    // insert element ที่ตำแหน่งที่กำหนด
    try list.insert(2, 3); // แทรก 3 ที่ index 2
    std.debug.print("After insert(2, 3): {any}\n", .{list.items});

    // insertSlice - แทรก slice ที่ตำแหน่งที่กำหนด
    const new_items = [_]i32{ 10, 20 };
    try list.insertSlice(0, &new_items);
    std.debug.print("After insertSlice(0, {{10,20}}): {any}\n", .{list.items});
}
```

### appendWriter สำหรับ u8 ArrayList

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ArrayList(u8) สามารถใช้ writer interface ได้
    var buffer = std.ArrayList(u8).init(allocator);
    defer buffer.deinit();

    const writer = buffer.writer();
    try writer.print("Hello, {}!\n", .{42});
    try writer.writeAll("This is a test\n");
    try writer.writeByte('A');

    std.debug.print("Buffer content:\n{s}\n", .{buffer.items});
}
```

---

## Step 353: pop, remove, removeRange

### การใช้ pop

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 10, 20, 30, 40, 50 });
    std.debug.print("Initial: {any}\n", .{list.items});

    // pop ลบ element ท้ายสุดและ return มา
    if (list.popOrNull()) |item| {
        std.debug.print("Popped: {}\n", .{item});
    }
    std.debug.print("After pop: {any}\n", .{list.items});

    // pop แบบไม่ตรวจสอบ (อันตรายถ้า list ว่าง)
    const item = list.pop();
    std.debug.print("Pop result: {}\n", .{item});
    std.debug.print("After second pop: {any}\n", .{list.items});
}
```

### การใช้ remove

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 10, 20, 30, 40, 50 });
    std.debug.print("Initial: {any}\n", .{list.items});

    // remove ลบ element ที่ index และ return value
    // วิธีนี้ shift elements ที่เหลือ (O(n))
    const removed = list.orderedRemove(2);
    std.debug.print("Removed at index 2: {}\n", .{removed});
    std.debug.print("After orderedRemove: {any}\n", .{list.items});

    // swapRemove - เร็วกว่าแต่ไม่รักษาลำดับ (O(1))
    const swapped = list.swapRemove(1);
    std.debug.print("SwapRemoved at index 1: {}\n", .{swapped});
    std.debug.print("After swapRemove: {any}\n", .{list.items});
}
```

### การใช้ removeRange

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 });
    std.debug.print("Initial: {any}\n", .{list.items});

    // replaceRange - แทนที่ช่วง elements
    const replacement = [_]i32{ 20, 30 };
    try list.replaceRange(2, 4, &replacement);
    std.debug.print("After replaceRange(2, 4, {{20,30}}): {any}\n", .{list.items});

    // shrinkRetainingCapacity - ลดขนาดแต่ยังคง capacity
    list.shrinkRetainingCapacity(5);
    std.debug.print("After shrink to 5: {any}\n", .{list.items});
    std.debug.print("Capacity preserved: {}\n", .{list.capacity});
}
```

### clearAndFree vs clearRetainingCapacity

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 1, 2, 3, 4, 5 });
    const cap_before = list.capacity;

    // clearRetainingCapacity - ลบข้อมูลทั้งหมดแต่เก็บ capacity ไว้
    list.clearRetainingCapacity();
    std.debug.print("After clearRetainingCapacity:\n", .{});
    std.debug.print("  Length: {}, Capacity: {}\n", .{ list.items.len, list.capacity });
    std.debug.print("  Capacity preserved: {}\n", .{list.capacity == cap_before});

    try list.appendSlice(&[_]i32{ 10, 20, 30 });

    // clearAndFree - ลบข้อมูลทั้งหมดและคืนหน่วยความจำ
    list.clearAndFree();
    std.debug.print("After clearAndFree:\n", .{});
    std.debug.print("  Length: {}, Capacity: {}\n", .{ list.items.len, list.capacity });
}
```

---

## Step 354: ArrayList เป็น Stack และ Queue

### ArrayList เป็น Stack (LIFO)

```zig
const std = @import("std");

// Stack implementation ใช้ ArrayList
const Stack = struct {
    list: std.ArrayList(i32),

    pub fn init(allocator: std.mem.Allocator) Stack {
        return .{ .list = std.ArrayList(i32).init(allocator) };
    }

    pub fn deinit(self: *Stack) void {
        self.list.deinit();
    }

    pub fn push(self: *Stack, value: i32) !void {
        try self.list.append(value);
    }

    pub fn pop(self: *Stack) ?i32 {
        return self.list.popOrNull();
    }

    pub fn peek(self: *const Stack) ?i32 {
        if (self.list.items.len == 0) return null;
        return self.list.items[self.list.items.len - 1];
    }

    pub fn isEmpty(self: *const Stack) bool {
        return self.list.items.len == 0;
    }

    pub fn size(self: *const Stack) usize {
        return self.list.items.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var stack = Stack.init(allocator);
    defer stack.deinit();

    // Push elements
    try stack.push(1);
    try stack.push(2);
    try stack.push(3);

    std.debug.print("Stack size: {}\n", .{stack.size()});
    std.debug.print("Peek: {?}\n", .{stack.peek()});

    // Pop elements (LIFO order)
    while (stack.pop()) |item| {
        std.debug.print("Popped: {}\n", .{item});
    }
}
```

### ArrayList เป็น Queue (FIFO)

```zig
const std = @import("std");

// Simple Queue ใช้ ArrayList (ไม่ efficient สำหรับ large queues)
const SimpleQueue = struct {
    list: std.ArrayList(i32),

    pub fn init(allocator: std.mem.Allocator) SimpleQueue {
        return .{ .list = std.ArrayList(i32).init(allocator) };
    }

    pub fn deinit(self: *SimpleQueue) void {
        self.list.deinit();
    }

    pub fn enqueue(self: *SimpleQueue, value: i32) !void {
        try self.list.append(value);
    }

    // dequeue O(n) - ไม่ efficient แต่ใช้งานได้
    pub fn dequeue(self: *SimpleQueue) ?i32 {
        if (self.list.items.len == 0) return null;
        const item = self.list.items[0];
        _ = self.list.orderedRemove(0);
        return item;
    }

    pub fn front(self: *const SimpleQueue) ?i32 {
        if (self.list.items.len == 0) return null;
        return self.list.items[0];
    }

    pub fn isEmpty(self: *const SimpleQueue) bool {
        return self.list.items.len == 0;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var queue = SimpleQueue.init(allocator);
    defer queue.deinit();

    // Enqueue
    try queue.enqueue(100);
    try queue.enqueue(200);
    try queue.enqueue(300);

    std.debug.print("Front: {?}\n", .{queue.front()});

    // Dequeue (FIFO order)
    while (queue.dequeue()) |item| {
        std.debug.print("Dequeued: {}\n", .{item});
    }
}
```

### ตัวอย่าง: ใช้ ArrayList สำหรับ Undo/Redo

```zig
const std = @import("std");

const Action = struct {
    description: []const u8,
    value: i32,
};

const UndoRedoManager = struct {
    history: std.ArrayList(Action),
    redo_stack: std.ArrayList(Action),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) UndoRedoManager {
        return .{
            .history = std.ArrayList(Action).init(allocator),
            .redo_stack = std.ArrayList(Action).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *UndoRedoManager) void {
        self.history.deinit();
        self.redo_stack.deinit();
    }

    pub fn doAction(self: *UndoRedoManager, action: Action) !void {
        try self.history.append(action);
        self.redo_stack.clearRetainingCapacity();
        std.debug.print("Did: {s} (value: {})\n", .{ action.description, action.value });
    }

    pub fn undo(self: *UndoRedoManager) !void {
        if (self.history.popOrNull()) |action| {
            try self.redo_stack.append(action);
            std.debug.print("Undid: {s}\n", .{action.description});
        } else {
            std.debug.print("Nothing to undo\n", .{});
        }
    }

    pub fn redo(self: *UndoRedoManager) !void {
        if (self.redo_stack.popOrNull()) |action| {
            try self.history.append(action);
            std.debug.print("Redid: {s}\n", .{action.description});
        } else {
            std.debug.print("Nothing to redo\n", .{});
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var manager = UndoRedoManager.init(allocator);
    defer manager.deinit();

    try manager.doAction(.{ .description = "Set x = 10", .value = 10 });
    try manager.doAction(.{ .description = "Set x = 20", .value = 20 });
    try manager.doAction(.{ .description = "Set x = 30", .value = 30 });

    try manager.undo();
    try manager.undo();
    try manager.redo();
}
```

---

## Step 355: items slice และ toOwnedSlice

### การเข้าถึง items slice

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 10, 20, 30, 40, 50 });

    // items เป็น []T slice ที่ point ไปยัง internal buffer
    const items = list.items;
    std.debug.print("items type: slice of i32\n", .{});
    std.debug.print("items.len: {}\n", .{items.len});
    std.debug.print("items[0]: {}\n", .{items[0]});

    // แก้ไขผ่าน items slice โดยตรง
    list.items[2] = 99;
    std.debug.print("After modifying items[2]: {any}\n", .{list.items});

    // slice ย่อย
    const sub_slice = list.items[1..4];
    std.debug.print("Sub-slice [1..4]: {any}\n", .{sub_slice});

    // อันตราย: items อาจ invalid หลัง append (reallocation)
    const items_before_append = list.items;
    try list.append(999); // อาจทำให้ realloc
    _ = items_before_append; // items เก่าอาจ dangling pointer
}
```

### การใช้ toOwnedSlice

```zig
const std = @import("std");

pub fn createNumbers(allocator: std.mem.Allocator, count: usize) ![]i32 {
    var list = std.ArrayList(i32).init(allocator);
    // ไม่ต้องเรียก defer list.deinit() เพราะจะ transfer ownership

    var i: i32 = 0;
    while (i < @as(i32, @intCast(count))) : (i += 1) {
        try list.append(i * i); // i^2
    }

    // toOwnedSlice โอน ownership ของ memory ไปยัง caller
    // ArrayList จะกลายเป็น empty หลังจากนี้
    return list.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // caller รับ ownership และต้องจัดการ free เอง
    const numbers = try createNumbers(allocator, 10);
    defer allocator.free(numbers);

    std.debug.print("Squares: {any}\n", .{numbers});
    std.debug.print("Type: []i32, len: {}\n", .{numbers.len});
}
```

### toOwnedSliceSentinel

```zig
const std = @import("std");

pub fn createCString(allocator: std.mem.Allocator, content: []const u8) ![:0]u8 {
    var list = std.ArrayList(u8).init(allocator);

    try list.appendSlice(content);

    // สร้าง null-terminated string
    return list.toOwnedSliceSentinel(0);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const cstr = try createCString(allocator, "Hello, World");
    defer allocator.free(cstr);

    std.debug.print("C string: {s}\n", .{cstr});
    std.debug.print("Length: {}\n", .{cstr.len});
    std.debug.print("Null terminator: {}\n", .{cstr[cstr.len]});
}
```

---

## Step 356: Capacity Management

### การจัดการ Capacity

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    std.debug.print("=== Capacity Growth ===\n", .{});

    // สังเกต capacity เปลี่ยนแปลงขณะ append
    var i: usize = 0;
    var last_capacity: usize = 0;
    while (i < 20) : (i += 1) {
        try list.append(@as(i32, @intCast(i)));
        if (list.capacity != last_capacity) {
            std.debug.print("len={:2}, capacity={:2} (grew from {})\n", .{
                list.items.len,
                list.capacity,
                last_capacity,
            });
            last_capacity = list.capacity;
        }
    }
}
```

### ensureTotalCapacity และ ensureUnusedCapacity

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // ensureTotalCapacity - รับประกันว่ามี capacity อย่างน้อยเท่าที่กำหนด
    try list.ensureTotalCapacity(100);
    std.debug.print("After ensureTotalCapacity(100): cap={}\n", .{list.capacity});

    // ensureUnusedCapacity - รับประกัน unused capacity อย่างน้อยเท่าที่กำหนด
    try list.append(1);
    try list.append(2);
    try list.ensureUnusedCapacity(50);
    std.debug.print("After ensureUnusedCapacity(50): cap={}\n", .{list.capacity});

    // resize - เปลี่ยนขนาด list
    try list.resize(10);
    std.debug.print("After resize(10): len={}, cap={}\n", .{
        list.items.len,
        list.capacity,
    });

    // shrinkAndFree - ลด capacity ให้เท่ากับ length
    try list.appendSlice(&[_]i32{ 100, 200, 300 });
    std.debug.print("Before shrink: len={}, cap={}\n", .{
        list.items.len,
        list.capacity,
    });
    list.shrinkAndFree(list.items.len);
    std.debug.print("After shrinkAndFree: len={}, cap={}\n", .{
        list.items.len,
        list.capacity,
    });
}
```

### Pre-allocation pattern สำหรับ performance

```zig
const std = @import("std");
const Timer = std.time.Timer;

pub fn withoutPrealloc(allocator: std.mem.Allocator, count: usize) !void {
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    var i: usize = 0;
    while (i < count) : (i += 1) {
        try list.append(@as(i32, @intCast(i)));
    }
}

pub fn withPrealloc(allocator: std.mem.Allocator, count: usize) !void {
    var list = try std.ArrayList(i32).initCapacity(allocator, count);
    defer list.deinit();

    var i: usize = 0;
    while (i < count) : (i += 1) {
        list.appendAssumeCapacity(@as(i32, @intCast(i)));
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const count = 1_000_000;

    var timer = try Timer.start();

    try withoutPrealloc(allocator, count);
    const t1 = timer.read();

    timer.reset();
    try withPrealloc(allocator, count);
    const t2 = timer.read();

    std.debug.print("Without pre-alloc: {}ms\n", .{t1 / 1_000_000});
    std.debug.print("With pre-alloc:    {}ms\n", .{t2 / 1_000_000});
    std.debug.print("Speedup: {d:.2}x\n", .{@as(f64, @floatFromInt(t1)) / @as(f64, @floatFromInt(t2))});
}
```

---

## Step 357: ArrayList Iteration

### การ Iterate ด้วยวิธีต่างๆ

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 10, 20, 30, 40, 50 });

    // วิธีที่ 1: for loop ด้วย items
    std.debug.print("Method 1 - for loop:\n", .{});
    for (list.items) |item| {
        std.debug.print("  {}\n", .{item});
    }

    // วิธีที่ 2: for loop พร้อม index
    std.debug.print("Method 2 - with index:\n", .{});
    for (list.items, 0..) |item, idx| {
        std.debug.print("  [{}] = {}\n", .{ idx, item });
    }

    // วิธีที่ 3: while loop ด้วย pointer
    std.debug.print("Method 3 - while with ptr:\n", .{});
    var ptr: [*]i32 = list.items.ptr;
    var i: usize = 0;
    while (i < list.items.len) : (i += 1) {
        std.debug.print("  {}\n", .{ptr[i]});
    }

    // วิธีที่ 4: แก้ไขขณะ iterate
    std.debug.print("Method 4 - modify while iterating:\n", .{});
    for (list.items) |*item| {
        item.* *= 2;
    }
    std.debug.print("After doubling: {any}\n", .{list.items});
}
```

### Reverse Iteration

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 1, 2, 3, 4, 5 });

    // Reverse iteration
    std.debug.print("Reverse iteration:\n", .{});
    var i: usize = list.items.len;
    while (i > 0) {
        i -= 1;
        std.debug.print("  {}\n", .{list.items[i]});
    }

    // std.mem.reverse ใช้ reverse slice
    std.mem.reverse(i32, list.items);
    std.debug.print("Reversed list: {any}\n", .{list.items});
}
```

### Filtering และ Mapping

```zig
const std = @import("std");

pub fn filter(comptime T: type, allocator: std.mem.Allocator, list: std.ArrayList(T), pred: fn (T) bool) !std.ArrayList(T) {
    var result = std.ArrayList(T).init(allocator);
    for (list.items) |item| {
        if (pred(item)) {
            try result.append(item);
        }
    }
    return result;
}

pub fn map(comptime T: type, comptime U: type, allocator: std.mem.Allocator, list: std.ArrayList(T), f: fn (T) U) !std.ArrayList(U) {
    var result = try std.ArrayList(U).initCapacity(allocator, list.items.len);
    for (list.items) |item| {
        result.appendAssumeCapacity(f(item));
    }
    return result;
}

fn isEven(x: i32) bool {
    return @mod(x, 2) == 0;
}

fn square(x: i32) i32 {
    return x * x;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 });

    // Filter even numbers
    var evens = try filter(i32, allocator, list, isEven);
    defer evens.deinit();
    std.debug.print("Even numbers: {any}\n", .{evens.items});

    // Map to squares
    var squares = try map(i32, i32, allocator, list, square);
    defer squares.deinit();
    std.debug.print("Squares: {any}\n", .{squares.items});
}
```

---

## Step 358: std.ArrayListUnmanaged

### ArrayListUnmanaged คืออะไร?

`ArrayListUnmanaged` คือ version ของ ArrayList ที่ไม่เก็บ allocator ไว้ภายใน ทำให้ขนาด struct เล็กลงแต่ต้องส่ง allocator ทุกครั้งที่เรียก method

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ArrayListUnmanaged ไม่เก็บ allocator
    var list = std.ArrayListUnmanaged(i32){};
    defer list.deinit(allocator); // ต้องส่ง allocator

    // ทุก method ที่ allocate ต้องส่ง allocator
    try list.append(allocator, 10);
    try list.append(allocator, 20);
    try list.append(allocator, 30);

    std.debug.print("Items: {any}\n", .{list.items});
    std.debug.print("Capacity: {}\n", .{list.capacity});

    // appendSlice
    try list.appendSlice(allocator, &[_]i32{ 40, 50 });
    std.debug.print("After appendSlice: {any}\n", .{list.items});
}
```

### เปรียบเทียบขนาด

```zig
const std = @import("std");

pub fn main() void {
    // ArrayList เก็บ allocator ไว้ด้วย
    const managed_size = @sizeOf(std.ArrayList(i32));
    // ArrayListUnmanaged ไม่เก็บ allocator
    const unmanaged_size = @sizeOf(std.ArrayListUnmanaged(i32));

    std.debug.print("ArrayList size: {} bytes\n", .{managed_size});
    std.debug.print("ArrayListUnmanaged size: {} bytes\n", .{unmanaged_size});
    std.debug.print("Allocator size: {} bytes\n", .{managed_size - unmanaged_size});
}
```

### ใช้ ArrayListUnmanaged ใน struct

```zig
const std = @import("std");

// struct ที่ใช้ ArrayListUnmanaged เพื่อประหยัดหน่วยความจำ
const ItemCollection = struct {
    items: std.ArrayListUnmanaged(u32) = .{},

    pub fn add(self: *ItemCollection, allocator: std.mem.Allocator, item: u32) !void {
        try self.items.append(allocator, item);
    }

    pub fn remove(self: *ItemCollection, index: usize) u32 {
        return self.items.swapRemove(index);
    }

    pub fn deinit(self: *ItemCollection, allocator: std.mem.Allocator) void {
        self.items.deinit(allocator);
    }

    pub fn count(self: *const ItemCollection) usize {
        return self.items.items.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var collection = ItemCollection{};
    defer collection.deinit(allocator);

    try collection.add(allocator, 100);
    try collection.add(allocator, 200);
    try collection.add(allocator, 300);

    std.debug.print("Count: {}\n", .{collection.count()});
    std.debug.print("Items: {any}\n", .{collection.items.items});

    const removed = collection.remove(1);
    std.debug.print("Removed: {}\n", .{removed});
    std.debug.print("After remove: {any}\n", .{collection.items.items});
}
```

---

## Step 359: 2D ArrayList (ArrayList ใน ArrayList)

### การสร้าง 2D ArrayList

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // 2D ArrayList: ArrayList ของ ArrayList
    var matrix = std.ArrayList(std.ArrayList(i32)).init(allocator);
    defer {
        // ต้อง deinit inner lists ก่อน
        for (matrix.items) |*row| {
            row.deinit();
        }
        matrix.deinit();
    }

    // สร้าง matrix 3x4
    var row_idx: usize = 0;
    while (row_idx < 3) : (row_idx += 1) {
        var row = std.ArrayList(i32).init(allocator);
        var col_idx: usize = 0;
        while (col_idx < 4) : (col_idx += 1) {
            try row.append(@as(i32, @intCast(row_idx * 4 + col_idx)));
        }
        try matrix.append(row);
    }

    // แสดง matrix
    std.debug.print("Matrix ({}x{}):\n", .{ matrix.items.len, matrix.items[0].items.len });
    for (matrix.items) |row| {
        for (row.items) |item| {
            std.debug.print("{:3}", .{item});
        }
        std.debug.print("\n", .{});
    }
}
```

### Dynamic 2D Structure

```zig
const std = @import("std");

const DynamicGrid = struct {
    rows: std.ArrayList(std.ArrayList(f64)),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) DynamicGrid {
        return .{
            .rows = std.ArrayList(std.ArrayList(f64)).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *DynamicGrid) void {
        for (self.rows.items) |*row| {
            row.deinit();
        }
        self.rows.deinit();
    }

    pub fn addRow(self: *DynamicGrid) !*std.ArrayList(f64) {
        try self.rows.append(std.ArrayList(f64).init(self.allocator));
        return &self.rows.items[self.rows.items.len - 1];
    }

    pub fn get(self: *const DynamicGrid, row: usize, col: usize) ?f64 {
        if (row >= self.rows.items.len) return null;
        const r = self.rows.items[row];
        if (col >= r.items.len) return null;
        return r.items[col];
    }

    pub fn set(self: *DynamicGrid, row: usize, col: usize, value: f64) void {
        self.rows.items[row].items[col] = value;
    }

    pub fn rowCount(self: *const DynamicGrid) usize {
        return self.rows.items.len;
    }

    pub fn colCount(self: *const DynamicGrid, row: usize) usize {
        if (row >= self.rows.items.len) return 0;
        return self.rows.items[row].items.len;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var grid = DynamicGrid.init(allocator);
    defer grid.deinit();

    // เพิ่ม rows และ columns แบบ dynamic
    var row1 = try grid.addRow();
    try row1.appendSlice(&[_]f64{ 1.0, 2.0, 3.0 });

    var row2 = try grid.addRow();
    try row2.appendSlice(&[_]f64{ 4.0, 5.0 });

    var row3 = try grid.addRow();
    try row3.appendSlice(&[_]f64{ 6.0, 7.0, 8.0, 9.0 });

    // Grid ไม่จำเป็นต้องเป็น rectangular
    for (0..grid.rowCount()) |r| {
        for (0..grid.colCount(r)) |c| {
            if (grid.get(r, c)) |val| {
                std.debug.print("{d:.1} ", .{val});
            }
        }
        std.debug.print("\n", .{});
    }
}
```

---

## Step 360: Real-world ArrayList Patterns

### Pattern 1: Builder Pattern

```zig
const std = @import("std");

const QueryBuilder = struct {
    parts: std.ArrayList([]const u8),
    params: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) QueryBuilder {
        return .{
            .parts = std.ArrayList([]const u8).init(allocator),
            .params = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *QueryBuilder) void {
        self.parts.deinit();
        self.params.deinit();
    }

    pub fn select(self: *QueryBuilder, table: []const u8) !*QueryBuilder {
        try self.parts.append("SELECT * FROM ");
        try self.parts.append(table);
        return self;
    }

    pub fn where(self: *QueryBuilder, condition: []const u8) !*QueryBuilder {
        try self.parts.append(" WHERE ");
        try self.parts.append(condition);
        return self;
    }

    pub fn limit(self: *QueryBuilder, n: usize) !*QueryBuilder {
        const s = try std.fmt.allocPrint(self.allocator, " LIMIT {}", .{n});
        try self.parts.append(s);
        try self.params.append(s); // track for deallocation
        return self;
    }

    pub fn build(self: *const QueryBuilder, allocator: std.mem.Allocator) ![]u8 {
        var total_len: usize = 0;
        for (self.parts.items) |part| {
            total_len += part.len;
        }

        var result = try allocator.alloc(u8, total_len);
        var offset: usize = 0;
        for (self.parts.items) |part| {
            @memcpy(result[offset..offset + part.len], part);
            offset += part.len;
        }
        return result;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var qb = QueryBuilder.init(allocator);
    defer qb.deinit();

    _ = try (try (try qb.select("users")).where("age > 18")).limit(10);

    const query = try qb.build(allocator);
    defer allocator.free(query);

    std.debug.print("Query: {s}\n", .{query});
}
```

### Pattern 2: Event System

```zig
const std = @import("std");

const EventType = enum { click, hover, keypress };

const EventHandler = *const fn (EventType) void;

const EventEmitter = struct {
    handlers: std.ArrayList(struct { event: EventType, handler: EventHandler }),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) EventEmitter {
        return .{
            .handlers = std.ArrayList(struct { event: EventType, handler: EventHandler }).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *EventEmitter) void {
        self.handlers.deinit();
    }

    pub fn on(self: *EventEmitter, event: EventType, handler: EventHandler) !void {
        try self.handlers.append(.{ .event = event, .handler = handler });
    }

    pub fn emit(self: *const EventEmitter, event: EventType) void {
        for (self.handlers.items) |entry| {
            if (entry.event == event) {
                entry.handler(event);
            }
        }
    }
};

fn onClickHandler(event: EventType) void {
    std.debug.print("Click handler called for event: {}\n", .{event});
}

fn onHoverHandler(event: EventType) void {
    std.debug.print("Hover handler called for event: {}\n", .{event});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var emitter = EventEmitter.init(allocator);
    defer emitter.deinit();

    try emitter.on(.click, onClickHandler);
    try emitter.on(.hover, onHoverHandler);
    try emitter.on(.click, onClickHandler); // handlers หลายตัวสำหรับ event เดียว

    emitter.emit(.click);
    emitter.emit(.hover);
    emitter.emit(.keypress); // ไม่มี handler
}
```

### Pattern 3: Sorted ArrayList

```zig
const std = @import("std");

fn SortedList(comptime T: type, comptime lessThan: fn (T, T) bool) type {
    return struct {
        list: std.ArrayList(T),

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{ .list = std.ArrayList(T).init(allocator) };
        }

        pub fn deinit(self: *Self) void {
            self.list.deinit();
        }

        pub fn insert(self: *Self, value: T) !void {
            // Binary search สำหรับตำแหน่งที่ถูกต้อง
            var lo: usize = 0;
            var hi: usize = self.list.items.len;

            while (lo < hi) {
                const mid = lo + (hi - lo) / 2;
                if (lessThan(self.list.items[mid], value)) {
                    lo = mid + 1;
                } else {
                    hi = mid;
                }
            }

            try self.list.insert(lo, value);
        }

        pub fn contains(self: *const Self, value: T) bool {
            var lo: usize = 0;
            var hi: usize = self.list.items.len;

            while (lo < hi) {
                const mid = lo + (hi - lo) / 2;
                if (lessThan(self.list.items[mid], value)) {
                    lo = mid + 1;
                } else if (lessThan(value, self.list.items[mid])) {
                    hi = mid;
                } else {
                    return true;
                }
            }
            return false;
        }
    };
}

fn intLessThan(a: i32, b: i32) bool {
    return a < b;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var sorted = SortedList(i32, intLessThan).init(allocator);
    defer sorted.deinit();

    // Insert ใน order สุ่ม
    try sorted.insert(5);
    try sorted.insert(2);
    try sorted.insert(8);
    try sorted.insert(1);
    try sorted.insert(9);
    try sorted.insert(3);

    // List จะเรียงลำดับเสมอ
    std.debug.print("Sorted: {any}\n", .{sorted.list.items});
    std.debug.print("Contains 5: {}\n", .{sorted.contains(5)});
    std.debug.print("Contains 7: {}\n", .{sorted.contains(7)});
}
```

### Pattern 4: Object Pool

```zig
const std = @import("std");

const Connection = struct {
    id: usize,
    in_use: bool,

    pub fn query(self: *Connection, sql: []const u8) void {
        std.debug.print("Connection {}: executing '{s}'\n", .{ self.id, sql });
    }
};

const ConnectionPool = struct {
    connections: std.ArrayList(Connection),
    allocator: std.mem.Allocator,
    next_id: usize,

    pub fn init(allocator: std.mem.Allocator, pool_size: usize) !ConnectionPool {
        var pool = ConnectionPool{
            .connections = try std.ArrayList(Connection).initCapacity(allocator, pool_size),
            .allocator = allocator,
            .next_id = 0,
        };

        var i: usize = 0;
        while (i < pool_size) : (i += 1) {
            pool.connections.appendAssumeCapacity(.{ .id = i, .in_use = false });
            pool.next_id = i + 1;
        }

        return pool;
    }

    pub fn deinit(self: *ConnectionPool) void {
        self.connections.deinit();
    }

    pub fn acquire(self: *ConnectionPool) ?*Connection {
        for (self.connections.items) |*conn| {
            if (!conn.in_use) {
                conn.in_use = true;
                return conn;
            }
        }
        return null;
    }

    pub fn release(self: *ConnectionPool, conn: *Connection) void {
        _ = self;
        conn.in_use = false;
    }

    pub fn availableCount(self: *const ConnectionPool) usize {
        var count: usize = 0;
        for (self.connections.items) |conn| {
            if (!conn.in_use) count += 1;
        }
        return count;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var pool = try ConnectionPool.init(allocator, 3);
    defer pool.deinit();

    std.debug.print("Available connections: {}\n", .{pool.availableCount()});

    if (pool.acquire()) |conn1| {
        conn1.query("SELECT * FROM users");

        if (pool.acquire()) |conn2| {
            conn2.query("INSERT INTO logs VALUES (...)");
            pool.release(conn2);
        }

        pool.release(conn1);
    }

    std.debug.print("After release, available: {}\n", .{pool.availableCount()});
}
```

---

## สรุปบทที่ 36

ในบทนี้เราได้เรียนรู้:

1. **ArrayList พื้นฐาน** - การสร้าง, init, deinit, และโครงสร้างภายใน
2. **การเพิ่ม elements** - append, appendSlice, insert พร้อมตัวเลือก AssumeCapacity
3. **การลบ elements** - pop, remove, orderedRemove, swapRemove, replaceRange
4. **Stack และ Queue** - การนำ ArrayList มาสร้าง data structures อื่น
5. **items slice และ toOwnedSlice** - การจัดการ ownership ของ memory
6. **Capacity management** - ensureTotalCapacity, ensureUnusedCapacity, pre-allocation
7. **Iteration** - วิธีต่างๆ ในการ iterate รวมถึง reverse iteration
8. **ArrayListUnmanaged** - version ที่ประหยัด memory กว่า
9. **2D ArrayList** - nested ArrayList สำหรับ matrix และ grid
10. **Real-world patterns** - Builder, Event System, Sorted List, Object Pool

`std.ArrayList` เป็น building block ที่สำคัญมากใน Zig และเข้าใจการทำงานของมันอย่างลึกซึ้งจะช่วยให้เขียนโค้ดได้มีประสิทธิภาพมากขึ้น
