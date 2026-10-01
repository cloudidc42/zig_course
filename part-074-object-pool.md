# ส่วนที่ 74: Object Pool Pattern

## ภาพรวม

Object Pool Pattern เป็น pattern การจัดการหน่วยความจำที่สร้าง objects ไว้ล่วงหน้าและนำมาใช้ซ้ำแทนการ allocate/deallocate บ่อยๆ ช่วยลด allocation overhead และลด fragmentation อย่างมาก

---

## Step 731: Object Pool Concept

### แนวคิดหลัก

```zig
const std = @import("std");

// Object Pool แก้ปัญหา:
// 1. Allocation overhead สูงสำหรับ small, frequent objects
// 2. Memory fragmentation
// 3. Cache miss จาก scattered memory

// ตัวอย่างปัญหา: สร้าง/ทำลาย connection บ่อยๆ
const Connection = struct {
    id: u32,
    host: [64]u8,
    port: u16,
    is_open: bool,
    bytes_sent: u64,
    bytes_recv: u64,

    pub fn connect(self: *Connection, host: []const u8, port: u16, id: u32) void {
        self.id = id;
        @memset(&self.host, 0);
        @memcpy(self.host[0..@min(host.len, 63)], host[0..@min(host.len, 63)]);
        self.port = port;
        self.is_open = true;
        self.bytes_sent = 0;
        self.bytes_recv = 0;
    }

    pub fn disconnect(self: *Connection) void {
        self.is_open = false;
    }

    pub fn send(self: *Connection, bytes: usize) void {
        self.bytes_sent += bytes;
    }

    pub fn receive(self: *Connection, bytes: usize) void {
        self.bytes_recv += bytes;
    }
};

// Without pool: allocate/free every time
pub fn withoutPool(allocator: std.mem.Allocator, iterations: usize) !void {
    const start = std.time.nanoTimestamp();

    var i: usize = 0;
    while (i < iterations) : (i += 1) {
        const conn = try allocator.create(Connection);
        conn.connect("example.com", 80, @intCast(i));
        conn.send(1024);
        conn.receive(512);
        conn.disconnect();
        allocator.destroy(conn);
    }

    const elapsed = std.time.nanoTimestamp() - start;
    std.debug.print("Without pool: {} ns ({} ns/iter)\n", .{
        elapsed, @divFloor(elapsed, @as(i64, @intCast(iterations)))
    });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    try withoutPool(alloc, 10000);
    std.debug.print("Connection struct size: {} bytes\n", .{@sizeOf(Connection)});
}
```

### Pool Concept Diagram

```zig
// Pool ทำงานแบบนี้:
//
// Initial state:
// [free][free][free][free][free]
//   ^
//   free_list -> slot0 -> slot1 -> slot2 -> slot3 -> slot4 -> null
//
// After 2 acquires:
// [used][used][free][free][free]
//               ^
//   free_list -> slot2 -> slot3 -> slot4 -> null
//
// After 1 release (slot0):
// [free][used][free][free][free]
//   ^
//   free_list -> slot0 -> slot2 -> slot3 -> slot4 -> null
```

---

## Step 732: Fixed-Size Object Pool

### Implementation พื้นฐาน

```zig
const std = @import("std");

fn FixedObjectPool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();
        const Node = struct {
            next: ?*Node,
            // T follows immediately after
        };

        // Storage สำหรับ objects
        storage: [capacity]T align(@alignOf(T)) = undefined,
        // Free list nodes (ใช้ space ใน storage เมื่อ object ไม่ได้ใช้)
        free_list: ?*Node,
        // ข้อมูล pool
        used_count: usize,
        total_count: usize,

        pub fn init(self: *Self) void {
            self.free_list = null;
            self.used_count = 0;
            self.total_count = capacity;

            // สร้าง free list โดย interpret storage เป็น nodes
            var i: usize = capacity;
            while (i > 0) {
                i -= 1;
                const node: *Node = @ptrCast(@alignCast(&self.storage[i]));
                node.next = self.free_list;
                self.free_list = node;
            }
        }

        pub fn acquire(self: *Self) ?*T {
            const node = self.free_list orelse return null;
            self.free_list = node.next;
            self.used_count += 1;
            const obj: *T = @ptrCast(@alignCast(node));
            return obj;
        }

        pub fn release(self: *Self, obj: *T) void {
            std.debug.assert(self.used_count > 0);
            const node: *Node = @ptrCast(@alignCast(obj));
            node.next = self.free_list;
            self.free_list = node;
            self.used_count -= 1;
        }

        pub fn available(self: *const Self) usize {
            return self.total_count - self.used_count;
        }

        pub fn contains(self: *const Self, obj: *const T) bool {
            const base = @intFromPtr(&self.storage[0]);
            const addr = @intFromPtr(obj);
            const size = @sizeOf(T);
            return addr >= base and addr < base + capacity * size and
                   (addr - base) % size == 0;
        }
    };
}

const Bullet = struct {
    x: f32,
    y: f32,
    vx: f32,
    vy: f32,
    damage: f32,
    active: bool,

    pub fn fire(self: *Bullet, x: f32, y: f32, angle: f32, speed: f32, dmg: f32) void {
        self.x = x;
        self.y = y;
        self.vx = @cos(angle) * speed;
        self.vy = @sin(angle) * speed;
        self.damage = dmg;
        self.active = true;
    }

    pub fn update(self: *Bullet, dt: f32) void {
        self.x += self.vx * dt;
        self.y += self.vy * dt;
    }

    pub fn isOffScreen(self: *const Bullet) bool {
        return self.x < 0 or self.x > 800 or self.y < 0 or self.y > 600;
    }
};

pub fn main() void {
    var pool: FixedObjectPool(Bullet, 256) = undefined;
    pool.init();

    std.debug.print("Pool capacity: {}, available: {}\n", .{
        pool.total_count, pool.available()
    });

    // Fire some bullets
    var active_bullets = std.ArrayList(*Bullet).init(std.heap.page_allocator);
    defer active_bullets.deinit();

    var i: usize = 0;
    while (i < 10) : (i += 1) {
        if (pool.acquire()) |bullet| {
            bullet.fire(400, 300, @as(f32, @floatFromInt(i)) * 0.628, 5.0, 10.0);
            active_bullets.append(bullet) catch unreachable;
        }
    }

    std.debug.print("After fire: available={}\n", .{pool.available()});

    // Update and release dead bullets
    const dt: f32 = 0.016;
    var j: usize = 0;
    while (j < active_bullets.items.len) {
        const bullet = active_bullets.items[j];
        bullet.update(dt);
        if (bullet.isOffScreen()) {
            pool.release(bullet);
            _ = active_bullets.swapRemove(j);
        } else {
            j += 1;
        }
    }

    std.debug.print("After update: active={}, available={}\n", .{
        active_bullets.items.len, pool.available()
    });

    // Release remaining
    for (active_bullets.items) |b| pool.release(b);
    std.debug.print("After release all: available={}\n", .{pool.available()});
}
```

---

## Step 733: Dynamic Object Pool

### Pool ที่ขยายตัวได้

```zig
const std = @import("std");

fn DynamicObjectPool(comptime T: type) type {
    return struct {
        const Self = @This();
        const CHUNK_SIZE = 64;

        const Chunk = struct {
            objects: []T,
            in_use: []bool,
            next: ?*Chunk,
        };

        chunks: ?*Chunk,
        free_count: usize,
        total_count: usize,
        allocator: std.mem.Allocator,

        pub fn init(allocator: std.mem.Allocator) Self {
            return Self{
                .chunks = null,
                .free_count = 0,
                .total_count = 0,
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            var chunk = self.chunks;
            while (chunk) |c| {
                const next = c.next;
                self.allocator.free(c.objects);
                self.allocator.free(c.in_use);
                self.allocator.destroy(c);
                chunk = next;
            }
        }

        fn addChunk(self: *Self) !void {
            const chunk = try self.allocator.create(Chunk);
            chunk.objects = try self.allocator.alloc(T, CHUNK_SIZE);
            chunk.in_use = try self.allocator.alloc(bool, CHUNK_SIZE);
            @memset(chunk.in_use, false);
            chunk.next = self.chunks;
            self.chunks = chunk;
            self.free_count += CHUNK_SIZE;
            self.total_count += CHUNK_SIZE;
        }

        pub fn acquire(self: *Self) !*T {
            // หา free slot
            var chunk = self.chunks;
            while (chunk) |c| {
                for (c.in_use, 0..) |*used, i| {
                    if (!used.*) {
                        used.* = true;
                        self.free_count -= 1;
                        return &c.objects[i];
                    }
                }
                chunk = c.next;
            }

            // ไม่มี slot ว่าง - เพิ่ม chunk ใหม่
            try self.addChunk();
            return self.acquire();
        }

        pub fn release(self: *Self, obj: *T) void {
            var chunk = self.chunks;
            while (chunk) |c| {
                const base = @intFromPtr(c.objects.ptr);
                const addr = @intFromPtr(obj);
                const size = @sizeOf(T);
                if (addr >= base and addr < base + CHUNK_SIZE * size) {
                    const index = (addr - base) / size;
                    std.debug.assert(c.in_use[index]);
                    c.in_use[index] = false;
                    self.free_count += 1;
                    return;
                }
                chunk = c.next;
            }
            unreachable; // obj ไม่ได้มาจาก pool นี้
        }

        pub fn stats(self: *const Self) struct { total: usize, free: usize, used: usize } {
            return .{
                .total = self.total_count,
                .free = self.free_count,
                .used = self.total_count - self.free_count,
            };
        }
    };
}

const DatabaseRow = struct {
    id: u64,
    table: [32]u8,
    data: [256]u8,
    timestamp: i64,

    pub fn init(self: *DatabaseRow, id: u64, table_name: []const u8) void {
        self.id = id;
        @memset(&self.table, 0);
        @memcpy(self.table[0..@min(table_name.len, 31)], table_name[0..@min(table_name.len, 31)]);
        @memset(&self.data, 0);
        self.timestamp = std.time.timestamp();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var pool = DynamicObjectPool(DatabaseRow).init(gpa.allocator());
    defer pool.deinit();

    std.debug.print("Initial: {}\n", .{pool.stats()});

    // Acquire หลาย rows
    var rows = std.ArrayList(*DatabaseRow).init(gpa.allocator());
    defer rows.deinit();

    for (0..150) |i| {
        const row = try pool.acquire();
        row.init(@intCast(i), "users");
        try rows.append(row);
    }

    std.debug.print("After 150 acquires: {}\n", .{pool.stats()});

    // Release 50
    for (rows.items[0..50]) |r| {
        pool.release(r);
    }

    std.debug.print("After 50 releases: {}\n", .{pool.stats()});

    // Acquire again
    for (0..30) |i| {
        const row = try pool.acquire();
        row.init(@intCast(i + 1000), "orders");
        try rows.append(row);
    }

    std.debug.print("After 30 more acquires: {}\n", .{pool.stats()});
}
```

---

## Step 734: Type-Safe Object Pool

### Pool พร้อม Type Safety

```zig
const std = @import("std");

// Handle-based pool สำหรับ type safety
fn TypedPool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        pub const Handle = struct {
            index: u32,
            generation: u32,

            pub fn invalid() Handle {
                return Handle{ .index = std.math.maxInt(u32), .generation = 0 };
            }

            pub fn isValid(self: Handle) bool {
                return self.index != std.math.maxInt(u32);
            }
        };

        const Slot = struct {
            object: T,
            generation: u32,
            in_use: bool,
        };

        slots: [capacity]Slot,
        free_indices: [capacity]u32,
        free_count: usize,

        pub fn init(self: *Self) void {
            self.free_count = capacity;
            for (&self.slots, 0..) |*slot, i| {
                slot.generation = 0;
                slot.in_use = false;
                self.free_indices[i] = @intCast(i);
            }
        }

        pub fn acquire(self: *Self) ?struct { handle: Handle, ptr: *T } {
            if (self.free_count == 0) return null;

            self.free_count -= 1;
            const index = self.free_indices[self.free_count];
            const slot = &self.slots[index];

            slot.in_use = true;
            const handle = Handle{
                .index = index,
                .generation = slot.generation,
            };

            return .{ .handle = handle, .ptr = &slot.object };
        }

        pub fn release(self: *Self, handle: Handle) bool {
            if (!handle.isValid()) return false;
            if (handle.index >= capacity) return false;

            const slot = &self.slots[handle.index];
            if (!slot.in_use) return false;
            if (slot.generation != handle.generation) return false; // Stale handle

            slot.in_use = false;
            slot.generation += 1; // Invalidate existing handles

            self.free_indices[self.free_count] = handle.index;
            self.free_count += 1;
            return true;
        }

        pub fn get(self: *Self, handle: Handle) ?*T {
            if (!handle.isValid()) return null;
            if (handle.index >= capacity) return null;

            const slot = &self.slots[handle.index];
            if (!slot.in_use) return null;
            if (slot.generation != handle.generation) return null; // Stale handle

            return &slot.object;
        }

        pub fn getConst(self: *const Self, handle: Handle) ?*const T {
            if (!handle.isValid()) return null;
            if (handle.index >= capacity) return null;

            const slot = &self.slots[handle.index];
            if (!slot.in_use) return null;
            if (slot.generation != handle.generation) return null;

            return &slot.object;
        }

        pub fn isValid(self: *const Self, handle: Handle) bool {
            if (!handle.isValid()) return false;
            if (handle.index >= capacity) return false;
            const slot = &self.slots[handle.index];
            return slot.in_use and slot.generation == handle.generation;
        }
    };
}

const Texture = struct {
    width: u32,
    height: u32,
    format: u8,
    data: [1024]u8, // ขนาดจำลอง

    pub fn init(self: *Texture, w: u32, h: u32, fmt: u8) void {
        self.width = w;
        self.height = h;
        self.format = fmt;
        @memset(&self.data, 0);
    }
};

pub fn main() void {
    var pool: TypedPool(Texture, 32) = undefined;
    pool.init();

    // Acquire texture
    const result1 = pool.acquire() orelse {
        std.debug.print("Pool full!\n", .{});
        return;
    };
    result1.ptr.init(256, 256, 1);
    const handle1 = result1.handle;

    const result2 = pool.acquire() orelse return;
    result2.ptr.init(512, 512, 2);
    const handle2 = result2.handle;

    std.debug.print("Handle1 valid: {}\n", .{pool.isValid(handle1)});
    std.debug.print("Handle2 valid: {}\n", .{pool.isValid(handle2)});

    // Get texture
    if (pool.get(handle1)) |tex| {
        std.debug.print("Texture1: {}x{}\n", .{tex.width, tex.height});
    }

    // Release
    _ = pool.release(handle1);
    std.debug.print("After release: handle1 valid: {}\n", .{pool.isValid(handle1)});

    // Use stale handle - ปลอดภัย!
    if (pool.get(handle1)) |_| {
        std.debug.print("ERROR: should not get released object\n", .{});
    } else {
        std.debug.print("Stale handle correctly rejected\n", .{});
    }

    _ = pool.release(handle2);
}
```

---

## Step 735: Pool with Initialization

### Pool ที่มี Constructor/Destructor

```zig
const std = @import("std");

fn ManagedPool(
    comptime T: type,
    comptime capacity: usize,
    comptime init_fn: fn (*T) void,
    comptime deinit_fn: fn (*T) void,
) type {
    return struct {
        const Self = @This();

        const FreeNode = struct {
            next: ?*FreeNode,
        };

        storage: [capacity]T align(@alignOf(T)),
        free_list: ?*FreeNode,
        initialized: [capacity]bool,
        used_count: usize,

        pub fn create(self: *Self) void {
            self.free_list = null;
            self.used_count = 0;
            @memset(&self.initialized, false);

            var i: usize = capacity;
            while (i > 0) {
                i -= 1;
                const node: *FreeNode = @ptrCast(@alignCast(&self.storage[i]));
                node.next = self.free_list;
                self.free_list = node;
            }
        }

        pub fn acquire(self: *Self) ?*T {
            const node = self.free_list orelse return null;
            self.free_list = node.next;

            const obj: *T = @ptrCast(@alignCast(node));
            const idx = (@intFromPtr(obj) - @intFromPtr(&self.storage[0])) / @sizeOf(T);

            if (!self.initialized[idx]) {
                init_fn(obj);
                self.initialized[idx] = true;
            } else {
                // Re-initialize เมื่อ reuse
                init_fn(obj);
            }

            self.used_count += 1;
            return obj;
        }

        pub fn release(self: *Self, obj: *T) void {
            deinit_fn(obj);
            const node: *FreeNode = @ptrCast(@alignCast(obj));
            node.next = self.free_list;
            self.free_list = node;
            self.used_count -= 1;
        }

        pub fn destroyAll(self: *Self) void {
            // deinit ทุก object ที่ initialized
            for (&self.storage, 0..) |*obj, i| {
                if (self.initialized[i]) {
                    deinit_fn(obj);
                    self.initialized[i] = false;
                }
            }
        }
    };
}

const NetworkSocket = struct {
    fd: i32,
    buffer: [4096]u8,
    buffer_len: usize,
    connected: bool,
    ref_count: u32,

    var next_fd: i32 = 1000;
};

fn socketInit(s: *NetworkSocket) void {
    s.fd = NetworkSocket.next_fd;
    NetworkSocket.next_fd += 1;
    s.buffer_len = 0;
    s.connected = true;
    s.ref_count = 1;
    std.debug.print("Socket {} opened\n", .{s.fd});
}

fn socketDeinit(s: *NetworkSocket) void {
    if (s.connected) {
        std.debug.print("Socket {} closed\n", .{s.fd});
        s.connected = false;
    }
}

pub fn main() void {
    var pool: ManagedPool(
        NetworkSocket,
        8,
        socketInit,
        socketDeinit,
    ) = undefined;
    pool.create();
    defer pool.destroyAll();

    // Acquire sockets
    const s1 = pool.acquire().?;
    const s2 = pool.acquire().?;
    const s3 = pool.acquire().?;

    std.debug.print("Active sockets: {}\n", .{pool.used_count});
    std.debug.print("s1.fd={}, s2.fd={}, s3.fd={}\n", .{s1.fd, s2.fd, s3.fd});

    // Release one
    pool.release(s2);
    std.debug.print("After release s2: {} active\n", .{pool.used_count});

    // Acquire again - reuses s2's slot
    const s4 = pool.acquire().?;
    std.debug.print("s4.fd={} (reused slot)\n", .{s4.fd});

    pool.release(s1);
    pool.release(s3);
    pool.release(s4);
}
```

---

## Step 736: Pool Statistics

### Pool ที่เก็บ Statistics

```zig
const std = @import("std");

fn StatisticalPool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        const Stats = struct {
            total_acquires: u64,
            total_releases: u64,
            pool_empty_count: u64,
            peak_usage: usize,
            current_usage: usize,
            double_free_count: u64,

            pub fn print(self: *const Stats) void {
                std.debug.print(
                    \\Pool Statistics:
                    \\  Total acquires:    {}
                    \\  Total releases:    {}
                    \\  Peak usage:        {}/{}
                    \\  Pool empty count:  {}
                    \\  Double frees:      {}
                    \\  Hit rate:          {d:.1}%
                    \\
                , .{
                    self.total_acquires,
                    self.total_releases,
                    self.peak_usage,
                    capacity,
                    self.pool_empty_count,
                    self.double_free_count,
                    if (self.total_acquires > 0)
                        @as(f64, @floatFromInt(self.total_acquires - self.pool_empty_count)) /
                        @as(f64, @floatFromInt(self.total_acquires)) * 100.0
                    else
                        0.0,
                });
            }
        };

        storage: [capacity]T align(@alignOf(T)),
        in_use: [capacity]bool,
        acquire_time: [capacity]i64,
        stats: Stats,

        pub fn init(self: *Self) void {
            @memset(&self.in_use, false);
            @memset(&self.acquire_time, 0);
            self.stats = std.mem.zeroes(Stats);
        }

        pub fn acquire(self: *Self) ?*T {
            self.stats.total_acquires += 1;

            for (&self.in_use, 0..) |*used, i| {
                if (!used.*) {
                    used.* = true;
                    self.acquire_time[i] = std.time.timestamp();
                    self.stats.current_usage += 1;
                    if (self.stats.current_usage > self.stats.peak_usage) {
                        self.stats.peak_usage = self.stats.current_usage;
                    }
                    return &self.storage[i];
                }
            }

            self.stats.pool_empty_count += 1;
            return null;
        }

        pub fn release(self: *Self, obj: *T) void {
            self.stats.total_releases += 1;

            const base = @intFromPtr(&self.storage[0]);
            const addr = @intFromPtr(obj);
            const index = (addr - base) / @sizeOf(T);

            if (index >= capacity or !self.in_use[index]) {
                self.stats.double_free_count += 1;
                std.debug.print("WARNING: Double free detected!\n", .{});
                return;
            }

            self.in_use[index] = false;
            self.stats.current_usage -= 1;
        }

        pub fn printStats(self: *const Self) void {
            self.stats.print();
        }

        pub fn utilizationPercent(self: *const Self) f64 {
            return @as(f64, @floatFromInt(self.stats.current_usage)) /
                   @as(f64, @floatFromInt(capacity)) * 100.0;
        }
    };
}

const Worker = struct {
    id: u32,
    task: []const u8,
    started: i64,
};

pub fn main() void {
    var pool: StatisticalPool(Worker, 16) = undefined;
    pool.init();

    var prng = std.Random.DefaultPrng.init(42);
    const rand = prng.random();

    // Simulate workload
    var active = std.ArrayList(*Worker).init(std.heap.page_allocator);
    defer active.deinit();

    var i: usize = 0;
    while (i < 100) : (i += 1) {
        // Randomly acquire or release
        if (active.items.len == 0 or rand.boolean()) {
            if (pool.acquire()) |w| {
                w.id = @intCast(i);
                w.task = "processing";
                w.started = std.time.timestamp();
                active.append(w) catch unreachable;
            }
        } else {
            const idx = rand.uintLessThan(usize, active.items.len);
            pool.release(active.items[idx]);
            _ = active.swapRemove(idx);
        }
    }

    // Release remaining
    for (active.items) |w| pool.release(w);
    active.clearRetainingCapacity();

    pool.printStats();
    std.debug.print("Utilization: {d:.1}%\n", .{pool.utilizationPercent()});
}
```

---

## Step 737: Thread-Safe Object Pool

### Concurrent Pool

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Condition = Thread.Condition;

fn ThreadSafePool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        storage: [capacity]T align(@alignOf(T)),
        in_use: [capacity]bool,
        mutex: Mutex,
        available_cond: Condition,
        used_count: usize,

        pub fn init(self: *Self) void {
            @memset(&self.in_use, false);
            self.mutex = .{};
            self.available_cond = .{};
            self.used_count = 0;
        }

        // Non-blocking: ส่งคืน null ถ้า pool เต็ม
        pub fn tryAcquire(self: *Self) ?*T {
            self.mutex.lock();
            defer self.mutex.unlock();

            for (&self.in_use, 0..) |*used, i| {
                if (!used.*) {
                    used.* = true;
                    self.used_count += 1;
                    return &self.storage[i];
                }
            }
            return null;
        }

        // Blocking: รอจนกว่า pool จะมี object ว่าง
        pub fn acquire(self: *Self) *T {
            self.mutex.lock();
            defer self.mutex.unlock();

            while (true) {
                for (&self.in_use, 0..) |*used, i| {
                    if (!used.*) {
                        used.* = true;
                        self.used_count += 1;
                        return &self.storage[i];
                    }
                }
                // Pool เต็ม - รอ
                self.available_cond.wait(&self.mutex);
            }
        }

        // Acquire ด้วย timeout
        pub fn acquireTimeout(self: *Self, timeout_ns: u64) ?*T {
            self.mutex.lock();
            defer self.mutex.unlock();

            const deadline = std.time.nanoTimestamp() + @as(i128, @intCast(timeout_ns));

            while (true) {
                for (&self.in_use, 0..) |*used, i| {
                    if (!used.*) {
                        used.* = true;
                        self.used_count += 1;
                        return &self.storage[i];
                    }
                }

                const now = std.time.nanoTimestamp();
                if (now >= deadline) return null;

                const remaining: u64 = @intCast(deadline - now);
                self.available_cond.timedWait(&self.mutex, remaining) catch return null;
            }
        }

        pub fn release(self: *Self, obj: *T) void {
            self.mutex.lock();
            defer self.mutex.unlock();

            const base = @intFromPtr(&self.storage[0]);
            const addr = @intFromPtr(obj);
            const index = (addr - base) / @sizeOf(T);

            std.debug.assert(index < capacity);
            std.debug.assert(self.in_use[index]);

            self.in_use[index] = false;
            self.used_count -= 1;
            self.available_cond.signal(); // Wake up waiting thread
        }

        pub fn usedCount(self: *Self) usize {
            self.mutex.lock();
            defer self.mutex.unlock();
            return self.used_count;
        }
    };
}

const ConnectionInfo = struct {
    id: u32,
    thread_id: usize,
    query: [256]u8,
};

var global_pool: ThreadSafePool(ConnectionInfo, 4) = undefined;
var completed: std.atomic.Value(u32) = std.atomic.Value(u32).init(0);

fn workerTask(thread_id: usize) void {
    var i: usize = 0;
    while (i < 5) : (i += 1) {
        // ขอ connection (blocking)
        const conn = global_pool.acquire();
        conn.id = @intCast(thread_id * 100 + i);
        conn.thread_id = thread_id;
        @memset(&conn.query, 0);

        const query = std.fmt.bufPrint(&conn.query, "SELECT * FROM table_{}", .{i}) catch unreachable;
        std.debug.print("Thread {}: got connection {}, query: {s}\n", .{
            thread_id, conn.id, query
        });

        // Simulate work
        std.time.sleep(1 * std.time.ns_per_ms);

        global_pool.release(conn);
    }
    _ = completed.fetchAdd(1, .monotonic);
}

pub fn main() !void {
    global_pool.init();

    const num_threads = 6;
    var threads: [num_threads]Thread = undefined;

    for (&threads, 0..) |*t, i| {
        t.* = try Thread.spawn(.{}, workerTask, .{i});
    }

    for (&threads) |*t| t.join();

    std.debug.print("All {} threads completed\n", .{completed.load(.monotonic)});
    std.debug.print("Pool used count: {}\n", .{global_pool.usedCount()});
}
```

---

## Step 738: Pool Fragmentation

### Handling Pool Fragmentation

```zig
const std = @import("std");

// Fragmentation analysis
fn PoolAnalyzer(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        in_use: [capacity]bool,

        pub fn init(self: *Self) void {
            @memset(&self.in_use, false);
        }

        pub fn markUsed(self: *Self, index: usize) void {
            self.in_use[index] = true;
        }

        pub fn markFree(self: *Self, index: usize) void {
            self.in_use[index] = false;
        }

        // วัด fragmentation
        pub fn fragmentation(self: *const Self) f64 {
            var free_runs: usize = 0;
            var in_run = false;
            var total_free: usize = 0;

            for (self.in_use) |used| {
                if (!used) {
                    total_free += 1;
                    if (!in_run) {
                        free_runs += 1;
                        in_run = true;
                    }
                } else {
                    in_run = false;
                }
            }

            if (total_free == 0) return 0.0;
            // Fragmentation = 1 - (largest free block / total free)
            return @as(f64, @floatFromInt(free_runs - 1)) /
                   @as(f64, @floatFromInt(total_free));
        }

        // หา contiguous free blocks
        pub fn largestFreeBlock(self: *const Self) usize {
            var max_run: usize = 0;
            var current_run: usize = 0;

            for (self.in_use) |used| {
                if (!used) {
                    current_run += 1;
                    if (current_run > max_run) max_run = current_run;
                } else {
                    current_run = 0;
                }
            }
            return max_run;
        }

        pub fn printMap(self: *const Self) void {
            std.debug.print("Pool map: [", .{});
            for (self.in_use) |used| {
                std.debug.print("{c}", .{if (used) @as(u8, 'X') else @as(u8, '.')});
            }
            std.debug.print("]\n", .{});
            std.debug.print("Fragmentation: {d:.2}%\n", .{self.fragmentation() * 100.0});
            std.debug.print("Largest free block: {}\n", .{self.largestFreeBlock()});
        }
    };
}

// Defragmenting Pool - compacts free space
fn DefragPool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        objects: [capacity]T align(@alignOf(T)),
        in_use: [capacity]bool,
        handles: [capacity]usize, // ถ้าใช้ handle-based system
        used_count: usize,

        pub fn init(self: *Self) void {
            @memset(&self.in_use, false);
            self.used_count = 0;
            for (&self.handles, 0..) |*h, i| h.* = i;
        }

        // Compact: เลื่อน used objects ไปข้างหน้า
        pub fn compact(self: *Self) void {
            var write_idx: usize = 0;
            for (0..capacity) |read_idx| {
                if (self.in_use[read_idx]) {
                    if (write_idx != read_idx) {
                        self.objects[write_idx] = self.objects[read_idx];
                        self.in_use[write_idx] = true;
                        self.in_use[read_idx] = false;
                    }
                    write_idx += 1;
                }
            }
            std.debug.print("Compacted: {} used objects at start\n", .{write_idx});
        }
    };
}

pub fn main() void {
    var analyzer: PoolAnalyzer(u64, 20) = undefined;
    analyzer.init();

    // Simulate allocation pattern ที่ทำให้ fragmented
    const alloc_pattern = [_]bool{
        true, true, true, true, true, true, true, true, true, true,
        false, false, false, false, false, false, false, false, false, false,
    };
    for (alloc_pattern, 0..) |used, i| {
        if (used) analyzer.markUsed(i);
    }

    analyzer.printMap();

    // Free odd indices
    var i: usize = 1;
    while (i < 20) : (i += 2) {
        if (analyzer.in_use[i]) {
            analyzer.markFree(i);
        }
    }

    std.debug.print("\nAfter freeing odd indices:\n", .{});
    analyzer.printMap();

    // Free some even indices
    analyzer.markFree(0);
    analyzer.markFree(4);
    analyzer.markFree(8);

    std.debug.print("\nAfter more frees:\n", .{});
    analyzer.printMap();
}
```

---

## Step 739: Pool vs Allocator Comparison

### เปรียบเทียบ Pool กับ Allocator

```zig
const std = @import("std");
const time = std.time;

const SmallObject = struct {
    id: u32,
    data: [32]u8,
    value: f64,
};

fn FixedPool(comptime T: type, comptime cap: usize) type {
    return struct {
        const Self = @This();
        items: [cap]T,
        used: [cap]bool,
        free_count: usize,

        pub fn init(self: *Self) void {
            @memset(&self.used, false);
            self.free_count = cap;
        }

        pub fn get(self: *Self) ?*T {
            for (&self.used, 0..) |*u, i| {
                if (!u.*) {
                    u.* = true;
                    self.free_count -= 1;
                    return &self.items[i];
                }
            }
            return null;
        }

        pub fn put(self: *Self, p: *T) void {
            const idx = (@intFromPtr(p) - @intFromPtr(&self.items[0])) / @sizeOf(T);
            self.used[idx] = false;
            self.free_count += 1;
        }
    };
}

fn benchPool(n: usize) !u64 {
    var pool: FixedPool(SmallObject, 1024) = undefined;
    pool.init();

    const start = time.nanoTimestamp();

    var i: usize = 0;
    while (i < n) : (i += 1) {
        const obj = pool.get() orelse {
            // Pool exhausted - reset (simulate frame reset)
            pool.init();
            continue;
        };
        obj.id = @intCast(i);
        obj.value = @as(f64, @floatFromInt(i)) * 0.1;
        pool.put(obj);
    }

    return @intCast(time.nanoTimestamp() - start);
}

fn benchAllocator(allocator: std.mem.Allocator, n: usize) !u64 {
    const start = time.nanoTimestamp();

    var i: usize = 0;
    while (i < n) : (i += 1) {
        const obj = try allocator.create(SmallObject);
        obj.id = @intCast(i);
        obj.value = @as(f64, @floatFromInt(i)) * 0.1;
        allocator.destroy(obj);
    }

    return @intCast(time.nanoTimestamp() - start);
}

fn benchArena(backing: std.mem.Allocator, n: usize, batch: usize) !u64 {
    const start = time.nanoTimestamp();

    var arena = std.heap.ArenaAllocator.init(backing);
    defer arena.deinit();

    var i: usize = 0;
    while (i < n) {
        const alloc = arena.allocator();
        const end = @min(i + batch, n);
        while (i < end) : (i += 1) {
            const obj = try alloc.create(SmallObject);
            obj.id = @intCast(i);
            obj.value = @as(f64, @floatFromInt(i)) * 0.1;
        }
        _ = arena.reset(.retain_capacity);
    }

    return @intCast(time.nanoTimestamp() - start);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const N: usize = 100_000;
    std.debug.print("=== Pool vs Allocator Benchmark (N={}) ===\n\n", .{N});
    std.debug.print("Object size: {} bytes\n\n", .{@sizeOf(SmallObject)});

    const pool_time = try benchPool(N);
    std.debug.print("Pool:       {} ns ({} ns/op)\n", .{ pool_time, pool_time / N });

    const gpa_time = try benchAllocator(allocator, N);
    std.debug.print("GPA:        {} ns ({} ns/op)\n", .{ gpa_time, gpa_time / N });

    const arena_time = try benchArena(allocator, N, 1000);
    std.debug.print("Arena(1k):  {} ns ({} ns/op)\n", .{ arena_time, arena_time / N });

    std.debug.print("\nPool speedup vs GPA:   {d:.2}x\n", .{
        @as(f64, @floatFromInt(gpa_time)) / @as(f64, @floatFromInt(pool_time))
    });
}
```

---

## Step 740: Real-World Pool Usage

### Game Entity Pool System

```zig
const std = @import("std");

// Entity component system ด้วย pools
const EntityId = struct {
    index: u16,
    generation: u16,

    pub fn invalid() EntityId {
        return .{ .index = std.math.maxInt(u16), .generation = 0 };
    }

    pub fn isValid(self: EntityId) bool {
        return self.index != std.math.maxInt(u16);
    }
};

const Transform = struct {
    x: f32, y: f32, z: f32,
    rx: f32, ry: f32, rz: f32,
    sx: f32, sy: f32, sz: f32,

    pub fn identity() Transform {
        return .{
            .x = 0, .y = 0, .z = 0,
            .rx = 0, .ry = 0, .rz = 0,
            .sx = 1, .sy = 1, .sz = 1,
        };
    }
};

const Health = struct {
    current: f32,
    maximum: f32,
    regen_rate: f32,

    pub fn init(max: f32) Health {
        return .{ .current = max, .maximum = max, .regen_rate = 1.0 };
    }

    pub fn isDead(self: *const Health) bool {
        return self.current <= 0;
    }

    pub fn damage(self: *Health, amount: f32) void {
        self.current = @max(0, self.current - amount);
    }

    pub fn heal(self: *Health, amount: f32) void {
        self.current = @min(self.maximum, self.current + amount);
    }
};

const Velocity = struct {
    x: f32, y: f32, z: f32,

    pub fn zero() Velocity {
        return .{ .x = 0, .y = 0, .z = 0 };
    }
};

const MAX_ENTITIES: usize = 1000;

const EntityWorld = struct {
    // Component pools
    transforms: [MAX_ENTITIES]Transform,
    healths: [MAX_ENTITIES]Health,
    velocities: [MAX_ENTITIES]Velocity,

    // Entity management
    generations: [MAX_ENTITIES]u16,
    alive: [MAX_ENTITIES]bool,
    has_transform: [MAX_ENTITIES]bool,
    has_health: [MAX_ENTITIES]bool,
    has_velocity: [MAX_ENTITIES]bool,

    free_list: std.ArrayList(u16),
    entity_count: usize,

    pub fn init(allocator: std.mem.Allocator) EntityWorld {
        var world = EntityWorld{
            .transforms = undefined,
            .healths = undefined,
            .velocities = undefined,
            .generations = [_]u16{0} ** MAX_ENTITIES,
            .alive = [_]bool{false} ** MAX_ENTITIES,
            .has_transform = [_]bool{false} ** MAX_ENTITIES,
            .has_health = [_]bool{false} ** MAX_ENTITIES,
            .has_velocity = [_]bool{false} ** MAX_ENTITIES,
            .free_list = std.ArrayList(u16).init(allocator),
            .entity_count = 0,
        };
        // Pre-populate free list
        var i: u16 = MAX_ENTITIES;
        while (i > 0) {
            i -= 1;
            world.free_list.append(i) catch unreachable;
        }
        return world;
    }

    pub fn deinit(self: *EntityWorld) void {
        self.free_list.deinit();
    }

    pub fn createEntity(self: *EntityWorld) ?EntityId {
        const index = self.free_list.popOrNull() orelse return null;
        self.alive[index] = true;
        self.entity_count += 1;
        return EntityId{
            .index = index,
            .generation = self.generations[index],
        };
    }

    pub fn destroyEntity(self: *EntityWorld, id: EntityId) void {
        if (!self.isAlive(id)) return;
        self.alive[id.index] = false;
        self.has_transform[id.index] = false;
        self.has_health[id.index] = false;
        self.has_velocity[id.index] = false;
        self.generations[id.index] +%= 1;
        self.free_list.append(id.index) catch unreachable;
        self.entity_count -= 1;
    }

    pub fn isAlive(self: *const EntityWorld, id: EntityId) bool {
        if (!id.isValid()) return false;
        return self.alive[id.index] and
               self.generations[id.index] == id.generation;
    }

    pub fn addTransform(self: *EntityWorld, id: EntityId, t: Transform) void {
        self.transforms[id.index] = t;
        self.has_transform[id.index] = true;
    }

    pub fn addHealth(self: *EntityWorld, id: EntityId, h: Health) void {
        self.healths[id.index] = h;
        self.has_health[id.index] = true;
    }

    pub fn addVelocity(self: *EntityWorld, id: EntityId, v: Velocity) void {
        self.velocities[id.index] = v;
        self.has_velocity[id.index] = true;
    }

    // System: update positions
    pub fn physicsSystem(self: *EntityWorld, dt: f32) void {
        for (0..MAX_ENTITIES) |i| {
            if (!self.alive[i]) continue;
            if (!self.has_transform[i] or !self.has_velocity[i]) continue;

            self.transforms[i].x += self.velocities[i].x * dt;
            self.transforms[i].y += self.velocities[i].y * dt;
            self.transforms[i].z += self.velocities[i].z * dt;
        }
    }

    // System: destroy dead entities
    pub fn cleanupSystem(self: *EntityWorld, ids: []const EntityId) void {
        for (ids) |id| {
            if (!self.isAlive(id)) continue;
            if (!self.has_health[id.index]) continue;
            if (self.healths[id.index].isDead()) {
                std.debug.print("Entity {} died\n", .{id.index});
                self.destroyEntity(id);
            }
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var world = EntityWorld.init(gpa.allocator());
    defer world.deinit();

    // Create entities
    var entity_ids = std.ArrayList(EntityId).init(gpa.allocator());
    defer entity_ids.deinit();

    for (0..10) |i| {
        const id = world.createEntity() orelse continue;
        world.addTransform(id, Transform.identity());
        world.addHealth(id, Health.init(100.0));
        world.addVelocity(id, Velocity{
            .x = @as(f32, @floatFromInt(i)) * 0.1,
            .y = 0,
            .z = 0,
        });
        try entity_ids.append(id);
    }

    std.debug.print("Created {} entities\n", .{world.entity_count});

    // Simulate
    const dt: f32 = 1.0 / 60.0;
    for (0..60) |frame| {
        world.physicsSystem(dt);

        // Damage some entities
        if (frame % 10 == 0) {
            for (entity_ids.items) |id| {
                if (world.isAlive(id) and world.has_health[id.index]) {
                    world.healths[id.index].damage(30.0);
                }
            }
        }

        world.cleanupSystem(entity_ids.items);
    }

    std.debug.print("After simulation: {} entities alive\n", .{world.entity_count});
}
```

---

## สรุป Part 74

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|--------------|
| 731 | Concept | Pool แก้ปัญหา allocation overhead |
| 732 | Fixed Pool | Implementation พื้นฐาน |
| 733 | Dynamic Pool | Pool ที่ขยายตัวได้ |
| 734 | Type-Safe | Handle-based safety |
| 735 | Initialization | Constructor/Destructor ใน pool |
| 736 | Statistics | Monitoring pool performance |
| 737 | Thread-Safe | Concurrent access |
| 738 | Fragmentation | ตรวจสอบและแก้ไข fragmentation |
| 739 | Comparison | Pool vs Allocator benchmark |
| 740 | Real-World | ECS game entity system |

### เมื่อไหรควรใช้ Object Pool

**ใช้ Pool เมื่อ:**
- สร้าง/ทำลาย objects บ่อยมาก (particles, bullets, connections)
- Objects มีขนาดคงที่และเท่ากัน
- ต้องการ predictable performance (games, real-time)
- ต้องการหลีกเลี่ยง heap fragmentation

**ไม่ควรใช้ Pool เมื่อ:**
- Objects มีขนาดแตกต่างกัน
- จำนวน objects ไม่แน่นอนและอาจมากมาย
- Objects มี lifetime ที่ซับซ้อน

---

*ต่อไป: Part 75 - Reference Counting ใน Zig*
