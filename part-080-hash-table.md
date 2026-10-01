# ส่วนที่ 80: Data Structures: Hash Table (ขั้นตอนที่ 791-800)

Hash Table เป็น data structure ที่ให้ O(1) average time สำหรับ insert, search, delete โดยใช้ hash function แปลง key เป็น index

---

## ขั้นตอนที่ 791: Hash Table Implementation

การสร้าง Hash Table พื้นฐานที่มี dynamic resizing

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn HashMap(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const INITIAL_CAPACITY = 16;
        const LOAD_FACTOR = 0.75;

        const Entry = struct {
            key: K,
            value: V,
            hash: u64,
            occupied: bool = false,
            deleted: bool = false,
        };

        allocator: Allocator,
        entries: []Entry,
        size: usize = 0,
        capacity: usize,
        tombstones: usize = 0,

        pub fn init(allocator: Allocator) !Self {
            const entries = try allocator.alloc(Entry, INITIAL_CAPACITY);
            @memset(entries, Entry{ .key = undefined, .value = undefined, .hash = 0 });
            return Self{
                .allocator = allocator,
                .entries = entries,
                .capacity = INITIAL_CAPACITY,
            };
        }

        pub fn deinit(self: *Self) void {
            self.allocator.free(self.entries);
        }

        fn hashKey(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0);
            if (@typeInfo(K) == .Int) {
                hasher.update(std.mem.asBytes(&key));
            } else if (@typeInfo(K) == .Pointer) {
                hasher.update(key);
            } else {
                hasher.update(std.mem.asBytes(&key));
            }
            return hasher.final();
        }

        fn probe(self: *Self, hash: u64, key: K) usize {
            var idx = @as(usize, @intCast(hash % self.capacity));
            var i: usize = 0;
            while (self.entries[idx].occupied or self.entries[idx].deleted) {
                if (self.entries[idx].occupied and
                    self.entries[idx].hash == hash and
                    keysEqual(self.entries[idx].key, key))
                {
                    return idx;
                }
                i += 1;
                idx = (idx + 1) % self.capacity; // linear probing
                if (i >= self.capacity) break;
            }
            return idx;
        }

        fn keysEqual(a: K, b: K) bool {
            if (@typeInfo(K) == .Pointer) {
                return std.mem.eql(u8, a, b);
            }
            return a == b;
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            // Check load factor
            if (@as(f64, @floatFromInt(self.size + self.tombstones + 1)) >
                @as(f64, @floatFromInt(self.capacity)) * LOAD_FACTOR)
            {
                try self.resize(self.capacity * 2);
            }

            const hash = hashKey(key);
            var idx = @as(usize, @intCast(hash % self.capacity));
            var first_tombstone: ?usize = null;
            var i: usize = 0;

            while (i < self.capacity) : (i += 1) {
                const entry = &self.entries[idx];
                if (!entry.occupied and !entry.deleted) {
                    // Empty slot
                    const insert_idx = first_tombstone orelse idx;
                    self.entries[insert_idx] = Entry{
                        .key = key,
                        .value = value,
                        .hash = hash,
                        .occupied = true,
                    };
                    self.size += 1;
                    if (first_tombstone != null) self.tombstones -= 1;
                    return;
                }
                if (entry.deleted) {
                    if (first_tombstone == null) first_tombstone = idx;
                } else if (entry.hash == hash and keysEqual(entry.key, key)) {
                    entry.value = value; // Update
                    return;
                }
                idx = (idx + 1) % self.capacity;
            }

            // Use tombstone slot
            if (first_tombstone) |ts_idx| {
                self.entries[ts_idx] = Entry{
                    .key = key,
                    .value = value,
                    .hash = hash,
                    .occupied = true,
                };
                self.size += 1;
                self.tombstones -= 1;
            }
        }

        pub fn get(self: *Self, key: K) ?V {
            const hash = hashKey(key);
            var idx = @as(usize, @intCast(hash % self.capacity));
            var i: usize = 0;

            while (i < self.capacity) : (i += 1) {
                const entry = &self.entries[idx];
                if (!entry.occupied and !entry.deleted) return null;
                if (entry.occupied and entry.hash == hash and keysEqual(entry.key, key)) {
                    return entry.value;
                }
                idx = (idx + 1) % self.capacity;
            }
            return null;
        }

        pub fn delete(self: *Self, key: K) bool {
            const hash = hashKey(key);
            var idx = @as(usize, @intCast(hash % self.capacity));
            var i: usize = 0;

            while (i < self.capacity) : (i += 1) {
                const entry = &self.entries[idx];
                if (!entry.occupied and !entry.deleted) return false;
                if (entry.occupied and entry.hash == hash and keysEqual(entry.key, key)) {
                    entry.occupied = false;
                    entry.deleted = true;
                    self.size -= 1;
                    self.tombstones += 1;
                    return true;
                }
                idx = (idx + 1) % self.capacity;
            }
            return false;
        }

        fn resize(self: *Self, new_capacity: usize) !void {
            const old_entries = self.entries;
            const new_entries = try self.allocator.alloc(Entry, new_capacity);
            @memset(new_entries, Entry{ .key = undefined, .value = undefined, .hash = 0 });

            self.entries = new_entries;
            self.capacity = new_capacity;
            self.size = 0;
            self.tombstones = 0;

            for (old_entries) |entry| {
                if (entry.occupied) {
                    try self.put(entry.key, entry.value);
                }
            }
            self.allocator.free(old_entries);
        }

        pub fn loadFactor(self: *Self) f64 {
            return @as(f64, @floatFromInt(self.size)) / @as(f64, @floatFromInt(self.capacity));
        }

        pub fn printStats(self: *Self) void {
            std.debug.print("HashMap: size={d}, capacity={d}, load={d:.2}, tombstones={d}\n", .{
                self.size, self.capacity, self.loadFactor(), self.tombstones,
            });
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Hash Table Implementation ===\n", .{});

    var map = try HashMap(i32, []const u8).init(allocator);
    defer map.deinit();

    try map.put(1, "หนึ่ง");
    try map.put(2, "สอง");
    try map.put(3, "สาม");
    try map.put(4, "สี่");
    try map.put(5, "ห้า");

    map.printStats();

    std.debug.print("get(3) = {s}\n", .{map.get(3) orelse "null"});
    std.debug.print("get(10) = {s}\n", .{map.get(10) orelse "null"});

    _ = map.delete(3);
    std.debug.print("หลังลบ 3: get(3) = {s}\n", .{map.get(3) orelse "null"});

    // เพิ่มข้อมูลจนต้อง resize
    var i: i32 = 10;
    while (i < 30) : (i += 1) {
        try map.put(i, "ค่า");
    }
    map.printStats();
}
```

---

## ขั้นตอนที่ 792: Hash Functions

Hash functions ที่ดีต้องกระจาย keys อย่างสม่ำเสมอและรวดเร็ว

```zig
const std = @import("std");

// =========================================
// Various hash functions
// =========================================

// 1. FNV-1a (Fowler–Noll–Vo)
fn fnv1a(data: []const u8) u64 {
    const FNV_OFFSET_BASIS: u64 = 14695981039346656037;
    const FNV_PRIME: u64 = 1099511628211;
    var hash = FNV_OFFSET_BASIS;
    for (data) |byte| {
        hash ^= byte;
        hash = hash *% FNV_PRIME;
    }
    return hash;
}

// 2. djb2
fn djb2(data: []const u8) u64 {
    var hash: u64 = 5381;
    for (data) |byte| {
        hash = ((hash << 5) +% hash) +% byte;
    }
    return hash;
}

// 3. MurmurHash3 (simplified)
fn murmur3_32(data: []const u8, seed: u32) u32 {
    const c1: u32 = 0xcc9e2d51;
    const c2: u32 = 0x1b873593;
    var h: u32 = seed;
    const n = data.len / 4;

    var i: usize = 0;
    while (i < n) : (i += 1) {
        var k = std.mem.readInt(u32, data[i * 4 ..][0..4], .little);
        k = k *% c1;
        k = (k << 15) | (k >> 17);
        k = k *% c2;
        h ^= k;
        h = (h << 13) | (h >> 19);
        h = h *% 5 +% 0xe6546b64;
    }

    var tail: u32 = 0;
    const remainder = data.len & 3;
    const tail_data = data[n * 4 ..];
    if (remainder >= 3) tail ^= @as(u32, tail_data[2]) << 16;
    if (remainder >= 2) tail ^= @as(u32, tail_data[1]) << 8;
    if (remainder >= 1) {
        tail ^= tail_data[0];
        tail = tail *% c1;
        tail = (tail << 15) | (tail >> 17);
        tail = tail *% c2;
        h ^= tail;
    }

    h ^= @intCast(data.len);
    h ^= h >> 16;
    h = h *% 0x85ebca6b;
    h ^= h >> 13;
    h = h *% 0xc2b2ae35;
    h ^= h >> 16;
    return h;
}

// 4. xxHash (simplified 32-bit)
fn xxhash32(data: []const u8, seed: u32) u32 {
    const PRIME1: u32 = 0x9e3779b1;
    const PRIME2: u32 = 0x85ebca77;
    const PRIME3: u32 = 0xc2b2ae3d;
    const PRIME4: u32 = 0x27d4eb2f;
    const PRIME5: u32 = 0x165667b1;

    var h: u32 = seed +% PRIME5 +% @as(u32, @intCast(data.len));

    var i: usize = 0;
    while (i + 4 <= data.len) : (i += 4) {
        const k = std.mem.readInt(u32, data[i..][0..4], .little);
        h +%= k *% PRIME3;
        h = ((h << 17) | (h >> 15)) *% PRIME4;
    }

    while (i < data.len) : (i += 1) {
        h +%= @as(u32, data[i]) *% PRIME5;
        h = ((h << 11) | (h >> 21)) *% PRIME1;
    }

    h ^= h >> 15;
    h = h *% PRIME2;
    h ^= h >> 13;
    h = h *% PRIME3;
    h ^= h >> 16;
    return h;
}

// 5. Polynomial rolling hash (สำหรับ strings)
fn polynomialHash(data: []const u8, base: u64, mod: u64) u64 {
    var hash: u64 = 0;
    var power: u64 = 1;
    for (data) |c| {
        hash = (hash + @as(u64, c) * power) % mod;
        power = (power * base) % mod;
    }
    return hash;
}

// Hash distribution test
fn testDistribution(allocator: std.mem.Allocator, words: [][]const u8, num_buckets: usize) !void {
    var buckets = try allocator.alloc(usize, num_buckets);
    defer allocator.free(buckets);
    @memset(buckets, 0);

    for (words) |word| {
        const h = fnv1a(word) % num_buckets;
        buckets[h] += 1;
    }

    var max: usize = 0;
    var min: usize = std.math.maxInt(usize);
    var non_empty: usize = 0;

    for (buckets) |count| {
        if (count > max) max = count;
        if (count > 0) {
            if (count < min) min = count;
            non_empty += 1;
        }
    }

    std.debug.print("Distribution: buckets={d}, used={d}, min={d}, max={d}\n", .{
        num_buckets, non_empty, min, max,
    });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Hash Functions Demo ===\n", .{});

    const test_strings = [_][]const u8{
        "hello", "world", "zig", "programming", "hash", "table",
    };

    std.debug.print("\nHash comparison:\n", .{});
    std.debug.print("{s:<20} {s:>20} {s:>20} {s:>12}\n", .{
        "String", "FNV-1a", "djb2", "MurmurHash3",
    });
    for (test_strings) |s| {
        std.debug.print("{s:<20} {d:>20} {d:>20} {d:>12}\n", .{
            s, fnv1a(s), djb2(s), murmur3_32(s, 42),
        });
    }

    // Polynomial hash สำหรับ substring matching
    std.debug.print("\nPolynomial Hash:\n", .{});
    const text = "สวัสดีครับ";
    const h = polynomialHash(text, 31, 1_000_000_007);
    std.debug.print("Hash of '{s}': {d}\n", .{ text, h });

    // Distribution test
    std.debug.print("\nDistribution Test (16 buckets):\n", .{});
    var words = [_][]const u8{ "a", "b", "c", "d", "e", "f", "g", "h" };
    try testDistribution(allocator, &words, 16);
}
```

---

## ขั้นตอนที่ 793: Collision Handling (Chaining)

Chaining ใช้ linked list เก็บ elements ที่ hash ชนกัน

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn ChainHashMap(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();

        const Entry = struct {
            key: K,
            value: V,
            next: ?*Entry = null,
        };

        allocator: Allocator,
        buckets: []?*Entry,
        size: usize = 0,
        capacity: usize,

        pub fn init(allocator: Allocator, capacity: usize) !Self {
            const buckets = try allocator.alloc(?*Entry, capacity);
            @memset(buckets, null);
            return Self{
                .allocator = allocator,
                .buckets = buckets,
                .capacity = capacity,
            };
        }

        pub fn deinit(self: *Self) void {
            for (self.buckets) |bucket| {
                var current = bucket;
                while (current) |entry| {
                    const next = entry.next;
                    self.allocator.destroy(entry);
                    current = next;
                }
            }
            self.allocator.free(self.buckets);
        }

        fn hashKey(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0);
            hasher.update(std.mem.asBytes(&key));
            return hasher.final();
        }

        fn keysEqual(a: K, b: K) bool {
            return a == b;
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            const idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var current = self.buckets[idx];

            // ค้นหา key ที่มีอยู่แล้ว
            while (current) |entry| {
                if (keysEqual(entry.key, key)) {
                    entry.value = value;
                    return;
                }
                current = entry.next;
            }

            // เพิ่ม entry ใหม่ที่หัว chain
            const entry = try self.allocator.create(Entry);
            entry.* = Entry{
                .key = key,
                .value = value,
                .next = self.buckets[idx],
            };
            self.buckets[idx] = entry;
            self.size += 1;
        }

        pub fn get(self: *Self, key: K) ?V {
            const idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var current = self.buckets[idx];
            while (current) |entry| {
                if (keysEqual(entry.key, key)) return entry.value;
                current = entry.next;
            }
            return null;
        }

        pub fn delete(self: *Self, key: K) bool {
            const idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var current = &self.buckets[idx];
            while (current.*) |entry| {
                if (keysEqual(entry.key, key)) {
                    current.* = entry.next;
                    self.allocator.destroy(entry);
                    self.size -= 1;
                    return true;
                }
                current = &entry.next;
            }
            return false;
        }

        // วิเคราะห์การกระจาย
        pub fn analyzeChains(self: *Self) void {
            var max_chain: usize = 0;
            var total_chain: usize = 0;
            var non_empty: usize = 0;

            for (self.buckets) |bucket| {
                var chain_len: usize = 0;
                var current = bucket;
                while (current) |entry| {
                    chain_len += 1;
                    current = entry.next;
                }
                if (chain_len > 0) {
                    non_empty += 1;
                    total_chain += chain_len;
                    if (chain_len > max_chain) max_chain = chain_len;
                }
            }

            std.debug.print("Chain Analysis: buckets={d}, used={d}/{d}, max_chain={d}, avg_chain={d:.2}\n", .{
                self.capacity,
                non_empty,
                self.capacity,
                max_chain,
                if (non_empty > 0) @as(f64, @floatFromInt(total_chain)) / @as(f64, @floatFromInt(non_empty)) else 0.0,
            });
        }
    };
}

// Word frequency counter ใช้ chaining
fn wordFrequency(allocator: Allocator, text: []const u8) !ChainHashMap(u64, u32) {
    var freq = try ChainHashMap(u64, u32).init(allocator, 64);
    var iter = std.mem.tokenizeAny(u8, text, " \t\n.,!?");

    while (iter.next()) |word| {
        var hasher = std.hash.Wyhash.init(0);
        hasher.update(word);
        const hash = hasher.final();
        const count = freq.get(hash) orelse 0;
        try freq.put(hash, count + 1);
    }
    return freq;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Chaining Hash Map Demo ===\n", .{});

    var map = try ChainHashMap(i32, []const u8).init(allocator, 8);
    defer map.deinit();

    // ใส่ข้อมูลที่อาจ collide (ใช้ capacity เล็ก)
    var i: i32 = 0;
    while (i < 20) : (i += 1) {
        var buf: [10]u8 = undefined;
        const s = try std.fmt.bufPrint(&buf, "value{d}", .{i});
        _ = s; // Just testing put
        try map.put(i, "test");
    }

    map.analyzeChains();
    std.debug.print("Size: {d}\n", .{map.size});
    std.debug.print("get(5): {s}\n", .{map.get(5) orelse "null"});
    _ = map.delete(5);
    std.debug.print("หลังลบ 5: get(5): {s}\n", .{map.get(5) orelse "null"});
}
```

---

## ขั้นตอนที่ 794: Open Addressing (Linear Probing)

Open Addressing เก็บ elements ไว้ใน array เดียวกัน โดยหาตำแหน่งว่างเมื่อเกิด collision

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Open Addressing with different probing strategies
fn OpenAddressHashMap(comptime K: type, comptime V: type, comptime ProbeStrategy: type) type {
    return struct {
        const Self = @This();
        const State = enum { empty, occupied, deleted };

        const Slot = struct {
            key: K = undefined,
            value: V = undefined,
            state: State = .empty,
            hash: u64 = 0,
        };

        allocator: Allocator,
        slots: []Slot,
        size: usize = 0,
        capacity: usize,
        probe: ProbeStrategy,

        pub fn init(allocator: Allocator, capacity: usize) !Self {
            const slots = try allocator.alloc(Slot, capacity);
            @memset(slots, Slot{});
            return Self{
                .allocator = allocator,
                .slots = slots,
                .capacity = capacity,
                .probe = ProbeStrategy{},
            };
        }

        pub fn deinit(self: *Self) void {
            self.allocator.free(self.slots);
        }

        fn hashKey(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0x1234);
            hasher.update(std.mem.asBytes(&key));
            return hasher.final();
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            const h = hashKey(key);
            var first_deleted: ?usize = null;
            var i: usize = 0;

            while (i < self.capacity) : (i += 1) {
                const idx = self.probe.next(@as(usize, @intCast(h % self.capacity)), i, self.capacity);
                switch (self.slots[idx].state) {
                    .empty => {
                        const insert_idx = first_deleted orelse idx;
                        self.slots[insert_idx] = .{
                            .key = key,
                            .value = value,
                            .state = .occupied,
                            .hash = h,
                        };
                        self.size += 1;
                        return;
                    },
                    .deleted => {
                        if (first_deleted == null) first_deleted = idx;
                    },
                    .occupied => {
                        if (self.slots[idx].hash == h and self.slots[idx].key == key) {
                            self.slots[idx].value = value;
                            return;
                        }
                    },
                }
            }
            return error.HashTableFull;
        }

        pub fn get(self: *Self, key: K) ?V {
            const h = hashKey(key);
            var i: usize = 0;
            while (i < self.capacity) : (i += 1) {
                const idx = self.probe.next(@as(usize, @intCast(h % self.capacity)), i, self.capacity);
                switch (self.slots[idx].state) {
                    .empty => return null,
                    .occupied => {
                        if (self.slots[idx].hash == h and self.slots[idx].key == key) {
                            return self.slots[idx].value;
                        }
                    },
                    .deleted => {},
                }
            }
            return null;
        }

        pub fn printSlots(self: *Self) void {
            std.debug.print("Slots: ", .{});
            for (self.slots) |slot| {
                switch (slot.state) {
                    .empty => std.debug.print("[ ] ", .{}),
                    .deleted => std.debug.print("[X] ", .{}),
                    .occupied => std.debug.print("[{d}] ", .{slot.key}),
                }
            }
            std.debug.print("\n", .{});
        }
    };
}

// Linear Probing: h(k, i) = (h(k) + i) % m
const LinearProbe = struct {
    pub fn next(_: LinearProbe, start: usize, i: usize, capacity: usize) usize {
        return (start + i) % capacity;
    }
};

// Quadratic Probing: h(k, i) = (h(k) + i^2) % m
const QuadraticProbe = struct {
    pub fn next(_: QuadraticProbe, start: usize, i: usize, capacity: usize) usize {
        return (start + i * i) % capacity;
    }
};

// Double Hashing: h(k, i) = (h1(k) + i*h2(k)) % m
const DoubleHashProbe = struct {
    pub fn next(_: DoubleHashProbe, start: usize, i: usize, capacity: usize) usize {
        // h2 = 1 + (start % (capacity - 1))
        const h2 = 1 + (start % (capacity - 1));
        return (start + i * h2) % capacity;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Open Addressing Demo ===\n", .{});

    const keys = [_]i32{ 5, 10, 15, 20, 25, 30 }; // หลายค่าที่ hash ใกล้กัน

    // Linear Probing
    std.debug.print("\nLinear Probing:\n", .{});
    var linear = try OpenAddressHashMap(i32, i32, LinearProbe).init(allocator, 11);
    defer linear.deinit();
    for (keys) |k| try linear.put(k, k * 10);
    linear.printSlots();

    // Quadratic Probing
    std.debug.print("\nQuadratic Probing:\n", .{});
    var quad = try OpenAddressHashMap(i32, i32, QuadraticProbe).init(allocator, 11);
    defer quad.deinit();
    for (keys) |k| try quad.put(k, k * 10);
    quad.printSlots();

    // Double Hashing
    std.debug.print("\nDouble Hashing:\n", .{});
    var dh = try OpenAddressHashMap(i32, i32, DoubleHashProbe).init(allocator, 11);
    defer dh.deinit();
    for (keys) |k| try dh.put(k, k * 10);
    dh.printSlots();

    std.debug.print("\nget(15) = {?d}\n", .{linear.get(15)});
}
```

---

## ขั้นตอนที่ 795: Robin Hood Hashing

Robin Hood Hashing ลด variance ของ probe sequence โดย "ขโมย" จากคนรวยให้คนจน

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn RobinHoodHashMap(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const EMPTY: u8 = 255;

        const Entry = struct {
            key: K,
            value: V,
            psl: u8 = 0, // Probe Sequence Length
            occupied: bool = false,
        };

        allocator: Allocator,
        entries: []Entry,
        capacity: usize,
        size: usize = 0,

        pub fn init(allocator: Allocator, capacity: usize) !Self {
            const entries = try allocator.alloc(Entry, capacity);
            for (entries) |*e| e.occupied = false;
            return Self{
                .allocator = allocator,
                .entries = entries,
                .capacity = capacity,
            };
        }

        pub fn deinit(self: *Self) void {
            self.allocator.free(self.entries);
        }

        fn hashKey(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0xDEAD);
            hasher.update(std.mem.asBytes(&key));
            return hasher.final();
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            var idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var entry = Entry{ .key = key, .value = value, .psl = 0, .occupied = true };

            var i: usize = 0;
            while (i < self.capacity) : (i += 1) {
                const slot = &self.entries[idx];

                if (!slot.occupied) {
                    // Empty slot - insert here
                    slot.* = entry;
                    self.size += 1;
                    return;
                }

                if (slot.key == entry.key) {
                    // Update existing
                    slot.value = entry.value;
                    return;
                }

                // Robin Hood: ถ้า slot มี PSL น้อยกว่า entry ที่กำลัง insert
                // ให้ "ขโมย" slot นั้น
                if (slot.psl < entry.psl) {
                    // Swap
                    const temp = slot.*;
                    slot.* = entry;
                    entry = temp;
                }

                entry.psl += 1;
                idx = (idx + 1) % self.capacity;
            }
            return error.HashTableFull;
        }

        pub fn get(self: *Self, key: K) ?V {
            var idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var psl: u8 = 0;

            while (idx < self.capacity) {
                const slot = &self.entries[idx];

                if (!slot.occupied) return null;
                if (slot.psl < psl) return null; // PSL ของ slot น้อยกว่า probe ปัจจุบัน = ไม่มี

                if (slot.key == key) return slot.value;

                psl += 1;
                idx = (idx + 1) % self.capacity;
            }
            return null;
        }

        pub fn delete(self: *Self, key: K) bool {
            var idx = @as(usize, @intCast(hashKey(key) % self.capacity));
            var psl: u8 = 0;

            // หา entry
            while (idx < self.capacity) {
                const slot = &self.entries[idx];
                if (!slot.occupied or slot.psl < psl) return false;
                if (slot.key == key) break;
                psl += 1;
                idx = (idx + 1) % self.capacity;
            }
            if (!self.entries[idx].occupied) return false;

            // Backward shift deletion
            var current = idx;
            while (true) {
                const next = (current + 1) % self.capacity;
                const next_slot = &self.entries[next];

                if (!next_slot.occupied or next_slot.psl == 0) {
                    self.entries[current].occupied = false;
                    break;
                }

                self.entries[current] = next_slot.*;
                self.entries[current].psl -= 1;
                current = next;
            }

            self.size -= 1;
            return true;
        }

        pub fn printPSL(self: *Self) void {
            std.debug.print("PSL distribution:\n", .{});
            for (self.entries, 0..) |entry, i| {
                if (entry.occupied) {
                    std.debug.print("  slot {d}: key={d}, psl={d}\n", .{ i, entry.key, entry.psl });
                }
            }
        }

        pub fn averagePSL(self: *Self) f64 {
            if (self.size == 0) return 0;
            var total: usize = 0;
            for (self.entries) |e| {
                if (e.occupied) total += e.psl;
            }
            return @as(f64, @floatFromInt(total)) / @as(f64, @floatFromInt(self.size));
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Robin Hood Hashing Demo ===\n", .{});

    var rh = try RobinHoodHashMap(i32, i32).init(allocator, 16);
    defer rh.deinit();

    var i: i32 = 0;
    while (i < 10) : (i += 1) {
        try rh.put(i * 7 + 3, i * 100);
    }

    rh.printPSL();
    std.debug.print("Average PSL: {d:.2}\n", .{rh.averagePSL()});
    std.debug.print("Size: {d}\n", .{rh.size});
}
```

---

## ขั้นตอนที่ 796: Cuckoo Hashing

Cuckoo Hashing รับประกัน O(1) worst-case lookup โดยใช้ 2 hash tables

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn CuckooHashMap(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const MAX_EVICTIONS = 100;

        const Slot = struct {
            key: K,
            value: V,
            occupied: bool = false,
        };

        allocator: Allocator,
        table1: []Slot,
        table2: []Slot,
        capacity: usize,
        size: usize = 0,

        pub fn init(allocator: Allocator, capacity: usize) !Self {
            const t1 = try allocator.alloc(Slot, capacity);
            const t2 = try allocator.alloc(Slot, capacity);
            for (t1) |*s| s.occupied = false;
            for (t2) |*s| s.occupied = false;
            return Self{
                .allocator = allocator,
                .table1 = t1,
                .table2 = t2,
                .capacity = capacity,
            };
        }

        pub fn deinit(self: *Self) void {
            self.allocator.free(self.table1);
            self.allocator.free(self.table2);
        }

        fn hash1(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0x1111);
            hasher.update(std.mem.asBytes(&key));
            return hasher.final();
        }

        fn hash2(key: K) u64 {
            var hasher = std.hash.Wyhash.init(0x2222);
            hasher.update(std.mem.asBytes(&key));
            return hasher.final();
        }

        pub fn get(self: *Self, key: K) ?V {
            const idx1 = @as(usize, @intCast(hash1(key) % self.capacity));
            if (self.table1[idx1].occupied and self.table1[idx1].key == key) {
                return self.table1[idx1].value;
            }
            const idx2 = @as(usize, @intCast(hash2(key) % self.capacity));
            if (self.table2[idx2].occupied and self.table2[idx2].key == key) {
                return self.table2[idx2].value;
            }
            return null;
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            // Check if already exists
            const idx1 = @as(usize, @intCast(hash1(key) % self.capacity));
            if (self.table1[idx1].occupied and self.table1[idx1].key == key) {
                self.table1[idx1].value = value;
                return;
            }
            const idx2 = @as(usize, @intCast(hash2(key) % self.capacity));
            if (self.table2[idx2].occupied and self.table2[idx2].key == key) {
                self.table2[idx2].value = value;
                return;
            }

            var cur_key = key;
            var cur_val = value;
            var evictions: usize = 0;

            while (evictions < MAX_EVICTIONS) : (evictions += 1) {
                const i1 = @as(usize, @intCast(hash1(cur_key) % self.capacity));

                if (!self.table1[i1].occupied) {
                    self.table1[i1] = .{ .key = cur_key, .value = cur_val, .occupied = true };
                    self.size += 1;
                    return;
                }

                // Evict current occupant
                const evicted_key = self.table1[i1].key;
                const evicted_val = self.table1[i1].value;
                self.table1[i1] = .{ .key = cur_key, .value = cur_val, .occupied = true };
                cur_key = evicted_key;
                cur_val = evicted_val;

                const i2 = @as(usize, @intCast(hash2(cur_key) % self.capacity));

                if (!self.table2[i2].occupied) {
                    self.table2[i2] = .{ .key = cur_key, .value = cur_val, .occupied = true };
                    self.size += 1;
                    return;
                }

                // Evict from table2
                const evicted_key2 = self.table2[i2].key;
                const evicted_val2 = self.table2[i2].value;
                self.table2[i2] = .{ .key = cur_key, .value = cur_val, .occupied = true };
                cur_key = evicted_key2;
                cur_val = evicted_val2;
            }

            return error.CyclicEviction; // ต้อง rehash
        }

        pub fn printStats(self: *Self) void {
            var t1_used: usize = 0;
            var t2_used: usize = 0;
            for (self.table1) |s| if (s.occupied) { t1_used += 1; };
            for (self.table2) |s| if (s.occupied) { t2_used += 1; };
            std.debug.print("Cuckoo: size={d}, table1={d}/{d}, table2={d}/{d}\n", .{
                self.size, t1_used, self.capacity, t2_used, self.capacity,
            });
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Cuckoo Hashing Demo ===\n", .{});

    var cuckoo = try CuckooHashMap(i32, i32).init(allocator, 16);
    defer cuckoo.deinit();

    var i: i32 = 0;
    while (i < 10) : (i += 1) {
        try cuckoo.put(i, i * i);
    }

    cuckoo.printStats();

    std.debug.print("get(5) = {?d}\n", .{cuckoo.get(5)});
    std.debug.print("get(7) = {?d}\n", .{cuckoo.get(7)});
    std.debug.print("get(15) = {?d}\n", .{cuckoo.get(15)});
}
```

---

## ขั้นตอนที่ 797: Bloom Filter

Bloom Filter เป็น probabilistic data structure ที่บอกว่า element อยู่ใน set หรือไม่ (อาจ false positive แต่ไม่มี false negative)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const BloomFilter = struct {
    allocator: Allocator,
    bits: []u64,
    bit_count: usize,
    hash_count: usize,
    items_added: usize = 0,

    pub fn init(allocator: Allocator, expected_items: usize, false_positive_rate: f64) !BloomFilter {
        // คำนวณขนาด bit array และจำนวน hash functions
        const ln2 = @log(2.0);
        const ln_fpr = -@log(false_positive_rate);
        const bit_count: usize = @intFromFloat(
            @as(f64, @floatFromInt(expected_items)) * ln_fpr / (ln2 * ln2)
        );
        const hash_count: usize = @intFromFloat(
            @as(f64, @floatFromInt(bit_count)) / @as(f64, @floatFromInt(expected_items)) * ln2
        );

        const num_words = (bit_count + 63) / 64;
        const bits = try allocator.alloc(u64, num_words);
        @memset(bits, 0);

        std.debug.print("Bloom Filter: bits={d}, hashes={d}\n", .{ bit_count, hash_count });

        return BloomFilter{
            .allocator = allocator,
            .bits = bits,
            .bit_count = bit_count,
            .hash_count = hash_count,
        };
    }

    pub fn deinit(self: *BloomFilter) void {
        self.allocator.free(self.bits);
    }

    fn getHashes(self: *BloomFilter, key: []const u8) [8]usize {
        var hashes: [8]usize = undefined;
        var hasher1 = std.hash.Wyhash.init(0xAAAA);
        hasher1.update(key);
        const h1 = hasher1.final();

        var hasher2 = std.hash.Wyhash.init(0xBBBB);
        hasher2.update(key);
        const h2 = hasher2.final();

        for (0..@min(8, self.hash_count)) |i| {
            hashes[i] = (h1 +% @as(u64, @intCast(i)) *% h2) % self.bit_count;
        }
        return hashes;
    }

    pub fn add(self: *BloomFilter, key: []const u8) void {
        const hashes = self.getHashes(key);
        for (0..self.hash_count) |i| {
            const bit = hashes[i];
            self.bits[bit / 64] |= @as(u64, 1) << @intCast(bit % 64);
        }
        self.items_added += 1;
    }

    pub fn contains(self: *BloomFilter, key: []const u8) bool {
        const hashes = self.getHashes(key);
        for (0..self.hash_count) |i| {
            const bit = hashes[i];
            if (self.bits[bit / 64] & (@as(u64, 1) << @intCast(bit % 64)) == 0) {
                return false;
            }
        }
        return true; // อาจ false positive
    }

    pub fn estimateFPR(self: *BloomFilter) f64 {
        const k = @as(f64, @floatFromInt(self.hash_count));
        const m = @as(f64, @floatFromInt(self.bit_count));
        const n = @as(f64, @floatFromInt(self.items_added));
        return std.math.pow(f64, 1.0 - @exp(-k * n / m), k);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Bloom Filter Demo ===\n", .{});

    var bf = try BloomFilter.init(allocator, 1000, 0.01); // 1000 items, 1% FPR
    defer bf.deinit();

    // เพิ่ม words
    const words = [_][]const u8{
        "apple", "banana", "cherry", "date", "elderberry",
        "fig", "grape", "honeydew", "kiwi", "lemon",
    };

    for (words) |word| bf.add(word);

    std.debug.print("\nContains check:\n", .{});
    for (words) |word| {
        std.debug.print("  '{s}': {} (ควรเป็น true)\n", .{ word, bf.contains(word) });
    }

    const not_in = [_][]const u8{ "mango", "orange", "papaya", "quince" };
    std.debug.print("\nItems not added:\n", .{});
    for (not_in) |word| {
        std.debug.print("  '{s}': {} (อาจ false positive)\n", .{ word, bf.contains(word) });
    }

    std.debug.print("\nEstimated FPR: {d:.4}\n", .{bf.estimateFPR()});

    // Use case: Web crawler URL deduplication
    std.debug.print("\n--- URL Deduplication ---\n", .{});
    var url_filter = try BloomFilter.init(allocator, 10000, 0.001);
    defer url_filter.deinit();

    const urls = [_][]const u8{
        "https://example.com/page1",
        "https://example.com/page2",
        "https://example.com/page1", // ซ้ำ
        "https://example.com/page3",
    };

    for (urls) |url| {
        if (!url_filter.contains(url)) {
            url_filter.add(url);
            std.debug.print("Crawl: {s}\n", .{url});
        } else {
            std.debug.print("Skip (ซ้ำ): {s}\n", .{url});
        }
    }
}
```

---

## ขั้นตอนที่ 798: Count-Min Sketch

Count-Min Sketch เป็น probabilistic data structure สำหรับ frequency estimation

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const CountMinSketch = struct {
    allocator: Allocator,
    table: [][]u32,
    depth: usize,  // จำนวน hash functions
    width: usize,  // ขนาดของแต่ละ row
    seeds: []u64,

    pub fn init(allocator: Allocator, epsilon: f64, delta: f64) !CountMinSketch {
        const e = @exp(1.0);
        const width: usize = @intFromFloat(@ceil(e / epsilon));
        const depth: usize = @intFromFloat(@ceil(@log(1.0 / delta)));

        const table = try allocator.alloc([]u32, depth);
        for (table) |*row| {
            row.* = try allocator.alloc(u32, width);
            @memset(row.*, 0);
        }

        const seeds = try allocator.alloc(u64, depth);
        for (seeds, 0..) |*seed, i| seed.* = @intCast(i * 0x1234 + 0xABCD);

        std.debug.print("CMS: depth={d}, width={d}\n", .{ depth, width });

        return CountMinSketch{
            .allocator = allocator,
            .table = table,
            .depth = depth,
            .width = width,
            .seeds = seeds,
        };
    }

    pub fn deinit(self: *CountMinSketch) void {
        for (self.table) |row| self.allocator.free(row);
        self.allocator.free(self.table);
        self.allocator.free(self.seeds);
    }

    fn getHash(self: *CountMinSketch, key: []const u8, row: usize) usize {
        var hasher = std.hash.Wyhash.init(self.seeds[row]);
        hasher.update(key);
        return @as(usize, @intCast(hasher.final() % self.width));
    }

    pub fn add(self: *CountMinSketch, key: []const u8, count: u32) void {
        for (0..self.depth) |i| {
            const col = self.getHash(key, i);
            self.table[i][col] +|= count;
        }
    }

    pub fn estimate(self: *CountMinSketch, key: []const u8) u32 {
        var min: u32 = std.math.maxInt(u32);
        for (0..self.depth) |i| {
            const col = self.getHash(key, i);
            if (self.table[i][col] < min) min = self.table[i][col];
        }
        return min;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Count-Min Sketch Demo ===\n", .{});

    // epsilon = 0.1 (10% error), delta = 0.05 (95% confidence)
    var cms = try CountMinSketch.init(allocator, 0.1, 0.05);
    defer cms.deinit();

    // จำลอง click stream
    const events = [_][]const u8{
        "pageA", "pageB", "pageA", "pageC", "pageA",
        "pageB", "pageA", "pageD", "pageB", "pageA",
        "pageA", "pageC", "pageB", "pageA", "pageE",
    };

    for (events) |event| cms.add(event, 1);

    const pages = [_][]const u8{ "pageA", "pageB", "pageC", "pageD", "pageE" };
    const actual = [_]u32{ 8, 4, 2, 1, 1 };

    std.debug.print("\nFrequency estimates:\n", .{});
    for (pages, actual) |page, act| {
        std.debug.print("  {s}: estimated={d}, actual={d}\n", .{
            page, cms.estimate(page), act,
        });
    }
}
```

---

## ขั้นตอนที่ 799: Consistent Hashing

Consistent Hashing ใช้สำหรับ distributed systems เพื่อกระจาย load อย่าง consistent

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const ConsistentHash = struct {
    const VirtualNode = struct {
        hash: u64,
        server: []const u8,
    };

    allocator: Allocator,
    ring: std.ArrayList(VirtualNode),
    replicas: usize,

    pub fn init(allocator: Allocator, replicas: usize) ConsistentHash {
        return .{
            .allocator = allocator,
            .ring = std.ArrayList(VirtualNode).init(allocator),
            .replicas = replicas,
        };
    }

    pub fn deinit(self: *ConsistentHash) void {
        self.ring.deinit();
    }

    fn hashString(s: []const u8) u64 {
        var hasher = std.hash.Wyhash.init(0);
        hasher.update(s);
        return hasher.final();
    }

    pub fn addServer(self: *ConsistentHash, server: []const u8) !void {
        var i: usize = 0;
        while (i < self.replicas) : (i += 1) {
            var buf: [256]u8 = undefined;
            const key = try std.fmt.bufPrint(&buf, "{s}#{d}", .{ server, i });
            try self.ring.append(.{
                .hash = hashString(key),
                .server = server,
            });
        }

        // Sort ring by hash
        std.mem.sort(VirtualNode, self.ring.items, {}, struct {
            fn lessThan(_: void, a: VirtualNode, b: VirtualNode) bool {
                return a.hash < b.hash;
            }
        }.lessThan);
    }

    pub fn removeServer(self: *ConsistentHash, server: []const u8) void {
        var i: usize = 0;
        while (i < self.ring.items.len) {
            if (std.mem.eql(u8, self.ring.items[i].server, server)) {
                _ = self.ring.orderedRemove(i);
            } else {
                i += 1;
            }
        }
    }

    pub fn getServer(self: *ConsistentHash, key: []const u8) ?[]const u8 {
        if (self.ring.items.len == 0) return null;
        const h = hashString(key);

        // Binary search for first node >= h
        var lo: usize = 0;
        var hi = self.ring.items.len;
        while (lo < hi) {
            const mid = lo + (hi - lo) / 2;
            if (self.ring.items[mid].hash < h) {
                lo = mid + 1;
            } else {
                hi = mid;
            }
        }

        // Wrap around
        const idx = lo % self.ring.items.len;
        return self.ring.items[idx].server;
    }

    pub fn distribution(self: *ConsistentHash, allocator: Allocator, keys: [][]const u8) !std.StringHashMap(usize) {
        var dist = std.StringHashMap(usize).init(allocator);
        for (keys) |key| {
            if (self.getServer(key)) |server| {
                const count = dist.get(server) orelse 0;
                try dist.put(server, count + 1);
            }
        }
        return dist;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Consistent Hashing Demo ===\n", .{});

    var ring = ConsistentHash.init(allocator, 150); // 150 virtual nodes per server
    defer ring.deinit();

    const servers = [_][]const u8{ "server-1", "server-2", "server-3" };
    for (servers) |s| try ring.addServer(s);

    std.debug.print("Ring size: {d} virtual nodes\n", .{ring.ring.items.len});

    // Test distribution
    var keys: [100][]u8 = undefined;
    for (&keys, 0..) |*k, i| {
        k.* = try std.fmt.allocPrint(allocator, "user:{d}", .{i});
    }
    defer for (keys) |k| allocator.free(k);

    var key_slices: [100][]const u8 = undefined;
    for (&key_slices, 0..) |*ks, i| ks.* = keys[i];

    var dist = try ring.distribution(allocator, &key_slices);
    defer dist.deinit();

    std.debug.print("\nDistribution (100 keys, 3 servers):\n", .{});
    for (servers) |s| {
        const count = dist.get(s) orelse 0;
        std.debug.print("  {s}: {d} keys ({d:.1}%)\n", .{ s, count, @as(f64, @floatFromInt(count)) });
    }

    // Add new server
    std.debug.print("\nเพิ่ม server-4:\n", .{});
    try ring.addServer("server-4");

    var dist2 = try ring.distribution(allocator, &key_slices);
    defer dist2.deinit();

    const all_servers = [_][]const u8{ "server-1", "server-2", "server-3", "server-4" };
    for (all_servers) |s| {
        const count = dist2.get(s) orelse 0;
        std.debug.print("  {s}: {d} keys\n", .{ s, count });
    }
}
```

---

## ขั้นตอนที่ 800: Hash Table Performance Analysis

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const BenchmarkResult = struct {
    name: []const u8,
    insert_ns: u64,
    lookup_ns: u64,
    delete_ns: u64,
    memory_bytes: usize,
    collisions: usize,
};

fn benchmark(allocator: Allocator, name: []const u8, n: usize) !BenchmarkResult {
    var map = std.AutoHashMap(u64, u64).init(allocator);
    defer map.deinit();

    // Pre-generate keys
    var keys = try allocator.alloc(u64, n);
    defer allocator.free(keys);

    var prng = std.rand.DefaultPrng.init(42);
    for (keys) |*k| k.* = prng.random().int(u64);

    // Benchmark insert
    const t0 = std.time.nanoTimestamp();
    for (keys) |k| try map.put(k, k * 2);
    const t1 = std.time.nanoTimestamp();

    // Benchmark lookup
    var found: usize = 0;
    for (keys) |k| {
        if (map.get(k) != null) found += 1;
    }
    const t2 = std.time.nanoTimestamp();

    // Benchmark delete
    for (keys[0..n / 2]) |k| _ = map.remove(k);
    const t3 = std.time.nanoTimestamp();

    _ = found;
    _ = name;

    return BenchmarkResult{
        .name = "std.AutoHashMap",
        .insert_ns = @intCast(@divTrunc(t1 - t0, @as(i128, @intCast(n)))),
        .lookup_ns = @intCast(@divTrunc(t2 - t1, @as(i128, @intCast(n)))),
        .delete_ns = @intCast(@divTrunc(t3 - t2, @as(i128, @intCast(n / 2)))),
        .memory_bytes = map.capacity() * @sizeOf(u64) * 2,
        .collisions = 0,
    };
}

// Analysis ของ load factor vs performance
fn loadFactorAnalysis(allocator: Allocator) !void {
    std.debug.print("\n=== Load Factor Analysis ===\n", .{});
    std.debug.print("{s:<15} {s:>10} {s:>10} {s:>12}\n", .{
        "Load Factor", "Size", "Capacity", "Miss Rate",
    });

    const load_factors = [_]f64{ 0.25, 0.50, 0.75, 0.875, 0.95 };

    for (load_factors) |lf| {
        const capacity: usize = 1024;
        const n: usize = @intFromFloat(@as(f64, @floatFromInt(capacity)) * lf);

        var map = std.AutoHashMap(u64, u64).init(allocator);
        defer map.deinit();

        var prng = std.rand.DefaultPrng.init(123);
        for (0..n) |_| {
            const k = prng.random().int(u64);
            try map.put(k, k);
        }

        // Estimate miss rate (คีย์ที่ไม่มีใน map)
        var misses: usize = 0;
        for (0..1000) |_| {
            const k = prng.random().int(u64);
            if (map.get(k) == null) misses += 1;
        }

        std.debug.print("{d:<15.2} {d:>10} {d:>10} {d:>12.1}%\n", .{
            lf, map.count(), map.capacity(), @as(f64, @floatFromInt(misses)) / 10.0,
        });
    }
}

// Hash collision analysis
fn collisionAnalysis() void {
    std.debug.print("\n=== Collision Analysis ===\n", .{});

    // Birthday paradox: ใน n slots ใส่ m items, P(collision) = 1 - n!/(n^m * (n-m)!)
    const n: f64 = 1024.0;
    std.debug.print("{s:<10} {s:>20} {s:>20}\n", .{
        "Items", "Expected Collisions", "Load Factor",
    });

    var m: f64 = 64.0;
    while (m <= n) : (m *= 2.0) {
        // Expected collisions = m - n * (1 - (1-1/n)^m)
        const p_no_collision = std.math.pow(f64, 1.0 - 1.0 / n, m);
        const expected = m - n * (1.0 - p_no_collision);
        std.debug.print("{d:<10.0} {d:>20.2} {d:>20.2}%\n", .{
            m, expected, m / n * 100.0,
        });
    }
}

// Real-world comparison
fn compareImplementations(allocator: Allocator) !void {
    std.debug.print("\n=== Implementation Comparison ===\n", .{});
    std.debug.print("{s:<20} {s:>12} {s:>12} {s:>12}\n", .{
        "Implementation", "Insert (ns)", "Lookup (ns)", "Delete (ns)",
    });

    const N = 100_000;

    // Zig std.AutoHashMap
    {
        var map = std.AutoHashMap(u32, u32).init(allocator);
        defer map.deinit();
        try map.ensureTotalCapacity(N);

        const t0 = std.time.nanoTimestamp();
        for (0..N) |i| try map.put(@intCast(i), @intCast(i));
        const t1 = std.time.nanoTimestamp();

        for (0..N) |i| _ = map.get(@intCast(i));
        const t2 = std.time.nanoTimestamp();

        for (0..N) |i| _ = map.remove(@intCast(i));
        const t3 = std.time.nanoTimestamp();

        std.debug.print("{s:<20} {d:>12} {d:>12} {d:>12}\n", .{
            "AutoHashMap",
            @divTrunc(t1 - t0, N),
            @divTrunc(t2 - t1, N),
            @divTrunc(t3 - t2, N),
        });
    }

    // StringHashMap
    {
        var map = std.StringHashMap(u32).init(allocator);
        defer map.deinit();

        var keys = try allocator.alloc([16]u8, N);
        defer allocator.free(keys);
        for (keys, 0..) |*k, i| {
            _ = try std.fmt.bufPrint(k, "key{d:0>12}", .{i});
        }

        const t0 = std.time.nanoTimestamp();
        for (keys, 0..) |*k, i| try map.put(k, @intCast(i));
        const t1 = std.time.nanoTimestamp();

        for (keys) |*k| _ = map.get(k);
        const t2 = std.time.nanoTimestamp();

        for (keys) |*k| _ = map.remove(k);
        const t3 = std.time.nanoTimestamp();

        std.debug.print("{s:<20} {d:>12} {d:>12} {d:>12}\n", .{
            "StringHashMap",
            @divTrunc(t1 - t0, N),
            @divTrunc(t2 - t1, N),
            @divTrunc(t3 - t2, N),
        });
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Hash Table Performance Analysis ===\n\n", .{});

    // Basic benchmark
    const result = try benchmark(allocator, "AutoHashMap", 10000);
    std.debug.print("Benchmark (10k ops):\n", .{});
    std.debug.print("  Insert: {d} ns/op\n", .{result.insert_ns});
    std.debug.print("  Lookup: {d} ns/op\n", .{result.lookup_ns});
    std.debug.print("  Delete: {d} ns/op\n", .{result.delete_ns});
    std.debug.print("  Memory: {d} bytes\n", .{result.memory_bytes});

    try loadFactorAnalysis(allocator);
    collisionAnalysis();
    try compareImplementations(allocator);

    std.debug.print("\n=== Summary ===\n", .{});
    std.debug.print("Best practices:\n", .{});
    std.debug.print("  1. Load factor 0.7-0.75 ให้ balance ดีที่สุด\n", .{});
    std.debug.print("  2. Robin Hood reduces variance in probe length\n", .{});
    std.debug.print("  3. Cuckoo hashing: O(1) worst-case lookup\n", .{});
    std.debug.print("  4. Bloom filter: space-efficient membership test\n", .{});
    std.debug.print("  5. Count-Min Sketch: frequency estimation\n", .{});
    std.debug.print("  6. Consistent hashing: distributed systems\n", .{});
}
```

---

## สรุปส่วนที่ 80: Hash Table

### เปรียบเทียบ Collision Resolution

| Method | Avg Lookup | Worst Lookup | Memory | ข้อดี | ข้อเสีย |
|--------|------------|--------------|--------|-------|---------|
| Chaining | O(1+α) | O(n) | High | Simple, handles high load | Cache unfriendly |
| Linear Probing | O(1/(1-α)) | O(n) | Low | Cache friendly | Clustering |
| Quadratic Probing | O(1/(1-α)) | O(n) | Low | Less clustering | Complex |
| Double Hashing | O(1/(1-α)) | O(n) | Low | Least clustering | Two hash functions |
| Robin Hood | O(1/(1-α)) | O(log n) | Low | Low variance | Complex delete |
| Cuckoo | O(1) | O(1) | Medium | O(1) worst lookup | Occasional rehash |

α = load factor

### Probabilistic Data Structures

| Structure | Space | Operations | Use Case |
|-----------|-------|------------|----------|
| Bloom Filter | O(n) | add, contains | Membership test |
| Count-Min Sketch | O(1) | add, estimate | Frequency counting |
| HyperLogLog | O(1) | add, cardinality | Unique count |
| MinHash | O(k) | add, similarity | Set similarity |

### ใน Zig ใช้

- `std.AutoHashMap(K, V)` สำหรับ primitive keys
- `std.StringHashMap(V)` สำหรับ string keys
- `std.HashMap(K, V, Context, max_load_pct)` สำหรับ custom hash
- `std.ArrayHashMap` สำหรับ ordered iteration

---

## บทสรุปคอร์ส Part 76-80

เราได้เรียนรู้:
- **Design Patterns**: Factory, Builder, Observer, Strategy, Command, State Machine, Iterator, Decorator, Pipeline, Event System
- **Linked List**: Singly, Doubly, Circular, Skip List, Intrusive, XOR
- **Binary Tree**: BST, AVL, Red-Black, B-Tree, Heap, Trie
- **Graph**: BFS, DFS, Dijkstra, MST, SCC, Topological Sort
- **Hash Table**: Chaining, Open Addressing, Robin Hood, Cuckoo, Bloom Filter, Count-Min Sketch, Consistent Hashing

*จบ Part 80 - สิ้นสุด Steps 791-800*
