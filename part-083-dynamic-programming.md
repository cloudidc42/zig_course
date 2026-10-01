# Part 83: Algorithms: Dynamic Programming
## Steps 821-830

---

## Step 821: DP Concept และ Memoization

### Dynamic Programming คืออะไร?

Dynamic Programming (DP) เป็นเทคนิคการเขียนโปรแกรมที่แก้ปัญหาซับซ้อนโดยแบ่งเป็นปัญหาย่อยที่ทับซ้อนกัน (overlapping subproblems) และเก็บผลลัพธ์เพื่อหลีกเลี่ยงการคำนวณซ้ำ

### หลักการพื้นฐาน

1. **Overlapping Subproblems**: ปัญหาย่อยเดียวกันถูกแก้ซ้ำหลายครั้ง
2. **Optimal Substructure**: คำตอบที่ดีที่สุดของปัญหาหลักสามารถสร้างจากคำตอบที่ดีที่สุดของปัญหาย่อย

### สองวิธีหลัก

```zig
// dp_concepts.zig
const std = @import("std");

// ================================
// วิธีที่ 1: Top-Down (Memoization)
// ================================

/// Fibonacci ด้วย Memoization (Top-Down)
const FibMemo = struct {
    cache: std.AutoHashMap(u64, u64),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) FibMemo {
        return .{
            .cache = std.AutoHashMap(u64, u64).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *FibMemo) void {
        self.cache.deinit();
    }
    
    pub fn fib(self: *FibMemo, n: u64) !u64 {
        if (n <= 1) return n;
        
        // ตรวจสอบ cache ก่อน
        if (self.cache.get(n)) |cached| return cached;
        
        // คำนวณและเก็บใน cache
        const result = try self.fib(n - 1) + try self.fib(n - 2);
        try self.cache.put(n, result);
        return result;
    }
};

// ================================
// วิธีที่ 2: Bottom-Up (Tabulation)
// ================================

/// Fibonacci ด้วย Tabulation (Bottom-Up)
pub fn fibTabulation(n: u32, allocator: std.mem.Allocator) !u64 {
    if (n <= 1) return n;
    
    const dp = try allocator.alloc(u64, n + 1);
    defer allocator.free(dp);
    
    dp[0] = 0;
    dp[1] = 1;
    
    for (2..n + 1) |i| {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    
    return dp[n];
}

/// Fibonacci แบบ Space-Optimized (O(1) space)
pub fn fibOptimized(n: u32) u64 {
    if (n <= 1) return n;
    
    var prev2: u64 = 0;
    var prev1: u64 = 1;
    
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}

/// เปรียบเทียบประสิทธิภาพ
pub fn measureTime() !void {
    const stdout = std.io.getStdOut().writer();
    
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    try stdout.print("=== Fibonacci Performance Comparison ===\n\n", .{});
    
    // Top-Down
    var memo = FibMemo.init(allocator);
    defer memo.deinit();
    
    const start1 = std.time.nanoTimestamp();
    const result1 = try memo.fib(40);
    const time1 = std.time.nanoTimestamp() - start1;
    try stdout.print("Memoization fib(40) = {d}, time: {d}ns\n", .{ result1, time1 });
    
    // Bottom-Up
    const start2 = std.time.nanoTimestamp();
    const result2 = try fibTabulation(40, allocator);
    const time2 = std.time.nanoTimestamp() - start2;
    try stdout.print("Tabulation fib(40) = {d}, time: {d}ns\n", .{ result2, time2 });
    
    // Optimized
    const start3 = std.time.nanoTimestamp();
    const result3 = fibOptimized(40);
    const time3 = std.time.nanoTimestamp() - start3;
    try stdout.print("Optimized fib(40) = {d}, time: {d}ns\n", .{ result3, time3 });
}

pub fn main() !void {
    try measureTime();
    
    const stdout = std.io.getStdOut().writer();
    
    // แสดง Fibonacci sequence
    try stdout.print("\nFibonacci sequence (0-20):\n", .{});
    for (0..21) |i| {
        try stdout.print("fib({d:2}) = {d}\n", .{ i, fibOptimized(@intCast(i)) });
    }
}
```

---

## Step 822: Fibonacci ด้วย DP

### Fibonacci ทุกรูปแบบ

```zig
// fibonacci_dp.zig
const std = @import("std");

/// Matrix exponentiation สำหรับ Fibonacci ที่รวดเร็ว O(log n)
fn matMul(a: [2][2]u64, b: [2][2]u64) [2][2]u64 {
    return [2][2]u64{
        [2]u64{
            a[0][0] * b[0][0] + a[0][1] * b[1][0],
            a[0][0] * b[0][1] + a[0][1] * b[1][1],
        },
        [2]u64{
            a[1][0] * b[0][0] + a[1][1] * b[1][0],
            a[1][0] * b[0][1] + a[1][1] * b[1][1],
        },
    };
}

fn matPow(m: [2][2]u64, n: u64) [2][2]u64 {
    if (n == 1) return m;
    
    if (n % 2 == 0) {
        const half = matPow(m, n / 2);
        return matMul(half, half);
    } else {
        return matMul(m, matPow(m, n - 1));
    }
}

pub fn fibMatrix(n: u64) u64 {
    if (n == 0) return 0;
    if (n == 1) return 1;
    
    const base = [2][2]u64{
        [2]u64{ 1, 1 },
        [2]u64{ 1, 0 },
    };
    
    const result = matPow(base, n);
    return result[0][1];
}

/// Tribonacci (T(n) = T(n-1) + T(n-2) + T(n-3))
pub fn tribonacci(n: u32, allocator: std.mem.Allocator) !u64 {
    if (n == 0) return 0;
    if (n <= 2) return 1;
    
    const dp = try allocator.alloc(u64, n + 1);
    defer allocator.free(dp);
    
    dp[0] = 0;
    dp[1] = 1;
    dp[2] = 1;
    
    for (3..n + 1) |i| {
        dp[i] = dp[i - 1] + dp[i - 2] + dp[i - 3];
    }
    
    return dp[n];
}

/// Climbing Stairs: ขึ้นบันได n ขั้น ทีละ 1 หรือ 2 ขั้น - มีกี่วิธี?
pub fn climbingStairs(n: u32) u64 {
    if (n <= 2) return n;
    
    var prev2: u64 = 1;
    var prev1: u64 = 2;
    
    var i: u32 = 3;
    while (i <= n) : (i += 1) {
        const curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}

/// Climbing Stairs ที่ขึ้นได้ k ขั้น
pub fn climbingStairsK(n: u32, k: u32, allocator: std.mem.Allocator) !u64 {
    const dp = try allocator.alloc(u64, n + 1);
    defer allocator.free(dp);
    @memset(dp, 0);
    
    dp[0] = 1;
    
    for (1..n + 1) |i| {
        var j: u32 = 1;
        while (j <= k and j <= i) : (j += 1) {
            dp[i] += dp[i - j];
        }
    }
    
    return dp[n];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Fibonacci DP Variants ===\n\n", .{});
    
    // Matrix Fibonacci
    try stdout.print("Matrix Fibonacci:\n", .{});
    const fibNums = [_]u64{ 0, 1, 5, 10, 20, 30, 40 };
    for (fibNums) |n| {
        try stdout.print("  fib({d}) = {d}\n", .{ n, fibMatrix(n) });
    }
    
    // Tribonacci
    try stdout.print("\nTribonacci:\n", .{});
    for (0..10) |i| {
        const t = try tribonacci(@intCast(i), allocator);
        try stdout.print("  T({d}) = {d}\n", .{ i, t });
    }
    
    // Climbing Stairs
    try stdout.print("\nClimbing Stairs:\n", .{});
    for (1..10) |n| {
        try stdout.print("  stairs({d}) = {d} ways\n", .{ n, climbingStairs(@intCast(n)) });
    }
    
    // Climbing Stairs with k steps
    try stdout.print("\nClimbing Stairs (k=3 steps max):\n", .{});
    for (1..8) |n| {
        const ways = try climbingStairsK(@intCast(n), 3, allocator);
        try stdout.print("  stairs({d}, k=3) = {d} ways\n", .{ n, ways });
    }
}
```

---

## Step 823: Longest Common Subsequence (LCS)

### LCS คืออะไร?

LCS หาลำดับย่อยที่ยาวที่สุดที่ปรากฏในทั้งสอง sequences (ไม่ต้องต่อเนื่อง)

**Complexity:**
- Time: O(mn)
- Space: O(mn) หรือ O(min(m,n)) แบบ optimized

```zig
// lcs.zig
const std = @import("std");

/// LCS Length
pub fn lcsLength(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const m = s1.len;
    const n = s2.len;
    
    // สร้าง 2D DP table
    const dp = try allocator.alloc([]usize, m + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n + 1);
        @memset(row.*, 0);
    }
    
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            if (s1[i - 1] == s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = @max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[m][n];
}

/// LCS String (backtrack เพื่อหา actual LCS)
pub fn lcsString(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) ![]u8 {
    const m = s1.len;
    const n = s2.len;
    
    const dp = try allocator.alloc([]usize, m + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n + 1);
        @memset(row.*, 0);
    }
    
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            if (s1[i - 1] == s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = @max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    // Backtrack เพื่อหา LCS string
    var result = std.ArrayList(u8).init(allocator);
    errdefer result.deinit();
    
    var i = m;
    var j = n;
    
    while (i > 0 and j > 0) {
        if (s1[i - 1] == s2[j - 1]) {
            try result.append(s1[i - 1]);
            i -= 1;
            j -= 1;
        } else if (dp[i - 1][j] > dp[i][j - 1]) {
            i -= 1;
        } else {
            j -= 1;
        }
    }
    
    // Reverse
    std.mem.reverse(u8, result.items);
    return result.toOwnedSlice();
}

/// Shortest Common Supersequence
pub fn scsLength(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const lcs = try lcsLength(s1, s2, allocator);
    return s1.len + s2.len - lcs;
}

/// Longest Common Substring (continuous)
pub fn lcSubstring(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const m = s1.len;
    const n = s2.len;
    var maxLen: usize = 0;
    
    const dp = try allocator.alloc([]usize, m + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n + 1);
        @memset(row.*, 0);
    }
    
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            if (s1[i - 1] == s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
                maxLen = @max(maxLen, dp[i][j]);
            } else {
                dp[i][j] = 0;
            }
        }
    }
    
    return maxLen;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Longest Common Subsequence ===\n\n", .{});
    
    const pairs = [_][2][]const u8{
        .{ "ABCBDAB", "BDCAB" },
        .{ "AGGTAB", "GXTXAYB" },
        .{ "programming", "gaming" },
    };
    
    for (pairs) |pair| {
        const s1 = pair[0];
        const s2 = pair[1];
        
        const len = try lcsLength(s1, s2, allocator);
        const lcs = try lcsString(s1, s2, allocator);
        defer allocator.free(lcs);
        
        const scs = try scsLength(s1, s2, allocator);
        const lcSub = try lcSubstring(s1, s2, allocator);
        
        try stdout.print("s1='{s}', s2='{s}'\n", .{ s1, s2 });
        try stdout.print("  LCS length: {d}, LCS: '{s}'\n", .{ len, lcs });
        try stdout.print("  SCS length: {d}\n", .{scs});
        try stdout.print("  Longest Common Substring: {d}\n\n", .{lcSub});
    }
}
```

---

## Step 824: Longest Increasing Subsequence (LIS)

### LIS คืออะไร?

LIS หาลำดับย่อยที่ยาวที่สุดที่มีค่าเพิ่มขึ้นอย่างเคร่งครัด

```zig
// lis.zig
const std = @import("std");

/// LIS แบบ O(n²) DP
pub fn lisN2(arr: []const i32, allocator: std.mem.Allocator) !usize {
    const n = arr.len;
    if (n == 0) return 0;
    
    const dp = try allocator.alloc(usize, n);
    defer allocator.free(dp);
    @memset(dp, 1); // ทุก element เป็น LIS ของตัวเอง
    
    var maxLen: usize = 1;
    
    for (1..n) |i| {
        for (0..i) |j| {
            if (arr[j] < arr[i]) {
                dp[i] = @max(dp[i], dp[j] + 1);
            }
        }
        maxLen = @max(maxLen, dp[i]);
    }
    
    return maxLen;
}

/// LIS แบบ O(n log n) ด้วย patience sorting
pub fn lisNLogN(arr: []const i32, allocator: std.mem.Allocator) !usize {
    const n = arr.len;
    if (n == 0) return 0;
    
    // tails[i] = ค่าเล็กที่สุดที่ LIS ความยาว i+1 จบด้วย
    var tails = std.ArrayList(i32).init(allocator);
    defer tails.deinit();
    
    for (arr) |v| {
        // Binary search หาตำแหน่งที่จะแทน
        var lo: usize = 0;
        var hi: usize = tails.items.len;
        
        while (lo < hi) {
            const mid = lo + (hi - lo) / 2;
            if (tails.items[mid] < v) {
                lo = mid + 1;
            } else {
                hi = mid;
            }
        }
        
        if (lo == tails.items.len) {
            try tails.append(v);
        } else {
            tails.items[lo] = v;
        }
    }
    
    return tails.items.len;
}

/// หา actual LIS sequence
pub fn lisSequence(arr: []const i32, allocator: std.mem.Allocator) ![]i32 {
    const n = arr.len;
    if (n == 0) return allocator.alloc(i32, 0);
    
    const dp = try allocator.alloc(usize, n);
    defer allocator.free(dp);
    const parent = try allocator.alloc(i64, n);
    defer allocator.free(parent);
    
    @memset(dp, 1);
    for (parent) |*p| p.* = -1;
    
    var maxLen: usize = 1;
    var maxIdx: usize = 0;
    
    for (1..n) |i| {
        for (0..i) |j| {
            if (arr[j] < arr[i] and dp[j] + 1 > dp[i]) {
                dp[i] = dp[j] + 1;
                parent[i] = @intCast(j);
            }
        }
        if (dp[i] > maxLen) {
            maxLen = dp[i];
            maxIdx = i;
        }
    }
    
    // Backtrack
    var result = try allocator.alloc(i32, maxLen);
    var idx: i64 = @intCast(maxIdx);
    var pos = maxLen;
    
    while (idx >= 0) {
        pos -= 1;
        result[pos] = arr[@intCast(idx)];
        idx = parent[@intCast(idx)];
    }
    
    return result;
}

/// Number of LIS (นับจำนวน LIS ทั้งหมด)
pub fn countLIS(arr: []const i32, allocator: std.mem.Allocator) !usize {
    const n = arr.len;
    if (n == 0) return 0;
    
    const dp = try allocator.alloc(usize, n);
    defer allocator.free(dp);
    const cnt = try allocator.alloc(usize, n);
    defer allocator.free(cnt);
    
    @memset(dp, 1);
    @memset(cnt, 1);
    
    var maxLen: usize = 1;
    
    for (1..n) |i| {
        for (0..i) |j| {
            if (arr[j] < arr[i]) {
                if (dp[j] + 1 > dp[i]) {
                    dp[i] = dp[j] + 1;
                    cnt[i] = cnt[j];
                } else if (dp[j] + 1 == dp[i]) {
                    cnt[i] += cnt[j];
                }
            }
        }
        maxLen = @max(maxLen, dp[i]);
    }
    
    var total: usize = 0;
    for (0..n) |i| {
        if (dp[i] == maxLen) total += cnt[i];
    }
    
    return total;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Longest Increasing Subsequence ===\n\n", .{});
    
    const arr = [_]i32{ 10, 9, 2, 5, 3, 7, 101, 18 };
    
    try stdout.print("Array: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n\n", .{});
    
    const len1 = try lisN2(&arr, allocator);
    const len2 = try lisNLogN(&arr, allocator);
    
    try stdout.print("LIS length (O(n²)): {d}\n", .{len1});
    try stdout.print("LIS length (O(n log n)): {d}\n", .{len2});
    
    const seq = try lisSequence(&arr, allocator);
    defer allocator.free(seq);
    
    try stdout.print("LIS sequence: ", .{});
    for (seq) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    const count = try countLIS(&arr, allocator);
    try stdout.print("Number of LIS: {d}\n", .{count});
    
    // อีกตัวอย่าง
    const arr2 = [_]i32{ 1, 3, 6, 7, 9, 4, 10, 5, 6 };
    try stdout.print("\nArray: ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    const seq2 = try lisSequence(&arr2, allocator);
    defer allocator.free(seq2);
    
    try stdout.print("LIS: ", .{});
    for (seq2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("(length: {d})\n", .{seq2.len});
}
```

---

## Step 825: Knapsack Problem

### Knapsack Problem คืออะไร?

มีของหลายชิ้น แต่ละชิ้นมีน้ำหนักและมูลค่า ต้องการเลือกของให้ได้มูลค่าสูงสุดโดยน้ำหนักรวมไม่เกิน W

```zig
// knapsack.zig
const std = @import("std");

const Item = struct {
    weight: usize,
    value: usize,
    name: []const u8,
};

/// 0/1 Knapsack - แต่ละชิ้นใช้ได้ครั้งเดียว
pub fn knapsack01(items: []const Item, capacity: usize, allocator: std.mem.Allocator) !usize {
    const n = items.len;
    
    // dp[i][w] = มูลค่าสูงสุดเมื่อใช้ items[0..i] และ capacity = w
    const dp = try allocator.alloc([]usize, n + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, capacity + 1);
        @memset(row.*, 0);
    }
    
    for (1..n + 1) |i| {
        for (0..capacity + 1) |w| {
            // ไม่เลือก item i
            dp[i][w] = dp[i - 1][w];
            
            // เลือก item i (ถ้าใส่ได้)
            if (items[i - 1].weight <= w) {
                const withItem = dp[i - 1][w - items[i - 1].weight] + items[i - 1].value;
                dp[i][w] = @max(dp[i][w], withItem);
            }
        }
    }
    
    return dp[n][capacity];
}

/// หาว่าเลือก items อะไรบ้าง
pub fn knapsack01WithItems(
    items: []const Item,
    capacity: usize,
    allocator: std.mem.Allocator,
) !struct { value: usize, selected: []bool } {
    const n = items.len;
    
    const dp = try allocator.alloc([]usize, n + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, capacity + 1);
        @memset(row.*, 0);
    }
    
    for (1..n + 1) |i| {
        for (0..capacity + 1) |w| {
            dp[i][w] = dp[i - 1][w];
            if (items[i - 1].weight <= w) {
                const withItem = dp[i - 1][w - items[i - 1].weight] + items[i - 1].value;
                dp[i][w] = @max(dp[i][w], withItem);
            }
        }
    }
    
    // Backtrack
    const selected = try allocator.alloc(bool, n);
    @memset(selected, false);
    
    var w = capacity;
    var i = n;
    while (i > 0) : (i -= 1) {
        if (dp[i][w] != dp[i - 1][w]) {
            selected[i - 1] = true;
            w -= items[i - 1].weight;
        }
    }
    
    return .{ .value = dp[n][capacity], .selected = selected };
}

/// Unbounded Knapsack - แต่ละชิ้นใช้ได้หลายครั้ง
pub fn knapsackUnbounded(items: []const Item, capacity: usize, allocator: std.mem.Allocator) !usize {
    const dp = try allocator.alloc(usize, capacity + 1);
    defer allocator.free(dp);
    @memset(dp, 0);
    
    for (1..capacity + 1) |w| {
        for (items) |item| {
            if (item.weight <= w) {
                dp[w] = @max(dp[w], dp[w - item.weight] + item.value);
            }
        }
    }
    
    return dp[capacity];
}

/// Fractional Knapsack (Greedy - ไม่ใช่ DP แต่เอาไว้เปรียบเทียบ)
pub fn knapsackFractional(items: []Item, capacity: usize) f64 {
    // เรียงลำดับตาม value/weight ratio
    for (0..items.len) |i| {
        for (i + 1..items.len) |j| {
            const ri: f64 = @as(f64, @floatFromInt(items[i].value)) /
                           @as(f64, @floatFromInt(items[i].weight));
            const rj: f64 = @as(f64, @floatFromInt(items[j].value)) /
                           @as(f64, @floatFromInt(items[j].weight));
            if (rj > ri) {
                const temp = items[i];
                items[i] = items[j];
                items[j] = temp;
            }
        }
    }
    
    var totalValue: f64 = 0;
    var remainCap: f64 = @floatFromInt(capacity);
    
    for (items) |item| {
        if (remainCap == 0) break;
        const w: f64 = @floatFromInt(item.weight);
        const v: f64 = @floatFromInt(item.value);
        
        if (w <= remainCap) {
            totalValue += v;
            remainCap -= w;
        } else {
            totalValue += v * (remainCap / w);
            remainCap = 0;
        }
    }
    
    return totalValue;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Knapsack Problem ===\n\n", .{});
    
    var items = [_]Item{
        .{ .weight = 2, .value = 6,  .name = "โทรศัพท์" },
        .{ .weight = 2, .value = 10, .name = "แล็ปท็อป" },
        .{ .weight = 3, .value = 12, .name = "กล้อง" },
        .{ .weight = 5, .value = 13, .name = "แท็บเล็ต" },
    };
    const capacity: usize = 5;
    
    try stdout.print("Items:\n", .{});
    for (items) |item| {
        const ratio: f64 = @as(f64, @floatFromInt(item.value)) /
                          @as(f64, @floatFromInt(item.weight));
        try stdout.print("  {s}: weight={d}, value={d}, ratio={d:.2}\n", .{
            item.name, item.weight, item.value, ratio
        });
    }
    try stdout.print("Capacity: {d}\n\n", .{capacity});
    
    // 0/1 Knapsack
    const result = try knapsack01WithItems(&items, capacity, allocator);
    defer allocator.free(result.selected);
    
    try stdout.print("0/1 Knapsack:\n", .{});
    try stdout.print("  Max value: {d}\n", .{result.value});
    try stdout.print("  Selected items: ", .{});
    for (result.selected, 0..) |sel, i| {
        if (sel) try stdout.print("{s} ", .{items[i].name});
    }
    try stdout.print("\n\n", .{});
    
    // Unbounded Knapsack
    const unbounded = try knapsackUnbounded(&items, capacity, allocator);
    try stdout.print("Unbounded Knapsack max value: {d}\n", .{unbounded});
    
    // Fractional Knapsack
    const fractional = knapsackFractional(&items, capacity);
    try stdout.print("Fractional Knapsack max value: {d:.2}\n", .{fractional});
}
```

---

## Step 826: Coin Change Problem

```zig
// coin_change.zig
const std = @import("std");

/// Coin Change: จำนวน coins น้อยที่สุดที่รวมได้ amount
pub fn coinChange(coins: []const u32, amount: u32, allocator: std.mem.Allocator) !i64 {
    const dp = try allocator.alloc(i64, amount + 1);
    defer allocator.free(dp);
    
    // Initialize ด้วยค่า infinity
    for (dp) |*v| v.* = @intCast(amount + 1);
    dp[0] = 0;
    
    for (1..amount + 1) |i| {
        for (coins) |coin| {
            if (coin <= i) {
                dp[i] = @min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    return if (dp[amount] > @as(i64, amount)) -1 else dp[amount];
}

/// นับจำนวนวิธีทั้งหมดที่จะแตกเงิน
pub fn coinChangeWays(coins: []const u32, amount: u32, allocator: std.mem.Allocator) !u64 {
    const dp = try allocator.alloc(u64, amount + 1);
    defer allocator.free(dp);
    @memset(dp, 0);
    dp[0] = 1;
    
    for (coins) |coin| {
        for (coin..amount + 1) |i| {
            dp[i] += dp[i - coin];
        }
    }
    
    return dp[amount];
}

/// Perfect Squares: จำนวนน้อยที่สุดของ perfect squares ที่รวมได้ n
pub fn numSquares(n: u32, allocator: std.mem.Allocator) !u32 {
    const dp = try allocator.alloc(u32, n + 1);
    defer allocator.free(dp);
    
    for (dp) |*v| v.* = n + 1;
    dp[0] = 0;
    
    for (1..n + 1) |i| {
        var j: u32 = 1;
        while (j * j <= i) : (j += 1) {
            dp[i] = @min(dp[i], dp[i - j * j] + 1);
        }
    }
    
    return dp[n];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Coin Change Problem ===\n\n", .{});
    
    const coins = [_]u32{ 1, 5, 10, 25 };
    const amounts = [_]u32{ 11, 30, 100, 7 };
    
    try stdout.print("Coins: 1¢, 5¢, 10¢, 25¢\n\n", .{});
    
    for (amounts) |amount| {
        const minCoins = try coinChange(&coins, amount, allocator);
        const ways = try coinChangeWays(&coins, amount, allocator);
        
        try stdout.print("Amount {d}¢:\n", .{amount});
        if (minCoins >= 0) {
            try stdout.print("  Min coins: {d}\n", .{minCoins});
        } else {
            try stdout.print("  ไม่สามารถแตกได้\n", .{});
        }
        try stdout.print("  จำนวนวิธี: {d}\n\n", .{ways});
    }
    
    // Perfect Squares
    try stdout.print("Perfect Squares:\n", .{});
    const nums = [_]u32{ 12, 13, 100, 43 };
    for (nums) |n| {
        const sq = try numSquares(n, allocator);
        try stdout.print("  numSquares({d}) = {d}\n", .{ n, sq });
    }
}
```

---

## Step 827: Edit Distance (Levenshtein)

```zig
// edit_distance.zig
const std = @import("std");

/// Edit Distance (Levenshtein) - จำนวน operations น้อยที่สุดที่แปลง s1 เป็น s2
pub fn editDistance(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const m = s1.len;
    const n = s2.len;
    
    const dp = try allocator.alloc([]usize, m + 1);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    for (dp, 0..) |*row, i| {
        row.* = try allocator.alloc(usize, n + 1);
        row.*[0] = i;
    }
    for (0..n + 1) |j| dp[0][j] = j;
    
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            if (s1[i - 1] == s2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + @min(
                    dp[i - 1][j],     // delete
                    @min(
                        dp[i][j - 1], // insert
                        dp[i - 1][j - 1] // replace
                    )
                );
            }
        }
    }
    
    return dp[m][n];
}

/// Damerau-Levenshtein Distance (เพิ่ม transposition)
pub fn damerauDistance(s1: []const u8, s2: []const u8, allocator: std.mem.Allocator) !usize {
    const m = s1.len;
    const n = s2.len;
    
    const dp = try allocator.alloc([]usize, m + 2);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n + 2);
        @memset(row.*, 0);
    }
    
    const inf = m + n + 1;
    dp[0][0] = inf;
    for (0..m + 1) |i| {
        dp[i + 1][0] = inf;
        dp[i + 1][1] = i;
    }
    for (0..n + 1) |j| {
        dp[0][j + 1] = inf;
        dp[1][j + 1] = j;
    }
    
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            const cost: usize = if (s1[i - 1] == s2[j - 1]) 0 else 1;
            dp[i + 1][j + 1] = @min(
                dp[i][j] + cost,
                @min(
                    dp[i + 1][j] + 1,
                    dp[i][j + 1] + 1
                )
            );
            
            // Transposition
            if (i > 1 and j > 1 and s1[i - 1] == s2[j - 2] and s1[i - 2] == s2[j - 1]) {
                dp[i + 1][j + 1] = @min(dp[i + 1][j + 1], dp[i - 1][j - 1] + cost);
            }
        }
    }
    
    return dp[m + 1][n + 1];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Edit Distance ===\n\n", .{});
    
    const pairs = [_][2][]const u8{
        .{ "kitten", "sitting" },
        .{ "saturday", "sunday" },
        .{ "intention", "execution" },
        .{ "abc", "abc" },
        .{ "", "abc" },
    };
    
    for (pairs) |pair| {
        const ed = try editDistance(pair[0], pair[1], allocator);
        try stdout.print("'{s}' -> '{s}': {d} operations\n", .{ pair[0], pair[1], ed });
    }
    
    // แสดง operations
    try stdout.print("\nตัวอย่าง 'kitten' -> 'sitting':\n", .{});
    try stdout.print("  kitten -> sitten (replace k->s)\n", .{});
    try stdout.print("  sitten -> sittin (replace e->i)\n", .{});
    try stdout.print("  sittin -> sitting (insert g)\n", .{});
    try stdout.print("  Total: 3 operations\n", .{});
}
```

---

## Step 828: Matrix Chain Multiplication

```zig
// matrix_chain.zig
const std = @import("std");

/// Matrix Chain Order - หาลำดับการคูณ matrices ที่ใช้ operations น้อยที่สุด
pub fn matrixChainOrder(dims: []const usize, allocator: std.mem.Allocator) !usize {
    const n = dims.len - 1; // จำนวน matrices
    
    // dp[i][j] = จำนวน operations น้อยที่สุดสำหรับ matrices i..j
    const dp = try allocator.alloc([]usize, n);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n);
        @memset(row.*, 0);
    }
    
    // chain ความยาว l
    for (2..n + 1) |l| {
        for (0..n - l + 1) |i| {
            const j = i + l - 1;
            dp[i][j] = std.math.maxInt(usize);
            
            for (i..j) |k| {
                const cost = dp[i][k] + dp[k + 1][j] + dims[i] * dims[k + 1] * dims[j + 1];
                dp[i][j] = @min(dp[i][j], cost);
            }
        }
    }
    
    return dp[0][n - 1];
}

/// หา optimal parenthesization
pub fn matrixChainOptimal(dims: []const usize, allocator: std.mem.Allocator) ![][]usize {
    const n = dims.len - 1;
    
    const dp = try allocator.alloc([]usize, n);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    
    const split = try allocator.alloc([]usize, n);
    for (dp) |*row| row.* = try allocator.alloc(usize, n);
    for (split) |*row| {
        row.* = try allocator.alloc(usize, n);
        @memset(row.*, 0);
    }
    for (dp) |row| @memset(row, 0);
    
    for (2..n + 1) |l| {
        for (0..n - l + 1) |i| {
            const j = i + l - 1;
            dp[i][j] = std.math.maxInt(usize);
            
            for (i..j) |k| {
                const cost = dp[i][k] + dp[k + 1][j] + dims[i] * dims[k + 1] * dims[j + 1];
                if (cost < dp[i][j]) {
                    dp[i][j] = cost;
                    split[i][j] = k;
                }
            }
        }
    }
    
    return split;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Matrix Chain Multiplication ===\n\n", .{});
    
    // dims[i] x dims[i+1] คือขนาดของ matrix i
    const dims = [_]usize{ 40, 20, 30, 10, 30 };
    
    try stdout.print("Matrices:\n", .{});
    for (0..dims.len - 1) |i| {
        try stdout.print("  A{d}: {d}x{d}\n", .{ i + 1, dims[i], dims[i + 1] });
    }
    
    const minOps = try matrixChainOrder(&dims, allocator);
    try stdout.print("\nจำนวน operations น้อยที่สุด: {d}\n", .{minOps});
    
    // อีกตัวอย่าง
    const dims2 = [_]usize{ 10, 30, 5, 60 };
    try stdout.print("\nMatrices 2:\n", .{});
    for (0..dims2.len - 1) |i| {
        try stdout.print("  A{d}: {d}x{d}\n", .{ i + 1, dims2[i], dims2[i + 1] });
    }
    
    const minOps2 = try matrixChainOrder(&dims2, allocator);
    try stdout.print("จำนวน operations น้อยที่สุด: {d}\n", .{minOps2});
    try stdout.print("(การคูณไม่เหมาะสม: {d})\n", .{
        dims2[0] * dims2[1] * dims2[2] * dims2[3] // naive
    });
}
```

---

## Step 829: Longest Palindromic Subsequence

```zig
// palindrome_dp.zig
const std = @import("std");

/// Longest Palindromic Subsequence
pub fn longestPalindromicSubsequence(s: []const u8, allocator: std.mem.Allocator) !usize {
    const n = s.len;
    
    // dp[i][j] = LPS ของ s[i..j]
    const dp = try allocator.alloc([]usize, n);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    for (dp) |*row| {
        row.* = try allocator.alloc(usize, n);
        @memset(row.*, 0);
    }
    
    // Base case: single characters
    for (0..n) |i| dp[i][i] = 1;
    
    // Fill ตามความยาว
    for (2..n + 1) |l| {
        for (0..n - l + 1) |i| {
            const j = i + l - 1;
            if (s[i] == s[j]) {
                dp[i][j] = dp[i + 1][j - 1] + 2;
            } else {
                dp[i][j] = @max(dp[i + 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[0][n - 1];
}

/// Longest Palindromic Substring
pub fn longestPalindromicSubstring(s: []const u8, allocator: std.mem.Allocator) ![]const u8 {
    const n = s.len;
    if (n == 0) return s[0..0];
    
    const dp = try allocator.alloc([]bool, n);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    for (dp) |*row| {
        row.* = try allocator.alloc(bool, n);
        @memset(row.*, false);
    }
    
    var start: usize = 0;
    var maxLen: usize = 1;
    
    // Single chars
    for (0..n) |i| dp[i][i] = true;
    
    // Two chars
    for (0..n - 1) |i| {
        if (s[i] == s[i + 1]) {
            dp[i][i + 1] = true;
            start = i;
            maxLen = 2;
        }
    }
    
    // Longer
    for (3..n + 1) |l| {
        for (0..n - l + 1) |i| {
            const j = i + l - 1;
            if (s[i] == s[j] and dp[i + 1][j - 1]) {
                dp[i][j] = true;
                if (l > maxLen) {
                    maxLen = l;
                    start = i;
                }
            }
        }
    }
    
    return s[start .. start + maxLen];
}

/// Minimum Insertions to make string palindrome
pub fn minInsertions(s: []const u8, allocator: std.mem.Allocator) !usize {
    const lps = try longestPalindromicSubsequence(s, allocator);
    return s.len - lps;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Palindrome DP ===\n\n", .{});
    
    const strings = [_][]const u8{ "BBABCBCAB", "GEEKSFORGEEKS", "abcba", "character" };
    
    for (strings) |s| {
        const lps = try longestPalindromicSubsequence(s, allocator);
        const lpSub = try longestPalindromicSubstring(s, allocator);
        const minIns = try minInsertions(s, allocator);
        
        try stdout.print("'{s}':\n", .{s});
        try stdout.print("  LPS length: {d}\n", .{lps});
        try stdout.print("  Longest palindromic substring: '{s}'\n", .{lpSub});
        try stdout.print("  Min insertions to palindrome: {d}\n\n", .{minIns});
    }
}
```

---

## Step 830: DP Patterns สรุป

```zig
// dp_patterns.zig
const std = @import("std");

/// Pattern 1: 1D DP - Maximum Subarray (Kadane's Algorithm)
pub fn maxSubarray(arr: []const i32) i32 {
    var maxSum = arr[0];
    var currSum = arr[0];
    
    for (arr[1..]) |v| {
        currSum = @max(v, currSum + v);
        maxSum = @max(maxSum, currSum);
    }
    
    return maxSum;
}

/// Pattern 2: House Robber
pub fn houseRobber(houses: []const u32) u64 {
    const n = houses.len;
    if (n == 0) return 0;
    if (n == 1) return houses[0];
    
    var prev2: u64 = houses[0];
    var prev1: u64 = @max(houses[0], houses[1]);
    
    for (2..n) |i| {
        const curr = @max(prev1, prev2 + houses[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}

/// Pattern 3: Stock Profit (Buy and Sell Once)
pub fn maxProfit(prices: []const u32) u64 {
    if (prices.len < 2) return 0;
    
    var minPrice = prices[0];
    var maxPro: u64 = 0;
    
    for (prices[1..]) |price| {
        maxPro = @max(maxPro, if (price > minPrice) price - minPrice else 0);
        minPrice = @min(minPrice, price);
    }
    
    return maxPro;
}

/// Pattern 4: Unique Paths (2D DP)
pub fn uniquePaths(m: usize, n: usize, allocator: std.mem.Allocator) !u64 {
    const dp = try allocator.alloc([]u64, m);
    defer {
        for (dp) |row| allocator.free(row);
        allocator.free(dp);
    }
    for (dp) |*row| {
        row.* = try allocator.alloc(u64, n);
        @memset(row.*, 1); // เส้นแรกและคอลัมน์แรก = 1
    }
    
    for (1..m) |i| {
        for (1..n) |j| {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    
    return dp[m - 1][n - 1];
}

/// Pattern 5: Partition Equal Subset Sum
pub fn canPartition(nums: []const u32, allocator: std.mem.Allocator) !bool {
    var total: u32 = 0;
    for (nums) |n| total += n;
    
    if (total % 2 != 0) return false;
    
    const target = total / 2;
    const dp = try allocator.alloc(bool, target + 1);
    defer allocator.free(dp);
    @memset(dp, false);
    dp[0] = true;
    
    for (nums) |num| {
        var j: usize = target;
        while (j >= num) : (j -= 1) {
            dp[j] = dp[j] or dp[j - num];
        }
    }
    
    return dp[target];
}

/// Pattern 6: Word Break
pub fn wordBreak(s: []const u8, wordDict: []const []const u8, allocator: std.mem.Allocator) !bool {
    const n = s.len;
    const dp = try allocator.alloc(bool, n + 1);
    defer allocator.free(dp);
    @memset(dp, false);
    dp[0] = true;
    
    for (1..n + 1) |i| {
        for (wordDict) |word| {
            const wlen = word.len;
            if (wlen <= i and dp[i - wlen] and std.mem.eql(u8, s[i - wlen .. i], word)) {
                dp[i] = true;
                break;
            }
        }
    }
    
    return dp[n];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== DP Patterns สรุป ===\n\n", .{});
    
    // Max Subarray
    const arr = [_]i32{ -2, 1, -3, 4, -1, 2, 1, -5, 4 };
    try stdout.print("1. Max Subarray: {d} (answer: 6)\n", .{maxSubarray(&arr)});
    
    // House Robber
    const houses = [_]u32{ 2, 7, 9, 3, 1 };
    try stdout.print("2. House Robber: {d} (answer: 12)\n", .{houseRobber(&houses)});
    
    // Stock Profit
    const prices = [_]u32{ 7, 1, 5, 3, 6, 4 };
    try stdout.print("3. Max Stock Profit: {d} (answer: 5)\n", .{maxProfit(&prices)});
    
    // Unique Paths
    const paths = try uniquePaths(3, 7, allocator);
    try stdout.print("4. Unique Paths (3x7): {d} (answer: 28)\n", .{paths});
    
    // Partition Equal Subset
    const nums = [_]u32{ 1, 5, 11, 5 };
    const canPart = try canPartition(&nums, allocator);
    try stdout.print("5. Can Partition [1,5,11,5]: {} (answer: true)\n", .{canPart});
    
    // Word Break
    const words = [_][]const u8{ "leet", "code" };
    const canBreak = try wordBreak("leetcode", &words, allocator);
    try stdout.print("6. Word Break 'leetcode': {} (answer: true)\n", .{canBreak});
    
    // สรุป DP Patterns
    try stdout.print("\n=== สรุป DP Patterns ===\n", .{});
    try stdout.print("1. Linear DP: Fibonacci, Climbing Stairs, House Robber\n", .{});
    try stdout.print("2. Grid DP: Unique Paths, Minimum Path Sum\n", .{});
    try stdout.print("3. String DP: LCS, Edit Distance, Palindrome\n", .{});
    try stdout.print("4. Subset DP: Knapsack, Coin Change, Partition\n", .{});
    try stdout.print("5. Interval DP: Matrix Chain, Burst Balloons\n", .{});
    try stdout.print("6. Tree DP: House Robber III, Binary Tree problems\n", .{});
}
```

### สรุป Part 83

ใน Part นี้เราได้เรียนรู้:
1. แนวคิด DP และ Memoization vs Tabulation
2. Fibonacci ทุกรูปแบบรวม Matrix Exponentiation
3. LCS - หาลำดับย่อยร่วมที่ยาวที่สุด
4. LIS - หาลำดับที่เพิ่มขึ้นที่ยาวที่สุด
5. Knapsack Problem ทั้ง 0/1 และ Unbounded
6. Coin Change - แตกเงิน
7. Edit Distance - วัดความแตกต่างของ strings
8. Matrix Chain Multiplication
9. Palindrome DP
10. DP Patterns หลากหลายรูปแบบ

---

*ต่อไป: Part 84 - Algorithms: Graph Algorithms*
