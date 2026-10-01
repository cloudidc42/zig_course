# Part 59: Profiling ใน Zig (Steps 581-590)

Profiling คือกระบวนการวัดประสิทธิภาพของโปรแกรมเพื่อหาจุดที่ช้า (bottlenecks) และปรับปรุงให้ดีขึ้น บทนี้จะครอบคลุมเครื่องมือ profiling ทั้งบน Linux และ macOS รวมถึงการเขียน benchmarks ด้วย Zig standard library

---

## Step 581: Profiling Overview

### ทำไมต้อง Profile?

```
กฎทอง 80/20 ของ Performance:
- 80% ของเวลาทำงานอยู่ใน 20% ของ code
- อย่า optimize โดยไม่ measure ก่อน
- "Premature optimization is the root of all evil" - Donald Knuth

ขั้นตอน Performance Optimization:
1. วัด (Profile) - หาว่าช้าที่ไหน
2. วิเคราะห์ - เข้าใจว่าทำไมถึงช้า
3. แก้ไข - optimize จุดที่ช้า
4. วัดใหม่ - ยืนยันว่าดีขึ้น
5. ทำซ้ำ
```

### ประเภทของ Profiling

```
1. CPU Profiling
   ├─ Sampling: เก็บ sample ของ call stack ทุก N milliseconds
   └─ Instrumentation: เพิ่ม code เพื่อ track ทุก function call

2. Memory Profiling
   ├─ Heap profiling: ดูว่า allocate ที่ไหนมาก
   └─ Timeline: ดูการใช้ memory ตามเวลา

3. I/O Profiling
   ├─ Disk I/O: read/write ที่ไหน, เท่าไหร่
   └─ Network I/O: latency, throughput

4. Cache Profiling
   ├─ L1/L2/L3 cache miss rates
   └─ TLB misses
```

### Build สำหรับ Profiling

```bash
# Build ด้วย debug symbols แต่ optimized
zig build-exe main.zig -O ReleaseFast -g

# หรือใน build.zig
```

```zig
// build.zig - build สำหรับ profiling
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    
    // Profile build: optimized แต่มี debug symbols
    const profile_exe = b.addExecutable(.{
        .name = "myapp_profile",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseFast,
    });
    profile_exe.strip = false;  // เก็บ debug symbols
    
    const profile_step = b.step("profile", "Build for profiling");
    const install_profile = b.addInstallArtifact(profile_exe, .{});
    profile_step.dependOn(&install_profile.step);
}
```

---

## Step 582: perf (Linux Profiling)

### ติดตั้งและใช้งาน perf

```bash
# ติดตั้ง perf
sudo apt-get install linux-tools-common linux-tools-generic
sudo apt-get install linux-tools-$(uname -r)

# ตรวจสอบ
perf --version
```

### perf stat - ดู Hardware Counters

```bash
# ดู basic statistics
perf stat ./myapp

# Output ตัวอย่าง:
#  Performance counter stats for './myapp':
#
#         1,234.56 msec task-clock                #    0.998 CPUs utilized
#                5      context-switches           #    4.050 /sec
#                0      cpu-migrations             #    0.000 /sec
#              123      page-faults                #  100.000 /sec
#    3,456,789,012      cycles                     #    2.800 GHz
#    2,345,678,901      instructions               #    0.68  insn per cycle
#      123,456,789      branches                   #  100.044 M/sec
#        1,234,567      branch-misses              #    1.00% of all branches

# ดู cache stats
perf stat -e cache-misses,cache-references,L1-dcache-misses ./myapp

# ดู memory operations
perf stat -e mem-loads,mem-stores ./myapp
```

### perf record และ report

```bash
# เก็บ sampling data
perf record -g ./myapp
# สร้างไฟล์ perf.data

# ดู report
perf report

# Output example:
# Samples: 10K of event 'cycles', 4000 Hz, Event count: 3456789012
#
# Children      Self  Command    Shared Object  Symbol
# +   45.23%   45.23%  myapp      myapp          [.] processData
# +   23.45%   23.45%  myapp      myapp          [.] sortItems
# +   12.34%   12.34%  myapp      libc.so        [.] malloc

# Interactive navigation:
# j/k: เลื่อน
# Enter: drill down
# q: ออก
```

### perf top - Real-time Profiling

```bash
# ดู hotspots แบบ real-time
perf top

# ดูเฉพาะ process ที่กำลัง run
perf top -p <pid>

# ดู source code annotations
perf annotate
```

### ตัวอย่าง Program สำหรับ Profiling

```zig
// profiling_target.zig - program สำหรับ profile
const std = @import("std");

// ฟังก์ชันที่ช้า (O(n²))
fn bubbleSort(arr: []i32) void {
    const n = arr.len;
    var i: usize = 0;
    while (i < n) : (i += 1) {
        var j: usize = 0;
        while (j < n - 1 - i) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

// ฟังก์ชันที่เร็วกว่า (O(n log n))
fn quickSort(arr: []i32, low: usize, high: usize) void {
    if (low >= high) return;
    
    const pivot_idx = partition(arr, low, high);
    if (pivot_idx > 0) quickSort(arr, low, pivot_idx - 1);
    quickSort(arr, pivot_idx + 1, high);
}

fn partition(arr: []i32, low: usize, high: usize) usize {
    const pivot = arr[high];
    var i = low;
    var j = low;
    
    while (j < high) : (j += 1) {
        if (arr[j] <= pivot) {
            const temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
            i += 1;
        }
    }
    
    const temp = arr[i];
    arr[i] = arr[high];
    arr[high] = temp;
    return i;
}

// String processing ที่ allocate มาก
fn processStrings(allocator: std.mem.Allocator, count: usize) !usize {
    var total_len: usize = 0;
    var i: usize = 0;
    
    while (i < count) : (i += 1) {
        const str = try std.fmt.allocPrint(allocator, "Item #{d:0>8}", .{i});
        defer allocator.free(str);
        total_len += str.len;
    }
    
    return total_len;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const N = 10000;
    var arr1 = try allocator.alloc(i32, N);
    defer allocator.free(arr1);
    var arr2 = try allocator.alloc(i32, N);
    defer allocator.free(arr2);
    
    // เติมข้อมูล random
    var prng = std.rand.DefaultPrng.init(42);
    const random = prng.random();
    for (arr1, arr2) |*a, *b| {
        const v = random.int(i32);
        a.* = v;
        b.* = v;
    }
    
    // Sort ด้วยสองวิธี
    const t1 = std.time.nanoTimestamp();
    bubbleSort(arr1);
    const t2 = std.time.nanoTimestamp();
    quickSort(arr2, 0, N - 1);
    const t3 = std.time.nanoTimestamp();
    
    std.debug.print("BubbleSort: {d}ms\n", .{@divTrunc(t2 - t1, 1_000_000)});
    std.debug.print("QuickSort:  {d}ms\n", .{@divTrunc(t3 - t2, 1_000_000)});
    
    // String processing
    const t4 = std.time.nanoTimestamp();
    const total = try processStrings(allocator, 10000);
    const t5 = std.time.nanoTimestamp();
    std.debug.print("String processing: {d}ms, total: {d}\n", .{
        @divTrunc(t5 - t4, 1_000_000),
        total,
    });
}
```

---

## Step 583: Instruments (macOS Profiling)

### Instruments Overview

Instruments เป็นเครื่องมือ profiling ที่มากับ Xcode สำหรับ macOS:

```bash
# เปิด Instruments
open -a Instruments

# หรือ command line
instruments -t "Time Profiler" -D /tmp/trace.trace ./myapp
```

### Command Line Tools สำหรับ macOS

```bash
# sample - ดู CPU usage
sample ./myapp 5 10  # sample 5 วินาที ทุก 10ms

# dtrace - powerful system-wide tracing
dtrace -n 'pid$target:::entry { @calls[probefunc] = count(); }' \
       -c ./myapp

# Instruments CLI
xcrun xctrace record \
    --template "Time Profiler" \
    --launch -- ./myapp \
    --output /tmp/profile.trace
```

### ตัวอย่าง DTrace Script

```dtrace
#!/usr/sbin/dtrace -s
/* profile.d - DTrace script สำหรับ Zig programs */

/* Profile CPU usage */
profile-99hz
/pid == $target/
{
    @[ustack()] = count();
}

END {
    trunc(@, 10);
    printa(@);
}
```

```bash
# Run DTrace script
sudo dtrace -s profile.d -c ./myapp
```

### Instruments Templates ที่มีประโยชน์

```
Time Profiler      - CPU profiling, call trees
Allocations        - Memory allocation tracking
Leaks              - Memory leak detection
Activity Monitor   - System resource usage
Network            - Network traffic analysis
Core Data          - Core Data performance
Metal System Trace - GPU profiling
```

---

## Step 584: Sampling vs Instrumentation Profiling

### Sampling Profiling

```
ทำงานอย่างไร:
1. Timer interrupt ทุก N ms (เช่น 1000 Hz)
2. บันทึก call stack ปัจจุบัน
3. นับว่าแต่ละ function ถูก sample กี่ครั้ง
4. ฟังก์ชันที่ถูก sample มาก = ช้า

ข้อดี:
+ Overhead ต่ำ (1-10%)
+ ไม่ต้อง recompile
+ เหมาะสำหรับ production

ข้อเสีย:
- ความแม่นยำต่ำสำหรับ functions ที่ run สั้นมาก
- Missing rare but slow paths
```

### Instrumentation Profiling

```
ทำงานอย่างไร:
1. เพิ่ม code ใน function entry/exit
2. บันทึก timestamp ทุกครั้ง
3. คำนวณเวลาที่ใช้จริง

ข้อดี:
+ ความแม่นยำสูง
+ เห็นทุก function call

ข้อเสีย:
- Overhead สูง (10-100x)
- เปลี่ยน timing behavior
- ไม่เหมาะสำหรับ production
```

### ตัวอย่าง Manual Instrumentation ใน Zig

```zig
// instrumented.zig - manual instrumentation
const std = @import("std");

// Simple profiling structure
const ProfileEntry = struct {
    name: []const u8,
    total_ns: i64,
    calls: u64,
};

var profile_entries: [100]ProfileEntry = undefined;
var profile_count: usize = 0;

pub fn profileBegin(name: []const u8) i64 {
    return std.time.nanoTimestamp();
}

pub fn profileEnd(name: []const u8, start_ns: i64) void {
    const elapsed = std.time.nanoTimestamp() - start_ns;
    
    // ค้นหา entry ที่มีอยู่แล้ว
    for (profile_entries[0..profile_count]) |*entry| {
        if (std.mem.eql(u8, entry.name, name)) {
            entry.total_ns += elapsed;
            entry.calls += 1;
            return;
        }
    }
    
    // สร้าง entry ใหม่
    if (profile_count < profile_entries.len) {
        profile_entries[profile_count] = .{
            .name = name,
            .total_ns = elapsed,
            .calls = 1,
        };
        profile_count += 1;
    }
}

pub fn profilePrint() void {
    const stdout = std.io.getStdOut().writer();
    
    stdout.print("\n=== Profile Results ===\n", .{}) catch {};
    stdout.print("{s:<30} {s:>10} {s:>10} {s:>15}\n", .{
        "Function", "Calls", "Total(ms)", "Avg(us)"
    }) catch {};
    stdout.print("{s}\n", .{"-" ** 70}) catch {};
    
    for (profile_entries[0..profile_count]) |entry| {
        const total_ms = @as(f64, @floatFromInt(entry.total_ns)) / 1_000_000.0;
        const avg_us = @as(f64, @floatFromInt(entry.total_ns)) / 
                       @as(f64, @floatFromInt(entry.calls)) / 1_000.0;
        
        stdout.print("{s:<30} {d:>10} {d:>10.2} {d:>15.2}\n", .{
            entry.name, entry.calls, total_ms, avg_us
        }) catch {};
    }
}

// ตัวอย่างการใช้
fn expensiveComputation(n: u64) u64 {
    const t = profileBegin("expensiveComputation");
    defer profileEnd("expensiveComputation", t);
    
    var result: u64 = 0;
    var i: u64 = 0;
    while (i < n) : (i += 1) {
        result += i * i;
    }
    return result;
}

fn stringProcessing(allocator: std.mem.Allocator, count: usize) !void {
    const t = profileBegin("stringProcessing");
    defer profileEnd("stringProcessing", t);
    
    var i: usize = 0;
    while (i < count) : (i += 1) {
        const s = try std.fmt.allocPrint(allocator, "Item {}", .{i});
        defer allocator.free(s);
        _ = s.len;
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Run functions หลายครั้ง
    var i: usize = 0;
    while (i < 1000) : (i += 1) {
        _ = expensiveComputation(1000);
    }
    
    try stringProcessing(allocator, 5000);
    
    profilePrint();
}
```

---

## Step 585: Flamegraphs

### สร้าง Flamegraph จาก perf

Flamegraph แสดง call stack ในรูปแบบ visual ที่เข้าใจง่าย

```bash
# ติดตั้ง FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph

# เก็บ profile data
perf record -g -F 999 ./myapp -- args

# แปลงเป็น flamegraph
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flamegraph.svg

# เปิดดู
firefox flamegraph.svg
# หรือ
xdg-open flamegraph.svg
```

### ตีความ Flamegraph

```
Flamegraph structure:
- แกน X: เวลาทั้งหมด (กว้างกว่า = ใช้เวลามากกว่า)
- แกน Y: call stack depth
- สีแดง/ส้ม: hot spots
- Function ที่กว้างที่สุดคือ bottleneck

การอ่าน:
1. มองหา "plateau" (บริเวณกว้างๆ ด้านบน)
2. ตามหา root cause (ลงมาตาม stack)
3. ตรวจสอบ unexpected wide frames
```

### async-profiler (Java-style) สำหรับ Zig

```bash
# ใช้ perf กับ stack unwinding ที่ดีขึ้น
perf record -g --call-graph dwarf ./myapp

# หรือใช้ bpftrace สำหรับ dynamic tracing
bpftrace -e '
profile:hz:99
/comm == "myapp"/
{
    @[ustack] = count();
}
END {
    exit();
}
' > stacks.txt

# สร้าง flamegraph
cat stacks.txt | stackcollapse-bpftrace.pl | flamegraph.pl > flame.svg
```

---

## Step 586: Memory Profiling

### Heap Profiling ด้วย Valgrind Massif

```bash
# เก็บ heap profile
valgrind --tool=massif --pages-as-heap=yes ./myapp

# Visualize ด้วย ms_print
ms_print massif.out.12345

# หรือ ด้วย massif-visualizer (GUI)
massif-visualizer massif.out.12345
```

### Custom Memory Allocator สำหรับ Profiling

```zig
// profiling_allocator.zig - track allocations
const std = @import("std");

pub const ProfilingAllocator = struct {
    backing_allocator: std.mem.Allocator,
    total_allocated: usize,
    total_freed: usize,
    current_usage: usize,
    peak_usage: usize,
    allocation_count: u64,
    free_count: u64,
    
    const Self = @This();
    
    pub fn init(backing: std.mem.Allocator) Self {
        return .{
            .backing_allocator = backing,
            .total_allocated = 0,
            .total_freed = 0,
            .current_usage = 0,
            .peak_usage = 0,
            .allocation_count = 0,
            .free_count = 0,
        };
    }
    
    pub fn allocator(self: *Self) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &vtable,
        };
    }
    
    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *Self = @alignCast(@ptrCast(ctx));
        const result = self.backing_allocator.rawAlloc(len, ptr_align, ret_addr);
        if (result != null) {
            self.total_allocated += len;
            self.current_usage += len;
            self.allocation_count += 1;
            if (self.current_usage > self.peak_usage) {
                self.peak_usage = self.current_usage;
            }
        }
        return result;
    }
    
    fn resize(ctx: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *Self = @alignCast(@ptrCast(ctx));
        const result = self.backing_allocator.rawResize(buf, buf_align, new_len, ret_addr);
        if (result) {
            if (new_len > buf.len) {
                const diff = new_len - buf.len;
                self.total_allocated += diff;
                self.current_usage += diff;
            } else {
                const diff = buf.len - new_len;
                self.total_freed += diff;
                self.current_usage -= diff;
            }
        }
        return result;
    }
    
    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *Self = @alignCast(@ptrCast(ctx));
        self.backing_allocator.rawFree(buf, buf_align, ret_addr);
        self.total_freed += buf.len;
        self.current_usage -= buf.len;
        self.free_count += 1;
    }
    
    const vtable = std.mem.Allocator.VTable{
        .alloc = alloc,
        .resize = resize,
        .free = free,
    };
    
    pub fn printStats(self: *const Self) void {
        std.debug.print("\n=== Memory Profile ===\n", .{});
        std.debug.print("Total allocated: {d} bytes ({d:.1} KB)\n", .{
            self.total_allocated,
            @as(f64, @floatFromInt(self.total_allocated)) / 1024.0,
        });
        std.debug.print("Total freed:     {d} bytes\n", .{self.total_freed});
        std.debug.print("Current usage:   {d} bytes\n", .{self.current_usage});
        std.debug.print("Peak usage:      {d} bytes\n", .{self.peak_usage});
        std.debug.print("Allocations:     {d}\n", .{self.allocation_count});
        std.debug.print("Frees:           {d}\n", .{self.free_count});
        if (self.current_usage > 0) {
            std.debug.print("WARNING: {} bytes still allocated (possible leak)\n", .{
                self.current_usage
            });
        }
    }
};
```

---

## Step 587: Benchmark ด้วย Zig std

### ใช้ std.time.Timer

```zig
// benchmark_basic.zig - ใช้ Timer
const std = @import("std");

fn fibonacci(n: u64) u64 {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

fn fibonacciIterative(n: u64) u64 {
    if (n <= 1) return n;
    var a: u64 = 0;
    var b: u64 = 1;
    var i: u64 = 2;
    while (i <= n) : (i += 1) {
        const c = a + b;
        a = b;
        b = c;
    }
    return b;
}

pub fn benchmarkFunction(
    comptime name: []const u8,
    comptime func: fn (u64) u64,
    input: u64,
    iterations: u64,
) void {
    var timer = std.time.Timer.start() catch return;
    
    var result: u64 = 0;
    var i: u64 = 0;
    while (i < iterations) : (i += 1) {
        result = func(input);
    }
    
    const elapsed_ns = timer.read();
    const elapsed_ms = @as(f64, @floatFromInt(elapsed_ns)) / 1_000_000.0;
    const per_iter_us = @as(f64, @floatFromInt(elapsed_ns)) / 
                        @as(f64, @floatFromInt(iterations)) / 1_000.0;
    
    std.debug.print("{s}: {d:.2}ms total, {d:.2}us/iter (result={})\n", .{
        name, elapsed_ms, per_iter_us, result
    });
}

pub fn main() !void {
    std.debug.print("=== Fibonacci Benchmark ===\n", .{});
    
    // n=30 เป็น input ที่ดีสำหรับ benchmark recursive fib
    benchmarkFunction("Recursive fib(30)", fibonacci, 30, 100);
    benchmarkFunction("Iterative fib(30)", fibonacciIterative, 30, 1_000_000);
    benchmarkFunction("Iterative fib(100)", fibonacciIterative, 100, 1_000_000);
}
```

### Zig Testing Framework สำหรับ Benchmarks

```zig
// bench_test.zig - benchmarks ใน test framework
const std = @import("std");

// ฟังก์ชัน sort ที่จะ benchmark
fn sortBenchmark(allocator: std.mem.Allocator, n: usize) !void {
    var data = try allocator.alloc(u32, n);
    defer allocator.free(data);
    
    // เติมข้อมูล
    var prng = std.rand.DefaultPrng.init(42);
    const random = prng.random();
    for (data) |*d| {
        d.* = random.int(u32);
    }
    
    // Sort
    std.sort.heap(u32, data, {}, std.sort.asc(u32));
}

test "benchmark sort performance" {
    const allocator = std.testing.allocator;
    
    const sizes = [_]usize{ 100, 1_000, 10_000, 100_000 };
    const iterations = [_]u32{ 10000, 1000, 100, 10 };
    
    for (sizes, iterations) |size, iters| {
        var timer = try std.time.Timer.start();
        
        var i: u32 = 0;
        while (i < iters) : (i += 1) {
            try sortBenchmark(allocator, size);
        }
        
        const elapsed = timer.read();
        const per_iter = @divTrunc(elapsed, iters);
        
        std.debug.print("sort({d}): {d}μs/iter\n", .{
            size,
            @divTrunc(per_iter, 1000),
        });
    }
}
```

### Comprehensive Benchmark Suite

```zig
// benchmarks.zig - comprehensive benchmark suite
const std = @import("std");

pub const BenchResult = struct {
    name: []const u8,
    iterations: u64,
    total_ns: u64,
    min_ns: u64,
    max_ns: u64,
    
    pub fn mean_ns(self: BenchResult) u64 {
        return self.total_ns / self.iterations;
    }
    
    pub fn print(self: BenchResult) void {
        const mean = self.mean_ns();
        std.debug.print(
            "{s:<30}: mean={d:>8}ns, min={d:>8}ns, max={d:>8}ns ({d} iters)\n",
            .{ self.name, mean, self.min_ns, self.max_ns, self.iterations }
        );
    }
};

pub fn bench(
    name: []const u8,
    iterations: u64,
    func: *const fn () void,
) BenchResult {
    // Warmup
    var warmup: u64 = 0;
    while (warmup < @min(iterations / 10, 100)) : (warmup += 1) {
        func();
    }
    
    var timer = std.time.Timer.start() catch unreachable;
    var total: u64 = 0;
    var min: u64 = std.math.maxInt(u64);
    var max: u64 = 0;
    
    var i: u64 = 0;
    while (i < iterations) : (i += 1) {
        timer.reset();
        func();
        const elapsed = timer.read();
        
        total += elapsed;
        if (elapsed < min) min = elapsed;
        if (elapsed > max) max = elapsed;
    }
    
    return .{
        .name = name,
        .iterations = iterations,
        .total_ns = total,
        .min_ns = min,
        .max_ns = max,
    };
}

// ฟังก์ชันสำหรับ benchmark
fn memcopyBench() void {
    var src = [_]u8{1} ** 1024;
    var dst: [1024]u8 = undefined;
    @memcpy(&dst, &src);
}

fn hashBench() void {
    const data = "Hello, World! This is a benchmark string.";
    _ = std.hash.Wyhash.hash(0, data);
}

fn allocationBench() void {
    const allocator = std.heap.page_allocator;
    const buf = allocator.alloc(u8, 256) catch return;
    allocator.free(buf);
}

pub fn main() void {
    std.debug.print("=== Benchmark Results ===\n\n", .{});
    
    const results = [_]BenchResult{
        bench("memcpy(1024)", 100_000, memcopyBench),
        bench("wyhash", 100_000, hashBench),
        bench("alloc+free(256)", 10_000, allocationBench),
    };
    
    for (results) |result| {
        result.print();
    }
}
```

---

## Step 588: Microbenchmark Pitfalls

### Common Mistakes ใน Benchmarking

```zig
// pitfalls.zig - ตัวอย่าง benchmark pitfalls
const std = @import("std");

// PITFALL 1: Dead code elimination
// Compiler อาจ optimize away ถ้า result ไม่ถูกใช้
fn badBenchmark() void {
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    // ถ้าไม่ใช้ sum, compiler อาจ eliminate loop ทั้งหมด!
    // _ = sum;  // นี่ไม่เพียงพอ
}

// SOLUTION: ใช้ volatile หรือ print result
fn goodBenchmark() u64 {
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    return sum;  // return ค่า เพื่อ prevent elimination
}

// หรือใช้ doNotOptimize
pub fn doNotOptimize(value: anytype) void {
    asm volatile ("" :: [value] "rm" (value) : "memory");
}

// PITFALL 2: Branch prediction warmup
fn benchmarkWithBranchPrediction() void {
    // Compiler จะ predict branches หลังจาก warmup
    // ควร randomize input เพื่อ realistic benchmark
    
    var arr = [_]i32{ 1, -2, 3, -4, 5 } ** 200;  // 1000 elements
    var sum: i64 = 0;
    
    for (arr) |x| {
        if (x > 0) {  // branch prediction จะ work ดีถ้า pattern predictable
            sum += x;
        }
    }
    doNotOptimize(sum);
}

// PITFALL 3: Cache effects
fn benchmarkWithCacheEffects(large_array: []const i32) void {
    var sum: i64 = 0;
    // Sequential access - cache friendly
    for (large_array) |x| {
        sum += x;
    }
    doNotOptimize(sum);
}

fn benchmarkWithCacheMisses(large_array: []const i32) void {
    var sum: i64 = 0;
    // Random access - cache unfriendly
    var prng = std.rand.DefaultPrng.init(42);
    var i: usize = 0;
    while (i < large_array.len) : (i += 1) {
        const idx = prng.random().uintLessThan(usize, large_array.len);
        sum += large_array[idx];
    }
    doNotOptimize(sum);
}

// PITFALL 4: Measurement overhead
fn demonstrateMeasurementOverhead() void {
    const iterations = 1_000_000;
    
    // ถ้า benchmark function เร็วมาก (ns range)
    // overhead ของ timer อาจมีผล
    var timer = std.time.Timer.start() catch return;
    
    var i: u64 = 0;
    while (i < iterations) : (i += 1) {
        // operation เร็วมาก: เช่น integer add
        var x: u64 = 42;
        x += 1;
        doNotOptimize(x);
    }
    
    const elapsed = timer.read();
    const per_iter = elapsed / iterations;
    std.debug.print("Per iteration: {}ns\n", .{per_iter});
    // ถ้า per_iter < ~10ns อาจ dominated โดย measurement overhead
}
```

### ป้องกัน Dead Code Elimination

```zig
// prevent_dce.zig - เทคนิคป้องกัน DCE
const std = @import("std");

// ใช้ volatile memory write
pub fn escapeToBlackHole(ptr: anytype) void {
    // Force compiler ให้คิดว่า memory ถูกอ่านจากภายนอก
    const volatile_ptr: *volatile @TypeOf(ptr.*) = @ptrCast(ptr);
    volatile_ptr.* = ptr.*;
}

// ใช้ inline assembly
pub fn clobber() void {
    asm volatile ("" ::: "memory");
}

// ตัวอย่างการใช้
fn properBenchmark() u64 {
    var result: u64 = 0;
    const N = 1_000_000;
    
    var i: u64 = 0;
    while (i < N) : (i += 1) {
        result = result *% 6364136223846793005 +% 1442695040888963407;  // LCG
    }
    
    // ป้องกัน DCE ด้วย volatile
    escapeToBlackHole(&result);
    
    return result;
}
```

---

## Step 589: Continuous Profiling

### Setup Continuous Profiling ด้วย perf

```bash
#!/bin/bash
# continuous_profile.sh - profile ทุก N นาที

PROFILE_DIR="/var/log/profiles/$(date +%Y-%m-%d)"
mkdir -p "$PROFILE_DIR"

while true; do
    TIMESTAMP=$(date +%H-%M-%S)
    PROFILE_FILE="$PROFILE_DIR/profile_$TIMESTAMP.data"
    
    echo "Profiling at $TIMESTAMP..."
    
    # Profile สำหรับ 30 วินาที
    perf record -g -F 99 -a --sleep 30 -o "$PROFILE_FILE" 2>/dev/null
    
    # สร้าง flamegraph
    perf script -i "$PROFILE_FILE" | \
        stackcollapse-perf.pl | \
        flamegraph.pl > "$PROFILE_DIR/flame_$TIMESTAMP.svg"
    
    echo "Profile saved: $PROFILE_FILE"
    
    # รอ 5 นาที ก่อน profile ครั้งถัดไป
    sleep 300
done
```

### CI/CD Integration สำหรับ Performance

```yaml
# .github/workflows/benchmark.yml
name: Performance Benchmarks

on:
  push:
    branches: [main]
  pull_request:

jobs:
  benchmark:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Zig
        uses: mlugg/setup-zig@v1
      
      - name: Build benchmarks
        run: zig build -Doptimize=ReleaseFast
      
      - name: Run benchmarks
        run: |
          ./zig-out/bin/benchmarks > benchmark_results.txt
          cat benchmark_results.txt
      
      - name: Compare with baseline
        run: |
          if [ -f baseline_benchmarks.txt ]; then
            python3 scripts/compare_benchmarks.py \
              baseline_benchmarks.txt \
              benchmark_results.txt
          else
            echo "No baseline found, saving current as baseline"
            cp benchmark_results.txt baseline_benchmarks.txt
          fi
      
      - name: Save benchmark results
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: benchmark_results.txt
```

### Benchmark Comparison Script

```python
#!/usr/bin/env python3
# compare_benchmarks.py - เปรียบเทียบ benchmark results

import sys
import re
from dataclasses import dataclass
from typing import Dict, Optional

@dataclass
class BenchResult:
    name: str
    mean_ns: float
    min_ns: float
    max_ns: float

def parse_results(filename: str) -> Dict[str, BenchResult]:
    results = {}
    pattern = r'(\w+)\s*:\s*mean=(\d+)ns,\s*min=(\d+)ns,\s*max=(\d+)ns'
    
    with open(filename) as f:
        for line in f:
            m = re.search(pattern, line)
            if m:
                name = m.group(1)
                results[name] = BenchResult(
                    name=name,
                    mean_ns=float(m.group(2)),
                    min_ns=float(m.group(3)),
                    max_ns=float(m.group(4)),
                )
    
    return results

def compare(baseline_file: str, current_file: str, threshold: float = 0.05):
    baseline = parse_results(baseline_file)
    current = parse_results(current_file)
    
    print("=== Benchmark Comparison ===")
    print(f"{'Name':<30} {'Baseline':>12} {'Current':>12} {'Change':>10} {'Status':>10}")
    print("-" * 80)
    
    regressions = []
    
    for name, curr in current.items():
        base = baseline.get(name)
        if base is None:
            print(f"{name:<30} {'N/A':>12} {curr.mean_ns:>12.0f}ns {'NEW':>10}")
            continue
        
        change = (curr.mean_ns - base.mean_ns) / base.mean_ns
        status = "OK"
        
        if change > threshold:
            status = "REGRESSION"
            regressions.append((name, change))
        elif change < -threshold:
            status = "IMPROVED"
        
        print(f"{name:<30} {base.mean_ns:>11.0f}ns {curr.mean_ns:>11.0f}ns "
              f"{change:>+9.1%} {status:>10}")
    
    if regressions:
        print(f"\nWARNING: {len(regressions)} regression(s) detected!")
        for name, change in regressions:
            print(f"  - {name}: {change:+.1%} slower")
        sys.exit(1)
    else:
        print("\nAll benchmarks within acceptable range.")

if __name__ == "__main__":
    compare(sys.argv[1], sys.argv[2])
```

---

## Step 590: Performance Regression Testing

### Regression Test Framework

```zig
// perf_regression.zig - performance regression testing
const std = @import("std");

pub const PerfTest = struct {
    name: []const u8,
    func: *const fn () void,
    max_ns: u64,      // threshold สูงสุดที่ยอมรับได้
    iterations: u64,
};

pub fn runPerfTests(tests: []const PerfTest) bool {
    var all_pass = true;
    
    std.debug.print("\n=== Performance Regression Tests ===\n", .{});
    
    for (tests) |t| {
        // Warmup
        var warmup: u64 = 0;
        while (warmup < 10) : (warmup += 1) {
            t.func();
        }
        
        // Measure
        var timer = std.time.Timer.start() catch continue;
        var total: u64 = 0;
        var i: u64 = 0;
        while (i < t.iterations) : (i += 1) {
            timer.reset();
            t.func();
            total += timer.read();
        }
        
        const mean_ns = total / t.iterations;
        const pass = mean_ns <= t.max_ns;
        const status = if (pass) "PASS" else "FAIL";
        
        std.debug.print("{s:<30}: {d:>8}ns (limit: {d:>8}ns) [{s}]\n", .{
            t.name, mean_ns, t.max_ns, status
        });
        
        if (!pass) all_pass = false;
    }
    
    return all_pass;
}

// ตัวอย่างฟังก์ชันที่จะทดสอบ
var test_buffer: [4096]u8 = undefined;

fn memsetTest() void {
    @memset(&test_buffer, 0);
}

fn memcopyTest() void {
    var dst: [4096]u8 = undefined;
    @memcpy(&dst, &test_buffer);
}

fn hashTest() void {
    const result = std.hash.Wyhash.hash(0, &test_buffer);
    _ = result;
}

pub fn main() void {
    const perf_tests = [_]PerfTest{
        .{ .name = "memset(4096)", .func = memsetTest, .max_ns = 1000, .iterations = 10_000 },
        .{ .name = "memcpy(4096)", .func = memcopyTest, .max_ns = 1000, .iterations = 10_000 },
        .{ .name = "wyhash(4096)", .func = hashTest, .max_ns = 2000, .iterations = 10_000 },
    };
    
    const pass = runPerfTests(&perf_tests);
    
    if (!pass) {
        std.debug.print("\nPERFORMANCE REGRESSION DETECTED!\n", .{});
        std.process.exit(1);
    } else {
        std.debug.print("\nAll performance tests passed!\n", .{});
    }
}
```

### สรุป Profiling Best Practices

```
Profiling Checklist:
┌─────────────────────────────────────────────────────────┐
│ 1. Profile ก่อน Optimize                               │
│    ├─ ใช้ perf/Instruments หา bottlenecks             │
│    ├─ อย่า optimize โดยเดา                            │
│    └─ วัดผลหลัง optimize ทุกครั้ง                     │
│                                                       │
│ 2. Build สำหรับ Profiling                              │
│    ├─ ใช้ -O ReleaseFast หรือ -O ReleaseSafe          │
│    ├─ อย่า strip debug symbols (-g)                   │
│    └─ Disable inlining ถ้าต้องการ precise stacks      │
│                                                       │
│ 3. Benchmark อย่างถูกต้อง                              │
│    ├─ Warmup ก่อน measure                             │
│    ├─ ทำหลาย iterations                               │
│    ├─ ป้องกัน dead code elimination                   │
│    ├─ ระวัง CPU frequency scaling                     │
│    └─ ทำซ้ำหลายครั้ง ดู variance                     │
│                                                       │
│ 4. Tools Overview                                      │
│    ├─ Linux: perf stat, perf record, flamegraph       │
│    ├─ macOS: Instruments, sample, dtrace              │
│    ├─ Memory: Valgrind massif, custom allocator       │
│    └─ CI: automated benchmark regression tests        │
└─────────────────────────────────────────────────────────┘
```

---

**บทสรุป Part 59:**

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Profiling ใน Zig:

1. **Step 581**: Profiling overview - ทำไมต้อง profile และประเภทของ profiling
2. **Step 582**: perf บน Linux - stat, record, report, flamegraph
3. **Step 583**: Instruments บน macOS - GUI และ command line tools
4. **Step 584**: Sampling vs Instrumentation - เปรียบเทียบและ manual instrumentation
5. **Step 585**: Flamegraphs - การสร้างและการตีความ
6. **Step 586**: Memory profiling ด้วย Valgrind massif และ custom allocator
7. **Step 587**: Benchmarks ด้วย std.time.Timer และ comprehensive benchmark suite
8. **Step 588**: Microbenchmark pitfalls - dead code elimination, cache effects
9. **Step 589**: Continuous profiling ใน CI/CD
10. **Step 590**: Performance regression testing

การ profile อย่างสม่ำเสมอและมี regression tests จะช่วยให้ performance ของโปรแกรมไม่ถดถอยโดยไม่รู้ตัว
