# ส่วนที่ 62: โปรเจค HTTP Server สมบูรณ์

## ขั้นตอนที่ 611-620: สร้าง HTTP Server ด้วย Zig

---

## ขั้นตอนที่ 611: Project Requirements

### ภาพรวม HTTP Server

เราจะสร้าง HTTP/1.1 server ที่ทำงานได้จริง รองรับ:
- Static file serving
- JSON API endpoints
- Middleware chain
- Connection keep-alive
- Error handling ที่ถูกต้อง

### โครงสร้างโปรเจค

```
http-server/
├── build.zig
├── src/
│   ├── main.zig
│   ├── server.zig
│   ├── request.zig
│   ├── response.zig
│   ├── router.zig
│   ├── middleware.zig
│   ├── static.zig
│   └── json_api.zig
└── public/
    ├── index.html
    └── style.css
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "http-server",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run the server");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## ขั้นตอนที่ 612: TCP Listener Setup

### src/server.zig

```zig
const std = @import("std");
const request = @import("request.zig");
const response = @import("response.zig");
const router = @import("router.zig");

pub const ServerConfig = struct {
    host: []const u8 = "127.0.0.1",
    port: u16 = 8080,
    max_connections: u32 = 100,
    read_timeout_ms: u32 = 5000,
    write_timeout_ms: u32 = 5000,
    static_dir: ?[]const u8 = null,
    keep_alive: bool = true,
    max_request_size: usize = 1024 * 1024, // 1MB
};

pub const Server = struct {
    allocator: std.mem.Allocator,
    config: ServerConfig,
    routes: router.Router,
    running: bool,
    listener: std.net.Server,

    pub fn init(allocator: std.mem.Allocator, config: ServerConfig) !Server {
        const address = try std.net.Address.parseIp(config.host, config.port);
        const listener = try address.listen(.{
            .reuse_address = true,
        });

        return Server{
            .allocator = allocator,
            .config = config,
            .routes = router.Router.init(allocator),
            .running = false,
            .listener = listener,
        };
    }

    pub fn deinit(self: *Server) void {
        self.routes.deinit();
        self.listener.deinit();
    }

    pub fn start(self: *Server) !void {
        self.running = true;

        std.log.info("Server listening on {s}:{d}", .{
            self.config.host,
            self.config.port,
        });

        while (self.running) {
            const connection = self.listener.accept() catch |err| {
                if (err == error.ConnectionAborted) continue;
                std.log.err("Accept error: {}", .{err});
                continue;
            };

            // Handle connection in current thread (single-threaded)
            self.handleConnection(connection) catch |err| {
                std.log.err("Connection error: {}", .{err});
            };
        }
    }

    fn handleConnection(self: *Server, conn: std.net.Server.Connection) !void {
        defer conn.stream.close();

        var keep_alive = self.config.keep_alive;

        while (keep_alive) {
            var req = request.Request.parse(
                self.allocator,
                conn.stream.reader(),
                self.config.max_request_size,
            ) catch |err| {
                switch (err) {
                    error.ConnectionClosed, error.EndOfStream => break,
                    else => {
                        const res = response.Response.init(self.allocator);
                        try res.sendError(conn.stream.writer(), 400, "Bad Request");
                        break;
                    },
                }
            };
            defer req.deinit();

            // ตรวจสอบ keep-alive
            const connection_header = req.getHeader("Connection");
            if (connection_header) |h| {
                if (std.ascii.eqlIgnoreCase(h, "close")) {
                    keep_alive = false;
                }
            } else {
                // HTTP/1.0 default = close, HTTP/1.1 default = keep-alive
                keep_alive = std.mem.eql(u8, req.version, "HTTP/1.1");
            }

            // Dispatch request
            var res = response.Response.init(self.allocator);
            defer res.deinit();

            self.routes.dispatch(&req, &res) catch |err| {
                std.log.err("Handler error: {}", .{err});
                res.setStatus(500);
                res.setBody("Internal Server Error");
            };

            // ส่ง response
            if (!keep_alive) {
                try res.setHeader("Connection", "close");
            } else {
                try res.setHeader("Connection", "keep-alive");
                try res.setHeader("Keep-Alive", "timeout=5, max=1000");
            }

            try res.send(conn.stream.writer());
        }
    }

    pub fn get(self: *Server, path: []const u8, handler: router.HandlerFn) !void {
        try self.routes.addRoute(.GET, path, handler);
    }

    pub fn post(self: *Server, path: []const u8, handler: router.HandlerFn) !void {
        try self.routes.addRoute(.POST, path, handler);
    }

    pub fn put(self: *Server, path: []const u8, handler: router.HandlerFn) !void {
        try self.routes.addRoute(.PUT, path, handler);
    }

    pub fn delete(self: *Server, path: []const u8, handler: router.HandlerFn) !void {
        try self.routes.addRoute(.DELETE, path, handler);
    }

    pub fn stop(self: *Server) void {
        self.running = false;
    }
};
```

### src/main.zig

```zig
const std = @import("std");
const Server = @import("server.zig").Server;
const ServerConfig = @import("server.zig").ServerConfig;
const json_api = @import("json_api.zig");
const static = @import("static.zig");
const middleware = @import("middleware.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const config = ServerConfig{
        .host = "127.0.0.1",
        .port = 8080,
        .static_dir = "public",
        .keep_alive = true,
    };

    var server = try Server.init(allocator, config);
    defer server.deinit();

    // Register routes
    try server.get("/", handleHome);
    try server.get("/api/users", json_api.handleGetUsers);
    try server.post("/api/users", json_api.handleCreateUser);
    try server.get("/api/users/:id", json_api.handleGetUser);
    try server.put("/api/users/:id", json_api.handleUpdateUser);
    try server.delete("/api/users/:id", json_api.handleDeleteUser);
    try server.get("/health", handleHealth);

    // Static files (catch-all)
    try server.get("/*", static.handleStatic);

    std.log.info("Starting HTTP server...", .{});
    try server.start();
}

fn handleHome(req: *@import("request.zig").Request, res: *@import("response.zig").Response) !void {
    _ = req;
    try res.setHeader("Content-Type", "text/html");
    res.setBody(
        \\<!DOCTYPE html>
        \\<html>
        \\<head><title>Zig HTTP Server</title></head>
        \\<body>
        \\  <h1>สวัสดี! นี่คือ Zig HTTP Server</h1>
        \\  <p>API Endpoints:</p>
        \\  <ul>
        \\    <li>GET /api/users - รายการ users</li>
        \\    <li>POST /api/users - สร้าง user ใหม่</li>
        \\  </ul>
        \\</body>
        \\</html>
    );
}

fn handleHealth(req: *@import("request.zig").Request, res: *@import("response.zig").Response) !void {
    _ = req;
    try res.setHeader("Content-Type", "application/json");
    res.setBody("{\"status\":\"ok\",\"version\":\"1.0.0\"}");
}
```

---

## ขั้นตอนที่ 613: HTTP Request Parser

### src/request.zig

```zig
const std = @import("std");

pub const Method = enum {
    GET,
    POST,
    PUT,
    DELETE,
    PATCH,
    HEAD,
    OPTIONS,

    pub fn fromString(s: []const u8) ?Method {
        if (std.mem.eql(u8, s, "GET")) return .GET;
        if (std.mem.eql(u8, s, "POST")) return .POST;
        if (std.mem.eql(u8, s, "PUT")) return .PUT;
        if (std.mem.eql(u8, s, "DELETE")) return .DELETE;
        if (std.mem.eql(u8, s, "PATCH")) return .PATCH;
        if (std.mem.eql(u8, s, "HEAD")) return .HEAD;
        if (std.mem.eql(u8, s, "OPTIONS")) return .OPTIONS;
        return null;
    }

    pub fn toString(self: Method) []const u8 {
        return switch (self) {
            .GET => "GET",
            .POST => "POST",
            .PUT => "PUT",
            .DELETE => "DELETE",
            .PATCH => "PATCH",
            .HEAD => "HEAD",
            .OPTIONS => "OPTIONS",
        };
    }
};

pub const QueryParam = struct {
    key: []const u8,
    value: []const u8,
};

pub const Request = struct {
    allocator: std.mem.Allocator,
    method: Method,
    path: []const u8,
    version: []const u8,
    headers: std.StringHashMap([]const u8),
    body: ?[]const u8,
    query_params: std.ArrayList(QueryParam),
    path_params: std.StringHashMap([]const u8),
    raw: []u8,

    pub fn parse(
        allocator: std.mem.Allocator,
        reader: anytype,
        max_size: usize,
    ) !Request {
        // อ่าน request line
        var buf = std.ArrayList(u8).init(allocator);
        defer buf.deinit();

        // อ่านทีละ byte จนกว่าจะพบ \r\n\r\n
        var total_size: usize = 0;
        var prev_char: u8 = 0;
        var prev_prev_char: u8 = 0;
        var prev_prev_prev_char: u8 = 0;

        while (true) {
            const byte = reader.readByte() catch |err| {
                if (err == error.EndOfStream and buf.items.len > 0) break;
                return err;
            };

            try buf.append(byte);
            total_size += 1;

            if (total_size > max_size) return error.RequestTooLarge;

            // ตรวจสอบ end of headers: \r\n\r\n
            if (byte == '\n' and prev_char == '\r' and
                prev_prev_char == '\n' and prev_prev_prev_char == '\r')
            {
                break;
            }

            prev_prev_prev_char = prev_prev_char;
            prev_prev_char = prev_char;
            prev_char = byte;
        }

        const raw_headers = try buf.toOwnedSlice();

        // Parse request line
        var lines = std.mem.splitSequence(u8, raw_headers, "\r\n");

        const request_line = lines.next() orelse return error.InvalidRequest;
        var parts = std.mem.splitScalar(u8, request_line, ' ');

        const method_str = parts.next() orelse return error.InvalidRequest;
        const raw_path = parts.next() orelse return error.InvalidRequest;
        const version = parts.next() orelse return error.InvalidRequest;

        const method = Method.fromString(method_str) orelse return error.InvalidMethod;

        // แยก path และ query string
        var path_query = std.mem.splitScalar(u8, raw_path, '?');
        const path = try allocator.dupe(u8, path_query.next() orelse raw_path);
        const query_string = path_query.next();

        // Parse headers
        var headers = std.StringHashMap([]const u8).init(allocator);
        while (lines.next()) |line| {
            if (line.len == 0) break;
            const colon_pos = std.mem.indexOfScalar(u8, line, ':') orelse continue;
            const key = std.mem.trim(u8, line[0..colon_pos], " \t");
            const value = std.mem.trim(u8, line[colon_pos + 1 ..], " \t");
            const key_lower = try std.ascii.allocLowerString(allocator, key);
            try headers.put(key_lower, try allocator.dupe(u8, value));
        }

        // Parse query params
        var query_params = std.ArrayList(QueryParam).init(allocator);
        if (query_string) |qs| {
            var param_iter = std.mem.splitScalar(u8, qs, '&');
            while (param_iter.next()) |param| {
                const eq_pos = std.mem.indexOfScalar(u8, param, '=');
                if (eq_pos) |pos| {
                    const k = try allocator.dupe(u8, param[0..pos]);
                    const v = try allocator.dupe(u8, param[pos + 1 ..]);
                    try query_params.append(.{ .key = k, .value = v });
                } else {
                    const k = try allocator.dupe(u8, param);
                    try query_params.append(.{ .key = k, .value = "" });
                }
            }
        }

        // อ่าน body ถ้ามี Content-Length
        var body: ?[]u8 = null;
        if (headers.get("content-length")) |cl_str| {
            const content_length = std.fmt.parseInt(usize, cl_str, 10) catch 0;
            if (content_length > 0 and content_length <= max_size) {
                const body_buf = try allocator.alloc(u8, content_length);
                const bytes_read = try reader.readAll(body_buf);
                if (bytes_read < content_length) {
                    allocator.free(body_buf);
                } else {
                    body = body_buf;
                }
            }
        }

        return Request{
            .allocator = allocator,
            .method = method,
            .path = path,
            .version = try allocator.dupe(u8, version),
            .headers = headers,
            .body = body,
            .query_params = query_params,
            .path_params = std.StringHashMap([]const u8).init(allocator),
            .raw = raw_headers,
        };
    }

    pub fn deinit(self: *Request) void {
        self.allocator.free(self.path);
        self.allocator.free(self.version);
        self.allocator.free(self.raw);

        if (self.body) |b| self.allocator.free(b);

        var header_iter = self.headers.iterator();
        while (header_iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.headers.deinit();

        for (self.query_params.items) |param| {
            self.allocator.free(param.key);
            self.allocator.free(param.value);
        }
        self.query_params.deinit();

        var param_iter = self.path_params.iterator();
        while (param_iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.path_params.deinit();
    }

    pub fn getHeader(self: *const Request, name: []const u8) ?[]const u8 {
        // ค้นหา header แบบ case-insensitive
        var lower_buf: [256]u8 = undefined;
        const lower_name = std.ascii.lowerString(lower_buf[0..name.len], name);
        return self.headers.get(lower_name);
    }

    pub fn getQuery(self: *const Request, key: []const u8) ?[]const u8 {
        for (self.query_params.items) |param| {
            if (std.mem.eql(u8, param.key, key)) return param.value;
        }
        return null;
    }

    pub fn getPathParam(self: *const Request, key: []const u8) ?[]const u8 {
        return self.path_params.get(key);
    }

    pub fn isJson(self: *const Request) bool {
        const ct = self.getHeader("content-type") orelse return false;
        return std.mem.startsWith(u8, ct, "application/json");
    }
};

// Tests
test "Parse GET request" {
    const allocator = std.testing.allocator;

    const raw = "GET /api/users?page=1 HTTP/1.1\r\nHost: localhost\r\nAccept: application/json\r\n\r\n";
    var stream = std.io.fixedBufferStream(raw);

    var req = try Request.parse(allocator, stream.reader(), 65536);
    defer req.deinit();

    try std.testing.expectEqual(Method.GET, req.method);
    try std.testing.expectEqualStrings("/api/users", req.path);
    try std.testing.expectEqualStrings("HTTP/1.1", req.version);
    try std.testing.expectEqualStrings("localhost", req.getHeader("Host").?);
    try std.testing.expectEqualStrings("1", req.getQuery("page").?);
}
```

---

## ขั้นตอนที่ 614: Route Matching

### src/router.zig

```zig
const std = @import("std");
const request = @import("request.zig");
const response = @import("response.zig");

pub const HandlerFn = *const fn (*request.Request, *response.Response) anyerror!void;

pub const Route = struct {
    method: request.Method,
    pattern: []const u8,
    handler: HandlerFn,

    /// ตรวจสอบว่า path ตรงกับ pattern หรือไม่
    pub fn matches(self: Route, method: request.Method, path: []const u8) bool {
        if (self.method != method) return false;
        return matchPattern(self.pattern, path);
    }

    /// จับ path parameters
    pub fn extractParams(
        self: Route,
        allocator: std.mem.Allocator,
        path: []const u8,
    ) !std.StringHashMap([]const u8) {
        var params = std.StringHashMap([]const u8).init(allocator);

        var pattern_parts = std.mem.splitScalar(u8, self.pattern, '/');
        var path_parts = std.mem.splitScalar(u8, path, '/');

        while (pattern_parts.next()) |pp| {
            const path_part = path_parts.next() orelse break;

            if (pp.len > 0 and pp[0] == ':') {
                // Dynamic segment
                const key = pp[1..];
                const value = path_part;
                try params.put(
                    try allocator.dupe(u8, key),
                    try allocator.dupe(u8, value),
                );
            }
        }

        return params;
    }
};

fn matchPattern(pattern: []const u8, path: []const u8) bool {
    // Wildcard match
    if (std.mem.endsWith(u8, pattern, "/*")) {
        const prefix = pattern[0 .. pattern.len - 2];
        return std.mem.startsWith(u8, path, prefix);
    }

    // Exact match หรือ dynamic segments
    var pp_iter = std.mem.splitScalar(u8, pattern, '/');
    var path_iter = std.mem.splitScalar(u8, path, '/');

    while (pp_iter.next()) |pp| {
        const path_part = path_iter.next() orelse return false;

        if (pp.len > 0 and pp[0] == ':') {
            // Dynamic - ตรงเสมอ
            continue;
        }

        if (!std.mem.eql(u8, pp, path_part)) return false;
    }

    // ตรวจสอบว่า path ไม่มีส่วนที่เหลือ
    return path_iter.next() == null;
}

pub const Router = struct {
    allocator: std.mem.Allocator,
    routes: std.ArrayList(Route),
    not_found_handler: ?HandlerFn,

    pub fn init(allocator: std.mem.Allocator) Router {
        return Router{
            .allocator = allocator,
            .routes = std.ArrayList(Route).init(allocator),
            .not_found_handler = null,
        };
    }

    pub fn deinit(self: *Router) void {
        for (self.routes.items) |route| {
            self.allocator.free(route.pattern);
        }
        self.routes.deinit();
    }

    pub fn addRoute(
        self: *Router,
        method: request.Method,
        pattern: []const u8,
        handler: HandlerFn,
    ) !void {
        const pattern_copy = try self.allocator.dupe(u8, pattern);
        try self.routes.append(.{
            .method = method,
            .pattern = pattern_copy,
            .handler = handler,
        });
    }

    pub fn setNotFound(self: *Router, handler: HandlerFn) void {
        self.not_found_handler = handler;
    }

    pub fn dispatch(
        self: *Router,
        req: *request.Request,
        res: *response.Response,
    ) !void {
        for (self.routes.items) |route| {
            if (route.matches(req.method, req.path)) {
                // Extract path parameters
                var params = try route.extractParams(self.allocator, req.path);
                defer {
                    var iter = params.iterator();
                    while (iter.next()) |entry| {
                        self.allocator.free(entry.key_ptr.*);
                        self.allocator.free(entry.value_ptr.*);
                    }
                    params.deinit();
                }

                // Copy params to request
                var param_iter = params.iterator();
                while (param_iter.next()) |entry| {
                    try req.path_params.put(
                        try self.allocator.dupe(u8, entry.key_ptr.*),
                        try self.allocator.dupe(u8, entry.value_ptr.*),
                    );
                }

                try route.handler(req, res);
                return;
            }
        }

        // 404 Not Found
        if (self.not_found_handler) |handler| {
            try handler(req, res);
        } else {
            res.setStatus(404);
            try res.setHeader("Content-Type", "application/json");
            res.setBody("{\"error\":\"Not Found\"}");
        }
    }
};

// Tests
test "Route matching" {
    try std.testing.expect(matchPattern("/api/users", "/api/users"));
    try std.testing.expect(matchPattern("/api/users/:id", "/api/users/123"));
    try std.testing.expect(!matchPattern("/api/users", "/api/posts"));
    try std.testing.expect(matchPattern("/static/*", "/static/css/style.css"));
}
```

---

## ขั้นตอนที่ 615: Middleware Chain

### src/middleware.zig

```zig
const std = @import("std");
const request = @import("request.zig");
const response = @import("response.zig");

pub const MiddlewareFn = *const fn (
    *request.Request,
    *response.Response,
    NextFn,
) anyerror!void;

pub const NextFn = *const fn (*request.Request, *response.Response) anyerror!void;

/// Logging middleware
pub fn logger(
    req: *request.Request,
    res: *response.Response,
    next: NextFn,
) !void {
    const start = std.time.milliTimestamp();

    try next(req, res);

    const elapsed = std.time.milliTimestamp() - start;
    std.log.info("{s} {s} {d} {d}ms", .{
        req.method.toString(),
        req.path,
        res.status_code,
        elapsed,
    });
}

/// CORS middleware
pub fn cors(
    req: *request.Request,
    res: *response.Response,
    next: NextFn,
) !void {
    try res.setHeader("Access-Control-Allow-Origin", "*");
    try res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    try res.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");

    if (req.method == .OPTIONS) {
        res.setStatus(204);
        return;
    }

    try next(req, res);
}

/// Rate limiting middleware (simple in-memory)
pub const RateLimiter = struct {
    allocator: std.mem.Allocator,
    requests: std.StringHashMap(RequestCount),
    max_requests: u32,
    window_ms: i64,

    const RequestCount = struct {
        count: u32,
        window_start: i64,
    };

    pub fn init(allocator: std.mem.Allocator, max_req: u32, window_ms: i64) RateLimiter {
        return RateLimiter{
            .allocator = allocator,
            .requests = std.StringHashMap(RequestCount).init(allocator),
            .max_requests = max_req,
            .window_ms = window_ms,
        };
    }

    pub fn deinit(self: *RateLimiter) void {
        var iter = self.requests.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
        }
        self.requests.deinit();
    }

    pub fn isAllowed(self: *RateLimiter, client_ip: []const u8) !bool {
        const now = std.time.milliTimestamp();

        if (self.requests.getPtr(client_ip)) |rc| {
            if (now - rc.window_start > self.window_ms) {
                // Reset window
                rc.count = 1;
                rc.window_start = now;
                return true;
            }

            if (rc.count >= self.max_requests) {
                return false;
            }

            rc.count += 1;
            return true;
        } else {
            // New client
            const key = try self.allocator.dupe(u8, client_ip);
            try self.requests.put(key, .{
                .count = 1,
                .window_start = now,
            });
            return true;
        }
    }
};

/// Authentication middleware
pub fn basicAuth(
    req: *request.Request,
    res: *response.Response,
    next: NextFn,
    username: []const u8,
    password: []const u8,
) !void {
    const auth_header = req.getHeader("Authorization") orelse {
        res.setStatus(401);
        try res.setHeader("WWW-Authenticate", "Basic realm=\"Protected\"");
        res.setBody("Unauthorized");
        return;
    };

    if (!std.mem.startsWith(u8, auth_header, "Basic ")) {
        res.setStatus(401);
        res.setBody("Unauthorized");
        return;
    }

    // Decode base64
    const encoded = auth_header[6..];
    var decode_buf: [256]u8 = undefined;
    const decoded_len = std.base64.standard.Decoder.calcSizeForSlice(encoded) catch {
        res.setStatus(401);
        return;
    };

    if (decoded_len > decode_buf.len) {
        res.setStatus(401);
        return;
    }

    std.base64.standard.Decoder.decode(&decode_buf, encoded) catch {
        res.setStatus(401);
        return;
    };

    const credentials = decode_buf[0..decoded_len];
    const colon_pos = std.mem.indexOfScalar(u8, credentials, ':') orelse {
        res.setStatus(401);
        return;
    };

    const req_username = credentials[0..colon_pos];
    const req_password = credentials[colon_pos + 1 ..];

    if (!std.mem.eql(u8, req_username, username) or
        !std.mem.eql(u8, req_password, password))
    {
        res.setStatus(401);
        res.setBody("Invalid credentials");
        return;
    }

    try next(req, res);
}
```

---

## ขั้นตอนที่ 616: JSON API Endpoints

### src/response.zig

```zig
const std = @import("std");

pub const Response = struct {
    allocator: std.mem.Allocator,
    status_code: u16,
    headers: std.StringHashMap([]const u8),
    body: ?[]const u8,
    body_owned: bool,

    pub fn init(allocator: std.mem.Allocator) Response {
        return Response{
            .allocator = allocator,
            .status_code = 200,
            .headers = std.StringHashMap([]const u8).init(allocator),
            .body = null,
            .body_owned = false,
        };
    }

    pub fn deinit(self: *Response) void {
        var iter = self.headers.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.headers.deinit();

        if (self.body_owned) {
            if (self.body) |b| self.allocator.free(b);
        }
    }

    pub fn setStatus(self: *Response, code: u16) void {
        self.status_code = code;
    }

    pub fn setHeader(self: *Response, name: []const u8, value: []const u8) !void {
        const k = try self.allocator.dupe(u8, name);
        const v = try self.allocator.dupe(u8, value);
        try self.headers.put(k, v);
    }

    pub fn setBody(self: *Response, body: []const u8) void {
        self.body = body;
        self.body_owned = false;
    }

    pub fn setBodyOwned(self: *Response, body: []u8) void {
        self.body = body;
        self.body_owned = true;
    }

    pub fn json(self: *Response, data: anytype) !void {
        try self.setHeader("Content-Type", "application/json");
        var buf = std.ArrayList(u8).init(self.allocator);
        try std.json.stringify(data, .{}, buf.writer());
        self.setBodyOwned(try buf.toOwnedSlice());
    }

    pub fn sendError(self: *const Response, writer: anytype, code: u16, message: []const u8) !void {
        const status_text = statusText(code);
        const body = std.fmt.allocPrint(
            self.allocator,
            "{{\"error\":\"{s}\"}}",
            .{message},
        ) catch message;
        defer self.allocator.free(body);

        try writer.print(
            "HTTP/1.1 {d} {s}\r\nContent-Type: application/json\r\nContent-Length: {d}\r\n\r\n{s}",
            .{ code, status_text, body.len, body },
        );
    }

    pub fn send(self: *Response, writer: anytype) !void {
        const status_text = statusText(self.status_code);
        const body = self.body orelse "";

        // Status line
        try writer.print("HTTP/1.1 {d} {s}\r\n", .{ self.status_code, status_text });

        // Headers
        var header_iter = self.headers.iterator();
        while (header_iter.next()) |entry| {
            try writer.print("{s}: {s}\r\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        }

        // Content-Length ถ้ายังไม่มี
        if (!self.headers.contains("Content-Length")) {
            try writer.print("Content-Length: {d}\r\n", .{body.len});
        }

        // Content-Type default
        if (!self.headers.contains("Content-Type")) {
            try writer.writeAll("Content-Type: text/plain\r\n");
        }

        // Date header
        try writer.writeAll("Server: Zig HTTP Server/1.0\r\n");

        try writer.writeAll("\r\n");
        try writer.writeAll(body);
    }

    fn statusText(code: u16) []const u8 {
        return switch (code) {
            200 => "OK",
            201 => "Created",
            204 => "No Content",
            301 => "Moved Permanently",
            302 => "Found",
            304 => "Not Modified",
            400 => "Bad Request",
            401 => "Unauthorized",
            403 => "Forbidden",
            404 => "Not Found",
            405 => "Method Not Allowed",
            409 => "Conflict",
            422 => "Unprocessable Entity",
            429 => "Too Many Requests",
            500 => "Internal Server Error",
            502 => "Bad Gateway",
            503 => "Service Unavailable",
            else => "Unknown",
        };
    }
};
```

### src/json_api.zig

```zig
const std = @import("std");
const request = @import("request.zig");
const response = @import("response.zig");

const User = struct {
    id: u32,
    name: []const u8,
    email: []const u8,
    created_at: i64,
};

// In-memory user store (production ควรใช้ database)
var users: std.ArrayList(User) = undefined;
var next_user_id: u32 = 1;
var users_initialized = false;

fn ensureInitialized(allocator: std.mem.Allocator) void {
    if (!users_initialized) {
        users = std.ArrayList(User).init(allocator);
        users_initialized = true;
    }
}

pub fn handleGetUsers(req: *request.Request, res: *response.Response) !void {
    ensureInitialized(res.allocator);
    _ = req;

    try res.setHeader("Content-Type", "application/json");

    var buf = std.ArrayList(u8).init(res.allocator);
    defer buf.deinit();

    const writer = buf.writer();
    try writer.writeAll("[");

    for (users.items, 0..) |user, i| {
        if (i > 0) try writer.writeAll(",");
        try writer.print(
            "{{\"id\":{d},\"name\":\"{s}\",\"email\":\"{s}\"}}",
            .{ user.id, user.name, user.email },
        );
    }

    try writer.writeAll("]");

    res.setBodyOwned(try buf.toOwnedSlice());
}

pub fn handleGetUser(req: *request.Request, res: *response.Response) !void {
    ensureInitialized(res.allocator);

    const id_str = req.getPathParam("id") orelse {
        res.setStatus(400);
        res.setBody("{\"error\":\"Missing id\"}");
        return;
    };

    const id = std.fmt.parseInt(u32, id_str, 10) catch {
        res.setStatus(400);
        res.setBody("{\"error\":\"Invalid id\"}");
        return;
    };

    for (users.items) |user| {
        if (user.id == id) {
            try res.setHeader("Content-Type", "application/json");
            const body = try std.fmt.allocPrint(
                res.allocator,
                "{{\"id\":{d},\"name\":\"{s}\",\"email\":\"{s}\"}}",
                .{ user.id, user.name, user.email },
            );
            res.setBodyOwned(body);
            return;
        }
    }

    res.setStatus(404);
    res.setBody("{\"error\":\"User not found\"}");
}

pub fn handleCreateUser(req: *request.Request, res: *response.Response) !void {
    ensureInitialized(res.allocator);

    const body = req.body orelse {
        res.setStatus(400);
        res.setBody("{\"error\":\"Request body required\"}");
        return;
    };

    const parsed = std.json.parseFromSlice(std.json.Value, res.allocator, body, .{}) catch {
        res.setStatus(400);
        res.setBody("{\"error\":\"Invalid JSON\"}");
        return;
    };
    defer parsed.deinit();

    const obj = parsed.value.object;

    const name = if (obj.get("name")) |v|
        if (v == .string) v.string else {
            res.setStatus(400);
            res.setBody("{\"error\":\"name must be string\"}");
            return;
        }
    else {
        res.setStatus(400);
        res.setBody("{\"error\":\"name required\"}");
        return;
    };

    const email = if (obj.get("email")) |v|
        if (v == .string) v.string else {
            res.setStatus(400);
            res.setBody("{\"error\":\"email must be string\"}");
            return;
        }
    else {
        res.setStatus(400);
        res.setBody("{\"error\":\"email required\"}");
        return;
    };

    const new_user = User{
        .id = next_user_id,
        .name = name,
        .email = email,
        .created_at = std.time.timestamp(),
    };
    next_user_id += 1;

    try users.append(new_user);

    res.setStatus(201);
    try res.setHeader("Content-Type", "application/json");

    const response_body = try std.fmt.allocPrint(
        res.allocator,
        "{{\"id\":{d},\"name\":\"{s}\",\"email\":\"{s}\"}}",
        .{ new_user.id, new_user.name, new_user.email },
    );
    res.setBodyOwned(response_body);
}

pub fn handleUpdateUser(req: *request.Request, res: *response.Response) !void {
    _ = req;
    res.setStatus(200);
    res.setBody("{\"message\":\"updated\"}");
}

pub fn handleDeleteUser(req: *request.Request, res: *response.Response) !void {
    ensureInitialized(res.allocator);

    const id_str = req.getPathParam("id") orelse {
        res.setStatus(400);
        res.setBody("{\"error\":\"Missing id\"}");
        return;
    };

    const id = std.fmt.parseInt(u32, id_str, 10) catch {
        res.setStatus(400);
        res.setBody("{\"error\":\"Invalid id\"}");
        return;
    };

    for (users.items, 0..) |user, i| {
        if (user.id == id) {
            _ = users.orderedRemove(i);
            res.setStatus(204);
            return;
        }
    }

    res.setStatus(404);
    res.setBody("{\"error\":\"User not found\"}");
}
```

---

## ขั้นตอนที่ 617: Static File Serving

### src/static.zig

```zig
const std = @import("std");
const request = @import("request.zig");
const response = @import("response.zig");

const mime_types = std.StaticStringMap([]const u8).initComptime(.{
    .{ ".html", "text/html; charset=utf-8" },
    .{ ".css", "text/css" },
    .{ ".js", "application/javascript" },
    .{ ".json", "application/json" },
    .{ ".png", "image/png" },
    .{ ".jpg", "image/jpeg" },
    .{ ".jpeg", "image/jpeg" },
    .{ ".gif", "image/gif" },
    .{ ".svg", "image/svg+xml" },
    .{ ".ico", "image/x-icon" },
    .{ ".txt", "text/plain" },
    .{ ".pdf", "application/pdf" },
    .{ ".woff", "font/woff" },
    .{ ".woff2", "font/woff2" },
});

pub fn handleStatic(req: *request.Request, res: *response.Response) !void {
    const static_dir = "public";

    // ป้องกัน path traversal
    if (std.mem.indexOf(u8, req.path, "..") != null) {
        res.setStatus(403);
        res.setBody("Forbidden");
        return;
    }

    // สร้าง file path
    const file_path = if (std.mem.eql(u8, req.path, "/"))
        try std.fmt.allocPrint(res.allocator, "{s}/index.html", .{static_dir})
    else
        try std.fmt.allocPrint(res.allocator, "{s}{s}", .{ static_dir, req.path });
    defer res.allocator.free(file_path);

    // เปิดไฟล์
    const file = std.fs.cwd().openFile(file_path, .{}) catch |err| {
        switch (err) {
            error.FileNotFound => {
                res.setStatus(404);
                res.setBody("<h1>404 Not Found</h1>");
                try res.setHeader("Content-Type", "text/html");
            },
            error.AccessDenied => {
                res.setStatus(403);
                res.setBody("Forbidden");
            },
            else => {
                res.setStatus(500);
                res.setBody("Internal Server Error");
            },
        }
        return;
    };
    defer file.close();

    // อ่านไฟล์
    const file_size = try file.getEndPos();
    const content = try res.allocator.alloc(u8, file_size);
    _ = try file.readAll(content);
    res.setBodyOwned(content);

    // ตั้งค่า Content-Type
    const ext = std.fs.path.extension(file_path);
    const mime = mime_types.get(ext) orelse "application/octet-stream";
    try res.setHeader("Content-Type", mime);

    // Cache headers
    try res.setHeader("Cache-Control", "public, max-age=3600");

    // ETag (simple hash)
    const etag = try std.fmt.allocPrint(res.allocator, "\"{d}-{d}\"", .{ file_size, std.time.timestamp() });
    defer res.allocator.free(etag);
    try res.setHeader("ETag", etag);

    // ตรวจสอบ If-None-Match
    if (req.getHeader("If-None-Match")) |client_etag| {
        if (std.mem.eql(u8, client_etag, etag)) {
            res.setStatus(304);
            return;
        }
    }
}
```

---

## ขั้นตอนที่ 618: Connection Keep-alive

Keep-alive ถูก implement ใน `server.zig` แล้วในขั้นตอนที่ 612

---

## ขั้นตอนที่ 619: Error Responses

```zig
// เพิ่มใน response.zig - Error response helpers

pub fn notFound(self: *Response) !void {
    self.setStatus(404);
    try self.setHeader("Content-Type", "application/json");
    self.setBody("{\"error\":\"Not Found\",\"code\":404}");
}

pub fn badRequest(self: *Response, message: []const u8) !void {
    self.setStatus(400);
    try self.setHeader("Content-Type", "application/json");
    const body = try std.fmt.allocPrint(
        self.allocator,
        "{{\"error\":\"{s}\",\"code\":400}}",
        .{message},
    );
    self.setBodyOwned(body);
}

pub fn internalError(self: *Response) void {
    self.setStatus(500);
    self.setBody("{\"error\":\"Internal Server Error\",\"code\":500}");
}

pub fn unauthorized(self: *Response) !void {
    self.setStatus(401);
    try self.setHeader("Content-Type", "application/json");
    try self.setHeader("WWW-Authenticate", "Bearer");
    self.setBody("{\"error\":\"Unauthorized\",\"code\":401}");
}
```

---

## ขั้นตอนที่ 620: Load Testing

### load_test.zig - เครื่องมือทดสอบ performance

```zig
const std = @import("std");

const LoadTestConfig = struct {
    host: []const u8 = "127.0.0.1",
    port: u16 = 8080,
    path: []const u8 = "/",
    method: []const u8 = "GET",
    concurrency: u32 = 10,
    total_requests: u32 = 1000,
    timeout_ms: u32 = 5000,
};

const Stats = struct {
    total: u32 = 0,
    success: u32 = 0,
    failed: u32 = 0,
    total_ms: i64 = 0,
    min_ms: i64 = std.math.maxInt(i64),
    max_ms: i64 = 0,

    pub fn avg(self: Stats) f64 {
        if (self.total == 0) return 0;
        return @as(f64, @floatFromInt(self.total_ms)) / @as(f64, @floatFromInt(self.total));
    }
};

pub fn runLoadTest(allocator: std.mem.Allocator, config: LoadTestConfig) !Stats {
    var stats = Stats{};
    const address = try std.net.Address.parseIp(config.host, config.port);

    for (0..config.total_requests) |_| {
        const start = std.time.milliTimestamp();

        const stream = std.net.tcpConnectToAddress(address) catch {
            stats.failed += 1;
            stats.total += 1;
            continue;
        };
        defer stream.close();

        const request_str = try std.fmt.allocPrint(
            allocator,
            "{s} {s} HTTP/1.1\r\nHost: {s}:{d}\r\nConnection: close\r\n\r\n",
            .{ config.method, config.path, config.host, config.port },
        );
        defer allocator.free(request_str);

        stream.writeAll(request_str) catch {
            stats.failed += 1;
            stats.total += 1;
            continue;
        };

        // อ่าน response status
        var response_buf: [4096]u8 = undefined;
        const bytes_read = stream.readAll(&response_buf) catch 0;

        const elapsed = std.time.milliTimestamp() - start;

        if (bytes_read > 12) {
            // ตรวจสอบ status code
            const status_line = response_buf[0..@min(bytes_read, 12)];
            if (std.mem.startsWith(u8, status_line, "HTTP/1.1 2")) {
                stats.success += 1;
            } else {
                stats.failed += 1;
            }
        } else {
            stats.failed += 1;
        }

        stats.total += 1;
        stats.total_ms += elapsed;
        stats.min_ms = @min(stats.min_ms, elapsed);
        stats.max_ms = @max(stats.max_ms, elapsed);
    }

    return stats;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const config = LoadTestConfig{
        .host = "127.0.0.1",
        .port = 8080,
        .path = "/health",
        .total_requests = 100,
    };

    std.log.info("Running load test: {d} requests", .{config.total_requests});

    const stats = try runLoadTest(allocator, config);

    const stdout = std.io.getStdOut().writer();
    try stdout.print("\n=== Load Test Results ===\n", .{});
    try stdout.print("Total Requests: {d}\n", .{stats.total});
    try stdout.print("Successful:     {d}\n", .{stats.success});
    try stdout.print("Failed:         {d}\n", .{stats.failed});
    try stdout.print("Average:        {d:.2}ms\n", .{stats.avg()});
    try stdout.print("Min:            {d}ms\n", .{stats.min_ms});
    try stdout.print("Max:            {d}ms\n", .{stats.max_ms});
    try stdout.print("RPS:            {d:.0}\n", .{
        @as(f64, @floatFromInt(stats.total)) / (@as(f64, @floatFromInt(stats.total_ms)) / 1000.0),
    });
}
```

### สรุป Part 62

เราได้สร้าง HTTP Server ที่มีฟีเจอร์:
- TCP listener ที่ handle connections
- HTTP request parser ที่สมบูรณ์
- Route matching รองรับ dynamic segments
- Middleware chain (logger, CORS, auth)
- JSON API endpoints
- Static file serving พร้อม MIME types
- Connection keep-alive
- Error handling ที่ถูกต้อง
- Load testing tool

---
*ต่อไป Part 63: File System Tool*
