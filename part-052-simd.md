# Part 52: SIMD และ Vector Operations ใน Zig
## Steps 511-520

SIMD (Single Instruction, Multiple Data) เป็นเทคนิคที่ช่วยให้ CPU ประมวลผลข้อมูลหลายชิ้นพร้อมกันในคำสั่งเดียว ทำให้โปรแกรมเร็วขึ้นได้มากถึง 4-16 เท่า Zig มีการสนับสนุน SIMD ผ่าน `@Vector` type ที่ใช้งานง่ายและ portable

---

## Step 511: SIMD คืออะไร? CPU Vector Registers

### แนวคิด SIMD

SIMD ย่อมาจาก **Single Instruction, Multiple Data** หมายความว่า CPU instruction หนึ่งคำสั่งสามารถทำงานกับข้อมูลหลายชิ้นพร้อมกัน

```
Scalar (ปกติ):
  a[0] + b[0] = c[0]  (1 operation)
  a[1] + b[1] = c[1]  (1 operation)
  a[2] + b[2] = c[2]  (1 operation)
  a[3] + b[3] = c[3]  (1 operation)
  รวม: 4 operations

SIMD:
  [a[0],a[1],a[2],a[3]] + [b[0],b[1],b[2],b[3]] = [c[0],c[1],c[2],c[3]]
  รวม: 1 operation!
```

### Vector Registers บน CPU

```
CPU Architecture    | Register Width | Float32 per reg | Float64 per reg
--------------------|----------------|-----------------|----------------
SSE (x86)           | 128-bit        | 4               | 2
AVX (x86)           | 256-bit        | 8               | 4
AVX-512 (x86)       | 512-bit        | 16              | 8
NEON (ARM)          | 128-bit        | 4               | 2
SVE (ARM)           | Variable       | 4-16            | 2-8
```

```zig
const std = @import("std");

pub fn main() void {
    // ดูขนาด vector registers ที่ CPU รองรับ
    std.debug.print("CPU Features:\n", .{});
    
    // ตรวจสอบ feature ผ่าน builtin
    const target = @import("builtin").cpu;
    _ = target;
    
    // Vector width สูงสุดบน CPU นี้
    std.debug.print("Vector width: {}\n", .{std.simd.suggestVectorLength(f32) orelse 1});
}
```

### SIMD Use Cases

SIMD เหมาะสำหรับ:
- Image processing (pixel operations)
- Audio processing (sample mixing)
- Physics simulation (particle updates)
- Machine learning (matrix operations)
- Cryptography (hash functions, encryption)
- String searching และ parsing
- Mathematical computations

---

## Step 512: @Vector Type ใน Zig

### การประกาศ Vector

```zig
const std = @import("std");

pub fn main() void {
    // ประกาศ vector ด้วย @Vector(len, type)
    const v4f32 = @Vector(4, f32);
    const v8i16 = @Vector(8, i16);
    const v16u8 = @Vector(16, u8);
    
    // สร้าง vector
    const a: v4f32 = .{ 1.0, 2.0, 3.0, 4.0 };
    const b: v4f32 = .{ 5.0, 6.0, 7.0, 8.0 };
    
    // Vector addition (SIMD instruction)
    const c = a + b;
    
    std.debug.print("c = {d}\n", .{c});
    // Output: c = { 6.0, 8.0, 10.0, 12.0 }
    
    // ขนาด vector
    std.debug.print("size of v4f32: {} bytes\n", .{@sizeOf(v4f32)});
    std.debug.print("size of v8i16: {} bytes\n", .{@sizeOf(v8i16)});
    std.debug.print("size of v16u8: {} bytes\n", .{@sizeOf(v16u8)});
    
    // Length ของ vector
    std.debug.print("v4f32 length: {}\n", .{@typeInfo(v4f32).vector.len});
}
```

### Vector Initialization Patterns

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);
const Vec8i = @Vector(8, i32);

pub fn vectorInitPatterns() void {
    // 1. Literal initialization
    const v1: Vec4 = .{ 1.0, 2.0, 3.0, 4.0 };
    
    // 2. Splat (broadcast single value)
    const v2: Vec4 = @splat(3.14);
    // v2 = { 3.14, 3.14, 3.14, 3.14 }
    
    // 3. From array
    const arr = [4]f32{ 10.0, 20.0, 30.0, 40.0 };
    const v3: Vec4 = arr;
    
    // 4. สร้างจาก comptime
    const v4: Vec4 = comptime blk: {
        var result: Vec4 = @splat(0);
        for (0..4) |i| {
            result[i] = @floatFromInt(i * i);
        }
        break :blk result;
    };
    
    std.debug.print("v1 = {d}\n", .{v1});
    std.debug.print("v2 = {d}\n", .{v2});
    std.debug.print("v3 = {d}\n", .{v3});
    std.debug.print("v4 = {d}\n", .{v4});
    
    // 5. แปลง vector เป็น array
    const back_to_array: [4]f32 = v1;
    std.debug.print("back: {d}\n", .{back_to_array});
}
```

### Vector Element Access

```zig
const Vec4 = @Vector(4, f32);

pub fn vectorAccess() void {
    var v: Vec4 = .{ 1.0, 2.0, 3.0, 4.0 };
    
    // อ่าน element (constant index)
    const x = v[0];
    const y = v[1];
    
    // แก้ไข element
    v[2] = 99.0;
    
    std.debug.print("x={d}, y={d}, v={d}\n", .{ x, y, v });
    
    // Dynamic index (ต้องระวัง: อาจไม่ vectorize)
    const idx: usize = 1;
    const elem = v[idx];
    std.debug.print("elem[{}]={d}\n", .{ idx, elem });
}
```

---

## Step 513: Vector Arithmetic Operations

### Basic Arithmetic

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);
const Vec4i = @Vector(4, i32);

pub fn vectorArithmetic() void {
    const a: Vec4 = .{ 1.0, 2.0, 3.0, 4.0 };
    const b: Vec4 = .{ 5.0, 6.0, 7.0, 8.0 };
    
    // Addition
    const add = a + b;
    std.debug.print("add: {d}\n", .{add});
    
    // Subtraction
    const sub = b - a;
    std.debug.print("sub: {d}\n", .{sub});
    
    // Multiplication (element-wise, NOT dot product)
    const mul = a * b;
    std.debug.print("mul: {d}\n", .{mul});
    
    // Division
    const div = b / a;
    std.debug.print("div: {d}\n", .{div});
    
    // Negation
    const neg = -a;
    std.debug.print("neg: {d}\n", .{neg});
    
    // Integer vectors
    const ia: Vec4i = .{ 10, 20, 30, 40 };
    const ib: Vec4i = .{ 3, 4, 5, 6 };
    
    // Integer division (truncated)
    const idiv = @divTrunc(ia, ib);
    std.debug.print("idiv: {}\n", .{idiv});
    
    // Modulo
    const imod = @rem(ia, ib);
    std.debug.print("imod: {}\n", .{imod});
}
```

### Math Functions บน Vectors

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);

pub fn vectorMathFunctions() void {
    const v: Vec4 = .{ 1.0, 4.0, 9.0, 16.0 };
    
    // Square root
    const sqrt_v = @sqrt(v);
    std.debug.print("sqrt: {d}\n", .{sqrt_v});
    // Output: sqrt: { 1.0, 2.0, 3.0, 4.0 }
    
    // Absolute value
    const neg_v: Vec4 = .{ -1.0, -2.0, 3.0, -4.0 };
    const abs_v = @abs(neg_v);
    std.debug.print("abs: {d}\n", .{abs_v});
    
    // Floor, Ceil, Round
    const frac_v: Vec4 = .{ 1.7, 2.3, -1.7, -2.3 };
    const floor_v = @floor(frac_v);
    const ceil_v = @ceil(frac_v);
    const round_v = @round(frac_v);
    
    std.debug.print("floor: {d}\n", .{floor_v});
    std.debug.print("ceil: {d}\n", .{ceil_v});
    std.debug.print("round: {d}\n", .{round_v});
    
    // Min/Max
    const a: Vec4 = .{ 1.0, 5.0, 3.0, 7.0 };
    const b: Vec4 = .{ 4.0, 2.0, 6.0, 8.0 };
    
    const min_v = @min(a, b);
    const max_v = @max(a, b);
    
    std.debug.print("min: {d}\n", .{min_v});
    std.debug.print("max: {d}\n", .{max_v});
}
```

### Horizontal Operations (Reduction)

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);
const Vec4i = @Vector(4, i32);

pub fn vectorReductions() void {
    const v: Vec4 = .{ 1.0, 2.0, 3.0, 4.0 };
    
    // Horizontal sum
    const sum = @reduce(.Add, v);
    std.debug.print("sum: {d}\n", .{sum}); // 10.0
    
    // Horizontal product
    const product = @reduce(.Mul, v);
    std.debug.print("product: {d}\n", .{product}); // 24.0
    
    // Min/Max across all elements
    const min_elem = @reduce(.Min, v);
    const max_elem = @reduce(.Max, v);
    std.debug.print("min: {d}, max: {d}\n", .{ min_elem, max_elem });
    
    // Boolean reductions
    const bools: @Vector(4, bool) = .{ true, false, true, true };
    
    const any_true = @reduce(.Or, bools);
    const all_true = @reduce(.And, bools);
    
    std.debug.print("any: {}, all: {}\n", .{ any_true, all_true });
    
    // XOR reduction
    const iv: Vec4i = .{ 0b1010, 0b1100, 0b0110, 0b1001 };
    const xor_all = @reduce(.Xor, iv);
    std.debug.print("xor: 0b{b:04}\n", .{xor_all});
}
```

---

## Step 514: Vector Comparison

### Comparison Operations

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);
const Vec4i = @Vector(4, i32);
const Vec4b = @Vector(4, bool);

pub fn vectorComparisons() void {
    const a: Vec4 = .{ 1.0, 5.0, 3.0, 7.0 };
    const b: Vec4 = .{ 4.0, 2.0, 3.0, 8.0 };
    
    // Less than
    const lt: Vec4b = a < b;
    std.debug.print("a < b: {}\n", .{lt});
    // { true, false, false, true }
    
    // Less than or equal
    const le: Vec4b = a <= b;
    std.debug.print("a <= b: {}\n", .{le});
    
    // Equal
    const eq: Vec4b = a == b;
    std.debug.print("a == b: {}\n", .{eq});
    
    // Not equal
    const ne: Vec4b = a != b;
    std.debug.print("a != b: {}\n", .{ne});
    
    // Greater than
    const gt: Vec4b = a > b;
    std.debug.print("a > b: {}\n", .{gt});
    
    // Greater than or equal
    const ge: Vec4b = a >= b;
    std.debug.print("a >= b: {}\n", .{ge});
}
```

### Vector Select (Blend/Mask)

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);

pub fn vectorSelect() void {
    const a: Vec4 = .{ 1.0, 2.0, 3.0, 4.0 };
    const b: Vec4 = .{ 5.0, 6.0, 7.0, 8.0 };
    
    // Mask: เลือก a ถ้า true, เลือก b ถ้า false
    const mask = @Vector(4, bool){ true, false, true, false };
    
    // @select เปรียบเสมือน ternary operator สำหรับ vectors
    const result = @select(f32, mask, a, b);
    std.debug.print("select: {d}\n", .{result});
    // { 1.0, 6.0, 3.0, 8.0 }
    
    // ตัวอย่าง: clamp ด้วย select
    const values: Vec4 = .{ -1.0, 0.5, 1.5, 2.0 };
    const zeros: Vec4 = @splat(0.0);
    const ones: Vec4 = @splat(1.0);
    
    const clamped = @select(f32, values > ones, ones,
                    @select(f32, values < zeros, zeros, values));
    std.debug.print("clamped: {d}\n", .{clamped});
    // { 0.0, 0.5, 1.0, 1.0 }
}
```

### ตัวอย่าง: SIMD Branch-Free Min/Max Clamp

```zig
const Vec8 = @Vector(8, f32);

pub fn clampVec8(values: Vec8, min_val: f32, max_val: f32) Vec8 {
    const min_vec: Vec8 = @splat(min_val);
    const max_vec: Vec8 = @splat(max_val);
    return @min(@max(values, min_vec), max_vec);
}

pub fn normalizePixels(pixels: []u8, output: []f32) void {
    const Vec16u8 = @Vector(16, u8);
    const Vec16f = @Vector(16, f32);
    const inv255: Vec16f = @splat(1.0 / 255.0);
    
    var i: usize = 0;
    while (i + 16 <= pixels.len) : (i += 16) {
        const src: Vec16u8 = pixels[i..][0..16].*;
        const as_ints: @Vector(16, u32) = src;
        const as_floats: Vec16f = @floatFromInt(as_ints);
        const normalized = as_floats * inv255;
        output[i..][0..16].* = normalized;
    }
    
    // Handle remainder
    while (i < pixels.len) : (i += 1) {
        output[i] = @as(f32, @floatFromInt(pixels[i])) / 255.0;
    }
}
```

---

## Step 515: Vector Shuffle และ Permute

### @shuffle

```zig
const std = @import("std");

pub fn vectorShuffle() void {
    const Vec4i = @Vector(4, i32);
    
    const a: Vec4i = .{ 1, 2, 3, 4 };
    const b: Vec4i = .{ 5, 6, 7, 8 };
    
    // @shuffle(type, a, b, mask)
    // mask: index 0-3 = elements from a
    //       index ~0 = ~3 = elements from b (bitwise not)
    
    // Reverse a
    const reversed = @shuffle(i32, a, undefined, [4]i32{ 3, 2, 1, 0 });
    std.debug.print("reversed: {}\n", .{reversed}); // { 4, 3, 2, 1 }
    
    // Interleave a และ b
    const interleaved = @shuffle(i32, a, b, [8]i32{ 0, ~@as(i32, 0), 1, ~@as(i32, 1), 2, ~@as(i32, 2), 3, ~@as(i32, 3) });
    std.debug.print("interleaved: {}\n", .{interleaved}); // { 1, 5, 2, 6, 3, 7, 4, 8 }
    
    // Extract even/odd elements
    const even = @shuffle(i32, a, undefined, [2]i32{ 0, 2 });
    const odd = @shuffle(i32, a, undefined, [2]i32{ 1, 3 });
    std.debug.print("even: {}, odd: {}\n", .{ even, odd });
    
    // Broadcast element 0 to all positions
    const broadcast = @shuffle(i32, a, undefined, [4]i32{ 0, 0, 0, 0 });
    std.debug.print("broadcast[0]: {}\n", .{broadcast}); // { 1, 1, 1, 1 }
}
```

### Practical Shuffle Patterns

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);

// Transpose 4x4 matrix ด้วย shuffle
pub fn transposeMatrix4x4(mat: [4]Vec4) [4]Vec4 {
    // Interleave rows 0,1 and 2,3
    const r01_lo = @shuffle(f32, mat[0], mat[1], [4]i32{ 0, ~@as(i32, 0), 1, ~@as(i32, 1) });
    const r01_hi = @shuffle(f32, mat[0], mat[1], [4]i32{ 2, ~@as(i32, 2), 3, ~@as(i32, 3) });
    const r23_lo = @shuffle(f32, mat[2], mat[3], [4]i32{ 0, ~@as(i32, 0), 1, ~@as(i32, 1) });
    const r23_hi = @shuffle(f32, mat[2], mat[3], [4]i32{ 2, ~@as(i32, 2), 3, ~@as(i32, 3) });
    
    // Interleave pairs
    return .{
        @shuffle(f32, r01_lo, r23_lo, [4]i32{ 0, 1, ~@as(i32, 0), ~@as(i32, 1) }),
        @shuffle(f32, r01_lo, r23_lo, [4]i32{ 2, 3, ~@as(i32, 2), ~@as(i32, 3) }),
        @shuffle(f32, r01_hi, r23_hi, [4]i32{ 0, 1, ~@as(i32, 0), ~@as(i32, 1) }),
        @shuffle(f32, r01_hi, r23_hi, [4]i32{ 2, 3, ~@as(i32, 2), ~@as(i32, 3) }),
    };
}

// AOS -> SOA conversion สำหรับ Vec3
pub fn aosToSoa(aos: []const [3]f32, x: []f32, y: []f32, z: []f32) void {
    for (aos, 0..) |v, i| {
        x[i] = v[0];
        y[i] = v[1];
        z[i] = v[2];
    }
}
```

---

## Step 516: Vector Load/Store Patterns

### Aligned และ Unaligned Load/Store

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);

pub fn vectorLoadStore(allocator: std.mem.Allocator) !void {
    // Aligned allocation สำหรับ SIMD
    const data = try allocator.alignedAlloc(f32, 16, 16);
    defer allocator.free(data);
    
    // Initialize data
    for (data, 0..) |*x, i| {
        x.* = @floatFromInt(i);
    }
    
    // Load vector from aligned memory
    const v: Vec4 = data[0..4].*;
    std.debug.print("loaded: {d}\n", .{v});
    
    // Store vector to memory
    const result = v * @as(Vec4, @splat(2.0));
    data[4..8].* = result;
    
    std.debug.print("stored: {d}\n", .{data[4..8]});
}

// Streaming (non-temporal) stores - bypass cache
pub fn streamStore(dest: []f32, value: f32) void {
    const Vec4 = @Vector(4, f32);
    const val_vec: Vec4 = @splat(value);
    
    var i: usize = 0;
    while (i + 4 <= dest.len) : (i += 4) {
        // Normal store (goes through cache)
        dest[i..][0..4].* = val_vec;
    }
    
    // Handle remaining elements
    while (i < dest.len) : (i += 1) {
        dest[i] = value;
    }
}
```

### Masked Load/Store (Partial Vectors)

```zig
const std = @import("std");

pub fn maskedVectorOps(data: []f32) void {
    const Vec4 = @Vector(4, f32);
    
    var i: usize = 0;
    
    // Full vector operations
    while (i + 4 <= data.len) : (i += 4) {
        const v: Vec4 = data[i..][0..4].*;
        const result = v * v;
        data[i..][0..4].* = result;
    }
    
    // Scalar tail (partial last vector)
    while (i < data.len) : (i += 1) {
        data[i] = data[i] * data[i];
    }
}

// Generic SIMD loop ที่ handle tail automatically
pub fn simdLoop(
    comptime VecLen: comptime_int,
    data: []f32,
    comptime func: fn(@Vector(VecLen, f32)) @Vector(VecLen, f32),
) void {
    var i: usize = 0;
    
    while (i + VecLen <= data.len) : (i += VecLen) {
        const v: @Vector(VecLen, f32) = data[i..][0..VecLen].*;
        data[i..][0..VecLen].* = func(v);
    }
    
    // Process remaining elements
    while (i < data.len) : (i += 1) {
        const v: @Vector(1, f32) = .{data[i]};
        data[i] = func(v)[0];
    }
}
```

---

## Step 517: SIMD สำหรับ String Operations

### SIMD String Search

```zig
const std = @import("std");

// Fast byte search ด้วย SIMD
pub fn findByteSimd(haystack: []const u8, needle: u8) ?usize {
    const Vec16 = @Vector(16, u8);
    const needle_vec: Vec16 = @splat(needle);
    
    var i: usize = 0;
    
    while (i + 16 <= haystack.len) : (i += 16) {
        const chunk: Vec16 = haystack[i..][0..16].*;
        const matches = chunk == needle_vec;
        
        // Check if any match
        if (@reduce(.Or, matches)) {
            // Find exact position
            for (0..16) |j| {
                if (matches[j]) return i + j;
            }
        }
    }
    
    // Handle remainder
    while (i < haystack.len) : (i += 1) {
        if (haystack[i] == needle) return i;
    }
    
    return null;
}

// Count occurrences ด้วย SIMD
pub fn countByteSimd(data: []const u8, target: u8) u32 {
    const Vec16 = @Vector(16, u8);
    const Vec16u16 = @Vector(16, u16);
    const target_vec: Vec16 = @splat(target);
    var count: u32 = 0;
    
    var i: usize = 0;
    while (i + 16 <= data.len) : (i += 16) {
        const chunk: Vec16 = data[i..][0..16].*;
        const matches: Vec16 = @intFromBool(chunk == target_vec);
        const matches_u16: Vec16u16 = matches;
        count += @reduce(.Add, matches_u16);
    }
    
    while (i < data.len) : (i += 1) {
        if (data[i] == target) count += 1;
    }
    
    return count;
}
```

### SIMD String to Uppercase/Lowercase

```zig
// แปลง lowercase เป็น uppercase ด้วย SIMD
pub fn toUpperSimd(text: []u8) void {
    const Vec32 = @Vector(32, u8);
    const lower_a: Vec32 = @splat('a');
    const lower_z: Vec32 = @splat('z');
    const diff: Vec32 = @splat('A' - 'a');
    
    var i: usize = 0;
    while (i + 32 <= text.len) : (i += 32) {
        var chunk: Vec32 = text[i..][0..32].*;
        
        // Find lowercase letters
        const is_lower = (chunk >= lower_a) & (chunk <= lower_z);
        
        // Convert: subtract 32 for lowercase letters
        const converted = chunk - @as(Vec32, @intFromBool(is_lower)) * diff;
        chunk = converted;
        
        text[i..][0..32].* = chunk;
    }
    
    // Handle tail
    while (i < text.len) : (i += 1) {
        if (text[i] >= 'a' and text[i] <= 'z') {
            text[i] -= 32;
        }
    }
}

// SIMD whitespace trimming
pub fn trimWhitespaceStart(text: []const u8) []const u8 {
    const Vec16 = @Vector(16, u8);
    const space: Vec16 = @splat(' ');
    const tab: Vec16 = @splat('\t');
    const newline: Vec16 = @splat('\n');
    
    var i: usize = 0;
    while (i + 16 <= text.len) {
        const chunk: Vec16 = text[i..][0..16].*;
        const is_space = (chunk == space) | (chunk == tab) | (chunk == newline);
        
        if (!@reduce(.And, is_space)) {
            // Found non-whitespace in this chunk
            for (0..16) |j| {
                if (!is_space[j]) return text[i + j ..];
            }
        }
        i += 16;
    }
    
    while (i < text.len) : (i += 1) {
        if (text[i] != ' ' and text[i] != '\t' and text[i] != '\n') {
            return text[i..];
        }
    }
    
    return text[text.len..];
}
```

---

## Step 518: SIMD สำหรับ Math (Dot Product, Matrix Multiply)

### Dot Product

```zig
const std = @import("std");
const Vec4 = @Vector(4, f32);
const Vec8 = @Vector(8, f32);

// Dot product สำหรับ Vec4
pub fn dotProduct4(a: Vec4, b: Vec4) f32 {
    return @reduce(.Add, a * b);
}

// Dot product สำหรับ array ใหญ่
pub fn dotProductLarge(a: []const f32, b: []const f32) f32 {
    std.debug.assert(a.len == b.len);
    
    var sum: Vec8 = @splat(0.0);
    var i: usize = 0;
    
    while (i + 8 <= a.len) : (i += 8) {
        const va: Vec8 = a[i..][0..8].*;
        const vb: Vec8 = b[i..][0..8].*;
        sum += va * vb;
    }
    
    var result = @reduce(.Add, sum);
    
    while (i < a.len) : (i += 1) {
        result += a[i] * b[i];
    }
    
    return result;
}

// Matrix 4x4 multiply ด้วย SIMD
pub fn matMul4x4(a: [4]Vec4, b_cols: [4]Vec4) [4]Vec4 {
    var result: [4]Vec4 = undefined;
    
    for (0..4) |row| {
        var r: Vec4 = @splat(0.0);
        for (0..4) |col| {
            // Broadcast a[row][col] to all lanes
            const scalar: Vec4 = @splat(a[row][col]);
            r += scalar * b_cols[col];
        }
        result[row] = r;
    }
    
    return result;
}
```

### L2 Norm และ Distance

```zig
const Vec4 = @Vector(4, f32);

// Euclidean norm
pub fn norm4(v: Vec4) f32 {
    return @sqrt(@reduce(.Add, v * v));
}

// Normalize vector
pub fn normalize4(v: Vec4) Vec4 {
    const len = norm4(v);
    const inv_len: Vec4 = @splat(1.0 / len);
    return v * inv_len;
}

// Distance between two vectors
pub fn distance4(a: Vec4, b: Vec4) f32 {
    const diff = a - b;
    return norm4(diff);
}

// Lerp (linear interpolation)
pub fn lerp4(a: Vec4, b: Vec4, t: f32) Vec4 {
    const tv: Vec4 = @splat(t);
    const one_minus_t: Vec4 = @splat(1.0 - t);
    return a * one_minus_t + b * tv;
}

// Fast approximate inverse square root (SIMD version)
pub fn fastInvSqrt(v: Vec4) Vec4 {
    const magic: @Vector(4, u32) = @splat(0x5f3759df);
    const half: Vec4 = @splat(0.5);
    const three_halves: Vec4 = @splat(1.5);
    
    const bits: @Vector(4, u32) = @bitCast(v);
    const approx_bits = magic - (bits >> @splat(1));
    var approx: Vec4 = @bitCast(approx_bits);
    
    // Newton-Raphson iteration
    approx = approx * (three_halves - half * v * approx * approx);
    
    return approx;
}
```

---

## Step 519: Platform-Specific SIMD (SSE, AVX, NEON)

### ตรวจสอบ CPU Features

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn checkCpuFeatures() void {
    const cpu = builtin.cpu;
    const arch = cpu.arch;
    
    switch (arch) {
        .x86_64, .x86 => {
            const features = cpu.features;
            const has_sse2 = std.Target.x86.featureSetHas(features, .sse2);
            const has_avx = std.Target.x86.featureSetHas(features, .avx);
            const has_avx2 = std.Target.x86.featureSetHas(features, .avx2);
            const has_avx512f = std.Target.x86.featureSetHas(features, .avx512f);
            
            std.debug.print("SSE2: {}\n", .{has_sse2});
            std.debug.print("AVX: {}\n", .{has_avx});
            std.debug.print("AVX2: {}\n", .{has_avx2});
            std.debug.print("AVX-512: {}\n", .{has_avx512f});
        },
        .aarch64 => {
            const features = cpu.features;
            const has_neon = std.Target.aarch64.featureSetHas(features, .neon);
            std.debug.print("NEON: {}\n", .{has_neon});
        },
        else => {
            std.debug.print("Unknown architecture\n", .{});
        },
    }
}
```

### Adaptive SIMD ตาม Platform

```zig
const builtin = @import("builtin");
const std = @import("std");

// เลือก vector width ตาม CPU
pub const SIMD_WIDTH = blk: {
    const arch = builtin.cpu.arch;
    const features = builtin.cpu.features;
    
    break :blk switch (arch) {
        .x86_64 => if (std.Target.x86.featureSetHas(features, .avx512f)) 16
                   else if (std.Target.x86.featureSetHas(features, .avx2)) 8
                   else if (std.Target.x86.featureSetHas(features, .sse2)) 4
                   else 1,
        .aarch64 => if (std.Target.aarch64.featureSetHas(features, .neon)) 4 else 1,
        else => 1,
    };
};

const SimdVec = @Vector(SIMD_WIDTH, f32);

pub fn adaptiveProcess(data: []f32) void {
    var i: usize = 0;
    
    while (i + SIMD_WIDTH <= data.len) : (i += SIMD_WIDTH) {
        const v: SimdVec = data[i..][0..SIMD_WIDTH].*;
        data[i..][0..SIMD_WIDTH].* = v * v;
    }
    
    while (i < data.len) : (i += 1) {
        data[i] *= data[i];
    }
}
```

### Platform-Specific Intrinsics ผ่าน Inline Assembly

```zig
const std = @import("std");

// ใช้ SSE intrinsics โดยตรง (x86_64 เท่านั้น)
pub fn addVec4SSE(a: [4]f32, b: [4]f32) [4]f32 {
    var result: [4]f32 = undefined;
    
    asm volatile (
        \\movups (%[a]), %%xmm0
        \\movups (%[b]), %%xmm1
        \\addps %%xmm1, %%xmm0
        \\movups %%xmm0, (%[c])
        :
        : [a] "r" (&a),
          [b] "r" (&b),
          [c] "r" (&result),
        : "xmm0", "xmm1", "memory"
    );
    
    return result;
}
```

---

## Step 520: Auto-Vectorization Hints

### เขียน Code ที่ Compiler สามารถ Vectorize ได้

```zig
const std = @import("std");

// ดี: Loop ที่ vectorizable
pub fn multiplyArrayGood(a: []f32, b: []const f32, c: []const f32) void {
    std.debug.assert(a.len == b.len);
    std.debug.assert(a.len == c.len);
    
    for (a, b, c) |*dst, src_b, src_c| {
        dst.* = src_b * src_c;
    }
}

// ไม่ดี: มี dependency ที่ prevent vectorization
pub fn dependentLoop(data: []f32) void {
    for (data[1..], data[0..data.len-1]) |*dst, src| {
        dst.* = src * 2.0; // reads previous element - dependency!
    }
}

// Pattern สำหรับ vectorizable reduction
pub fn sumArray(data: []const f32) f32 {
    var sum: f32 = 0;
    for (data) |x| {
        sum += x; // Compiler สามารถ vectorize loop นี้ได้
    }
    return sum;
}
```

### ใช้ std.simd.suggestVectorLength

```zig
const std = @import("std");

pub fn optimalVectorOp(data: []f32) void {
    // ขอ vector length ที่เหมาะสมที่สุดสำหรับ f32 บน CPU นี้
    const vec_len = comptime std.simd.suggestVectorLength(f32) orelse 4;
    const Vec = @Vector(vec_len, f32);
    
    var i: usize = 0;
    while (i + vec_len <= data.len) : (i += vec_len) {
        const v: Vec = data[i..][0..vec_len].*;
        data[i..][0..vec_len].* = v * v + v;
    }
    
    while (i < data.len) : (i += 1) {
        data[i] = data[i] * data[i] + data[i];
    }
}
```

### Complete SIMD Example: Image Grayscale Conversion

```zig
const std = @import("std");

// แปลง RGB เป็น Grayscale ด้วย SIMD
// Grayscale = 0.299*R + 0.587*G + 0.114*B
pub fn rgbToGrayscaleSimd(
    r: []const u8,
    g: []const u8,
    b: []const u8,
    gray: []u8,
) void {
    std.debug.assert(r.len == g.len);
    std.debug.assert(r.len == b.len);
    std.debug.assert(r.len == gray.len);
    
    const Vec16u8 = @Vector(16, u8);
    const Vec16f = @Vector(16, f32);
    
    const wr: Vec16f = @splat(0.299);
    const wg: Vec16f = @splat(0.587);
    const wb: Vec16f = @splat(0.114);
    
    var i: usize = 0;
    while (i + 16 <= r.len) : (i += 16) {
        // Load
        const vr: Vec16u8 = r[i..][0..16].*;
        const vg: Vec16u8 = g[i..][0..16].*;
        const vb: Vec16u8 = b[i..][0..16].*;
        
        // Convert to float
        const fr: Vec16f = @floatFromInt(@as(@Vector(16, u32), vr));
        const fg: Vec16f = @floatFromInt(@as(@Vector(16, u32), vg));
        const fb: Vec16f = @floatFromInt(@as(@Vector(16, u32), vb));
        
        // Weighted sum
        const result_f = fr * wr + fg * wg + fb * wb;
        
        // Convert back to u8
        const result_u32: @Vector(16, u32) = @intFromFloat(result_f);
        const result_u8: Vec16u8 = @truncate(result_u32);
        
        gray[i..][0..16].* = result_u8;
    }
    
    // Tail
    while (i < r.len) : (i += 1) {
        const rf: f32 = @floatFromInt(r[i]);
        const gf: f32 = @floatFromInt(g[i]);
        const bf: f32 = @floatFromInt(b[i]);
        gray[i] = @intFromFloat(rf * 0.299 + gf * 0.587 + bf * 0.114);
    }
}

pub fn main() !void {
    const allocator = std.heap.page_allocator;
    
    const size = 1024 * 1024; // 1 megapixel
    const r = try allocator.alloc(u8, size);
    defer allocator.free(r);
    const g = try allocator.alloc(u8, size);
    defer allocator.free(g);
    const b = try allocator.alloc(u8, size);
    defer allocator.free(b);
    const gray = try allocator.alloc(u8, size);
    defer allocator.free(gray);
    
    // Fill with test data
    for (r, g, b, 0..) |*rv, *gv, *bv, i| {
        rv.* = @intCast(i % 256);
        gv.* = @intCast((i * 2) % 256);
        bv.* = @intCast((i * 3) % 256);
    }
    
    var timer = try std.time.Timer.start();
    
    rgbToGrayscaleSimd(r, g, b, gray);
    
    const elapsed = timer.read();
    std.debug.print("Converted {} pixels in {}ns\n", .{ size, elapsed });
    std.debug.print("Throughput: {d:.2} Mpixels/s\n", .{
        @as(f64, @floatFromInt(size)) / @as(f64, @floatFromInt(elapsed)) * 1000.0
    });
}
```

---

## สรุป Part 52

ในบทนี้เราได้เรียนรู้:

1. **SIMD Basics** - แนวคิดและ CPU vector registers
2. **@Vector Type** - การประกาศและใช้งาน vectors ใน Zig
3. **Arithmetic** - การคำนวณบน vectors รวมถึง reductions
4. **Comparisons** - การเปรียบเทียบและ select/blend
5. **Shuffle/Permute** - การจัดเรียง elements ใหม่
6. **Load/Store** - การอ่าน/เขียน memory อย่างมีประสิทธิภาพ
7. **String Operations** - การใช้ SIMD สำหรับ text processing
8. **Math Operations** - dot product, matrix multiply, normalization
9. **Platform SIMD** - SSE, AVX, NEON specifics
10. **Auto-vectorization** - เขียน code ที่ compiler vectorize ได้

> **Tips**: SIMD ทำให้เร็วขึ้นมาก แต่ต้องระวัง memory alignment, vector length ที่เหมาะสม, และ handle tail elements เสมอ
