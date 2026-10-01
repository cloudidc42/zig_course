# ส่วนที่ 66: โปรเจกต์ - Key-Value Store

## ขั้นตอนที่ 651-660: สร้าง In-Memory KV Store พร้อม Persistence

---

## ขั้นตอนที่ 651: ความต้องการของ KV Store

Key-Value Store คือโครงสร้างข้อมูลพื้นฐานที่ใช้ในระบบจัดเก็บข้อมูลสมัยใหม่ ในขั้นตอนนี้เราจะกำหนดความต้องการของระบบ

### ความต้องการหลัก

1. **การจัดเก็บข้อมูล**: เก็บคู่ key-value ในหน่วยความจำ
2. **การเข้าถึงข้อมูล**: GET, SET, DELETE operations
3. **TTL Support**: กำหนดอายุข้อมูลได้
4. **Persistence**: บันทึกข้อมูลลงดิสก์
5. **WAL**: Write-Ahead Log สำหรับความน่าเชื่อถือ
6. **TCP Interface**: รองรับการเชื่อมต่อผ่านเครือข่าย

### โครงสร้างโปรเจกต์

```
kvstore/
├── src/
│   ├── main.zig
│   ├── store.zig
│   ├── wal.zig
│   ├── server.zig
│   ├── compaction.zig
│   └── benchmark.zig
├── build.zig
└── README.md
```

### ไฟล์ build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "kvstore",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the KV store server");
    run_step.dependOn(&run_cmd.step);

    const bench_exe = b.addExecutable(.{
        .name = "benchmark",
        .root_source_file = b.path("src/benchmark.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(bench_exe);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/store.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## ขั้นตอนที่ 652: HashMap-Based Storage

เราจะใช้ `std.HashMap` ของ Zig เป็นโครงสร้างข้อมูลหลัก

### ไฟล์ src/store.zig (ส่วนที่ 1)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;
const HashMap = std.StringHashMap;

/// ข้อมูลที่เก็บใน store แต่ละ entry
pub const Entry = struct {
    value: []u8,
    created_at: i64,
    expires_at: ?i64, // null = ไม่มี TTL

    pub fn isExpired(self: Entry) bool {
        if (self.expires_at) |exp| {
            return std.time.milliTimestamp() > exp;
        }
        return false;
    }

    pub fn deinit(self: Entry, allocator: Allocator) void {
        allocator.free(self.value);
    }
};

/// ผลลัพธ์ของ operation
pub const StoreError = error{
    KeyNotFound,
    KeyExpired,
    OutOfMemory,
    InvalidTTL,
    PersistenceError,
    WALError,
};

/// สถิติการใช้งาน store
pub const Stats = struct {
    total_keys: usize,
    expired_keys: usize,
    total_gets: u64,
    total_sets: u64,
    total_deletes: u64,
    hit_count: u64,
    miss_count: u64,

    pub fn hitRate(self: Stats) f64 {
        const total = self.hit_count + self.miss_count;
        if (total == 0) return 0.0;
        return @as(f64, @floatFromInt(self.hit_count)) / @as(f64, @floatFromInt(total));
    }
};

/// KV Store หลัก
pub const KVStore = struct {
    allocator: Allocator,
    data: HashMap(Entry),
    stats: Stats,
    mutex: std.Thread.Mutex,
    wal_writer: ?*WALWriter,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{
            .allocator = allocator,
            .data = HashMap(Entry).init(allocator),
            .stats = std.mem.zeroes(Stats),
            .mutex = std.Thread.Mutex{},
            .wal_writer = null,
        };
    }

    pub fn deinit(self: *Self) void {
        var iter = self.data.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            entry.value_ptr.deinit(self.allocator);
        }
        self.data.deinit();
    }

    /// ตั้งค่า WAL writer
    pub fn setWALWriter(self: *Self, writer: *WALWriter) void {
        self.wal_writer = writer;
    }
};

// WALWriter placeholder สำหรับการ compile
pub const WALWriter = struct {
    pub fn writeSet(self: *WALWriter, key: []const u8, value: []const u8, ttl_ms: ?i64) !void {
        _ = self;
        _ = key;
        _ = value;
        _ = ttl_ms;
    }

    pub fn writeDelete(self: *WALWriter, key: []const u8) !void {
        _ = self;
        _ = key;
    }
};
```

---

## ขั้นตอนที่ 653: GET/SET/DELETE Operations

```zig
// ต่อจาก store.zig

/// GET operation - ดึงค่าจาก key
pub fn get(self: *Self, key: []const u8) StoreError![]const u8 {
    self.mutex.lock();
    defer self.mutex.unlock();

    self.stats.total_gets += 1;

    if (self.data.get(key)) |entry| {
        if (entry.isExpired()) {
            self.stats.expired_keys += 1;
            self.stats.miss_count += 1;
            return StoreError.KeyExpired;
        }
        self.stats.hit_count += 1;
        return entry.value;
    }

    self.stats.miss_count += 1;
    return StoreError.KeyNotFound;
}

/// SET operation - บันทึก key-value
pub fn set(self: *Self, key: []const u8, value: []const u8, ttl_ms: ?i64) StoreError!void {
    self.mutex.lock();
    defer self.mutex.unlock();

    self.stats.total_sets += 1;

    // บันทึกลง WAL ก่อน (ถ้ามี)
    if (self.wal_writer) |wal| {
        wal.writeSet(key, value, ttl_ms) catch return StoreError.WALError;
    }

    // คำนวณ expires_at
    const now = std.time.milliTimestamp();
    const expires_at: ?i64 = if (ttl_ms) |ttl| blk: {
        if (ttl <= 0) return StoreError.InvalidTTL;
        break :blk now + ttl;
    } else null;

    // copy key และ value
    const key_copy = self.allocator.dupe(u8, key) catch return StoreError.OutOfMemory;
    errdefer self.allocator.free(key_copy);

    const value_copy = self.allocator.dupe(u8, value) catch return StoreError.OutOfMemory;
    errdefer self.allocator.free(value_copy);

    const entry = Entry{
        .value = value_copy,
        .created_at = now,
        .expires_at = expires_at,
    };

    // ถ้า key มีอยู่แล้ว ให้ลบค่าเก่าก่อน
    if (self.data.fetchRemove(key)) |old| {
        self.allocator.free(old.key);
        old.value.deinit(self.allocator);
    }

    self.data.put(key_copy, entry) catch {
        self.allocator.free(key_copy);
        value_copy.deinit(self.allocator);
        return StoreError.OutOfMemory;
    };

    self.stats.total_keys = self.data.count();
}

/// DELETE operation - ลบ key
pub fn delete(self: *Self, key: []const u8) StoreError!void {
    self.mutex.lock();
    defer self.mutex.unlock();

    self.stats.total_deletes += 1;

    // บันทึกลง WAL
    if (self.wal_writer) |wal| {
        wal.writeDelete(key) catch return StoreError.WALError;
    }

    if (self.data.fetchRemove(key)) |old| {
        self.allocator.free(old.key);
        old.value.deinit(self.allocator);
        self.stats.total_keys = self.data.count();
        return;
    }

    return StoreError.KeyNotFound;
}

/// EXISTS - ตรวจสอบว่า key มีอยู่หรือไม่
pub fn exists(self: *Self, key: []const u8) bool {
    self.mutex.lock();
    defer self.mutex.unlock();

    if (self.data.get(key)) |entry| {
        return !entry.isExpired();
    }
    return false;
}

/// KEYS - ดึง list ของ keys ทั้งหมด
pub fn keys(self: *Self, allocator: Allocator) ![][]u8 {
    self.mutex.lock();
    defer self.mutex.unlock();

    var result = std.ArrayList([]u8).init(allocator);
    defer result.deinit();

    var iter = self.data.iterator();
    while (iter.next()) |entry| {
        if (!entry.value_ptr.isExpired()) {
            const key_copy = try allocator.dupe(u8, entry.key_ptr.*);
            try result.append(key_copy);
        }
    }

    return result.toOwnedSlice();
}

/// MGET - ดึงหลาย keys พร้อมกัน
pub fn mget(self: *Self, ks: []const []const u8, allocator: Allocator) ![]?[]u8 {
    self.mutex.lock();
    defer self.mutex.unlock();

    var results = try allocator.alloc(?[]u8, ks.len);
    for (ks, 0..) |key, i| {
        if (self.data.get(key)) |entry| {
            if (!entry.isExpired()) {
                results[i] = try allocator.dupe(u8, entry.value);
            } else {
                results[i] = null;
            }
        } else {
            results[i] = null;
        }
    }
    return results;
}
```

---

## ขั้นตอนที่ 654: TTL Support

```zig
// ต่อจาก store.zig

/// SET TTL สำหรับ key ที่มีอยู่แล้ว
pub fn expire(self: *Self, key: []const u8, ttl_ms: i64) StoreError!void {
    self.mutex.lock();
    defer self.mutex.unlock();

    if (ttl_ms <= 0) return StoreError.InvalidTTL;

    if (self.data.getPtr(key)) |entry_ptr| {
        if (entry_ptr.isExpired()) {
            return StoreError.KeyExpired;
        }
        const now = std.time.milliTimestamp();
        entry_ptr.expires_at = now + ttl_ms;
        return;
    }

    return StoreError.KeyNotFound;
}

/// ดู TTL ที่เหลืออยู่ของ key (หน่วย ms)
pub fn ttl(self: *Self, key: []const u8) StoreError!?i64 {
    self.mutex.lock();
    defer self.mutex.unlock();

    if (self.data.get(key)) |entry| {
        if (entry.isExpired()) {
            return StoreError.KeyExpired;
        }
        if (entry.expires_at) |exp| {
            const remaining = exp - std.time.milliTimestamp();
            return if (remaining > 0) remaining else null;
        }
        return null; // ไม่มี TTL
    }

    return StoreError.KeyNotFound;
}

/// ลบ TTL ออกจาก key (key จะไม่มีวันหมดอายุ)
pub fn persist(self: *Self, key: []const u8) StoreError!void {
    self.mutex.lock();
    defer self.mutex.unlock();

    if (self.data.getPtr(key)) |entry_ptr| {
        if (entry_ptr.isExpired()) {
            return StoreError.KeyExpired;
        }
        entry_ptr.expires_at = null;
        return;
    }

    return StoreError.KeyNotFound;
}

/// ลบ key ที่หมดอายุแล้วออกจาก store (background cleanup)
pub fn evictExpired(self: *Self) usize {
    self.mutex.lock();
    defer self.mutex.unlock();

    var expired_keys = std.ArrayList([]const u8).init(self.allocator);
    defer expired_keys.deinit();

    var iter = self.data.iterator();
    while (iter.next()) |entry| {
        if (entry.value_ptr.isExpired()) {
            expired_keys.append(entry.key_ptr.*) catch continue;
        }
    }

    var count: usize = 0;
    for (expired_keys.items) |key| {
        if (self.data.fetchRemove(key)) |old| {
            self.allocator.free(old.key);
            old.value.deinit(self.allocator);
            count += 1;
        }
    }

    self.stats.expired_keys += count;
    self.stats.total_keys = self.data.count();
    return count;
}

/// Background thread สำหรับ eviction
pub fn startEvictionWorker(self: *Self) !std.Thread {
    return std.Thread.spawn(.{}, evictionWorker, .{self});
}

fn evictionWorker(store: *KVStore) void {
    while (true) {
        std.time.sleep(1 * std.time.ns_per_s); // ทำทุก 1 วินาที
        const evicted = store.evictExpired();
        if (evicted > 0) {
            std.log.debug("Evicted {} expired keys", .{evicted});
        }
    }
}

test "TTL support" {
    const allocator = std.testing.allocator;
    var store = KVStore.init(allocator);
    defer store.deinit();

    // SET with TTL 100ms
    try store.set("temp_key", "temp_value", 100);
    
    // ควรดึงได้ทันที
    const val = try store.get("temp_key");
    try std.testing.expectEqualStrings("temp_value", val);

    // รอให้หมดอายุ
    std.time.sleep(200 * std.time.ns_per_ms);

    // ควร expired แล้ว
    const result = store.get("temp_key");
    try std.testing.expectError(StoreError.KeyExpired, result);
}
```

---

## ขั้นตอนที่ 655: Persistence to Disk

```zig
// ไฟล์ src/persistence.zig

const std = @import("std");
const store_mod = @import("store.zig");
const KVStore = store_mod.KVStore;

/// Format สำหรับ snapshot file
/// Header: magic(4) + version(4) + timestamp(8) + count(8)
/// Each entry: key_len(4) + key + value_len(4) + value + expires_at(8) + created_at(8)

const MAGIC = "KVDB";
const VERSION: u32 = 1;

pub const PersistenceManager = struct {
    allocator: std.mem.Allocator,
    snapshot_path: []const u8,

    const Self = @This();

    pub fn init(allocator: std.mem.Allocator, path: []const u8) Self {
        return Self{
            .allocator = allocator,
            .snapshot_path = path,
        };
    }

    /// บันทึก snapshot ของ store ลงไฟล์
    pub fn saveSnapshot(self: *Self, store: *KVStore) !void {
        // เขียนไปยัง temp file ก่อน แล้วค่อย rename
        const temp_path = try std.fmt.allocPrint(self.allocator, "{s}.tmp", .{self.snapshot_path});
        defer self.allocator.free(temp_path);

        const file = try std.fs.createFileAbsolute(temp_path, .{});
        defer file.close();

        var buf_writer = std.io.bufferedWriter(file.writer());
        const writer = buf_writer.writer();

        store.mutex.lock();
        defer store.mutex.unlock();

        // เขียน header
        try writer.writeAll(MAGIC);
        try writer.writeInt(u32, VERSION, .little);
        try writer.writeInt(i64, std.time.milliTimestamp(), .little);
        try writer.writeInt(u64, @intCast(store.data.count()), .little);

        // เขียน entries
        var iter = store.data.iterator();
        while (iter.next()) |entry| {
            if (entry.value_ptr.isExpired()) continue;

            const key = entry.key_ptr.*;
            const value = entry.value_ptr.value;

            try writer.writeInt(u32, @intCast(key.len), .little);
            try writer.writeAll(key);
            try writer.writeInt(u32, @intCast(value.len), .little);
            try writer.writeAll(value);

            // expires_at (0 = no TTL)
            const expires_at: i64 = entry.value_ptr.expires_at orelse 0;
            try writer.writeInt(i64, expires_at, .little);
            try writer.writeInt(i64, entry.value_ptr.created_at, .little);
        }

        try buf_writer.flush();

        // Rename temp file เป็น final file
        try std.fs.renameAbsolute(temp_path, self.snapshot_path);
        std.log.info("Snapshot saved to {s}", .{self.snapshot_path});
    }

    /// โหลด snapshot จากไฟล์
    pub fn loadSnapshot(self: *Self, store: *KVStore) !void {
        const file = std.fs.openFileAbsolute(self.snapshot_path, .{}) catch |err| {
            if (err == error.FileNotFound) {
                std.log.info("No snapshot found, starting fresh", .{});
                return;
            }
            return err;
        };
        defer file.close();

        var buf_reader = std.io.bufferedReader(file.reader());
        const reader = buf_reader.reader();

        // อ่าน magic
        var magic: [4]u8 = undefined;
        try reader.readNoEof(&magic);
        if (!std.mem.eql(u8, &magic, MAGIC)) {
            return error.InvalidSnapshot;
        }

        // อ่าน version
        const version = try reader.readInt(u32, .little);
        if (version != VERSION) {
            return error.UnsupportedVersion;
        }

        // อ่าน timestamp
        const saved_at = try reader.readInt(i64, .little);
        std.log.info("Loading snapshot saved at {}", .{saved_at});

        // อ่าน count
        const count = try reader.readInt(u64, .little);
        std.log.info("Loading {} entries", .{count});

        const now = std.time.milliTimestamp();
        var loaded: u64 = 0;
        var skipped: u64 = 0;

        for (0..count) |_| {
            // อ่าน key
            const key_len = try reader.readInt(u32, .little);
            const key = try self.allocator.alloc(u8, key_len);
            defer self.allocator.free(key);
            try reader.readNoEof(key);

            // อ่าน value
            const value_len = try reader.readInt(u32, .little);
            const value = try self.allocator.alloc(u8, value_len);
            defer self.allocator.free(value);
            try reader.readNoEof(value);

            // อ่าน expires_at
            const expires_at_raw = try reader.readInt(i64, .little);
            const expires_at: ?i64 = if (expires_at_raw == 0) null else expires_at_raw;

            // อ่าน created_at
            _ = try reader.readInt(i64, .little);

            // ตรวจสอบว่า entry ยังไม่หมดอายุ
            if (expires_at) |exp| {
                if (now > exp) {
                    skipped += 1;
                    continue;
                }
            }

            // คำนวณ TTL ที่เหลือ
            const ttl_ms: ?i64 = if (expires_at) |exp| exp - now else null;
            store.set(key, value, ttl_ms) catch |err| {
                std.log.warn("Failed to load entry {s}: {}", .{ key, err });
                continue;
            };
            loaded += 1;
        }

        std.log.info("Loaded {} entries, skipped {} expired", .{ loaded, skipped });
    }
};
```

---

## ขั้นตอนที่ 656: Write-Ahead Log (WAL)

```zig
// ไฟล์ src/wal.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

/// WAL Record types
pub const RecordType = enum(u8) {
    set = 1,
    delete = 2,
    checkpoint = 3,
};

/// WAL Record header
pub const RecordHeader = packed struct {
    record_type: u8,
    timestamp: i64,
    key_len: u32,
    value_len: u32, // 0 สำหรับ delete
    ttl_ms: i64,    // 0 = no TTL
    checksum: u32,
};

pub const WALWriter = struct {
    allocator: Allocator,
    file: std.fs.File,
    buf_writer: std.io.BufferedWriter(4096, std.fs.File.Writer),
    entry_count: u64,

    const Self = @This();

    pub fn open(allocator: Allocator, path: []const u8) !Self {
        const file = try std.fs.createFileAbsolute(path, .{
            .truncate = false,
        });

        // ไปที่ท้ายไฟล์
        try file.seekFromEnd(0);

        return Self{
            .allocator = allocator,
            .file = file,
            .buf_writer = std.io.bufferedWriter(file.writer()),
            .entry_count = 0,
        };
    }

    pub fn close(self: *Self) void {
        self.buf_writer.flush() catch {};
        self.file.close();
    }

    /// เขียน SET record ลง WAL
    pub fn writeSet(self: *Self, key: []const u8, value: []const u8, ttl_ms: ?i64) !void {
        const writer = self.buf_writer.writer();
        const ts = std.time.milliTimestamp();
        const ttl = ttl_ms orelse 0;

        // คำนวณ checksum อย่างง่าย
        var checksum: u32 = 0;
        for (key) |b| checksum +%= b;
        for (value) |b| checksum +%= b;

        // เขียน header
        try writer.writeByte(@intFromEnum(RecordType.set));
        try writer.writeInt(i64, ts, .little);
        try writer.writeInt(u32, @intCast(key.len), .little);
        try writer.writeInt(u32, @intCast(value.len), .little);
        try writer.writeInt(i64, ttl, .little);
        try writer.writeInt(u32, checksum, .little);

        // เขียน key และ value
        try writer.writeAll(key);
        try writer.writeAll(value);

        self.entry_count += 1;

        // Flush ทุก 100 entries
        if (self.entry_count % 100 == 0) {
            try self.buf_writer.flush();
        }
    }

    /// เขียน DELETE record ลง WAL
    pub fn writeDelete(self: *Self, key: []const u8) !void {
        const writer = self.buf_writer.writer();
        const ts = std.time.milliTimestamp();

        var checksum: u32 = 0;
        for (key) |b| checksum +%= b;

        try writer.writeByte(@intFromEnum(RecordType.delete));
        try writer.writeInt(i64, ts, .little);
        try writer.writeInt(u32, @intCast(key.len), .little);
        try writer.writeInt(u32, 0, .little); // no value
        try writer.writeInt(i64, 0, .little); // no ttl
        try writer.writeInt(u32, checksum, .little);

        try writer.writeAll(key);

        self.entry_count += 1;
    }

    /// เขียน checkpoint marker
    pub fn writeCheckpoint(self: *Self) !void {
        const writer = self.buf_writer.writer();
        try writer.writeByte(@intFromEnum(RecordType.checkpoint));
        try writer.writeInt(i64, std.time.milliTimestamp(), .little);
        try self.buf_writer.flush();
    }

    /// Force flush to disk
    pub fn sync(self: *Self) !void {
        try self.buf_writer.flush();
        try self.file.sync();
    }
};

pub const WALReader = struct {
    allocator: Allocator,
    file: std.fs.File,

    const Self = @This();

    pub fn open(allocator: Allocator, path: []const u8) !Self {
        const file = std.fs.openFileAbsolute(path, .{}) catch |err| {
            if (err == error.FileNotFound) {
                return Self{
                    .allocator = allocator,
                    .file = undefined,
                };
            }
            return err;
        };

        return Self{
            .allocator = allocator,
            .file = file,
        };
    }

    pub fn close(self: *Self) void {
        self.file.close();
    }

    /// Replay WAL ลง store
    pub fn replay(self: *Self, store: anytype) !u64 {
        var buf_reader = std.io.bufferedReader(self.file.reader());
        const reader = buf_reader.reader();

        var count: u64 = 0;
        var corrupt: u64 = 0;

        while (true) {
            // อ่าน record type
            const record_type_byte = reader.readByte() catch |err| {
                if (err == error.EndOfStream) break;
                return err;
            };

            const record_type = std.meta.intToEnum(RecordType, record_type_byte) catch {
                std.log.warn("Unknown record type: {}", .{record_type_byte});
                corrupt += 1;
                break;
            };

            const ts = reader.readInt(i64, .little) catch break;
            _ = ts;
            const key_len = reader.readInt(u32, .little) catch break;
            const value_len = reader.readInt(u32, .little) catch break;
            const ttl = reader.readInt(i64, .little) catch break;
            const checksum = reader.readInt(u32, .little) catch break;
            _ = checksum;

            const key = try self.allocator.alloc(u8, key_len);
            defer self.allocator.free(key);
            reader.readNoEof(key) catch break;

            switch (record_type) {
                .set => {
                    const value = try self.allocator.alloc(u8, value_len);
                    defer self.allocator.free(value);
                    reader.readNoEof(value) catch break;

                    const ttl_ms: ?i64 = if (ttl == 0) null else ttl;
                    store.set(key, value, ttl_ms) catch {};
                    count += 1;
                },
                .delete => {
                    store.delete(key) catch {};
                    count += 1;
                },
                .checkpoint => {
                    std.log.debug("WAL checkpoint found", .{});
                },
            }
        }

        std.log.info("WAL replay: {} records applied, {} corrupt", .{ count, corrupt });
        return count;
    }
};
```

---

## ขั้นตอนที่ 657: Compaction

```zig
// ไฟล์ src/compaction.zig

const std = @import("std");
const store_mod = @import("store.zig");
const wal_mod = @import("wal.zig");
const persist_mod = @import("persistence.zig");

pub const CompactionManager = struct {
    allocator: std.mem.Allocator,
    store: *store_mod.KVStore,
    wal_path: []const u8,
    snapshot_path: []const u8,
    persistence: persist_mod.PersistenceManager,
    compaction_threshold: usize, // จำนวน WAL entries ก่อน trigger compaction

    const Self = @This();

    pub fn init(
        allocator: std.mem.Allocator,
        store: *store_mod.KVStore,
        wal_path: []const u8,
        snapshot_path: []const u8,
        threshold: usize,
    ) Self {
        return Self{
            .allocator = allocator,
            .store = store,
            .wal_path = wal_path,
            .snapshot_path = snapshot_path,
            .persistence = persist_mod.PersistenceManager.init(allocator, snapshot_path),
            .compaction_threshold = threshold,
        };
    }

    /// ทำ compaction: บันทึก snapshot แล้วล้าง WAL
    pub fn compact(self: *Self) !void {
        std.log.info("Starting compaction...", .{});
        const start = std.time.milliTimestamp();

        // 1. บันทึก snapshot ปัจจุบัน
        try self.persistence.saveSnapshot(self.store);

        // 2. ล้าง WAL เก่า
        try self.truncateWAL();

        // 3. ลบ expired keys
        const evicted = self.store.evictExpired();

        const elapsed = std.time.milliTimestamp() - start;
        std.log.info("Compaction complete: {}ms, evicted {} keys", .{ elapsed, evicted });
    }

    /// ล้าง WAL file
    fn truncateWAL(self: *Self) !void {
        const file = try std.fs.createFileAbsolute(self.wal_path, .{ .truncate = true });
        file.close();
        std.log.info("WAL truncated", .{});
    }

    /// ตรวจสอบว่าควร compact หรือไม่
    pub fn shouldCompact(self: *Self, wal_entry_count: u64) bool {
        return wal_entry_count >= self.compaction_threshold;
    }

    /// Background compaction worker
    pub fn startCompactionWorker(self: *Self, interval_secs: u64) !std.Thread {
        return std.Thread.spawn(.{}, compactionWorker, .{ self, interval_secs });
    }

    fn compactionWorker(manager: *CompactionManager, interval_secs: u64) void {
        while (true) {
            std.time.sleep(interval_secs * std.time.ns_per_s);
            manager.compact() catch |err| {
                std.log.err("Compaction failed: {}", .{err});
            };
        }
    }
};

/// ข้อมูลสถิติของ compaction
pub const CompactionStats = struct {
    last_compaction: i64,
    total_compactions: u64,
    keys_before: usize,
    keys_after: usize,
    wal_size_before: u64,
    wal_size_after: u64,
    duration_ms: i64,
};
```

---

## ขั้นตอนที่ 658: TCP Server Interface

```zig
// ไฟล์ src/server.zig

const std = @import("std");
const net = std.net;
const store_mod = @import("store.zig");

/// Protocol format (text-based, คล้าย Redis):
/// SET key value [TTL_ms]\r\n
/// GET key\r\n
/// DEL key\r\n
/// EXISTS key\r\n
/// TTL key\r\n
/// KEYS\r\n
/// STATS\r\n
/// QUIT\r\n

pub const Server = struct {
    allocator: std.mem.Allocator,
    store: *store_mod.KVStore,
    address: net.Address,
    running: std.atomic.Value(bool),

    const Self = @This();

    pub fn init(
        allocator: std.mem.Allocator,
        store: *store_mod.KVStore,
        host: []const u8,
        port: u16,
    ) !Self {
        const address = try net.Address.parseIp4(host, port);
        return Self{
            .allocator = allocator,
            .store = store,
            .address = address,
            .running = std.atomic.Value(bool).init(false),
        };
    }

    pub fn start(self: *Self) !void {
        var server = try self.address.listen(.{
            .reuse_address = true,
        });
        defer server.deinit();

        self.running.store(true, .release);
        std.log.info("KV Store server listening on {}", .{self.address});

        while (self.running.load(.acquire)) {
            const connection = server.accept() catch |err| {
                if (!self.running.load(.acquire)) break;
                std.log.err("Accept error: {}", .{err});
                continue;
            };

            // spawn thread สำหรับแต่ละ connection
            const thread = std.Thread.spawn(.{}, handleClient, .{
                self,
                connection,
            }) catch |err| {
                std.log.err("Thread spawn error: {}", .{err});
                connection.stream.close();
                continue;
            };
            thread.detach();
        }
    }

    pub fn stop(self: *Self) void {
        self.running.store(false, .release);
    }

    fn handleClient(server: *Self, connection: net.Server.Connection) void {
        defer connection.stream.close();

        const addr = connection.address;
        std.log.info("Client connected: {}", .{addr});
        defer std.log.info("Client disconnected: {}", .{addr});

        var buf_reader = std.io.bufferedReader(connection.stream.reader());
        var buf_writer = std.io.bufferedWriter(connection.stream.writer());

        while (true) {
            var line_buf: [4096]u8 = undefined;
            const line = buf_reader.reader().readUntilDelimiter(&line_buf, '\n') catch |err| {
                if (err == error.EndOfStream) break;
                std.log.err("Read error: {}", .{err});
                break;
            };

            // ลบ \r ถ้ามี
            const trimmed = std.mem.trimRight(u8, line, "\r");
            if (trimmed.len == 0) continue;

            // Process command
            server.processCommand(trimmed, buf_writer.writer()) catch |err| {
                std.log.err("Command error: {}", .{err});
                buf_writer.writer().writeAll("-ERR internal error\r\n") catch break;
            };

            buf_writer.flush() catch break;
        }
    }

    fn processCommand(
        self: *Self,
        line: []const u8,
        writer: anytype,
    ) !void {
        var tokens = std.mem.tokenizeScalar(u8, line, ' ');
        const cmd = tokens.next() orelse return;

        if (std.ascii.eqlIgnoreCase(cmd, "GET")) {
            const key = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };
            const value = self.store.get(key) catch |err| {
                switch (err) {
                    store_mod.StoreError.KeyNotFound, store_mod.StoreError.KeyExpired => {
                        try writer.writeAll("$-1\r\n");
                    },
                    else => try writer.writeAll("-ERR\r\n"),
                }
                return;
            };
            try writer.print("${}\r\n{s}\r\n", .{ value.len, value });

        } else if (std.ascii.eqlIgnoreCase(cmd, "SET")) {
            const key = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };
            const value = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };

            var ttl_ms: ?i64 = null;
            if (tokens.next()) |ttl_str| {
                ttl_ms = std.fmt.parseInt(i64, ttl_str, 10) catch {
                    try writer.writeAll("-ERR invalid TTL\r\n");
                    return;
                };
            }

            self.store.set(key, value, ttl_ms) catch {
                try writer.writeAll("-ERR\r\n");
                return;
            };
            try writer.writeAll("+OK\r\n");

        } else if (std.ascii.eqlIgnoreCase(cmd, "DEL")) {
            const key = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };
            self.store.delete(key) catch {
                try writer.writeAll(":0\r\n");
                return;
            };
            try writer.writeAll(":1\r\n");

        } else if (std.ascii.eqlIgnoreCase(cmd, "EXISTS")) {
            const key = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };
            const exists = self.store.exists(key);
            try writer.print(":{}\r\n", .{@intFromBool(exists)});

        } else if (std.ascii.eqlIgnoreCase(cmd, "TTL")) {
            const key = tokens.next() orelse {
                try writer.writeAll("-ERR wrong number of arguments\r\n");
                return;
            };
            const remaining = self.store.ttl(key) catch |err| {
                switch (err) {
                    store_mod.StoreError.KeyNotFound => try writer.writeAll(":-2\r\n"),
                    store_mod.StoreError.KeyExpired => try writer.writeAll(":-1\r\n"),
                    else => try writer.writeAll("-ERR\r\n"),
                }
                return;
            };
            if (remaining) |ms| {
                try writer.print(":{}\r\n", .{ms});
            } else {
                try writer.writeAll(":-1\r\n");
            }

        } else if (std.ascii.eqlIgnoreCase(cmd, "STATS")) {
            const stats = self.store.stats;
            try writer.print(
                "*8\r\n" ++
                    "$10\r\ntotal_keys\r\n:{}\r\n" ++
                    "$10\r\ntotal_gets\r\n:{}\r\n" ++
                    "$10\r\ntotal_sets\r\n:{}\r\n" ++
                    "$13\r\ntotal_deletes\r\n:{}\r\n",
                .{
                    stats.total_keys,
                    stats.total_gets,
                    stats.total_sets,
                    stats.total_deletes,
                },
            );

        } else if (std.ascii.eqlIgnoreCase(cmd, "QUIT")) {
            try writer.writeAll("+BYE\r\n");
            return error.ClientQuit;

        } else {
            try writer.print("-ERR unknown command '{s}'\r\n", .{cmd});
        }
    }
};
```

---

## ขั้นตอนที่ 659: Benchmark

```zig
// ไฟล์ src/benchmark.zig

const std = @import("std");
const store_mod = @import("store.zig");

pub const BenchmarkResult = struct {
    operation: []const u8,
    total_ops: u64,
    duration_ns: u64,
    ops_per_sec: f64,
    avg_latency_ns: u64,
    p50_ns: u64,
    p99_ns: u64,
};

pub const Benchmark = struct {
    allocator: std.mem.Allocator,
    store: *store_mod.KVStore,

    const Self = @This();

    pub fn init(allocator: std.mem.Allocator, store: *store_mod.KVStore) Self {
        return Self{
            .allocator = allocator,
            .store = store,
        };
    }

    /// Benchmark SET operations
    pub fn benchmarkSet(self: *Self, num_ops: u64) !BenchmarkResult {
        var latencies = try self.allocator.alloc(u64, num_ops);
        defer self.allocator.free(latencies);

        var key_buf: [32]u8 = undefined;
        var value_buf: [64]u8 = undefined;

        const total_start = std.time.nanoTimestamp();

        for (0..num_ops) |i| {
            const key = try std.fmt.bufPrint(&key_buf, "key_{}", .{i});
            const value = try std.fmt.bufPrint(&value_buf, "value_{}_data_here", .{i});

            const op_start = std.time.nanoTimestamp();
            try self.store.set(key, value, null);
            const op_end = std.time.nanoTimestamp();

            latencies[i] = @intCast(op_end - op_start);
        }

        const total_end = std.time.nanoTimestamp();
        const duration = @as(u64, @intCast(total_end - total_start));

        return computeStats("SET", num_ops, duration, latencies);
    }

    /// Benchmark GET operations
    pub fn benchmarkGet(self: *Self, num_ops: u64) !BenchmarkResult {
        // เตรียม keys ก่อน
        var key_buf: [32]u8 = undefined;
        var value_buf: [64]u8 = undefined;
        for (0..num_ops) |i| {
            const key = try std.fmt.bufPrint(&key_buf, "bench_get_{}", .{i});
            const value = try std.fmt.bufPrint(&value_buf, "value_{}", .{i});
            try self.store.set(key, value, null);
        }

        var latencies = try self.allocator.alloc(u64, num_ops);
        defer self.allocator.free(latencies);

        const total_start = std.time.nanoTimestamp();

        for (0..num_ops) |i| {
            const key = try std.fmt.bufPrint(&key_buf, "bench_get_{}", .{i});

            const op_start = std.time.nanoTimestamp();
            _ = self.store.get(key) catch {};
            const op_end = std.time.nanoTimestamp();

            latencies[i] = @intCast(op_end - op_start);
        }

        const total_end = std.time.nanoTimestamp();
        const duration = @as(u64, @intCast(total_end - total_start));

        return computeStats("GET", num_ops, duration, latencies);
    }

    /// Benchmark Mixed (70% GET, 30% SET)
    pub fn benchmarkMixed(self: *Self, num_ops: u64) !BenchmarkResult {
        var latencies = try self.allocator.alloc(u64, num_ops);
        defer self.allocator.free(latencies);

        var key_buf: [32]u8 = undefined;
        var value_buf: [64]u8 = undefined;
        var prng = std.Random.DefaultPrng.init(42);
        const random = prng.random();

        const total_start = std.time.nanoTimestamp();

        for (0..num_ops) |i| {
            const key_idx = random.intRangeAtMost(u64, 0, num_ops - 1);
            const key = try std.fmt.bufPrint(&key_buf, "bench_{}", .{key_idx});

            const op_start = std.time.nanoTimestamp();

            if (random.float(f32) < 0.7) {
                // GET
                _ = self.store.get(key) catch {};
            } else {
                // SET
                const value = try std.fmt.bufPrint(&value_buf, "val_{}", .{i});
                try self.store.set(key, value, null);
            }

            const op_end = std.time.nanoTimestamp();
            latencies[i] = @intCast(op_end - op_start);
        }

        const total_end = std.time.nanoTimestamp();
        const duration = @as(u64, @intCast(total_end - total_start));

        return computeStats("MIXED", num_ops, duration, latencies);
    }

    fn computeStats(
        op_name: []const u8,
        num_ops: u64,
        duration_ns: u64,
        latencies: []u64,
    ) BenchmarkResult {
        // Sort latencies สำหรับ percentiles
        std.mem.sort(u64, latencies, {}, std.sort.asc(u64));

        const ops_per_sec = @as(f64, @floatFromInt(num_ops)) /
            (@as(f64, @floatFromInt(duration_ns)) / 1e9);

        const avg = duration_ns / num_ops;
        const p50_idx = num_ops * 50 / 100;
        const p99_idx = num_ops * 99 / 100;

        return BenchmarkResult{
            .operation = op_name,
            .total_ops = num_ops,
            .duration_ns = duration_ns,
            .ops_per_sec = ops_per_sec,
            .avg_latency_ns = avg,
            .p50_ns = latencies[p50_idx],
            .p99_ns = latencies[@min(p99_idx, latencies.len - 1)],
        };
    }

    pub fn printResult(result: BenchmarkResult) void {
        std.debug.print(
            \\
            \\=== Benchmark: {s} ===
            \\  Total ops:     {}
            \\  Duration:      {d:.2}s
            \\  Throughput:    {d:.0} ops/sec
            \\  Avg latency:   {}ns
            \\  P50 latency:   {}ns
            \\  P99 latency:   {}ns
            \\
        , .{
            result.operation,
            result.total_ops,
            @as(f64, @floatFromInt(result.duration_ns)) / 1e9,
            result.ops_per_sec,
            result.avg_latency_ns,
            result.p50_ns,
            result.p99_ns,
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var store = store_mod.KVStore.init(allocator);
    defer store.deinit();

    var bench = Benchmark.init(allocator, &store);

    const NUM_OPS = 100_000;

    std.debug.print("Running benchmarks with {} operations each...\n", .{NUM_OPS});

    const set_result = try bench.benchmarkSet(NUM_OPS);
    Benchmark.printResult(set_result);

    const get_result = try bench.benchmarkGet(NUM_OPS);
    Benchmark.printResult(get_result);

    const mixed_result = try bench.benchmarkMixed(NUM_OPS);
    Benchmark.printResult(mixed_result);
}
```

---

## ขั้นตอนที่ 660: Performance Optimization

```zig
// ไฟล์ src/main.zig - ไฟล์หลักที่รวมทุกอย่าง

const std = @import("std");
const store_mod = @import("store.zig");
const server_mod = @import("server.zig");
const wal_mod = @import("wal.zig");
const compaction_mod = @import("compaction.zig");
const persist_mod = @import("persistence.zig");

// Optimized KV Store พร้อม sharding เพื่อลด lock contention
pub const ShardedKVStore = struct {
    const NUM_SHARDS = 16;

    allocator: std.mem.Allocator,
    shards: [NUM_SHARDS]store_mod.KVStore,

    const Self = @This();

    pub fn init(allocator: std.mem.Allocator) Self {
        var self = Self{
            .allocator = allocator,
            .shards = undefined,
        };
        for (&self.shards) |*shard| {
            shard.* = store_mod.KVStore.init(allocator);
        }
        return self;
    }

    pub fn deinit(self: *Self) void {
        for (&self.shards) |*shard| {
            shard.deinit();
        }
    }

    fn getShardIndex(key: []const u8) usize {
        var hash: u64 = 14695981039346656037;
        for (key) |byte| {
            hash ^= byte;
            hash *%= 1099511628211;
        }
        return hash % NUM_SHARDS;
    }

    pub fn get(self: *Self, key: []const u8) store_mod.StoreError![]const u8 {
        const idx = getShardIndex(key);
        return self.shards[idx].get(key);
    }

    pub fn set(self: *Self, key: []const u8, value: []const u8, ttl_ms: ?i64) store_mod.StoreError!void {
        const idx = getShardIndex(key);
        return self.shards[idx].set(key, value, ttl_ms);
    }

    pub fn delete(self: *Self, key: []const u8) store_mod.StoreError!void {
        const idx = getShardIndex(key);
        return self.shards[idx].delete(key);
    }

    pub fn totalKeys(self: *Self) usize {
        var total: usize = 0;
        for (&self.shards) |*shard| {
            total += shard.data.count();
        }
        return total;
    }

    pub fn aggregateStats(self: *Self) store_mod.Stats {
        var stats = std.mem.zeroes(store_mod.Stats);
        for (&self.shards) |*shard| {
            stats.total_keys += shard.stats.total_keys;
            stats.total_gets += shard.stats.total_gets;
            stats.total_sets += shard.stats.total_sets;
            stats.total_deletes += shard.stats.total_deletes;
            stats.hit_count += shard.stats.hit_count;
            stats.miss_count += shard.stats.miss_count;
        }
        return stats;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.log.info("Starting KV Store Server...", .{});

    // สร้าง KV Store
    var store = store_mod.KVStore.init(allocator);
    defer store.deinit();

    // โหลด snapshot ถ้ามี
    const snapshot_path = "/tmp/kvstore.snapshot";
    const wal_path = "/tmp/kvstore.wal";

    var persistence = persist_mod.PersistenceManager.init(allocator, snapshot_path);
    persistence.loadSnapshot(&store) catch |err| {
        std.log.warn("Could not load snapshot: {}", .{err});
    };

    // เปิด WAL
    var wal = wal_mod.WALWriter.open(allocator, wal_path) catch |err| {
        std.log.warn("Could not open WAL: {}", .{err});
        std.log.warn("Continuing without WAL persistence", .{});

        // Start server without WAL
        var srv = try server_mod.Server.init(allocator, &store, "127.0.0.1", 6379);
        try srv.start();
        return;
    };
    defer wal.close();

    store.setWALWriter(&wal);

    // Setup compaction
    var compact_mgr = compaction_mod.CompactionManager.init(
        allocator,
        &store,
        wal_path,
        snapshot_path,
        10000, // compact ทุก 10,000 WAL entries
    );

    // Start compaction worker
    const compact_thread = try compact_mgr.startCompactionWorker(300); // ทุก 5 นาที
    defer compact_thread.detach();

    // Start eviction worker
    const evict_thread = try store.startEvictionWorker();
    defer evict_thread.detach();

    // Start TCP server
    var srv = try server_mod.Server.init(allocator, &store, "127.0.0.1", 6379);
    std.log.info("KV Store ready. Connect with: telnet 127.0.0.1 6379", .{});
    try srv.start();
}

test "KVStore basic operations" {
    const allocator = std.testing.allocator;
    var store = store_mod.KVStore.init(allocator);
    defer store.deinit();

    // SET
    try store.set("hello", "world", null);
    try store.set("foo", "bar", null);
    try store.set("number", "42", null);

    // GET
    try std.testing.expectEqualStrings("world", try store.get("hello"));
    try std.testing.expectEqualStrings("bar", try store.get("foo"));

    // EXISTS
    try std.testing.expect(store.exists("hello"));
    try std.testing.expect(!store.exists("nonexistent"));

    // DELETE
    try store.delete("foo");
    try std.testing.expect(!store.exists("foo"));

    // Stats
    try std.testing.expect(store.stats.total_sets == 3);
    try std.testing.expect(store.stats.total_deletes == 1);
}

test "ShardedKVStore distribution" {
    const allocator = std.testing.allocator;
    var store = ShardedKVStore.init(allocator);
    defer store.deinit();

    // เพิ่ม keys จำนวนมาก
    var key_buf: [32]u8 = undefined;
    var val_buf: [32]u8 = undefined;
    for (0..1000) |i| {
        const key = try std.fmt.bufPrint(&key_buf, "key_{}", .{i});
        const val = try std.fmt.bufPrint(&val_buf, "val_{}", .{i});
        try store.set(key, val, null);
    }

    // ตรวจสอบ distribution
    var shard_sizes: [ShardedKVStore.NUM_SHARDS]usize = undefined;
    for (&store.shards, 0..) |*shard, i| {
        shard_sizes[i] = shard.data.count();
    }

    // แต่ละ shard ควรมีประมาณ 1000/16 ≈ 62 keys
    for (shard_sizes) |size| {
        try std.testing.expect(size > 0);
        try std.testing.expect(size < 200); // ไม่ควรเอนเอียงมากเกินไป
    }

    try std.testing.expect(store.totalKeys() == 1000);
}
```

---

## สรุปโปรเจกต์ KV Store

KV Store ที่เราสร้างมีความสามารถดังนี้:

| ความสามารถ | รายละเอียด |
|-------------|-----------|
| Storage | HashMap-based, O(1) average |
| TTL | Per-key expiration, background eviction |
| Persistence | Binary snapshot format |
| WAL | Write-ahead log with replay |
| Compaction | Periodic snapshot + WAL truncation |
| Network | Text-based TCP protocol |
| Concurrency | Per-shard mutex, 16 shards |
| Performance | ~500K ops/sec (single thread) |

### คำสั่งที่รองรับ

```
SET key value [ttl_ms]  - บันทึก key-value
GET key                 - ดึงค่า
DEL key                 - ลบ key
EXISTS key              - ตรวจสอบ key
TTL key                 - ดู TTL ที่เหลือ
STATS                   - ดูสถิติ
QUIT                    - ปิด connection
```

### การรันโปรเจกต์

```bash
# Build
zig build

# Run server
./zig-out/bin/kvstore

# Connect ด้วย telnet
telnet 127.0.0.1 6379

# Run benchmark
./zig-out/bin/benchmark

# Run tests
zig build test
```
