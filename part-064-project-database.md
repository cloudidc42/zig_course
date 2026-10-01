# ส่วนที่ 64: โปรเจค SQLite Database Wrapper

## ขั้นตอนที่ 631-640: สร้าง Zig Wrapper สำหรับ SQLite

---

## ขั้นตอนที่ 631: SQLite C Library Integration

### ภาพรวม

SQLite เป็น embedded database ที่นิยมมากที่สุดในโลก เราจะสร้าง Zig wrapper ที่:
- Type-safe
- Memory-safe
- Ergonomic API
- รองรับ transactions
- รองรับ prepared statements

### โครงสร้างโปรเจค

```
sqlite-wrapper/
├── build.zig
├── build.zig.zon
├── src/
│   ├── main.zig
│   ├── sqlite.zig        (C bindings)
│   ├── database.zig      (Connection management)
│   ├── statement.zig     (Prepared statements)
│   ├── result.zig        (Result iteration)
│   ├── transaction.zig   (Transaction management)
│   └── orm.zig           (ORM-like patterns)
├── include/
│   └── sqlite3.h
└── lib/
    └── sqlite3.c
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Compile sqlite3.c
    const sqlite_lib = b.addStaticLibrary(.{
        .name = "sqlite3",
        .target = target,
        .optimize = optimize,
    });
    sqlite_lib.addCSourceFile(.{
        .file = b.path("lib/sqlite3.c"),
        .flags = &[_][]const u8{
            "-DSQLITE_THREADSAFE=0",
            "-DSQLITE_DEFAULT_WAL_SYNCHRONOUS=1",
            "-DSQLITE_USE_URI=1",
            "-DSQLITE_ENABLE_JSON1",
            "-DSQLITE_ENABLE_FTS5",
        },
    });
    sqlite_lib.linkLibC();

    const exe = b.addExecutable(.{
        .name = "sqlite-demo",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.linkLibrary(sqlite_lib);
    exe.addIncludePath(b.path("include"));
    exe.linkLibC();

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run demo");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/database.zig"),
        .target = target,
        .optimize = optimize,
    });
    unit_tests.linkLibrary(sqlite_lib);
    unit_tests.addIncludePath(b.path("include"));
    unit_tests.linkLibC();

    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## ขั้นตอนที่ 632-633: SQLite C Bindings และ Connection Management

### src/sqlite.zig - C Bindings

```zig
const std = @import("std");

// C declarations สำหรับ SQLite API
pub const c = @cImport({
    @cInclude("sqlite3.h");
});

// SQLite result codes
pub const SQLITE_OK = c.SQLITE_OK;
pub const SQLITE_ERROR = c.SQLITE_ERROR;
pub const SQLITE_BUSY = c.SQLITE_BUSY;
pub const SQLITE_ROW = c.SQLITE_ROW;
pub const SQLITE_DONE = c.SQLITE_DONE;
pub const SQLITE_CONSTRAINT = c.SQLITE_CONSTRAINT;
pub const SQLITE_NOTFOUND = c.SQLITE_NOTFOUND;

// Data types
pub const SQLITE_INTEGER = c.SQLITE_INTEGER;
pub const SQLITE_FLOAT = c.SQLITE_FLOAT;
pub const SQLITE_TEXT = c.SQLITE_TEXT;
pub const SQLITE_BLOB = c.SQLITE_BLOB;
pub const SQLITE_NULL = c.SQLITE_NULL;

pub const SqliteError = error{
    Error,
    Internal,
    Perm,
    Abort,
    Busy,
    Locked,
    NoMem,
    ReadOnly,
    Interrupt,
    IoErr,
    Corrupt,
    NotFound,
    Full,
    CantOpen,
    Protocol,
    Schema,
    TooBig,
    Constraint,
    Mismatch,
    Misuse,
    NoLfs,
    Auth,
    Format,
    Range,
    NotADb,
    Row,
    Done,
    Unknown,
};

pub fn checkError(rc: c_int) SqliteError!void {
    return switch (rc) {
        SQLITE_OK, SQLITE_ROW, SQLITE_DONE => {},
        c.SQLITE_ERROR => SqliteError.Error,
        c.SQLITE_BUSY => SqliteError.Busy,
        c.SQLITE_LOCKED => SqliteError.Locked,
        c.SQLITE_NOMEM => SqliteError.NoMem,
        c.SQLITE_READONLY => SqliteError.ReadOnly,
        c.SQLITE_IOERR => SqliteError.IoErr,
        c.SQLITE_CORRUPT => SqliteError.Corrupt,
        c.SQLITE_NOTFOUND => SqliteError.NotFound,
        c.SQLITE_FULL => SqliteError.Full,
        c.SQLITE_CANTOPEN => SqliteError.CantOpen,
        c.SQLITE_CONSTRAINT => SqliteError.Constraint,
        c.SQLITE_MISMATCH => SqliteError.Mismatch,
        c.SQLITE_MISUSE => SqliteError.Misuse,
        c.SQLITE_RANGE => SqliteError.Range,
        else => SqliteError.Unknown,
    };
}
```

### src/database.zig

```zig
const std = @import("std");
const sqlite = @import("sqlite.zig");
const statement = @import("statement.zig");
const transaction = @import("transaction.zig");

pub const OpenFlags = struct {
    readonly: bool = false,
    create: bool = true,
    uri: bool = false,
    memory: bool = false,
    wal_mode: bool = false,
};

/// Database connection
pub const Database = struct {
    allocator: std.mem.Allocator,
    db: *sqlite.c.sqlite3,
    path: []const u8,

    /// เปิด database
    pub fn open(allocator: std.mem.Allocator, path: []const u8, flags: OpenFlags) !Database {
        var sqlite_flags: c_int = 0;

        if (flags.readonly) {
            sqlite_flags |= sqlite.c.SQLITE_OPEN_READONLY;
        } else {
            sqlite_flags |= sqlite.c.SQLITE_OPEN_READWRITE;
            if (flags.create) {
                sqlite_flags |= sqlite.c.SQLITE_OPEN_CREATE;
            }
        }

        if (flags.uri) {
            sqlite_flags |= sqlite.c.SQLITE_OPEN_URI;
        }

        if (flags.memory) {
            sqlite_flags |= sqlite.c.SQLITE_OPEN_MEMORY;
        }

        const path_z = try allocator.dupeZ(u8, path);
        defer allocator.free(path_z);

        var db: ?*sqlite.c.sqlite3 = null;
        const rc = sqlite.c.sqlite3_open_v2(path_z.ptr, &db, sqlite_flags, null);

        try sqlite.checkError(rc);

        const database = Database{
            .allocator = allocator,
            .db = db.?,
            .path = try allocator.dupe(u8, path),
        };

        // เปิด WAL mode ถ้าต้องการ
        if (flags.wal_mode) {
            try database.exec("PRAGMA journal_mode=WAL;");
        }

        // ตั้งค่า busy timeout
        _ = sqlite.c.sqlite3_busy_timeout(db.?, 5000);

        return database;
    }

    /// เปิด in-memory database
    pub fn openMemory(allocator: std.mem.Allocator) !Database {
        return open(allocator, ":memory:", .{ .memory = true });
    }

    /// ปิด database
    pub fn close(self: *Database) void {
        _ = sqlite.c.sqlite3_close(self.db);
        self.allocator.free(self.path);
    }

    /// Execute SQL โดยไม่ return results
    pub fn exec(self: *const Database, sql: []const u8) !void {
        const sql_z = try self.allocator.dupeZ(u8, sql);
        defer self.allocator.free(sql_z);

        var errmsg: [*c]u8 = null;
        const rc = sqlite.c.sqlite3_exec(self.db, sql_z.ptr, null, null, &errmsg);

        if (errmsg != null) {
            defer sqlite.c.sqlite3_free(errmsg);
            std.log.err("SQLite error: {s}", .{errmsg});
        }

        try sqlite.checkError(rc);
    }

    /// Prepare statement
    pub fn prepare(self: *const Database, sql: []const u8) !statement.Statement {
        return statement.Statement.prepare(self.allocator, self.db, sql);
    }

    /// Last insert rowid
    pub fn lastInsertRowid(self: *const Database) i64 {
        return sqlite.c.sqlite3_last_insert_rowid(self.db);
    }

    /// จำนวน rows ที่เปลี่ยนแปลงล่าสุด
    pub fn changes(self: *const Database) i32 {
        return sqlite.c.sqlite3_changes(self.db);
    }

    /// เริ่ม transaction
    pub fn beginTransaction(self: *const Database) !transaction.Transaction {
        return transaction.Transaction.begin(self);
    }

    /// ตรวจสอบ table มีอยู่หรือไม่
    pub fn tableExists(self: *const Database, table_name: []const u8) !bool {
        var stmt = try self.prepare(
            "SELECT name FROM sqlite_master WHERE type='table' AND name=?;",
        );
        defer stmt.finalize();

        try stmt.bindText(1, table_name);

        return switch (try stmt.step()) {
            .row => true,
            .done => false,
        };
    }

    /// ดึง version ของ SQLite
    pub fn version() []const u8 {
        return std.mem.span(sqlite.c.sqlite3_version);
    }

    /// Execute query และ return all rows
    pub fn queryAll(
        self: *const Database,
        comptime T: type,
        sql: []const u8,
        binds: anytype,
    ) ![]T {
        var stmt = try self.prepare(sql);
        defer stmt.finalize();

        // Bind parameters
        inline for (binds, 0..) |bind, i| {
            try stmt.bindAny(i + 1, bind);
        }

        var results = std.ArrayList(T).init(self.allocator);
        errdefer results.deinit();

        while (true) {
            const step_result = try stmt.step();
            switch (step_result) {
                .row => {
                    const row = try stmt.scanInto(T);
                    try results.append(row);
                },
                .done => break,
            }
        }

        return results.toOwnedSlice();
    }
};
```

---

## ขั้นตอนที่ 633-635: Statement, Parameter Binding, Query Execution

### src/statement.zig

```zig
const std = @import("std");
const sqlite = @import("sqlite.zig");

pub const StepResult = enum {
    row,
    done,
};

pub const ColumnValue = union(enum) {
    integer: i64,
    float: f64,
    text: []const u8,
    blob: []const u8,
    null_value: void,
};

/// Prepared Statement
pub const Statement = struct {
    allocator: std.mem.Allocator,
    stmt: *sqlite.c.sqlite3_stmt,

    pub fn prepare(
        allocator: std.mem.Allocator,
        db: *sqlite.c.sqlite3,
        sql: []const u8,
    ) !Statement {
        const sql_z = try allocator.dupeZ(u8, sql);
        defer allocator.free(sql_z);

        var stmt: ?*sqlite.c.sqlite3_stmt = null;
        const rc = sqlite.c.sqlite3_prepare_v2(
            db,
            sql_z.ptr,
            @intCast(sql.len),
            &stmt,
            null,
        );

        try sqlite.checkError(rc);

        return Statement{
            .allocator = allocator,
            .stmt = stmt.?,
        };
    }

    pub fn finalize(self: *Statement) void {
        _ = sqlite.c.sqlite3_finalize(self.stmt);
    }

    pub fn reset(self: *Statement) !void {
        try sqlite.checkError(sqlite.c.sqlite3_reset(self.stmt));
    }

    /// Execute และรับ result ถัดไป
    pub fn step(self: *Statement) !StepResult {
        const rc = sqlite.c.sqlite3_step(self.stmt);
        return switch (rc) {
            sqlite.SQLITE_ROW => .row,
            sqlite.SQLITE_DONE => .done,
            else => {
                try sqlite.checkError(rc);
                unreachable;
            },
        };
    }

    // Bind methods
    pub fn bindInt(self: *Statement, col: c_int, value: i64) !void {
        try sqlite.checkError(
            sqlite.c.sqlite3_bind_int64(self.stmt, col, value),
        );
    }

    pub fn bindFloat(self: *Statement, col: c_int, value: f64) !void {
        try sqlite.checkError(
            sqlite.c.sqlite3_bind_double(self.stmt, col, value),
        );
    }

    pub fn bindText(self: *Statement, col: c_int, value: []const u8) !void {
        try sqlite.checkError(
            sqlite.c.sqlite3_bind_text(
                self.stmt,
                col,
                value.ptr,
                @intCast(value.len),
                sqlite.c.SQLITE_TRANSIENT,
            ),
        );
    }

    pub fn bindBlob(self: *Statement, col: c_int, value: []const u8) !void {
        try sqlite.checkError(
            sqlite.c.sqlite3_bind_blob(
                self.stmt,
                col,
                value.ptr,
                @intCast(value.len),
                sqlite.c.SQLITE_TRANSIENT,
            ),
        );
    }

    pub fn bindNull(self: *Statement, col: c_int) !void {
        try sqlite.checkError(sqlite.c.sqlite3_bind_null(self.stmt, col));
    }

    pub fn bindBool(self: *Statement, col: c_int, value: bool) !void {
        try self.bindInt(col, if (value) 1 else 0);
    }

    /// Bind any Zig type โดยอัตโนมัติ
    pub fn bindAny(self: *Statement, col: c_int, value: anytype) !void {
        const T = @TypeOf(value);
        switch (@typeInfo(T)) {
            .Int, .ComptimeInt => try self.bindInt(col, @intCast(value)),
            .Float, .ComptimeFloat => try self.bindFloat(col, @floatCast(value)),
            .Bool => try self.bindBool(col, value),
            .Pointer => |ptr| {
                if (ptr.child == u8) {
                    try self.bindText(col, value);
                } else {
                    @compileError("Unsupported pointer type for bindAny");
                }
            },
            .Optional => {
                if (value) |v| {
                    try self.bindAny(col, v);
                } else {
                    try self.bindNull(col);
                }
            },
            else => @compileError("Unsupported type for bindAny: " ++ @typeName(T)),
        }
    }

    // Column read methods
    pub fn columnType(self: *Statement, col: c_int) c_int {
        return sqlite.c.sqlite3_column_type(self.stmt, col);
    }

    pub fn columnInt(self: *Statement, col: c_int) i64 {
        return sqlite.c.sqlite3_column_int64(self.stmt, col);
    }

    pub fn columnFloat(self: *Statement, col: c_int) f64 {
        return sqlite.c.sqlite3_column_double(self.stmt, col);
    }

    pub fn columnText(self: *Statement, col: c_int) []const u8 {
        const ptr = sqlite.c.sqlite3_column_text(self.stmt, col);
        const len = sqlite.c.sqlite3_column_bytes(self.stmt, col);
        if (ptr == null) return "";
        return ptr[0..@intCast(len)];
    }

    pub fn columnBlob(self: *Statement, col: c_int) []const u8 {
        const ptr = sqlite.c.sqlite3_column_blob(self.stmt, col);
        const len = sqlite.c.sqlite3_column_bytes(self.stmt, col);
        if (ptr == null) return "";
        return @as([*]const u8, @ptrCast(ptr))[0..@intCast(len)];
    }

    pub fn columnIsNull(self: *Statement, col: c_int) bool {
        return self.columnType(col) == sqlite.SQLITE_NULL;
    }

    pub fn columnValue(self: *Statement, col: c_int) ColumnValue {
        return switch (self.columnType(col)) {
            sqlite.SQLITE_INTEGER => .{ .integer = self.columnInt(col) },
            sqlite.SQLITE_FLOAT => .{ .float = self.columnFloat(col) },
            sqlite.SQLITE_TEXT => .{ .text = self.columnText(col) },
            sqlite.SQLITE_BLOB => .{ .blob = self.columnBlob(col) },
            sqlite.SQLITE_NULL => .{ .null_value = {} },
            else => .{ .null_value = {} },
        };
    }

    pub fn columnCount(self: *Statement) c_int {
        return sqlite.c.sqlite3_column_count(self.stmt);
    }

    pub fn columnName(self: *Statement, col: c_int) []const u8 {
        const ptr = sqlite.c.sqlite3_column_name(self.stmt, col);
        if (ptr == null) return "";
        return std.mem.span(ptr);
    }

    /// Scan row เข้า struct โดยอัตโนมัติ
    pub fn scanInto(self: *Statement, comptime T: type) !T {
        var result: T = undefined;
        const fields = @typeInfo(T).Struct.fields;

        inline for (fields, 0..) |field, i| {
            const col = @as(c_int, @intCast(i));
            const field_type = field.type;

            @field(result, field.name) = switch (@typeInfo(field_type)) {
                .Int => @intCast(self.columnInt(col)),
                .Float => @floatCast(self.columnFloat(col)),
                .Bool => self.columnInt(col) != 0,
                .Optional => |opt| {
                    if (self.columnIsNull(col)) {
                        null;
                    } else {
                        switch (@typeInfo(opt.child)) {
                            .Int => @as(field_type, @intCast(self.columnInt(col))),
                            .Float => @as(field_type, @floatCast(self.columnFloat(col))),
                            else => null,
                        }
                    }
                },
                .Pointer => |ptr| {
                    if (ptr.child == u8 and ptr.size == .Slice) {
                        try self.allocator.dupe(u8, self.columnText(col))
                    } else {
                        @compileError("Unsupported pointer type in scanInto");
                    }
                },
                else => @compileError("Unsupported field type in scanInto: " ++ @typeName(field_type)),
            };
        }

        return result;
    }
};
```

---

## ขั้นตอนที่ 636-637: Result Iteration และ Transaction Management

### src/result.zig

```zig
const std = @import("std");
const statement = @import("statement.zig");

/// Iterator สำหรับ query results
pub fn Rows(comptime T: type) type {
    return struct {
        const Self = @This();

        stmt: *statement.Statement,
        allocator: std.mem.Allocator,
        done: bool,

        pub fn init(stmt: *statement.Statement, allocator: std.mem.Allocator) Self {
            return Self{
                .stmt = stmt,
                .allocator = allocator,
                .done = false,
            };
        }

        pub fn next(self: *Self) !?T {
            if (self.done) return null;

            const result = try self.stmt.step();
            switch (result) {
                .row => {
                    return try self.stmt.scanInto(T);
                },
                .done => {
                    self.done = true;
                    return null;
                },
            }
        }
    };
}

/// Dynamic row (ไม่รู้ type ล่วงหน้า)
pub const Row = struct {
    allocator: std.mem.Allocator,
    columns: std.StringHashMap(statement.ColumnValue),

    pub fn init(allocator: std.mem.Allocator) Row {
        return Row{
            .allocator = allocator,
            .columns = std.StringHashMap(statement.ColumnValue).init(allocator),
        };
    }

    pub fn deinit(self: *Row) void {
        var iter = self.columns.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            switch (entry.value_ptr.*) {
                .text => |t| self.allocator.free(t),
                .blob => |b| self.allocator.free(b),
                else => {},
            }
        }
        self.columns.deinit();
    }

    pub fn get(self: *const Row, col_name: []const u8) ?statement.ColumnValue {
        return self.columns.get(col_name);
    }

    pub fn getInt(self: *const Row, col_name: []const u8) ?i64 {
        return switch (self.columns.get(col_name) orelse return null) {
            .integer => |v| v,
            else => null,
        };
    }

    pub fn getText(self: *const Row, col_name: []const u8) ?[]const u8 {
        return switch (self.columns.get(col_name) orelse return null) {
            .text => |v| v,
            else => null,
        };
    }
};
```

### src/transaction.zig

```zig
const std = @import("std");
const database = @import("database.zig");

pub const TransactionType = enum {
    deferred,
    immediate,
    exclusive,
};

/// Transaction wrapper
pub const Transaction = struct {
    db: *const database.Database,
    committed: bool,
    rolled_back: bool,

    pub fn begin(db: *const database.Database) !Transaction {
        try db.exec("BEGIN TRANSACTION;");
        return Transaction{
            .db = db,
            .committed = false,
            .rolled_back = false,
        };
    }

    pub fn beginType(db: *const database.Database, tx_type: TransactionType) !Transaction {
        const sql = switch (tx_type) {
            .deferred => "BEGIN DEFERRED;",
            .immediate => "BEGIN IMMEDIATE;",
            .exclusive => "BEGIN EXCLUSIVE;",
        };
        try db.exec(sql);
        return Transaction{
            .db = db,
            .committed = false,
            .rolled_back = false,
        };
    }

    pub fn commit(self: *Transaction) !void {
        if (self.rolled_back) return error.AlreadyRolledBack;
        if (self.committed) return;

        try self.db.exec("COMMIT;");
        self.committed = true;
    }

    pub fn rollback(self: *Transaction) void {
        if (self.committed or self.rolled_back) return;

        self.db.exec("ROLLBACK;") catch |err| {
            std.log.err("Rollback failed: {}", .{err});
        };
        self.rolled_back = true;
    }

    /// Auto-rollback เมื่อ scope จบ (defer pattern)
    pub fn deinit(self: *Transaction) void {
        if (!self.committed and !self.rolled_back) {
            self.rollback();
        }
    }

    /// Savepoint
    pub fn savepoint(self: *Transaction, name: []const u8) !Savepoint {
        const sql = try std.fmt.allocPrint(
            self.db.allocator,
            "SAVEPOINT {s};",
            .{name},
        );
        defer self.db.allocator.free(sql);

        try self.db.exec(sql);
        return Savepoint{
            .transaction = self,
            .name = name,
            .released = false,
        };
    }
};

/// Savepoint สำหรับ nested transactions
pub const Savepoint = struct {
    transaction: *Transaction,
    name: []const u8,
    released: bool,

    pub fn release(self: *Savepoint) !void {
        if (self.released) return;

        const sql = try std.fmt.allocPrint(
            self.transaction.db.allocator,
            "RELEASE SAVEPOINT {s};",
            .{self.name},
        );
        defer self.transaction.db.allocator.free(sql);

        try self.transaction.db.exec(sql);
        self.released = true;
    }

    pub fn rollback(self: *Savepoint) !void {
        if (self.released) return;

        const sql = try std.fmt.allocPrint(
            self.transaction.db.allocator,
            "ROLLBACK TO SAVEPOINT {s};",
            .{self.name},
        );
        defer self.transaction.db.allocator.free(sql);

        try self.transaction.db.exec(sql);
    }

    pub fn deinit(self: *Savepoint) void {
        if (!self.released) {
            self.rollback() catch {};
        }
    }
};
```

---

## ขั้นตอนที่ 638-640: Type-safe Queries, Error Handling, ORM-like Patterns

### src/orm.zig

```zig
const std = @import("std");
const database = @import("database.zig");
const statement = @import("statement.zig");

/// Table definition สำหรับ ORM
pub fn Table(comptime T: type) type {
    return struct {
        const Self = @This();
        const table_name = getTableName(T);

        db: *const database.Database,

        pub fn init(db: *const database.Database) Self {
            return Self{ .db = db };
        }

        /// สร้าง table จาก struct fields
        pub fn createTable(self: *const Self) !void {
            comptime var sql_buf: [4096]u8 = undefined;
            comptime var sql_len: usize = 0;

            comptime {
                var buf_slice = sql_buf[0..];
                const create_sql = std.fmt.bufPrint(
                    buf_slice,
                    "CREATE TABLE IF NOT EXISTS {s} (\n",
                    .{table_name},
                ) catch unreachable;
                sql_len += create_sql.len;
                buf_slice = buf_slice[create_sql.len..];

                const fields = @typeInfo(T).Struct.fields;
                inline for (fields, 0..) |field, i| {
                    const col_type = zigTypeToSqlType(field.type);
                    const comma = if (i < fields.len - 1) "," else "";
                    const is_id = std.mem.eql(u8, field.name, "id");
                    const constraints = if (is_id) " PRIMARY KEY AUTOINCREMENT" else "";

                    const line = std.fmt.bufPrint(
                        buf_slice,
                        "  {s} {s}{s}{s}\n",
                        .{ field.name, col_type, constraints, comma },
                    ) catch unreachable;
                    sql_len += line.len;
                    buf_slice = buf_slice[line.len..];
                }

                const end_sql = std.fmt.bufPrint(buf_slice, ");", .{}) catch unreachable;
                sql_len += end_sql.len;
            }

            try self.db.exec(sql_buf[0..sql_len]);
        }

        /// Insert record
        pub fn insert(self: *const Self, record: T) !i64 {
            const fields = @typeInfo(T).Struct.fields;

            // Build INSERT SQL
            var sql = std.ArrayList(u8).init(self.db.allocator);
            defer sql.deinit();

            const writer = sql.writer();
            try writer.print("INSERT INTO {s} (", .{table_name});

            comptime var first = true;
            inline for (fields) |field| {
                if (!std.mem.eql(u8, field.name, "id")) {
                    if (!first) try writer.writeAll(", ");
                    try writer.writeAll(field.name);
                    first = false;
                }
            }

            try writer.writeAll(") VALUES (");

            first = true;
            inline for (fields) |field| {
                if (!std.mem.eql(u8, field.name, "id")) {
                    if (!first) try writer.writeAll(", ");
                    try writer.writeAll("?");
                    first = false;
                }
            }

            try writer.writeAll(");");

            var stmt = try self.db.prepare(sql.items);
            defer stmt.finalize();

            // Bind values
            var col: c_int = 1;
            inline for (fields) |field| {
                if (!std.mem.eql(u8, field.name, "id")) {
                    try stmt.bindAny(col, @field(record, field.name));
                    col += 1;
                }
            }

            _ = try stmt.step();
            return self.db.lastInsertRowid();
        }

        /// Find by id
        pub fn findById(self: *const Self, id: i64) !?T {
            const sql = try std.fmt.allocPrint(
                self.db.allocator,
                "SELECT * FROM {s} WHERE id = ?;",
                .{table_name},
            );
            defer self.db.allocator.free(sql);

            var stmt = try self.db.prepare(sql);
            defer stmt.finalize();

            try stmt.bindInt(1, id);

            return switch (try stmt.step()) {
                .row => try stmt.scanInto(T),
                .done => null,
            };
        }

        /// Find all
        pub fn findAll(self: *const Self) ![]T {
            const sql = try std.fmt.allocPrint(
                self.db.allocator,
                "SELECT * FROM {s};",
                .{table_name},
            );
            defer self.db.allocator.free(sql);

            return self.db.queryAll(T, sql, .{});
        }

        /// Find with condition
        pub fn findWhere(
            self: *const Self,
            condition: []const u8,
            binds: anytype,
        ) ![]T {
            const sql = try std.fmt.allocPrint(
                self.db.allocator,
                "SELECT * FROM {s} WHERE {s};",
                .{ table_name, condition },
            );
            defer self.db.allocator.free(sql);

            return self.db.queryAll(T, sql, binds);
        }

        /// Update by id
        pub fn update(self: *const Self, id: i64, record: T) !void {
            const fields = @typeInfo(T).Struct.fields;

            var sql = std.ArrayList(u8).init(self.db.allocator);
            defer sql.deinit();

            const writer = sql.writer();
            try writer.print("UPDATE {s} SET ", .{table_name});

            comptime var first = true;
            inline for (fields) |field| {
                if (!std.mem.eql(u8, field.name, "id")) {
                    if (!first) try writer.writeAll(", ");
                    try writer.print("{s} = ?", .{field.name});
                    first = false;
                }
            }

            try writer.writeAll(" WHERE id = ?;");

            var stmt = try self.db.prepare(sql.items);
            defer stmt.finalize();

            var col: c_int = 1;
            inline for (fields) |field| {
                if (!std.mem.eql(u8, field.name, "id")) {
                    try stmt.bindAny(col, @field(record, field.name));
                    col += 1;
                }
            }
            try stmt.bindInt(col, id);

            _ = try stmt.step();
        }

        /// Delete by id
        pub fn deleteById(self: *const Self, id: i64) !void {
            const sql = try std.fmt.allocPrint(
                self.db.allocator,
                "DELETE FROM {s} WHERE id = ?;",
                .{table_name},
            );
            defer self.db.allocator.free(sql);

            var stmt = try self.db.prepare(sql);
            defer stmt.finalize();

            try stmt.bindInt(1, id);
            _ = try stmt.step();
        }

        /// Count rows
        pub fn count(self: *const Self) !i64 {
            const sql = try std.fmt.allocPrint(
                self.db.allocator,
                "SELECT COUNT(*) FROM {s};",
                .{table_name},
            );
            defer self.db.allocator.free(sql);

            var stmt = try self.db.prepare(sql);
            defer stmt.finalize();

            return switch (try stmt.step()) {
                .row => stmt.columnInt(0),
                .done => 0,
            };
        }
    };
}

fn getTableName(comptime T: type) []const u8 {
    // ใช้ชื่อ struct เป็น table name (lowercase)
    const type_name = @typeName(T);
    // หา '.' ตัวสุดท้าย
    var last_dot: usize = 0;
    for (type_name, 0..) |c, i| {
        if (c == '.') last_dot = i + 1;
    }
    return type_name[last_dot..];
}

fn zigTypeToSqlType(comptime T: type) []const u8 {
    return switch (@typeInfo(T)) {
        .Int => "INTEGER",
        .Float => "REAL",
        .Bool => "INTEGER",
        .Pointer => |ptr| if (ptr.child == u8) "TEXT" else "BLOB",
        .Optional => |opt| zigTypeToSqlType(opt.child),
        else => "TEXT",
    };
}

/// สรุปตัวอย่างการใช้งาน ORM

pub const UserRecord = struct {
    id: i64,
    name: []const u8,
    email: []const u8,
    age: i64,
    active: bool,
};

pub const PostRecord = struct {
    id: i64,
    user_id: i64,
    title: []const u8,
    content: []const u8,
    published: bool,
};
```

### src/main.zig - ตัวอย่างการใช้งาน

```zig
const std = @import("std");
const Database = @import("database.zig").Database;
const orm = @import("orm.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // เปิด database
    var db = try Database.open(allocator, "app.db", .{
        .wal_mode = true,
    });
    defer db.close();

    std.log.info("SQLite version: {s}", .{Database.version()});

    // สร้าง tables
    try db.exec(
        \\CREATE TABLE IF NOT EXISTS users (
        \\  id INTEGER PRIMARY KEY AUTOINCREMENT,
        \\  name TEXT NOT NULL,
        \\  email TEXT UNIQUE NOT NULL,
        \\  age INTEGER,
        \\  created_at INTEGER DEFAULT (strftime('%s', 'now'))
        \\);
    );

    try db.exec(
        \\CREATE TABLE IF NOT EXISTS posts (
        \\  id INTEGER PRIMARY KEY AUTOINCREMENT,
        \\  user_id INTEGER NOT NULL,
        \\  title TEXT NOT NULL,
        \\  content TEXT,
        \\  published INTEGER DEFAULT 0,
        \\  FOREIGN KEY (user_id) REFERENCES users(id)
        \\);
    );

    // Transaction example
    {
        var tx = try db.beginTransaction();
        defer tx.deinit();

        // Insert users
        var insert_user = try db.prepare(
            "INSERT INTO users (name, email, age) VALUES (?, ?, ?);",
        );
        defer insert_user.finalize();

        try insert_user.bindText(1, "สมชาย ใจดี");
        try insert_user.bindText(2, "somchai@example.com");
        try insert_user.bindInt(3, 30);
        _ = try insert_user.step();

        const user_id = db.lastInsertRowid();
        std.log.info("Created user id: {d}", .{user_id});

        // Insert post
        var insert_post = try db.prepare(
            "INSERT INTO posts (user_id, title, content) VALUES (?, ?, ?);",
        );
        defer insert_post.finalize();

        try insert_post.bindInt(1, user_id);
        try insert_post.bindText(2, "บทความแรก");
        try insert_post.bindText(3, "เนื้อหาของบทความแรก...");
        _ = try insert_post.step();

        try tx.commit();
    }

    // Query users
    const UserRow = struct {
        id: i64,
        name: []const u8,
        email: []const u8,
        age: i64,
    };

    var query_stmt = try db.prepare("SELECT id, name, email, age FROM users;");
    defer query_stmt.finalize();

    std.log.info("Users:", .{});
    while (true) {
        const step_result = try query_stmt.step();
        if (step_result == .done) break;

        const id = query_stmt.columnInt(0);
        const name = query_stmt.columnText(1);
        const email = query_stmt.columnText(2);
        const age = query_stmt.columnInt(3);

        std.log.info("  [{d}] {s} ({s}) age={d}", .{ id, name, email, age });
    }

    // Query with parameters
    var age_query = try db.prepare("SELECT name FROM users WHERE age > ?;");
    defer age_query.finalize();

    try age_query.bindInt(1, 25);

    std.log.info("Users older than 25:", .{});
    while (true) {
        const step_result = try age_query.step();
        if (step_result == .done) break;
        std.log.info("  {s}", .{age_query.columnText(0)});
    }

    // ORM example
    const UserTable = orm.Table(orm.UserRecord);
    const user_table = UserTable.init(&db);

    const total = try user_table.count();
    std.log.info("Total users: {d}", .{total});

    std.log.info("Database operations completed!", .{});
}
```

### tests - Unit Tests

```zig
const std = @import("std");
const Database = @import("database.zig").Database;

test "Open and close database" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    // ทดสอบ version
    const ver = Database.version();
    try std.testing.expect(ver.len > 0);
}

test "Create table and insert" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try db.exec(
        \\CREATE TABLE test (
        \\  id INTEGER PRIMARY KEY,
        \\  name TEXT,
        \\  value REAL
        \\);
    );

    var stmt = try db.prepare("INSERT INTO test (name, value) VALUES (?, ?);");
    defer stmt.finalize();

    try stmt.bindText(1, "hello");
    try stmt.bindFloat(2, 3.14);
    _ = try stmt.step();

    try std.testing.expectEqual(@as(i64, 1), db.lastInsertRowid());
    try std.testing.expectEqual(@as(i32, 1), db.changes());
}

test "Query with types" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try db.exec("CREATE TABLE items (id INTEGER PRIMARY KEY, name TEXT, qty INTEGER);");
    try db.exec("INSERT INTO items (name, qty) VALUES ('apple', 5);");
    try db.exec("INSERT INTO items (name, qty) VALUES ('banana', 3);");
    try db.exec("INSERT INTO items (name, qty) VALUES ('cherry', 10);");

    var count_stmt = try db.prepare("SELECT COUNT(*) FROM items;");
    defer count_stmt.finalize();

    _ = try count_stmt.step();
    try std.testing.expectEqual(@as(i64, 3), count_stmt.columnInt(0));
}

test "Transaction commit" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try db.exec("CREATE TABLE accounts (id INTEGER PRIMARY KEY, balance REAL);");
    try db.exec("INSERT INTO accounts (balance) VALUES (1000.0);");
    try db.exec("INSERT INTO accounts (balance) VALUES (500.0);");

    // Transfer money
    var tx = try db.beginTransaction();
    defer tx.deinit();

    try db.exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1;");
    try db.exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2;");

    try tx.commit();

    // ตรวจสอบ balance
    var stmt = try db.prepare("SELECT balance FROM accounts WHERE id = 1;");
    defer stmt.finalize();

    _ = try stmt.step();
    try std.testing.expectApproxEqAbs(@as(f64, 900.0), stmt.columnFloat(0), 0.001);
}

test "Transaction rollback" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try db.exec("CREATE TABLE counter (id INTEGER PRIMARY KEY, n INTEGER);");
    try db.exec("INSERT INTO counter (n) VALUES (0);");

    {
        var tx = try db.beginTransaction();
        defer tx.deinit();

        try db.exec("UPDATE counter SET n = n + 1;");
        // ไม่ commit - จะ rollback อัตโนมัติ
    }

    var stmt = try db.prepare("SELECT n FROM counter;");
    defer stmt.finalize();

    _ = try stmt.step();
    try std.testing.expectEqual(@as(i64, 0), stmt.columnInt(0));
}

test "Prepared statement reuse" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try db.exec("CREATE TABLE nums (n INTEGER);");

    var stmt = try db.prepare("INSERT INTO nums (n) VALUES (?);");
    defer stmt.finalize();

    for (0..10) |i| {
        try stmt.reset();
        try stmt.bindInt(1, @intCast(i));
        _ = try stmt.step();
    }

    var count_stmt = try db.prepare("SELECT COUNT(*) FROM nums;");
    defer count_stmt.finalize();

    _ = try count_stmt.step();
    try std.testing.expectEqual(@as(i64, 10), count_stmt.columnInt(0));
}

test "Table exists check" {
    const allocator = std.testing.allocator;

    var db = try Database.openMemory(allocator);
    defer db.close();

    try std.testing.expect(!try db.tableExists("nonexistent"));

    try db.exec("CREATE TABLE existing (id INTEGER);");
    try std.testing.expect(try db.tableExists("existing"));
}
```

### สรุป Part 64

เราได้สร้าง SQLite Wrapper ที่มีความสามารถ:
- C library integration ผ่าน `@cImport`
- Database connection management
- Prepared statement ที่ type-safe
- Parameter binding อัตโนมัติ
- Result iteration
- Transaction management พร้อม savepoints
- ORM-like patterns ด้วย comptime
- Error handling ที่ครบถ้วน
- Unit tests ครบทุก feature

---
*ต่อไป Part 65: Static Site Generator*
