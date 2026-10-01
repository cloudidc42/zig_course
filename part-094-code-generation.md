# Part 94: Code Generation ใน Zig

## Steps 931-940

---

## Step 931: Code Generation Concepts

### Code Generation คืออะไร?

Code Generation คือกระบวนการสร้างโค้ดโดยอัตโนมัติ แทนที่จะเขียนเองทั้งหมด มีหลายระดับ:

1. **Compile-time generation** - สร้าง code ขณะ compile (ด้วย comptime)
2. **Build-time generation** - สร้าง code ก่อน compile (ด้วย build scripts)
3. **Runtime generation** - สร้าง code ขณะรันโปรแกรม

### ประโยชน์ของ Code Generation

```
ลด boilerplate:         เขียนน้อยลง maintain ง่ายขึ้น
Consistency:            code ที่ generate จะมี pattern เดียวกัน
Type safety:            สร้าง type-safe APIs จาก schemas
Performance:            optimize ตาม target
Cross-language:         สร้าง bindings ระหว่างภาษา
```

### ประเภทของ Code Generation ใน Zig

```zig
const std = @import("std");

// 1. Comptime code generation
pub fn generateStructFields(comptime fields: []const []const u8) type {
    comptime var struct_fields: [fields.len]std.builtin.Type.StructField = undefined;
    inline for (fields, 0..) |field_name, i| {
        struct_fields[i] = .{
            .name = field_name,
            .type = []const u8,
            .default_value = null,
            .is_comptime = false,
            .alignment = @alignOf([]const u8),
        };
    }
    return @Type(.{
        .Struct = .{
            .layout = .auto,
            .fields = &struct_fields,
            .decls = &.{},
            .is_tuple = false,
        },
    });
}

// 2. Build-time code generation (via build.zig)
// ใช้ addWriteFile หรือ GenerateFileStep เพื่อสร้าง .zig files

// 3. Runtime code generation (string-based)
pub fn generateZigStruct(
    name: []const u8,
    fields: []const struct { name: []const u8, type: []const u8 },
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();

    try buf.appendSlice("pub const ");
    try buf.appendSlice(name);
    try buf.appendSlice(" = struct {\n");

    for (fields) |field| {
        try buf.appendSlice("    ");
        try buf.appendSlice(field.name);
        try buf.appendSlice(": ");
        try buf.appendSlice(field.type);
        try buf.appendSlice(",\n");
    }

    try buf.appendSlice("};\n");

    return buf.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Example 1: Comptime struct generation
    const DynamicStruct = generateStructFields(&[_][]const u8{ "first_name", "last_name", "email" });
    var person: DynamicStruct = undefined;
    person.first_name = "Alice";
    person.last_name = "Smith";
    person.email = "alice@example.com";
    std.debug.print("Generated struct - name: {s} {s}\n", .{ person.first_name, person.last_name });

    // Example 2: Runtime Zig code generation
    const fields = [_]struct { name: []const u8, type: []const u8 }{
        .{ .name = "id", .type = "u64" },
        .{ .name = "name", .type = "[]const u8" },
        .{ .name = "created_at", .type = "i64" },
    };
    const code = try generateZigStruct("User", &fields, allocator);
    defer allocator.free(code);
    std.debug.print("Generated code:\n{s}\n", .{code});
}
```

---

## Step 932: Generating Zig Code at Comptime

### Comptime Zig Code Generation

```zig
const std = @import("std");

// Generate getter/setter pairs for all fields
pub fn generateAccessors(comptime T: type) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            // Create a struct with all accessor methods
            comptime var methods_code: []const u8 = "";
            _ = methods_code;
            _ = s;

            return struct {
                data: T,

                const Self = @This();

                // Generate getters using comptime
                pub fn get(self: Self, comptime field_name: []const u8) @TypeOf(@field(self.data, field_name)) {
                    return @field(self.data, field_name);
                }

                // Generate setters using comptime
                pub fn set(self: *Self, comptime field_name: []const u8, value: @TypeOf(@field(self.data, field_name))) void {
                    @field(self.data, field_name) = value;
                }

                // Generate a map of all values
                pub fn toMap(self: Self, allocator: std.mem.Allocator) !std.StringHashMap([]const u8) {
                    var map = std.StringHashMap([]const u8).init(allocator);
                    errdefer map.deinit();
                    
                    inline for (s.fields) |field| {
                        const val = @field(self.data, field.name);
                        const str = try std.fmt.allocPrint(allocator, "{any}", .{val});
                        try map.put(field.name, str);
                    }
                    return map;
                }
            };
        },
        else => @compileError("generateAccessors requires struct"),
    }
}

// Generate a dispatch table
pub fn generateDispatch(comptime handlers: anytype) type {
    return struct {
        pub fn dispatch(command: []const u8, args: anytype) !void {
            const info = @typeInfo(@TypeOf(handlers));
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        if (std.mem.eql(u8, field.name, command)) {
                            const handler = @field(handlers, field.name);
                            try handler(args);
                            return;
                        }
                    }
                },
                else => {},
            }
            return error.UnknownCommand;
        }

        pub fn listCommands() []const []const u8 {
            const info = @typeInfo(@TypeOf(handlers));
            switch (info) {
                .Struct => |s| {
                    comptime var names: [s.fields.len][]const u8 = undefined;
                    inline for (s.fields, 0..) |field, i| {
                        names[i] = field.name;
                    }
                    return &names;
                },
                else => return &.{},
            }
        }
    };
}

// Generate enum from struct fields
pub fn fieldsAsEnum(comptime T: type) type {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            comptime var fields: [s.fields.len]std.builtin.Type.EnumField = undefined;
            inline for (s.fields, 0..) |field, i| {
                fields[i] = .{
                    .name = field.name,
                    .value = i,
                };
            }
            return @Type(.{
                .Enum = .{
                    .tag_type = u32,
                    .fields = &fields,
                    .decls = &.{},
                    .is_exhaustive = true,
                },
            });
        },
        else => @compileError("fieldsAsEnum requires struct"),
    }
}

// Generate type-safe field access
pub fn FieldAccessor(comptime T: type) type {
    const FieldEnum = fieldsAsEnum(T);
    
    return struct {
        pub fn get(value: T, field: FieldEnum) []const u8 {
            return switch (field) {
                inline else => |f| blk: {
                    const field_name = @tagName(f);
                    const val = @field(value, field_name);
                    _ = val;
                    break :blk "value"; // Simplified
                },
            };
        }
    };
}

const Config = struct {
    host: []const u8,
    port: u16,
    debug: bool,
};

pub fn comptimeCodegenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Generate accessors
    const AccessibleConfig = generateAccessors(Config);
    var config = AccessibleConfig{ .data = Config{
        .host = "localhost",
        .port = 8080,
        .debug = false,
    } };

    const host = config.get("host");
    std.debug.print("Host: {s}\n", .{host});

    config.set("debug", true);
    std.debug.print("Debug: {}\n", .{config.get("debug")});

    var map = try config.toMap(allocator);
    defer {
        var it = map.valueIterator();
        while (it.next()) |v| allocator.free(v.*);
        map.deinit();
    }
    std.debug.print("Config as map:\n", .{});
    var iter = map.iterator();
    while (iter.next()) |e| {
        std.debug.print("  {s}: {s}\n", .{ e.key_ptr.*, e.value_ptr.* });
    }

    // Generate dispatch table
    const Dispatcher = generateDispatch(.{
        .start = struct {
            fn f(args: anytype) !void {
                _ = args;
                std.debug.print("Starting...\n", .{});
            }
        }.f,
        .stop = struct {
            fn f(args: anytype) !void {
                _ = args;
                std.debug.print("Stopping...\n", .{});
            }
        }.f,
        .status = struct {
            fn f(args: anytype) !void {
                _ = args;
                std.debug.print("Status: OK\n", .{});
            }
        }.f,
    });

    const commands = Dispatcher.listCommands();
    std.debug.print("Available commands: ", .{});
    for (commands) |cmd| std.debug.print("{s} ", .{cmd});
    std.debug.print("\n", .{});

    try Dispatcher.dispatch("start", .{});
    try Dispatcher.dispatch("status", .{});
}
```

---

## Step 933: Generating C Headers

### Auto-Generate C Header Files

```zig
const std = @import("std");

// C type mapping
pub fn toCType(comptime ZigType: type) []const u8 {
    return switch (@typeInfo(ZigType)) {
        .Int => |i| switch (i.bits) {
            8 => if (i.signedness == .signed) "int8_t" else "uint8_t",
            16 => if (i.signedness == .signed) "int16_t" else "uint16_t",
            32 => if (i.signedness == .signed) "int32_t" else "uint32_t",
            64 => if (i.signedness == .signed) "int64_t" else "uint64_t",
            else => "int",
        },
        .Float => |f| switch (f.bits) {
            32 => "float",
            64 => "double",
            else => "double",
        },
        .Bool => "bool",
        .Void => "void",
        .Pointer => |p| if (p.child == u8) "const char*" else "void*",
        else => "void*",
    };
}

// Generate C struct
pub fn generateCStruct(
    comptime T: type,
    comptime name: []const u8,
    writer: anytype,
) !void {
    try writer.print("typedef struct {{\n", .{});

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields) |field| {
                const c_type = comptime toCType(field.type);
                try writer.print("    {s} {s};\n", .{ c_type, field.name });
            }
        },
        else => @compileError("generateCStruct requires struct"),
    }

    try writer.print("}} {s};\n\n", .{name});
}

// Generate C function declaration
pub fn generateCFunction(
    comptime ReturnType: type,
    comptime fn_name: []const u8,
    comptime params: []const struct { name: []const u8, type: type },
    writer: anytype,
) !void {
    const ret_c = comptime toCType(ReturnType);
    try writer.print("{s} {s}(", .{ ret_c, fn_name });

    for (params, 0..) |param, i| {
        if (i > 0) try writer.writeAll(", ");
        const param_c = comptime toCType(param.type);
        try writer.print("{s} {s}", .{ param_c, param.name });
    }

    try writer.writeAll(");\n");
}

// Generate complete C header from Zig module
pub fn generateCHeader(
    comptime module_name: []const u8,
    comptime types: []const type,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const writer = buf.writer();

    // Header guard
    const guard = std.ascii.upperString(try allocator.dupe(u8, module_name), try allocator.dupe(u8, module_name));
    defer allocator.free(guard);
    
    try writer.print("#ifndef {s}_H\n", .{guard});
    try writer.print("#define {s}_H\n\n", .{guard});
    try writer.writeAll("#include <stdint.h>\n");
    try writer.writeAll("#include <stdbool.h>\n\n");
    try writer.writeAll("#ifdef __cplusplus\nextern \"C\" {\n#endif\n\n");

    // Generate struct declarations
    inline for (types) |T| {
        try generateCStruct(T, @typeName(T), writer);
    }

    try writer.writeAll("#ifdef __cplusplus\n}\n#endif\n\n");
    try writer.print("#endif /* {s}_H */\n", .{guard});

    return buf.toOwnedSlice();
}

const Point = struct {
    x: f32,
    y: f32,
};

const Color = struct {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
};

pub fn cHeaderExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();
    const writer = buf.writer();

    // Generate struct
    try generateCStruct(Point, "Point", writer);
    try generateCStruct(Color, "Color", writer);

    // Generate function declaration
    try generateCFunction(Point, "addPoints", &.{
        .{ .name = "a", .type = Point },
        .{ .name = "b", .type = Point },
    }, writer);

    std.debug.print("Generated C header:\n{s}\n", .{buf.items});
}
```

---

## Step 934: Generating Documentation

### Auto-Generate API Documentation

```zig
const std = @import("std");

// Documentation annotation
pub const Doc = struct {
    description: []const u8,
    params: []const ParamDoc,
    returns: ?[]const u8,
    example: ?[]const u8,
    deprecated: bool,
    since: ?[]const u8,

    pub const ParamDoc = struct {
        name: []const u8,
        description: []const u8,
        required: bool,
    };
};

// Extract doc from function type info
pub fn extractDocs(comptime T: type) void {
    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            std.debug.print("# {s}\n\n", .{@typeName(T)});
            std.debug.print("## Fields\n\n", .{});
            inline for (s.fields) |field| {
                std.debug.print("- `{s}`: {s}\n", .{ field.name, @typeName(field.type) });
            }
            std.debug.print("\n## Methods\n\n", .{});
            inline for (s.decls) |decl| {
                std.debug.print("### {s}\n\n", .{decl.name});
            }
        },
        else => {},
    }
}

// Generate Markdown documentation
pub fn DocGenerator(comptime T: type) type {
    return struct {
        pub fn generateMarkdown(allocator: std.mem.Allocator) ![]const u8 {
            var buf = std.ArrayList(u8).init(allocator);
            errdefer buf.deinit();
            const w = buf.writer();

            try w.print("# {s}\n\n", .{@typeName(T)});

            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    try w.writeAll("## Fields\n\n");
                    try w.writeAll("| Name | Type | Description |\n");
                    try w.writeAll("|------|------|-------------|\n");
                    inline for (s.fields) |field| {
                        try w.print("| `{s}` | `{s}` | - |\n", .{
                            field.name,
                            @typeName(field.type),
                        });
                    }
                    try w.writeByte('\n');

                    if (s.decls.len > 0) {
                        try w.writeAll("## Methods\n\n");
                        inline for (s.decls) |decl| {
                            try w.print("### `{s}`\n\n", .{decl.name});
                            try w.writeAll("_No documentation available_\n\n");
                        }
                    }
                },
                .Enum => |e| {
                    try w.writeAll("## Values\n\n");
                    inline for (e.fields) |field| {
                        try w.print("- `{s}` = {}\n", .{ field.name, field.value });
                    }
                },
                else => try w.writeAll("_No documentation_\n"),
            }

            return buf.toOwnedSlice();
        }

        pub fn generateJson(allocator: std.mem.Allocator) ![]const u8 {
            var buf = std.ArrayList(u8).init(allocator);
            errdefer buf.deinit();
            const w = buf.writer();

            try w.writeAll("{\n");
            try w.print("  \"name\": \"{s}\",\n", .{@typeName(T)});
            try w.writeAll("  \"fields\": [\n");

            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields, 0..) |field, i| {
                        if (i > 0) try w.writeAll(",\n");
                        try w.writeAll("    {\n");
                        try w.print("      \"name\": \"{s}\",\n", .{field.name});
                        try w.print("      \"type\": \"{s}\"\n", .{@typeName(field.type)});
                        try w.writeAll("    }");
                    }
                    try w.writeByte('\n');
                },
                else => {},
            }

            try w.writeAll("  ]\n}\n");
            return buf.toOwnedSlice();
        }
    };
}

const ApiEndpoint = struct {
    path: []const u8,
    method: []const u8,
    handler: []const u8,
    description: []const u8,
    requires_auth: bool,
};

pub fn docGenerationExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Generate documentation
    const ApiDoc = DocGenerator(ApiEndpoint);
    
    const md = try ApiDoc.generateMarkdown(allocator);
    defer allocator.free(md);
    std.debug.print("Markdown docs:\n{s}\n", .{md});

    const json = try ApiDoc.generateJson(allocator);
    defer allocator.free(json);
    std.debug.print("JSON docs:\n{s}\n", .{json});
}
```

---

## Step 935: Protocol Buffer Generation

### Protobuf-like Code Generation

```zig
const std = @import("std");

// Field types matching protobuf
pub const FieldType = enum {
    int32,
    int64,
    uint32,
    uint64,
    float,
    double,
    bool_type,
    string,
    bytes,
    message,
};

pub const ProtoField = struct {
    name: []const u8,
    field_number: u32,
    field_type: FieldType,
    required: bool,
    repeated: bool,
    message_type: ?[]const u8,
};

pub const ProtoMessage = struct {
    name: []const u8,
    fields: []const ProtoField,
};

// Generate Zig struct from protobuf definition
pub fn generateFromProto(
    msg: ProtoMessage,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("pub const {s} = struct {{\n", .{msg.name});

    for (msg.fields) |field| {
        const zig_type = protoTypeToZig(field.field_type, field.message_type);
        
        if (field.repeated) {
            try w.print("    {s}: []const {s},\n", .{ field.name, zig_type });
        } else if (!field.required) {
            try w.print("    {s}: ?{s} = null,\n", .{ field.name, zig_type });
        } else {
            try w.print("    {s}: {s},\n", .{ field.name, zig_type });
        }
    }

    // Generate field number constants
    try w.writeAll("\n    pub const field_numbers = struct {\n");
    for (msg.fields) |field| {
        try w.print("        pub const {s} = {};\n", .{ field.name, field.field_number });
    }
    try w.writeAll("    };\n");

    // Generate serializer
    try w.writeAll("\n    pub fn serialize(self: @This(), writer: anytype) !void {\n");
    for (msg.fields) |field| {
        try w.print("        // Field {}: {s} (tag {})\n", .{
            field.field_number,
            field.name,
            field.field_number,
        });
        try w.print("        _ = self.{s};\n", .{field.name});
    }
    try w.writeAll("        _ = writer;\n");
    try w.writeAll("    }\n");

    try w.writeAll("};\n");
    return buf.toOwnedSlice();
}

fn protoTypeToZig(field_type: FieldType, message_type: ?[]const u8) []const u8 {
    return switch (field_type) {
        .int32 => "i32",
        .int64 => "i64",
        .uint32 => "u32",
        .uint64 => "u64",
        .float => "f32",
        .double => "f64",
        .bool_type => "bool",
        .string => "[]const u8",
        .bytes => "[]const u8",
        .message => message_type orelse "void",
    };
}

// Generate binary encoder
pub fn generateEncoder(comptime T: type) type {
    return struct {
        pub fn encode(value: T, writer: anytype) !void {
            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields, 0..) |field, field_num| {
                        const val = @field(value, field.name);
                        const tag = (field_num + 1) << 3; // field_number << 3 | wire_type
                        
                        switch (@typeInfo(field.type)) {
                            .Int => {
                                try writer.writeByte(@intCast(tag | 0)); // varint wire type
                                try encodeVarint(@intCast(val), writer);
                            },
                            .Bool => {
                                try writer.writeByte(@intCast(tag | 0));
                                try writer.writeByte(if (val) 1 else 0);
                            },
                            .Pointer => |p| {
                                if (p.size == .Slice and p.child == u8) {
                                    try writer.writeByte(@intCast(tag | 2)); // length-delimited
                                    try encodeVarint(val.len, writer);
                                    try writer.writeAll(val);
                                }
                            },
                            else => {},
                        }
                    }
                },
                else => {},
            }
        }

        fn encodeVarint(value: usize, writer: anytype) !void {
            var v = value;
            while (v >= 0x80) {
                try writer.writeByte(@intCast((v & 0x7F) | 0x80));
                v >>= 7;
            }
            try writer.writeByte(@intCast(v));
        }
    };
}

pub fn protobufGenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const msg = ProtoMessage{
        .name = "Person",
        .fields = &[_]ProtoField{
            .{ .name = "id", .field_number = 1, .field_type = .int32, .required = true, .repeated = false, .message_type = null },
            .{ .name = "name", .field_number = 2, .field_type = .string, .required = true, .repeated = false, .message_type = null },
            .{ .name = "email", .field_number = 3, .field_type = .string, .required = false, .repeated = false, .message_type = null },
            .{ .name = "phones", .field_number = 4, .field_type = .string, .required = false, .repeated = true, .message_type = null },
        },
    };

    const code = try generateFromProto(msg, allocator);
    defer allocator.free(code);
    std.debug.print("Generated from proto:\n{s}\n", .{code});
}
```

---

## Step 936: Database Schema Generation

### ORM Schema Generation

```zig
const std = @import("std");

// SQL type mapping
pub fn zigToSqlType(comptime T: type) []const u8 {
    return switch (@typeInfo(T)) {
        .Int => |i| switch (i.bits) {
            8, 16 => "SMALLINT",
            32 => "INTEGER",
            64 => "BIGINT",
            else => "INTEGER",
        },
        .Float => "REAL",
        .Bool => "BOOLEAN",
        .Pointer => |p| if (p.child == u8) "TEXT" else "BLOB",
        .Optional => |o| zigToSqlType(o.child), // Nullable
        else => "TEXT",
    };
}

// Check if field is nullable
pub fn isNullable(comptime T: type) bool {
    return @typeInfo(T) == .Optional;
}

// Generate CREATE TABLE statement
pub fn generateCreateTable(
    comptime T: type,
    comptime table_name: []const u8,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("CREATE TABLE IF NOT EXISTS {s} (\n", .{table_name});

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields, 0..) |field, i| {
                if (i > 0) try w.writeAll(",\n");
                
                const sql_type = comptime zigToSqlType(field.type);
                const nullable = comptime isNullable(field.type);
                
                try w.print("    {s} {s}", .{ field.name, sql_type });
                
                // Auto primary key for 'id' field
                if (std.mem.eql(u8, field.name, "id")) {
                    try w.writeAll(" PRIMARY KEY");
                } else if (!nullable) {
                    try w.writeAll(" NOT NULL");
                }
            }
        },
        else => @compileError("generateCreateTable requires struct"),
    }

    try w.writeAll("\n);\n");
    return buf.toOwnedSlice();
}

// Generate INSERT statement template
pub fn generateInsertTemplate(
    comptime T: type,
    comptime table_name: []const u8,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("INSERT INTO {s} (", .{table_name});

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            inline for (s.fields, 0..) |field, i| {
                if (i > 0) try w.writeAll(", ");
                try w.writeAll(field.name);
            }
            try w.writeAll(") VALUES (");
            inline for (s.fields, 0..) |_, i| {
                if (i > 0) try w.writeAll(", ");
                try w.print("?{}", .{i + 1});
            }
        },
        else => {},
    }

    try w.writeAll(");");
    return buf.toOwnedSlice();
}

// Generate migration file
pub fn generateMigration(
    comptime T: type,
    comptime table_name: []const u8,
    comptime version: u32,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("-- Migration version: {}\n", .{version});
    try w.print("-- Table: {s}\n\n", .{table_name});

    try w.writeAll("-- Up\n");
    const create_sql = try generateCreateTable(T, table_name, allocator);
    defer allocator.free(create_sql);
    try w.writeAll(create_sql);
    try w.writeByte('\n');

    try w.writeAll("-- Down\n");
    try w.print("DROP TABLE IF EXISTS {s};\n", .{table_name});

    return buf.toOwnedSlice();
}

// Generate ORM query methods
pub fn generateQueryMethods(comptime T: type, comptime table_name: []const u8) type {
    return struct {
        pub fn findAll(allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator, "SELECT * FROM {s}", .{table_name});
        }

        pub fn findById(id: anytype, allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator, "SELECT * FROM {s} WHERE id = {}", .{ table_name, id });
        }

        pub fn findWhere(condition: []const u8, allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator, "SELECT * FROM {s} WHERE {s}", .{ table_name, condition });
        }

        pub fn insertSql(allocator: std.mem.Allocator) ![]const u8 {
            return generateInsertTemplate(T, table_name, allocator);
        }

        pub fn deleteSql(id: anytype, allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator, "DELETE FROM {s} WHERE id = {}", .{ table_name, id });
        }
    };
}

const Article = struct {
    id: u64,
    title: []const u8,
    content: []const u8,
    author_id: u64,
    published: bool,
    created_at: i64,
    updated_at: ?i64,
};

pub fn dbSchemaGenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Generate CREATE TABLE
    const create_sql = try generateCreateTable(Article, "articles", allocator);
    defer allocator.free(create_sql);
    std.debug.print("CREATE TABLE:\n{s}\n", .{create_sql});

    // Generate INSERT template
    const insert_sql = try generateInsertTemplate(Article, "articles", allocator);
    defer allocator.free(insert_sql);
    std.debug.print("INSERT template:\n{s}\n", .{insert_sql});

    // Generate migration
    const migration = try generateMigration(Article, "articles", 1, allocator);
    defer allocator.free(migration);
    std.debug.print("Migration:\n{s}\n", .{migration});

    // Use generated query methods
    const ArticleQueries = generateQueryMethods(Article, "articles");
    const find_sql = try ArticleQueries.findAll(allocator);
    defer allocator.free(find_sql);
    std.debug.print("Find all: {s}\n", .{find_sql});

    const find_by_id = try ArticleQueries.findById(42, allocator);
    defer allocator.free(find_by_id);
    std.debug.print("Find by id: {s}\n", .{find_by_id});
}
```

---

## Step 937: OpenAPI/Swagger Generation

### REST API Specification Generation

```zig
const std = @import("std");

// HTTP method enum
pub const HttpMethod = enum { GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS };

// API endpoint definition
pub const Endpoint = struct {
    path: []const u8,
    method: HttpMethod,
    summary: []const u8,
    description: []const u8,
    tags: []const []const u8,
    params: []const Param,
    request_body: ?type,
    response: type,

    pub const Param = struct {
        name: []const u8,
        location: enum { path, query, header },
        description: []const u8,
        required: bool,
        param_type: type,
    };
};

// Generate OpenAPI YAML
pub fn generateOpenApiYaml(
    api_title: []const u8,
    api_version: []const u8,
    endpoints: []const Endpoint,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    // OpenAPI header
    try w.writeAll("openapi: '3.0.0'\n");
    try w.writeAll("info:\n");
    try w.print("  title: '{s}'\n", .{api_title});
    try w.print("  version: '{s}'\n", .{api_version});
    try w.writeByte('\n');

    // Paths
    try w.writeAll("paths:\n");
    for (endpoints) |endpoint| {
        try w.print("  {s}:\n", .{endpoint.path});
        try w.print("    {s}:\n", .{@tagName(endpoint.method)});
        try w.print("      summary: '{s}'\n", .{endpoint.summary});
        
        if (endpoint.tags.len > 0) {
            try w.writeAll("      tags:\n");
            for (endpoint.tags) |tag| {
                try w.print("        - {s}\n", .{tag});
            }
        }

        if (endpoint.params.len > 0) {
            try w.writeAll("      parameters:\n");
            for (endpoint.params) |param| {
                try w.print("        - name: {s}\n", .{param.name});
                try w.print("          in: {s}\n", .{@tagName(param.location)});
                try w.print("          required: {}\n", .{param.required});
                try w.print("          description: '{s}'\n", .{param.description});
            }
        }

        try w.writeAll("      responses:\n");
        try w.writeAll("        '200':\n");
        try w.writeAll("          description: Success\n");
    }

    return buf.toOwnedSlice();
}

// Generate TypeScript client
pub fn generateTypeScriptClient(
    endpoints: []const struct {
        name: []const u8,
        path: []const u8,
        method: HttpMethod,
    },
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.writeAll("// Auto-generated API client\n\n");
    try w.writeAll("const API_BASE = process.env.API_URL || 'http://localhost:8080';\n\n");
    try w.writeAll("export const api = {\n");

    for (endpoints, 0..) |ep, i| {
        if (i > 0) try w.writeByte('\n');
        try w.print("  async {s}(params?: any): Promise<any> {{\n", .{ep.name});
        try w.print("    const response = await fetch(`${{API_BASE}}{s}`, {{\n", .{ep.path});
        try w.print("      method: '{s}',\n", .{@tagName(ep.method)});
        try w.writeAll("      headers: { 'Content-Type': 'application/json' },\n");
        try w.writeAll("      ...(params && { body: JSON.stringify(params) }),\n");
        try w.writeAll("    });\n");
        try w.writeAll("    return response.json();\n");
        try w.writeAll("  },\n");
    }

    try w.writeAll("};\n");
    return buf.toOwnedSlice();
}

pub fn openApiGenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const endpoints = [_]Endpoint{
        .{
            .path = "/users",
            .method = .GET,
            .summary = "List all users",
            .description = "Returns a paginated list of users",
            .tags = &.{"users"},
            .params = &.{
                .{ .name = "page", .location = .query, .description = "Page number", .required = false, .param_type = u32 },
                .{ .name = "limit", .location = .query, .description = "Items per page", .required = false, .param_type = u32 },
            },
            .request_body = null,
            .response = void,
        },
        .{
            .path = "/users/{id}",
            .method = .GET,
            .summary = "Get user by ID",
            .description = "Returns a single user",
            .tags = &.{"users"},
            .params = &.{
                .{ .name = "id", .location = .path, .description = "User ID", .required = true, .param_type = u64 },
            },
            .request_body = null,
            .response = void,
        },
    };

    const yaml = try generateOpenApiYaml("My API", "1.0.0", &endpoints, allocator);
    defer allocator.free(yaml);
    std.debug.print("OpenAPI YAML:\n{s}\n", .{yaml});

    const ts_endpoints = [_]struct {
        name: []const u8,
        path: []const u8,
        method: HttpMethod,
    }{
        .{ .name = "getUsers", .path = "/users", .method = .GET },
        .{ .name = "getUserById", .path = "/users/{id}", .method = .GET },
        .{ .name = "createUser", .path = "/users", .method = .POST },
        .{ .name = "updateUser", .path = "/users/{id}", .method = .PUT },
        .{ .name = "deleteUser", .path = "/users/{id}", .method = .DELETE },
    };

    const ts_client = try generateTypeScriptClient(&ts_endpoints, allocator);
    defer allocator.free(ts_client);
    std.debug.print("TypeScript client:\n{s}\n", .{ts_client});
}
```

---

## Step 938: Code Generation for Testing

### Generating Test Code

```zig
const std = @import("std");

// Generate test cases for a function
pub fn generateTestsForFn(
    comptime fn_name: []const u8,
    comptime cases: []const struct {
        input: anytype,
        expected: anytype,
        description: []const u8,
    },
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("// Auto-generated tests for {s}\n\n", .{fn_name});

    for (cases, 0..) |case, i| {
        try w.print("test \"{s} case {}\" {{\n", .{ fn_name, i });
        try w.print("    // {s}\n", .{case.description});
        try w.print("    const result = {s}({any});\n", .{ fn_name, case.input });
        try w.print("    try std.testing.expectEqual({any}, result);\n", .{case.expected});
        try w.writeAll("}\n\n");
    }

    return buf.toOwnedSlice();
}

// Generate mock object
pub fn generateMock(comptime T: type, allocator: std.mem.Allocator) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("// Mock for {s}\n", .{@typeName(T)});
    try w.print("pub const Mock{s} = struct {{\n", .{@typeName(T)});

    const info = @typeInfo(T);
    switch (info) {
        .Struct => |s| {
            // Record calls
            try w.writeAll("    call_counts: std.StringHashMap(u32),\n");
            try w.writeAll("    return_values: std.StringHashMap([]const u8),\n\n");

            inline for (s.decls) |decl| {
                try w.print("    {s}_calls: u32 = 0,\n", .{decl.name});
            }
        },
        else => {},
    }

    try w.writeAll("};\n");
    return buf.toOwnedSlice();
}

// Generate property test
pub fn generatePropertyTest(
    comptime property_name: []const u8,
    comptime T: type,
    comptime property: fn (T) bool,
    allocator: std.mem.Allocator,
) ![]const u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    const w = buf.writer();

    try w.print("// Property test: {s}\n", .{property_name});
    try w.print("test \"property: {s}\" {{\n", .{property_name});
    try w.writeAll("    const seed: u64 = 12345;\n");
    try w.writeAll("    var rng = std.rand.DefaultPrng.init(seed);\n");
    try w.writeAll("    const random = rng.random();\n");
    try w.writeAll("    var i: usize = 0;\n");
    try w.writeAll("    while (i < 100) : (i += 1) {\n");
    try w.print("        const val = random.int({s});\n", .{@typeName(T)});
    try w.print("        try std.testing.expect({s}(val));\n", .{property_name});
    try w.writeAll("    }\n");
    try w.writeAll("}\n");

    _ = property;
    return buf.toOwnedSlice();
}

// Snapshot testing generator
pub const SnapshotStore = struct {
    snapshots: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
    dirty: bool,

    pub fn init(allocator: std.mem.Allocator) SnapshotStore {
        return SnapshotStore{
            .snapshots = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
            .dirty = false,
        };
    }

    pub fn deinit(self: *SnapshotStore) void {
        var it = self.snapshots.valueIterator();
        while (it.next()) |v| self.allocator.free(v.*);
        self.snapshots.deinit();
    }

    pub fn match(self: *SnapshotStore, name: []const u8, value: []const u8) !void {
        if (self.snapshots.get(name)) |snapshot| {
            if (!std.mem.eql(u8, snapshot, value)) {
                std.debug.print("Snapshot mismatch for '{s}':\n", .{name});
                std.debug.print("  Expected:\n{s}\n", .{snapshot});
                std.debug.print("  Got:\n{s}\n", .{value});
                return error.SnapshotMismatch;
            }
        } else {
            // First time - create snapshot
            const copy = try self.allocator.dupe(u8, value);
            try self.snapshots.put(name, copy);
            self.dirty = true;
            std.debug.print("Created snapshot: {s}\n", .{name});
        }
    }
};

pub fn testCodegenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Generate test code
    const test_code = try generateTestsForFn(
        "add",
        &.{
            .{ .input = @as(i32, 1), .expected = @as(i32, 3), .description = "positive numbers" },
            .{ .input = @as(i32, -1), .expected = @as(i32, 1), .description = "negative + positive" },
            .{ .input = @as(i32, 0), .expected = @as(i32, 2), .description = "zero" },
        },
        allocator,
    );
    defer allocator.free(test_code);
    std.debug.print("Generated test code:\n{s}\n", .{test_code});

    // Snapshot test
    var store = SnapshotStore.init(allocator);
    defer store.deinit();

    try store.match("greeting", "Hello, World!");
    try store.match("greeting", "Hello, World!"); // Should pass
    
    store.match("different", "New value") catch |e| {
        std.debug.print("Expected: {}\n", .{e});
    };
}
```

---

## Step 939: Build-Time Code Generation

### Code Generation ใน build.zig

```zig
// build.zig - Build-time code generation
const std = @import("std");

// ตัวอย่าง build.zig ที่สร้าง code
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Step 1: Generate code before compilation
    const codegen_step = b.addSystemCommand(&.{
        "zig", "run", "tools/codegen.zig", "--", "src/generated.zig",
    });

    // Step 2: Compile main program
    const exe = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Make compilation depend on code generation
    exe.step.dependOn(&codegen_step.step);
    b.installArtifact(exe);
}

// tools/codegen.zig - Code generator tool
// pub fn main() !void {
//     const output_path = std.process.args().next() orelse "generated.zig";
//     const file = try std.fs.createFileAbsolute(output_path, .{});
//     defer file.close();
//     
//     // Generate code...
// }
```

```zig
// Code generation tool example
const std = @import("std");

pub const CodegenConfig = struct {
    output_file: []const u8,
    template: []const u8,
    variables: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, output: []const u8, template: []const u8) CodegenConfig {
        return CodegenConfig{
            .output_file = output,
            .template = template,
            .variables = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *CodegenConfig) void {
        self.variables.deinit();
    }

    pub fn setVar(self: *CodegenConfig, name: []const u8, value: []const u8) !void {
        try self.variables.put(name, value);
    }

    pub fn generate(self: *CodegenConfig) !void {
        var output = std.ArrayList(u8).init(self.allocator);
        defer output.deinit();

        var i: usize = 0;
        while (i < self.template.len) {
            if (i + 1 < self.template.len and
                self.template[i] == '{' and self.template[i + 1] == '{')
            {
                const start = i + 2;
                var end = start;
                while (end + 1 < self.template.len and
                    !(self.template[end] == '}' and self.template[end + 1] == '}'))
                {
                    end += 1;
                }
                const var_name = std.mem.trim(u8, self.template[start..end], " ");
                if (self.variables.get(var_name)) |val| {
                    try output.appendSlice(val);
                }
                i = end + 2;
            } else {
                try output.append(self.template[i]);
                i += 1;
            }
        }

        // Write to file
        const file = try std.fs.cwd().createFile(self.output_file, .{});
        defer file.close();
        try file.writeAll(output.items);

        std.debug.print("Generated: {s}\n", .{self.output_file});
    }
};

pub fn buildTimeCodegenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Template for a generated Zig file
    const template =
        \\// Auto-generated file - do not edit
        \\// Generated at: {{ timestamp }}
        \\
        \\pub const VERSION = "{{ version }}";
        \\pub const BUILD_DATE = "{{ date }}";
        \\pub const GIT_HASH = "{{ git_hash }}";
        \\
        \\pub const Config = struct {
        \\    api_url: []const u8 = "{{ api_url }}",
        \\    debug: bool = {{ debug }},
        \\};
    ;

    var config = CodegenConfig.init(
        allocator,
        "/tmp/generated_config.zig",
        template,
    );
    defer config.deinit();

    try config.setVar("timestamp", "2024-01-01T00:00:00Z");
    try config.setVar("version", "1.0.0");
    try config.setVar("date", "2024-01-01");
    try config.setVar("git_hash", "abc123def456");
    try config.setVar("api_url", "https://api.example.com");
    try config.setVar("debug", "false");

    try config.generate();
}
```

---

## Step 940: Complete Code Generation Pipeline

### Full Code Generation System

```zig
const std = @import("std");

// Complete code generation pipeline
pub const Pipeline = struct {
    allocator: std.mem.Allocator,
    stages: std.ArrayList(Stage),
    context: std.StringHashMap([]const u8),

    pub const Stage = struct {
        name: []const u8,
        transform: *const fn (input: []const u8, ctx: *std.StringHashMap([]const u8), allocator: std.mem.Allocator) anyerror![]const u8,
    };

    pub fn init(allocator: std.mem.Allocator) Pipeline {
        return Pipeline{
            .allocator = allocator,
            .stages = std.ArrayList(Stage).init(allocator),
            .context = std.StringHashMap([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *Pipeline) void {
        self.stages.deinit();
        self.context.deinit();
    }

    pub fn addStage(self: *Pipeline, stage: Stage) !void {
        try self.stages.append(stage);
    }

    pub fn setContext(self: *Pipeline, key: []const u8, value: []const u8) !void {
        try self.context.put(key, value);
    }

    pub fn run(self: *Pipeline, input: []const u8) ![]const u8 {
        var current = try self.allocator.dupe(u8, input);

        for (self.stages.items) |stage| {
            std.debug.print("Running stage: {s}\n", .{stage.name});
            const output = try stage.transform(current, &self.context, self.allocator);
            self.allocator.free(current);
            current = output;
        }

        return current;
    }
};

// Stage: Variable substitution
fn substituteVars(
    input: []const u8,
    ctx: *std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
) ![]const u8 {
    var output = std.ArrayList(u8).init(allocator);
    errdefer output.deinit();
    
    var i: usize = 0;
    while (i < input.len) {
        if (i + 1 < input.len and input[i] == '$' and input[i + 1] == '{') {
            const start = i + 2;
            var end = start;
            while (end < input.len and input[end] != '}') end += 1;
            
            const var_name = input[start..end];
            if (ctx.get(var_name)) |val| {
                try output.appendSlice(val);
            } else {
                try output.appendSlice(input[i .. end + 1]);
            }
            i = end + 1;
        } else {
            try output.append(input[i]);
            i += 1;
        }
    }
    
    return output.toOwnedSlice();
}

// Stage: Format code
fn formatCode(
    input: []const u8,
    ctx: *std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
) ![]const u8 {
    _ = ctx;
    // In real implementation, run zig fmt
    return allocator.dupe(u8, input);
}

// Stage: Add copyright header
fn addCopyrightHeader(
    input: []const u8,
    ctx: *std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,
) ![]const u8 {
    const author = ctx.get("author") orelse "Unknown";
    const year = ctx.get("year") orelse "2024";
    
    var output = std.ArrayList(u8).init(allocator);
    errdefer output.deinit();
    
    try output.appendSlice("// Copyright (c) ");
    try output.appendSlice(year);
    try output.append(' ');
    try output.appendSlice(author);
    try output.appendSlice("\n// Auto-generated - do not edit\n\n");
    try output.appendSlice(input);
    
    return output.toOwnedSlice();
}

// Schema-based full code generation
pub fn SchemaCodegen(comptime T: type) type {
    return struct {
        pub fn generateAll(prefix: []const u8, allocator: std.mem.Allocator) !CodeFiles {
            var files = CodeFiles.init(allocator);
            errdefer files.deinit();

            // Generate struct definition
            const struct_code = try generateStruct(prefix, allocator);
            try files.add("model.zig", struct_code);

            // Generate repository
            const repo_code = try generateRepository(prefix, allocator);
            try files.add("repository.zig", repo_code);

            // Generate tests
            const test_code = try generateTests(prefix, allocator);
            try files.add("tests.zig", test_code);

            return files;
        }

        fn generateStruct(prefix: []const u8, allocator: std.mem.Allocator) ![]const u8 {
            var buf = std.ArrayList(u8).init(allocator);
            errdefer buf.deinit();
            const w = buf.writer();

            const type_name = @typeName(T);
            _ = prefix;
            try w.print("// {s} model\n\n", .{type_name});
            try w.writeAll("const std = @import(\"std\");\n\n");
            try w.print("pub const {s} = struct {{\n", .{type_name});

            const info = @typeInfo(T);
            switch (info) {
                .Struct => |s| {
                    inline for (s.fields) |field| {
                        try w.print("    {s}: {s},\n", .{ field.name, @typeName(field.type) });
                    }
                },
                else => {},
            }

            try w.writeAll("};\n");
            return buf.toOwnedSlice();
        }

        fn generateRepository(_: []const u8, allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator,
                "// Repository for {s}\n\npub const Repository = struct {{\n    // TODO\n}};\n",
                .{@typeName(T)},
            );
        }

        fn generateTests(_: []const u8, allocator: std.mem.Allocator) ![]const u8 {
            return std.fmt.allocPrint(allocator,
                "// Tests for {s}\n\nconst std = @import(\"std\");\n\ntest \"basic\" {{\n    // TODO\n}}\n",
                .{@typeName(T)},
            );
        }
    };
}

pub const CodeFiles = struct {
    files: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) CodeFiles {
        return CodeFiles{
            .files = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *CodeFiles) void {
        var it = self.files.valueIterator();
        while (it.next()) |v| self.allocator.free(v.*);
        self.files.deinit();
    }

    pub fn add(self: *CodeFiles, name: []const u8, content: []const u8) !void {
        try self.files.put(name, content);
    }

    pub fn writeAll(self: *CodeFiles, output_dir: []const u8) !void {
        var dir = try std.fs.cwd().makeOpenPath(output_dir, .{});
        defer dir.close();

        var iter = self.files.iterator();
        while (iter.next()) |entry| {
            const file = try dir.createFile(entry.key_ptr.*, .{});
            defer file.close();
            try file.writeAll(entry.value_ptr.*);
            std.debug.print("Written: {s}/{s}\n", .{ output_dir, entry.key_ptr.* });
        }
    }
};

const Product = struct {
    id: u64,
    name: []const u8,
    price: f64,
    stock: u32,
};

pub fn completeCodegenExample() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Pipeline-based code generation
    var pipeline = Pipeline.init(allocator);
    defer pipeline.deinit();

    try pipeline.setContext("author", "Alice");
    try pipeline.setContext("year", "2024");
    try pipeline.setContext("api_url", "https://api.example.com");

    try pipeline.addStage(.{
        .name = "substitute_vars",
        .transform = substituteVars,
    });
    try pipeline.addStage(.{
        .name = "add_copyright",
        .transform = addCopyrightHeader,
    });
    try pipeline.addStage(.{
        .name = "format",
        .transform = formatCode,
    });

    const template = "const API_URL = \"${api_url}\";\n";
    const result = try pipeline.run(template);
    defer allocator.free(result);
    std.debug.print("Pipeline output:\n{s}\n", .{result});

    // Schema-based codegen
    const ProductCodegen = SchemaCodegen(Product);
    var files = try ProductCodegen.generateAll("product", allocator);
    defer files.deinit();

    var iter = files.files.iterator();
    while (iter.next()) |entry| {
        std.debug.print("\n=== {s} ===\n{s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
}

pub fn main() !void {
    try comptimeCodegenExample();
    try cHeaderExample();
    try docGenerationExample();
    try protobufGenExample();
    try dbSchemaGenExample();
    try openApiGenExample();
    try testCodegenExample();
    try buildTimeCodegenExample();
    try completeCodegenExample();
}
```

---

## สรุป Part 94

ในส่วนนี้เราได้เรียนรู้:

1. **Code Generation Concepts** - ประเภทและประโยชน์ของ code generation
2. **Comptime Generation** - สร้าง code ณ compile time
3. **C Header Generation** - สร้าง .h files จาก Zig types
4. **Documentation Generation** - สร้าง docs อัตโนมัติ
5. **Protocol Buffer Generation** - Protobuf-like message generation
6. **Database Schema Generation** - สร้าง SQL DDL และ ORM
7. **OpenAPI Generation** - REST API spec generation
8. **Test Code Generation** - สร้าง test cases อัตโนมัติ
9. **Build-Time Generation** - code generation ใน build process
10. **Complete Pipeline** - Full code generation workflow

### Best Practices

- แยก generator จาก generated code
- ใช้ templates สำหรับ flexibility
- Generate tests พร้อมกับ production code
- Version control แบบ explicit สำหรับ generated files
- ใช้ comptime เมื่อทำได้ เพื่อ zero-overhead
