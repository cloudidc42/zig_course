# ส่วนที่ 75: Reference Counting ใน Zig

## ภาพรวม

Reference Counting (RC) เป็น pattern การจัดการ memory ที่ track จำนวน references ไปยัง object และ free เมื่อ count เป็น 0 ใช้ใน Python, Swift, Rust (Rc/Arc) และ Objective-C ใน Part นี้เราจะ implement RC ใน Zig

---

## Step 741: Reference Counting Concept

### แนวคิดพื้นฐาน

```zig
const std = @import("std");

// Reference Counting แก้ปัญหา shared ownership
// เมื่อ object ถูก share ระหว่างหลาย owners
// แต่ละ owner "retain" (เพิ่ม count) และ "release" (ลด count)
// เมื่อ count = 0 -> free

// ปัญหาที่ RC แก้ได้:
// 1. Shared ownership โดยไม่รู้ว่าใครจะ free ก่อน
// 2. Callback/closure ที่ hold reference
// 3. Cache ที่แชร์ object ระหว่างหลายที่
// 4. Observer pattern

// ตัวอย่างโดยไม่มี RC (ปัญหา double-free)
const BadExample = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) !BadExample {
        return BadExample{
            .data = try allocator.alloc(u8, 100),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: BadExample) void {
        self.allocator.free(self.data);
    }
};

pub fn badSharing(allocator: std.mem.Allocator) !void {
    var obj = try BadExample.init(allocator);
    // ถ้า pass ไปหลายที่และ deinit สองครั้ง -> double free!
    // obj.deinit();
    // obj.deinit(); // CRASH!
    obj.deinit();
}

// ด้วย RC: แต่ละ holder retain และ release
// -> safe shared ownership
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    try badSharing(alloc);
    std.debug.print("Reference counting basics demonstrated\n", .{});

    std.debug.print(
        \\
        \\RC Lifecycle:
        \\  create()  -> ref_count = 1
        \\  retain()  -> ref_count += 1
        \\  release() -> ref_count -= 1
        \\              if ref_count == 0: destroy
        \\
    , .{});
}
```

### RC vs อื่นๆ

```zig
// Reference Counting vs Other Approaches:
//
// Manual Memory:        RC:
//   single owner          shared ownership
//   fast                  slight overhead (counter ops)
//   error-prone           safe (no leaks if no cycles)
//   explicit free         automatic free
//
// Garbage Collection:   RC:
//   no cycles issue       cycles = leaks (need weak refs)
//   pause time (GC)       no pauses (deterministic)
//   runtime overhead      compile-time knowable
//   opaque                transparent
//
// Arena:                RC:
//   batch free            individual free
//   no cycles             handles cycles with weak
//   reset = free all      precise lifetime
//   very fast alloc       moderate overhead
```

---

## Step 742: Simple Rc(T) Implementation

### Single-threaded Reference Counter

```zig
const std = @import("std");

fn Rc(comptime T: type) type {
    return struct {
        const Self = @This();

        const Inner = struct {
            value: T,
            ref_count: usize,
            allocator: std.mem.Allocator,
        };

        inner: *Inner,

        // สร้าง Rc ใหม่ด้วย ref_count = 1
        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = Inner{
                .value = value,
                .ref_count = 1,
                .allocator = allocator,
            };
            return Self{ .inner = inner };
        }

        // เพิ่ม ref_count (clone)
        pub fn retain(self: Self) Self {
            self.inner.ref_count += 1;
            return Self{ .inner = self.inner };
        }

        // ลด ref_count และ free ถ้า = 0
        pub fn release(self: Self) void {
            std.debug.assert(self.inner.ref_count > 0);
            self.inner.ref_count -= 1;
            if (self.inner.ref_count == 0) {
                // Call deinit ถ้ามี
                if (comptime std.meta.hasFn(T, "deinit")) {
                    self.inner.value.deinit();
                }
                self.inner.allocator.destroy(self.inner);
            }
        }

        // Access value (immutable)
        pub fn get(self: Self) *const T {
            return &self.inner.value;
        }

        // Access value (mutable) - อันตรายถ้า shared!
        pub fn getMut(self: Self) *T {
            std.debug.assert(self.inner.ref_count == 1); // Only safe when unique
            return &self.inner.value;
        }

        // Check uniqueness
        pub fn isUnique(self: Self) bool {
            return self.inner.ref_count == 1;
        }

        // Get ref count
        pub fn refCount(self: Self) usize {
            return self.inner.ref_count;
        }

        // Make unique copy (copy-on-write)
        pub fn makeUnique(self: *Self, allocator: std.mem.Allocator) !void {
            if (self.isUnique()) return;

            const old = self.*;
            self.* = try create(allocator, self.inner.value);
            old.release();
        }
    };
}

const Document = struct {
    content: []u8,
    version: u32,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, content: []const u8) !Document {
        return Document{
            .content = try allocator.dupe(u8, content),
            .version = 1,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Document) void {
        self.allocator.free(self.content);
        std.debug.print("Document freed (version {})\n", .{self.version});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // สร้าง Rc<Document>
    const doc = try Document.init(alloc, "Hello, Reference Counting!");
    var rc1 = try Rc(Document).create(alloc, doc);
    std.debug.print("Created. ref_count={}\n", .{rc1.refCount()});

    // Clone
    var rc2 = rc1.retain();
    std.debug.print("After retain. ref_count={}\n", .{rc1.refCount()});

    // Clone again
    var rc3 = rc1.retain();
    std.debug.print("After retain again. ref_count={}\n", .{rc1.refCount()});

    // Read value
    std.debug.print("Content: {s}\n", .{rc1.get().content});

    // Release
    rc3.release();
    std.debug.print("After release rc3. ref_count={}\n", .{rc1.refCount()});

    rc2.release();
    std.debug.print("After release rc2. ref_count={}\n", .{rc1.refCount()});

    rc1.release(); // Last release - จะ free Document
    std.debug.print("After last release\n", .{});
}
```

---

## Step 743: Thread-Safe Arc(T) Implementation

### Atomic Reference Counter

```zig
const std = @import("std");
const Atomic = std.atomic.Value;

fn Arc(comptime T: type) type {
    return struct {
        const Self = @This();

        const Inner = struct {
            value: T,
            ref_count: Atomic(u32),
            allocator: std.mem.Allocator,
        };

        inner: *Inner,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = Inner{
                .value = value,
                .ref_count = Atomic(u32).init(1),
                .allocator = allocator,
            };
            return Self{ .inner = inner };
        }

        // Thread-safe retain
        pub fn retain(self: Self) Self {
            // Relaxed สำหรับ increment (ไม่จำเป็นต้อง sync)
            const prev = self.inner.ref_count.fetchAdd(1, .monotonic);
            std.debug.assert(prev > 0); // ห้าม retain object ที่ dead
            return Self{ .inner = self.inner };
        }

        // Thread-safe release
        pub fn release(self: Self) void {
            // Release memory ordering สำหรับ decrement
            const prev = self.inner.ref_count.fetchSub(1, .release);
            if (prev == 1) {
                // Acquire fence สำหรับ synchronize กับ release operations ก่อนหน้า
                std.atomic.fence(.acquire);
                // เราเป็น thread สุดท้ายที่ hold reference
                if (comptime std.meta.hasFn(T, "deinit")) {
                    self.inner.value.deinit();
                }
                self.inner.allocator.destroy(self.inner);
            }
        }

        pub fn get(self: Self) *const T {
            return &self.inner.value;
        }

        pub fn refCount(self: Self) u32 {
            return self.inner.ref_count.load(.monotonic);
        }

        pub fn isUnique(self: Self) bool {
            return self.inner.ref_count.load(.acquire) == 1;
        }
    };
}

const SharedCounter = struct {
    value: Atomic(i64),

    pub fn init(v: i64) SharedCounter {
        return SharedCounter{ .value = Atomic(i64).init(v) };
    }

    pub fn increment(self: *SharedCounter) void {
        _ = self.value.fetchAdd(1, .seq_cst);
    }

    pub fn decrement(self: *SharedCounter) void {
        _ = self.value.fetchSub(1, .seq_cst);
    }

    pub fn load(self: *const SharedCounter) i64 {
        return self.value.load(.seq_cst);
    }
};

fn threadWorker(arc: Arc(SharedCounter), iterations: usize) void {
    var local_arc = arc.retain();
    defer local_arc.release();

    var i: usize = 0;
    while (i < iterations) : (i += 1) {
        // Safe เพราะ Arc ป้องกันการ free ขณะใช้งาน
        const counter: *const SharedCounter = local_arc.get();
        // Cannot call increment/decrement ผ่าน const ptr
        // ในกรณีจริงเราต้องใช้ interior mutability pattern
        _ = counter;
    }
    std.debug.print("Thread done, arc ref_count={}\n", .{local_arc.refCount()});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    const counter = SharedCounter.init(0);
    var arc = try Arc(SharedCounter).create(alloc, counter);

    std.debug.print("Initial ref_count={}\n", .{arc.refCount()});

    const thread1 = try std.Thread.spawn(.{}, threadWorker, .{ arc, 1000 });
    const thread2 = try std.Thread.spawn(.{}, threadWorker, .{ arc, 1000 });

    thread1.join();
    thread2.join();

    std.debug.print("After threads joined, ref_count={}\n", .{arc.refCount()});
    arc.release();
    std.debug.print("Arc released\n", .{});
}
```

---

## Step 744: Weak References

### Weak Reference Implementation

```zig
const std = @import("std");
const Atomic = std.atomic.Value;

fn RcInner(comptime T: type) type {
    return struct {
        value: T,
        strong_count: Atomic(u32),
        weak_count: Atomic(u32), // รวม "implicit weak" จาก strong refs
        allocator: std.mem.Allocator,
        is_dropped: bool, // value ถูก drop แล้วหรือยัง
    };
}

fn Weak(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = RcInner(T);
        const Strong = StrongRc(T);

        inner: ?*Inner,

        pub fn upgrade(self: Self) ?Strong {
            const inner = self.inner orelse return null;

            // CAS loop: increment strong count ถ้า > 0
            var count = inner.strong_count.load(.relaxed);
            while (count > 0) {
                if (inner.strong_count.cmpxchgWeak(
                    count, count + 1, .acquire, .relaxed
                )) |actual| {
                    count = actual;
                } else {
                    return Strong{ .inner = inner };
                }
            }
            return null; // Object destroyed
        }

        pub fn deinit(self: Self) void {
            const inner = self.inner orelse return;
            if (inner.weak_count.fetchSub(1, .release) == 1) {
                std.atomic.fence(.acquire);
                inner.allocator.destroy(inner);
            }
        }

        pub fn isAlive(self: Self) bool {
            const inner = self.inner orelse return false;
            return !inner.is_dropped;
        }
    };
}

fn StrongRc(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = RcInner(T);
        const WeakRef = Weak(T);

        inner: *Inner,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = Inner{
                .value = value,
                .strong_count = Atomic(u32).init(1),
                .weak_count = Atomic(u32).init(1), // +1 สำหรับ "strong group"
                .allocator = allocator,
                .is_dropped = false,
            };
            return Self{ .inner = inner };
        }

        pub fn retain(self: Self) Self {
            _ = self.inner.strong_count.fetchAdd(1, .monotonic);
            return Self{ .inner = self.inner };
        }

        pub fn downgrade(self: Self) WeakRef {
            _ = self.inner.weak_count.fetchAdd(1, .monotonic);
            return WeakRef{ .inner = self.inner };
        }

        pub fn release(self: Self) void {
            if (self.inner.strong_count.fetchSub(1, .release) == 1) {
                std.atomic.fence(.acquire);
                self.inner.is_dropped = true;
                // Drop value ถ้ามี deinit
                if (comptime std.meta.hasFn(T, "deinit")) {
                    self.inner.value.deinit();
                }
                // Decrement weak count สำหรับ "strong group"
                if (self.inner.weak_count.fetchSub(1, .release) == 1) {
                    std.atomic.fence(.acquire);
                    self.inner.allocator.destroy(self.inner);
                }
            }
        }

        pub fn get(self: Self) *const T {
            return &self.inner.value;
        }

        pub fn strongCount(self: Self) u32 {
            return self.inner.strong_count.load(.monotonic);
        }

        pub fn weakCount(self: Self) u32 {
            return self.inner.weak_count.load(.monotonic) - 1; // -1 for strong group
        }
    };
}

const Resource = struct {
    name: []const u8,
    data: u64,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // สร้าง strong reference
    const strong = try StrongRc(Resource).create(alloc, Resource{
        .name = "important resource",
        .data = 42,
    });

    std.debug.print("Strong count: {}, Weak count: {}\n", .{
        strong.strongCount(), strong.weakCount()
    });

    // สร้าง weak reference
    const weak1 = strong.downgrade();
    const weak2 = strong.downgrade();

    std.debug.print("After downgrade - Strong: {}, Weak: {}\n", .{
        strong.strongCount(), strong.weakCount()
    });

    // Upgrade weak -> strong
    if (weak1.upgrade()) |upgraded| {
        std.debug.print("Upgraded! value.data={}\n", .{upgraded.get().data});
        std.debug.print("Strong count now: {}\n", .{upgraded.strongCount()});
        upgraded.release();
    }

    std.debug.print("Weak1 alive: {}\n", .{weak1.isAlive()});

    // Release strong
    strong.release();

    std.debug.print("After strong release:\n", .{});
    std.debug.print("Weak1 alive: {}\n", .{weak1.isAlive()});

    // ลอง upgrade หลัง strong release
    if (weak1.upgrade()) |_| {
        std.debug.print("ERROR: should not upgrade dead object\n", .{});
    } else {
        std.debug.print("Correctly failed to upgrade dead weak ref\n", .{});
    }

    weak1.deinit();
    weak2.deinit();
}
```

---

## Step 745: Cycle Detection

### Breaking Reference Cycles

```zig
const std = @import("std");

// Cycle ทำให้ memory leak ใน RC!
// A -> B -> A (cycle: A และ B ไม่ถูก free)

// แก้โดยใช้ weak reference สำหรับ back-reference

const Node = struct {
    id: u32,
    value: i32,
    // ใช้ parent แบบ weak เพื่อป้องกัน cycle
    parent: ?*anyopaque, // จะเป็น Weak ใน implementation จริง
    children: std.ArrayList(*Node),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, id: u32, value: i32) !*Node {
        const node = try allocator.create(Node);
        node.* = Node{
            .id = id,
            .value = value,
            .parent = null,
            .children = std.ArrayList(*Node).init(allocator),
            .allocator = allocator,
        };
        return node;
    }

    pub fn addChild(self: *Node, child: *Node) !void {
        child.parent = self; // weak-like reference (ไม่ extend lifetime)
        try self.children.append(child);
    }

    pub fn deinit(self: *Node) void {
        std.debug.print("Node {} freed\n", .{self.id});
        for (self.children.items) |child| {
            child.deinit();
            self.allocator.destroy(child);
        }
        self.children.deinit();
    }
};

// RC Tree ที่ถูกต้อง - parent เป็น weak
const TreeNode = struct {
    id: u32,
    value: i32,
    ref_count: usize,
    parent_weak: ?*TreeNode, // Weak - ไม่ increment ref_count
    children_strong: std.ArrayList(*TreeNode), // Strong - increment ref_count
    allocator: std.mem.Allocator,

    pub fn create(allocator: std.mem.Allocator, id: u32, value: i32) !*TreeNode {
        const node = try allocator.create(TreeNode);
        node.* = TreeNode{
            .id = id,
            .value = value,
            .ref_count = 1,
            .parent_weak = null,
            .children_strong = std.ArrayList(*TreeNode).init(allocator),
            .allocator = allocator,
        };
        return node;
    }

    pub fn retain(self: *TreeNode) *TreeNode {
        self.ref_count += 1;
        return self;
    }

    pub fn release(self: *TreeNode) void {
        self.ref_count -= 1;
        if (self.ref_count == 0) {
            self.destroy();
        }
    }

    fn destroy(self: *TreeNode) void {
        std.debug.print("TreeNode {} destroyed\n", .{self.id});
        // Release children (strong refs)
        for (self.children_strong.items) |child| {
            child.release();
        }
        self.children_strong.deinit();
        self.allocator.destroy(self);
    }

    pub fn addChild(self: *TreeNode, child: *TreeNode) !void {
        // ตั้ง parent เป็น weak (ไม่ increment)
        child.parent_weak = self;
        // ตั้ง child เป็น strong (increment)
        _ = child.retain();
        try self.children_strong.append(child);
    }

    pub fn getParent(self: *TreeNode) ?*TreeNode {
        return self.parent_weak; // อาจ invalid ถ้า parent ถูก destroy
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // สร้าง tree ที่ปลอดภัยจาก cycles
    const root = try TreeNode.create(alloc, 1, 100);
    const child1 = try TreeNode.create(alloc, 2, 200);
    const child2 = try TreeNode.create(alloc, 3, 300);
    const grandchild = try TreeNode.create(alloc, 4, 400);

    try root.addChild(child1);
    try root.addChild(child2);
    try child1.addChild(grandchild);

    std.debug.print("Tree structure:\n", .{});
    std.debug.print("  root({}): {} children\n", .{ root.id, root.children_strong.items.len });
    std.debug.print("  child1({}): {} children\n", .{ child1.id, child1.children_strong.items.len });

    // Navigate up using weak parent
    if (grandchild.getParent()) |p| {
        std.debug.print("  grandchild parent: {}\n", .{p.id});
    }

    std.debug.print("\nReleasing tree...\n", .{});
    // ปล่อย ref ที่ caller ถือไว้
    child1.release();
    child2.release();
    grandchild.release();

    // Root release จะ destroy ทั้ง tree (chain)
    root.release();

    std.debug.print("Tree freed without memory leaks\n", .{});
}
```

---

## Step 746: Rc with Custom Destructor

### Custom Destructor

```zig
const std = @import("std");

fn RcWithDtor(comptime T: type, comptime dtor: fn (*T, std.mem.Allocator) void) type {
    return struct {
        const Self = @This();

        const Inner = struct {
            value: T,
            ref_count: usize,
            allocator: std.mem.Allocator,
        };

        inner: *Inner,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = Inner{
                .value = value,
                .ref_count = 1,
                .allocator = allocator,
            };
            return Self{ .inner = inner };
        }

        pub fn retain(self: Self) Self {
            self.inner.ref_count += 1;
            return Self{ .inner = self.inner };
        }

        pub fn release(self: Self) void {
            self.inner.ref_count -= 1;
            if (self.inner.ref_count == 0) {
                dtor(&self.inner.value, self.inner.allocator);
                self.inner.allocator.destroy(self.inner);
            }
        }

        pub fn get(self: Self) *const T {
            return &self.inner.value;
        }

        pub fn getMut(self: Self) *T {
            return &self.inner.value;
        }
    };
}

// File handle ที่ manage ด้วย Rc
const FileResource = struct {
    path: []u8,
    handle_id: u32,
    bytes_written: u64,
};

fn fileResourceDtor(r: *const FileResource, allocator: std.mem.Allocator) void {
    std.debug.print("Closing file: {s} (wrote {} bytes)\n", .{
        r.path, r.bytes_written
    });
    allocator.free(r.path);
}

const RcFile = RcWithDtor(FileResource, fileResourceDtor);

// Database connection ที่ manage ด้วย Rc
const DbConnection = struct {
    host: [64]u8,
    port: u16,
    connection_id: u32,
    query_count: u64,
};

fn dbConnectionDtor(c: *const DbConnection, _: std.mem.Allocator) void {
    std.debug.print("Disconnecting from {s}:{} (ran {} queries)\n", .{
        c.host[0..@min(std.mem.indexOfScalar(u8, &c.host, 0) orelse 64, 64)],
        c.port,
        c.query_count,
    });
}

const RcDbConn = RcWithDtor(DbConnection, dbConnectionDtor);

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // File resource
    const path = try alloc.dupe(u8, "/tmp/data.txt");
    const file_rc = try RcFile.create(alloc, FileResource{
        .path = path,
        .handle_id = 42,
        .bytes_written = 0,
    });

    const file_rc2 = file_rc.retain();
    file_rc.getMut().bytes_written += 1024;

    file_rc.release(); // ref_count = 1
    file_rc2.release(); // ref_count = 0 -> calls fileResourceDtor

    // DB connection
    var db_conn = DbConnection{
        .host = undefined,
        .port = 5432,
        .connection_id = 1,
        .query_count = 0,
    };
    @memset(&db_conn.host, 0);
    @memcpy(db_conn.host[0..9], "localhost");

    const conn_rc = try RcDbConn.create(alloc, db_conn);
    const conn_rc2 = conn_rc.retain();
    const conn_rc3 = conn_rc.retain();

    // Use connection
    conn_rc.getMut().query_count += 10;
    conn_rc2.getMut().query_count += 5;

    conn_rc.release();
    conn_rc2.release();
    conn_rc3.release(); // จะเรียก dbConnectionDtor
}
```

---

## Step 747: Intrusive Reference Counting

### Intrusive RC - Counter อยู่ใน Object

```zig
const std = @import("std");

// Intrusive RC: ref_count อยู่ใน T เอง (ไม่ต้อง allocate wrapper)
const IntrusiveBase = struct {
    ref_count: usize,

    pub fn init(self: *IntrusiveBase) void {
        self.ref_count = 1;
    }

    pub fn retain(self: *IntrusiveBase) void {
        self.ref_count += 1;
    }

    pub fn release(self: *IntrusiveBase, comptime T: type, allocator: std.mem.Allocator) void {
        self.ref_count -= 1;
        if (self.ref_count == 0) {
            const obj: *T = @fieldParentPtr("rc_base", self);
            if (comptime std.meta.hasFn(T, "onDestroy")) {
                obj.onDestroy();
            }
            allocator.destroy(obj);
        }
    }
};

fn IntrusiveRc(comptime T: type) type {
    return struct {
        const Self = @This();

        ptr: *T,
        allocator: std.mem.Allocator,

        pub fn create(allocator: std.mem.Allocator, value: T) !Self {
            const ptr = try allocator.create(T);
            ptr.* = value;
            // T ต้องมี field rc_base: IntrusiveBase
            const base: *IntrusiveBase = &ptr.rc_base;
            base.init();
            return Self{ .ptr = ptr, .allocator = allocator };
        }

        pub fn retain(self: Self) Self {
            self.ptr.rc_base.retain();
            return Self{ .ptr = self.ptr, .allocator = self.allocator };
        }

        pub fn release(self: Self) void {
            self.ptr.rc_base.release(T, self.allocator);
        }

        pub fn get(self: Self) *T {
            return self.ptr;
        }

        pub fn getConst(self: Self) *const T {
            return self.ptr;
        }

        pub fn refCount(self: Self) usize {
            return self.ptr.rc_base.ref_count;
        }
    };
}

const Widget = struct {
    rc_base: IntrusiveBase, // ต้องมี field นี้
    id: u32,
    name: [32]u8,
    x: i32,
    y: i32,
    width: i32,
    height: i32,

    pub fn init(id: u32, name: []const u8, x: i32, y: i32, w: i32, h: i32) Widget {
        var widget = Widget{
            .rc_base = undefined, // จะ init ใน IntrusiveRc.create
            .id = id,
            .name = undefined,
            .x = x, .y = y,
            .width = w, .height = h,
        };
        @memset(&widget.name, 0);
        @memcpy(widget.name[0..@min(name.len, 31)], name[0..@min(name.len, 31)]);
        return widget;
    }

    pub fn onDestroy(self: *Widget) void {
        std.debug.print("Widget {} ({s}) destroyed\n", .{
            self.id,
            self.name[0..@min(std.mem.indexOfScalar(u8, &self.name, 0) orelse 32, 32)],
        });
    }

    pub fn getName(self: *const Widget) []const u8 {
        const len = std.mem.indexOfScalar(u8, &self.name, 0) orelse self.name.len;
        return self.name[0..len];
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // สร้าง widgets
    const button = try IntrusiveRc(Widget).create(alloc, Widget.init(
        1, "Button", 100, 200, 80, 30
    ));

    const label = try IntrusiveRc(Widget).create(alloc, Widget.init(
        2, "Label", 10, 10, 200, 20
    ));

    std.debug.print("Button ref_count: {}\n", .{button.refCount()});

    // Share widget
    const button2 = button.retain();
    std.debug.print("After share, ref_count: {}\n", .{button.refCount()});

    // Use
    std.debug.print("Button name: {s}\n", .{button.getConst().getName()});
    std.debug.print("Label name: {s}\n", .{label.getConst().getName()});

    // Release
    button.release();
    std.debug.print("After release, ref_count: {}\n", .{button2.refCount()});

    button2.release(); // จะ destroy
    label.release();   // จะ destroy
}
```

---

## Step 748: Performance of Rc vs GC

### Performance Comparison

```zig
const std = @import("std");
const time = std.time;

fn SimpleRc(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = struct {
            value: T,
            count: usize,
            alloc: std.mem.Allocator,
        };
        inner: *Inner,

        pub fn new(allocator: std.mem.Allocator, v: T) !Self {
            const p = try allocator.create(Inner);
            p.* = .{ .value = v, .count = 1, .alloc = allocator };
            return .{ .inner = p };
        }

        pub fn clone(self: Self) Self {
            self.inner.count += 1;
            return .{ .inner = self.inner };
        }

        pub fn drop(self: Self) void {
            self.inner.count -= 1;
            if (self.inner.count == 0) {
                self.inner.alloc.destroy(self.inner);
            }
        }

        pub fn val(self: Self) *const T { return &self.inner.value; }
    };
}

fn benchRcCloneRelease(alloc: std.mem.Allocator, n: usize) !u64 {
    const rc = try SimpleRc(u64).new(alloc, 42);
    const start = time.nanoTimestamp();

    var i: usize = 0;
    while (i < n) : (i += 1) {
        const rc2 = rc.clone();
        rc2.drop();
    }

    const elapsed = @as(u64, @intCast(time.nanoTimestamp() - start));
    rc.drop();
    return elapsed;
}

fn benchDirectAlloc(alloc: std.mem.Allocator, n: usize) !u64 {
    const start = time.nanoTimestamp();

    var i: usize = 0;
    while (i < n) : (i += 1) {
        const p = try alloc.create(u64);
        p.* = 42;
        alloc.destroy(p);
    }

    return @intCast(time.nanoTimestamp() - start);
}

fn benchArenaAlloc(backing: std.mem.Allocator, n: usize) !u64 {
    var arena = std.heap.ArenaAllocator.init(backing);
    defer arena.deinit();
    const alloc = arena.allocator();

    const start = time.nanoTimestamp();

    var i: usize = 0;
    while (i < n) : (i += 1) {
        const p = try alloc.create(u64);
        p.* = 42;
    }

    return @intCast(time.nanoTimestamp() - start);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    const N: usize = 100_000;
    std.debug.print("=== RC Performance Analysis (N={}) ===\n\n", .{N});

    const rc_time = try benchRcCloneRelease(alloc, N);
    std.debug.print("Rc clone+drop: {} ns ({} ns/op)\n", .{ rc_time, rc_time / N });

    const direct_time = try benchDirectAlloc(alloc, N);
    std.debug.print("Direct alloc:  {} ns ({} ns/op)\n", .{ direct_time, direct_time / N });

    const arena_time = try benchArenaAlloc(alloc, N);
    std.debug.print("Arena alloc:   {} ns ({} ns/op)\n", .{ arena_time, arena_time / N });

    std.debug.print("\nRC overhead vs direct: {d:.2}x\n", .{
        @as(f64, @floatFromInt(rc_time)) / @as(f64, @floatFromInt(direct_time))
    });

    std.debug.print(
        \\
        \\Performance Notes:
        \\  - RC add ~1-2 ns overhead per clone/drop (counter op)
        \\  - Thread-safe Arc: ~10-50 ns (atomic CAS)
        \\  - GC (estimated): variable pause, 10-100x overhead
        \\  - Arena: fastest allocation, no individual free
        \\  - Manual: fastest but error-prone
        \\
    , .{});
}
```

---

## Step 749: Rc Patterns ใน Real Code

### Real-World RC Patterns

```zig
const std = @import("std");

// Pattern 1: Shared Configuration
fn SharedConfig(comptime T: type) type {
    return struct {
        const Self = @This();
        const Inner = struct {
            config: T,
            ref_count: usize,
            allocator: std.mem.Allocator,
        };
        inner: *Inner,

        pub fn create(allocator: std.mem.Allocator, config: T) !Self {
            const inner = try allocator.create(Inner);
            inner.* = .{ .config = config, .ref_count = 1, .allocator = allocator };
            return .{ .inner = inner };
        }

        pub fn retain(self: Self) Self {
            self.inner.ref_count += 1;
            return .{ .inner = self.inner };
        }

        pub fn release(self: Self) void {
            self.inner.ref_count -= 1;
            if (self.inner.ref_count == 0) {
                self.inner.allocator.destroy(self.inner);
            }
        }

        pub fn get(self: Self) *const T { return &self.inner.config; }
    };
}

const AppConfig = struct {
    log_level: u8,
    max_connections: u32,
    timeout_ms: u32,
    feature_flags: u64,
};

const Service = struct {
    name: []const u8,
    config: SharedConfig(AppConfig),

    pub fn init(name: []const u8, config: SharedConfig(AppConfig)) Service {
        return .{ .name = name, .config = config.retain() };
    }

    pub fn deinit(self: *Service) void {
        self.config.release();
    }

    pub fn run(self: *const Service) void {
        const cfg = self.config.get();
        std.debug.print("Service {s}: log_level={}, max_conn={}\n", .{
            self.name, cfg.log_level, cfg.max_connections
        });
    }
};

// Pattern 2: Event Handler with Rc
const EventHandler = struct {
    id: u32,
    name: []const u8,
    call_count: u32,
    ref_count: usize,
    allocator: std.mem.Allocator,

    pub fn create(allocator: std.mem.Allocator, id: u32, name: []const u8) !*EventHandler {
        const handler = try allocator.create(EventHandler);
        handler.* = .{
            .id = id,
            .name = name,
            .call_count = 0,
            .ref_count = 1,
            .allocator = allocator,
        };
        return handler;
    }

    pub fn retain(self: *EventHandler) *EventHandler {
        self.ref_count += 1;
        return self;
    }

    pub fn release(self: *EventHandler) void {
        self.ref_count -= 1;
        if (self.ref_count == 0) {
            std.debug.print("Handler {} ({s}) freed after {} calls\n", .{
                self.id, self.name, self.call_count
            });
            self.allocator.destroy(self);
        }
    }

    pub fn handle(self: *EventHandler, event: []const u8) void {
        self.call_count += 1;
        std.debug.print("Handler {s}: {s} (call #{})\n", .{
            self.name, event, self.call_count
        });
    }
};

const EventEmitter = struct {
    handlers: std.ArrayList(*EventHandler),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) EventEmitter {
        return .{
            .handlers = std.ArrayList(*EventHandler).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *EventEmitter) void {
        for (self.handlers.items) |h| h.release();
        self.handlers.deinit();
    }

    pub fn subscribe(self: *EventEmitter, handler: *EventHandler) !void {
        try self.handlers.append(handler.retain()); // เพิ่ม ref
    }

    pub fn unsubscribe(self: *EventEmitter, handler: *EventHandler) void {
        for (self.handlers.items, 0..) |h, i| {
            if (h == handler) {
                _ = self.handlers.swapRemove(i);
                h.release(); // ลด ref
                return;
            }
        }
    }

    pub fn emit(self: *EventEmitter, event: []const u8) void {
        for (self.handlers.items) |h| h.handle(event);
    }
};

// Pattern 3: Cache ด้วย Rc
fn RcCache(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const RcValue = struct {
            value: V,
            ref_count: usize,
            allocator: std.mem.Allocator,

            pub fn retain(self: *@This()) void {
                self.ref_count += 1;
            }

            pub fn release(self: *@This()) void {
                self.ref_count -= 1;
                if (self.ref_count == 0) {
                    self.allocator.destroy(self);
                }
            }
        };

        map: std.AutoHashMap(K, *RcValue),
        allocator: std.mem.Allocator,

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .map = std.AutoHashMap(K, *RcValue).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            var it = self.map.iterator();
            while (it.next()) |entry| {
                entry.value_ptr.*.release();
            }
            self.map.deinit();
        }

        pub fn put(self: *Self, key: K, value: V) !*RcValue {
            const rv = try self.allocator.create(RcValue);
            rv.* = .{ .value = value, .ref_count = 1, .allocator = self.allocator };
            try self.map.put(key, rv);
            return rv;
        }

        pub fn get(self: *Self, key: K) ?*RcValue {
            if (self.map.get(key)) |rv| {
                rv.retain();
                return rv;
            }
            return null;
        }

        pub fn remove(self: *Self, key: K) void {
            if (self.map.fetchRemove(key)) |entry| {
                entry.value.release();
            }
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    // Pattern 1: Shared Config
    std.debug.print("=== Shared Config Pattern ===\n", .{});
    const config_rc = try SharedConfig(AppConfig).create(alloc, AppConfig{
        .log_level = 2,
        .max_connections = 100,
        .timeout_ms = 5000,
        .feature_flags = 0xFF,
    });

    var svc1 = Service.init("auth", config_rc);
    var svc2 = Service.init("api", config_rc);
    var svc3 = Service.init("db", config_rc);
    config_rc.release(); // Original reference

    svc1.run();
    svc2.run();
    svc3.run();

    svc1.deinit();
    svc2.deinit();
    svc3.deinit();

    // Pattern 2: Event Emitter
    std.debug.print("\n=== Event Emitter Pattern ===\n", .{});
    var emitter = EventEmitter.init(alloc);
    defer emitter.deinit();

    const h1 = try EventHandler.create(alloc, 1, "logger");
    const h2 = try EventHandler.create(alloc, 2, "metrics");
    const h3 = try EventHandler.create(alloc, 3, "alert");

    try emitter.subscribe(h1);
    try emitter.subscribe(h2);
    try emitter.subscribe(h3);

    emitter.emit("user.login");
    emitter.emit("user.logout");

    emitter.unsubscribe(h2);
    emitter.emit("error.critical");

    h1.release();
    h2.release();
    h3.release();

    // Pattern 3: RC Cache
    std.debug.print("\n=== RC Cache Pattern ===\n", .{});
    var cache = RcCache(u32, []const u8).init(alloc);
    defer cache.deinit();

    _ = try cache.put(1, "user_data_1");
    _ = try cache.put(2, "user_data_2");
    _ = try cache.put(3, "user_data_3");

    if (cache.get(1)) |entry| {
        std.debug.print("Cache hit: {s}\n", .{entry.value});
        entry.release();
    }

    cache.remove(2);
    std.debug.print("Cache entry 2 removed\n", .{});
}
```

---

## Step 750: Summary of Memory Management

### สรุป Memory Management ใน Zig

```zig
const std = @import("std");

// Memory Management Summary

// 1. Manual Memory (Stack)
pub fn stackExample() void {
    var x: i32 = 42;
    var arr: [100]u8 = undefined;
    @memset(&arr, 0);
    std.debug.print("Stack: x={}, arr[0]={}\n", .{x, arr[0]});
    // Free อัตโนมัติเมื่อ function return
}

// 2. Manual Memory (Heap)
pub fn heapExample(alloc: std.mem.Allocator) !void {
    const buf = try alloc.alloc(u8, 1024);
    defer alloc.free(buf); // Must free manually!
    @memset(buf, 0xFF);
    std.debug.print("Heap: {} bytes\n", .{buf.len});
}

// 3. Arena Allocator
pub fn arenaExample(backing: std.mem.Allocator) !void {
    var arena = std.heap.ArenaAllocator.init(backing);
    defer arena.deinit(); // Free ทั้งหมดพร้อมกัน

    const a = arena.allocator();
    const buf1 = try a.alloc(u8, 100);
    const buf2 = try a.alloc(u32, 50);
    const str = try std.fmt.allocPrint(a, "value={}", .{42});
    _ = buf1; _ = buf2;
    std.debug.print("Arena: {s}\n", .{str});
}

// 4. Object Pool
pub fn poolExample() void {
    // Pool pre-allocates objects
    var pool: [16]struct { active: bool, value: u32 } = undefined;
    for (&pool) |*slot| slot.active = false;

    // Acquire
    for (&pool) |*slot| {
        if (!slot.active) {
            slot.active = true;
            slot.value = 42;
            break;
        }
    }
    std.debug.print("Pool: object acquired\n", .{});
}

// 5. Reference Counting
pub fn rcExample(alloc: std.mem.Allocator) !void {
    const Inner = struct {
        value: u64,
        count: usize,
        allocator: std.mem.Allocator,
    };
    const inner = try alloc.create(Inner);
    inner.* = .{ .value = 42, .count = 1, .allocator = alloc };

    // Retain
    inner.count += 1;
    inner.count += 1;

    // Release
    inner.count -= 1;
    inner.count -= 1;
    inner.count -= 1;
    if (inner.count == 0) {
        alloc.destroy(inner);
        std.debug.print("RC: freed\n", .{});
    }
}

// Decision Matrix
pub fn printDecisionMatrix() void {
    std.debug.print(
        \\
        \\=== Memory Management Decision Matrix ===
        \\
        \\Approach        | Overhead | Safety | Complexity | Best For
        \\----------------|----------|--------|------------|------------------
        \\Stack           | None     | High   | Low        | Small, local data
        \\Manual Heap     | Low      | Low    | Medium     | Known lifetime
        \\Arena           | Very Low | High   | Low        | Scoped allocation
        \\Object Pool     | Low      | Medium | Medium     | Frequent alloc/free
        \\Reference Count | Medium   | High   | Medium     | Shared ownership
        \\Slab Allocator  | Low      | Medium | High       | Same-size objects
        \\Buddy Allocator | Medium   | Medium | High       | General purpose
        \\
        \\=== When to Use What ===
        \\
        \\Stack:        Local variables, small arrays, struct values
        \\Manual Heap:  Single owner, known cleanup point
        \\Arena:        Request/response, parser, game frame
        \\Pool:         Particles, bullets, connections, frequent churn
        \\Rc/Arc:       Shared ownership, callback/observer, caches
        \\Slab:         Many same-size objects (kernel, DB)
        \\Buddy:        General allocator building block
        \\
        \\=== Zig-Specific Tips ===
        \\
        \\1. Always pass allocator explicitly - no hidden allocation
        \\2. Use defer for cleanup - RAII-like pattern
        \\3. prefer ArenaAllocator for request-scoped work
        \\4. Use FixedBufferAllocator for stack-based allocation
        \\5. Test with std.testing.allocator to detect leaks
        \\6. Profile before optimizing allocators
        \\7. Consider comptime to avoid runtime allocation
        \\
    , .{});
}

// Final comprehensive example
const AppMemoryStrategy = struct {
    // Long-lived: GPA
    global_alloc: std.heap.GeneralPurposeAllocator(.{}),

    // Request-scoped: Arena
    request_arena: std.heap.ArenaAllocator,

    // Frequent objects: Pool
    connection_pool: struct {
        items: [64]struct { active: bool, id: u32 },
        count: usize,

        fn init(self: *@This()) void {
            for (&self.items) |*item| item.active = false;
            self.count = 0;
        }

        fn acquire(self: *@This()) ?*struct { active: bool, id: u32 } {
            for (&self.items) |*item| {
                if (!item.active) {
                    item.active = true;
                    self.count += 1;
                    return item;
                }
            }
            return null;
        }

        fn release(self: *@This(), item: *struct { active: bool, id: u32 }) void {
            item.active = false;
            self.count -= 1;
        }
    },

    pub fn init(self: *AppMemoryStrategy) void {
        self.global_alloc = std.heap.GeneralPurposeAllocator(.{}){};
        self.request_arena = std.heap.ArenaAllocator.init(self.global_alloc.allocator());
        self.connection_pool.init();
    }

    pub fn deinit(self: *AppMemoryStrategy) void {
        self.request_arena.deinit();
        _ = self.global_alloc.deinit();
    }

    pub fn handleRequest(self: *AppMemoryStrategy, path: []const u8) !void {
        defer _ = self.request_arena.reset(.retain_capacity);
        const arena_alloc = self.request_arena.allocator();

        // Use pool for connection
        const conn = self.connection_pool.acquire() orelse return error.NoConnections;
        defer self.connection_pool.release(conn);
        conn.id = 42;

        // Use arena for request processing
        const processed = try std.fmt.allocPrint(arena_alloc,
            "Processed: {s} via connection {}", .{ path, conn.id });

        std.debug.print("{s}\n", .{processed});
        // arena freed on return via defer
    }
};

pub fn main() !void {
    stackExample();

    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    try heapExample(alloc);
    try arenaExample(alloc);
    poolExample();
    try rcExample(alloc);

    printDecisionMatrix();

    // Comprehensive example
    var app: AppMemoryStrategy = undefined;
    app.init();
    defer app.deinit();

    try app.handleRequest("/api/users");
    try app.handleRequest("/api/products");
    try app.handleRequest("/api/orders");

    std.debug.print("\nAll patterns demonstrated successfully!\n", .{});
}
```

---

## สรุป Part 75 และ Memory Management Series

### สรุป Reference Counting

| Step | หัวข้อ | ความสำคัญ |
|------|--------|----------|
| 741 | Concept | เข้าใจว่า RC คืออะไรและแก้ปัญหาอะไร |
| 742 | Simple Rc(T) | Single-threaded implementation |
| 743 | Arc(T) | Thread-safe atomic version |
| 744 | Weak References | ป้องกัน cycles |
| 745 | Cycle Detection | หาและแก้ reference cycles |
| 746 | Custom Destructor | RC พร้อม cleanup logic |
| 747 | Intrusive RC | Counter อยู่ใน object |
| 748 | Performance | RC overhead analysis |
| 749 | Real Patterns | Shared config, events, cache |
| 750 | Summary | Decision matrix ทั้งหมด |

### Memory Management Hierarchy ใน Zig

```
Stack (fastest, safest)
  ↓
FixedBufferAllocator (stack-like heap)
  ↓
Arena (batch lifetime)
  ↓
Object Pool (frequent same-type objects)
  ↓
Slab Allocator (same-size objects)
  ↓
Buddy Allocator (power-of-2, low fragmentation)
  ↓
Free List (general, fragmentation possible)
  ↓
GPA / GeneralPurposeAllocator (default choice)
  ↓
Reference Counting (shared ownership)
  ↓
Garbage Collection (last resort, not in Zig)
```

### กฎทองของ Memory Management ใน Zig

1. **เจ้าของชัดเจน**: กำหนดให้ชัดว่าใครเป็นเจ้าของ memory
2. **ใช้ defer**: cleanup resource ทันทีที่ acquire
3. **Arena first**: ลองใช้ Arena ก่อนเสมอถ้ามี scope ชัดเจน
4. **Pool สำหรับ churn**: ถ้า allocate/free บ่อย ใช้ pool
5. **RC สำหรับ sharing**: ถ้าต้องการ shared ownership ใช้ RC
6. **Test memory**: ใช้ `std.testing.allocator` หรือ GPA detect leaks
7. **Profile ก่อน optimize**: อย่า over-engineer allocators

---

*จบ Part 71-75: Advanced Memory Management ใน Zig*

*หัวข้อถัดไป: Part 76 - Concurrency Patterns ใน Zig*
