# Part 100: Production Best Practices ใน Zig
## Steps 991-1000

---

## Step 991: Production Deployment Checklist

### Checklist ก่อน Deploy ขึ้น Production

```zig
// production_checklist.zig
const std = @import("std");

// Production Deployment Checklist:

// ============ Build ============
// [ ] Build ด้วย -Doptimize=ReleaseSafe หรือ ReleaseFast
// [ ] ไม่มี compile warnings
// [ ] zig fmt ผ่านทั้งหมด
// [ ] Tests ผ่านทั้งหมด ทุก platform ที่ deploy

// ============ Security ============
// [ ] Secrets ไม่อยู่ใน source code
// [ ] ใช้ environment variables สำหรับ secrets
// [ ] Input validation ครบทุก user input
// [ ] No buffer overflows (ใช้ safe operations)
// [ ] Dependencies มี known security vulnerabilities หรือไม่?

// ============ Performance ============
// [ ] Benchmark ผ่าน requirements
// [ ] Memory usage อยู่ในขอบเขต
// [ ] No memory leaks (ตรวจด้วย valgrind หรือ ASan)

// ============ Reliability ============
// [ ] Error handling ครบทุก code path
// [ ] Graceful shutdown implemented
// [ ] Health check endpoint
// [ ] Logging configured อย่างถูกต้อง

// ============ Operations ============
// [ ] Monitoring configured
// [ ] Alerting rules set
// [ ] Runbook เขียนแล้ว
// [ ] Rollback plan มี

pub fn main() void {
    std.debug.print("Production ready!\n", .{});
}
```

### Build Configurations สำหรับ Production

```zig
// production_build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Production settings
    if (optimize != .Debug) {
        // Strip debug symbols ลด binary size
        exe.root_module.strip = true;
        
        // Enable LTO (Link Time Optimization)
        exe.want_lto = true;
    }
    
    // Compile options
    exe.root_module.addCMacro("PRODUCTION", "1");
    
    b.installArtifact(exe);
}
```

### Release Build Commands

```bash
# ReleaseSafe: Optimize แต่ยัง check bounds/overflow
zig build -Doptimize=ReleaseSafe

# ReleaseFast: Maximum optimization (ไม่ check safety)  
zig build -Doptimize=ReleaseFast

# ReleaseSmall: Optimize สำหรับ binary size
zig build -Doptimize=ReleaseSmall

# Cross-compile สำหรับ Linux x86_64
zig build -Dtarget=x86_64-linux-gnu -Doptimize=ReleaseSafe

# Cross-compile สำหรับ ARM
zig build -Dtarget=aarch64-linux-gnu -Doptimize=ReleaseSafe

# Static binary (ไม่ depend on system libraries)
zig build -Dtarget=x86_64-linux-musl -Doptimize=ReleaseSafe
```

---

## Step 992: Configuration Management

### การจัดการ Configuration ใน Production

```zig
// config_management.zig
const std = @import("std");

/// Application configuration
pub const Config = struct {
    // Server settings
    host: []const u8 = "0.0.0.0",
    port: u16 = 8080,
    
    // Database settings
    db_url: []const u8,
    db_pool_size: u32 = 10,
    db_timeout_ms: u32 = 5000,
    
    // Cache settings
    redis_url: ?[]const u8 = null,
    cache_ttl_seconds: u32 = 300,
    
    // Feature flags
    enable_metrics: bool = true,
    enable_tracing: bool = false,
    
    // Logging
    log_level: LogLevel = .info,
    log_format: LogFormat = .json,
    
    pub const LogLevel = enum { debug, info, warn, err };
    pub const LogFormat = enum { text, json };
    
    /// โหลด config จาก environment variables
    pub fn fromEnv(allocator: std.mem.Allocator) !Config {
        var config = Config{
            .db_url = std.posix.getenv("DATABASE_URL") orelse
                return error.MissingDatabaseUrl,
        };
        
        // Optional settings with defaults
        if (std.posix.getenv("PORT")) |port_str| {
            config.port = try std.fmt.parseInt(u16, port_str, 10);
        }
        
        if (std.posix.getenv("HOST")) |host| {
            config.host = host;
        }
        
        if (std.posix.getenv("DB_POOL_SIZE")) |pool_str| {
            config.db_pool_size = try std.fmt.parseInt(u32, pool_str, 10);
        }
        
        if (std.posix.getenv("REDIS_URL")) |redis_url| {
            config.redis_url = redis_url;
        }
        
        if (std.posix.getenv("LOG_LEVEL")) |level_str| {
            config.log_level = parseLogLevel(level_str) orelse .info;
        }
        
        if (std.posix.getenv("LOG_FORMAT")) |format_str| {
            if (std.mem.eql(u8, format_str, "json")) {
                config.log_format = .json;
            } else if (std.mem.eql(u8, format_str, "text")) {
                config.log_format = .text;
            }
        }
        
        _ = allocator;
        return config;
    }
    
    fn parseLogLevel(s: []const u8) ?LogLevel {
        if (std.mem.eql(u8, s, "debug")) return .debug;
        if (std.mem.eql(u8, s, "info")) return .info;
        if (std.mem.eql(u8, s, "warn")) return .warn;
        if (std.mem.eql(u8, s, "error")) return .err;
        return null;
    }
    
    /// ตรวจสอบว่า config valid
    pub fn validate(self: Config) !void {
        if (self.port == 0) return error.InvalidPort;
        if (self.db_url.len == 0) return error.InvalidDatabaseUrl;
        if (self.db_pool_size == 0) return error.InvalidPoolSize;
    }
    
    /// Print config (ซ่อน secrets)
    pub fn print(self: Config) void {
        std.debug.print("Configuration:\n", .{});
        std.debug.print("  host:          {s}\n", .{self.host});
        std.debug.print("  port:          {d}\n", .{self.port});
        std.debug.print("  db_url:        {s}\n", .{maskSecret(self.db_url)});
        std.debug.print("  db_pool_size:  {d}\n", .{self.db_pool_size});
        std.debug.print("  redis_url:     {s}\n", .{
            if (self.redis_url) |url| maskSecret(url) else "not configured",
        });
        std.debug.print("  log_level:     {s}\n", .{@tagName(self.log_level)});
        std.debug.print("  enable_metrics:{}\n", .{self.enable_metrics});
    }
    
    fn maskSecret(s: []const u8) []const u8 {
        // แสดงแค่ scheme และ host ซ่อน password
        // "postgres://user:password@host/db" -> "postgres://***@host/db"
        return if (s.len > 8) "***" else s;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    const config = Config.fromEnv(gpa.allocator()) catch |err| {
        std.debug.print("Configuration error: {}\n", .{err});
        std.process.exit(1);
    };
    
    config.validate() catch |err| {
        std.debug.print("Invalid configuration: {}\n", .{err});
        std.process.exit(1);
    };
    
    config.print();
}
```

### Config File Support

```zig
// config_file.zig
const std = @import("std");

pub const ConfigFile = struct {
    server: ServerConfig = .{},
    database: DatabaseConfig,
    
    const ServerConfig = struct {
        host: []const u8 = "0.0.0.0",
        port: u16 = 8080,
    };
    
    const DatabaseConfig = struct {
        url: []const u8,
        pool_size: u32 = 10,
    };
    
    pub fn loadFromFile(path: []const u8, allocator: std.mem.Allocator) !ConfigFile {
        const file = try std.fs.cwd().openFile(path, .{});
        defer file.close();
        
        const content = try file.readToEndAlloc(allocator, 1 * 1024 * 1024);
        defer allocator.free(content);
        
        // Parse JSON config file
        const parsed = try std.json.parseFromSlice(ConfigFile, allocator, content, .{
            .ignore_unknown_fields = true,
        });
        defer parsed.deinit();
        
        return parsed.value;
    }
};
```

---

## Step 993: Logging ใน Production

### Structured Logging

```zig
// structured_logging.zig
const std = @import("std");

pub const Logger = struct {
    level: Level,
    format: Format,
    writer: std.io.AnyWriter,
    
    pub const Level = enum(u8) {
        debug = 0,
        info = 1,
        warn = 2,
        err = 3,
        
        pub fn fromString(s: []const u8) Level {
            if (std.mem.eql(u8, s, "debug")) return .debug;
            if (std.mem.eql(u8, s, "warn")) return .warn;
            if (std.mem.eql(u8, s, "error")) return .err;
            return .info;
        }
    };
    
    pub const Format = enum { text, json };
    
    pub fn init(level: Level, format: Format, writer: std.io.AnyWriter) Logger {
        return .{ .level = level, .format = format, .writer = writer };
    }
    
    pub fn log(
        self: *const Logger,
        level: Level,
        comptime fmt: []const u8,
        args: anytype,
    ) void {
        if (@intFromEnum(level) < @intFromEnum(self.level)) return;
        
        const timestamp = std.time.timestamp();
        
        switch (self.format) {
            .text => {
                self.writer.print("[{d}] {s}: ", .{ timestamp, @tagName(level) }) catch return;
                self.writer.print(fmt ++ "\n", args) catch return;
            },
            .json => {
                self.writer.print(
                    "{{\"ts\":{d},\"level\":\"{s}\",\"msg\":\"",
                    .{ timestamp, @tagName(level) },
                ) catch return;
                self.writer.print(fmt, args) catch return;
                self.writer.writeAll("\"}\n") catch return;
            },
        }
    }
    
    pub fn debug(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.debug, fmt, args);
    }
    
    pub fn info(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.info, fmt, args);
    }
    
    pub fn warn(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.warn, fmt, args);
    }
    
    pub fn err(self: *const Logger, comptime fmt: []const u8, args: anytype) void {
        self.log(.err, fmt, args);
    }
};

// Structured log fields
pub const Fields = struct {
    request_id: ?[]const u8 = null,
    user_id: ?u64 = null,
    service: ?[]const u8 = null,
    duration_ms: ?u64 = null,
    status: ?u16 = null,
    
    pub fn logJson(self: Fields, writer: anytype) !void {
        try writer.writeAll("{");
        var first = true;
        
        if (self.request_id) |id| {
            if (!first) try writer.writeAll(",");
            try writer.print("\"request_id\":\"{s}\"", .{id});
            first = false;
        }
        
        if (self.user_id) |uid| {
            if (!first) try writer.writeAll(",");
            try writer.print("\"user_id\":{d}", .{uid});
            first = false;
        }
        
        if (self.duration_ms) |dur| {
            if (!first) try writer.writeAll(",");
            try writer.print("\"duration_ms\":{d}", .{dur});
            first = false;
        }
        
        if (self.status) |s| {
            if (!first) try writer.writeAll(",");
            try writer.print("\"status\":{d}", .{s});
        }
        
        try writer.writeAll("}");
    }
};

pub fn main() !void {
    const stdout = std.io.getStdOut();
    const logger = Logger.init(.info, .json, stdout.writer().any());
    
    logger.info("Server starting on port {d}", .{8080});
    logger.warn("High memory usage: {d}%", .{85});
    logger.err("Database connection failed: {s}", .{"timeout"});
}
```

### Log Rotation

```zig
// log_rotation.zig
const std = @import("std");

pub const RotatingFileLogger = struct {
    dir: std.fs.Dir,
    base_name: []const u8,
    max_size_bytes: u64,
    max_files: u32,
    current_file: ?std.fs.File,
    current_size: u64,
    allocator: std.mem.Allocator,
    
    pub fn init(
        dir_path: []const u8,
        base_name: []const u8,
        max_size_mb: u32,
        max_files: u32,
        allocator: std.mem.Allocator,
    ) !RotatingFileLogger {
        const dir = try std.fs.cwd().openDir(dir_path, .{});
        
        return RotatingFileLogger{
            .dir = dir,
            .base_name = base_name,
            .max_size_bytes = @as(u64, max_size_mb) * 1024 * 1024,
            .max_files = max_files,
            .current_file = null,
            .current_size = 0,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *RotatingFileLogger) void {
        if (self.current_file) |f| f.close();
        self.dir.close();
    }
    
    pub fn write(self: *RotatingFileLogger, data: []const u8) !void {
        // ตรวจสอบว่าต้อง rotate หรือไม่
        if (self.current_size + data.len > self.max_size_bytes) {
            try self.rotate();
        }
        
        // เปิด file ถ้ายังไม่ได้เปิด
        if (self.current_file == null) {
            const filename = try std.fmt.allocPrint(
                self.allocator,
                "{s}.log",
                .{self.base_name},
            );
            defer self.allocator.free(filename);
            
            self.current_file = try self.dir.createFile(filename, .{
                .truncate = false,
            });
            
            // Seek to end
            try self.current_file.?.seekFromEnd(0);
        }
        
        // เขียน data
        try self.current_file.?.writeAll(data);
        self.current_size += data.len;
    }
    
    fn rotate(self: *RotatingFileLogger) !void {
        // ปิด current file
        if (self.current_file) |f| {
            f.close();
            self.current_file = null;
        }
        
        // Rename เก่าๆ
        var i = self.max_files - 1;
        while (i > 0) : (i -= 1) {
            const old_name = try std.fmt.allocPrint(
                self.allocator,
                "{s}.{d}.log",
                .{ self.base_name, i - 1 },
            );
            defer self.allocator.free(old_name);
            
            const new_name = try std.fmt.allocPrint(
                self.allocator,
                "{s}.{d}.log",
                .{ self.base_name, i },
            );
            defer self.allocator.free(new_name);
            
            self.dir.rename(old_name, new_name) catch |err| switch (err) {
                error.FileNotFound => {},
                else => return err,
            };
        }
        
        // Rename current log
        const current_name = try std.fmt.allocPrint(
            self.allocator,
            "{s}.log",
            .{self.base_name},
        );
        defer self.allocator.free(current_name);
        
        const archive_name = try std.fmt.allocPrint(
            self.allocator,
            "{s}.0.log",
            .{self.base_name},
        );
        defer self.allocator.free(archive_name);
        
        self.dir.rename(current_name, archive_name) catch |err| switch (err) {
            error.FileNotFound => {},
            else => return err,
        };
        
        self.current_size = 0;
    }
};
```

---

## Step 994: Health Checks

### การ implement Health Checks

```zig
// health_checks.zig
const std = @import("std");

pub const HealthChecker = struct {
    allocator: std.mem.Allocator,
    checks: std.ArrayList(Check),
    
    pub const Status = enum { healthy, degraded, unhealthy };
    
    pub const CheckResult = struct {
        name: []const u8,
        status: Status,
        message: ?[]const u8 = null,
        duration_ms: u64,
    };
    
    pub const OverallHealth = struct {
        status: Status,
        checks: []const CheckResult,
        timestamp: i64,
    };
    
    pub const Check = struct {
        name: []const u8,
        func: *const fn (allocator: std.mem.Allocator) anyerror!CheckResult,
    };
    
    pub fn init(allocator: std.mem.Allocator) HealthChecker {
        return .{
            .allocator = allocator,
            .checks = std.ArrayList(Check).init(allocator),
        };
    }
    
    pub fn deinit(self: *HealthChecker) void {
        self.checks.deinit();
    }
    
    pub fn register(
        self: *HealthChecker,
        name: []const u8,
        func: *const fn (allocator: std.mem.Allocator) anyerror!CheckResult,
    ) !void {
        try self.checks.append(.{ .name = name, .func = func });
    }
    
    pub fn runAll(self: *HealthChecker) !OverallHealth {
        var results = std.ArrayList(CheckResult).init(self.allocator);
        defer results.deinit();
        
        var overall_status = Status.healthy;
        
        for (self.checks.items) |check| {
            const result = check.func(self.allocator) catch |err| CheckResult{
                .name = check.name,
                .status = .unhealthy,
                .message = @errorName(err),
                .duration_ms = 0,
            };
            
            try results.append(result);
            
            switch (result.status) {
                .healthy => {},
                .degraded => if (overall_status == .healthy) {
                    overall_status = .degraded;
                },
                .unhealthy => overall_status = .unhealthy,
            }
        }
        
        return OverallHealth{
            .status = overall_status,
            .checks = try results.toOwnedSlice(),
            .timestamp = std.time.timestamp(),
        };
    }
    
    pub fn toJson(health: OverallHealth, writer: anytype) !void {
        try writer.print(
            \\{{"status":"{s}","timestamp":{d},"checks":[
        , .{ @tagName(health.status), health.timestamp });
        
        for (health.checks, 0..) |check, i| {
            if (i > 0) try writer.writeAll(",");
            try writer.print(
                \\{{"name":"{s}","status":"{s}"
            , .{ check.name, @tagName(check.status) });
            
            if (check.message) |msg| {
                try writer.print(",\"message\":\"{s}\"", .{msg});
            }
            
            try writer.print(",\"duration_ms\":{d}}}", .{check.duration_ms});
        }
        
        try writer.writeAll("]}");
    }
};

// ตัวอย่าง health check functions
pub fn checkDatabase(allocator: std.mem.Allocator) !HealthChecker.CheckResult {
    const timer = try std.time.Timer.start();
    
    // Mock: ตรวจสอบ database connection
    // ในชีวิตจริงจะ connect และ run simple query
    const connected = true; // mock
    
    const duration_ms = timer.read() / 1_000_000;
    
    return HealthChecker.CheckResult{
        .name = "database",
        .status = if (connected) .healthy else .unhealthy,
        .message = if (!connected) "Cannot connect to database" else null,
        .duration_ms = duration_ms,
        .allocator = allocator,
    };
}

const CheckResult2 = struct {
    name: []const u8,
    status: HealthChecker.Status,
    message: ?[]const u8 = null,
    duration_ms: u64,
    allocator: std.mem.Allocator,
};

pub fn checkMemory(allocator: std.mem.Allocator) !HealthChecker.CheckResult {
    const timer = try std.time.Timer.start();
    
    // ตรวจสอบ memory usage
    // Mock: ถ้าใช้เกิน 90% ถือว่า degraded
    const memory_usage_percent: u32 = 75; // mock value
    
    const duration_ms = timer.read() / 1_000_000;
    
    const status: HealthChecker.Status = if (memory_usage_percent > 90)
        .unhealthy
    else if (memory_usage_percent > 75)
        .degraded
    else
        .healthy;
    
    return HealthChecker.CheckResult{
        .name = "memory",
        .status = status,
        .duration_ms = duration_ms,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var checker = HealthChecker.init(allocator);
    defer checker.deinit();
    
    try checker.register("database", checkDatabase);
    try checker.register("memory", checkMemory);
    
    const health = try checker.runAll();
    defer allocator.free(health.checks);
    
    const stdout = std.io.getStdOut().writer();
    try HealthChecker.toJson(health, stdout);
    try stdout.writeAll("\n");
}
```

---

## Step 995: Graceful Shutdown

### การ implement Graceful Shutdown

```zig
// graceful_shutdown.zig
const std = @import("std");

pub const ShutdownHandler = struct {
    signal_received: std.atomic.Value(bool) = std.atomic.Value(bool).init(false),
    callbacks: std.ArrayList(*const fn () void),
    allocator: std.mem.Allocator,
    timeout_ms: u64,
    
    pub fn init(allocator: std.mem.Allocator, timeout_ms: u64) ShutdownHandler {
        return .{
            .callbacks = std.ArrayList(*const fn () void).init(allocator),
            .allocator = allocator,
            .timeout_ms = timeout_ms,
        };
    }
    
    pub fn deinit(self: *ShutdownHandler) void {
        self.callbacks.deinit();
    }
    
    pub fn register(self: *ShutdownHandler, callback: *const fn () void) !void {
        try self.callbacks.append(callback);
    }
    
    pub fn isShuttingDown(self: *const ShutdownHandler) bool {
        return self.signal_received.load(.acquire);
    }
    
    pub fn initiateShutdown(self: *ShutdownHandler) void {
        self.signal_received.store(true, .release);
        std.debug.print("Initiating graceful shutdown...\n", .{});
    }
    
    pub fn wait(self: *ShutdownHandler) void {
        const start = std.time.milliTimestamp();
        
        // รัน shutdown callbacks (in reverse order)
        var i: usize = self.callbacks.items.len;
        while (i > 0) {
            i -= 1;
            const elapsed = std.time.milliTimestamp() - start;
            if (@as(u64, @intCast(elapsed)) >= self.timeout_ms) {
                std.debug.print("Shutdown timeout! Forcing exit.\n", .{});
                break;
            }
            self.callbacks.items[i]();
        }
        
        const elapsed = std.time.milliTimestamp() - start;
        std.debug.print("Shutdown complete in {d}ms\n", .{elapsed});
    }
};

// Signal handling
pub fn setupSignalHandlers(handler: *ShutdownHandler) !void {
    // ใน production ต้อง handle SIGTERM และ SIGINT
    // ตัวอย่างนี้ simplified
    _ = handler;
    
    if (@import("builtin").os.tag == .linux or @import("builtin").os.tag == .macos) {
        // ใช้ posix signal handlers
        const action = std.posix.Sigaction{
            .handler = .{ .handler = std.posix.SIG.DFL },
            .mask = std.posix.sigemptyset(),
            .flags = 0,
        };
        _ = action;
        // std.posix.sigaction(std.posix.SIG.TERM, &action, null) catch {};
        // std.posix.sigaction(std.posix.SIG.INT, &action, null) catch {};
    }
}

// ตัวอย่าง server ที่ใช้ graceful shutdown
pub const Server = struct {
    handler: *ShutdownHandler,
    active_connections: std.atomic.Value(u32) = std.atomic.Value(u32).init(0),
    
    pub fn start(self: *Server) !void {
        std.debug.print("Server started\n", .{});
        
        // Main loop
        while (!self.handler.isShuttingDown()) {
            // รับ connections ใหม่
            self.handleConnection();
            std.time.sleep(10 * std.time.ns_per_ms);
        }
        
        std.debug.print("No longer accepting new connections\n", .{});
        
        // รอให้ active connections จบ
        while (self.active_connections.load(.acquire) > 0) {
            std.debug.print(
                "Waiting for {d} active connections...\n",
                .{self.active_connections.load(.acquire)},
            );
            std.time.sleep(100 * std.time.ns_per_ms);
        }
        
        std.debug.print("All connections closed\n", .{});
    }
    
    fn handleConnection(self: *Server) void {
        _ = self.active_connections.fetchAdd(1, .acq_rel);
        defer _ = self.active_connections.fetchSub(1, .acq_rel);
        
        // Handle connection...
        std.time.sleep(1 * std.time.ns_per_ms);
    }
};

fn dbCleanup() void {
    std.debug.print("Closing database connections...\n", .{});
}

fn cacheFlush() void {
    std.debug.print("Flushing cache...\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var handler = ShutdownHandler.init(allocator, 30_000); // 30s timeout
    defer handler.deinit();
    
    // ลงทะเบียน cleanup callbacks
    try handler.register(dbCleanup);
    try handler.register(cacheFlush);
    
    var server = Server{ .handler = &handler };
    
    // Simulate receiving shutdown signal after 2 seconds
    const t = try std.Thread.spawn(.{}, struct {
        fn run(h: *ShutdownHandler) void {
            std.time.sleep(2 * std.time.ns_per_s);
            h.initiateShutdown();
        }
    }.run, .{&handler});
    
    try server.start();
    t.join();
    handler.wait();
}
```

---

## Step 996: Resource Limits

### การจัดการ Resource Limits

```zig
// resource_limits.zig
const std = @import("std");

pub const ResourceLimiter = struct {
    max_memory_bytes: u64,
    max_open_files: u32,
    max_goroutines: u32, // threads ใน Zig
    
    current_memory: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    current_files: std.atomic.Value(u32) = std.atomic.Value(u32).init(0),
    current_threads: std.atomic.Value(u32) = std.atomic.Value(u32).init(0),
    
    pub fn checkMemory(self: *ResourceLimiter, size: u64) bool {
        const current = self.current_memory.load(.monotonic);
        return current + size <= self.max_memory_bytes;
    }
    
    pub fn allocate(self: *ResourceLimiter, size: u64) bool {
        var current = self.current_memory.load(.monotonic);
        while (current + size <= self.max_memory_bytes) {
            const new = current + size;
            if (self.current_memory.cmpxchgWeak(
                current, new, .acq_rel, .monotonic,
            )) |actual| {
                current = actual;
            } else {
                return true; // success
            }
        }
        return false; // exceeded limit
    }
    
    pub fn free(self: *ResourceLimiter, size: u64) void {
        _ = self.current_memory.fetchSub(size, .acq_rel);
    }
    
    pub fn stats(self: *const ResourceLimiter) void {
        std.debug.print("Resource Usage:\n", .{});
        std.debug.print("  Memory: {d}/{d} MB ({d:.1}%)\n", .{
            self.current_memory.load(.monotonic) / (1024 * 1024),
            self.max_memory_bytes / (1024 * 1024),
            @as(f64, @floatFromInt(self.current_memory.load(.monotonic))) /
                @as(f64, @floatFromInt(self.max_memory_bytes)) * 100.0,
        });
        std.debug.print("  Files: {d}/{d}\n", .{
            self.current_files.load(.monotonic),
            self.max_open_files,
        });
        std.debug.print("  Threads: {d}/{d}\n", .{
            self.current_threads.load(.monotonic),
            self.max_goroutines,
        });
    }
};

// OS-level resource limits (Linux)
pub fn setProcessLimits() !void {
    // ใน production ควร set resource limits ด้วย systemd หรือ Docker
    // แต่สามารถ set ใน code ได้เช่นกัน
    
    if (@import("builtin").os.tag == .linux) {
        // setrlimit(RLIMIT_NOFILE, ...) - max open files
        // setrlimit(RLIMIT_AS, ...) - max virtual memory
        // setrlimit(RLIMIT_NPROC, ...) - max processes/threads
    }
}

// Rate limiting
pub const RateLimiter = struct {
    max_requests: u32,
    window_ms: u64,
    requests: std.ArrayList(i64),
    mutex: std.Thread.Mutex = .{},
    allocator: std.mem.Allocator,
    
    pub fn init(max_requests: u32, window_ms: u64, allocator: std.mem.Allocator) RateLimiter {
        return .{
            .max_requests = max_requests,
            .window_ms = window_ms,
            .requests = std.ArrayList(i64).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *RateLimiter) void {
        self.requests.deinit();
    }
    
    pub fn allow(self: *RateLimiter) !bool {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        const now = std.time.milliTimestamp();
        const window_start = now - @as(i64, @intCast(self.window_ms));
        
        // ลบ requests เก่า
        var i: usize = 0;
        while (i < self.requests.items.len) {
            if (self.requests.items[i] < window_start) {
                _ = self.requests.swapRemove(i);
            } else {
                i += 1;
            }
        }
        
        if (self.requests.items.len >= self.max_requests) {
            return false; // Rate limit exceeded
        }
        
        try self.requests.append(now);
        return true;
    }
};
```

---

## Step 997: Monitoring และ Alerting

### การ implement Monitoring

```zig
// monitoring.zig
const std = @import("std");

pub const Metrics = struct {
    allocator: std.mem.Allocator,
    counters: std.StringHashMap(std.atomic.Value(u64)),
    gauges: std.StringHashMap(std.atomic.Value(i64)),
    histograms: std.StringHashMap(Histogram),
    mutex: std.Thread.Mutex = .{},
    
    pub const Histogram = struct {
        buckets: [10]u64 = [_]u64{0} ** 10,
        bounds: [9]f64 = .{ 1, 5, 10, 25, 50, 100, 250, 500, 1000 },
        count: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
        sum: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
        
        pub fn observe(self: *Histogram, value: f64) void {
            _ = self.count.fetchAdd(1, .acq_rel);
            _ = self.sum.fetchAdd(@as(u64, @intFromFloat(value)), .acq_rel);
            
            for (self.bounds, 0..) |bound, i| {
                if (value <= bound) {
                    _ = @atomicRmw(u64, &self.buckets[i], .Add, 1, .acq_rel);
                    return;
                }
            }
            // "+Inf" bucket
            _ = @atomicRmw(u64, &self.buckets[9], .Add, 1, .acq_rel);
        }
    };
    
    pub fn init(allocator: std.mem.Allocator) Metrics {
        return .{
            .allocator = allocator,
            .counters = std.StringHashMap(std.atomic.Value(u64)).init(allocator),
            .gauges = std.StringHashMap(std.atomic.Value(i64)).init(allocator),
            .histograms = std.StringHashMap(Histogram).init(allocator),
        };
    }
    
    pub fn deinit(self: *Metrics) void {
        self.counters.deinit();
        self.gauges.deinit();
        self.histograms.deinit();
    }
    
    pub fn increment(self: *Metrics, name: []const u8) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        if (self.counters.getPtr(name)) |counter| {
            _ = counter.fetchAdd(1, .acq_rel);
        } else {
            self.counters.put(name, std.atomic.Value(u64).init(1)) catch {};
        }
    }
    
    pub fn set(self: *Metrics, name: []const u8, value: i64) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        if (self.gauges.getPtr(name)) |gauge| {
            gauge.store(value, .release);
        } else {
            self.gauges.put(name, std.atomic.Value(i64).init(value)) catch {};
        }
    }
    
    pub fn observe(self: *Metrics, name: []const u8, value: f64) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        if (self.histograms.getPtr(name)) |hist| {
            hist.observe(value);
        } else {
            var hist = Histogram{};
            hist.observe(value);
            self.histograms.put(name, hist) catch {};
        }
    }
    
    /// Export ใน Prometheus format
    pub fn exportPrometheus(self: *const Metrics, writer: anytype) !void {
        var it = self.counters.iterator();
        while (it.next()) |entry| {
            try writer.print("# TYPE {s} counter\n", .{entry.key_ptr.*});
            try writer.print("{s} {d}\n", .{
                entry.key_ptr.*,
                entry.value_ptr.load(.acquire),
            });
        }
        
        var git = self.gauges.iterator();
        while (git.next()) |entry| {
            try writer.print("# TYPE {s} gauge\n", .{entry.key_ptr.*});
            try writer.print("{s} {d}\n", .{
                entry.key_ptr.*,
                entry.value_ptr.load(.acquire),
            });
        }
    }
};

// Application metrics
var global_metrics: ?*Metrics = null;

pub fn getMetrics() *Metrics {
    return global_metrics orelse unreachable;
}

pub fn recordRequest(method: []const u8, path: []const u8, status: u16, duration_ms: f64) void {
    const metrics = getMetrics();
    metrics.increment("http_requests_total");
    metrics.observe("http_request_duration_ms", duration_ms);
    
    if (status >= 500) {
        metrics.increment("http_errors_total");
    }
    
    _ = method;
    _ = path;
}
```

---

## Step 998: Incident Response

### การ Handle Incidents ใน Production

```zig
// incident_response.zig
const std = @import("std");

pub const IncidentLevel = enum {
    /// ไม่กระทบ users
    p4_low,
    /// กระทบ users บางส่วน
    p3_moderate,
    /// กระทบ users หลัก
    p2_high,
    /// Service down ทั้งหมด
    p1_critical,
    
    pub fn label(self: IncidentLevel) []const u8 {
        return switch (self) {
            .p4_low => "P4 - Low",
            .p3_moderate => "P3 - Moderate",
            .p2_high => "P2 - High",
            .p1_critical => "P1 - Critical",
        };
    }
    
    pub fn responseTime(self: IncidentLevel) []const u8 {
        return switch (self) {
            .p4_low => "48 hours",
            .p3_moderate => "4 hours",
            .p2_high => "1 hour",
            .p1_critical => "15 minutes",
        };
    }
};

// Runbook template
pub const Runbook = struct {
    service: []const u8,
    symptom: []const u8,
    checks: []const Step,
    remediation: []const Step,
    
    pub const Step = struct {
        description: []const u8,
        command: ?[]const u8 = null,
    };
    
    pub fn print(self: Runbook, writer: anytype) !void {
        try writer.print("# Runbook: {s}\n\n", .{self.service});
        try writer.print("## Symptom\n{s}\n\n", .{self.symptom});
        
        try writer.print("## Diagnostic Steps\n", .{});
        for (self.checks, 1..) |step, i| {
            try writer.print("{d}. {s}\n", .{ i, step.description });
            if (step.command) |cmd| {
                try writer.print("   ```\n   {s}\n   ```\n", .{cmd});
            }
        }
        
        try writer.print("\n## Remediation\n", .{});
        for (self.remediation, 1..) |step, i| {
            try writer.print("{d}. {s}\n", .{ i, step.description });
            if (step.command) |cmd| {
                try writer.print("   ```\n   {s}\n   ```\n", .{cmd});
            }
        }
    }
};

// Circuit breaker pattern - ป้องกัน cascading failures
pub fn CircuitBreaker(T: type) type {
    return struct {
        state: State = .closed,
        failure_count: u32 = 0,
        success_count: u32 = 0,
        last_failure_time: i64 = 0,
        
        // Configuration
        failure_threshold: u32 = 5,
        success_threshold: u32 = 3,
        timeout_ms: u64 = 60_000, // 1 minute
        
        const Self = @This();
        
        pub const State = enum {
            closed,   // ปกติ
            open,     // หยุดส่ง requests
            half_open, // ทดสอบว่า recover แล้วหรือยัง
        };
        
        pub const Error = error{CircuitOpen};
        
        pub fn call(self: *Self, func: fn () anyerror!T) anyerror!T {
            switch (self.state) {
                .open => {
                    // ตรวจสอบว่าหมด timeout หรือยัง
                    const now = std.time.milliTimestamp();
                    if (@as(u64, @intCast(now - self.last_failure_time)) >= self.timeout_ms) {
                        self.state = .half_open;
                    } else {
                        return error.CircuitOpen;
                    }
                },
                .closed, .half_open => {},
            }
            
            const result = func() catch |err| {
                self.recordFailure();
                return err;
            };
            
            self.recordSuccess();
            return result;
        }
        
        fn recordFailure(self: *Self) void {
            self.failure_count += 1;
            self.last_failure_time = std.time.milliTimestamp();
            
            if (self.failure_count >= self.failure_threshold) {
                self.state = .open;
                std.debug.print("Circuit OPEN - too many failures\n", .{});
            }
        }
        
        fn recordSuccess(self: *Self) void {
            if (self.state == .half_open) {
                self.success_count += 1;
                if (self.success_count >= self.success_threshold) {
                    self.state = .closed;
                    self.failure_count = 0;
                    self.success_count = 0;
                    std.debug.print("Circuit CLOSED - service recovered\n", .{});
                }
            }
        }
    };
}
```

---

## Step 999: Performance Tuning ใน Production

### การ tune Performance

```zig
// production_perf.zig
const std = @import("std");

// Memory allocator สำหรับ production
pub fn createProductionAllocator() std.mem.Allocator {
    // สำหรับ server applications ที่มี predictable allocation patterns
    // Fixed Buffer Allocator เร็วมาก แต่ต้องรู้ขนาดล่วงหน้า
    
    // สำหรับ general purpose:
    var gpa = std.heap.GeneralPurposeAllocator(.{
        .safety = false, // ปิด safety checks ใน production
    }){};
    return gpa.allocator();
}

// Thread pool สำหรับ concurrent workloads
pub const ThreadPool = struct {
    threads: []std.Thread,
    queue: WorkQueue,
    running: std.atomic.Value(bool) = std.atomic.Value(bool).init(true),
    allocator: std.mem.Allocator,
    
    const WorkQueue = struct {
        items: std.ArrayList(WorkItem),
        mutex: std.Thread.Mutex = .{},
        cond: std.Thread.Condition = .{},
        
        const WorkItem = struct {
            func: *const fn (context: *anyopaque) void,
            context: *anyopaque,
        };
        
        pub fn push(self: *WorkQueue, item: WorkItem) !void {
            self.mutex.lock();
            defer self.mutex.unlock();
            try self.items.append(item);
            self.cond.signal();
        }
        
        pub fn pop(self: *WorkQueue) ?WorkItem {
            self.mutex.lock();
            defer self.mutex.unlock();
            
            if (self.items.items.len == 0) return null;
            return self.items.orderedRemove(0);
        }
    };
    
    pub fn init(thread_count: u32, allocator: std.mem.Allocator) !ThreadPool {
        var pool = ThreadPool{
            .threads = try allocator.alloc(std.Thread, thread_count),
            .queue = .{ .items = std.ArrayList(WorkQueue.WorkItem).init(allocator) },
            .allocator = allocator,
        };
        
        for (pool.threads) |*thread| {
            thread.* = try std.Thread.spawn(.{}, workerLoop, .{&pool});
        }
        
        return pool;
    }
    
    pub fn deinit(self: *ThreadPool) void {
        self.running.store(false, .release);
        self.queue.cond.broadcast();
        
        for (self.threads) |thread| {
            thread.join();
        }
        
        self.allocator.free(self.threads);
        self.queue.items.deinit();
    }
    
    fn workerLoop(pool: *ThreadPool) void {
        while (pool.running.load(.acquire)) {
            if (pool.queue.pop()) |item| {
                item.func(item.context);
            } else {
                pool.queue.mutex.lock();
                pool.queue.cond.wait(&pool.queue.mutex);
                pool.queue.mutex.unlock();
            }
        }
    }
};

// Memory pooling สำหรับ frequent allocations
pub fn Pool(T: type) type {
    return struct {
        free_list: std.ArrayList(*T),
        allocator: std.mem.Allocator,
        mutex: std.Thread.Mutex = .{},
        
        const Self = @This();
        
        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .free_list = std.ArrayList(*T).init(allocator),
                .allocator = allocator,
            };
        }
        
        pub fn deinit(self: *Self) void {
            for (self.free_list.items) |item| {
                self.allocator.destroy(item);
            }
            self.free_list.deinit();
        }
        
        pub fn acquire(self: *Self) !*T {
            self.mutex.lock();
            defer self.mutex.unlock();
            
            if (self.free_list.items.len > 0) {
                return self.free_list.pop();
            }
            
            return try self.allocator.create(T);
        }
        
        pub fn release(self: *Self, item: *T) void {
            self.mutex.lock();
            defer self.mutex.unlock();
            
            self.free_list.append(item) catch {
                self.allocator.destroy(item);
            };
        }
    };
}

// Performance tips สำหรับ production
pub fn productionTips() void {
    std.debug.print(
        \\Performance Tips สำหรับ Production:
        \\
        \\1. Allocator Selection:
        \\   - FixedBufferAllocator สำหรับ fixed-size allocations
        \\   - Arena allocator สำหรับ request-scoped data
        \\   - GPA ปิด safety checks ใน ReleaseFast
        \\
        \\2. SIMD Operations:
        \\   - ใช้ @Vector สำหรับ parallel operations
        \\   - Zig auto-vectorizes บาง loops
        \\
        \\3. Cache Optimization:
        \\   - เรียงข้อมูล struct ตาม access pattern (SoA vs AoS)
        \\   - ใช้ prefetch hints สำหรับ predictable access
        \\
        \\4. Concurrency:
        \\   - Thread pool แทน spawning threads ใหม่ทุกครั้ง
        \\   - Lock-free data structures เมื่อทำได้
        \\   - Reduce contention บน shared resources
        \\
        \\5. I/O:
        \\   - Async I/O (io_uring บน Linux)
        \\   - Batch operations เมื่อทำได้
        \\   - Connection pooling
        \\
    , .{});
}
```

---

## Step 1000: หลักสูตรสรุปและก้าวต่อไป

### สรุปหลักสูตร Zig Programming Course ทั้งหมด

```zig
//! # หลักสูตร Zig Programming ครบ 1000 Steps
//! 
//! ยินดีด้วย! คุณได้เรียนจบหลักสูตร Zig Programming อย่างสมบูรณ์แล้ว
//! มาดูสิ่งที่คุณได้เรียนรู้ตลอดหลักสูตรนี้กัน

const std = @import("std");

pub fn main() void {
    std.debug.print("\n", .{});
    std.debug.print("=" ** 60 ++ "\n", .{});
    std.debug.print("ยินดีด้วย! คุณเรียนจบหลักสูตร Zig Programming แล้ว!\n", .{});
    std.debug.print("=" ** 60 ++ "\n\n", .{});
    
    printCourseSummary();
    printCareerPaths();
    printNextSteps();
}

fn printCourseSummary() void {
    std.debug.print("สิ่งที่คุณได้เรียนรู้:\n\n", .{});
    
    const sections = [_]struct {
        parts: []const u8,
        topic: []const u8,
        key_concepts: []const u8,
    }{
        .{
            .parts = "Part 1-10",
            .topic = "พื้นฐาน Zig",
            .key_concepts = "syntax, types, operators, control flow",
        },
        .{
            .parts = "Part 11-20",
            .topic = "Functions และ Scope",
            .key_concepts = "functions, comptime, inline, var vs const",
        },
        .{
            .parts = "Part 21-30",
            .topic = "Memory Management",
            .key_concepts = "allocators, slices, pointers, ownership",
        },
        .{
            .parts = "Part 31-40",
            .topic = "Error Handling",
            .key_concepts = "error unions, try, catch, errdefer",
        },
        .{
            .parts = "Part 41-50",
            .topic = "Structs และ Enums",
            .key_concepts = "structs, enums, unions, methods",
        },
        .{
            .parts = "Part 51-60",
            .topic = "Generics และ Comptime",
            .key_concepts = "comptime, generics, type inference",
        },
        .{
            .parts = "Part 61-70",
            .topic = "Concurrency",
            .key_concepts = "threads, async/await, atomics",
        },
        .{
            .parts = "Part 71-80",
            .topic = "Systems Programming",
            .key_concepts = "C interop, OS APIs, system calls",
        },
        .{
            .parts = "Part 81-90",
            .topic = "Testing และ Quality",
            .key_concepts = "unit tests, integration tests, fuzzing",
        },
        .{
            .parts = "Part 91-100",
            .topic = "Production Ready",
            .key_concepts = "benchmarking, docs, packages, deployment",
        },
    };
    
    for (sections) |s| {
        std.debug.print("  {s}: {s}\n", .{ s.parts, s.topic });
        std.debug.print("    Concepts: {s}\n\n", .{s.key_concepts});
    }
}

fn printCareerPaths() void {
    std.debug.print("Career Paths สำหรับ Zig Developers:\n\n", .{});
    
    std.debug.print("1. Systems Programmer\n", .{});
    std.debug.print("   - ทำงาน OS, drivers, embedded systems\n", .{});
    std.debug.print("   - เงินเดือน: สูงมาก\n", .{});
    std.debug.print("   - ทักษะเพิ่มเติม: hardware, assembly, OS internals\n\n", .{});
    
    std.debug.print("2. Game Developer\n", .{});
    std.debug.print("   - ใช้ mach engine หรือ custom engine\n", .{});
    std.debug.print("   - เงินเดือน: สูง\n", .{});
    std.debug.print("   - ทักษะเพิ่มเติม: graphics, physics, game design\n\n", .{});
    
    std.debug.print("3. WebAssembly Developer\n", .{});
    std.debug.print("   - Compile Zig ไปเป็น WASM\n", .{});
    std.debug.print("   - เงินเดือน: สูง\n", .{});
    std.debug.print("   - ทักษะเพิ่มเติม: web tech, JavaScript\n\n", .{});
    
    std.debug.print("4. Network/Systems Software Engineer\n", .{});
    std.debug.print("   - HTTP servers, databases, protocols\n", .{});
    std.debug.print("   - เงินเดือน: สูงมาก\n", .{});
    std.debug.print("   - ทักษะเพิ่มเติม: networking, distributed systems\n\n", .{});
    
    std.debug.print("5. Open Source Maintainer\n", .{});
    std.debug.print("   - สร้างและ maintain libraries\n", .{});
    std.debug.print("   - รายได้จาก sponsorship/consulting\n\n", .{});
    
    std.debug.print("6. Zig Compiler/Toolchain Developer\n", .{});
    std.debug.print("   - ทำงานบน Zig compiler เอง\n", .{});
    std.debug.print("   - ต้องการความรู้ compiler theory\n\n", .{});
}

fn printNextSteps() void {
    std.debug.print("ก้าวต่อไปในฐานะ Zig Professional:\n\n", .{});
    
    std.debug.print("สัปดาห์ที่ 1-4: สร้าง Real Project\n", .{});
    std.debug.print("  - เลือก project ที่คุณสนใจจริงๆ\n", .{});
    std.debug.print("  - CLI tool, web server, game, หรือ library\n", .{});
    std.debug.print("  - Deploy ขึ้น GitHub\n\n", .{});
    
    std.debug.print("เดือนที่ 2-3: Contribute to Open Source\n", .{});
    std.debug.print("  - Fix bug ใน Zig standard library\n", .{});
    std.debug.print("  - เพิ่ม feature ใน library ที่ใช้\n", .{});
    std.debug.print("  - เขียน blog post เกี่ยวกับ Zig\n\n", .{});
    
    std.debug.print("เดือนที่ 4-6: Build Your Portfolio\n", .{});
    std.debug.print("  - GitHub profile ที่มี Zig projects\n", .{});
    std.debug.print("  - Technical blog หรือ YouTube channel\n", .{});
    std.debug.print("  - Speak ที่ local meetup\n\n", .{});
    
    std.debug.print("Resources สำหรับ Learning ต่อ:\n\n", .{});
    std.debug.print("  Documentation:\n", .{});
    std.debug.print("  - ziglang.org/documentation/master/\n", .{});
    std.debug.print("  - ziglang.org/learn/\n\n", .{});
    
    std.debug.print("  Community:\n", .{});
    std.debug.print("  - ziglang.org/join-discord\n", .{});
    std.debug.print("  - ziggit.dev (forum)\n", .{});
    std.debug.print("  - github.com/ziglang/zig/discussions\n\n", .{});
    
    std.debug.print("  Projects to Study:\n", .{});
    std.debug.print("  - Bun (JavaScript runtime ที่ใช้ Zig)\n", .{});
    std.debug.print("  - TigerBeetle (database ที่เขียนด้วย Zig)\n", .{});
    std.debug.print("  - Mach Engine (game engine)\n\n", .{});
    
    std.debug.print("  Books and Courses:\n", .{});
    std.debug.print("  - ziglings (exercises)\n", .{});
    std.debug.print("  - Learning Zig (book)\n\n", .{});
}
```

### Complete Production Application Example

```zig
// complete_production_app.zig
//! ตัวอย่าง Production Application ที่ใช้ best practices ทั้งหมด

const std = @import("std");

// ============ Configuration ============

pub const Config = struct {
    host: []const u8 = "0.0.0.0",
    port: u16 = 8080,
    log_level: LogLevel = .info,
    shutdown_timeout_ms: u64 = 30_000,
    
    pub const LogLevel = enum { debug, info, warn, err };
    
    pub fn fromEnv() Config {
        var config = Config{};
        
        if (std.posix.getenv("PORT")) |p| {
            config.port = std.fmt.parseInt(u16, p, 10) catch config.port;
        }
        
        if (std.posix.getenv("HOST")) |h| {
            config.host = h;
        }
        
        return config;
    }
};

// ============ Application State ============

pub const AppState = struct {
    allocator: std.mem.Allocator,
    config: Config,
    running: std.atomic.Value(bool) = std.atomic.Value(bool).init(true),
    request_count: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    start_time: i64,
    
    pub fn init(allocator: std.mem.Allocator, config: Config) AppState {
        return .{
            .allocator = allocator,
            .config = config,
            .start_time = std.time.timestamp(),
        };
    }
    
    pub fn uptime(self: *const AppState) i64 {
        return std.time.timestamp() - self.start_time;
    }
    
    pub fn incrementRequests(self: *AppState) u64 {
        return self.request_count.fetchAdd(1, .acq_rel) + 1;
    }
};

// ============ Request Handler ============

pub const Request = struct {
    method: []const u8,
    path: []const u8,
    id: []const u8,
};

pub const Response = struct {
    status: u16,
    body: []const u8,
    content_type: []const u8 = "application/json",
};

pub fn handleRequest(state: *AppState, req: Request) Response {
    const request_num = state.incrementRequests();
    
    std.debug.print("[{s}] {s} {s} (request #{d})\n", .{
        req.id, req.method, req.path, request_num,
    });
    
    // Health check endpoint
    if (std.mem.eql(u8, req.path, "/health")) {
        return .{
            .status = 200,
            .body =
            \\{"status":"healthy","uptime_seconds":120}
            ,
        };
    }
    
    // Metrics endpoint
    if (std.mem.eql(u8, req.path, "/metrics")) {
        return .{
            .status = 200,
            .body = "# HELP requests_total\nrequests_total 1\n",
            .content_type = "text/plain",
        };
    }
    
    // Default response
    return .{
        .status = 404,
        .body = "{\"error\":\"not found\"}",
    };
}

// ============ Main ============

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Load configuration
    const config = Config.fromEnv();
    std.debug.print("Starting server on {s}:{d}\n", .{ config.host, config.port });
    
    // Create application state
    var state = AppState.init(allocator, config);
    
    // Setup graceful shutdown
    const shutdown_thread = try std.Thread.spawn(.{}, struct {
        fn run(s: *AppState) void {
            std.time.sleep(5 * std.time.ns_per_s); // Simulate SIGTERM after 5s
            std.debug.print("Received shutdown signal\n", .{});
            s.running.store(false, .release);
        }
    }.run, .{&state});
    
    // Simulate request handling
    var i: u32 = 0;
    while (state.running.load(.acquire) and i < 10) : (i += 1) {
        const req = Request{
            .method = "GET",
            .path = if (i % 3 == 0) "/health" else "/api/data",
            .id = "req-0001",
        };
        
        const resp = handleRequest(&state, req);
        std.debug.print("Response: {d}\n", .{resp.status});
        
        std.time.sleep(500 * std.time.ns_per_ms);
    }
    
    shutdown_thread.join();
    
    std.debug.print("\nServer stopped. Served {d} requests.\n", .{
        state.request_count.load(.acquire),
    });
    std.debug.print("Uptime: {d} seconds\n", .{state.uptime()});
}
```

### คำแนะนำสุดท้าย

```zig
// final_advice.zig
//! คำแนะนำสุดท้ายจากหลักสูตร

const std = @import("std");

pub fn main() void {
    std.debug.print(
        \\
        \\  ╔══════════════════════════════════════════════════════╗
        \\  ║          คำแนะนำสุดท้ายสำหรับ Zig Developer         ║
        \\  ╚══════════════════════════════════════════════════════╝
        \\
        \\  1. KISS (Keep It Simple, Stupid)
        \\     Zig เน้น explicitness และ simplicity
        \\     อย่าเพิ่ม abstraction ที่ไม่จำเป็น
        \\
        \\  2. Read Error Messages Carefully  
        \\     Zig มี error messages ที่ดีมาก
        \\     อ่านให้ละเอียดก่อน search ใน Google
        \\
        \\  3. Test Driven Development
        \\     เขียน test ก่อน implementation
        \\     ใช้ `zig test` เป็นประจำ
        \\
        \\  4. Profile Before Optimizing
        \\     อย่า optimize code ที่ไม่ใช่ bottleneck
        \\     วัดก่อน แล้วค่อย optimize
        \\
        \\  5. Read Source Code
        \\     Zig standard library เขียนด้วย Zig
        \\     อ่าน source code เป็นการเรียนรู้ที่ดีที่สุด
        \\
        \\  6. Community is Key
        \\     ถามคำถามใน Discord เมื่อติด
        \\     ช่วยเหลือคนอื่นเมื่อทำได้
        \\
        \\  7. Never Stop Learning
        \\     Zig ยังอยู่ในช่วงพัฒนา
        \\     ติดตาม changelogs และ blog posts อย่างสม่ำเสมอ
        \\
        \\  ขอบคุณที่เรียนหลักสูตรนี้จนจบ!
        \\  ขอให้โชคดีในการเดินทางสู่การเป็น Zig Professional!
        \\
    , .{});
}
```

---

## สรุปครบหลักสูตร: 100 Parts, 1000 Steps

### สิ่งที่คุณได้เรียนรู้

**Foundation (Parts 1-20)**
- Zig syntax, types, operators ทั้งหมด
- Functions, closures, comptime programming
- Control flow, loops, error handling พื้นฐาน

**Memory & Safety (Parts 21-40)**
- Memory model ของ Zig
- Allocators ทุกประเภท
- Pointers, slices, ownership
- Error handling ที่ปลอดภัย

**Data Structures & Algorithms (Parts 41-60)**
- Structs, enums, unions
- Generic programming ด้วย comptime
- Standard library data structures
- Common algorithms

**Systems Programming (Parts 61-80)**
- Concurrency: threads, async, atomics
- C interop: linking, extern functions
- OS APIs: file system, networking, processes
- Low-level: SIMD, inline assembly

**Professional Development (Parts 81-100)**
- Testing: unit, integration, fuzz testing
- Performance: profiling, benchmarking, optimization
- Quality: code review, refactoring
- Production: deployment, monitoring, incident response
- Ecosystem: packages, documentation, open source

### ขอบคุณ

หลักสูตรนี้ออกแบบมาเพื่อพาคุณจากผู้เริ่มต้นสู่ Zig Professional อย่างสมบูรณ์ ทุก Step มีโค้ดตัวอย่างที่ทำงานได้จริงและอธิบายแนวคิดเป็นภาษาไทยที่เข้าใจง่าย

**ก้าวต่อไปของคุณ:**
1. สร้าง project จริงที่ใช้ความรู้ทั้งหมด
2. Contribute ให้กับ Zig ecosystem
3. แชร์ความรู้กับ community
4. ไม่หยุดเรียนรู้

**Zig Community พร้อมต้อนรับคุณเสมอ!**

---

*"If you cannot explain it simply, you do not understand it well enough."*  
*— อัลเบิร์ต ไอน์สไตน์*

*ใน Zig: ถ้าโค้ดของคุณซับซ้อนเกิน แสดงว่ายังมีวิธีที่ดีกว่า*
