# ส่วนที่ 33: std.math - Mathematical Operations

## ภาพรวมของบทเรียน

`std.math` ใน Zig ให้ฟังก์ชันทางคณิตศาสตร์ครบครัน ตั้งแต่ constants พื้นฐาน ไปจนถึง trigonometry, logarithms, hyperbolic functions และการจัดการค่า special float values ทั้งหมดออกแบบให้ type-safe และ compile-time computable

---

## Step 321: Mathematical Constants (pi, e, tau)

### Constants ใน std.math

```zig
const std = @import("std");
const math = std.math;

pub fn mathConstants() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Pi (π) ≈ 3.14159265...
    try stdout.print("π (pi)  = {d:.15}\n", .{math.pi});
    
    // Tau (τ) = 2π ≈ 6.28318530...
    try stdout.print("τ (tau) = {d:.15}\n", .{math.tau});
    
    // Euler's number (e) ≈ 2.71828182...
    try stdout.print("e       = {d:.15}\n", .{math.e});
    
    // Square root of 2 ≈ 1.41421356...
    try stdout.print("√2      = {d:.15}\n", .{math.sqrt2});
    
    // Natural log of 2 ≈ 0.69314718...
    try stdout.print("ln(2)   = {d:.15}\n", .{math.ln2});
    
    // Natural log of 10 ≈ 2.30258509...
    try stdout.print("ln(10)  = {d:.15}\n", .{math.ln10});
    
    // Log base 2 of e ≈ 1.44269504...
    try stdout.print("log2(e) = {d:.15}\n", .{math.log2e});
    
    // Log base 10 of e ≈ 0.43429448...
    try stdout.print("log10(e)= {d:.15}\n", .{math.log10e});
    
    // Phi (Golden Ratio) ≈ 1.61803398...
    try stdout.print("φ (phi) = {d:.15}\n", .{math.phi});
}
```

### การใช้ Constants ใน Calculations

```zig
const std = @import("std");
const math = std.math;

pub fn circleCalculations(radius: f64) !void {
    const stdout = std.io.getStdOut().writer();
    
    // พื้นที่วงกลม = π × r²
    const area = math.pi * radius * radius;
    
    // เส้นรอบวง = 2π × r = τ × r
    const circumference = math.tau * radius;
    
    // ปริมาตรทรงกลม = (4/3)π × r³
    const sphere_volume = (4.0 / 3.0) * math.pi * radius * radius * radius;
    
    // พื้นที่ผิวทรงกลม = 4π × r²
    const sphere_surface = 4.0 * math.pi * radius * radius;
    
    try stdout.print("วงกลมรัศมี {d:.2}:\n", .{radius});
    try stdout.print("  พื้นที่ = {d:.6}\n", .{area});
    try stdout.print("  เส้นรอบวง = {d:.6}\n", .{circumference});
    try stdout.print("  ปริมาตรทรงกลม = {d:.6}\n", .{sphere_volume});
    try stdout.print("  พื้นที่ผิวทรงกลม = {d:.6}\n", .{sphere_surface});
}

pub fn degreesToRadians(degrees: f64) f64 {
    return degrees * math.pi / 180.0;
}

pub fn radiansToDegrees(radians: f64) f64 {
    return radians * 180.0 / math.pi;
}

pub fn angleConversions() !void {
    const stdout = std.io.getStdOut().writer();
    
    const angles_deg = [_]f64{ 0, 30, 45, 60, 90, 120, 180, 270, 360 };
    
    try stdout.print("การแปลงมุม:\n", .{});
    try stdout.print("{s:>8} | {s:>12} | {s:>12}\n", .{
        "Degrees",
        "Radians",
        "π fraction",
    });
    try stdout.print("-" .* 36 ++ "\n", .{});
    
    for (angles_deg) |deg| {
        const rad = degreesToRadians(deg);
        const pi_fraction = rad / math.pi;
        try stdout.print("{d:>8.1} | {d:>12.6} | {d:>12.6}π\n", .{
            deg,
            rad,
            pi_fraction,
        });
    }
}
```

---

## Step 322: Basic Math (abs, floor, ceil, round)

### Absolute Value

```zig
const std = @import("std");
const math = std.math;

pub fn absoluteValue() !void {
    const stdout = std.io.getStdOut().writer();
    
    // abs สำหรับ integers
    const int_pos = math.absCast(-42);
    const int_neg = math.absCast(42);
    try stdout.print("abs(-42) = {d}\n", .{int_pos});
    try stdout.print("abs(42)  = {d}\n", .{int_neg});
    
    // abs สำหรับ floats
    const float_neg: f64 = -3.14;
    const float_pos: f64 = 3.14;
    try stdout.print("fabs(-3.14) = {d}\n", .{@abs(float_neg)});
    try stdout.print("fabs(3.14)  = {d}\n", .{@abs(float_pos)});
    
    // ตัวอย่างการใช้งาน: distance
    const x1: f64 = 3.0;
    const x2: f64 = 7.0;
    const distance = @abs(x2 - x1);
    try stdout.print("ระยะห่างระหว่าง {d} และ {d} = {d}\n", .{ x1, x2, distance });
}
```

### Floor, Ceil, Round

```zig
const std = @import("std");
const math = std.math;

pub fn roundingFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    const values = [_]f64{ -2.7, -2.5, -2.3, -1.0, 0.0, 1.3, 1.5, 1.7, 2.0 };
    
    try stdout.print("{s:>8} | {s:>8} | {s:>8} | {s:>8} | {s:>8}\n", .{
        "Value",
        "floor",
        "ceil",
        "round",
        "trunc",
    });
    try stdout.print("-" .* 44 ++ "\n", .{});
    
    for (values) |v| {
        try stdout.print("{d:>8.2} | {d:>8.1} | {d:>8.1} | {d:>8.1} | {d:>8.1}\n", .{
            v,
            math.floor(v),
            math.ceil(v),
            math.round(v),
            math.trunc(v),
        });
    }
    
    // Floor: ปัดลง (ต่ำสุดที่ >= value)
    try stdout.print("\nFloor ปัดลงเสมอ:\n", .{});
    try stdout.print("  floor(2.9) = {d}\n", .{math.floor(2.9)});
    try stdout.print("  floor(-2.1) = {d}\n", .{math.floor(-2.1)});
    
    // Ceil: ปัดขึ้น (สูงสุดที่ <= value)
    try stdout.print("\nCeil ปัดขึ้นเสมอ:\n", .{});
    try stdout.print("  ceil(2.1) = {d}\n", .{math.ceil(2.1)});
    try stdout.print("  ceil(-2.9) = {d}\n", .{math.ceil(-2.9)});
    
    // Round: ปัดตามปกติ (0.5 ขึ้น)
    try stdout.print("\nRound ปัดตามปกติ:\n", .{});
    try stdout.print("  round(2.5) = {d}\n", .{math.round(2.5)});
    try stdout.print("  round(2.4) = {d}\n", .{math.round(2.4)});
    try stdout.print("  round(-2.5) = {d}\n", .{math.round(-2.5)});
}
```

### Min และ Max

```zig
const std = @import("std");
const math = std.math;

pub fn minMaxOperations() !void {
    const stdout = std.io.getStdOut().writer();
    
    // min/max สำหรับ scalars
    try stdout.print("min(3, 7) = {d}\n", .{@min(3, 7)});
    try stdout.print("max(3, 7) = {d}\n", .{@max(3, 7)});
    
    // min/max สำหรับ floats
    try stdout.print("min(3.14, 2.71) = {d}\n", .{@min(3.14, 2.71)});
    try stdout.print("max(3.14, 2.71) = {d}\n", .{@max(3.14, 2.71)});
    
    // Clamp: จำกัดค่าในช่วง [min, max]
    const value: f64 = 15.0;
    const clamped = math.clamp(value, 0.0, 10.0);
    try stdout.print("clamp(15.0, 0, 10) = {d}\n", .{clamped});
    
    // หา min/max ใน array
    const numbers = [_]i32{ 42, -7, 100, 13, 55, -99, 0 };
    var min_val = numbers[0];
    var max_val = numbers[0];
    
    for (numbers[1..]) |n| {
        min_val = @min(min_val, n);
        max_val = @max(max_val, n);
    }
    
    try stdout.print("\nใน array {any}:\n", .{numbers});
    try stdout.print("  Min = {d}\n", .{min_val});
    try stdout.print("  Max = {d}\n", .{max_val});
}
```

---

## Step 323: Power and Roots (pow, sqrt, cbrt)

### Power Functions

```zig
const std = @import("std");
const math = std.math;

pub fn powerFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // pow: x^y
    try stdout.print("2^10 = {d}\n", .{math.pow(f64, 2.0, 10.0)});
    try stdout.print("3^3  = {d}\n", .{math.pow(f64, 3.0, 3.0)});
    try stdout.print("2^0.5 = {d:.6}\n", .{math.pow(f64, 2.0, 0.5)});
    try stdout.print("(-1)^2 = {d}\n", .{math.pow(f64, -1.0, 2.0)});
    
    // powi: x^n (integer exponent)
    try stdout.print("\n2^8 (powi) = {d}\n", .{math.powi(i64, 2, 8) catch 0});
    try stdout.print("3^5 (powi) = {d}\n", .{math.powi(i64, 3, 5) catch 0});
    
    // Overflow detection
    const result = math.powi(i32, 2, 31);
    if (result) |val| {
        try stdout.print("2^31 = {d}\n", .{val});
    } else |err| {
        try stdout.print("2^31 overflow: {}\n", .{err});
    }
    
    // Integer powers
    try stdout.print("\nInteger powers of 2:\n", .{});
    var i: u32 = 0;
    while (i <= 10) : (i += 1) {
        const p = math.powi(u64, 2, i) catch break;
        try stdout.print("  2^{d} = {d}\n", .{ i, p });
    }
}
```

### Square Root และ Cube Root

```zig
const std = @import("std");
const math = std.math;

pub fn rootFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // sqrt: square root
    try stdout.print("√2   = {d:.10}\n", .{@sqrt(2.0)});
    try stdout.print("√4   = {d:.10}\n", .{@sqrt(4.0)});
    try stdout.print("√9   = {d:.10}\n", .{@sqrt(9.0)});
    try stdout.print("√100 = {d:.10}\n", .{@sqrt(100.0)});
    try stdout.print("√2   = {d:.10}\n", .{math.sqrt(2.0)});
    
    // cbrt: cube root
    try stdout.print("\n∛8   = {d:.10}\n", .{math.cbrt(8.0)});
    try stdout.print("∛27  = {d:.10}\n", .{math.cbrt(27.0)});
    try stdout.print("∛-8  = {d:.10}\n", .{math.cbrt(-8.0)});
    try stdout.print("∛2   = {d:.10}\n", .{math.cbrt(2.0)});
    
    // Pythagorean theorem
    const a: f64 = 3.0;
    const b: f64 = 4.0;
    const hypotenuse = math.hypot(a, b);
    try stdout.print("\nทฤษฎีบทพีทาโกรัส: a={d}, b={d}\n", .{ a, b });
    try stdout.print("  c = √(a² + b²) = {d}\n", .{hypotenuse});
    
    // Integer square root
    try stdout.print("\nInteger sqrt:\n", .{});
    const values = [_]u64{ 0, 1, 4, 9, 16, 25, 100, 144 };
    for (values) |v| {
        const sq = math.sqrt(v);
        try stdout.print("  isqrt({d}) = {d}\n", .{ v, sq });
    }
}
```

---

## Step 324: Logarithms (log, log2, log10, ln)

### Natural Logarithm

```zig
const std = @import("std");
const math = std.math;

pub fn logarithmFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Natural log (ln) - log base e
    try stdout.print("Natural logarithm (ln):\n", .{});
    try stdout.print("  ln(1)    = {d:.10}\n", .{@log(1.0)});
    try stdout.print("  ln(e)    = {d:.10}\n", .{@log(math.e)});
    try stdout.print("  ln(10)   = {d:.10}\n", .{@log(10.0)});
    try stdout.print("  ln(100)  = {d:.10}\n", .{@log(100.0)});
    try stdout.print("  ln(0.5)  = {d:.10}\n", .{@log(0.5)});
    
    // Log base 2
    try stdout.print("\nLogarithm base 2 (log₂):\n", .{});
    try stdout.print("  log₂(1)  = {d:.10}\n", .{math.log2(1.0)});
    try stdout.print("  log₂(2)  = {d:.10}\n", .{math.log2(2.0)});
    try stdout.print("  log₂(4)  = {d:.10}\n", .{math.log2(4.0)});
    try stdout.print("  log₂(8)  = {d:.10}\n", .{math.log2(8.0)});
    try stdout.print("  log₂(1024) = {d:.10}\n", .{math.log2(1024.0)});
    
    // Log base 10
    try stdout.print("\nLogarithm base 10 (log₁₀):\n", .{});
    try stdout.print("  log₁₀(1)    = {d:.10}\n", .{math.log10(1.0)});
    try stdout.print("  log₁₀(10)   = {d:.10}\n", .{math.log10(10.0)});
    try stdout.print("  log₁₀(100)  = {d:.10}\n", .{math.log10(100.0)});
    try stdout.print("  log₁₀(1000) = {d:.10}\n", .{math.log10(1000.0)});
    
    // General log (any base)
    try stdout.print("\nCustom base logarithm:\n", .{});
    try stdout.print("  log₃(9)   = {d:.6}\n", .{math.log(f64, 3.0, 9.0)});
    try stdout.print("  log₅(125) = {d:.6}\n", .{math.log(f64, 5.0, 125.0)});
}
```

### Exponential Functions

```zig
const std = @import("std");
const math = std.math;

pub fn exponentialFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // exp(x) = e^x
    try stdout.print("Exponential (e^x):\n", .{});
    try stdout.print("  e^0  = {d:.10}\n", .{@exp(0.0)});
    try stdout.print("  e^1  = {d:.10}\n", .{@exp(1.0)});
    try stdout.print("  e^2  = {d:.10}\n", .{@exp(2.0)});
    try stdout.print("  e^-1 = {d:.10}\n", .{@exp(-1.0)});
    try stdout.print("  e^ln(5) = {d:.10}\n", .{@exp(@log(5.0))});
    
    // exp2(x) = 2^x
    try stdout.print("\n2^x:\n", .{});
    try stdout.print("  2^0  = {d}\n", .{@exp2(0.0)});
    try stdout.print("  2^1  = {d}\n", .{@exp2(1.0)});
    try stdout.print("  2^8  = {d}\n", .{@exp2(8.0)});
    try stdout.print("  2^10 = {d}\n", .{@exp2(10.0)});
    
    // เปรียบเทียบ log และ exp
    try stdout.print("\nexp(ln(x)) = x:\n", .{});
    const x: f64 = 42.0;
    try stdout.print("  x = {d}\n", .{x});
    try stdout.print("  ln(x) = {d:.10}\n", .{@log(x)});
    try stdout.print("  exp(ln(x)) = {d:.10}\n", .{@exp(@log(x))});
}
```

---

## Step 325: Trigonometry (sin, cos, tan, asin, acos, atan)

### Basic Trigonometric Functions

```zig
const std = @import("std");
const math = std.math;

pub fn trigonometricFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ค่ามุมสำคัญ (radians)
    const angles = [_]struct { name: []const u8, rad: f64 }{
        .{ .name = "0°", .rad = 0.0 },
        .{ .name = "30°", .rad = math.pi / 6.0 },
        .{ .name = "45°", .rad = math.pi / 4.0 },
        .{ .name = "60°", .rad = math.pi / 3.0 },
        .{ .name = "90°", .rad = math.pi / 2.0 },
        .{ .name = "120°", .rad = 2.0 * math.pi / 3.0 },
        .{ .name = "180°", .rad = math.pi },
        .{ .name = "270°", .rad = 3.0 * math.pi / 2.0 },
        .{ .name = "360°", .rad = math.tau },
    };
    
    try stdout.print("{s:>6} | {s:>10} | {s:>10} | {s:>10}\n", .{
        "Angle",
        "sin",
        "cos",
        "tan",
    });
    try stdout.print("-" .* 44 ++ "\n", .{});
    
    for (angles) |a| {
        const s = @sin(a.rad);
        const c = @cos(a.rad);
        
        // tan = sin/cos (undefined เมื่อ cos = 0)
        if (@abs(c) < 1e-10) {
            try stdout.print("{s:>6} | {d:>10.6} | {d:>10.6} | {s:>10}\n", .{
                a.name, s, c, "undefined",
            });
        } else {
            const t = @tan(a.rad);
            try stdout.print("{s:>6} | {d:>10.6} | {d:>10.6} | {d:>10.6}\n", .{
                a.name, s, c, t,
            });
        }
    }
}
```

### Inverse Trigonometric Functions

```zig
const std = @import("std");
const math = std.math;

pub fn inverseTrigFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // asin: arcsin (result in radians, domain: [-1, 1])
    try stdout.print("arcsin:\n", .{});
    const asin_vals = [_]f64{ -1.0, -0.5, 0.0, 0.5, 1.0 };
    for (asin_vals) |v| {
        const result = math.asin(v);
        try stdout.print("  asin({d:5.2}) = {d:.6} rad = {d:.2}°\n", .{
            v,
            result,
            result * 180.0 / math.pi,
        });
    }
    
    // acos: arccos (result in radians, domain: [-1, 1])
    try stdout.print("\narccos:\n", .{});
    for (asin_vals) |v| {
        const result = math.acos(v);
        try stdout.print("  acos({d:5.2}) = {d:.6} rad = {d:.2}°\n", .{
            v,
            result,
            result * 180.0 / math.pi,
        });
    }
    
    // atan: arctan (result in radians)
    try stdout.print("\narctan:\n", .{});
    const atan_vals = [_]f64{ -10.0, -1.0, 0.0, 1.0, 10.0 };
    for (atan_vals) |v| {
        const result = math.atan(v);
        try stdout.print("  atan({d:5.1}) = {d:.6} rad = {d:.2}°\n", .{
            v,
            result,
            result * 180.0 / math.pi,
        });
    }
    
    // atan2: arctan2(y, x) - สำคัญมากสำหรับการคำนวณมุม
    try stdout.print("\narctan2(y, x):\n", .{});
    const points = [_][2]f64{
        .{ 1.0, 1.0 },   // 45°
        .{ 1.0, -1.0 },  // -45° หรือ 315°
        .{ -1.0, 1.0 },  // 135°
        .{ -1.0, -1.0 }, // -135° หรือ 225°
        .{ 0.0, 1.0 },   // 0°
        .{ 1.0, 0.0 },   // 90°
    };
    
    for (points) |p| {
        const angle = math.atan2(p[0], p[1]);
        try stdout.print("  atan2({d:5.1}, {d:5.1}) = {d:.2}°\n", .{
            p[0],
            p[1],
            angle * 180.0 / math.pi,
        });
    }
}
```

### Trigonometry Applications

```zig
const std = @import("std");
const math = std.math;

// คำนวณตำแหน่งบน circle
pub const Point2D = struct {
    x: f64,
    y: f64,
    
    pub fn distanceTo(self: Point2D, other: Point2D) f64 {
        const dx = other.x - self.x;
        const dy = other.y - self.y;
        return math.hypot(dx, dy);
    }
    
    pub fn angleTo(self: Point2D, other: Point2D) f64 {
        return math.atan2(other.y - self.y, other.x - self.x);
    }
};

pub fn pointOnCircle(center: Point2D, radius: f64, angle_rad: f64) Point2D {
    return Point2D{
        .x = center.x + radius * @cos(angle_rad),
        .y = center.y + radius * @sin(angle_rad),
    };
}

pub fn demonstrateTrigApplications() !void {
    const stdout = std.io.getStdOut().writer();
    
    const center = Point2D{ .x = 0.0, .y = 0.0 };
    const radius: f64 = 5.0;
    
    try stdout.print("จุดบนวงกลมรัศมี {d}:\n", .{radius});
    
    var i: u32 = 0;
    while (i < 8) : (i += 1) {
        const angle = @as(f64, @floatFromInt(i)) * math.tau / 8.0;
        const point = pointOnCircle(center, radius, angle);
        try stdout.print("  {d:.0}°: ({d:.4}, {d:.4})\n", .{
            angle * 180.0 / math.pi,
            point.x,
            point.y,
        });
    }
    
    // คำนวณระยะห่างระหว่างสองจุด
    const p1 = Point2D{ .x = 3.0, .y = 4.0 };
    const p2 = Point2D{ .x = 0.0, .y = 0.0 };
    const dist = p1.distanceTo(p2);
    const angle = p1.angleTo(p2);
    
    try stdout.print("\nระหว่าง ({d}, {d}) และ ({d}, {d}):\n", .{
        p1.x, p1.y, p2.x, p2.y,
    });
    try stdout.print("  ระยะห่าง = {d:.4}\n", .{dist});
    try stdout.print("  มุม = {d:.2}°\n", .{angle * 180.0 / math.pi});
}
```

---

## Step 326: Hyperbolic Functions

### Hyperbolic Trigonometry

```zig
const std = @import("std");
const math = std.math;

pub fn hyperbolicFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    const values = [_]f64{ -2.0, -1.0, -0.5, 0.0, 0.5, 1.0, 2.0 };
    
    // sinh(x) = (e^x - e^-x) / 2
    try stdout.print("Hyperbolic sine (sinh):\n", .{});
    for (values) |v| {
        try stdout.print("  sinh({d:5.1}) = {d:.6}\n", .{ v, math.sinh(v) });
    }
    
    // cosh(x) = (e^x + e^-x) / 2
    try stdout.print("\nHyperbolic cosine (cosh):\n", .{});
    for (values) |v| {
        try stdout.print("  cosh({d:5.1}) = {d:.6}\n", .{ v, math.cosh(v) });
    }
    
    // tanh(x) = sinh(x) / cosh(x)
    try stdout.print("\nHyperbolic tangent (tanh):\n", .{});
    for (values) |v| {
        try stdout.print("  tanh({d:5.1}) = {d:.6}\n", .{ v, math.tanh(v) });
    }
    
    // Identity: cosh²(x) - sinh²(x) = 1
    try stdout.print("\nตรวจสอบ identity: cosh²(x) - sinh²(x) = 1\n", .{});
    for (values) |v| {
        const c = math.cosh(v);
        const s = math.sinh(v);
        const identity = c * c - s * s;
        try stdout.print("  x={d:4.1}: cosh²-sinh² = {d:.10}\n", .{ v, identity });
    }
}
```

### Inverse Hyperbolic Functions

```zig
const std = @import("std");
const math = std.math;

pub fn inverseHyperbolicFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    // asinh: inverse hyperbolic sine (domain: all reals)
    try stdout.print("Inverse hyperbolic sine (asinh):\n", .{});
    const asinh_vals = [_]f64{ -2.0, -1.0, 0.0, 1.0, 2.0 };
    for (asinh_vals) |v| {
        const result = math.asinh(v);
        try stdout.print("  asinh({d:5.1}) = {d:.6}\n", .{ v, result });
        // ตรวจสอบ: sinh(asinh(v)) = v
        try stdout.print("    verify: sinh(result) = {d:.6}\n", .{math.sinh(result)});
    }
    
    // acosh: inverse hyperbolic cosine (domain: x >= 1)
    try stdout.print("\nInverse hyperbolic cosine (acosh):\n", .{});
    const acosh_vals = [_]f64{ 1.0, 1.5, 2.0, 5.0, 10.0 };
    for (acosh_vals) |v| {
        const result = math.acosh(v);
        try stdout.print("  acosh({d:5.1}) = {d:.6}\n", .{ v, result });
    }
    
    // atanh: inverse hyperbolic tangent (domain: -1 < x < 1)
    try stdout.print("\nInverse hyperbolic tangent (atanh):\n", .{});
    const atanh_vals = [_]f64{ -0.9, -0.5, 0.0, 0.5, 0.9 };
    for (atanh_vals) |v| {
        const result = math.atanh(v);
        try stdout.print("  atanh({d:5.2}) = {d:.6}\n", .{ v, result });
    }
}
```

---

## Step 327: Float Special Values (inf, nan, isNan, isInf)

### การจัดการค่า Special Float

```zig
const std = @import("std");
const math = std.math;

pub fn floatSpecialValues() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Infinity
    const pos_inf = math.inf(f64);
    const neg_inf = -math.inf(f64);
    
    try stdout.print("Infinity:\n", .{});
    try stdout.print("  +∞ = {d}\n", .{pos_inf});
    try stdout.print("  -∞ = {d}\n", .{neg_inf});
    try stdout.print("  isInf(+∞) = {}\n", .{math.isInf(pos_inf)});
    try stdout.print("  isPositiveInf(+∞) = {}\n", .{math.isPositiveInf(pos_inf)});
    try stdout.print("  isNegativeInf(-∞) = {}\n", .{math.isNegativeInf(neg_inf)});
    
    // NaN (Not a Number)
    const nan_val = math.nan(f64);
    
    try stdout.print("\nNaN:\n", .{});
    try stdout.print("  NaN = {d}\n", .{nan_val});
    try stdout.print("  isNan(NaN) = {}\n", .{math.isNan(nan_val)});
    try stdout.print("  NaN == NaN = {}\n", .{nan_val == nan_val}); // false!
    
    // Operations ที่ให้ผล special values
    try stdout.print("\nOperations ที่ให้ special values:\n", .{});
    try stdout.print("  1.0 / 0.0 = {d}\n", .{@as(f64, 1.0) / 0.0});
    try stdout.print("  -1.0 / 0.0 = {d}\n", .{@as(f64, -1.0) / 0.0});
    try stdout.print("  0.0 / 0.0 = {d}\n", .{@as(f64, 0.0) / 0.0});
    try stdout.print("  sqrt(-1) = {d}\n", .{math.sqrt(-1.0)});
    try stdout.print("  log(-1) = {d}\n", .{@log(-1.0)});
    try stdout.print("  inf + inf = {d}\n", .{pos_inf + pos_inf});
    try stdout.print("  inf - inf = {d}\n", .{pos_inf - pos_inf});
    try stdout.print("  inf * 0 = {d}\n", .{pos_inf * 0.0});
}
```

### การตรวจสอบและ Sanitize Values

```zig
const std = @import("std");
const math = std.math;

pub fn sanitizeFloat(x: f64, default: f64) f64 {
    if (math.isNan(x) or math.isInf(x)) {
        return default;
    }
    return x;
}

pub fn safeDiv(a: f64, b: f64) f64 {
    if (@abs(b) < math.floatEps(f64)) {
        if (a > 0) return math.inf(f64);
        if (a < 0) return -math.inf(f64);
        return math.nan(f64);
    }
    return a / b;
}

pub fn demonstrateSafeFloatOps() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Float epsilon (smallest representable difference)
    try stdout.print("Float epsilon (f32): {e}\n", .{math.floatEps(f32)});
    try stdout.print("Float epsilon (f64): {e}\n", .{math.floatEps(f64)});
    
    // Float min/max
    try stdout.print("Float max (f64): {e}\n", .{math.floatMax(f64)});
    try stdout.print("Float min positive (f64): {e}\n", .{math.floatMin(f64)});
    
    // Safe operations
    try stdout.print("\nSafe division:\n", .{});
    try stdout.print("  safeDiv(10, 2) = {d}\n", .{safeDiv(10.0, 2.0)});
    try stdout.print("  safeDiv(10, 0) = {d}\n", .{safeDiv(10.0, 0.0)});
    try stdout.print("  safeDiv(-10, 0) = {d}\n", .{safeDiv(-10.0, 0.0)});
    try stdout.print("  safeDiv(0, 0) = {d}\n", .{safeDiv(0.0, 0.0)});
    
    // Sanitize NaN/Inf
    const values = [_]f64{
        1.0,
        math.nan(f64),
        math.inf(f64),
        -math.inf(f64),
        0.0 / 0.0,
        1.0 / 0.0,
    };
    
    try stdout.print("\nSanitize values (default=0):\n", .{});
    for (values) |v| {
        try stdout.print("  sanitize({d:8}) = {d}\n", .{
            v,
            sanitizeFloat(v, 0.0),
        });
    }
}
```

---

## Step 328: Integer Math (divTrunc, divFloor, rem)

### Integer Division Modes

```zig
const std = @import("std");
const math = std.math;

pub fn integerDivision() !void {
    const stdout = std.io.getStdOut().writer();
    
    // Zig มีหลายโหมดสำหรับ integer division
    
    // divTrunc: truncate towards zero (C-style)
    // ผลลัพธ์มี sign ตาม dividend
    try stdout.print("divTrunc (truncate towards zero):\n", .{});
    try stdout.print("  divTrunc(7, 2)   = {d}\n", .{math.divTrunc(i32, 7, 2) catch unreachable});
    try stdout.print("  divTrunc(-7, 2)  = {d}\n", .{math.divTrunc(i32, -7, 2) catch unreachable});
    try stdout.print("  divTrunc(7, -2)  = {d}\n", .{math.divTrunc(i32, 7, -2) catch unreachable});
    try stdout.print("  divTrunc(-7, -2) = {d}\n", .{math.divTrunc(i32, -7, -2) catch unreachable});
    
    // divFloor: floor division (Python-style)
    // ผลลัพธ์ปัดลงเสมอ
    try stdout.print("\ndivFloor (floor division):\n", .{});
    try stdout.print("  divFloor(7, 2)   = {d}\n", .{math.divFloor(i32, 7, 2) catch unreachable});
    try stdout.print("  divFloor(-7, 2)  = {d}\n", .{math.divFloor(i32, -7, 2) catch unreachable});
    try stdout.print("  divFloor(7, -2)  = {d}\n", .{math.divFloor(i32, 7, -2) catch unreachable});
    try stdout.print("  divFloor(-7, -2) = {d}\n", .{math.divFloor(i32, -7, -2) catch unreachable});
    
    // divExact: exact division (error ถ้าหารไม่ลงตัว)
    try stdout.print("\ndivExact (exact division):\n", .{});
    if (math.divExact(i32, 8, 2)) |result| {
        try stdout.print("  divExact(8, 2) = {d}\n", .{result});
    } else |err| {
        try stdout.print("  divExact(8, 2) error: {}\n", .{err});
    }
    
    if (math.divExact(i32, 7, 2)) |result| {
        try stdout.print("  divExact(7, 2) = {d}\n", .{result});
    } else |err| {
        try stdout.print("  divExact(7, 2) error: {}\n", .{err});
    }
}
```

### Remainder Operations

```zig
const std = @import("std");
const math = std.math;

pub fn remainderOperations() !void {
    const stdout = std.io.getStdOut().writer();
    
    // rem: remainder (sign follows dividend, like C %)
    try stdout.print("rem (C-style remainder):\n", .{});
    try stdout.print("  rem(7, 3)   = {d}\n", .{math.rem(i32, 7, 3) catch unreachable});
    try stdout.print("  rem(-7, 3)  = {d}\n", .{math.rem(i32, -7, 3) catch unreachable});
    try stdout.print("  rem(7, -3)  = {d}\n", .{math.rem(i32, 7, -3) catch unreachable});
    try stdout.print("  rem(-7, -3) = {d}\n", .{math.rem(i32, -7, -3) catch unreachable});
    
    // mod: modulo (sign follows divisor, always non-negative for positive divisor)
    try stdout.print("\nmod (Python-style modulo):\n", .{});
    try stdout.print("  mod(7, 3)   = {d}\n", .{math.mod(i32, 7, 3) catch unreachable});
    try stdout.print("  mod(-7, 3)  = {d}\n", .{math.mod(i32, -7, 3) catch unreachable});
    try stdout.print("  mod(7, -3)  = {d}\n", .{math.mod(i32, 7, -3) catch unreachable});
    try stdout.print("  mod(-7, -3) = {d}\n", .{math.mod(i32, -7, -3) catch unreachable});
    
    // ตัวอย่างการใช้งาน: circular indexing
    try stdout.print("\nCircular indexing (size=5):\n", .{});
    const size: i32 = 5;
    const indices = [_]i32{ 0, 1, 4, 5, 6, 10, -1, -5, -6 };
    for (indices) |idx| {
        const circular = math.mod(i32, idx, size) catch unreachable;
        try stdout.print("  index[{d:3}] (mod {d}) = {d}\n", .{ idx, size, circular });
    }
}
```

### Overflow-safe Integer Operations

```zig
const std = @import("std");
const math = std.math;

pub fn overflowSafeOps() !void {
    const stdout = std.io.getStdOut().writer();
    
    // add/sub/mul ที่มี overflow detection
    try stdout.print("Overflow-safe operations:\n", .{});
    
    // addWithOverflow
    const max_u8: u8 = 255;
    const result_add = @addWithOverflow(max_u8, 1);
    try stdout.print("  255 + 1 = {d}, overflow = {}\n", .{
        result_add[0],
        result_add[1] == 1,
    });
    
    // subWithOverflow
    const min_i8: i8 = -128;
    const result_sub = @subWithOverflow(min_i8, 1);
    try stdout.print("  -128 - 1 = {d}, overflow = {}\n", .{
        result_sub[0],
        result_sub[1] == 1,
    });
    
    // mulWithOverflow
    const big: i16 = 1000;
    const result_mul = @mulWithOverflow(big, big);
    try stdout.print("  1000 * 1000 = {d}, overflow = {}\n", .{
        result_mul[0],
        result_mul[1] == 1,
    });
    
    // Saturating arithmetic (clamp ที่ boundary แทนที่จะ overflow)
    try stdout.print("\nSaturating arithmetic:\n", .{});
    try stdout.print("  u8 sat_add(250, 10) = {d}\n", .{
        @as(u8, 250) +| @as(u8, 10),
    });
    try stdout.print("  i8 sat_add(120, 20) = {d}\n", .{
        @as(i8, 120) +| @as(i8, 20),
    });
    try stdout.print("  u8 sat_sub(5, 10) = {d}\n", .{
        @as(u8, 5) -| @as(u8, 10),
    });
    
    // Wrapping arithmetic (wrap around ที่ boundary)
    try stdout.print("\nWrapping arithmetic:\n", .{});
    try stdout.print("  u8 wrap_add(250, 10) = {d}\n", .{
        @as(u8, 250) +% @as(u8, 10),
    });
    try stdout.print("  u8 wrap_sub(5, 10) = {d}\n", .{
        @as(u8, 5) -% @as(u8, 10),
    });
}
```

---

## Step 329: std.math.big - Big Integers

### BigInt พื้นฐาน

```zig
const std = @import("std");
const big = std.math.big;

pub fn bigIntBasics(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง BigInt จาก integer
    var a = try big.int.Managed.initSet(allocator, 123456789012345678);
    defer a.deinit();
    
    var b = try big.int.Managed.initSet(allocator, 987654321098765432);
    defer b.deinit();
    
    // Addition
    var sum = try big.int.Managed.init(allocator);
    defer sum.deinit();
    try sum.add(&a.toConst(), &b.toConst());
    try stdout.print("a + b = {}\n", .{sum});
    
    // Multiplication
    var product = try big.int.Managed.init(allocator);
    defer product.deinit();
    try product.mul(&a.toConst(), &b.toConst());
    try stdout.print("a * b = {}\n", .{product});
}
```

### Factorial ด้วย BigInt

```zig
const std = @import("std");
const big = std.math.big;

pub fn factorial(n: u32, allocator: std.mem.Allocator) !big.int.Managed {
    var result = try big.int.Managed.initSet(allocator, 1);
    errdefer result.deinit();
    
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        var temp = try big.int.Managed.initSet(allocator, i);
        defer temp.deinit();
        
        try result.mul(&result.toConst(), &temp.toConst());
    }
    
    return result;
}

pub fn demonstrateFactorial(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const ns = [_]u32{ 10, 20, 50, 100 };
    
    for (ns) |n| {
        var fact = try factorial(n, allocator);
        defer fact.deinit();
        
        try stdout.print("{d}! = {}\n", .{ n, fact });
    }
}
```

### Fibonacci ด้วย BigInt

```zig
const std = @import("std");
const big = std.math.big;

pub fn fibonacci(n: u32, allocator: std.mem.Allocator) !big.int.Managed {
    if (n <= 1) {
        return try big.int.Managed.initSet(allocator, n);
    }
    
    var a = try big.int.Managed.initSet(allocator, 0);
    defer a.deinit();
    var b = try big.int.Managed.initSet(allocator, 1);
    defer b.deinit();
    
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        var temp = try big.int.Managed.init(allocator);
        try temp.add(&a.toConst(), &b.toConst());
        
        // swap: a = b, b = temp
        var old_a = a;
        a = b;
        b = temp;
        old_a.deinit();
    }
    
    // return b (ซึ่งตอนนี้คือ fibonacci(n))
    var result = try big.int.Managed.init(allocator);
    try result.copy(b.toConst());
    return result;
}

pub fn demonstrateFibonacci(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Fibonacci numbers:\n", .{});
    
    const indices = [_]u32{ 10, 50, 100, 200 };
    
    for (indices) |n| {
        var fib = try fibonacci(n, allocator);
        defer fib.deinit();
        try stdout.print("  F({d:3}) = {}\n", .{ n, fib });
    }
}
```

---

## Step 330: Numeric Algorithms

### Numerical Integration (Simpson's Rule)

```zig
const std = @import("std");
const math = std.math;

// Simpson's Rule สำหรับ numerical integration
pub fn integrate(
    f: *const fn (x: f64) f64,
    a: f64,
    b: f64,
    n: usize, // ต้องเป็นเลขคู่
) f64 {
    const h = (b - a) / @as(f64, @floatFromInt(n));
    var sum: f64 = f(a) + f(b);
    
    var i: usize = 1;
    while (i < n) : (i += 1) {
        const x = a + @as(f64, @floatFromInt(i)) * h;
        if (i % 2 == 0) {
            sum += 2.0 * f(x);
        } else {
            sum += 4.0 * f(x);
        }
    }
    
    return (h / 3.0) * sum;
}

fn sinFunc(x: f64) f64 { return @sin(x); }
fn xSquared(x: f64) f64 { return x * x; }
fn expFunc(x: f64) f64 { return @exp(x); }

pub fn demonstrateIntegration() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Numerical Integration (Simpson's Rule):\n", .{});
    
    // ∫₀^π sin(x) dx = 2
    const sin_result = integrate(sinFunc, 0.0, math.pi, 1000);
    try stdout.print("  ∫₀^π sin(x) dx ≈ {d:.8} (ควรเป็น 2.0)\n", .{sin_result});
    
    // ∫₀^1 x² dx = 1/3
    const x2_result = integrate(xSquared, 0.0, 1.0, 1000);
    try stdout.print("  ∫₀^1 x² dx ≈ {d:.8} (ควรเป็น 0.33333...)\n", .{x2_result});
    
    // ∫₀^1 e^x dx = e - 1
    const exp_result = integrate(expFunc, 0.0, 1.0, 1000);
    try stdout.print("  ∫₀^1 e^x dx ≈ {d:.8} (ควรเป็น {d:.8})\n", .{
        exp_result,
        math.e - 1.0,
    });
}
```

### Root Finding (Newton-Raphson)

```zig
const std = @import("std");
const math = std.math;

// Newton-Raphson method สำหรับหา root ของ f(x) = 0
pub fn newtonRaphson(
    f: *const fn (x: f64) f64,
    df: *const fn (x: f64) f64,
    initial: f64,
    tolerance: f64,
    max_iterations: usize,
) ?f64 {
    var x = initial;
    
    var i: usize = 0;
    while (i < max_iterations) : (i += 1) {
        const fx = f(x);
        if (@abs(fx) < tolerance) return x;
        
        const dfx = df(x);
        if (@abs(dfx) < math.floatEps(f64)) return null; // derivative ≈ 0
        
        x = x - fx / dfx;
    }
    
    return null; // ไม่ converge
}

// f(x) = x² - 2 (root คือ √2)
fn fSqrt2(x: f64) f64 { return x * x - 2.0; }
fn dfSqrt2(x: f64) f64 { return 2.0 * x; }

// f(x) = x³ - x - 2 (root ≈ 1.5214...)
fn fCubic(x: f64) f64 { return x * x * x - x - 2.0; }
fn dfCubic(x: f64) f64 { return 3.0 * x * x - 1.0; }

pub fn demonstrateRootFinding() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Root Finding (Newton-Raphson):\n", .{});
    
    // หา √2
    if (newtonRaphson(fSqrt2, dfSqrt2, 1.0, 1e-10, 100)) |root| {
        try stdout.print("  √2 ≈ {d:.10}\n", .{root});
        try stdout.print("  actual: {d:.10}\n", .{math.sqrt(2.0)});
    }
    
    // หา root ของ cubic
    if (newtonRaphson(fCubic, dfCubic, 2.0, 1e-10, 100)) |root| {
        try stdout.print("  x³ - x - 2 = 0: x ≈ {d:.10}\n", .{root});
        try stdout.print("  verify: f(root) = {d:.2e}\n", .{fCubic(root)});
    }
}
```

### Statistics Functions

```zig
const std = @import("std");
const math = std.math;

pub fn statistics(data: []const f64) struct {
    mean: f64,
    variance: f64,
    std_dev: f64,
    min: f64,
    max: f64,
    median: f64,
} {
    if (data.len == 0) {
        return .{
            .mean = 0,
            .variance = 0,
            .std_dev = 0,
            .min = 0,
            .max = 0,
            .median = 0,
        };
    }
    
    // Mean
    var sum: f64 = 0;
    var min_val = data[0];
    var max_val = data[0];
    
    for (data) |x| {
        sum += x;
        min_val = @min(min_val, x);
        max_val = @max(max_val, x);
    }
    
    const mean = sum / @as(f64, @floatFromInt(data.len));
    
    // Variance
    var sq_diff_sum: f64 = 0;
    for (data) |x| {
        const diff = x - mean;
        sq_diff_sum += diff * diff;
    }
    const variance = sq_diff_sum / @as(f64, @floatFromInt(data.len));
    
    // Median (simplified - assumes sorted data)
    const n = data.len;
    const median = if (n % 2 == 0)
        (data[n / 2 - 1] + data[n / 2]) / 2.0
    else
        data[n / 2];
    
    return .{
        .mean = mean,
        .variance = variance,
        .std_dev = math.sqrt(variance),
        .min = min_val,
        .max = max_val,
        .median = median,
    };
}

pub fn demonstrateStatistics() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ข้อมูลทดสอบ (ต้อง sorted สำหรับ median ที่ถูกต้อง)
    const data = [_]f64{ 2, 4, 4, 4, 5, 5, 7, 9 };
    
    const stats = statistics(&data);
    
    try stdout.print("Statistics for {any}:\n", .{data});
    try stdout.print("  Mean     = {d:.4}\n", .{stats.mean});
    try stdout.print("  Variance = {d:.4}\n", .{stats.variance});
    try stdout.print("  Std Dev  = {d:.4}\n", .{stats.std_dev});
    try stdout.print("  Min      = {d:.4}\n", .{stats.min});
    try stdout.print("  Max      = {d:.4}\n", .{stats.max});
    try stdout.print("  Median   = {d:.4}\n", .{stats.median});
}
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Calculator

```zig
const std = @import("std");
const math = std.math;

pub const Calculator = struct {
    pub fn evaluate(expr: []const u8) !f64 {
        // Simple expression parser
        var buf: [256]u8 = undefined;
        _ = buf;
        _ = expr;
        return 0.0; // placeholder
    }
    
    pub fn sin_degrees(deg: f64) f64 {
        return @sin(deg * math.pi / 180.0);
    }
    
    pub fn cos_degrees(deg: f64) f64 {
        return @cos(deg * math.pi / 180.0);
    }
    
    pub fn compound_interest(principal: f64, rate: f64, n: f64, t: f64) f64 {
        // A = P(1 + r/n)^(nt)
        return principal * math.pow(f64, 1.0 + rate / n, n * t);
    }
};

pub fn demonstrateCalculator() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("sin(30°) = {d:.6}\n", .{Calculator.sin_degrees(30.0)});
    try stdout.print("cos(60°) = {d:.6}\n", .{Calculator.cos_degrees(60.0)});
    
    const investment = Calculator.compound_interest(
        10000.0, // principal
        0.05,    // 5% annual rate
        12.0,    // compounded monthly
        10.0,    // 10 years
    );
    try stdout.print("\nเงินลงทุน 10,000 บาท, ดอกเบี้ย 5%/ปี, ทบต้นรายเดือน, 10 ปี:\n", .{});
    try stdout.print("  = {d:.2} บาท\n", .{investment});
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Mathematical Constants** - pi, e, tau, sqrt2 และ constants อื่นๆ
2. **Basic Math** - abs, floor, ceil, round, trunc, min, max, clamp
3. **Power & Roots** - pow, powi, sqrt, cbrt, hypot
4. **Logarithms** - ln, log2, log10, log(base, x), exp, exp2
5. **Trigonometry** - sin, cos, tan, asin, acos, atan, atan2
6. **Hyperbolic** - sinh, cosh, tanh, asinh, acosh, atanh
7. **Special Float Values** - inf, nan, isNan, isInf, floatEps
8. **Integer Math** - divTrunc, divFloor, rem, mod, overflow operations
9. **BigInt** - การคำนวณตัวเลขขนาดใหญ่
10. **Numeric Algorithms** - integration, root finding, statistics

**Best Practices:**
- ระวัง NaN และ Infinity เสมอ
- ใช้ `math.floatEps` สำหรับการเปรียบเทียบ floats
- ใช้ BigInt เมื่อตัวเลขเกิน u64/i64
- ตรวจสอบ domain ของฟังก์ชัน (เช่น log ต้องการ x > 0)

---

*ต่อไปในส่วนที่ 34: std.sort - การเรียงลำดับข้อมูล*
