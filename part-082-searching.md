# Part 82: Algorithms: Searching ใน Zig
## Steps 811-820

---

## Step 811: Linear Search

### Linear Search คืออะไร?

Linear Search (Sequential Search) เป็นอัลกอริทึมการค้นหาที่ง่ายที่สุด ตรวจสอบแต่ละ element ตามลำดับจนพบค่าที่ต้องการ

**Complexity:**
- Best case: O(1) - พบที่ตำแหน่งแรก
- Average case: O(n)
- Worst case: O(n) - ไม่พบหรือพบที่ตำแหน่งสุดท้าย
- Space: O(1)

```zig
// linear_search.zig
const std = @import("std");

/// Linear Search แบบพื้นฐาน - คืนค่า index หรือ null
pub fn linearSearch(comptime T: type, arr: []const T, target: T) ?usize {
    for (arr, 0..) |item, i| {
        if (item == target) return i;
    }
    return null;
}

/// Linear Search แบบ generic ที่ใช้ comparator function
pub fn linearSearchBy(
    comptime T: type,
    arr: []const T,
    target: T,
    comptime lessThan: fn (T, T) bool,
) ?usize {
    for (arr, 0..) |item, i| {
        if (!lessThan(item, target) and !lessThan(target, item)) {
            return i;
        }
    }
    return null;
}

/// Linear Search หาทุก occurrence
pub fn linearSearchAll(
    comptime T: type,
    arr: []const T,
    target: T,
    allocator: std.mem.Allocator,
) ![]usize {
    var results = std.ArrayList(usize).init(allocator);
    errdefer results.deinit();
    
    for (arr, 0..) |item, i| {
        if (item == target) {
            try results.append(i);
        }
    }
    
    return results.toOwnedSlice();
}

/// Sentinel Linear Search - ลดการตรวจสอบ bounds
pub fn sentinelSearch(comptime T: type, arr: []T, target: T) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    // บันทึกค่าสุดท้าย
    const last = arr[n - 1];
    arr[n - 1] = target; // วาง sentinel
    defer arr[n - 1] = last; // คืนค่า
    
    var i: usize = 0;
    while (arr[i] != target) : (i += 1) {}
    
    if (i < n - 1 or last == target) {
        return i;
    }
    return null;
}

/// Linear Search พร้อม frequency count (สำหรับ cache-friendly access)
pub fn linearSearchWithCount(
    comptime T: type,
    arr: []T,
    target: T,
    counts: []usize,
) ?usize {
    for (arr, 0..) |item, i| {
        counts[i] += 1; // นับการเข้าถึง
        if (item == target) {
            // Move-to-front: เพื่อ cache items ที่ค้นบ่อย
            if (i > 0) {
                const temp = arr[i];
                arr[i] = arr[i - 1];
                arr[i - 1] = temp;
                const tempCount = counts[i];
                counts[i] = counts[i - 1];
                counts[i - 1] = tempCount;
                return i - 1;
            }
            return i;
        }
    }
    return null;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    const arr = [_]i32{ 4, 2, 7, 1, 9, 3, 5, 8, 6 };
    
    try stdout.print("=== Linear Search ===\n", .{});
    try stdout.print("Array: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    // ค้นหาค่าที่มี
    if (linearSearch(i32, &arr, 7)) |idx| {
        try stdout.print("ค้นหา 7: พบที่ index {d}\n", .{idx});
    }
    
    // ค้นหาค่าที่ไม่มี
    if (linearSearch(i32, &arr, 10)) |idx| {
        try stdout.print("ค้นหา 10: พบที่ index {d}\n", .{idx});
    } else {
        try stdout.print("ค้นหา 10: ไม่พบ\n", .{});
    }
    
    // ค้นหาทุก occurrence
    const arr2 = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6, 1, 8 };
    const indices = try linearSearchAll(i32, &arr2, 1, allocator);
    defer allocator.free(indices);
    
    try stdout.print("\nค้นหา 1 ทุก occurrence: ", .{});
    for (indices) |idx| try stdout.print("{d} ", .{idx});
    try stdout.print("\n", .{});
}
```

---

## Step 812: Binary Search

### Binary Search คืออะไร?

Binary Search ทำงานบนข้อมูลที่เรียงลำดับแล้ว โดยแบ่งครึ่ง search space ในแต่ละขั้นตอน

**Complexity:**
- Best case: O(1) - พบที่กลาง array
- Average/Worst case: O(log n)
- Space: O(1) สำหรับ iterative, O(log n) สำหรับ recursive

```zig
// binary_search.zig
const std = @import("std");

/// Binary Search แบบ iterative
pub fn binarySearch(comptime T: type, arr: []const T, target: T) ?usize {
    if (arr.len == 0) return null;
    
    var left: usize = 0;
    var right: usize = arr.len - 1;
    
    while (left <= right) {
        const mid = left + (right - left) / 2; // หลีกเลี่ยง overflow
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            if (mid == 0) break;
            right = mid - 1;
        }
    }
    
    return null;
}

/// Binary Search แบบ recursive
pub fn binarySearchRecursive(comptime T: type, arr: []const T, target: T, left: usize, right: usize) ?usize {
    if (left > right) return null;
    
    const mid = left + (right - left) / 2;
    
    if (arr[mid] == target) {
        return mid;
    } else if (arr[mid] < target) {
        return binarySearchRecursive(T, arr, target, mid + 1, right);
    } else {
        if (mid == 0) return null;
        return binarySearchRecursive(T, arr, target, left, mid - 1);
    }
}

/// หา first occurrence ของค่า
pub fn binarySearchFirst(comptime T: type, arr: []const T, target: T) ?usize {
    if (arr.len == 0) return null;
    
    var left: usize = 0;
    var right: usize = arr.len - 1;
    var result: ?usize = null;
    
    while (left <= right) {
        const mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            if (mid == 0) break;
            right = mid - 1; // ค้นหาต่อทางซ้าย
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            if (mid == 0) break;
            right = mid - 1;
        }
    }
    
    return result;
}

/// หา last occurrence ของค่า
pub fn binarySearchLast(comptime T: type, arr: []const T, target: T) ?usize {
    if (arr.len == 0) return null;
    
    var left: usize = 0;
    var right: usize = arr.len - 1;
    var result: ?usize = null;
    
    while (left <= right) {
        const mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            left = mid + 1; // ค้นหาต่อทางขวา
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            if (mid == 0) break;
            right = mid - 1;
        }
    }
    
    return result;
}

/// Lower Bound: หา index แรกที่ >= target
pub fn lowerBound(comptime T: type, arr: []const T, target: T) usize {
    var left: usize = 0;
    var right: usize = arr.len;
    
    while (left < right) {
        const mid = left + (right - left) / 2;
        if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    return left;
}

/// Upper Bound: หา index แรกที่ > target
pub fn upperBound(comptime T: type, arr: []const T, target: T) usize {
    var left: usize = 0;
    var right: usize = arr.len;
    
    while (left < right) {
        const mid = left + (right - left) / 2;
        if (arr[mid] <= target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    return left;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 };
    const withDups = [_]i32{ 1, 2, 2, 2, 3, 4, 4, 5, 5, 5 };
    
    try stdout.print("=== Binary Search ===\n", .{});
    try stdout.print("Array: ", .{});
    for (sorted) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    // ค้นหาปกติ
    if (binarySearch(i32, &sorted, 11)) |idx| {
        try stdout.print("Binary Search 11: index {d}\n", .{idx});
    }
    
    if (binarySearch(i32, &sorted, 6)) |_| {
        try stdout.print("Binary Search 6: พบ\n", .{});
    } else {
        try stdout.print("Binary Search 6: ไม่พบ\n", .{});
    }
    
    // First/Last occurrence
    try stdout.print("\nArray with duplicates: ", .{});
    for (withDups) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    if (binarySearchFirst(i32, &withDups, 2)) |idx| {
        try stdout.print("First 2: index {d}\n", .{idx});
    }
    if (binarySearchLast(i32, &withDups, 2)) |idx| {
        try stdout.print("Last 2: index {d}\n", .{idx});
    }
    
    // Lower/Upper bound
    const lb = lowerBound(i32, &withDups, 4);
    const ub = upperBound(i32, &withDups, 4);
    try stdout.print("Lower bound of 4: {d}, Upper bound: {d}\n", .{ lb, ub });
    try stdout.print("จำนวน 4 ใน array: {d}\n", .{ ub - lb });
}
```

---

## Step 813: Jump Search

### Jump Search คืออะไร?

Jump Search ทำงานบน sorted array โดยกระโดดไปเป็นก้อน (blocks) แล้วทำ linear search ในก้อนที่พบ

**Complexity:**
- Time: O(√n)
- Space: O(1)

```zig
// jump_search.zig
const std = @import("std");

/// Jump Search พื้นฐาน
pub fn jumpSearch(comptime T: type, arr: []const T, target: T) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    // ขนาดก้อนที่เหมาะสมคือ √n
    const step = @as(usize, @intFromFloat(@sqrt(@as(f64, @floatFromInt(n)))));
    
    var prev: usize = 0;
    var curr: usize = step;
    
    // กระโดดจนพบก้อนที่อาจมีค่า target
    while (curr < n and arr[curr] < target) {
        prev = curr;
        curr += step;
    }
    curr = @min(curr, n - 1);
    
    // Linear search ในก้อนนั้น
    var i = prev;
    while (i <= curr) : (i += 1) {
        if (arr[i] == target) return i;
        if (arr[i] > target) break;
    }
    
    return null;
}

/// Jump Search แบบ backward (เดินถอยหลัง)
pub fn jumpSearchBackward(comptime T: type, arr: []const T, target: T) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    const step = @as(usize, @intFromFloat(@sqrt(@as(f64, @floatFromInt(n)))));
    
    // เริ่มจากท้าย array
    var curr: usize = n - 1;
    
    // กระโดดถอยหลัง
    while (curr > 0 and arr[curr] > target) {
        if (curr < step) {
            curr = 0;
        } else {
            curr -= step;
        }
    }
    
    // Linear search ไปข้างหน้า
    var i = curr;
    while (i < n) : (i += 1) {
        if (arr[i] == target) return i;
        if (arr[i] > target) break;
    }
    
    return null;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const arr = [_]i32{ 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377 };
    
    try stdout.print("=== Jump Search ===\n", .{});
    try stdout.print("Array (Fibonacci): ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    const targets = [_]i32{ 55, 100, 0, 377 };
    for (targets) |t| {
        if (jumpSearch(i32, &arr, t)) |idx| {
            try stdout.print("Jump Search {d}: พบที่ index {d}\n", .{ t, idx });
        } else {
            try stdout.print("Jump Search {d}: ไม่พบ\n", .{t});
        }
    }
}
```

---

## Step 814: Interpolation Search

### Interpolation Search คืออะไร?

Interpolation Search เหมาะกับข้อมูลที่กระจายสม่ำเสมอ ใช้การประมาณตำแหน่งของ target โดยการ interpolation

**Complexity:**
- Best/Average case: O(log log n) - เมื่อข้อมูลกระจายสม่ำเสมอ
- Worst case: O(n) - เมื่อข้อมูลไม่กระจายสม่ำเสมอ

```zig
// interpolation_search.zig
const std = @import("std");

/// Interpolation Search สำหรับ integers
pub fn interpolationSearch(arr: []const i32, target: i32) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    var low: usize = 0;
    var high: usize = n - 1;
    
    while (low <= high and target >= arr[low] and target <= arr[high]) {
        if (low == high) {
            if (arr[low] == target) return low;
            return null;
        }
        
        // คำนวณตำแหน่งโดย interpolation
        const range_val = arr[high] - arr[low];
        if (range_val == 0) {
            if (arr[low] == target) return low;
            return null;
        }
        
        const pos = low + @as(usize, @intCast(
            @divTrunc(
                (@as(i64, target - arr[low]) * @as(i64, high - low)),
                @as(i64, range_val)
            )
        ));
        
        if (arr[pos] == target) {
            return pos;
        } else if (arr[pos] < target) {
            low = pos + 1;
        } else {
            if (pos == 0) break;
            high = pos - 1;
        }
    }
    
    return null;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ข้อมูลที่กระจายสม่ำเสมอ - Interpolation Search ทำงานดีมาก
    const uniform = [_]i32{ 10, 20, 30, 40, 50, 60, 70, 80, 90, 100 };
    
    try stdout.print("=== Interpolation Search ===\n", .{});
    try stdout.print("Uniform array: ", .{});
    for (uniform) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    const targets = [_]i32{ 30, 75, 100, 55 };
    for (targets) |t| {
        if (interpolationSearch(&uniform, t)) |idx| {
            try stdout.print("ค้นหา {d}: พบที่ index {d}\n", .{ t, idx });
        } else {
            try stdout.print("ค้นหา {d}: ไม่พบ\n", .{t});
        }
    }
}
```

---

## Step 815: Exponential Search

### Exponential Search คืออะไร?

Exponential Search รวม Exponential expansion กับ Binary Search เหมาะสำหรับ unbounded arrays หรือเมื่อไม่รู้ขนาดข้อมูล

**Complexity:**
- Time: O(log n)
- Space: O(1)

```zig
// exponential_search.zig
const std = @import("std");

/// Binary Search helper
fn binarySearchRange(comptime T: type, arr: []const T, target: T, left: usize, right: usize) ?usize {
    var lo = left;
    var hi = @min(right, arr.len - 1);
    
    while (lo <= hi) {
        const mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) {
            lo = mid + 1;
        } else {
            if (mid == 0) break;
            hi = mid - 1;
        }
    }
    return null;
}

/// Exponential Search
pub fn exponentialSearch(comptime T: type, arr: []const T, target: T) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    // ตรวจสอบ element แรก
    if (arr[0] == target) return 0;
    
    // ขยาย range แบบ exponential จนพบ element ที่ >= target
    var bound: usize = 1;
    while (bound < n and arr[bound] < target) {
        bound *= 2;
    }
    
    // Binary search ใน range [bound/2, min(bound, n-1)]
    return binarySearchRange(T, arr, target, bound / 2, @min(bound, n - 1));
}

/// Fibonacci Search - ใช้ Fibonacci numbers แทน exponential
pub fn fibonacciSearch(comptime T: type, arr: []const T, target: T) ?usize {
    const n = arr.len;
    if (n == 0) return null;
    
    // หา Fibonacci number ที่ >= n
    var fibMm2: usize = 0; // (m-2)th Fibonacci
    var fibMm1: usize = 1; // (m-1)th Fibonacci
    var fibM: usize = 1;   // mth Fibonacci
    
    while (fibM < n) {
        fibMm2 = fibMm1;
        fibMm1 = fibM;
        fibM = fibMm1 + fibMm2;
    }
    
    var offset: usize = 0;
    
    while (fibM > 1) {
        const i = @min(offset + fibMm2, n - 1);
        
        if (arr[i] == target) {
            return i;
        } else if (arr[i] < target) {
            fibM = fibMm1;
            fibMm1 = fibMm2;
            fibMm2 = fibM - fibMm1;
            offset = i;
        } else {
            fibM = fibMm2;
            fibMm1 = fibMm1 - fibMm2;
            fibMm2 = fibM - fibMm1;
        }
    }
    
    if (fibMm1 == 1 and offset + 1 < n and arr[offset + 1] == target) {
        return offset + 1;
    }
    
    return null;
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    const arr = [_]i32{ 2, 3, 4, 10, 40, 50, 65, 70, 72, 90 };
    
    try stdout.print("=== Exponential & Fibonacci Search ===\n", .{});
    try stdout.print("Array: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    const targets = [_]i32{ 10, 72, 5, 90 };
    for (targets) |t| {
        const expResult = exponentialSearch(i32, &arr, t);
        const fibResult = fibonacciSearch(i32, &arr, t);
        
        try stdout.print("Target {d}: Exponential={?d}, Fibonacci={?d}\n", .{
            t, expResult, fibResult
        });
    }
}
```

---

## Step 816: String Searching (KMP Algorithm)

### KMP Algorithm คืออะไร?

Knuth-Morris-Pratt (KMP) Algorithm ค้นหา pattern ใน text โดยใช้ข้อมูลเกี่ยวกับ pattern เองเพื่อหลีกเลี่ยงการเปรียบเทียบซ้ำ

**Complexity:**
- Preprocessing: O(m) โดย m = ความยาว pattern
- Search: O(n) โดย n = ความยาว text
- Space: O(m)

```zig
// kmp_search.zig
const std = @import("std");

/// สร้าง failure function (partial match table) สำหรับ KMP
fn buildKMPTable(pattern: []const u8, table: []i64) void {
    const m = pattern.len;
    table[0] = -1;
    
    if (m == 1) return;
    
    table[1] = 0;
    var pos: usize = 2;
    var cnd: i64 = 0;
    
    while (pos < m) {
        if (pattern[pos - 1] == pattern[@intCast(cnd)]) {
            table[pos] = cnd + 1;
            pos += 1;
            cnd += 1;
        } else if (cnd > 0) {
            cnd = table[@intCast(cnd)];
        } else {
            table[pos] = 0;
            pos += 1;
        }
    }
}

/// KMP Search - คืน slice ของ positions ที่พบ pattern
pub fn kmpSearch(
    text: []const u8,
    pattern: []const u8,
    allocator: std.mem.Allocator,
) ![]usize {
    var positions = std.ArrayList(usize).init(allocator);
    errdefer positions.deinit();
    
    const n = text.len;
    const m = pattern.len;
    
    if (m == 0 or m > n) return positions.toOwnedSlice();
    
    // สร้าง failure table
    const table = try allocator.alloc(i64, m);
    defer allocator.free(table);
    buildKMPTable(pattern, table);
    
    var j: usize = 0; // position ใน text
    var k: usize = 0; // position ใน pattern
    
    while (j < n) {
        if (text[j] == pattern[k]) {
            j += 1;
            k += 1;
            if (k == m) {
                // พบ pattern
                try positions.append(j - k);
                if (table[k] >= 0) {
                    k = @intCast(table[k]);
                } else {
                    k = 0;
                    j += 1;
                }
            }
        } else {
            if (table[k] >= 0) {
                k = @intCast(table[k]);
            } else {
                k = 0;
                j += 1;
            }
        }
    }
    
    return positions.toOwnedSlice();
}

/// ตรวจสอบว่า pattern อยู่ใน text หรือไม่
pub fn kmpContains(text: []const u8, pattern: []const u8, allocator: std.mem.Allocator) !bool {
    const positions = try kmpSearch(text, pattern, allocator);
    defer allocator.free(positions);
    return positions.len > 0;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== KMP String Search ===\n\n", .{});
    
    const text1 = "AABAACAADAABAABA";
    const pattern1 = "AABA";
    
    try stdout.print("Text: {s}\n", .{text1});
    try stdout.print("Pattern: {s}\n\n", .{pattern1});
    
    const positions = try kmpSearch(text1, pattern1, allocator);
    defer allocator.free(positions);
    
    try stdout.print("พบ pattern ที่ positions: ", .{});
    for (positions) |pos| try stdout.print("{d} ", .{pos});
    try stdout.print("\n\n", .{});
    
    // ทดสอบเพิ่มเติม
    const text2 = "สวัสดีชาวโลก สวัสดี";
    const pattern2 = "สวัสดี";
    
    try stdout.print("Text: {s}\n", .{text2});
    try stdout.print("Pattern: {s}\n", .{pattern2});
    
    const contains = try kmpContains(text2, pattern2, allocator);
    try stdout.print("ค้นหา pattern: {}\n", .{contains});
}
```

---

## Step 817: String Searching (Boyer-Moore)

### Boyer-Moore Algorithm คืออะไร?

Boyer-Moore เป็นหนึ่งในอัลกอริทึมค้นหา string ที่มีประสิทธิภาพสูงที่สุดในทางปฏิบัติ เดิน text จากขวาไปซ้าย

**Complexity:**
- Preprocessing: O(m + σ) โดย σ คือขนาด alphabet
- Search: O(nm) worst case, O(n/m) best case

```zig
// boyer_moore.zig
const std = @import("std");

const ALPHABET_SIZE = 256;

/// Bad Character Heuristic
fn buildBadChar(pattern: []const u8, badChar: *[ALPHABET_SIZE]i64) void {
    for (badChar) |*v| v.* = -1;
    
    for (pattern, 0..) |c, i| {
        badChar[c] = @intCast(i);
    }
}

/// Boyer-Moore แบบ Bad Character เท่านั้น (Boyer-Moore-Horspool)
pub fn boyerMooreHorspool(
    text: []const u8,
    pattern: []const u8,
    allocator: std.mem.Allocator,
) ![]usize {
    var positions = std.ArrayList(usize).init(allocator);
    errdefer positions.deinit();
    
    const n = text.len;
    const m = pattern.len;
    
    if (m == 0 or m > n) return positions.toOwnedSlice();
    
    var badChar: [ALPHABET_SIZE]i64 = undefined;
    buildBadChar(pattern, &badChar);
    
    var s: usize = 0; // shift ของ pattern ใน text
    
    while (s <= n - m) {
        var j: i64 = @intCast(m - 1);
        
        // เปรียบเทียบจากขวาไปซ้าย
        while (j >= 0 and pattern[@intCast(j)] == text[s + @as(usize, @intCast(j))]) {
            j -= 1;
        }
        
        if (j < 0) {
            // พบ pattern
            try positions.append(s);
            s += 1;
        } else {
            // เลื่อน pattern ตาม bad character rule
            const shift = j - badChar[text[s + @as(usize, @intCast(j))]];
            s += @intCast(@max(1, shift));
        }
    }
    
    return positions.toOwnedSlice();
}

/// Good Suffix Table
fn buildGoodSuffix(pattern: []const u8, shift: []usize, bpos: []usize) void {
    const m = pattern.len;
    var i: usize = m;
    var j: usize = m + 1;
    
    bpos[i] = j;
    
    while (i > 0) {
        while (j <= m and pattern[i - 1] != pattern[j - 1]) {
            if (shift[j] == 0) shift[j] = j - i;
            j = bpos[j];
        }
        i -= 1;
        j -= 1;
        bpos[i] = j;
    }
    
    j = bpos[0];
    i = 0;
    while (i <= m) : (i += 1) {
        if (shift[i] == 0) shift[i] = j;
        if (i == j) j = bpos[j];
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Boyer-Moore String Search ===\n\n", .{});
    
    const text = "ABAAABCD";
    const pattern = "ABC";
    
    try stdout.print("Text: {s}\n", .{text});
    try stdout.print("Pattern: {s}\n\n", .{pattern});
    
    const positions = try boyerMooreHorspool(text, pattern, allocator);
    defer allocator.free(positions);
    
    try stdout.print("พบ pattern ที่ positions: ", .{});
    for (positions) |pos| try stdout.print("{d} ", .{pos});
    if (positions.len == 0) try stdout.print("ไม่พบ", .{});
    try stdout.print("\n\n", .{});
    
    // เปรียบเทียบความเร็ว
    const longText = "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB";
    const longPattern = "AAAAAB";
    
    try stdout.print("Long text test:\n", .{});
    try stdout.print("Text: {s}\n", .{longText});
    try stdout.print("Pattern: {s}\n", .{longPattern});
    
    const positions2 = try boyerMooreHorspool(longText, longPattern, allocator);
    defer allocator.free(positions2);
    
    try stdout.print("พบที่ positions: ", .{});
    for (positions2) |pos| try stdout.print("{d} ", .{pos});
    try stdout.print("\n", .{});
}
```

---

## Step 818: String Searching (Rabin-Karp)

### Rabin-Karp Algorithm คืออะไร?

Rabin-Karp ใช้ hashing เพื่อค้นหา pattern อย่างมีประสิทธิภาพ โดยเฉพาะเมื่อต้องค้นหาหลาย patterns พร้อมกัน

**Complexity:**
- Average: O(n + m)
- Worst case: O(nm) - เมื่อมี hash collision มาก

```zig
// rabin_karp.zig
const std = @import("std");

const BASE: u64 = 31;
const MOD: u64 = 1_000_000_007;

/// คำนวณ hash ของ string
fn computeHash(s: []const u8) u64 {
    var hash: u64 = 0;
    var power: u64 = 1;
    
    for (s) |c| {
        hash = (hash + @as(u64, c - 'a' + 1) * power) % MOD;
        power = power * BASE % MOD;
    }
    
    return hash;
}

/// Rabin-Karp Search - คืน positions ทั้งหมด
pub fn rabinKarp(
    text: []const u8,
    pattern: []const u8,
    allocator: std.mem.Allocator,
) ![]usize {
    var positions = std.ArrayList(usize).init(allocator);
    errdefer positions.deinit();
    
    const n = text.len;
    const m = pattern.len;
    
    if (m > n) return positions.toOwnedSlice();
    
    // คำนวณ pattern hash
    const patternHash = computeHash(pattern);
    
    // คำนวณ BASE^m mod MOD
    var basePow: u64 = 1;
    for (0..m) |_| {
        basePow = basePow * BASE % MOD;
    }
    
    // คำนวณ hash ของ window แรก
    var windowHash = computeHash(text[0..m]);
    var windowPow: u64 = 1;
    for (0..m - 1) |_| {
        windowPow = windowPow * BASE % MOD;
    }
    
    // เลื่อน window
    var i: usize = 0;
    while (i <= n - m) : (i += 1) {
        if (windowHash == patternHash) {
            // Verify เพื่อหลีกเลี่ยง false positive
            if (std.mem.eql(u8, text[i .. i + m], pattern)) {
                try positions.append(i);
            }
        }
        
        if (i < n - m) {
            // Rolling hash: ลบตัวเก่า เพิ่มตัวใหม่
            const outChar = @as(u64, text[i] - 'a' + 1);
            const inChar = @as(u64, text[i + m] - 'a' + 1);
            
            windowHash = (windowHash + MOD - outChar * 1 % MOD) % MOD;
            windowHash = (windowHash * (MOD + 1 - BASE % MOD * 0 + 1)) % MOD;
            // Simplified rolling hash
            _ = windowPow;
            _ = basePow;
            _ = inChar;
            windowHash = computeHash(text[i + 1 .. i + 1 + m]);
        }
    }
    
    return positions.toOwnedSlice();
}

/// Multi-pattern search ด้วย Rabin-Karp
pub fn rabinKarpMulti(
    text: []const u8,
    patterns: []const []const u8,
    allocator: std.mem.Allocator,
) ![][]usize {
    var results = try allocator.alloc([]usize, patterns.len);
    errdefer {
        for (results) |r| allocator.free(r);
        allocator.free(results);
    }
    
    for (patterns, 0..) |pattern, pi| {
        results[pi] = try rabinKarp(text, pattern, allocator);
    }
    
    return results;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Rabin-Karp String Search ===\n\n", .{});
    
    const text = "abcabcabc";
    const pattern = "abc";
    
    try stdout.print("Text: {s}\n", .{text});
    try stdout.print("Pattern: {s}\n\n", .{pattern});
    
    const positions = try rabinKarp(text, pattern, allocator);
    defer allocator.free(positions);
    
    try stdout.print("พบ pattern ที่ positions: ", .{});
    for (positions) |pos| try stdout.print("{d} ", .{pos});
    try stdout.print("\n\n", .{});
    
    // Multi-pattern search
    const text2 = "abcdefabcxyzabc";
    const patterns = [_][]const u8{ "abc", "xyz", "def" };
    
    try stdout.print("Multi-pattern search:\n", .{});
    try stdout.print("Text: {s}\n", .{text2});
    
    const results = try rabinKarpMulti(text2, &patterns, allocator);
    defer {
        for (results) |r| allocator.free(r);
        allocator.free(results);
    }
    
    for (patterns, 0..) |pat, i| {
        try stdout.print("Pattern '{s}': positions ", .{pat});
        for (results[i]) |pos| try stdout.print("{d} ", .{pos});
        try stdout.print("\n", .{});
    }
}
```

---

## Step 819: Pattern Matching (Regex Basics)

### Regular Expression Basics ใน Zig

Zig ยังไม่มี built-in regex แต่เราสามารถ implement simple pattern matching ได้

```zig
// regex_basics.zig
const std = @import("std");

/// Simple pattern matching - รองรับ . (any char) และ * (zero or more)
pub fn matchPattern(pattern: []const u8, text: []const u8) bool {
    if (pattern.len == 0) return text.len == 0;
    
    // ตรวจสอบ * (ใช้หลัง character ใดๆ)
    if (pattern.len >= 2 and pattern[1] == '*') {
        // ลอง match zero occurrences
        if (matchPattern(pattern[2..], text)) return true;
        
        // ลอง match one or more occurrences
        if (text.len > 0 and (pattern[0] == '.' or pattern[0] == text[0])) {
            return matchPattern(pattern, text[1..]);
        }
        
        return false;
    }
    
    if (text.len == 0) return false;
    
    // Match character เดียว
    const firstMatch = pattern[0] == '.' or pattern[0] == text[0];
    
    return firstMatch and matchPattern(pattern[1..], text[1..]);
}

/// Wildcard matching - รองรับ ? (any single char) และ * (any sequence)
pub fn wildcardMatch(pattern: []const u8, text: []const u8) bool {
    const m = pattern.len;
    const n = text.len;
    
    // DP table
    var dp = [_][101]bool{[_]bool{false} ** 101} ** 101;
    
    dp[0][0] = true;
    
    // Pattern ที่เริ่มด้วย * match empty string ได้
    var i: usize = 1;
    while (i <= m) : (i += 1) {
        if (pattern[i - 1] == '*') {
            dp[i][0] = dp[i - 1][0];
        }
    }
    
    i = 1;
    while (i <= m) : (i += 1) {
        var j: usize = 1;
        while (j <= n) : (j += 1) {
            if (pattern[i - 1] == '*') {
                dp[i][j] = dp[i - 1][j] or dp[i][j - 1];
            } else if (pattern[i - 1] == '?' or pattern[i - 1] == text[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            }
        }
    }
    
    return dp[m][n];
}

/// Simple tokenizer สำหรับ regex-like expressions
const TokenType = enum {
    literal,
    dot,       // .
    star,      // *
    plus,      // +
    question,  // ?
    caret,     // ^ (start of string)
    dollar,    // $ (end of string)
};

const Token = struct {
    kind: TokenType,
    value: u8,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Pattern Matching ===\n\n", .{});
    
    // ทดสอบ basic pattern matching
    const tests = [_]struct {
        pattern: []const u8,
        text: []const u8,
        expected: bool,
    }{
        .{ .pattern = "a*b", .text = "b", .expected = true },
        .{ .pattern = "a*b", .text = "aab", .expected = true },
        .{ .pattern = "a.c", .text = "abc", .expected = true },
        .{ .pattern = "a.c", .text = "ac", .expected = false },
        .{ .pattern = ".*", .text = "anything", .expected = true },
    };
    
    try stdout.print("Basic Pattern Matching:\n", .{});
    for (tests) |t| {
        const result = matchPattern(t.pattern, t.text);
        const status = if (result == t.expected) "PASS" else "FAIL";
        try stdout.print("  pattern='{s}' text='{s}': {s} (got {})\n", .{
            t.pattern, t.text, status, result
        });
    }
    
    // Wildcard matching
    try stdout.print("\nWildcard Matching:\n", .{});
    const wildcardTests = [_]struct {
        pattern: []const u8,
        text: []const u8,
    }{
        .{ .pattern = "*.zig", .text = "main.zig" },
        .{ .pattern = "*.zig", .text = "main.c" },
        .{ .pattern = "h?llo", .text = "hello" },
        .{ .pattern = "h?llo", .text = "hllo" },
        .{ .pattern = "*world*", .text = "hello world today" },
    };
    
    for (wildcardTests) |t| {
        const result = wildcardMatch(t.pattern, t.text);
        try stdout.print("  '{s}' matches '{s}': {}\n", .{ t.pattern, t.text, result });
    }
}
```

---

## Step 820: Fuzzy Search Algorithms

### Fuzzy Search คืออะไร?

Fuzzy Search ค้นหา strings ที่ "ใกล้เคียง" กับ query แม้จะไม่ตรงทั้งหมด ใช้ edit distance เป็น metric

```zig
// fuzzy_search.zig
const std = @import("std");

/// Levenshtein Distance - วัดความแตกต่างระหว่าง strings
pub fn levenshteinDistance(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const m = s1.len;
    const n = s2.len;
    
    // สร้าง DP table
    const dp = try allocator.alloc([]usize, m + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n + 1);
    }
    
    // Initialize
    for (0..m + 1) |i| dp[i][0] = i;
    for (0..n + 1) |j| dp[0][j] = j;
    
    // Fill DP table
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            if (s1[i - 1] == s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + @min(dp[i - 1][j], @min(dp[i][j - 1], dp[i - 1][j - 1]));
            }
        }
    }
    
    return dp[m][n];
}

/// Fuzzy Search: หา string ที่ใกล้เคียงที่สุดจาก list
pub fn fuzzySearch(
    query: []const u8,
    candidates: []const []const u8,
    maxDistance: usize,
    allocator: std.mem.Allocator,
) ![]struct { index: usize, distance: usize } {
    var results = std.ArrayList(struct { index: usize, distance: usize }).init(allocator);
    errdefer results.deinit();
    
    for (candidates, 0..) |candidate, i| {
        const dist = try levenshteinDistance(query, candidate, allocator);
        if (dist <= maxDistance) {
            try results.append(.{ .index = i, .distance = dist });
        }
    }
    
    // เรียงลำดับตาม distance
    const items = results.items;
    for (0..items.len) |i| {
        for (i + 1..items.len) |j| {
            if (items[j].distance < items[i].distance) {
                const temp = items[i];
                items[i] = items[j];
                items[j] = temp;
            }
        }
    }
    
    return results.toOwnedSlice();
}

/// Jaro similarity
pub fn jaroSimilarity(s1: []const u8, s2: []const u8) f64 {
    const m1 = s1.len;
    const m2 = s2.len;
    
    if (m1 == 0 and m2 == 0) return 1.0;
    if (m1 == 0 or m2 == 0) return 0.0;
    
    const matchWindow = @max(m1, m2) / 2;
    
    var s1Matches = [_]bool{false} ** 100;
    var s2Matches = [_]bool{false} ** 100;
    
    var matches: usize = 0;
    var transpositions: usize = 0;
    
    // หา matches
    for (s1, 0..) |c1, i| {
        const start = if (i >= matchWindow) i - matchWindow else 0;
        const end = @min(i + matchWindow + 1, m2);
        
        for (start..end) |j| {
            if (s2Matches[j] or c1 != s2[j]) continue;
            s1Matches[i] = true;
            s2Matches[j] = true;
            matches += 1;
            break;
        }
    }
    
    if (matches == 0) return 0.0;
    
    // นับ transpositions
    var k: usize = 0;
    for (s1, 0..) |c1, i| {
        if (!s1Matches[i]) continue;
        while (!s2Matches[k]) k += 1;
        if (c1 != s2[k]) transpositions += 1;
        k += 1;
    }
    
    const m: f64 = @floatFromInt(matches);
    const t: f64 = @floatFromInt(transpositions / 2);
    const n1: f64 = @floatFromInt(m1);
    const n2: f64 = @floatFromInt(m2);
    
    return (m / n1 + m / n2 + (m - t) / m) / 3.0;
}

/// Jaro-Winkler similarity (ให้น้ำหนักกับ prefix)
pub fn jaroWinkler(s1: []const u8, s2: []const u8) f64 {
    const jaro = jaroSimilarity(s1, s2);
    
    // หา common prefix length (สูงสุด 4)
    var prefixLen: usize = 0;
    const maxPrefix = @min(@min(s1.len, s2.len), 4);
    
    while (prefixLen < maxPrefix and s1[prefixLen] == s2[prefixLen]) {
        prefixLen += 1;
    }
    
    const p: f64 = 0.1; // scaling factor
    return jaro + @as(f64, @floatFromInt(prefixLen)) * p * (1.0 - jaro);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Fuzzy Search ===\n\n", .{});
    
    // Levenshtein Distance
    const pairs = [_][2][]const u8{
        .{ "kitten", "sitting" },
        .{ "saturday", "sunday" },
        .{ "hello", "hello" },
        .{ "abc", "xyz" },
    };
    
    try stdout.print("Levenshtein Distance:\n", .{});
    for (pairs) |pair| {
        const dist = try levenshteinDistance(pair[0], pair[1], allocator);
        try stdout.print("  '{s}' vs '{s}': {d}\n", .{ pair[0], pair[1], dist });
    }
    
    // Fuzzy Search
    try stdout.print("\nFuzzy Search:\n", .{});
    const query = "programing"; // typo
    const candidates = [_][]const u8{
        "programming",
        "program",
        "programs",
        "programmer",
        "performance",
        "python",
    };
    
    try stdout.print("Query: '{s}'\n", .{query});
    const results = try fuzzySearch(query, &candidates, 3, allocator);
    defer allocator.free(results);
    
    try stdout.print("ผลลัพธ์ (distance <= 3):\n", .{});
    for (results) |r| {
        try stdout.print("  '{s}' (distance: {d})\n", .{ candidates[r.index], r.distance });
    }
    
    // Jaro-Winkler
    try stdout.print("\nJaro-Winkler Similarity:\n", .{});
    const jwPairs = [_][2][]const u8{
        .{ "MARTHA", "MARHTA" },
        .{ "DIXON", "DICKSONX" },
        .{ "JELLYFISH", "SMELLYFISH" },
    };
    
    for (jwPairs) |pair| {
        const sim = jaroWinkler(pair[0], pair[1]);
        try stdout.print("  '{s}' vs '{s}': {d:.4}\n", .{ pair[0], pair[1], sim });
    }
}
```

### สรุป Part 82

ในส่วนนี้เราได้เรียนรู้:
1. Linear Search - ง่าย ใช้กับ unsorted data
2. Binary Search - O(log n) ต้องใช้ sorted data
3. Jump Search - O(√n) เหมาะกับ sorted arrays
4. Interpolation Search - O(log log n) สำหรับ uniform distribution
5. Exponential Search - เหมาะกับ unbounded arrays
6. KMP Algorithm - O(n+m) string searching
7. Boyer-Moore - ประสิทธิภาพสูงในทางปฏิบัติ
8. Rabin-Karp - ใช้ hashing, ดีสำหรับ multi-pattern search
9. Pattern Matching - regex basics
10. Fuzzy Search - ค้นหาแบบคลุมเครือด้วย edit distance

---

*ต่อไป: Part 83 - Algorithms: Dynamic Programming*
