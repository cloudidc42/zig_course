# ส่วนที่ 76: Design Patterns ใน Zig (ขั้นตอนที่ 751-760)

Design Patterns คือแนวทางการแก้ปัญหาที่ผ่านการพิสูจน์แล้วสำหรับปัญหาที่พบบ่อยในการออกแบบซอฟต์แวร์ ใน Zig เราสามารถนำ patterns เหล่านี้มาใช้ได้โดยปรับให้เหมาะกับระบบ type และ memory management ของ Zig

---

## ขั้นตอนที่ 751: Factory Pattern

Factory Pattern คือ pattern ที่ใช้สร้าง object โดยไม่ต้องระบุ class ที่แน่ชัด ทำให้สามารถสลับการสร้าง object ได้อย่างยืดหยุ่น

### แนวคิดหลัก

- แยกการสร้าง object ออกจากการใช้งาน
- ใช้ interface หรือ union เพื่อจัดการ polymorphism
- รองรับการเพิ่ม type ใหม่โดยไม่กระทบ code เดิม

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// กำหนด interface สำหรับ Shape
const ShapeType = enum {
    circle,
    rectangle,
    triangle,
};

const Shape = union(ShapeType) {
    circle: Circle,
    rectangle: Rectangle,
    triangle: Triangle,

    pub fn area(self: Shape) f64 {
        return switch (self) {
            .circle => |c| c.area(),
            .rectangle => |r| r.area(),
            .triangle => |t| t.area(),
        };
    }

    pub fn perimeter(self: Shape) f64 {
        return switch (self) {
            .circle => |c| c.perimeter(),
            .rectangle => |r| r.perimeter(),
            .triangle => |t| t.perimeter(),
        };
    }

    pub fn name(self: Shape) []const u8 {
        return switch (self) {
            .circle => "วงกลม",
            .rectangle => "สี่เหลี่ยม",
            .triangle => "สามเหลี่ยม",
        };
    }
};

const Circle = struct {
    radius: f64,

    pub fn area(self: Circle) f64 {
        return std.math.pi * self.radius * self.radius;
    }

    pub fn perimeter(self: Circle) f64 {
        return 2.0 * std.math.pi * self.radius;
    }
};

const Rectangle = struct {
    width: f64,
    height: f64,

    pub fn area(self: Rectangle) f64 {
        return self.width * self.height;
    }

    pub fn perimeter(self: Rectangle) f64 {
        return 2.0 * (self.width + self.height);
    }
};

const Triangle = struct {
    a: f64,
    b: f64,
    c: f64,

    pub fn area(self: Triangle) f64 {
        const s = (self.a + self.b + self.c) / 2.0;
        return @sqrt(s * (s - self.a) * (s - self.b) * (s - self.c));
    }

    pub fn perimeter(self: Triangle) f64 {
        return self.a + self.b + self.c;
    }
};

// Factory function
fn createShape(shape_type: ShapeType, params: anytype) Shape {
    return switch (shape_type) {
        .circle => Shape{ .circle = Circle{ .radius = params.radius } },
        .rectangle => Shape{ .rectangle = Rectangle{ .width = params.width, .height = params.height } },
        .triangle => Shape{ .triangle = Triangle{ .a = params.a, .b = params.b, .c = params.c } },
    };
}

// ShapeFactory struct แบบ OOP
const ShapeFactory = struct {
    allocator: Allocator,

    pub fn init(allocator: Allocator) ShapeFactory {
        return ShapeFactory{ .allocator = allocator };
    }

    pub fn createCircle(self: *ShapeFactory, radius: f64) Shape {
        _ = self;
        return Shape{ .circle = Circle{ .radius = radius } };
    }

    pub fn createRectangle(self: *ShapeFactory, width: f64, height: f64) Shape {
        _ = self;
        return Shape{ .rectangle = Rectangle{ .width = width, .height = height } };
    }

    pub fn createFromString(self: *ShapeFactory, type_str: []const u8, params: anytype) !Shape {
        _ = self;
        if (std.mem.eql(u8, type_str, "circle")) {
            return Shape{ .circle = Circle{ .radius = params.radius } };
        } else if (std.mem.eql(u8, type_str, "rectangle")) {
            return Shape{ .rectangle = Rectangle{ .width = params.width, .height = params.height } };
        } else {
            return error.UnknownShapeType;
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var factory = ShapeFactory.init(allocator);

    // สร้าง shapes ด้วย factory
    const shapes = [_]Shape{
        factory.createCircle(5.0),
        factory.createRectangle(4.0, 6.0),
        createShape(.triangle, .{ .a = 3.0, .b = 4.0, .c = 5.0 }),
    };

    std.debug.print("=== Factory Pattern Demo ===\n", .{});
    for (shapes) |shape| {
        std.debug.print("{s}: พื้นที่ = {d:.2}, เส้นรอบ = {d:.2}\n", .{
            shape.name(),
            shape.area(),
            shape.perimeter(),
        });
    }

    // ใช้ createFromString
    const dynamic_shape = try factory.createFromString("circle", .{ .radius = 3.0 });
    std.debug.print("\nสร้างจาก string: {s} พื้นที่ = {d:.2}\n", .{
        dynamic_shape.name(),
        dynamic_shape.area(),
    });
}
```

### ตัวอย่างเพิ่มเติม: Abstract Factory

```zig
const std = @import("std");

// Database factory - สร้าง database connections ชนิดต่างๆ
const DatabaseType = enum { sqlite, postgresql, mysql };

const DatabaseConfig = struct {
    host: []const u8 = "localhost",
    port: u16 = 5432,
    database: []const u8,
    username: []const u8 = "root",
    password: []const u8 = "",
};

const DatabaseConnection = struct {
    db_type: DatabaseType,
    config: DatabaseConfig,
    connected: bool = false,

    pub fn connect(self: *DatabaseConnection) !void {
        std.debug.print("เชื่อมต่อ {s} ที่ {s}:{d}\n", .{
            @tagName(self.db_type),
            self.config.host,
            self.config.port,
        });
        self.connected = true;
    }

    pub fn disconnect(self: *DatabaseConnection) void {
        if (self.connected) {
            std.debug.print("ยกเลิกการเชื่อมต่อ\n", .{});
            self.connected = false;
        }
    }

    pub fn query(self: *DatabaseConnection, sql: []const u8) !void {
        if (!self.connected) return error.NotConnected;
        std.debug.print("Execute: {s}\n", .{sql});
    }
};

fn createDatabaseConnection(db_type: DatabaseType, config: DatabaseConfig) DatabaseConnection {
    var adjusted_config = config;
    // ปรับ port ตาม database type
    adjusted_config.port = switch (db_type) {
        .sqlite => 0,
        .postgresql => 5432,
        .mysql => 3306,
    };
    return DatabaseConnection{
        .db_type = db_type,
        .config = adjusted_config,
    };
}
```

---

## ขั้นตอนที่ 752: Builder Pattern

Builder Pattern ใช้สร้าง object ที่ซับซ้อนทีละขั้นตอน ช่วยแยกกระบวนการสร้างออกจาก representation

### แนวคิดหลัก

- สร้าง object ทีละส่วน
- รองรับ method chaining
- มี validation ก่อน build

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// HTTP Request Builder
const HttpMethod = enum { GET, POST, PUT, DELETE, PATCH };

const HttpRequest = struct {
    method: HttpMethod,
    url: []const u8,
    headers: std.StringHashMap([]const u8),
    body: ?[]const u8,
    timeout_ms: u32,
    follow_redirects: bool,
    max_redirects: u8,

    pub fn deinit(self: *HttpRequest) void {
        self.headers.deinit();
    }
};

const HttpRequestBuilder = struct {
    allocator: Allocator,
    method: HttpMethod = .GET,
    url: []const u8 = "",
    headers: std.StringHashMap([]const u8),
    body: ?[]const u8 = null,
    timeout_ms: u32 = 30000,
    follow_redirects: bool = true,
    max_redirects: u8 = 5,

    pub fn init(allocator: Allocator) HttpRequestBuilder {
        return HttpRequestBuilder{
            .allocator = allocator,
            .headers = std.StringHashMap([]const u8).init(allocator),
        };
    }

    pub fn setMethod(self: *HttpRequestBuilder, method: HttpMethod) *HttpRequestBuilder {
        self.method = method;
        return self;
    }

    pub fn setUrl(self: *HttpRequestBuilder, url: []const u8) *HttpRequestBuilder {
        self.url = url;
        return self;
    }

    pub fn addHeader(self: *HttpRequestBuilder, key: []const u8, value: []const u8) !*HttpRequestBuilder {
        try self.headers.put(key, value);
        return self;
    }

    pub fn setBody(self: *HttpRequestBuilder, body: []const u8) *HttpRequestBuilder {
        self.body = body;
        return self;
    }

    pub fn setTimeout(self: *HttpRequestBuilder, ms: u32) *HttpRequestBuilder {
        self.timeout_ms = ms;
        return self;
    }

    pub fn setFollowRedirects(self: *HttpRequestBuilder, follow: bool, max: u8) *HttpRequestBuilder {
        self.follow_redirects = follow;
        self.max_redirects = max;
        return self;
    }

    pub fn build(self: *HttpRequestBuilder) !HttpRequest {
        // Validation
        if (self.url.len == 0) {
            return error.MissingUrl;
        }
        if (self.method == .POST or self.method == .PUT) {
            // POST/PUT ควรมี Content-Type header
            if (!self.headers.contains("Content-Type")) {
                try self.headers.put("Content-Type", "application/json");
            }
        }

        return HttpRequest{
            .method = self.method,
            .url = self.url,
            .headers = self.headers,
            .body = self.body,
            .timeout_ms = self.timeout_ms,
            .follow_redirects = self.follow_redirects,
            .max_redirects = self.max_redirects,
        };
    }
};

// Query Builder สำหรับ SQL
const QueryBuilder = struct {
    allocator: Allocator,
    table: []const u8 = "",
    conditions: std.ArrayList([]const u8),
    columns: std.ArrayList([]const u8),
    order_by: ?[]const u8 = null,
    limit_val: ?u32 = null,
    offset_val: ?u32 = null,

    pub fn init(allocator: Allocator) QueryBuilder {
        return QueryBuilder{
            .allocator = allocator,
            .conditions = std.ArrayList([]const u8).init(allocator),
            .columns = std.ArrayList([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *QueryBuilder) void {
        self.conditions.deinit();
        self.columns.deinit();
    }

    pub fn from(self: *QueryBuilder, table: []const u8) *QueryBuilder {
        self.table = table;
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

    pub fn build(self: *QueryBuilder, buf: []u8) ![]u8 {
        var fbs = std.io.fixedBufferStream(buf);
        const writer = fbs.writer();

        try writer.writeAll("SELECT ");

        if (self.columns.items.len == 0) {
            try writer.writeAll("*");
        } else {
            for (self.columns.items, 0..) |col, i| {
                if (i > 0) try writer.writeAll(", ");
                try writer.writeAll(col);
            }
        }

        try writer.print(" FROM {s}", .{self.table});

        if (self.conditions.items.len > 0) {
            try writer.writeAll(" WHERE ");
            for (self.conditions.items, 0..) |cond, i| {
                if (i > 0) try writer.writeAll(" AND ");
                try writer.writeAll(cond);
            }
        }

        if (self.order_by) |col| {
            try writer.print(" ORDER BY {s}", .{col});
        }

        if (self.limit_val) |n| {
            try writer.print(" LIMIT {d}", .{n});
        }

        if (self.offset_val) |n| {
            try writer.print(" OFFSET {d}", .{n});
        }

        return fbs.getWritten();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Builder Pattern Demo ===\n", .{});

    // HTTP Request Builder
    var builder = HttpRequestBuilder.init(allocator);
    _ = builder.setMethod(.POST)
        .setUrl("https://api.example.com/users")
        .setBody("{\"name\": \"John\"}");
    _ = try builder.addHeader("Authorization", "Bearer token123");
    _ = builder.setTimeout(5000);

    var request = try builder.build();
    defer request.deinit();

    std.debug.print("Method: {s}\n", .{@tagName(request.method)});
    std.debug.print("URL: {s}\n", .{request.url});
    std.debug.print("Timeout: {d}ms\n", .{request.timeout_ms});

    // SQL Query Builder
    var qb = QueryBuilder.init(allocator);
    defer qb.deinit();

    _ = qb.from("users");
    _ = try qb.select("id");
    _ = try qb.select("name");
    _ = try qb.select("email");
    _ = try qb.where("age > 18");
    _ = try qb.where("active = true");
    _ = qb.orderBy("name");
    _ = qb.limit(10);
    _ = qb.offset(20);

    var buf: [512]u8 = undefined;
    const query = try qb.build(&buf);
    std.debug.print("\nSQL Query: {s}\n", .{query});
}
```

---

## ขั้นตอนที่ 753: Observer Pattern

Observer Pattern ใช้สร้างระบบ notification ที่ object หนึ่งสามารถแจ้ง object อื่นๆ เมื่อมีการเปลี่ยนแปลง

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Event types
const EventType = enum {
    price_changed,
    stock_updated,
    user_login,
    user_logout,
    error_occurred,
};

const Event = struct {
    event_type: EventType,
    data: []const u8,
    timestamp: i64,
};

// Observer interface ผ่าน function pointer
const Observer = struct {
    context: *anyopaque,
    onEventFn: *const fn (ctx: *anyopaque, event: Event) void,

    pub fn notify(self: Observer, event: Event) void {
        self.onEventFn(self.context, event);
    }
};

// EventEmitter (Subject)
const EventEmitter = struct {
    allocator: Allocator,
    observers: std.ArrayList(Observer),

    pub fn init(allocator: Allocator) EventEmitter {
        return EventEmitter{
            .allocator = allocator,
            .observers = std.ArrayList(Observer).init(allocator),
        };
    }

    pub fn deinit(self: *EventEmitter) void {
        self.observers.deinit();
    }

    pub fn subscribe(self: *EventEmitter, observer: Observer) !void {
        try self.observers.append(observer);
    }

    pub fn unsubscribe(self: *EventEmitter, context: *anyopaque) void {
        var i: usize = 0;
        while (i < self.observers.items.len) {
            if (self.observers.items[i].context == context) {
                _ = self.observers.orderedRemove(i);
            } else {
                i += 1;
            }
        }
    }

    pub fn emit(self: *EventEmitter, event: Event) void {
        for (self.observers.items) |observer| {
            observer.notify(event);
        }
    }
};

// Concrete Observer: Logger
const Logger = struct {
    name: []const u8,
    log_count: u32 = 0,

    pub fn getObserver(self: *Logger) Observer {
        return Observer{
            .context = self,
            .onEventFn = onEvent,
        };
    }

    fn onEvent(ctx: *anyopaque, event: Event) void {
        const self: *Logger = @ptrCast(@alignCast(ctx));
        self.log_count += 1;
        std.debug.print("[{s}] Event #{d}: {s} - {s}\n", .{
            self.name,
            self.log_count,
            @tagName(event.event_type),
            event.data,
        });
    }
};

// Concrete Observer: Metrics Collector
const MetricsCollector = struct {
    event_counts: [5]u32 = [_]u32{0} ** 5,

    pub fn getObserver(self: *MetricsCollector) Observer {
        return Observer{
            .context = self,
            .onEventFn = onEvent,
        };
    }

    fn onEvent(ctx: *anyopaque, event: Event) void {
        const self: *MetricsCollector = @ptrCast(@alignCast(ctx));
        const idx = @intFromEnum(event.event_type);
        self.event_counts[idx] += 1;
    }

    pub fn printStats(self: *MetricsCollector) void {
        std.debug.print("\n=== Metrics ===\n", .{});
        inline for (std.meta.fields(EventType)) |field| {
            std.debug.print("{s}: {d}\n", .{
                field.name,
                self.event_counts[field.value],
            });
        }
    }
};

// Stock Price Monitor
const StockMonitor = struct {
    emitter: EventEmitter,
    symbol: []const u8,
    current_price: f64,

    pub fn init(allocator: Allocator, symbol: []const u8, initial_price: f64) StockMonitor {
        return StockMonitor{
            .emitter = EventEmitter.init(allocator),
            .symbol = symbol,
            .current_price = initial_price,
        };
    }

    pub fn deinit(self: *StockMonitor) void {
        self.emitter.deinit();
    }

    pub fn updatePrice(self: *StockMonitor, new_price: f64) !void {
        const old_price = self.current_price;
        self.current_price = new_price;

        var buf: [128]u8 = undefined;
        const msg = try std.fmt.bufPrint(&buf, "{s}: {d:.2} -> {d:.2}", .{
            self.symbol,
            old_price,
            new_price,
        });

        self.emitter.emit(Event{
            .event_type = .price_changed,
            .data = msg,
            .timestamp = std.time.milliTimestamp(),
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Observer Pattern Demo ===\n", .{});

    var stock = StockMonitor.init(allocator, "AAPL", 150.0);
    defer stock.deinit();

    var logger1 = Logger{ .name = "Console Logger" };
    var logger2 = Logger{ .name = "File Logger" };
    var metrics = MetricsCollector{};

    try stock.emitter.subscribe(logger1.getObserver());
    try stock.emitter.subscribe(logger2.getObserver());
    try stock.emitter.subscribe(metrics.getObserver());

    try stock.updatePrice(152.50);
    try stock.updatePrice(148.75);
    try stock.updatePrice(155.00);

    // Unsubscribe logger2
    stock.emitter.unsubscribe(logger2.context);
    std.debug.print("\n[Logger2 unsubscribed]\n", .{});
    try stock.updatePrice(157.25);

    metrics.printStats();
}
```

---

## ขั้นตอนที่ 754: Strategy Pattern

Strategy Pattern ให้สามารถเปลี่ยนอัลกอริทึมได้ runtime โดยไม่ต้องแก้ไข client code

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Sorting Strategy
const SortStrategy = struct {
    context: *anyopaque,
    sortFn: *const fn (ctx: *anyopaque, data: []i32) void,

    pub fn sort(self: SortStrategy, data: []i32) void {
        self.sortFn(self.context, data);
    }
};

// Bubble Sort Strategy
const BubbleSortStrategy = struct {
    comparisons: u64 = 0,
    swaps: u64 = 0,

    pub fn getStrategy(self: *BubbleSortStrategy) SortStrategy {
        return SortStrategy{
            .context = self,
            .sortFn = sort,
        };
    }

    fn sort(ctx: *anyopaque, data: []i32) void {
        const self: *BubbleSortStrategy = @ptrCast(@alignCast(ctx));
        self.comparisons = 0;
        self.swaps = 0;
        const n = data.len;
        var i: usize = 0;
        while (i < n - 1) : (i += 1) {
            var j: usize = 0;
            while (j < n - 1 - i) : (j += 1) {
                self.comparisons += 1;
                if (data[j] > data[j + 1]) {
                    const tmp = data[j];
                    data[j] = data[j + 1];
                    data[j + 1] = tmp;
                    self.swaps += 1;
                }
            }
        }
    }
};

// Quick Sort Strategy
const QuickSortStrategy = struct {
    comparisons: u64 = 0,

    pub fn getStrategy(self: *QuickSortStrategy) SortStrategy {
        return SortStrategy{
            .context = self,
            .sortFn = sort,
        };
    }

    fn sort(ctx: *anyopaque, data: []i32) void {
        const self: *QuickSortStrategy = @ptrCast(@alignCast(ctx));
        self.comparisons = 0;
        quickSort(self, data, 0, data.len);
    }

    fn quickSort(self: *QuickSortStrategy, data: []i32, lo: usize, hi: usize) void {
        if (hi - lo <= 1) return;
        const pivot_idx = partition(self, data, lo, hi);
        if (pivot_idx > lo) quickSort(self, data, lo, pivot_idx);
        quickSort(self, data, pivot_idx + 1, hi);
    }

    fn partition(self: *QuickSortStrategy, data: []i32, lo: usize, hi: usize) usize {
        const pivot = data[hi - 1];
        var i = lo;
        var j = lo;
        while (j < hi - 1) : (j += 1) {
            self.comparisons += 1;
            if (data[j] <= pivot) {
                const tmp = data[i];
                data[i] = data[j];
                data[j] = tmp;
                i += 1;
            }
        }
        const tmp = data[i];
        data[i] = data[hi - 1];
        data[hi - 1] = tmp;
        return i;
    }
};

// Sorter ที่ใช้ Strategy
const Sorter = struct {
    strategy: SortStrategy,

    pub fn init(strategy: SortStrategy) Sorter {
        return Sorter{ .strategy = strategy };
    }

    pub fn setStrategy(self: *Sorter, strategy: SortStrategy) void {
        self.strategy = strategy;
    }

    pub fn sort(self: *Sorter, data: []i32) void {
        self.strategy.sort(data);
    }
};

// Payment Strategy
const PaymentMethod = union(enum) {
    credit_card: CreditCardPayment,
    bank_transfer: BankTransferPayment,
    crypto: CryptoPayment,

    pub fn process(self: PaymentMethod, amount: f64) !void {
        return switch (self) {
            .credit_card => |p| p.process(amount),
            .bank_transfer => |p| p.process(amount),
            .crypto => |p| p.process(amount),
        };
    }

    pub fn fee(self: PaymentMethod, amount: f64) f64 {
        return switch (self) {
            .credit_card => |p| p.calculateFee(amount),
            .bank_transfer => |p| p.calculateFee(amount),
            .crypto => |p| p.calculateFee(amount),
        };
    }
};

const CreditCardPayment = struct {
    card_number: []const u8,
    fee_percent: f64 = 2.5,

    pub fn process(self: CreditCardPayment, amount: f64) !void {
        std.debug.print("ชำระด้วยบัตรเครดิต *{s}: {d:.2} บาท\n", .{
            self.card_number[self.card_number.len - 4 ..],
            amount,
        });
    }

    pub fn calculateFee(self: CreditCardPayment, amount: f64) f64 {
        return amount * self.fee_percent / 100.0;
    }
};

const BankTransferPayment = struct {
    account: []const u8,
    fee_percent: f64 = 0.5,

    pub fn process(self: BankTransferPayment, amount: f64) !void {
        std.debug.print("โอนเงินไปยังบัญชี {s}: {d:.2} บาท\n", .{
            self.account,
            amount,
        });
    }

    pub fn calculateFee(self: BankTransferPayment, amount: f64) f64 {
        return @min(amount * self.fee_percent / 100.0, 25.0); // สูงสุด 25 บาท
    }
};

const CryptoPayment = struct {
    wallet: []const u8,
    fee_percent: f64 = 1.0,

    pub fn process(self: CryptoPayment, amount: f64) !void {
        std.debug.print("ชำระด้วย Crypto wallet {s}: {d:.2} USDT\n", .{
            self.wallet[0..8],
            amount / 35.0,
        });
    }

    pub fn calculateFee(self: CryptoPayment, amount: f64) f64 {
        return amount * self.fee_percent / 100.0;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    std.debug.print("=== Strategy Pattern Demo ===\n", .{});

    // Sorting Strategy
    var bubble = BubbleSortStrategy{};
    var quick = QuickSortStrategy{};

    var data1 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    var data2 = data1;

    var sorter = Sorter.init(bubble.getStrategy());
    sorter.sort(&data1);
    std.debug.print("Bubble Sort: comparisons={d}, swaps={d}\n", .{
        bubble.comparisons,
        bubble.swaps,
    });

    sorter.setStrategy(quick.getStrategy());
    sorter.sort(&data2);
    std.debug.print("Quick Sort: comparisons={d}\n", .{quick.comparisons});

    // Payment Strategy
    std.debug.print("\n=== Payment Strategies ===\n", .{});
    const amount = 1000.0;

    const payments = [_]PaymentMethod{
        .{ .credit_card = .{ .card_number = "1234567890123456" } },
        .{ .bank_transfer = .{ .account = "1234567890" } },
        .{ .crypto = .{ .wallet = "0xABCDEF1234567890" } },
    };

    for (payments) |payment| {
        try payment.process(amount);
        std.debug.print("  ค่าธรรมเนียม: {d:.2} บาท\n", .{payment.fee(amount)});
    }
}
```

---

## ขั้นตอนที่ 755: Command Pattern

Command Pattern encapsulates actions เป็น object ทำให้สามารถ undo/redo, queue, และ log commands ได้

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Command interface
const Command = struct {
    context: *anyopaque,
    executeFn: *const fn (ctx: *anyopaque) !void,
    undoFn: *const fn (ctx: *anyopaque) void,
    description: []const u8,

    pub fn execute(self: Command) !void {
        return self.executeFn(self.context);
    }

    pub fn undo(self: Command) void {
        self.undoFn(self.context);
    }
};

// Text Editor
const TextEditor = struct {
    allocator: Allocator,
    text: std.ArrayList(u8),
    history: std.ArrayList(Command),
    undo_stack: std.ArrayList(Command),

    pub fn init(allocator: Allocator) TextEditor {
        return TextEditor{
            .allocator = allocator,
            .text = std.ArrayList(u8).init(allocator),
            .history = std.ArrayList(Command).init(allocator),
            .undo_stack = std.ArrayList(Command).init(allocator),
        };
    }

    pub fn deinit(self: *TextEditor) void {
        self.text.deinit();
        self.history.deinit();
        self.undo_stack.deinit();
    }

    pub fn executeCommand(self: *TextEditor, cmd: Command) !void {
        try cmd.execute();
        try self.history.append(cmd);
        // ล้าง undo stack เมื่อมีคำสั่งใหม่
        self.undo_stack.clearRetainingCapacity();
    }

    pub fn undo(self: *TextEditor) !void {
        if (self.history.items.len == 0) {
            std.debug.print("ไม่มีคำสั่งที่สามารถ undo ได้\n", .{});
            return;
        }
        const cmd = self.history.pop();
        cmd.undo();
        try self.undo_stack.append(cmd);
    }

    pub fn redo(self: *TextEditor) !void {
        if (self.undo_stack.items.len == 0) {
            std.debug.print("ไม่มีคำสั่งที่สามารถ redo ได้\n", .{});
            return;
        }
        const cmd = self.undo_stack.pop();
        try cmd.execute();
        try self.history.append(cmd);
    }

    pub fn printText(self: *TextEditor) void {
        std.debug.print("ข้อความ: '{s}'\n", .{self.text.items});
    }
};

// Concrete Command: Insert Text
const InsertCommand = struct {
    editor: *TextEditor,
    position: usize,
    text: []const u8,
    inserted: bool = false,

    pub fn getCommand(self: *InsertCommand) Command {
        return Command{
            .context = self,
            .executeFn = execute,
            .undoFn = undo,
            .description = "Insert",
        };
    }

    fn execute(ctx: *anyopaque) !void {
        const self: *InsertCommand = @ptrCast(@alignCast(ctx));
        const pos = @min(self.position, self.editor.text.items.len);
        try self.editor.text.insertSlice(pos, self.text);
        self.inserted = true;
        std.debug.print("Insert '{s}' ที่ตำแหน่ง {d}\n", .{ self.text, pos });
    }

    fn undo(ctx: *anyopaque) void {
        const self: *InsertCommand = @ptrCast(@alignCast(ctx));
        if (!self.inserted) return;
        const pos = @min(self.position, self.editor.text.items.len);
        const end = @min(pos + self.text.len, self.editor.text.items.len);
        self.editor.text.replaceRangeAssumeCapacity(pos, end - pos, &[_]u8{});
        self.inserted = false;
        std.debug.print("Undo insert '{s}'\n", .{self.text});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Command Pattern Demo ===\n", .{});

    var editor = TextEditor.init(allocator);
    defer editor.deinit();

    var cmd1 = InsertCommand{
        .editor = &editor,
        .position = 0,
        .text = "Hello",
    };

    var cmd2 = InsertCommand{
        .editor = &editor,
        .position = 5,
        .text = " World",
    };

    var cmd3 = InsertCommand{
        .editor = &editor,
        .position = 11,
        .text = "!",
    };

    try editor.executeCommand(cmd1.getCommand());
    editor.printText();

    try editor.executeCommand(cmd2.getCommand());
    editor.printText();

    try editor.executeCommand(cmd3.getCommand());
    editor.printText();

    std.debug.print("\n-- Undo --\n", .{});
    try editor.undo();
    editor.printText();

    try editor.undo();
    editor.printText();

    std.debug.print("\n-- Redo --\n", .{});
    try editor.redo();
    editor.printText();
}
```

---

## ขั้นตอนที่ 756: State Machine Pattern

State Machine Pattern จัดการสถานะของ object และการเปลี่ยนแปลงระหว่างสถานะต่างๆ

```zig
const std = @import("std");

// Traffic Light State Machine
const TrafficLightState = enum {
    red,
    yellow,
    green,
};

const TrafficLight = struct {
    current_state: TrafficLightState = .red,
    duration: [3]u32 = .{ 30, 5, 25 }, // วินาทีสำหรับแต่ละ state

    pub fn next(self: *TrafficLight) void {
        const old_state = self.current_state;
        self.current_state = switch (self.current_state) {
            .red => .green,
            .green => .yellow,
            .yellow => .red,
        };
        std.debug.print("{s} -> {s}\n", .{
            @tagName(old_state),
            @tagName(self.current_state),
        });
    }

    pub fn getDuration(self: TrafficLight) u32 {
        return self.duration[@intFromEnum(self.current_state)];
    }

    pub fn canGo(self: TrafficLight) bool {
        return self.current_state == .green;
    }
};

// Vending Machine State Machine
const VendingMachineState = enum {
    idle,
    has_money,
    dispensing,
    maintenance,
};

const VendingMachineError = error{
    InvalidOperation,
    InsufficientFunds,
    OutOfStock,
};

const VendingMachine = struct {
    state: VendingMachineState = .idle,
    balance: u32 = 0, // สตางค์
    item_price: u32 = 500, // 5 บาท = 500 สตางค์
    stock: u32 = 10,

    pub fn insertCoin(self: *VendingMachine, amount: u32) !void {
        switch (self.state) {
            .idle, .has_money => {
                self.balance += amount;
                std.debug.print("ใส่เหรียญ {d} บาท, ยอดรวม: {d} บาท\n", .{
                    amount / 100,
                    self.balance / 100,
                });
                if (self.balance >= self.item_price) {
                    self.state = .has_money;
                    std.debug.print("กดปุ่มเพื่อรับสินค้า\n", .{});
                }
            },
            .dispensing => return VendingMachineError.InvalidOperation,
            .maintenance => return VendingMachineError.InvalidOperation,
        }
    }

    pub fn selectItem(self: *VendingMachine) !void {
        switch (self.state) {
            .has_money => {
                if (self.stock == 0) return VendingMachineError.OutOfStock;
                if (self.balance < self.item_price) return VendingMachineError.InsufficientFunds;

                self.state = .dispensing;
                const change = self.balance - self.item_price;
                self.balance = 0;
                self.stock -= 1;

                std.debug.print("กำลังส่งสินค้า...\n", .{});
                if (change > 0) {
                    std.debug.print("ทอนเงิน: {d} บาท\n", .{change / 100});
                }
                self.state = if (self.stock > 0) .idle else .maintenance;
            },
            .idle => return VendingMachineError.InvalidOperation,
            .dispensing => return VendingMachineError.InvalidOperation,
            .maintenance => return VendingMachineError.InvalidOperation,
        }
    }

    pub fn cancel(self: *VendingMachine) void {
        if (self.balance > 0) {
            std.debug.print("คืนเงิน: {d} บาท\n", .{self.balance / 100});
            self.balance = 0;
        }
        self.state = .idle;
    }

    pub fn printStatus(self: VendingMachine) void {
        std.debug.print("สถานะ: {s}, สินค้าเหลือ: {d}, ยอดเงิน: {d} บาท\n", .{
            @tagName(self.state),
            self.stock,
            self.balance / 100,
        });
    }
};

pub fn main() !void {
    std.debug.print("=== State Machine Pattern Demo ===\n", .{});

    // Traffic Light
    std.debug.print("\n--- Traffic Light ---\n", .{});
    var light = TrafficLight{};
    var i: usize = 0;
    while (i < 6) : (i += 1) {
        std.debug.print("State: {s} ({d} วินาที)\n", .{
            @tagName(light.current_state),
            light.getDuration(),
        });
        light.next();
    }

    // Vending Machine
    std.debug.print("\n--- Vending Machine ---\n", .{});
    var machine = VendingMachine{};
    machine.printStatus();

    try machine.insertCoin(100);
    try machine.insertCoin(200);
    try machine.insertCoin(200);
    try machine.selectItem();
    machine.printStatus();
}
```

---

## ขั้นตอนที่ 757: Iterator Pattern

Iterator Pattern ให้สามารถเข้าถึง elements ของ collection โดยไม่ต้องรู้โครงสร้างภายใน

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Generic Iterator
fn Iterator(comptime T: type) type {
    return struct {
        const Self = @This();

        context: *anyopaque,
        nextFn: *const fn (ctx: *anyopaque) ?T,
        resetFn: *const fn (ctx: *anyopaque) void,

        pub fn next(self: *Self) ?T {
            return self.nextFn(self.context);
        }

        pub fn reset(self: *Self) void {
            self.resetFn(self.context);
        }
    };
}

// Range Iterator
fn RangeIterator(comptime T: type) type {
    return struct {
        const Self = @This();
        start: T,
        end: T,
        step: T,
        current: T,

        pub fn init(start: T, end: T, step: T) Self {
            return Self{
                .start = start,
                .end = end,
                .step = step,
                .current = start,
            };
        }

        pub fn next(self: *Self) ?T {
            if (self.current >= self.end) return null;
            const val = self.current;
            self.current += self.step;
            return val;
        }

        pub fn reset(self: *Self) void {
            self.current = self.start;
        }
    };
}

// Fibonacci Iterator
const FibIterator = struct {
    a: u64 = 0,
    b: u64 = 1,
    count: usize = 0,
    max_count: usize,

    pub fn init(max_count: usize) FibIterator {
        return FibIterator{ .max_count = max_count };
    }

    pub fn next(self: *FibIterator) ?u64 {
        if (self.count >= self.max_count) return null;
        const val = self.a;
        const next_val = self.a + self.b;
        self.a = self.b;
        self.b = next_val;
        self.count += 1;
        return val;
    }

    pub fn reset(self: *FibIterator) void {
        self.a = 0;
        self.b = 1;
        self.count = 0;
    }
};

// Tree Iterator (In-order traversal)
fn TreeNode(comptime T: type) type {
    return struct {
        value: T,
        left: ?*@This() = null,
        right: ?*@This() = null,
    };
}

fn InOrderIterator(comptime T: type) type {
    return struct {
        const Self = @This();
        const Node = TreeNode(T);

        stack: std.ArrayList(*Node),
        current: ?*Node,

        pub fn init(allocator: Allocator, root: ?*Node) !Self {
            var self = Self{
                .stack = std.ArrayList(*Node).init(allocator),
                .current = root,
            };
            return self;
        }

        pub fn deinit(self: *Self) void {
            self.stack.deinit();
        }

        pub fn next(self: *Self) !?T {
            while (self.current != null or self.stack.items.len > 0) {
                // ไปทางซ้ายสุด
                while (self.current) |node| {
                    try self.stack.append(node);
                    self.current = node.left;
                }

                if (self.stack.items.len == 0) return null;

                const node = self.stack.pop();
                self.current = node.right;
                return node.value;
            }
            return null;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Iterator Pattern Demo ===\n", .{});

    // Range Iterator
    std.debug.print("\nRange (0..20 step 2): ", .{});
    var range = RangeIterator(i32).init(0, 20, 2);
    while (range.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // Fibonacci Iterator
    std.debug.print("Fibonacci (first 10): ", .{});
    var fib = FibIterator.init(10);
    while (fib.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});

    // Tree In-order Iterator
    // สร้าง BST: 4-2-6-1-3-5-7
    var nodes: [7]TreeNode(i32) = undefined;
    nodes[0] = .{ .value = 4 };
    nodes[1] = .{ .value = 2 };
    nodes[2] = .{ .value = 6 };
    nodes[3] = .{ .value = 1 };
    nodes[4] = .{ .value = 3 };
    nodes[5] = .{ .value = 5 };
    nodes[6] = .{ .value = 7 };

    nodes[0].left = &nodes[1];
    nodes[0].right = &nodes[2];
    nodes[1].left = &nodes[3];
    nodes[1].right = &nodes[4];
    nodes[2].left = &nodes[5];
    nodes[2].right = &nodes[6];

    std.debug.print("Tree In-order: ", .{});
    var iter = try InOrderIterator(i32).init(allocator, &nodes[0]);
    defer iter.deinit();

    while (try iter.next()) |val| {
        std.debug.print("{d} ", .{val});
    }
    std.debug.print("\n", .{});
}
```

---

## ขั้นตอนที่ 758: Decorator Pattern

Decorator Pattern เพิ่มความสามารถให้กับ object โดยไม่ต้องแก้ไข class เดิม

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Coffee Ordering System
const Coffee = struct {
    context: *anyopaque,
    getCostFn: *const fn (ctx: *anyopaque) f64,
    getDescriptionFn: *const fn (ctx: *anyopaque) []const u8,

    pub fn getCost(self: Coffee) f64 {
        return self.getCostFn(self.context);
    }

    pub fn getDescription(self: Coffee) []const u8 {
        return self.getDescriptionFn(self.context);
    }
};

// Base coffee
const SimpleCoffee = struct {
    pub fn getCoffee(self: *SimpleCoffee) Coffee {
        return Coffee{
            .context = self,
            .getCostFn = getCost,
            .getDescriptionFn = getDescription,
        };
    }

    fn getCost(_: *anyopaque) f64 {
        return 25.0;
    }

    fn getDescription(_: *anyopaque) []const u8 {
        return "กาแฟดำ";
    }
};

// Milk Decorator
const MilkDecorator = struct {
    inner: Coffee,
    cost_extra: f64 = 10.0,

    pub fn getCoffee(self: *MilkDecorator) Coffee {
        return Coffee{
            .context = self,
            .getCostFn = getCost,
            .getDescriptionFn = getDescription,
        };
    }

    fn getCost(ctx: *anyopaque) f64 {
        const self: *MilkDecorator = @ptrCast(@alignCast(ctx));
        return self.inner.getCost() + self.cost_extra;
    }

    fn getDescription(ctx: *anyopaque) []const u8 {
        const self: *MilkDecorator = @ptrCast(@alignCast(ctx));
        _ = self;
        return "กาแฟนม";
    }
};

// Sugar Decorator
const SugarDecorator = struct {
    inner: Coffee,
    cost_extra: f64 = 5.0,

    pub fn getCoffee(self: *SugarDecorator) Coffee {
        return Coffee{
            .context = self,
            .getCostFn = getCost,
            .getDescriptionFn = getDescription,
        };
    }

    fn getCost(ctx: *anyopaque) f64 {
        const self: *SugarDecorator = @ptrCast(@alignCast(ctx));
        return self.inner.getCost() + self.cost_extra;
    }

    fn getDescription(ctx: *anyopaque) []const u8 {
        const self: *SugarDecorator = @ptrCast(@alignCast(ctx));
        _ = self;
        return "กาแฟหวาน";
    }
};

// Logging Decorator สำหรับ functions
fn LoggedFn(comptime T: type) type {
    return struct {
        const Self = @This();
        inner: T,
        name: []const u8,
        call_count: u32 = 0,

        pub fn call(self: *Self, args: anytype) @typeInfo(T).Fn.return_type.? {
            self.call_count += 1;
            std.debug.print("[{s}] call #{d}\n", .{ self.name, self.call_count });
            return @call(.auto, self.inner, args);
        }
    };
}

pub fn main() !void {
    std.debug.print("=== Decorator Pattern Demo ===\n", .{});

    var base = SimpleCoffee{};
    var coffee = base.getCoffee();
    std.debug.print("{s}: {d:.0} บาท\n", .{ coffee.getDescription(), coffee.getCost() });

    var with_milk = MilkDecorator{ .inner = coffee };
    var milk_coffee = with_milk.getCoffee();
    std.debug.print("{s}: {d:.0} บาท\n", .{ milk_coffee.getDescription(), milk_coffee.getCost() });

    var with_sugar = SugarDecorator{ .inner = milk_coffee };
    var final_coffee = with_sugar.getCoffee();
    std.debug.print("{s}+น้ำตาล: {d:.0} บาท\n", .{ final_coffee.getDescription(), final_coffee.getCost() });
}
```

---

## ขั้นตอนที่ 759: Pipeline Pattern

Pipeline Pattern ประมวลผลข้อมูลผ่านชุดของขั้นตอนที่เชื่อมต่อกัน

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Data Pipeline
fn Pipeline(comptime In: type, comptime Out: type) type {
    return struct {
        const Self = @This();
        const StepFn = *const fn (Allocator, In) anyerror!Out;

        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return Self{ .allocator = allocator };
        }
    };
}

// Text Processing Pipeline
const TextProcessor = struct {
    allocator: Allocator,

    pub fn init(allocator: Allocator) TextProcessor {
        return TextProcessor{ .allocator = allocator };
    }

    // Step 1: Trim whitespace
    pub fn trim(_: Allocator, input: []const u8) []const u8 {
        return std.mem.trim(u8, input, " \t\n\r");
    }

    // Step 2: Convert to lowercase
    pub fn toLower(allocator: Allocator, input: []const u8) ![]u8 {
        const result = try allocator.dupe(u8, input);
        for (result) |*c| {
            c.* = std.ascii.toLower(c.*);
        }
        return result;
    }

    // Step 3: Replace spaces with underscores
    pub fn spacesToUnderscore(allocator: Allocator, input: []const u8) ![]u8 {
        const result = try allocator.dupe(u8, input);
        for (result) |*c| {
            if (c.* == ' ') c.* = '_';
        }
        return result;
    }

    // Step 4: Remove special characters
    pub fn removeSpecial(allocator: Allocator, input: []const u8) ![]u8 {
        var output = std.ArrayList(u8).init(allocator);
        for (input) |c| {
            if (std.ascii.isAlphanumeric(c) or c == '_') {
                try output.append(c);
            }
        }
        return output.toOwnedSlice();
    }

    // Run full pipeline
    pub fn process(self: *TextProcessor, input: []const u8) ![]u8 {
        const trimmed = trim(self.allocator, input);
        const lowered = try toLower(self.allocator, trimmed);
        defer self.allocator.free(lowered);
        const underscored = try spacesToUnderscore(self.allocator, lowered);
        defer self.allocator.free(underscored);
        return removeSpecial(self.allocator, underscored);
    }
};

// Number Processing Pipeline
const NumberPipeline = struct {
    allocator: Allocator,
    data: []f64,

    pub fn init(allocator: Allocator, data: []f64) NumberPipeline {
        return NumberPipeline{ .allocator = allocator, .data = data };
    }

    pub fn filter(self: *NumberPipeline, predicate: *const fn (f64) bool) !*NumberPipeline {
        var result = std.ArrayList(f64).init(self.allocator);
        for (self.data) |val| {
            if (predicate(val)) try result.append(val);
        }
        self.allocator.free(self.data);
        self.data = try result.toOwnedSlice();
        return self;
    }

    pub fn map(self: *NumberPipeline, transform: *const fn (f64) f64) !*NumberPipeline {
        for (self.data) |*val| {
            val.* = transform(val.*);
        }
        return self;
    }

    pub fn reduce(self: *NumberPipeline, initial: f64, reducer: *const fn (f64, f64) f64) f64 {
        var acc = initial;
        for (self.data) |val| {
            acc = reducer(acc, val);
        }
        return acc;
    }

    pub fn sum(self: *NumberPipeline) f64 {
        return self.reduce(0.0, struct {
            fn f(a: f64, b: f64) f64 {
                return a + b;
            }
        }.f);
    }

    pub fn deinit(self: *NumberPipeline) void {
        self.allocator.free(self.data);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Pipeline Pattern Demo ===\n", .{});

    // Text Pipeline
    var processor = TextProcessor.init(allocator);
    const inputs = [_][]const u8{
        "  Hello World!  ",
        "Zig Programming Language",
        "  Data Structures & Algorithms  ",
    };

    std.debug.print("\nText Processing Pipeline:\n", .{});
    for (inputs) |input| {
        const result = try processor.process(input);
        defer allocator.free(result);
        std.debug.print("  '{s}' -> '{s}'\n", .{ input, result });
    }

    // Number Pipeline
    std.debug.print("\nNumber Pipeline:\n", .{});
    var numbers = try allocator.dupe(f64, &[_]f64{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 });
    var pipeline = NumberPipeline.init(allocator, numbers);

    _ = try pipeline.filter(struct {
        fn f(x: f64) bool { return @mod(x, 2.0) == 0.0; }
    }.f);
    _ = try pipeline.map(struct {
        fn f(x: f64) f64 { return x * x; }
    }.f);

    const total = pipeline.sum();
    std.debug.print("เลขคู่ยกกำลังสอง รวม = {d:.0}\n", .{total});
    pipeline.deinit();
}
```

---

## ขั้นตอนที่ 760: Event System Pattern

Event System Pattern เป็นระบบที่ใช้จัดการ events แบบ decoupled ระหว่าง components

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Event System
const EventId = u32;

const EventHandler = struct {
    id: EventId,
    context: *anyopaque,
    handleFn: *const fn (ctx: *anyopaque, data: *const anyopaque) void,

    pub fn handle(self: EventHandler, data: *const anyopaque) void {
        self.handleFn(self.context, data);
    }
};

const EventBus = struct {
    allocator: Allocator,
    handlers: std.StringHashMap(std.ArrayList(EventHandler)),
    next_id: EventId = 0,

    pub fn init(allocator: Allocator) EventBus {
        return EventBus{
            .allocator = allocator,
            .handlers = std.StringHashMap(std.ArrayList(EventHandler)).init(allocator),
        };
    }

    pub fn deinit(self: *EventBus) void {
        var iter = self.handlers.valueIterator();
        while (iter.next()) |list| {
            list.deinit();
        }
        self.handlers.deinit();
    }

    pub fn subscribe(self: *EventBus, event_name: []const u8, handler: EventHandler) !EventId {
        const result = try self.handlers.getOrPut(event_name);
        if (!result.found_existing) {
            result.value_ptr.* = std.ArrayList(EventHandler).init(self.allocator);
        }
        try result.value_ptr.append(handler);
        return handler.id;
    }

    pub fn unsubscribe(self: *EventBus, event_name: []const u8, handler_id: EventId) void {
        if (self.handlers.get(event_name)) |*list| {
            var i: usize = 0;
            while (i < list.items.len) {
                if (list.items[i].id == handler_id) {
                    _ = list.orderedRemove(i);
                } else {
                    i += 1;
                }
            }
        }
    }

    pub fn publish(self: *EventBus, event_name: []const u8, data: *const anyopaque) void {
        if (self.handlers.get(event_name)) |list| {
            for (list.items) |handler| {
                handler.handle(data);
            }
        }
    }

    pub fn nextId(self: *EventBus) EventId {
        const id = self.next_id;
        self.next_id += 1;
        return id;
    }
};

// User Events
const UserRegisteredEvent = struct {
    user_id: u64,
    email: []const u8,
    username: []const u8,
};

const UserLoginEvent = struct {
    user_id: u64,
    ip_address: []const u8,
    timestamp: i64,
};

// Email Service (subscribes to events)
const EmailService = struct {
    sent_count: u32 = 0,
    bus: *EventBus,

    pub fn init(bus: *EventBus) EmailService {
        return EmailService{ .bus = bus };
    }

    pub fn subscribeToRegistration(self: *EmailService) !void {
        const handler = EventHandler{
            .id = self.bus.nextId(),
            .context = self,
            .handleFn = onUserRegistered,
        };
        _ = try self.bus.subscribe("user.registered", handler);
    }

    fn onUserRegistered(ctx: *anyopaque, data: *const anyopaque) void {
        const self: *EmailService = @ptrCast(@alignCast(ctx));
        const event: *const UserRegisteredEvent = @ptrCast(@alignCast(data));
        self.sent_count += 1;
        std.debug.print("[EmailService] ส่งอีเมลยืนยันไปยัง {s} (User ID: {d})\n", .{
            event.email,
            event.user_id,
        });
    }
};

// Analytics Service
const AnalyticsService = struct {
    event_count: u32 = 0,
    bus: *EventBus,

    pub fn init(bus: *EventBus) AnalyticsService {
        return AnalyticsService{ .bus = bus };
    }

    pub fn subscribeToAll(self: *AnalyticsService) !void {
        _ = try self.bus.subscribe("user.registered", EventHandler{
            .id = self.bus.nextId(),
            .context = self,
            .handleFn = onAnyEvent,
        });
        _ = try self.bus.subscribe("user.login", EventHandler{
            .id = self.bus.nextId(),
            .context = self,
            .handleFn = onAnyEvent,
        });
    }

    fn onAnyEvent(ctx: *anyopaque, _: *const anyopaque) void {
        const self: *AnalyticsService = @ptrCast(@alignCast(ctx));
        self.event_count += 1;
        std.debug.print("[Analytics] บันทึก event #{d}\n", .{self.event_count});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Event System Pattern Demo ===\n", .{});

    var bus = EventBus.init(allocator);
    defer bus.deinit();

    var email_service = EmailService.init(&bus);
    var analytics = AnalyticsService.init(&bus);

    try email_service.subscribeToRegistration();
    try analytics.subscribeToAll();

    // Publish events
    const reg_event = UserRegisteredEvent{
        .user_id = 1001,
        .email = "user@example.com",
        .username = "user1",
    };

    std.debug.print("\n[Event] user.registered\n", .{});
    bus.publish("user.registered", &reg_event);

    const login_event = UserLoginEvent{
        .user_id = 1001,
        .ip_address = "192.168.1.1",
        .timestamp = std.time.milliTimestamp(),
    };

    std.debug.print("\n[Event] user.login\n", .{});
    bus.publish("user.login", &login_event);

    std.debug.print("\nสรุป: ส่งอีเมล {d} ฉบับ, บันทึก {d} events\n", .{
        email_service.sent_count,
        analytics.event_count,
    });
}
```

---

## สรุปส่วนที่ 76: Design Patterns

| Pattern | วัตถุประสงค์หลัก | ใช้เมื่อ |
|---------|----------------|---------|
| Factory | สร้าง objects | ต้องการ flexibility ในการสร้าง objects |
| Builder | สร้าง objects ซับซ้อน | Object มีหลาย configuration |
| Observer | Notification system | ต้องการ loose coupling ระหว่าง components |
| Strategy | เปลี่ยน algorithm | มีหลาย algorithm สำหรับงานเดียวกัน |
| Command | Encapsulate actions | ต้องการ undo/redo หรือ queue actions |
| State Machine | จัดการ states | Object มีหลายสถานะและ transition |
| Iterator | Traverse collections | ต้องการ abstract การเข้าถึง elements |
| Decorator | เพิ่ม functionality | ต้องการเพิ่มความสามารถแบบ dynamic |
| Pipeline | ประมวลผลต่อเนื่อง | ข้อมูลต้องผ่านหลายขั้นตอน |
| Event System | Decoupled communication | Components ต้องสื่อสารแบบ loose coupling |

### ข้อควรจำสำหรับ Zig

1. ใช้ `union(enum)` แทน interface/virtual dispatch
2. ใช้ function pointers สำหรับ callback-based patterns
3. `anyopaque` pointer สำหรับ type erasure
4. Memory management ต้องจัดการเอง - ระวัง leaks
5. Compile-time generics ด้วย `comptime` ช่วยได้มาก

---

*ส่วนถัดไป: Part 77 - Data Structures: Linked List*
