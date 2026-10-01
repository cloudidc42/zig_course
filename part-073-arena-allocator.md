# ส่วนที่ 73: Arena Allocator Pattern

## ภาพรวม

Arena Allocator เป็น pattern ที่ทรงพลังมากใน Zig ช่วยให้ allocate หน่วยความจำได้เร็ว และ free ทั้งหมดพร้อมกันในครั้งเดียว เหมาะสำหรับกรณีที่มี lifetime ชัดเจน เช่น HTTP request, game frame, หรือ parsing

---

## Step 721: Arena Allocator Concept

### Arena คืออะไร

```zig
const std = @import("std");

// Arena ทำงานแบบ "allocate ได้เรื่อยๆ, free ทีเดียวทั้งหมด"
// ไม่มี overhead ของการ track individual allocations

// Conceptual implementation (simplified)
const SimpleArena = struct {
    buffer: []u8,
    pos: usize,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, size: usize) !SimpleArena {
        return SimpleArena{
            .buffer = try allocator.alloc(u8, size),
            .pos = 0,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *SimpleArena) void {
        self.allocator.free(self.buffer);
    }

    pub fn alloc(self: *SimpleArena, comptime T: type, n: usize) ![]T {
        const alignment = @alignOf(T);
        const size = @sizeOf(T) * n;
        const aligned_pos = (self.pos + alignment - 1) & ~(alignment - 1);

        if (aligned_pos + size > self.buffer.len) {
            return error.OutOfMemory;
        }

        const result: []T = @alignCast(std.mem.bytesAsSlice(
            T, self.buffer[aligned_pos..aligned_pos + size]
        ));
        self.pos = aligned_pos + size;
        return result;
    }

    // Free ทุกอย่างพร้อมกัน - O(1)
    pub fn freeAll(self: *SimpleArena) void {
        self.pos = 0;
    }

    pub fn usedBytes(self: *const SimpleArena) usize {
        return self.pos;
    }
};

// เปรียบเทียบ: ถ้าใช้ GPA จะต้อง free แต่ละอัน
pub fn withGPA(allocator: std.mem.Allocator) !void {
    // ต้อง track ทุก allocation
    const strings = [_][]const u8{ "apple", "banana", "cherry", "date" };
    var processed = try allocator.alloc([]u8, strings.len);
    defer {
        for (processed) |s| allocator.free(s);
        allocator.free(processed);
    }

    for (strings, 0..) |s, i| {
        processed[i] = try allocator.dupe(u8, s);
    }

    std.debug.print("GPA: need to free {} items\n", .{processed.len});
}

// ด้วย Arena ง่ายกว่ามาก
pub fn withArena(allocator: std.mem.Allocator) !void {
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit(); // Free ทั้งหมดในครั้งเดียว

    const a = arena.allocator();
    const strings = [_][]const u8{ "apple", "banana", "cherry", "date" };

    var processed = try a.alloc([]u8, strings.len);
    for (strings, 0..) |s, i| {
        processed[i] = try a.dupe(u8, s);
    }

    std.debug.print("Arena: just deinit arena to free everything\n", .{});
    // ไม่ต้อง free แต่ละอัน!
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Simple arena example
    var arena = try SimpleArena.init(allocator, 4096);
    defer arena.deinit();

    const ints = try arena.alloc(i32, 100);
    for (ints, 0..) |*v, i| v.* = @intCast(i);

    const floats = try arena.alloc(f64, 50);
    for (floats, 0..) |*v, i| v.* = @as(f64, @floatFromInt(i)) * 0.1;

    std.debug.print("Used: {} bytes\n", .{arena.usedBytes()});

    arena.freeAll(); // Free ทั้งหมดใน O(1)
    std.debug.print("After reset: {} bytes\n", .{arena.usedBytes()});

    try withGPA(allocator);
    try withArena(allocator);
}
```

### ข้อดีและข้อเสียของ Arena

```zig
// ข้อดี Arena:
// 1. Allocation เร็วมาก - แค่ bump pointer
// 2. Deallocation เร็ว - free ทุกอย่างพร้อมกัน O(1)
// 3. ไม่มี fragmentation
// 4. Cache friendly - memory contiguous
// 5. โค้ดง่าย - ไม่ต้อง track individual lifetimes

// ข้อเสีย:
// 1. ไม่สามารถ free individual allocations
// 2. ต้องรู้ lifetime ล่วงหน้า
// 3. ถ้าใช้นานเกินไปจะ waste memory
```

---

## Step 722: ArenaAllocator Implementation

### Zig's Built-in ArenaAllocator

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // สร้าง ArenaAllocator
    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();

    // ขอ allocator จาก arena
    const alloc = arena.allocator();

    // Allocate ได้เรื่อยๆ
    const buf1 = try alloc.alloc(u8, 100);
    const buf2 = try alloc.alloc(u32, 50);
    const str = try alloc.dupe(u8, "Hello, Arena!");

    // ใช้งาน
    @memset(buf1, 0xAA);
    for (buf2, 0..) |*v, i| v.* = @intCast(i);
    std.debug.print("String: {s}\n", .{str});

    // Reset - free ทั้งหมดแต่ยังใช้ arena ต่อได้
    const reset_result = arena.reset(.retain_capacity);
    std.debug.print("Reset result: {}\n", .{reset_result});

    // Allocate ใหม่หลัง reset
    const buf3 = try alloc.alloc(u8, 200);
    @memset(buf3, 0xBB);
    std.debug.print("After reset, new buf3[0]: 0x{X}\n", .{buf3[0]});

    _ = buf1;
    _ = buf2;
}
```

### ArenaAllocator State Machine

```zig
const std = @import("std");

// Arena มี states ต่างๆ
const ArenaState = enum {
    empty,
    in_use,
    full,
};

const ManagedArena = struct {
    arena: std.heap.ArenaAllocator,
    state: ArenaState,
    used_bytes: usize,
    max_bytes: usize,

    pub fn init(backing: std.mem.Allocator, max_size: usize) ManagedArena {
        return ManagedArena{
            .arena = std.heap.ArenaAllocator.init(backing),
            .state = .empty,
            .used_bytes = 0,
            .max_bytes = max_size,
        };
    }

    pub fn deinit(self: *ManagedArena) void {
        self.arena.deinit();
    }

    pub fn allocator(self: *ManagedArena) std.mem.Allocator {
        return self.arena.allocator();
    }

    pub fn begin(self: *ManagedArena) void {
        self.state = .in_use;
        self.used_bytes = 0;
    }

    pub fn end(self: *ManagedArena) void {
        _ = self.arena.reset(.retain_capacity);
        self.state = .empty;
        self.used_bytes = 0;
    }

    pub fn isActive(self: *const ManagedArena) bool {
        return self.state == .in_use;
    }
};

pub fn processRequest(arena: *ManagedArena, request: []const u8) ![]u8 {
    arena.begin();
    defer arena.end();

    const alloc = arena.allocator();

    // Process request ใน arena scope
    const upper = try alloc.alloc(u8, request.len);
    for (request, 0..) |c, i| {
        upper[i] = std.ascii.toUpper(c);
    }

    const response = try std.fmt.allocPrint(alloc, "Response: {s}", .{upper});
    // Note: response จะ invalid หลัง arena.end()
    // ในกรณีจริงควร return ก่อน end()

    return response;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var managed = ManagedArena.init(gpa.allocator(), 1024 * 1024);
    defer managed.deinit();

    // จำลอง multiple requests
    const requests = [_][]const u8{
        "hello world",
        "foo bar baz",
        "zig programming",
    };

    for (requests) |req| {
        managed.begin();
        const alloc = managed.allocator();

        const processed = try std.fmt.allocPrint(alloc, "PROCESSED: {s}", .{req});
        std.debug.print("{s}\n", .{processed});

        managed.end();
    }
}
```

---

## Step 723: Arena for Request/Response

### HTTP Request Arena Pattern

```zig
const std = @import("std");

const HttpRequest = struct {
    method: []const u8,
    path: []const u8,
    headers: std.StringHashMap([]const u8),
    body: []const u8,
    arena: std.mem.Allocator,

    pub fn parse(arena: std.mem.Allocator, raw: []const u8) !HttpRequest {
        var req = HttpRequest{
            .method = "",
            .path = "",
            .headers = std.StringHashMap([]const u8).init(arena),
            .body = "",
            .arena = arena,
        };

        var lines = std.mem.splitScalar(u8, raw, '\n');

        // Parse request line
        if (lines.next()) |request_line| {
            const trimmed = std.mem.trimRight(u8, request_line, "\r");
            var parts = std.mem.splitScalar(u8, trimmed, ' ');
            req.method = try arena.dupe(u8, parts.next() orelse "");
            req.path = try arena.dupe(u8, parts.next() orelse "");
        }

        // Parse headers
        while (lines.next()) |line| {
            const trimmed = std.mem.trimRight(u8, line, "\r");
            if (trimmed.len == 0) break; // Empty line = end of headers

            if (std.mem.indexOf(u8, trimmed, ": ")) |colon_pos| {
                const key = try arena.dupe(u8, trimmed[0..colon_pos]);
                const value = try arena.dupe(u8, trimmed[colon_pos + 2..]);
                try req.headers.put(key, value);
            }
        }

        // Rest is body
        if (lines.rest().len > 0) {
            req.body = try arena.dupe(u8, lines.rest());
        }

        return req;
    }

    pub fn getHeader(self: *const HttpRequest, name: []const u8) ?[]const u8 {
        return self.headers.get(name);
    }
};

const HttpResponse = struct {
    status: u16,
    headers: std.StringHashMap([]const u8),
    body: []const u8,
    arena: std.mem.Allocator,

    pub fn init(arena: std.mem.Allocator, status: u16) HttpResponse {
        return HttpResponse{
            .status = status,
            .headers = std.StringHashMap([]const u8).init(arena),
            .body = "",
            .arena = arena,
        };
    }

    pub fn setHeader(self: *HttpResponse, name: []const u8, value: []const u8) !void {
        try self.headers.put(
            try self.arena.dupe(u8, name),
            try self.arena.dupe(u8, value),
        );
    }

    pub fn setBody(self: *HttpResponse, body: []const u8) !void {
        self.body = try self.arena.dupe(u8, body);
    }

    pub fn serialize(self: *const HttpResponse) ![]u8 {
        var buf = std.ArrayList(u8).init(self.arena);
        const w = buf.writer();

        try w.print("HTTP/1.1 {d}\r\n", .{self.status});

        var it = self.headers.iterator();
        while (it.next()) |entry| {
            try w.print("{s}: {s}\r\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        }
        try w.writeAll("\r\n");
        try w.writeAll(self.body);

        return buf.toOwnedSlice();
    }
};

const RequestHandler = struct {
    backing_allocator: std.mem.Allocator,

    pub fn handle(self: *RequestHandler, raw_request: []const u8) ![]u8 {
        // Arena ครอบคลุม request/response cycle ทั้งหมด
        var arena = std.heap.ArenaAllocator.init(self.backing_allocator);
        defer arena.deinit(); // Free ทุกอย่างหลัง request เสร็จ

        const alloc = arena.allocator();

        const request = try HttpRequest.parse(alloc, raw_request);
        var response = HttpResponse.init(alloc, 200);

        // Route
        if (std.mem.eql(u8, request.method, "GET")) {
            if (std.mem.eql(u8, request.path, "/hello")) {
                try response.setHeader("Content-Type", "text/plain");
                try response.setBody("Hello, World!");
            } else {
                response.status = 404;
                try response.setBody("Not Found");
            }
        } else {
            response.status = 405;
            try response.setBody("Method Not Allowed");
        }

        const content_length = try std.fmt.allocPrint(alloc, "{}", .{response.body.len});
        try response.setHeader("Content-Length", content_length);

        // Serialize สำหรับส่งออก - ต้อง copy ไปยัง backing allocator
        const serialized = try response.serialize();
        return self.backing_allocator.dupe(u8, serialized);
        // Arena จะ free serialized ตอน deinit
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var handler = RequestHandler{ .backing_allocator = gpa.allocator() };

    const raw_request =
        "GET /hello HTTP/1.1\r\n" ++
        "Host: localhost:8080\r\n" ++
        "User-Agent: zig-test\r\n" ++
        "\r\n";

    const response = try handler.handle(raw_request);
    defer gpa.allocator().free(response);

    std.debug.print("Response:\n{s}\n", .{response});
}
```

---

## Step 724: Arena for Parsing

### JSON Parser ด้วย Arena

```zig
const std = @import("std");

const JsonValueType = enum { null_val, bool_val, number_val, string_val, array_val, object_val };

const JsonValue = union(JsonValueType) {
    null_val: void,
    bool_val: bool,
    number_val: f64,
    string_val: []const u8,
    array_val: []JsonValue,
    object_val: []JsonField,
};

const JsonField = struct {
    key: []const u8,
    value: JsonValue,
};

const JsonParser = struct {
    source: []const u8,
    pos: usize,
    arena: std.mem.Allocator,

    pub fn init(arena: std.mem.Allocator, source: []const u8) JsonParser {
        return JsonParser{ .source = source, .pos = 0, .arena = arena };
    }

    fn skipWhitespace(self: *JsonParser) void {
        while (self.pos < self.source.len and
               (self.source[self.pos] == ' ' or
                self.source[self.pos] == '\t' or
                self.source[self.pos] == '\n' or
                self.source[self.pos] == '\r')) {
            self.pos += 1;
        }
    }

    fn peek(self: *JsonParser) ?u8 {
        if (self.pos >= self.source.len) return null;
        return self.source[self.pos];
    }

    fn consume(self: *JsonParser) ?u8 {
        if (self.pos >= self.source.len) return null;
        const c = self.source[self.pos];
        self.pos += 1;
        return c;
    }

    pub fn parseValue(self: *JsonParser) !JsonValue {
        self.skipWhitespace();
        const c = self.peek() orelse return error.UnexpectedEof;

        return switch (c) {
            'n' => self.parseNull(),
            't', 'f' => self.parseBool(),
            '"' => self.parseString(),
            '[' => self.parseArray(),
            '{' => self.parseObject(),
            '-', '0'...'9' => self.parseNumber(),
            else => error.InvalidJson,
        };
    }

    fn parseNull(self: *JsonParser) !JsonValue {
        if (std.mem.startsWith(u8, self.source[self.pos..], "null")) {
            self.pos += 4;
            return JsonValue{ .null_val = {} };
        }
        return error.InvalidJson;
    }

    fn parseBool(self: *JsonParser) !JsonValue {
        if (std.mem.startsWith(u8, self.source[self.pos..], "true")) {
            self.pos += 4;
            return JsonValue{ .bool_val = true };
        }
        if (std.mem.startsWith(u8, self.source[self.pos..], "false")) {
            self.pos += 5;
            return JsonValue{ .bool_val = false };
        }
        return error.InvalidJson;
    }

    fn parseString(self: *JsonParser) !JsonValue {
        _ = self.consume(); // '"'
        const start = self.pos;
        while (self.pos < self.source.len) {
            const c = self.source[self.pos];
            if (c == '"') {
                const str = try self.arena.dupe(u8, self.source[start..self.pos]);
                self.pos += 1;
                return JsonValue{ .string_val = str };
            }
            if (c == '\\') self.pos += 1; // Skip escaped char
            self.pos += 1;
        }
        return error.UnterminatedString;
    }

    fn parseNumber(self: *JsonParser) !JsonValue {
        const start = self.pos;
        if (self.source[self.pos] == '-') self.pos += 1;
        while (self.pos < self.source.len and
               (self.source[self.pos] >= '0' and self.source[self.pos] <= '9' or
                self.source[self.pos] == '.' or
                self.source[self.pos] == 'e' or
                self.source[self.pos] == 'E' or
                self.source[self.pos] == '+' or
                self.source[self.pos] == '-')) {
            self.pos += 1;
        }
        const num_str = self.source[start..self.pos];
        const num = try std.fmt.parseFloat(f64, num_str);
        return JsonValue{ .number_val = num };
    }

    fn parseArray(self: *JsonParser) !JsonValue {
        _ = self.consume(); // '['
        var items = std.ArrayList(JsonValue).init(self.arena);

        self.skipWhitespace();
        if (self.peek() == ']') {
            _ = self.consume();
            return JsonValue{ .array_val = try items.toOwnedSlice() };
        }

        while (true) {
            const item = try self.parseValue();
            try items.append(item);
            self.skipWhitespace();
            const c = self.consume() orelse return error.UnexpectedEof;
            if (c == ']') break;
            if (c != ',') return error.InvalidJson;
        }

        return JsonValue{ .array_val = try items.toOwnedSlice() };
    }

    fn parseObject(self: *JsonParser) !JsonValue {
        _ = self.consume(); // '{'
        var fields = std.ArrayList(JsonField).init(self.arena);

        self.skipWhitespace();
        if (self.peek() == '}') {
            _ = self.consume();
            return JsonValue{ .object_val = try fields.toOwnedSlice() };
        }

        while (true) {
            self.skipWhitespace();
            const key_val = try self.parseString();
            const key = key_val.string_val;

            self.skipWhitespace();
            if (self.consume() != ':') return error.InvalidJson;

            const value = try self.parseValue();
            try fields.append(.{ .key = key, .value = value });

            self.skipWhitespace();
            const c = self.consume() orelse return error.UnexpectedEof;
            if (c == '}') break;
            if (c != ',') return error.InvalidJson;
        }

        return JsonValue{ .object_val = try fields.toOwnedSlice() };
    }
};

fn printJsonValue(value: JsonValue, indent: usize) void {
    const spaces = "                                "[0..indent * 2];
    switch (value) {
        .null_val => std.debug.print("null", .{}),
        .bool_val => |b| std.debug.print("{}", .{b}),
        .number_val => |n| std.debug.print("{d}", .{n}),
        .string_val => |s| std.debug.print("\"{s}\"", .{s}),
        .array_val => |arr| {
            std.debug.print("[\n", .{});
            for (arr) |item| {
                std.debug.print("{s}  ", .{spaces});
                printJsonValue(item, indent + 1);
                std.debug.print(",\n", .{});
            }
            std.debug.print("{s}]", .{spaces});
        },
        .object_val => |fields| {
            std.debug.print("{{\n", .{});
            for (fields) |field| {
                std.debug.print("{s}  \"{s}\": ", .{ spaces, field.key });
                printJsonValue(field.value, indent + 1);
                std.debug.print(",\n", .{});
            }
            std.debug.print("{s}}}", .{spaces});
        },
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // Arena สำหรับ parsing - free ทั้งหมดหลังใช้
    var arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer arena.deinit();

    const json_src =
        \\{
        \\  "name": "Alice",
        \\  "age": 30,
        \\  "active": true,
        \\  "scores": [95.5, 87.0, 92.3]
        \\}
    ;

    var parser = JsonParser.init(arena.allocator(), json_src);
    const value = try parser.parseValue();

    printJsonValue(value, 0);
    std.debug.print("\n", .{});
    // Arena freed ทั้งหมด JSON tree ตอน deinit
}
```

---

## Step 725: Arena for Game Frames

### Game Frame Arena

```zig
const std = @import("std");

const Vec2 = struct { x: f32, y: f32 };

const Particle = struct {
    pos: Vec2,
    vel: Vec2,
    color: u32,
    lifetime: f32,
    size: f32,
};

const DrawCommand = union(enum) {
    rect: struct { x: f32, y: f32, w: f32, h: f32, color: u32 },
    circle: struct { x: f32, y: f32, r: f32, color: u32 },
    text: struct { x: f32, y: f32, text: []const u8, color: u32 },
    particle: Particle,
};

const FrameData = struct {
    particles: []Particle,
    draw_commands: []DrawCommand,
    frame_number: u64,
    delta_time: f32,
};

const GameFrameArena = struct {
    arenas: [2]std.heap.ArenaAllocator, // Double buffering
    current: u8,
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator) GameFrameArena {
        return GameFrameArena{
            .arenas = .{
                std.heap.ArenaAllocator.init(backing),
                std.heap.ArenaAllocator.init(backing),
            },
            .current = 0,
            .backing = backing,
        };
    }

    pub fn deinit(self: *GameFrameArena) void {
        self.arenas[0].deinit();
        self.arenas[1].deinit();
    }

    // สลับ arena และ reset อันที่ไม่ได้ใช้
    pub fn nextFrame(self: *GameFrameArena) std.mem.Allocator {
        const prev = self.current;
        self.current = 1 - self.current;
        _ = self.arenas[prev].reset(.retain_capacity);
        return self.arenas[self.current].allocator();
    }

    pub fn currentAllocator(self: *GameFrameArena) std.mem.Allocator {
        return self.arenas[self.current].allocator();
    }
};

const GameState = struct {
    particles: std.ArrayList(Particle),
    score: i32,
    frame: u64,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) GameState {
        return GameState{
            .particles = std.ArrayList(Particle).init(allocator),
            .score = 0,
            .frame = 0,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *GameState) void {
        self.particles.deinit();
    }

    pub fn spawnParticle(self: *GameState, x: f32, y: f32) !void {
        try self.particles.append(Particle{
            .pos = .{ .x = x, .y = y },
            .vel = .{ .x = 1.0, .y = -2.0 },
            .color = 0xFF0000FF,
            .lifetime = 2.0,
            .size = 5.0,
        });
    }

    pub fn update(self: *GameState, dt: f32) void {
        self.frame += 1;
        // Update particles
        var i: usize = 0;
        while (i < self.particles.items.len) {
            const p = &self.particles.items[i];
            p.pos.x += p.vel.x * dt;
            p.pos.y += p.vel.y * dt;
            p.lifetime -= dt;
            if (p.lifetime <= 0) {
                _ = self.particles.swapRemove(i);
            } else {
                i += 1;
            }
        }
    }

    // สร้าง frame data ใน arena (ชั่วคราว)
    pub fn buildFrameData(
        self: *const GameState,
        frame_arena: std.mem.Allocator,
        dt: f32,
    ) !*FrameData {
        const frame = try frame_arena.create(FrameData);
        frame.frame_number = self.frame;
        frame.delta_time = dt;

        // Copy particles to frame data
        frame.particles = try frame_arena.dupe(Particle, self.particles.items);

        // Build draw commands
        var cmds = std.ArrayList(DrawCommand).init(frame_arena);

        // Background
        try cmds.append(.{ .rect = .{ .x = 0, .y = 0, .w = 800, .h = 600, .color = 0x000000FF } });

        // Particles
        for (self.particles.items) |p| {
            try cmds.append(.{ .particle = p });
        }

        // HUD
        const score_text = try std.fmt.allocPrint(frame_arena, "Score: {}", .{self.score});
        try cmds.append(.{ .text = .{ .x = 10, .y = 10, .text = score_text, .color = 0xFFFFFFFF } });

        frame.draw_commands = try cmds.toOwnedSlice();
        return frame;
    }
};

fn renderFrame(frame: *const FrameData) void {
    std.debug.print("Frame #{}: {} particles, {} draw commands\n", .{
        frame.frame_number,
        frame.particles.len,
        frame.draw_commands.len,
    });
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var game = GameState.init(gpa.allocator());
    defer game.deinit();

    var frame_arena = GameFrameArena.init(gpa.allocator());
    defer frame_arena.deinit();

    // Simulate 5 frames
    var f: usize = 0;
    while (f < 5) : (f += 1) {
        const dt: f32 = 1.0 / 60.0;

        // Spawn some particles every frame
        if (f % 2 == 0) {
            try game.spawnParticle(400.0 + @as(f32, @floatFromInt(f)) * 10.0, 300.0);
        }

        game.update(dt);

        // Get frame arena (resets previous frame)
        const frame_alloc = frame_arena.nextFrame();

        // Build frame data ใน arena
        const frame_data = try game.buildFrameData(frame_alloc, dt);
        renderFrame(frame_data);

        // frame_data valid เฉพาะ frame นี้
        // จะถูก free อัตโนมัติตอน nextFrame() ครั้งถัดไป
    }

    std.debug.print("Simulation complete\n", .{});
}
```

---

## Step 726: Arena with Scratch Buffer

### Arena + Scratch Buffer Pattern

```zig
const std = @import("std");

// Scratch buffer สำหรับ temporary allocations ระหว่าง operation
const ScratchBuffer = struct {
    backing_arena: *std.heap.ArenaAllocator,
    checkpoint: std.heap.ArenaAllocator.State,

    pub fn begin(arena: *std.heap.ArenaAllocator) ScratchBuffer {
        return ScratchBuffer{
            .backing_arena = arena,
            .checkpoint = arena.state,
        };
    }

    pub fn end(self: *ScratchBuffer) void {
        self.backing_arena.state = self.checkpoint;
    }

    pub fn allocator(self: *ScratchBuffer) std.mem.Allocator {
        return self.backing_arena.allocator();
    }
};

pub fn complexOperation(arena: *std.heap.ArenaAllocator, input: []const u8) ![]u8 {
    // ใช้ scratch สำหรับ temp data
    var scratch = ScratchBuffer.begin(arena);
    defer scratch.end();

    const temp_alloc = scratch.allocator();

    // Temporary processing
    const temp1 = try temp_alloc.dupe(u8, input);
    for (temp1) |*c| c.* = std.ascii.toUpper(c.*);

    const temp2 = try std.fmt.allocPrint(temp_alloc, "PROCESSED: {s}", .{temp1});

    // Result ต้อง allocate ใน arena หลักไม่ใช่ scratch
    const result = try arena.allocator().dupe(u8, temp2);

    // temp1 และ temp2 จะถูก free ตอน scratch.end()
    return result;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var main_arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer main_arena.deinit();

    const inputs = [_][]const u8{ "hello", "world", "zig" };

    var results = std.ArrayList([]u8).init(main_arena.allocator());
    for (inputs) |input| {
        const result = try complexOperation(&main_arena, input);
        try results.append(result);
    }

    for (results.items) |r| {
        std.debug.print("{s}\n", .{r});
    }
}
```

---

## Step 727: Nested Arenas

### Nested Arena Pattern

```zig
const std = @import("std");

const Task = struct {
    id: u32,
    name: []const u8,
    subtasks: []SubTask,
};

const SubTask = struct {
    id: u32,
    name: []const u8,
    result: []const u8,
};

fn processSubTask(arena: std.mem.Allocator, id: u32, name: []const u8) !SubTask {
    // Subtask ใช้ nested arena
    var sub_arena = std.heap.ArenaAllocator.init(arena);
    defer sub_arena.deinit();
    const sub_alloc = sub_arena.allocator();

    // Temp processing
    const temp = try std.fmt.allocPrint(sub_alloc, "Processing {s}...", .{name});
    std.debug.print("  {s}\n", .{temp});

    // Result ใน parent arena
    const result = try std.fmt.allocPrint(arena, "Result of {s}", .{name});
    const owned_name = try arena.dupe(u8, name);

    return SubTask{ .id = id, .name = owned_name, .result = result };
}

fn processTask(parent_arena: std.mem.Allocator, task_id: u32) !Task {
    // Task ใช้ nested arena
    var task_arena = std.heap.ArenaAllocator.init(parent_arena);
    // ไม่ต้อง deinit - parent arena จัดการให้

    const alloc = task_arena.allocator();
    const name = try std.fmt.allocPrint(alloc, "Task_{}", .{task_id});

    const subtask_names = [_][]const u8{ "init", "process", "cleanup" };
    var subtasks = try alloc.alloc(SubTask, subtask_names.len);

    std.debug.print("Processing {s}:\n", .{name});
    for (subtask_names, 0..) |sn, i| {
        subtasks[i] = try processSubTask(alloc, @intCast(i), sn);
    }

    return Task{
        .id = task_id,
        .name = name,
        .subtasks = subtasks,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var root_arena = std.heap.ArenaAllocator.init(gpa.allocator());
    defer root_arena.deinit(); // Free ทุกอย่าง including nested arenas

    const alloc = root_arena.allocator();
    var tasks = try alloc.alloc(Task, 3);

    for (0..3) |i| {
        tasks[i] = try processTask(alloc, @intCast(i + 1));
    }

    std.debug.print("\nAll tasks complete:\n", .{});
    for (tasks) |task| {
        std.debug.print("  Task {}: {} subtasks\n", .{ task.id, task.subtasks.len });
    }
}
```

---

## Step 728: Arena Capacity Management

### Managing Arena Memory

```zig
const std = @import("std");

const ArenaStats = struct {
    total_bytes: usize,
    used_bytes: usize,
    num_allocations: usize,
    num_resets: usize,
};

const InstrumentedArena = struct {
    inner: std.heap.ArenaAllocator,
    stats: ArenaStats,
    high_watermark: usize,

    pub fn init(backing: std.mem.Allocator) InstrumentedArena {
        return InstrumentedArena{
            .inner = std.heap.ArenaAllocator.init(backing),
            .stats = .{
                .total_bytes = 0,
                .used_bytes = 0,
                .num_allocations = 0,
                .num_resets = 0,
            },
            .high_watermark = 0,
        };
    }

    pub fn deinit(self: *InstrumentedArena) void {
        self.inner.deinit();
    }

    pub fn allocator(self: *InstrumentedArena) std.mem.Allocator {
        // Wrap inner allocator สำหรับ tracking
        return self.inner.allocator();
    }

    pub fn reset(self: *InstrumentedArena, mode: std.heap.ArenaAllocator.ResetMode) bool {
        self.stats.num_resets += 1;
        if (self.stats.used_bytes > self.high_watermark) {
            self.high_watermark = self.stats.used_bytes;
        }
        self.stats.used_bytes = 0;
        self.stats.num_allocations = 0;
        return self.inner.reset(mode);
    }

    pub fn printStats(self: *const InstrumentedArena) void {
        std.debug.print(
            \\Arena Stats:
            \\  High watermark: {} bytes
            \\  Current: {} bytes
            \\  Allocations: {}
            \\  Resets: {}
            \\
        , .{
            self.high_watermark,
            self.stats.used_bytes,
            self.stats.num_allocations,
            self.stats.num_resets,
        });
    }
};

// Growable arena ที่รองรับ larger allocations
const GrowableArena = struct {
    chunks: std.ArrayList([]u8),
    current_chunk: []u8,
    chunk_pos: usize,
    chunk_size: usize,
    backing: std.mem.Allocator,

    pub fn init(backing: std.mem.Allocator, initial_chunk_size: usize) !GrowableArena {
        var self = GrowableArena{
            .chunks = std.ArrayList([]u8).init(backing),
            .current_chunk = &[_]u8{},
            .chunk_pos = 0,
            .chunk_size = initial_chunk_size,
            .backing = backing,
        };
        try self.growChunk();
        return self;
    }

    pub fn deinit(self: *GrowableArena) void {
        for (self.chunks.items) |chunk| {
            self.backing.free(chunk);
        }
        self.chunks.deinit();
    }

    fn growChunk(self: *GrowableArena) !void {
        const chunk = try self.backing.alloc(u8, self.chunk_size);
        try self.chunks.append(chunk);
        self.current_chunk = chunk;
        self.chunk_pos = 0;
        self.chunk_size *= 2; // Double next chunk size
    }

    pub fn alloc(self: *GrowableArena, size: usize) ![]u8 {
        if (self.chunk_pos + size > self.current_chunk.len) {
            try self.growChunk();
            if (size > self.current_chunk.len) {
                return error.AllocationTooLarge;
            }
        }

        const ptr = self.current_chunk[self.chunk_pos..self.chunk_pos + size];
        self.chunk_pos += size;
        return ptr;
    }

    pub fn reset(self: *GrowableArena) void {
        // Free all but first chunk
        while (self.chunks.items.len > 1) {
            const last = self.chunks.pop();
            self.backing.free(last);
        }
        if (self.chunks.items.len > 0) {
            self.current_chunk = self.chunks.items[0];
            self.chunk_pos = 0;
        }
    }

    pub fn totalBytes(self: *const GrowableArena) usize {
        var total: usize = 0;
        for (self.chunks.items) |chunk| total += chunk.len;
        return total;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    // Growable arena
    var g_arena = try GrowableArena.init(gpa.allocator(), 256);
    defer g_arena.deinit();

    std.debug.print("Initial capacity: {} bytes\n", .{g_arena.totalBytes()});

    // Allocate ไปเรื่อยๆ จนต้อง grow
    for (0..10) |i| {
        const buf = try g_arena.alloc(100);
        @memset(buf, @intCast(i));
        std.debug.print("Chunk count: {}, total: {} bytes\n", .{
            g_arena.chunks.items.len, g_arena.totalBytes()
        });
    }

    g_arena.reset();
    std.debug.print("After reset: {} bytes, {} chunks\n", .{
        g_arena.totalBytes(), g_arena.chunks.items.len
    });
}
```

---

## Step 729: Arena Performance Analysis

### Performance Comparison

```zig
const std = @import("std");
const time = std.time;

fn benchGPA(allocator: std.mem.Allocator, n: usize) !u64 {
    const start = time.nanoTimestamp();

    var ptrs = try allocator.alloc(?[]u8, n);
    defer allocator.free(ptrs);
    @memset(ptrs, null);

    for (0..n) |i| {
        ptrs[i] = try allocator.alloc(u8, 64 + (i % 128));
        @memset(ptrs[i].?, @intCast(i % 256));
    }

    for (ptrs) |p| {
        if (p) |ptr| allocator.free(ptr);
    }

    return @intCast(time.nanoTimestamp() - start);
}

fn benchArena(backing: std.mem.Allocator, n: usize) !u64 {
    const start = time.nanoTimestamp();

    var arena = std.heap.ArenaAllocator.init(backing);
    defer arena.deinit();
    const alloc = arena.allocator();

    for (0..n) |i| {
        const buf = try alloc.alloc(u8, 64 + (i % 128));
        @memset(buf, @intCast(i % 256));
    }
    // Free ทั้งหมดพร้อมกัน

    return @intCast(time.nanoTimestamp() - start);
}

fn benchArenaWithReset(backing: std.mem.Allocator, n: usize, batch_size: usize) !u64 {
    const start = time.nanoTimestamp();

    var arena = std.heap.ArenaAllocator.init(backing);
    defer arena.deinit();
    const alloc = arena.allocator();

    var i: usize = 0;
    while (i < n) : (i += batch_size) {
        const end = @min(i + batch_size, n);
        for (i..end) |j| {
            const buf = try alloc.alloc(u8, 64 + (j % 128));
            @memset(buf, @intCast(j % 256));
        }
        _ = arena.reset(.retain_capacity);
    }

    return @intCast(time.nanoTimestamp() - start);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const N: usize = 10000;

    std.debug.print("=== Arena Performance Analysis (N={}) ===\n\n", .{N});

    const gpa_time = try benchGPA(allocator, N);
    std.debug.print("GPA (alloc+free each): {} ns ({} ns/op)\n", .{
        gpa_time, gpa_time / N
    });

    const arena_time = try benchArena(allocator, N);
    std.debug.print("Arena (batch free):    {} ns ({} ns/op)\n", .{
        arena_time, arena_time / N
    });

    const reset_time = try benchArenaWithReset(allocator, N, 100);
    std.debug.print("Arena (reset/100):     {} ns ({} ns/op)\n", .{
        reset_time, reset_time / N
    });

    std.debug.print("\nSpeedup (arena vs GPA): {d:.2}x\n", .{
        @as(f64, @floatFromInt(gpa_time)) / @as(f64, @floatFromInt(arena_time))
    });
}
```

---

## Step 730: When to Use ArenaAllocator

### เมื่อไหรควรใช้ Arena

```zig
const std = @import("std");

// Use Case 1: HTTP Server Request Handling
const ServerContext = struct {
    pub fn handleRequest(
        backing: std.mem.Allocator,
        request_data: []const u8,
    ) ![]u8 {
        var request_arena = std.heap.ArenaAllocator.init(backing);
        defer request_arena.deinit();

        const alloc = request_arena.allocator();
        _ = request_data;

        // Parse, process, generate response - ทุกอย่างใน arena
        const response = try alloc.dupe(u8, "HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nHello");
        const to_send = try backing.dupe(u8, response); // Copy ออกก่อน arena deinit
        return to_send;
    }
};

// Use Case 2: Compiler/Parser
const CompileContext = struct {
    source: []const u8,

    pub fn compile(backing: std.mem.Allocator, source: []const u8) !void {
        // แต่ละ phase ใช้ arena ของตัวเอง
        var parse_arena = std.heap.ArenaAllocator.init(backing);
        defer parse_arena.deinit();

        var sema_arena = std.heap.ArenaAllocator.init(backing);
        defer sema_arena.deinit();

        const parse_alloc = parse_arena.allocator();
        const sema_alloc = sema_arena.allocator();

        // Parse phase
        const ast_nodes = try parse_alloc.alloc(u8, source.len);
        @memcpy(ast_nodes, source);
        std.debug.print("Parsed {} bytes\n", .{ast_nodes.len});

        // Semantic analysis phase
        const ir_nodes = try sema_alloc.alloc(u8, source.len * 2);
        @memset(ir_nodes, 0);
        std.debug.print("Generated {} IR bytes\n", .{ir_nodes.len});

        // parse_arena freed, sema_arena freed
    }
};

// Use Case 3: One-time initialization
pub fn loadConfig(backing: std.mem.Allocator, config_path: []const u8) ![]u8 {
    var temp_arena = std.heap.ArenaAllocator.init(backing);
    defer temp_arena.deinit();

    const temp_alloc = temp_arena.allocator();
    _ = config_path;

    // Parse config ใน temp arena
    const raw_config = try temp_alloc.dupe(u8, "{\"key\": \"value\"}");
    const parsed = try std.fmt.allocPrint(temp_alloc, "config={s}", .{raw_config});

    // Return permanent copy
    return backing.dupe(u8, parsed);
}

// ไม่ควรใช้ Arena เมื่อ:
// 1. Objects มี lifetime ที่แตกต่างกันมาก
// 2. ต้องการ free บางส่วนก่อน
// 3. Object มีอายุนานกว่า scope ที่ชัดเจน

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Request handling
    const response = try ServerContext.handleRequest(allocator, "GET /hello HTTP/1.1");
    defer allocator.free(response);
    std.debug.print("Server response: {s}\n", .{response});

    // Compilation
    try CompileContext.compile(allocator, "fn main() void {}");

    // Config loading
    const config = try loadConfig(allocator, "config.json");
    defer allocator.free(config);
    std.debug.print("Config: {s}\n", .{config});

    std.debug.print(
        \\
        \\=== When to Use ArenaAllocator ===
        \\
        \\USE when:
        \\  - Request/response cycle (HTTP, RPC)
        \\  - Game frame processing
        \\  - Parsing/compilation phases
        \\  - Temporary string formatting
        \\  - Tree building (AST, scene graph)
        \\  - Batch processing with clear boundaries
        \\
        \\DON'T USE when:
        \\  - Objects have varying lifetimes
        \\  - Need to free subset of allocations
        \\  - Long-lived data structures (databases, caches)
        \\  - Objects outlive their phase/scope
        \\
    , .{});
}
```

---

## สรุป Part 73

| Step | หัวข้อ | ประเด็นสำคัญ |
|------|--------|-------------|
| 721 | Concept | Arena = fast alloc, batch free |
| 722 | Implementation | ArenaAllocator.init/reset/deinit |
| 723 | Request/Response | ใช้ arena ต่อ request |
| 724 | Parsing | Arena เหมาะสำหรับ AST |
| 725 | Game Frames | Double-buffered frame arena |
| 726 | Scratch Buffer | Temporary work inside arena |
| 727 | Nested Arenas | Hierarchical lifetime management |
| 728 | Capacity | Growable arena, watermarks |
| 729 | Performance | Arena เร็วกว่า GPA 2-10x |
| 730 | When to Use | Guidelines สำหรับการตัดสินใจ |

### ข้อสรุปหลัก

ArenaAllocator เป็นเครื่องมือที่ดีที่สุดเมื่อ:
1. มี **scope ที่ชัดเจน** (request, frame, phase)
2. **Allocate มาก** แต่ free พร้อมกัน
3. ต้องการ **performance** สูง
4. ต้องการ **โค้ดที่ง่าย** ไม่ต้อง track lifetimes

---

*ต่อไป: Part 74 - Object Pool Pattern*
