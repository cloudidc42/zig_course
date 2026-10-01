# Part 46: Multithreading ใน Zig

## ภาพรวม

Multithreading เป็นเทคนิคการเขียนโปรแกรมที่ช่วยให้โปรแกรมสามารถทำงานหลายอย่างพร้อมกันได้ Zig มีการสนับสนุน multithreading ผ่าน `std.Thread` และ primitive synchronization ต่างๆ ในบทนี้เราจะเรียนรู้วิธีใช้งาน threads อย่างปลอดภัยและมีประสิทธิภาพ

---

## Step 451: Threads ใน Zig - std.Thread

Thread (เธรด) คือหน่วยการทำงานที่เล็กที่สุดที่ระบบปฏิบัติการสามารถจัดการได้ หลาย thread สามารถทำงานพร้อมกันภายใน process เดียวกัน

### ความแตกต่างระหว่าง Process และ Thread

```
Process:
┌─────────────────────────────────────┐
│  Memory Space (Heap, Stack, Code)   │
│  ┌──────────┐  ┌──────────┐        │
│  │ Thread 1 │  │ Thread 2 │        │
│  │  Stack   │  │  Stack   │        │
│  └──────────┘  └──────────┘        │
│         Shared Heap/Data            │
└─────────────────────────────────────┘
```

### std.Thread ใน Zig

```zig
const std = @import("std");
const Thread = std.Thread;

pub fn main() !void {
    // ข้อมูลพื้นฐานเกี่ยวกับ Thread
    std.debug.print("Thread API พื้นฐานใน Zig\n", .{});
    
    // ดู CPU count
    const cpu_count = try Thread.getCpuCount();
    std.debug.print("จำนวน CPU: {d}\n", .{cpu_count});
}
```

### Thread States

Thread สามารถอยู่ในสถานะต่างๆ ได้ดังนี้:
- **Created**: Thread ถูกสร้างแต่ยังไม่เริ่มทำงาน
- **Running**: Thread กำลังทำงานอยู่
- **Blocked**: Thread รอทรัพยากรบางอย่าง
- **Terminated**: Thread ทำงานเสร็จแล้ว

### Thread vs Async

ใน Zig, threads เป็นการทำงานจริงๆ แบบ parallel ในขณะที่ async/await เป็น cooperative concurrency:

```zig
const std = @import("std");

// ตัวอย่างการใช้ thread อย่างง่าย
fn simpleTask(id: u32) void {
    std.debug.print("Task {d} กำลังทำงาน\n", .{id});
    
    // จำลองการทำงาน
    var i: u32 = 0;
    while (i < 1000000) : (i += 1) {}
    
    std.debug.print("Task {d} เสร็จแล้ว\n", .{id});
}
```

---

## Step 452: Thread.spawn

`Thread.spawn` ใช้สำหรับสร้าง thread ใหม่และเริ่มการทำงาน

### รูปแบบพื้นฐาน

```zig
const std = @import("std");
const Thread = std.Thread;

fn workerFn(data: u32) void {
    std.debug.print("Worker {d} เริ่มทำงาน\n", .{data});
    
    // ทำงาน...
    std.time.sleep(100 * std.time.ns_per_ms); // รอ 100ms
    
    std.debug.print("Worker {d} เสร็จแล้ว\n", .{data});
}

pub fn main() !void {
    // spawn thread
    const thread = try Thread.spawn(.{}, workerFn, .{42});
    
    std.debug.print("Thread ถูกสร้างแล้ว\n", .{});
    
    // รอ thread ทำงานเสร็จ
    thread.join();
    
    std.debug.print("Thread เสร็จสิ้น\n", .{});
}
```

### การส่งข้อมูลให้ Thread

```zig
const std = @import("std");
const Thread = std.Thread;

// struct สำหรับส่งข้อมูลหลายค่า
const WorkerData = struct {
    id: u32,
    iterations: u64,
    message: []const u8,
};

fn workerWithData(data: WorkerData) void {
    std.debug.print("Worker {d}: {s}\n", .{ data.id, data.message });
    
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < data.iterations) : (i += 1) {
        sum += i;
    }
    
    std.debug.print("Worker {d}: sum = {d}\n", .{ data.id, sum });
}

pub fn main() !void {
    const data1 = WorkerData{
        .id = 1,
        .iterations = 1000000,
        .message = "คำนวณ sum",
    };
    
    const data2 = WorkerData{
        .id = 2,
        .iterations = 2000000,
        .message = "คำนวณ sum ขนาดใหญ่",
    };
    
    const t1 = try Thread.spawn(.{}, workerWithData, .{data1});
    const t2 = try Thread.spawn(.{}, workerWithData, .{data2});
    
    t1.join();
    t2.join();
    
    std.debug.print("ทุก thread เสร็จแล้ว\n", .{});
}
```

### Thread Options

```zig
const std = @import("std");
const Thread = std.Thread;

pub fn main() !void {
    // กำหนด stack size
    const options = Thread.SpawnConfig{
        .stack_size = 1024 * 1024, // 1MB stack
    };
    
    const thread = try Thread.spawn(options, myWorker, .{});
    thread.join();
}

fn myWorker() void {
    std.debug.print("Worker พร้อม stack ที่กำหนดเอง\n", .{});
}
```

### การสร้าง Thread หลายตัว

```zig
const std = @import("std");
const Thread = std.Thread;

fn computeChunk(start: u64, end: u64, result: *u64) void {
    var sum: u64 = 0;
    var i: u64 = start;
    while (i < end) : (i += 1) {
        sum += i;
    }
    result.* = sum;
}

pub fn main() !void {
    const NUM_THREADS = 4;
    const TOTAL = 1000000;
    const CHUNK = TOTAL / NUM_THREADS;
    
    var threads: [NUM_THREADS]Thread = undefined;
    var results: [NUM_THREADS]u64 = [_]u64{0} ** NUM_THREADS;
    
    // สร้าง threads
    var i: usize = 0;
    while (i < NUM_THREADS) : (i += 1) {
        const start = i * CHUNK;
        const end = if (i == NUM_THREADS - 1) TOTAL else start + CHUNK;
        threads[i] = try Thread.spawn(.{}, computeChunk, .{ start, end, &results[i] });
    }
    
    // รอทุก thread เสร็จ
    for (&threads) |*t| {
        t.join();
    }
    
    // รวมผลลัพธ์
    var total: u64 = 0;
    for (results) |r| {
        total += r;
    }
    
    std.debug.print("ผลรวม 0..{d} = {d}\n", .{ TOTAL, total });
}
```

---

## Step 453: Thread.join

`Thread.join` ใช้รอให้ thread ทำงานเสร็จก่อนดำเนินการต่อ

### รูปแบบการใช้งาน

```zig
const std = @import("std");
const Thread = std.Thread;

fn longTask(seconds: u64) void {
    std.debug.print("เริ่มงานที่ใช้เวลา {d} วินาที\n", .{seconds});
    std.time.sleep(seconds * std.time.ns_per_s);
    std.debug.print("งาน {d} วินาที เสร็จแล้ว\n", .{seconds});
}

pub fn main() !void {
    const start_time = std.time.nanoTimestamp();
    
    // สร้าง threads พร้อมกัน
    const t1 = try Thread.spawn(.{}, longTask, .{@as(u64, 2)});
    const t2 = try Thread.spawn(.{}, longTask, .{@as(u64, 3)});
    const t3 = try Thread.spawn(.{}, longTask, .{@as(u64, 1)});
    
    // รอทุก thread
    t1.join();
    t2.join();
    t3.join();
    
    const elapsed = std.time.nanoTimestamp() - start_time;
    std.debug.print("เวลาทั้งหมด: {d}ms (ควรประมาณ 3000ms)\n", .{@divTrunc(elapsed, std.time.ns_per_ms)});
}
```

### Thread Detach

บางครั้งเราไม่ต้องการรอ thread - ใช้ `detach()`:

```zig
const std = @import("std");
const Thread = std.Thread;

fn backgroundTask() void {
    // งานที่รันอยู่เบื้องหลัง
    std.time.sleep(5 * std.time.ns_per_s);
    std.debug.print("Background task เสร็จ\n", .{});
}

pub fn main() !void {
    const thread = try Thread.spawn(.{}, backgroundTask, .{});
    
    // ไม่รอ - ปล่อยให้ทำงานเบื้องหลัง
    thread.detach();
    
    std.debug.print("Main thread ดำเนินต่อได้เลย\n", .{});
    
    // หมายเหตุ: เมื่อ main() จบ process ก็จบ
    // background task อาจยังไม่เสร็จ!
}
```

---

## Step 454: Mutex - std.Thread.Mutex

Mutex (Mutual Exclusion) ใช้ป้องกันการเข้าถึง shared resource พร้อมกันจากหลาย threads

### ปัญหา Race Condition

```zig
const std = @import("std");
const Thread = std.Thread;

// ตัวอย่างที่ผิด - Race condition!
var counter: u64 = 0;

fn incrementUnsafe() void {
    var i: u32 = 0;
    while (i < 100000) : (i += 1) {
        counter += 1; // ไม่ปลอดภัย!
    }
}

pub fn mainBad() !void {
    const t1 = try Thread.spawn(.{}, incrementUnsafe, .{});
    const t2 = try Thread.spawn(.{}, incrementUnsafe, .{});
    
    t1.join();
    t2.join();
    
    // counter อาจไม่ใช่ 200000!
    std.debug.print("Counter: {d} (ควรเป็น 200000)\n", .{counter});
}
```

### แก้ด้วย Mutex

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;

var mutex = Mutex{};
var counter: u64 = 0;

fn incrementSafe() void {
    var i: u32 = 0;
    while (i < 100000) : (i += 1) {
        mutex.lock();
        defer mutex.unlock();
        counter += 1; // ปลอดภัยแล้ว!
    }
}

pub fn main() !void {
    const t1 = try Thread.spawn(.{}, incrementSafe, .{});
    const t2 = try Thread.spawn(.{}, incrementSafe, .{});
    
    t1.join();
    t2.join();
    
    // counter จะเป็น 200000 เสมอ
    std.debug.print("Counter: {d}\n", .{counter});
}
```

### Mutex กับ Struct

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;

const SafeCounter = struct {
    mutex: Mutex = .{},
    value: u64 = 0,
    
    pub fn increment(self: *SafeCounter) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        self.value += 1;
    }
    
    pub fn get(self: *SafeCounter) u64 {
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.value;
    }
    
    pub fn reset(self: *SafeCounter) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        self.value = 0;
    }
};

fn worker(counter: *SafeCounter) void {
    var i: u32 = 0;
    while (i < 50000) : (i += 1) {
        counter.increment();
    }
}

pub fn main() !void {
    var counter = SafeCounter{};
    
    var threads: [4]Thread = undefined;
    for (&threads) |*t| {
        t.* = try Thread.spawn(.{}, worker, .{&counter});
    }
    
    for (&threads) |*t| {
        t.join();
    }
    
    std.debug.print("Final counter: {d} (ควรเป็น 200000)\n", .{counter.get()});
}
```

### tryLock

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;

var mutex = Mutex{};

fn tryToWork() void {
    if (mutex.tryLock()) {
        defer mutex.unlock();
        std.debug.print("ได้ lock แล้ว กำลังทำงาน\n", .{});
        std.time.sleep(100 * std.time.ns_per_ms);
    } else {
        std.debug.print("ไม่ได้ lock ทำงานอื่นแทน\n", .{});
    }
}
```

---

## Step 455: RwLock - std.Thread.RwLock

RwLock (Read-Write Lock) อนุญาตให้หลาย reader อ่านพร้อมกัน แต่ writer ต้องการสิทธิ์เฉพาะตัว

### ทำไมต้องใช้ RwLock

- **Mutex**: อ่านหรือเขียนได้ครั้งละ 1 thread เท่านั้น
- **RwLock**: หลาย thread อ่านได้พร้อมกัน, เขียนต้องรอทุกคนออกก่อน

```zig
const std = @import("std");
const Thread = std.Thread;
const RwLock = Thread.RwLock;

const Database = struct {
    rwlock: RwLock = .{},
    data: [100]u32 = undefined,
    size: usize = 0,
    
    pub fn read(self: *Database, index: usize) ?u32 {
        self.rwlock.lockShared();
        defer self.rwlock.unlockShared();
        
        if (index >= self.size) return null;
        return self.data[index];
    }
    
    pub fn write(self: *Database, value: u32) bool {
        self.rwlock.lock();
        defer self.rwlock.unlock();
        
        if (self.size >= self.data.len) return false;
        self.data[self.size] = value;
        self.size += 1;
        return true;
    }
    
    pub fn getAll(self: *Database, allocator: std.mem.Allocator) ![]u32 {
        self.rwlock.lockShared();
        defer self.rwlock.unlockShared();
        
        const copy = try allocator.alloc(u32, self.size);
        @memcpy(copy, self.data[0..self.size]);
        return copy;
    }
};

fn readerTask(db: *Database, reader_id: u32) void {
    var i: u32 = 0;
    while (i < 10) : (i += 1) {
        const val = db.read(i);
        if (val) |v| {
            std.debug.print("Reader {d}: index {d} = {d}\n", .{ reader_id, i, v });
        }
        std.time.sleep(10 * std.time.ns_per_ms);
    }
}

fn writerTask(db: *Database, writer_id: u32) void {
    var i: u32 = 0;
    while (i < 5) : (i += 1) {
        const value = writer_id * 100 + i;
        if (db.write(value)) {
            std.debug.print("Writer {d}: เขียน {d}\n", .{ writer_id, value });
        }
        std.time.sleep(50 * std.time.ns_per_ms);
    }
}

pub fn main() !void {
    var db = Database{};
    
    // เริ่ม writer ก่อน
    const w1 = try Thread.spawn(.{}, writerTask, .{ &db, @as(u32, 1) });
    
    // เริ่ม readers
    const r1 = try Thread.spawn(.{}, readerTask, .{ &db, @as(u32, 1) });
    const r2 = try Thread.spawn(.{}, readerTask, .{ &db, @as(u32, 2) });
    const r3 = try Thread.spawn(.{}, readerTask, .{ &db, @as(u32, 3) });
    
    w1.join();
    r1.join();
    r2.join();
    r3.join();
}
```

---

## Step 456: Semaphore

Semaphore ควบคุมจำนวน thread ที่สามารถเข้าถึง resource พร้อมกันได้

### การใช้งาน Semaphore

```zig
const std = @import("std");
const Thread = std.Thread;
const Semaphore = Thread.Semaphore;

// จำกัดการเข้าถึง resource พร้อมกัน 3 thread
var semaphore = Semaphore{ .permits = 3 };

fn accessResource(id: u32) void {
    std.debug.print("Thread {d} รอ semaphore\n", .{id});
    semaphore.wait();
    defer semaphore.post();
    
    std.debug.print("Thread {d} กำลังใช้ resource\n", .{id});
    std.time.sleep(200 * std.time.ns_per_ms);
    std.debug.print("Thread {d} ปล่อย resource\n", .{id});
}

pub fn main() !void {
    const NUM_THREADS = 8;
    var threads: [NUM_THREADS]Thread = undefined;
    
    var i: u32 = 0;
    while (i < NUM_THREADS) : (i += 1) {
        threads[i] = try Thread.spawn(.{}, accessResource, .{i});
    }
    
    for (&threads) |*t| {
        t.join();
    }
}
```

### Connection Pool ด้วย Semaphore

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Semaphore = Thread.Semaphore;

const ConnectionPool = struct {
    semaphore: Semaphore,
    mutex: Mutex = .{},
    connections: [10]bool = [_]bool{false} ** 10,
    
    pub fn init(max_connections: u32) ConnectionPool {
        return .{ .semaphore = .{ .permits = max_connections } };
    }
    
    pub fn acquire(self: *ConnectionPool) usize {
        self.semaphore.wait();
        
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (&self.connections, 0..) |*conn, i| {
            if (!conn.*) {
                conn.* = true;
                return i;
            }
        }
        
        unreachable; // semaphore รับประกัน
    }
    
    pub fn release(self: *ConnectionPool, id: usize) void {
        self.mutex.lock();
        self.connections[id] = false;
        self.mutex.unlock();
        
        self.semaphore.post();
    }
};

fn useConnection(pool: *ConnectionPool, task_id: u32) void {
    const conn_id = pool.acquire();
    defer pool.release(conn_id);
    
    std.debug.print("Task {d} ใช้ connection {d}\n", .{ task_id, conn_id });
    std.time.sleep(100 * std.time.ns_per_ms);
    std.debug.print("Task {d} ปล่อย connection {d}\n", .{ task_id, conn_id });
}

pub fn main() !void {
    var pool = ConnectionPool.init(3); // สูงสุด 3 connections
    
    var threads: [10]Thread = undefined;
    var i: u32 = 0;
    while (i < 10) : (i += 1) {
        threads[i] = try Thread.spawn(.{}, useConnection, .{ &pool, i });
    }
    
    for (&threads) |*t| {
        t.join();
    }
}
```

---

## Step 457: Atomic operations - std.atomic

Atomic operations ทำงานแบบ lock-free และมีประสิทธิภาพสูงกว่า mutex สำหรับการดำเนินการอย่างง่าย

### พื้นฐาน Atomic

```zig
const std = @import("std");
const Thread = std.Thread;
const Atomic = std.atomic.Value;

pub fn main() !void {
    var counter = Atomic(u64).init(0);
    
    // Atomic load
    const val = counter.load(.acquire);
    std.debug.print("Current value: {d}\n", .{val});
    
    // Atomic store
    counter.store(42, .release);
    
    // Atomic add
    const old = counter.fetchAdd(1, .acq_rel);
    std.debug.print("Old: {d}, New: {d}\n", .{ old, counter.load(.acquire) });
    
    // Compare and swap
    const result = counter.cmpxchgStrong(43, 100, .acq_rel, .acquire);
    if (result == null) {
        std.debug.print("CAS สำเร็จ: {d}\n", .{counter.load(.acquire)});
    } else |old_val| {
        std.debug.print("CAS ล้มเหลว old={d}\n", .{old_val});
    }
}
```

### Atomic Counter

```zig
const std = @import("std");
const Thread = std.Thread;
const Atomic = std.atomic.Value;

var atomic_counter = Atomic(u64).init(0);

fn atomicIncrement() void {
    var i: u32 = 0;
    while (i < 100000) : (i += 1) {
        _ = atomic_counter.fetchAdd(1, .monotonic);
    }
}

pub fn main() !void {
    const NUM_THREADS = 4;
    var threads: [NUM_THREADS]Thread = undefined;
    
    for (&threads) |*t| {
        t.* = try Thread.spawn(.{}, atomicIncrement, .{});
    }
    
    for (&threads) |*t| {
        t.join();
    }
    
    std.debug.print("Final: {d} (ควรเป็น {d})\n", .{
        atomic_counter.load(.acquire),
        @as(u64, 100000) * NUM_THREADS,
    });
}
```

### Memory Ordering

```zig
const std = @import("std");
const Atomic = std.atomic.Value;

// Memory ordering ต่างๆ:
// .unordered  - ไม่มีการรับประกัน ordering
// .monotonic  - ไม่มี sync แต่ atomic
// .acquire    - operations หลังจากนี้ไม่ถูก reorder ขึ้นมาก่อน
// .release    - operations ก่อนหน้านี้ไม่ถูก reorder ลงมาหลัง
// .acq_rel    - ทั้ง acquire และ release
// .seq_cst    - sequential consistency (แข็งแกร่งที่สุด)

var flag = Atomic(bool).init(false);
var data: u32 = 0;

fn producer() void {
    data = 42;                    // เขียนข้อมูล
    flag.store(true, .release);   // ประกาศว่าข้อมูลพร้อม
}

fn consumer() void {
    while (!flag.load(.acquire)) {} // รอจนกว่าจะพร้อม
    // ข้อมูลรับประกันว่าถูก publish แล้ว
    std.debug.print("ได้รับ data: {d}\n", .{data});
}

pub fn main() !void {
    const t1 = try Thread.spawn(.{}, producer, .{});
    const t2 = try Thread.spawn(.{}, consumer, .{});
    t1.join();
    t2.join();
}
```

---

## Step 458: Thread-safe data structures

การสร้าง data structures ที่ปลอดภัยสำหรับ multi-threaded environment

### Thread-safe Queue

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Condition = Thread.Condition;

fn ThreadSafeQueue(comptime T: type) type {
    return struct {
        const Self = @This();
        const Node = struct {
            value: T,
            next: ?*Node = null,
        };
        
        mutex: Mutex = .{},
        not_empty: Condition = .{},
        head: ?*Node = null,
        tail: ?*Node = null,
        len: usize = 0,
        allocator: std.mem.Allocator,
        
        pub fn init(allocator: std.mem.Allocator) Self {
            return .{ .allocator = allocator };
        }
        
        pub fn push(self: *Self, value: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .value = value };
            
            self.mutex.lock();
            defer self.mutex.unlock();
            
            if (self.tail) |tail| {
                tail.next = node;
            } else {
                self.head = node;
            }
            self.tail = node;
            self.len += 1;
            
            self.not_empty.signal();
        }
        
        pub fn pop(self: *Self) ?T {
            self.mutex.lock();
            defer self.mutex.unlock();
            
            const node = self.head orelse return null;
            self.head = node.next;
            if (self.head == null) self.tail = null;
            self.len -= 1;
            
            const value = node.value;
            self.allocator.destroy(node);
            return value;
        }
        
        pub fn popWait(self: *Self) T {
            self.mutex.lock();
            defer self.mutex.unlock();
            
            while (self.head == null) {
                self.not_empty.wait(&self.mutex);
            }
            
            const node = self.head.?;
            self.head = node.next;
            if (self.head == null) self.tail = null;
            self.len -= 1;
            
            const value = node.value;
            self.allocator.destroy(node);
            return value;
        }
        
        pub fn getLen(self: *Self) usize {
            self.mutex.lock();
            defer self.mutex.unlock();
            return self.len;
        }
    };
}

const IntQueue = ThreadSafeQueue(u32);

fn producer(queue: *IntQueue) !void {
    var i: u32 = 0;
    while (i < 10) : (i += 1) {
        try queue.push(i * 10);
        std.debug.print("Produced: {d}\n", .{i * 10});
        std.time.sleep(10 * std.time.ns_per_ms);
    }
}

fn consumer(queue: *IntQueue) void {
    var count: u32 = 0;
    while (count < 10) : (count += 1) {
        const val = queue.popWait();
        std.debug.print("Consumed: {d}\n", .{val});
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var queue = IntQueue.init(allocator);
    
    const p = try Thread.spawn(.{}, producer, .{&queue});
    const c = try Thread.spawn(.{}, consumer, .{&queue});
    
    p.join();
    c.join();
}
```

---

## Step 459: Thread pool

Thread pool ช่วยจัดการ threads อย่างมีประสิทธิภาพโดยสร้าง thread ไว้ล่วงหน้า

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Condition = Thread.Condition;

const Task = struct {
    func: *const fn (*anyopaque) void,
    data: *anyopaque,
};

const ThreadPool = struct {
    threads: []Thread,
    task_queue: std.ArrayList(Task),
    mutex: Mutex = .{},
    work_available: Condition = .{},
    shutdown: bool = false,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, num_threads: usize) !ThreadPool {
        var pool = ThreadPool{
            .threads = try allocator.alloc(Thread, num_threads),
            .task_queue = std.ArrayList(Task).init(allocator),
            .allocator = allocator,
        };
        
        for (pool.threads) |*t| {
            t.* = try Thread.spawn(.{}, workerLoop, .{&pool});
        }
        
        return pool;
    }
    
    pub fn deinit(self: *ThreadPool) void {
        self.mutex.lock();
        self.shutdown = true;
        self.mutex.unlock();
        self.work_available.broadcast();
        
        for (self.threads) |t| {
            t.join();
        }
        
        self.allocator.free(self.threads);
        self.task_queue.deinit();
    }
    
    pub fn submit(self: *ThreadPool, task: Task) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        try self.task_queue.append(task);
        self.work_available.signal();
    }
    
    fn workerLoop(pool: *ThreadPool) void {
        while (true) {
            pool.mutex.lock();
            
            while (pool.task_queue.items.len == 0 and !pool.shutdown) {
                pool.work_available.wait(&pool.mutex);
            }
            
            if (pool.shutdown and pool.task_queue.items.len == 0) {
                pool.mutex.unlock();
                return;
            }
            
            const task = pool.task_queue.orderedRemove(0);
            pool.mutex.unlock();
            
            task.func(task.data);
        }
    }
};

const WorkItem = struct {
    id: u32,
    value: u64,
    result: u64 = 0,
};

fn computeTask(data: *anyopaque) void {
    const item: *WorkItem = @ptrCast(@alignCast(data));
    
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < item.value) : (i += 1) {
        sum += i;
    }
    item.result = sum;
    
    std.debug.print("Task {d}: sum(0..{d}) = {d}\n", .{ item.id, item.value, sum });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var pool = try ThreadPool.init(allocator, 4);
    defer pool.deinit();
    
    var work_items: [8]WorkItem = undefined;
    var i: u32 = 0;
    while (i < 8) : (i += 1) {
        work_items[i] = .{ .id = i, .value = (i + 1) * 100000 };
        try pool.submit(.{
            .func = computeTask,
            .data = &work_items[i],
        });
    }
    
    std.time.sleep(2 * std.time.ns_per_s);
    std.debug.print("ทุก task เสร็จแล้ว\n", .{});
}
```

---

## Step 460: Real-world multithreading patterns

### Producer-Consumer Pattern

```zig
const std = @import("std");
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Condition = Thread.Condition;

const BUFFER_SIZE = 10;

const ProducerConsumer = struct {
    buffer: [BUFFER_SIZE]u32 = undefined,
    head: usize = 0,
    tail: usize = 0,
    count: usize = 0,
    mutex: Mutex = .{},
    not_full: Condition = .{},
    not_empty: Condition = .{},
    
    pub fn produce(self: *ProducerConsumer, value: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        while (self.count == BUFFER_SIZE) {
            self.not_full.wait(&self.mutex);
        }
        
        self.buffer[self.tail] = value;
        self.tail = (self.tail + 1) % BUFFER_SIZE;
        self.count += 1;
        
        self.not_empty.signal();
    }
    
    pub fn consume(self: *ProducerConsumer) u32 {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        while (self.count == 0) {
            self.not_empty.wait(&self.mutex);
        }
        
        const value = self.buffer[self.head];
        self.head = (self.head + 1) % BUFFER_SIZE;
        self.count -= 1;
        
        self.not_full.signal();
        return value;
    }
};

var shared_pc = ProducerConsumer{};
var done = std.atomic.Value(bool).init(false);

fn producerFn() void {
    var i: u32 = 0;
    while (i < 50) : (i += 1) {
        shared_pc.produce(i);
        std.debug.print("Produced: {d}\n", .{i});
    }
    done.store(true, .release);
}

fn consumerFn() void {
    while (true) {
        if (done.load(.acquire) and shared_pc.count == 0) break;
        
        if (shared_pc.count > 0) {
            const val = shared_pc.consume();
            std.debug.print("Consumed: {d}\n", .{val});
        }
    }
}

pub fn main() !void {
    const p = try Thread.spawn(.{}, producerFn, .{});
    const c = try Thread.spawn(.{}, consumerFn, .{});
    
    p.join();
    c.join();
    
    std.debug.print("Producer-Consumer เสร็จสิ้น\n", .{});
}
```

### Parallel Map Pattern

```zig
const std = @import("std");
const Thread = std.Thread;

fn parallelMap(
    comptime T: type,
    comptime R: type,
    input: []const T,
    output: []R,
    func: *const fn (T) R,
    allocator: std.mem.Allocator,
) !void {
    const cpu_count = try Thread.getCpuCount();
    const num_threads = @min(cpu_count, input.len);
    const chunk_size = (input.len + num_threads - 1) / num_threads;
    
    const threads = try allocator.alloc(Thread, num_threads);
    defer allocator.free(threads);
    
    const Context = struct {
        input: []const T,
        output: []R,
        func: *const fn (T) R,
        start: usize,
        end: usize,
        
        fn run(ctx: @This()) void {
            var i = ctx.start;
            while (i < ctx.end) : (i += 1) {
                ctx.output[i] = ctx.func(ctx.input[i]);
            }
        }
    };
    
    const contexts = try allocator.alloc(Context, num_threads);
    defer allocator.free(contexts);
    
    for (threads, 0..) |*t, i| {
        const start = i * chunk_size;
        const end = @min(start + chunk_size, input.len);
        contexts[i] = .{
            .input = input,
            .output = output,
            .func = func,
            .start = start,
            .end = end,
        };
        t.* = try Thread.spawn(.{}, Context.run, .{contexts[i]});
    }
    
    for (threads) |t| {
        t.join();
    }
}

fn square(x: u64) u64 {
    return x * x;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const input = [_]u64{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    var output: [10]u64 = undefined;
    
    try parallelMap(u64, u64, &input, &output, square, allocator);
    
    std.debug.print("Input: {any}\n", .{input});
    std.debug.print("Output (squared): {any}\n", .{output});
}
```

---

## สรุป Part 46

ในบทนี้เราได้เรียนรู้:

| Topic | การใช้งาน |
|-------|-----------|
| `std.Thread` | สร้างและจัดการ threads |
| `Thread.spawn` | สร้าง thread ใหม่ |
| `Thread.join` | รอ thread ทำงานเสร็จ |
| `Mutex` | ป้องกัน race condition |
| `RwLock` | อนุญาต multiple readers |
| `Semaphore` | จำกัดจำนวน concurrent access |
| `std.atomic` | Lock-free operations |
| Thread-safe structures | Queue, stack ที่ปลอดภัย |
| Thread pool | จัดการ threads อย่างมีประสิทธิภาพ |
| Patterns | Producer-Consumer, Parallel Map |

### ข้อควรระวัง

1. **Deadlock**: อย่า lock mutex ซ้อนกันโดยไม่ระวัง
2. **Race condition**: ใช้ synchronization เสมอเมื่อแชร์ข้อมูล
3. **Memory ordering**: เลือก ordering ที่เหมาะสมสำหรับ atomic operations
4. **Over-synchronization**: Mutex มาก เกินไปทำให้ช้าลง

ใน Part ถัดไปเราจะเรียนรู้เรื่อง Networking พื้นฐานใน Zig!
