# ส่วนที่ 72: Custom Allocators ใน Zig

## ภาพรวม

Zig มี allocator interface ที่ยืดหยุ่นมาก ทำให้เราสามารถสร้าง custom allocator สำหรับ use case เฉพาะได้ ใน Part นี้เราจะสร้าง allocator หลายประเภทตั้งแต่ง่ายไปหาซับซ้อน

---

## Step 711: Allocator Interface Deep Dive

### Allocator Interface ใน Zig

```zig
const std = @import("std");

// std.mem.Allocator VTable
// pub const VTable = struct {
//     alloc: *const fn (ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8,
//     resize: *const fn (ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool,
//     free: *const fn (ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void,
// };

// Minimal allocator implementation
const MinimalAllocator = struct {
    const Self = @This();

    // State ของ allocator
    backing: std.mem.Allocator,
    alloc_count: usize = 0,
    free_count: usize = 0,

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const result = self.backing.rawAlloc(len, ptr_align, ret_addr);
        if (result != null) self.alloc_count += 1;
        return result;
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *Self = @ptrCast(@alignCast(ctx));
        return self.backing.rawResize(buf, buf_align, new_len, ret_addr);
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        self.free_count += 1;
        self.backing.rawFree(buf, buf_align, ret_addr);
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = alloc,
        .resize = resize,
        .free = free,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var min_alloc = MinimalAllocator{ .backing = gpa.allocator() };
    const alloc = min_alloc.allocator();

    const buf = try alloc.alloc(u8, 100);
    defer alloc.free(buf);

    const str = try alloc.dupe(u8, "Hello, World!");
    defer alloc.free(str);

    std.debug.print("Alloc count: {}, Free count: {}\n", .{
        min_alloc.alloc_count, min_alloc.free_count
    });
}
```

### การทดสอบ Allocator Interface

```zig
const std = @import("std");

// Null Allocator - สำหรับ testing
const NullAllocator = struct {
    fn alloc(_: *anyopaque, _: usize, _: u8, _: usize) ?[*]u8 {
        return null; // ไม่เคย allocate สำเร็จ
    }

    fn resize(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }

    fn free(_: *anyopaque, _: []u8, _: u8, _: usize) void {}

    const vtable = std.mem.Allocator.VTable{
        .alloc = alloc,
        .resize = resize,
        .free = free,
    };

    var instance: NullAllocator = .{};

    pub fn get() std.mem.Allocator {
        return .{ .ptr = &instance, .vtable = &vtable };
    }
};

// Failing Allocator - fail หลังจาก N allocations
const FailingAllocator = struct {
    const Self = @This();

    backing: std.mem.Allocator,
    fail_after: usize,
    alloc_count: usize,

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        if (self.alloc_count >= self.fail_after) {
            std.debug.print("FailingAllocator: forced failure\n", .{});
            return null;
        }
        self.alloc_count += 1;
        return self.backing.rawAlloc(len, ptr_align, ret_addr);
    }

    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *Self = @ptrCast(@alignCast(ctx));
        return self.backing.rawResize(buf, buf_align, new_len, ret_addr);
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        self.backing.rawFree(buf, buf_align, ret_addr);
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = alloc,
        .resize = resize,
        .free = free,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }
};

pub fn allocateThings(allocator: std.mem.Allocator) !void {
    const a = try allocator.alloc(u8, 10);
    defer allocator.free(a);
    const b = try allocator.alloc(u8, 20);
    defer allocator.free(b);
    const c = try allocator.alloc(u8, 30);
    defer allocator.free(c);
    std.debug.print("All allocations succeeded\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // Test null allocator
    const null_alloc = NullAllocator.get();
    const result = null_alloc.alloc(u8, 10);
    std.debug.print("Null alloc result: {?}\n", .{result});

    // Test failing allocator
    var failing = FailingAllocator{
        .backing = gpa.allocator(),
        .fail_after = 2,
        .alloc_count = 0,
    };

    allocateThings(failing.allocator()) catch |err| {
        std.debug.print("Expected failure: {}\n", .{err});
    };
}
```

---

## Step 712: Building a Bump Allocator

### Bump Allocator คืออะไร

Bump allocator (หรือ linear allocator) คือ allocator ที่ง่ายที่สุด - แค่ "bump" pointer ไปข้างหน้า ไม่สามารถ free individual allocations ได้

```zig
const std = @import("std");

const BumpAllocator = struct {
    const Self = @This();

    buffer: []u8,
    offset: usize,

    pub fn init(buffer: []u8) Self {
        return Self{ .buffer = buffer, .offset = 0 };
    }

    pub fn reset(self: *Self) void {
        self.offset = 0;
    }

    pub fn remaining(self: *const Self) usize {
        return self.buffer.len - self.offset;
    }

    pub fn used(self: *const Self) usize {
        return self.offset;
    }

    fn alignOffset(offset: usize, alignment: usize) usize {
        return (offset + alignment - 1) & ~(alignment - 1);
    }

    fn allocFn(ctx: *anyopaque, len: usize, ptr_align: u8, _: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const alignment = @as(usize, 1) << @intCast(ptr_align);
        const aligned_offset = alignOffset(self.offset, alignment);

        if (aligned_offset + len > self.buffer.len) {
            return null; // Out of memory
        }

        const ptr = self.buffer[aligned_offset..].ptr;
        self.offset = aligned_offset + len;
        return ptr;
    }

    fn resizeFn(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false; // Bump allocator ไม่ support resize
    }

    fn freeFn(_: *anyopaque, _: []u8, _: u8, _: usize) void {
        // No-op: ไม่ free individual allocations
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }
};

// Typed Bump Allocator พร้อม alignment
const TypedBumpAllocator = struct {
    const Self = @This();
    const ALIGNMENT = 8;

    memory: []align(ALIGNMENT) u8,
    cursor: usize,
    high_water: usize,
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator, capacity: usize) !Self {
        const mem = try backing.alignedAlloc(u8, ALIGNMENT, capacity);
        return Self{
            .memory = mem,
            .cursor = 0,
            .high_water = 0,
            .backing = backing,
        };
    }

    pub fn deinit(self: *Self) void {
        self.backing.free(self.memory);
    }

    pub fn alloc(self: *Self, comptime T: type, n: usize) ![]T {
        const alignment = @alignOf(T);
        const size = @sizeOf(T) * n;
        const aligned = (self.cursor + alignment - 1) & ~(alignment - 1);

        if (aligned + size > self.memory.len) return error.OutOfMemory;

        const result: []T = @alignCast(std.mem.bytesAsSlice(T, self.memory[aligned..aligned + size]));
        self.cursor = aligned + size;
        if (self.cursor > self.high_water) self.high_water = self.cursor;
        return result;
    }

    pub fn reset(self: *Self) void {
        self.cursor = 0;
    }

    pub fn printStats(self: *const Self) void {
        std.debug.print("BumpAllocator: used={}/{}, peak={}\n", .{
            self.cursor, self.memory.len, self.high_water
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // Simple bump allocator
    var buffer: [4096]u8 = undefined;
    var bump = BumpAllocator.init(&buffer);
    const alloc = bump.allocator();

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u32, 50);
    const c = try alloc.alloc(u64, 25);

    std.debug.print("Allocated: {} + {} + {} = {} bytes used\n", .{
        a.len, b.len * 4, c.len * 8, bump.used()
    });

    // Reset และ reuse
    bump.reset();
    std.debug.print("After reset: {} bytes used\n", .{bump.used()});

    // Typed bump allocator
    var typed = try TypedBumpAllocator.init(gpa.allocator(), 1024 * 64);
    defer typed.deinit();

    const ints = try typed.alloc(i32, 100);
    const floats = try typed.alloc(f64, 50);
    const chars = try typed.alloc(u8, 200);

    for (ints, 0..) |*v, i| v.* = @intCast(i);
    for (floats, 0..) |*v, i| v.* = @as(f64, @floatFromInt(i)) * 0.5;
    for (chars, 0..) |*v, i| v.* = @intCast(i % 256);

    typed.printStats();
    typed.reset();
    typed.printStats();

    _ = b;
    _ = c;
}
```

---

## Step 713: Building a Stack Allocator

### Stack Allocator

```zig
const std = @import("std");

const StackAllocator = struct {
    const Self = @This();
    const Header = struct {
        size: usize,
        prev_offset: usize,
        alignment_padding: usize,
    };

    memory: []u8,
    top: usize,
    prev_alloc_offset: usize,

    pub fn init(memory: []u8) Self {
        return Self{
            .memory = memory,
            .top = 0,
            .prev_alloc_offset = 0,
        };
    }

    // Marker สำหรับ restore
    pub const Marker = struct {
        top: usize,
        prev: usize,
    };

    pub fn getMarker(self: *const Self) Marker {
        return .{ .top = self.top, .prev = self.prev_alloc_offset };
    }

    pub fn freeToMarker(self: *Self, marker: Marker) void {
        self.top = marker.top;
        self.prev_alloc_offset = marker.prev;
    }

    fn allocFn(ctx: *anyopaque, len: usize, ptr_align: u8, _: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const alignment = @as(usize, 1) << @intCast(ptr_align);
        const header_size = @sizeOf(Header);

        // Align header
        const header_offset = (self.top + @alignOf(Header) - 1) & ~(@alignOf(Header) - 1);
        // Align data after header
        const data_start = (header_offset + header_size + alignment - 1) & ~(alignment - 1);
        const padding = data_start - (header_offset + header_size);
        const total_needed = (data_start - self.top) + len;

        if (self.top + total_needed > self.memory.len) return null;

        const header: *Header = @ptrCast(@alignCast(&self.memory[header_offset]));
        header.* = .{
            .size = len,
            .prev_offset = self.prev_alloc_offset,
            .alignment_padding = padding,
        };

        self.prev_alloc_offset = header_offset;
        self.top = data_start + len;

        return self.memory[data_start..].ptr;
    }

    fn resizeFn(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, _: u8, _: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        // Only free ถ้าเป็น last allocation
        const buf_end = @intFromPtr(buf.ptr) + buf.len;
        const top_addr = @intFromPtr(self.memory.ptr) + self.top;
        if (buf_end == top_addr) {
            // Last allocation - can free
            const header: *Header = @ptrCast(@alignCast(&self.memory[self.prev_alloc_offset]));
            self.top = self.prev_alloc_offset;
            self.prev_alloc_offset = header.prev_offset;
        }
        // Otherwise: ignore (LIFO violation)
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }

    pub fn used(self: *const Self) usize {
        return self.top;
    }
};

pub fn main() !void {
    var buffer: [8192]u8 align(16) = undefined;
    var stack = StackAllocator.init(&buffer);
    const alloc = stack.allocator();

    // Allocate in LIFO order
    const mark1 = stack.getMarker();

    const data1 = try alloc.alloc(u8, 100);
    @memset(data1, 'A');

    const mark2 = stack.getMarker();

    const data2 = try alloc.alloc(u32, 50);
    for (data2, 0..) |*v, i| v.* = @intCast(i);

    std.debug.print("After two allocs: {} bytes used\n", .{stack.used()});
    std.debug.print("data1[0]: {c}, data2[0]: {}\n", .{data1[0], data2[0]});

    // Pop back to mark2
    stack.freeToMarker(mark2);
    std.debug.print("After free to mark2: {} bytes used\n", .{stack.used()});

    // Pop back to mark1
    stack.freeToMarker(mark1);
    std.debug.print("After free to mark1: {} bytes used\n", .{stack.used()});

    // Reuse memory
    const data3 = try alloc.alloc(u8, 200);
    @memset(data3, 'B');
    std.debug.print("After reuse: {} bytes used, data3[0]: {c}\n", .{
        stack.used(), data3[0]
    });
}
```

---

## Step 714: Building a Ring Buffer Allocator

### Ring Buffer Allocator สำหรับ Streaming Data

```zig
const std = @import("std");

const RingAllocator = struct {
    const Self = @This();
    const Header = packed struct {
        size: u32,
        in_use: bool,
        _padding: u31 = 0,
    };

    memory: []u8,
    read_pos: usize,
    write_pos: usize,
    full: bool,

    pub fn init(memory: []u8) Self {
        return Self{
            .memory = memory,
            .read_pos = 0,
            .write_pos = 0,
            .full = false,
        };
    }

    fn available(self: *const Self) usize {
        if (self.full) return 0;
        if (self.write_pos >= self.read_pos) {
            return self.memory.len - self.write_pos + self.read_pos;
        }
        return self.read_pos - self.write_pos;
    }

    fn allocFn(ctx: *anyopaque, len: usize, ptr_align: u8, _: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const alignment = @as(usize, 1) << @intCast(ptr_align);
        const header_size = @sizeOf(Header);
        const total = header_size + len;

        if (total > self.available()) return null;

        // Check for wrap-around
        const aligned_write = (self.write_pos + @alignOf(Header) - 1) & ~(@alignOf(Header) - 1);
        const data_start = (aligned_write + header_size + alignment - 1) & ~(alignment - 1);

        if (data_start + len <= self.memory.len) {
            // Contiguous
            const header: *Header = @ptrCast(@alignCast(&self.memory[aligned_write]));
            header.* = .{ .size = @intCast(len), .in_use = true };
            self.write_pos = data_start + len;
            if (self.write_pos == self.read_pos) self.full = true;
            return self.memory[data_start..].ptr;
        }

        // Need to wrap
        if (len + header_size > self.read_pos) return null; // Not enough at start

        const header: *Header = @ptrCast(@alignCast(&self.memory[0]));
        header.* = .{ .size = @intCast(len), .in_use = true };
        self.write_pos = header_size + len;
        return self.memory[header_size..].ptr;
    }

    fn resizeFn(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, _: u8, _: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const header_size = @sizeOf(Header);
        const header_ptr = buf.ptr - header_size;
        const header: *Header = @ptrCast(@alignCast(header_ptr));
        header.in_use = false;

        // Advance read_pos ถ้า header ที่ read_pos ถูก free
        while (self.read_pos != self.write_pos or self.full) {
            const h: *Header = @ptrCast(@alignCast(&self.memory[self.read_pos]));
            if (h.in_use) break;
            self.read_pos = (self.read_pos + header_size + h.size) % self.memory.len;
            self.full = false;
        }
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }
};

pub fn main() !void {
    var buffer: [1024]u8 align(8) = undefined;
    var ring = RingAllocator.init(&buffer);
    const alloc = ring.allocator();

    // Simulate streaming allocations
    const a = try alloc.alloc(u8, 50);
    @memset(a, 1);
    std.debug.print("Allocated 50 bytes\n", .{});

    const b = try alloc.alloc(u8, 100);
    @memset(b, 2);
    std.debug.print("Allocated 100 bytes\n", .{});

    // Free first allocation
    alloc.free(a);
    std.debug.print("Freed 50 bytes\n", .{});

    // Can allocate more
    const c = try alloc.alloc(u8, 75);
    @memset(c, 3);
    std.debug.print("Allocated 75 more bytes\n", .{});

    alloc.free(b);
    alloc.free(c);
    std.debug.print("Freed all\n", .{});
}
```

---

## Step 715: Building a Free List Allocator

### Free List Allocator

```zig
const std = @import("std");

const FreeListAllocator = struct {
    const Self = @This();

    const FreeNode = struct {
        size: usize,
        next: ?*FreeNode,
    };

    const AllocHeader = struct {
        size: usize,
        magic: u32,
    };

    const MAGIC: u32 = 0xABCD1234;

    memory: []align(16) u8,
    free_list: ?*FreeNode,
    total_allocated: usize,
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator, capacity: usize) !Self {
        const memory = try backing.alignedAlloc(u8, 16, capacity);
        var self = Self{
            .memory = memory,
            .free_list = null,
            .total_allocated = 0,
            .backing = backing,
        };

        // Initialize with one big free block
        const node: *FreeNode = @ptrCast(@alignCast(memory.ptr));
        node.* = .{
            .size = capacity - @sizeOf(FreeNode),
            .next = null,
        };
        self.free_list = node;
        return self;
    }

    pub fn deinit(self: *Self) void {
        self.backing.free(self.memory);
    }

    fn findFit(self: *Self, size: usize) ?*FreeNode {
        var current = self.free_list;
        var best: ?*FreeNode = null;
        var best_size: usize = std.math.maxInt(usize);

        while (current) |node| {
            if (node.size >= size and node.size < best_size) {
                best = node;
                best_size = node.size;
            }
            current = node.next;
        }
        return best;
    }

    fn removeFromFreeList(self: *Self, target: *FreeNode) void {
        if (self.free_list == target) {
            self.free_list = target.next;
            return;
        }
        var current = self.free_list;
        while (current) |node| {
            if (node.next == target) {
                node.next = target.next;
                return;
            }
            current = node.next;
        }
    }

    fn addToFreeList(self: *Self, node: *FreeNode) void {
        node.next = self.free_list;
        self.free_list = node;
        self.coalesce();
    }

    fn coalesce(self: *Self) void {
        // Simple coalescing - sort ก่อนแล้ว merge adjacent blocks
        var changed = true;
        while (changed) {
            changed = false;
            var curr = self.free_list;
            while (curr) |node| {
                const node_end = @intFromPtr(node) + @sizeOf(FreeNode) + node.size;
                var other = self.free_list;
                while (other) |other_node| {
                    if (other_node != node and @intFromPtr(other_node) == node_end) {
                        // Merge other into node
                        node.size += @sizeOf(FreeNode) + other_node.size;
                        self.removeFromFreeList(other_node);
                        changed = true;
                        break;
                    }
                    other = other_node.next;
                }
                curr = node.next;
            }
        }
    }

    fn allocFn(ctx: *anyopaque, len: usize, ptr_align: u8, _: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        _ = ptr_align;
        const header_size = @sizeOf(AllocHeader);
        const total_needed = header_size + len;
        const min_node_size = @sizeOf(FreeNode);

        const node = self.findFit(total_needed) orelse return null;
        self.removeFromFreeList(node);

        // Split ถ้า block ใหญ่พอ
        if (node.size >= total_needed + min_node_size + 16) {
            const remaining_offset = @intFromPtr(node) + total_needed;
            const remaining: *FreeNode = @ptrFromInt(remaining_offset);
            remaining.* = .{
                .size = node.size - total_needed,
                .next = null,
            };
            self.addToFreeList(remaining);
        }

        const header: *AllocHeader = @ptrCast(node);
        header.* = .{ .size = len, .magic = MAGIC };
        self.total_allocated += len;

        return @ptrFromInt(@intFromPtr(header) + header_size);
    }

    fn resizeFn(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, _: u8, _: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const header_size = @sizeOf(AllocHeader);
        const header: *AllocHeader = @ptrFromInt(@intFromPtr(buf.ptr) - header_size);
        std.debug.assert(header.magic == MAGIC);
        self.total_allocated -= header.size;

        const node: *FreeNode = @ptrCast(header);
        node.* = .{ .size = header_size - @sizeOf(FreeNode) + buf.len, .next = null };
        self.addToFreeList(node);
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }

    pub fn freeBlockCount(self: *const Self) usize {
        var count: usize = 0;
        var curr = self.free_list;
        while (curr) |node| {
            count += 1;
            curr = node.next;
        }
        return count;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var fl_alloc = try FreeListAllocator.init(gpa.allocator(), 1024 * 64);
    defer fl_alloc.deinit();
    const alloc = fl_alloc.allocator();

    std.debug.print("Initial free blocks: {}\n", .{fl_alloc.freeBlockCount()});

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u8, 200);
    const c = try alloc.alloc(u8, 150);

    std.debug.print("After 3 allocs: {} bytes, {} free blocks\n", .{
        fl_alloc.total_allocated, fl_alloc.freeBlockCount()
    });

    alloc.free(b); // Free middle
    std.debug.print("After free middle: {} bytes, {} free blocks\n", .{
        fl_alloc.total_allocated, fl_alloc.freeBlockCount()
    });

    alloc.free(a);
    alloc.free(c);
    std.debug.print("After free all: {} bytes, {} free blocks\n", .{
        fl_alloc.total_allocated, fl_alloc.freeBlockCount()
    });
}
```

---

## Step 716: Building a Buddy Allocator

### Buddy Allocator

```zig
const std = @import("std");

const BuddyAllocator = struct {
    const Self = @This();
    const MIN_ORDER = 4;  // 16 bytes
    const MAX_ORDER = 20; // 1MB
    const NUM_ORDERS = MAX_ORDER - MIN_ORDER + 1;

    const FreeList = struct {
        head: ?usize, // index เป็น offset จาก base
    };

    memory: []align(16) u8,
    free_lists: [NUM_ORDERS]?usize,
    backing: std.mem.Allocator,

    fn blockSize(order: u8) usize {
        return @as(usize, 1) << order;
    }

    fn orderForSize(size: usize) u8 {
        var order: u8 = MIN_ORDER;
        while (blockSize(order) < size and order < MAX_ORDER) : (order += 1) {}
        return order;
    }

    pub fn init(backing: std.mem.Allocator, capacity: usize) !Self {
        const actual_order = orderForSize(capacity);
        const actual_capacity = blockSize(actual_order);
        const memory = try backing.alignedAlloc(u8, 16, actual_capacity);

        var self = Self{
            .memory = memory,
            .free_lists = [_]?usize{null} ** NUM_ORDERS,
            .backing = backing,
        };

        // เพิ่ม block เดียวที่ order สูงสุด
        const order_idx = actual_order - MIN_ORDER;
        self.free_lists[order_idx] = 0;
        const next_ptr: *?usize = @ptrCast(@alignCast(&memory[0]));
        next_ptr.* = null;

        return self;
    }

    pub fn deinit(self: *Self) void {
        self.backing.free(self.memory);
    }

    fn buddyOf(offset: usize, order: u8) usize {
        return offset ^ blockSize(order);
    }

    fn allocAtOrder(self: *Self, order: u8) ?usize {
        const idx = order - MIN_ORDER;
        if (idx >= NUM_ORDERS) return null;

        if (self.free_lists[idx]) |offset| {
            // Pop from free list
            const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[offset]));
            self.free_lists[idx] = next_ptr.*;
            return offset;
        }

        // ไม่มี block ขนาดนี้ - ขอจากขนาดใหญ่กว่าแล้วแบ่ง
        if (order >= MAX_ORDER) return null;

        const larger = self.allocAtOrder(order + 1) orelse return null;

        // แบ่งเป็นสอง buddy
        const buddy = larger + blockSize(order);

        // เพิ่ม buddy เข้า free list
        const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[buddy]));
        next_ptr.* = self.free_lists[idx];
        self.free_lists[idx] = buddy;

        return larger;
    }

    fn freeAtOrder(self: *Self, offset: usize, order: u8) void {
        const idx = order - MIN_ORDER;
        const buddy = buddyOf(offset, order);

        // ค้นหา buddy ใน free list
        if (order < MAX_ORDER) {
            var prev: ?*?usize = null;
            var curr_idx = self.free_lists[idx];
            while (curr_idx) |curr_offset| {
                if (curr_offset == buddy) {
                    // Merge with buddy
                    if (prev) |p| {
                        const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[curr_offset]));
                        p.* = next_ptr.*;
                    } else {
                        const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[curr_offset]));
                        self.free_lists[idx] = next_ptr.*;
                    }
                    const merged = @min(offset, buddy);
                    self.freeAtOrder(merged, order + 1);
                    return;
                }
                const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[curr_offset]));
                prev = next_ptr;
                curr_idx = next_ptr.*;
            }
        }

        // No merge - add to free list
        const next_ptr: *?usize = @ptrCast(@alignCast(&self.memory[offset]));
        next_ptr.* = self.free_lists[idx];
        self.free_lists[idx] = offset;
    }

    fn allocFn(ctx: *anyopaque, len: usize, _: u8, _: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const order = orderForSize(len + @sizeOf(u8)); // +1 for order storage
        const offset = self.allocAtOrder(order) orelse return null;

        // Store order ใน byte แรก
        self.memory[offset] = order;
        return self.memory[offset + 1 ..].ptr;
    }

    fn resizeFn(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, _: u8, _: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const offset = @intFromPtr(buf.ptr) - @intFromPtr(self.memory.ptr) - 1;
        const order = self.memory[offset];
        self.freeAtOrder(offset, order);
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var buddy = try BuddyAllocator.init(gpa.allocator(), 1024 * 1024);
    defer buddy.deinit();
    const alloc = buddy.allocator();

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u8, 1000);
    const c = try alloc.alloc(u8, 10000);

    std.debug.print("Allocated: {}, {}, {} bytes\n", .{a.len, b.len, c.len});

    alloc.free(b);
    alloc.free(a);
    alloc.free(c);
    std.debug.print("All freed\n", .{});
}
```

---

## Step 717: Thread-Local Allocator

### Thread-Local Allocator Pattern

```zig
const std = @import("std");
const Thread = std.Thread;

// Thread-local storage สำหรับ allocator
threadlocal var tl_arena: ?std.heap.ArenaAllocator = null;
threadlocal var tl_initialized: bool = false;

const ThreadLocalAllocator = struct {
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator) @This() {
        return .{ .backing = backing };
    }

    pub fn getThreadAllocator(self: *@This()) std.mem.Allocator {
        if (!tl_initialized) {
            tl_arena = std.heap.ArenaAllocator.init(self.backing);
            tl_initialized = true;
        }
        return tl_arena.?.allocator();
    }

    pub fn resetThread() void {
        if (tl_initialized) {
            _ = tl_arena.?.reset(.free_all);
        }
    }

    pub fn deinitThread() void {
        if (tl_initialized) {
            tl_arena.?.deinit();
            tl_initialized = false;
        }
    }
};

fn workerThread(backing: std.mem.Allocator, id: usize) void {
    var tla = ThreadLocalAllocator.init(backing);
    defer ThreadLocalAllocator.deinitThread();

    const alloc = tla.getThreadAllocator();

    // Allocate thread-local data
    const buf = alloc.alloc(u8, 1024) catch return;
    @memset(buf, @intCast(id));

    std.debug.print("Thread {}: allocated {} bytes, first byte: {}\n", .{
        id, buf.len, buf[0]
    });

    // ไม่ต้อง free - arena จะ cleanup ใน deinitThread
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const threads = try allocator.alloc(Thread, 4);
    defer allocator.free(threads);

    for (threads, 0..) |*t, i| {
        t.* = try Thread.spawn(.{}, workerThread, .{ allocator, i });
    }

    for (threads) |t| {
        t.join();
    }

    std.debug.print("All threads done\n", .{});
}
```

---

## Step 718: Allocator Benchmarking

### Benchmark Framework

```zig
const std = @import("std");
const time = std.time;

const BenchResult = struct {
    name: []const u8,
    iterations: u64,
    total_ns: u64,
    alloc_size: usize,

    pub fn print(self: BenchResult) void {
        const ns_per_op = self.total_ns / self.iterations;
        const ops_per_sec = @as(u64, std.time.ns_per_s) / @max(1, ns_per_op);
        std.debug.print("{s}: {d} ns/op, {d} ops/sec\n", .{
            self.name, ns_per_op, ops_per_sec
        });
    }
};

fn benchAllocator(
    name: []const u8,
    allocator: std.mem.Allocator,
    alloc_size: usize,
    iterations: u64,
) BenchResult {
    const start = time.nanoTimestamp();

    var i: u64 = 0;
    while (i < iterations) : (i += 1) {
        const buf = allocator.alloc(u8, alloc_size) catch continue;
        allocator.free(buf);
    }

    const end = time.nanoTimestamp();
    return BenchResult{
        .name = name,
        .iterations = iterations,
        .total_ns = @intCast(end - start),
        .alloc_size = alloc_size,
    };
}

fn benchAllocatorNoFree(
    name: []const u8,
    allocator: std.mem.Allocator,
    alloc_size: usize,
    iterations: u64,
) BenchResult {
    const start = time.nanoTimestamp();
    var i: u64 = 0;
    while (i < iterations) : (i += 1) {
        _ = allocator.alloc(u8, alloc_size) catch break;
    }
    const end = time.nanoTimestamp();
    return BenchResult{
        .name = name,
        .iterations = i,
        .total_ns = @intCast(end - start),
        .alloc_size = alloc_size,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const iterations: u64 = 10000;
    const alloc_size: usize = 64;

    std.debug.print("=== Allocator Benchmark ({}B x {} iterations) ===\n", .{
        alloc_size, iterations
    });

    // GPA benchmark
    {
        const result = benchAllocator("GPA", gpa.allocator(), alloc_size, iterations);
        result.print();
    }

    // Bump allocator benchmark
    {
        var buf: [1024 * 1024]u8 = undefined;
        // Simple bump - count based
        var offset: usize = 0;
        const start = time.nanoTimestamp();
        var count: u64 = 0;
        while (count < iterations and offset + alloc_size <= buf.len) : (count += 1) {
            _ = buf[offset..offset + alloc_size];
            offset += alloc_size;
        }
        const end = time.nanoTimestamp();
        const total = @as(u64, @intCast(end - start));
        std.debug.print("Bump (no free): {} ns/op\n", .{total / iterations});
    }

    // Arena benchmark
    {
        var arena = std.heap.ArenaAllocator.init(gpa.allocator());
        defer arena.deinit();
        const result = benchAllocatorNoFree("Arena", arena.allocator(), alloc_size, iterations);
        result.print();
    }

    // Page allocator benchmark
    {
        const result = benchAllocator("Page", std.heap.page_allocator, 4096, 100);
        result.print();
    }
}
```

---

## Step 719: Allocator Debugging Utilities

### Debug Allocator Wrapper

```zig
const std = @import("std");

const DebugAllocator = struct {
    const Self = @This();
    const Allocation = struct {
        addr: usize,
        size: usize,
        ret_addr: usize,
        stack_trace: [8]usize,
        stack_depth: usize,
    };

    backing: std.mem.Allocator,
    allocations: std.AutoHashMap(usize, Allocation),
    total_allocated: usize,
    peak_allocated: usize,
    alloc_count: usize,
    free_count: usize,
    double_free_count: usize,
    mutex: std.Thread.Mutex,

    pub fn init(backing: std.mem.Allocator) !*Self {
        const self = try backing.create(Self);
        self.* = .{
            .backing = backing,
            .allocations = std.AutoHashMap(usize, Allocation).init(backing),
            .total_allocated = 0,
            .peak_allocated = 0,
            .alloc_count = 0,
            .free_count = 0,
            .double_free_count = 0,
            .mutex = .{},
        };
        return self;
    }

    pub fn deinit(self: *Self) void {
        if (self.allocations.count() > 0) {
            std.debug.print("MEMORY LEAK: {} allocations not freed!\n", .{
                self.allocations.count()
            });
            var it = self.allocations.iterator();
            while (it.next()) |entry| {
                std.debug.print("  Leaked {} bytes at 0x{X}\n", .{
                    entry.value_ptr.size, entry.key_ptr.*
                });
            }
        }
        self.allocations.deinit();
        const backing = self.backing;
        backing.destroy(self);
    }

    fn allocFn(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        const ptr = self.backing.rawAlloc(len, ptr_align, ret_addr) orelse return null;

        self.mutex.lock();
        defer self.mutex.unlock();

        const addr = @intFromPtr(ptr);
        self.allocations.put(addr, .{
            .addr = addr,
            .size = len,
            .ret_addr = ret_addr,
            .stack_trace = [_]usize{0} ** 8,
            .stack_depth = 0,
        }) catch {};

        self.total_allocated += len;
        if (self.total_allocated > self.peak_allocated) {
            self.peak_allocated = self.total_allocated;
        }
        self.alloc_count += 1;
        return ptr;
    }

    fn resizeFn(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *Self = @ptrCast(@alignCast(ctx));
        if (!self.backing.rawResize(buf, buf_align, new_len, ret_addr)) return false;

        self.mutex.lock();
        defer self.mutex.unlock();

        const addr = @intFromPtr(buf.ptr);
        if (self.allocations.getPtr(addr)) |entry| {
            self.total_allocated = self.total_allocated - entry.size + new_len;
            entry.size = new_len;
        }
        return true;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));

        self.mutex.lock();
        defer self.mutex.unlock();

        const addr = @intFromPtr(buf.ptr);
        if (self.allocations.fetchRemove(addr)) |removed| {
            if (removed.value.size != buf.len) {
                std.debug.print("SIZE MISMATCH: allocated {} but freeing {}\n", .{
                    removed.value.size, buf.len
                });
            }
            self.total_allocated -= removed.value.size;
            self.free_count += 1;
        } else {
            self.double_free_count += 1;
            std.debug.print("DOUBLE FREE or INVALID FREE at 0x{X}!\n", .{addr});
            return;
        }

        self.backing.rawFree(buf, buf_align, ret_addr);
    }

    const vtable = std.mem.Allocator.VTable{
        .alloc = allocFn,
        .resize = resizeFn,
        .free = freeFn,
    };

    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{ .ptr = self, .vtable = &vtable };
    }

    pub fn printReport(self: *const Self) void {
        std.debug.print(
            \\=== Debug Allocator Report ===
            \\Total allocations: {}
            \\Total frees: {}
            \\Active allocations: {}
            \\Current memory: {} bytes
            \\Peak memory: {} bytes
            \\Double frees: {}
            \\
        , .{
            self.alloc_count,
            self.free_count,
            self.allocations.count(),
            self.total_allocated,
            self.peak_allocated,
            self.double_free_count,
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var debug = try DebugAllocator.init(gpa.allocator());
    defer debug.deinit();
    const alloc = debug.allocator();

    const a = try alloc.alloc(u8, 100);
    const b = try alloc.alloc(u32, 50);
    defer alloc.free(b);

    @memset(a, 0xFF);
    for (b) |*v| v.* = 42;

    debug.printReport();

    alloc.free(a);
    debug.printReport();
}
```

---

## Step 720: Choosing the Right Allocator

### Decision Guide

```zig
const std = @import("std");

// Decision framework สำหรับเลือก allocator

// 1. General Purpose - ใช้เมื่อไม่แน่ใจ
pub fn exampleGPA() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    const data = try alloc.alloc(u8, 1024);
    defer alloc.free(data);
    std.debug.print("GPA: good for general use\n", .{});
}

// 2. Arena - ใช้เมื่อ lifetime ชัดเจน
pub fn exampleArena() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();
    const alloc = arena.allocator();

    // Allocate มาก ไม่ต้อง free ทีละอัน
    for (0..100) |i| {
        const s = try std.fmt.allocPrint(alloc, "item_{}", .{i});
        _ = s; // ใช้งาน s
    }
    // Arena จะ free ทั้งหมดตอน deinit
    std.debug.print("Arena: good for request-scoped allocation\n", .{});
}

// 3. Fixed Buffer - ใช้เมื่อรู้ขนาด
pub fn exampleFixedBuffer() !void {
    var buffer: [4096]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const alloc = fba.allocator();

    const data = try alloc.alloc(u8, 100);
    _ = data;
    std.debug.print("FixedBuffer: good for stack allocation\n", .{});
}

// 4. Page allocator - ใช้สำหรับ large allocations
pub fn examplePageAllocator() !void {
    const alloc = std.heap.page_allocator;
    const data = try alloc.alloc(u8, 1024 * 1024); // 1MB
    defer alloc.free(data);
    std.debug.print("Page allocator: good for large, infrequent allocations\n", .{});
}

// Comparison table
pub fn printComparisonTable() void {
    std.debug.print(
        \\
        \\=== Allocator Comparison ===
        \\
        \\Allocator          | Speed | Memory | Thread-safe | Free individual?
        \\--------------------|-------|--------|-------------|------------------
        \\GeneralPurpose     | Med   | Good   | No          | Yes
        \\ArenaAllocator     | Fast  | Good   | No          | Batch only
        \\FixedBuffer        | Fast  | Fixed  | No          | Yes (LIFO)
        \\BumpAllocator      | Fast  | Poor   | No          | No
        \\page_allocator     | Slow  | Waste  | Yes         | Yes (4KB min)
        \\MemoryPool         | Fast  | Good   | Opt.        | Yes (same type)
        \\SlabAllocator      | Fast  | Good   | No          | Yes (same size)
        \\BuddyAllocator     | Med   | Good   | No          | Yes (power-of-2)
        \\
    , .{});
}

pub fn main() !void {
    try exampleGPA();
    try exampleArena();
    try exampleFixedBuffer();
    try examplePageAllocator();
    printComparisonTable();
}
```

---

## สรุป Part 72

| Step | Allocator | Best For |
|------|-----------|----------|
| 711 | Interface | เข้าใจ Allocator API |
| 712 | Bump | Temporary allocations, reset ทั้งหมด |
| 713 | Stack | LIFO allocations |
| 714 | Ring Buffer | Streaming data |
| 715 | Free List | General purpose, fragmentation tolerable |
| 716 | Buddy | Power-of-2 sizes, minimal fragmentation |
| 717 | Thread-Local | Per-thread allocation |
| 718 | Benchmarking | วัดประสิทธิภาพ |
| 719 | Debug | ตรวจหา memory leaks |
| 720 | Choosing | เลือก allocator ที่เหมาะสม |

---

*ต่อไป: Part 73 - Arena Allocator Pattern*
