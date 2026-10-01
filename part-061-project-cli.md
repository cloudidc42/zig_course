# ส่วนที่ 61: โปรเจค CLI Application ที่สมบูรณ์

## ขั้นตอนที่ 601-610: สร้าง Todo CLI Application

---

## ขั้นตอนที่ 601: Project Overview และ Requirements

### ภาพรวมโปรเจค

เราจะสร้าง CLI Application สำหรับจัดการ Todo List ที่มีฟีเจอร์ครบถ้วน โดยใช้ภาษา Zig ล้วนๆ

### ความสามารถของแอปพลิเคชัน

```
todo [command] [options]

Commands:
  add <title>          เพิ่ม todo ใหม่
  list                 แสดงรายการ todo ทั้งหมด
  done <id>            ทำเครื่องหมายว่าเสร็จแล้ว
  delete <id>          ลบ todo
  search <query>       ค้นหา todo
  filter <status>      กรองตามสถานะ
  stats                แสดงสถิติ
  clear                ลบ todo ทั้งหมด

Options:
  -f, --file <path>    ระบุ file path (default: ~/.todos.json)
  -c, --color          เปิดใช้งาน color output
  -v, --verbose        แสดงข้อมูลเพิ่มเติม
  -h, --help           แสดง help
```

### สิ่งที่จะได้เรียนรู้

- การ parse command-line arguments อย่างมืออาชีพ
- การจัดการ data structures ที่ซับซ้อน
- การอ่านเขียน JSON file
- การจัดการ error ที่ครบถ้วน
- การเขียน unit tests
- การออกแบบ API ที่ดี

---

## ขั้นตอนที่ 602: Project Structure และ build.zig

### โครงสร้างโปรเจค

```
todo/
├── build.zig
├── build.zig.zon
├── src/
│   ├── main.zig
│   ├── cli.zig
│   ├── todo.zig
│   ├── storage.zig
│   ├── output.zig
│   └── search.zig
└── tests/
    ├── todo_test.zig
    ├── storage_test.zig
    └── cli_test.zig
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง executable หลัก
    const exe = b.addExecutable(.{
        .name = "todo",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    // สร้าง run step
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run todo application");
    run_step.dependOn(&run_cmd.step);

    // สร้าง test step
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);

    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

### build.zig.zon

```zig
.{
    .name = "todo-cli",
    .version = "1.0.0",
    .minimum_zig_version = "0.14.0",
    .dependencies = .{},
    .paths = .{"."},
}
```

### src/main.zig (Entry Point)

```zig
const std = @import("std");
const cli = @import("cli.zig");
const output = @import("output.zig");

pub fn main() !void {
    // สร้าง allocator
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // ดึง arguments
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    // รัน CLI
    cli.run(allocator, args) catch |err| {
        switch (err) {
            error.InvalidArgument => {
                try output.printError("Invalid argument. Use 'todo --help' for usage.");
                std.process.exit(1);
            },
            error.FileNotFound => {
                try output.printError("Todo file not found.");
                std.process.exit(1);
            },
            error.OutOfMemory => {
                try output.printError("Out of memory.");
                std.process.exit(1);
            },
            else => {
                try output.printError("Unexpected error occurred.");
                std.process.exit(1);
            },
        }
    };
}
```

---

## ขั้นตอนที่ 603: Command-line Argument Parsing

### src/cli.zig

```zig
const std = @import("std");
const todo = @import("todo.zig");
const storage = @import("storage.zig");
const output = @import("output.zig");
const search = @import("search.zig");

pub const Command = enum {
    add,
    list,
    done,
    delete,
    search,
    filter,
    stats,
    clear,
    help,
};

pub const Options = struct {
    file_path: []const u8,
    use_color: bool,
    verbose: bool,
    command: Command,
    args: []const []const u8,

    pub fn deinit(self: *Options, allocator: std.mem.Allocator) void {
        allocator.free(self.file_path);
        for (self.args) |arg| {
            allocator.free(arg);
        }
        allocator.free(self.args);
    }
};

pub const ParseError = error{
    NoCommand,
    UnknownCommand,
    MissingArgument,
    InvalidFlag,
    OutOfMemory,
};

/// Parse command-line arguments
pub fn parseArgs(allocator: std.mem.Allocator, args: []const []const u8) ParseError!Options {
    if (args.len < 2) {
        return ParseError.NoCommand;
    }

    var file_path: []const u8 = undefined;
    var use_color = false;
    var verbose = false;
    var command: Command = undefined;
    var command_args = std.ArrayList([]const u8).init(allocator);
    defer command_args.deinit();

    // กำหนด default file path
    const home_dir = std.posix.getenv("HOME") orelse ".";
    file_path = try std.fmt.allocPrint(allocator, "{s}/.todos.json", .{home_dir});

    var i: usize = 1;
    var command_found = false;

    while (i < args.len) : (i += 1) {
        const arg = args[i];

        if (std.mem.eql(u8, arg, "--help") or std.mem.eql(u8, arg, "-h")) {
            command = .help;
            command_found = true;
        } else if (std.mem.eql(u8, arg, "--color") or std.mem.eql(u8, arg, "-c")) {
            use_color = true;
        } else if (std.mem.eql(u8, arg, "--verbose") or std.mem.eql(u8, arg, "-v")) {
            verbose = true;
        } else if (std.mem.eql(u8, arg, "--file") or std.mem.eql(u8, arg, "-f")) {
            i += 1;
            if (i >= args.len) return ParseError.MissingArgument;
            allocator.free(file_path);
            file_path = try allocator.dupe(u8, args[i]);
        } else if (!command_found) {
            // Parse command
            if (std.mem.eql(u8, arg, "add")) {
                command = .add;
            } else if (std.mem.eql(u8, arg, "list")) {
                command = .list;
            } else if (std.mem.eql(u8, arg, "done")) {
                command = .done;
            } else if (std.mem.eql(u8, arg, "delete") or std.mem.eql(u8, arg, "del")) {
                command = .delete;
            } else if (std.mem.eql(u8, arg, "search")) {
                command = .search;
            } else if (std.mem.eql(u8, arg, "filter")) {
                command = .filter;
            } else if (std.mem.eql(u8, arg, "stats")) {
                command = .stats;
            } else if (std.mem.eql(u8, arg, "clear")) {
                command = .clear;
            } else {
                allocator.free(file_path);
                return ParseError.UnknownCommand;
            }
            command_found = true;
        } else {
            // เก็บ arguments ที่เหลือ
            const arg_copy = try allocator.dupe(u8, arg);
            try command_args.append(arg_copy);
        }
    }

    if (!command_found) {
        allocator.free(file_path);
        return ParseError.NoCommand;
    }

    return Options{
        .file_path = file_path,
        .use_color = use_color,
        .verbose = verbose,
        .command = command,
        .args = try command_args.toOwnedSlice(),
    };
}

/// รัน CLI application
pub fn run(allocator: std.mem.Allocator, args: []const []const u8) !void {
    var opts = parseArgs(allocator, args) catch |err| {
        switch (err) {
            ParseError.NoCommand, ParseError.UnknownCommand => {
                try printHelp();
                return;
            },
            ParseError.MissingArgument => {
                try output.printError("Missing argument for flag");
                return error.InvalidArgument;
            },
            else => return err,
        }
    };
    defer opts.deinit(allocator);

    // ตั้งค่า color output
    output.setColor(opts.use_color);

    switch (opts.command) {
        .help => try printHelp(),
        .add => try handleAdd(allocator, opts),
        .list => try handleList(allocator, opts),
        .done => try handleDone(allocator, opts),
        .delete => try handleDelete(allocator, opts),
        .search => try handleSearch(allocator, opts),
        .filter => try handleFilter(allocator, opts),
        .stats => try handleStats(allocator, opts),
        .clear => try handleClear(allocator, opts),
    }
}

fn printHelp() !void {
    const help_text =
        \\Todo CLI - จัดการ Todo List ของคุณ
        \\
        \\การใช้งาน:
        \\  todo [command] [options]
        \\
        \\Commands:
        \\  add <title>          เพิ่ม todo ใหม่
        \\  list                 แสดงรายการ todo ทั้งหมด
        \\  done <id>            ทำเครื่องหมายว่าเสร็จแล้ว
        \\  delete <id>          ลบ todo
        \\  search <query>       ค้นหา todo
        \\  filter <status>      กรองตามสถานะ (pending/done/all)
        \\  stats                แสดงสถิติ
        \\  clear                ลบ todo ทั้งหมด
        \\
        \\Options:
        \\  -f, --file <path>    ระบุ file path
        \\  -c, --color          เปิดใช้งาน color output
        \\  -v, --verbose        แสดงข้อมูลเพิ่มเติม
        \\  -h, --help           แสดง help นี้
        \\
        \\ตัวอย่าง:
        \\  todo add "ซื้อของที่ตลาด"
        \\  todo list
        \\  todo done 1
        \\  todo search "ตลาด"
        \\  todo filter pending
        \\
    ;
    const stdout = std.io.getStdOut().writer();
    try stdout.print("{s}", .{help_text});
}

fn handleAdd(allocator: std.mem.Allocator, opts: Options) !void {
    if (opts.args.len == 0) {
        try output.printError("กรุณาระบุ title ของ todo");
        return error.InvalidArgument;
    }

    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    const title = opts.args[0];
    const new_todo = try store.addTodo(title);

    try output.printSuccess("เพิ่ม todo สำเร็จ:");
    try output.printTodo(new_todo, opts.use_color);
}

fn handleList(allocator: std.mem.Allocator, opts: Options) !void {
    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    const todos = store.getAllTodos();

    if (todos.len == 0) {
        try output.printInfo("ไม่มี todo ในขณะนี้ ลองเพิ่มด้วย 'todo add <title>'");
        return;
    }

    try output.printTodoList(todos, opts.use_color);
}

fn handleDone(allocator: std.mem.Allocator, opts: Options) !void {
    if (opts.args.len == 0) {
        try output.printError("กรุณาระบุ ID ของ todo");
        return error.InvalidArgument;
    }

    const id = std.fmt.parseInt(u32, opts.args[0], 10) catch {
        try output.printError("ID ต้องเป็นตัวเลข");
        return error.InvalidArgument;
    };

    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    try store.completeTodo(id);
    try output.printSuccess("ทำเครื่องหมาย todo เสร็จแล้ว");
}

fn handleDelete(allocator: std.mem.Allocator, opts: Options) !void {
    if (opts.args.len == 0) {
        try output.printError("กรุณาระบุ ID ของ todo");
        return error.InvalidArgument;
    }

    const id = std.fmt.parseInt(u32, opts.args[0], 10) catch {
        try output.printError("ID ต้องเป็นตัวเลข");
        return error.InvalidArgument;
    };

    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    try store.deleteTodo(id);
    try output.printSuccess("ลบ todo สำเร็จ");
}

fn handleSearch(allocator: std.mem.Allocator, opts: Options) !void {
    if (opts.args.len == 0) {
        try output.printError("กรุณาระบุ คำค้นหา");
        return error.InvalidArgument;
    }

    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    const query = opts.args[0];
    const results = try search.searchTodos(allocator, store.getAllTodos(), query);
    defer allocator.free(results);

    if (results.len == 0) {
        try output.printInfo("ไม่พบ todo ที่ตรงกับคำค้นหา");
        return;
    }

    try output.printTodoList(results, opts.use_color);
}

fn handleFilter(allocator: std.mem.Allocator, opts: Options) !void {
    const filter_status = if (opts.args.len > 0) opts.args[0] else "all";

    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    const filtered = try search.filterTodos(allocator, store.getAllTodos(), filter_status);
    defer allocator.free(filtered);

    try output.printTodoList(filtered, opts.use_color);
}

fn handleStats(allocator: std.mem.Allocator, opts: Options) !void {
    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    const todos = store.getAllTodos();
    var completed: usize = 0;
    var pending: usize = 0;

    for (todos) |t| {
        if (t.completed) {
            completed += 1;
        } else {
            pending += 1;
        }
    }

    const stdout = std.io.getStdOut().writer();
    try stdout.print("\nสถิติ Todo:\n", .{});
    try stdout.print("  ทั้งหมด:    {d}\n", .{todos.len});
    try stdout.print("  เสร็จแล้ว:  {d}\n", .{completed});
    try stdout.print("  ค้างอยู่:   {d}\n", .{pending});

    if (todos.len > 0) {
        const percent = (completed * 100) / todos.len;
        try stdout.print("  ความคืบหน้า: {d}%\n", .{percent});
    }
}

fn handleClear(allocator: std.mem.Allocator, opts: Options) !void {
    var store = try storage.Storage.init(allocator, opts.file_path);
    defer store.deinit();

    try store.clearAll();
    try output.printSuccess("ลบ todo ทั้งหมดสำเร็จ");
}
```

---

## ขั้นตอนที่ 604: Todo Data Structure และ Operations

### src/todo.zig

```zig
const std = @import("std");

pub const TodoStatus = enum {
    pending,
    completed,
    archived,

    pub fn toString(self: TodoStatus) []const u8 {
        return switch (self) {
            .pending => "pending",
            .completed => "completed",
            .archived => "archived",
        };
    }

    pub fn fromString(s: []const u8) ?TodoStatus {
        if (std.mem.eql(u8, s, "pending")) return .pending;
        if (std.mem.eql(u8, s, "completed")) return .completed;
        if (std.mem.eql(u8, s, "archived")) return .archived;
        return null;
    }
};

pub const Priority = enum {
    low,
    medium,
    high,
    urgent,

    pub fn toString(self: Priority) []const u8 {
        return switch (self) {
            .low => "low",
            .medium => "medium",
            .high => "high",
            .urgent => "urgent",
        };
    }

    pub fn fromString(s: []const u8) Priority {
        if (std.mem.eql(u8, s, "high")) return .high;
        if (std.mem.eql(u8, s, "urgent")) return .urgent;
        if (std.mem.eql(u8, s, "low")) return .low;
        return .medium;
    }
};

pub const Todo = struct {
    id: u32,
    title: []const u8,
    description: ?[]const u8,
    completed: bool,
    status: TodoStatus,
    priority: Priority,
    created_at: i64,  // Unix timestamp
    updated_at: i64,
    tags: []const []const u8,

    /// สร้าง Todo ใหม่
    pub fn create(
        allocator: std.mem.Allocator,
        id: u32,
        title: []const u8,
    ) !Todo {
        const now = std.time.timestamp();
        const title_copy = try allocator.dupe(u8, title);

        return Todo{
            .id = id,
            .title = title_copy,
            .description = null,
            .completed = false,
            .status = .pending,
            .priority = .medium,
            .created_at = now,
            .updated_at = now,
            .tags = &[_][]const u8{},
        };
    }

    /// ทำสำเนา Todo
    pub fn clone(self: Todo, allocator: std.mem.Allocator) !Todo {
        const title_copy = try allocator.dupe(u8, self.title);

        const desc_copy = if (self.description) |d|
            try allocator.dupe(u8, d)
        else
            null;

        var tags_copy = try allocator.alloc([]const u8, self.tags.len);
        for (self.tags, 0..) |tag, i| {
            tags_copy[i] = try allocator.dupe(u8, tag);
        }

        return Todo{
            .id = self.id,
            .title = title_copy,
            .description = desc_copy,
            .completed = self.completed,
            .status = self.status,
            .priority = self.priority,
            .created_at = self.created_at,
            .updated_at = self.updated_at,
            .tags = tags_copy,
        };
    }

    /// คืนหน่วยความจำ
    pub fn deinit(self: *Todo, allocator: std.mem.Allocator) void {
        allocator.free(self.title);
        if (self.description) |d| allocator.free(d);
        for (self.tags) |tag| allocator.free(tag);
        allocator.free(self.tags);
    }

    /// ทำเครื่องหมายว่าเสร็จแล้ว
    pub fn complete(self: *Todo) void {
        self.completed = true;
        self.status = .completed;
        self.updated_at = std.time.timestamp();
    }

    /// เพิ่ม tag
    pub fn addTag(self: *Todo, allocator: std.mem.Allocator, tag: []const u8) !void {
        // ตรวจสอบว่ามี tag อยู่แล้วหรือไม่
        for (self.tags) |existing_tag| {
            if (std.mem.eql(u8, existing_tag, tag)) return;
        }

        const new_tags = try allocator.alloc([]const u8, self.tags.len + 1);
        @memcpy(new_tags[0..self.tags.len], self.tags);
        new_tags[self.tags.len] = try allocator.dupe(u8, tag);

        allocator.free(self.tags);
        self.tags = new_tags;
        self.updated_at = std.time.timestamp();
    }

    /// แปลง timestamp เป็น string ที่อ่านได้
    pub fn formatDate(self: Todo, allocator: std.mem.Allocator) ![]u8 {
        const epoch_seconds = std.time.epoch.EpochSeconds{
            .secs = @intCast(self.created_at),
        };
        const day_seconds = epoch_seconds.getDaySeconds();
        const epoch_day = epoch_seconds.getEpochDay();
        const year_day = epoch_day.calculateYearDay();
        const month_day = year_day.calculateMonthDay();

        return std.fmt.allocPrint(
            allocator,
            "{d:0>4}-{d:0>2}-{d:0>2} {d:0>2}:{d:0>2}",
            .{
                year_day.year,
                month_day.month.numeric(),
                month_day.day_index + 1,
                day_seconds.getHoursIntoDay(),
                day_seconds.getMinutesIntoHour(),
            },
        );
    }

    /// ตรวจสอบว่า title ตรงกับ query หรือไม่
    pub fn matchesQuery(self: Todo, query: []const u8) bool {
        // ค้นหาแบบ case-insensitive ใน title
        var i: usize = 0;
        while (i + query.len <= self.title.len) : (i += 1) {
            if (std.ascii.eqlIgnoreCase(self.title[i..][0..query.len], query)) {
                return true;
            }
        }

        // ค้นหาใน description
        if (self.description) |desc| {
            i = 0;
            while (i + query.len <= desc.len) : (i += 1) {
                if (std.ascii.eqlIgnoreCase(desc[i..][0..query.len], query)) {
                    return true;
                }
            }
        }

        // ค้นหาใน tags
        for (self.tags) |tag| {
            if (std.ascii.eqlIgnoreCase(tag, query)) return true;
        }

        return false;
    }
};

/// TodoList - จัดการ collection ของ Todo
pub const TodoList = struct {
    allocator: std.mem.Allocator,
    items: std.ArrayList(Todo),
    next_id: u32,

    pub fn init(allocator: std.mem.Allocator) TodoList {
        return TodoList{
            .allocator = allocator,
            .items = std.ArrayList(Todo).init(allocator),
            .next_id = 1,
        };
    }

    pub fn deinit(self: *TodoList) void {
        for (self.items.items) |*item| {
            item.deinit(self.allocator);
        }
        self.items.deinit();
    }

    pub fn add(self: *TodoList, title: []const u8) !*Todo {
        const new_todo = try Todo.create(self.allocator, self.next_id, title);
        self.next_id += 1;
        try self.items.append(new_todo);
        return &self.items.items[self.items.items.len - 1];
    }

    pub fn findById(self: *TodoList, id: u32) ?*Todo {
        for (self.items.items) |*item| {
            if (item.id == id) return item;
        }
        return null;
    }

    pub fn complete(self: *TodoList, id: u32) !void {
        if (self.findById(id)) |t| {
            t.complete();
        } else {
            return error.NotFound;
        }
    }

    pub fn delete(self: *TodoList, id: u32) !void {
        for (self.items.items, 0..) |item, i| {
            if (item.id == id) {
                var removed = self.items.orderedRemove(i);
                removed.deinit(self.allocator);
                return;
            }
        }
        return error.NotFound;
    }

    pub fn getAll(self: *TodoList) []const Todo {
        return self.items.items;
    }

    pub fn clear(self: *TodoList) void {
        for (self.items.items) |*item| {
            item.deinit(self.allocator);
        }
        self.items.clearRetainingCapacity();
    }

    pub fn count(self: *TodoList) usize {
        return self.items.items.len;
    }

    pub fn completedCount(self: *TodoList) usize {
        var n: usize = 0;
        for (self.items.items) |item| {
            if (item.completed) n += 1;
        }
        return n;
    }
};

// Unit tests
test "Todo creation" {
    const allocator = std.testing.allocator;
    var t = try Todo.create(allocator, 1, "Test todo");
    defer t.deinit(allocator);

    try std.testing.expectEqual(@as(u32, 1), t.id);
    try std.testing.expectEqualStrings("Test todo", t.title);
    try std.testing.expect(!t.completed);
    try std.testing.expectEqual(TodoStatus.pending, t.status);
}

test "TodoList operations" {
    const allocator = std.testing.allocator;
    var list = TodoList.init(allocator);
    defer list.deinit();

    const t1 = try list.add("First todo");
    _ = t1;
    const t2 = try list.add("Second todo");
    _ = t2;

    try std.testing.expectEqual(@as(usize, 2), list.count());

    try list.complete(1);
    try std.testing.expectEqual(@as(usize, 1), list.completedCount());

    try list.delete(2);
    try std.testing.expectEqual(@as(usize, 1), list.count());
}
```

---

## ขั้นตอนที่ 605: JSON File Persistence

### src/storage.zig

```zig
const std = @import("std");
const todo = @import("todo.zig");

pub const StorageError = error{
    FileNotFound,
    InvalidFormat,
    WriteError,
    OutOfMemory,
};

/// Storage จัดการการอ่านเขียน todo ลง JSON file
pub const Storage = struct {
    allocator: std.mem.Allocator,
    file_path: []const u8,
    list: todo.TodoList,

    pub fn init(allocator: std.mem.Allocator, file_path: []const u8) !Storage {
        var storage = Storage{
            .allocator = allocator,
            .file_path = file_path,
            .list = todo.TodoList.init(allocator),
        };

        // โหลดข้อมูลจาก file ถ้ามี
        storage.load() catch |err| {
            switch (err) {
                error.FileNotFound => {}, // ไม่มี file = เริ่มต้นใหม่
                else => return err,
            }
        };

        return storage;
    }

    pub fn deinit(self: *Storage) void {
        self.list.deinit();
    }

    /// โหลดข้อมูลจาก JSON file
    fn load(self: *Storage) !void {
        const file = std.fs.cwd().openFile(self.file_path, .{}) catch |err| {
            if (err == error.FileNotFound) return error.FileNotFound;
            return err;
        };
        defer file.close();

        const file_size = try file.getEndPos();
        if (file_size == 0) return;

        const content = try self.allocator.alloc(u8, file_size);
        defer self.allocator.free(content);

        _ = try file.readAll(content);

        // Parse JSON
        const parsed = std.json.parseFromSlice(
            std.json.Value,
            self.allocator,
            content,
            .{},
        ) catch return StorageError.InvalidFormat;
        defer parsed.deinit();

        const root = parsed.value;
        if (root != .object) return StorageError.InvalidFormat;

        // อ่าน next_id
        if (root.object.get("next_id")) |next_id_val| {
            if (next_id_val == .integer) {
                self.list.next_id = @intCast(next_id_val.integer);
            }
        }

        // อ่าน todos array
        const todos_val = root.object.get("todos") orelse return;
        if (todos_val != .array) return StorageError.InvalidFormat;

        for (todos_val.array.items) |item| {
            if (item != .object) continue;

            const id = if (item.object.get("id")) |v|
                if (v == .integer) @as(u32, @intCast(v.integer)) else continue
            else continue;

            const title = if (item.object.get("title")) |v|
                if (v == .string) v.string else continue
            else continue;

            var new_todo = try todo.Todo.create(self.allocator, id, title);

            if (item.object.get("completed")) |v| {
                if (v == .bool) new_todo.completed = v.bool;
            }

            if (item.object.get("created_at")) |v| {
                if (v == .integer) new_todo.created_at = v.integer;
            }

            if (item.object.get("description")) |v| {
                if (v == .string) {
                    new_todo.description = try self.allocator.dupe(u8, v.string);
                }
            }

            if (item.object.get("status")) |v| {
                if (v == .string) {
                    new_todo.status = todo.TodoStatus.fromString(v.string) orelse .pending;
                }
            }

            try self.list.items.append(new_todo);
        }
    }

    /// บันทึกข้อมูลลง JSON file
    fn save(self: *Storage) !void {
        var buffer = std.ArrayList(u8).init(self.allocator);
        defer buffer.deinit();

        var writer = buffer.writer();

        try writer.writeAll("{\n");
        try writer.print("  \"next_id\": {d},\n", .{self.list.next_id});
        try writer.writeAll("  \"todos\": [\n");

        const items = self.list.items.items;
        for (items, 0..) |item, i| {
            try writer.writeAll("    {\n");
            try writer.print("      \"id\": {d},\n", .{item.id});

            // Escape JSON string
            try writer.writeAll("      \"title\": \"");
            try writeJsonString(writer, item.title);
            try writer.writeAll("\",\n");

            try writer.print("      \"completed\": {s},\n", .{if (item.completed) "true" else "false"});
            try writer.print("      \"status\": \"{s}\",\n", .{item.status.toString()});
            try writer.print("      \"priority\": \"{s}\",\n", .{item.priority.toString()});
            try writer.print("      \"created_at\": {d},\n", .{item.created_at});
            try writer.print("      \"updated_at\": {d}", .{item.updated_at});

            if (item.description) |desc| {
                try writer.writeAll(",\n      \"description\": \"");
                try writeJsonString(writer, desc);
                try writer.writeAll("\"");
            }

            if (item.tags.len > 0) {
                try writer.writeAll(",\n      \"tags\": [");
                for (item.tags, 0..) |tag, j| {
                    try writer.writeAll("\"");
                    try writeJsonString(writer, tag);
                    try writer.writeAll("\"");
                    if (j < item.tags.len - 1) try writer.writeAll(", ");
                }
                try writer.writeAll("]");
            }

            try writer.writeAll("\n    }");
            if (i < items.len - 1) try writer.writeAll(",");
            try writer.writeAll("\n");
        }

        try writer.writeAll("  ]\n}\n");

        // เขียน file
        const file = try std.fs.cwd().createFile(self.file_path, .{});
        defer file.close();

        try file.writeAll(buffer.items);
    }

    /// Escape JSON string
    fn writeJsonString(writer: anytype, s: []const u8) !void {
        for (s) |c| {
            switch (c) {
                '"' => try writer.writeAll("\\\""),
                '\\' => try writer.writeAll("\\\\"),
                '\n' => try writer.writeAll("\\n"),
                '\r' => try writer.writeAll("\\r"),
                '\t' => try writer.writeAll("\\t"),
                else => try writer.writeByte(c),
            }
        }
    }

    pub fn addTodo(self: *Storage, title: []const u8) !*todo.Todo {
        const new_todo = try self.list.add(title);
        try self.save();
        return new_todo;
    }

    pub fn completeTodo(self: *Storage, id: u32) !void {
        try self.list.complete(id);
        try self.save();
    }

    pub fn deleteTodo(self: *Storage, id: u32) !void {
        try self.list.delete(id);
        try self.save();
    }

    pub fn getAllTodos(self: *Storage) []const todo.Todo {
        return self.list.getAll();
    }

    pub fn clearAll(self: *Storage) !void {
        self.list.clear();
        try self.save();
    }
};
```

---

## ขั้นตอนที่ 606: List, Add, Complete, Delete Commands

ฟังก์ชันเหล่านี้ถูก implement ใน `cli.zig` แล้วในขั้นตอนที่ 603 โดยมีการเรียกใช้ผ่าน `Storage` และ `output` module

### src/output.zig

```zig
const std = @import("std");
const todo = @import("todo.zig");

var use_color: bool = false;

pub fn setColor(enabled: bool) void {
    use_color = enabled;
}

// ANSI color codes
const Colors = struct {
    pub const reset = "\x1b[0m";
    pub const bold = "\x1b[1m";
    pub const red = "\x1b[31m";
    pub const green = "\x1b[32m";
    pub const yellow = "\x1b[33m";
    pub const blue = "\x1b[34m";
    pub const magenta = "\x1b[35m";
    pub const cyan = "\x1b[36m";
    pub const white = "\x1b[37m";
    pub const gray = "\x1b[90m";
};

pub fn printError(msg: []const u8) !void {
    const stderr = std.io.getStdErr().writer();
    if (use_color) {
        try stderr.print("{s}Error:{s} {s}\n", .{ Colors.red, Colors.reset, msg });
    } else {
        try stderr.print("Error: {s}\n", .{msg});
    }
}

pub fn printSuccess(msg: []const u8) !void {
    const stdout = std.io.getStdOut().writer();
    if (use_color) {
        try stdout.print("{s}{s}✓{s} {s}\n", .{ Colors.bold, Colors.green, Colors.reset, msg });
    } else {
        try stdout.print("✓ {s}\n", .{msg});
    }
}

pub fn printInfo(msg: []const u8) !void {
    const stdout = std.io.getStdOut().writer();
    if (use_color) {
        try stdout.print("{s}ℹ{s} {s}\n", .{ Colors.blue, Colors.reset, msg });
    } else {
        try stdout.print("ℹ {s}\n", .{msg});
    }
}

pub fn printTodo(t: *const todo.Todo, color: bool) !void {
    const stdout = std.io.getStdOut().writer();
    const status_icon = if (t.completed) "✓" else "○";

    if (color) {
        const status_color = if (t.completed) Colors.green else Colors.yellow;
        try stdout.print(
            "  {s}{s}{s} [{d:0>3}] {s}\n",
            .{ status_color, status_icon, Colors.reset, t.id, t.title },
        );
    } else {
        try stdout.print(
            "  {s} [{d:0>3}] {s}\n",
            .{ status_icon, t.id, t.title },
        );
    }
}

pub fn printTodoList(todos: []const todo.Todo, color: bool) !void {
    const stdout = std.io.getStdOut().writer();

    if (color) {
        try stdout.print("\n{s}รายการ Todo ({d} รายการ):{s}\n", .{
            Colors.bold, todos.len, Colors.reset,
        });
    } else {
        try stdout.print("\nรายการ Todo ({d} รายการ):\n", .{todos.len});
    }

    try stdout.print("{s}\n", .{"─" ** 50});

    for (todos) |*t| {
        try printTodo(t, color);
    }

    try stdout.print("{s}\n\n", .{"─" ** 50});
}
```

---

## ขั้นตอนที่ 607: Search และ Filter

### src/search.zig

```zig
const std = @import("std");
const todo = @import("todo.zig");

/// ค้นหา todos ที่ตรงกับ query
pub fn searchTodos(
    allocator: std.mem.Allocator,
    todos: []const todo.Todo,
    query: []const u8,
) ![]const todo.Todo {
    var results = std.ArrayList(todo.Todo).init(allocator);
    defer results.deinit();

    for (todos) |t| {
        if (t.matchesQuery(query)) {
            const cloned = try t.clone(allocator);
            try results.append(cloned);
        }
    }

    return results.toOwnedSlice();
}

/// กรอง todos ตามสถานะ
pub fn filterTodos(
    allocator: std.mem.Allocator,
    todos: []const todo.Todo,
    status: []const u8,
) ![]const todo.Todo {
    var results = std.ArrayList(todo.Todo).init(allocator);
    defer results.deinit();

    for (todos) |t| {
        const should_include = blk: {
            if (std.mem.eql(u8, status, "all")) break :blk true;
            if (std.mem.eql(u8, status, "done") or std.mem.eql(u8, status, "completed")) {
                break :blk t.completed;
            }
            if (std.mem.eql(u8, status, "pending") or std.mem.eql(u8, status, "todo")) {
                break :blk !t.completed;
            }
            break :blk true;
        };

        if (should_include) {
            const cloned = try t.clone(allocator);
            try results.append(cloned);
        }
    }

    return results.toOwnedSlice();
}

/// เรียงลำดับ todos
pub fn sortTodos(
    todos: []todo.Todo,
    sort_by: []const u8,
    descending: bool,
) void {
    const SortContext = struct {
        sort_field: []const u8,
        desc: bool,

        fn lessThan(ctx: @This(), a: todo.Todo, b: todo.Todo) bool {
            const result = blk: {
                if (std.mem.eql(u8, ctx.sort_field, "id")) {
                    break :blk a.id < b.id;
                } else if (std.mem.eql(u8, ctx.sort_field, "title")) {
                    break :blk std.mem.lessThan(u8, a.title, b.title);
                } else if (std.mem.eql(u8, ctx.sort_field, "created")) {
                    break :blk a.created_at < b.created_at;
                } else if (std.mem.eql(u8, ctx.sort_field, "priority")) {
                    break :blk @intFromEnum(a.priority) < @intFromEnum(b.priority);
                }
                break :blk a.id < b.id;
            };

            return if (ctx.desc) !result else result;
        }
    };

    const ctx = SortContext{ .sort_field = sort_by, .desc = descending };
    std.mem.sort(todo.Todo, todos, ctx, SortContext.lessThan);
}

/// fuzzy search ด้วย Levenshtein distance
pub fn fuzzySearch(
    allocator: std.mem.Allocator,
    todos: []const todo.Todo,
    query: []const u8,
    max_distance: usize,
) ![]const todo.Todo {
    var results = std.ArrayList(todo.Todo).init(allocator);
    defer results.deinit();

    for (todos) |t| {
        const distance = levenshteinDistance(allocator, t.title, query) catch continue;
        if (distance <= max_distance) {
            const cloned = try t.clone(allocator);
            try results.append(cloned);
        }
    }

    return results.toOwnedSlice();
}

/// คำนวณ Levenshtein distance
fn levenshteinDistance(
    allocator: std.mem.Allocator,
    s1: []const u8,
    s2: []const u8,
) !usize {
    const m = s1.len;
    const n = s2.len;

    // สร้าง matrix
    const matrix = try allocator.alloc([]usize, m + 1);
    defer {
        for (matrix) |row| allocator.free(row);
        allocator.free(matrix);
    }

    for (matrix) |*row| {
        row.* = try allocator.alloc(usize, n + 1);
    }

    // Initialize
    for (0..m + 1) |i| matrix[i][0] = i;
    for (0..n + 1) |j| matrix[0][j] = j;

    // Fill matrix
    for (1..m + 1) |i| {
        for (1..n + 1) |j| {
            const cost: usize = if (s1[i - 1] == s2[j - 1]) 0 else 1;
            matrix[i][j] = @min(
                @min(matrix[i - 1][j] + 1, matrix[i][j - 1] + 1),
                matrix[i - 1][j - 1] + cost,
            );
        }
    }

    return matrix[m][n];
}

// Tests
test "searchTodos" {
    const allocator = std.testing.allocator;

    var t1 = try todo.Todo.create(allocator, 1, "ซื้อของ");
    defer t1.deinit(allocator);
    var t2 = try todo.Todo.create(allocator, 2, "ไปออกกำลังกาย");
    defer t2.deinit(allocator);

    const todos = [_]todo.Todo{ t1, t2 };
    const results = try searchTodos(allocator, &todos, "ซื้อ");
    defer {
        for (results) |*r| {
            var mutable = r.*;
            mutable.deinit(allocator);
        }
        allocator.free(results);
    }

    try std.testing.expectEqual(@as(usize, 1), results.len);
}
```

---

## ขั้นตอนที่ 608: Color Output Terminal

Color output ถูก implement ใน `output.zig` แล้ว นี่คือการขยาย feature เพิ่มเติม:

```zig
// เพิ่มใน output.zig

pub const ProgressBar = struct {
    total: usize,
    current: usize,
    width: usize,

    pub fn init(total: usize) ProgressBar {
        return ProgressBar{
            .total = total,
            .current = 0,
            .width = 40,
        };
    }

    pub fn render(self: ProgressBar, writer: anytype) !void {
        const percent = if (self.total > 0)
            (self.current * 100) / self.total
        else 0;

        const filled = (self.current * self.width) / (if (self.total > 0) self.total else 1);

        try writer.writeAll("[");

        var i: usize = 0;
        while (i < self.width) : (i += 1) {
            if (i < filled) {
                if (use_color) {
                    try writer.print("{s}█{s}", .{ Colors.green, Colors.reset });
                } else {
                    try writer.writeAll("█");
                }
            } else {
                try writer.writeAll("░");
            }
        }

        try writer.print("] {d}%\n", .{percent});
    }
};

pub fn printTable(todos: []const todo.Todo, color: bool) !void {
    const stdout = std.io.getStdOut().writer();
    const col_widths = [_]usize{ 5, 40, 10, 20 };

    // Header
    if (color) {
        try stdout.print("{s}", .{Colors.bold});
    }
    try stdout.print(
        " {s:<5} {s:<40} {s:<10} {s:<20}\n",
        .{ "ID", "Title", "Status", "Created" },
    );
    if (color) {
        try stdout.print("{s}", .{Colors.reset});
    }

    // Separator
    for (col_widths) |w| {
        var j: usize = 0;
        while (j < w + 1) : (j += 1) {
            try stdout.writeAll("─");
        }
    }
    try stdout.writeAll("\n");

    // Rows
    for (todos) |*t| {
        const status = if (t.completed) "done" else "pending";
        const title_truncated = if (t.title.len > 38)
            t.title[0..38]
        else
            t.title;

        if (color) {
            const row_color = if (t.completed) Colors.gray else Colors.white;
            try stdout.print("{s}", .{row_color});
        }

        try stdout.print(
            " {d:<5} {s:<40} {s:<10}\n",
            .{ t.id, title_truncated, status },
        );

        if (color) {
            try stdout.print("{s}", .{Colors.reset});
        }
    }
}
```

---

## ขั้นตอนที่ 609: Error Handling ที่ครบถ้วน

```zig
// src/errors.zig - Error types ครบถ้วน
const std = @import("std");

pub const AppError = error{
    // Storage errors
    StorageReadError,
    StorageWriteError,
    InvalidJsonFormat,
    FileLocked,

    // Todo errors
    TodoNotFound,
    DuplicateTitle,
    InvalidId,
    EmptyTitle,

    // CLI errors
    InvalidArgument,
    MissingRequiredArg,
    UnknownCommand,

    // System errors
    OutOfMemory,
    PermissionDenied,
    DiskFull,
};

pub const ErrorContext = struct {
    err: AppError,
    message: []const u8,
    location: ?[]const u8,

    pub fn format(
        self: ErrorContext,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        if (self.location) |loc| {
            try writer.print("[{s}] {s}: {s}", .{ loc, @errorName(self.err), self.message });
        } else {
            try writer.print("{s}: {s}", .{ @errorName(self.err), self.message });
        }
    }
};

pub fn handleError(err: anyerror) void {
    const stderr = std.io.getStdErr().writer();
    const msg = switch (err) {
        error.TodoNotFound => "ไม่พบ Todo ที่ระบุ",
        error.InvalidId => "ID ไม่ถูกต้อง",
        error.EmptyTitle => "Title ต้องไม่ว่าง",
        error.StorageReadError => "ไม่สามารถอ่านไฟล์ข้อมูลได้",
        error.StorageWriteError => "ไม่สามารถเขียนไฟล์ข้อมูลได้",
        error.PermissionDenied => "ไม่มีสิทธิ์เข้าถึงไฟล์",
        error.OutOfMemory => "หน่วยความจำไม่เพียงพอ",
        else => "เกิดข้อผิดพลาดที่ไม่คาดคิด",
    };
    stderr.print("ข้อผิดพลาด: {s}\n", .{msg}) catch {};
}

/// Result type สำหรับ operations ที่อาจ fail
pub fn Result(comptime T: type) type {
    return union(enum) {
        ok: T,
        err: struct {
            code: AppError,
            message: []const u8,
        },

        pub fn isOk(self: @This()) bool {
            return self == .ok;
        }

        pub fn unwrap(self: @This()) T {
            return switch (self) {
                .ok => |v| v,
                .err => |e| {
                    std.debug.panic("Unwrap on error: {s}", .{e.message});
                },
            };
        }
    };
}
```

---

## ขั้นตอนที่ 610: Testing

### tests/todo_test.zig

```zig
const std = @import("std");
const todo = @import("../src/todo.zig");
const storage = @import("../src/storage.zig");
const search = @import("../src/search.zig");

test "สร้าง Todo ใหม่" {
    const allocator = std.testing.allocator;

    var t = try todo.Todo.create(allocator, 1, "ทดสอบ");
    defer t.deinit(allocator);

    try std.testing.expectEqual(@as(u32, 1), t.id);
    try std.testing.expectEqualStrings("ทดสอบ", t.title);
    try std.testing.expect(!t.completed);
    try std.testing.expectEqual(todo.TodoStatus.pending, t.status);
    try std.testing.expectEqual(todo.Priority.medium, t.priority);
}

test "Complete todo" {
    const allocator = std.testing.allocator;

    var t = try todo.Todo.create(allocator, 1, "ทดสอบ");
    defer t.deinit(allocator);

    try std.testing.expect(!t.completed);
    t.complete();
    try std.testing.expect(t.completed);
    try std.testing.expectEqual(todo.TodoStatus.completed, t.status);
}

test "TodoList - add and count" {
    const allocator = std.testing.allocator;

    var list = todo.TodoList.init(allocator);
    defer list.deinit();

    _ = try list.add("Task 1");
    _ = try list.add("Task 2");
    _ = try list.add("Task 3");

    try std.testing.expectEqual(@as(usize, 3), list.count());
    try std.testing.expectEqual(@as(usize, 0), list.completedCount());
}

test "TodoList - complete and delete" {
    const allocator = std.testing.allocator;

    var list = todo.TodoList.init(allocator);
    defer list.deinit();

    const t1 = try list.add("Task 1");
    _ = t1;
    const t2 = try list.add("Task 2");
    _ = t2;

    try list.complete(1);
    try std.testing.expectEqual(@as(usize, 1), list.completedCount());

    try list.delete(2);
    try std.testing.expectEqual(@as(usize, 1), list.count());

    // ลบ todo ที่ไม่มี
    try std.testing.expectError(error.NotFound, list.delete(999));
}

test "Todo matchesQuery" {
    const allocator = std.testing.allocator;

    var t = try todo.Todo.create(allocator, 1, "ซื้อของตลาด");
    defer t.deinit(allocator);

    try std.testing.expect(t.matchesQuery("ตลาด"));
    try std.testing.expect(t.matchesQuery("ซื้อ"));
    try std.testing.expect(!t.matchesQuery("gym"));
}

test "Search todos" {
    const allocator = std.testing.allocator;

    var t1 = try todo.Todo.create(allocator, 1, "ซื้อของ");
    defer t1.deinit(allocator);
    var t2 = try todo.Todo.create(allocator, 2, "ออกกำลังกาย");
    defer t2.deinit(allocator);
    var t3 = try todo.Todo.create(allocator, 3, "อ่านหนังสือ");
    defer t3.deinit(allocator);

    const todos = [_]todo.Todo{ t1, t2, t3 };

    const results = try search.searchTodos(allocator, &todos, "ซื้อ");
    defer {
        for (results) |r| {
            var mutable = r;
            mutable.deinit(allocator);
        }
        allocator.free(results);
    }

    try std.testing.expectEqual(@as(usize, 1), results.len);
    try std.testing.expectEqualStrings("ซื้อของ", results[0].title);
}

test "Filter todos by status" {
    const allocator = std.testing.allocator;

    var t1 = try todo.Todo.create(allocator, 1, "Task 1");
    defer t1.deinit(allocator);
    var t2 = try todo.Todo.create(allocator, 2, "Task 2");
    defer t2.deinit(allocator);
    t1.complete();

    const todos = [_]todo.Todo{ t1, t2 };

    // Filter completed
    const done_todos = try search.filterTodos(allocator, &todos, "done");
    defer {
        for (done_todos) |r| {
            var mutable = r;
            mutable.deinit(allocator);
        }
        allocator.free(done_todos);
    }
    try std.testing.expectEqual(@as(usize, 1), done_todos.len);

    // Filter pending
    const pending_todos = try search.filterTodos(allocator, &todos, "pending");
    defer {
        for (pending_todos) |r| {
            var mutable = r;
            mutable.deinit(allocator);
        }
        allocator.free(pending_todos);
    }
    try std.testing.expectEqual(@as(usize, 1), pending_todos.len);
}

test "Storage - save and load" {
    const allocator = std.testing.allocator;

    // สร้าง temp file
    const tmp_path = "/tmp/test_todos.json";

    // สร้าง storage และเพิ่ม todos
    {
        var store = try storage.Storage.init(allocator, tmp_path);
        defer store.deinit();

        _ = try store.addTodo("Task A");
        _ = try store.addTodo("Task B");
        _ = try store.addTodo("Task C");
    }

    // โหลดจาก file ใหม่
    {
        var store = try storage.Storage.init(allocator, tmp_path);
        defer store.deinit();

        const todos = store.getAllTodos();
        try std.testing.expectEqual(@as(usize, 3), todos.len);
    }

    // ลบ temp file
    try std.fs.cwd().deleteFile(tmp_path);
}

// Integration test
test "Full workflow" {
    const allocator = std.testing.allocator;
    const tmp_path = "/tmp/test_workflow.json";

    var store = try storage.Storage.init(allocator, tmp_path);
    defer store.deinit();
    defer std.fs.cwd().deleteFile(tmp_path) catch {};

    // Add todos
    _ = try store.addTodo("เขียน unit tests");
    _ = try store.addTodo("Review code");
    _ = try store.addTodo("Deploy ขึ้น production");

    try std.testing.expectEqual(@as(usize, 3), store.getAllTodos().len);

    // Complete first todo
    try store.completeTodo(1);

    // Delete second todo
    try store.deleteTodo(2);

    const remaining = store.getAllTodos();
    try std.testing.expectEqual(@as(usize, 2), remaining.len);
    try std.testing.expect(remaining[0].completed);
    try std.testing.expect(!remaining[1].completed);
}
```

### การรันโปรแกรม

```bash
# Build
zig build

# รัน tests
zig build test

# ใช้งาน
./zig-out/bin/todo add "ซื้อของที่ตลาด"
./zig-out/bin/todo add "ออกกำลังกาย"
./zig-out/bin/todo list
./zig-out/bin/todo done 1
./zig-out/bin/todo search "ตลาด"
./zig-out/bin/todo stats
./zig-out/bin/todo --color list
```

### สรุป Part 61

เราได้สร้าง Todo CLI Application ที่สมบูรณ์พร้อม:
- Command-line argument parsing ที่ flexible
- Data structures สำหรับจัดการ todos
- JSON persistence
- Color terminal output
- Search และ filter
- Error handling ที่ครบถ้วน
- Unit tests ครอบคลุม

---
*ต่อไป Part 62: HTTP Server สมบูรณ์*
