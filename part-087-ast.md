# ส่วนที่ 87: Parsing - Abstract Syntax Tree (AST)

## ขั้นตอนที่ 861-870: การสร้างและจัดการ AST ใน Zig

AST (Abstract Syntax Tree) คือโครงสร้างข้อมูลที่แทนโปรแกรมในรูปแบบของ tree โดยลบรายละเอียดที่ไม่จำเป็น (เช่น วงเล็บ, semicolons) ออกไป

---

## Step 861: AST Concept และ Design

### ทำไมต้องมี AST?

AST คือ bridge ระหว่าง parsing กับ compilation/interpretation มีประโยชน์ดังนี้:
- **Separation of concerns**: Parser สร้าง AST, Compiler ใช้ AST
- **Multiple passes**: สามารถ traverse หลายครั้งได้
- **Transformations**: ง่ายต่อการ transform (optimize, desugar)
- **Analysis**: ง่ายต่อการวิเคราะห์โครงสร้างโปรแกรม

```zig
// ast_design.zig - ออกแบบ AST สำหรับภาษาง่ายๆ

const std = @import("std");

// ตัวอย่าง source code:
// fn add(a: int, b: int): int {
//     return a + b;
// }
//
// AST ที่ได้:
// Program
// └── FunctionDecl "add"
//     ├── Params
//     │   ├── Param "a": int
//     │   └── Param "b": int
//     ├── ReturnType: int
//     └── Body (Block)
//         └── ReturnStmt
//             └── BinaryExpr "+"
//                 ├── Identifier "a"
//                 └── Identifier "b"

// หลักการออกแบบ AST ที่ดี:
// 1. แต่ละ node มี type ชัดเจน
// 2. เก็บ source location สำหรับ error reporting
// 3. ใช้ memory อย่างมีประสิทธิภาพ
// 4. รองรับ visitor pattern

pub const SourceLocation = struct {
    line: u32,
    column: u32,
    file: []const u8,
    
    pub fn format(
        self: SourceLocation,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{s}:{d}:{d}", .{ self.file, self.line, self.column });
    }
};

pub const NodeId = u32;

// ทุก node มี base fields เหล่านี้
pub const NodeBase = struct {
    id: NodeId,
    loc: SourceLocation,
};
```

---

## Step 862: AST Node Types (Union)

```zig
// ast_nodes.zig - ประเภทต่างๆ ของ AST nodes

const std = @import("std");

pub const TypeRef = union(enum) {
    primitive: PrimitiveType,
    named: []const u8,
    array: struct {
        element_type: *TypeRef,
        size: ?u64,
    },
    pointer: struct {
        pointee: *TypeRef,
        is_const: bool,
    },
    function: struct {
        params: []TypeRef,
        return_type: *TypeRef,
    },
    optional: *TypeRef,
    
    pub const PrimitiveType = enum {
        int, 
        float, 
        bool, 
        string, 
        void,
        u8, u16, u32, u64,
        i8, i16, i32, i64,
        f32, f64,
    };
};

// Expression nodes
pub const Expr = union(enum) {
    // Literals
    integer_lit: struct {
        value: i64,
        base: NodeBase,
    },
    float_lit: struct {
        value: f64,
        base: NodeBase,
    },
    string_lit: struct {
        value: []const u8,
        base: NodeBase,
    },
    bool_lit: struct {
        value: bool,
        base: NodeBase,
    },
    null_lit: struct {
        base: NodeBase,
    },
    array_lit: struct {
        elements: []*Expr,
        base: NodeBase,
    },
    
    // References
    identifier: struct {
        name: []const u8,
        base: NodeBase,
    },
    
    // Operations
    binary_op: struct {
        op: BinaryOp,
        left: *Expr,
        right: *Expr,
        base: NodeBase,
    },
    unary_op: struct {
        op: UnaryOp,
        operand: *Expr,
        base: NodeBase,
    },
    
    // Compound expressions
    call: struct {
        callee: *Expr,
        args: []*Expr,
        base: NodeBase,
    },
    index: struct {
        array: *Expr,
        index: *Expr,
        base: NodeBase,
    },
    field_access: struct {
        object: *Expr,
        field: []const u8,
        base: NodeBase,
    },
    assignment: struct {
        target: *Expr,
        value: *Expr,
        op: ?BinaryOp, // สำหรับ +=, -=, etc.
        base: NodeBase,
    },
    conditional: struct {
        condition: *Expr,
        then_expr: *Expr,
        else_expr: *Expr,
        base: NodeBase,
    },
    cast: struct {
        expr: *Expr,
        target_type: TypeRef,
        base: NodeBase,
    },
    
    pub fn getBase(self: *const Expr) NodeBase {
        return switch (self.*) {
            .integer_lit => |n| n.base,
            .float_lit => |n| n.base,
            .string_lit => |n| n.base,
            .bool_lit => |n| n.base,
            .null_lit => |n| n.base,
            .array_lit => |n| n.base,
            .identifier => |n| n.base,
            .binary_op => |n| n.base,
            .unary_op => |n| n.base,
            .call => |n| n.base,
            .index => |n| n.base,
            .field_access => |n| n.base,
            .assignment => |n| n.base,
            .conditional => |n| n.base,
            .cast => |n| n.base,
        };
    }
};

pub const BinaryOp = enum {
    add, sub, mul, div, mod,
    eq, ne, lt, le, gt, ge,
    and_, or_,
    bitand, bitor, bitxor,
    shl, shr,
};

pub const UnaryOp = enum {
    neg,    // -
    not,    // !
    bitnot, // ~
    addr,   // &
    deref,  // *
};

// Statement nodes
pub const Stmt = union(enum) {
    expr_stmt: struct {
        expr: *Expr,
        base: NodeBase,
    },
    var_decl: struct {
        name: []const u8,
        type_ref: ?TypeRef,
        initializer: ?*Expr,
        is_const: bool,
        is_mut: bool,
        base: NodeBase,
    },
    block: struct {
        statements: []*Stmt,
        base: NodeBase,
    },
    if_stmt: struct {
        condition: *Expr,
        then_block: *Stmt,
        else_clause: ?*Stmt,
        base: NodeBase,
    },
    while_stmt: struct {
        condition: *Expr,
        body: *Stmt,
        label: ?[]const u8,
        base: NodeBase,
    },
    for_stmt: struct {
        init: ?*Stmt,
        condition: ?*Expr,
        update: ?*Expr,
        body: *Stmt,
        label: ?[]const u8,
        base: NodeBase,
    },
    for_in: struct {
        variable: []const u8,
        iterable: *Expr,
        body: *Stmt,
        base: NodeBase,
    },
    return_stmt: struct {
        value: ?*Expr,
        base: NodeBase,
    },
    break_stmt: struct {
        label: ?[]const u8,
        value: ?*Expr,
        base: NodeBase,
    },
    continue_stmt: struct {
        label: ?[]const u8,
        base: NodeBase,
    },
};

// Top-level declaration nodes
pub const Decl = union(enum) {
    function: FunctionDecl,
    struct_type: StructDecl,
    enum_type: EnumDecl,
    type_alias: TypeAliasDecl,
    import: ImportDecl,
    
    pub const FunctionDecl = struct {
        name: []const u8,
        type_params: []TypeParam,
        params: []FuncParam,
        return_type: ?TypeRef,
        body: ?*Stmt,  // null สำหรับ extern functions
        is_pub: bool,
        is_extern: bool,
        base: NodeBase,
        
        pub const TypeParam = struct {
            name: []const u8,
            constraint: ?TypeRef,
        };
        
        pub const FuncParam = struct {
            name: []const u8,
            type_ref: TypeRef,
            default_value: ?*Expr,
            is_mut: bool,
        };
    };
    
    pub const StructDecl = struct {
        name: []const u8,
        type_params: []Decl.FunctionDecl.TypeParam,
        fields: []StructField,
        methods: []FunctionDecl,
        is_pub: bool,
        base: NodeBase,
        
        pub const StructField = struct {
            name: []const u8,
            type_ref: TypeRef,
            default_value: ?*Expr,
            is_pub: bool,
        };
    };
    
    pub const EnumDecl = struct {
        name: []const u8,
        variants: []EnumVariant,
        is_pub: bool,
        base: NodeBase,
        
        pub const EnumVariant = struct {
            name: []const u8,
            fields: ?[]TypeRef,  // สำหรับ tagged union
            value: ?*Expr,       // สำหรับ integer enum
        };
    };
    
    pub const TypeAliasDecl = struct {
        name: []const u8,
        aliased_type: TypeRef,
        is_pub: bool,
        base: NodeBase,
    };
    
    pub const ImportDecl = struct {
        path: []const u8,
        alias: ?[]const u8,
        items: ?[][]const u8,
        base: NodeBase,
    };
};

// Root of the AST
pub const Program = struct {
    declarations: []Decl,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) Program {
        return Program{
            .declarations = &.{},
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Program) void {
        // TODO: free all nodes
        _ = self;
    }
};
```

---

## Step 863: AST Builder

```zig
// ast_builder.zig - Helper สำหรับสร้าง AST nodes

const std = @import("std");

pub const AstBuilder = struct {
    allocator: std.mem.Allocator,
    next_id: NodeId = 0,
    
    pub fn init(allocator: std.mem.Allocator) AstBuilder {
        return AstBuilder{ .allocator = allocator };
    }
    
    fn newId(self: *AstBuilder) NodeId {
        const id = self.next_id;
        self.next_id += 1;
        return id;
    }
    
    fn newBase(self: *AstBuilder, loc: SourceLocation) NodeBase {
        return NodeBase{ .id = self.newId(), .loc = loc };
    }
    
    // Expression builders
    pub fn intLit(self: *AstBuilder, value: i64, loc: SourceLocation) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .integer_lit = .{ .value = value, .base = self.newBase(loc) } };
        return node;
    }
    
    pub fn floatLit(self: *AstBuilder, value: f64, loc: SourceLocation) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .float_lit = .{ .value = value, .base = self.newBase(loc) } };
        return node;
    }
    
    pub fn stringLit(self: *AstBuilder, value: []const u8, loc: SourceLocation) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .string_lit = .{ .value = value, .base = self.newBase(loc) } };
        return node;
    }
    
    pub fn boolLit(self: *AstBuilder, value: bool, loc: SourceLocation) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .bool_lit = .{ .value = value, .base = self.newBase(loc) } };
        return node;
    }
    
    pub fn ident(self: *AstBuilder, name: []const u8, loc: SourceLocation) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .identifier = .{ .name = name, .base = self.newBase(loc) } };
        return node;
    }
    
    pub fn binOp(
        self: *AstBuilder,
        op: BinaryOp,
        left: *Expr,
        right: *Expr,
        loc: SourceLocation,
    ) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .binary_op = .{
            .op = op,
            .left = left,
            .right = right,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    pub fn unaryOp(
        self: *AstBuilder,
        op: UnaryOp,
        operand: *Expr,
        loc: SourceLocation,
    ) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .unary_op = .{
            .op = op,
            .operand = operand,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    pub fn callExpr(
        self: *AstBuilder,
        callee: *Expr,
        args: []*Expr,
        loc: SourceLocation,
    ) !*Expr {
        const node = try self.allocator.create(Expr);
        node.* = .{ .call = .{
            .callee = callee,
            .args = args,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    // Statement builders
    pub fn blockStmt(
        self: *AstBuilder,
        statements: []*Stmt,
        loc: SourceLocation,
    ) !*Stmt {
        const node = try self.allocator.create(Stmt);
        node.* = .{ .block = .{
            .statements = statements,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    pub fn ifStmt(
        self: *AstBuilder,
        condition: *Expr,
        then_block: *Stmt,
        else_clause: ?*Stmt,
        loc: SourceLocation,
    ) !*Stmt {
        const node = try self.allocator.create(Stmt);
        node.* = .{ .if_stmt = .{
            .condition = condition,
            .then_block = then_block,
            .else_clause = else_clause,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    pub fn returnStmt(
        self: *AstBuilder,
        value: ?*Expr,
        loc: SourceLocation,
    ) !*Stmt {
        const node = try self.allocator.create(Stmt);
        node.* = .{ .return_stmt = .{
            .value = value,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    pub fn varDecl(
        self: *AstBuilder,
        name: []const u8,
        type_ref: ?TypeRef,
        initializer: ?*Expr,
        is_const: bool,
        loc: SourceLocation,
    ) !*Stmt {
        const node = try self.allocator.create(Stmt);
        node.* = .{ .var_decl = .{
            .name = name,
            .type_ref = type_ref,
            .initializer = initializer,
            .is_const = is_const,
            .is_mut = !is_const,
            .base = self.newBase(loc),
        }};
        return node;
    }
    
    // Declaration builders
    pub fn funcDecl(
        self: *AstBuilder,
        name: []const u8,
        params: []Decl.FunctionDecl.FuncParam,
        return_type: ?TypeRef,
        body: ?*Stmt,
        loc: SourceLocation,
    ) !*Decl {
        const node = try self.allocator.create(Decl);
        node.* = .{ .function = .{
            .name = name,
            .type_params = &.{},
            .params = params,
            .return_type = return_type,
            .body = body,
            .is_pub = false,
            .is_extern = false,
            .base = self.newBase(loc),
        }};
        return node;
    }
};

// ตัวอย่างการสร้าง AST ด้วย builder
test "build simple AST" {
    const allocator = std.testing.allocator;
    var builder = AstBuilder.init(allocator);
    
    const loc = SourceLocation{ .line = 1, .column = 1, .file = "test.lang" };
    
    // สร้าง: a + b
    const a = try builder.ident("a", loc);
    const b = try builder.ident("b", loc);
    const add = try builder.binOp(.add, a, b, loc);
    
    try std.testing.expect(add.* == .binary_op);
    try std.testing.expectEqual(BinaryOp.add, add.binary_op.op);
}
```

---

## Step 864: AST Traversal - Visitor Pattern

```zig
// ast_visitor.zig - Visitor pattern สำหรับ AST traversal

const std = @import("std");

// Generic Visitor interface
pub fn Visitor(comptime Context: type) type {
    return struct {
        const Self = @This();
        
        context: Context,
        
        // Expr visitors
        visitIntLit: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitFloatLit: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitStringLit: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitBoolLit: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitIdent: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitBinaryOp: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitUnaryOp: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        visitCall: *const fn (ctx: *Context, node: *const Expr) anyerror!void = defaultVisit,
        
        // Stmt visitors
        visitBlock: *const fn (ctx: *Context, node: *const Stmt) anyerror!void = defaultVisit,
        visitIfStmt: *const fn (ctx: *Context, node: *const Stmt) anyerror!void = defaultVisit,
        visitWhileStmt: *const fn (ctx: *Context, node: *const Stmt) anyerror!void = defaultVisit,
        visitReturnStmt: *const fn (ctx: *Context, node: *const Stmt) anyerror!void = defaultVisit,
        visitVarDecl: *const fn (ctx: *Context, node: *const Stmt) anyerror!void = defaultVisit,
        
        // Decl visitors
        visitFuncDecl: *const fn (ctx: *Context, node: *const Decl) anyerror!void = defaultVisit,
        visitStructDecl: *const fn (ctx: *Context, node: *const Decl) anyerror!void = defaultVisit,
        
        fn defaultVisit(_: *Context, _: anytype) anyerror!void {}
        
        // Traverse expression
        pub fn visitExpr(self: *Self, expr: *const Expr) !void {
            switch (expr.*) {
                .integer_lit => try self.visitIntLit(&self.context, expr),
                .float_lit => try self.visitFloatLit(&self.context, expr),
                .string_lit => try self.visitStringLit(&self.context, expr),
                .bool_lit => try self.visitBoolLit(&self.context, expr),
                .identifier => try self.visitIdent(&self.context, expr),
                .binary_op => |n| {
                    try self.visitExpr(n.left);
                    try self.visitExpr(n.right);
                    try self.visitBinaryOp(&self.context, expr);
                },
                .unary_op => |n| {
                    try self.visitExpr(n.operand);
                    try self.visitUnaryOp(&self.context, expr);
                },
                .call => |n| {
                    try self.visitExpr(n.callee);
                    for (n.args) |arg| try self.visitExpr(arg);
                    try self.visitCall(&self.context, expr);
                },
                .field_access => |n| {
                    try self.visitExpr(n.object);
                },
                .index => |n| {
                    try self.visitExpr(n.array);
                    try self.visitExpr(n.index);
                },
                .assignment => |n| {
                    try self.visitExpr(n.target);
                    try self.visitExpr(n.value);
                },
                else => {},
            }
        }
        
        // Traverse statement
        pub fn visitStmt(self: *Self, stmt: *const Stmt) !void {
            switch (stmt.*) {
                .expr_stmt => |n| try self.visitExpr(n.expr),
                .block => |n| {
                    for (n.statements) |s| try self.visitStmt(s);
                    try self.visitBlock(&self.context, stmt);
                },
                .if_stmt => |n| {
                    try self.visitExpr(n.condition);
                    try self.visitStmt(n.then_block);
                    if (n.else_clause) |else_s| try self.visitStmt(else_s);
                    try self.visitIfStmt(&self.context, stmt);
                },
                .while_stmt => |n| {
                    try self.visitExpr(n.condition);
                    try self.visitStmt(n.body);
                    try self.visitWhileStmt(&self.context, stmt);
                },
                .return_stmt => |n| {
                    if (n.value) |v| try self.visitExpr(v);
                    try self.visitReturnStmt(&self.context, stmt);
                },
                .var_decl => |n| {
                    if (n.initializer) |init| try self.visitExpr(init);
                    try self.visitVarDecl(&self.context, stmt);
                },
                else => {},
            }
        }
    };
}

// ตัวอย่าง: Visitor สำหรับนับจำนวน function calls
pub const CallCounter = struct {
    count: u32 = 0,
    
    fn onCall(self: *CallCounter, _: *const Expr) !void {
        self.count += 1;
    }
};

// ตัวอย่าง: Identifier collector
pub const IdentifierCollector = struct {
    identifiers: std.StringHashMap(u32),
    
    pub fn init(allocator: std.mem.Allocator) IdentifierCollector {
        return .{ .identifiers = std.StringHashMap(u32).init(allocator) };
    }
    
    pub fn deinit(self: *IdentifierCollector) void {
        self.identifiers.deinit();
    }
    
    fn onIdent(self: *IdentifierCollector, expr: *const Expr) !void {
        const name = expr.identifier.name;
        const result = try self.identifiers.getOrPut(name);
        if (!result.found_existing) {
            result.value_ptr.* = 0;
        }
        result.value_ptr.* += 1;
    }
};
```

---

## Step 865: AST Pretty Printing

```zig
// ast_printer.zig - Pretty printer สำหรับ AST

const std = @import("std");

pub const AstPrinter = struct {
    writer: std.io.AnyWriter,
    indent_level: u32 = 0,
    indent_str: []const u8 = "  ",
    
    pub fn init(writer: std.io.AnyWriter) AstPrinter {
        return AstPrinter{ .writer = writer };
    }
    
    fn indent(self: *AstPrinter) !void {
        var i: u32 = 0;
        while (i < self.indent_level) : (i += 1) {
            try self.writer.writeAll(self.indent_str);
        }
    }
    
    fn push(self: *AstPrinter) void {
        self.indent_level += 1;
    }
    
    fn pop(self: *AstPrinter) void {
        if (self.indent_level > 0) self.indent_level -= 1;
    }
    
    pub fn printProgram(self: *AstPrinter, program: *const Program) !void {
        try self.writer.writeAll("Program\n");
        self.push();
        for (program.declarations) |*decl| {
            try self.printDecl(decl);
        }
        self.pop();
    }
    
    pub fn printDecl(self: *AstPrinter, decl: *const Decl) !void {
        switch (decl.*) {
            .function => |f| {
                try self.indent();
                try self.writer.print("FunctionDecl '{s}'\n", .{f.name});
                self.push();
                
                if (f.params.len > 0) {
                    try self.indent();
                    try self.writer.writeAll("Params:\n");
                    self.push();
                    for (f.params) |p| {
                        try self.indent();
                        try self.writer.print("Param '{s}'\n", .{p.name});
                    }
                    self.pop();
                }
                
                if (f.return_type) |_| {
                    try self.indent();
                    try self.writer.writeAll("ReturnType: ...\n");
                }
                
                if (f.body) |body| {
                    try self.indent();
                    try self.writer.writeAll("Body:\n");
                    self.push();
                    try self.printStmt(body);
                    self.pop();
                }
                
                self.pop();
            },
            .struct_type => |s| {
                try self.indent();
                try self.writer.print("StructDecl '{s}'\n", .{s.name});
                self.push();
                for (s.fields) |field| {
                    try self.indent();
                    try self.writer.print("Field '{s}'\n", .{field.name});
                }
                self.pop();
            },
            else => {
                try self.indent();
                try self.writer.writeAll("Decl ...\n");
            },
        }
    }
    
    pub fn printStmt(self: *AstPrinter, stmt: *const Stmt) !void {
        switch (stmt.*) {
            .block => |n| {
                try self.indent();
                try self.writer.writeAll("Block\n");
                self.push();
                for (n.statements) |s| try self.printStmt(s);
                self.pop();
            },
            .if_stmt => |n| {
                try self.indent();
                try self.writer.writeAll("IfStmt\n");
                self.push();
                try self.indent();
                try self.writer.writeAll("Condition:\n");
                self.push();
                try self.printExpr(n.condition);
                self.pop();
                try self.indent();
                try self.writer.writeAll("Then:\n");
                self.push();
                try self.printStmt(n.then_block);
                self.pop();
                if (n.else_clause) |else_s| {
                    try self.indent();
                    try self.writer.writeAll("Else:\n");
                    self.push();
                    try self.printStmt(else_s);
                    self.pop();
                }
                self.pop();
            },
            .return_stmt => |n| {
                try self.indent();
                try self.writer.writeAll("ReturnStmt\n");
                if (n.value) |val| {
                    self.push();
                    try self.printExpr(val);
                    self.pop();
                }
            },
            .var_decl => |n| {
                try self.indent();
                try self.writer.print("VarDecl '{s}' ({})\n", .{
                    n.name,
                    if (n.is_const) @as([]const u8, "const") else "var",
                });
                if (n.initializer) |init| {
                    self.push();
                    try self.printExpr(init);
                    self.pop();
                }
            },
            .expr_stmt => |n| {
                try self.indent();
                try self.writer.writeAll("ExprStmt\n");
                self.push();
                try self.printExpr(n.expr);
                self.pop();
            },
            else => {
                try self.indent();
                try self.writer.writeAll("Stmt ...\n");
            },
        }
    }
    
    pub fn printExpr(self: *AstPrinter, expr: *const Expr) !void {
        switch (expr.*) {
            .integer_lit => |n| {
                try self.indent();
                try self.writer.print("IntLit({})\n", .{n.value});
            },
            .float_lit => |n| {
                try self.indent();
                try self.writer.print("FloatLit({})\n", .{n.value});
            },
            .string_lit => |n| {
                try self.indent();
                try self.writer.print("StringLit(\"{s}\")\n", .{n.value});
            },
            .bool_lit => |n| {
                try self.indent();
                try self.writer.print("BoolLit({})\n", .{n.value});
            },
            .identifier => |n| {
                try self.indent();
                try self.writer.print("Ident('{s}')\n", .{n.name});
            },
            .binary_op => |n| {
                try self.indent();
                try self.writer.print("BinaryOp({s})\n", .{@tagName(n.op)});
                self.push();
                try self.printExpr(n.left);
                try self.printExpr(n.right);
                self.pop();
            },
            .unary_op => |n| {
                try self.indent();
                try self.writer.print("UnaryOp({s})\n", .{@tagName(n.op)});
                self.push();
                try self.printExpr(n.operand);
                self.pop();
            },
            .call => |n| {
                try self.indent();
                try self.writer.writeAll("CallExpr\n");
                self.push();
                try self.indent();
                try self.writer.writeAll("Callee:\n");
                self.push();
                try self.printExpr(n.callee);
                self.pop();
                if (n.args.len > 0) {
                    try self.indent();
                    try self.writer.print("Args ({d}):\n", .{n.args.len});
                    self.push();
                    for (n.args) |arg| try self.printExpr(arg);
                    self.pop();
                }
                self.pop();
            },
            else => {
                try self.indent();
                try self.writer.writeAll("Expr ...\n");
            },
        }
    }
};

test "pretty print AST" {
    const allocator = std.testing.allocator;
    var builder = AstBuilder.init(allocator);
    const loc = SourceLocation{ .line = 1, .column = 1, .file = "test" };
    
    // สร้าง: 1 + 2 * 3
    const one = try builder.intLit(1, loc);
    const two = try builder.intLit(2, loc);
    const three = try builder.intLit(3, loc);
    const mul = try builder.binOp(.mul, two, three, loc);
    const add = try builder.binOp(.add, one, mul, loc);
    
    var buf = std.ArrayList(u8).init(allocator);
    defer buf.deinit();
    
    var printer = AstPrinter.init(buf.writer().any());
    try printer.printExpr(add);
    
    const output = buf.items;
    try std.testing.expect(std.mem.indexOf(u8, output, "BinaryOp(add)") != null);
    try std.testing.expect(std.mem.indexOf(u8, output, "BinaryOp(mul)") != null);
    try std.testing.expect(std.mem.indexOf(u8, output, "IntLit(1)") != null);
}
```

---

## Step 866: AST Transformation

```zig
// ast_transform.zig - การ transform AST

const std = @import("std");

// Transformer pattern - เหมือน Visitor แต่ return node ใหม่
pub const AstTransformer = struct {
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) AstTransformer {
        return .{ .allocator = allocator };
    }
    
    // Transform expression - return transformed copy
    pub fn transformExpr(self: *AstTransformer, expr: *const Expr) !*Expr {
        const new_expr = try self.allocator.create(Expr);
        
        switch (expr.*) {
            .binary_op => |n| {
                const new_left = try self.transformExpr(n.left);
                const new_right = try self.transformExpr(n.right);
                new_expr.* = .{ .binary_op = .{
                    .op = n.op,
                    .left = new_left,
                    .right = new_right,
                    .base = n.base,
                }};
            },
            .unary_op => |n| {
                const new_operand = try self.transformExpr(n.operand);
                new_expr.* = .{ .unary_op = .{
                    .op = n.op,
                    .operand = new_operand,
                    .base = n.base,
                }};
            },
            .call => |n| {
                const new_callee = try self.transformExpr(n.callee);
                const new_args = try self.allocator.alloc(*Expr, n.args.len);
                for (n.args, 0..) |arg, i| {
                    new_args[i] = try self.transformExpr(arg);
                }
                new_expr.* = .{ .call = .{
                    .callee = new_callee,
                    .args = new_args,
                    .base = n.base,
                }};
            },
            else => new_expr.* = expr.*,
        }
        
        return new_expr;
    }
};

// ตัวอย่าง: Desugaring transformation
// แปลง a += b ให้เป็น a = a + b
pub const DesugarTransformer = struct {
    base: AstTransformer,
    
    pub fn init(allocator: std.mem.Allocator) DesugarTransformer {
        return .{ .base = AstTransformer.init(allocator) };
    }
    
    pub fn desugarExpr(self: *DesugarTransformer, expr: *const Expr) !*Expr {
        switch (expr.*) {
            .assignment => |n| {
                if (n.op) |op| {
                    // a += b => a = a + b
                    const allocator = self.base.allocator;
                    const loc = n.base.loc;
                    
                    var builder = AstBuilder.init(allocator);
                    const left_copy = try self.desugarExpr(n.target);
                    const right_desugar = try self.desugarExpr(n.value);
                    const bin = try builder.binOp(op, left_copy, right_desugar, loc);
                    
                    const new_expr = try allocator.create(Expr);
                    new_expr.* = .{ .assignment = .{
                        .target = try self.desugarExpr(n.target),
                        .value = bin,
                        .op = null, // compound assignment ถูก expand แล้ว
                        .base = n.base,
                    }};
                    return new_expr;
                }
                return self.base.transformExpr(expr);
            },
            else => return self.base.transformExpr(expr),
        }
    }
};
```

---

## Step 867: AST Optimization

```zig
// ast_optimizer.zig - การ optimize AST

const std = @import("std");

pub const ConstantFolder = struct {
    allocator: std.mem.Allocator,
    folded_count: u32 = 0,
    
    pub fn init(allocator: std.mem.Allocator) ConstantFolder {
        return .{ .allocator = allocator };
    }
    
    // Constant folding: 1 + 2 => 3
    pub fn foldExpr(self: *ConstantFolder, expr: *const Expr) !*Expr {
        switch (expr.*) {
            .binary_op => |n| {
                const folded_left = try self.foldExpr(n.left);
                const folded_right = try self.foldExpr(n.right);
                
                // ลอง fold ถ้าทั้งสองฝั่งเป็น constant
                if (folded_left.* == .integer_lit and folded_right.* == .integer_lit) {
                    const lv = folded_left.integer_lit.value;
                    const rv = folded_right.integer_lit.value;
                    
                    const result: ?i64 = switch (n.op) {
                        .add => lv + rv,
                        .sub => lv - rv,
                        .mul => lv * rv,
                        .div => if (rv != 0) @divTrunc(lv, rv) else null,
                        .mod => if (rv != 0) @rem(lv, rv) else null,
                        else => null,
                    };
                    
                    if (result) |r| {
                        self.folded_count += 1;
                        const new_node = try self.allocator.create(Expr);
                        new_node.* = .{ .integer_lit = .{
                            .value = r,
                            .base = n.base,
                        }};
                        return new_node;
                    }
                }
                
                // Float constant folding
                if (folded_left.* == .float_lit and folded_right.* == .float_lit) {
                    const lv = folded_left.float_lit.value;
                    const rv = folded_right.float_lit.value;
                    
                    const result: ?f64 = switch (n.op) {
                        .add => lv + rv,
                        .sub => lv - rv,
                        .mul => lv * rv,
                        .div => lv / rv,
                        else => null,
                    };
                    
                    if (result) |r| {
                        self.folded_count += 1;
                        const new_node = try self.allocator.create(Expr);
                        new_node.* = .{ .float_lit = .{
                            .value = r,
                            .base = n.base,
                        }};
                        return new_node;
                    }
                }
                
                // Boolean constant folding
                if (folded_left.* == .bool_lit and folded_right.* == .bool_lit) {
                    const lv = folded_left.bool_lit.value;
                    const rv = folded_right.bool_lit.value;
                    
                    const result: ?bool = switch (n.op) {
                        .and_ => lv and rv,
                        .or_ => lv or rv,
                        else => null,
                    };
                    
                    if (result) |r| {
                        self.folded_count += 1;
                        const new_node = try self.allocator.create(Expr);
                        new_node.* = .{ .bool_lit = .{
                            .value = r,
                            .base = n.base,
                        }};
                        return new_node;
                    }
                }
                
                // ไม่สามารถ fold ได้ return node เดิม (ที่ optimize children แล้ว)
                const new_node = try self.allocator.create(Expr);
                new_node.* = .{ .binary_op = .{
                    .op = n.op,
                    .left = folded_left,
                    .right = folded_right,
                    .base = n.base,
                }};
                return new_node;
            },
            
            .unary_op => |n| {
                const folded_operand = try self.foldExpr(n.operand);
                
                if (folded_operand.* == .integer_lit) {
                    const v = folded_operand.integer_lit.value;
                    if (n.op == .neg) {
                        self.folded_count += 1;
                        const new_node = try self.allocator.create(Expr);
                        new_node.* = .{ .integer_lit = .{ .value = -v, .base = n.base } };
                        return new_node;
                    }
                }
                
                if (folded_operand.* == .bool_lit and n.op == .not) {
                    self.folded_count += 1;
                    const new_node = try self.allocator.create(Expr);
                    new_node.* = .{ .bool_lit = .{
                        .value = !folded_operand.bool_lit.value,
                        .base = n.base,
                    }};
                    return new_node;
                }
                
                const new_node = try self.allocator.create(Expr);
                new_node.* = .{ .unary_op = .{
                    .op = n.op,
                    .operand = folded_operand,
                    .base = n.base,
                }};
                return new_node;
            },
            
            else => {
                // ไม่ต้อง fold - return copy
                const new_node = try self.allocator.create(Expr);
                new_node.* = expr.*;
                return new_node;
            },
        }
    }
};

test "constant folding" {
    const allocator = std.testing.allocator;
    var builder = AstBuilder.init(allocator);
    const loc = SourceLocation{ .line = 1, .column = 1, .file = "test" };
    
    // 2 + 3 * 4 => 2 + 12 => 14
    const two = try builder.intLit(2, loc);
    const three = try builder.intLit(3, loc);
    const four = try builder.intLit(4, loc);
    const mul = try builder.binOp(.mul, three, four, loc);
    const add = try builder.binOp(.add, two, mul, loc);
    
    var folder = ConstantFolder.init(allocator);
    const result = try folder.foldExpr(add);
    
    try std.testing.expect(result.* == .integer_lit);
    try std.testing.expectEqual(@as(i64, 14), result.integer_lit.value);
    try std.testing.expectEqual(@as(u32, 2), folder.folded_count);
}
```

---

## Step 868: Symbol Table

```zig
// symbol_table.zig - การจัดการ symbol table สำหรับ name resolution

const std = @import("std");

pub const SymbolKind = enum {
    variable,
    function,
    type_def,
    parameter,
    field,
};

pub const Symbol = struct {
    name: []const u8,
    kind: SymbolKind,
    type_ref: ?TypeRef,
    decl_loc: SourceLocation,
    is_initialized: bool,
    is_used: bool,
};

pub const Scope = struct {
    symbols: std.StringHashMap(Symbol),
    parent: ?*Scope,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, parent: ?*Scope) Scope {
        return Scope{
            .symbols = std.StringHashMap(Symbol).init(allocator),
            .parent = parent,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Scope) void {
        self.symbols.deinit();
    }
    
    pub fn define(self: *Scope, symbol: Symbol) !void {
        if (self.symbols.contains(symbol.name)) {
            return error.DuplicateDefinition;
        }
        try self.symbols.put(symbol.name, symbol);
    }
    
    pub fn lookup(self: *Scope, name: []const u8) ?*Symbol {
        if (self.symbols.getPtr(name)) |sym| return sym;
        if (self.parent) |parent| return parent.lookup(name);
        return null;
    }
    
    pub fn lookupLocal(self: *Scope, name: []const u8) ?*Symbol {
        return self.symbols.getPtr(name);
    }
};

pub const SymbolTable = struct {
    allocator: std.mem.Allocator,
    global_scope: Scope,
    current_scope: *Scope,
    scope_stack: std.ArrayList(*Scope),
    
    pub fn init(allocator: std.mem.Allocator) !SymbolTable {
        const st = SymbolTable{
            .allocator = allocator,
            .global_scope = Scope.init(allocator, null),
            .current_scope = undefined,
            .scope_stack = std.ArrayList(*Scope).init(allocator),
        };
        return st;
    }
    
    pub fn deinit(self: *SymbolTable) void {
        for (self.scope_stack.items) |scope| {
            scope.deinit();
            self.allocator.destroy(scope);
        }
        self.scope_stack.deinit();
        self.global_scope.deinit();
    }
    
    pub fn enterScope(self: *SymbolTable) !void {
        const new_scope = try self.allocator.create(Scope);
        new_scope.* = Scope.init(self.allocator, self.current_scope);
        try self.scope_stack.append(new_scope);
        self.current_scope = new_scope;
    }
    
    pub fn exitScope(self: *SymbolTable) void {
        if (self.scope_stack.items.len > 0) {
            const scope = self.scope_stack.pop();
            self.current_scope = scope.parent orelse &self.global_scope;
        }
    }
    
    pub fn define(self: *SymbolTable, symbol: Symbol) !void {
        try self.current_scope.define(symbol);
    }
    
    pub fn lookup(self: *SymbolTable, name: []const u8) ?*Symbol {
        return self.current_scope.lookup(name);
    }
};

// Name resolver - resolves identifiers ใน AST
pub const NameResolver = struct {
    symbol_table: SymbolTable,
    errors: std.ArrayList([]const u8),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) !NameResolver {
        return NameResolver{
            .symbol_table = try SymbolTable.init(allocator),
            .errors = std.ArrayList([]const u8).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *NameResolver) void {
        self.symbol_table.deinit();
        for (self.errors.items) |e| self.allocator.free(e);
        self.errors.deinit();
    }
    
    pub fn resolveProgram(self: *NameResolver, program: *const Program) !void {
        // First pass: collect top-level declarations
        for (program.declarations) |*decl| {
            try self.collectDecl(decl);
        }
        
        // Second pass: resolve bodies
        for (program.declarations) |*decl| {
            try self.resolveDecl(decl);
        }
    }
    
    fn collectDecl(self: *NameResolver, decl: *const Decl) !void {
        switch (decl.*) {
            .function => |f| {
                const loc = f.base.loc;
                try self.symbol_table.define(.{
                    .name = f.name,
                    .kind = .function,
                    .type_ref = null,
                    .decl_loc = loc,
                    .is_initialized = true,
                    .is_used = false,
                });
            },
            else => {},
        }
    }
    
    fn resolveDecl(self: *NameResolver, decl: *const Decl) !void {
        switch (decl.*) {
            .function => |f| {
                try self.symbol_table.enterScope();
                defer self.symbol_table.exitScope();
                
                // Add parameters to scope
                for (f.params) |p| {
                    try self.symbol_table.define(.{
                        .name = p.name,
                        .kind = .parameter,
                        .type_ref = p.type_ref,
                        .decl_loc = f.base.loc,
                        .is_initialized = true,
                        .is_used = false,
                    });
                }
                
                if (f.body) |body| {
                    try self.resolveStmt(body);
                }
            },
            else => {},
        }
    }
    
    fn resolveStmt(self: *NameResolver, stmt: *const Stmt) !void {
        switch (stmt.*) {
            .block => |n| {
                try self.symbol_table.enterScope();
                defer self.symbol_table.exitScope();
                for (n.statements) |s| try self.resolveStmt(s);
            },
            .var_decl => |n| {
                if (n.initializer) |init| try self.resolveExpr(init);
                try self.symbol_table.define(.{
                    .name = n.name,
                    .kind = .variable,
                    .type_ref = n.type_ref,
                    .decl_loc = n.base.loc,
                    .is_initialized = n.initializer != null,
                    .is_used = false,
                });
            },
            .if_stmt => |n| {
                try self.resolveExpr(n.condition);
                try self.resolveStmt(n.then_block);
                if (n.else_clause) |e| try self.resolveStmt(e);
            },
            .return_stmt => |n| {
                if (n.value) |v| try self.resolveExpr(v);
            },
            .expr_stmt => |n| try self.resolveExpr(n.expr),
            else => {},
        }
    }
    
    fn resolveExpr(self: *NameResolver, expr: *const Expr) !void {
        switch (expr.*) {
            .identifier => |n| {
                if (self.symbol_table.lookup(n.name) == null) {
                    const msg = try std.fmt.allocPrint(
                        self.allocator,
                        "Undefined variable '{s}' at {s}",
                        .{ n.name, n.base.loc },
                    );
                    try self.errors.append(msg);
                } else {
                    // Mark as used
                    if (self.symbol_table.lookup(n.name)) |sym| {
                        sym.is_used = true;
                    }
                }
            },
            .binary_op => |n| {
                try self.resolveExpr(n.left);
                try self.resolveExpr(n.right);
            },
            .call => |n| {
                try self.resolveExpr(n.callee);
                for (n.args) |arg| try self.resolveExpr(arg);
            },
            else => {},
        }
    }
};
```

---

## Step 869: Type Checking

```zig
// type_checker.zig - Type checking บน AST

const std = @import("std");

pub const TypeError = struct {
    message: []const u8,
    loc: SourceLocation,
};

pub const TypeInfo = union(enum) {
    int,
    float,
    bool,
    string,
    void,
    function: struct {
        param_types: []TypeInfo,
        return_type: *TypeInfo,
    },
    unknown,
    
    pub fn eql(a: TypeInfo, b: TypeInfo) bool {
        return switch (a) {
            .int => b == .int,
            .float => b == .float,
            .bool => b == .bool,
            .string => b == .string,
            .void => b == .void,
            .unknown => true, // unknown เป็น compatible กับทุกอย่าง
            .function => false, // simplified
        };
    }
    
    pub fn name(self: TypeInfo) []const u8 {
        return switch (self) {
            .int => "int",
            .float => "float",
            .bool => "bool",
            .string => "string",
            .void => "void",
            .function => "function",
            .unknown => "unknown",
        };
    }
};

pub const TypeChecker = struct {
    allocator: std.mem.Allocator,
    errors: std.ArrayList(TypeError),
    type_map: std.AutoHashMap(NodeId, TypeInfo), // node id -> type
    symbol_types: std.StringHashMap(TypeInfo),
    
    pub fn init(allocator: std.mem.Allocator) TypeChecker {
        return TypeChecker{
            .allocator = allocator,
            .errors = std.ArrayList(TypeError).init(allocator),
            .type_map = std.AutoHashMap(NodeId, TypeInfo).init(allocator),
            .symbol_types = std.StringHashMap(TypeInfo).init(allocator),
        };
    }
    
    pub fn deinit(self: *TypeChecker) void {
        for (self.errors.items) |e| self.allocator.free(e.message);
        self.errors.deinit();
        self.type_map.deinit();
        self.symbol_types.deinit();
    }
    
    fn addError(self: *TypeChecker, loc: SourceLocation, comptime fmt: []const u8, args: anytype) !void {
        const msg = try std.fmt.allocPrint(self.allocator, fmt, args);
        try self.errors.append(.{ .message = msg, .loc = loc });
    }
    
    pub fn inferExprType(self: *TypeChecker, expr: *const Expr) !TypeInfo {
        const ty: TypeInfo = switch (expr.*) {
            .integer_lit => .int,
            .float_lit => .float,
            .bool_lit => .bool,
            .string_lit => .string,
            .null_lit => .unknown,
            
            .identifier => |n| {
                if (self.symbol_types.get(n.name)) |t| t else .unknown;
            },
            
            .binary_op => |n| {
                const left_type = try self.inferExprType(n.left);
                const right_type = try self.inferExprType(n.right);
                
                // Check compatibility
                if (!left_type.eql(right_type)) {
                    try self.addError(
                        n.base.loc,
                        "Type mismatch: cannot apply '{s}' to '{s}' and '{s}'",
                        .{ @tagName(n.op), left_type.name(), right_type.name() },
                    );
                    return .unknown;
                }
                
                // Determine result type based on operation
                return switch (n.op) {
                    .add, .sub, .mul, .div, .mod => left_type,
                    .eq, .ne, .lt, .le, .gt, .ge => TypeInfo.bool,
                    .and_, .or_ => TypeInfo.bool,
                    else => left_type,
                };
            },
            
            .unary_op => |n| {
                const operand_type = try self.inferExprType(n.operand);
                return switch (n.op) {
                    .neg => if (operand_type == .int or operand_type == .float)
                        operand_type else blk: {
                        try self.addError(n.base.loc, "Cannot negate non-numeric type", .{});
                        break :blk .unknown;
                    },
                    .not => if (operand_type == .bool) TypeInfo.bool else blk: {
                        try self.addError(n.base.loc, "Cannot apply '!' to non-boolean", .{});
                        break :blk .unknown;
                    },
                    else => operand_type,
                };
            },
            
            .call => .unknown, // simplified - would look up function signature
            
            else => .unknown,
        };
        
        // Store type in map
        try self.type_map.put(expr.getBase().id, ty);
        return ty;
    }
    
    pub fn checkStmt(self: *TypeChecker, stmt: *const Stmt) !void {
        switch (stmt.*) {
            .var_decl => |n| {
                if (n.initializer) |init| {
                    const init_type = try self.inferExprType(init);
                    
                    // Store type for later lookups
                    if (n.type_ref) |_| {
                        // TODO: check against declared type
                    }
                    
                    try self.symbol_types.put(n.name, init_type);
                }
            },
            .if_stmt => |n| {
                const cond_type = try self.inferExprType(n.condition);
                if (cond_type != .bool and cond_type != .unknown) {
                    try self.addError(
                        n.base.loc,
                        "If condition must be boolean, got '{s}'",
                        .{cond_type.name()},
                    );
                }
                try self.checkStmt(n.then_block);
                if (n.else_clause) |e| try self.checkStmt(e);
            },
            .block => |n| {
                for (n.statements) |s| try self.checkStmt(s);
            },
            .return_stmt => |n| {
                if (n.value) |v| _ = try self.inferExprType(v);
            },
            .expr_stmt => |n| _ = try self.inferExprType(n.expr),
            else => {},
        }
    }
};
```

---

## Step 870: Complete AST for Simple Language

```zig
// complete_ast.zig - AST สมบูรณ์สำหรับภาษาง่ายๆ

const std = @import("std");

// รวมทุกอย่างเข้าด้วยกัน
pub const Ast = struct {
    allocator: std.mem.Allocator,
    arena: std.heap.ArenaAllocator, // ใช้ arena allocator สำหรับ AST nodes
    program: *Program,
    
    pub fn init(allocator: std.mem.Allocator) !Ast {
        var arena = std.heap.ArenaAllocator.init(allocator);
        const program = try arena.allocator().create(Program);
        program.* = Program.init(arena.allocator());
        
        return Ast{
            .allocator = allocator,
            .arena = arena,
            .program = program,
        };
    }
    
    pub fn deinit(self: *Ast) void {
        self.arena.deinit();
    }
    
    pub fn arena_alloc(self: *Ast) std.mem.Allocator {
        return self.arena.allocator();
    }
};

// Complete example: parse and analyze a simple program
pub fn analyzeProgram(source: []const u8, allocator: std.mem.Allocator) !void {
    _ = source;
    
    var ast = try Ast.init(allocator);
    defer ast.deinit();
    
    // สร้าง AST manually สำหรับ:
    // fn factorial(n: int): int {
    //     if (n <= 1) {
    //         return 1;
    //     }
    //     return n * factorial(n - 1);
    // }
    
    const arena = ast.arena_alloc();
    var builder = AstBuilder.init(arena);
    const loc = SourceLocation{ .line = 1, .column = 1, .file = "test.lang" };
    
    // Build: n <= 1
    const n_ident1 = try builder.ident("n", loc);
    const one1 = try builder.intLit(1, loc);
    const cond = try builder.binOp(.le, n_ident1, one1, loc);
    
    // Build: return 1
    const one2 = try builder.intLit(1, loc);
    const ret1 = try builder.returnStmt(one2, loc);
    
    // Build if block
    var if_stmts = [_]*Stmt{ret1};
    const if_block = try builder.blockStmt(&if_stmts, loc);
    
    // Build: factorial(n - 1) call
    const n_ident2 = try builder.ident("n", loc);
    const one3 = try builder.intLit(1, loc);
    const n_minus_1 = try builder.binOp(.sub, n_ident2, one3, loc);
    const factorial_ident = try builder.ident("factorial", loc);
    var call_args = [_]*Expr{n_minus_1};
    const recursive_call = try builder.callExpr(factorial_ident, &call_args, loc);
    
    // Build: n * factorial(n - 1)
    const n_ident3 = try builder.ident("n", loc);
    const mul_expr = try builder.binOp(.mul, n_ident3, recursive_call, loc);
    
    // Build: return n * factorial(n-1)
    const ret2 = try builder.returnStmt(mul_expr, loc);
    
    // Build function body
    const if_stmt_node = try arena.create(Stmt);
    if_stmt_node.* = .{ .if_stmt = .{
        .condition = cond,
        .then_block = if_block,
        .else_clause = null,
        .base = .{ .id = 0, .loc = loc },
    }};
    
    var body_stmts = [_]*Stmt{ if_stmt_node, ret2 };
    const fn_body = try builder.blockStmt(&body_stmts, loc);
    
    // Build function parameter
    var params = [_]Decl.FunctionDecl.FuncParam{.{
        .name = "n",
        .type_ref = .{ .primitive = .int },
        .default_value = null,
        .is_mut = false,
    }};
    
    const fn_decl = try builder.funcDecl(
        "factorial",
        &params,
        TypeRef{ .primitive = .int },
        fn_body,
        loc,
    );
    
    // Type check
    var type_checker = TypeChecker.init(allocator);
    defer type_checker.deinit();
    
    if (fn_decl.function.body) |body| {
        try type_checker.checkStmt(body);
    }
    
    // Name resolution
    var resolver = try NameResolver.init(allocator);
    defer resolver.deinit();
    
    // Pretty print
    var output = std.ArrayList(u8).init(allocator);
    defer output.deinit();
    
    var printer = AstPrinter.init(output.writer().any());
    try printer.printDecl(fn_decl);
    
    std.debug.print("AST Output:\n{s}\n", .{output.items});
    
    if (type_checker.errors.items.len > 0) {
        std.debug.print("Type Errors:\n", .{});
        for (type_checker.errors.items) |err| {
            std.debug.print("  {s}\n", .{err.message});
        }
    } else {
        std.debug.print("Type checking passed!\n", .{});
    }
}

test "complete AST analysis" {
    const allocator = std.testing.allocator;
    try analyzeProgram("", allocator);
}
```

---

## สรุป Part 87

ในส่วนนี้เราได้เรียนรู้:

1. **Step 861**: AST concept - ทำไมต้องมี AST และหลักการออกแบบ
2. **Step 862**: AST node types - การใช้ union types ใน Zig สำหรับ AST
3. **Step 863**: AST builder - Helper functions สำหรับสร้าง nodes ง่ายๆ
4. **Step 864**: AST traversal - Visitor pattern สำหรับ traverse AST
5. **Step 865**: AST pretty printing - แสดงผล AST ในรูปแบบที่อ่านได้
6. **Step 866**: AST transformation - การ transform AST (เช่น desugaring)
7. **Step 867**: AST optimization - Constant folding
8. **Step 868**: Symbol table - การจัดการ scope และ name resolution
9. **Step 869**: Type checking - การตรวจสอบ type ใน AST
10. **Step 870**: Complete AST - รวมทุกอย่างสำหรับภาษาตัวอย่าง

AST เป็นหัวใจสำคัญของ compiler และ interpreter ที่ดี การออกแบบ AST ที่ดีทำให้การพัฒนา feature ใหม่ๆ ง่ายขึ้นมาก
