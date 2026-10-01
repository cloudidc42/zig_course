# ส่วนที่ 71: Advanced Memory Patterns ใน Zig

## ภาพรวม

ในส่วนนี้เราจะศึกษา patterns การจัดการหน่วยความจำขั้นสูงใน Zig ซึ่งเป็นพื้นฐานสำคัญสำหรับการเขียนโปรแกรมที่มีประสิทธิภาพและปลอดภัย Zig ให้ control ระดับต่ำเหมือน C แต่มี safety features ที่ช่วยป้องกันข้อผิดพลาดที่พบบ่อย

---

## Step 701: Memory Ownership Patterns

### แนวคิดหลัก

Memory ownership คือการกำหนดว่า "ใคร" เป็นเจ้าของหน่วยความจำและรับผิดชอบในการ free มัน ใน Zig เราต้องจัดการเรื่องนี้ด้วยตนเอง แต่มี patterns ที่ช่วยให้ทำได้อย่างถูกต้อง

### Single Owner Pattern

```zig
const std = @import("std");

// Pattern: Single Owner - มีเจ้าของเพียงคนเดียว
const Buffer = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, size: usize) !Buffer {
        const data = try allocator.alloc(u8, size);
        return Buffer{
            .data = data,
            .allocator = allocator,
        };
    }

    // เจ้าของรับผิดชอบ free
    pub fn deinit(self: Buffer) void {
        self.allocator.free(self.data);
    }

    pub fn write(self: *Buffer, offset: usize, bytes: []const u8) void {
        @memcpy(self.data[offset..offset + bytes.len], bytes);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Buffer เป็นเจ้าของ data
    var buf = try Buffer.init(allocator, 1024);
    defer buf.deinit(); // รับผิดชอบ free ตอน exit

    buf.write(0, "Hello, Zig!");
    std.debug.print("Buffer: {s}\n", .{buf.data[0..11]});
}
```

### Transfer of Ownership Pattern

```zig
const std = @import("std");

const OwnedSlice = struct {
    items: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, data: []const u8) !OwnedSlice {
        const copy = try allocator.dupe(u8, data);
        return OwnedSlice{ .items = copy, .allocator = allocator };
    }

    // โอน ownership ให้ผู้รับ - ผู้รับต้อง free เอง
    pub fn transfer(self: *OwnedSlice) []u8 {
        const items = self.items;
        self.items = &[_]u8{}; // clear pointer เพื่อป้องกัน double-free
        return items;
    }

    pub fn deinit(self: OwnedSlice) void {
        if (self.items.len > 0) {
            self.allocator.free(self.items);
        }
    }
};

pub fn processData(allocator: std.mem.Allocator) ![]u8 {
    var owned = try OwnedSlice.init(allocator, "Important data");
    defer owned.deinit(); // จะ free ถ้า transfer ไม่ถูกเรียก

    // ... process ...

    return owned.transfer(); // โอน ownership ให้ caller
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const data = try processData(allocator);
    defer allocator.free(data); // ตอนนี้เราเป็นเจ้าของ

    std.debug.print("Got: {s}\n", .{data});
}
```

### Shared Ownership ด้วย Reference Counting

```zig
const std = @import("std");
const Atomic = std.atomic.Value;

fn SharedData(comptime T: type) type {
    return struct {
        const Self = @This();

        value: T,
        ref_count: Atomic(u32),
        allocator: std.mem.Allocator,

        pub fn create(allocator: std.mem.Allocator, value: T) !*Self {
            const self = try allocator.create(Self);
            self.* = .{
                .value = value,
                .ref_count = Atomic(u32).init(1),
                .allocator = allocator,
            };
            return self;
        }

        pub fn retain(self: *Self) *Self {
            _ = self.ref_count.fetchAdd(1, .monotonic);
            return self;
        }

        pub fn release(self: *Self) void {
            if (self.ref_count.fetchSub(1, .release) == 1) {
                std.atomic.fence(.acquire);
                self.allocator.destroy(self);
            }
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const shared = try SharedData(i32).create(allocator, 42);
    const ref1 = shared.retain();
    const ref2 = shared.retain();

    std.debug.print("Value: {}\n", .{shared.value});
    std.debug.print("Ref count: {}\n", .{shared.ref_count.load(.monotonic)});

    shared.release();
    ref1.release();
    ref2.release(); // free ตรงนี้
}
```

---

## Step 702: Move Semantics ใน Zig

### แนวคิด Move Semantics

ใน Zig ไม่มี move semantics แบบ Rust แต่เราสามารถจำลองได้ด้วย patterns ต่างๆ

```zig
const std = @import("std");

// Moved type - เมื่อ "move" แล้ว object เดิมใช้ไม่ได้
const UniqueResource = struct {
    handle: ?*anyopaque,
    allocator: std.mem.Allocator,

    pub fn create(allocator: std.mem.Allocator) !UniqueResource {
        const handle = try allocator.create(u64);
        handle.* = 0xDEADBEEF;
        return UniqueResource{
            .handle = @ptrCast(handle),
            .allocator = allocator,
        };
    }

    // Move: โอน ownership และ invalidate ต้นทาง
    pub fn move(self: *UniqueResource) UniqueResource {
        const result = UniqueResource{
            .handle = self.handle,
            .allocator = self.allocator,
        };
        self.handle = null; // Invalidate
        return result;
    }

    pub fn isValid(self: UniqueResource) bool {
        return self.handle != null;
    }

    pub fn deinit(self: UniqueResource) void {
        if (self.handle) |handle| {
            const ptr: *u64 = @ptrCast(@alignCast(handle));
            self.allocator.destroy(ptr);
        }
    }
};

pub fn takeOwnership(res: UniqueResource) void {
    defer res.deinit();
    const ptr: *u64 = @ptrCast(@alignCast(res.handle.?));
    std.debug.print("Resource value: 0x{X}\n", .{ptr.*});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var res = try UniqueResource.create(allocator);
    std.debug.print("Original valid: {}\n", .{res.isValid()});

    const moved = res.move();
    std.debug.print("After move, original valid: {}\n", .{res.isValid()});
    std.debug.print("Moved valid: {}\n", .{moved.isValid()});

    takeOwnership(moved); // โอนให้ function นี้จัดการ
}
```

### Swap Pattern

```zig
const std = @import("std");

fn swap(comptime T: type, a: *T, b: *T) void {
    const tmp = a.*;
    a.* = b.*;
    b.* = tmp;
}

const Container = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, size: usize) !Container {
        return Container{
            .data = try allocator.alloc(u8, size),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: Container) void {
        self.allocator.free(self.data);
    }

    // Swap ใช้แทน move ในบางกรณี
    pub fn swapContents(self: *Container, other: *Container) void {
        swap([]u8, &self.data, &other.data);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var a = try Container.init(allocator, 8);
    defer a.deinit();
    var b = try Container.init(allocator, 16);
    defer b.deinit();

    std.debug.print("Before swap: a.len={}, b.len={}\n", .{ a.data.len, b.data.len });
    a.swapContents(&b);
    std.debug.print("After swap: a.len={}, b.len={}\n", .{ a.data.len, b.data.len });
}
```

---

## Step 703: Borrowing Patterns (Const References)

### แนวคิด Borrowing

Borrowing คือการ "ยืม" access ชั่วคราวโดยไม่รับ ownership

```zig
const std = @import("std");

const Database = struct {
    records: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Database {
        return Database{
            .records = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Database) void {
        for (self.records.items) |record| {
            self.allocator.free(record);
        }
        self.records.deinit();
    }

    pub fn addRecord(self: *Database, data: []const u8) !void {
        const owned = try self.allocator.dupe(u8, data);
        try self.records.append(owned);
    }

    // Immutable borrow - ยืมดู ไม่แก้ไข
    pub fn getRecord(self: *const Database, index: usize) ?[]const u8 {
        if (index >= self.records.items.len) return null;
        return self.records.items[index];
    }

    // Mutable borrow - ยืมแก้ไข (จำกัดการใช้งาน)
    pub fn updateRecord(self: *Database, index: usize, new_data: []const u8) !void {
        if (index >= self.records.items.len) return error.IndexOutOfBounds;
        self.allocator.free(self.records.items[index]);
        self.records.items[index] = try self.allocator.dupe(u8, new_data);
    }
};

// Borrowed view - ไม่ own data
const DatabaseView = struct {
    db: *const Database,
    prefix_filter: []const u8,

    pub fn init(db: *const Database, filter: []const u8) DatabaseView {
        return DatabaseView{ .db = db, .prefix_filter = filter };
    }

    pub fn iterate(self: DatabaseView, callback: fn ([]const u8) void) void {
        for (self.db.records.items) |record| {
            if (std.mem.startsWith(u8, record, self.prefix_filter)) {
                callback(record);
            }
        }
    }
};

fn printRecord(record: []const u8) void {
    std.debug.print("  Record: {s}\n", .{record});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var db = Database.init(allocator);
    defer db.deinit();

    try db.addRecord("user:alice");
    try db.addRecord("user:bob");
    try db.addRecord("admin:charlie");
    try db.addRecord("user:dave");

    // Borrowed view - ไม่ต้อง free
    const user_view = DatabaseView.init(&db, "user:");
    std.debug.print("Users:\n", .{});
    user_view.iterate(printRecord);

    const admin_view = DatabaseView.init(&db, "admin:");
    std.debug.print("Admins:\n", .{});
    admin_view.iterate(printRecord);
}
```

---

## Step 704: RAII Pattern Implementation

### Resource Acquisition Is Initialization

```zig
const std = @import("std");

// RAII File Handle
const FileHandle = struct {
    file: std.fs.File,
    path: []const u8,
    allocator: std.mem.Allocator,

    pub fn open(allocator: std.mem.Allocator, path: []const u8) !FileHandle {
        const owned_path = try allocator.dupe(u8, path);
        errdefer allocator.free(owned_path);

        const file = try std.fs.cwd().openFile(path, .{ .mode = .read_write });
        return FileHandle{
            .file = file,
            .path = owned_path,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: FileHandle) void {
        self.file.close();
        self.allocator.free(self.path);
        std.debug.print("Closed file: {s}\n", .{self.path});
    }

    pub fn read(self: FileHandle, buf: []u8) !usize {
        return self.file.read(buf);
    }

    pub fn write(self: FileHandle, data: []const u8) !void {
        try self.file.writeAll(data);
    }
};

// RAII Lock
const Mutex = struct {
    locked: bool,

    pub fn init() Mutex {
        return Mutex{ .locked = false };
    }

    pub fn lock(self: *Mutex) MutexGuard {
        self.locked = true;
        std.debug.print("Lock acquired\n", .{});
        return MutexGuard{ .mutex = self };
    }
};

const MutexGuard = struct {
    mutex: *Mutex,

    pub fn deinit(self: MutexGuard) void {
        self.mutex.locked = false;
        std.debug.print("Lock released\n", .{});
    }
};

// RAII Transaction
const Transaction = struct {
    committed: bool,
    operations: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,

    pub fn begin(allocator: std.mem.Allocator) Transaction {
        std.debug.print("Transaction started\n", .{});
        return Transaction{
            .committed = false,
            .operations = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn addOperation(self: *Transaction, op: []const u8) !void {
        try self.operations.append(op);
    }

    pub fn commit(self: *Transaction) void {
        self.committed = true;
        std.debug.print("Transaction committed with {} operations\n", .{self.operations.items.len});
    }

    pub fn deinit(self: *Transaction) void {
        if (!self.committed) {
            std.debug.print("Transaction rolled back!\n", .{});
        }
        self.operations.deinit();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Test Transaction RAII
    {
        var tx = Transaction.begin(allocator);
        defer tx.deinit(); // Rollback ถ้า commit ไม่ถูกเรียก

        try tx.addOperation("INSERT user");
        try tx.addOperation("UPDATE balance");
        tx.commit();
    }

    {
        var tx = Transaction.begin(allocator);
        defer tx.deinit(); // จะ rollback เพราะไม่ commit

        try tx.addOperation("DELETE everything");
        // ไม่ commit -> rollback อัตโนมัติ
    }

    // Test Mutex RAII
    var mutex = Mutex.init();
    {
        const guard = mutex.lock();
        defer guard.deinit();
        std.debug.print("Critical section\n", .{});
    }
    std.debug.print("Mutex locked: {}\n", .{mutex.locked});

    _ = allocator;
}
```

---

## Step 705: Smart Pointer Patterns

### Box(T) - Heap Allocation Wrapper

```zig
const std = @import("std");

fn Box(comptime T: type) type {
    return struct {
        const Self = @This();

        ptr: *T,
        allocator: std.mem.Allocator,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const ptr = try allocator.create(T);
            ptr.* = value;
            return Self{ .ptr = ptr, .allocator = allocator };
        }

        pub fn deinit(self: Self) void {
            self.allocator.destroy(self.ptr);
        }

        pub fn get(self: Self) *T {
            return self.ptr;
        }

        pub fn getConst(self: Self) *const T {
            return self.ptr;
        }

        pub fn unbox(self: Self) T {
            const value = self.ptr.*;
            self.allocator.destroy(self.ptr);
            return value;
        }
    };
}

// Nullable smart pointer
fn Maybe(comptime T: type) type {
    return struct {
        const Self = @This();

        value: ?*T,
        allocator: std.mem.Allocator,

        pub fn some(allocator: std.mem.Allocator, val: T) !Self {
            const ptr = try allocator.create(T);
            ptr.* = val;
            return Self{ .value = ptr, .allocator = allocator };
        }

        pub fn none(allocator: std.mem.Allocator) Self {
            return Self{ .value = null, .allocator = allocator };
        }

        pub fn deinit(self: Self) void {
            if (self.value) |ptr| {
                self.allocator.destroy(ptr);
            }
        }

        pub fn isPresent(self: Self) bool {
            return self.value != null;
        }

        pub fn get(self: Self) ?*T {
            return self.value;
        }

        pub fn getOrDefault(self: Self, default: T) T {
            if (self.value) |ptr| return ptr.*;
            return default;
        }

        pub fn map(self: Self, comptime R: type, f: fn (T) R) Maybe(R) {
            // simplified version
            _ = self;
            _ = f;
            return Maybe(R).none(self.allocator);
        }
    };
}

const Node = struct {
    value: i32,
    next: ?Box(Node),

    pub fn deinit(self: Node) void {
        if (self.next) |next| {
            next.get().deinit();
            next.deinit();
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Box usage
    const boxed = try Box(i32).create(allocator, 42);
    defer boxed.deinit();
    std.debug.print("Boxed value: {}\n", .{boxed.get().*});

    // Maybe usage
    const some_val = try Maybe(f64).some(allocator, 3.14);
    defer some_val.deinit();
    std.debug.print("Maybe value: {?}\n", .{if (some_val.get()) |p| p.* else null});

    const no_val = Maybe(f64).none(allocator);
    defer no_val.deinit();
    std.debug.print("Default: {}\n", .{no_val.getOrDefault(0.0)});

    // Unbox
    const boxed2 = try Box(i32).create(allocator, 100);
    const unboxed = boxed2.unbox(); // ptr freed ใน unbox
    std.debug.print("Unboxed: {}\n", .{unboxed});
}
```

---

## Step 706: Weak References

### แนวคิด Weak Reference

Weak reference คือ reference ที่ไม่นับ reference count และอาจกลายเป็น invalid ได้

```zig
const std = @import("std");
const Atomic = std.atomic.Value;

fn StrongRef(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = struct {
            value: T,
            strong_count: Atomic(u32),
            weak_count: Atomic(u32),
            allocator: std.mem.Allocator,
        };

        inner: *Inner,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = .{
                .value = value,
                .strong_count = Atomic(u32).init(1),
                .weak_count = Atomic(u32).init(1), // หนึ่งสำหรับ strong refs ทั้งหมด
                .allocator = allocator,
            };
            return Self{ .inner = inner };
        }

        pub fn clone(self: Self) Self {
            _ = self.inner.strong_count.fetchAdd(1, .monotonic);
            return Self{ .inner = self.inner };
        }

        pub fn downgrade(self: Self) WeakRef(T) {
            _ = self.inner.weak_count.fetchAdd(1, .monotonic);
            return WeakRef(T){ .inner = self.inner };
        }

        pub fn deinit(self: Self) void {
            if (self.inner.strong_count.fetchSub(1, .release) == 1) {
                std.atomic.fence(.acquire);
                // Drop the value
                std.debug.print("Value dropped\n", .{});
                // Decrement weak count (สำหรับ strong refs ทั้งหมด)
                if (self.inner.weak_count.fetchSub(1, .release) == 1) {
                    std.atomic.fence(.acquire);
                    self.inner.allocator.destroy(self.inner);
                }
            }
        }

        pub fn get(self: Self) *const T {
            return &self.inner.value;
        }
    };
}

fn WeakRef(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = StrongRef(T).Inner;

        inner: *Inner,

        // upgrade จาก weak เป็น strong (อาจ fail ถ้า value ถูก drop แล้ว)
        pub fn upgrade(self: Self) ?StrongRef(T) {
            var count = self.inner.strong_count.load(.relaxed);
            while (count > 0) {
                if (self.inner.strong_count.cmpxchgWeak(
                    count,
                    count + 1,
                    .acquire,
                    .relaxed,
                )) |_| {
                    count = self.inner.strong_count.load(.relaxed);
                } else {
                    return StrongRef(T){ .inner = self.inner };
                }
            }
            return null; // Value ถูก drop แล้ว
        }

        pub fn deinit(self: Self) void {
            if (self.inner.weak_count.fetchSub(1, .release) == 1) {
                std.atomic.fence(.acquire);
                self.inner.allocator.destroy(self.inner);
            }
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const strong1 = try StrongRef(i32).create(allocator, 42);
    const weak = strong1.downgrade();

    // ตอนนี้ยัง upgrade ได้
    if (weak.upgrade()) |upgraded| {
        std.debug.print("Upgraded: {}\n", .{upgraded.get().*});
        upgraded.deinit();
    }

    strong1.deinit(); // Drop value

    // หลัง drop ไม่สามารถ upgrade ได้ (ใน implementation จริง)
    std.debug.print("Strong ref released\n", .{});
    weak.deinit();
}
```

---

## Step 707: Memory Pools

### Fixed-Size Memory Pool

```zig
const std = @import("std");

fn MemoryPool(comptime T: type, comptime capacity: usize) type {
    return struct {
        const Self = @This();

        storage: [capacity]T align(@alignOf(T)),
        used: [capacity]bool,
        free_count: usize,

        pub fn init() Self {
            return Self{
                .storage = undefined,
                .used = [_]bool{false} ** capacity,
                .free_count = capacity,
            };
        }

        pub fn alloc(self: *Self) ?*T {
            for (&self.used, 0..) |*slot, i| {
                if (!slot.*) {
                    slot.* = true;
                    self.free_count -= 1;
                    return &self.storage[i];
                }
            }
            return null;
        }

        pub fn free(self: *Self, ptr: *T) void {
            const base = @intFromPtr(&self.storage[0]);
            const addr = @intFromPtr(ptr);
            const index = (addr - base) / @sizeOf(T);
            std.debug.assert(index < capacity);
            std.debug.assert(self.used[index]);
            self.used[index] = false;
            self.free_count += 1;
        }

        pub fn freeCount(self: *const Self) usize {
            return self.free_count;
        }

        pub fn usedCount(self: *const Self) usize {
            return capacity - self.free_count;
        }
    };
}

const Particle = struct {
    x: f32,
    y: f32,
    vx: f32,
    vy: f32,
    lifetime: f32,

    pub fn init(x: f32, y: f32) Particle {
        return Particle{
            .x = x, .y = y,
            .vx = 0, .vy = 0,
            .lifetime = 1.0,
        };
    }

    pub fn update(self: *Particle, dt: f32) void {
        self.x += self.vx * dt;
        self.y += self.vy * dt;
        self.lifetime -= dt;
    }

    pub fn isAlive(self: *const Particle) bool {
        return self.lifetime > 0;
    }
};

pub fn main() void {
    var pool = MemoryPool(Particle, 100).init();

    std.debug.print("Initial free: {}\n", .{pool.freeCount()});

    // Allocate particles
    var particles = std.ArrayList(*Particle).init(std.heap.page_allocator);
    defer particles.deinit();

    var i: usize = 0;
    while (i < 5) : (i += 1) {
        if (pool.alloc()) |particle| {
            particle.* = Particle.init(
                @as(f32, @floatFromInt(i)) * 10.0,
                0.0,
            );
            particles.append(particle) catch unreachable;
        }
    }

    std.debug.print("After alloc: free={}, used={}\n", .{
        pool.freeCount(), pool.usedCount()
    });

    // Simulate and free dead particles
    for (particles.items) |p| {
        p.lifetime = -1.0; // Mark as dead
    }

    for (particles.items) |p| {
        if (!p.isAlive()) {
            pool.free(p);
        }
    }

    std.debug.print("After free: free={}, used={}\n", .{
        pool.freeCount(), pool.usedCount()
    });
}
```

---

## Step 708: Slab Allocator Pattern

### Slab Allocator สำหรับ Objects ขนาดเดียวกัน

```zig
const std = @import("std");

const SlabConfig = struct {
    object_size: usize,
    objects_per_slab: usize,
};

const Slab = struct {
    memory: []u8,
    free_list: ?*FreeNode,
    used_count: usize,
    total_count: usize,
    allocator: std.mem.Allocator,

    const FreeNode = struct {
        next: ?*FreeNode,
    };

    pub fn init(
        allocator: std.mem.Allocator,
        object_size: usize,
        objects_per_slab: usize,
    ) !Slab {
        const aligned_size = std.mem.alignForward(usize, object_size, @alignOf(FreeNode));
        const total_size = aligned_size * objects_per_slab;
        const memory = try allocator.alignedAlloc(u8, @alignOf(FreeNode), total_size);

        var slab = Slab{
            .memory = memory,
            .free_list = null,
            .used_count = 0,
            .total_count = objects_per_slab,
            .allocator = allocator,
        };

        // สร้าง free list
        var offset = total_size;
        while (offset >= aligned_size) {
            offset -= aligned_size;
            const node: *FreeNode = @ptrCast(@alignCast(&memory[offset]));
            node.next = slab.free_list;
            slab.free_list = node;
        }

        return slab;
    }

    pub fn deinit(self: *Slab) void {
        self.allocator.free(self.memory);
    }

    pub fn alloc(self: *Slab) ?[*]u8 {
        if (self.free_list) |node| {
            self.free_list = node.next;
            self.used_count += 1;
            return @ptrCast(node);
        }
        return null;
    }

    pub fn free(self: *Slab, ptr: [*]u8) void {
        const node: *FreeNode = @ptrCast(@alignCast(ptr));
        node.next = self.free_list;
        self.free_list = node;
        self.used_count -= 1;
    }

    pub fn isFull(self: *const Slab) bool {
        return self.used_count == self.total_count;
    }

    pub fn isEmpty(self: *const Slab) bool {
        return self.used_count == 0;
    }
};

const SlabAllocator = struct {
    slabs: std.ArrayList(Slab),
    object_size: usize,
    objects_per_slab: usize,
    backing_allocator: std.mem.Allocator,

    pub fn init(
        allocator: std.mem.Allocator,
        object_size: usize,
        objects_per_slab: usize,
    ) SlabAllocator {
        return SlabAllocator{
            .slabs = std.ArrayList(Slab).init(allocator),
            .object_size = object_size,
            .objects_per_slab = objects_per_slab,
            .backing_allocator = allocator,
        };
    }

    pub fn deinit(self: *SlabAllocator) void {
        for (self.slabs.items) |*slab| {
            slab.deinit();
        }
        self.slabs.deinit();
    }

    pub fn alloc(self: *SlabAllocator) !*anyopaque {
        // หา slab ที่มีที่ว่าง
        for (self.slabs.items) |*slab| {
            if (!slab.isFull()) {
                if (slab.alloc()) |ptr| {
                    return ptr;
                }
            }
        }

        // สร้าง slab ใหม่
        const new_slab = try Slab.init(
            self.backing_allocator,
            self.object_size,
            self.objects_per_slab,
        );
        try self.slabs.append(new_slab);

        const slab = &self.slabs.items[self.slabs.items.len - 1];
        return slab.alloc().?;
    }

    pub fn free(self: *SlabAllocator, ptr: *anyopaque) void {
        const addr = @intFromPtr(ptr);
        for (self.slabs.items) |*slab| {
            const base = @intFromPtr(slab.memory.ptr);
            const end = base + slab.memory.len;
            if (addr >= base and addr < end) {
                slab.free(@ptrCast(ptr));
                return;
            }
        }
        unreachable; // ptr ไม่ได้มาจาก allocator นี้
    }

    pub fn stats(self: *const SlabAllocator) void {
        var total_used: usize = 0;
        var total_capacity: usize = 0;
        for (self.slabs.items) |*slab| {
            total_used += slab.used_count;
            total_capacity += slab.total_count;
        }
        std.debug.print("Slabs: {}, Used: {}/{}\n", .{
            self.slabs.items.len, total_used, total_capacity
        });
    }
};

const SmallObject = struct {
    id: u32,
    data: [32]u8,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var slab_alloc = SlabAllocator.init(allocator, @sizeOf(SmallObject), 10);
    defer slab_alloc.deinit();

    var ptrs: [25]*SmallObject = undefined;
    for (&ptrs, 0..) |*p, i| {
        const raw = try slab_alloc.alloc();
        p.* = @ptrCast(@alignCast(raw));
        p.*.id = @intCast(i);
    }

    slab_alloc.stats();

    // Free บางส่วน
    for (ptrs[0..10]) |p| {
        slab_alloc.free(p);
    }

    slab_alloc.stats();
}
```

---

## Step 709: Memory Budget Management

### Memory Budget System

```zig
const std = @import("std");

const MemoryBudget = struct {
    total_budget: usize,
    current_usage: usize,
    peak_usage: usize,
    allocation_count: usize,
    deallocation_count: usize,
    backing_allocator: std.mem.Allocator,

    const Self = @This();

    pub fn init(backing: std.mem.Allocator, budget: usize) Self {
        return Self{
            .total_budget = budget,
            .current_usage = 0,
            .peak_usage = 0,
            .allocation_count = 0,
            .deallocation_count = 0,
            .backing_allocator = backing,
        };
    }

    fn allocFn(ctx: *anyopaque, n: usize, log2_ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *Self = @ptrCast(@alignCast(ctx));
        if (self.current_usage + n > self.total_budget) {
            std.debug.print("BUDGET EXCEEDED: need {} but only {} remaining\n", .{
                n, self.total_budget - self.current_usage
            });
            return null;
        }
        const ptr = self.backing_allocator.rawAlloc(n, log2_ptr_align, ret_addr) orelse return null;
        self.current_usage += n;
        if (self.current_usage > self.peak_usage) {
            self.peak_usage = self.current_usage;
        }
        self.allocation_count += 1;
        return ptr;
    }

    fn resizeFn(ctx: *anyopaque, buf: []u8, log2_buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *Self = @ptrCast(@alignCast(ctx));
        if (new_len > buf.len) {
            const extra = new_len - buf.len;
            if (self.current_usage + extra > self.total_budget) return false;
        }
        if (!self.backing_allocator.rawResize(buf, log2_buf_align, new_len, ret_addr)) return false;
        if (new_len > buf.len) {
            self.current_usage += new_len - buf.len;
            if (self.current_usage > self.peak_usage) self.peak_usage = self.current_usage;
        } else {
            self.current_usage -= buf.len - new_len;
        }
        return true;
    }

    fn freeFn(ctx: *anyopaque, buf: []u8, log2_buf_align: u8, ret_addr: usize) void {
        const self: *Self = @ptrCast(@alignCast(ctx));
        self.current_usage -= buf.len;
        self.deallocation_count += 1;
        self.backing_allocator.rawFree(buf, log2_buf_align, ret_addr);
    }

    pub fn allocator(self: *Self) std.mem.Allocator {
        return std.mem.Allocator{
            .ptr = self,
            .vtable = &.{
                .alloc = allocFn,
                .resize = resizeFn,
                .free = freeFn,
            },
        };
    }

    pub fn printStats(self: *const Self) void {
        std.debug.print(
            \\Memory Budget Stats:
            \\  Budget: {} bytes
            \\  Current: {} bytes ({d:.1}%)
            \\  Peak: {} bytes ({d:.1}%)
            \\  Allocations: {}
            \\  Deallocations: {}
            \\
        , .{
            self.total_budget,
            self.current_usage,
            @as(f64, @floatFromInt(self.current_usage)) / @as(f64, @floatFromInt(self.total_budget)) * 100.0,
            self.peak_usage,
            @as(f64, @floatFromInt(self.peak_usage)) / @as(f64, @floatFromInt(self.total_budget)) * 100.0,
            self.allocation_count,
            self.deallocation_count,
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var budget = MemoryBudget.init(gpa.allocator(), 1024 * 1024); // 1MB budget
    const alloc = budget.allocator();

    // Allocate ต่างๆ
    const buf1 = try alloc.alloc(u8, 1024);
    defer alloc.free(buf1);

    const buf2 = try alloc.alloc(u8, 4096);
    defer alloc.free(buf2);

    budget.printStats();

    // ลอง exceed budget
    const oversized = alloc.alloc(u8, 2 * 1024 * 1024) catch |err| {
        std.debug.print("Expected error: {}\n", .{err});
        null
    };
    if (oversized) |buf| {
        alloc.free(buf);
    }

    budget.printStats();
}
```

---

## Step 710: Zero-Copy Patterns

### Zero-Copy Buffer

```zig
const std = @import("std");

// Scatter-Gather I/O Pattern
const IoVec = struct {
    base: []const u8,
};

const ScatterGather = struct {
    vecs: std.ArrayList(IoVec),

    pub fn init(allocator: std.mem.Allocator) ScatterGather {
        return ScatterGather{ .vecs = std.ArrayList(IoVec).init(allocator) };
    }

    pub fn deinit(self: *ScatterGather) void {
        self.vecs.deinit();
    }

    // Zero-copy: เพิ่ม reference โดยไม่ copy
    pub fn addChunk(self: *ScatterGather, chunk: []const u8) !void {
        try self.vecs.append(IoVec{ .base = chunk });
    }

    pub fn totalLen(self: *const ScatterGather) usize {
        var total: usize = 0;
        for (self.vecs.items) |vec| {
            total += vec.base.len;
        }
        return total;
    }

    // Copy เฉพาะเมื่อจำเป็น
    pub fn materialize(self: *const ScatterGather, allocator: std.mem.Allocator) ![]u8 {
        const total = self.totalLen();
        const buf = try allocator.alloc(u8, total);
        var offset: usize = 0;
        for (self.vecs.items) |vec| {
            @memcpy(buf[offset..offset + vec.base.len], vec.base);
            offset += vec.base.len;
        }
        return buf;
    }
};

// Ring Buffer สำหรับ zero-copy streaming
const RingBuffer = struct {
    buffer: []u8,
    read_pos: usize,
    write_pos: usize,
    len: usize,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, capacity: usize) !RingBuffer {
        return RingBuffer{
            .buffer = try allocator.alloc(u8, capacity),
            .read_pos = 0,
            .write_pos = 0,
            .len = 0,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *RingBuffer) void {
        self.allocator.free(self.buffer);
    }

    pub fn write(self: *RingBuffer, data: []const u8) !usize {
        const available = self.buffer.len - self.len;
        const to_write = @min(data.len, available);

        if (to_write == 0) return 0;

        const first_chunk = @min(to_write, self.buffer.len - self.write_pos);
        @memcpy(self.buffer[self.write_pos..self.write_pos + first_chunk], data[0..first_chunk]);

        if (first_chunk < to_write) {
            const second_chunk = to_write - first_chunk;
            @memcpy(self.buffer[0..second_chunk], data[first_chunk..to_write]);
        }

        self.write_pos = (self.write_pos + to_write) % self.buffer.len;
        self.len += to_write;
        return to_write;
    }

    // Zero-copy read: return slice ที่ชี้ไปที่ buffer โดยตรง
    pub fn readSlice(self: *RingBuffer, max_len: usize) []const u8 {
        const available = @min(self.len, max_len);
        const contiguous = @min(available, self.buffer.len - self.read_pos);
        return self.buffer[self.read_pos..self.read_pos + contiguous];
    }

    pub fn consume(self: *RingBuffer, n: usize) void {
        std.debug.assert(n <= self.len);
        self.read_pos = (self.read_pos + n) % self.buffer.len;
        self.len -= n;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Scatter-Gather
    var sg = ScatterGather.init(allocator);
    defer sg.deinit();

    const header = "HTTP/1.1 200 OK\r\n";
    const body = "Hello, World!";

    try sg.addChunk(header); // ไม่ copy
    try sg.addChunk(body);   // ไม่ copy

    std.debug.print("Total length: {}\n", .{sg.totalLen()});

    const materialized = try sg.materialize(allocator);
    defer allocator.free(materialized);
    std.debug.print("Materialized: {s}\n", .{materialized});

    // Ring Buffer
    var ring = try RingBuffer.init(allocator, 64);
    defer ring.deinit();

    _ = try ring.write("Hello, ");
    _ = try ring.write("World!");

    const data = ring.readSlice(100);
    std.debug.print("Ring buffer data: {s}\n", .{data});
    ring.consume(data.len);

    std.debug.print("After consume, len: {}\n", .{ring.len});
}
```

---

## สรุป Part 71

ใน Part นี้เราได้เรียนรู้ patterns การจัดการหน่วยความจำขั้นสูงใน Zig:

| Step | Pattern | ประโยชน์ |
|------|---------|----------|
| 701 | Memory Ownership | กำหนดความรับผิดชอบชัดเจน |
| 702 | Move Semantics | โอน ownership อย่างปลอดภัย |
| 703 | Borrowing | ใช้ข้อมูลโดยไม่ copy |
| 704 | RAII | cleanup อัตโนมัติ |
| 705 | Smart Pointers | จัดการ lifetime ปลอดภัย |
| 706 | Weak References | อ้างอิงโดยไม่ extend lifetime |
| 707 | Memory Pools | allocation เร็ว ไม่ fragmentation |
| 708 | Slab Allocator | optimize สำหรับ fixed-size objects |
| 709 | Memory Budget | ควบคุมการใช้ memory |
| 710 | Zero-Copy | ลด data copying |

### Tips สำคัญ

1. **ใช้ `defer` เสมอ** เพื่อ cleanup resource
2. **กำหนด ownership ให้ชัดเจน** ใน function signatures
3. **ใช้ memory pool** เมื่อ allocate/free บ่อยๆ สำหรับ objects ขนาดเดียว
4. **ใช้ zero-copy** เมื่อ performance สำคัญ
5. **Track memory budget** ใน production systems

---

*ต่อไป: Part 72 - Custom Allocators ใน Zig*
