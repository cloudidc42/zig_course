# Part 17: Allocators ใน Zig (Steps 161-170)

Zig มีแนวคิดที่ทรงพลังอย่างหนึ่งคือ **Allocator interface** — แทนที่จะใช้ `malloc`/`free` โดยตรงเหมือน C, Zig บังคับให้คุณระบุ allocator อย่างชัดเจนทุกครั้งที่ต้องการ dynamic memory การออกแบบนี้ทำให้โค้ดชัดเจน ทดสอบง่าย และควบคุม memory ได้ดีกว่า

---

## Step 161: Allocator Interface คืออะไร?

### แนวคิดหลัก

ใน Zig, `std.mem.Allocator` คือ **interface** (ในรูปแบบ struct ที่มี function pointers) สำหรับการจัดการ memory แบบ dynamic ทุก allocator ใน standard library และ third-party library ใช้ interface เดียวกัน ทำให้สามารถสลับ allocator ได้โดยไม่ต้องแก้โค้ด

```zig
const std = @import("std");

pub fn main() !void {
    // GeneralPurposeAllocator เป็นหนึ่งใน allocator ที่ Zig มีให้
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // เราได้ allocator interface มาจาก .allocator()
    const allocator = gpa.allocator();

    // ใช้ allocate memory
    const buffer = try allocator.alloc(u8, 1024);
    defer allocator.free(buffer);

    std.debug.print("Allocated {} bytes\n", .{buffer.len});
}
```

### Allocator Interface มีอะไรบ้าง?

```zig
// std.mem.Allocator คือ struct นี้ (simplified)
pub const Allocator = struct {
    ptr: *anyopaque,
    vtable: *const VTable,

    pub const VTable = struct {
        alloc: *const fn (ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8,
        resize: *const fn (ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool,
        free: *const fn (ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void,
    };
};
```

### Methods หลักของ Allocator

```zig
const std = @import("std");

pub fn demonstrateAllocatorMethods() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // 1. alloc - สร้าง slice ของ type ที่กำหนด
    const numbers = try allocator.alloc(i32, 10);
    defer allocator.free(numbers);
    std.debug.print("alloc: {} integers\n", .{numbers.len});

    // 2. create - สร้าง pointer ไปยัง single value
    const single = try allocator.create(i32);
    defer allocator.destroy(single);
    single.* = 42;
    std.debug.print("create: value = {}\n", .{single.*});

    // 3. realloc - เปลี่ยนขนาด (อาจย้าย memory)
    const grown = try allocator.realloc(numbers, 20);
    defer allocator.free(grown);
    std.debug.print("realloc: {} integers\n", .{grown.len});

    // 4. dupe - copy slice และ allocate memory ใหม่
    const original = "Hello, Zig!";
    const copy = try allocator.dupe(u8, original);
    defer allocator.free(copy);
    std.debug.print("dupe: {s}\n", .{copy});

    // 5. dupeZ - copy slice และเพิ่ม null terminator (สำหรับ C strings)
    const c_string = try allocator.dupeZ(u8, "C string");
    defer allocator.free(c_string);
    std.debug.print("dupeZ: {s}\n", .{c_string});
}

pub fn main() !void {
    try demonstrateAllocatorMethods();
}
```

### ทำไม Zig ถึงออกแบบแบบนี้?

1. **Explicit is better than implicit** - คุณรู้ว่า memory ถูก allocate ที่ไหน
2. **Testability** - ใช้ `std.testing.allocator` ใน tests เพื่อ detect memory leaks อัตโนมัติ
3. **Flexibility** - เปลี่ยน allocator strategy ได้โดยไม่ต้องแก้ business logic
4. **No global state** - ไม่มี hidden global malloc/free
5. **Performance** - เลือก allocator ที่เหมาะสมกับ use case

---

## Step 162: GeneralPurposeAllocator - สำหรับ Debug

`GeneralPurposeAllocator` (GPA) เป็น allocator ที่ใช้สำหรับ **development และ debugging** มันตรวจจับ:
- Memory leaks
- Double frees
- Use-after-free

### การใช้งานพื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง GPA ด้วย default configuration
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    // deinit คืน enum ที่บอกว่ามี leak หรือไม่
    defer {
        const result = gpa.deinit();
        if (result == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }

    const allocator = gpa.allocator();

    // Allocate memory
    const data = try allocator.alloc(u8, 100);
    // ถ้าเราลืม free data, deinit จะแจ้ง leak
    defer allocator.free(data);

    std.debug.print("GPA allocated {} bytes successfully\n", .{data.len});
}
```

### GPA Configuration Options

```zig
const std = @import("std");

pub fn main() !void {
    // GPA รับ comptime config struct
    var gpa = std.heap.GeneralPurposeAllocator(.{
        // เปิด/ปิด safety checks (default: true ใน debug mode)
        .safety = true,
        // เปิด stack traces สำหรับ allocation
        .stack_trace_frames = 10,
        // ใช้ verbose logging
        .verbose_log = false,
        // กำหนด backing allocator (default: page_allocator)
        // .backing_allocator = std.heap.page_allocator,
    }){};
    defer _ = gpa.deinit();

    const allocator = gpa.allocator();

    // Demo: GPA ตรวจจับ double free
    const buf = try allocator.alloc(u8, 64);
    allocator.free(buf);

    // ถ้า uncomment บรรทัดนี้ จะ panic ด้วย "double free"
    // allocator.free(buf);

    std.debug.print("GPA with custom config works!\n", .{});
}
```

### ตรวจจับ Memory Leak

```zig
const std = @import("std");

// ฟังก์ชันที่มี bug: ลืม free memory
fn leakyFunction(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    // bug: ลืม defer allocator.free(data);
    for (data, 0..) |*v, i| {
        v.* = @intCast(i);
    }
    std.debug.print("leakyFunction: first value = {}\n", .{data[0]});
    // data ไม่ถูก free! -> memory leak
}

// ฟังก์ชันที่ถูกต้อง: free memory เสมอ
fn properFunction(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(i32, 10);
    defer allocator.free(data); // <- ถูกต้อง
    for (data, 0..) |*v, i| {
        v.* = @intCast(i);
    }
    std.debug.print("properFunction: first value = {}\n", .{data[0]});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const check = gpa.deinit();
        switch (check) {
            .ok => std.debug.print("No memory leaks!\n", .{}),
            .leak => std.debug.print("WARNING: Memory leaks detected!\n", .{}),
        }
    }

    const allocator = gpa.allocator();

    // ลองเรียกฟังก์ชันที่ถูกต้องก่อน
    try properFunction(allocator);

    // ถ้า uncomment บรรทัดนี้ จะมี leak
    // try leakyFunction(allocator);
}
```

### GPA กับ Testing

```zig
const std = @import("std");
const testing = std.testing;

fn createList(allocator: std.mem.Allocator, items: []const u32) ![]u32 {
    const result = try allocator.alloc(u32, items.len);
    @memcpy(result, items);
    return result;
}

test "createList allocates correctly" {
    // std.testing.allocator ใช้ GPA ภายใน และ fail ถ้ามี leak
    const allocator = testing.allocator;

    const items = [_]u32{ 1, 2, 3, 4, 5 };
    const list = try createList(allocator, &items);
    defer allocator.free(list);

    try testing.expectEqual(@as(usize, 5), list.len);
    try testing.expectEqual(@as(u32, 1), list[0]);
    try testing.expectEqual(@as(u32, 5), list[4]);
}
```

### เมื่อใดควรใช้ GPA?

- **Development และ debugging** ทุกครั้ง
- **Testing** ใช้ `std.testing.allocator`
- **Production** ถ้า safety สำคัญกว่า performance
- **ไม่เหมาะ** สำหรับ performance-critical code (มี overhead จาก safety checks)

---

## Step 163: ArenaAllocator - Allocate เยอะ Free ครั้งเดียว

`ArenaAllocator` เป็น allocator ที่เหมาะสำหรับงานที่ต้อง **allocate หลายชิ้นแล้ว free ทีเดียว** เช่น การ parse request, build tree structure, หรือ render frame

### แนวคิดของ Arena

```
ArenaAllocator:
┌─────────────────────────────────────────┐
│ backing allocator (เช่น GPA)            │
│  ┌───────────────────────────────────┐  │
│  │ Arena Buffer 1                    │  │
│  │ [alloc1][alloc2][alloc3]...       │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ Arena Buffer 2 (เมื่อ 1 เต็ม)    │  │
│  │ [alloc4][alloc5]...               │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

deinit() -> free ทุก buffer ทีเดียว (fast!)
```

### การใช้งานพื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // สร้าง ArenaAllocator โดยใช้ GPA เป็น backing allocator
    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    // deinit จะ free memory ทั้งหมดที่ allocate ผ่าน arena
    defer arena.deinit();

    const allocator = arena.allocator();

    // Allocate เยอะ ๆ โดยไม่ต้อง free แต่ละชิ้น
    const str1 = try allocator.dupe(u8, "Hello");
    const str2 = try allocator.dupe(u8, "World");
    const numbers = try allocator.alloc(i32, 100);

    // ไม่ต้อง defer free สำหรับแต่ละ allocation!
    // arena.deinit() จะ free ทั้งหมด

    std.debug.print("{s} {s}, {} numbers\n", .{ str1, str2, numbers.len });
}
```

### รูปแบบ Request/Response

```zig
const std = @import("std");

const Request = struct {
    method: []const u8,
    path: []const u8,
    body: []const u8,
};

const Response = struct {
    status: u16,
    body: []const u8,
};

// Arena เหมาะมากสำหรับ request handling
fn handleRequest(
    parent_allocator: std.mem.Allocator,
    raw_request: []const u8,
) !Response {
    // สร้าง arena สำหรับ request นี้
    var arena = std.heap.ArenaAllocator.init(parent_allocator);
    defer arena.deinit(); // free ทั้งหมดเมื่อ request จบ

    const alloc = arena.allocator();

    // Parse request (allocate หลายชิ้น)
    const method = try alloc.dupe(u8, "GET");
    const path = try alloc.dupe(u8, raw_request[0..@min(raw_request.len, 50)]);
    const body = try alloc.alloc(u8, 0);

    const req = Request{
        .method = method,
        .path = path,
        .body = body,
    };

    // Process และ build response (allocate เพิ่มเติม)
    var response_body = std.ArrayList(u8).init(alloc);
    try response_body.appendSlice("HTTP/1.1 200 OK\r\n\r\n");
    try response_body.appendSlice("Hello from path: ");
    try response_body.appendSlice(req.path);

    // ต้อง copy response ออกก่อน arena.deinit()
    const final_body = try parent_allocator.dupe(u8, response_body.items);

    return Response{
        .status = 200,
        .body = final_body,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const response = try handleRequest(allocator, "/api/users");
    defer allocator.free(response.body);

    std.debug.print("Response status: {}\n", .{response.status});
    std.debug.print("Response body: {s}\n", .{response.body});
}
```

### Arena กับ Tree Building

```zig
const std = @import("std");

const Node = struct {
    value: i32,
    children: []Node,
};

fn buildTree(allocator: std.mem.Allocator, depth: usize) !Node {
    if (depth == 0) {
        return Node{ .value = 0, .children = &.{} };
    }

    const children = try allocator.alloc(Node, 2);
    children[0] = try buildTree(allocator, depth - 1);
    children[1] = try buildTree(allocator, depth - 1);

    return Node{
        .value = @intCast(depth),
        .children = children,
    };
}

fn countNodes(node: Node) usize {
    var count: usize = 1;
    for (node.children) |child| {
        count += countNodes(child);
    }
    return count;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // ใช้ Arena สำหรับ tree - free ทีเดียวเมื่อไม่ต้องการ
    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();

    const tree = try buildTree(arena.allocator(), 10);
    std.debug.print("Tree nodes: {}\n", .{countNodes(tree)});
    // arena.deinit() จะ free ทุก node พร้อมกัน
}
```

### Arena Reset Pattern

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();

    // ใช้ arena ซ้ำได้โดย reset
    for (0..5) |frame| {
        // reset arena เพื่อ reuse memory (faster กว่า deinit+init)
        // false = keep allocated memory for reuse
        _ = arena.reset(.retain_capacity);

        const alloc = arena.allocator();

        // Allocate สำหรับ frame นี้
        const data = try alloc.alloc(f32, 1000);
        for (data, 0..) |*v, i| {
            v.* = @floatFromInt(i * frame);
        }

        std.debug.print("Frame {}: first={:.2}, last={:.2}\n", .{
            frame, data[0], data[data.len - 1],
        });
        // ไม่ต้อง free - arena.reset จะจัดการ
    }
}
```

### เมื่อใดควรใช้ ArenaAllocator?

- **Request handlers** ใน web servers
- **Parse operations** ที่สร้าง temporary data structures
- **Game frame** ที่ allocate เยอะแล้ว clear ต่อ frame
- **Compiler passes** ที่ build AST แล้วทิ้ง
- **ไม่เหมาะ** ถ้าต้องการ free memory บางส่วนระหว่างทาง

---

## Step 164: FixedBufferAllocator - Allocate จาก Fixed Buffer

`FixedBufferAllocator` allocate จาก **buffer ที่กำหนดไว้ล่วงหน้า** ไม่ได้ call OS เลย เหมาะสำหรับ embedded systems, stack allocation, หรือสถานการณ์ที่ต้องการ deterministic memory

### การใช้งานพื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    // สร้าง buffer บน stack
    var buffer: [1024]u8 = undefined;

    // สร้าง FixedBufferAllocator จาก buffer
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    // Allocate จาก buffer
    const slice1 = try allocator.alloc(u8, 100);
    const slice2 = try allocator.alloc(u32, 50); // 200 bytes

    std.debug.print("slice1: {} bytes\n", .{slice1.len});
    std.debug.print("slice2: {} u32s ({} bytes)\n", .{ slice2.len, slice2.len * @sizeOf(u32) });

    // ลอง allocate เกิน buffer
    const result = allocator.alloc(u8, 800); // เกินที่เหลือ
    if (result) |_| {
        std.debug.print("Unexpectedly succeeded\n", .{});
    } else |err| {
        std.debug.print("Expected error: {}\n", .{err}); // OutOfMemory
    }
}
```

### ตรวจสอบ Memory Usage

```zig
const std = @import("std");

pub fn main() !void {
    var buffer: [4096]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    std.debug.print("Total buffer: {} bytes\n", .{buffer.len});

    const a = try allocator.alloc(u8, 512);
    std.debug.print("After alloc 512: used {} bytes\n", .{fba.end_index});

    const b = try allocator.alloc(u8, 256);
    std.debug.print("After alloc 256: used {} bytes\n", .{fba.end_index});

    // FBA ไม่สามารถ free แต่ละชิ้นได้ (ยกเว้น reset ทั้งหมด)
    _ = a;
    _ = b;

    // reset กลับเป็น 0
    fba.reset();
    std.debug.print("After reset: used {} bytes\n", .{fba.end_index});
}
```

### FixedBufferAllocator บน Stack vs Heap

```zig
const std = @import("std");

// Stack-based FBA: เร็วที่สุด, ใช้ใน function scope
fn stackAllocExample() !void {
    var stack_buf: [512]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&stack_buf);
    const alloc = fba.allocator();

    const data = try alloc.alloc(u8, 100);
    @memset(data, 0xFF);
    std.debug.print("Stack FBA: {} bytes\n", .{data.len});
    // data ถูก free อัตโนมัติเมื่อ function return (เพราะอยู่บน stack)
}

// Heap-based FBA: สำหรับ larger buffers
fn heapAllocExample(parent_alloc: std.mem.Allocator) !void {
    // Allocate buffer จาก heap
    const heap_buf = try parent_alloc.alloc(u8, 64 * 1024); // 64KB
    defer parent_alloc.free(heap_buf);

    var fba = std.heap.FixedBufferAllocator.init(heap_buf);
    const alloc = fba.allocator();

    const data = try alloc.alloc(u8, 1000);
    @memset(data, 0xAB);
    std.debug.print("Heap FBA: {} bytes\n", .{data.len});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    try stackAllocExample();
    try heapAllocExample(gpa.allocator());
}
```

### FixedBufferAllocator สำหรับ Embedded

```zig
const std = @import("std");

// Simulate embedded system with limited memory
const HEAP_SIZE = 8 * 1024; // 8KB total heap

var embedded_heap: [HEAP_SIZE]u8 = undefined;
var embedded_allocator_state = std.heap.FixedBufferAllocator.init(&embedded_heap);

pub fn getEmbeddedAllocator() std.mem.Allocator {
    return embedded_allocator_state.allocator();
}

const SensorData = struct {
    temperature: f32,
    humidity: f32,
    timestamp: u64,
};

pub fn main() !void {
    const alloc = getEmbeddedAllocator();

    // Allocate sensor buffer
    const readings = try alloc.alloc(SensorData, 10);
    for (readings, 0..) |*r, i| {
        r.* = SensorData{
            .temperature = 25.0 + @as(f32, @floatFromInt(i)) * 0.5,
            .humidity = 60.0,
            .timestamp = @intCast(i * 1000),
        };
    }

    std.debug.print("Embedded: {} readings, {} bytes used\n", .{
        readings.len,
        embedded_allocator_state.end_index,
    });
}
```

### เมื่อใดควรใช้ FixedBufferAllocator?

- **Embedded systems** ที่ไม่มี OS heap
- **Real-time systems** ที่ต้องการ deterministic allocation time
- **Hot paths** ที่ต้องการ zero-overhead allocation
- **Security-sensitive code** ที่ต้องการ memory isolation
- **ไม่เหมาะ** ถ้าไม่รู้ขนาด memory ที่ต้องการล่วงหน้า

---

## Step 165: HeapAllocator - OS Heap

`HeapAllocator` ใช้ **OS-level heap** functions ซึ่งบน Windows ใช้ `HeapAlloc` และบน POSIX ใช้ `malloc`

### การใช้งาน (Windows-specific แต่ portable)

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    // HeapAllocator มีให้ใช้บน Windows เป็นหลัก
    // บน Linux/macOS มักจะใช้ c_allocator หรือ page_allocator แทน
    if (builtin.os.tag == .windows) {
        const heap_allocator = std.heap.HeapAllocator.init();
        const allocator = heap_allocator.allocator();

        const data = try allocator.alloc(u8, 1024);
        defer allocator.free(data);

        std.debug.print("HeapAllocator: {} bytes\n", .{data.len});
    } else {
        // บน non-Windows ใช้ c_allocator แทน
        const allocator = std.heap.c_allocator;

        const data = try allocator.alloc(u8, 1024);
        defer allocator.free(data);

        std.debug.print("c_allocator (non-Windows): {} bytes\n", .{data.len});
    }
}
```

### Portable Heap Usage

```zig
const std = @import("std");
const builtin = @import("builtin");

// ฟังก์ชัน helper เพื่อได้ allocator ที่เหมาะสมกับ platform
pub fn getSystemAllocator() std.mem.Allocator {
    return switch (builtin.os.tag) {
        .windows => blk: {
            // Windows: ใช้ HeapAllocator
            break :blk std.heap.page_allocator; // simplified
        },
        else => blk: {
            // Unix: ใช้ c_allocator
            break :blk std.heap.c_allocator;
        },
    };
}

pub fn main() !void {
    const allocator = getSystemAllocator();

    const buf = try allocator.alloc(u8, 4096);
    defer allocator.free(buf);

    @memset(buf, 0);
    std.debug.print("System allocator: {} bytes\n", .{buf.len});
}
```

---

## Step 166: page_allocator - Page-Aligned Allocation

`std.heap.page_allocator` allocate memory โดยตรงจาก OS ใน unit ของ **memory pages** (ปกติ 4KB) เหมาะสำหรับ large allocations

### แนวคิด

```
Memory Page: ปกติ 4096 bytes (4KB)

page_allocator:
- alloc(100 bytes) -> ขอ 1 page (4096 bytes) จาก OS
- alloc(5000 bytes) -> ขอ 2 pages (8192 bytes) จาก OS
- ไม่เหมาะกับ small allocations (สิ้นเปลือง memory)
```

### การใช้งาน

```zig
const std = @import("std");

pub fn main() !void {
    const page_size = std.mem.page_size;
    std.debug.print("System page size: {} bytes\n", .{page_size});

    // page_allocator allocate โดยตรงจาก OS
    const large_buf = try std.heap.page_allocator.alloc(u8, 64 * 1024); // 64KB
    defer std.heap.page_allocator.free(large_buf);

    std.debug.print("Allocated {} bytes (page-aligned)\n", .{large_buf.len});
    std.debug.print("Address aligned to page: {}\n", .{
        @intFromPtr(large_buf.ptr) % page_size == 0,
    });

    // ใช้เป็น backing allocator สำหรับ ArenaAllocator
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();

    const alloc = arena.allocator();
    const data = try alloc.alloc(u32, 1000);
    std.debug.print("Arena on page_allocator: {} u32s\n", .{data.len});
}
```

### page_allocator กับ Large Buffers

```zig
const std = @import("std");

// Memory-mapped file simulation
fn loadLargeData() ![]u8 {
    // page_allocator เหมาะสำหรับ large buffers
    const size = 1 * 1024 * 1024; // 1MB
    return try std.heap.page_allocator.alloc(u8, size);
}

pub fn main() !void {
    const data = try loadLargeData();
    defer std.heap.page_allocator.free(data);

    // Initialize data
    @memset(data, 0);
    data[0] = 0xFF;
    data[data.len - 1] = 0xFF;

    std.debug.print("Large buffer: {} bytes\n", .{data.len});
    std.debug.print("First byte: 0x{X:0>2}\n", .{data[0]});
    std.debug.print("Last byte: 0x{X:0>2}\n", .{data[data.len - 1]});
}
```

### เมื่อใดควรใช้ page_allocator?

- **Large allocations** (>= ไม่กี่ pages)
- **เป็น backing allocator** สำหรับ Arena หรือ GPA
- **Memory-mapped operations**
- **ไม่เหมาะ** กับ small/frequent allocations (overhead สูง)

---

## Step 167: C Allocator (std.heap.c_allocator)

`std.heap.c_allocator` ใช้ C runtime's `malloc`/`free`/`realloc` เหมาะสำหรับ **interoperability กับ C code**

### การใช้งาน

```zig
const std = @import("std");

pub fn main() !void {
    // c_allocator ใช้ malloc/free ข้างใต้
    const allocator = std.heap.c_allocator;

    const buf = try allocator.alloc(u8, 256);
    defer allocator.free(buf);

    @memset(buf, 'A');
    std.debug.print("c_allocator: {s}\n", .{buf[0..10]});

    // realloc
    const bigger = try allocator.realloc(buf, 512);
    defer allocator.free(bigger);

    std.debug.print("After realloc: {} bytes\n", .{bigger.len});
}
```

### Interop กับ C Library

```zig
const std = @import("std");
const c = @cImport({
    @cInclude("stdlib.h");
    @cInclude("string.h");
});

// ฟังก์ชันที่ต้องส่ง pointer ที่ malloc'd ให้ C
fn createCString(allocator: std.mem.Allocator, s: []const u8) ![*:0]u8 {
    // ใช้ c_allocator เพื่อให้ C code สามารถ free ได้
    const buf = try allocator.allocSentinel(u8, s.len, 0);
    @memcpy(buf, s);
    return buf.ptr;
}

pub fn main() !void {
    // ถ้าจะ share memory กับ C library ต้องใช้ c_allocator
    const allocator = std.heap.c_allocator;

    const cstr = try createCString(allocator, "Hello from Zig");
    defer allocator.free(cstr[0 .. std.mem.len(cstr) + 1]);

    std.debug.print("C string: {s}\n", .{cstr});
}
```

### c_allocator vs malloc โดยตรง

```zig
const std = @import("std");
const c = @cImport(@cInclude("stdlib.h"));

pub fn main() !void {
    // วิธีที่ 1: ใช้ c_allocator (Zig-idiomatic)
    {
        const allocator = std.heap.c_allocator;
        const buf = try allocator.alloc(u8, 100);
        defer allocator.free(buf);
        std.debug.print("c_allocator way: {} bytes\n", .{buf.len});
    }

    // วิธีที่ 2: ใช้ malloc โดยตรง (C-style, ไม่แนะนำใน Zig)
    {
        const ptr = c.malloc(100) orelse return error.OutOfMemory;
        defer c.free(ptr);
        const buf = @as([*]u8, @ptrCast(ptr))[0..100];
        std.debug.print("malloc way: {} bytes\n", .{buf.len});
    }
}
```

### เมื่อใดควรใช้ c_allocator?

- **Interop กับ C libraries** ที่คาดหวัง malloc'd memory
- **FFI boundaries** ที่ C code จะ free memory
- **เมื่อต้องการ malloc behavior** ที่เหมือน C
- **ไม่เหมาะ** สำหรับ pure Zig code (ใช้ GPA แทน)

---

## Step 168: การส่ง Allocator เป็น Parameter

รูปแบบสำคัญที่สุดใน Zig คือการ **ส่ง allocator เป็น parameter** แทนที่จะ hardcode allocator ไว้ใน code

### Pattern พื้นฐาน

```zig
const std = @import("std");

// GOOD: รับ allocator เป็น parameter
fn buildMessage(allocator: std.mem.Allocator, name: []const u8, count: u32) ![]u8 {
    return std.fmt.allocPrint(allocator, "Hello, {s}! Count: {}", .{ name, count });
}

// BAD: hardcode allocator (ทำให้ test ยาก, ไม่ flexible)
fn buildMessageBad(name: []const u8, count: u32) ![]u8 {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    return std.fmt.allocPrint(gpa.allocator(), "Hello, {s}! Count: {}", .{ name, count });
    // leak! gpa ไม่ถูก deinit
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const msg = try buildMessage(gpa.allocator(), "Zig", 42);
    defer gpa.allocator().free(msg);

    std.debug.print("{s}\n", .{msg});
}
```

### Allocator ใน Struct

```zig
const std = @import("std");

// Struct ที่เก็บ allocator ไว้
const StringBuffer = struct {
    allocator: std.mem.Allocator,
    data: std.ArrayList(u8),

    pub fn init(allocator: std.mem.Allocator) StringBuffer {
        return StringBuffer{
            .allocator = allocator,
            .data = std.ArrayList(u8).init(allocator),
        };
    }

    pub fn deinit(self: *StringBuffer) void {
        self.data.deinit();
    }

    pub fn append(self: *StringBuffer, s: []const u8) !void {
        try self.data.appendSlice(s);
    }

    pub fn appendFormatted(self: *StringBuffer, comptime fmt: []const u8, args: anytype) !void {
        const s = try std.fmt.allocPrint(self.allocator, fmt, args);
        defer self.allocator.free(s);
        try self.data.appendSlice(s);
    }

    pub fn toOwned(self: *StringBuffer) ![]u8 {
        return self.data.toOwnedSlice();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var buf = StringBuffer.init(gpa.allocator());
    defer buf.deinit();

    try buf.append("Hello");
    try buf.appendFormatted(", {}!", .{42});
    try buf.append(" World");

    const result = try buf.toOwned();
    defer gpa.allocator().free(result);

    std.debug.print("{s}\n", .{result});
}
```

### Allocator Propagation Pattern

```zig
const std = @import("std");

// ระดับล่าง: ใช้ allocator โดยตรง
fn parseToken(allocator: std.mem.Allocator, input: []const u8) ![]u8 {
    return allocator.dupe(u8, input);
}

// ระดับกลาง: ส่ง allocator ต่อ
fn parseLine(allocator: std.mem.Allocator, line: []const u8) ![][]u8 {
    var tokens = std.ArrayList([]u8).init(allocator);
    errdefer {
        for (tokens.items) |t| allocator.free(t);
        tokens.deinit();
    }

    var iter = std.mem.splitScalar(u8, line, ' ');
    while (iter.next()) |word| {
        if (word.len > 0) {
            const token = try parseToken(allocator, word);
            try tokens.append(token);
        }
    }

    return tokens.toOwnedSlice();
}

// ระดับบน: สร้าง allocator และส่งลงไป
fn parseFile(allocator: std.mem.Allocator, content: []const u8) ![][]u8 {
    var all_tokens = std.ArrayList([]u8).init(allocator);
    errdefer {
        for (all_tokens.items) |t| allocator.free(t);
        all_tokens.deinit();
    }

    var lines = std.mem.splitScalar(u8, content, '\n');
    while (lines.next()) |line| {
        const tokens = try parseLine(allocator, line);
        defer {
            // free the slice itself (tokens are now owned by all_tokens)
            allocator.free(tokens);
        }
        try all_tokens.appendSlice(tokens);
    }

    return all_tokens.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const content = "hello world\nzig is awesome\nlearn allocators";
    const tokens = try parseFile(allocator, content);
    defer {
        for (tokens) |t| allocator.free(t);
        allocator.free(tokens);
    }

    std.debug.print("Parsed {} tokens:\n", .{tokens.len});
    for (tokens) |token| {
        std.debug.print("  '{s}'\n", .{token});
    }
}
```

### Thread-Safe Allocator Pattern

```zig
const std = @import("std");

// ThreadSafeAllocator ห่อ allocator ด้วย mutex
const ThreadSafeAllocator = struct {
    mutex: std.Thread.Mutex,
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator) ThreadSafeAllocator {
        return .{
            .mutex = .{},
            .backing = backing,
        };
    }

    pub fn allocator(self: *ThreadSafeAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *ThreadSafeAllocator = @ptrCast(@alignCast(ctx));
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.backing.rawAlloc(len, ptr_align, ret_addr);
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *ThreadSafeAllocator = @ptrCast(@alignCast(ctx));
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.backing.rawResize(buf, buf_align, new_len, ret_addr);
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *ThreadSafeAllocator = @ptrCast(@alignCast(ctx));
        self.mutex.lock();
        defer self.mutex.unlock();
        self.backing.rawFree(buf, buf_align, ret_addr);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var ts_alloc = ThreadSafeAllocator.init(gpa.allocator());
    const allocator = ts_alloc.allocator();

    const buf = try allocator.alloc(u8, 100);
    defer allocator.free(buf);

    std.debug.print("ThreadSafe allocator: {} bytes\n", .{buf.len});
}
```

---

## Step 169: Custom Allocator Implementation

การสร้าง custom allocator ใน Zig ต้อง implement `alloc`, `resize`, และ `free` functions

### Minimal Custom Allocator

```zig
const std = @import("std");

// Simple bump allocator (เหมือน ArenaAllocator แต่ simplified)
const BumpAllocator = struct {
    buffer: []u8,
    pos: usize,

    pub fn init(buffer: []u8) BumpAllocator {
        return .{ .buffer = buffer, .pos = 0 };
    }

    pub fn allocator(self: *BumpAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    pub fn reset(self: *BumpAllocator) void {
        self.pos = 0;
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        _ = ret_addr;
        const self: *BumpAllocator = @ptrCast(@alignCast(ctx));

        // Align position
        const aligned_pos = std.mem.alignForward(usize, self.pos, @as(usize, 1) << @intCast(ptr_align));

        // Check if enough space
        if (aligned_pos + len > self.buffer.len) return null;

        const result = self.buffer[aligned_pos..].ptr;
        self.pos = aligned_pos + len;
        return result;
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        _ = ctx;
        _ = buf;
        _ = buf_align;
        _ = new_len;
        _ = ret_addr;
        // Bump allocator ไม่ support resize
        return false;
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        _ = ctx;
        _ = buf;
        _ = buf_align;
        _ = ret_addr;
        // Bump allocator ไม่ support individual free
        // (ต้อง reset ทั้งหมด)
    }
};

pub fn main() !void {
    var backing: [4096]u8 = undefined;
    var bump = BumpAllocator.init(&backing);
    const alloc = bump.allocator();

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u32, 50);
    const c = try alloc.alloc(u64, 25);

    std.debug.print("Bump allocator positions:\n", .{});
    std.debug.print("  a: {} bytes\n", .{a.len});
    std.debug.print("  b: {} u32s\n", .{b.len});
    std.debug.print("  c: {} u64s\n", .{c.len});
    std.debug.print("  total used: {} bytes\n", .{bump.pos});

    bump.reset();
    std.debug.print("After reset: {} bytes used\n", .{bump.pos});
}
```

### Logging Allocator (Decorator Pattern)

```zig
const std = @import("std");

// Allocator ที่ log ทุก operation
const LoggingAllocator = struct {
    backing: std.mem.Allocator,
    alloc_count: usize,
    free_count: usize,
    total_allocated: usize,

    pub fn init(backing: std.mem.Allocator) LoggingAllocator {
        return .{
            .backing = backing,
            .alloc_count = 0,
            .free_count = 0,
            .total_allocated = 0,
        };
    }

    pub fn allocator(self: *LoggingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    pub fn printStats(self: *const LoggingAllocator) void {
        std.debug.print("=== Allocator Stats ===\n", .{});
        std.debug.print("  Allocations:   {}\n", .{self.alloc_count});
        std.debug.print("  Frees:         {}\n", .{self.free_count});
        std.debug.print("  Live allocs:   {}\n", .{self.alloc_count - self.free_count});
        std.debug.print("  Total bytes:   {}\n", .{self.total_allocated});
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *LoggingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.backing.rawAlloc(len, ptr_align, ret_addr);
        if (result != null) {
            self.alloc_count += 1;
            self.total_allocated += len;
            std.debug.print("[ALLOC] {} bytes\n", .{len});
        }
        return result;
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *LoggingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.backing.rawResize(buf, buf_align, new_len, ret_addr);
        if (result) {
            std.debug.print("[RESIZE] {} -> {} bytes\n", .{ buf.len, new_len });
        }
        return result;
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *LoggingAllocator = @ptrCast(@alignCast(ctx));
        self.free_count += 1;
        std.debug.print("[FREE] {} bytes\n", .{buf.len});
        self.backing.rawFree(buf, buf_align, ret_addr);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var logger = LoggingAllocator.init(gpa.allocator());
    const alloc = logger.allocator();

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u32, 50);
    alloc.free(a);
    const c = try alloc.alloc(u64, 10);
    alloc.free(b);
    alloc.free(c);

    logger.printStats();
}
```

### Counting Allocator

```zig
const std = @import("std");

// Allocator ที่นับจำนวน bytes ที่ใช้อยู่
const CountingAllocator = struct {
    backing: std.mem.Allocator,
    current_bytes: usize,
    peak_bytes: usize,

    pub fn init(backing: std.mem.Allocator) CountingAllocator {
        return .{
            .backing = backing,
            .current_bytes = 0,
            .peak_bytes = 0,
        };
    }

    pub fn allocator(self: *CountingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.backing.rawAlloc(len, ptr_align, ret_addr);
        if (result != null) {
            self.current_bytes += len;
            if (self.current_bytes > self.peak_bytes) {
                self.peak_bytes = self.current_bytes;
            }
        }
        return result;
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.backing.rawResize(buf, buf_align, new_len, ret_addr);
        if (result) {
            if (new_len > buf.len) {
                self.current_bytes += new_len - buf.len;
            } else {
                self.current_bytes -= buf.len - new_len;
            }
            if (self.current_bytes > self.peak_bytes) {
                self.peak_bytes = self.current_bytes;
            }
        }
        return result;
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        self.current_bytes -= buf.len;
        self.backing.rawFree(buf, buf_align, ret_addr);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var counter = CountingAllocator.init(gpa.allocator());
    const alloc = counter.allocator();

    const bufs = [_]usize{ 1000, 2000, 500, 3000 };
    var allocations = std.ArrayList([]u8).init(gpa.allocator());
    defer allocations.deinit();

    for (bufs) |size| {
        const buf = try alloc.alloc(u8, size);
        try allocations.append(buf);
        std.debug.print("Allocated {}: current={}, peak={}\n", .{
            size, counter.current_bytes, counter.peak_bytes,
        });
    }

    for (allocations.items) |buf| {
        alloc.free(buf);
    }

    std.debug.print("After free: current={}, peak={}\n", .{
        counter.current_bytes, counter.peak_bytes,
    });
}
```

---

## Step 170: Allocator Benchmarking และการเลือกใช้

### Benchmark Framework

```zig
const std = @import("std");

const BenchResult = struct {
    name: []const u8,
    alloc_ns: u64,
    free_ns: u64,
    total_ns: u64,
};

fn benchAllocator(
    name: []const u8,
    allocator: std.mem.Allocator,
    iterations: usize,
    alloc_size: usize,
) !BenchResult {
    var timer = try std.time.Timer.start();

    // Benchmark allocation
    var bufs = try std.heap.page_allocator.alloc([]u8, iterations);
    defer std.heap.page_allocator.free(bufs);

    const alloc_start = timer.read();
    for (bufs) |*buf| {
        buf.* = try allocator.alloc(u8, alloc_size);
    }
    const alloc_end = timer.read();

    // Touch memory to avoid optimization
    for (bufs) |buf| {
        buf[0] = 0xFF;
    }

    // Benchmark free
    const free_start = timer.read();
    for (bufs) |buf| {
        allocator.free(buf);
    }
    const free_end = timer.read();

    return BenchResult{
        .name = name,
        .alloc_ns = (alloc_end - alloc_start) / iterations,
        .free_ns = (free_end - free_start) / iterations,
        .total_ns = ((alloc_end - alloc_start) + (free_end - free_start)) / iterations,
    };
}

pub fn main() !void {
    const iterations = 10000;
    const alloc_size = 256;

    std.debug.print("Allocator Benchmark ({} iterations, {} bytes each)\n", .{
        iterations, alloc_size,
    });
    std.debug.print("{s:-<60}\n", .{""});
    std.debug.print("{s:<25} {s:>10} {s:>10} {s:>10}\n", .{
        "Allocator", "Alloc ns", "Free ns", "Total ns",
    });
    std.debug.print("{s:-<60}\n", .{""});

    // GPA Benchmark
    {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer _ = gpa.deinit();
        const result = try benchAllocator("GPA", gpa.allocator(), iterations, alloc_size);
        std.debug.print("{s:<25} {d:>10} {d:>10} {d:>10}\n", .{
            result.name, result.alloc_ns, result.free_ns, result.total_ns,
        });
    }

    // C Allocator Benchmark
    {
        const result = try benchAllocator("c_allocator", std.heap.c_allocator, iterations, alloc_size);
        std.debug.print("{s:<25} {d:>10} {d:>10} {d:>10}\n", .{
            result.name, result.alloc_ns, result.free_ns, result.total_ns,
        });
    }

    // Arena Allocator Benchmark
    {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer _ = gpa.deinit();
        var arena = std.heap.ArenaAllocator.init(gpa.allocator());
        defer arena.deinit();

        var timer = try std.time.Timer.start();
        const start = timer.read();

        for (0..iterations) |_| {
            const buf = try arena.allocator().alloc(u8, alloc_size);
            buf[0] = 0xFF;
        }

        const arena_alloc_ns = (timer.read() - start) / iterations;

        std.debug.print("{s:<25} {d:>10} {d:>10} {d:>10}\n", .{
            "ArenaAllocator",
            arena_alloc_ns,
            0, // Arena free ทีเดียวตอน deinit
            arena_alloc_ns,
        });
    }

    // Fixed Buffer Allocator Benchmark
    {
        var backing: [iterations * (alloc_size + 16)]u8 = undefined;
        var fba = std.heap.FixedBufferAllocator.init(&backing);

        var timer = try std.time.Timer.start();
        const start = timer.read();

        for (0..iterations) |_| {
            const buf = try fba.allocator().alloc(u8, alloc_size);
            buf[0] = 0xFF;
        }

        const fba_alloc_ns = (timer.read() - start) / iterations;

        std.debug.print("{s:<25} {d:>10} {d:>10} {d:>10}\n", .{
            "FixedBufferAllocator",
            fba_alloc_ns,
            0, // FBA ไม่มี per-item free
            fba_alloc_ns,
        });
    }

    std.debug.print("{s:-<60}\n", .{""});
}
```

### Decision Matrix: เลือก Allocator อย่างไร?

```zig
const std = @import("std");

// ตัวอย่างการเลือก allocator ตาม use case
const UseCase = enum {
    debug_development,     // Development/testing
    web_request_handler,   // Handle HTTP requests
    game_frame,           // Per-frame allocations
    embedded_system,      // No OS, limited memory
    c_interop,            // Interface with C libraries
    large_data_processing, // Process large datasets
    long_running_server,  // Server with mixed allocations
};

fn recommendAllocator(use_case: UseCase) []const u8 {
    return switch (use_case) {
        .debug_development =>
            "GeneralPurposeAllocator - ตรวจจับ leaks/double-free",
        .web_request_handler =>
            "ArenaAllocator per-request - fast alloc, free ทีเดียว",
        .game_frame =>
            "ArenaAllocator per-frame + reset() - zero fragmentation",
        .embedded_system =>
            "FixedBufferAllocator - no OS dependency, deterministic",
        .c_interop =>
            "c_allocator - compatible กับ C free()",
        .large_data_processing =>
            "page_allocator หรือ ArenaAllocator - large contiguous blocks",
        .long_running_server =>
            "GPA (production) หรือ per-request Arena + GPA backing",
    };
}

pub fn main() void {
    const use_cases = [_]UseCase{
        .debug_development,
        .web_request_handler,
        .game_frame,
        .embedded_system,
        .c_interop,
        .large_data_processing,
        .long_running_server,
    };

    std.debug.print("Allocator Recommendations:\n", .{});
    std.debug.print("{s:-<70}\n", .{""});

    for (use_cases) |uc| {
        std.debug.print("{s:<30} -> {s}\n", .{
            @tagName(uc),
            recommendAllocator(uc),
        });
    }
}
```

### Real-World: Web Server Pattern

```zig
const std = @import("std");

// Production web server pattern
const Server = struct {
    // GPA เป็น backing allocator สำหรับทั้ง server
    gpa: std.heap.GeneralPurposeAllocator(.{}),

    pub fn init() Server {
        return .{ .gpa = std.heap.GeneralPurposeAllocator(.{}){} };
    }

    pub fn deinit(self: *Server) void {
        _ = self.gpa.deinit();
    }

    pub fn handleRequest(self: *Server, raw: []const u8) ![]u8 {
        // Arena per-request: ทุก temp allocation ใน request
        var request_arena = std.heap.ArenaAllocator.init(self.gpa.allocator());
        defer request_arena.deinit(); // free ทุกอย่างเมื่อ request จบ

        const arena_alloc = request_arena.allocator();

        // Parse phase: allocate freely ใน arena
        const parsed = try parseRequest(arena_alloc, raw);

        // Process phase: สร้าง response
        const response = try buildResponse(arena_alloc, parsed);

        // Copy response ออกจาก arena (ส่งกลับ caller)
        return try self.gpa.allocator().dupe(u8, response);
    }
};

const ParsedRequest = struct {
    method: []const u8,
    path: []const u8,
};

fn parseRequest(alloc: std.mem.Allocator, raw: []const u8) !ParsedRequest {
    // Simplified parsing
    const method = try alloc.dupe(u8, if (raw.len > 3) raw[0..3] else "GET");
    const path = try alloc.dupe(u8, if (raw.len > 0) raw else "/");
    return ParsedRequest{ .method = method, .path = path };
}

fn buildResponse(alloc: std.mem.Allocator, req: ParsedRequest) ![]u8 {
    return std.fmt.allocPrint(
        alloc,
        "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n{s} {s}",
        .{ req.method, req.path },
    );
}

pub fn main() !void {
    var server = Server.init();
    defer server.deinit();

    // Simulate handling requests
    const requests = [_][]const u8{
        "GET /api/users",
        "POST /api/data",
        "GET /health",
    };

    for (requests) |req| {
        const response = try server.handleRequest(req);
        defer server.gpa.allocator().free(response);
        std.debug.print("Response: {s}\n\n", .{response});
    }
}
```

### Real-World: Game Loop Pattern

```zig
const std = @import("std");

const Vec2 = struct { x: f32, y: f32 };
const Particle = struct {
    pos: Vec2,
    vel: Vec2,
    lifetime: f32,
};

const GameState = struct {
    // Persistent allocator สำหรับ game objects ที่อยู่นาน
    persistent_alloc: std.mem.Allocator,
    // Frame allocator สำหรับ temp calculations ต่อ frame
    frame_arena: std.heap.ArenaAllocator,

    pub fn init(backing: std.mem.Allocator) GameState {
        return .{
            .persistent_alloc = backing,
            .frame_arena = std.heap.ArenaAllocator.init(backing),
        };
    }

    pub fn deinit(self: *GameState) void {
        self.frame_arena.deinit();
    }

    pub fn beginFrame(self: *GameState) void {
        // Reset frame arena ทุก frame
        _ = self.frame_arena.reset(.retain_capacity);
    }

    pub fn frameAllocator(self: *GameState) std.mem.Allocator {
        return self.frame_arena.allocator();
    }
};

fn spawnParticles(alloc: std.mem.Allocator, count: usize) ![]Particle {
    const particles = try alloc.alloc(Particle, count);
    for (particles, 0..) |*p, i| {
        p.* = Particle{
            .pos = .{ .x = @floatFromInt(i * 10), .y = 0 },
            .vel = .{ .x = 1.0, .y = -1.0 },
            .lifetime = 2.0,
        };
    }
    return particles;
}

fn updateParticles(particles: []Particle, dt: f32) void {
    for (particles) |*p| {
        p.pos.x += p.vel.x * dt;
        p.pos.y += p.vel.y * dt;
        p.lifetime -= dt;
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var game = GameState.init(gpa.allocator());
    defer game.deinit();

    const dt: f32 = 1.0 / 60.0;

    for (0..3) |frame| {
        game.beginFrame();

        // Spawn temporary particles (frame allocator)
        const frame_particles = try spawnParticles(game.frameAllocator(), 100);
        updateParticles(frame_particles, dt);

        std.debug.print("Frame {}: {} particles updated\n", .{
            frame, frame_particles.len,
        });
        // frame_particles ถูก free อัตโนมัติตอน beginFrame ของ frame หน้า
    }
}
```

### Summary: Allocator Comparison Table

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Zig Allocator Comparison                         │
├──────────────────────┬──────────┬──────────┬───────────┬───────────┤
│ Allocator            │ Speed    │ Memory   │ Debug     │ Use Case  │
├──────────────────────┼──────────┼──────────┼───────────┼───────────┤
│ GeneralPurpose       │ Medium   │ Efficient│ Excellent │ Dev/Test  │
│ ArenaAllocator       │ Very Fast│ Efficient│ Good      │ Temp data │
│ FixedBufferAllocator │ Fastest  │ Fixed    │ Limited   │ Embedded  │
│ page_allocator       │ Slow     │ Page-    │ None      │ Large buf │
│                      │          │ aligned  │           │           │
│ c_allocator          │ Fast     │ Efficient│ None      │ C interop │
│ HeapAllocator        │ Fast     │ Efficient│ None      │ Windows   │
└──────────────────────┴──────────┴──────────┴───────────┴───────────┘
```

---

## แบบฝึกหัด

### ระดับเริ่มต้น

**แบบฝึกหัด 1:** เขียนโปรแกรมที่ใช้ `GeneralPurposeAllocator` เพื่อ allocate array ของ strings จากนั้นเพิ่ม `defer` ที่ถูกต้องเพื่อไม่ให้เกิด memory leak

```zig
// โจทย์: เติม code ให้ครบ
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    // TODO: เพิ่ม defer สำหรับ gpa

    const allocator = gpa.allocator();
    const words = [_][]const u8{ "apple", "banana", "cherry", "date" };

    // TODO: allocate array ของ strings และ copy words เข้าไป
    // TODO: print ทุก string
    // TODO: free ทุก string
}
```

**แบบฝึกหัด 2:** ใช้ `ArenaAllocator` เพื่อ build linked list โดยไม่ต้อง free แต่ละ node ด้วยตัวเอง

```zig
// โจทย์: สร้าง linked list ด้วย ArenaAllocator
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

// TODO: implement buildList(allocator, values) !*Node
// TODO: implement printList(head) void

pub fn main() !void {
    // TODO: ใช้ ArenaAllocator สร้าง list [1, 2, 3, 4, 5]
    // TODO: print list
}
```

### ระดับกลาง

**แบบฝึกหัด 3:** สร้าง `StackAllocator` ที่ allocate จาก fixed buffer และ support `free` สำหรับ element สุดท้ายเท่านั้น (LIFO)

```zig
const std = @import("std");

// โจทย์: implement StackAllocator
// - alloc: allocate จาก top ของ stack
// - free: free ได้เฉพาะ allocation สุดท้าย (LIFO)
// - ถ้า free ไม่ใช่ element สุดท้าย ให้ panic

const StackAllocator = struct {
    buffer: []u8,
    top: usize,

    pub fn init(buffer: []u8) StackAllocator { ... }
    pub fn allocator(self: *StackAllocator) std.mem.Allocator { ... }

    // implement alloc, resize, free
};
```

**แบบฝึกหัด 4:** เขียน `MemoryProfiler` ที่ wrap allocator และ track:
- จำนวน allocations ที่ยังมีชีวิตอยู่
- Largest single allocation
- Average allocation size
- Histogram ของ allocation sizes

### ระดับสูง

**แบบฝึกหัด 5:** Implement `PoolAllocator` สำหรับ allocate objects ขนาดเดียวกัน

```zig
// Pool Allocator: เหมาะสำหรับ allocate objects ขนาดเดียวกันเยอะๆ
// เช่น game entities, connection objects, etc.

fn PoolAllocator(comptime T: type) type {
    return struct {
        const Self = @This();
        const ChunkSize = 64;

        // TODO: implement
        // - สร้าง free list ของ T objects
        // - alloc: เอาจาก free list หรือ allocate chunk ใหม่
        // - free: ใส่กลับ free list
        // - เร็วกว่า GPA สำหรับ same-size objects

        pub fn init(backing: std.mem.Allocator) Self { ... }
        pub fn deinit(self: *Self) void { ... }
        pub fn create(self: *Self) !*T { ... }
        pub fn destroy(self: *Self, ptr: *T) void { ... }
    };
}
```

**แบบฝึกหัด 6:** เขียน benchmark เปรียบเทียบ allocators ด้วย use case ที่ต่างกัน:
1. Many small allocations (< 64 bytes)
2. Few large allocations (> 1MB)
3. Mixed sizes with lifetime tracking
4. Arena pattern (alloc many, free all)

---

## สรุป

### สิ่งที่เรียนรู้ใน Part 17

| Step | หัวข้อ | ประเด็นสำคัญ |
|------|--------|-------------|
| 161 | Allocator Interface | ทุก allocator ใช้ interface เดียวกัน |
| 162 | GeneralPurposeAllocator | ตรวจจับ leak/double-free, ใช้ development |
| 163 | ArenaAllocator | alloc เยอะ free ครั้งเดียว, pattern สำหรับ request |
| 164 | FixedBufferAllocator | allocate จาก stack/fixed buffer, embedded |
| 165 | HeapAllocator | OS heap, Windows-primary |
| 166 | page_allocator | Page-aligned, large allocations, backing |
| 167 | c_allocator | malloc/free, C interop |
| 168 | Allocator as Parameter | ส่ง allocator ผ่านทุก function ที่ต้องการ |
| 169 | Custom Allocator | implement vtable: alloc/resize/free |
| 170 | Benchmarking & Selection | เลือก allocator ตาม use case |

### Best Practices

1. **Always pass allocators explicitly** - ไม่ hardcode allocator ใน library code
2. **Use GPA during development** - ตรวจจับ bugs อัตโนมัติ
3. **Use `std.testing.allocator` in tests** - detect leaks ใน unit tests
4. **Use Arena for batch operations** - request handlers, parse, build
5. **Use FBA for embedded/real-time** - deterministic, no OS dependency
6. **Defer free immediately** - ทันทีที่ allocate เขียน `defer free`
7. **Consider ownership** - ใครเป็น owner ต้อง free

### Pattern Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    Allocator Pattern Guide                   │
├──────────────────────────────┬──────────────────────────────┤
│ Situation                    │ Recommended Pattern           │
├──────────────────────────────┼──────────────────────────────┤
│ Development                  │ GPA always                   │
│ Unit tests                   │ std.testing.allocator        │
│ HTTP request handler         │ Arena per-request            │
│ Game frame                   │ Arena + reset() per-frame    │
│ Parse/build tree             │ Arena (free when done)       │
│ Embedded/MCU                 │ FixedBufferAllocator         │
│ C library integration        │ c_allocator                  │
│ Large files/buffers          │ page_allocator               │
│ Long-running with mixed      │ GPA + Arena for temp         │
│ Production server            │ c_allocator หรือ custom      │
└──────────────────────────────┴──────────────────────────────┘
```

### อ่านต่อ

- **Part 18**: Error Handling Advanced - Zig error model เชิงลึก
- **Part 19**: Comptime Programming - Metaprogramming ใน Zig
- **Part 20**: Concurrency - Threads และ async/await ใน Zig

---

*Part 17 จบแล้ว - คุณได้เรียนรู้ Allocator ทุกประเภทใน Zig และวิธีเลือกใช้ให้เหมาะสมกับแต่ละ use case*
