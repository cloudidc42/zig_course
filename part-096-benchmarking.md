# Part 96: Benchmarking ใน Zig
## Steps 951-960

---

## Step 951: Benchmarking Concepts

### แนวคิดหลักของ Benchmarking

Benchmarking คือกระบวนการวัดประสิทธิภาพของโค้ดอย่างเป็นระบบ เพื่อให้เข้าใจว่าโปรแกรมทำงานช้าหรือเร็วเพียงใด และเพื่อเปรียบเทียบการ implementation ต่างๆ

### ประเภทของ Benchmarks

**1. Micro-benchmarks**
- วัดฟังก์ชันหรือ operation เดียวที่เล็กมาก
- เช่น การ sort array, การค้นหาใน hash map
- ผลลัพธ์มักเป็น nanoseconds หรือ microseconds

**2. Macro-benchmarks**
- วัดทั้งระบบหรือ workflow ใหญ่
- เช่น throughput ของ web server, เวลาในการประมวลผล file ขนาดใหญ่
- ผลลัพธ์มักเป็น milliseconds หรือ seconds

**3. Profiling**
- ศึกษาว่าเวลาถูกใช้ไปที่ไหนบ้างใน program
- ช่วยหา bottlenecks

### หลักการพื้นฐาน

```zig
// benchmark_concepts.zig
const std = @import("std");

pub fn main() void {
    // หลักการ 1: วัดสิ่งที่สำคัญจริงๆ
    // หลักการ 2: รัน benchmark หลายๆ ครั้ง
    // หลักการ 3: ควบคุม environment
    // หลักการ 4: วิเคราะห์ผลอย่างเป็นระบบ
    
    std.debug.print("Benchmarking ใน Zig\n", .{});
    
    // ตัวอย่าง: การวัดเวลาแบบง่าย
    const start = std.time.nanoTimestamp();
    
    // งานที่ต้องการวัด
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    
    const end = std.time.nanoTimestamp();
    const elapsed_ns = end - start;
    
    std.debug.print("Sum: {d}\n", .{sum});
    std.debug.print("Time: {d} ns\n", .{elapsed_ns});
    std.debug.print("Time: {d:.3} ms\n", .{@as(f64, @floatFromInt(elapsed_ns)) / 1_000_000.0});
}
```

### สิ่งที่ควรระวัง

```zig
// benchmark_antipatterns.zig
const std = @import("std");

// Anti-pattern 1: ไม่ warm up CPU/cache
fn badBenchmark() void {
    const start = std.time.nanoTimestamp();
    // วัดครั้งแรกเพียงครั้งเดียว - ไม่น่าเชื่อถือ!
    expensiveOperation();
    const end = std.time.nanoTimestamp();
    std.debug.print("Time: {d} ns\n", .{end - start});
}

// Anti-pattern 2: ปล่อยให้ compiler optimize งานออกไป
fn alwaysZero() u64 {
    var x: u64 = 0;
    var i: u64 = 0;
    while (i < 1000) : (i += 1) {
        x += i; // compiler อาจ optimize ออก เพราะไม่ได้ใช้ x
    }
    return x;
}

fn expensiveOperation() void {
    std.time.sleep(1000);
}
```

### Warm-up และ Multiple Runs

```zig
// proper_benchmark.zig
const std = @import("std");

pub fn benchmark(
    comptime func: anytype,
    args: anytype,
    comptime iterations: u32,
    comptime warmup: u32,
) !BenchmarkResult {
    // Warm-up phase
    var wi: u32 = 0;
    while (wi < warmup) : (wi += 1) {
        _ = @call(.auto, func, args);
    }
    
    // Measurement phase
    var times: [iterations]i128 = undefined;
    var i: u32 = 0;
    while (i < iterations) : (i += 1) {
        const start = std.time.nanoTimestamp();
        _ = @call(.auto, func, args);
        const end = std.time.nanoTimestamp();
        times[i] = end - start;
    }
    
    return BenchmarkResult.compute(&times);
}

const BenchmarkResult = struct {
    min_ns: i128,
    max_ns: i128,
    mean_ns: f64,
    median_ns: f64,
    
    fn compute(times: []const i128) BenchmarkResult {
        var min: i128 = times[0];
        var max: i128 = times[0];
        var sum: i128 = 0;
        
        for (times) |t| {
            if (t < min) min = t;
            if (t > max) max = t;
            sum += t;
        }
        
        const mean = @as(f64, @floatFromInt(sum)) / @as(f64, @floatFromInt(times.len));
        
        // Median
        var sorted = times; // copy
        std.mem.sort(i128, &sorted, {}, std.sort.asc(i128));
        const median_idx = sorted.len / 2;
        const median = @as(f64, @floatFromInt(sorted[median_idx]));
        
        return .{
            .min_ns = min,
            .max_ns = max,
            .mean_ns = mean,
            .median_ns = median,
        };
    }
    
    fn print(self: BenchmarkResult, name: []const u8) void {
        std.debug.print("\nBenchmark: {s}\n", .{name});
        std.debug.print("  Min:    {d:.2} ns\n", .{@as(f64, @floatFromInt(self.min_ns))});
        std.debug.print("  Max:    {d:.2} ns\n", .{@as(f64, @floatFromInt(self.max_ns))});
        std.debug.print("  Mean:   {d:.2} ns\n", .{self.mean_ns});
        std.debug.print("  Median: {d:.2} ns\n", .{self.median_ns});
    }
};
```

---

## Step 952: std.time.Timer สำหรับ Benchmarks

### การใช้งาน std.time.Timer

`std.time.Timer` เป็น high-resolution timer ใน Zig standard library

```zig
// timer_basics.zig
const std = @import("std");

pub fn main() !void {
    // สร้าง timer
    var timer = try std.time.Timer.start();
    
    // งานที่ต้องการวัด
    doWork();
    
    // อ่านเวลาที่ผ่านไป (nanoseconds)
    const elapsed = timer.read();
    
    std.debug.print("Elapsed: {d} ns\n", .{elapsed});
    std.debug.print("Elapsed: {d:.3} µs\n", .{@as(f64, @floatFromInt(elapsed)) / 1000.0});
    std.debug.print("Elapsed: {d:.3} ms\n", .{@as(f64, @floatFromInt(elapsed)) / 1_000_000.0});
    
    // reset timer และวัดใหม่
    timer.reset();
    doMoreWork();
    const elapsed2 = timer.read();
    std.debug.print("Second operation: {d} ns\n", .{elapsed2});
    
    // lap - อ่านค่าและ reset พร้อมกัน
    timer.reset();
    var lap: u64 = undefined;
    
    for (0..5) |i| {
        doWork();
        lap = timer.lap();
        std.debug.print("Lap {d}: {d} ns\n", .{ i + 1, lap });
    }
}

fn doWork() void {
    var sum: u64 = 0;
    for (0..100_000) |i| {
        sum += i;
    }
    std.mem.doNotOptimizeAway(sum);
}

fn doMoreWork() void {
    var arr: [1000]u64 = undefined;
    for (&arr, 0..) |*v, i| {
        v.* = i * i;
    }
    std.mem.doNotOptimizeAway(arr);
}
```

### Timer Wrapper สำหรับ Benchmarking

```zig
// timer_wrapper.zig
const std = @import("std");

pub const Benchmark = struct {
    name: []const u8,
    timer: std.time.Timer,
    samples: std.ArrayList(u64),
    allocator: std.mem.Allocator,
    
    pub fn init(name: []const u8, allocator: std.mem.Allocator) !Benchmark {
        return Benchmark{
            .name = name,
            .timer = try std.time.Timer.start(),
            .samples = std.ArrayList(u64).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Benchmark) void {
        self.samples.deinit();
    }
    
    pub fn startMeasurement(self: *Benchmark) void {
        self.timer.reset();
    }
    
    pub fn endMeasurement(self: *Benchmark) !void {
        const elapsed = self.timer.read();
        try self.samples.append(elapsed);
    }
    
    pub fn run(
        self: *Benchmark,
        comptime func: anytype,
        args: anytype,
        iterations: u32,
    ) !void {
        // Warm up
        for (0..std.math.min(10, iterations)) |_| {
            _ = @call(.auto, func, args);
        }
        
        // Measure
        for (0..iterations) |_| {
            self.startMeasurement();
            _ = @call(.auto, func, args);
            try self.endMeasurement();
        }
    }
    
    pub fn report(self: *const Benchmark) void {
        if (self.samples.items.len == 0) {
            std.debug.print("{s}: no samples\n", .{self.name});
            return;
        }
        
        const stats = Statistics.compute(self.samples.items);
        
        std.debug.print("\n=== Benchmark: {s} ===\n", .{self.name});
        std.debug.print("Samples: {d}\n", .{self.samples.items.len});
        std.debug.print("Min:     {d:.2} ns ({d:.3} µs)\n", .{
            @as(f64, @floatFromInt(stats.min)),
            @as(f64, @floatFromInt(stats.min)) / 1000.0,
        });
        std.debug.print("Max:     {d:.2} ns ({d:.3} µs)\n", .{
            @as(f64, @floatFromInt(stats.max)),
            @as(f64, @floatFromInt(stats.max)) / 1000.0,
        });
        std.debug.print("Mean:    {d:.2} ns ({d:.3} µs)\n", .{
            stats.mean,
            stats.mean / 1000.0,
        });
        std.debug.print("Median:  {d:.2} ns ({d:.3} µs)\n", .{
            stats.median,
            stats.median / 1000.0,
        });
        std.debug.print("StdDev:  {d:.2} ns\n", .{stats.stddev});
        std.debug.print("CV:      {d:.2}%\n", .{stats.cv * 100.0});
    }
};

const Statistics = struct {
    min: u64,
    max: u64,
    mean: f64,
    median: f64,
    stddev: f64,
    cv: f64, // coefficient of variation
    
    fn compute(samples: []const u64) Statistics {
        var min = samples[0];
        var max = samples[0];
        var sum: u128 = 0;
        
        for (samples) |s| {
            if (s < min) min = s;
            if (s > max) max = s;
            sum += s;
        }
        
        const n = @as(f64, @floatFromInt(samples.len));
        const mean = @as(f64, @floatFromInt(sum)) / n;
        
        // Standard deviation
        var variance_sum: f64 = 0;
        for (samples) |s| {
            const diff = @as(f64, @floatFromInt(s)) - mean;
            variance_sum += diff * diff;
        }
        const stddev = @sqrt(variance_sum / n);
        const cv = if (mean > 0) stddev / mean else 0;
        
        // Median (simplified - no sorting copy here)
        const median = mean; // placeholder
        
        return .{
            .min = min,
            .max = max,
            .mean = mean,
            .median = median,
            .stddev = stddev,
            .cv = cv,
        };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var bench = try Benchmark.init("Array Sum", allocator);
    defer bench.deinit();
    
    const arr = [_]u64{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 } ** 1000;
    
    try bench.run(sumArray, .{&arr}, 1000);
    bench.report();
}

fn sumArray(arr: []const u64) u64 {
    var sum: u64 = 0;
    for (arr) |v| sum += v;
    return sum;
}
```

### nanoTimestamp vs Timer

```zig
// timestamp_comparison.zig
const std = @import("std");

pub fn main() !void {
    // วิธีที่ 1: nanoTimestamp (ง่ายกว่า)
    {
        const start = std.time.nanoTimestamp();
        doWork();
        const end = std.time.nanoTimestamp();
        const elapsed = end - start;
        std.debug.print("nanoTimestamp: {d} ns\n", .{elapsed});
    }
    
    // วิธีที่ 2: Timer (แนะนำกว่า - reset ง่าย, ใช้ monotonic clock)
    {
        var timer = try std.time.Timer.start();
        doWork();
        const elapsed = timer.read();
        std.debug.print("Timer: {d} ns\n", .{elapsed});
    }
    
    // วิธีที่ 3: milliTimestamp (ความแม่นยำต่ำกว่า)
    {
        const start = std.time.milliTimestamp();
        doWork();
        const end = std.time.milliTimestamp();
        std.debug.print("milliTimestamp: {d} ms\n", .{end - start});
    }
}

fn doWork() void {
    var x: u64 = 0;
    for (0..1_000_000) |i| x += i;
    std.mem.doNotOptimizeAway(x);
}
```

---

## Step 953: Micro-benchmarking Pitfalls

### ปัญหาที่พบบ่อย

**1. Compiler Optimization**

```zig
// optimization_pitfall.zig
const std = @import("std");

// ปัญหา: compiler optimize งานออก เพราะไม่ใช้ผลลัพธ์
fn badBenchmarkLoop() void {
    var timer = std.time.Timer.start() catch unreachable;
    
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        // compiler อาจเอาทิ้งทั้ง loop เพราะ sum ไม่ได้ถูกใช้
        var sum: u64 = 0;
        sum += i;
    }
    
    const elapsed = timer.read();
    std.debug.print("Bad: {d} ns (อาจผิดพลาด!)\n", .{elapsed});
}

// วิธีแก้: ใช้ std.mem.doNotOptimizeAway
fn goodBenchmarkLoop() void {
    var timer = std.time.Timer.start() catch unreachable;
    
    var sum: u64 = 0;
    var i: u64 = 0;
    while (i < 1_000_000) : (i += 1) {
        sum += i;
    }
    std.mem.doNotOptimizeAway(sum); // บอก compiler ว่า sum ถูกใช้
    
    const elapsed = timer.read();
    std.debug.print("Good: {d} ns\n", .{elapsed});
}

pub fn main() !void {
    badBenchmarkLoop();
    goodBenchmarkLoop();
}
```

**2. CPU Frequency Scaling**

```zig
// frequency_scaling.zig
const std = @import("std");

// ปัญหา: CPU frequency เปลี่ยนระหว่าง benchmark
// วิธีแก้: ใช้ CPU governor แบบ "performance" และ warm up ก่อน

pub fn benchmarkWithWarmup(
    comptime func: fn () void,
    warmup_iters: u32,
    measure_iters: u32,
) void {
    // Warm up เพื่อให้ CPU ทำงานที่ frequency สูง
    std.debug.print("Warming up... ({d} iterations)\n", .{warmup_iters});
    for (0..warmup_iters) |_| {
        func();
    }
    
    // วัดผลจริง
    var timer = std.time.Timer.start() catch unreachable;
    var total_ns: u64 = 0;
    
    for (0..measure_iters) |_| {
        timer.reset();
        func();
        total_ns += timer.read();
    }
    
    const avg_ns = total_ns / measure_iters;
    std.debug.print("Average: {d} ns over {d} iterations\n", .{ avg_ns, measure_iters });
}

fn workload() void {
    var x: u64 = 0;
    for (0..10_000) |i| x = x *% i +% 1;
    std.mem.doNotOptimizeAway(x);
}

pub fn main() void {
    benchmarkWithWarmup(workload, 100, 1000);
}
```

**3. Cache Effects**

```zig
// cache_effects.zig
const std = @import("std");

const ARRAY_SIZE = 64 * 1024 * 1024; // 64 MB

// Sequential access - cache friendly
fn sequentialAccess(arr: []u64) u64 {
    var sum: u64 = 0;
    for (arr) |v| sum += v;
    return sum;
}

// Random access - cache unfriendly
fn randomAccess(arr: []u64, indices: []usize) u64 {
    var sum: u64 = 0;
    for (indices) |idx| sum += arr[idx];
    return sum;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง large array เพื่อทดสอบ cache effects
    const arr = try allocator.alloc(u64, ARRAY_SIZE / @sizeOf(u64));
    defer allocator.free(arr);
    
    for (arr, 0..) |*v, i| v.* = i;
    
    // Sequential benchmark
    var timer = try std.time.Timer.start();
    const sum1 = sequentialAccess(arr);
    const seq_time = timer.read();
    std.mem.doNotOptimizeAway(sum1);
    
    std.debug.print("Sequential: {d} ns\n", .{seq_time});
    std.debug.print("Throughput: {d:.2} GB/s\n", .{
        @as(f64, @floatFromInt(ARRAY_SIZE)) / @as(f64, @floatFromInt(seq_time)),
    });
}
```

**4. Branch Prediction Effects**

```zig
// branch_prediction.zig
const std = @import("std");

// Predictable branch
fn predictableBranch(arr: []const i32) i64 {
    var sum: i64 = 0;
    for (arr) |v| {
        if (v > 0) sum += v; // array ที่ sorted จะทำให้ branch predictable
    }
    return sum;
}

// Unpredictable branch
fn unpredictableBranch(arr: []const i32) i64 {
    var sum: i64 = 0;
    for (arr) |v| {
        if (v > 128) sum += v; // random data จะทำให้ branch unpredictable
    }
    return sum;
}

// Branchless version
fn branchless(arr: []const i32) i64 {
    var sum: i64 = 0;
    for (arr) |v| {
        // ไม่มี branch - ใช้ arithmetic แทน
        sum += @as(i64, v) & -@as(i64, @intFromBool(v > 0));
    }
    return sum;
}

pub fn main() !void {
    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();
    
    var arr: [100_000]i32 = undefined;
    for (&arr) |*v| v.* = random.intRangeLessThan(i32, -256, 256);
    
    var timer = try std.time.Timer.start();
    
    const ITERS = 100;
    
    timer.reset();
    for (0..ITERS) |_| {
        const r = predictableBranch(&arr);
        std.mem.doNotOptimizeAway(r);
    }
    const t1 = timer.read() / ITERS;
    
    timer.reset();
    for (0..ITERS) |_| {
        const r = unpredictableBranch(&arr);
        std.mem.doNotOptimizeAway(r);
    }
    const t2 = timer.read() / ITERS;
    
    timer.reset();
    for (0..ITERS) |_| {
        const r = branchless(&arr);
        std.mem.doNotOptimizeAway(r);
    }
    const t3 = timer.read() / ITERS;
    
    std.debug.print("Predictable branch:   {d} ns\n", .{t1});
    std.debug.print("Unpredictable branch: {d} ns\n", .{t2});
    std.debug.print("Branchless:          {d} ns\n", .{t3});
}
```

---

## Step 954: Statistical Analysis ของ Benchmarks

### การวิเคราะห์ผลทางสถิติ

```zig
// statistical_analysis.zig
const std = @import("std");
const math = std.math;

pub const Stats = struct {
    n: usize,
    min: f64,
    max: f64,
    mean: f64,
    median: f64,
    p95: f64,  // 95th percentile
    p99: f64,  // 99th percentile
    stddev: f64,
    cv: f64,   // coefficient of variation
    iqr: f64,  // interquartile range
    
    pub fn compute(allocator: std.mem.Allocator, samples: []const u64) !Stats {
        if (samples.len == 0) return error.EmptySamples;
        
        // สร้าง sorted copy
        const sorted = try allocator.dupe(u64, samples);
        defer allocator.free(sorted);
        std.mem.sort(u64, sorted, {}, std.sort.asc(u64));
        
        const n = samples.len;
        const nf = @as(f64, @floatFromInt(n));
        
        // Min, Max
        const min = @as(f64, @floatFromInt(sorted[0]));
        const max = @as(f64, @floatFromInt(sorted[n - 1]));
        
        // Mean
        var sum: u128 = 0;
        for (samples) |s| sum += s;
        const mean = @as(f64, @floatFromInt(sum)) / nf;
        
        // Standard deviation
        var variance_sum: f64 = 0;
        for (samples) |s| {
            const diff = @as(f64, @floatFromInt(s)) - mean;
            variance_sum += diff * diff;
        }
        const stddev = @sqrt(variance_sum / nf);
        
        // Percentiles
        const median = percentile(sorted, 50);
        const p95 = percentile(sorted, 95);
        const p99 = percentile(sorted, 99);
        
        // IQR
        const q1 = percentile(sorted, 25);
        const q3 = percentile(sorted, 75);
        const iqr = q3 - q1;
        
        return Stats{
            .n = n,
            .min = min,
            .max = max,
            .mean = mean,
            .median = median,
            .p95 = p95,
            .p99 = p99,
            .stddev = stddev,
            .cv = if (mean > 0) stddev / mean else 0,
            .iqr = iqr,
        };
    }
    
    fn percentile(sorted: []const u64, p: u8) f64 {
        const n = sorted.len;
        const idx_f = @as(f64, @floatFromInt(p)) / 100.0 * @as(f64, @floatFromInt(n - 1));
        const idx_low = @as(usize, @intFromFloat(idx_f));
        const idx_high = @min(idx_low + 1, n - 1);
        const frac = idx_f - @as(f64, @floatFromInt(idx_low));
        
        const v_low = @as(f64, @floatFromInt(sorted[idx_low]));
        const v_high = @as(f64, @floatFromInt(sorted[idx_high]));
        
        return v_low + frac * (v_high - v_low);
    }
    
    pub fn print(self: Stats, name: []const u8) void {
        std.debug.print("\n{'='<50}\n", .{});
        std.debug.print("Benchmark: {s}\n", .{name});
        std.debug.print("{'='<50}\n", .{});
        std.debug.print("Samples:  {d}\n", .{self.n});
        std.debug.print("Min:      {d:.2} ns\n", .{self.min});
        std.debug.print("Max:      {d:.2} ns\n", .{self.max});
        std.debug.print("Mean:     {d:.2} ns\n", .{self.mean});
        std.debug.print("Median:   {d:.2} ns\n", .{self.median});
        std.debug.print("P95:      {d:.2} ns\n", .{self.p95});
        std.debug.print("P99:      {d:.2} ns\n", .{self.p99});
        std.debug.print("StdDev:   {d:.2} ns\n", .{self.stddev});
        std.debug.print("CV:       {d:.2}%\n", .{self.cv * 100.0});
        std.debug.print("IQR:      {d:.2} ns\n", .{self.iqr});
        
        // แสดง quality indicator
        if (self.cv < 0.05) {
            std.debug.print("Quality:  Excellent (CV < 5%)\n", .{});
        } else if (self.cv < 0.10) {
            std.debug.print("Quality:  Good (CV < 10%)\n", .{});
        } else if (self.cv < 0.20) {
            std.debug.print("Quality:  Fair (CV < 20%)\n", .{});
        } else {
            std.debug.print("Quality:  Poor (CV >= 20%) - ควรรันใหม่\n", .{});
        }
    }
    
    // ตรวจหา outliers ด้วย IQR method
    pub fn detectOutliers(
        self: Stats,
        allocator: std.mem.Allocator,
        samples: []const u64,
    ) ![]u64 {
        const lower_fence = self.median - 1.5 * self.iqr;
        const upper_fence = self.median + 1.5 * self.iqr;
        
        var outliers = std.ArrayList(u64).init(allocator);
        for (samples) |s| {
            const sf = @as(f64, @floatFromInt(s));
            if (sf < lower_fence or sf > upper_fence) {
                try outliers.append(s);
            }
        }
        
        return outliers.toOwnedSlice();
    }
};

// Bootstrap confidence intervals
pub fn bootstrapCI(
    allocator: std.mem.Allocator,
    samples: []const u64,
    confidence: f64, // 0.95 สำหรับ 95% CI
    n_bootstrap: u32,
) !struct { lower: f64, upper: f64 } {
    var prng = std.Random.DefaultPrng.init(42);
    const random = prng.random();
    
    var bootstrap_means = try allocator.alloc(f64, n_bootstrap);
    defer allocator.free(bootstrap_means);
    
    const temp = try allocator.alloc(u64, samples.len);
    defer allocator.free(temp);
    
    for (bootstrap_means) |*bm| {
        // สุ่มตัวอย่าง with replacement
        for (temp) |*t| {
            t.* = samples[random.intRangeLessThan(usize, 0, samples.len)];
        }
        
        var sum: u128 = 0;
        for (temp) |t| sum += t;
        bm.* = @as(f64, @floatFromInt(sum)) / @as(f64, @floatFromInt(temp.len));
    }
    
    std.mem.sort(f64, bootstrap_means, {}, std.sort.asc(f64));
    
    const alpha = 1.0 - confidence;
    const lower_idx = @as(usize, @intFromFloat(alpha / 2.0 * @as(f64, @floatFromInt(n_bootstrap))));
    const upper_idx = @as(usize, @intFromFloat((1.0 - alpha / 2.0) * @as(f64, @floatFromInt(n_bootstrap))));
    
    return .{
        .lower = bootstrap_means[@min(lower_idx, n_bootstrap - 1)],
        .upper = bootstrap_means[@min(upper_idx, n_bootstrap - 1)],
    };
}
```

---

## Step 955: Memory Benchmarking

### วัดการใช้ Memory

```zig
// memory_benchmarking.zig
const std = @import("std");

// Counting allocator - นับจำนวน allocations
pub const CountingAllocator = struct {
    parent_allocator: std.mem.Allocator,
    alloc_count: usize = 0,
    dealloc_count: usize = 0,
    total_bytes_allocated: usize = 0,
    peak_bytes: usize = 0,
    current_bytes: usize = 0,
    
    pub fn allocator(self: *CountingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .remap = remap,
                .free = free,
            },
        };
    }
    
    fn alloc(
        ctx: *anyopaque,
        len: usize,
        ptr_align: std.mem.Alignment,
        ret_addr: usize,
    ) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.parent_allocator.rawAlloc(len, ptr_align, ret_addr);
        if (result != null) {
            self.alloc_count += 1;
            self.total_bytes_allocated += len;
            self.current_bytes += len;
            if (self.current_bytes > self.peak_bytes) {
                self.peak_bytes = self.current_bytes;
            }
        }
        return result;
    }
    
    fn resize(
        ctx: *anyopaque,
        buf: []u8,
        buf_align: std.mem.Alignment,
        new_len: usize,
        ret_addr: usize,
    ) bool {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        const result = self.parent_allocator.rawResize(buf, buf_align, new_len, ret_addr);
        if (result) {
            if (new_len > buf.len) {
                const diff = new_len - buf.len;
                self.total_bytes_allocated += diff;
                self.current_bytes += diff;
            } else {
                self.current_bytes -= buf.len - new_len;
            }
            if (self.current_bytes > self.peak_bytes) {
                self.peak_bytes = self.current_bytes;
            }
        }
        return result;
    }
    
    fn remap(
        ctx: *anyopaque,
        buf: []u8,
        buf_align: std.mem.Alignment,
        new_len: usize,
        ret_addr: usize,
    ) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        return self.parent_allocator.rawRemap(buf, buf_align, new_len, ret_addr);
    }
    
    fn free(
        ctx: *anyopaque,
        buf: []u8,
        buf_align: std.mem.Alignment,
        ret_addr: usize,
    ) void {
        const self: *CountingAllocator = @ptrCast(@alignCast(ctx));
        self.dealloc_count += 1;
        self.current_bytes -= buf.len;
        self.parent_allocator.rawFree(buf, buf_align, ret_addr);
    }
    
    pub fn reset(self: *CountingAllocator) void {
        self.alloc_count = 0;
        self.dealloc_count = 0;
        self.total_bytes_allocated = 0;
        self.peak_bytes = 0;
        self.current_bytes = 0;
    }
    
    pub fn report(self: *const CountingAllocator, name: []const u8) void {
        std.debug.print("\nMemory Report: {s}\n", .{name});
        std.debug.print("  Allocations:    {d}\n", .{self.alloc_count});
        std.debug.print("  Deallocations:  {d}\n", .{self.dealloc_count});
        std.debug.print("  Total allocated:{d} bytes ({d:.2} KB)\n", .{
            self.total_bytes_allocated,
            @as(f64, @floatFromInt(self.total_bytes_allocated)) / 1024.0,
        });
        std.debug.print("  Peak usage:     {d} bytes ({d:.2} KB)\n", .{
            self.peak_bytes,
            @as(f64, @floatFromInt(self.peak_bytes)) / 1024.0,
        });
        std.debug.print("  Current usage:  {d} bytes\n", .{self.current_bytes});
        if (self.alloc_count != self.dealloc_count) {
            std.debug.print("  WARNING: Memory leak! {d} allocations not freed\n", .{
                self.alloc_count - self.dealloc_count,
            });
        }
    }
};

pub fn main() !void {
    var base_allocator = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = base_allocator.deinit();
    
    var counting = CountingAllocator{
        .parent_allocator = base_allocator.allocator(),
    };
    const allocator = counting.allocator();
    
    // วัด memory สำหรับ ArrayList
    {
        var list = std.ArrayList(u64).init(allocator);
        defer list.deinit();
        
        for (0..10_000) |i| {
            try list.append(i);
        }
    }
    
    counting.report("ArrayList 10,000 elements");
    counting.reset();
    
    // วัด memory สำหรับ HashMap
    {
        var map = std.AutoHashMap(u64, u64).init(allocator);
        defer map.deinit();
        
        for (0..1_000) |i| {
            try map.put(i, i * i);
        }
    }
    
    counting.report("HashMap 1,000 entries");
}
```

---

## Step 956: CPU Profiling Integration

### การ integrate กับ CPU Profiler

```zig
// cpu_profiling.zig
const std = @import("std");
const builtin = @import("builtin");

// Simple profiling markers สำหรับ external profilers
pub const Profiler = struct {
    // สำหรับ Linux perf
    pub fn beginSection(name: []const u8) void {
        if (builtin.os.tag == .linux) {
            // ใช้ USDT probes หรือ perf_event_open
            _ = name;
        }
    }
    
    pub fn endSection(name: []const u8) void {
        if (builtin.os.tag == .linux) {
            _ = name;
        }
    }
};

// Manual call stack tracking
pub const CallTracer = struct {
    const MAX_DEPTH = 64;
    const MAX_NAME_LEN = 64;
    
    stack: [MAX_DEPTH]Frame = undefined,
    depth: usize = 0,
    allocator: std.mem.Allocator,
    records: std.StringHashMap(Record),
    
    const Frame = struct {
        name: [MAX_NAME_LEN]u8,
        name_len: usize,
        start_ns: u64,
    };
    
    const Record = struct {
        call_count: u64,
        total_ns: u64,
        min_ns: u64,
        max_ns: u64,
    };
    
    pub fn init(allocator: std.mem.Allocator) CallTracer {
        return .{
            .allocator = allocator,
            .records = std.StringHashMap(Record).init(allocator),
        };
    }
    
    pub fn deinit(self: *CallTracer) void {
        self.records.deinit();
    }
    
    pub fn enter(self: *CallTracer, name: []const u8) void {
        if (self.depth >= MAX_DEPTH) return;
        
        var frame = &self.stack[self.depth];
        const len = @min(name.len, MAX_NAME_LEN);
        @memcpy(frame.name[0..len], name[0..len]);
        frame.name_len = len;
        frame.start_ns = std.time.nanoTimestamp();
        self.depth += 1;
    }
    
    pub fn exit(self: *CallTracer) void {
        if (self.depth == 0) return;
        
        self.depth -= 1;
        const frame = &self.stack[self.depth];
        const elapsed: u64 = @intCast(std.time.nanoTimestamp() - @as(i128, @intCast(frame.start_ns)));
        const name = frame.name[0..frame.name_len];
        
        if (self.records.getPtr(name)) |record| {
            record.call_count += 1;
            record.total_ns += elapsed;
            if (elapsed < record.min_ns) record.min_ns = elapsed;
            if (elapsed > record.max_ns) record.max_ns = elapsed;
        } else {
            self.records.put(name, .{
                .call_count = 1,
                .total_ns = elapsed,
                .min_ns = elapsed,
                .max_ns = elapsed,
            }) catch {};
        }
    }
    
    pub fn report(self: *const CallTracer) void {
        std.debug.print("\n{'='<70}\n", .{});
        std.debug.print("{s:<30} {s:>10} {s:>12} {s:>12} {s:>12}\n", .{
            "Function", "Calls", "Total(ns)", "Avg(ns)", "Max(ns)",
        });
        std.debug.print("{'='<70}\n", .{});
        
        var it = self.records.iterator();
        while (it.next()) |entry| {
            const r = entry.value_ptr.*;
            const avg = r.total_ns / r.call_count;
            std.debug.print("{s:<30} {d:>10} {d:>12} {d:>12} {d:>12}\n", .{
                entry.key_ptr.*,
                r.call_count,
                r.total_ns,
                avg,
                r.max_ns,
            });
        }
    }
};

// ตัวอย่างการใช้งาน
var global_tracer: ?*CallTracer = null;

fn tracedFunction(tracer: *CallTracer, depth: u32) void {
    tracer.enter("tracedFunction");
    defer tracer.exit();
    
    if (depth > 0) {
        tracedFunction(tracer, depth - 1);
        tracedFunction(tracer, depth - 1);
    }
    
    // simulate work
    var x: u64 = 0;
    for (0..1000) |i| x += i;
    std.mem.doNotOptimizeAway(x);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var tracer = CallTracer.init(gpa.allocator());
    defer tracer.deinit();
    
    tracedFunction(&tracer, 5);
    tracer.report();
}
```

---

## Step 957: Benchmark Comparison

### เปรียบเทียบ Implementations

```zig
// benchmark_comparison.zig
const std = @import("std");

// ฟังก์ชันที่จะเปรียบเทียบ

// Implementation A: Linear search
fn linearSearch(haystack: []const u32, needle: u32) ?usize {
    for (haystack, 0..) |v, i| {
        if (v == needle) return i;
    }
    return null;
}

// Implementation B: Binary search (ต้อง sorted)
fn binarySearch(haystack: []const u32, needle: u32) ?usize {
    var low: usize = 0;
    var high: usize = haystack.len;
    
    while (low < high) {
        const mid = low + (high - low) / 2;
        if (haystack[mid] == needle) return mid;
        if (haystack[mid] < needle) {
            low = mid + 1;
        } else {
            high = mid;
        }
    }
    return null;
}

// Implementation C: Hash map lookup
fn hashLookup(map: *const std.AutoHashMap(u32, usize), needle: u32) ?usize {
    return map.get(needle);
}

pub const ComparisonResult = struct {
    name_a: []const u8,
    name_b: []const u8,
    time_a_ns: u64,
    time_b_ns: u64,
    
    pub fn print(self: ComparisonResult) void {
        const ratio = @as(f64, @floatFromInt(self.time_a_ns)) / @as(f64, @floatFromInt(self.time_b_ns));
        
        std.debug.print("\n=== Comparison ===\n", .{});
        std.debug.print("{s}: {d} ns\n", .{ self.name_a, self.time_a_ns });
        std.debug.print("{s}: {d} ns\n", .{ self.name_b, self.time_b_ns });
        
        if (ratio > 1.0) {
            std.debug.print("{s} is {d:.2}x faster than {s}\n", .{
                self.name_b,
                ratio,
                self.name_a,
            });
        } else {
            std.debug.print("{s} is {d:.2}x faster than {s}\n", .{
                self.name_a,
                1.0 / ratio,
                self.name_b,
            });
        }
    }
};

fn measureOps(
    comptime func: anytype,
    args: anytype,
    iters: u32,
) u64 {
    // Warm up
    for (0..10) |_| _ = @call(.auto, func, args);
    
    var timer = std.time.Timer.start() catch unreachable;
    for (0..iters) |_| {
        const r = @call(.auto, func, args);
        std.mem.doNotOptimizeAway(r);
    }
    return timer.read() / iters;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const SIZE = 10_000;
    const ITERS = 10_000;
    
    // สร้าง data
    var data: [SIZE]u32 = undefined;
    for (&data, 0..) |*v, i| v.* = @as(u32, @intCast(i * 2)); // even numbers
    const needle: u32 = SIZE - 1; // ค้นหาตัวที่ไม่มีใน array (worst case)
    
    // สร้าง hash map
    var map = std.AutoHashMap(u32, usize).init(allocator);
    defer map.deinit();
    for (data, 0..) |v, i| try map.put(v, i);
    
    // วัด linear search
    const linear_time = measureOps(linearSearch, .{ &data, needle }, ITERS);
    
    // วัด binary search
    const binary_time = measureOps(binarySearch, .{ &data, needle }, ITERS);
    
    // วัด hash lookup
    const hash_time = measureOps(hashLookup, .{ &map, needle }, ITERS);
    
    std.debug.print("Search Benchmark (array size: {d})\n", .{SIZE});
    std.debug.print("  Linear search:  {d} ns\n", .{linear_time});
    std.debug.print("  Binary search:  {d} ns\n", .{binary_time});
    std.debug.print("  Hash map:       {d} ns\n", .{hash_time});
    
    const speedup_binary = @as(f64, @floatFromInt(linear_time)) / @as(f64, @floatFromInt(binary_time));
    const speedup_hash = @as(f64, @floatFromInt(linear_time)) / @as(f64, @floatFromInt(hash_time));
    
    std.debug.print("\nSpeedup vs Linear:\n", .{});
    std.debug.print("  Binary search: {d:.1}x faster\n", .{speedup_binary});
    std.debug.print("  Hash map:      {d:.1}x faster\n", .{speedup_hash});
}
```

---

## Step 958: Continuous Benchmarking

### การ Setup Continuous Benchmarking

```zig
// continuous_benchmarking.zig
const std = @import("std");

// Benchmark Registry
pub const BenchmarkRegistry = struct {
    benchmarks: std.ArrayList(BenchmarkEntry),
    allocator: std.mem.Allocator,
    
    const BenchmarkEntry = struct {
        name: []const u8,
        func: *const fn (std.mem.Allocator) anyerror!BenchmarkResult,
    };
    
    pub const BenchmarkResult = struct {
        name: []const u8,
        mean_ns: f64,
        stddev_ns: f64,
        iterations: u32,
        timestamp: i64,
    };
    
    pub fn init(allocator: std.mem.Allocator) BenchmarkRegistry {
        return .{
            .benchmarks = std.ArrayList(BenchmarkEntry).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *BenchmarkRegistry) void {
        self.benchmarks.deinit();
    }
    
    pub fn register(
        self: *BenchmarkRegistry,
        name: []const u8,
        func: *const fn (std.mem.Allocator) anyerror!BenchmarkResult,
    ) !void {
        try self.benchmarks.append(.{ .name = name, .func = func });
    }
    
    pub fn runAll(self: *BenchmarkRegistry) !std.ArrayList(BenchmarkResult) {
        var results = std.ArrayList(BenchmarkResult).init(self.allocator);
        
        for (self.benchmarks.items) |entry| {
            std.debug.print("Running: {s}...\n", .{entry.name});
            const result = try entry.func(self.allocator);
            try results.append(result);
            std.debug.print("  {d:.2} ns (±{d:.2} ns)\n", .{ result.mean_ns, result.stddev_ns });
        }
        
        return results;
    }
    
    pub fn saveResults(
        results: []const BenchmarkResult,
        filename: []const u8,
        allocator: std.mem.Allocator,
    ) !void {
        const file = try std.fs.cwd().createFile(filename, .{});
        defer file.close();
        
        const writer = file.writer();
        
        // เขียน JSON format
        try writer.writeAll("[\n");
        for (results, 0..) |result, i| {
            try writer.print(
                \\  {{"name": "{s}", "mean_ns": {d:.2}, "stddev_ns": {d:.2}, "iterations": {d}, "timestamp": {d}}}
            , .{
                result.name,
                result.mean_ns,
                result.stddev_ns,
                result.iterations,
                result.timestamp,
            });
            if (i < results.len - 1) try writer.writeAll(",\n");
        }
        try writer.writeAll("\n]\n");
        
        _ = allocator;
    }
};

// ตัวอย่าง benchmark function
fn benchmarkStringCopy(allocator: std.mem.Allocator) !BenchmarkRegistry.BenchmarkResult {
    const ITERS: u32 = 10_000;
    const src = "Hello, World! This is a test string for benchmarking.";
    
    // Warm up
    for (0..100) |_| {
        const copy = try allocator.dupe(u8, src);
        allocator.free(copy);
    }
    
    var timer = try std.time.Timer.start();
    var samples: [ITERS]u64 = undefined;
    
    for (&samples) |*s| {
        timer.reset();
        const copy = try allocator.dupe(u8, src);
        s.* = timer.read();
        allocator.free(copy);
    }
    
    // คำนวณ statistics
    var sum: u128 = 0;
    for (samples) |s| sum += s;
    const mean = @as(f64, @floatFromInt(sum)) / ITERS;
    
    var var_sum: f64 = 0;
    for (samples) |s| {
        const diff = @as(f64, @floatFromInt(s)) - mean;
        var_sum += diff * diff;
    }
    const stddev = @sqrt(var_sum / ITERS);
    
    return .{
        .name = "String Copy",
        .mean_ns = mean,
        .stddev_ns = stddev,
        .iterations = ITERS,
        .timestamp = std.time.timestamp(),
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var registry = BenchmarkRegistry.init(allocator);
    defer registry.deinit();
    
    try registry.register("String Copy", benchmarkStringCopy);
    
    const results = try registry.runAll();
    defer results.deinit();
    
    try BenchmarkRegistry.saveResults(
        results.items,
        "/tmp/benchmark_results.json",
        allocator,
    );
    
    std.debug.print("\nResults saved to /tmp/benchmark_results.json\n", .{});
}
```

---

## Step 959: Benchmark Reporting

### การสร้าง Benchmark Reports

```zig
// benchmark_reporting.zig
const std = @import("std");

pub const Reporter = struct {
    pub const Format = enum { text, json, csv, markdown };
    
    pub fn report(
        results: []const Result,
        format: Format,
        writer: anytype,
    ) !void {
        switch (format) {
            .text => try reportText(results, writer),
            .json => try reportJson(results, writer),
            .csv => try reportCsv(results, writer),
            .markdown => try reportMarkdown(results, writer),
        }
    }
    
    pub const Result = struct {
        name: []const u8,
        mean_ns: f64,
        stddev_ns: f64,
        min_ns: f64,
        max_ns: f64,
        iterations: u32,
        ops_per_sec: f64,
    };
    
    fn reportText(results: []const Result, writer: anytype) !void {
        try writer.writeAll("\nBenchmark Results\n");
        try writer.writeAll("=================\n\n");
        
        for (results) |r| {
            try writer.print("{s}:\n", .{r.name});
            try writer.print("  Mean:     {d:.2} ns\n", .{r.mean_ns});
            try writer.print("  Std Dev:  {d:.2} ns\n", .{r.stddev_ns});
            try writer.print("  Min:      {d:.2} ns\n", .{r.min_ns});
            try writer.print("  Max:      {d:.2} ns\n", .{r.max_ns});
            try writer.print("  Ops/sec:  {d:.0}\n", .{r.ops_per_sec});
            try writer.print("  Iters:    {d}\n\n", .{r.iterations});
        }
    }
    
    fn reportJson(results: []const Result, writer: anytype) !void {
        try writer.writeAll("[\n");
        for (results, 0..) |r, i| {
            try writer.print(
                \\  {{"name": "{s}", "mean_ns": {d:.2}, "stddev_ns": {d:.2}, "min_ns": {d:.2}, "max_ns": {d:.2}, "ops_per_sec": {d:.0}, "iterations": {d}}}
            , .{
                r.name, r.mean_ns, r.stddev_ns, r.min_ns, r.max_ns, r.ops_per_sec, r.iterations,
            });
            if (i < results.len - 1) try writer.writeAll(",\n");
        }
        try writer.writeAll("\n]\n");
    }
    
    fn reportCsv(results: []const Result, writer: anytype) !void {
        try writer.writeAll("name,mean_ns,stddev_ns,min_ns,max_ns,ops_per_sec,iterations\n");
        for (results) |r| {
            try writer.print("{s},{d:.2},{d:.2},{d:.2},{d:.2},{d:.0},{d}\n", .{
                r.name, r.mean_ns, r.stddev_ns, r.min_ns, r.max_ns, r.ops_per_sec, r.iterations,
            });
        }
    }
    
    fn reportMarkdown(results: []const Result, writer: anytype) !void {
        try writer.writeAll("# Benchmark Results\n\n");
        try writer.writeAll("| Name | Mean (ns) | Std Dev (ns) | Min (ns) | Max (ns) | Ops/sec |\n");
        try writer.writeAll("|------|-----------|-------------|----------|----------|--------|\n");
        for (results) |r| {
            try writer.print("| {s} | {d:.2} | {d:.2} | {d:.2} | {d:.2} | {d:.0} |\n", .{
                r.name, r.mean_ns, r.stddev_ns, r.min_ns, r.max_ns, r.ops_per_sec,
            });
        }
    }
};

pub fn main() !void {
    const results = [_]Reporter.Result{
        .{
            .name = "Array Sort (1000)",
            .mean_ns = 45231.5,
            .stddev_ns = 1234.7,
            .min_ns = 43100.0,
            .max_ns = 52000.0,
            .iterations = 10000,
            .ops_per_sec = 22110.5,
        },
        .{
            .name = "Hash Map Insert",
            .mean_ns = 125.3,
            .stddev_ns = 18.2,
            .min_ns = 105.0,
            .max_ns = 200.0,
            .iterations = 100000,
            .ops_per_sec = 7981000.0,
        },
    };
    
    const stdout = std.io.getStdOut().writer();
    
    std.debug.print("=== Text Format ===\n", .{});
    try Reporter.report(&results, .text, stdout);
    
    std.debug.print("\n=== Markdown Format ===\n", .{});
    try Reporter.report(&results, .markdown, stdout);
    
    std.debug.print("\n=== CSV Format ===\n", .{});
    try Reporter.report(&results, .csv, stdout);
}
```

---

## Step 960: Complete Benchmark Suite

### Benchmark Suite ครบถ้วน

```zig
// complete_benchmark_suite.zig
const std = @import("std");

// ============ Benchmark Framework ============

pub const BenchmarkSuite = struct {
    allocator: std.mem.Allocator,
    benchmarks: std.ArrayList(BenchmarkDef),
    results: std.ArrayList(BenchmarkResult),
    config: Config,
    
    pub const Config = struct {
        warmup_iters: u32 = 100,
        measure_iters: u32 = 1000,
        time_limit_ns: u64 = 5_000_000_000, // 5 seconds max
        filter: ?[]const u8 = null,
    };
    
    const BenchmarkDef = struct {
        name: []const u8,
        group: []const u8,
        func: *const fn (*BenchmarkContext) anyerror!void,
    };
    
    pub const BenchmarkResult = struct {
        name: []const u8,
        group: []const u8,
        stats: BenchmarkStats,
    };
    
    pub const BenchmarkStats = struct {
        iterations: u32,
        mean_ns: f64,
        median_ns: f64,
        p95_ns: f64,
        p99_ns: f64,
        stddev_ns: f64,
        min_ns: u64,
        max_ns: u64,
        ops_per_sec: f64,
        bytes_per_sec: ?f64 = null,
    };
    
    pub fn init(allocator: std.mem.Allocator, config: Config) BenchmarkSuite {
        return .{
            .allocator = allocator,
            .benchmarks = std.ArrayList(BenchmarkDef).init(allocator),
            .results = std.ArrayList(BenchmarkResult).init(allocator),
            .config = config,
        };
    }
    
    pub fn deinit(self: *BenchmarkSuite) void {
        self.benchmarks.deinit();
        self.results.deinit();
    }
    
    pub fn add(
        self: *BenchmarkSuite,
        group: []const u8,
        name: []const u8,
        func: *const fn (*BenchmarkContext) anyerror!void,
    ) !void {
        try self.benchmarks.append(.{
            .name = name,
            .group = group,
            .func = func,
        });
    }
    
    pub fn run(self: *BenchmarkSuite) !void {
        for (self.benchmarks.items) |bench| {
            // Filter
            if (self.config.filter) |filter| {
                if (std.mem.indexOf(u8, bench.name, filter) == null and
                    std.mem.indexOf(u8, bench.group, filter) == null) continue;
            }
            
            std.debug.print("Running [{s}] {s}...\n", .{ bench.group, bench.name });
            
            var ctx = BenchmarkContext{
                .allocator = self.allocator,
            };
            
            // Warmup
            var wi: u32 = 0;
            while (wi < self.config.warmup_iters) : (wi += 1) {
                try bench.func(&ctx);
            }
            
            // Measure
            var samples = try self.allocator.alloc(u64, self.config.measure_iters);
            defer self.allocator.free(samples);
            
            var timer = try std.time.Timer.start();
            var total_time: u64 = 0;
            
            for (samples) |*sample| {
                timer.reset();
                try bench.func(&ctx);
                const t = timer.read();
                sample.* = t;
                total_time += t;
                
                if (total_time >= self.config.time_limit_ns) break;
            }
            
            const stats = computeStats(samples);
            
            try self.results.append(.{
                .name = bench.name,
                .group = bench.group,
                .stats = stats,
            });
        }
    }
    
    fn computeStats(samples: []const u64) BenchmarkStats {
        var min = samples[0];
        var max = samples[0];
        var sum: u128 = 0;
        
        for (samples) |s| {
            if (s < min) min = s;
            if (s > max) max = s;
            sum += s;
        }
        
        const n = @as(f64, @floatFromInt(samples.len));
        const mean = @as(f64, @floatFromInt(sum)) / n;
        
        var var_sum: f64 = 0;
        for (samples) |s| {
            const diff = @as(f64, @floatFromInt(s)) - mean;
            var_sum += diff * diff;
        }
        const stddev = @sqrt(var_sum / n);
        
        // Percentiles (simplified)
        const median = mean;
        const p95 = mean + 1.645 * stddev;
        const p99 = mean + 2.326 * stddev;
        
        const ops_per_sec = 1_000_000_000.0 / mean;
        
        return .{
            .iterations = @as(u32, @intCast(samples.len)),
            .mean_ns = mean,
            .median_ns = median,
            .p95_ns = p95,
            .p99_ns = p99,
            .stddev_ns = stddev,
            .min_ns = min,
            .max_ns = max,
            .ops_per_sec = ops_per_sec,
        };
    }
    
    pub fn printSummary(self: *const BenchmarkSuite) void {
        std.debug.print("\n{'='<80}\n", .{});
        std.debug.print("BENCHMARK SUMMARY\n", .{});
        std.debug.print("{'='<80}\n\n", .{});
        
        var current_group: []const u8 = "";
        
        for (self.results.items) |result| {
            if (!std.mem.eql(u8, result.group, current_group)) {
                std.debug.print("\n[{s}]\n", .{result.group});
                current_group = result.group;
            }
            
            const s = result.stats;
            std.debug.print("  {s:<40} {d:>10.2} ns  {d:>12.0} ops/s\n", .{
                result.name,
                s.mean_ns,
                s.ops_per_sec,
            });
        }
        
        std.debug.print("\n{'='<80}\n", .{});
    }
};

pub const BenchmarkContext = struct {
    allocator: std.mem.Allocator,
    bytes_processed: u64 = 0,
};

// ============ Benchmark Definitions ============

fn benchmarkSort(ctx: *BenchmarkContext) !void {
    var arr: [1000]u32 = undefined;
    var prng = std.Random.DefaultPrng.init(42);
    for (&arr) |*v| v.* = prng.random().int(u32);
    
    std.mem.sort(u32, &arr, {}, std.sort.asc(u32));
    std.mem.doNotOptimizeAway(arr[0]);
    ctx.bytes_processed += arr.len * @sizeOf(u32);
}

fn benchmarkStringSearch(ctx: *BenchmarkContext) !void {
    const haystack = "The quick brown fox jumps over the lazy dog";
    const needle = "fox";
    const result = std.mem.indexOf(u8, haystack, needle);
    std.mem.doNotOptimizeAway(result);
    ctx.bytes_processed += haystack.len;
}

fn benchmarkHashMapOps(ctx: *BenchmarkContext) !void {
    var map = std.AutoHashMap(u32, u32).init(ctx.allocator);
    defer map.deinit();
    
    for (0..100) |i| {
        try map.put(@as(u32, @intCast(i)), @as(u32, @intCast(i * i)));
    }
    
    for (0..100) |i| {
        const v = map.get(@as(u32, @intCast(i)));
        std.mem.doNotOptimizeAway(v);
    }
}

fn benchmarkMemCopy(ctx: *BenchmarkContext) !void {
    const SIZE = 64 * 1024;
    const src = try ctx.allocator.alloc(u8, SIZE);
    defer ctx.allocator.free(src);
    const dst = try ctx.allocator.alloc(u8, SIZE);
    defer ctx.allocator.free(dst);
    
    @memcpy(dst, src);
    std.mem.doNotOptimizeAway(dst[0]);
    ctx.bytes_processed += SIZE;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var suite = BenchmarkSuite.init(allocator, .{
        .warmup_iters = 50,
        .measure_iters = 500,
    });
    defer suite.deinit();
    
    // ลงทะเบียน benchmarks
    try suite.add("Algorithms", "Sort 1000 elements", benchmarkSort);
    try suite.add("Strings", "String search", benchmarkStringSearch);
    try suite.add("Data Structures", "HashMap 100 ops", benchmarkHashMapOps);
    try suite.add("Memory", "64KB memcpy", benchmarkMemCopy);
    
    // รัน benchmarks
    try suite.run();
    
    // แสดงผล
    suite.printSummary();
}
```

---

## สรุป Part 96

ใน Part นี้เราได้เรียนรู้เกี่ยวกับ Benchmarking ใน Zig ครบถ้วน:

- **Step 951**: แนวคิดพื้นฐานของ benchmarking, ประเภทต่างๆ และหลักการที่สำคัญ
- **Step 952**: การใช้งาน `std.time.Timer` สำหรับการวัดเวลาที่แม่นยำ
- **Step 953**: Pitfalls ที่ต้องระวัง เช่น compiler optimization, cache effects, branch prediction
- **Step 954**: การวิเคราะห์ผลทางสถิติ รวมถึง percentiles, standard deviation, confidence intervals
- **Step 955**: การวัดการใช้ memory ด้วย Counting Allocator
- **Step 956**: การ integrate กับ CPU profilers และการทำ call tracing
- **Step 957**: เทคนิคการเปรียบเทียบ implementations ต่างๆ
- **Step 958**: การตั้งค่า continuous benchmarking workflow
- **Step 959**: การสร้าง benchmark reports ในรูปแบบต่างๆ (text, JSON, CSV, Markdown)
- **Step 960**: การสร้าง complete benchmark suite ที่ใช้งานได้จริง

การทำ benchmarking ที่ดีต้องอาศัยทั้งความรู้ทางสถิติและความเข้าใจเกี่ยวกับ hardware เพื่อให้ผลลัพธ์ที่น่าเชื่อถือ
