# ส่วนที่ 35: std.hash_map - Hash Map Data Structure

## ภาพรวมของบทเรียน

`std.hash_map` ใน Zig ให้ hash map implementation ที่มีประสิทธิภาพสูง รองรับ generic types, custom hash functions, และ multiple variants สำหรับ use cases ต่างๆ บทนี้ครอบคลุมตั้งแต่การใช้งานพื้นฐาน ไปจนถึง performance tuning และ real-world patterns

---

## Step 341: std.AutoHashMap(K, V) Basics

### การสร้าง AutoHashMap

```zig
const std = @import("std");

pub fn autoHashMapBasics(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // AutoHashMap ใช้ hash function อัตโนมัติตาม key type
    // เหมาะสำหรับ integer, float, pointer keys
    var map = std.AutoHashMap(i32, []const u8).init(allocator);
    defer map.deinit();
    
    // เพิ่ม key-value pairs
    try map.put(1, "หนึ่ง");
    try map.put(2, "สอง");
    try map.put(3, "สาม");
    try map.put(10, "สิบ");
    try map.put(100, "ร้อย");
    
    try stdout.print("HashMap มี {d} entries\n", .{map.count()});
    
    // ดึงค่า
    if (map.get(2)) |val| {
        try stdout.print("key=2: {s}\n", .{val});
    }
    
    if (map.get(99)) |val| {
        try stdout.print("key=99: {s}\n", .{val});
    } else {
        try stdout.print("key=99: ไม่พบ\n", .{});
    }
    
    // ตรวจสอบว่ามี key อยู่
    try stdout.print("contains 3: {}\n", .{map.contains(3)});
    try stdout.print("contains 5: {}\n", .{map.contains(5)});
}
```

### AutoHashMap Types ต่างๆ

```zig
const std = @import("std");

pub fn differentHashMapTypes(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // Integer key -> Integer value
    var int_map = std.AutoHashMap(u64, u64).init(allocator);
    defer int_map.deinit();
    
    // ใช้สำหรับ frequency count
    const numbers = [_]u64{ 1, 2, 3, 2, 1, 3, 3, 2, 1, 1 };
    for (numbers) |n| {
        const entry = try int_map.getOrPut(n);
        if (entry.found_existing) {
            entry.value_ptr.* += 1;
        } else {
            entry.value_ptr.* = 1;
        }
    }
    
    try stdout.print("Frequency count:\n", .{});
    var iter = int_map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  {d}: {d} ครั้ง\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
    
    // Pointer key (ใช้ pointer address เป็น key)
    var ptr_map = std.AutoHashMap(*const u8, []const u8).init(allocator);
    defer ptr_map.deinit();
    
    const byte1: u8 = 42;
    const byte2: u8 = 99;
    
    try ptr_map.put(&byte1, "first byte");
    try ptr_map.put(&byte2, "second byte");
    
    if (ptr_map.get(&byte1)) |val| {
        try stdout.print("\nPointer map: {s}\n", .{val});
    }
    
    // Enum key
    const Direction = enum { north, south, east, west };
    var dir_map = std.AutoHashMap(Direction, []const u8).init(allocator);
    defer dir_map.deinit();
    
    try dir_map.put(.north, "เหนือ");
    try dir_map.put(.south, "ใต้");
    try dir_map.put(.east, "ตะวันออก");
    try dir_map.put(.west, "ตะวันตก");
    
    try stdout.print("\nDirection map:\n", .{});
    for ([_]Direction{ .north, .south, .east, .west }) |dir| {
        if (dir_map.get(dir)) |name| {
            try stdout.print("  {s} = {s}\n", .{ @tagName(dir), name });
        }
    }
}
```

---

## Step 342: put, get, remove Operations

### CRUD Operations

```zig
const std = @import("std");

pub fn crudOperations(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.AutoHashMap(u32, f64).init(allocator);
    defer map.deinit();
    
    // CREATE: put
    try map.put(1, 1.0);
    try map.put(2, 4.0);
    try map.put(3, 9.0);
    try map.put(4, 16.0);
    try map.put(5, 25.0);
    
    try stdout.print("หลัง put 5 entries: count = {d}\n", .{map.count()});
    
    // READ: get
    if (map.get(3)) |val| {
        try stdout.print("get(3) = {d}\n", .{val});
    }
    
    // READ: getPtr (pointer to value สำหรับ in-place modification)
    if (map.getPtr(3)) |ptr| {
        ptr.* *= 2.0; // แก้ไขค่าใน map โดยตรง
        try stdout.print("หลัง multiply 3 by 2: get(3) = {d}\n", .{map.get(3).?});
    }
    
    // UPDATE: put กับ key ที่มีอยู่แล้ว (overwrite)
    try map.put(3, 100.0);
    try stdout.print("หลัง update key 3: get(3) = {d}\n", .{map.get(3).?});
    
    // DELETE: remove
    const removed = map.remove(3);
    try stdout.print("remove(3) = {}\n", .{removed}); // true ถ้าลบสำเร็จ
    try stdout.print("หลัง remove(3): count = {d}\n", .{map.count()});
    try stdout.print("get(3) after remove = {?}\n", .{map.get(3)});
    
    // fetchRemove: ลบและคืนค่าที่ถูกลบ
    if (map.fetchRemove(4)) |kv| {
        try stdout.print("fetchRemove(4): key={d}, value={d}\n", .{
            kv.key,
            kv.value,
        });
    }
}
```

### getOrPut และ getOrPutValue

```zig
const std = @import("std");

pub fn getOrPutOperations(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.AutoHashMap([]const u8, u32).init(allocator);
    defer map.deinit();
    
    // Word count ด้วย getOrPut
    const words = [_][]const u8{
        "the", "quick", "brown", "fox",
        "the", "lazy", "dog", "the",
        "fox", "jumped", "over", "the",
    };
    
    for (words) |word| {
        // getOrPut ส่งคืน struct ที่มี found_existing และ value_ptr
        const entry = try map.getOrPut(word);
        if (entry.found_existing) {
            entry.value_ptr.* += 1;
        } else {
            entry.value_ptr.* = 1;
        }
    }
    
    try stdout.print("Word counts:\n", .{});
    var iter = map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  '{s}': {d}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
    
    // getOrPutValue: ใส่ default value ถ้าไม่มี key
    var defaults = std.AutoHashMap(u32, u32).init(allocator);
    defer defaults.deinit();
    
    _ = try defaults.getOrPutValue(1, 100); // ใส่ 100 ถ้าไม่มี key 1
    _ = try defaults.getOrPutValue(1, 200); // ไม่ overwrite เพราะมีอยู่แล้ว
    
    try stdout.print("\ngetOrPutValue(1, default=100): {d}\n", .{defaults.get(1).?});
    
    // putNoClobber: ใส่เฉพาะถ้าไม่มี key นั้น
    var no_clobber = std.AutoHashMap(u32, u32).init(allocator);
    defer no_clobber.deinit();
    
    try no_clobber.put(1, 100);
    
    // putNoClobber จะ error ถ้า key มีอยู่แล้ว
    no_clobber.putNoClobber(1, 200) catch |err| {
        try stdout.print("putNoClobber error (expected): {}\n", .{err});
    };
    
    try stdout.print("ค่าหลัง putNoClobber: {d}\n", .{no_clobber.get(1).?});
}
```

---

## Step 343: std.StringHashMap(V)

### StringHashMap พื้นฐาน

```zig
const std = @import("std");

pub fn stringHashMapBasics(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // StringHashMap ออกแบบมาเพื่อใช้ string เป็น key โดยเฉพาะ
    // ใช้ std.hash_map.StringContext สำหรับ string hashing/equality
    var map = std.StringHashMap(u32).init(allocator);
    defer map.deinit();
    
    // เพิ่ม key-value
    try map.put("Thailand", 67_000_000);
    try map.put("Japan", 125_000_000);
    try map.put("China", 1_400_000_000);
    try map.put("USA", 330_000_000);
    try map.put("India", 1_380_000_000);
    
    try stdout.print("ประชากรโลก:\n", .{});
    
    var iter = map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  {s}: {d:,}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
    
    // ค้นหา
    if (map.get("Thailand")) |pop| {
        try stdout.print("\nประชากรไทย: {d:,}\n", .{pop});
    }
    
    // ลบ
    _ = map.remove("USA");
    try stdout.print("หลัง remove USA: count = {d}\n", .{map.count()});
}
```

### String Key ต้องระวัง Ownership

```zig
const std = @import("std");

pub fn stringKeyOwnership(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ระวัง! StringHashMap ไม่ copy key โดยอัตโนมัติ
    // key ต้องมีอายุยาวกว่า HashMap
    
    var map = std.StringHashMap(u32).init(allocator);
    defer map.deinit();
    
    // วิธีที่ 1: ใช้ string literals (ปลอดภัย - อยู่ใน binary)
    try map.put("literal_key", 42);
    
    // วิธีที่ 2: Allocate key strings เอง
    const key1 = try allocator.dupe(u8, "dynamic_key_1");
    const key2 = try allocator.dupe(u8, "dynamic_key_2");
    
    try map.put(key1, 100);
    try map.put(key2, 200);
    
    // ต้อง free keys หลัง remove จาก map
    // (หรือ free หลัง map.deinit() ถ้าคิดว่า keys จะอยู่ตลอด lifetime)
    if (map.fetchRemove(key1)) |kv| {
        try stdout.print("Removed: {s} = {d}\n", .{ kv.key, kv.value });
        allocator.free(kv.key); // free key หลังจาก remove
    }
    
    // key2 จะ leak ถ้าไม่ free ก่อน deinit
    // ดังนั้นต้อง iterate และ free ทุก key ก่อน deinit
    var iter = map.iterator();
    while (iter.next()) |entry| {
        allocator.free(entry.key_ptr.*);
    }
    
    try stdout.print("String keys managed properly\n", .{});
    _ = stdout;
}
```

### String Map สำหรับ Configuration

```zig
const std = @import("std");

pub const Config = struct {
    data: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) Config {
        return Config{
            .data = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Config) void {
        // Free all keys and values
        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.data.deinit();
    }
    
    pub fn set(self: *Config, key: []const u8, value: []const u8) !void {
        const key_copy = try self.allocator.dupe(u8, key);
        errdefer self.allocator.free(key_copy);
        
        const value_copy = try self.allocator.dupe(u8, value);
        errdefer self.allocator.free(value_copy);
        
        // ถ้า key มีอยู่แล้ว ให้ free old value
        if (self.data.getPtr(key)) |old_val_ptr| {
            self.allocator.free(old_val_ptr.*);
            old_val_ptr.* = value_copy;
            self.allocator.free(key_copy); // free new key เพราะใช้ key เดิม
            return;
        }
        
        try self.data.put(key_copy, value_copy);
    }
    
    pub fn get(self: *const Config, key: []const u8) ?[]const u8 {
        return self.data.get(key);
    }
    
    pub fn getOrDefault(self: *const Config, key: []const u8, default: []const u8) []const u8 {
        return self.data.get(key) orelse default;
    }
};

pub fn demonstrateConfig(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var config = Config.init(allocator);
    defer config.deinit();
    
    try config.set("database.host", "localhost");
    try config.set("database.port", "5432");
    try config.set("database.name", "myapp");
    try config.set("server.port", "8080");
    try config.set("server.workers", "4");
    
    try stdout.print("Configuration:\n", .{});
    try stdout.print("  DB Host: {s}\n", .{config.getOrDefault("database.host", "")});
    try stdout.print("  DB Port: {s}\n", .{config.getOrDefault("database.port", "5432")});
    try stdout.print("  Server Port: {s}\n", .{config.getOrDefault("server.port", "8080")});
    try stdout.print("  SSL: {s}\n", .{config.getOrDefault("server.ssl", "false")});
}
```

---

## Step 344: HashMap Iteration

### Iteration Patterns

```zig
const std = @import("std");

pub fn hashMapIteration(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.AutoHashMap(u32, []const u8).init(allocator);
    defer map.deinit();
    
    // เติมข้อมูล
    try map.put(1, "หนึ่ง");
    try map.put(2, "สอง");
    try map.put(3, "สาม");
    try map.put(4, "สี่");
    try map.put(5, "ห้า");
    
    // วิธีที่ 1: Iterator
    try stdout.print("Iterator:\n", .{});
    var iter = map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  {d} => {s}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
    
    // วิธีที่ 2: Key iterator
    try stdout.print("\nKeys only:\n  ", .{});
    var key_iter = map.keyIterator();
    while (key_iter.next()) |key| {
        try stdout.print("{d} ", .{key.*});
    }
    try stdout.print("\n", .{});
    
    // วิธีที่ 3: Value iterator
    try stdout.print("\nValues only:\n  ", .{});
    var val_iter = map.valueIterator();
    while (val_iter.next()) |val| {
        try stdout.print("{s} ", .{val.*});
    }
    try stdout.print("\n", .{});
    
    // Modification ระหว่าง iteration (ระวัง!)
    // ห้าม add/remove entries ระหว่าง iteration
    // แต่สามารถแก้ไข value ผ่าน pointer ได้
    var mod_iter = map.iterator();
    while (mod_iter.next()) |entry| {
        if (entry.key_ptr.* % 2 == 0) {
            // แก้ไข value ของ even keys
            entry.value_ptr.* = "EVEN";
        }
    }
    
    try stdout.print("\nหลัง modify even values:\n", .{});
    var final_iter = map.iterator();
    while (final_iter.next()) |entry| {
        try stdout.print("  {d} => {s}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
}
```

### Sorting HashMap Output

```zig
const std = @import("std");

pub fn sortedHashMapOutput(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.AutoHashMap(u32, u32).init(allocator);
    defer map.deinit();
    
    // สร้างข้อมูล
    var prng = std.Random.DefaultPrng.init(42);
    const rand = prng.random();
    
    var i: u32 = 0;
    while (i < 10) : (i += 1) {
        try map.put(rand.uintLessThan(u32, 100), rand.uintLessThan(u32, 1000));
    }
    
    // Collect entries สำหรับ sorting
    const Entry = struct {
        key: u32,
        value: u32,
    };
    
    var entries = std.ArrayList(Entry).init(allocator);
    defer entries.deinit();
    
    var iter = map.iterator();
    while (iter.next()) |entry| {
        try entries.append(.{
            .key = entry.key_ptr.*,
            .value = entry.value_ptr.*,
        });
    }
    
    // Sort by key
    std.sort.block(Entry, entries.items, {}, struct {
        fn lessThan(_: void, a: Entry, b: Entry) bool {
            return a.key < b.key;
        }
    }.lessThan);
    
    try stdout.print("Sorted HashMap output:\n", .{});
    for (entries.items) |entry| {
        try stdout.print("  {d:3} => {d}\n", .{ entry.key, entry.value });
    }
}
```

---

## Step 345: HashMap กับ Allocator

### Custom Allocator Patterns

```zig
const std = @import("std");

pub fn hashMapWithDifferentAllocators() !void {
    const stdout = std.io.getStdOut().writer();
    
    // 1. Arena allocator: สำหรับ temp data
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const arena_alloc = arena.allocator();
    
    {
        var temp_map = std.AutoHashMap(u32, u32).init(arena_alloc);
        // ไม่ต้อง call deinit เพราะ arena จะ free ทุกอย่าง
        
        try temp_map.put(1, 100);
        try temp_map.put(2, 200);
        
        try stdout.print("Arena map count: {d}\n", .{temp_map.count()});
    }
    
    // 2. FixedBufferAllocator: สำหรับ stack-based HashMap (no heap)
    var buffer: [4096]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const fba_alloc = fba.allocator();
    
    var small_map = std.AutoHashMap(u8, u8).init(fba_alloc);
    defer small_map.deinit();
    
    var j: u8 = 0;
    while (j < 10) : (j += 1) {
        small_map.put(j, j * j) catch |err| {
            try stdout.print("หน่วยความจำเต็ม ที่ j={d}: {}\n", .{ j, err });
            break;
        };
    }
    
    try stdout.print("Stack-based map count: {d}\n", .{small_map.count()});
    
    // 3. GeneralPurposeAllocator: สำหรับ tracking leaks
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const status = gpa.deinit();
        if (status == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }
    const gpa_alloc = gpa.allocator();
    
    var tracked_map = std.AutoHashMap(u32, u32).init(gpa_alloc);
    defer tracked_map.deinit();
    
    try tracked_map.put(42, 1337);
    try stdout.print("GPA map count: {d}\n", .{tracked_map.count()});
}
```

### Capacity Management

```zig
const std = @import("std");

pub fn hashMapCapacity(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // สร้าง HashMap พร้อม pre-allocated capacity
    var map = std.AutoHashMap(u32, u32).init(allocator);
    defer map.deinit();
    
    // ensureTotalCapacity: จอง capacity ล่วงหน้า
    // ช่วยลด rehashing
    try map.ensureTotalCapacity(100);
    
    try stdout.print("Capacity หลัง ensureTotalCapacity(100): {d}\n", .{
        map.capacity(),
    });
    
    // เติมข้อมูล
    var i: u32 = 0;
    while (i < 100) : (i += 1) {
        try map.put(i, i * i);
    }
    
    try stdout.print("Count: {d}, Capacity: {d}\n", .{
        map.count(),
        map.capacity(),
    });
    
    // ensureUnusedCapacity: จอง capacity เพิ่ม
    try map.ensureUnusedCapacity(50);
    try stdout.print("Capacity หลัง ensureUnusedCapacity(50): {d}\n", .{
        map.capacity(),
    });
    
    // clearRetainingCapacity: เคลียร์ข้อมูลแต่เก็บ capacity ไว้
    map.clearRetainingCapacity();
    try stdout.print("หลัง clearRetainingCapacity: count={d}, capacity={d}\n", .{
        map.count(),
        map.capacity(),
    });
    
    // clearAndFree: เคลียร์ข้อมูลและ free memory
    // ต้องเติมข้อมูลใหม่ก่อน
    try map.put(1, 1);
    map.clearAndFree();
    try stdout.print("หลัง clearAndFree: count={d}, capacity={d}\n", .{
        map.count(),
        map.capacity(),
    });
}
```

---

## Step 346: Custom Hash Functions

### เขียน Custom Hash Function

```zig
const std = @import("std");

// Custom hash สำหรับ struct key
const Point = struct {
    x: i32,
    y: i32,
};

// Context ที่กำหนด hash และ equals สำหรับ Point
const PointContext = struct {
    pub fn hash(self: PointContext, p: Point) u64 {
        _ = self;
        // Combine x and y hashes
        var hasher = std.hash.Wyhash.init(0);
        hasher.update(std.mem.asBytes(&p.x));
        hasher.update(std.mem.asBytes(&p.y));
        return hasher.final();
    }
    
    pub fn eql(self: PointContext, a: Point, b: Point) bool {
        _ = self;
        return a.x == b.x and a.y == b.y;
    }
};

pub fn customHashFunction(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // ใช้ HashMap ที่มี custom context
    var map = std.HashMap(Point, []const u8, PointContext, std.hash_map.default_max_load_percentage).init(allocator);
    defer map.deinit();
    
    try map.put(.{ .x = 0, .y = 0 }, "origin");
    try map.put(.{ .x = 1, .y = 0 }, "right");
    try map.put(.{ .x = 0, .y = 1 }, "up");
    try map.put(.{ .x = -1, .y = -1 }, "diagonal");
    
    // ค้นหา
    const search_point = Point{ .x = 1, .y = 0 };
    if (map.get(search_point)) |val| {
        try stdout.print("({d},{d}) = {s}\n", .{
            search_point.x,
            search_point.y,
            val,
        });
    }
    
    // Iterate
    try stdout.print("\nAll points:\n", .{});
    var iter = map.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  ({d},{d}) = {s}\n", .{
            entry.key_ptr.x,
            entry.key_ptr.y,
            entry.value_ptr.*,
        });
    }
}
```

### Hash Functions ต่างๆ ใน std

```zig
const std = @import("std");

pub fn exploreHashFunctions() !void {
    const stdout = std.io.getStdOut().writer();
    
    const data = "Hello, Zig!";
    
    // Wyhash - fast non-cryptographic hash (ใช้ใน HashMap)
    const wyhash = std.hash.Wyhash.hash(0, data);
    try stdout.print("Wyhash: {x}\n", .{wyhash});
    
    // FNV-1a hash
    const fnv = std.hash.Fnv1a_64.hash(data);
    try stdout.print("FNV-1a: {x}\n", .{fnv});
    
    // CityHash (Adler32 as alternative)
    const adler = std.hash.Adler32.hash(data);
    try stdout.print("Adler32: {x}\n", .{adler});
    
    // CRC32
    const crc = std.hash.Crc32.hash(data);
    try stdout.print("CRC32: {x}\n", .{crc});
    
    // Test hash distribution
    try stdout.print("\nHash distribution test:\n", .{});
    
    var buckets = [_]u32{0} ** 16;
    var i: u32 = 0;
    while (i < 10000) : (i += 1) {
        const key = i;
        const h = std.hash.Wyhash.hash(0, std.mem.asBytes(&key));
        buckets[h % buckets.len] += 1;
    }
    
    for (buckets, 0..) |count, bucket| {
        try stdout.print("  bucket[{d:2}]: {d}\n", .{ bucket, count });
    }
}
```

### Case-insensitive String HashMap

```zig
const std = @import("std");

// Context สำหรับ case-insensitive string keys
const CaseInsensitiveContext = struct {
    pub fn hash(self: CaseInsensitiveContext, key: []const u8) u64 {
        _ = self;
        var hasher = std.hash.Wyhash.init(0);
        for (key) |c| {
            hasher.update(&[_]u8{std.ascii.toLower(c)});
        }
        return hasher.final();
    }
    
    pub fn eql(self: CaseInsensitiveContext, a: []const u8, b: []const u8) bool {
        _ = self;
        return std.ascii.eqlIgnoreCase(a, b);
    }
};

pub fn caseInsensitiveMap(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.HashMap(
        []const u8,
        u32,
        CaseInsensitiveContext,
        std.hash_map.default_max_load_percentage,
    ).init(allocator);
    defer map.deinit();
    
    try map.put("Thailand", 1);
    try map.put("JAPAN", 2);
    try map.put("China", 3);
    
    // ค้นหาด้วย case ต่างๆ
    try stdout.print("Case-insensitive lookup:\n", .{});
    try stdout.print("  'thailand' = {?d}\n", .{map.get("thailand")});
    try stdout.print("  'THAILAND' = {?d}\n", .{map.get("THAILAND")});
    try stdout.print("  'Thailand' = {?d}\n", .{map.get("Thailand")});
    try stdout.print("  'japan' = {?d}\n", .{map.get("japan")});
    try stdout.print("  'Japan' = {?d}\n", .{map.get("Japan")});
}
```

---

## Step 347: HashMap Resizing และ Load Factor

### Load Factor และ Performance

```zig
const std = @import("std");

pub fn loadFactorDemo(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    // default_max_load_percentage = 80 (80%)
    // หมายความว่า HashMap จะ resize เมื่อ count/capacity > 0.8
    
    // ดู default load factor
    try stdout.print("Default max load percentage: {d}%\n", .{
        std.hash_map.default_max_load_percentage,
    });
    
    // HashMap ที่มี load factor ต่ำ (ใช้หน่วยความจำมากกว่า แต่เร็วกว่า)
    // 50% load factor
    var fast_map = std.HashMap(
        u32,
        u32,
        std.hash_map.AutoContext(u32),
        50, // max load percentage
    ).init(allocator);
    defer fast_map.deinit();
    
    // HashMap ที่มี load factor สูง (ประหยัดหน่วยความจำ แต่ช้ากว่า)
    // 95% load factor
    var memory_map = std.HashMap(
        u32,
        u32,
        std.hash_map.AutoContext(u32),
        95, // max load percentage
    ).init(allocator);
    defer memory_map.deinit();
    
    // Benchmark
    const N: u32 = 10000;
    var i: u32 = 0;
    
    // Test fast_map
    var timer = try std.time.Timer.start();
    i = 0;
    while (i < N) : (i += 1) {
        try fast_map.put(i, i * i);
    }
    const fast_put = timer.read();
    
    timer.reset();
    i = 0;
    var sum1: u64 = 0;
    while (i < N) : (i += 1) {
        sum1 += fast_map.get(i) orelse 0;
    }
    const fast_get = timer.read();
    
    // Test memory_map
    timer.reset();
    i = 0;
    while (i < N) : (i += 1) {
        try memory_map.put(i, i * i);
    }
    const memory_put = timer.read();
    
    timer.reset();
    i = 0;
    var sum2: u64 = 0;
    while (i < N) : (i += 1) {
        sum2 += memory_map.get(i) orelse 0;
    }
    const memory_get = timer.read();
    
    try stdout.print("\nLoad Factor Comparison (N={d}):\n", .{N});
    try stdout.print("              | {s:>10} | {s:>10}\n", .{ "50%", "95%" });
    try stdout.print("-" .* 36 ++ "\n", .{});
    try stdout.print("  put time   | {d:>10} | {d:>10}\n", .{
        fast_put / std.time.ns_per_us,
        memory_put / std.time.ns_per_us,
    });
    try stdout.print("  get time   | {d:>10} | {d:>10}\n", .{
        fast_get / std.time.ns_per_us,
        memory_get / std.time.ns_per_us,
    });
    try stdout.print("  capacity   | {d:>10} | {d:>10}\n", .{
        fast_map.capacity(),
        memory_map.capacity(),
    });
    
    _ = sum1;
    _ = sum2;
}
```

### Memory Usage Analysis

```zig
const std = @import("std");

pub fn analyzeMemoryUsage(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var map = std.AutoHashMap(u64, u64).init(allocator);
    defer map.deinit();
    
    try stdout.print("HashMap growth pattern:\n", .{});
    try stdout.print("{s:>8} | {s:>10} | {s:>10} | {s:>8}\n", .{
        "Count",
        "Capacity",
        "Mem (bytes)",
        "Load%",
    });
    try stdout.print("-" .* 44 ++ "\n", .{});
    
    var i: u64 = 1;
    var prev_capacity: u32 = 0;
    
    while (i <= 1024) : (i *= 2) {
        // เติมข้อมูลถึง i entries
        while (map.count() < i) {
            const key = map.count();
            try map.put(@intCast(key), key * key);
        }
        
        const cap = map.capacity();
        const load = (map.count() * 100) / @max(cap, 1);
        const mem = cap * (@sizeOf(u64) * 2 + @sizeOf(u64)); // approximate
        
        const resize_marker: []const u8 = if (cap != prev_capacity) " <<RESIZE" else "";
        try stdout.print("{d:>8} | {d:>10} | {d:>10} | {d:>7}%{s}\n", .{
            map.count(),
            cap,
            mem,
            load,
            resize_marker,
        });
        
        prev_capacity = cap;
    }
}
```

---

## Step 348: MultiArrayList vs HashMap

### เปรียบเทียบ Data Structures

```zig
const std = @import("std");

// Multi Array List: เหมาะสำหรับ iteration ที่ต้องการ cache-friendly
// HashMap: เหมาะสำหรับ random access ด้วย key

const Entity = struct {
    id: u32,
    health: f32,
    damage: f32,
    active: bool,
};

pub fn multiArrayListVsHashMap(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const N: usize = 10000;
    
    // === MultiArrayList ===
    var mal = std.MultiArrayList(Entity){};
    defer mal.deinit(allocator);
    
    try mal.ensureTotalCapacity(allocator, N);
    
    var timer = try std.time.Timer.start();
    
    // เพิ่ม entities
    var i: u32 = 0;
    while (i < N) : (i += 1) {
        try mal.append(allocator, .{
            .id = i,
            .health = 100.0,
            .damage = 10.0,
            .active = true,
        });
    }
    
    // Update health ทุก entity (cache-friendly)
    const health_slice = mal.items(.health);
    for (health_slice) |*h| {
        h.* -= 1.0;
    }
    
    const mal_time = timer.read();
    
    // === HashMap ===
    var hmap = std.AutoHashMap(u32, Entity).init(allocator);
    defer hmap.deinit();
    
    try hmap.ensureTotalCapacity(N);
    
    timer.reset();
    
    // เพิ่ม entities
    i = 0;
    while (i < N) : (i += 1) {
        try hmap.put(i, .{
            .id = i,
            .health = 100.0,
            .damage = 10.0,
            .active = true,
        });
    }
    
    // Update health ทุก entity (cache-unfriendly)
    var iter = hmap.iterator();
    while (iter.next()) |entry| {
        entry.value_ptr.health -= 1.0;
    }
    
    const hmap_time = timer.read();
    
    try stdout.print("N = {d} entities:\n", .{N});
    try stdout.print("  MultiArrayList: {d} us\n", .{mal_time / std.time.ns_per_us});
    try stdout.print("  HashMap:        {d} us\n", .{hmap_time / std.time.ns_per_us});
    try stdout.print("\nMultiArrayList ดีกว่าเมื่อ:\n", .{});
    try stdout.print("  - Iterate ทุก elements บ่อยๆ\n", .{});
    try stdout.print("  - Access fields เฉพาะ (เช่น health ทั้งหมด)\n", .{});
    try stdout.print("  - ต้องการ cache-friendly access\n", .{});
    try stdout.print("\nHashMap ดีกว่าเมื่อ:\n", .{});
    try stdout.print("  - ค้นหาด้วย key บ่อยๆ\n", .{});
    try stdout.print("  - Dynamic insertion/deletion\n", .{});
    try stdout.print("  - Key-value relationship ชัดเจน\n", .{});
}
```

---

## Step 349: HashMap Patterns (Memoization, Word Count)

### Memoization Pattern

```zig
const std = @import("std");

// Memoized Fibonacci
const MemoFib = struct {
    cache: std.AutoHashMap(u64, u64),
    
    pub fn init(allocator: std.mem.Allocator) MemoFib {
        return MemoFib{
            .cache = std.AutoHashMap(u64, u64).init(allocator),
        };
    }
    
    pub fn deinit(self: *MemoFib) void {
        self.cache.deinit();
    }
    
    pub fn fib(self: *MemoFib, n: u64) !u64 {
        if (n <= 1) return n;
        
        // ตรวจสอบ cache
        if (self.cache.get(n)) |cached| {
            return cached;
        }
        
        // คำนวณและ cache
        const result = (try self.fib(n - 1)) + (try self.fib(n - 2));
        try self.cache.put(n, result);
        
        return result;
    }
};

pub fn demonstrateMemoization(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var memo = MemoFib.init(allocator);
    defer memo.deinit();
    
    try stdout.print("Memoized Fibonacci:\n", .{});
    
    var timer = try std.time.Timer.start();
    
    const test_values = [_]u64{ 10, 20, 30, 40, 50, 70 };
    for (test_values) |n| {
        timer.reset();
        const result = try memo.fib(n);
        const elapsed = timer.read();
        try stdout.print("  fib({d:2}) = {d:20} ({d} ns)\n", .{
            n,
            result,
            elapsed,
        });
    }
    
    try stdout.print("Cache size: {d}\n", .{memo.cache.count()});
}
```

### Word Count Pattern

```zig
const std = @import("std");

pub fn wordCount(
    text: []const u8,
    allocator: std.mem.Allocator,
) !std.StringHashMap(u32) {
    var counts = std.StringHashMap(u32).init(allocator);
    
    // Tokenize
    var it = std.mem.tokenizeAny(u8, text, " \t\n\r,.!?;:\"'()[]{}");
    while (it.next()) |word| {
        // Convert to lowercase
        const lower_word = try allocator.alloc(u8, word.len);
        errdefer allocator.free(lower_word);
        
        for (word, 0..) |c, i| {
            lower_word[i] = std.ascii.toLower(c);
        }
        
        const entry = try counts.getOrPut(lower_word);
        if (entry.found_existing) {
            entry.value_ptr.* += 1;
            allocator.free(lower_word); // ไม่ต้องเก็บ key ซ้ำ
        } else {
            entry.value_ptr.* = 1;
        }
    }
    
    return counts;
}

pub fn topNWords(
    counts: *const std.StringHashMap(u32),
    n: usize,
    allocator: std.mem.Allocator,
) ![]struct { word: []const u8, count: u32 } {
    const Entry = struct { word: []const u8, count: u32 };
    
    var entries = std.ArrayList(Entry).init(allocator);
    defer entries.deinit();
    
    var iter = counts.iterator();
    while (iter.next()) |entry| {
        try entries.append(.{
            .word = entry.key_ptr.*,
            .count = entry.value_ptr.*,
        });
    }
    
    // Sort by count descending
    std.sort.pdq(Entry, entries.items, {}, struct {
        fn lessThan(_: void, a: Entry, b: Entry) bool {
            return a.count > b.count;
        }
    }.lessThan);
    
    const result_len = @min(n, entries.items.len);
    return try allocator.dupe(Entry, entries.items[0..result_len]);
}

pub fn demonstrateWordCount(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const text =
        \\The quick brown fox jumps over the lazy dog.
        \\The dog barked at the fox. The fox ran away.
        \\A quick brown dog outfoxed the lazy fox.
    ;
    
    var counts = try wordCount(text, allocator);
    defer {
        var iter = counts.iterator();
        while (iter.next()) |entry| {
            allocator.free(entry.key_ptr.*);
        }
        counts.deinit();
    }
    
    try stdout.print("Word Count Results:\n", .{});
    try stdout.print("Total unique words: {d}\n\n", .{counts.count()});
    
    const top = try topNWords(&counts, 10, allocator);
    defer allocator.free(top);
    
    try stdout.print("Top 10 words:\n", .{});
    for (top, 0..) |entry, rank| {
        try stdout.print("  {d:2}. {s:<15} : {d}\n", .{
            rank + 1,
            entry.word,
            entry.count,
        });
    }
}
```

### Graph Adjacency List Pattern

```zig
const std = @import("std");

pub const Graph = struct {
    adjacency: std.AutoHashMap(u32, std.ArrayList(u32)),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) Graph {
        return Graph{
            .adjacency = std.AutoHashMap(u32, std.ArrayList(u32)).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Graph) void {
        var iter = self.adjacency.iterator();
        while (iter.next()) |entry| {
            entry.value_ptr.deinit();
        }
        self.adjacency.deinit();
    }
    
    pub fn addEdge(self: *Graph, from: u32, to: u32) !void {
        const entry = try self.adjacency.getOrPut(from);
        if (!entry.found_existing) {
            entry.value_ptr.* = std.ArrayList(u32).init(self.allocator);
        }
        try entry.value_ptr.append(to);
        
        // Ensure 'to' node exists
        const to_entry = try self.adjacency.getOrPut(to);
        if (!to_entry.found_existing) {
            to_entry.value_ptr.* = std.ArrayList(u32).init(self.allocator);
        }
    }
    
    pub fn neighbors(self: *const Graph, node: u32) []const u32 {
        if (self.adjacency.get(node)) |adj| {
            return adj.items;
        }
        return &[_]u32{};
    }
    
    pub fn bfs(
        self: *const Graph,
        start: u32,
        allocator: std.mem.Allocator,
    ) !std.ArrayList(u32) {
        var visited = std.AutoHashMap(u32, void).init(allocator);
        defer visited.deinit();
        
        var queue = std.ArrayList(u32).init(allocator);
        defer queue.deinit();
        
        var result = std.ArrayList(u32).init(allocator);
        
        try queue.append(start);
        try visited.put(start, {});
        
        while (queue.items.len > 0) {
            const node = queue.orderedRemove(0);
            try result.append(node);
            
            for (self.neighbors(node)) |neighbor| {
                if (!visited.contains(neighbor)) {
                    try visited.put(neighbor, {});
                    try queue.append(neighbor);
                }
            }
        }
        
        return result;
    }
};

pub fn demonstrateGraph(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var graph = Graph.init(allocator);
    defer graph.deinit();
    
    // สร้าง graph
    // 1 -- 2 -- 4
    // |    |    |
    // 3 -- 5 -- 6
    
    try graph.addEdge(1, 2);
    try graph.addEdge(1, 3);
    try graph.addEdge(2, 4);
    try graph.addEdge(2, 5);
    try graph.addEdge(3, 5);
    try graph.addEdge(4, 6);
    try graph.addEdge(5, 6);
    
    // BFS จาก node 1
    var traversal = try graph.bfs(1, allocator);
    defer traversal.deinit();
    
    try stdout.print("BFS traversal from node 1: ", .{});
    for (traversal.items, 0..) |node, i| {
        if (i > 0) try stdout.print(" -> ", .{});
        try stdout.print("{d}", .{node});
    }
    try stdout.print("\n", .{});
    
    // แสดง neighbors ของแต่ละ node
    try stdout.print("\nAdjacency list:\n", .{});
    var iter = graph.adjacency.iterator();
    while (iter.next()) |entry| {
        try stdout.print("  {d}: [", .{entry.key_ptr.*});
        for (entry.value_ptr.items, 0..) |n, i| {
            if (i > 0) try stdout.print(", ", .{});
            try stdout.print("{d}", .{n});
        }
        try stdout.print("]\n", .{});
    }
}
```

---

## Step 350: Performance Tuning

### HashMap Performance Optimization

```zig
const std = @import("std");

pub fn hashMapPerformanceTips(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    const N: usize = 100_000;
    
    try stdout.print("=== HashMap Performance Tips ===\n\n", .{});
    
    // Tip 1: Pre-allocate capacity
    {
        var map1 = std.AutoHashMap(u64, u64).init(allocator);
        defer map1.deinit();
        
        var map2 = std.AutoHashMap(u64, u64).init(allocator);
        defer map2.deinit();
        try map2.ensureTotalCapacity(N);
        
        var timer = try std.time.Timer.start();
        for (0..N) |i| try map1.put(@intCast(i), @intCast(i));
        const without_prealloc = timer.read();
        
        timer.reset();
        for (0..N) |i| try map2.put(@intCast(i), @intCast(i));
        const with_prealloc = timer.read();
        
        try stdout.print("Tip 1: Pre-allocate capacity\n", .{});
        try stdout.print("  Without: {d} us\n", .{without_prealloc / std.time.ns_per_us});
        try stdout.print("  With:    {d} us\n", .{with_prealloc / std.time.ns_per_us});
    }
    
    // Tip 2: Batch operations
    {
        var map = std.AutoHashMap(u64, u64).init(allocator);
        defer map.deinit();
        try map.ensureTotalCapacity(N);
        
        // ดีกว่า: เพิ่มทีเดียวหลาย entries
        for (0..N) |i| {
            try map.putAssumeCapacity(@intCast(i), @intCast(i * i));
        }
        
        try stdout.print("\nTip 2: putAssumeCapacity (ไม่ check resize)\n", .{});
        try stdout.print("  Used when capacity is guaranteed\n", .{});
    }
    
    // Tip 3: getOrPutAssumeCapacity
    {
        var counts = std.AutoHashMap(u32, u32).init(allocator);
        defer counts.deinit();
        
        try counts.ensureTotalCapacity(1000);
        
        var data: [1000]u32 = undefined;
        var prng = std.Random.DefaultPrng.init(42);
        for (&data) |*v| v.* = prng.random().uintLessThan(u32, 100);
        
        for (data) |v| {
            const entry = counts.getOrPutAssumeCapacity(v);
            if (entry.found_existing) {
                entry.value_ptr.* += 1;
            } else {
                entry.value_ptr.* = 1;
            }
        }
        
        try stdout.print("\nTip 3: getOrPutAssumeCapacity\n", .{});
        try stdout.print("  Unique values: {d}\n", .{counts.count()});
    }
}
```

### Cache-Optimized HashMap Usage

```zig
const std = @import("std");

// ตัวอย่าง: LRU Cache ด้วย HashMap
pub fn LruCache(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const Entry = struct {
            value: V,
            order: u64,
        };
        
        map: std.AutoHashMap(K, Entry),
        capacity: usize,
        counter: u64,
        
        pub fn init(allocator: std.mem.Allocator, capacity: usize) Self {
            return Self{
                .map = std.AutoHashMap(K, Entry).init(allocator),
                .capacity = capacity,
                .counter = 0,
            };
        }
        
        pub fn deinit(self: *Self) void {
            self.map.deinit();
        }
        
        pub fn get(self: *Self, key: K) ?V {
            if (self.map.getPtr(key)) |entry| {
                // Update access order
                self.counter += 1;
                entry.order = self.counter;
                return entry.value;
            }
            return null;
        }
        
        pub fn put(self: *Self, key: K, value: V) !void {
            // Evict LRU entry ถ้าเต็ม
            if (self.map.count() >= self.capacity and !self.map.contains(key)) {
                // หา oldest entry
                var oldest_key: ?K = null;
                var oldest_order: u64 = std.math.maxInt(u64);
                
                var iter = self.map.iterator();
                while (iter.next()) |e| {
                    if (e.value_ptr.order < oldest_order) {
                        oldest_order = e.value_ptr.order;
                        oldest_key = e.key_ptr.*;
                    }
                }
                
                if (oldest_key) |k| {
                    _ = self.map.remove(k);
                }
            }
            
            self.counter += 1;
            try self.map.put(key, .{
                .value = value,
                .order = self.counter,
            });
        }
        
        pub fn count(self: *const Self) usize {
            return self.map.count();
        }
    };
}

pub fn demonstrateLruCache(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var cache = LruCache(u32, u32).init(allocator, 3);
    defer cache.deinit();
    
    try stdout.print("LRU Cache (capacity=3):\n", .{});
    
    // เพิ่มข้อมูล
    try cache.put(1, 100);
    try stdout.print("put(1,100): size={d}\n", .{cache.count()});
    
    try cache.put(2, 200);
    try stdout.print("put(2,200): size={d}\n", .{cache.count()});
    
    try cache.put(3, 300);
    try stdout.print("put(3,300): size={d}\n", .{cache.count()});
    
    // Access key 1 เพื่อ refresh LRU order
    _ = cache.get(1);
    try stdout.print("get(1) = {?d} (refreshed)\n", .{cache.get(1)});
    
    // เพิ่ม key ใหม่ - ควรลบ key 2 (least recently used)
    try cache.put(4, 400);
    try stdout.print("put(4,400): size={d}\n", .{cache.count()});
    try stdout.print("get(2) = {?d} (should be evicted)\n", .{cache.get(2)});
    try stdout.print("get(1) = {?d} (should exist)\n", .{cache.get(1)});
    try stdout.print("get(3) = {?d} (should exist)\n", .{cache.get(3)});
    try stdout.print("get(4) = {?d} (should exist)\n", .{cache.get(4)});
}
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Phone Book

```zig
const std = @import("std");

pub const PhoneBook = struct {
    data: std.StringHashMap(std.ArrayList([]const u8)),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) PhoneBook {
        return PhoneBook{
            .data = std.StringHashMap(std.ArrayList([]const u8)).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *PhoneBook) void {
        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            for (entry.value_ptr.items) |num| {
                self.allocator.free(num);
            }
            entry.value_ptr.deinit();
            self.allocator.free(entry.key_ptr.*);
        }
        self.data.deinit();
    }
    
    pub fn addContact(
        self: *PhoneBook,
        name: []const u8,
        phone: []const u8,
    ) !void {
        const entry = try self.data.getOrPut(name);
        if (!entry.found_existing) {
            const name_copy = try self.allocator.dupe(u8, name);
            entry.key_ptr.* = name_copy;
            entry.value_ptr.* = std.ArrayList([]const u8).init(self.allocator);
        }
        
        const phone_copy = try self.allocator.dupe(u8, phone);
        try entry.value_ptr.append(phone_copy);
    }
    
    pub fn lookup(self: *const PhoneBook, name: []const u8) ?[]const []const u8 {
        if (self.data.get(name)) |list| {
            return list.items;
        }
        return null;
    }
    
    pub fn searchByPrefix(
        self: *const PhoneBook,
        prefix: []const u8,
        allocator: std.mem.Allocator,
    ) !std.ArrayList([]const u8) {
        var results = std.ArrayList([]const u8).init(allocator);
        
        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            if (std.mem.startsWith(u8, entry.key_ptr.*, prefix)) {
                try results.append(entry.key_ptr.*);
            }
        }
        
        return results;
    }
};

pub fn demonstratePhoneBook(allocator: std.mem.Allocator) !void {
    const stdout = std.io.getStdOut().writer();
    
    var book = PhoneBook.init(allocator);
    defer book.deinit();
    
    // เพิ่มรายชื่อ
    try book.addContact("สมชาย ใจดี", "081-234-5678");
    try book.addContact("สมชาย ใจดี", "02-123-4567"); // คนเดียวกัน หลายเบอร์
    try book.addContact("สมหญิง รักษ์ดี", "082-345-6789");
    try book.addContact("วิชัย มานะ", "083-456-7890");
    try book.addContact("สมบัติ ขยัน", "084-567-8901");
    
    // ค้นหา
    if (book.lookup("สมชาย ใจดี")) |phones| {
        try stdout.print("เบอร์โทรของ สมชาย ใจดี:\n", .{});
        for (phones) |phone| {
            try stdout.print("  {s}\n", .{phone});
        }
    }
    
    // ค้นหาด้วย prefix
    var results = try book.searchByPrefix("สม", allocator);
    defer results.deinit();
    
    try stdout.print("\nรายชื่อที่ขึ้นต้นด้วย 'สม':\n", .{});
    for (results.items) |name| {
        try stdout.print("  {s}\n", .{name});
    }
}
```

### แบบฝึกหัดที่ 2: Inverted Index

```zig
const std = @import("std");

pub fn buildInvertedIndex(
    documents: []const struct { id: u32, content: []const u8 },
    allocator: std.mem.Allocator,
) !std.StringHashMap(std.ArrayList(u32)) {
    var index = std.StringHashMap(std.ArrayList(u32)).init(allocator);
    
    for (documents) |doc| {
        var words = std.mem.tokenizeAny(u8, doc.content, " \t\n,.!?");
        
        var seen = std.StringHashMap(void).init(allocator);
        defer seen.deinit();
        
        while (words.next()) |word| {
            // ข้าม word ที่เคยเห็นแล้วใน document เดียวกัน
            if (seen.contains(word)) continue;
            try seen.put(word, {});
            
            const entry = try index.getOrPut(word);
            if (!entry.found_existing) {
                entry.value_ptr.* = std.ArrayList(u32).init(allocator);
            }
            try entry.value_ptr.append(doc.id);
        }
    }
    
    return index;
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **std.AutoHashMap** - HashMap สำหรับ integer, enum, pointer keys
2. **put/get/remove** - CRUD operations พื้นฐาน
3. **std.StringHashMap** - HashMap เฉพาะสำหรับ string keys
4. **Iteration** - วิธี iterate ผ่าน all entries
5. **Allocator** - การใช้ different allocators กับ HashMap
6. **Custom Hash** - การเขียน custom hash function
7. **Load Factor** - เข้าใจ resize และ memory usage
8. **MultiArrayList vs HashMap** - เลือก data structure ที่เหมาะสม
9. **Patterns** - Memoization, Word Count, Graph, LRU Cache
10. **Performance** - การ optimize HashMap performance

**เมื่อไหร่ควรใช้อะไร:**
- **AutoHashMap** - integer/enum/pointer keys
- **StringHashMap** - string keys ที่จัดการ memory เอง
- **HashMap with custom context** - complex keys หรือ case-insensitive
- **ต้องจำ:** HashMap ไม่ copy keys/values - ต้องจัดการ lifetime เอง

**Performance Tips:**
- Pre-allocate ด้วย `ensureTotalCapacity`
- ใช้ `getOrPut` แทน `contains` + `put`
- ใช้ `putAssumeCapacity` เมื่อรู้ว่า capacity เพียงพอ
- เลือก load factor ตาม memory/speed tradeoff

---

*ส่วนที่ 35 จบแล้ว! ต่อไปในส่วนที่ 36: std.ArrayList และ data structures อื่นๆ*
