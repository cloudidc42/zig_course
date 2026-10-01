# Part 45: Async ใน Zig

> **หลักสูตร Zig ภาษาไทย** | Steps 441-450

---

## Step 441: Async/Await concept ใน Zig

> **⚠️ หมายเหตุสำคัญ**: Async/Await ใน Zig ยังอยู่ในระหว่างการพัฒนาใหม่ใน Zig 0.14+
> ฟีเจอร์ที่อธิบายในบทนี้บางส่วนอาจเปลี่ยนแปลงในอนาคต

### Async คืออะไร?

Async programming ช่วยให้โปรแกรมทำงานได้โดยไม่ต้อง block thread เมื่อรอ I/O

```
Synchronous (Blocking):
Thread 1: [งาน A] ──wait─── [งาน B] ──wait─── [งาน C]
                   ↑ block!         ↑ block!
Thread ว่างขณะรอ

Asynchronous (Non-blocking):
Thread 1: [เริ่ม A] → [เริ่ม B] → [เริ่ม C] → [ทำ A เสร็จ] → [ทำ B เสร็จ] → ...
          ↓                                     ↑
          รอ I/O                                ได้รับ callback
```

### ทำไม Async ถึงสำคัญ?

```
สถานการณ์: Web server รับ 1000 requests พร้อมกัน

แบบ Synchronous (1 thread per request):
- ต้องใช้ 1000 threads
- แต่ละ thread ใช้ ~8MB stack
- รวม 8GB RAM แค่ threads!
- Context switching overhead มาก

แบบ Async (event loop):
- ใช้ไม่กี่ threads (หรือแค่ 1)
- I/O operations ไม่ block
- รับ 1000 requests ได้ด้วย RAM เพียงเล็กน้อย
- Scale ได้ดีกว่ามาก
```

### Async Models ต่างๆ

| Model | ตัวอย่าง | ลักษณะ |
|-------|---------|---------|
| Callbacks | Node.js (เดิม) | ซับซ้อน, Callback hell |
| Promises | JavaScript | ดีกว่า callbacks |
| Async/Await | Python, C#, Rust | อ่านง่าย |
| Coroutines | Lua, Python generators | Cooperative multitasking |
| Green threads | Erlang, Go goroutines | OS-independent threads |
| Zig async | Zig | Stackless coroutines |

### Zig Async Model

Zig ใช้ **stackless coroutines** ซึ่งแตกต่างจาก stackful (green threads):

```
Stackful Coroutine (เช่น goroutines):
- แต่ละ coroutine มี stack ของตัวเอง (~2-8KB minimum)
- OS scheduler จัดการ
- ใช้ RAM มากกว่า

Stackless Coroutine (Zig async):
- ไม่มี dedicated stack
- State เก็บใน "frame" บน heap
- Suspend/Resume ด้วย compiler magic
- ใช้ RAM น้อยกว่า
```

---

## Step 442: async function

> **หมายเหตุ**: ใน Zig 0.12+ async syntax กำลังถูก redesign ตัวอย่างด้านล่างแสดง concept และ syntax เดิม พร้อมทางเลือกที่ใช้ได้ในปัจจุบัน

### Syntax เดิม (Zig 0.10 และก่อนหน้า)

```zig
// async function ที่สามารถ suspend ได้
const std = @import("std");

// ฟังก์ชัน async ธรรมดา
fn fetchData() ![]const u8 {
    // simulate I/O delay
    return "data from server";
}

// async function ที่ suspend และ resume
fn asyncTask() void {
    // ฟังก์ชันนี้สามารถ suspend ระหว่างการทำงาน
    std.debug.print("Task started\n", .{});
    // ... ทำงานบางอย่าง ...
    std.debug.print("Task finished\n", .{});
}
```

### Async Functions ในสไตล์ปัจจุบัน

```zig
// Zig 0.14+ - ใช้ threads หรือ event loop แทน
const std = @import("std");

// ฟังก์ชันที่ทำงานแบบ non-blocking ด้วย threads
fn workerTask(context: *WorkContext) void {
    std.debug.print("Worker {}: started\n", .{context.id});

    // simulate work
    std.time.sleep(context.delay_ms * std.time.ns_per_ms);

    context.result = context.id * 2;
    std.debug.print("Worker {}: done, result={}\n", .{ context.id, context.result });
}

const WorkContext = struct {
    id: usize,
    delay_ms: u64,
    result: usize = 0,
};

pub fn main() !void {
    var ctx1 = WorkContext{ .id = 1, .delay_ms = 100 };
    var ctx2 = WorkContext{ .id = 2, .delay_ms = 50 };
    var ctx3 = WorkContext{ .id = 3, .delay_ms = 75 };

    // สร้าง threads
    const t1 = try std.Thread.spawn(.{}, workerTask, .{&ctx1});
    const t2 = try std.Thread.spawn(.{}, workerTask, .{&ctx2});
    const t3 = try std.Thread.spawn(.{}, workerTask, .{&ctx3});

    // รอทุก threads
    t1.join();
    t2.join();
    t3.join();

    std.debug.print("Results: {}, {}, {}\n", .{
        ctx1.result,
        ctx2.result,
        ctx3.result,
    });
}
```

### Async Pattern ด้วย Callbacks (แบบ C-style)

```zig
const std = @import("std");

// Callback type
const TaskCallback = *const fn (result: i32, user_data: ?*anyopaque) void;

// Async task ที่ใช้ callback
fn asyncCompute(input: i32, callback: TaskCallback, user_data: ?*anyopaque) void {
    // simulate async work
    const result = input * input;
    callback(result, user_data);
}

fn onResult(result: i32, user_data: ?*anyopaque) void {
    _ = user_data;
    std.debug.print("Result: {}\n", .{result});
}

pub fn main() void {
    asyncCompute(5, onResult, null);
    asyncCompute(10, onResult, null);
}
```

---

## Step 443: await expression

### Concept ของ await

`await` ใน async programming หมายถึงการ "รอ" ผลลัพธ์จาก async operation โดยไม่ block thread

```
โดยไม่ใช้ async/await (blocking):
fn main() void {
    const data = fetchFromNetwork();  // block จนกว่าจะได้ข้อมูล
    processData(data);
}

ด้วย async/await (non-blocking):
fn main() async void {
    const data = await fetchFromNetwork();  // yield thread, resume เมื่อได้ข้อมูล
    processData(data);
}
```

### await ใน Zig (แบบดั้งเดิม)

```zig
// Zig async/await syntax (เวอร์ชันดั้งเดิม)
const std = @import("std");

fn expensiveCalculation(n: u64) u64 {
    var result: u64 = 0;
    for (0..n) |i| result += i;
    return result;
}

// ใช้ async frame
fn asyncSum(n: u64) u64 {
    return expensiveCalculation(n);
}

pub fn main() void {
    // เรียกแบบ sync
    const result = asyncSum(1000);
    std.debug.print("Sum: {}\n", .{result});
}
```

### Future/Promise Pattern ใน Zig

```zig
const std = @import("std");

// Simple Future implementation
fn Future(comptime T: type) type {
    return struct {
        const Self = @This();

        value: ?T = null,
        error_val: ?anyerror = null,
        completed: bool = false,
        mutex: std.Thread.Mutex = .{},
        cond: std.Thread.Condition = .{},

        pub fn setValue(self: *Self, val: T) void {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.value = val;
            self.completed = true;
            self.cond.signal();
        }

        pub fn setError(self: *Self, err: anyerror) void {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.error_val = err;
            self.completed = true;
            self.cond.signal();
        }

        pub fn wait(self: *Self) !T {
            self.mutex.lock();
            defer self.mutex.unlock();

            while (!self.completed) {
                self.cond.wait(&self.mutex);
            }

            if (self.error_val) |err| return err;
            return self.value.?;
        }
    };
}

// ตัวอย่างการใช้ Future
const IntFuture = Future(i64);

fn computeInBackground(future: *IntFuture, input: i64) void {
    // Simulate heavy computation
    std.time.sleep(100 * std.time.ns_per_ms);

    if (input < 0) {
        future.setError(error.NegativeInput);
    } else {
        future.setValue(input * input);
    }
}

pub fn futureExample() !void {
    var future = IntFuture{};

    const thread = try std.Thread.spawn(.{}, computeInBackground, .{ &future, 42 });
    defer thread.join();

    std.debug.print("Waiting for result...\n", .{});
    const result = try future.wait();
    std.debug.print("Result: {}\n", .{result});
}

pub fn main() !void {
    try futureExample();
}
```

---

## Step 444: async frame

### Concept ของ async frame

**async frame** คือ struct ที่เก็บ state ของ async function ระหว่างที่มัน suspend

```
async function → frame struct
─────────────────────────────
fn foo() async void {
    var x: i32 = 1;     ─┐
    suspend;             │ เก็บใน frame:
    var y: i32 = 2;     ─┤ { x: i32, y: i32, pc: usize, ... }
    suspend;             │
    return x + y;       ─┘
}
```

### Frame ขนาด

```zig
// ดูขนาด frame ของ async function
const std = @import("std");

fn myAsyncFn(x: i32) i32 {
    var result: i32 = x * 2;
    return result;
}

pub fn main() void {
    // ขนาด frame สะท้อนถึง variables และ state ที่ต้องเก็บ
    const FrameType = @Frame(myAsyncFn);
    std.debug.print("Frame size: {} bytes\n", .{@sizeOf(FrameType)});
}
```

### Manual Frame Management

```zig
const std = @import("std");

// Stack-allocated frame
fn stackFrameExample() void {
    // Zig สามารถ allocate frame บน stack
    var frame: @Frame(someAsyncWork) = undefined;

    // เริ่ม async function
    _ = async someAsyncWork(&frame);

    // Resume
    resume frame;
}

fn someAsyncWork() void {
    std.debug.print("Working...\n", .{});
    suspend {}  // รอ
    std.debug.print("Resumed!\n", .{});
}
```

### Context Propagation Pattern

```zig
const std = @import("std");

// Context ที่ส่งผ่าน async chain
const Context = struct {
    request_id: u64,
    deadline: i64,  // Unix timestamp
    allocator: std.mem.Allocator,

    pub fn isExpired(self: Context) bool {
        return std.time.timestamp() > self.deadline;
    }
};

fn processRequest(ctx: Context, data: []const u8) ![]u8 {
    if (ctx.isExpired()) return error.DeadlineExceeded;

    // Process data
    const result = try ctx.allocator.dupe(u8, data);
    return result;
}

fn handleRequest(ctx: Context) !void {
    if (ctx.isExpired()) {
        std.debug.print("Request {} expired\n", .{ctx.request_id});
        return error.DeadlineExceeded;
    }

    const data = "Hello, World!";
    const result = try processRequest(ctx, data);
    defer ctx.allocator.free(result);

    std.debug.print("Request {}: {s}\n", .{ ctx.request_id, result });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    const ctx = Context{
        .request_id = 12345,
        .deadline = std.time.timestamp() + 30,  // 30 วินาที
        .allocator = gpa.allocator(),
    };

    try handleRequest(ctx);
}
```

---

## Step 445: Event loops

### Event Loop คืออะไร?

Event loop คือ loop หลักที่รอและประมวลผล events (I/O, timers, callbacks)

```
Event Loop:
┌─────────────────────────────────────────┐
│  while (true) {                         │
│    events = poll_for_events();          │
│    for (event in events) {              │
│      handler = dispatch(event);         │
│      handler.execute();                 │
│    }                                    │
│  }                                      │
└─────────────────────────────────────────┘
```

### Simple Event Loop

```zig
const std = @import("std");

// Event types
pub const EventKind = enum {
    timer,
    io_ready,
    custom,
};

pub const Event = struct {
    kind: EventKind,
    data: ?*anyopaque,
    handler: *const fn (event: *Event) void,
};

// Simple event loop
pub const EventLoop = struct {
    queue: std.ArrayList(Event),
    running: bool = false,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) EventLoop {
        return .{
            .queue = std.ArrayList(Event).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *EventLoop) void {
        self.queue.deinit();
    }

    pub fn post(self: *EventLoop, event: Event) !void {
        try self.queue.append(event);
    }

    pub fn run(self: *EventLoop) void {
        self.running = true;

        while (self.running and self.queue.items.len > 0) {
            // ดึง event แรก
            const event = self.queue.orderedRemove(0);

            // ประมวลผล
            var ev = event;
            ev.handler(&ev);
        }

        self.running = false;
    }

    pub fn stop(self: *EventLoop) void {
        self.running = false;
    }
};

// ตัวอย่าง handlers
fn onTimer(event: *Event) void {
    _ = event;
    std.debug.print("Timer fired!\n", .{});
}

fn onIoReady(event: *Event) void {
    _ = event;
    std.debug.print("I/O ready!\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var loop = EventLoop.init(allocator);
    defer loop.deinit();

    // Post events
    try loop.post(.{ .kind = .timer, .data = null, .handler = onTimer });
    try loop.post(.{ .kind = .io_ready, .data = null, .handler = onIoReady });
    try loop.post(.{ .kind = .timer, .data = null, .handler = onTimer });

    // Run event loop
    loop.run();
}
```

### I/O Event Loop ด้วย epoll (Linux)

```zig
const std = @import("std");
const linux = std.os.linux;

// Simple epoll-based event loop
pub const IOLoop = struct {
    epoll_fd: std.posix.fd_t,
    events: [64]linux.epoll_event,

    pub fn init() !IOLoop {
        const epoll_fd = try std.posix.epoll_create1(0);
        return .{
            .epoll_fd = epoll_fd,
            .events = undefined,
        };
    }

    pub fn deinit(self: *IOLoop) void {
        std.posix.close(self.epoll_fd);
    }

    pub fn addFd(self: *IOLoop, fd: std.posix.fd_t, events: u32) !void {
        var ev = linux.epoll_event{
            .events = events,
            .data = .{ .fd = fd },
        };
        try std.posix.epoll_ctl(self.epoll_fd, linux.EPOLL.CTL_ADD, fd, &ev);
    }

    pub fn wait(self: *IOLoop, timeout_ms: i32) ![]linux.epoll_event {
        const n = std.posix.epoll_wait(
            self.epoll_fd,
            &self.events,
            timeout_ms,
        );
        return self.events[0..n];
    }
};

// ตัวอย่างการใช้ I/O loop
pub fn ioLoopExample() !void {
    var loop = try IOLoop.init();
    defer loop.deinit();

    // Monitor stdin
    try loop.addFd(0, linux.EPOLL.IN);

    std.debug.print("Waiting for input (5 seconds timeout)...\n", .{});

    const ready_events = try loop.wait(5000);  // 5 second timeout

    if (ready_events.len == 0) {
        std.debug.print("Timeout!\n", .{});
    } else {
        for (ready_events) |ev| {
            std.debug.print("fd {} is ready (events={})\n", .{ ev.data.fd, ev.events });
        }
    }
}
```

---

## Step 446: Coroutines

### Coroutines คืออะไร?

Coroutines คือฟังก์ชันที่สามารถ pause และ resume การทำงานได้

```
Normal function:
main() → foo() → return
          ↑ runs to completion

Coroutine:
main() → coroutine() → suspend → resume → suspend → resume → done
          ↑ can pause              ↑ continues from where it paused
```

### Coroutine Pattern ด้วย Thread

```zig
const std = @import("std");

// Coroutine-like behavior ด้วย channels
pub fn Channel(comptime T: type) type {
    return struct {
        const Self = @This();

        mutex: std.Thread.Mutex = .{},
        has_value: std.Thread.Condition = .{},
        consumed: std.Thread.Condition = .{},
        value: ?T = null,
        closed: bool = false,

        pub fn send(self: *Self, val: T) void {
            self.mutex.lock();
            defer self.mutex.unlock();

            // รอถ้า channel ยังมีค่าอยู่
            while (self.value != null and !self.closed) {
                self.consumed.wait(&self.mutex);
            }

            if (!self.closed) {
                self.value = val;
                self.has_value.signal();
            }
        }

        pub fn recv(self: *Self) ?T {
            self.mutex.lock();
            defer self.mutex.unlock();

            // รอจนมีค่า
            while (self.value == null and !self.closed) {
                self.has_value.wait(&self.mutex);
            }

            const val = self.value;
            self.value = null;
            self.consumed.signal();
            return val;
        }

        pub fn close(self: *Self) void {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.closed = true;
            self.has_value.broadcast();
            self.consumed.broadcast();
        }
    };
}

// Generator coroutine pattern
const IntChannel = Channel(i64);

fn fibonacci_generator(ch: *IntChannel) void {
    var a: i64 = 0;
    var b: i64 = 1;

    for (0..20) |_| {
        ch.send(a);
        const next = a + b;
        a = b;
        b = next;
    }

    ch.close();
}

pub fn main() !void {
    var ch = IntChannel{};

    const thread = try std.Thread.spawn(.{}, fibonacci_generator, .{&ch});
    defer thread.join();

    std.debug.print("Fibonacci sequence:\n", .{});
    while (ch.recv()) |n| {
        std.debug.print("{} ", .{n});
    }
    std.debug.print("\n", .{});
}
```

### Producer-Consumer Pattern

```zig
const std = @import("std");

// Thread-safe queue
pub fn Queue(comptime T: type) type {
    return struct {
        const Self = @This();

        items: std.ArrayList(T),
        mutex: std.Thread.Mutex = .{},
        not_empty: std.Thread.Condition = .{},
        not_full: std.Thread.Condition = .{},
        max_size: usize,
        closed: bool = false,

        pub fn init(allocator: std.mem.Allocator, max_size: usize) Self {
            return .{
                .items = std.ArrayList(T).init(allocator),
                .max_size = max_size,
            };
        }

        pub fn deinit(self: *Self) void {
            self.items.deinit();
        }

        pub fn push(self: *Self, item: T) !void {
            self.mutex.lock();
            defer self.mutex.unlock();

            while (self.items.items.len >= self.max_size and !self.closed) {
                self.not_full.wait(&self.mutex);
            }

            if (self.closed) return error.QueueClosed;

            try self.items.append(item);
            self.not_empty.signal();
        }

        pub fn pop(self: *Self) ?T {
            self.mutex.lock();
            defer self.mutex.unlock();

            while (self.items.items.len == 0 and !self.closed) {
                self.not_empty.wait(&self.mutex);
            }

            if (self.items.items.len == 0) return null;

            const item = self.items.orderedRemove(0);
            self.not_full.signal();
            return item;
        }

        pub fn close(self: *Self) void {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.closed = true;
            self.not_empty.broadcast();
            self.not_full.broadcast();
        }
    };
}

const WorkQueue = Queue(u32);

fn producer(queue: *WorkQueue) !void {
    defer queue.close();

    for (0..10) |i| {
        const val: u32 = @intCast(i);
        try queue.push(val);
        std.debug.print("Produced: {}\n", .{val});
        std.time.sleep(10 * std.time.ns_per_ms);
    }
}

fn consumer(id: usize, queue: *WorkQueue) void {
    while (queue.pop()) |val| {
        std.debug.print("Consumer {}: got {}\n", .{ id, val });
        std.time.sleep(50 * std.time.ns_per_ms);
    }
    std.debug.print("Consumer {}: done\n", .{id});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var queue = WorkQueue.init(gpa.allocator(), 5);
    defer queue.deinit();

    // Start consumers
    const c1 = try std.Thread.spawn(.{}, consumer, .{ 1, &queue });
    const c2 = try std.Thread.spawn(.{}, consumer, .{ 2, &queue });

    // Produce items
    try producer(&queue);

    c1.join();
    c2.join();
}
```

---

## Step 447: async กับ I/O

### Async I/O ด้วย Non-blocking Sockets

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

// Non-blocking TCP server
pub fn startNonBlockingServer(port: u16) !void {
    const addr = net.Address.initIp4(.{ 0, 0, 0, 0 }, port);

    const server = try net.Address.listen(addr, .{
        .reuse_address = true,
        .kernel_backlog = 128,
    });
    defer server.deinit();

    std.debug.print("Server listening on port {}\n", .{port});

    // Set non-blocking
    const server_fd = server.stream.handle;
    _ = try posix.fcntl(server_fd, posix.F.SETFL, posix.O.NONBLOCK);

    // Accept connections
    while (true) {
        const conn = server.accept() catch |err| switch (err) {
            error.WouldBlock => {
                // ไม่มี connection ใหม่ ทำงานอื่นก่อน
                std.time.sleep(1 * std.time.ns_per_ms);
                continue;
            },
            else => return err,
        };

        // Handle connection in a thread
        const thread = try std.Thread.spawn(.{}, handleConnection, .{conn});
        thread.detach();
    }
}

fn handleConnection(conn: net.Server.Connection) void {
    defer conn.stream.close();

    const writer = conn.stream.writer();
    const reader = conn.stream.reader();

    var buf: [1024]u8 = undefined;
    const n = reader.read(&buf) catch return;

    if (n == 0) return;

    const request = buf[0..n];
    std.debug.print("Received: {s}\n", .{request});

    // Send response
    writer.writeAll("HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nHello") catch return;
}
```

### Async File I/O

```zig
const std = @import("std");

// Async-like file reading ด้วย thread pool
pub const FileReadTask = struct {
    path: []const u8,
    result: ?[]u8 = null,
    err: ?anyerror = null,
    completed: bool = false,
    mutex: std.Thread.Mutex = .{},
    done: std.Thread.Condition = .{},

    pub fn read(self: *FileReadTask, allocator: std.mem.Allocator) void {
        const file = std.fs.cwd().openFile(self.path, .{}) catch |err| {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.err = err;
            self.completed = true;
            self.done.signal();
            return;
        };
        defer file.close();

        const content = file.readToEndAlloc(allocator, 1024 * 1024) catch |err| {
            self.mutex.lock();
            defer self.mutex.unlock();
            self.err = err;
            self.completed = true;
            self.done.signal();
            return;
        };

        self.mutex.lock();
        defer self.mutex.unlock();
        self.result = content;
        self.completed = true;
        self.done.signal();
    }

    pub fn await_result(self: *FileReadTask) ![]u8 {
        self.mutex.lock();
        defer self.mutex.unlock();

        while (!self.completed) {
            self.done.wait(&self.mutex);
        }

        if (self.err) |err| return err;
        return self.result.?;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // เริ่มอ่านหลายไฟล์พร้อมกัน
    var tasks = [_]FileReadTask{
        .{ .path = "/etc/hostname" },
        .{ .path = "/etc/os-release" },
    };

    // Start reading in background threads
    const threads = try allocator.alloc(std.Thread, tasks.len);
    defer allocator.free(threads);

    for (&tasks, 0..) |*task, i| {
        threads[i] = try std.Thread.spawn(.{}, FileReadTask.read, .{ task, allocator });
    }

    // Wait and print results
    for (&tasks, 0..) |*task, i| {
        threads[i].join();
        const content = task.await_result() catch |err| {
            std.debug.print("Error reading {s}: {}\n", .{ task.path, err });
            continue;
        };
        defer allocator.free(content);

        std.debug.print("File: {s}\n{s}\n---\n", .{ task.path, content });
    }
}
```

---

## Step 448: Error handling ใน async

### Error ใน Async Context

```zig
const std = @import("std");

// Error types สำหรับ async operations
pub const AsyncError = error{
    Timeout,
    ConnectionRefused,
    DeadlineExceeded,
    Cancelled,
    IoError,
    OutOfMemory,
};

// Result type สำหรับ async operations
pub fn AsyncResult(comptime T: type) type {
    return union(enum) {
        ok: T,
        err: anyerror,

        pub fn unwrap(self: @This()) !T {
            return switch (self) {
                .ok => |val| val,
                .err => |e| e,
            };
        }
    };
}

// Async task ที่อาจ fail
const StringResult = AsyncResult([]const u8);

fn fetchWithTimeout(url: []const u8, timeout_ms: u64) StringResult {
    _ = url;

    // simulate network request
    std.time.sleep(timeout_ms * std.time.ns_per_ms / 2);

    // สำเร็จ
    return .{ .ok = "response data" };
}

fn fetchWithFailure(url: []const u8) StringResult {
    _ = url;
    // จำลอง failure
    return .{ .err = AsyncError.ConnectionRefused };
}

pub fn main() !void {
    // สำเร็จ
    const result1 = fetchWithTimeout("http://example.com", 1000);
    const data = result1.unwrap() catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    std.debug.print("Got: {s}\n", .{data});

    // ล้มเหลว
    const result2 = fetchWithFailure("http://broken.com");
    _ = result2.unwrap() catch |err| {
        std.debug.print("Expected error: {}\n", .{err});
    };
}
```

### Error Propagation ใน Async Chain

```zig
const std = @import("std");

const AppError = error{
    DatabaseError,
    NetworkError,
    ValidationError,
    NotFound,
};

// Simulated async operations
fn validateInput(input: []const u8) AppError![]const u8 {
    if (input.len == 0) return AppError.ValidationError;
    return input;
}

fn fetchFromDb(id: []const u8) AppError![]const u8 {
    if (std.mem.eql(u8, id, "invalid")) return AppError.NotFound;
    return "user data from db";
}

fn sendResponse(data: []const u8) AppError!void {
    if (data.len > 1000) return AppError.NetworkError;
    std.debug.print("Response sent: {s}\n", .{data});
}

// Chain of async operations ด้วย error propagation
fn handleRequest(user_id: []const u8) AppError!void {
    // ทุก error จะ propagate ขึ้นมาโดยอัตโนมัติ
    const validated_id = try validateInput(user_id);
    const data = try fetchFromDb(validated_id);
    try sendResponse(data);
}

pub fn main() void {
    // สำเร็จ
    handleRequest("user123") catch |err| {
        std.debug.print("Error handling user123: {}\n", .{err});
        return;
    };

    // ล้มเหลว
    handleRequest("") catch |err| {
        std.debug.print("Error handling empty: {}\n", .{err});
    };

    handleRequest("invalid") catch |err| {
        std.debug.print("Error handling invalid: {}\n", .{err});
    };
}
```

### Retry Logic

```zig
const std = @import("std");

const RetryConfig = struct {
    max_attempts: u32 = 3,
    delay_ms: u64 = 100,
    backoff_multiplier: f64 = 2.0,
};

fn withRetry(
    comptime T: type,
    config: RetryConfig,
    operation: *const fn () anyerror!T,
) anyerror!T {
    var delay = config.delay_ms;
    var attempt: u32 = 0;

    while (attempt < config.max_attempts) : (attempt += 1) {
        if (operation()) |result| {
            return result;
        } else |err| {
            if (attempt + 1 >= config.max_attempts) return err;

            std.debug.print("Attempt {} failed: {}, retrying in {}ms...\n", .{
                attempt + 1,
                err,
                delay,
            });

            std.time.sleep(delay * std.time.ns_per_ms);
            delay = @intFromFloat(@as(f64, @floatFromInt(delay)) * config.backoff_multiplier);
        }
    }

    return error.MaxRetriesExceeded;
}

var attempt_count: u32 = 0;

fn unreliableOperation() !u32 {
    attempt_count += 1;
    if (attempt_count < 3) return error.TemporaryFailure;
    return 42;
}

pub fn main() !void {
    attempt_count = 0;

    const result = try withRetry(u32, .{
        .max_attempts = 5,
        .delay_ms = 10,
    }, unreliableOperation);

    std.debug.print("Success after {} attempts: {}\n", .{ attempt_count, result });
}
```

---

## Step 449: Async patterns

### Pattern 1: Fan-Out / Fan-In

```zig
const std = @import("std");

// Fan-out: ส่งงานไปหลาย workers
// Fan-in: รวบรวมผลลัพธ์กลับมา

const WorkItem = struct {
    input: u64,
    result: u64 = 0,
};

fn worker(item: *WorkItem) void {
    // Simulate computation
    var result: u64 = 0;
    for (0..item.input) |i| result += i;
    item.result = result;
}

pub fn fanOutFanIn(inputs: []u64, allocator: std.mem.Allocator) ![]u64 {
    // สร้าง work items
    const items = try allocator.alloc(WorkItem, inputs.len);
    defer allocator.free(items);

    for (inputs, items) |input, *item| {
        item.* = .{ .input = input };
    }

    // Fan-out: สร้าง thread สำหรับแต่ละ item
    const threads = try allocator.alloc(std.Thread, inputs.len);
    defer allocator.free(threads);

    for (items, threads) |*item, *thread| {
        thread.* = try std.Thread.spawn(.{}, worker, .{item});
    }

    // Fan-in: รวบรวมผลลัพธ์
    for (threads) |thread| {
        thread.join();
    }

    // สร้าง results
    const results = try allocator.alloc(u64, inputs.len);
    for (items, results) |item, *result| {
        result.* = item.result;
    }

    return results;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const inputs = [_]u64{ 100, 200, 300, 400, 500 };
    const results = try fanOutFanIn(&inputs, allocator);
    defer allocator.free(results);

    for (inputs, results) |input, result| {
        std.debug.print("sum(0..{}) = {}\n", .{ input, result });
    }
}
```

### Pattern 2: Pipeline

```zig
const std = @import("std");

// Pipeline: ข้อมูลไหลผ่าน stages
// Stage 1: parse → Stage 2: validate → Stage 3: transform → Stage 4: output

const DataChannel = struct {
    items: std.ArrayList([]const u8),
    mutex: std.Thread.Mutex = .{},
    not_empty: std.Thread.Condition = .{},
    closed: bool = false,

    pub fn init(allocator: std.mem.Allocator) DataChannel {
        return .{ .items = std.ArrayList([]const u8).init(allocator) };
    }

    pub fn deinit(self: *DataChannel) void {
        self.items.deinit();
    }

    pub fn send(self: *DataChannel, item: []const u8) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        try self.items.append(item);
        self.not_empty.signal();
    }

    pub fn recv(self: *DataChannel) ?[]const u8 {
        self.mutex.lock();
        defer self.mutex.unlock();

        while (self.items.items.len == 0 and !self.closed) {
            self.not_empty.wait(&self.mutex);
        }

        if (self.items.items.len == 0) return null;
        return self.items.orderedRemove(0);
    }

    pub fn close(self: *DataChannel) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        self.closed = true;
        self.not_empty.broadcast();
    }
};

// Stage 1: Source
fn sourceStage(out: *DataChannel) !void {
    defer out.close();
    const items = [_][]const u8{ "hello", "world", "foo", "bar" };
    for (items) |item| {
        try out.send(item);
    }
}

// Stage 2: Transform (uppercase)
fn upperStage(in: *DataChannel, out: *DataChannel, allocator: std.mem.Allocator) !void {
    defer out.close();

    while (in.recv()) |item| {
        const upper = try std.ascii.allocUpperString(allocator, item);
        defer allocator.free(upper);
        try out.send(upper);  // ถ้าต้องการใช้ต่อต้อง duplicate
    }
}

// Stage 3: Sink (print)
fn sinkStage(in: *DataChannel) void {
    while (in.recv()) |item| {
        std.debug.print("Output: {s}\n", .{item});
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var ch1 = DataChannel.init(allocator);
    defer ch1.deinit();
    var ch2 = DataChannel.init(allocator);
    defer ch2.deinit();

    const t1 = try std.Thread.spawn(.{}, sourceStage, .{&ch1});
    const t2 = try std.Thread.spawn(.{}, upperStage, .{ &ch1, &ch2, allocator });
    const t3 = try std.Thread.spawn(.{}, sinkStage, .{&ch2});

    t1.join();
    t2.join();
    t3.join();
}
```

### Pattern 3: Worker Pool

```zig
const std = @import("std");

pub const Task = struct {
    id: usize,
    work: *const fn (id: usize) void,
};

pub const WorkerPool = struct {
    threads: []std.Thread,
    queue: std.ArrayList(Task),
    mutex: std.Thread.Mutex = .{},
    not_empty: std.Thread.Condition = .{},
    shutdown: bool = false,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, num_workers: usize) !WorkerPool {
        var pool = WorkerPool{
            .threads = try allocator.alloc(std.Thread, num_workers),
            .queue = std.ArrayList(Task).init(allocator),
            .allocator = allocator,
        };

        for (pool.threads) |*thread| {
            thread.* = try std.Thread.spawn(.{}, workerLoop, .{&pool});
        }

        return pool;
    }

    pub fn deinit(self: *WorkerPool) void {
        // Signal shutdown
        self.mutex.lock();
        self.shutdown = true;
        self.not_empty.broadcast();
        self.mutex.unlock();

        // Wait for all threads
        for (self.threads) |thread| thread.join();

        self.threads = &[_]std.Thread{};
        self.allocator.free(self.threads);
        self.queue.deinit();
    }

    pub fn submit(self: *WorkerPool, task: Task) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        try self.queue.append(task);
        self.not_empty.signal();
    }

    fn workerLoop(pool: *WorkerPool) void {
        while (true) {
            pool.mutex.lock();

            while (pool.queue.items.len == 0 and !pool.shutdown) {
                pool.not_empty.wait(&pool.mutex);
            }

            if (pool.shutdown and pool.queue.items.len == 0) {
                pool.mutex.unlock();
                break;
            }

            const task = pool.queue.orderedRemove(0);
            pool.mutex.unlock();

            // Execute task
            task.work(task.id);
        }
    }
};

fn exampleWork(id: usize) void {
    std.debug.print("Task {} running on thread {}\n", .{
        id,
        std.Thread.getCurrentId(),
    });
    std.time.sleep(10 * std.time.ns_per_ms);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var pool = try WorkerPool.init(allocator, 4);  // 4 workers
    defer pool.deinit();

    // Submit tasks
    for (0..20) |i| {
        try pool.submit(.{ .id = i, .work = exampleWork });
    }

    // รอให้งานเสร็จก่อน deinit
    std.time.sleep(500 * std.time.ns_per_ms);
}
```

---

## Step 450: Async สถานะปัจจุบันใน Zig (limitations)

### สถานะปัจจุบัน (Zig 0.14)

> **สำคัญมาก**: Async/Await ใน Zig กำลังผ่านการ redesign ครั้งใหญ่

```
ประวัติ async ใน Zig:
──────────────────────
Zig 0.6-0.10: มี async/await แต่ไม่สมบูรณ์
Zig 0.11-0.12: async ถูก "disabled" สำหรับ self-hosted compiler
Zig 0.13-0.14: async ยังอยู่ระหว่างการ redesign
อนาคต:         async/await จะถูก redesign ด้วย approach ใหม่
```

### ข้อจำกัดในปัจจุบัน

```zig
// ⚠️ สิ่งเหล่านี้อาจไม่ทำงานใน Zig 0.14:

// 1. async/await syntax อาจไม่ compile
// async fn myFunc() void { ... }  // ERROR ใน current Zig

// 2. @Frame() อาจไม่ทำงาน
// const frame = @Frame(myFunc);  // อาจ error

// 3. suspend/resume อาจไม่ทำงาน
// fn foo() void { suspend; }  // อาจ error
```

### ทางเลือกที่ใช้ได้ในปัจจุบัน

**1. Threads (แนะนำสำหรับ CPU-bound)**

```zig
const std = @import("std");

pub fn main() !void {
    const t1 = try std.Thread.spawn(.{}, heavyWork, .{1});
    const t2 = try std.Thread.spawn(.{}, heavyWork, .{2});
    t1.join();
    t2.join();
}

fn heavyWork(id: usize) void {
    std.debug.print("Thread {} working\n", .{id});
}
```

**2. libxev สำหรับ async I/O**

```zig
// build.zig.zon
// .libxev = .{
//     .url = "https://github.com/mitchellh/libxev/...",
//     .hash = "...",
// }

// ใช้งาน:
// const xev = @import("xev");
// var loop = try xev.Loop.init(.{});
// defer loop.deinit();
```

**3. std.event.Loop (ใน future releases)**

```zig
// ใน Zig future versions:
// const loop = std.event.Loop.instance.?;
// const result = try loop.runDetached(allocator, myTask, .{});
```

### Comparison กับ ภาษาอื่น

```
Language    | Async Model              | Status
──────────────────────────────────────────────
Zig         | Stackless coroutines     | In redesign
Rust        | Futures + tokio          | Mature
Go          | Goroutines (green thread)| Mature
JavaScript  | Event loop + Promises    | Mature
Python      | asyncio + coroutines     | Mature
C           | Manual (libuv, etc.)     | Ecosystem-dependent
```

### เมื่อไหร่ควรใช้อะไร

```
Use threads when:
✓ CPU-intensive work
✓ ต้องการ parallel computation
✓ ใช้ blocking APIs
✓ Stability สำคัญกว่า efficiency

Use async when (future):
✓ I/O-bound work (network, file)
✓ High concurrency (1000s connections)
✓ ต้องการ minimal memory per task
✓ Event-driven architecture
```

### Example: HTTP Server ที่ Scalable (วิธีปัจจุบัน)

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const addr = net.Address.initIp4(.{ 127, 0, 0, 1 }, 8080);
    var server = try addr.listen(.{
        .reuse_address = true,
    });
    defer server.deinit();

    std.debug.print("Server running on http://localhost:8080\n", .{});

    // Thread pool สำหรับรับ connections
    var thread_pool: std.Thread.Pool = undefined;
    try std.Thread.Pool.init(&thread_pool, .{
        .allocator = allocator,
        .n_jobs = 8,  // 8 worker threads
    });
    defer thread_pool.deinit();

    while (true) {
        const conn = try server.accept();

        // ส่งงานไป thread pool
        try thread_pool.spawn(handleHttp, .{conn});
    }
}

fn handleHttp(conn: net.Server.Connection) void {
    defer conn.stream.close();

    var buf: [4096]u8 = undefined;
    const n = conn.stream.read(&buf) catch return;
    if (n == 0) return;

    // Simple HTTP response
    const response =
        "HTTP/1.1 200 OK\r\n" ++
        "Content-Type: text/html\r\n" ++
        "Content-Length: 29\r\n" ++
        "\r\n" ++
        "<h1>Hello from Zig Server!</h1>";

    conn.stream.writeAll(response) catch return;
}
```

### อนาคตของ Async ใน Zig

```
Zig Team's Plans:
─────────────────
1. Redesign async ให้ไม่ต้องการ async keyword
2. ใช้ "function colors" approach ที่แตกต่างออกไป
3. Integration กับ io_uring (Linux), kqueue (macOS), IOCP (Windows)
4. Zero-overhead abstraction
5. Better error messages

Timeline:
- ยังไม่แน่ชัด
- ติดตามได้ที่: https://github.com/ziglang/zig/issues/6025
```

---

## สรุป Part 45: Async ใน Zig

| Step | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 441 | Async concept | ทำไมต้องใช้ async |
| 442 | async function | ฟังก์ชัน async |
| 443 | await expression | การรอผลลัพธ์ |
| 444 | async frame | State management |
| 445 | Event loops | การ handle events |
| 446 | Coroutines | Suspend/Resume |
| 447 | Async I/O | Non-blocking I/O |
| 448 | Error handling | Errors ใน async |
| 449 | Async patterns | Fan-out, Pipeline, Pool |
| 450 | Limitations | สถานะปัจจุบัน |

### สิ่งที่ควรทำตอนนี้ (Zig 0.14)

```zig
// 1. ใช้ Threads สำหรับ concurrency
const thread = try std.Thread.spawn(.{}, myTask, .{args});
defer thread.join();

// 2. ใช้ Thread Pool สำหรับ multiple tasks
var pool: std.Thread.Pool = undefined;
try std.Thread.Pool.init(&pool, .{ .allocator = allocator });
defer pool.deinit();

// 3. ใช้ Mutex/Condition สำหรับ synchronization
var mutex = std.Thread.Mutex{};
mutex.lock();
defer mutex.unlock();

// 4. ใช้ Channel pattern สำหรับ communication
// (implement เอง หรือใช้ library)

// 5. ติดตาม async development ที่:
// https://github.com/ziglang/zig
// https://ziglang.org/news/
```

### Resources สำหรับ Async ใน Zig

```
Libraries ที่น่าสนใจ:
─────────────────────
1. libxev - Cross-platform event loop
   https://github.com/mitchellh/libxev

2. tigerbeetle's io_uring bindings
   https://github.com/tigerbeetle/tigerbeetle

3. zig-network - Networking library
   https://github.com/MasterQ32/zig-network

4. zap - Fast HTTP server
   https://github.com/zigzap/zap
```

---

## สรุปทั้งหมด Parts 41-45

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 41 | Build System พื้นฐาน | 401-410 |
| 42 | Build System ขั้นสูง | 411-420 |
| 43 | C Interop พื้นฐาน | 421-430 |
| 44 | C Interop ขั้นสูง | 431-440 |
| 45 | Async ใน Zig | 441-450 |

> **ต่อไป**: Part 46 - Comptime Advanced (Metaprogramming ขั้นสูง)
