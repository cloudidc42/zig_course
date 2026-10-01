# ส่วนที่ 70: โปรเจกต์ - Simple Shell

## ขั้นตอนที่ 691-700: สร้าง Shell ง่ายๆ ด้วย Zig

---

## ขั้นตอนที่ 691: Shell Architecture

Shell คือ command interpreter ที่รับ input จากผู้ใช้และ execute commands

### ความสามารถที่จะสร้าง

1. **Input**: Line editing ด้วย readline-style
2. **Tokenization**: แยก command และ arguments
3. **PATH Resolution**: หา executable
4. **Process Execution**: fork/exec
5. **Pipes**: command1 | command2
6. **Redirection**: >, <, >>
7. **Built-ins**: cd, pwd, echo, exit, env
8. **Environment Variables**: get/set
9. **Job Control**: background jobs

### โครงสร้างโปรเจกต์

```
zigsh/
├── src/
│   ├── main.zig
│   ├── shell.zig
│   ├── lexer.zig
│   ├── executor.zig
│   ├── builtins.zig
│   ├── job.zig
│   └── readline.zig
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
        .name = "zigsh",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the shell");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/lexer.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

### ไฟล์ src/shell.zig - โครงสร้างหลัก

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;
const lexer_mod = @import("lexer.zig");
const executor_mod = @import("executor.zig");
const builtins_mod = @import("builtins.zig");
const job_mod = @import("job.zig");
const readline_mod = @import("readline.zig");

pub const ShellConfig = struct {
    prompt: []const u8 = "zigsh$ ",
    history_file: ?[]const u8 = null,
    history_size: usize = 1000,
    interactive: bool = true,
};

pub const Shell = struct {
    allocator: Allocator,
    config: ShellConfig,
    env: std.StringHashMap([]u8),
    jobs: job_mod.JobManager,
    history: std.ArrayList([]u8),
    exit_code: u8,
    running: bool,

    const Self = @This();

    pub fn init(allocator: Allocator, config: ShellConfig) !Self {
        var shell = Self{
            .allocator = allocator,
            .config = config,
            .env = std.StringHashMap([]u8).init(allocator),
            .jobs = job_mod.JobManager.init(allocator),
            .history = std.ArrayList([]u8).init(allocator),
            .exit_code = 0,
            .running = true,
        };

        // Import environment variables จาก parent process
        try shell.importEnv();

        return shell;
    }

    pub fn deinit(self: *Self) void {
        var iter = self.env.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.env.deinit();
        self.jobs.deinit();

        for (self.history.items) |item| self.allocator.free(item);
        self.history.deinit();
    }

    fn importEnv(self: *Self) !void {
        var env_map = try std.process.getEnvMap(self.allocator);
        defer env_map.deinit();

        var iter = env_map.iterator();
        while (iter.next()) |entry| {
            const key = try self.allocator.dupe(u8, entry.key_ptr.*);
            errdefer self.allocator.free(key);
            const val = try self.allocator.dupe(u8, entry.value_ptr.*);
            errdefer self.allocator.free(val);
            try self.env.put(key, val);
        }
    }

    /// เพิ่ม command ลง history
    pub fn addHistory(self: *Self, cmd: []const u8) !void {
        const cmd_copy = try self.allocator.dupe(u8, cmd);
        try self.history.append(cmd_copy);

        // จำกัดขนาด history
        while (self.history.items.len > self.config.history_size) {
            const old = self.history.orderedRemove(0);
            self.allocator.free(old);
        }
    }

    /// ดึง environment variable
    pub fn getEnv(self: *Self, name: []const u8) ?[]const u8 {
        return self.env.get(name);
    }

    /// ตั้งค่า environment variable
    pub fn setEnv(self: *Self, name: []const u8, value: []const u8) !void {
        const key = try self.allocator.dupe(u8, name);
        errdefer self.allocator.free(key);
        const val = try self.allocator.dupe(u8, value);
        errdefer self.allocator.free(val);

        if (self.env.fetchRemove(name)) |old| {
            self.allocator.free(old.key);
            self.allocator.free(old.value);
        }

        try self.env.put(key, val);
    }

    /// ลบ environment variable
    pub fn unsetEnv(self: *Self, name: []const u8) void {
        if (self.env.fetchRemove(name)) |old| {
            self.allocator.free(old.key);
            self.allocator.free(old.value);
        }
    }
};
```

---

## ขั้นตอนที่ 692: Input Reading with Line Editing

```zig
// ไฟล์ src/readline.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const os = std.os;
const posix = std.posix;

/// Terminal mode สำหรับ raw input
pub const Terminal = struct {
    original_termios: posix.termios,
    is_raw: bool,
    fd: posix.fd_t,

    const Self = @This();

    pub fn init() !Self {
        const fd = posix.STDIN_FILENO;
        const original = try posix.tcgetattr(fd);

        return Self{
            .original_termios = original,
            .is_raw = false,
            .fd = fd,
        };
    }

    pub fn enableRaw(self: *Self) !void {
        if (self.is_raw) return;

        var raw = self.original_termios;

        // Input flags
        raw.iflag.BRKINT = false;
        raw.iflag.ICRNL = false;
        raw.iflag.INPCK = false;
        raw.iflag.ISTRIP = false;
        raw.iflag.IXON = false;

        // Output flags
        raw.oflag.OPOST = false;

        // Control flags
        raw.cflag.CSIZE = .CS8;

        // Local flags
        raw.lflag.ECHO = false;
        raw.lflag.ICANON = false;
        raw.lflag.IEXTEN = false;
        raw.lflag.ISIG = false;

        raw.cc[@intFromEnum(posix.V.MIN)] = 1;
        raw.cc[@intFromEnum(posix.V.TIME)] = 0;

        try posix.tcsetattr(self.fd, .FLUSH, raw);
        self.is_raw = true;
    }

    pub fn disableRaw(self: *Self) !void {
        if (!self.is_raw) return;
        try posix.tcsetattr(self.fd, .FLUSH, self.original_termios);
        self.is_raw = false;
    }

    pub fn deinit(self: *Self) void {
        self.disableRaw() catch {};
    }
};

/// Simple line editor
pub const LineEditor = struct {
    allocator: Allocator,
    history: []const []const u8,
    terminal: Terminal,

    const Self = @This();

    pub fn init(allocator: Allocator, history: []const []const u8) !Self {
        return Self{
            .allocator = allocator,
            .history = history,
            .terminal = try Terminal.init(),
        };
    }

    pub fn deinit(self: *Self) void {
        self.terminal.deinit();
    }

    /// อ่าน line พร้อม line editing
    pub fn readLine(self: *Self, prompt: []const u8) !?[]u8 {
        const stdout = std.io.getStdOut().writer();
        const stdin = std.io.getStdIn().reader();

        try stdout.writeAll(prompt);

        var buf = std.ArrayList(u8).init(self.allocator);
        errdefer buf.deinit();

        var cursor: usize = 0;
        var hist_idx: usize = self.history.len;

        // Enable raw mode
        try self.terminal.enableRaw();
        defer self.terminal.disableRaw() catch {};

        while (true) {
            var char_buf: [4]u8 = undefined;
            const n = try stdin.read(&char_buf);
            if (n == 0) continue;

            const ch = char_buf[0];

            switch (ch) {
                // Enter
                '\r', '\n' => {
                    try stdout.writeAll("\r\n");
                    return buf.toOwnedSlice();
                },

                // Ctrl+C
                3 => {
                    try stdout.writeAll("^C\r\n");
                    buf.clearRetainingCapacity();
                    cursor = 0;
                    try stdout.writeAll(prompt);
                },

                // Ctrl+D (EOF)
                4 => {
                    if (buf.items.len == 0) {
                        try stdout.writeAll("\r\n");
                        return null;
                    }
                },

                // Backspace / Delete
                127, 8 => {
                    if (cursor > 0) {
                        cursor -= 1;
                        _ = buf.orderedRemove(cursor);
                        // Redraw
                        try self.redrawLine(stdout, prompt, buf.items, cursor);
                    }
                },

                // Arrow keys (ESC sequences)
                27 => {
                    if (n >= 3 and char_buf[1] == '[') {
                        switch (char_buf[2]) {
                            'A' => { // Up arrow - previous history
                                if (hist_idx > 0) {
                                    hist_idx -= 1;
                                    buf.clearRetainingCapacity();
                                    try buf.appendSlice(self.history[hist_idx]);
                                    cursor = buf.items.len;
                                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                                }
                            },
                            'B' => { // Down arrow - next history
                                if (hist_idx < self.history.len) {
                                    hist_idx += 1;
                                    buf.clearRetainingCapacity();
                                    if (hist_idx < self.history.len) {
                                        try buf.appendSlice(self.history[hist_idx]);
                                    }
                                    cursor = buf.items.len;
                                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                                }
                            },
                            'C' => { // Right arrow
                                if (cursor < buf.items.len) {
                                    cursor += 1;
                                    try stdout.writeAll("\x1b[C");
                                }
                            },
                            'D' => { // Left arrow
                                if (cursor > 0) {
                                    cursor -= 1;
                                    try stdout.writeAll("\x1b[D");
                                }
                            },
                            else => {},
                        }
                    }
                },

                // Ctrl+A - ไปต้นบรรทัด
                1 => {
                    cursor = 0;
                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                },

                // Ctrl+E - ไปท้ายบรรทัด
                5 => {
                    cursor = buf.items.len;
                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                },

                // Ctrl+K - ลบจาก cursor ถึงท้าย
                11 => {
                    buf.shrinkRetainingCapacity(cursor);
                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                },

                // Printable characters
                32...126 => {
                    try buf.insert(cursor, ch);
                    cursor += 1;
                    try self.redrawLine(stdout, prompt, buf.items, cursor);
                },

                else => {},
            }
        }
    }

    fn redrawLine(
        self: *Self,
        writer: anytype,
        prompt: []const u8,
        line: []const u8,
        cursor: usize,
    ) !void {
        _ = self;
        // ไปต้นบรรทัด ลบทั้งหมด แล้วเขียนใหม่
        try writer.writeAll("\r\x1b[K");
        try writer.writeAll(prompt);
        try writer.writeAll(line);

        // ย้าย cursor ไปตำแหน่งที่ถูกต้อง
        if (cursor < line.len) {
            try writer.print("\x1b[{}D", .{line.len - cursor});
        }
    }
};

/// Simple readline โดยไม่ใช้ raw mode (fallback)
pub fn readLineSimple(allocator: Allocator, prompt: []const u8) !?[]u8 {
    const stdout = std.io.getStdOut().writer();
    try stdout.writeAll(prompt);

    const stdin = std.io.getStdIn().reader();
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();

    stdin.readUntilDelimiterArrayList(&buf, '\n', 65536) catch |err| {
        if (err == error.EndOfStream) {
            if (buf.items.len == 0) return null;
        } else {
            return err;
        }
    };

    // ลบ \r ถ้ามี
    while (buf.items.len > 0 and buf.items[buf.items.len - 1] == '\r') {
        _ = buf.pop();
    }

    return buf.toOwnedSlice();
}
```

---

## ขั้นตอนที่ 693: Command Tokenization

```zig
// ไฟล์ src/lexer.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

/// Token type สำหรับ shell
pub const TokenType = enum {
    word,           // command หรือ argument
    pipe,           // |
    redirect_in,    // <
    redirect_out,   // >
    redirect_append, // >>
    background,     // &
    semicolon,      // ;
    and_op,         // &&
    or_op,          // ||
    newline,
    eof,
};

pub const Token = struct {
    token_type: TokenType,
    value: []u8, // owning slice
    allocator: Allocator,

    pub fn deinit(self: *Token) void {
        self.allocator.free(self.value);
    }
};

/// Shell command tokenizer
pub const Lexer = struct {
    source: []const u8,
    pos: usize,
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator, source: []const u8) Self {
        return Self{
            .source = source,
            .pos = 0,
            .allocator = allocator,
        };
    }

    pub fn tokenize(self: *Self) ![]Token {
        var tokens = std.ArrayList(Token).init(self.allocator);
        errdefer {
            for (tokens.items) |*t| t.deinit();
            tokens.deinit();
        }

        while (true) {
            self.skipWhitespace();
            if (self.pos >= self.source.len) break;

            const token = try self.nextToken();
            if (token.token_type == .eof) {
                var t = token;
                t.deinit();
                break;
            }
            try tokens.append(token);
        }

        return tokens.toOwnedSlice();
    }

    fn nextToken(self: *Self) !Token {
        self.skipWhitespace();

        if (self.pos >= self.source.len) {
            return Token{
                .token_type = .eof,
                .value = try self.allocator.dupe(u8, ""),
                .allocator = self.allocator,
            };
        }

        const ch = self.source[self.pos];

        switch (ch) {
            '\n' => {
                self.pos += 1;
                return Token{
                    .token_type = .newline,
                    .value = try self.allocator.dupe(u8, "\n"),
                    .allocator = self.allocator,
                };
            },
            '|' => {
                self.pos += 1;
                if (self.pos < self.source.len and self.source[self.pos] == '|') {
                    self.pos += 1;
                    return Token{
                        .token_type = .or_op,
                        .value = try self.allocator.dupe(u8, "||"),
                        .allocator = self.allocator,
                    };
                }
                return Token{
                    .token_type = .pipe,
                    .value = try self.allocator.dupe(u8, "|"),
                    .allocator = self.allocator,
                };
            },
            '<' => {
                self.pos += 1;
                return Token{
                    .token_type = .redirect_in,
                    .value = try self.allocator.dupe(u8, "<"),
                    .allocator = self.allocator,
                };
            },
            '>' => {
                self.pos += 1;
                if (self.pos < self.source.len and self.source[self.pos] == '>') {
                    self.pos += 1;
                    return Token{
                        .token_type = .redirect_append,
                        .value = try self.allocator.dupe(u8, ">>"),
                        .allocator = self.allocator,
                    };
                }
                return Token{
                    .token_type = .redirect_out,
                    .value = try self.allocator.dupe(u8, ">"),
                    .allocator = self.allocator,
                };
            },
            '&' => {
                self.pos += 1;
                if (self.pos < self.source.len and self.source[self.pos] == '&') {
                    self.pos += 1;
                    return Token{
                        .token_type = .and_op,
                        .value = try self.allocator.dupe(u8, "&&"),
                        .allocator = self.allocator,
                    };
                }
                return Token{
                    .token_type = .background,
                    .value = try self.allocator.dupe(u8, "&"),
                    .allocator = self.allocator,
                };
            },
            ';' => {
                self.pos += 1;
                return Token{
                    .token_type = .semicolon,
                    .value = try self.allocator.dupe(u8, ";"),
                    .allocator = self.allocator,
                };
            },
            '"', '\'' => {
                return self.lexQuotedWord();
            },
            else => {
                return self.lexWord();
            },
        }
    }

    fn lexWord(self: *Self) !Token {
        var word = std.ArrayList(u8).init(self.allocator);
        defer word.deinit();

        while (self.pos < self.source.len) {
            const ch = self.source[self.pos];

            switch (ch) {
                ' ', '\t', '\n', '|', '<', '>', '&', ';' => break,
                '\\' => {
                    self.pos += 1;
                    if (self.pos < self.source.len) {
                        try word.append(self.source[self.pos]);
                        self.pos += 1;
                    }
                },
                '$' => {
                    self.pos += 1;
                    const var_name = try self.readVarName();
                    defer self.allocator.free(var_name);
                    // Variable expansion จะทำใน executor
                    try word.append('$');
                    try word.appendSlice(var_name);
                },
                else => {
                    try word.append(ch);
                    self.pos += 1;
                },
            }
        }

        return Token{
            .token_type = .word,
            .value = try word.toOwnedSlice(),
            .allocator = self.allocator,
        };
    }

    fn lexQuotedWord(self: *Self) !Token {
        const quote = self.source[self.pos];
        self.pos += 1;

        var word = std.ArrayList(u8).init(self.allocator);
        defer word.deinit();

        while (self.pos < self.source.len and self.source[self.pos] != quote) {
            if (self.source[self.pos] == '\\' and quote == '"') {
                self.pos += 1;
                if (self.pos < self.source.len) {
                    try word.append(self.source[self.pos]);
                    self.pos += 1;
                }
            } else {
                try word.append(self.source[self.pos]);
                self.pos += 1;
            }
        }

        if (self.pos < self.source.len) self.pos += 1; // closing quote

        return Token{
            .token_type = .word,
            .value = try word.toOwnedSlice(),
            .allocator = self.allocator,
        };
    }

    fn readVarName(self: *Self) ![]u8 {
        if (self.pos >= self.source.len) return self.allocator.dupe(u8, "");

        if (self.source[self.pos] == '{') {
            self.pos += 1;
            const start = self.pos;
            while (self.pos < self.source.len and self.source[self.pos] != '}') {
                self.pos += 1;
            }
            const name = self.source[start..self.pos];
            if (self.pos < self.source.len) self.pos += 1;
            return self.allocator.dupe(u8, name);
        }

        const start = self.pos;
        while (self.pos < self.source.len and
            (std.ascii.isAlphanumeric(self.source[self.pos]) or self.source[self.pos] == '_'))
        {
            self.pos += 1;
        }
        return self.allocator.dupe(u8, self.source[start..self.pos]);
    }

    fn skipWhitespace(self: *Self) void {
        while (self.pos < self.source.len and
            (self.source[self.pos] == ' ' or self.source[self.pos] == '\t'))
        {
            self.pos += 1;
        }
    }
};

test "Lexer basic" {
    const allocator = std.testing.allocator;
    var lexer = Lexer.init(allocator, "ls -la /tmp");
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        allocator.free(tokens);
    }

    try std.testing.expect(tokens.len == 3);
    try std.testing.expectEqualStrings("ls", tokens[0].value);
    try std.testing.expectEqualStrings("-la", tokens[1].value);
    try std.testing.expectEqualStrings("/tmp", tokens[2].value);
}

test "Lexer pipe" {
    const allocator = std.testing.allocator;
    var lexer = Lexer.init(allocator, "cat file.txt | grep hello");
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        allocator.free(tokens);
    }

    try std.testing.expect(tokens.len == 5);
    try std.testing.expect(tokens[2].token_type == .pipe);
}
```

---

## ขั้นตอนที่ 694: PATH Resolution

```zig
// ต่อจาก executor.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

/// หา executable ใน PATH
pub fn resolvePath(allocator: Allocator, cmd: []const u8, env_path: ?[]const u8) !?[]u8 {
    // ถ้าเป็น absolute path หรือ relative path
    if (std.mem.startsWith(u8, cmd, "/") or
        std.mem.startsWith(u8, cmd, "./") or
        std.mem.startsWith(u8, cmd, "../"))
    {
        // ตรวจสอบว่าไฟล์มีอยู่และ executable
        const stat = std.fs.cwd().statFile(cmd) catch return null;
        _ = stat;
        return allocator.dupe(u8, cmd);
    }

    // ค้นหาใน PATH
    const path_env = env_path orelse "/usr/bin:/bin:/usr/local/bin";

    var iter = std.mem.tokenizeScalar(u8, path_env, ':');
    while (iter.next()) |dir| {
        const full_path = try std.fmt.allocPrint(allocator, "{s}/{s}", .{ dir, cmd });
        errdefer allocator.free(full_path);

        // ตรวจสอบว่าไฟล์มีอยู่
        std.fs.accessAbsolute(full_path, .{ .mode = .execute }) catch {
            allocator.free(full_path);
            continue;
        };

        return full_path;
    }

    return null;
}

/// Command ที่ parse แล้ว
pub const Command = struct {
    argv: [][]u8,
    stdin_file: ?[]u8,   // null = inherit
    stdout_file: ?[]u8,  // null = inherit
    stdout_append: bool,
    background: bool,
    allocator: Allocator,

    pub fn deinit(self: *Command) void {
        for (self.argv) |arg| self.allocator.free(arg);
        self.allocator.free(self.argv);
        if (self.stdin_file) |f| self.allocator.free(f);
        if (self.stdout_file) |f| self.allocator.free(f);
    }
};

/// Pipeline ของ commands
pub const Pipeline = struct {
    commands: []Command,
    background: bool,
    allocator: Allocator,

    pub fn deinit(self: *Pipeline) void {
        for (self.commands) |*cmd| cmd.deinit();
        self.allocator.free(self.commands);
    }
};

/// Parse tokens เป็น pipeline
pub fn parsePipeline(allocator: Allocator, tokens: []const @import("lexer.zig").Token) !Pipeline {
    var commands = std.ArrayList(Command).init(allocator);
    errdefer {
        for (commands.items) |*cmd| cmd.deinit();
        commands.deinit();
    }

    var current_args = std.ArrayList([]u8).init(allocator);
    defer current_args.deinit();

    var stdin_file: ?[]u8 = null;
    var stdout_file: ?[]u8 = null;
    var stdout_append = false;
    var background = false;

    var i: usize = 0;
    while (i < tokens.len) : (i += 1) {
        const token = tokens[i];

        switch (token.token_type) {
            .word => {
                const arg = try allocator.dupe(u8, token.value);
                try current_args.append(arg);
            },
            .pipe => {
                if (current_args.items.len > 0) {
                    try commands.append(Command{
                        .argv = try current_args.toOwnedSlice(),
                        .stdin_file = stdin_file,
                        .stdout_file = stdout_file,
                        .stdout_append = stdout_append,
                        .background = false,
                        .allocator = allocator,
                    });
                    current_args = std.ArrayList([]u8).init(allocator);
                    stdin_file = null;
                    stdout_file = null;
                    stdout_append = false;
                }
            },
            .redirect_in => {
                i += 1;
                if (i < tokens.len) {
                    stdin_file = try allocator.dupe(u8, tokens[i].value);
                }
            },
            .redirect_out => {
                i += 1;
                if (i < tokens.len) {
                    if (stdout_file) |old| allocator.free(old);
                    stdout_file = try allocator.dupe(u8, tokens[i].value);
                    stdout_append = false;
                }
            },
            .redirect_append => {
                i += 1;
                if (i < tokens.len) {
                    if (stdout_file) |old| allocator.free(old);
                    stdout_file = try allocator.dupe(u8, tokens[i].value);
                    stdout_append = true;
                }
            },
            .background => {
                background = true;
            },
            else => {},
        }
    }

    if (current_args.items.len > 0) {
        try commands.append(Command{
            .argv = try current_args.toOwnedSlice(),
            .stdin_file = stdin_file,
            .stdout_file = stdout_file,
            .stdout_append = stdout_append,
            .background = background,
            .allocator = allocator,
        });
    }

    return Pipeline{
        .commands = try commands.toOwnedSlice(),
        .background = background,
        .allocator = allocator,
    };
}
```

---

## ขั้นตอนที่ 695: Process Execution (fork/exec)

```zig
// ไฟล์ src/executor.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const posix = std.posix;
const builtins_mod = @import("builtins.zig");
const shell_mod = @import("shell.zig");

/// Execute single command
pub fn executeCommand(
    shell: *shell_mod.Shell,
    cmd: *Command,
    stdin_fd: posix.fd_t,
    stdout_fd: posix.fd_t,
) !posix.pid_t {
    // ตรวจสอบ built-in commands ก่อน
    if (builtins_mod.isBuiltin(cmd.argv[0])) {
        // Built-ins รันใน parent process
        const exit_code = try builtins_mod.executeBuiltin(shell, cmd.argv);
        shell.exit_code = exit_code;
        return 0; // ไม่มี child PID
    }

    // หา executable
    const path_env = shell.getEnv("PATH");
    const exe_path = try resolvePath(shell.allocator, cmd.argv[0], path_env) orelse {
        try std.io.getStdErr().writer().print(
            "zigsh: {s}: command not found\n",
            .{cmd.argv[0]},
        );
        shell.exit_code = 127;
        return error.CommandNotFound;
    };
    defer shell.allocator.free(exe_path);

    // Fork
    const pid = try posix.fork();

    if (pid == 0) {
        // Child process

        // Set up stdin
        if (stdin_fd != posix.STDIN_FILENO) {
            try posix.dup2(stdin_fd, posix.STDIN_FILENO);
            posix.close(stdin_fd);
        }

        // Set up stdout
        if (stdout_fd != posix.STDOUT_FILENO) {
            try posix.dup2(stdout_fd, posix.STDOUT_FILENO);
            posix.close(stdout_fd);
        }

        // Open stdin file ถ้ามี
        if (cmd.stdin_file) |in_file| {
            const fd = try posix.open(in_file, .{ .ACCMODE = .RDONLY }, 0);
            try posix.dup2(fd, posix.STDIN_FILENO);
            posix.close(fd);
        }

        // Open stdout file ถ้ามี
        if (cmd.stdout_file) |out_file| {
            const flags = posix.O{
                .ACCMODE = .WRONLY,
                .CREAT = true,
                .TRUNC = !cmd.stdout_append,
                .APPEND = cmd.stdout_append,
            };
            const fd = try posix.open(out_file, flags, 0o644);
            try posix.dup2(fd, posix.STDOUT_FILENO);
            posix.close(fd);
        }

        // สร้าง argv สำหรับ execve
        var argv_ptrs = try shell.allocator.alloc(?[*:0]const u8, cmd.argv.len + 1);
        for (cmd.argv, 0..) |arg, idx| {
            argv_ptrs[idx] = try shell.allocator.dupeZ(u8, arg);
        }
        argv_ptrs[cmd.argv.len] = null;

        // สร้าง envp
        var envp_list = std.ArrayList(?[*:0]const u8).init(shell.allocator);
        var env_iter = shell.env.iterator();
        while (env_iter.next()) |entry| {
            const env_str = try std.fmt.allocPrintZ(
                shell.allocator,
                "{s}={s}",
                .{ entry.key_ptr.*, entry.value_ptr.* },
            );
            try envp_list.append(env_str);
        }
        try envp_list.append(null);

        const exe_path_z = try shell.allocator.dupeZ(u8, exe_path);

        const err = posix.execveZ(exe_path_z, @ptrCast(argv_ptrs.ptr), @ptrCast(envp_list.items.ptr));
        std.debug.print("zigsh: exec failed: {}\n", .{err});
        posix.exit(1);
    }

    return pid;
}

/// Execute pipeline
pub fn executePipeline(shell: *shell_mod.Shell, pipeline: *Pipeline) !u8 {
    const n = pipeline.commands.len;
    if (n == 0) return 0;

    // สร้าง pipes
    var pipes = try shell.allocator.alloc([2]posix.fd_t, n - 1);
    defer shell.allocator.free(pipes);

    for (pipes) |*pipe| {
        pipe.* = try posix.pipe();
    }

    var pids = try shell.allocator.alloc(posix.pid_t, n);
    defer shell.allocator.free(pids);

    for (pipeline.commands, 0..) |*cmd, i| {
        const stdin_fd = if (i == 0) posix.STDIN_FILENO else pipes[i - 1][0];
        const stdout_fd = if (i == n - 1) posix.STDOUT_FILENO else pipes[i][1];

        pids[i] = executeCommand(shell, cmd, stdin_fd, stdout_fd) catch 0;

        // ปิด pipe fds ใน parent
        if (i > 0) posix.close(pipes[i - 1][0]);
        if (i < n - 1) posix.close(pipes[i][1]);
    }

    // รอ child processes ถ้าไม่ใช่ background
    var exit_code: u8 = 0;
    if (!pipeline.background) {
        for (pids) |pid| {
            if (pid > 0) {
                const result = posix.waitpid(pid, 0);
                if (result.pid == pid) {
                    exit_code = @intCast(posix.W.EXITSTATUS(result.status));
                }
            }
        }
    } else {
        std.debug.print("[bg] pids:", .{});
        for (pids) |pid| {
            if (pid > 0) std.debug.print(" {}", .{pid});
        }
        std.debug.print("\n", .{});
    }

    return exit_code;
}
```

---

## ขั้นตอนที่ 696: Pipe Operator Support

(ดูใน executor.zig ด้านบน - executePipeline)

```zig
// เพิ่มเติม - การจัดการ pipes ขั้นสูง

/// สร้าง pipe pair
fn createPipe() !struct { read: posix.fd_t, write: posix.fd_t } {
    const fds = try posix.pipe();
    return .{ .read = fds[0], .write = fds[1] };
}

/// ตัวอย่างการ run pipeline: ls | grep .zig | wc -l
pub fn runPipelineExample(allocator: Allocator) !void {
    _ = allocator;
    // สร้าง pipes
    const pipe1 = try createPipe();
    const pipe2 = try createPipe();

    // Fork process 1: ls
    const pid1 = try posix.fork();
    if (pid1 == 0) {
        posix.close(pipe1.read);
        posix.close(pipe2.read);
        posix.close(pipe2.write);
        try posix.dup2(pipe1.write, posix.STDOUT_FILENO);
        posix.close(pipe1.write);
        _ = posix.execvpeZ("ls", &[_:null]?[*:0]const u8{
            "ls", null,
        }, @ptrFromInt(0)) catch {};
        posix.exit(1);
    }
    posix.close(pipe1.write);

    // Fork process 2: grep .zig
    const pid2 = try posix.fork();
    if (pid2 == 0) {
        posix.close(pipe2.read);
        try posix.dup2(pipe1.read, posix.STDIN_FILENO);
        posix.close(pipe1.read);
        try posix.dup2(pipe2.write, posix.STDOUT_FILENO);
        posix.close(pipe2.write);
        _ = posix.execvpeZ("grep", &[_:null]?[*:0]const u8{
            "grep", ".zig", null,
        }, @ptrFromInt(0)) catch {};
        posix.exit(1);
    }
    posix.close(pipe1.read);
    posix.close(pipe2.write);

    // Fork process 3: wc -l
    const pid3 = try posix.fork();
    if (pid3 == 0) {
        try posix.dup2(pipe2.read, posix.STDIN_FILENO);
        posix.close(pipe2.read);
        _ = posix.execvpeZ("wc", &[_:null]?[*:0]const u8{
            "wc", "-l", null,
        }, @ptrFromInt(0)) catch {};
        posix.exit(1);
    }
    posix.close(pipe2.read);

    // รอทุก processes
    _ = posix.waitpid(pid1, 0);
    _ = posix.waitpid(pid2, 0);
    _ = posix.waitpid(pid3, 0);
}
```

---

## ขั้นตอนที่ 697: Redirection (>, <, >>)

(ดูใน executor.zig - executeCommand)

```zig
// เพิ่มเติม - heredoc และ process substitution

/// Here-document (<<)
pub fn setupHeredoc(allocator: Allocator, content: []const u8) !posix.fd_t {
    const fds = try posix.pipe();
    const write_fd = fds[1];
    const read_fd = fds[0];

    // เขียน content ลง pipe
    var written: usize = 0;
    while (written < content.len) {
        const n = try posix.write(write_fd, content[written..]);
        written += n;
    }
    posix.close(write_fd);
    _ = allocator;

    return read_fd;
}

/// ทดสอบ redirections
pub fn testRedirections() !void {
    // Test output redirection: echo hello > /tmp/test.txt
    const out_file = try posix.open(
        "/tmp/test.txt",
        .{ .ACCMODE = .WRONLY, .CREAT = true, .TRUNC = true },
        0o644,
    );
    defer posix.close(out_file);

    const saved_stdout = try posix.dup(posix.STDOUT_FILENO);
    defer posix.close(saved_stdout);

    try posix.dup2(out_file, posix.STDOUT_FILENO);
    _ = try posix.write(posix.STDOUT_FILENO, "hello from zigsh\n");
    try posix.dup2(saved_stdout, posix.STDOUT_FILENO);

    std.debug.print("Output written to /tmp/test.txt\n", .{});
}
```

---

## ขั้นตอนที่ 698: Built-in Commands

```zig
// ไฟล์ src/builtins.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const posix = std.posix;
const shell_mod = @import("shell.zig");

const BUILTINS = [_][]const u8{
    "cd", "pwd", "echo", "exit", "export", "unset",
    "env", "type", "history", "jobs", "fg", "bg", "kill",
};

pub fn isBuiltin(name: []const u8) bool {
    for (BUILTINS) |builtin| {
        if (std.mem.eql(u8, name, builtin)) return true;
    }
    return false;
}

pub fn executeBuiltin(shell: *shell_mod.Shell, argv: []const []u8) !u8 {
    if (argv.len == 0) return 0;

    const cmd = argv[0];

    if (std.mem.eql(u8, cmd, "cd")) return builtinCd(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "pwd")) return builtinPwd(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "echo")) return builtinEcho(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "exit")) return builtinExit(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "export")) return builtinExport(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "unset")) return builtinUnset(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "env")) return builtinEnv(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "type")) return builtinType(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "history")) return builtinHistory(shell, argv[1..]);
    if (std.mem.eql(u8, cmd, "jobs")) return builtinJobs(shell, argv[1..]);

    return 1; // ไม่พบ builtin
}

fn builtinCd(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    const stdout = std.io.getStdErr().writer();

    const target = blk: {
        if (args.len == 0) {
            break :blk shell.getEnv("HOME") orelse "/";
        }
        if (std.mem.eql(u8, args[0], "-")) {
            break :blk shell.getEnv("OLDPWD") orelse {
                try stdout.writeAll("zigsh: cd: OLDPWD not set\n");
                return 1;
            };
        }
        break :blk args[0];
    };

    // บันทึก current directory
    var cwd_buf: [std.fs.max_path_bytes]u8 = undefined;
    const cwd = posix.getcwd(&cwd_buf) catch "/";

    // เปลี่ยน directory
    posix.chdir(target) catch |err| {
        try stdout.print("zigsh: cd: {s}: {}\n", .{ target, err });
        return 1;
    };

    // อัปเดต environment
    try shell.setEnv("OLDPWD", cwd);

    var new_cwd_buf: [std.fs.max_path_bytes]u8 = undefined;
    const new_cwd = posix.getcwd(&new_cwd_buf) catch target;
    try shell.setEnv("PWD", new_cwd);

    return 0;
}

fn builtinPwd(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    _ = args;
    _ = shell;
    const stdout = std.io.getStdOut().writer();

    var buf: [std.fs.max_path_bytes]u8 = undefined;
    const cwd = posix.getcwd(&buf) catch {
        try stdout.writeAll("zigsh: pwd: error\n");
        return 1;
    };

    try stdout.print("{s}\n", .{cwd});
    return 0;
}

fn builtinEcho(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    const stdout = std.io.getStdOut().writer();
    _ = shell;

    var no_newline = false;
    var start: usize = 0;

    if (args.len > 0 and std.mem.eql(u8, args[0], "-n")) {
        no_newline = true;
        start = 1;
    }

    for (args[start..], 0..) |arg, i| {
        if (i > 0) try stdout.writeByte(' ');

        // Process escape sequences
        var j: usize = 0;
        while (j < arg.len) : (j += 1) {
            if (arg[j] == '\\' and j + 1 < arg.len) {
                j += 1;
                switch (arg[j]) {
                    'n' => try stdout.writeByte('\n'),
                    't' => try stdout.writeByte('\t'),
                    'r' => try stdout.writeByte('\r'),
                    '\\' => try stdout.writeByte('\\'),
                    else => {
                        try stdout.writeByte('\\');
                        try stdout.writeByte(arg[j]);
                    },
                }
            } else {
                try stdout.writeByte(arg[j]);
            }
        }
    }

    if (!no_newline) try stdout.writeByte('\n');
    return 0;
}

fn builtinExit(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    const code: u8 = if (args.len > 0)
        std.fmt.parseInt(u8, args[0], 10) catch shell.exit_code
    else
        shell.exit_code;

    shell.running = false;
    shell.exit_code = code;
    return code;
}

fn builtinExport(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    const stderr = std.io.getStdErr().writer();

    for (args) |arg| {
        if (std.mem.indexOf(u8, arg, "=")) |eq_pos| {
            const name = arg[0..eq_pos];
            const value = arg[eq_pos + 1 ..];
            try shell.setEnv(name, value);
        } else {
            try stderr.print("zigsh: export: {s}: invalid argument\n", .{arg});
            return 1;
        }
    }
    return 0;
}

fn builtinUnset(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    for (args) |name| {
        shell.unsetEnv(name);
    }
    return 0;
}

fn builtinEnv(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    _ = args;
    const stdout = std.io.getStdOut().writer();

    var iter = shell.env.iterator();
    while (iter.next()) |entry| {
        try stdout.print("{s}={s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
    return 0;
}

fn builtinType(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    const stdout = std.io.getStdOut().writer();

    for (args) |cmd| {
        if (isBuiltin(cmd)) {
            try stdout.print("{s} is a shell builtin\n", .{cmd});
        } else {
            const path_env = shell.getEnv("PATH");
            if (try resolvePath(shell.allocator, cmd, path_env)) |path| {
                defer shell.allocator.free(path);
                try stdout.print("{s} is {s}\n", .{ cmd, path });
            } else {
                try stdout.print("{s}: not found\n", .{cmd});
            }
        }
    }
    return 0;
}

fn builtinHistory(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    _ = args;
    const stdout = std.io.getStdOut().writer();

    for (shell.history.items, 1..) |item, i| {
        try stdout.print("{:4}  {s}\n", .{ i, item });
    }
    return 0;
}

fn builtinJobs(shell: *shell_mod.Shell, args: []const []u8) !u8 {
    _ = args;
    return shell.jobs.listJobs();
}
```

---

## ขั้นตอนที่ 699: Environment Variables

```zig
// ไฟล์ src/job.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const posix = std.posix;

pub const JobStatus = enum {
    running,
    stopped,
    done,
};

pub const Job = struct {
    id: usize,
    pid: posix.pid_t,
    command: []u8,
    status: JobStatus,
    allocator: Allocator,

    pub fn deinit(self: *Job) void {
        self.allocator.free(self.command);
    }
};

pub const JobManager = struct {
    allocator: Allocator,
    jobs: std.ArrayList(Job),
    next_id: usize,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{
            .allocator = allocator,
            .jobs = std.ArrayList(Job).init(allocator),
            .next_id = 1,
        };
    }

    pub fn deinit(self: *Self) void {
        for (self.jobs.items) |*job| job.deinit();
        self.jobs.deinit();
    }

    pub fn addJob(self: *Self, pid: posix.pid_t, command: []const u8) !*Job {
        const id = self.next_id;
        self.next_id += 1;

        try self.jobs.append(Job{
            .id = id,
            .pid = pid,
            .command = try self.allocator.dupe(u8, command),
            .status = .running,
            .allocator = self.allocator,
        });

        return &self.jobs.items[self.jobs.items.len - 1];
    }

    pub fn removeJob(self: *Self, id: usize) void {
        for (self.jobs.items, 0..) |*job, i| {
            if (job.id == id) {
                job.deinit();
                _ = self.jobs.orderedRemove(i);
                return;
            }
        }
    }

    pub fn updateJobs(self: *Self) void {
        var i: usize = 0;
        while (i < self.jobs.items.len) {
            const job = &self.jobs.items[i];
            // ตรวจสอบสถานะ non-blocking
            const result = posix.waitpid(job.pid, posix.W.NOHANG);
            if (result.pid == job.pid) {
                std.debug.print("[{}] Done: {s}\n", .{ job.id, job.command });
                job.deinit();
                _ = self.jobs.orderedRemove(i);
            } else {
                i += 1;
            }
        }
    }

    pub fn listJobs(self: *Self) !u8 {
        const stdout = std.io.getStdOut().writer();
        for (self.jobs.items) |job| {
            const status_str = switch (job.status) {
                .running => "Running",
                .stopped => "Stopped",
                .done => "Done",
            };
            try stdout.print("[{}] {} {s:10} {s}\n", .{
                job.id,
                job.pid,
                status_str,
                job.command,
            });
        }
        return 0;
    }
};
```

---

## ขั้นตอนที่ 700: Job Control Basics

```zig
// ไฟล์ src/main.zig - Shell หลัก

const std = @import("std");
const posix = std.posix;
const Allocator = std.mem.Allocator;
const shell_mod = @import("shell.zig");
const lexer_mod = @import("lexer.zig");
const executor_mod = @import("executor.zig");
const readline_mod = @import("readline.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    var shell = try shell_mod.Shell.init(allocator, .{
        .prompt = "zigsh$ ",
        .interactive = true,
    });
    defer shell.deinit();

    if (args.len > 1) {
        // Non-interactive mode: รัน script file
        return runScript(&shell, args[1]);
    }

    // Interactive mode
    try runInteractive(&shell);
}

fn runInteractive(shell: *shell_mod.Shell) !void {
    const stdin_isatty = posix.isatty(posix.STDIN_FILENO);

    while (shell.running) {
        // อัปเดต job statuses
        shell.jobs.updateJobs();

        // สร้าง prompt
        var prompt_buf: [256]u8 = undefined;
        const cwd_buf_inner = [_]u8{0} ** std.fs.max_path_bytes;
        var cwd_buf = cwd_buf_inner;
        const cwd = posix.getcwd(&cwd_buf) catch "?";

        const prompt = std.fmt.bufPrint(
            &prompt_buf,
            "{s}:{s}$ ",
            .{ shell.getEnv("USER") orelse "user", cwd },
        ) catch shell.config.prompt;

        // อ่าน input
        const line = blk: {
            if (stdin_isatty) {
                break :blk try readline_mod.readLineSimple(shell.allocator, prompt);
            } else {
                break :blk try readline_mod.readLineSimple(shell.allocator, "");
            }
        };

        if (line == null) {
            // EOF
            if (stdin_isatty) {
                std.debug.print("\nexit\n", .{});
            }
            break;
        }
        defer shell.allocator.free(line.?);

        const trimmed = std.mem.trim(u8, line.?, " \t");
        if (trimmed.len == 0) continue;

        // เพิ่มลง history
        try shell.addHistory(trimmed);

        // Execute command
        executeLine(shell, trimmed) catch |err| {
            std.io.getStdErr().writer().print("zigsh: {}\n", .{err}) catch {};
        };
    }
}

fn runScript(shell: *shell_mod.Shell, path: []const u8) !void {
    const file = try std.fs.openFileAbsolute(path, .{});
    defer file.close();

    const content = try file.readToEndAlloc(shell.allocator, 1024 * 1024);
    defer shell.allocator.free(content);

    var lines = std.mem.splitScalar(u8, content, '\n');
    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \t\r");
        if (trimmed.len == 0 or trimmed[0] == '#') continue;

        executeLine(shell, trimmed) catch |err| {
            std.io.getStdErr().writer().print("Error: {}\n", .{err}) catch {};
            if (shell.exit_code != 0) break;
        };
    }
}

fn executeLine(shell: *shell_mod.Shell, line: []const u8) !void {
    // ตรวจสอบ shell comments
    if (line[0] == '#') return;

    // Variable expansion: $VAR
    const expanded = try expandVariables(shell, line);
    defer shell.allocator.free(expanded);

    // Tokenize
    var lexer = lexer_mod.Lexer.init(shell.allocator, expanded);
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        shell.allocator.free(tokens);
    }

    if (tokens.len == 0) return;

    // Parse pipeline
    var pipeline = try executor_mod.parsePipeline(shell.allocator, tokens);
    defer pipeline.deinit();

    if (pipeline.commands.len == 0) return;

    // Execute
    const exit_code = try executor_mod.executePipeline(shell, &pipeline);
    shell.exit_code = exit_code;
}

fn expandVariables(shell: *shell_mod.Shell, line: []const u8) ![]u8 {
    var result = std.ArrayList(u8).init(shell.allocator);
    errdefer result.deinit();

    var i: usize = 0;
    while (i < line.len) {
        if (line[i] == '$') {
            i += 1;
            if (i >= line.len) {
                try result.append('$');
                continue;
            }

            // $?  - exit code
            if (line[i] == '?') {
                const code_str = try std.fmt.allocPrint(shell.allocator, "{}", .{shell.exit_code});
                defer shell.allocator.free(code_str);
                try result.appendSlice(code_str);
                i += 1;
                continue;
            }

            // $$ - PID
            if (line[i] == '$') {
                const pid_str = try std.fmt.allocPrint(shell.allocator, "{}", .{posix.getpid()});
                defer shell.allocator.free(pid_str);
                try result.appendSlice(pid_str);
                i += 1;
                continue;
            }

            // ${VAR} or $VAR
            var var_start = i;
            var var_end = i;
            var has_braces = false;

            if (line[i] == '{') {
                has_braces = true;
                i += 1;
                var_start = i;
                while (i < line.len and line[i] != '}') i += 1;
                var_end = i;
                if (i < line.len) i += 1;
            } else {
                while (i < line.len and
                    (std.ascii.isAlphanumeric(line[i]) or line[i] == '_'))
                {
                    i += 1;
                }
                var_end = i;
            }

            const var_name = line[var_start..var_end];
            if (shell.getEnv(var_name)) |val| {
                try result.appendSlice(val);
            }
            _ = has_braces;

        } else {
            try result.append(line[i]);
            i += 1;
        }
    }

    return result.toOwnedSlice();
}

test "Shell tokenization" {
    const allocator = std.testing.allocator;
    var lexer = lexer_mod.Lexer.init(allocator, "echo hello world");
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        allocator.free(tokens);
    }

    try std.testing.expect(tokens.len == 3);
    try std.testing.expectEqualStrings("echo", tokens[0].value);
    try std.testing.expectEqualStrings("hello", tokens[1].value);
    try std.testing.expectEqualStrings("world", tokens[2].value);
}

test "Shell pipe tokenization" {
    const allocator = std.testing.allocator;
    var lexer = lexer_mod.Lexer.init(allocator, "ls | grep .zig");
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        allocator.free(tokens);
    }

    try std.testing.expect(tokens.len == 4);
    try std.testing.expect(tokens[1].token_type == .pipe);
}

test "Shell redirect tokenization" {
    const allocator = std.testing.allocator;
    var lexer = lexer_mod.Lexer.init(allocator, "echo hello > /tmp/out.txt");
    const tokens = try lexer.tokenize();
    defer {
        for (tokens) |*t| {
            var tok = t.*;
            tok.deinit();
        }
        allocator.free(tokens);
    }

    try std.testing.expect(tokens.len == 4);
    try std.testing.expect(tokens[2].token_type == .redirect_out);
}

test "Variable expansion" {
    const allocator = std.testing.allocator;
    var shell = try shell_mod.Shell.init(allocator, .{});
    defer shell.deinit();

    try shell.setEnv("NAME", "World");
    const expanded = try expandVariables(&shell, "Hello $NAME!");
    defer allocator.free(expanded);

    try std.testing.expectEqualStrings("Hello World!", expanded);
}
```

---

## สรุปโปรเจกต์ Simple Shell

| ความสามารถ | รายละเอียด |
|-------------|-----------|
| Line Editing | History, cursor movement, Ctrl shortcuts |
| Tokenization | Words, pipes, redirections, operators |
| PATH Resolution | ค้นหา executables ใน PATH |
| Process Execution | fork/exec พร้อม stdin/stdout setup |
| Pipes | Multi-command pipelines |
| Redirections | >, <, >> |
| Built-ins | cd, pwd, echo, exit, export, unset, env, history, jobs |
| Environment | Import/export, variable expansion ($VAR) |
| Job Control | Background jobs (&), jobs listing |
| Scripts | รันไฟล์ script |

### Built-in Commands

| Command | รายละเอียด |
|---------|-----------|
| `cd [dir]` | เปลี่ยน directory |
| `pwd` | แสดง current directory |
| `echo [-n] [args]` | แสดงข้อความ |
| `exit [code]` | ออกจาก shell |
| `export VAR=val` | ตั้ง environment variable |
| `unset VAR` | ลบ environment variable |
| `env` | แสดง environment ทั้งหมด |
| `type cmd` | แสดงประเภทของ command |
| `history` | แสดง command history |
| `jobs` | แสดง background jobs |

### การรันโปรเจกต์

```bash
# Build
zig build

# Run interactive shell
./zig-out/bin/zigsh

# Run script
./zig-out/bin/zigsh script.sh

# Run tests
zig build test

# ตัวอย่างคำสั่งใน shell
zigsh$ echo Hello World
Hello World
zigsh$ ls -la | grep .zig
zigsh$ echo "test" > /tmp/out.txt
zigsh$ cat /tmp/out.txt
test
zigsh$ pwd
/home/user
zigsh$ export MYVAR=hello
zigsh$ echo $MYVAR
hello
zigsh$ sleep 10 &
[bg] pids: 12345
zigsh$ jobs
[1] 12345 Running     sleep 10
zigsh$ exit
```

### ข้อจำกัดและการพัฒนาต่อ

1. **Tab completion**: เพิ่ม tab completion สำหรับ files และ commands
2. **Aliases**: รองรับ command aliases
3. **Functions**: Shell functions
4. **Arrays**: Array variables
5. **Arithmetic**: `$(( expr ))` expansion
6. **Command substitution**: `$(command)`
7. **Job control**: fg, bg, kill, Ctrl+Z
8. **Signal handling**: SIGINT, SIGTERM, SIGCHLD
