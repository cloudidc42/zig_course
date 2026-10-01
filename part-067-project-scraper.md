# ส่วนที่ 67: โปรเจกต์ - Web Scraper

## ขั้นตอนที่ 661-670: สร้าง Web Scraper ด้วย Zig

---

## ขั้นตอนที่ 661: HTTP Client Implementation

Web scraper ต้องการ HTTP client สำหรับดึงเนื้อหาเว็บ Zig มี `std.http` ให้ใช้งาน

### โครงสร้างโปรเจกต์

```
scraper/
├── src/
│   ├── main.zig
│   ├── http_client.zig
│   ├── html_parser.zig
│   ├── crawler.zig
│   ├── robots.zig
│   ├── rate_limiter.zig
│   └── exporter.zig
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
        .name = "scraper",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    const run_step = b.step("run", "Run the scraper");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/html_parser.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

### ไฟล์ src/http_client.zig

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub const HttpError = error{
    ConnectionFailed,
    RequestFailed,
    ResponseTooLarge,
    InvalidUrl,
    TooManyRedirects,
    Timeout,
    OutOfMemory,
};

pub const HttpResponse = struct {
    status_code: u16,
    headers: std.StringHashMap([]u8),
    body: []u8,
    allocator: Allocator,

    pub fn deinit(self: *HttpResponse) void {
        var iter = self.headers.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.headers.deinit();
        self.allocator.free(self.body);
    }

    pub fn getHeader(self: *const HttpResponse, name: []const u8) ?[]const u8 {
        return self.headers.get(name);
    }

    pub fn contentType(self: *const HttpResponse) []const u8 {
        return self.getHeader("content-type") orelse "text/html";
    }
};

pub const HttpClientConfig = struct {
    user_agent: []const u8 = "ZigScraper/1.0",
    timeout_ms: u32 = 30000,
    max_response_size: usize = 10 * 1024 * 1024, // 10MB
    follow_redirects: bool = true,
    max_redirects: u8 = 5,
};

pub const HttpClient = struct {
    allocator: Allocator,
    config: HttpClientConfig,

    const Self = @This();

    pub fn init(allocator: Allocator, config: HttpClientConfig) Self {
        return Self{
            .allocator = allocator,
            .config = config,
        };
    }

    /// GET request
    pub fn get(self: *Self, url: []const u8) !HttpResponse {
        return self.request("GET", url, null, null);
    }

    /// POST request
    pub fn post(self: *Self, url: []const u8, body: []const u8, content_type: []const u8) !HttpResponse {
        var headers = std.StringHashMap([]const u8).init(self.allocator);
        defer headers.deinit();
        try headers.put("Content-Type", content_type);
        return self.request("POST", url, body, headers);
    }

    /// Generic request method
    pub fn request(
        self: *Self,
        method: []const u8,
        url: []const u8,
        body: ?[]const u8,
        extra_headers: ?std.StringHashMap([]const u8),
    ) !HttpResponse {
        var uri = std.Uri.parse(url) catch return HttpError.InvalidUrl;

        var client = std.http.Client{ .allocator = self.allocator };
        defer client.deinit();

        // สร้าง request
        var server_header_buffer: [16384]u8 = undefined;
        var req = client.open(
            std.http.Method.parse(method),
            uri,
            .{
                .server_header_buffer = &server_header_buffer,
                .extra_headers = if (extra_headers) |h| blk: {
                    var list = std.ArrayList(std.http.Header).init(self.allocator);
                    defer list.deinit();
                    var iter = h.iterator();
                    while (iter.next()) |entry| {
                        try list.append(.{
                            .name = entry.key_ptr.*,
                            .value = entry.value_ptr.*,
                        });
                    }
                    break :blk list.items;
                } else &[_]std.http.Header{
                    .{ .name = "User-Agent", .value = self.config.user_agent },
                    .{ .name = "Accept", .value = "text/html,application/xhtml+xml,*/*" },
                },
            },
        ) catch return HttpError.ConnectionFailed;
        defer req.deinit();

        // ส่ง request body ถ้ามี
        if (body) |b| {
            req.transfer_encoding = .{ .content_length = b.len };
        }

        req.send() catch return HttpError.RequestFailed;

        if (body) |b| {
            req.writeAll(b) catch return HttpError.RequestFailed;
            req.finish() catch return HttpError.RequestFailed;
        }

        req.wait() catch return HttpError.RequestFailed;

        const status_code: u16 = @intFromEnum(req.response.status);

        // อ่าน response body
        var body_list = std.ArrayList(u8).init(self.allocator);
        errdefer body_list.deinit();

        var buf: [4096]u8 = undefined;
        while (true) {
            const bytes_read = req.reader().read(&buf) catch break;
            if (bytes_read == 0) break;

            if (body_list.items.len + bytes_read > self.config.max_response_size) {
                return HttpError.ResponseTooLarge;
            }

            try body_list.appendSlice(buf[0..bytes_read]);
        }

        // อ่าน headers
        var headers_map = std.StringHashMap([]u8).init(self.allocator);
        errdefer {
            var iter = headers_map.iterator();
            while (iter.next()) |entry| {
                self.allocator.free(entry.key_ptr.*);
                self.allocator.free(entry.value_ptr.*);
            }
            headers_map.deinit();
        }

        var headers_iter = req.response.iterateHeaders();
        while (headers_iter.next()) |header| {
            const key = try self.allocator.dupe(u8, header.name);
            errdefer self.allocator.free(key);
            const val = try self.allocator.dupe(u8, header.value);
            errdefer self.allocator.free(val);

            // lowercase key
            for (key) |*c| c.* = std.ascii.toLower(c.*);

            try headers_map.put(key, val);
        }

        return HttpResponse{
            .status_code = status_code,
            .headers = headers_map,
            .body = try body_list.toOwnedSlice(),
            .allocator = self.allocator,
        };
    }
};
```

---

## ขั้นตอนที่ 662: HTML Parsing (Basic)

```zig
// ไฟล์ src/html_parser.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

/// HTML Token types
pub const TokenType = enum {
    open_tag,
    close_tag,
    self_closing_tag,
    text,
    comment,
    doctype,
    eof,
};

/// HTML Token
pub const Token = struct {
    token_type: TokenType,
    tag_name: []const u8,
    attributes: std.StringHashMap([]u8),
    text: []const u8,
    allocator: Allocator,

    pub fn deinit(self: *Token) void {
        var iter = self.attributes.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.attributes.deinit();
    }

    pub fn getAttribute(self: *const Token, name: []const u8) ?[]const u8 {
        return self.attributes.get(name);
    }
};

/// HTML Tokenizer
pub const Tokenizer = struct {
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

    pub fn next(self: *Self) !?Token {
        self.skipWhitespace();
        if (self.pos >= self.source.len) return null;

        if (self.source[self.pos] == '<') {
            return try self.parseTag();
        } else {
            return try self.parseText();
        }
    }

    fn parseTag(self: *Self) !Token {
        self.pos += 1; // skip '<'

        // Comment
        if (self.startsWith("!--")) {
            self.pos += 3;
            const start = self.pos;
            while (self.pos + 2 < self.source.len) {
                if (self.startsWith("-->")) {
                    const comment_text = self.source[start..self.pos];
                    self.pos += 3;
                    return Token{
                        .token_type = .comment,
                        .tag_name = "",
                        .attributes = std.StringHashMap([]u8).init(self.allocator),
                        .text = comment_text,
                        .allocator = self.allocator,
                    };
                }
                self.pos += 1;
            }
        }

        // DOCTYPE
        if (self.startsWithIgnoreCase("!DOCTYPE")) {
            while (self.pos < self.source.len and self.source[self.pos] != '>') {
                self.pos += 1;
            }
            if (self.pos < self.source.len) self.pos += 1;
            return Token{
                .token_type = .doctype,
                .tag_name = "DOCTYPE",
                .attributes = std.StringHashMap([]u8).init(self.allocator),
                .text = "",
                .allocator = self.allocator,
            };
        }

        // Close tag
        const is_close = self.pos < self.source.len and self.source[self.pos] == '/';
        if (is_close) self.pos += 1;

        // Tag name
        const name_start = self.pos;
        while (self.pos < self.source.len and
            !std.ascii.isWhitespace(self.source[self.pos]) and
            self.source[self.pos] != '>' and
            self.source[self.pos] != '/')
        {
            self.pos += 1;
        }
        const tag_name = self.source[name_start..self.pos];

        self.skipWhitespace();

        // Attributes
        var attrs = std.StringHashMap([]u8).init(self.allocator);
        errdefer {
            var iter = attrs.iterator();
            while (iter.next()) |entry| {
                self.allocator.free(entry.key_ptr.*);
                self.allocator.free(entry.value_ptr.*);
            }
            attrs.deinit();
        }

        while (self.pos < self.source.len and
            self.source[self.pos] != '>' and
            self.source[self.pos] != '/')
        {
            try self.parseAttribute(&attrs);
            self.skipWhitespace();
        }

        // Self-closing?
        const is_self_closing = self.pos < self.source.len and self.source[self.pos] == '/';
        if (is_self_closing) self.pos += 1;

        if (self.pos < self.source.len and self.source[self.pos] == '>') {
            self.pos += 1;
        }

        const token_type: TokenType = if (is_close)
            .close_tag
        else if (is_self_closing)
            .self_closing_tag
        else
            .open_tag;

        return Token{
            .token_type = token_type,
            .tag_name = tag_name,
            .attributes = attrs,
            .text = "",
            .allocator = self.allocator,
        };
    }

    fn parseAttribute(self: *Self, attrs: *std.StringHashMap([]u8)) !void {
        const name_start = self.pos;
        while (self.pos < self.source.len and
            !std.ascii.isWhitespace(self.source[self.pos]) and
            self.source[self.pos] != '=' and
            self.source[self.pos] != '>' and
            self.source[self.pos] != '/')
        {
            self.pos += 1;
        }
        const attr_name = self.source[name_start..self.pos];
        if (attr_name.len == 0) {
            self.pos += 1;
            return;
        }

        self.skipWhitespace();

        var attr_value: []const u8 = "";
        if (self.pos < self.source.len and self.source[self.pos] == '=') {
            self.pos += 1;
            self.skipWhitespace();
            attr_value = try self.parseAttributeValue();
        }

        const key = try self.allocator.dupe(u8, attr_name);
        errdefer self.allocator.free(key);
        // lowercase key
        for (key) |*c| c.* = std.ascii.toLower(c.*);

        const val = try self.allocator.dupe(u8, attr_value);
        errdefer self.allocator.free(val);

        try attrs.put(key, val);
    }

    fn parseAttributeValue(self: *Self) ![]const u8 {
        if (self.pos >= self.source.len) return "";

        if (self.source[self.pos] == '"' or self.source[self.pos] == '\'') {
            const quote = self.source[self.pos];
            self.pos += 1;
            const start = self.pos;
            while (self.pos < self.source.len and self.source[self.pos] != quote) {
                self.pos += 1;
            }
            const value = self.source[start..self.pos];
            if (self.pos < self.source.len) self.pos += 1;
            return value;
        } else {
            const start = self.pos;
            while (self.pos < self.source.len and
                !std.ascii.isWhitespace(self.source[self.pos]) and
                self.source[self.pos] != '>' and
                self.source[self.pos] != '/')
            {
                self.pos += 1;
            }
            return self.source[start..self.pos];
        }
    }

    fn parseText(self: *Self) !Token {
        const start = self.pos;
        while (self.pos < self.source.len and self.source[self.pos] != '<') {
            self.pos += 1;
        }
        const text = std.mem.trim(u8, self.source[start..self.pos], " \t\r\n");
        return Token{
            .token_type = .text,
            .tag_name = "",
            .attributes = std.StringHashMap([]u8).init(self.allocator),
            .text = text,
            .allocator = self.allocator,
        };
    }

    fn skipWhitespace(self: *Self) void {
        while (self.pos < self.source.len and std.ascii.isWhitespace(self.source[self.pos])) {
            self.pos += 1;
        }
    }

    fn startsWith(self: *Self, prefix: []const u8) bool {
        if (self.pos + prefix.len > self.source.len) return false;
        return std.mem.eql(u8, self.source[self.pos .. self.pos + prefix.len], prefix);
    }

    fn startsWithIgnoreCase(self: *Self, prefix: []const u8) bool {
        if (self.pos + prefix.len > self.source.len) return false;
        return std.ascii.eqlIgnoreCase(self.source[self.pos .. self.pos + prefix.len], prefix);
    }
};
```

---

## ขั้นตอนที่ 663: CSS Selector-Like Queries

```zig
// ต่อจาก html_parser.zig

/// DOM Node
pub const Node = struct {
    tag_name: []const u8,
    attributes: std.StringHashMap([]u8),
    text_content: []u8,
    children: std.ArrayList(*Node),
    parent: ?*Node,
    allocator: Allocator,

    pub fn init(allocator: Allocator, tag_name: []const u8) !*Node {
        const node = try allocator.create(Node);
        node.* = Node{
            .tag_name = try allocator.dupe(u8, tag_name),
            .attributes = std.StringHashMap([]u8).init(allocator),
            .text_content = try allocator.dupe(u8, ""),
            .children = std.ArrayList(*Node).init(allocator),
            .parent = null,
            .allocator = allocator,
        };
        return node;
    }

    pub fn deinit(self: *Node) void {
        self.allocator.free(self.tag_name);
        self.allocator.free(self.text_content);

        var iter = self.attributes.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.attributes.deinit();

        for (self.children.items) |child| {
            child.deinit();
            self.allocator.destroy(child);
        }
        self.children.deinit();
    }

    pub fn getAttribute(self: *const Node, name: []const u8) ?[]const u8 {
        return self.attributes.get(name);
    }

    pub fn hasClass(self: *const Node, class_name: []const u8) bool {
        const classes = self.getAttribute("class") orelse return false;
        var iter = std.mem.tokenizeScalar(u8, classes, ' ');
        while (iter.next()) |cls| {
            if (std.mem.eql(u8, cls, class_name)) return true;
        }
        return false;
    }

    pub fn getId(self: *const Node) ?[]const u8 {
        return self.getAttribute("id");
    }

    /// ค้นหา nodes ด้วย simple selector
    pub fn querySelectorAll(
        self: *Node,
        selector: []const u8,
        allocator: Allocator,
    ) !std.ArrayList(*Node) {
        var results = std.ArrayList(*Node).init(allocator);
        try self.queryAll(selector, &results);
        return results;
    }

    fn queryAll(self: *Node, selector: []const u8, results: *std.ArrayList(*Node)) !void {
        if (matchesSelector(self, selector)) {
            try results.append(self);
        }
        for (self.children.items) |child| {
            try child.queryAll(selector, results);
        }
    }

    /// ค้นหา node แรกที่ match
    pub fn querySelector(self: *Node, selector: []const u8) ?*Node {
        if (matchesSelector(self, selector)) return self;
        for (self.children.items) |child| {
            if (child.querySelector(selector)) |found| return found;
        }
        return null;
    }
};

/// ตรวจสอบว่า node match selector หรือไม่
fn matchesSelector(node: *const Node, selector: []const u8) bool {
    // Tag selector: div, p, a
    if (selector[0] != '.' and selector[0] != '#' and selector[0] != '[') {
        return std.ascii.eqlIgnoreCase(node.tag_name, selector);
    }

    // Class selector: .classname
    if (selector[0] == '.') {
        return node.hasClass(selector[1..]);
    }

    // ID selector: #id
    if (selector[0] == '#') {
        const id = node.getId() orelse return false;
        return std.mem.eql(u8, id, selector[1..]);
    }

    // Attribute selector: [attr] or [attr=value]
    if (selector[0] == '[' and selector[selector.len - 1] == ']') {
        const inner = selector[1 .. selector.len - 1];
        if (std.mem.indexOf(u8, inner, "=")) |eq_pos| {
            const attr_name = inner[0..eq_pos];
            const attr_value = inner[eq_pos + 1 ..];
            const actual = node.getAttribute(attr_name) orelse return false;
            return std.mem.eql(u8, actual, attr_value);
        } else {
            return node.getAttribute(inner) != null;
        }
    }

    return false;
}

/// HTML Document parser สร้าง DOM tree
pub const HtmlDocument = struct {
    root: *Node,
    allocator: Allocator,

    const Self = @This();

    pub fn parse(allocator: Allocator, html: []const u8) !Self {
        const root = try Node.init(allocator, "document");
        var current = root;

        var tokenizer = Tokenizer.init(allocator, html);

        while (try tokenizer.next()) |*token| {
            defer {
                var t = token.*;
                t.deinit();
            }

            switch (token.token_type) {
                .open_tag => {
                    const child = try Node.init(allocator, token.tag_name);
                    child.parent = current;

                    // copy attributes
                    var attr_iter = token.attributes.iterator();
                    while (attr_iter.next()) |attr| {
                        const k = try allocator.dupe(u8, attr.key_ptr.*);
                        errdefer allocator.free(k);
                        const v = try allocator.dupe(u8, attr.value_ptr.*);
                        errdefer allocator.free(v);
                        try child.attributes.put(k, v);
                    }

                    try current.children.append(child);

                    // void elements ไม่มี close tag
                    const is_void = isVoidElement(token.tag_name);
                    if (!is_void) {
                        current = child;
                    }
                },
                .close_tag => {
                    if (current.parent) |parent| {
                        current = parent;
                    }
                },
                .self_closing_tag => {
                    const child = try Node.init(allocator, token.tag_name);
                    child.parent = current;
                    try current.children.append(child);
                },
                .text => {
                    if (token.text.len > 0) {
                        allocator.free(current.text_content);
                        current.text_content = try allocator.dupe(u8, token.text);
                    }
                },
                else => {},
            }
        }

        return Self{
            .root = root,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Self) void {
        self.root.deinit();
        self.allocator.destroy(self.root);
    }

    pub fn querySelector(self: *Self, selector: []const u8) ?*Node {
        return self.root.querySelector(selector);
    }

    pub fn querySelectorAll(self: *Self, selector: []const u8) !std.ArrayList(*Node) {
        return self.root.querySelectorAll(selector, self.allocator);
    }
};

fn isVoidElement(tag: []const u8) bool {
    const void_elements = [_][]const u8{
        "area", "base", "br", "col", "embed", "hr", "img",
        "input", "link", "meta", "param", "source", "track", "wbr",
    };
    for (void_elements) |ve| {
        if (std.ascii.eqlIgnoreCase(tag, ve)) return true;
    }
    return false;
}
```

---

## ขั้นตอนที่ 664: Link Extraction

```zig
// ต่อจาก html_parser.zig

pub const Link = struct {
    href: []u8,
    text: []u8,
    is_external: bool,
    allocator: Allocator,

    pub fn deinit(self: *Link) void {
        self.allocator.free(self.href);
        self.allocator.free(self.text);
    }
};

/// ดึง links ทั้งหมดจาก document
pub fn extractLinks(
    doc: *HtmlDocument,
    base_url: []const u8,
    allocator: Allocator,
) !std.ArrayList(Link) {
    var links = std.ArrayList(Link).init(allocator);
    errdefer {
        for (links.items) |*link| link.deinit();
        links.deinit();
    }

    const anchors = try doc.querySelectorAll("a");
    defer anchors.deinit();

    for (anchors.items) |anchor| {
        const href_raw = anchor.getAttribute("href") orelse continue;

        // ข้ามลิงค์ที่ไม่ใช่ HTTP
        if (std.mem.startsWith(u8, href_raw, "javascript:")) continue;
        if (std.mem.startsWith(u8, href_raw, "mailto:")) continue;
        if (std.mem.startsWith(u8, href_raw, "tel:")) continue;
        if (href_raw.len == 0 or std.mem.eql(u8, href_raw, "#")) continue;

        // Resolve relative URL
        const absolute_url = try resolveUrl(allocator, base_url, href_raw);
        errdefer allocator.free(absolute_url);

        // ดึง link text
        const link_text = try getLinkText(anchor, allocator);
        errdefer allocator.free(link_text);

        // ตรวจสอบว่าเป็น external link หรือไม่
        const is_external = !std.mem.startsWith(u8, absolute_url, getOrigin(base_url));

        try links.append(Link{
            .href = absolute_url,
            .text = link_text,
            .is_external = is_external,
            .allocator = allocator,
        });
    }

    return links;
}

fn resolveUrl(allocator: Allocator, base: []const u8, relative: []const u8) ![]u8 {
    // ถ้าเป็น absolute URL แล้ว
    if (std.mem.startsWith(u8, relative, "http://") or
        std.mem.startsWith(u8, relative, "https://"))
    {
        return allocator.dupe(u8, relative);
    }

    // Protocol-relative URL
    if (std.mem.startsWith(u8, relative, "//")) {
        const protocol = if (std.mem.startsWith(u8, base, "https")) "https" else "http";
        return std.fmt.allocPrint(allocator, "{s}:{s}", .{ protocol, relative });
    }

    // Root-relative URL
    if (relative[0] == '/') {
        const origin = getOrigin(base);
        return std.fmt.allocPrint(allocator, "{s}{s}", .{ origin, relative });
    }

    // Relative URL
    const base_dir = getBaseDir(base);
    return std.fmt.allocPrint(allocator, "{s}/{s}", .{ base_dir, relative });
}

fn getOrigin(url: []const u8) []const u8 {
    // หา end of host (ตำแหน่งที่ 3 ของ '/')
    var slash_count: usize = 0;
    for (url, 0..) |c, i| {
        if (c == '/') {
            slash_count += 1;
            if (slash_count == 3) return url[0..i];
        }
    }
    return url;
}

fn getBaseDir(url: []const u8) []const u8 {
    var last_slash: usize = 0;
    for (url, 0..) |c, i| {
        if (c == '/') last_slash = i;
    }
    return url[0..last_slash];
}

fn getLinkText(node: *const Node, allocator: Allocator) ![]u8 {
    var text = std.ArrayList(u8).init(allocator);
    defer text.deinit();

    try collectText(node, &text);
    const result = std.mem.trim(u8, text.items, " \t\r\n");
    return allocator.dupe(u8, result);
}

fn collectText(node: *const Node, out: *std.ArrayList(u8)) !void {
    if (node.text_content.len > 0) {
        try out.appendSlice(node.text_content);
        try out.append(' ');
    }
    for (node.children.items) |child| {
        try collectText(child, out);
    }
}
```

---

## ขั้นตอนที่ 665: Crawling with Breadth-First

```zig
// ไฟล์ src/crawler.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const http_mod = @import("http_client.zig");
const html_mod = @import("html_parser.zig");

pub const CrawlResult = struct {
    url: []u8,
    status_code: u16,
    title: []u8,
    links: []html_mod.Link,
    text_content: []u8,
    crawled_at: i64,
    allocator: Allocator,

    pub fn deinit(self: *CrawlResult) void {
        self.allocator.free(self.url);
        self.allocator.free(self.title);
        for (self.links) |*link| {
            var l = link.*;
            l.deinit();
        }
        self.allocator.free(self.links);
        self.allocator.free(self.text_content);
    }
};

pub const CrawlerConfig = struct {
    max_depth: usize = 3,
    max_pages: usize = 100,
    same_domain_only: bool = true,
    follow_external: bool = false,
    delay_ms: u64 = 1000, // 1 วินาทีระหว่าง requests
    user_agent: []const u8 = "ZigBot/1.0",
};

pub const Crawler = struct {
    allocator: Allocator,
    config: CrawlerConfig,
    http_client: http_mod.HttpClient,
    visited: std.StringHashMap(bool),
    queue: std.ArrayList(QueueItem),
    results: std.ArrayList(CrawlResult),

    const QueueItem = struct {
        url: []u8,
        depth: usize,
    };

    const Self = @This();

    pub fn init(allocator: Allocator, config: CrawlerConfig) Self {
        return Self{
            .allocator = allocator,
            .config = config,
            .http_client = http_mod.HttpClient.init(allocator, .{
                .user_agent = config.user_agent,
            }),
            .visited = std.StringHashMap(bool).init(allocator),
            .queue = std.ArrayList(QueueItem).init(allocator),
            .results = std.ArrayList(CrawlResult).init(allocator),
        };
    }

    pub fn deinit(self: *Self) void {
        var vis_iter = self.visited.iterator();
        while (vis_iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
        }
        self.visited.deinit();

        for (self.queue.items) |item| {
            self.allocator.free(item.url);
        }
        self.queue.deinit();

        for (self.results.items) |*result| {
            result.deinit();
        }
        self.results.deinit();
    }

    /// เริ่ม crawl จาก URL
    pub fn crawl(self: *Self, start_url: []const u8) !void {
        const start_url_copy = try self.allocator.dupe(u8, start_url);
        try self.queue.append(.{ .url = start_url_copy, .depth = 0 });

        std.log.info("Starting crawl from: {s}", .{start_url});

        while (self.queue.items.len > 0 and
            self.results.items.len < self.config.max_pages)
        {
            // BFS: ดึง item แรกจาก queue
            const item = self.queue.orderedRemove(0);
            defer self.allocator.free(item.url);

            // ข้าม URL ที่เคยเยี่ยมชมแล้ว
            if (self.visited.contains(item.url)) continue;

            // Mark as visited
            const url_key = try self.allocator.dupe(u8, item.url);
            try self.visited.put(url_key, true);

            // Crawl หน้านี้
            std.log.info("Crawling [{}/{}]: {s}", .{
                self.results.items.len + 1,
                self.config.max_pages,
                item.url,
            });

            const result = self.crawlPage(item.url) catch |err| {
                std.log.warn("Failed to crawl {s}: {}", .{ item.url, err });
                continue;
            };

            // เพิ่ม links ลง queue ถ้ายังไม่ถึง max depth
            if (item.depth < self.config.max_depth) {
                for (result.links) |link| {
                    if (!self.visited.contains(link.href)) {
                        // ถ้า same_domain_only ตรวจสอบ domain
                        if (self.config.same_domain_only and link.is_external) continue;

                        const link_url = try self.allocator.dupe(u8, link.href);
                        try self.queue.append(.{
                            .url = link_url,
                            .depth = item.depth + 1,
                        });
                    }
                }
            }

            try self.results.append(result);

            // Rate limiting
            if (self.config.delay_ms > 0) {
                std.time.sleep(self.config.delay_ms * std.time.ns_per_ms);
            }
        }

        std.log.info("Crawl complete: {} pages", .{self.results.items.len});
    }

    fn crawlPage(self: *Self, url: []const u8) !CrawlResult {
        var response = try self.http_client.get(url);
        defer response.deinit();

        var doc = try html_mod.HtmlDocument.parse(self.allocator, response.body);
        defer doc.deinit();

        // ดึง title
        const title_text = blk: {
            if (doc.querySelector("title")) |title_node| {
                break :blk try self.allocator.dupe(u8, title_node.text_content);
            }
            break :blk try self.allocator.dupe(u8, "");
        };
        errdefer self.allocator.free(title_text);

        // ดึง links
        var links = try html_mod.extractLinks(&doc, url, self.allocator);
        errdefer {
            for (links.items) |*link| link.deinit();
            links.deinit();
        }

        // ดึง text content จาก body
        const body_text = blk: {
            if (doc.querySelector("body")) |body_node| {
                var text_buf = std.ArrayList(u8).init(self.allocator);
                defer text_buf.deinit();
                try html_mod.collectText(body_node, &text_buf);
                break :blk try text_buf.toOwnedSlice();
            }
            break :blk try self.allocator.dupe(u8, "");
        };
        errdefer self.allocator.free(body_text);

        return CrawlResult{
            .url = try self.allocator.dupe(u8, url),
            .status_code = response.status_code,
            .title = title_text,
            .links = try links.toOwnedSlice(),
            .text_content = body_text,
            .crawled_at = std.time.milliTimestamp(),
            .allocator = self.allocator,
        };
    }
};
```

---

## ขั้นตอนที่ 666: Rate Limiting

```zig
// ไฟล์ src/rate_limiter.zig

const std = @import("std");

/// Token Bucket Rate Limiter
pub const RateLimiter = struct {
    tokens: f64,
    max_tokens: f64,
    refill_rate: f64, // tokens per second
    last_refill: i64,
    mutex: std.Thread.Mutex,

    const Self = @This();

    /// สร้าง rate limiter
    /// requests_per_second: จำนวน requests ที่อนุญาตต่อวินาที
    pub fn init(requests_per_second: f64) Self {
        return Self{
            .tokens = requests_per_second,
            .max_tokens = requests_per_second,
            .refill_rate = requests_per_second,
            .last_refill = std.time.milliTimestamp(),
            .mutex = std.Thread.Mutex{},
        };
    }

    /// รอจนกว่าจะได้ token แล้วใช้งาน
    pub fn acquire(self: *Self) void {
        while (!self.tryAcquire()) {
            std.time.sleep(10 * std.time.ns_per_ms); // รอ 10ms แล้วลองใหม่
        }
    }

    /// พยายามดึง token โดยไม่ block
    pub fn tryAcquire(self: *Self) bool {
        self.mutex.lock();
        defer self.mutex.unlock();

        self.refill();

        if (self.tokens >= 1.0) {
            self.tokens -= 1.0;
            return true;
        }
        return false;
    }

    fn refill(self: *Self) void {
        const now = std.time.milliTimestamp();
        const elapsed_sec = @as(f64, @floatFromInt(now - self.last_refill)) / 1000.0;

        self.tokens = @min(
            self.max_tokens,
            self.tokens + elapsed_sec * self.refill_rate,
        );
        self.last_refill = now;
    }

    /// ดู tokens ที่เหลืออยู่
    pub fn availableTokens(self: *Self) f64 {
        self.mutex.lock();
        defer self.mutex.unlock();
        self.refill();
        return self.tokens;
    }
};

/// Per-domain rate limiter
pub const DomainRateLimiter = struct {
    allocator: std.mem.Allocator,
    limiters: std.StringHashMap(*RateLimiter),
    default_rps: f64,
    mutex: std.Thread.Mutex,

    const Self = @This();

    pub fn init(allocator: std.mem.Allocator, default_rps: f64) Self {
        return Self{
            .allocator = allocator,
            .limiters = std.StringHashMap(*RateLimiter).init(allocator),
            .default_rps = default_rps,
            .mutex = std.Thread.Mutex{},
        };
    }

    pub fn deinit(self: *Self) void {
        var iter = self.limiters.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.destroy(entry.value_ptr.*);
        }
        self.limiters.deinit();
    }

    /// ดึง rate limiter สำหรับ domain
    pub fn getLimiter(self: *Self, domain: []const u8) !*RateLimiter {
        self.mutex.lock();
        defer self.mutex.unlock();

        if (self.limiters.get(domain)) |limiter| {
            return limiter;
        }

        const limiter = try self.allocator.create(RateLimiter);
        limiter.* = RateLimiter.init(self.default_rps);

        const domain_copy = try self.allocator.dupe(u8, domain);
        try self.limiters.put(domain_copy, limiter);
        return limiter;
    }

    /// รอ rate limit สำหรับ URL
    pub fn acquireForUrl(self: *Self, url: []const u8) !void {
        const domain = extractDomain(url);
        const limiter = try self.getLimiter(domain);
        limiter.acquire();
    }

    fn extractDomain(url: []const u8) []const u8 {
        // ข้าม protocol
        var start: usize = 0;
        if (std.mem.indexOf(u8, url, "://")) |idx| {
            start = idx + 3;
        }

        // หา end of domain
        const rest = url[start..];
        if (std.mem.indexOfScalar(u8, rest, '/')) |slash_idx| {
            return rest[0..slash_idx];
        }
        return rest;
    }
};

test "RateLimiter basic" {
    var limiter = RateLimiter.init(10.0); // 10 requests/sec

    // ควรได้ 10 tokens ทันที
    var count: usize = 0;
    while (limiter.tryAcquire()) {
        count += 1;
        if (count > 20) break; // safety
    }
    try std.testing.expect(count == 10);

    // ควรไม่ได้ token เพิ่ม
    try std.testing.expect(!limiter.tryAcquire());
}
```

---

## ขั้นตอนที่ 667: Robots.txt Handling

```zig
// ไฟล์ src/robots.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const http_mod = @import("http_client.zig");

pub const RobotRule = struct {
    user_agent: []u8,
    disallow: std.ArrayList([]u8),
    allow: std.ArrayList([]u8),
    crawl_delay: ?u32,
    allocator: Allocator,

    pub fn init(allocator: Allocator, user_agent: []const u8) !RobotRule {
        return RobotRule{
            .user_agent = try allocator.dupe(u8, user_agent),
            .disallow = std.ArrayList([]u8).init(allocator),
            .allow = std.ArrayList([]u8).init(allocator),
            .crawl_delay = null,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *RobotRule) void {
        self.allocator.free(self.user_agent);
        for (self.disallow.items) |path| self.allocator.free(path);
        self.disallow.deinit();
        for (self.allow.items) |path| self.allocator.free(path);
        self.allow.deinit();
    }

    pub fn isAllowed(self: *const RobotRule, path: []const u8) bool {
        // ตรวจ allow rules ก่อน (specific wins)
        for (self.allow.items) |allow_path| {
            if (pathMatches(path, allow_path)) return true;
        }

        // ตรวจ disallow rules
        for (self.disallow.items) |disallow_path| {
            if (pathMatches(path, disallow_path)) return false;
        }

        return true; // default: allowed
    }

    fn pathMatches(path: []const u8, pattern: []const u8) bool {
        if (pattern.len == 0) return false;
        if (std.mem.eql(u8, pattern, "/")) return true;

        // Wildcard support
        if (std.mem.endsWith(u8, pattern, "*")) {
            return std.mem.startsWith(u8, path, pattern[0 .. pattern.len - 1]);
        }

        return std.mem.startsWith(u8, path, pattern);
    }
};

pub const RobotsTxt = struct {
    rules: std.ArrayList(RobotRule),
    sitemaps: std.ArrayList([]u8),
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{
            .rules = std.ArrayList(RobotRule).init(allocator),
            .sitemaps = std.ArrayList([]u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Self) void {
        for (self.rules.items) |*rule| rule.deinit();
        self.rules.deinit();
        for (self.sitemaps.items) |sm| self.allocator.free(sm);
        self.sitemaps.deinit();
    }

    /// Parse robots.txt content
    pub fn parse(allocator: Allocator, content: []const u8) !Self {
        var robots = Self.init(allocator);
        errdefer robots.deinit();

        var current_rule: ?RobotRule = null;
        var lines = std.mem.splitScalar(u8, content, '\n');

        while (lines.next()) |line_raw| {
            const line = std.mem.trim(u8, line_raw, " \t\r");

            // ข้าม comments และบรรทัดว่าง
            if (line.len == 0 or line[0] == '#') continue;

            if (std.mem.indexOf(u8, line, ":")) |colon_pos| {
                const directive = std.mem.trim(u8, line[0..colon_pos], " ");
                const value = std.mem.trim(u8, line[colon_pos + 1 ..], " ");

                if (std.ascii.eqlIgnoreCase(directive, "User-agent")) {
                    // บันทึก rule เก่าก่อน
                    if (current_rule) |*rule| {
                        try robots.rules.append(rule.*);
                        current_rule = null;
                    }
                    current_rule = try RobotRule.init(allocator, value);

                } else if (std.ascii.eqlIgnoreCase(directive, "Disallow")) {
                    if (current_rule) |*rule| {
                        const path = try allocator.dupe(u8, value);
                        try rule.disallow.append(path);
                    }

                } else if (std.ascii.eqlIgnoreCase(directive, "Allow")) {
                    if (current_rule) |*rule| {
                        const path = try allocator.dupe(u8, value);
                        try rule.allow.append(path);
                    }

                } else if (std.ascii.eqlIgnoreCase(directive, "Crawl-delay")) {
                    if (current_rule) |*rule| {
                        rule.crawl_delay = std.fmt.parseInt(u32, value, 10) catch null;
                    }

                } else if (std.ascii.eqlIgnoreCase(directive, "Sitemap")) {
                    const sm = try allocator.dupe(u8, value);
                    try robots.sitemaps.append(sm);
                }
            }
        }

        // บันทึก rule สุดท้าย
        if (current_rule) |*rule| {
            try robots.rules.append(rule.*);
        }

        return robots;
    }

    /// ตรวจสอบว่า URL ถูกอนุญาตสำหรับ user agent นี้
    pub fn isAllowed(self: *const Self, user_agent: []const u8, path: []const u8) bool {
        // หา rule ที่ match user agent
        var specific_rule: ?*const RobotRule = null;
        var wildcard_rule: ?*const RobotRule = null;

        for (self.rules.items) |*rule| {
            if (std.mem.eql(u8, rule.user_agent, "*")) {
                wildcard_rule = rule;
            } else if (std.ascii.eqlIgnoreCase(rule.user_agent, user_agent)) {
                specific_rule = rule;
            }
        }

        if (specific_rule) |rule| return rule.isAllowed(path);
        if (wildcard_rule) |rule| return rule.isAllowed(path);
        return true; // ไม่มี rules = allowed
    }

    /// Fetch robots.txt จาก URL
    pub fn fetch(allocator: Allocator, base_url: []const u8) !Self {
        var client = http_mod.HttpClient.init(allocator, .{});

        const robots_url = try std.fmt.allocPrint(
            allocator,
            "{s}/robots.txt",
            .{getOrigin(base_url)},
        );
        defer allocator.free(robots_url);

        var response = client.get(robots_url) catch {
            // ถ้าไม่มี robots.txt = allowed ทั้งหมด
            return Self.init(allocator);
        };
        defer response.deinit();

        if (response.status_code != 200) {
            return Self.init(allocator);
        }

        return Self.parse(allocator, response.body);
    }

    fn getOrigin(url: []const u8) []const u8 {
        var slash_count: usize = 0;
        for (url, 0..) |c, i| {
            if (c == '/') {
                slash_count += 1;
                if (slash_count == 3) return url[0..i];
            }
        }
        return url;
    }
};
```

---

## ขั้นตอนที่ 668: Data Extraction

```zig
// ต่อจาก html_parser.zig

/// Extracted data จากหน้าเว็บ
pub const ExtractedData = struct {
    url: []u8,
    title: []u8,
    description: []u8,
    keywords: []u8,
    headings: std.ArrayList(Heading),
    images: std.ArrayList(Image),
    paragraphs: std.ArrayList([]u8),
    allocator: Allocator,

    pub const Heading = struct {
        level: u8, // 1-6
        text: []u8,
    };

    pub const Image = struct {
        src: []u8,
        alt: []u8,
    };

    pub fn deinit(self: *ExtractedData) void {
        self.allocator.free(self.url);
        self.allocator.free(self.title);
        self.allocator.free(self.description);
        self.allocator.free(self.keywords);

        for (self.headings.items) |h| {
            self.allocator.free(h.text);
        }
        self.headings.deinit();

        for (self.images.items) |img| {
            self.allocator.free(img.src);
            self.allocator.free(img.alt);
        }
        self.images.deinit();

        for (self.paragraphs.items) |p| self.allocator.free(p);
        self.paragraphs.deinit();
    }
};

pub const DataExtractor = struct {
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{ .allocator = allocator };
    }

    pub fn extract(self: *Self, doc: *HtmlDocument, url: []const u8) !ExtractedData {
        var data = ExtractedData{
            .url = try self.allocator.dupe(u8, url),
            .title = try self.extractTitle(doc),
            .description = try self.extractMetaContent(doc, "description"),
            .keywords = try self.extractMetaContent(doc, "keywords"),
            .headings = std.ArrayList(ExtractedData.Heading).init(self.allocator),
            .images = std.ArrayList(ExtractedData.Image).init(self.allocator),
            .paragraphs = std.ArrayList([]u8).init(self.allocator),
            .allocator = self.allocator,
        };
        errdefer data.deinit();

        try self.extractHeadings(doc, &data.headings);
        try self.extractImages(doc, &data.images);
        try self.extractParagraphs(doc, &data.paragraphs);

        return data;
    }

    fn extractTitle(self: *Self, doc: *HtmlDocument) ![]u8 {
        if (doc.querySelector("title")) |node| {
            return self.allocator.dupe(u8, node.text_content);
        }
        return self.allocator.dupe(u8, "");
    }

    fn extractMetaContent(self: *Self, doc: *HtmlDocument, name: []const u8) ![]u8 {
        const metas = try doc.querySelectorAll("meta");
        defer metas.deinit();

        for (metas.items) |meta| {
            const meta_name = meta.getAttribute("name") orelse continue;
            if (std.ascii.eqlIgnoreCase(meta_name, name)) {
                const content = meta.getAttribute("content") orelse "";
                return self.allocator.dupe(u8, content);
            }
        }
        return self.allocator.dupe(u8, "");
    }

    fn extractHeadings(
        self: *Self,
        doc: *HtmlDocument,
        headings: *std.ArrayList(ExtractedData.Heading),
    ) !void {
        const heading_tags = [_][]const u8{ "h1", "h2", "h3", "h4", "h5", "h6" };

        for (heading_tags, 1..) |tag, level| {
            const nodes = try doc.querySelectorAll(tag);
            defer nodes.deinit();

            for (nodes.items) |node| {
                if (node.text_content.len > 0) {
                    try headings.append(.{
                        .level = @intCast(level),
                        .text = try self.allocator.dupe(u8, node.text_content),
                    });
                }
            }
        }
    }

    fn extractImages(
        self: *Self,
        doc: *HtmlDocument,
        images: *std.ArrayList(ExtractedData.Image),
    ) !void {
        const imgs = try doc.querySelectorAll("img");
        defer imgs.deinit();

        for (imgs.items) |img| {
            const src = img.getAttribute("src") orelse continue;
            const alt = img.getAttribute("alt") orelse "";
            try images.append(.{
                .src = try self.allocator.dupe(u8, src),
                .alt = try self.allocator.dupe(u8, alt),
            });
        }
    }

    fn extractParagraphs(
        self: *Self,
        doc: *HtmlDocument,
        paragraphs: *std.ArrayList([]u8),
    ) !void {
        const ps = try doc.querySelectorAll("p");
        defer ps.deinit();

        for (ps.items) |p| {
            if (p.text_content.len > 10) { // ข้ามย่อหน้าที่สั้นมาก
                try paragraphs.append(try self.allocator.dupe(u8, p.text_content));
            }
        }
    }
};
```

---

## ขั้นตอนที่ 669: Export to JSON/CSV

```zig
// ไฟล์ src/exporter.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const crawler_mod = @import("crawler.zig");
const html_mod = @import("html_parser.zig");

pub const Exporter = struct {
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{ .allocator = allocator };
    }

    /// Export ผลลัพธ์เป็น JSON
    pub fn toJson(
        self: *Self,
        results: []const crawler_mod.CrawlResult,
        writer: anytype,
    ) !void {
        _ = self;
        try writer.writeAll("[\n");

        for (results, 0..) |result, i| {
            try writer.writeAll("  {\n");
            try writer.print("    \"url\": \"{s}\",\n", .{jsonEscape(result.url)});
            try writer.print("    \"status_code\": {},\n", .{result.status_code});
            try writer.print("    \"title\": \"{s}\",\n", .{jsonEscape(result.title)});
            try writer.print("    \"crawled_at\": {},\n", .{result.crawled_at});
            try writer.print("    \"links_count\": {},\n", .{result.links.len});

            // Links array
            try writer.writeAll("    \"links\": [\n");
            for (result.links, 0..) |link, j| {
                try writer.print("      \"{s}\"", .{jsonEscape(link.href)});
                if (j < result.links.len - 1) try writer.writeAll(",");
                try writer.writeAll("\n");
            }
            try writer.writeAll("    ]\n");

            if (i < results.len - 1) {
                try writer.writeAll("  },\n");
            } else {
                try writer.writeAll("  }\n");
            }
        }

        try writer.writeAll("]\n");
    }

    /// Export ผลลัพธ์เป็น CSV
    pub fn toCsv(
        self: *Self,
        results: []const crawler_mod.CrawlResult,
        writer: anytype,
    ) !void {
        _ = self;
        // Header
        try writer.writeAll("url,status_code,title,links_count,crawled_at\n");

        for (results) |result| {
            try writer.print("{s},{},{s},{},{}\n", .{
                csvEscape(result.url),
                result.status_code,
                csvEscape(result.title),
                result.links.len,
                result.crawled_at,
            });
        }
    }

    /// บันทึก JSON ลงไฟล์
    pub fn saveJson(self: *Self, results: []const crawler_mod.CrawlResult, path: []const u8) !void {
        const file = try std.fs.createFileAbsolute(path, .{});
        defer file.close();
        var buf_writer = std.io.bufferedWriter(file.writer());
        try self.toJson(results, buf_writer.writer());
        try buf_writer.flush();
        std.log.info("Saved JSON to {s}", .{path});
    }

    /// บันทึก CSV ลงไฟล์
    pub fn saveCsv(self: *Self, results: []const crawler_mod.CrawlResult, path: []const u8) !void {
        const file = try std.fs.createFileAbsolute(path, .{});
        defer file.close();
        var buf_writer = std.io.bufferedWriter(file.writer());
        try self.toCsv(results, buf_writer.writer());
        try buf_writer.flush();
        std.log.info("Saved CSV to {s}", .{path});
    }
};

fn jsonEscape(s: []const u8) []const u8 {
    // Simple version - ในการใช้งานจริงควร escape characters พิเศษ
    _ = s;
    return s;
}

fn csvEscape(s: []const u8) []const u8 {
    _ = s;
    return s;
}
```

---

## ขั้นตอนที่ 670: Complete Scraper Example

```zig
// ไฟล์ src/main.zig

const std = @import("std");
const crawler_mod = @import("crawler.zig");
const exporter_mod = @import("exporter.zig");
const robots_mod = @import("robots.zig");
const rate_mod = @import("rate_limiter.zig");
const html_mod = @import("html_parser.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Parse command line args
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 2) {
        std.debug.print(
            \\Usage: scraper <url> [options]
            \\Options:
            \\  --depth <n>      Max crawl depth (default: 3)
            \\  --max <n>        Max pages to crawl (default: 100)
            \\  --delay <ms>     Delay between requests in ms (default: 1000)
            \\  --output <file>  Output file prefix (default: results)
            \\  --format <fmt>   Output format: json, csv, both (default: json)
            \\
        , .{});
        return;
    }

    const start_url = args[1];

    // Parse options
    var max_depth: usize = 3;
    var max_pages: usize = 100;
    var delay_ms: u64 = 1000;
    var output_prefix: []const u8 = "results";
    var format: []const u8 = "json";

    var i: usize = 2;
    while (i < args.len) : (i += 1) {
        const arg = args[i];
        if (std.mem.eql(u8, arg, "--depth") and i + 1 < args.len) {
            i += 1;
            max_depth = std.fmt.parseInt(usize, args[i], 10) catch 3;
        } else if (std.mem.eql(u8, arg, "--max") and i + 1 < args.len) {
            i += 1;
            max_pages = std.fmt.parseInt(usize, args[i], 10) catch 100;
        } else if (std.mem.eql(u8, arg, "--delay") and i + 1 < args.len) {
            i += 1;
            delay_ms = std.fmt.parseInt(u64, args[i], 10) catch 1000;
        } else if (std.mem.eql(u8, arg, "--output") and i + 1 < args.len) {
            i += 1;
            output_prefix = args[i];
        } else if (std.mem.eql(u8, arg, "--format") and i + 1 < args.len) {
            i += 1;
            format = args[i];
        }
    }

    std.log.info("Starting scraper:", .{});
    std.log.info("  URL: {s}", .{start_url});
    std.log.info("  Max depth: {}", .{max_depth});
    std.log.info("  Max pages: {}", .{max_pages});
    std.log.info("  Delay: {}ms", .{delay_ms});

    // สร้าง crawler
    var crawler = crawler_mod.Crawler.init(allocator, .{
        .max_depth = max_depth,
        .max_pages = max_pages,
        .delay_ms = delay_ms,
        .same_domain_only = true,
        .user_agent = "ZigScraper/1.0 (Educational Bot)",
    });
    defer crawler.deinit();

    // เริ่ม crawl
    const start_time = std.time.milliTimestamp();
    crawler.crawl(start_url) catch |err| {
        std.log.err("Crawl failed: {}", .{err});
        return err;
    };
    const elapsed = std.time.milliTimestamp() - start_time;

    std.log.info("Crawl completed in {}ms", .{elapsed});
    std.log.info("Pages crawled: {}", .{crawler.results.items.len});

    // Export results
    var exporter = exporter_mod.Exporter.init(allocator);

    if (std.mem.eql(u8, format, "json") or std.mem.eql(u8, format, "both")) {
        const json_path = try std.fmt.allocPrint(allocator, "{s}.json", .{output_prefix});
        defer allocator.free(json_path);
        try exporter.saveJson(crawler.results.items, json_path);
    }

    if (std.mem.eql(u8, format, "csv") or std.mem.eql(u8, format, "both")) {
        const csv_path = try std.fmt.allocPrint(allocator, "{s}.csv", .{output_prefix});
        defer allocator.free(csv_path);
        try exporter.saveCsv(crawler.results.items, csv_path);
    }

    // สรุปผล
    std.debug.print("\n=== Crawl Summary ===\n", .{});
    std.debug.print("Pages crawled: {}\n", .{crawler.results.items.len});
    std.debug.print("Time elapsed: {d:.1}s\n", .{@as(f64, @floatFromInt(elapsed)) / 1000.0});

    var total_links: usize = 0;
    for (crawler.results.items) |result| {
        total_links += result.links.len;
    }
    std.debug.print("Total links found: {}\n", .{total_links});
    std.debug.print("Output saved to: {s}.*\n", .{output_prefix});
}

test "HTML parser basic" {
    const allocator = std.testing.allocator;
    const html =
        \\<!DOCTYPE html>
        \\<html>
        \\<head><title>Test Page</title></head>
        \\<body>
        \\  <h1>Hello World</h1>
        \\  <p>This is a test.</p>
        \\  <a href="/about">About</a>
        \\  <a href="https://example.com">External</a>
        \\</body>
        \\</html>
    ;

    var doc = try html_mod.HtmlDocument.parse(allocator, html);
    defer doc.deinit();

    const title = doc.querySelector("title");
    try std.testing.expect(title != null);

    const links = try doc.querySelectorAll("a");
    defer links.deinit();
    try std.testing.expect(links.items.len == 2);
}

test "Robots.txt parsing" {
    const allocator = std.testing.allocator;
    const content =
        \\User-agent: *
        \\Disallow: /admin/
        \\Disallow: /private/
        \\Allow: /admin/public/
        \\Crawl-delay: 1
        \\
        \\User-agent: ZigBot
        \\Disallow: /secret/
    ;

    var robots = try robots_mod.RobotsTxt.parse(allocator, content);
    defer robots.deinit();

    try std.testing.expect(!robots.isAllowed("*", "/admin/dashboard"));
    try std.testing.expect(robots.isAllowed("*", "/admin/public/"));
    try std.testing.expect(robots.isAllowed("*", "/public/page"));
    try std.testing.expect(!robots.isAllowed("ZigBot", "/secret/data"));
}
```

---

## สรุปโปรเจกต์ Web Scraper

| ความสามารถ | รายละเอียด |
|-------------|-----------|
| HTTP Client | ใช้ std.http, รองรับ redirect |
| HTML Parser | Tokenizer + DOM tree |
| Selectors | Tag, Class, ID, Attribute |
| Link Extraction | Absolute URL resolution |
| Crawling | BFS, configurable depth |
| Rate Limiting | Token bucket, per-domain |
| Robots.txt | Full parsing + rule matching |
| Data Extraction | Title, meta, headings, images |
| Export | JSON + CSV formats |

### การรันโปรเจกต์

```bash
# Build
zig build

# Scrape หน้าเว็บ
./zig-out/bin/scraper https://example.com --depth 2 --max 50

# Export เป็น CSV
./zig-out/bin/scraper https://example.com --format csv --output data

# Run tests
zig build test
```
