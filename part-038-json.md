# Part 38: std.json - JSON Handling ใน Zig

## บทนำ

`std.json` คือ standard library ของ Zig สำหรับการ parse และ serialize JSON ซึ่งมาพร้อมกับ Zig ในตัว โดยไม่ต้องติดตั้ง library เพิ่มเติม บทนี้จะครอบคลุมการใช้งาน JSON ตั้งแต่พื้นฐานจนถึงการใช้งานจริงในระดับ production

---

## Step 371: JSON Parsing พื้นฐาน

### JSON คืออะไร?

JSON (JavaScript Object Notation) เป็น format สำหรับแลกเปลี่ยนข้อมูล ที่มนุษย์อ่านง่ายและเครื่องประมวลผลง่าย

```json
{
  "name": "Alice",
  "age": 30,
  "active": true,
  "scores": [95, 87, 92],
  "address": {
    "city": "Bangkok",
    "country": "Thailand"
  }
}
```

### การ Parse JSON พื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{"name":"Alice","age":30,"active":true}
    ;

    // Parse JSON เป็น dynamic value
    const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, .{});
    defer parsed.deinit();

    // เข้าถึง root object
    const root = parsed.value;

    switch (root) {
        .object => |obj| {
            std.debug.print("Keys in JSON object:\n", .{});
            var iter = obj.iterator();
            while (iter.next()) |entry| {
                std.debug.print("  {s}: ", .{entry.key_ptr.*});
                switch (entry.value_ptr.*) {
                    .string => |s| std.debug.print("{s}\n", .{s}),
                    .integer => |n| std.debug.print("{}\n", .{n}),
                    .bool => |b| std.debug.print("{}\n", .{b}),
                    else => std.debug.print("(other type)\n", .{}),
                }
            }
        },
        else => std.debug.print("Not an object\n", .{}),
    }
}
```

### std.json.Value Types

```zig
const std = @import("std");

pub fn printValue(value: std.json.Value, indent: usize) void {
    const spaces = "                "; // 16 spaces for indentation
    const pad = spaces[0..@min(indent * 2, spaces.len)];

    switch (value) {
        .null => std.debug.print("null", .{}),
        .bool => |b| std.debug.print("{}", .{b}),
        .integer => |n| std.debug.print("{}", .{n}),
        .float => |f| std.debug.print("{d}", .{f}),
        .number_string => |s| std.debug.print("{s}", .{s}),
        .string => |s| std.debug.print("\"{s}\"", .{s}),
        .array => |arr| {
            std.debug.print("[\n", .{});
            for (arr.items) |item| {
                std.debug.print("{s}  ", .{pad});
                printValue(item, indent + 1);
                std.debug.print(",\n", .{});
            }
            std.debug.print("{s}]", .{pad});
        },
        .object => |obj| {
            std.debug.print("{{\n", .{});
            var iter = obj.iterator();
            while (iter.next()) |entry| {
                std.debug.print("{s}  \"{s}\": ", .{ pad, entry.key_ptr.* });
                printValue(entry.value_ptr.*, indent + 1);
                std.debug.print(",\n", .{});
            }
            std.debug.print("{s}}}", .{pad});
        },
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{
        \\  "name": "Bob",
        \\  "age": 25,
        \\  "scores": [90, 85, 92],
        \\  "address": {"city": "Bangkok"},
        \\  "active": true,
        \\  "balance": 1234.56
        \\}
    ;

    const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, .{});
    defer parsed.deinit();

    printValue(parsed.value, 0);
    std.debug.print("\n", .{});
}
```

---

## Step 372: std.json.parseFromSlice

### Parse Options

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{
        \\  "name": "Charlie",
        \\  "items": [1, 2, 3],
        \\  "nested": {"key": "value"}
        \\}
    ;

    // ParseOptions ที่สำคัญ
    const options = std.json.ParseOptions{
        .duplicate_field_behavior = .use_last,  // จัดการ duplicate keys
        .ignore_unknown_fields = false,          // error ถ้า field ไม่รู้จัก
        .max_value_len = null,                   // ไม่จำกัดความยาว
    };

    const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, options);
    defer parsed.deinit();

    // เข้าถึงค่าต่างๆ
    const root = parsed.value.object;

    if (root.get("name")) |name| {
        std.debug.print("Name: {s}\n", .{name.string});
    }

    if (root.get("items")) |items| {
        std.debug.print("Items: ", .{});
        for (items.array.items) |item| {
            std.debug.print("{} ", .{item.integer});
        }
        std.debug.print("\n", .{});
    }

    if (root.get("nested")) |nested| {
        if (nested.object.get("key")) |key| {
            std.debug.print("nested.key = {s}\n", .{key.string});
        }
    }
}
```

### การ Navigate JSON Tree

```zig
const std = @import("std");

// Helper function สำหรับ navigate JSON
fn getField(value: std.json.Value, path: []const u8) ?std.json.Value {
    var current = value;
    var iter = std.mem.tokenizeScalar(u8, path, '.');

    while (iter.next()) |key| {
        switch (current) {
            .object => |obj| {
                // Try to parse as array index
                if (std.fmt.parseInt(usize, key, 10)) |_| {
                    return null; // objects don't use numeric keys like this
                } else |_| {}

                current = obj.get(key) orelse return null;
            },
            .array => |arr| {
                const idx = std.fmt.parseInt(usize, key, 10) catch return null;
                if (idx >= arr.items.len) return null;
                current = arr.items[idx];
            },
            else => return null,
        }
    }

    return current;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{
        \\  "user": {
        \\    "name": "Dave",
        \\    "address": {
        \\      "city": "Chiang Mai",
        \\      "zip": "50000"
        \\    },
        \\    "phones": ["081-234-5678", "02-345-6789"]
        \\  }
        \\}
    ;

    const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, .{});
    defer parsed.deinit();

    const paths = [_][]const u8{
        "user.name",
        "user.address.city",
        "user.address.zip",
        "user.phones.0",
        "user.phones.1",
        "user.nonexistent",
    };

    for (paths) |path| {
        if (getField(parsed.value, path)) |val| {
            switch (val) {
                .string => |s| std.debug.print("{s} = {s}\n", .{ path, s }),
                .integer => |n| std.debug.print("{s} = {}\n", .{ path, n }),
                else => std.debug.print("{s} = (other)\n", .{path}),
            }
        } else {
            std.debug.print("{s} = (not found)\n", .{path});
        }
    }
}
```

---

## Step 373: JSON to Struct

### Parsing เป็น Struct

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    age: u32,
    email: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{"name":"Alice","age":28,"email":"alice@example.com"}
    ;

    // Parse โดยตรงเป็น struct
    const parsed = try std.json.parseFromSlice(User, allocator, json_str, .{});
    defer parsed.deinit();

    const user = parsed.value;
    std.debug.print("Name: {s}\n", .{user.name});
    std.debug.print("Age: {}\n", .{user.age});
    std.debug.print("Email: {s}\n", .{user.email});
}
```

### Struct ที่มี Optional Fields

```zig
const std = @import("std");

const Product = struct {
    id: u64,
    name: []const u8,
    price: f64,
    description: ?[]const u8,  // optional field
    tags: ?[][]const u8,       // optional array
    in_stock: bool = true,     // default value
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // JSON พร้อม optional fields
    const json1 =
        \\{
        \\  "id": 1,
        \\  "name": "Laptop",
        \\  "price": 25000.0,
        \\  "description": "High performance laptop",
        \\  "tags": ["electronics", "computer"],
        \\  "in_stock": true
        \\}
    ;

    const json2 =
        \\{
        \\  "id": 2,
        \\  "name": "Keyboard",
        \\  "price": 1500.0
        \\}
    ;

    const p1 = try std.json.parseFromSlice(Product, allocator, json1, .{
        .ignore_unknown_fields = true,
    });
    defer p1.deinit();

    const p2 = try std.json.parseFromSlice(Product, allocator, json2, .{
        .ignore_unknown_fields = true,
    });
    defer p2.deinit();

    std.debug.print("Product 1:\n", .{});
    std.debug.print("  ID: {}\n", .{p1.value.id});
    std.debug.print("  Name: {s}\n", .{p1.value.name});
    std.debug.print("  Price: {d:.2}\n", .{p1.value.price});
    if (p1.value.description) |desc| {
        std.debug.print("  Description: {s}\n", .{desc});
    }
    if (p1.value.tags) |tags| {
        std.debug.print("  Tags: ", .{});
        for (tags) |tag| std.debug.print("{s} ", .{tag});
        std.debug.print("\n", .{});
    }

    std.debug.print("\nProduct 2:\n", .{});
    std.debug.print("  ID: {}\n", .{p2.value.id});
    std.debug.print("  Name: {s}\n", .{p2.value.name});
    std.debug.print("  Description: {s}\n", .{if (p2.value.description) |d| d else "(none)"});
    std.debug.print("  In stock (default): {}\n", .{p2.value.in_stock});
}
```

### Nested Struct Parsing

```zig
const std = @import("std");

const Address = struct {
    street: []const u8,
    city: []const u8,
    country: []const u8,
    zip: []const u8,
};

const Contact = struct {
    type: []const u8,
    value: []const u8,
};

const Person = struct {
    id: u64,
    first_name: []const u8,
    last_name: []const u8,
    age: u8,
    address: Address,
    contacts: []Contact,
    active: bool,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{
        \\  "id": 1001,
        \\  "first_name": "สมชาย",
        \\  "last_name": "ใจดี",
        \\  "age": 35,
        \\  "address": {
        \\    "street": "123 ถนนสุขุมวิท",
        \\    "city": "กรุงเทพ",
        \\    "country": "ไทย",
        \\    "zip": "10110"
        \\  },
        \\  "contacts": [
        \\    {"type": "email", "value": "somchai@example.com"},
        \\    {"type": "phone", "value": "081-234-5678"}
        \\  ],
        \\  "active": true
        \\}
    ;

    const person = try std.json.parseFromSlice(Person, allocator, json_str, .{});
    defer person.deinit();

    const p = person.value;
    std.debug.print("=== Person Record ===\n", .{});
    std.debug.print("ID: {}\n", .{p.id});
    std.debug.print("Name: {s} {s}\n", .{ p.first_name, p.last_name });
    std.debug.print("Age: {}\n", .{p.age});
    std.debug.print("Address: {s}, {s}, {s} {s}\n", .{
        p.address.street,
        p.address.city,
        p.address.country,
        p.address.zip,
    });
    std.debug.print("Contacts:\n", .{});
    for (p.contacts) |contact| {
        std.debug.print("  {s}: {s}\n", .{ contact.type, contact.value });
    }
    std.debug.print("Active: {}\n", .{p.active});
}
```

---

## Step 374: JSON Serialization

### stringify พื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Serialize primitive values
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    // Integer
    try std.json.stringify(42, .{}, buf.writer());
    std.debug.print("Integer: {s}\n", .{buf.items});
    buf.clearRetainingCapacity();

    // String
    try std.json.stringify("Hello, World!", .{}, buf.writer());
    std.debug.print("String: {s}\n", .{buf.items});
    buf.clearRetainingCapacity();

    // Boolean
    try std.json.stringify(true, .{}, buf.writer());
    std.debug.print("Bool: {s}\n", .{buf.items});
    buf.clearRetainingCapacity();

    // Array
    try std.json.stringify([_]i32{ 1, 2, 3 }, .{}, buf.writer());
    std.debug.print("Array: {s}\n", .{buf.items});
    buf.clearRetainingCapacity();

    // Null
    try std.json.stringify(null, .{}, buf.writer());
    std.debug.print("Null: {s}\n", .{buf.items});
}
```

### Serialize Struct

```zig
const std = @import("std");

const Config = struct {
    host: []const u8,
    port: u16,
    max_connections: u32,
    timeout_ms: u64,
    debug: bool,
    allowed_origins: []const []const u8,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const origins = [_][]const u8{ "https://example.com", "https://api.example.com" };

    const config = Config{
        .host = "localhost",
        .port = 8080,
        .max_connections = 1000,
        .timeout_ms = 30000,
        .debug = false,
        .allowed_origins = &origins,
    };

    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    // stringify ธรรมดา (compact)
    try std.json.stringify(config, .{}, buf.writer());
    std.debug.print("Compact:\n{s}\n\n", .{buf.items});
    buf.clearRetainingCapacity();

    // stringify พร้อม pretty print
    try std.json.stringify(config, .{ .whitespace = .indent_2 }, buf.writer());
    std.debug.print("Pretty:\n{s}\n", .{buf.items});
}
```

### Custom JSON Marshaling

```zig
const std = @import("std");

const Point = struct {
    x: f64,
    y: f64,

    pub fn jsonStringify(self: Point, jws: anytype) !void {
        try jws.beginArray();
        try jws.write(self.x);
        try jws.write(self.y);
        try jws.endArray();
    }
};

const Rectangle = struct {
    top_left: Point,
    bottom_right: Point,
    color: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const rect = Rectangle{
        .top_left = .{ .x = 0.0, .y = 0.0 },
        .bottom_right = .{ .x = 100.0, .y = 50.0 },
        .color = "red",
    };

    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    try std.json.stringify(rect, .{ .whitespace = .indent_2 }, buf.writer());
    std.debug.print("{s}\n", .{buf.items});
}
```

---

## Step 375: Nested JSON Structures

### Complex Nested JSON

```zig
const std = @import("std");

const Metadata = struct {
    created_at: []const u8,
    updated_at: []const u8,
    version: u32,
};

const Permissions = struct {
    read: bool,
    write: bool,
    admin: bool,
};

const Role = struct {
    name: []const u8,
    permissions: Permissions,
};

const UserProfile = struct {
    id: u64,
    username: []const u8,
    email: []const u8,
    roles: []Role,
    metadata: Metadata,
    settings: ?std.json.Value,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{
        \\  "id": 42,
        \\  "username": "admin_user",
        \\  "email": "admin@company.com",
        \\  "roles": [
        \\    {
        \\      "name": "admin",
        \\      "permissions": {"read": true, "write": true, "admin": true}
        \\    },
        \\    {
        \\      "name": "editor",
        \\      "permissions": {"read": true, "write": true, "admin": false}
        \\    }
        \\  ],
        \\  "metadata": {
        \\    "created_at": "2024-01-01T00:00:00Z",
        \\    "updated_at": "2024-06-15T12:30:00Z",
        \\    "version": 3
        \\  },
        \\  "settings": {"theme": "dark", "language": "th"}
        \\}
    ;

    const profile = try std.json.parseFromSlice(UserProfile, allocator, json_str, .{
        .ignore_unknown_fields = true,
    });
    defer profile.deinit();

    const p = profile.value;
    std.debug.print("User: {} - {s} ({s})\n", .{ p.id, p.username, p.email });
    std.debug.print("Roles:\n", .{});
    for (p.roles) |role| {
        std.debug.print("  {s}: read={}, write={}, admin={}\n", .{
            role.name,
            role.permissions.read,
            role.permissions.write,
            role.permissions.admin,
        });
    }
    std.debug.print("Created: {s}\n", .{p.metadata.created_at});
    std.debug.print("Version: {}\n", .{p.metadata.version});

    if (p.settings) |settings| {
        if (settings.object.get("theme")) |theme| {
            std.debug.print("Theme: {s}\n", .{theme.string});
        }
    }
}
```

---

## Step 376: JSON Arrays

### Parse JSON Arrays

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Array of primitives
    const numbers_json = "[1, 2, 3, 4, 5]";
    const nums = try std.json.parseFromSlice([]i32, allocator, numbers_json, .{});
    defer nums.deinit();

    std.debug.print("Numbers: {any}\n", .{nums.value});

    // Array of strings
    const names_json = "[\"Alice\", \"Bob\", \"Charlie\"]";
    const names = try std.json.parseFromSlice([][]const u8, allocator, names_json, .{});
    defer names.deinit();

    std.debug.print("Names: ", .{});
    for (names.value) |name| {
        std.debug.print("{s} ", .{name});
    }
    std.debug.print("\n", .{});

    // Array of objects
    const Item = struct {
        id: u32,
        name: []const u8,
        price: f64,
    };

    const items_json =
        \\[
        \\  {"id": 1, "name": "Apple", "price": 15.0},
        \\  {"id": 2, "name": "Banana", "price": 8.5},
        \\  {"id": 3, "name": "Cherry", "price": 120.0}
        \\]
    ;

    const items = try std.json.parseFromSlice([]Item, allocator, items_json, .{});
    defer items.deinit();

    var total: f64 = 0;
    for (items.value) |item| {
        std.debug.print("  {s}: {d:.2} THB\n", .{ item.name, item.price });
        total += item.price;
    }
    std.debug.print("Total: {d:.2} THB\n", .{total});
}
```

### Build JSON Array

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    // สร้าง JSON array แบบ manual
    var jws = std.json.writeStream(buf.writer(), .{ .whitespace = .indent_2 });

    try jws.beginArray();

    const data = [_]struct { name: []const u8, score: u32 }{
        .{ .name = "Alice", .score = 95 },
        .{ .name = "Bob", .score = 87 },
        .{ .name = "Charlie", .score = 92 },
    };

    for (data) |item| {
        try jws.beginObject();
        try jws.objectField("name");
        try jws.write(item.name);
        try jws.objectField("score");
        try jws.write(item.score);
        try jws.objectField("grade");
        try jws.write(if (item.score >= 90) "A" else "B");
        try jws.endObject();
    }

    try jws.endArray();

    std.debug.print("{s}\n", .{buf.items});
}
```

---

## Step 377: JSON Error Handling

### ประเภทของ Errors

```zig
const std = @import("std");

pub fn safeParseJson(allocator: std.mem.Allocator, json_str: []const u8) void {
    const result = std.json.parseFromSlice(std.json.Value, allocator, json_str, .{}) catch |err| {
        switch (err) {
            error.SyntaxError => std.debug.print("JSON syntax error\n", .{}),
            error.UnexpectedEndOfInput => std.debug.print("Unexpected end of JSON\n", .{}),
            error.InvalidCharacter => std.debug.print("Invalid character in JSON\n", .{}),
            error.OutOfMemory => std.debug.print("Out of memory\n", .{}),
            else => std.debug.print("Unknown error: {}\n", .{err}),
        }
        return;
    };
    defer result.deinit();

    std.debug.print("Parsed successfully\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("Valid JSON: ", .{});
    safeParseJson(allocator, "{\"key\": \"value\"}");

    std.debug.print("Missing closing brace: ", .{});
    safeParseJson(allocator, "{\"key\": \"value\"");

    std.debug.print("Invalid JSON: ", .{});
    safeParseJson(allocator, "not json at all");

    std.debug.print("Trailing comma: ", .{});
    safeParseJson(allocator, "{\"key\": \"value\",}");
}
```

### Validation ก่อน Parse

```zig
const std = @import("std");

const ValidationError = error{
    MissingRequiredField,
    InvalidFieldType,
    ValueOutOfRange,
};

pub fn validateUserJson(value: std.json.Value) !void {
    const obj = switch (value) {
        .object => |o| o,
        else => return ValidationError.InvalidFieldType,
    };

    // Check required fields
    const required = [_][]const u8{ "id", "name", "email" };
    for (required) |field| {
        if (!obj.contains(field)) {
            std.debug.print("Missing required field: {s}\n", .{field});
            return ValidationError.MissingRequiredField;
        }
    }

    // Validate types
    if (obj.get("id")) |id_val| {
        if (id_val != .integer) return ValidationError.InvalidFieldType;
        if (id_val.integer <= 0) return ValidationError.ValueOutOfRange;
    }

    if (obj.get("name")) |name_val| {
        if (name_val != .string) return ValidationError.InvalidFieldType;
        if (name_val.string.len == 0) return ValidationError.ValueOutOfRange;
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const valid_json = "{\"id\":1,\"name\":\"Alice\",\"email\":\"alice@example.com\"}";
    const missing_field_json = "{\"id\":1,\"name\":\"Alice\"}";
    const invalid_type_json = "{\"id\":\"not-a-number\",\"name\":\"Alice\",\"email\":\"a@b.com\"}";

    const test_cases = [_][]const u8{ valid_json, missing_field_json, invalid_type_json };
    const labels = [_][]const u8{ "Valid", "Missing field", "Invalid type" };

    for (test_cases, labels) |json_str, label| {
        std.debug.print("{s}: ", .{label});
        const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, .{});
        defer parsed.deinit();

        validateUserJson(parsed.value) catch |err| {
            std.debug.print("Validation error: {}\n", .{err});
            continue;
        };
        std.debug.print("OK\n", .{});
    }
}
```

---

## Step 378: Streaming JSON Parser

### ใช้ Scanner สำหรับ Streaming

```zig
const std = @import("std");

pub fn processLargeJson(allocator: std.mem.Allocator, json_str: []const u8) !void {
    var scanner = std.json.Scanner.initCompleteInput(allocator, json_str);
    defer scanner.deinit();

    var depth: usize = 0;
    var token_count: usize = 0;

    while (true) {
        const token = try scanner.next();
        token_count += 1;

        switch (token) {
            .end_of_document => break,
            .object_begin => {
                depth += 1;
                std.debug.print("Object start (depth: {})\n", .{depth});
            },
            .object_end => {
                depth -= 1;
                std.debug.print("Object end (depth: {})\n", .{depth});
            },
            .array_begin => {
                depth += 1;
                std.debug.print("Array start (depth: {})\n", .{depth});
            },
            .array_end => {
                depth -= 1;
                std.debug.print("Array end (depth: {})\n", .{depth});
            },
            .string => |s| std.debug.print("String: {s}\n", .{s}),
            .number => |n| std.debug.print("Number: {s}\n", .{n}),
            .true => std.debug.print("Bool: true\n", .{}),
            .false => std.debug.print("Bool: false\n", .{}),
            .null => std.debug.print("Null\n", .{}),
        }
    }

    std.debug.print("Total tokens: {}\n", .{token_count});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const json_str =
        \\{"name":"Alice","scores":[90,85],"address":{"city":"Bangkok"}}
    ;

    try processLargeJson(allocator, json_str);
}
```

### Streaming Write

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();

    // ใช้ writeStream สำหรับ streaming JSON generation
    var stream = std.json.writeStream(output.writer(), .{ .whitespace = .indent_2 });

    try stream.beginObject();

    try stream.objectField("version");
    try stream.write("1.0");

    try stream.objectField("timestamp");
    try stream.write(1234567890);

    try stream.objectField("events");
    try stream.beginArray();

    const events = [_]struct { id: u32, name: []const u8, value: f64 }{
        .{ .id = 1, .name = "click", .value = 0.95 },
        .{ .id = 2, .name = "hover", .value = 0.5 },
        .{ .id = 3, .name = "scroll", .value = 0.75 },
    };

    for (events) |event| {
        try stream.beginObject();
        try stream.objectField("id");
        try stream.write(event.id);
        try stream.objectField("name");
        try stream.write(event.name);
        try stream.objectField("value");
        try stream.write(event.value);
        try stream.endObject();
    }

    try stream.endArray();
    try stream.endObject();

    std.debug.print("{s}\n", .{output.items});
}
```

---

## Step 379: Custom JSON Types

### Enum Serialization

```zig
const std = @import("std");

const Status = enum {
    pending,
    active,
    inactive,
    deleted,

    pub fn jsonStringify(self: Status, jws: anytype) !void {
        try jws.write(@tagName(self));
    }

    pub fn jsonParse(
        allocator: std.mem.Allocator,
        source: anytype,
        options: std.json.ParseOptions,
    ) !Status {
        _ = options;
        _ = allocator;

        const s = try source.nextAllocMax(allocator, .alloc_if_needed, std.json.default_max_value_len);
        defer switch (s) {
            .allocated_slice => |v| allocator.free(v),
            else => {},
        };
        const str = switch (s) {
            .allocated_slice => |v| v,
            .string => |v| v,
            else => return error.UnexpectedToken,
        };

        if (std.mem.eql(u8, str, "pending")) return .pending;
        if (std.mem.eql(u8, str, "active")) return .active;
        if (std.mem.eql(u8, str, "inactive")) return .inactive;
        if (std.mem.eql(u8, str, "deleted")) return .deleted;

        return error.InvalidEnumTag;
    }
};

const Task = struct {
    id: u32,
    title: []const u8,
    status: Status,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const task = Task{ .id = 1, .title = "Learn Zig", .status = .active };

    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    try std.json.stringify(task, .{}, buf.writer());
    std.debug.print("Serialized: {s}\n", .{buf.items});

    // Parse กลับ
    const parsed = try std.json.parseFromSlice(Task, allocator, buf.items, .{});
    defer parsed.deinit();

    std.debug.print("Parsed status: {s}\n", .{@tagName(parsed.value.status)});
}
```

### Tagged Union JSON

```zig
const std = @import("std");

const Shape = union(enum) {
    circle: struct { radius: f64 },
    rectangle: struct { width: f64, height: f64 },
    triangle: struct { base: f64, height: f64 },

    pub fn area(self: Shape) f64 {
        return switch (self) {
            .circle => |c| std.math.pi * c.radius * c.radius,
            .rectangle => |r| r.width * r.height,
            .triangle => |t| 0.5 * t.base * t.height,
        };
    }
};

pub fn parseShape(allocator: std.mem.Allocator, json_str: []const u8) !Shape {
    const parsed = try std.json.parseFromSlice(std.json.Value, allocator, json_str, .{});
    defer parsed.deinit();

    const obj = parsed.value.object;
    const shape_type = obj.get("type") orelse return error.MissingType;

    if (std.mem.eql(u8, shape_type.string, "circle")) {
        const radius = obj.get("radius") orelse return error.MissingField;
        return .{ .circle = .{ .radius = radius.float } };
    } else if (std.mem.eql(u8, shape_type.string, "rectangle")) {
        const w = obj.get("width") orelse return error.MissingField;
        const h = obj.get("height") orelse return error.MissingField;
        return .{ .rectangle = .{ .width = w.float, .height = h.float } };
    } else if (std.mem.eql(u8, shape_type.string, "triangle")) {
        const b = obj.get("base") orelse return error.MissingField;
        const h = obj.get("height") orelse return error.MissingField;
        return .{ .triangle = .{ .base = b.float, .height = h.float } };
    }

    return error.UnknownShapeType;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const shapes_json = [_][]const u8{
        "{\"type\":\"circle\",\"radius\":5.0}",
        "{\"type\":\"rectangle\",\"width\":10.0,\"height\":4.0}",
        "{\"type\":\"triangle\",\"base\":6.0,\"height\":3.0}",
    };

    for (shapes_json) |json_str| {
        const shape = try parseShape(allocator, json_str);
        std.debug.print("{s} area: {d:.2}\n", .{ @tagName(shape), shape.area() });
    }
}
```

---

## Step 380: Real-world JSON Patterns (API Response)

### REST API Response Pattern

```zig
const std = @import("std");

fn ApiResponse(comptime T: type) type {
    return struct {
        success: bool,
        data: ?T,
        error_message: ?[]const u8,
        pagination: ?Pagination,

        const Pagination = struct {
            page: u32,
            per_page: u32,
            total: u32,
            total_pages: u32,
        };
    };
}

const UserSummary = struct {
    id: u64,
    username: []const u8,
    email: []const u8,
    created_at: []const u8,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const api_response_json =
        \\{
        \\  "success": true,
        \\  "data": [
        \\    {"id": 1, "username": "alice", "email": "alice@example.com", "created_at": "2024-01-01"},
        \\    {"id": 2, "username": "bob", "email": "bob@example.com", "created_at": "2024-01-05"},
        \\    {"id": 3, "username": "charlie", "email": "charlie@example.com", "created_at": "2024-01-10"}
        \\  ],
        \\  "error_message": null,
        \\  "pagination": {
        \\    "page": 1,
        \\    "per_page": 10,
        \\    "total": 3,
        \\    "total_pages": 1
        \\  }
        \\}
    ;

    const ResponseType = ApiResponse([]UserSummary);
    const response = try std.json.parseFromSlice(ResponseType, allocator, api_response_json, .{
        .ignore_unknown_fields = true,
    });
    defer response.deinit();

    if (response.value.success) {
        std.debug.print("API Success!\n", .{});
        if (response.value.data) |users| {
            std.debug.print("Users ({}):\n", .{users.len});
            for (users) |user| {
                std.debug.print("  [{d}] {s} - {s} (since {s})\n", .{
                    user.id, user.username, user.email, user.created_at,
                });
            }
        }
        if (response.value.pagination) |page| {
            std.debug.print("Page {}/{} (total: {})\n", .{
                page.page, page.total_pages, page.total,
            });
        }
    } else {
        if (response.value.error_message) |msg| {
            std.debug.print("API Error: {s}\n", .{msg});
        }
    }
}
```

### JSON Config File Management

```zig
const std = @import("std");

const AppConfig = struct {
    database: DatabaseConfig,
    server: ServerConfig,
    logging: LoggingConfig,

    const DatabaseConfig = struct {
        host: []const u8,
        port: u16,
        name: []const u8,
        username: []const u8,
        pool_size: u32 = 10,
    };

    const ServerConfig = struct {
        host: []const u8,
        port: u16,
        workers: u32,
        max_request_size: u64,
    };

    const LoggingConfig = struct {
        level: []const u8,
        file: ?[]const u8,
        console: bool,
    };
};

pub fn loadConfig(allocator: std.mem.Allocator, path: []const u8) !std.json.Parsed(AppConfig) {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    const content = try file.readToEndAlloc(allocator, 1024 * 1024); // max 1MB
    defer allocator.free(content);

    return std.json.parseFromSlice(AppConfig, allocator, content, .{
        .ignore_unknown_fields = true,
    });
}

pub fn saveConfig(allocator: std.mem.Allocator, config: AppConfig, path: []const u8) !void {
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();

    try std.json.stringify(config, .{ .whitespace = .indent_2 }, buf.writer());

    const file = try std.fs.cwd().createFile(path, .{});
    defer file.close();

    try file.writeAll(buf.items);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const default_config = AppConfig{
        .database = .{
            .host = "localhost",
            .port = 5432,
            .name = "myapp",
            .username = "admin",
            .pool_size = 20,
        },
        .server = .{
            .host = "0.0.0.0",
            .port = 8080,
            .workers = 4,
            .max_request_size = 10 * 1024 * 1024,
        },
        .logging = .{
            .level = "info",
            .file = "app.log",
            .console = true,
        },
    };

    // Save config
    try saveConfig(allocator, default_config, "/tmp/app_config.json");
    std.debug.print("Config saved\n", .{});

    // Load config
    const loaded = loadConfig(allocator, "/tmp/app_config.json") catch |err| {
        std.debug.print("Could not load config: {}\n", .{err});
        return;
    };
    defer loaded.deinit();

    std.debug.print("Database: {s}:{d}/{s}\n", .{
        loaded.value.database.host,
        loaded.value.database.port,
        loaded.value.database.name,
    });
    std.debug.print("Server: {s}:{d}\n", .{
        loaded.value.server.host,
        loaded.value.server.port,
    });
    std.debug.print("Log level: {s}\n", .{loaded.value.logging.level});
}
```

---

## สรุปบทที่ 38

ในบทนี้เราได้เรียนรู้:

1. **JSON Parsing พื้นฐาน** - std.json.Value types และการ navigate JSON tree
2. **parseFromSlice** - ParseOptions และการเข้าถึงค่าต่างๆ
3. **JSON to Struct** - การ parse เป็น typed struct รวมถึง optional fields และ nested structs
4. **JSON Serialization** - stringify พร้อม pretty print และ custom marshaling
5. **Nested JSON** - การจัดการ complex nested structures
6. **JSON Arrays** - parse และ build JSON arrays
7. **Error Handling** - การจัดการ errors และ validation
8. **Streaming Parser** - ใช้ Scanner สำหรับ streaming processing
9. **Custom JSON Types** - enum serialization และ tagged union
10. **Real-world Patterns** - API response, config file management

`std.json` เป็น module ที่ครบครันสำหรับการทำงานกับ JSON ใน Zig โดยรองรับทั้ง static typing ผ่าน struct และ dynamic typing ผ่าน `std.json.Value`
