# Part 48: HTTP Server ใน Zig

## ภาพรวม

HTTP (HyperText Transfer Protocol) เป็นโปรโตคอลหลักของ World Wide Web ในบทนี้เราจะสร้าง HTTP server ตั้งแต่ศูนย์โดยใช้ Zig เพื่อทำความเข้าใจวิธีการทำงานของ HTTP อย่างลึกซึ้ง

---

## Step 471: HTTP protocol basics

HTTP เป็น text-based protocol ที่ทำงานบน TCP

### HTTP Request Format

```
GET /path HTTP/1.1\r\n
Host: localhost:8080\r\n
User-Agent: Mozilla/5.0\r\n
Accept: text/html\r\n
\r\n
[body - ถ้ามี]
```

### HTTP Response Format

```
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 13\r\n
\r\n
Hello, World!
```

### HTTP Status Codes

```zig
const std = @import("std");

const HttpStatus = enum(u16) {
    // 2xx Success
    ok = 200,
    created = 201,
    accepted = 202,
    no_content = 204,
    
    // 3xx Redirection
    moved_permanently = 301,
    found = 302,
    not_modified = 304,
    
    // 4xx Client Errors
    bad_request = 400,
    unauthorized = 401,
    forbidden = 403,
    not_found = 404,
    method_not_allowed = 405,
    
    // 5xx Server Errors
    internal_server_error = 500,
    not_implemented = 501,
    service_unavailable = 503,
    
    pub fn text(self: HttpStatus) []const u8 {
        return switch (self) {
            .ok => "OK",
            .created => "Created",
            .accepted => "Accepted",
            .no_content => "No Content",
            .moved_permanently => "Moved Permanently",
            .found => "Found",
            .not_modified => "Not Modified",
            .bad_request => "Bad Request",
            .unauthorized => "Unauthorized",
            .forbidden => "Forbidden",
            .not_found => "Not Found",
            .method_not_allowed => "Method Not Allowed",
            .internal_server_error => "Internal Server Error",
            .not_implemented => "Not Implemented",
            .service_unavailable => "Service Unavailable",
        };
    }
};

pub fn main() void {
    const status = HttpStatus.ok;
    std.debug.print("Status: {d} {s}\n", .{ @intFromEnum(status), status.text() });
}
```

### HTTP Methods

```zig
const HttpMethod = enum {
    GET,
    POST,
    PUT,
    DELETE,
    PATCH,
    HEAD,
    OPTIONS,
    
    pub fn fromString(s: []const u8) ?HttpMethod {
        const methods = std.StaticStringMap(HttpMethod).initComptime(.{
            .{ "GET", .GET },
            .{ "POST", .POST },
            .{ "PUT", .PUT },
            .{ "DELETE", .DELETE },
            .{ "PATCH", .PATCH },
            .{ "HEAD", .HEAD },
            .{ "OPTIONS", .OPTIONS },
        });
        return methods.get(s);
    }
    
    pub fn hasBody(self: HttpMethod) bool {
        return switch (self) {
            .POST, .PUT, .PATCH => true,
            else => false,
        };
    }
};
```

---

## Step 472: สร้าง HTTP server จาก scratch

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;

pub const HttpServer = struct {
    allocator: std.mem.Allocator,
    port: u16,
    running: std.atomic.Value(bool),
    
    pub fn init(allocator: std.mem.Allocator, port: u16) HttpServer {
        return .{
            .allocator = allocator,
            .port = port,
            .running = std.atomic.Value(bool).init(false),
        };
    }
    
    pub fn start(self: *HttpServer) !void {
        const addr = try net.Address.parseIp4("0.0.0.0", self.port);
        var server = try addr.listen(.{
            .reuse_address = true,
        });
        defer server.deinit();
        
        self.running.store(true, .release);
        std.debug.print("HTTP Server listening on http://localhost:{d}\n", .{self.port});
        
        while (self.running.load(.acquire)) {
            const conn = server.accept() catch |err| {
                if (!self.running.load(.acquire)) break;
                std.debug.print("Accept error: {}\n", .{err});
                continue;
            };
            
            const ctx = ConnectionContext{
                .stream = conn.stream,
                .address = conn.address,
                .allocator = self.allocator,
            };
            
            const t = Thread.spawn(.{}, handleConnection, .{ctx}) catch |err| {
                std.debug.print("Thread spawn error: {}\n", .{err});
                conn.stream.close();
                continue;
            };
            t.detach();
        }
    }
    
    pub fn stop(self: *HttpServer) void {
        self.running.store(false, .release);
    }
};

const ConnectionContext = struct {
    stream: net.Stream,
    address: net.Address,
    allocator: std.mem.Allocator,
};

fn handleConnection(ctx: ConnectionContext) void {
    defer ctx.stream.close();
    
    var arena = std.heap.ArenaAllocator.init(ctx.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    // อ่าน request
    var request_buf: [8192]u8 = undefined;
    const n = ctx.stream.read(&request_buf) catch return;
    if (n == 0) return;
    
    // ส่ง response ง่ายๆ
    const response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: 13\r\n\r\nHello, World!";
    _ = allocator;
    ctx.stream.writeAll(response) catch return;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    
    var server = HttpServer.init(gpa.allocator(), 8080);
    try server.start();
}
```

---

## Step 473: HTTP request parsing

```zig
const std = @import("std");

pub const HttpMethod = enum {
    GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS,
    
    pub fn fromString(s: []const u8) ?HttpMethod {
        if (std.mem.eql(u8, s, "GET")) return .GET;
        if (std.mem.eql(u8, s, "POST")) return .POST;
        if (std.mem.eql(u8, s, "PUT")) return .PUT;
        if (std.mem.eql(u8, s, "DELETE")) return .DELETE;
        if (std.mem.eql(u8, s, "PATCH")) return .PATCH;
        if (std.mem.eql(u8, s, "HEAD")) return .HEAD;
        if (std.mem.eql(u8, s, "OPTIONS")) return .OPTIONS;
        return null;
    }
};

pub const HttpRequest = struct {
    method: HttpMethod,
    path: []const u8,
    version: []const u8,
    headers: std.StringHashMap([]const u8),
    body: []const u8,
    query_string: ?[]const u8,
    
    pub fn deinit(self: *HttpRequest) void {
        self.headers.deinit();
    }
    
    pub fn getHeader(self: *const HttpRequest, name: []const u8) ?[]const u8 {
        return self.headers.get(name);
    }
    
    pub fn getContentLength(self: *const HttpRequest) usize {
        const cl = self.getHeader("content-length") orelse return 0;
        return std.fmt.parseInt(usize, cl, 10) catch 0;
    }
};

pub const ParseError = error{
    InvalidRequest,
    InvalidMethod,
    InvalidPath,
    HeaderTooLarge,
    MalformedHeader,
};

pub fn parseRequest(allocator: std.mem.Allocator, data: []const u8) !HttpRequest {
    var lines = std.mem.splitSequence(u8, data, "\r\n");
    
    // Parse request line
    const request_line = lines.next() orelse return ParseError.InvalidRequest;
    var parts = std.mem.splitScalar(u8, request_line, ' ');
    
    const method_str = parts.next() orelse return ParseError.InvalidRequest;
    const method = HttpMethod.fromString(method_str) orelse return ParseError.InvalidMethod;
    
    const full_path = parts.next() orelse return ParseError.InvalidPath;
    const version = parts.next() orelse return ParseError.InvalidRequest;
    
    // แยก path และ query string
    var path = full_path;
    var query_string: ?[]const u8 = null;
    
    if (std.mem.indexOfScalar(u8, full_path, '?')) |qpos| {
        path = full_path[0..qpos];
        query_string = full_path[qpos + 1..];
    }
    
    // Parse headers
    var headers = std.StringHashMap([]const u8).init(allocator);
    errdefer headers.deinit();
    
    while (lines.next()) |line| {
        if (line.len == 0) break; // เจอ empty line = จบ headers
        
        const colon_pos = std.mem.indexOfScalar(u8, line, ':') orelse return ParseError.MalformedHeader;
        
        const key_raw = line[0..colon_pos];
        const value_raw = std.mem.trimLeft(u8, line[colon_pos + 1..], " ");
        
        // lowercase key
        const key = try allocator.alloc(u8, key_raw.len);
        for (key_raw, 0..) |c, i| {
            key[i] = std.ascii.toLower(c);
        }
        
        try headers.put(key, value_raw);
    }
    
    // ดึง body
    const header_end = std.mem.indexOf(u8, data, "\r\n\r\n") orelse data.len;
    const body = if (header_end + 4 < data.len) data[header_end + 4..] else "";
    
    return HttpRequest{
        .method = method,
        .path = path,
        .version = version,
        .headers = headers,
        .body = body,
        .query_string = query_string,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const raw_request =
        "GET /api/users?page=1&limit=10 HTTP/1.1\r\n" ++
        "Host: localhost:8080\r\n" ++
        "Accept: application/json\r\n" ++
        "Authorization: Bearer token123\r\n" ++
        "\r\n";
    
    var req = try parseRequest(allocator, raw_request);
    defer req.deinit();
    
    std.debug.print("Method: {}\n", .{req.method});
    std.debug.print("Path: {s}\n", .{req.path});
    std.debug.print("Version: {s}\n", .{req.version});
    std.debug.print("Query: {s}\n", .{req.query_string orelse "none"});
    std.debug.print("Host: {s}\n", .{req.getHeader("host") orelse "none"});
    std.debug.print("Auth: {s}\n", .{req.getHeader("authorization") orelse "none"});
}
```

---

## Step 474: HTTP response building

```zig
const std = @import("std");

pub const HttpResponse = struct {
    allocator: std.mem.Allocator,
    status: u16,
    status_text: []const u8,
    headers: std.StringHashMap([]const u8),
    body: std.ArrayList(u8),
    
    pub fn init(allocator: std.mem.Allocator) HttpResponse {
        return .{
            .allocator = allocator,
            .status = 200,
            .status_text = "OK",
            .headers = std.StringHashMap([]const u8).init(allocator),
            .body = std.ArrayList(u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *HttpResponse) void {
        self.headers.deinit();
        self.body.deinit();
    }
    
    pub fn setStatus(self: *HttpResponse, code: u16) void {
        self.status = code;
        self.status_text = statusText(code);
    }
    
    pub fn setHeader(self: *HttpResponse, key: []const u8, value: []const u8) !void {
        try self.headers.put(key, value);
    }
    
    pub fn writeBody(self: *HttpResponse, data: []const u8) !void {
        try self.body.appendSlice(data);
    }
    
    pub fn json(self: *HttpResponse, data: []const u8) !void {
        try self.setHeader("Content-Type", "application/json");
        try self.writeBody(data);
    }
    
    pub fn html(self: *HttpResponse, data: []const u8) !void {
        try self.setHeader("Content-Type", "text/html; charset=utf-8");
        try self.writeBody(data);
    }
    
    pub fn text(self: *HttpResponse, data: []const u8) !void {
        try self.setHeader("Content-Type", "text/plain; charset=utf-8");
        try self.writeBody(data);
    }
    
    pub fn redirect(self: *HttpResponse, location: []const u8) !void {
        self.setStatus(302);
        try self.setHeader("Location", location);
    }
    
    pub fn serialize(self: *HttpResponse, writer: anytype) !void {
        // Status line
        try writer.print("HTTP/1.1 {d} {s}\r\n", .{ self.status, self.status_text });
        
        // Content-Length header อัตโนมัติ
        var cl_buf: [20]u8 = undefined;
        const cl_str = try std.fmt.bufPrint(&cl_buf, "{d}", .{self.body.items.len});
        try writer.print("Content-Length: {s}\r\n", .{cl_str});
        
        // Custom headers
        var iter = self.headers.iterator();
        while (iter.next()) |entry| {
            try writer.print("{s}: {s}\r\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        }
        
        // End of headers
        try writer.writeAll("\r\n");
        
        // Body
        try writer.writeAll(self.body.items);
    }
    
    fn statusText(code: u16) []const u8 {
        return switch (code) {
            200 => "OK",
            201 => "Created",
            204 => "No Content",
            301 => "Moved Permanently",
            302 => "Found",
            400 => "Bad Request",
            401 => "Unauthorized",
            403 => "Forbidden",
            404 => "Not Found",
            405 => "Method Not Allowed",
            500 => "Internal Server Error",
            else => "Unknown",
        };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var resp = HttpResponse.init(allocator);
    defer resp.deinit();
    
    try resp.setHeader("Server", "ZigHTTP/1.0");
    try resp.setHeader("X-Frame-Options", "DENY");
    try resp.json("{\"message\": \"Hello, World!\", \"status\": \"ok\"}");
    
    const stdout = std.io.getStdOut().writer();
    try resp.serialize(stdout);
}
```

---

## Step 475: Routing

```zig
const std = @import("std");

pub const RouteHandler = *const fn (
    req: *const HttpRequest,
    res: *HttpResponse,
    allocator: std.mem.Allocator,
) anyerror!void;

pub const Route = struct {
    method: HttpMethod,
    pattern: []const u8,
    handler: RouteHandler,
    
    pub fn matches(self: Route, method: HttpMethod, path: []const u8) bool {
        if (self.method != method) return false;
        return matchPattern(self.pattern, path);
    }
};

fn matchPattern(pattern: []const u8, path: []const u8) bool {
    if (std.mem.eql(u8, pattern, path)) return true;
    
    // Wildcard matching
    if (std.mem.endsWith(u8, pattern, "*")) {
        const prefix = pattern[0..pattern.len - 1];
        return std.mem.startsWith(u8, path, prefix);
    }
    
    // Parameter matching (:param)
    var pat_it = std.mem.splitScalar(u8, pattern, '/');
    var path_it = std.mem.splitScalar(u8, path, '/');
    
    while (true) {
        const pat_seg = pat_it.next();
        const path_seg = path_it.next();
        
        if (pat_seg == null and path_seg == null) return true;
        if (pat_seg == null or path_seg == null) return false;
        
        const ps = pat_seg.?;
        const pp = path_seg.?;
        
        if (std.mem.startsWith(u8, ps, ":")) continue; // parameter segment
        if (!std.mem.eql(u8, ps, pp)) return false;
    }
}

pub const Router = struct {
    routes: std.ArrayList(Route),
    not_found_handler: ?RouteHandler = null,
    
    pub fn init(allocator: std.mem.Allocator) Router {
        return .{ .routes = std.ArrayList(Route).init(allocator) };
    }
    
    pub fn deinit(self: *Router) void {
        self.routes.deinit();
    }
    
    pub fn get(self: *Router, pattern: []const u8, handler: RouteHandler) !void {
        try self.routes.append(.{ .method = .GET, .pattern = pattern, .handler = handler });
    }
    
    pub fn post(self: *Router, pattern: []const u8, handler: RouteHandler) !void {
        try self.routes.append(.{ .method = .POST, .pattern = pattern, .handler = handler });
    }
    
    pub fn put(self: *Router, pattern: []const u8, handler: RouteHandler) !void {
        try self.routes.append(.{ .method = .PUT, .pattern = pattern, .handler = handler });
    }
    
    pub fn delete(self: *Router, pattern: []const u8, handler: RouteHandler) !void {
        try self.routes.append(.{ .method = .DELETE, .pattern = pattern, .handler = handler });
    }
    
    pub fn handle(
        self: *Router,
        req: *const HttpRequest,
        res: *HttpResponse,
        allocator: std.mem.Allocator,
    ) !void {
        for (self.routes.items) |route| {
            if (route.matches(req.method, req.path)) {
                try route.handler(req, res, allocator);
                return;
            }
        }
        
        if (self.not_found_handler) |handler| {
            try handler(req, res, allocator);
        } else {
            res.setStatus(404);
            try res.text("404 Not Found");
        }
    }
};

// ตัวอย่าง handlers
fn homeHandler(req: *const HttpRequest, res: *HttpResponse, _: std.mem.Allocator) !void {
    _ = req;
    try res.html("<h1>Welcome to Zig HTTP Server!</h1>");
}

fn apiHandler(req: *const HttpRequest, res: *HttpResponse, allocator: std.mem.Allocator) !void {
    const response = try std.fmt.allocPrint(
        allocator,
        "{{\"path\": \"{s}\", \"method\": \"{s}\"}}",
        .{ req.path, @tagName(req.method) },
    );
    defer allocator.free(response);
    try res.json(response);
}

// Types ที่จำเป็น
const HttpMethod = enum { GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS };
const HttpRequest = struct {
    method: HttpMethod,
    path: []const u8,
    query_string: ?[]const u8 = null,
    body: []const u8 = "",
};
const HttpResponse = struct {
    status_code: u16 = 200,
    body_data: std.ArrayList(u8),
    content_type: []const u8 = "text/plain",
    
    pub fn init(allocator: std.mem.Allocator) HttpResponse {
        return .{ .body_data = std.ArrayList(u8).init(allocator) };
    }
    pub fn deinit(self: *HttpResponse) void { self.body_data.deinit(); }
    pub fn setStatus(self: *HttpResponse, code: u16) void { self.status_code = code; }
    pub fn text(self: *HttpResponse, data: []const u8) !void {
        self.content_type = "text/plain";
        try self.body_data.appendSlice(data);
    }
    pub fn html(self: *HttpResponse, data: []const u8) !void {
        self.content_type = "text/html";
        try self.body_data.appendSlice(data);
    }
    pub fn json(self: *HttpResponse, data: []const u8) !void {
        self.content_type = "application/json";
        try self.body_data.appendSlice(data);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    var router = Router.init(allocator);
    defer router.deinit();
    
    try router.get("/", homeHandler);
    try router.get("/api/*", apiHandler);
    
    // ทดสอบ routing
    const requests = [_]HttpRequest{
        .{ .method = .GET, .path = "/" },
        .{ .method = .GET, .path = "/api/users" },
        .{ .method = .GET, .path = "/not-found" },
    };
    
    for (requests) |req| {
        var res = HttpResponse.init(allocator);
        defer res.deinit();
        
        try router.handle(&req, &res, allocator);
        std.debug.print("Route {s}: {d} bytes\n", .{ req.path, res.body_data.items.len });
    }
}
```

---

## Step 476: Middleware pattern

```zig
const std = @import("std");

// Middleware คือ function ที่รัน ก่อน/หลัง handler หลัก
pub const MiddlewareFn = *const fn (
    req: *Request,
    res: *Response,
    next: *const fn (*Request, *Response) anyerror!void,
) anyerror!void;

pub const Request = struct {
    method: []const u8,
    path: []const u8,
    headers: std.StringHashMap([]const u8),
    body: []const u8,
    context: std.StringHashMap([]const u8), // middleware data
    
    pub fn init(allocator: std.mem.Allocator) Request {
        return .{
            .method = "",
            .path = "",
            .headers = std.StringHashMap([]const u8).init(allocator),
            .body = "",
            .context = std.StringHashMap([]const u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *Request) void {
        self.headers.deinit();
        self.context.deinit();
    }
};

pub const Response = struct {
    status: u16 = 200,
    headers: std.StringHashMap([]const u8),
    body: std.ArrayList(u8),
    
    pub fn init(allocator: std.mem.Allocator) Response {
        return .{
            .headers = std.StringHashMap([]const u8).init(allocator),
            .body = std.ArrayList(u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *Response) void {
        self.headers.deinit();
        self.body.deinit();
    }
};

// Logging middleware
pub fn loggingMiddleware(
    req: *Request,
    res: *Response,
    next: *const fn (*Request, *Response) anyerror!void,
) anyerror!void {
    const start = std.time.nanoTimestamp();
    
    std.debug.print("[LOG] {} {s} {s}\n", .{
        std.time.timestamp(),
        req.method,
        req.path,
    });
    
    try next(req, res);
    
    const elapsed = std.time.nanoTimestamp() - start;
    std.debug.print("[LOG] Response: {d} ({d}µs)\n", .{
        res.status,
        @divTrunc(elapsed, 1000),
    });
}

// CORS middleware
pub fn corsMiddleware(
    req: *Request,
    res: *Response,
    next: *const fn (*Request, *Response) anyerror!void,
) anyerror!void {
    try res.headers.put("Access-Control-Allow-Origin", "*");
    try res.headers.put("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
    try res.headers.put("Access-Control-Allow-Headers", "Content-Type, Authorization");
    
    if (std.mem.eql(u8, req.method, "OPTIONS")) {
        res.status = 204;
        return;
    }
    
    try next(req, res);
}

// Auth middleware
pub fn authMiddleware(
    req: *Request,
    res: *Response,
    next: *const fn (*Request, *Response) anyerror!void,
) anyerror!void {
    const auth = req.headers.get("authorization") orelse {
        res.status = 401;
        try res.body.appendSlice("{\"error\": \"Unauthorized\"}");
        return;
    };
    
    if (!std.mem.startsWith(u8, auth, "Bearer ")) {
        res.status = 401;
        try res.body.appendSlice("{\"error\": \"Invalid token format\"}");
        return;
    }
    
    const token = auth[7..];
    // ในความเป็นจริงต้อง validate token
    try req.context.put("user_token", token);
    
    try next(req, res);
}

// Rate limiting middleware (ง่ายๆ)
var request_count = std.atomic.Value(u32).init(0);
const MAX_REQUESTS_PER_SECOND = 100;

pub fn rateLimitMiddleware(
    req: *Request,
    res: *Response,
    next: *const fn (*Request, *Response) anyerror!void,
) anyerror!void {
    const count = request_count.fetchAdd(1, .monotonic);
    
    if (count >= MAX_REQUESTS_PER_SECOND) {
        res.status = 429;
        try res.body.appendSlice("{\"error\": \"Too Many Requests\"}");
        _ = req;
        return;
    }
    
    try next(req, res);
}
```

---

## Step 477: Serving static files

```zig
const std = @import("std");

pub const StaticFileServer = struct {
    root_dir: []const u8,
    allocator: std.mem.Allocator,
    
    const MIME_TYPES = std.StaticStringMap([]const u8).initComptime(.{
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
        .{ ".wasm", "application/wasm" },
    });
    
    pub fn init(allocator: std.mem.Allocator, root_dir: []const u8) StaticFileServer {
        return .{ .root_dir = root_dir, .allocator = allocator };
    }
    
    pub fn serveFile(self: *StaticFileServer, path: []const u8, writer: anytype) !void {
        // Sanitize path - ป้องกัน path traversal
        if (std.mem.indexOf(u8, path, "..") != null) {
            try self.writeError(writer, 403, "Forbidden");
            return;
        }
        
        // สร้าง full path
        const full_path = try std.fs.path.join(self.allocator, &.{
            self.root_dir,
            path,
        });
        defer self.allocator.free(full_path);
        
        // ลองเปิดเป็น directory ก่อน
        var dir = std.fs.openDirAbsolute(full_path, .{}) catch null;
        if (dir) |*d| {
            defer d.close();
            // serve index.html
            const index_path = try std.fs.path.join(self.allocator, &.{ full_path, "index.html" });
            defer self.allocator.free(index_path);
            return self.serveActualFile(index_path, writer);
        }
        
        try self.serveActualFile(full_path, writer);
    }
    
    fn serveActualFile(self: *StaticFileServer, path: []const u8, writer: anytype) !void {
        const file = std.fs.openFileAbsolute(path, .{}) catch |err| {
            if (err == error.FileNotFound) {
                try self.writeError(writer, 404, "Not Found");
            } else {
                try self.writeError(writer, 500, "Internal Server Error");
            }
            return;
        };
        defer file.close();
        
        const stat = try file.stat();
        const content_type = getMimeType(path);
        
        // HTTP headers
        try writer.print(
            "HTTP/1.1 200 OK\r\nContent-Type: {s}\r\nContent-Length: {d}\r\nCache-Control: max-age=3600\r\n\r\n",
            .{ content_type, stat.size },
        );
        
        // Stream file content
        var buf: [8192]u8 = undefined;
        while (true) {
            const n = try file.read(&buf);
            if (n == 0) break;
            try writer.writeAll(buf[0..n]);
        }
    }
    
    fn writeError(_: *StaticFileServer, writer: anytype, code: u16, message: []const u8) !void {
        try writer.print(
            "HTTP/1.1 {d} {s}\r\nContent-Type: text/plain\r\nContent-Length: {d}\r\n\r\n{s}",
            .{ code, message, message.len, message },
        );
    }
    
    fn getMimeType(path: []const u8) []const u8 {
        const ext = std.fs.path.extension(path);
        return MIME_TYPES.get(ext) orelse "application/octet-stream";
    }
};
```

---

## Step 478: JSON API endpoints

```zig
const std = @import("std");

// Simple JSON builder
pub const JsonBuilder = struct {
    buffer: std.ArrayList(u8),
    
    pub fn init(allocator: std.mem.Allocator) JsonBuilder {
        return .{ .buffer = std.ArrayList(u8).init(allocator) };
    }
    
    pub fn deinit(self: *JsonBuilder) void {
        self.buffer.deinit();
    }
    
    pub fn object(self: *JsonBuilder) !void {
        try self.buffer.append('{');
    }
    
    pub fn endObject(self: *JsonBuilder) !void {
        // ลบ comma สุดท้ายถ้ามี
        if (self.buffer.items.len > 0 and self.buffer.items[self.buffer.items.len - 1] == ',') {
            _ = self.buffer.pop();
        }
        try self.buffer.append('}');
    }
    
    pub fn field(self: *JsonBuilder, key: []const u8, value: anytype) !void {
        try self.buffer.writer().print("\"{s}\":", .{key});
        try self.writeValue(value);
        try self.buffer.append(',');
    }
    
    fn writeValue(self: *JsonBuilder, value: anytype) !void {
        const T = @TypeOf(value);
        switch (@typeInfo(T)) {
            .Int, .ComptimeInt => try self.buffer.writer().print("{d}", .{value}),
            .Float, .ComptimeFloat => try self.buffer.writer().print("{d}", .{value}),
            .Bool => try self.buffer.appendSlice(if (value) "true" else "false"),
            .Pointer => |ptr| {
                if (ptr.child == u8) {
                    try self.buffer.writer().print("\"{s}\"", .{value});
                }
            },
            .Optional => {
                if (value) |v| {
                    try self.writeValue(v);
                } else {
                    try self.buffer.appendSlice("null");
                }
            },
            else => try self.buffer.appendSlice("null"),
        }
    }
    
    pub fn getString(self: *JsonBuilder) []const u8 {
        return self.buffer.items;
    }
};

// User API example
const User = struct {
    id: u32,
    name: []const u8,
    email: []const u8,
    active: bool,
};

const users_data = [_]User{
    .{ .id = 1, .name = "สมชาย", .email = "somchai@example.com", .active = true },
    .{ .id = 2, .name = "สมหญิง", .email = "somying@example.com", .active = true },
    .{ .id = 3, .name = "วิชัย", .email = "wichai@example.com", .active = false },
};

pub fn serveUsers(allocator: std.mem.Allocator) ![]u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    
    try buf.appendSlice("[");
    for (users_data, 0..) |user, i| {
        if (i > 0) try buf.appendSlice(",");
        
        const user_json = try std.fmt.allocPrint(
            allocator,
            "{{\"id\":{d},\"name\":\"{s}\",\"email\":\"{s}\",\"active\":{s}}}",
            .{
                user.id,
                user.name,
                user.email,
                if (user.active) "true" else "false",
            },
        );
        defer allocator.free(user_json);
        try buf.appendSlice(user_json);
    }
    try buf.appendSlice("]");
    
    return buf.toOwnedSlice();
}

pub fn serveUser(allocator: std.mem.Allocator, id: u32) !?[]u8 {
    for (users_data) |user| {
        if (user.id == id) {
            return try std.fmt.allocPrint(
                allocator,
                "{{\"id\":{d},\"name\":\"{s}\",\"email\":\"{s}\",\"active\":{s}}}",
                .{
                    user.id,
                    user.name,
                    user.email,
                    if (user.active) "true" else "false",
                },
            );
        }
    }
    return null;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const users_json = try serveUsers(allocator);
    defer allocator.free(users_json);
    std.debug.print("Users: {s}\n", .{users_json});
    
    if (try serveUser(allocator, 1)) |user_json| {
        defer allocator.free(user_json);
        std.debug.print("User 1: {s}\n", .{user_json});
    }
}
```

---

## Step 479: Error handling ใน HTTP server

```zig
const std = @import("std");

pub const HttpError = error{
    BadRequest,
    Unauthorized,
    Forbidden,
    NotFound,
    MethodNotAllowed,
    RequestTooLarge,
    InternalServerError,
    ServiceUnavailable,
};

pub fn httpErrorCode(err: HttpError) u16 {
    return switch (err) {
        HttpError.BadRequest => 400,
        HttpError.Unauthorized => 401,
        HttpError.Forbidden => 403,
        HttpError.NotFound => 404,
        HttpError.MethodNotAllowed => 405,
        HttpError.RequestTooLarge => 413,
        HttpError.InternalServerError => 500,
        HttpError.ServiceUnavailable => 503,
    };
}

pub fn httpErrorMessage(err: HttpError) []const u8 {
    return switch (err) {
        HttpError.BadRequest => "Bad Request",
        HttpError.Unauthorized => "Unauthorized",
        HttpError.Forbidden => "Forbidden",
        HttpError.NotFound => "Not Found",
        HttpError.MethodNotAllowed => "Method Not Allowed",
        HttpError.RequestTooLarge => "Request Too Large",
        HttpError.InternalServerError => "Internal Server Error",
        HttpError.ServiceUnavailable => "Service Unavailable",
    };
}

pub fn sendErrorResponse(writer: anytype, err: HttpError) !void {
    const code = httpErrorCode(err);
    const message = httpErrorMessage(err);
    
    const body = try std.fmt.allocPrint(
        std.heap.page_allocator,
        "{{\"error\": \"{s}\", \"code\": {d}}}",
        .{ message, code },
    );
    defer std.heap.page_allocator.free(body);
    
    try writer.print(
        "HTTP/1.1 {d} {s}\r\n" ++
        "Content-Type: application/json\r\n" ++
        "Content-Length: {d}\r\n\r\n{s}",
        .{ code, message, body.len, body },
    );
}

// Error recovery wrapper
pub fn withErrorHandling(
    handler: *const fn () anyerror!void,
    writer: anytype,
) void {
    handler() catch |err| {
        if (err == HttpError.NotFound) {
            sendErrorResponse(writer, HttpError.NotFound) catch {};
        } else if (err == HttpError.Unauthorized) {
            sendErrorResponse(writer, HttpError.Unauthorized) catch {};
        } else {
            std.debug.print("Unhandled error: {}\n", .{err});
            sendErrorResponse(writer, HttpError.InternalServerError) catch {};
        }
    };
}
```

---

## Step 480: Production HTTP server considerations

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;

// Production-ready HTTP server configuration
pub const ServerConfig = struct {
    host: []const u8 = "0.0.0.0",
    port: u16 = 8080,
    max_connections: u32 = 1000,
    read_timeout_ms: u64 = 30000,  // 30 seconds
    write_timeout_ms: u64 = 30000,
    max_request_size: usize = 10 * 1024 * 1024, // 10MB
    keep_alive: bool = true,
    keep_alive_timeout_s: u64 = 65,
    num_workers: u32 = 0,  // 0 = auto (CPU count)
    
    pub fn getWorkerCount(self: ServerConfig) !u32 {
        if (self.num_workers > 0) return self.num_workers;
        return @intCast(try Thread.getCpuCount());
    }
};

// Server metrics
pub const ServerMetrics = struct {
    total_requests: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    active_connections: std.atomic.Value(u32) = std.atomic.Value(u32).init(0),
    errors: std.atomic.Value(u64) = std.atomic.Value(u64).init(0),
    
    pub fn recordRequest(self: *ServerMetrics) void {
        _ = self.total_requests.fetchAdd(1, .monotonic);
    }
    
    pub fn recordConnection(self: *ServerMetrics) void {
        _ = self.active_connections.fetchAdd(1, .monotonic);
    }
    
    pub fn releaseConnection(self: *ServerMetrics) void {
        _ = self.active_connections.fetchSub(1, .monotonic);
    }
    
    pub fn recordError(self: *ServerMetrics) void {
        _ = self.errors.fetchAdd(1, .monotonic);
    }
    
    pub fn print(self: *const ServerMetrics) void {
        std.debug.print(
            "Metrics: requests={d}, active={d}, errors={d}\n",
            .{
                self.total_requests.load(.acquire),
                self.active_connections.load(.acquire),
                self.errors.load(.acquire),
            },
        );
    }
};

pub var global_metrics = ServerMetrics{};

pub fn main() !void {
    const config = ServerConfig{
        .port = 8080,
        .max_connections = 500,
        .keep_alive = true,
    };
    
    const worker_count = try config.getWorkerCount();
    std.debug.print("Starting server with {d} workers on :{d}\n", .{
        worker_count, config.port,
    });
    
    // แสดง metrics ทุก 5 วินาที
    const metrics_thread = try Thread.spawn(.{}, printMetricsLoop, .{});
    metrics_thread.detach();
    
    // Server loop (simplified)
    const addr = try net.Address.parseIp4(config.host, config.port);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    while (true) {
        const conn = try server.accept();
        global_metrics.recordConnection();
        
        const t = Thread.spawn(.{}, handleProdConnection, .{conn}) catch |err| {
            std.debug.print("Thread error: {}\n", .{err});
            global_metrics.recordError();
            global_metrics.releaseConnection();
            conn.stream.close();
            continue;
        };
        t.detach();
    }
}

fn handleProdConnection(conn: anytype) void {
    defer {
        conn.stream.close();
        global_metrics.releaseConnection();
    }
    
    global_metrics.recordRequest();
    
    var buf: [8192]u8 = undefined;
    const n = conn.stream.read(&buf) catch {
        global_metrics.recordError();
        return;
    };
    
    if (n == 0) return;
    
    const response = "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK";
    conn.stream.writeAll(response) catch {
        global_metrics.recordError();
    };
}

fn printMetricsLoop() void {
    while (true) {
        std.time.sleep(5 * std.time.ns_per_s);
        global_metrics.print();
    }
}
```

---

## สรุป Part 48

ในบทนี้เราได้เรียนรู้:

| Topic | เนื้อหา |
|-------|---------|
| HTTP Protocol | Request/Response format |
| HTTP Server | สร้างจาก scratch |
| Request Parsing | Parse HTTP headers และ body |
| Response Building | สร้าง HTTP responses |
| Routing | จับคู่ URL กับ handlers |
| Middleware | Logging, CORS, Auth |
| Static Files | Serve files พร้อม MIME types |
| JSON API | สร้าง REST API |
| Error Handling | จัดการ HTTP errors |
| Production | Metrics, Config, Workers |

ใน Part ถัดไปเราจะเรียนรู้ TCP/UDP Programming ขั้นสูง!
