# ส่วนที่ 88: Compiler Basics ใน Zig

## ขั้นตอนที่ 871-880: การสร้าง Compiler อย่างง่ายด้วย Zig

Compiler คือโปรแกรมที่แปลงภาษา source code ให้กลายเป็น target code (machine code, bytecode, หรือ IR) ในส่วนนี้เราจะสร้าง mini compiler ที่สมบูรณ์

---

## Step 871: Compiler Architecture

### สถาปัตยกรรมของ Compiler

```
Source Code
    ↓
[Lexer/Scanner]     → Tokens
    ↓
[Parser]            → AST
    ↓
[Semantic Analysis] → Annotated AST
    ↓
[IR Generation]     → Intermediate Representation
    ↓
[Optimization]      → Optimized IR
    ↓
[Code Generation]   → Target Code
    ↓
Target Code (bytecode, machine code)
```

```zig
// compiler_arch.zig - โครงสร้างหลักของ Compiler

const std = @import("std");

pub const CompilerConfig = struct {
    optimize: bool = false,
    debug_info: bool = true,
    target: Target = .bytecode,
    
    pub const Target = enum {
        bytecode,    // Stack-based VM bytecode
        llvm_ir,     // LLVM IR
        wasm,        // WebAssembly
    };
};

pub const CompilerError = union(enum) {
    lex_error: struct {
        message: []const u8,
        line: u32,
        column: u32,
    },
    parse_error: struct {
        message: []const u8,
        line: u32,
        column: u32,
    },
    semantic_error: struct {
        message: []const u8,
        line: u32,
        column: u32,
    },
    codegen_error: struct {
        message: []const u8,
    },
};

pub const CompileResult = struct {
    errors: []CompilerError,
    bytecode: ?[]const u8,
    symbols: ?SymbolTable,
    
    pub fn success(self: *const CompileResult) bool {
        return self.errors.len == 0;
    }
};

// Pipeline ของ Compiler
pub const Compiler = struct {
    allocator: std.mem.Allocator,
    config: CompilerConfig,
    
    pub fn init(allocator: std.mem.Allocator, config: CompilerConfig) Compiler {
        return Compiler{
            .allocator = allocator,
            .config = config,
        };
    }
    
    pub fn compile(self: *Compiler, source: []const u8) !CompileResult {
        var errors = std.ArrayList(CompilerError).init(self.allocator);
        
        // Phase 1: Lexing
        var lexer = Lexer.init(source);
        const tokens = lexer.tokenize(self.allocator) catch |err| {
            try errors.append(.{ .lex_error = .{
                .message = @errorName(err),
                .line = lexer.line,
                .column = lexer.column,
            }});
            return CompileResult{
                .errors = try errors.toOwnedSlice(),
                .bytecode = null,
                .symbols = null,
            };
        };
        defer self.allocator.free(tokens);
        
        // Phase 2: Parsing
        var parser = Parser.init(tokens, self.allocator);
        defer parser.deinit();
        
        const ast = parser.parseProgram() catch |err| {
            try errors.append(.{ .parse_error = .{
                .message = @errorName(err),
                .line = 0,
                .column = 0,
            }});
            return CompileResult{
                .errors = try errors.toOwnedSlice(),
                .bytecode = null,
                .symbols = null,
            };
        };
        defer self.allocator.free(ast);
        
        // Phase 3: Semantic Analysis
        var analyzer = SemanticAnalyzer.init(self.allocator);
        defer analyzer.deinit();
        
        analyzer.analyze(ast) catch {};
        
        for (analyzer.errors.items) |err| {
            try errors.append(.{ .semantic_error = .{
                .message = err.message,
                .line = err.loc.line,
                .column = err.loc.column,
            }});
        }
        
        if (errors.items.len > 0) {
            return CompileResult{
                .errors = try errors.toOwnedSlice(),
                .bytecode = null,
                .symbols = null,
            };
        }
        
        // Phase 4: Code Generation
        var codegen = BytecodeGenerator.init(self.allocator);
        defer codegen.deinit();
        
        const bytecode = codegen.generate(ast) catch |err| {
            try errors.append(.{ .codegen_error = .{
                .message = @errorName(err),
            }});
            return CompileResult{
                .errors = try errors.toOwnedSlice(),
                .bytecode = null,
                .symbols = null,
            };
        };
        
        return CompileResult{
            .errors = try errors.toOwnedSlice(),
            .bytecode = bytecode,
            .symbols = null,
        };
    }
};
```

---

## Step 872: Code Generation Concepts

### แนวคิดการ Generate Code

```zig
// code_gen_concepts.zig - แนวคิดพื้นฐานของ Code Generation

// Intermediate Representation (IR)
// เป็น lower-level กว่า AST แต่ higher-level กว่า machine code
// ทำให้ optimization และ code gen ง่ายขึ้น

pub const IrInstr = union(enum) {
    // Constants
    const_int: i64,
    const_float: f64,
    const_bool: bool,
    const_string: []const u8,
    const_null: void,
    
    // Variables
    load_local: u32,     // โหลด local variable #n
    store_local: u32,    // เก็บ local variable #n
    load_global: []const u8,
    store_global: []const u8,
    
    // Arithmetic
    add: void,
    sub: void,
    mul: void,
    div: void,
    mod: void,
    neg: void,
    
    // Comparison
    eq: void,
    ne: void,
    lt: void,
    le: void,
    gt: void,
    ge: void,
    
    // Logical
    and_op: void,
    or_op: void,
    not_op: void,
    
    // Control flow
    jump: u32,          // ไปที่ instruction #n
    jump_if: u32,       // ถ้า top of stack เป็น true ไป #n
    jump_if_false: u32,
    
    // Functions
    call: struct {
        name: []const u8,
        arg_count: u32,
    },
    call_indirect: u32,  // call function at stack position
    ret: void,
    ret_val: void,       // return with value
    
    // Stack manipulation
    pop: void,
    dup: void,
    
    // Objects/Arrays
    make_array: u32,    // สร้าง array ขนาด n
    array_get: void,
    array_set: void,
    make_object: u32,
    object_get: []const u8,
    object_set: []const u8,
    
    // Special
    nop: void,
    halt: void,
    print: void,        // สำหรับ built-in print
};

// Basic Block - sequence ของ instructions ที่ไม่มี branch ตรงกลาง
pub const BasicBlock = struct {
    label: []const u8,
    instructions: []IrInstr,
    successors: []u32, // indices ของ blocks ที่ตามมา
    predecessors: []u32,
};

// Control Flow Graph
pub const Cfg = struct {
    blocks: []BasicBlock,
    entry: u32,
    exit: u32,
    
    pub fn dominators(self: *const Cfg, allocator: std.mem.Allocator) ![]u32 {
        // TODO: compute dominator tree
        _ = self;
        _ = allocator;
        return &.{};
    }
};

// Function แสดงผล IR instructions
pub fn printIr(instrs: []const IrInstr, writer: anytype) !void {
    for (instrs, 0..) |instr, i| {
        try writer.print("{d:4}: ", .{i});
        switch (instr) {
            .const_int => |v| try writer.print("CONST_INT {d}\n", .{v}),
            .const_float => |v| try writer.print("CONST_FLOAT {d}\n", .{v}),
            .const_bool => |v| try writer.print("CONST_BOOL {}\n", .{v}),
            .const_string => |v| try writer.print("CONST_STR \"{s}\"\n", .{v}),
            .const_null => try writer.writeAll("CONST_NULL\n"),
            .load_local => |n| try writer.print("LOAD_LOCAL {d}\n", .{n}),
            .store_local => |n| try writer.print("STORE_LOCAL {d}\n", .{n}),
            .load_global => |name| try writer.print("LOAD_GLOBAL {s}\n", .{name}),
            .store_global => |name| try writer.print("STORE_GLOBAL {s}\n", .{name}),
            .add => try writer.writeAll("ADD\n"),
            .sub => try writer.writeAll("SUB\n"),
            .mul => try writer.writeAll("MUL\n"),
            .div => try writer.writeAll("DIV\n"),
            .mod => try writer.writeAll("MOD\n"),
            .neg => try writer.writeAll("NEG\n"),
            .eq => try writer.writeAll("EQ\n"),
            .ne => try writer.writeAll("NE\n"),
            .lt => try writer.writeAll("LT\n"),
            .le => try writer.writeAll("LE\n"),
            .gt => try writer.writeAll("GT\n"),
            .ge => try writer.writeAll("GE\n"),
            .and_op => try writer.writeAll("AND\n"),
            .or_op => try writer.writeAll("OR\n"),
            .not_op => try writer.writeAll("NOT\n"),
            .jump => |target| try writer.print("JUMP {d}\n", .{target}),
            .jump_if => |target| try writer.print("JUMP_IF {d}\n", .{target}),
            .jump_if_false => |target| try writer.print("JUMP_IF_FALSE {d}\n", .{target}),
            .call => |c| try writer.print("CALL {s} ({d} args)\n", .{ c.name, c.arg_count }),
            .ret => try writer.writeAll("RET\n"),
            .ret_val => try writer.writeAll("RET_VAL\n"),
            .pop => try writer.writeAll("POP\n"),
            .dup => try writer.writeAll("DUP\n"),
            .halt => try writer.writeAll("HALT\n"),
            .print => try writer.writeAll("PRINT\n"),
            else => try writer.writeAll("...\n"),
        }
    }
}
```

---

## Step 873: Stack-Based VM Bytecode

```zig
// vm_bytecode.zig - Stack-based VM bytecode format

const std = @import("std");

// Opcode definitions (1 byte each)
pub const Opcode = enum(u8) {
    // Literals
    op_nil    = 0x00,
    op_true   = 0x01,
    op_false  = 0x02,
    op_i8     = 0x03,  // followed by 1 byte signed int
    op_i16    = 0x04,  // followed by 2 byte signed int
    op_i32    = 0x05,  // followed by 4 byte signed int
    op_i64    = 0x06,  // followed by 8 byte signed int
    op_f32    = 0x07,  // followed by 4 byte float
    op_f64    = 0x08,  // followed by 8 byte float
    op_const  = 0x09,  // constant pool index (2 bytes)
    
    // Stack
    op_pop    = 0x10,
    op_dup    = 0x11,
    op_swap   = 0x12,
    
    // Local variables
    op_get_local  = 0x20,  // local index (1 byte)
    op_set_local  = 0x21,
    op_get_local_w = 0x22, // local index (2 bytes)
    op_set_local_w = 0x23,
    
    // Global variables
    op_get_global = 0x30,  // string constant index
    op_set_global = 0x31,
    
    // Arithmetic
    op_add_i = 0x40,
    op_sub_i = 0x41,
    op_mul_i = 0x42,
    op_div_i = 0x43,
    op_mod_i = 0x44,
    op_neg_i = 0x45,
    op_add_f = 0x48,
    op_sub_f = 0x49,
    op_mul_f = 0x4A,
    op_div_f = 0x4B,
    op_neg_f = 0x4C,
    
    // Comparison
    op_eq    = 0x50,
    op_ne    = 0x51,
    op_lt_i  = 0x52,
    op_le_i  = 0x53,
    op_gt_i  = 0x54,
    op_ge_i  = 0x55,
    op_lt_f  = 0x56,
    op_le_f  = 0x57,
    op_gt_f  = 0x58,
    op_ge_f  = 0x59,
    
    // Logical
    op_and   = 0x60,
    op_or    = 0x61,
    op_not   = 0x62,
    
    // Control flow
    op_jump        = 0x70,  // 2-byte offset
    op_jump_true   = 0x71,
    op_jump_false  = 0x72,
    
    // Functions
    op_call    = 0x80,  // function name index + arg count
    op_return  = 0x81,
    op_return_val = 0x82,
    
    // Objects/Arrays
    op_make_array  = 0x90,
    op_array_get   = 0x91,
    op_array_set   = 0x92,
    op_array_len   = 0x93,
    
    // Built-ins
    op_print   = 0xF0,
    op_halt    = 0xFF,
    
    _,
};

// Bytecode chunk (สำหรับ function หนึ่งฟังก์ชัน)
pub const Chunk = struct {
    code: std.ArrayList(u8),
    constants: std.ArrayList(Value),
    lines: std.ArrayList(u32), // สำหรับ debug info
    
    pub fn init(allocator: std.mem.Allocator) Chunk {
        return Chunk{
            .code = std.ArrayList(u8).init(allocator),
            .constants = std.ArrayList(Value).init(allocator),
            .lines = std.ArrayList(u32).init(allocator),
        };
    }
    
    pub fn deinit(self: *Chunk) void {
        self.code.deinit();
        self.constants.deinit();
        self.lines.deinit();
    }
    
    pub fn write(self: *Chunk, byte: u8, line: u32) !void {
        try self.code.append(byte);
        try self.lines.append(line);
    }
    
    pub fn writeOpcode(self: *Chunk, op: Opcode, line: u32) !void {
        try self.write(@intFromEnum(op), line);
    }
    
    pub fn addConstant(self: *Chunk, value: Value) !u16 {
        const idx = @as(u16, @intCast(self.constants.items.len));
        try self.constants.append(value);
        return idx;
    }
    
    // Emit constant instruction
    pub fn emitConst(self: *Chunk, value: Value, line: u32) !void {
        const idx = try self.addConstant(value);
        try self.writeOpcode(.op_const, line);
        try self.write(@intCast(idx >> 8), line);
        try self.write(@intCast(idx & 0xFF), line);
    }
    
    // Emit jump instruction (returns patch location)
    pub fn emitJump(self: *Chunk, op: Opcode, line: u32) !usize {
        try self.writeOpcode(op, line);
        try self.write(0xFF, line); // placeholder
        try self.write(0xFF, line); // placeholder
        return self.code.items.len - 2;
    }
    
    // Patch jump destination
    pub fn patchJump(self: *Chunk, offset: usize) void {
        const jump = self.code.items.len - offset - 2;
        self.code.items[offset] = @intCast(jump >> 8);
        self.code.items[offset + 1] = @intCast(jump & 0xFF);
    }
    
    // Emit loop back jump
    pub fn emitLoop(self: *Chunk, loop_start: usize, line: u32) !void {
        try self.writeOpcode(.op_jump, line);
        const offset = self.code.items.len - loop_start + 2;
        // Negative jump (backward)
        const neg_offset = @as(i32, @intCast(offset));
        try self.write(@intCast((neg_offset >> 8) & 0xFF), line);
        try self.write(@intCast(neg_offset & 0xFF), line);
    }
    
    // Disassemble for debugging
    pub fn disassemble(self: *const Chunk, name: []const u8, writer: anytype) !void {
        try writer.print("=== {s} ===\n", .{name});
        
        var offset: usize = 0;
        while (offset < self.code.items.len) {
            offset = try self.disassembleInstruction(offset, writer);
        }
    }
    
    fn disassembleInstruction(self: *const Chunk, offset: usize, writer: anytype) !usize {
        try writer.print("{d:04}: ", .{offset});
        
        if (offset > 0 and self.lines.items[offset] == self.lines.items[offset - 1]) {
            try writer.writeAll("   | ");
        } else {
            try writer.print("{d:4} ", .{self.lines.items[offset]});
        }
        
        const byte = self.code.items[offset];
        const op: Opcode = @enumFromInt(byte);
        
        return switch (op) {
            .op_nil => simpleInstr("OP_NIL", offset, writer),
            .op_true => simpleInstr("OP_TRUE", offset, writer),
            .op_false => simpleInstr("OP_FALSE", offset, writer),
            .op_const => try self.constInstr("OP_CONST", offset, writer),
            .op_add_i => simpleInstr("OP_ADD_I", offset, writer),
            .op_sub_i => simpleInstr("OP_SUB_I", offset, writer),
            .op_mul_i => simpleInstr("OP_MUL_I", offset, writer),
            .op_div_i => simpleInstr("OP_DIV_I", offset, writer),
            .op_neg_i => simpleInstr("OP_NEG_I", offset, writer),
            .op_eq => simpleInstr("OP_EQ", offset, writer),
            .op_lt_i => simpleInstr("OP_LT_I", offset, writer),
            .op_pop => simpleInstr("OP_POP", offset, writer),
            .op_get_local => try self.byteInstr("OP_GET_LOCAL", offset, writer),
            .op_set_local => try self.byteInstr("OP_SET_LOCAL", offset, writer),
            .op_jump => try self.jumpInstr("OP_JUMP", offset, writer),
            .op_jump_true => try self.jumpInstr("OP_JUMP_TRUE", offset, writer),
            .op_jump_false => try self.jumpInstr("OP_JUMP_FALSE", offset, writer),
            .op_return => simpleInstr("OP_RETURN", offset, writer),
            .op_print => simpleInstr("OP_PRINT", offset, writer),
            .op_halt => simpleInstr("OP_HALT", offset, writer),
            else => blk: {
                try writer.print("Unknown opcode: {d}\n", .{byte});
                break :blk offset + 1;
            },
        };
    }
    
    fn simpleInstr(name: []const u8, offset: usize, writer: anytype) usize {
        writer.print("{s}\n", .{name}) catch {};
        return offset + 1;
    }
    
    fn constInstr(self: *const Chunk, name: []const u8, offset: usize, writer: anytype) !usize {
        const idx = (@as(u16, self.code.items[offset + 1]) << 8) | self.code.items[offset + 2];
        const val = self.constants.items[idx];
        try writer.print("{s} {d} (= {})\n", .{ name, idx, val });
        return offset + 3;
    }
    
    fn byteInstr(_: *const Chunk, name: []const u8, offset: usize, writer: anytype) !usize {
        const slot = _byte;
        _ = slot;
        try writer.print("{s}\n", .{name});
        return offset + 2;
    }
    
    fn jumpInstr(self: *const Chunk, name: []const u8, offset: usize, writer: anytype) !usize {
        const hi = self.code.items[offset + 1];
        const lo = self.code.items[offset + 2];
        const target = offset + 3 + (@as(usize, hi) << 8 | lo);
        try writer.print("{s} -> {d}\n", .{ name, target });
        return offset + 3;
    }
};
```

---

## Step 874: Bytecode Generation

```zig
// bytecode_gen.zig - การ generate bytecode จาก AST

const std = @import("std");

pub const BytecodeGenerator = struct {
    allocator: std.mem.Allocator,
    chunk: Chunk,
    locals: std.ArrayList(Local),
    scope_depth: u32,
    
    pub const Local = struct {
        name: []const u8,
        depth: u32,
        is_captured: bool,
    };
    
    pub fn init(allocator: std.mem.Allocator) BytecodeGenerator {
        return BytecodeGenerator{
            .allocator = allocator,
            .chunk = Chunk.init(allocator),
            .locals = std.ArrayList(Local).init(allocator),
            .scope_depth = 0,
        };
    }
    
    pub fn deinit(self: *BytecodeGenerator) void {
        self.chunk.deinit();
        self.locals.deinit();
    }
    
    pub fn generate(self: *BytecodeGenerator, stmts: []*StmtNode) ![]const u8 {
        for (stmts) |stmt| {
            try self.compileStmt(stmt);
        }
        try self.chunk.writeOpcode(.op_halt, 0);
        return self.chunk.code.toOwnedSlice();
    }
    
    fn compileStmt(self: *BytecodeGenerator, stmt: *const StmtNode) !void {
        switch (stmt.*) {
            .expression_stmt => |expr| {
                try self.compileExpr(expr);
                try self.chunk.writeOpcode(.op_pop, 0);
            },
            .var_decl => |n| {
                if (n.initializer) |init| {
                    try self.compileExpr(init);
                } else {
                    try self.chunk.writeOpcode(.op_nil, 0);
                }
                
                if (self.scope_depth > 0) {
                    // Local variable
                    try self.addLocal(n.name);
                } else {
                    // Global variable
                    const idx = try self.chunk.addConstant(.{ .string = n.name });
                    try self.chunk.writeOpcode(.op_set_global, 0);
                    try self.chunk.write(@intCast(idx >> 8), 0);
                    try self.chunk.write(@intCast(idx & 0xFF), 0);
                }
            },
            .block => |stmts| {
                self.beginScope();
                for (stmts) |s| try self.compileStmt(s);
                try self.endScope();
            },
            .if_stmt => |n| {
                try self.compileExpr(n.condition);
                
                const jump_false = try self.chunk.emitJump(.op_jump_false, 0);
                try self.chunk.writeOpcode(.op_pop, 0);
                
                try self.compileStmt(n.then_branch);
                
                const jump_end = try self.chunk.emitJump(.op_jump, 0);
                self.chunk.patchJump(jump_false);
                try self.chunk.writeOpcode(.op_pop, 0);
                
                if (n.else_branch) |else_s| {
                    try self.compileStmt(else_s);
                }
                self.chunk.patchJump(jump_end);
            },
            .while_stmt => |n| {
                const loop_start = self.chunk.code.items.len;
                
                try self.compileExpr(n.condition);
                const jump_false = try self.chunk.emitJump(.op_jump_false, 0);
                try self.chunk.writeOpcode(.op_pop, 0);
                
                try self.compileStmt(n.body);
                
                // Loop back
                try self.chunk.emitLoop(loop_start, 0);
                
                self.chunk.patchJump(jump_false);
                try self.chunk.writeOpcode(.op_pop, 0);
            },
            .return_stmt => |n| {
                if (n) |val| {
                    try self.compileExpr(val);
                    try self.chunk.writeOpcode(.op_return_val, 0);
                } else {
                    try self.chunk.writeOpcode(.op_nil, 0);
                    try self.chunk.writeOpcode(.op_return_val, 0);
                }
            },
            else => {},
        }
    }
    
    fn compileExpr(self: *BytecodeGenerator, expr: *const ExprNode) !void {
        switch (expr.*) {
            .number => |v| {
                if (v == @floor(v) and v >= -32768 and v <= 32767) {
                    const iv = @as(i64, @intFromFloat(v));
                    try self.chunk.emitConst(.{ .integer = iv }, 0);
                } else {
                    try self.chunk.emitConst(.{ .float = v }, 0);
                }
            },
            .string => |v| {
                try self.chunk.emitConst(.{ .string = v }, 0);
            },
            .boolean => |v| {
                try self.chunk.writeOpcode(if (v) .op_true else .op_false, 0);
            },
            .null_val => {
                try self.chunk.writeOpcode(.op_nil, 0);
            },
            .identifier => |name| {
                // ค้นหา local variable ก่อน
                if (self.resolveLocal(name)) |idx| {
                    try self.chunk.writeOpcode(.op_get_local, 0);
                    try self.chunk.write(@intCast(idx), 0);
                } else {
                    // ใช้ global variable
                    const const_idx = try self.chunk.addConstant(.{ .string = name });
                    try self.chunk.writeOpcode(.op_get_global, 0);
                    try self.chunk.write(@intCast(const_idx >> 8), 0);
                    try self.chunk.write(@intCast(const_idx & 0xFF), 0);
                }
            },
            .binary => |n| {
                // Short-circuit evaluation สำหรับ && และ ||
                if (std.mem.eql(u8, n.op, "&&")) {
                    try self.compileExpr(n.left);
                    const jump = try self.chunk.emitJump(.op_jump_false, 0);
                    try self.chunk.writeOpcode(.op_pop, 0);
                    try self.compileExpr(n.right);
                    self.chunk.patchJump(jump);
                    return;
                }
                if (std.mem.eql(u8, n.op, "||")) {
                    try self.compileExpr(n.left);
                    const jump = try self.chunk.emitJump(.op_jump_true, 0);
                    try self.chunk.writeOpcode(.op_pop, 0);
                    try self.compileExpr(n.right);
                    self.chunk.patchJump(jump);
                    return;
                }
                
                try self.compileExpr(n.left);
                try self.compileExpr(n.right);
                
                const op: Opcode = if (std.mem.eql(u8, n.op, "+")) .op_add_i
                    else if (std.mem.eql(u8, n.op, "-")) .op_sub_i
                    else if (std.mem.eql(u8, n.op, "*")) .op_mul_i
                    else if (std.mem.eql(u8, n.op, "/")) .op_div_i
                    else if (std.mem.eql(u8, n.op, "%")) .op_mod_i
                    else if (std.mem.eql(u8, n.op, "==")) .op_eq
                    else if (std.mem.eql(u8, n.op, "!=")) blk: {
                        // != คือ !(==)
                        try self.chunk.writeOpcode(.op_eq, 0);
                        try self.chunk.writeOpcode(.op_not, 0);
                        return;
                    }
                    else if (std.mem.eql(u8, n.op, "<")) .op_lt_i
                    else if (std.mem.eql(u8, n.op, "<=")) .op_le_i
                    else if (std.mem.eql(u8, n.op, ">")) .op_gt_i
                    else if (std.mem.eql(u8, n.op, ">=")) .op_ge_i
                    else .op_nop;
                
                try self.chunk.writeOpcode(op, 0);
            },
            .unary => |n| {
                try self.compileExpr(n.operand);
                if (std.mem.eql(u8, n.op, "-")) {
                    try self.chunk.writeOpcode(.op_neg_i, 0);
                } else if (std.mem.eql(u8, n.op, "!")) {
                    try self.chunk.writeOpcode(.op_not, 0);
                }
            },
            .call => |n| {
                // Compile arguments (left to right)
                for (n.args) |arg| try self.compileExpr(arg);
                
                // Get function name
                if (n.callee.* == .identifier) {
                    const name = n.callee.identifier;
                    const name_idx = try self.chunk.addConstant(.{ .string = name });
                    try self.chunk.writeOpcode(.op_call, 0);
                    try self.chunk.write(@intCast(name_idx >> 8), 0);
                    try self.chunk.write(@intCast(name_idx & 0xFF), 0);
                    try self.chunk.write(@intCast(n.args.len), 0);
                }
            },
            .assign => |n| {
                try self.compileExpr(n.value);
                if (n.target.* == .identifier) {
                    const name = n.target.identifier;
                    if (self.resolveLocal(name)) |idx| {
                        try self.chunk.writeOpcode(.op_set_local, 0);
                        try self.chunk.write(@intCast(idx), 0);
                    } else {
                        const idx = try self.chunk.addConstant(.{ .string = name });
                        try self.chunk.writeOpcode(.op_set_global, 0);
                        try self.chunk.write(@intCast(idx >> 8), 0);
                        try self.chunk.write(@intCast(idx & 0xFF), 0);
                    }
                }
            },
            else => {},
        }
    }
    
    fn beginScope(self: *BytecodeGenerator) void {
        self.scope_depth += 1;
    }
    
    fn endScope(self: *BytecodeGenerator) !void {
        self.scope_depth -= 1;
        // Pop locals that go out of scope
        while (self.locals.items.len > 0 and
               self.locals.items[self.locals.items.len - 1].depth > self.scope_depth) 
        {
            try self.chunk.writeOpcode(.op_pop, 0);
            _ = self.locals.pop();
        }
    }
    
    fn addLocal(self: *BytecodeGenerator, name: []const u8) !void {
        try self.locals.append(.{
            .name = name,
            .depth = self.scope_depth,
            .is_captured = false,
        });
    }
    
    fn resolveLocal(self: *BytecodeGenerator, name: []const u8) ?usize {
        var i = self.locals.items.len;
        while (i > 0) {
            i -= 1;
            if (std.mem.eql(u8, self.locals.items[i].name, name)) {
                return i;
            }
        }
        return null;
    }
};
```

---

## Step 875: Constant Folding Optimization

```zig
// constant_folding.zig - Constant folding pass

const std = @import("std");

// Constant folding ใน bytecode level
pub const BytecodePeepholeOptimizer = struct {
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) BytecodePeepholeOptimizer {
        return .{ .allocator = allocator };
    }
    
    // Peephole optimization: ดู window ของ instructions และ optimize
    pub fn optimize(self: *BytecodePeepholeOptimizer, chunk: *Chunk) !void {
        _ = self;
        var i: usize = 0;
        while (i < chunk.code.items.len) {
            const op: Opcode = @enumFromInt(chunk.code.items[i]);
            
            // Pattern: CONST_INT x, CONST_INT y, ADD_I => CONST_INT (x+y)
            if (op == .op_const and i + 6 < chunk.code.items.len) {
                const next_op: Opcode = @enumFromInt(chunk.code.items[i + 3]);
                if (next_op == .op_const) {
                    const third_op: Opcode = @enumFromInt(chunk.code.items[i + 6]);
                    if (third_op == .op_add_i or third_op == .op_sub_i or
                        third_op == .op_mul_i or third_op == .op_div_i) 
                    {
                        // Try to evaluate constants
                        const idx1 = (@as(u16, chunk.code.items[i + 1]) << 8) | chunk.code.items[i + 2];
                        const idx2 = (@as(u16, chunk.code.items[i + 4]) << 8) | chunk.code.items[i + 5];
                        
                        const v1 = chunk.constants.items[idx1];
                        const v2 = chunk.constants.items[idx2];
                        
                        if (v1 == .integer and v2 == .integer) {
                            const result: i64 = switch (third_op) {
                                .op_add_i => v1.integer + v2.integer,
                                .op_sub_i => v1.integer - v2.integer,
                                .op_mul_i => v1.integer * v2.integer,
                                .op_div_i => if (v2.integer != 0) @divTrunc(v1.integer, v2.integer) else {
                                    i += 1;
                                    continue;
                                },
                                else => {
                                    i += 1;
                                    continue;
                                },
                            };
                            
                            // Replace 7 bytes with 3 bytes (CONST + 2-byte index)
                            const new_idx = try chunk.addConstant(.{ .integer = result });
                            
                            // NOP out old instructions
                            chunk.code.items[i] = @intFromEnum(Opcode.op_const);
                            chunk.code.items[i + 1] = @intCast(new_idx >> 8);
                            chunk.code.items[i + 2] = @intCast(new_idx & 0xFF);
                            chunk.code.items[i + 3] = @intFromEnum(Opcode.op_nop);
                            chunk.code.items[i + 4] = @intFromEnum(Opcode.op_nop);
                            chunk.code.items[i + 5] = @intFromEnum(Opcode.op_nop);
                            chunk.code.items[i + 6] = @intFromEnum(Opcode.op_nop);
                            
                            i += 7;
                            continue;
                        }
                    }
                }
            }
            
            i += opcodeSize(op);
        }
    }
    
    fn opcodeSize(op: Opcode) usize {
        return switch (op) {
            .op_const => 3,
            .op_get_local, .op_set_local => 2,
            .op_get_global, .op_set_global => 3,
            .op_jump, .op_jump_true, .op_jump_false => 3,
            .op_call => 4,
            else => 1,
        };
    }
};
```

---

## Step 876: Dead Code Elimination

```zig
// dead_code.zig - Dead code elimination

const std = @import("std");

pub const DeadCodeEliminator = struct {
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) DeadCodeEliminator {
        return .{ .allocator = allocator };
    }
    
    // Mark reachable instructions
    pub fn eliminateDeadCode(self: *DeadCodeEliminator, chunk: *Chunk) !void {
        const len = chunk.code.items.len;
        var reachable = try self.allocator.alloc(bool, len);
        defer self.allocator.free(reachable);
        @memset(reachable, false);
        
        // BFS/DFS จาก entry point
        var worklist = std.ArrayList(usize).init(self.allocator);
        defer worklist.deinit();
        
        try worklist.append(0);
        
        while (worklist.items.len > 0) {
            const pc = worklist.pop();
            if (pc >= len or reachable[pc]) continue;
            reachable[pc] = true;
            
            const op: Opcode = @enumFromInt(chunk.code.items[pc]);
            const size = opcodeSize(op);
            
            switch (op) {
                .op_jump => {
                    const offset = (@as(usize, chunk.code.items[pc + 1]) << 8) | chunk.code.items[pc + 2];
                    try worklist.append(pc + 3 + offset);
                },
                .op_jump_true, .op_jump_false => {
                    const offset = (@as(usize, chunk.code.items[pc + 1]) << 8) | chunk.code.items[pc + 2];
                    try worklist.append(pc + 3 + offset); // jump target
                    try worklist.append(pc + 3); // fall through
                },
                .op_return, .op_return_val, .op_halt => {
                    // no successors
                },
                else => {
                    if (pc + size < len) {
                        try worklist.append(pc + size);
                    }
                },
            }
        }
        
        // NOP out unreachable instructions
        var eliminated: u32 = 0;
        var i: usize = 0;
        while (i < len) {
            const op: Opcode = @enumFromInt(chunk.code.items[i]);
            const size = opcodeSize(op);
            
            if (!reachable[i]) {
                for (0..size) |j| {
                    chunk.code.items[i + j] = @intFromEnum(Opcode.op_nop);
                }
                eliminated += 1;
            }
            
            i += size;
        }
        
        std.debug.print("Eliminated {} dead instructions\n", .{eliminated});
    }
    
    fn opcodeSize(op: Opcode) usize {
        return switch (op) {
            .op_const => 3,
            .op_get_local, .op_set_local => 2,
            .op_get_global, .op_set_global => 3,
            .op_jump, .op_jump_true, .op_jump_false => 3,
            .op_call => 4,
            else => 1,
        };
    }
};
```

---

## Step 877: Register Allocation Basics

```zig
// register_alloc.zig - Register allocation concepts

const std = @import("std");

// สำหรับ register-based VM หรือ native code generation
// เราต้องจัดสรร registers ให้กับ virtual registers

pub const RegAllocator = struct {
    // Physical registers available
    num_regs: u8,
    // ตำแหน่งปัจจุบัน: virtual reg -> physical reg
    reg_map: std.AutoHashMap(u32, u8),
    // Physical regs ที่ถูกใช้
    used_regs: std.bit_set.DynamicBitSet,
    // Stack ของ spilled registers (ที่ overflow ไปยัง memory)
    spill_slots: std.ArrayList(u32),
    
    pub fn init(allocator: std.mem.Allocator, num_regs: u8) !RegAllocator {
        return RegAllocator{
            .num_regs = num_regs,
            .reg_map = std.AutoHashMap(u32, u8).init(allocator),
            .used_regs = try std.bit_set.DynamicBitSet.initEmpty(allocator, num_regs),
            .spill_slots = std.ArrayList(u32).init(allocator),
        };
    }
    
    pub fn deinit(self: *RegAllocator) void {
        self.reg_map.deinit();
        self.used_regs.deinit();
        self.spill_slots.deinit();
    }
    
    // Allocate physical register สำหรับ virtual register
    pub fn allocReg(self: *RegAllocator, vreg: u32) !u8 {
        // ถ้ามีอยู่แล้ว return เดิม
        if (self.reg_map.get(vreg)) |preg| return preg;
        
        // หา free physical register
        var i: u8 = 0;
        while (i < self.num_regs) : (i += 1) {
            if (!self.used_regs.isSet(i)) {
                self.used_regs.set(i);
                try self.reg_map.put(vreg, i);
                return i;
            }
        }
        
        // ไม่มี register ว่าง - ต้อง spill
        return error.NoFreeRegisters;
    }
    
    // Free physical register
    pub fn freeReg(self: *RegAllocator, vreg: u32) void {
        if (self.reg_map.get(vreg)) |preg| {
            self.used_regs.unset(preg);
            _ = self.reg_map.remove(vreg);
        }
    }
    
    // Linear scan register allocation
    pub fn linearScan(
        self: *RegAllocator,
        live_intervals: []const LiveInterval,
    ) !void {
        // Sort intervals by start point
        var sorted = try std.ArrayList(LiveInterval).initCapacity(
            self.reg_map.allocator,
            live_intervals.len,
        );
        defer sorted.deinit();
        
        for (live_intervals) |interval| {
            try sorted.append(interval);
        }
        
        std.sort.block(LiveInterval, sorted.items, {}, LiveInterval.compareStart);
        
        var active = std.ArrayList(LiveInterval).init(self.reg_map.allocator);
        defer active.deinit();
        
        for (sorted.items) |interval| {
            // Expire old intervals
            var j: usize = 0;
            while (j < active.items.len) {
                if (active.items[j].end < interval.start) {
                    self.freeReg(active.items[j].vreg);
                    _ = active.orderedRemove(j);
                } else {
                    j += 1;
                }
            }
            
            // Allocate register
            _ = self.allocReg(interval.vreg) catch {
                // Spill the interval with farthest end point
                if (active.items.len > 0) {
                    var spill_idx: usize = 0;
                    for (active.items, 0..) |a, k| {
                        if (a.end > active.items[spill_idx].end) {
                            spill_idx = k;
                        }
                    }
                    
                    const spill = active.items[spill_idx];
                    if (spill.end > interval.end) {
                        // Spill the one with farther end
                        const preg = self.reg_map.get(spill.vreg).?;
                        try self.reg_map.put(interval.vreg, preg);
                        _ = self.reg_map.remove(spill.vreg);
                        _ = active.orderedRemove(spill_idx);
                        try active.append(interval);
                        try self.spill_slots.append(spill.vreg);
                    } else {
                        try self.spill_slots.append(interval.vreg);
                    }
                }
            };
            
            try active.append(interval);
        }
    }
};

pub const LiveInterval = struct {
    vreg: u32,
    start: usize,
    end: usize,
    
    fn compareStart(_: void, a: LiveInterval, b: LiveInterval) bool {
        return a.start < b.start;
    }
};
```

---

## Step 878: LLVM IR Generation Concepts

```zig
// llvm_ir_gen.zig - การสร้าง LLVM IR (conceptual)
// ในทางปฏิบัติจะใช้ LLVM C API หรือ Zig LLVM bindings

const std = @import("std");

// LLVM IR Types
pub const LlvmType = union(enum) {
    void,
    i1, i8, i16, i32, i64,
    f32, f64,
    ptr: *LlvmType,
    array: struct { element: *LlvmType, size: u64 },
    function: struct { ret: *LlvmType, params: []*LlvmType },
    
    pub fn format(
        self: LlvmType,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        switch (self) {
            .void => try writer.writeAll("void"),
            .i1 => try writer.writeAll("i1"),
            .i8 => try writer.writeAll("i8"),
            .i16 => try writer.writeAll("i16"),
            .i32 => try writer.writeAll("i32"),
            .i64 => try writer.writeAll("i64"),
            .f32 => try writer.writeAll("float"),
            .f64 => try writer.writeAll("double"),
            .ptr => |t| try writer.print("ptr", .{}), // LLVM 15+ uses opaque pointer
            else => try writer.writeAll("..."),
        }
    }
};

// LLVM IR Value reference
pub const LlvmValue = union(enum) {
    local: struct { name: []const u8, ty: LlvmType },
    global: struct { name: []const u8, ty: LlvmType },
    const_int: struct { ty: LlvmType, value: i64 },
    const_float: struct { ty: LlvmType, value: f64 },
    const_bool: bool,
    const_null: LlvmType,
};

// LLVM IR Instruction
pub const LlvmInstr = union(enum) {
    // Arithmetic
    add: struct { result: []const u8, lhs: LlvmValue, rhs: LlvmValue, ty: LlvmType },
    sub: struct { result: []const u8, lhs: LlvmValue, rhs: LlvmValue, ty: LlvmType },
    mul: struct { result: []const u8, lhs: LlvmValue, rhs: LlvmValue, ty: LlvmType },
    sdiv: struct { result: []const u8, lhs: LlvmValue, rhs: LlvmValue, ty: LlvmType },
    
    // Memory
    alloca: struct { result: []const u8, ty: LlvmType },
    load: struct { result: []const u8, ptr: LlvmValue, ty: LlvmType },
    store: struct { value: LlvmValue, ptr: LlvmValue, ty: LlvmType },
    
    // Comparison
    icmp: struct { result: []const u8, pred: []const u8, lhs: LlvmValue, rhs: LlvmValue },
    fcmp: struct { result: []const u8, pred: []const u8, lhs: LlvmValue, rhs: LlvmValue },
    
    // Control flow
    br_uncond: struct { target: []const u8 },
    br_cond: struct { cond: LlvmValue, true_bb: []const u8, false_bb: []const u8 },
    
    // Function
    call: struct {
        result: ?[]const u8,
        func: []const u8,
        args: []LlvmValue,
        ret_ty: LlvmType,
    },
    ret: struct { value: ?LlvmValue },
    
    pub fn print(self: LlvmInstr, writer: anytype) !void {
        switch (self) {
            .add => |i| try writer.print("  %{s} = add {any} %{s}, %{s}\n",
                .{i.result, i.ty, "lhs", "rhs"}),
            .alloca => |i| try writer.print("  %{s} = alloca {any}\n",
                .{i.result, i.ty}),
            .load => |i| try writer.print("  %{s} = load {any}, ptr %{s}\n",
                .{i.result, i.ty, "ptr"}),
            .store => |i| try writer.print("  store {any} %{s}, ptr %{s}\n",
                .{i.ty, "val", "ptr"}),
            .ret => |i| {
                if (i.value != null) {
                    try writer.print("  ret ...\n", .{});
                } else {
                    try writer.writeAll("  ret void\n");
                }
            },
            .br_uncond => |i| try writer.print("  br label %{s}\n", .{i.target}),
            .br_cond => |i| try writer.print("  br i1 %cond, label %{s}, label %{s}\n",
                .{i.true_bb, i.false_bb}),
            else => try writer.writeAll("  ...\n"),
        }
    }
};

// ตัวอย่าง LLVM IR output
pub fn exampleLlvmIr(writer: anytype) !void {
    try writer.writeAll(
        \\; LLVM IR example for factorial
        \\define i64 @factorial(i64 %n) {
        \\entry:
        \\  %result = alloca i64
        \\  %cond = icmp sle i64 %n, 1
        \\  br i1 %cond, label %base_case, label %recursive
        \\
        \\base_case:
        \\  store i64 1, ptr %result
        \\  br label %exit
        \\
        \\recursive:
        \\  %n_minus_1 = sub i64 %n, 1
        \\  %rec_result = call i64 @factorial(i64 %n_minus_1)
        \\  %product = mul i64 %n, %rec_result
        \\  store i64 %product, ptr %result
        \\  br label %exit
        \\
        \\exit:
        \\  %ret_val = load i64, ptr %result
        \\  ret i64 %ret_val
        \\}
    );
}
```

---

## Step 879: Simple Bytecode Interpreter

```zig
// vm.zig - Stack-based VM interpreter

const std = @import("std");

pub const Value = union(enum) {
    nil: void,
    boolean: bool,
    integer: i64,
    float: f64,
    string: []const u8,
    
    pub fn format(
        self: Value,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        switch (self) {
            .nil => try writer.writeAll("nil"),
            .boolean => |b| try writer.print("{}", .{b}),
            .integer => |i| try writer.print("{d}", .{i}),
            .float => |f| try writer.print("{d}", .{f}),
            .string => |s| try writer.print("\"{s}\"", .{s}),
        }
    }
    
    pub fn isTruthy(self: Value) bool {
        return switch (self) {
            .nil => false,
            .boolean => |b| b,
            .integer => |i| i != 0,
            .float => |f| f != 0.0,
            .string => |s| s.len > 0,
        };
    }
};

pub const VmError = error{
    StackOverflow,
    StackUnderflow,
    DivisionByZero,
    TypeMismatch,
    UndefinedVariable,
    UndefinedFunction,
    InvalidOpcode,
};

pub const CallFrame = struct {
    chunk: *const Chunk,
    ip: usize,        // instruction pointer
    stack_base: usize, // base ของ stack สำหรับ local vars
};

pub const Vm = struct {
    allocator: std.mem.Allocator,
    stack: [256]Value,
    stack_top: usize,
    globals: std.StringHashMap(Value),
    functions: std.StringHashMap(*Chunk),
    call_frames: [64]CallFrame,
    frame_count: usize,
    
    pub fn init(allocator: std.mem.Allocator) Vm {
        return Vm{
            .allocator = allocator,
            .stack = undefined,
            .stack_top = 0,
            .globals = std.StringHashMap(Value).init(allocator),
            .functions = std.StringHashMap(*Chunk).init(allocator),
            .call_frames = undefined,
            .frame_count = 0,
        };
    }
    
    pub fn deinit(self: *Vm) void {
        self.globals.deinit();
        self.functions.deinit();
    }
    
    fn push(self: *Vm, value: Value) VmError!void {
        if (self.stack_top >= self.stack.len) return VmError.StackOverflow;
        self.stack[self.stack_top] = value;
        self.stack_top += 1;
    }
    
    fn pop(self: *Vm) VmError!Value {
        if (self.stack_top == 0) return VmError.StackUnderflow;
        self.stack_top -= 1;
        return self.stack[self.stack_top];
    }
    
    fn peek(self: *Vm, distance: usize) Value {
        return self.stack[self.stack_top - 1 - distance];
    }
    
    fn currentFrame(self: *Vm) *CallFrame {
        return &self.call_frames[self.frame_count - 1];
    }
    
    fn readByte(self: *Vm) u8 {
        const frame = self.currentFrame();
        const byte = frame.chunk.code.items[frame.ip];
        frame.ip += 1;
        return byte;
    }
    
    fn readShort(self: *Vm) u16 {
        const hi = self.readByte();
        const lo = self.readByte();
        return (@as(u16, hi) << 8) | lo;
    }
    
    fn readConst(self: *Vm) Value {
        const idx = self.readShort();
        return self.currentFrame().chunk.constants.items[idx];
    }
    
    pub fn run(self: *Vm, chunk: *const Chunk) !Value {
        // Push initial call frame
        self.call_frames[0] = CallFrame{
            .chunk = chunk,
            .ip = 0,
            .stack_base = 0,
        };
        self.frame_count = 1;
        
        while (true) {
            const byte = self.readByte();
            const op: Opcode = @enumFromInt(byte);
            
            switch (op) {
                .op_nil => try self.push(.{ .nil = {} }),
                .op_true => try self.push(.{ .boolean = true }),
                .op_false => try self.push(.{ .boolean = false }),
                
                .op_const => {
                    const val = self.readConst();
                    try self.push(val);
                },
                
                .op_add_i => {
                    const b = try self.pop();
                    const a = try self.pop();
                    if (a == .integer and b == .integer) {
                        try self.push(.{ .integer = a.integer + b.integer });
                    } else if (a == .float or b == .float) {
                        const af: f64 = if (a == .float) a.float else @floatFromInt(a.integer);
                        const bf: f64 = if (b == .float) b.float else @floatFromInt(b.integer);
                        try self.push(.{ .float = af + bf });
                    } else {
                        return VmError.TypeMismatch;
                    }
                },
                
                .op_sub_i => {
                    const b = try self.pop();
                    const a = try self.pop();
                    if (a == .integer and b == .integer) {
                        try self.push(.{ .integer = a.integer - b.integer });
                    } else return VmError.TypeMismatch;
                },
                
                .op_mul_i => {
                    const b = try self.pop();
                    const a = try self.pop();
                    if (a == .integer and b == .integer) {
                        try self.push(.{ .integer = a.integer * b.integer });
                    } else return VmError.TypeMismatch;
                },
                
                .op_div_i => {
                    const b = try self.pop();
                    const a = try self.pop();
                    if (a == .integer and b == .integer) {
                        if (b.integer == 0) return VmError.DivisionByZero;
                        try self.push(.{ .integer = @divTrunc(a.integer, b.integer) });
                    } else return VmError.TypeMismatch;
                },
                
                .op_neg_i => {
                    const a = try self.pop();
                    if (a == .integer) {
                        try self.push(.{ .integer = -a.integer });
                    } else return VmError.TypeMismatch;
                },
                
                .op_eq => {
                    const b = try self.pop();
                    const a = try self.pop();
                    const result = switch (a) {
                        .integer => |ai| if (b == .integer) ai == b.integer else false,
                        .float => |af| if (b == .float) af == b.float else false,
                        .boolean => |ab| if (b == .boolean) ab == b.boolean else false,
                        .string => |s| if (b == .string) std.mem.eql(u8, s, b.string) else false,
                        .nil => b == .nil,
                    };
                    try self.push(.{ .boolean = result });
                },
                
                .op_lt_i => {
                    const b = try self.pop();
                    const a = try self.pop();
                    if (a == .integer and b == .integer) {
                        try self.push(.{ .boolean = a.integer < b.integer });
                    } else return VmError.TypeMismatch;
                },
                
                .op_not => {
                    const a = try self.pop();
                    try self.push(.{ .boolean = !a.isTruthy() });
                },
                
                .op_pop => _ = try self.pop(),
                
                .op_get_local => {
                    const slot = self.readByte();
                    const frame = self.currentFrame();
                    try self.push(self.stack[frame.stack_base + slot]);
                },
                
                .op_set_local => {
                    const slot = self.readByte();
                    const frame = self.currentFrame();
                    self.stack[frame.stack_base + slot] = self.peek(0);
                },
                
                .op_get_global => {
                    const idx = self.readShort();
                    const name = self.currentFrame().chunk.constants.items[idx].string;
                    const val = self.globals.get(name) orelse {
                        std.debug.print("Undefined variable: {s}\n", .{name});
                        return VmError.UndefinedVariable;
                    };
                    try self.push(val);
                },
                
                .op_set_global => {
                    const idx = self.readShort();
                    const name = self.currentFrame().chunk.constants.items[idx].string;
                    try self.globals.put(name, self.peek(0));
                },
                
                .op_jump => {
                    const offset = self.readShort();
                    self.currentFrame().ip += offset;
                },
                
                .op_jump_false => {
                    const offset = self.readShort();
                    if (!self.peek(0).isTruthy()) {
                        self.currentFrame().ip += offset;
                    }
                },
                
                .op_jump_true => {
                    const offset = self.readShort();
                    if (self.peek(0).isTruthy()) {
                        self.currentFrame().ip += offset;
                    }
                },
                
                .op_print => {
                    const val = try self.pop();
                    const stdout = std.io.getStdOut().writer();
                    try stdout.print("{any}\n", .{val});
                },
                
                .op_return_val => {
                    const result = try self.pop();
                    self.frame_count -= 1;
                    if (self.frame_count == 0) return result;
                    
                    // Restore stack
                    const old_frame = self.call_frames[self.frame_count];
                    self.stack_top = old_frame.stack_base;
                    try self.push(result);
                },
                
                .op_halt => {
                    return if (self.stack_top > 0) self.peek(0) else Value{ .nil = {} };
                },
                
                .op_nop => {}, // do nothing
                
                else => return VmError.InvalidOpcode,
            }
        }
    }
};
```

---

## Step 880: Complete Mini-Compiler

```zig
// mini_compiler.zig - Mini compiler ที่สมบูรณ์

const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Source code
    const source =
        \\var x = 10;
        \\var y = 20;
        \\var sum = x + y;
        \\var i = 0;
        \\while (i < 5) {
        \\    sum = sum + i;
        \\    i = i + 1;
        \\}
        \\print(sum);
    ;
    
    std.debug.print("=== Mini Compiler Demo ===\n\n", .{});
    std.debug.print("Source:\n{s}\n\n", .{source});
    
    // Step 1: Lex
    std.debug.print("Step 1: Lexing...\n", .{});
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    std.debug.print("  Got {} tokens\n", .{tokens.len});
    
    // Step 2: Parse
    std.debug.print("Step 2: Parsing...\n", .{});
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    if (parser.errors.items.len > 0) {
        std.debug.print("Parse errors:\n", .{});
        for (parser.errors.items) |err| {
            std.debug.print("  {s}\n", .{err.message});
        }
        return;
    }
    std.debug.print("  Got {} statements\n", .{stmts.len});
    
    // Step 3: Generate bytecode
    std.debug.print("Step 3: Generating bytecode...\n", .{});
    var codegen = BytecodeGenerator.init(allocator);
    defer codegen.deinit();
    
    for (stmts) |stmt| {
        try codegen.compileStmt(stmt);
    }
    try codegen.chunk.writeOpcode(.op_halt, 0);
    
    // Step 4: Disassemble
    std.debug.print("\nBytecode:\n", .{});
    const stdout = std.io.getStdOut().writer();
    try codegen.chunk.disassemble("main", stdout);
    
    // Step 5: Execute
    std.debug.print("\nExecution:\n", .{});
    var vm = Vm.init(allocator);
    defer vm.deinit();
    
    const result = try vm.run(&codegen.chunk);
    std.debug.print("Result: {any}\n", .{result});
}

test "mini compiler end-to-end" {
    const allocator = std.testing.allocator;
    
    const source = "var x = 1 + 2 * 3;";
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    try std.testing.expectEqual(@as(usize, 0), parser.errors.items.len);
    
    var codegen = BytecodeGenerator.init(allocator);
    defer codegen.deinit();
    
    for (stmts) |stmt| {
        try codegen.compileStmt(stmt);
    }
    try codegen.chunk.writeOpcode(.op_halt, 0);
    
    var vm = Vm.init(allocator);
    defer vm.deinit();
    
    _ = try vm.run(&codegen.chunk);
    
    // ตรวจสอบว่า x มีค่า 7 (1 + 2*3 = 7)
    const x_val = vm.globals.get("x");
    try std.testing.expect(x_val != null);
    try std.testing.expectEqual(@as(i64, 7), x_val.?.integer);
}
```

---

## สรุป Part 88

ในส่วนนี้เราได้เรียนรู้:

1. **Step 871**: Compiler architecture - pipeline ตั้งแต่ lexing ถึง code generation
2. **Step 872**: Code generation concepts - IR และ bytecode format
3. **Step 873**: Stack-based VM bytecode - opcode definitions และ chunk format
4. **Step 874**: Bytecode generation - compile AST เป็น bytecode
5. **Step 875**: Constant folding - peephole optimization ใน bytecode level
6. **Step 876**: Dead code elimination - mark and sweep unreachable code
7. **Step 877**: Register allocation - linear scan algorithm
8. **Step 878**: LLVM IR generation - concepts และ IR format
9. **Step 879**: Simple bytecode interpreter - stack-based VM execution
10. **Step 880**: Complete mini-compiler - รวมทุก phase เข้าด้วยกัน

การสร้าง compiler ต้องการความเข้าใจในหลายด้าน ตั้งแต่ language theory ไปจนถึง computer architecture แต่ด้วย Zig เราสามารถสร้าง compiler ที่มีประสิทธิภาพและปลอดภัยได้
