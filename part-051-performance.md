# Part 51: Performance Optimization ใน Zig
## Steps 501-510

การเพิ่มประสิทธิภาพโปรแกรม (Performance Optimization) เป็นหนึ่งในจุดแข็งที่สำคัญที่สุดของภาษา Zig ในบทนี้เราจะเรียนรู้เทคนิคและเครื่องมือต่างๆ ที่ช่วยให้โปรแกรม Zig ทำงานได้เร็วที่สุดเท่าที่เป็นไปได้

---

## Step 501: Performance Philosophy ใน Zig

### แนวคิดหลักด้านประสิทธิภาพของ Zig

Zig ถูกออกแบบมาโดยมีหลักการพื้นฐานว่า "No hidden control flow, no hidden allocations" ซึ่งหมายความว่านักพัฒนาสามารถมองเห็นและควบคุมทุกอย่างที่เกิดขึ้นในโปรแกรมได้อย่างชัดเจน

```zig
const std = @import("std");

// Zig ไม่มี hidden allocations - ทุก allocation ต้องระบุชัดเจน
pub fn main() !void {
    // ดีกว่า: ใช้ stack allocation เมื่อทำได้
    var buffer: [1024]u8 = undefined;
    
    // แทนที่จะใช้ heap allocation ที่ไม่จำเป็น
    // var buffer = try allocator.alloc(u8, 1024);
    
    std.debug.print("Buffer size: {}\n", .{buffer.len});
}
```

### หลักการ Performance ใน Zig

1. **Zero-cost abstractions**: Abstractions ใน Zig ไม่มีค่าใช้จ่ายเพิ่มเติมในขณะรันไทม์
2. **Compile-time evaluation**: การคำนวณที่ทำได้ในเวลา compile จะถูกทำในเวลา compile
3. **Manual memory management**: นักพัฒนาควบคุม memory ได้โบ้สมบูรณ์
4. **No garbage collector**: ไม่มี GC overhead
5. **Direct hardware access**: สามารถเข้าถึง hardware ได้โดยตรง

```zig
const std = @import("std");

// Compile-time computation ลด runtime overhead
const LOOKUP_TABLE = blk: {
    var table: [256]u8 = undefined;
    for (&table, 0..) |*entry, i| {
        entry.* = @intCast(i * 2 % 256);
    }
    break :blk table;
};

pub fn fastLookup(index: u8) u8 {
    return LOOKUP_TABLE[index];
}

// ตัวอย่าง: การเปรียบเทียบ compile-time vs runtime
pub fn main() void {
    // นี่คือ compile-time - ไม่มี overhead เลย
    const result = comptime blk: {
        var sum: u32 = 0;
        var i: u32 = 0;
        while (i < 100) : (i += 1) {
            sum += i;
        }
        break :blk sum;
    };
    
    std.debug.print("Compile-time sum: {}\n", .{result});
    std.debug.print("Lookup result: {}\n", .{fastLookup(42)});
}
```

### Build Mode และ Optimization Level

```zig
// build.zig - ตั้งค่า optimization level
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    b.installArtifact(exe);
}
```

คำสั่ง build modes:
- `zig build-exe main.zig` - Debug mode (เหมาะสำหรับ development)
- `zig build-exe -O ReleaseSafe main.zig` - Safe release (มี safety checks)
- `zig build-exe -O ReleaseFast main.zig` - Fast release (ไม่มี safety checks)
- `zig build-exe -O ReleaseSmall main.zig` - Small release (optimized สำหรับขนาด)

```zig
const std = @import("std");

// การใช้ @setRuntimeSafety เพื่อควบคุม safety ในระดับ function
pub fn unsafeButFast(slice: []u8, index: usize) u8 {
    @setRuntimeSafety(false); // ปิด safety checks สำหรับ function นี้
    return slice[index]; // ไม่มี bounds checking
}

pub fn safeVersion(slice: []u8, index: usize) u8 {
    return slice[index]; // มี bounds checking
}
```

---

## Step 502: Profiling Tools (perf, valgrind, zig built-in)

### การใช้ Linux perf

```bash
# Build with debug symbols
zig build-exe -O ReleaseFast --strip false main.zig

# Profile ด้วย perf
perf record ./main
perf report

# หรือดู statistics
perf stat ./main

# Sampling with specific events
perf record -e cache-misses,cache-references,cycles,instructions ./main
perf report
```

### การใช้ Valgrind Callgrind

```bash
# Build with debug info
zig build-exe main.zig

# Run ด้วย Callgrind
valgrind --tool=callgrind ./main

# Analyze ด้วย KCachegrind
kcachegrind callgrind.out.*
```

### Valgrind Cachegrind สำหรับ cache analysis

```bash
# วิเคราะห์ cache behavior
valgrind --tool=cachegrind ./main
cg_annotate cachegrind.out.*
```

### Zig Built-in Profiling ด้วย std.time.Timer

```zig
const std = @import("std");

pub fn profileFunction(comptime name: []const u8, func: anytype, args: anytype) void {
    var timer = std.time.Timer.start() catch unreachable;
    
    const result = @call(.auto, func, args);
    _ = result;
    
    const elapsed = timer.read();
    std.debug.print("{s}: {d}ns ({d}ms)\n", .{
        name,
        elapsed,
        elapsed / 1_000_000,
    });
}

fn slowFunction() void {
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    std.debug.print("Sum: {}\n", .{sum});
}

fn fastFunction() void {
    // Gauss formula
    const n: u64 = 999_999;
    const sum = n * (n + 1) / 2;
    std.debug.print("Sum: {}\n", .{sum});
}

pub fn main() !void {
    std.debug.print("=== Profiling Demo ===\n", .{});
    
    profileFunction("Slow loop", slowFunction, .{});
    profileFunction("Fast formula", fastFunction, .{});
}
```

### Tracy Profiler Integration

```zig
// ใช้ Tracy สำหรับ visual profiling
const tracy = @import("tracy");

pub fn someFunction() void {
    const zone = tracy.Zone(@src());
    defer zone.end();
    
    // โค้ดที่ต้องการ profile
    expensiveOperation();
}
```

### Custom Benchmark Framework

```zig
const std = @import("std");

const BenchmarkResult = struct {
    name: []const u8,
    iterations: u64,
    total_ns: u64,
    min_ns: u64,
    max_ns: u64,
    
    pub fn avgNs(self: BenchmarkResult) u64 {
        return self.total_ns / self.iterations;
    }
    
    pub fn print(self: BenchmarkResult) void {
        std.debug.print(
            "{s}: avg={d}ns, min={d}ns, max={d}ns ({d} iters)\n",
            .{ self.name, self.avgNs(), self.min_ns, self.max_ns, self.iterations }
        );
    }
};

pub fn benchmark(
    comptime name: []const u8,
    iterations: u64,
    func: anytype,
    args: anytype
) BenchmarkResult {
    var timer = std.time.Timer.start() catch unreachable;
    var total: u64 = 0;
    var min: u64 = std.math.maxInt(u64);
    var max: u64 = 0;
    
    // Warmup
    for (0..10) |_| {
        _ = @call(.auto, func, args);
    }
    
    for (0..iterations) |_| {
        timer.reset();
        _ = @call(.auto, func, args);
        const elapsed = timer.read();
        
        total += elapsed;
        min = @min(min, elapsed);
        max = @max(max, elapsed);
    }
    
    return .{
        .name = name,
        .iterations = iterations,
        .total_ns = total,
        .min_ns = min,
        .max_ns = max,
    };
}
```

---

## Step 503: Memory Layout Optimization

### การจัดการ Struct Padding

```zig
const std = @import("std");

// ไม่ดี: มี padding มาก
const BadLayout = struct {
    a: u8,    // 1 byte
    // 7 bytes padding
    b: u64,   // 8 bytes
    c: u8,    // 1 byte
    // 7 bytes padding
    d: u64,   // 8 bytes
};

// ดี: เรียง fields ตามขนาดจากใหญ่ไปเล็ก
const GoodLayout = struct {
    b: u64,   // 8 bytes
    d: u64,   // 8 bytes
    a: u8,    // 1 byte
    c: u8,    // 1 byte
    // 6 bytes padding
};

// ดีที่สุด: ใช้ packed struct (ไม่มี padding เลย)
const PackedLayout = packed struct {
    a: u8,
    c: u8,
    b: u64,
    d: u64,
};

pub fn main() void {
    std.debug.print("BadLayout size: {}\n", .{@sizeOf(BadLayout)});
    std.debug.print("GoodLayout size: {}\n", .{@sizeOf(GoodLayout)});
    std.debug.print("PackedLayout size: {}\n", .{@sizeOf(PackedLayout)});
    
    // แสดง alignment
    std.debug.print("BadLayout align: {}\n", .{@alignOf(BadLayout)});
    std.debug.print("GoodLayout align: {}\n", .{@alignOf(GoodLayout)});
}
```

### การใช้ extern struct สำหรับ C compatibility

```zig
// extern struct รับประกัน layout เหมือน C
const CCompatible = extern struct {
    x: f32,
    y: f32,
    z: f32,
};

// packed struct สำหรับ bit manipulation
const BitFlags = packed struct(u32) {
    flag_a: u1,
    flag_b: u1,
    flag_c: u1,
    reserved: u5,
    value: u24,
};

pub fn useBitFlags() void {
    var flags = BitFlags{
        .flag_a = 1,
        .flag_b = 0,
        .flag_c = 1,
        .reserved = 0,
        .value = 12345,
    };
    
    // เข้าถึง bits โดยตรง
    flags.flag_b = 1;
    std.debug.print("Flags as u32: {}\n", .{@as(u32, @bitCast(flags))});
}
```

### Alignment และ Memory Access

```zig
const std = @import("std");

// การระบุ alignment
const AlignedData = struct {
    // ใช้ align() เพื่อระบุ alignment
    data: [64]u8 align(64), // Cache-line aligned
};

pub fn alignedAllocation(allocator: std.mem.Allocator) !void {
    // Allocate ด้วย specific alignment
    const ptr = try allocator.alignedAlloc(u8, 64, 1024);
    defer allocator.free(ptr);
    
    std.debug.print("Is 64-byte aligned: {}\n", .{
        @intFromPtr(ptr.ptr) % 64 == 0
    });
}

// การใช้ @ptrCast กับ alignment
pub fn reinterpretMemory(data: []u8) []u32 {
    std.debug.assert(data.len % 4 == 0);
    std.debug.assert(@intFromPtr(data.ptr) % 4 == 0);
    
    return @as([*]u32, @ptrCast(@alignCast(data.ptr)))[0..data.len / 4];
}
```

---

## Step 504: Cache-Friendly Data Structures (SoA vs AoS)

### Array of Structures (AoS) vs Structure of Arrays (SoA)

```zig
const std = @import("std");

// AoS (Array of Structures) - ทั่วไปแต่ไม่ cache-friendly เมื่อเข้าถึง fields เดียว
const ParticleAoS = struct {
    x: f32,
    y: f32,
    z: f32,
    vx: f32,
    vy: f32,
    vz: f32,
    mass: f32,
    charge: f32,
};

// SoA (Structure of Arrays) - cache-friendly เมื่อ process fields เดียวกันทั้งหมด
const ParticlesSoA = struct {
    x: []f32,
    y: []f32,
    z: []f32,
    vx: []f32,
    vy: []f32,
    vz: []f32,
    mass: []f32,
    charge: []f32,
    count: usize,
    
    pub fn init(allocator: std.mem.Allocator, count: usize) !ParticlesSoA {
        return .{
            .x = try allocator.alloc(f32, count),
            .y = try allocator.alloc(f32, count),
            .z = try allocator.alloc(f32, count),
            .vx = try allocator.alloc(f32, count),
            .vy = try allocator.alloc(f32, count),
            .vz = try allocator.alloc(f32, count),
            .mass = try allocator.alloc(f32, count),
            .charge = try allocator.alloc(f32, count),
            .count = count,
        };
    }
    
    pub fn deinit(self: ParticlesSoA, allocator: std.mem.Allocator) void {
        allocator.free(self.x);
        allocator.free(self.y);
        allocator.free(self.z);
        allocator.free(self.vx);
        allocator.free(self.vy);
        allocator.free(self.vz);
        allocator.free(self.mass);
        allocator.free(self.charge);
    }
};

// อัปเดตตำแหน่ง - SoA เร็วกว่า AoS เพราะ x,y,z อยู่ติดกันใน memory
pub fn updatePositionsSoA(particles: *ParticlesSoA, dt: f32) void {
    const count = particles.count;
    
    // Loop นี้ vectorizable โดย LLVM เพราะ x[] เป็น contiguous array
    for (0..count) |i| {
        particles.x[i] += particles.vx[i] * dt;
    }
    for (0..count) |i| {
        particles.y[i] += particles.vy[i] * dt;
    }
    for (0..count) |i| {
        particles.z[i] += particles.vz[i] * dt;
    }
}

pub fn updatePositionsAoS(particles: []ParticleAoS, dt: f32) void {
    // ใน AoS, x, y, z ไม่ติดกันใน memory (มี fields อื่นคั่น)
    for (particles) |*p| {
        p.x += p.vx * dt;
        p.y += p.vy * dt;
        p.z += p.vz * dt;
    }
}
```

### Hybrid AoSoA (Array of Structures of Arrays)

```zig
// AoSoA - balance ระหว่าง cache performance และ code simplicity
const BATCH_SIZE = 8; // SIMD width

const ParticleBatch = struct {
    x: [BATCH_SIZE]f32,
    y: [BATCH_SIZE]f32,
    z: [BATCH_SIZE]f32,
    vx: [BATCH_SIZE]f32,
    vy: [BATCH_SIZE]f32,
    vz: [BATCH_SIZE]f32,
};

pub fn updateBatchPositions(batch: *ParticleBatch, dt: f32) void {
    for (0..BATCH_SIZE) |i| {
        batch.x[i] += batch.vx[i] * dt;
        batch.y[i] += batch.vy[i] * dt;
        batch.z[i] += batch.vz[i] * dt;
    }
}
```

### Cache Line Awareness

```zig
const std = @import("std");

// Cache line บน x86_64 คือ 64 bytes
const CACHE_LINE_SIZE = 64;

// False sharing - ไม่ดี
const SharedCounterBad = struct {
    counter_a: std.atomic.Value(u64),
    counter_b: std.atomic.Value(u64),
};

// Padded - ดีกว่า (ป้องกัน false sharing)
const SharedCounterGood = struct {
    counter_a: std.atomic.Value(u64) align(CACHE_LINE_SIZE),
    _pad_a: [CACHE_LINE_SIZE - @sizeOf(std.atomic.Value(u64))]u8 = undefined,
    counter_b: std.atomic.Value(u64) align(CACHE_LINE_SIZE),
    _pad_b: [CACHE_LINE_SIZE - @sizeOf(std.atomic.Value(u64))]u8 = undefined,
};
```

---

## Step 505: Avoiding Allocations in Hot Paths

### Stack Allocation Strategies

```zig
const std = @import("std");

// ไม่ดี: allocate ทุก call
pub fn processDataBad(allocator: std.mem.Allocator, data: []const u8) ![]u8 {
    var result = try allocator.alloc(u8, data.len);
    // process...
    return result;
}

// ดี: ใช้ caller-provided buffer
pub fn processDataGood(data: []const u8, output: []u8) usize {
    // ไม่มี allocation
    const len = @min(data.len, output.len);
    for (data[0..len], 0..) |byte, i| {
        output[i] = byte ^ 0xFF; // example transform
    }
    return len;
}

// ดีมาก: ใช้ std.BoundedArray สำหรับ dynamic-ish arrays บน stack
pub fn processWithBoundedArray(data: []const u8) void {
    var result = std.BoundedArray(u8, 4096){};
    
    for (data) |byte| {
        if (result.len < result.capacity()) {
            result.appendAssumeCapacity(byte ^ 0xFF);
        }
    }
    
    std.debug.print("Processed {} bytes\n", .{result.len});
}
```

### Arena Allocator สำหรับ Batch Operations

```zig
const std = @import("std");

pub fn batchProcess(
    allocator: std.mem.Allocator,
    items: []const []const u8,
) !void {
    // Arena allocator - free ทั้งหมดด้วย single call
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit(); // free ทุกอย่างด้วยการ call เดียว
    
    const arena_alloc = arena.allocator();
    
    for (items) |item| {
        // Allocations ทั้งหมดอยู่ใน arena
        const processed = try arena_alloc.alloc(u8, item.len);
        @memcpy(processed, item);
        // ไม่ต้อง free แต่ละชิ้น
    }
    
    // arena.deinit() จะ free ทุกอย่างในครั้งเดียว
}
```

### Object Pool Pattern

```zig
const std = @import("std");

fn ObjectPool(comptime T: type, comptime max_size: usize) type {
    return struct {
        const Self = @This();
        
        pool: [max_size]T = undefined,
        free_list: [max_size]usize = undefined,
        free_count: usize = 0,
        
        pub fn init(self: *Self) void {
            for (0..max_size) |i| {
                self.free_list[i] = i;
            }
            self.free_count = max_size;
        }
        
        pub fn acquire(self: *Self) ?*T {
            if (self.free_count == 0) return null;
            
            self.free_count -= 1;
            const index = self.free_list[self.free_count];
            return &self.pool[index];
        }
        
        pub fn release(self: *Self, obj: *T) void {
            const index = (@intFromPtr(obj) - @intFromPtr(&self.pool[0])) / @sizeOf(T);
            self.free_list[self.free_count] = index;
            self.free_count += 1;
        }
    };
}

const Connection = struct {
    id: u32,
    buffer: [1024]u8,
    active: bool,
};

pub fn main() void {
    var pool: ObjectPool(Connection, 100) = undefined;
    pool.init();
    
    // Acquire connection (no heap allocation!)
    const conn = pool.acquire() orelse {
        std.debug.print("Pool exhausted!\n", .{});
        return;
    };
    
    conn.* = .{ .id = 1, .buffer = undefined, .active = true };
    
    // Use connection...
    
    // Return to pool (no free!)
    pool.release(conn);
}
```

---

## Step 506: Branch Prediction Optimization

### การใช้ @branchHint

```zig
const std = @import("std");

pub fn checkBounds(slice: []const u8, index: usize) ?u8 {
    // บอก CPU ว่า branch ไหนน่าจะเกิดบ่อยกว่า
    if (@branchHint(false, index >= slice.len)) {
        return null; // unlikely: out of bounds
    }
    return slice[index]; // likely: in bounds
}

pub fn processData(data: []const u8) u64 {
    var sum: u64 = 0;
    
    for (data) |byte| {
        // @branchHint บอกว่า byte != 0 มีโอกาสเกิดมากกว่า
        if (@branchHint(true, byte != 0)) {
            sum += byte;
        }
    }
    
    return sum;
}
```

### Branchless Programming

```zig
const std = @import("std");

// มี branch - อาจทำให้ branch predictor มีปัญหา
pub fn clampBranch(value: i32, min: i32, max: i32) i32 {
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

// Branchless - ดีกว่าสำหรับ unpredictable data
pub fn clampBranchless(value: i32, min_val: i32, max_val: i32) i32 {
    return @min(@max(value, min_val), max_val);
}

// Branchless absolute value
pub fn absValue(x: i32) i32 {
    const mask = x >> 31; // -1 ถ้าลบ, 0 ถ้าบวก
    return (x + mask) ^ mask;
}

// Branchless max
pub fn maxBranchless(a: i32, b: i32) i32 {
    const diff = a - b;
    const sign = diff >> 31; // -1 ถ้า a < b, 0 ถ้า a >= b
    return a - (diff & sign);
}

// Lookup table แทน if-else chain
const CHAR_TYPE_TABLE = blk: {
    var table: [256]u8 = undefined;
    for (&table, 0..) |*entry, i| {
        entry.* = if (i >= 'a' and i <= 'z') 1
                  else if (i >= 'A' and i <= 'Z') 2
                  else if (i >= '0' and i <= '9') 3
                  else 0;
    }
    break :blk table;
};

pub fn getCharType(c: u8) u8 {
    return CHAR_TYPE_TABLE[c]; // No branches!
}
```

### Loop Unrolling

```zig
// ปกติ - LLVM จะ unroll อัตโนมัติในหลายกรณี
pub fn sumNormal(data: []const u32) u64 {
    var sum: u64 = 0;
    for (data) |x| {
        sum += x;
    }
    return sum;
}

// Manual unroll สำหรับกรณีที่ต้องการ
pub fn sumUnrolled4(data: []const u32) u64 {
    var sum: u64 = 0;
    var i: usize = 0;
    
    // Process 4 elements ต่อ iteration
    while (i + 4 <= data.len) : (i += 4) {
        sum += data[i];
        sum += data[i + 1];
        sum += data[i + 2];
        sum += data[i + 3];
    }
    
    // Handle remaining
    while (i < data.len) : (i += 1) {
        sum += data[i];
    }
    
    return sum;
}
```

---

## Step 507: Inlining และ Function Call Overhead

### @inlineCall และ inline functions

```zig
const std = @import("std");

// Inline function - compiler ฝัง code ลงในจุดที่เรียก
inline fn addInline(a: u32, b: u32) u32 {
    return a + b;
}

// ปกติ - compiler ตัดสินใจเอง
fn addNormal(a: u32, b: u32) u32 {
    return a + b;
}

// บังคับไม่ให้ inline
noinline fn addNoInline(a: u32, b: u32) u32 {
    return a + b;
}

// callconv เพื่อระบุ calling convention
pub fn callConvExample() void {
    // C calling convention
    const add_c: *const fn(u32, u32) callconv(.C) u32 = &struct {
        fn f(a: u32, b: u32) callconv(.C) u32 {
            return a + b;
        }
    }.f;
    
    _ = add_c(1, 2);
}

// การใช้ @call สำหรับ control inlining
pub fn demonstrateCall(a: u32, b: u32) u32 {
    // .auto - ให้ compiler ตัดสินใจ
    const r1 = @call(.auto, addNormal, .{ a, b });
    
    // .always_inline - บังคับ inline
    const r2 = @call(.always_inline, addNormal, .{ a, b });
    
    // .never_inline - บังคับไม่ inline
    const r3 = @call(.never_inline, addNormal, .{ a, b });
    
    return r1 + r2 + r3;
}
```

### Function Pointer Overhead

```zig
const std = @import("std");

// Function pointer มี indirect call overhead
const Transform = *const fn(u32) u32;

pub fn applyTransform(data: []u32, transform: Transform) void {
    for (data) |*x| {
        x.* = transform(x.*);
    }
}

// ดีกว่า: ใช้ comptime เพื่อ eliminate indirection
pub fn applyTransformFast(comptime transform: fn(u32) u32, data: []u32) void {
    for (data) |*x| {
        x.* = transform(x.*); // Direct call, inlineable
    }
}

fn doubleValue(x: u32) u32 { return x * 2; }
fn squareValue(x: u32) u32 { return x * x; }

pub fn main() void {
    var data = [_]u32{ 1, 2, 3, 4, 5 };
    
    // Indirect call (slower)
    applyTransform(&data, &doubleValue);
    
    // Direct call (faster, inlineable)
    applyTransformFast(squareValue, &data);
    
    for (data) |x| {
        std.debug.print("{} ", .{x});
    }
    std.debug.print("\n", .{});
}
```

---

## Step 508: LLVM Optimization Hints

### การใช้ @expect (ผ่าน @branchHint)

```zig
const std = @import("std");

// ให้ LLVM รู้ว่า path ไหนเกิดบ่อย
pub fn parseNumber(s: []const u8) !i64 {
    if (@branchHint(false, s.len == 0)) {
        return error.Empty;
    }
    
    return std.fmt.parseInt(i64, s, 10);
}

// @setEvalBranchQuota สำหรับ comptime computation ที่ซับซ้อน
pub fn computeFactorial(comptime n: u64) u64 {
    @setEvalBranchQuota(10000);
    comptime {
        if (n == 0) return 1;
        return n * computeFactorial(n - 1);
    }
}
```

### Restrict-like Optimization ด้วย noalias

```zig
// noalias บอก compiler ว่า pointers ไม่ overlap กัน
// ช่วยให้ LLVM optimize ได้ดีกว่า
pub extern fn memcpy_fast(
    noalias dest: [*]u8,
    noalias src: [*]const u8,
    len: usize
) callconv(.C) void;

// Zig version
pub fn copyNoAlias(
    noalias dest: []u8,
    noalias src: []const u8,
) void {
    @memcpy(dest, src);
}
```

### Volatile สำหรับ Hardware Access

```zig
// volatile ป้องกัน compiler optimize ออก
pub fn readHardwareRegister(addr: usize) u32 {
    const reg_ptr: *volatile u32 = @ptrFromInt(addr);
    return reg_ptr.*;
}

pub fn writeHardwareRegister(addr: usize, value: u32) void {
    const reg_ptr: *volatile u32 = @ptrFromInt(addr);
    reg_ptr.* = value;
}

// ตัวอย่าง: spin lock ด้วย volatile
pub fn spinWait(flag: *volatile bool) void {
    while (flag.*) {
        // asm("pause") - hint to CPU for spin loop
        asm volatile ("pause" ::: "memory");
    }
}
```

### Link Section Attributes

```zig
// วาง code/data ใน specific memory sections
export var hot_data linksection(".data.hot") = [_]u32{1} ** 1024;

pub fn hotFunction() linksection(".text.hot") void {
    // code ที่ถูกเรียกบ่อยมาก
}

pub fn coldFunction() linksection(".text.cold") void {
    // code ที่ถูกเรียกน้อย (เช่น error handling)
}
```

---

## Step 509: Benchmark ด้วย std.time.Timer

### Basic Benchmarking

```zig
const std = @import("std");

pub fn main() !void {
    var timer = try std.time.Timer.start();
    
    // ทำ operation ที่ต้องการวัด
    var sum: u64 = 0;
    for (0..1_000_000) |i| {
        sum += i;
    }
    
    const elapsed_ns = timer.read();
    
    std.debug.print("Sum: {}\n", .{sum});
    std.debug.print("Time: {}ns ({}μs, {}ms)\n", .{
        elapsed_ns,
        elapsed_ns / 1000,
        elapsed_ns / 1_000_000,
    });
}
```

### Comprehensive Benchmark Suite

```zig
const std = @import("std");

const Stats = struct {
    count: u64,
    total: u64,
    min: u64,
    max: u64,
    
    pub fn init() Stats {
        return .{
            .count = 0,
            .total = 0,
            .min = std.math.maxInt(u64),
            .max = 0,
        };
    }
    
    pub fn record(self: *Stats, value: u64) void {
        self.count += 1;
        self.total += value;
        self.min = @min(self.min, value);
        self.max = @max(self.max, value);
    }
    
    pub fn mean(self: Stats) u64 {
        if (self.count == 0) return 0;
        return self.total / self.count;
    }
    
    pub fn print(self: Stats, name: []const u8) void {
        std.debug.print(
            "{s}:\n  mean={d}ns  min={d}ns  max={d}ns  n={d}\n",
            .{ name, self.mean(), self.min, self.max, self.count }
        );
    }
};

fn runBenchmark(
    comptime func: fn() void,
    warmup_iters: u32,
    bench_iters: u32,
) Stats {
    var stats = Stats.init();
    var timer = std.time.Timer.start() catch unreachable;
    
    // Warmup
    for (0..warmup_iters) |_| {
        func();
    }
    
    // Benchmark
    for (0..bench_iters) |_| {
        timer.reset();
        func();
        stats.record(timer.read());
    }
    
    return stats;
}

// Functions to benchmark
var global_result: u64 = 0;

fn benchmarkLinearSearch() void {
    const data = [_]u32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 } ** 100;
    const target: u32 = 7;
    
    for (data, 0..) |x, i| {
        if (x == target) {
            global_result = i;
            return;
        }
    }
}

fn benchmarkBinarySearch() void {
    var data: [1000]u32 = undefined;
    for (&data, 0..) |*x, i| {
        x.* = @intCast(i);
    }
    
    const result = std.sort.binarySearch(
        u32,
        700,
        &data,
        {},
        struct {
            fn order(_: void, key: u32, item: u32) std.math.Order {
                return std.math.order(key, item);
            }
        }.order,
    );
    
    global_result = if (result) |r| r else 0;
}

pub fn main() !void {
    std.debug.print("=== Benchmark Suite ===\n\n", .{});
    
    const linear_stats = runBenchmark(benchmarkLinearSearch, 100, 1000);
    linear_stats.print("Linear Search");
    
    const binary_stats = runBenchmark(benchmarkBinarySearch, 100, 1000);
    binary_stats.print("Binary Search");
    
    std.debug.print("\nResult (prevent optimization): {}\n", .{global_result});
}
```

### Preventing Dead Code Elimination

```zig
const std = @import("std");

// ป้องกัน compiler eliminate code ที่ result ไม่ถูกใช้
pub fn doNotOptimize(value: anytype) void {
    // asm volatile ป้องกัน optimization
    asm volatile (""
        :
        : [val] "r" (value),
        : "memory"
    );
}

pub fn benchmarkWithBarrier() void {
    var timer = std.time.Timer.start() catch unreachable;
    
    var result: u64 = 0;
    for (0..1_000_000) |i| {
        result += i * i;
    }
    
    // ป้องกัน dead code elimination
    doNotOptimize(result);
    
    const elapsed = timer.read();
    std.debug.print("Time: {}ns\n", .{elapsed});
}
```

---

## Step 510: Production Optimization Checklist

### Checklist สำหรับ Production Build

```zig
// ========================
// PRODUCTION OPTIMIZATION CHECKLIST
// ========================

// 1. Build Mode
// zig build -Doptimize=ReleaseFast  (speed)
// zig build -Doptimize=ReleaseSmall (size)
// zig build -Doptimize=ReleaseSafe  (safe + fast)

// 2. Target Architecture
// zig build -Dtarget=native (optimize สำหรับ CPU ที่รัน build)

// 3. LTO (Link Time Optimization)
// เพิ่มใน build.zig:
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Strip debug symbols ใน release
    if (optimize != .Debug) {
        exe.root_module.strip = true;
    }
    
    b.installArtifact(exe);
}
```

### Code-Level Optimization Patterns

```zig
const std = @import("std");

// Pattern 1: Avoid repeated computations
pub fn optimizedLoop(data: []const f32) f32 {
    const len = data.len; // Cache length
    const inv_len = 1.0 / @as(f32, @floatFromInt(len)); // Compute once
    
    var sum: f32 = 0.0;
    for (data) |x| {
        sum += x;
    }
    
    return sum * inv_len; // Multiply instead of divide
}

// Pattern 2: Strength reduction
pub fn strengthReduction(n: u32) u32 {
    // แทน multiply ด้วย shift
    return n << 3; // n * 8
}

// Pattern 3: Early exit
pub fn findFirst(data: []const u32, target: u32) ?usize {
    for (data, 0..) |x, i| {
        if (x == target) return i; // Exit เมื่อเจอ
    }
    return null;
}

// Pattern 4: Precomputed constants
const INV_255: f32 = 1.0 / 255.0;

pub fn normalizeColor(r: u8, g: u8, b: u8) [3]f32 {
    return .{
        @as(f32, @floatFromInt(r)) * INV_255,
        @as(f32, @floatFromInt(g)) * INV_255,
        @as(f32, @floatFromInt(b)) * INV_255,
    };
}

// Pattern 5: Data-driven dispatch แทน switch
const OPERATION_TABLE = [_]fn(u32, u32) u32{
    struct { fn f(a: u32, b: u32) u32 { return a + b; } }.f,
    struct { fn f(a: u32, b: u32) u32 { return a - b; } }.f,
    struct { fn f(a: u32, b: u32) u32 { return a * b; } }.f,
};

pub fn dispatchOperation(op: u8, a: u32, b: u32) u32 {
    if (op >= OPERATION_TABLE.len) return 0;
    return OPERATION_TABLE[op](a, b);
}
```

### Memory Optimization Patterns

```zig
const std = @import("std");

// Pattern: Custom allocator สำหรับ specific use case
pub fn StackAllocator(comptime size: usize) type {
    return struct {
        const Self = @This();
        
        buffer: [size]u8 align(16) = undefined,
        pos: usize = 0,
        
        pub fn allocator(self: *Self) std.mem.Allocator {
            return .{
                .ptr = self,
                .vtable = &.{
                    .alloc = alloc,
                    .resize = resize,
                    .reuseOrFreeListUnchecked = undefined,
                    .free = free,
                },
            };
        }
        
        fn alloc(
            ctx: *anyopaque,
            len: usize,
            ptr_align: u8,
            _: usize,
        ) ?[*]u8 {
            const self: *Self = @ptrCast(@alignCast(ctx));
            const aligned_pos = std.mem.alignForward(usize, self.pos, @as(usize, 1) << @intCast(ptr_align));
            
            if (aligned_pos + len > size) return null;
            
            self.pos = aligned_pos + len;
            return self.buffer[aligned_pos..].ptr;
        }
        
        fn resize(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
            return false;
        }
        
        fn free(_: *anyopaque, _: []u8, _: u8, _: usize) void {
            // Stack allocator ไม่ free แต่ละชิ้น
        }
        
        pub fn reset(self: *Self) void {
            self.pos = 0;
        }
    };
}

pub fn main() !void {
    var stack_alloc = StackAllocator(4096){};
    const alloc = stack_alloc.allocator();
    
    // Allocate จาก stack
    const buf1 = try alloc.alloc(u8, 100);
    const buf2 = try alloc.alloc(u8, 200);
    
    _ = buf1;
    _ = buf2;
    
    // Reset ทั้งหมด (O(1))
    stack_alloc.reset();
    
    std.debug.print("Stack allocator working!\n", .{});
}
```

### Final Performance Tips Summary

```zig
// สรุป Performance Tips:
// 
// 1. MEASURE FIRST - อย่า optimize โดยไม่มีข้อมูล
// 2. ใช้ -O ReleaseFast สำหรับ production
// 3. เรียง struct fields ตามขนาด (ใหญ่ → เล็ก)
// 4. ใช้ SoA แทน AoS เมื่อ process หลาย elements ด้วยกัน
// 5. หลีกเลี่ยง allocations ใน hot path
// 6. ใช้ @branchHint เพื่อ guide branch predictor
// 7. ใช้ inline สำหรับ small, hot functions
// 8. ใช้ comptime computation ให้มากที่สุด
// 9. หลีกเลี่ยง function pointers ใน hot path
// 10. ใช้ arena allocator สำหรับ batch operations
// 11. เขียน cache-friendly code (sequential access)
// 12. ใช้ packed struct เพื่อลด memory usage
// 13. Align data ที่เข้าถึงบ่อยให้ตรงกับ cache line
// 14. ใช้ noalias เมื่อ pointers ไม่ overlap
// 15. Profile โค้ดจริงๆ ก่อน optimize

const std = @import("std");

pub fn main() void {
    std.debug.print("Performance optimization is an art!\n", .{});
    std.debug.print("Always measure, then optimize.\n", .{});
}
```

---

## สรุป Part 51

ในบทนี้เราได้เรียนรู้:

1. **Performance Philosophy** - Zig ออกแบบมาเพื่อ performance โดยไม่มี hidden costs
2. **Profiling Tools** - perf, valgrind, และ built-in timer
3. **Memory Layout** - การจัด struct เพื่อลด padding
4. **SoA vs AoS** - เลือก data layout ที่เหมาะสม
5. **Avoiding Allocations** - stack allocation, arena, object pool
6. **Branch Prediction** - @branchHint, branchless code
7. **Inlining** - ควบคุม function call overhead
8. **LLVM Hints** - ช่วย compiler optimize ได้ดีกว่า
9. **Benchmarking** - วัดผลอย่างถูกต้อง
10. **Production Checklist** - ขั้นตอน optimize สำหรับ production

> **คำแนะนำ**: Performance optimization ที่ดีที่สุดคือการ profile ก่อนแล้วค่อย optimize จุดที่เป็น bottleneck จริงๆ ไม่ใช่ optimize ทุกอย่าง
