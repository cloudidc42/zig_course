# Part 91: DSL (Domain Specific Language) ใน Zig

## Steps 901-910

---

## Step 901: DSL Concept และ Use Cases

### DSL คืออะไร?

Domain Specific Language (DSL) คือภาษาโปรแกรมมิ่งที่ออกแบบมาเฉพาะสำหรับโดเมนหรือปัญหาเฉพาะด้าน แทนที่จะเป็น General Purpose Language (GPL) ที่ใช้ได้หลากหลาย

### ประเภทของ DSL

1. **External DSL** - ภาษาที่แยกต่างหากจากภาษาหลัก เช่น SQL, HTML, CSS, Regex
2. **Internal DSL** - DSL ที่สร้างขึ้นภายในภาษาหลัก โดยใช้ syntax และ features ของภาษานั้น

### ทำไมต้องใช้ DSL?

```
ข้อดีของ DSL:
- ความชัดเจน: โค้ดอ่านง่ายขึ้นสำหรับ domain expert
- ความปลอดภัย: จำกัด operations ที่ทำได้
- ความถูกต้อง: บังคับ constraints ตั้งแต่ compile time
- Productivity: เขียนโค้ดน้อยลงสำหรับงานที่ซ้ำๆ
```

### Use Cases ของ DSL ใน Zig

```zig
// ตัวอย่าง use cases:
// 1. Query Builder - สร้าง SQL queries
// 2. Configuration - กำหนด config แบบ type-safe
// 3. State Machine - นิยาม state transitions
// 4. Build System - กำหนด build rules
// 5. Testing - เขียน test cases แบบ declarative

const std = @import("std");

// ตัวอย่าง DSL อย่างง่าย - Configuration DSL
pub fn main() !void {
    // แทนที่จะเขียนแบบนี้:
    var config_verbose = false;
    var config_port: u16 = 8080;
    var config_host = "localhost";
    _ = config_verbose;
    _ = config_port;
    _ = config_host;

    // เราสามารถใช้ DSL แบบนี้:
    // const config = Config.new()
    //     .verbose(false)
    //     .port(8080)
    //     .host("localhost")
    //     .build();
    
    std.debug.print("DSL concept demonstrated\n", .{});
}
```

### ลักษณะของ DSL ที่ดี

1. **Fluency** - อ่านเหมือนภาษาธรรมชาติ
2. **Expressiveness** - แสดง intent ได้ชัดเจน
3. **Safety** - ป้องกัน invalid states
4. **Composability** - ประกอบกันได้

---

## Step 902: Internal DSL ด้วย Method Chaining

### Method Chaining คืออะไร?

Method chaining คือเทคนิคที่แต่ละ method return `self` ทำให้เรียก methods ต่อเนื่องกันได้

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Builder Pattern ด้วย Method Chaining
pub const HttpRequest = struct {
    allocator: Allocator,
    method: []const u8,
    url: []const u8,
    headers: std.ArrayList(Header),
    body: ?[]const u8,
    timeout_ms: u32,

    pub const Header = struct {
        name: []const u8,
        value: []const u8,
    };

    pub fn init(allocator: Allocator) HttpRequest {
        return HttpRequest{
            .allocator = allocator,
            .method = "GET",
            .url = "",
            .headers = std.ArrayList(Header).init(allocator),
            .body = null,
            .timeout_ms = 30000,
        };
    }

    pub fn deinit(self: *HttpRequest) void {
        self.headers.deinit();
    }

    // Method chaining - return pointer to self
    pub fn method(self: *HttpRequest, m: []const u8) *HttpRequest {
        self.method = m;
        return self;
    }

    pub fn url(self: *HttpRequest, u: []const u8) *HttpRequest {
        self.url = u;
        return self;
    }

    pub fn header(self: *HttpRequest, name: []const u8, value: []const u8) !*HttpRequest {
        try self.headers.append(Header{ .name = name, .value = value });
        return self;
    }

    pub fn body(self: *HttpRequest, b: []const u8) *HttpRequest {
        self.body = b;
        return self;
    }

    pub fn timeout(self: *HttpRequest, ms: u32) *HttpRequest {
        self.timeout_ms = ms;
        return self;
    }

    pub fn send(self: *HttpRequest) !void {
        std.debug.print("Sending {s} request to {s}\n", .{ self.method, self.url });
        std.debug.print("Headers count: {}\n", .{self.headers.items.len});
        if (self.body) |b| {
            std.debug.print("Body: {s}\n", .{b});
        }
    }
};

// ตัวอย่างการใช้งาน DSL แบบ Method Chaining
pub fn demonstrateMethodChaining() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var request = HttpRequest.init(allocator);
    defer request.deinit();

    // DSL ที่อ่านง่าย
    _ = request.method("POST");
    _ = request.url("https://api.example.com/users");
    _ = try request.header("Content-Type", "application/json");
    _ = try request.header("Authorization", "Bearer token123");
    _ = request.body("{\"name\": \"John\"}");
    _ = request.timeout(5000);
    try request.send();
}
```

### Fluent Interface Pattern

```zig
// Query DSL ด้วย Fluent Interface
pub const QueryBuilder = struct {
    allocator: Allocator,
    table: []const u8,
    conditions: std.ArrayList([]const u8),
    columns: std.ArrayList([]const u8),
    order_by: ?[]const u8,
    limit_val: ?u32,
    offset_val: ?u32,

    pub fn init(allocator: Allocator) QueryBuilder {
        return QueryBuilder{
            .allocator = allocator,
            .table = "",
            .conditions = std.ArrayList([]const u8).init(allocator),
            .columns = std.ArrayList([]const u8).init(allocator),
            .order_by = null,
            .limit_val = null,
            .offset_val = null,
        };
    }

    pub fn deinit(self: *QueryBuilder) void {
        self.conditions.deinit();
        self.columns.deinit();
    }

    pub fn from(self: *QueryBuilder, table_name: []const u8) *QueryBuilder {
        self.table = table_name;
        return self;
    }

    pub fn select(self: *QueryBuilder, column: []const u8) !*QueryBuilder {
        try self.columns.append(column);
        return self;
    }

    pub fn where(self: *QueryBuilder, condition: []const u8) !*QueryBuilder {
        try self.conditions.append(condition);
        return self;
    }

    pub fn orderBy(self: *QueryBuilder, column: []const u8) *QueryBuilder {
        self.order_by = column;
        return self;
    }

    pub fn limit(self: *QueryBuilder, n: u32) *QueryBuilder {
        self.limit_val = n;
        return self;
    }

    pub fn offset(self: *QueryBuilder, n: u32) *QueryBuilder {
        self.offset_val = n;
        return self;
    }

    pub fn build(self: *QueryBuilder) ![]const u8 {
        var result = std.ArrayList(u8).init(self.allocator);
        errdefer result.deinit();

        try result.appendSlice("SELECT ");
        
        if (self.columns.items.len == 0) {
            try result.append('*');
        } else {
            for (self.columns.items, 0..) |col, i| {
                if (i > 0) try result.appendSlice(", ");
                try result.appendSlice(col);
            }
        }

        try result.appendSlice(" FROM ");
        try result.appendSlice(self.table);

        if (self.conditions.items.len > 0) {
            try result.appendSlice(" WHERE ");
            for (self.conditions.items, 0..) |cond, i| {
                if (i > 0) try result.appendSlice(" AND ");
                try result.appendSlice(cond);
            }
        }

        if (self.order_by) |ob| {
            try result.appendSlice(" ORDER BY ");
            try result.appendSlice(ob);
        }

        if (self.limit_val) |l| {
            var buf: [32]u8 = undefined;
            const str = try std.fmt.bufPrint(&buf, " LIMIT {}", .{l});
            try result.appendSlice(str);
        }

        if (self.offset_val) |o| {
            var buf: [32]u8 = undefined;
            const str = try std.fmt.bufPrint(&buf, " OFFSET {}", .{o});
            try result.appendSlice(str);
        }

        return result.toOwnedSlice();
    }
};
```

---

## Step 903: Query Builder DSL

### SQL Query Builder แบบ Type-Safe

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Type-safe column definition
pub fn Column(comptime T: type) type {
    return struct {
        name: []const u8,
        table: []const u8,

        const Self = @This();
        pub const ValueType = T;

        pub fn eq(self: Self, value: T) Condition {
            _ = self;
            _ = value;
            return Condition{ .op = .Equal, .column = self.name };
        }

        pub fn gt(self: Self, value: T) Condition {
            _ = value;
            return Condition{ .op = .GreaterThan, .column = self.name };
        }

        pub fn lt(self: Self, value: T) Condition {
            _ = value;
            return Condition{ .op = .LessThan, .column = self.name };
        }

        pub fn like(self: Self, pattern: []const u8) Condition {
            _ = pattern;
            return Condition{ .op = .Like, .column = self.name };
        }
    };
}

pub const Operator = enum {
    Equal,
    NotEqual,
    GreaterThan,
    LessThan,
    GreaterThanOrEqual,
    LessThanOrEqual,
    Like,
    In,
    IsNull,
    IsNotNull,
};

pub const Condition = struct {
    op: Operator,
    column: []const u8,

    pub fn and_(self: Condition, other: Condition) CompositeCondition {
        return CompositeCondition{
            .left = self,
            .right = other,
            .logical = .And,
        };
    }

    pub fn or_(self: Condition, other: Condition) CompositeCondition {
        return CompositeCondition{
            .left = self,
            .right = other,
            .logical = .Or,
        };
    }
};

pub const CompositeCondition = struct {
    left: Condition,
    right: Condition,
    logical: enum { And, Or },
};

// Schema definition DSL
pub fn Table(comptime name: []const u8, comptime fields: anytype) type {
    return struct {
        pub const table_name = name;
        pub const schema = fields;

        pub fn query(allocator: Allocator) SelectQuery {
            return SelectQuery.init(allocator, name);
        }
    };
}

pub const SelectQuery = struct {
    allocator: Allocator,
    table_name: []const u8,
    where_clauses: std.ArrayList([]const u8),
    selected_columns: std.ArrayList([]const u8),
    joins: std.ArrayList(JoinClause),
    group_by_cols: std.ArrayList([]const u8),
    having_clause: ?[]const u8,
    order_cols: std.ArrayList(OrderClause),
    limit_n: ?u32,
    offset_n: ?u32,
    distinct: bool,

    pub const JoinClause = struct {
        join_type: []const u8,
        table: []const u8,
        condition: []const u8,
    };

    pub const OrderClause = struct {
        column: []const u8,
        direction: enum { Asc, Desc },
    };

    pub fn init(allocator: Allocator, table: []const u8) SelectQuery {
        return SelectQuery{
            .allocator = allocator,
            .table_name = table,
            .where_clauses = std.ArrayList([]const u8).init(allocator),
            .selected_columns = std.ArrayList([]const u8).init(allocator),
            .joins = std.ArrayList(JoinClause).init(allocator),
            .group_by_cols = std.ArrayList([]const u8).init(allocator),
            .having_clause = null,
            .order_cols = std.ArrayList(OrderClause).init(allocator),
            .limit_n = null,
            .offset_n = null,
            .distinct = false,
        };
    }

    pub fn deinit(self: *SelectQuery) void {
        self.where_clauses.deinit();
        self.selected_columns.deinit();
        self.joins.deinit();
        self.group_by_cols.deinit();
        self.order_cols.deinit();
    }

    pub fn selectDistinct(self: *SelectQuery) *SelectQuery {
        self.distinct = true;
        return self;
    }

    pub fn col(self: *SelectQuery, column: []const u8) !*SelectQuery {
        try self.selected_columns.append(column);
        return self;
    }

    pub fn whereStr(self: *SelectQuery, condition: []const u8) !*SelectQuery {
        try self.where_clauses.append(condition);
        return self;
    }

    pub fn join(self: *SelectQuery, table: []const u8, condition: []const u8) !*SelectQuery {
        try self.joins.append(JoinClause{
            .join_type = "INNER JOIN",
            .table = table,
            .condition = condition,
        });
        return self;
    }

    pub fn leftJoin(self: *SelectQuery, table: []const u8, condition: []const u8) !*SelectQuery {
        try self.joins.append(JoinClause{
            .join_type = "LEFT JOIN",
            .table = table,
            .condition = condition,
        });
        return self;
    }

    pub fn groupBy(self: *SelectQuery, column: []const u8) !*SelectQuery {
        try self.group_by_cols.append(column);
        return self;
    }

    pub fn having(self: *SelectQuery, condition: []const u8) *SelectQuery {
        self.having_clause = condition;
        return self;
    }

    pub fn orderAsc(self: *SelectQuery, column: []const u8) !*SelectQuery {
        try self.order_cols.append(OrderClause{ .column = column, .direction = .Asc });
        return self;
    }

    pub fn orderDesc(self: *SelectQuery, column: []const u8) !*SelectQuery {
        try self.order_cols.append(OrderClause{ .column = column, .direction = .Desc });
        return self;
    }

    pub fn limitTo(self: *SelectQuery, n: u32) *SelectQuery {
        self.limit_n = n;
        return self;
    }

    pub fn offsetBy(self: *SelectQuery, n: u32) *SelectQuery {
        self.offset_n = n;
        return self;
    }

    pub fn toSql(self: *SelectQuery) ![]const u8 {
        var sql = std.ArrayList(u8).init(self.allocator);
        errdefer sql.deinit();

        try sql.appendSlice("SELECT ");
        if (self.distinct) try sql.appendSlice("DISTINCT ");

        if (self.selected_columns.items.len == 0) {
            try sql.append('*');
        } else {
            for (self.selected_columns.items, 0..) |c, i| {
                if (i > 0) try sql.appendSlice(", ");
                try sql.appendSlice(c);
            }
        }

        try sql.appendSlice(" FROM ");
        try sql.appendSlice(self.table_name);

        for (self.joins.items) |j| {
            try sql.append(' ');
            try sql.appendSlice(j.join_type);
            try sql.append(' ');
            try sql.appendSlice(j.table);
            try sql.appendSlice(" ON ");
            try sql.appendSlice(j.condition);
        }

        if (self.where_clauses.items.len > 0) {
            try sql.appendSlice(" WHERE ");
            for (self.where_clauses.items, 0..) |w, i| {
                if (i > 0) try sql.appendSlice(" AND ");
                try sql.appendSlice(w);
            }
        }

        if (self.group_by_cols.items.len > 0) {
            try sql.appendSlice(" GROUP BY ");
            for (self.group_by_cols.items, 0..) |g, i| {
                if (i > 0) try sql.appendSlice(", ");
                try sql.appendSlice(g);
            }
        }

        if (self.having_clause) |h| {
            try sql.appendSlice(" HAVING ");
            try sql.appendSlice(h);
        }

        if (self.order_cols.items.len > 0) {
            try sql.appendSlice(" ORDER BY ");
            for (self.order_cols.items, 0..) |o, i| {
                if (i > 0) try sql.appendSlice(", ");
                try sql.appendSlice(o.column);
                switch (o.direction) {
                    .Asc => try sql.appendSlice(" ASC"),
                    .Desc => try sql.appendSlice(" DESC"),
                }
            }
        }

        if (self.limit_n) |l| {
            var buf: [32]u8 = undefined;
            const s = try std.fmt.bufPrint(&buf, " LIMIT {}", .{l});
            try sql.appendSlice(s);
        }

        if (self.offset_n) |o| {
            var buf: [32]u8 = undefined;
            const s = try std.fmt.bufPrint(&buf, " OFFSET {}", .{o});
            try sql.appendSlice(s);
        }

        return sql.toOwnedSlice();
    }
};

// ตัวอย่างการใช้งาน
pub fn queryBuilderExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var q = SelectQuery.init(allocator, "users");
    defer q.deinit();

    _ = try q.col("users.id");
    _ = try q.col("users.name");
    _ = try q.col("orders.total");
    _ = try q.leftJoin("orders", "users.id = orders.user_id");
    _ = try q.whereStr("users.active = 1");
    _ = try q.whereStr("orders.total > 100");
    _ = try q.orderDesc("orders.total");
    _ = q.limitTo(10);
    _ = q.offsetBy(20);

    const sql = try q.toSql();
    defer allocator.free(sql);

    std.debug.print("Generated SQL:\n{s}\n", .{sql});
}
```

---

## Step 904: Configuration DSL

### Type-Safe Configuration DSL

```zig
const std = @import("std");

// Configuration schema using comptime types
pub fn Config(comptime schema: type) type {
    return struct {
        data: schema,

        const Self = @This();

        pub fn init(defaults: schema) Self {
            return Self{ .data = defaults };
        }

        pub fn get(self: Self, comptime field: []const u8) @TypeOf(@field(self.data, field)) {
            return @field(self.data, field);
        }

        pub fn validate(self: Self) !void {
            const info = @typeInfo(schema);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        const val = @field(self.data, field.name);
                        _ = val;
                        // Custom validation logic can be added here
                    }
                },
                else => {},
            }
        }
    };
}

// Server configuration DSL
pub const ServerConfig = struct {
    host: []const u8 = "localhost",
    port: u16 = 8080,
    max_connections: u32 = 1000,
    timeout_seconds: u32 = 30,
    tls_enabled: bool = false,
    tls_cert_path: ?[]const u8 = null,
    tls_key_path: ?[]const u8 = null,
    log_level: LogLevel = .Info,
    worker_threads: u32 = 4,

    pub const LogLevel = enum {
        Debug,
        Info,
        Warning,
        Error,
        Critical,
    };
};

// Builder สำหรับ configuration
pub const ServerConfigBuilder = struct {
    config: ServerConfig,

    pub fn init() ServerConfigBuilder {
        return ServerConfigBuilder{
            .config = ServerConfig{},
        };
    }

    pub fn host(self: *ServerConfigBuilder, h: []const u8) *ServerConfigBuilder {
        self.config.host = h;
        return self;
    }

    pub fn port(self: *ServerConfigBuilder, p: u16) *ServerConfigBuilder {
        self.config.port = p;
        return self;
    }

    pub fn maxConnections(self: *ServerConfigBuilder, n: u32) *ServerConfigBuilder {
        self.config.max_connections = n;
        return self;
    }

    pub fn timeout(self: *ServerConfigBuilder, secs: u32) *ServerConfigBuilder {
        self.config.timeout_seconds = secs;
        return self;
    }

    pub fn withTls(self: *ServerConfigBuilder, cert: []const u8, key: []const u8) *ServerConfigBuilder {
        self.config.tls_enabled = true;
        self.config.tls_cert_path = cert;
        self.config.tls_key_path = key;
        return self;
    }

    pub fn logLevel(self: *ServerConfigBuilder, level: ServerConfig.LogLevel) *ServerConfigBuilder {
        self.config.log_level = level;
        return self;
    }

    pub fn workers(self: *ServerConfigBuilder, n: u32) *ServerConfigBuilder {
        self.config.worker_threads = n;
        return self;
    }

    pub fn build(self: *ServerConfigBuilder) !ServerConfig {
        // Validate configuration
        if (self.config.tls_enabled) {
            if (self.config.tls_cert_path == null) {
                return error.MissingTlsCert;
            }
            if (self.config.tls_key_path == null) {
                return error.MissingTlsKey;
            }
        }
        if (self.config.port == 0) {
            return error.InvalidPort;
        }
        return self.config;
    }
};

// Environment-based configuration
pub fn loadConfig(comptime T: type, env_prefix: []const u8) T {
    var config = T{};
    const info = @typeInfo(T);
    _ = env_prefix;

    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                // ในระบบจริงจะอ่านจาก environment variables
                _ = @field(config, field.name);
            }
        },
        else => {},
    }

    return config;
}

// TOML-like configuration DSL
pub const ConfigSection = struct {
    name: []const u8,
    entries: std.StringHashMap(ConfigValue),
    allocator: std.mem.Allocator,

    pub const ConfigValue = union(enum) {
        string: []const u8,
        integer: i64,
        float: f64,
        boolean: bool,
        array: []ConfigValue,
    };

    pub fn init(allocator: std.mem.Allocator, section_name: []const u8) ConfigSection {
        return ConfigSection{
            .name = section_name,
            .entries = std.StringHashMap(ConfigValue).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *ConfigSection) void {
        self.entries.deinit();
    }

    pub fn set(self: *ConfigSection, key: []const u8, value: ConfigValue) !void {
        try self.entries.put(key, value);
    }

    pub fn getString(self: *ConfigSection, key: []const u8) ?[]const u8 {
        if (self.entries.get(key)) |v| {
            switch (v) {
                .string => |s| return s,
                else => return null,
            }
        }
        return null;
    }

    pub fn getInt(self: *ConfigSection, key: []const u8) ?i64 {
        if (self.entries.get(key)) |v| {
            switch (v) {
                .integer => |i| return i,
                else => return null,
            }
        }
        return null;
    }
};

pub fn configDslExample() !void {
    // ตัวอย่างการสร้าง server configuration
    var builder = ServerConfigBuilder.init();
    const config = try builder
        .host("0.0.0.0")
        .port(9000)
        .maxConnections(5000)
        .timeout(60)
        .logLevel(.Debug)
        .workers(8)
        .build();

    std.debug.print("Server: {s}:{}\n", .{ config.host, config.port });
    std.debug.print("Max connections: {}\n", .{config.max_connections});
    std.debug.print("Workers: {}\n", .{config.worker_threads});
}
```

---

## Step 905: Parser Combinator DSL

### Parser Combinator เพื่อสร้าง Parser แบบ Declarative

```zig
const std = @import("std");

// Parser result type
pub fn ParseResult(comptime T: type) type {
    return struct {
        value: T,
        remaining: []const u8,
    };
}

// Parser type - function ที่รับ string และ return result หรือ error
pub fn Parser(comptime T: type) type {
    return *const fn (input: []const u8) ?ParseResult(T);
}

// Primitive parsers
pub fn charParser(comptime c: u8) Parser(u8) {
    return struct {
        fn parse(input: []const u8) ?ParseResult(u8) {
            if (input.len > 0 and input[0] == c) {
                return ParseResult(u8){ .value = c, .remaining = input[1..] };
            }
            return null;
        }
    }.parse;
}

pub fn digitParser(input: []const u8) ?ParseResult(u8) {
    if (input.len > 0 and input[0] >= '0' and input[0] <= '9') {
        return ParseResult(u8){ .value = input[0] - '0', .remaining = input[1..] };
    }
    return null;
}

pub fn letterParser(input: []const u8) ?ParseResult(u8) {
    if (input.len > 0 and ((input[0] >= 'a' and input[0] <= 'z') or
        (input[0] >= 'A' and input[0] <= 'Z')))
    {
        return ParseResult(u8){ .value = input[0], .remaining = input[1..] };
    }
    return null;
}

pub fn whitespaceParser(input: []const u8) ?ParseResult(void) {
    var i: usize = 0;
    while (i < input.len and (input[i] == ' ' or input[i] == '\t' or input[i] == '\n')) {
        i += 1;
    }
    if (i > 0) {
        return ParseResult(void){ .value = {}, .remaining = input[i..] };
    }
    return null;
}

// DSL สำหรับ combining parsers
pub const ParserDsl = struct {
    // Parse a literal string
    pub fn literal(comptime s: []const u8) Parser([]const u8) {
        return struct {
            fn parse(input: []const u8) ?ParseResult([]const u8) {
                if (std.mem.startsWith(u8, input, s)) {
                    return ParseResult([]const u8){
                        .value = s,
                        .remaining = input[s.len..],
                    };
                }
                return null;
            }
        }.parse;
    }

    // Parse one or more repetitions
    pub fn many1(comptime T: type, parser: Parser(T), allocator: std.mem.Allocator) !?ParseResult([]T) {
        _ = parser;
        _ = allocator;
        // Simplified implementation
        return null;
    }

    // Optional parser
    pub fn optional(comptime T: type, parser: Parser(T)) Parser(?T) {
        _ = parser;
        return struct {
            fn parse(input: []const u8) ?ParseResult(?T) {
                return ParseResult(?T){ .value = null, .remaining = input };
            }
        }.parse;
    }
};

// JSON-like DSL Parser
pub const JsonValue = union(enum) {
    null_val: void,
    bool_val: bool,
    number: f64,
    string: []const u8,
    array: []JsonValue,
    object: std.StringHashMap(JsonValue),
};

// Simple expression parser DSL
pub const ExprParser = struct {
    pub const Expr = union(enum) {
        num: f64,
        add: struct { left: *Expr, right: *Expr },
        mul: struct { left: *Expr, right: *Expr },
        neg: *Expr,
    };

    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) ExprParser {
        return ExprParser{ .allocator = allocator };
    }

    pub fn parseNumber(self: ExprParser, input: []const u8) ?struct { expr: Expr, rest: []const u8 } {
        _ = self;
        var i: usize = 0;
        var has_dot = false;

        if (i < input.len and input[i] == '-') i += 1;

        while (i < input.len and input[i] >= '0' and input[i] <= '9') {
            i += 1;
        }

        if (i < input.len and input[i] == '.') {
            has_dot = true;
            i += 1;
            while (i < input.len and input[i] >= '0' and input[i] <= '9') {
                i += 1;
            }
        }

        if (i == 0 or (i == 1 and input[0] == '-')) return null;
        _ = has_dot;

        const num_str = input[0..i];
        const num = std.fmt.parseFloat(f64, num_str) catch return null;
        
        return .{ .expr = Expr{ .num = num }, .rest = input[i..] };
    }
};
```

---

## Step 906: Template DSL

### Template Engine ด้วย Comptime

```zig
const std = @import("std");

// Compile-time template processing
pub fn Template(comptime template_str: []const u8) type {
    return struct {
        pub fn render(writer: anytype, context: anytype) !void {
            comptime var i = 0;
            inline while (i < template_str.len) {
                if (template_str[i] == '{' and i + 1 < template_str.len and template_str[i + 1] == '{') {
                    // Find closing }}
                    comptime var j = i + 2;
                    inline while (j < template_str.len) : (j += 1) {
                        if (template_str[j] == '}' and j + 1 < template_str.len and template_str[j + 1] == '}') {
                            break;
                        }
                    }

                    const var_name = comptime template_str[i + 2 .. j];
                    const trimmed = comptime std.mem.trim(u8, var_name, " ");

                    // Look up field in context
                    if (@hasField(@TypeOf(context), trimmed)) {
                        const val = @field(context, trimmed);
                        try writer.print("{any}", .{val});
                    }

                    i = j + 2;
                } else {
                    try writer.writeByte(template_str[i]);
                    i += 1;
                }
            }
        }
    };
}

// Runtime template engine
pub const TemplateEngine = struct {
    allocator: std.mem.Allocator,
    templates: std.StringHashMap([]const u8),

    pub fn init(allocator: std.mem.Allocator) TemplateEngine {
        return TemplateEngine{
            .allocator = allocator,
            .templates = std.StringHashMap([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *TemplateEngine) void {
        self.templates.deinit();
    }

    pub fn register(self: *TemplateEngine, name: []const u8, template: []const u8) !void {
        try self.templates.put(name, template);
    }

    pub fn render(self: *TemplateEngine, name: []const u8, context: std.StringHashMap([]const u8)) ![]const u8 {
        const template = self.templates.get(name) orelse return error.TemplateNotFound;
        var output = std.ArrayList(u8).init(self.allocator);
        errdefer output.deinit();

        var i: usize = 0;
        while (i < template.len) {
            if (i + 1 < template.len and template[i] == '{' and template[i + 1] == '{') {
                // Find variable name
                const start = i + 2;
                var end = start;
                while (end + 1 < template.len and !(template[end] == '}' and template[end + 1] == '}')) {
                    end += 1;
                }
                const var_name = std.mem.trim(u8, template[start..end], " ");
                if (context.get(var_name)) |value| {
                    try output.appendSlice(value);
                }
                i = end + 2;
            } else if (i + 1 < template.len and template[i] == '{' and template[i + 1] == '%') {
                // Handle block tags like {% if %}, {% for %}
                // Simplified - just skip for now
                var end = i + 2;
                while (end + 1 < template.len and !(template[end] == '%' and template[end + 1] == '}')) {
                    end += 1;
                }
                i = end + 2;
            } else {
                try output.append(template[i]);
                i += 1;
            }
        }

        return output.toOwnedSlice();
    }
};

// HTML Template DSL
pub const HtmlBuilder = struct {
    allocator: std.mem.Allocator,
    content: std.ArrayList(u8),

    pub fn init(allocator: std.mem.Allocator) HtmlBuilder {
        return HtmlBuilder{
            .allocator = allocator,
            .content = std.ArrayList(u8).init(allocator),
        };
    }

    pub fn deinit(self: *HtmlBuilder) void {
        self.content.deinit();
    }

    pub fn tag(self: *HtmlBuilder, name: []const u8, attrs: []const u8, children: []const u8) !*HtmlBuilder {
        try self.content.append('<');
        try self.content.appendSlice(name);
        if (attrs.len > 0) {
            try self.content.append(' ');
            try self.content.appendSlice(attrs);
        }
        try self.content.append('>');
        try self.content.appendSlice(children);
        try self.content.appendSlice("</");
        try self.content.appendSlice(name);
        try self.content.append('>');
        return self;
    }

    pub fn text(self: *HtmlBuilder, t: []const u8) !*HtmlBuilder {
        try self.content.appendSlice(t);
        return self;
    }

    pub fn build(self: *HtmlBuilder) []const u8 {
        return self.content.items;
    }
};

pub fn templateDslExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Template engine example
    var engine = TemplateEngine.init(allocator);
    defer engine.deinit();

    try engine.register("greeting", "Hello, {{ name }}! You are {{ age }} years old.");

    var ctx = std.StringHashMap([]const u8).init(allocator);
    defer ctx.deinit();
    try ctx.put("name", "Alice");
    try ctx.put("age", "30");

    const result = try engine.render("greeting", ctx);
    defer allocator.free(result);

    std.debug.print("{s}\n", .{result});

    // HTML Builder example
    var html = HtmlBuilder.init(allocator);
    defer html.deinit();

    _ = try html.tag("div", "class=\"container\"", "Hello World");
    std.debug.print("{s}\n", .{html.build()});
}
```

---

## Step 907: State Machine DSL

### State Machine Definition ด้วย DSL

```zig
const std = @import("std");

// State machine DSL
pub fn StateMachine(comptime StateType: type, comptime EventType: type) type {
    return struct {
        const Self = @This();
        const TransitionFn = *const fn (state: StateType, event: EventType) ?StateType;
        const ActionFn = *const fn (from: StateType, to: StateType, event: EventType) void;

        current_state: StateType,
        transitions: std.ArrayList(TransitionRule),
        on_enter: std.AutoHashMap(StateType, ActionFn),
        on_exit: std.AutoHashMap(StateType, ActionFn),
        allocator: std.mem.Allocator,

        pub const TransitionRule = struct {
            from: StateType,
            event: EventType,
            to: StateType,
            guard: ?*const fn (state: StateType, event: EventType) bool,
            action: ?ActionFn,
        };

        pub fn init(allocator: std.mem.Allocator, initial: StateType) Self {
            return Self{
                .current_state = initial,
                .transitions = std.ArrayList(TransitionRule).init(allocator),
                .on_enter = std.AutoHashMap(StateType, ActionFn).init(allocator),
                .on_exit = std.AutoHashMap(StateType, ActionFn).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.transitions.deinit();
            self.on_enter.deinit();
            self.on_exit.deinit();
        }

        // DSL method สำหรับ define transition
        pub fn on(self: *Self, from: StateType, event: EventType) TransitionBuilder {
            return TransitionBuilder{
                .sm = self,
                .from = from,
                .event = event,
                .guard = null,
                .action = null,
            };
        }

        pub fn send(self: *Self, event: EventType) !bool {
            for (self.transitions.items) |rule| {
                if (rule.from == self.current_state and rule.event == event) {
                    // Check guard
                    if (rule.guard) |guard| {
                        if (!guard(self.current_state, event)) continue;
                    }

                    const old_state = self.current_state;
                    
                    // Execute exit action
                    if (self.on_exit.get(old_state)) |exit_fn| {
                        exit_fn(old_state, rule.to, event);
                    }

                    // Execute transition action
                    if (rule.action) |action| {
                        action(old_state, rule.to, event);
                    }

                    self.current_state = rule.to;

                    // Execute enter action
                    if (self.on_enter.get(rule.to)) |enter_fn| {
                        enter_fn(old_state, rule.to, event);
                    }

                    return true;
                }
            }
            return false;
        }

        pub fn state(self: Self) StateType {
            return self.current_state;
        }

        pub const TransitionBuilder = struct {
            sm: *Self,
            from: StateType,
            event: EventType,
            guard: ?*const fn (StateType, EventType) bool,
            action: ?ActionFn,

            pub fn when(self: TransitionBuilder, guard_fn: *const fn (StateType, EventType) bool) TransitionBuilder {
                var b = self;
                b.guard = guard_fn;
                return b;
            }

            pub fn do(self: TransitionBuilder, action_fn: ActionFn) TransitionBuilder {
                var b = self;
                b.action = action_fn;
                return b;
            }

            pub fn goto(self: TransitionBuilder, to: StateType) !void {
                try self.sm.transitions.append(TransitionRule{
                    .from = self.from,
                    .event = self.event,
                    .to = to,
                    .guard = self.guard,
                    .action = self.action,
                });
            }
        };
    };
}

// ตัวอย่าง: Traffic Light State Machine
pub fn trafficLightExample() !void {
    const State = enum { Red, Yellow, Green };
    const Event = enum { Timer, Emergency };

    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const TrafficSM = StateMachine(State, Event);
    var sm = TrafficSM.init(allocator, .Red);
    defer sm.deinit();

    // Define transitions using DSL
    try sm.on(.Red, .Timer).goto(.Green);
    try sm.on(.Green, .Timer).goto(.Yellow);
    try sm.on(.Yellow, .Timer).goto(.Red);

    // Define emergency transitions
    try sm.on(.Green, .Emergency).goto(.Red);
    try sm.on(.Yellow, .Emergency).goto(.Red);

    std.debug.print("Initial state: {}\n", .{sm.state()});

    _ = try sm.send(.Timer);
    std.debug.print("After Timer: {}\n", .{sm.state()});

    _ = try sm.send(.Timer);
    std.debug.print("After Timer: {}\n", .{sm.state()});

    _ = try sm.send(.Emergency);
    std.debug.print("After Emergency: {}\n", .{sm.state()});
}
```

---

## Step 908: Build System DSL

### Build Configuration DSL

```zig
const std = @import("std");

// Build step type
pub const BuildStep = struct {
    name: []const u8,
    description: []const u8,
    dependencies: std.ArrayList([]const u8),
    command: ?[]const u8,
    inputs: std.ArrayList([]const u8),
    outputs: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, name: []const u8) BuildStep {
        return BuildStep{
            .name = name,
            .description = "",
            .dependencies = std.ArrayList([]const u8).init(allocator),
            .command = null,
            .inputs = std.ArrayList([]const u8).init(allocator),
            .outputs = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *BuildStep) void {
        self.dependencies.deinit();
        self.inputs.deinit();
        self.outputs.deinit();
    }
};

// Build graph DSL
pub const BuildGraph = struct {
    allocator: std.mem.Allocator,
    steps: std.StringHashMap(BuildStep),
    variables: std.StringHashMap([]const u8),

    pub fn init(allocator: std.mem.Allocator) BuildGraph {
        return BuildGraph{
            .allocator = allocator,
            .steps = std.StringHashMap(BuildStep).init(allocator),
            .variables = std.StringHashMap([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *BuildGraph) void {
        var it = self.steps.valueIterator();
        while (it.next()) |step| {
            step.deinit();
        }
        self.steps.deinit();
        self.variables.deinit();
    }

    // DSL สำหรับ define variables
    pub fn define(self: *BuildGraph, name: []const u8, value: []const u8) !*BuildGraph {
        try self.variables.put(name, value);
        return self;
    }

    // DSL สำหรับ create step
    pub fn step(self: *BuildGraph, name: []const u8) !StepBuilder {
        const s = BuildStep.init(self.allocator, name);
        try self.steps.put(name, s);
        return StepBuilder{
            .graph = self,
            .step_name = name,
        };
    }

    pub fn execute(self: *BuildGraph, target: []const u8) !void {
        // Topological sort and execute
        std.debug.print("Building target: {s}\n", .{target});
        if (self.steps.get(target)) |s| {
            std.debug.print("Executing step: {s}\n", .{s.name});
            std.debug.print("Description: {s}\n", .{s.description});
        }
    }

    pub const StepBuilder = struct {
        graph: *BuildGraph,
        step_name: []const u8,

        pub fn description(self: StepBuilder, desc: []const u8) StepBuilder {
            if (self.graph.steps.getPtr(self.step_name)) |s| {
                s.description = desc;
            }
            return self;
        }

        pub fn dependsOn(self: StepBuilder, dep: []const u8) !StepBuilder {
            if (self.graph.steps.getPtr(self.step_name)) |s| {
                try s.dependencies.append(dep);
            }
            return self;
        }

        pub fn input(self: StepBuilder, file: []const u8) !StepBuilder {
            if (self.graph.steps.getPtr(self.step_name)) |s| {
                try s.inputs.append(file);
            }
            return self;
        }

        pub fn output(self: StepBuilder, file: []const u8) !StepBuilder {
            if (self.graph.steps.getPtr(self.step_name)) |s| {
                try s.outputs.append(file);
            }
            return self;
        }

        pub fn run(self: StepBuilder, cmd: []const u8) StepBuilder {
            if (self.graph.steps.getPtr(self.step_name)) |s| {
                s.command = cmd;
            }
            return self;
        }
    };
};

pub fn buildSystemExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var build = BuildGraph.init(allocator);
    defer build.deinit();

    // Define build variables
    _ = try build.define("CC", "zig cc");
    _ = try build.define("CFLAGS", "-O2 -Wall");

    // Define build steps using DSL
    _ = (try build.step("compile"))
        .description("Compile source files")
        .run("$(CC) $(CFLAGS) -c src/main.c -o build/main.o");

    _ = try (try build.step("link"))
        .description("Link object files")
        .dependsOn("compile");

    _ = try (try build.step("test"))
        .description("Run tests")
        .dependsOn("compile");

    _ = try (try build.step("all"))
        .dependsOn("link")
        .dependsOn("test");

    try build.execute("all");
}
```

---

## Step 909: Testing DSL

### Testing Framework DSL

```zig
const std = @import("std");

// Test assertion DSL
pub const Expect = struct {
    value: anytype,
    test_name: []const u8,

    pub fn init(comptime T: type, value: T, name: []const u8) struct {
        val: T,
        name: []const u8,

        const Self = @This();

        pub fn toBe(self: Self, expected: T) !void {
            if (self.val != expected) {
                std.debug.print("FAIL [{s}]: expected {any}, got {any}\n", .{ self.name, expected, self.val });
                return error.AssertionFailed;
            }
            std.debug.print("PASS [{s}]\n", .{self.name});
        }

        pub fn toBeTrue(self: Self) !void {
            const bool_val = self.val;
            if (!bool_val) {
                std.debug.print("FAIL [{s}]: expected true\n", .{self.name});
                return error.AssertionFailed;
            }
            std.debug.print("PASS [{s}]\n", .{self.name});
        }

        pub fn toBeFalse(self: Self) !void {
            const bool_val = self.val;
            if (bool_val) {
                std.debug.print("FAIL [{s}]: expected false\n", .{self.name});
                return error.AssertionFailed;
            }
            std.debug.print("PASS [{s}]\n", .{self.name});
        }

        pub fn toBeGreaterThan(self: Self, threshold: T) !void {
            if (self.val <= threshold) {
                std.debug.print("FAIL [{s}]: expected > {any}, got {any}\n", .{ self.name, threshold, self.val });
                return error.AssertionFailed;
            }
            std.debug.print("PASS [{s}]\n", .{self.name});
        }

        pub fn toBeLessThan(self: Self, threshold: T) !void {
            if (self.val >= threshold) {
                std.debug.print("FAIL [{s}]: expected < {any}, got {any}\n", .{ self.name, threshold, self.val });
                return error.AssertionFailed;
            }
            std.debug.print("PASS [{s}]\n", .{self.name});
        }
    } {
        return .{ .val = value, .name = name };
    }
};

// Test suite DSL
pub const TestSuite = struct {
    name: []const u8,
    tests: std.ArrayList(TestCase),
    before_each: ?*const fn () anyerror!void,
    after_each: ?*const fn () anyerror!void,
    allocator: std.mem.Allocator,
    passed: u32,
    failed: u32,

    pub const TestCase = struct {
        name: []const u8,
        run_fn: *const fn () anyerror!void,
    };

    pub fn init(allocator: std.mem.Allocator, suite_name: []const u8) TestSuite {
        return TestSuite{
            .name = suite_name,
            .tests = std.ArrayList(TestCase).init(allocator),
            .before_each = null,
            .after_each = null,
            .allocator = allocator,
            .passed = 0,
            .failed = 0,
        };
    }

    pub fn deinit(self: *TestSuite) void {
        self.tests.deinit();
    }

    // DSL method
    pub fn it(self: *TestSuite, test_name: []const u8, run_fn: *const fn () anyerror!void) !void {
        try self.tests.append(TestCase{
            .name = test_name,
            .run_fn = run_fn,
        });
    }

    pub fn beforeEach(self: *TestSuite, setup_fn: *const fn () anyerror!void) void {
        self.before_each = setup_fn;
    }

    pub fn afterEach(self: *TestSuite, teardown_fn: *const fn () anyerror!void) void {
        self.after_each = teardown_fn;
    }

    pub fn run(self: *TestSuite) !void {
        std.debug.print("\n=== {s} ===\n", .{self.name});

        for (self.tests.items) |test_case| {
            if (self.before_each) |setup| {
                setup() catch |e| {
                    std.debug.print("Setup failed: {}\n", .{e});
                    continue;
                };
            }

            test_case.run_fn() catch |e| {
                std.debug.print("  FAIL: {s} - {}\n", .{ test_case.name, e });
                self.failed += 1;
                continue;
            };

            if (self.after_each) |teardown| {
                teardown() catch |e| {
                    std.debug.print("Teardown failed: {}\n", .{e});
                };
            }

            std.debug.print("  PASS: {s}\n", .{test_case.name});
            self.passed += 1;
        }

        std.debug.print("\nResults: {} passed, {} failed\n", .{ self.passed, self.failed });
    }
};

// Matcher DSL
pub fn expect(value: anytype) ExpectChain(@TypeOf(value)) {
    return ExpectChain(@TypeOf(value)).init(value);
}

pub fn ExpectChain(comptime T: type) type {
    return struct {
        value: T,
        negated: bool = false,

        const Self = @This();

        pub fn init(val: T) Self {
            return Self{ .value = val };
        }

        pub fn not(self: Self) Self {
            return Self{ .value = self.value, .negated = !self.negated };
        }

        pub fn toEqual(self: Self, expected: T) !void {
            const matches = self.value == expected;
            const should_match = !self.negated;
            if (matches != should_match) {
                return error.ExpectationFailed;
            }
        }

        pub fn toBeDefined(self: Self) !void {
            // Check for optional types
            const info = @typeInfo(T);
            switch (info) {
                .Optional => {
                    const is_defined = self.value != null;
                    const should_be_defined = !self.negated;
                    if (is_defined != should_be_defined) {
                        return error.ExpectationFailed;
                    }
                },
                else => {}, // Non-optional types are always defined
            }
        }
    };
}

// Test runner
pub fn describe(name: []const u8, allocator: std.mem.Allocator, setup_fn: *const fn (*TestSuite) anyerror!void) !void {
    var suite = TestSuite.init(allocator, name);
    defer suite.deinit();

    try setup_fn(&suite);
    try suite.run();
}
```

---

## Step 910: Complete DSL Implementation

### Complete DSL Example: Rule Engine

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Rule Engine DSL - Complete implementation
pub const RuleEngine = struct {
    allocator: Allocator,
    rules: std.ArrayList(Rule),
    facts: std.StringHashMap(FactValue),

    pub const FactValue = union(enum) {
        boolean: bool,
        integer: i64,
        float: f64,
        string: []const u8,
    };

    pub const Condition = struct {
        fact_name: []const u8,
        operator: ConditionOp,
        value: FactValue,

        pub const ConditionOp = enum {
            Eq, NotEq, Gt, Lt, GtEq, LtEq, Contains,
        };

        pub fn evaluate(self: Condition, facts: std.StringHashMap(FactValue)) bool {
            const fact = facts.get(self.fact_name) orelse return false;

            return switch (self.operator) {
                .Eq => switch (fact) {
                    .boolean => |b| switch (self.value) {
                        .boolean => |bv| b == bv,
                        else => false,
                    },
                    .integer => |i| switch (self.value) {
                        .integer => |iv| i == iv,
                        else => false,
                    },
                    .float => |f| switch (self.value) {
                        .float => |fv| f == fv,
                        else => false,
                    },
                    .string => |s| switch (self.value) {
                        .string => |sv| std.mem.eql(u8, s, sv),
                        else => false,
                    },
                },
                .Gt => switch (fact) {
                    .integer => |i| switch (self.value) {
                        .integer => |iv| i > iv,
                        else => false,
                    },
                    .float => |f| switch (self.value) {
                        .float => |fv| f > fv,
                        else => false,
                    },
                    else => false,
                },
                .Lt => switch (fact) {
                    .integer => |i| switch (self.value) {
                        .integer => |iv| i < iv,
                        else => false,
                    },
                    .float => |f| switch (self.value) {
                        .float => |fv| f < fv,
                        else => false,
                    },
                    else => false,
                },
                else => false,
            };
        }
    };

    pub const Action = struct {
        action_type: ActionType,
        params: std.StringHashMap([]const u8),

        pub const ActionType = enum {
            SetFact,
            RemoveFact,
            Log,
            Emit,
        };
    };

    pub const Rule = struct {
        name: []const u8,
        priority: i32,
        conditions: std.ArrayList(Condition),
        actions: std.ArrayList(Action),
        enabled: bool,
        allocator: Allocator,

        pub fn init(allocator: Allocator, name: []const u8) Rule {
            return Rule{
                .name = name,
                .priority = 0,
                .conditions = std.ArrayList(Condition).init(allocator),
                .actions = std.ArrayList(Action).init(allocator),
                .enabled = true,
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Rule) void {
            self.conditions.deinit();
            for (self.actions.items) |*action| {
                action.params.deinit();
            }
            self.actions.deinit();
        }

        pub fn matches(self: Rule, facts: std.StringHashMap(FactValue)) bool {
            if (!self.enabled) return false;
            for (self.conditions.items) |cond| {
                if (!cond.evaluate(facts)) return false;
            }
            return true;
        }
    };

    pub fn init(allocator: Allocator) RuleEngine {
        return RuleEngine{
            .allocator = allocator,
            .rules = std.ArrayList(Rule).init(allocator),
            .facts = std.StringHashMap(FactValue).init(allocator),
        };
    }

    pub fn deinit(self: *RuleEngine) void {
        for (self.rules.items) |*rule| {
            rule.deinit();
        }
        self.rules.deinit();
        self.facts.deinit();
    }

    // DSL methods
    pub fn when(self: *RuleEngine, rule_name: []const u8) RuleBuilder {
        return RuleBuilder{
            .engine = self,
            .rule = Rule.init(self.allocator, rule_name),
        };
    }

    pub fn assert(self: *RuleEngine, fact: []const u8, value: FactValue) !void {
        try self.facts.put(fact, value);
    }

    pub fn retract(self: *RuleEngine, fact: []const u8) void {
        _ = self.facts.remove(fact);
    }

    pub fn fire(self: *RuleEngine) !u32 {
        var fired: u32 = 0;
        
        // Sort by priority
        std.sort.pdq(Rule, self.rules.items, {}, struct {
            fn lessThan(_: void, a: Rule, b: Rule) bool {
                return a.priority > b.priority;
            }
        }.lessThan);

        for (self.rules.items) |*rule| {
            if (rule.matches(self.facts)) {
                try self.executeRule(rule);
                fired += 1;
            }
        }
        return fired;
    }

    fn executeRule(self: *RuleEngine, rule: *Rule) !void {
        std.debug.print("Firing rule: {s}\n", .{rule.name});
        for (rule.actions.items) |action| {
            switch (action.action_type) {
                .Log => {
                    if (action.params.get("message")) |msg| {
                        std.debug.print("  LOG: {s}\n", .{msg});
                    }
                },
                .SetFact => {
                    if (action.params.get("fact")) |fact_name| {
                        if (action.params.get("value")) |val| {
                            try self.facts.put(fact_name, FactValue{ .string = val });
                        }
                    }
                },
                else => {},
            }
        }
    }

    pub const RuleBuilder = struct {
        engine: *RuleEngine,
        rule: Rule,

        pub fn condition(self: *RuleBuilder, fact: []const u8, op: Condition.ConditionOp, value: FactValue) !*RuleBuilder {
            try self.rule.conditions.append(Condition{
                .fact_name = fact,
                .operator = op,
                .value = value,
            });
            return self;
        }

        pub fn priority(self: *RuleBuilder, p: i32) *RuleBuilder {
            self.rule.priority = p;
            return self;
        }

        pub fn then(self: *RuleBuilder) ActionBuilder {
            return ActionBuilder{ .rule_builder = self };
        }

        pub const ActionBuilder = struct {
            rule_builder: *RuleBuilder,

            pub fn log(self: ActionBuilder, message: []const u8) !ActionBuilder {
                var params = std.StringHashMap([]const u8).init(self.rule_builder.engine.allocator);
                try params.put("message", message);
                try self.rule_builder.rule.actions.append(Action{
                    .action_type = .Log,
                    .params = params,
                });
                return self;
            }

            pub fn setFact(self: ActionBuilder, fact: []const u8, value: []const u8) !ActionBuilder {
                var params = std.StringHashMap([]const u8).init(self.rule_builder.engine.allocator);
                try params.put("fact", fact);
                try params.put("value", value);
                try self.rule_builder.rule.actions.append(Action{
                    .action_type = .SetFact,
                    .params = params,
                });
                return self;
            }

            pub fn commit(self: ActionBuilder) !void {
                try self.rule_builder.engine.rules.append(self.rule_builder.rule);
            }
        };
    };
};

pub fn ruleEngineExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var engine = RuleEngine.init(allocator);
    defer engine.deinit();

    // Define rules using DSL
    var rule1_builder = engine.when("high-temperature-alert");
    _ = rule1_builder.priority(10);
    _ = try rule1_builder.condition("temperature", .Gt, .{ .integer = 35 });
    const action1 = rule1_builder.then();
    _ = try action1.log("Temperature is above 35 degrees!");
    try action1.setFact("alert_level", "high");

    // Assert facts
    try engine.assert("temperature", .{ .integer = 38 });
    try engine.assert("humidity", .{ .integer = 80 });

    // Fire rules
    const fired = try engine.fire();
    std.debug.print("\nTotal rules fired: {}\n", .{fired});
}

pub fn main() !void {
    std.debug.print("=== DSL Examples ===\n\n", .{});
    
    try queryBuilderExample();
    try configDslExample();
    try templateDslExample();
    try trafficLightExample();
    try buildSystemExample();
    try ruleEngineExample();
}
```

---

## สรุป Part 91

ในส่วนนี้เราได้เรียนรู้:

1. **DSL Concepts** - ความแตกต่างระหว่าง internal และ external DSL
2. **Method Chaining** - เทคนิคสร้าง fluent API
3. **Query Builder** - สร้าง type-safe SQL queries
4. **Configuration DSL** - กำหนด config แบบ builder pattern
5. **Parser Combinator** - สร้าง composable parsers
6. **Template DSL** - engine สำหรับ text templates
7. **State Machine DSL** - นิยาม state transitions แบบ declarative
8. **Build System DSL** - กำหนด build graph
9. **Testing DSL** - เขียน tests แบบ BDD
10. **Rule Engine** - complete DSL implementation

### Best Practices สำหรับ DSL ใน Zig

- ใช้ comptime เมื่อต้องการ compile-time validation
- Return `*Self` เพื่อ enable method chaining
- แยก Builder จาก final object
- ใช้ error union สำหรับ validation
- Design API ให้อ่านเหมือนภาษาธรรมชาติ
