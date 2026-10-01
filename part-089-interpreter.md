# ส่วนที่ 89: Interpreter Implementation

## ขั้นตอนที่ 881-890: การสร้าง Interpreter สมบูรณ์ด้วย Zig

Interpreter คือโปรแกรมที่ execute source code โดยตรง (หรือผ่าน AST) โดยไม่ต้องแปลงเป็น machine code ก่อน ในส่วนนี้เราจะสร้าง interpreter ที่สมบูรณ์พร้อม closures, garbage collection, และ REPL

---

## Step 881: Interpreter Architecture

### โครงสร้างของ Tree-walking Interpreter

```zig
// interpreter_arch.zig - โครงสร้างหลักของ Interpreter

const std = @import("std");

// Value types ที่ interpreter จัดการ
pub const ValueType = enum {
    nil,
    boolean,
    integer,
    float,
    string,
    list,
    map,
    function,
    builtin_fn,
    class_instance,
    class_def,
};

// Value representation (tagged union)
pub const Value = union(ValueType) {
    nil: void,
    boolean: bool,
    integer: i64,
    float: f64,
    string: *StringObj,
    list: *ListObj,
    map: *MapObj,
    function: *FunctionObj,
    builtin_fn: *BuiltinFnObj,
    class_instance: *InstanceObj,
    class_def: *ClassObj,
    
    pub fn isTruthy(self: Value) bool {
        return switch (self) {
            .nil => false,
            .boolean => |b| b,
            .integer => |i| i != 0,
            .float => |f| f != 0.0,
            .string => |s| s.data.len > 0,
            else => true,
        };
    }
    
    pub fn typeName(self: Value) []const u8 {
        return switch (self) {
            .nil => "nil",
            .boolean => "boolean",
            .integer => "integer",
            .float => "float",
            .string => "string",
            .list => "list",
            .map => "map",
            .function => "function",
            .builtin_fn => "builtin_function",
            .class_instance => "instance",
            .class_def => "class",
        };
    }
    
    pub fn equals(a: Value, b: Value) bool {
        return switch (a) {
            .nil => b == .nil,
            .boolean => |av| b == .boolean and av == b.boolean,
            .integer => |av| switch (b) {
                .integer => |bv| av == bv,
                .float => |bv| @as(f64, @floatFromInt(av)) == bv,
                else => false,
            },
            .float => |av| switch (b) {
                .float => |bv| av == bv,
                .integer => |bv| av == @as(f64, @floatFromInt(bv)),
                else => false,
            },
            .string => |av| b == .string and std.mem.eql(u8, av.data, b.string.data),
            else => false,
        };
    }
};

// Heap-allocated objects
pub const StringObj = struct {
    data: []const u8,
    hash: u64,
    is_interned: bool,
};

pub const ListObj = struct {
    items: std.ArrayList(Value),
};

pub const MapObj = struct {
    entries: std.StringHashMap(Value),
};

pub const FunctionObj = struct {
    name: ?[]const u8,
    params: [][]const u8,
    body: *AstNode,
    closure: *Environment,
    arity: u8,
};

pub const BuiltinFnObj = struct {
    name: []const u8,
    func: *const fn (args: []Value, allocator: std.mem.Allocator) InterpreterError!Value,
    arity: i8, // -1 = variadic
};

pub const ClassObj = struct {
    name: []const u8,
    methods: std.StringHashMap(*FunctionObj),
    superclass: ?*ClassObj,
};

pub const InstanceObj = struct {
    class: *ClassObj,
    fields: std.StringHashMap(Value),
};

// Error types
pub const InterpreterError = error{
    UndefinedVariable,
    NotCallable,
    WrongArity,
    TypeError,
    IndexOutOfBounds,
    UndefinedField,
    DivisionByZero,
    StackOverflow,
    RuntimeError,
    OutOfMemory,
    Return, // ใช้สำหรับ return statement
    Break,
    Continue,
};

// Return value (ใช้ error system เพื่อ unwind call stack)
pub const ReturnValue = struct {
    value: Value,
};
```

---

## Step 882: Environment/Scope Handling

```zig
// environment.zig - การจัดการ environment และ scope

const std = @import("std");

pub const Environment = struct {
    values: std.StringHashMap(Value),
    enclosing: ?*Environment, // parent scope
    allocator: std.mem.Allocator,
    ref_count: u32, // สำหรับ garbage collection
    
    pub fn init(allocator: std.mem.Allocator, enclosing: ?*Environment) !*Environment {
        const env = try allocator.create(Environment);
        env.* = Environment{
            .values = std.StringHashMap(Value).init(allocator),
            .enclosing = enclosing,
            .allocator = allocator,
            .ref_count = 1,
        };
        return env;
    }
    
    pub fn deinit(self: *Environment) void {
        self.values.deinit();
        self.allocator.destroy(self);
    }
    
    // Define ตัวแปรใหม่ใน scope ปัจจุบัน
    pub fn define(self: *Environment, name: []const u8, value: Value) !void {
        try self.values.put(name, value);
    }
    
    // ค้นหาตัวแปรใน scope chain
    pub fn get(self: *Environment, name: []const u8) ?Value {
        if (self.values.get(name)) |v| return v;
        if (self.enclosing) |enc| return enc.get(name);
        return null;
    }
    
    // อัพเดตตัวแปรที่มีอยู่แล้ว
    pub fn set(self: *Environment, name: []const u8, value: Value) bool {
        if (self.values.contains(name)) {
            self.values.put(name, value) catch return false;
            return true;
        }
        if (self.enclosing) |enc| return enc.set(name, value);
        return false;
    }
    
    // Get ที่ depth ที่กำหนด (สำหรับ closure)
    pub fn getAt(self: *Environment, depth: u32, name: []const u8) ?Value {
        var env: *Environment = self;
        var d: u32 = 0;
        while (d < depth) : (d += 1) {
            env = env.enclosing orelse return null;
        }
        return env.values.get(name);
    }
    
    // Set ที่ depth ที่กำหนด
    pub fn setAt(self: *Environment, depth: u32, name: []const u8, value: Value) bool {
        var env: *Environment = self;
        var d: u32 = 0;
        while (d < depth) : (d += 1) {
            env = env.enclosing orelse return false;
        }
        env.values.put(name, value) catch return false;
        return true;
    }
    
    // สร้าง child scope
    pub fn createChild(self: *Environment) !*Environment {
        return Environment.init(self.allocator, self);
    }
};

// Scope guard สำหรับ automatic scope management
pub const ScopeGuard = struct {
    interpreter: *Interpreter,
    previous: *Environment,
    
    pub fn init(interp: *Interpreter, new_env: *Environment) ScopeGuard {
        const guard = ScopeGuard{
            .interpreter = interp,
            .previous = interp.environment,
        };
        interp.environment = new_env;
        return guard;
    }
    
    pub fn deinit(self: *ScopeGuard) void {
        // ลบ env ปัจจุบันถ้าไม่มีใครอ้างถึง
        const current = self.interpreter.environment;
        self.interpreter.environment = self.previous;
        if (current.ref_count == 0) {
            current.deinit();
        }
    }
};
```

---

## Step 883: Function Call Implementation

```zig
// function_calls.zig - การ implement function calls

const std = @import("std");

pub const Interpreter = struct {
    allocator: std.mem.Allocator,
    environment: *Environment,
    globals: *Environment,
    call_stack: std.ArrayList(CallInfo),
    return_value: ?Value,
    
    pub const CallInfo = struct {
        function_name: []const u8,
        depth: u32,
    };
    
    const MAX_CALL_DEPTH = 1000;
    
    pub fn init(allocator: std.mem.Allocator) !Interpreter {
        const globals = try Environment.init(allocator, null);
        return Interpreter{
            .allocator = allocator,
            .environment = globals,
            .globals = globals,
            .call_stack = std.ArrayList(CallInfo).init(allocator),
            .return_value = null,
        };
    }
    
    pub fn deinit(self: *Interpreter) void {
        self.call_stack.deinit();
        self.globals.deinit();
    }
    
    // Call a function
    pub fn callFunction(
        self: *Interpreter,
        callee: Value,
        args: []Value,
    ) InterpreterError!Value {
        // Check call depth
        if (self.call_stack.items.len >= MAX_CALL_DEPTH) {
            return InterpreterError.StackOverflow;
        }
        
        switch (callee) {
            .function => |func| {
                return self.callUserFunction(func, args);
            },
            .builtin_fn => |builtin| {
                return self.callBuiltin(builtin, args);
            },
            .class_def => |class| {
                return self.instantiateClass(class, args);
            },
            else => return InterpreterError.NotCallable,
        }
    }
    
    fn callUserFunction(
        self: *Interpreter,
        func: *FunctionObj,
        args: []Value,
    ) InterpreterError!Value {
        // Arity check
        if (args.len != func.params.len) {
            std.debug.print("Expected {} args, got {}\n", .{ func.params.len, args.len });
            return InterpreterError.WrongArity;
        }
        
        // สร้าง environment ใหม่ที่ใช้ closure environment เป็น parent
        const fn_env = try Environment.init(self.allocator, func.closure);
        defer fn_env.deinit();
        
        // Bind parameters
        for (func.params, args) |param, arg| {
            try fn_env.define(param, arg);
        }
        
        // Push call info
        try self.call_stack.append(.{
            .function_name = func.name orelse "<anonymous>",
            .depth = @intCast(self.call_stack.items.len),
        });
        defer _ = self.call_stack.pop();
        
        // Execute function body ใน new environment
        const saved_env = self.environment;
        self.environment = fn_env;
        defer self.environment = saved_env;
        
        // Execute body
        self.executeStatement(func.body) catch |err| {
            if (err == InterpreterError.Return) {
                return self.return_value orelse Value{ .nil = {} };
            }
            return err;
        };
        
        return Value{ .nil = {} };
    }
    
    fn callBuiltin(
        self: *Interpreter,
        builtin: *BuiltinFnObj,
        args: []Value,
    ) InterpreterError!Value {
        // Check arity (ยกเว้น variadic functions)
        if (builtin.arity >= 0 and args.len != @as(usize, @intCast(builtin.arity))) {
            return InterpreterError.WrongArity;
        }
        return builtin.func(args, self.allocator);
    }
    
    fn instantiateClass(
        self: *Interpreter,
        class: *ClassObj,
        args: []Value,
    ) InterpreterError!Value {
        const instance = try self.allocator.create(InstanceObj);
        instance.* = InstanceObj{
            .class = class,
            .fields = std.StringHashMap(Value).init(self.allocator),
        };
        
        // Call constructor if exists
        if (class.methods.get("init")) |init_method| {
            const instance_val = Value{ .class_instance = instance };
            var new_args = try self.allocator.alloc(Value, args.len + 1);
            defer self.allocator.free(new_args);
            new_args[0] = instance_val;
            @memcpy(new_args[1..], args);
            _ = try self.callUserFunction(init_method, new_args);
        }
        
        return Value{ .class_instance = instance };
    }
    
    // Evaluate expression
    pub fn evaluateExpr(self: *Interpreter, expr: *const ExprNode) InterpreterError!Value {
        switch (expr.*) {
            .number => |v| {
                if (v == @floor(v)) {
                    return Value{ .integer = @intFromFloat(v) };
                }
                return Value{ .float = v };
            },
            .string => |v| {
                const s = try self.allocator.create(StringObj);
                s.* = .{ .data = v, .hash = hashString(v), .is_interned = false };
                return Value{ .string = s };
            },
            .boolean => |v| return Value{ .boolean = v },
            .null_val => return Value{ .nil = {} },
            
            .identifier => |name| {
                const val = self.environment.get(name) orelse {
                    std.debug.print("Undefined variable: {s}\n", .{name});
                    return InterpreterError.UndefinedVariable;
                };
                return val;
            },
            
            .binary => |n| {
                // Short-circuit evaluation
                if (std.mem.eql(u8, n.op, "&&")) {
                    const left = try self.evaluateExpr(n.left);
                    if (!left.isTruthy()) return left;
                    return self.evaluateExpr(n.right);
                }
                if (std.mem.eql(u8, n.op, "||")) {
                    const left = try self.evaluateExpr(n.left);
                    if (left.isTruthy()) return left;
                    return self.evaluateExpr(n.right);
                }
                
                const left = try self.evaluateExpr(n.left);
                const right = try self.evaluateExpr(n.right);
                return self.evalBinaryOp(n.op, left, right);
            },
            
            .unary => |n| {
                const operand = try self.evaluateExpr(n.operand);
                return self.evalUnaryOp(n.op, operand);
            },
            
            .call => |n| {
                const callee = try self.evaluateExpr(n.callee);
                var args = try self.allocator.alloc(Value, n.args.len);
                defer self.allocator.free(args);
                for (n.args, 0..) |arg, i| {
                    args[i] = try self.evaluateExpr(arg);
                }
                return self.callFunction(callee, args);
            },
            
            .assign => |n| {
                const value = try self.evaluateExpr(n.value);
                if (n.target.* == .identifier) {
                    const name = n.target.identifier;
                    if (!self.environment.set(name, value)) {
                        return InterpreterError.UndefinedVariable;
                    }
                }
                return value;
            },
            
            .field_access => |n| {
                const obj = try self.evaluateExpr(n.object);
                return self.getField(obj, n.field);
            },
            
            .index => |n| {
                const obj = try self.evaluateExpr(n.object);
                const idx = try self.evaluateExpr(n.idx);
                return self.getIndex(obj, idx);
            },
            
            else => return Value{ .nil = {} },
        }
    }
    
    fn evalBinaryOp(self: *Interpreter, op: []const u8, left: Value, right: Value) InterpreterError!Value {
        _ = self;
        if (std.mem.eql(u8, op, "+")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .integer = lv + rv },
                    .float => |rv| Value{ .float = @as(f64, @floatFromInt(lv)) + rv },
                    else => InterpreterError.TypeError,
                },
                .float => |lv| switch (right) {
                    .float => |rv| Value{ .float = lv + rv },
                    .integer => |rv| Value{ .float = lv + @as(f64, @floatFromInt(rv)) },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "-")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .integer = lv - rv },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "*")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .integer = lv * rv },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "/")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| if (rv == 0) InterpreterError.DivisionByZero 
                                    else Value{ .integer = @divTrunc(lv, rv) },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "==")) return Value{ .boolean = Value.equals(left, right) };
        if (std.mem.eql(u8, op, "!=")) return Value{ .boolean = !Value.equals(left, right) };
        if (std.mem.eql(u8, op, "<")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .boolean = lv < rv },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "<=")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .boolean = lv <= rv },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, ">")) {
            return switch (left) {
                .integer => |lv| switch (right) {
                    .integer => |rv| Value{ .boolean = lv > rv },
                    else => InterpreterError.TypeError,
                },
                else => InterpreterError.TypeError,
            };
        }
        return Value{ .nil = {} };
    }
    
    fn evalUnaryOp(_: *Interpreter, op: []const u8, operand: Value) InterpreterError!Value {
        if (std.mem.eql(u8, op, "-")) {
            return switch (operand) {
                .integer => |v| Value{ .integer = -v },
                .float => |v| Value{ .float = -v },
                else => InterpreterError.TypeError,
            };
        }
        if (std.mem.eql(u8, op, "!")) {
            return Value{ .boolean = !operand.isTruthy() };
        }
        return InterpreterError.TypeError;
    }
    
    fn getField(_: *Interpreter, obj: Value, field: []const u8) InterpreterError!Value {
        return switch (obj) {
            .class_instance => |inst| {
                if (inst.fields.get(field)) |v| return v;
                if (inst.class.methods.get(field)) |method| {
                    return Value{ .function = method };
                }
                return InterpreterError.UndefinedField;
            },
            else => InterpreterError.TypeError,
        };
    }
    
    fn getIndex(_: *Interpreter, obj: Value, idx: Value) InterpreterError!Value {
        return switch (obj) {
            .list => |list| {
                const i: usize = switch (idx) {
                    .integer => |v| if (v < 0) 
                        @intCast(@as(i64, @intCast(list.items.items.len)) + v)
                    else 
                        @intCast(v),
                    else => return InterpreterError.TypeError,
                };
                if (i >= list.items.items.len) return InterpreterError.IndexOutOfBounds;
                return list.items.items[i];
            },
            .map => |map| {
                const key: []const u8 = switch (idx) {
                    .string => |s| s.data,
                    else => return InterpreterError.TypeError,
                };
                return map.entries.get(key) orelse Value{ .nil = {} };
            },
            else => InterpreterError.TypeError,
        };
    }
    
    fn executeStatement(self: *Interpreter, stmt: *const StmtNode) InterpreterError!void {
        switch (stmt.*) {
            .expression_stmt => |expr| {
                _ = try self.evaluateExpr(expr);
            },
            .var_decl => |n| {
                const value = if (n.initializer) |init|
                    try self.evaluateExpr(init)
                else
                    Value{ .nil = {} };
                try self.environment.define(n.name, value);
            },
            .block => |stmts| {
                const child_env = try self.environment.createChild();
                defer child_env.deinit();
                
                const saved = self.environment;
                self.environment = child_env;
                defer self.environment = saved;
                
                for (stmts) |s| {
                    try self.executeStatement(s);
                }
            },
            .if_stmt => |n| {
                const cond = try self.evaluateExpr(n.condition);
                if (cond.isTruthy()) {
                    try self.executeStatement(n.then_branch);
                } else if (n.else_branch) |else_s| {
                    try self.executeStatement(else_s);
                }
            },
            .while_stmt => |n| {
                while (true) {
                    const cond = try self.evaluateExpr(n.condition);
                    if (!cond.isTruthy()) break;
                    
                    self.executeStatement(n.body) catch |err| {
                        if (err == InterpreterError.Break) break;
                        if (err == InterpreterError.Continue) continue;
                        return err;
                    };
                }
            },
            .return_stmt => |val| {
                if (val) |v| {
                    self.return_value = try self.evaluateExpr(v);
                } else {
                    self.return_value = Value{ .nil = {} };
                }
                return InterpreterError.Return;
            },
            .fn_decl => |n| {
                const func = try self.allocator.create(FunctionObj);
                func.* = FunctionObj{
                    .name = n.name,
                    .params = n.params,
                    .body = n.body,
                    .closure = self.environment,
                    .arity = @intCast(n.params.len),
                };
                try self.environment.define(n.name, Value{ .function = func });
            },
            else => {},
        }
    }
};

fn hashString(s: []const u8) u64 {
    var hash: u64 = 14695981039346656037;
    for (s) |c| {
        hash ^= c;
        hash *%= 1099511628211;
    }
    return hash;
}
```

---

## Step 884: Closures Implementation

```zig
// closures.zig - การ implement closures

// Closures ใน interpreter ทำงานได้เพราะ FunctionObj เก็บ reference ไปยัง
// environment ณ ตอนที่ function ถูก define ไว้

// ตัวอย่างที่แสดงให้เห็นว่า closures ทำงานอย่างไร:
//
// fn makeCounter() {
//     var count = 0;
//     fn increment() {
//         count = count + 1;
//         return count;
//     }
//     return increment;
// }
//
// var counter = makeCounter();
// counter(); // returns 1
// counter(); // returns 2 - count ยังคงอยู่ใน closure!

pub const ClosureDemo = struct {
    pub fn demonstrate(allocator: std.mem.Allocator) !void {
        var interp = try Interpreter.init(allocator);
        defer interp.deinit();
        
        // Define builtins
        try setupBuiltins(&interp);
        
        // Source code ที่ใช้ closures
        const source =
            \\fn makeAdder(x) {
            \\    fn add(y) {
            \\        return x + y;
            \\    }
            \\    return add;
            \\}
            \\var add5 = makeAdder(5);
            \\var result1 = add5(3);
            \\var result2 = add5(10);
        ;
        
        // Parse and execute
        var lexer = Lexer.init(source);
        const tokens = try lexer.tokenize(allocator);
        defer allocator.free(tokens);
        
        var parser = CompleteParser.init(tokens, allocator);
        defer parser.deinit();
        
        const stmts = try parser.parseProgram();
        defer allocator.free(stmts);
        
        for (stmts) |stmt| {
            try interp.executeStatement(stmt);
        }
        
        // result1 should be 8 (5+3)
        const r1 = interp.environment.get("result1");
        const r2 = interp.environment.get("result2");
        
        std.debug.print("add5(3) = {any}\n", .{r1});
        std.debug.print("add5(10) = {any}\n", .{r2});
    }
};

// Upvalue สำหรับ closure optimization
// แทนที่จะเก็บ environment ทั้งหมด เราสามารถเก็บเฉพาะ variables ที่ถูก capture
pub const Upvalue = struct {
    name: []const u8,
    value: *Value, // pointer ไปยัง value ที่ captured
    is_closed: bool, // true ถ้า variable ออกจาก scope แล้ว
    closed_value: Value, // copy ของ value เมื่อ close
    
    pub fn capture(env: *Environment, name: []const u8, allocator: std.mem.Allocator) !*Upvalue {
        const uv = try allocator.create(Upvalue);
        if (env.values.getPtr(name)) |ptr| {
            uv.* = Upvalue{
                .name = name,
                .value = ptr,
                .is_closed = false,
                .closed_value = undefined,
            };
        } else {
            uv.* = Upvalue{
                .name = name,
                .value = &uv.closed_value,
                .is_closed = true,
                .closed_value = Value{ .nil = {} },
            };
        }
        return uv;
    }
    
    pub fn close(self: *Upvalue) void {
        if (!self.is_closed) {
            self.closed_value = self.value.*;
            self.value = &self.closed_value;
            self.is_closed = true;
        }
    }
    
    pub fn get(self: *const Upvalue) Value {
        return self.value.*;
    }
    
    pub fn set(self: *Upvalue, value: Value) void {
        self.value.* = value;
    }
};
```

---

## Step 885: First-Class Functions

```zig
// first_class_fn.zig - First-class functions

// ตัวอย่างที่แสดง first-class functions:
// - Functions เป็น values
// - Functions สามารถส่งเป็น arguments ได้
// - Functions สามารถ return จาก functions ได้
// - Functions สามารถเก็บใน variables ได้

pub fn setupBuiltins(interp: *Interpreter) !void {
    const allocator = interp.allocator;
    
    // map function: apply fn to each element
    const map_builtin = try allocator.create(BuiltinFnObj);
    map_builtin.* = .{
        .name = "map",
        .func = builtinMap,
        .arity = 2,
    };
    try interp.globals.define("map", Value{ .builtin_fn = map_builtin });
    
    // filter function: keep elements matching predicate
    const filter_builtin = try allocator.create(BuiltinFnObj);
    filter_builtin.* = .{
        .name = "filter",
        .func = builtinFilter,
        .arity = 2,
    };
    try interp.globals.define("filter", Value{ .builtin_fn = filter_builtin });
    
    // reduce function: fold list to single value
    const reduce_builtin = try allocator.create(BuiltinFnObj);
    reduce_builtin.* = .{
        .name = "reduce",
        .func = builtinReduce,
        .arity = 3,
    };
    try interp.globals.define("reduce", Value{ .builtin_fn = reduce_builtin });
    
    // print function
    const print_builtin = try allocator.create(BuiltinFnObj);
    print_builtin.* = .{
        .name = "print",
        .func = builtinPrint,
        .arity = -1, // variadic
    };
    try interp.globals.define("print", Value{ .builtin_fn = print_builtin });
    
    // len function
    const len_builtin = try allocator.create(BuiltinFnObj);
    len_builtin.* = .{
        .name = "len",
        .func = builtinLen,
        .arity = 1,
    };
    try interp.globals.define("len", Value{ .builtin_fn = len_builtin });
}

fn builtinMap(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return InterpreterError.WrongArity;
    
    const list = switch (args[0]) {
        .list => |l| l,
        else => return InterpreterError.TypeError,
    };
    
    const func = args[1];
    
    const result = try allocator.create(ListObj);
    result.* = ListObj{ .items = std.ArrayList(Value).init(allocator) };
    
    // Apply function to each element
    for (list.items.items) |item| {
        var call_args = [_]Value{item};
        // We'd need an interpreter reference here - simplified for demo
        _ = func;
        try result.items.append(item); // placeholder
        _ = call_args;
    }
    
    return Value{ .list = result };
}

fn builtinFilter(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return InterpreterError.WrongArity;
    
    const list = switch (args[0]) {
        .list => |l| l,
        else => return InterpreterError.TypeError,
    };
    
    const result = try allocator.create(ListObj);
    result.* = ListObj{ .items = std.ArrayList(Value).init(allocator) };
    
    for (list.items.items) |item| {
        // ต้องมี interpreter reference จริงๆ - simplified
        _ = item;
        _ = args[1];
    }
    
    return Value{ .list = result };
}

fn builtinReduce(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    _ = allocator;
    if (args.len != 3) return InterpreterError.WrongArity;
    
    const list = switch (args[0]) {
        .list => |l| l,
        else => return InterpreterError.TypeError,
    };
    
    var acc = args[2]; // initial value
    
    for (list.items.items) |item| {
        _ = item;
        _ = args[1];
        // Apply reducer function - simplified
        acc = acc; // placeholder
    }
    
    return acc;
}

fn builtinPrint(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    _ = allocator;
    const stdout = std.io.getStdOut().writer();
    for (args, 0..) |arg, i| {
        if (i > 0) stdout.writeAll(" ") catch {};
        switch (arg) {
            .nil => stdout.writeAll("nil") catch {},
            .boolean => |b| stdout.print("{}", .{b}) catch {},
            .integer => |v| stdout.print("{d}", .{v}) catch {},
            .float => |v| stdout.print("{d}", .{v}) catch {},
            .string => |s| stdout.writeAll(s.data) catch {},
            else => stdout.writeAll("[object]") catch {},
        }
    }
    stdout.writeAll("\n") catch {};
    return Value{ .nil = {} };
}

fn builtinLen(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    _ = allocator;
    if (args.len != 1) return InterpreterError.WrongArity;
    return switch (args[0]) {
        .list => |l| Value{ .integer = @intCast(l.items.items.len) },
        .string => |s| Value{ .integer = @intCast(s.data.len) },
        .map => |m| Value{ .integer = @intCast(m.entries.count()) },
        else => InterpreterError.TypeError,
    };
}
```

---

## Step 886: Garbage Collection (Mark-and-Sweep)

```zig
// gc.zig - Mark-and-Sweep Garbage Collection

const std = @import("std");

// Object header สำหรับ GC
pub const GcObject = struct {
    is_marked: bool = false,
    next: ?*GcObject = null, // linked list ของ objects ทั้งหมด
    
    pub const Kind = enum {
        string,
        list,
        map,
        function,
        instance,
    };
};

pub const Gc = struct {
    allocator: std.mem.Allocator,
    objects: ?*GcObject, // head of linked list
    gray_stack: std.ArrayList(*GcObject), // worklist สำหรับ tri-color marking
    bytes_allocated: usize,
    next_gc: usize,
    
    const INITIAL_HEAP_SIZE = 1024 * 1024; // 1 MB
    const HEAP_GROW_FACTOR = 2;
    
    pub fn init(allocator: std.mem.Allocator) Gc {
        return Gc{
            .allocator = allocator,
            .objects = null,
            .gray_stack = std.ArrayList(*GcObject).init(allocator),
            .bytes_allocated = 0,
            .next_gc = INITIAL_HEAP_SIZE,
        };
    }
    
    pub fn deinit(self: *Gc) void {
        self.sweepAll();
        self.gray_stack.deinit();
    }
    
    // Allocate GC-managed memory
    pub fn alloc(self: *Gc, comptime T: type) !*T {
        const obj = try self.allocator.create(T);
        self.bytes_allocated += @sizeOf(T);
        
        // Add to object list
        const gc_obj = @as(*GcObject, @ptrCast(obj));
        gc_obj.is_marked = false;
        gc_obj.next = self.objects;
        self.objects = gc_obj;
        
        return obj;
    }
    
    // Mark phase
    pub fn markRoots(self: *Gc, interpreter: *const Interpreter) !void {
        // Mark stack
        for (interpreter.call_stack.items) |frame| {
            _ = frame;
            // Mark values in call frame
        }
        
        // Mark globals
        var it = interpreter.globals.values.valueIterator();
        while (it.next()) |val| {
            try self.markValue(val.*);
        }
    }
    
    fn markValue(self: *Gc, value: Value) !void {
        switch (value) {
            .string => |s| try self.markObject(@ptrCast(s)),
            .list => |l| try self.markObject(@ptrCast(l)),
            .map => |m| try self.markObject(@ptrCast(m)),
            .function => |f| try self.markObject(@ptrCast(f)),
            .class_instance => |i| try self.markObject(@ptrCast(i)),
            else => {},
        }
    }
    
    fn markObject(self: *Gc, obj: *GcObject) !void {
        if (obj.is_marked) return;
        obj.is_marked = true;
        try self.gray_stack.append(obj);
    }
    
    // Trace phase (process gray objects)
    pub fn traceReferences(self: *Gc) !void {
        while (self.gray_stack.items.len > 0) {
            const obj = self.gray_stack.pop();
            try self.blackenObject(obj);
        }
    }
    
    fn blackenObject(self: *Gc, obj: *GcObject) !void {
        // ต้อง determine object type จาก header
        // Simplified - in practice use tagged pointer or type field
        _ = obj;
        _ = self;
    }
    
    // Sweep phase - free unmarked objects
    pub fn sweep(self: *Gc) void {
        var previous: ?*GcObject = null;
        var current = self.objects;
        
        while (current) |obj| {
            if (obj.is_marked) {
                obj.is_marked = false; // reset for next collection
                previous = obj;
                current = obj.next;
            } else {
                // Free this object
                const unreachable_obj = obj;
                current = obj.next;
                
                if (previous) |prev| {
                    prev.next = current;
                } else {
                    self.objects = current;
                }
                
                self.freeObject(unreachable_obj);
            }
        }
    }
    
    fn freeObject(self: *Gc, obj: *GcObject) void {
        // Free based on type - simplified
        self.allocator.destroy(obj);
        // Note: ใน implementation จริงต้องรู้ size ของ object ด้วย
    }
    
    fn sweepAll(self: *Gc) void {
        var current = self.objects;
        while (current) |obj| {
            const next = obj.next;
            self.allocator.destroy(obj);
            current = next;
        }
        self.objects = null;
    }
    
    // Trigger collection if needed
    pub fn maybeCollect(self: *Gc, interpreter: *const Interpreter) !void {
        if (self.bytes_allocated >= self.next_gc) {
            try self.collect(interpreter);
        }
    }
    
    pub fn collect(self: *Gc, interpreter: *const Interpreter) !void {
        const before = self.bytes_allocated;
        
        // Phase 1: Mark
        try self.markRoots(interpreter);
        
        // Phase 2: Trace
        try self.traceReferences();
        
        // Phase 3: Sweep
        self.sweep();
        
        // Adjust next GC threshold
        self.next_gc = self.bytes_allocated * HEAP_GROW_FACTOR;
        
        const after = self.bytes_allocated;
        std.debug.print("GC: freed {} bytes ({} -> {})\n", .{
            before - after, before, after,
        });
    }
};
```

---

## Step 887: Error Handling in Interpreter

```zig
// error_handling.zig - Error handling ใน interpreter

const std = @import("std");

// Runtime error with stack trace
pub const RuntimeError = struct {
    message: []const u8,
    stack_trace: []StackFrame,
    
    pub const StackFrame = struct {
        function: []const u8,
        file: []const u8,
        line: u32,
    };
    
    pub fn format(
        self: *const RuntimeError,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("RuntimeError: {s}\n", .{self.message});
        try writer.writeAll("Stack trace:\n");
        for (self.stack_trace) |frame| {
            try writer.print("  at {s} ({s}:{d})\n", .{
                frame.function,
                frame.file,
                frame.line,
            });
        }
    }
};

// Error handler ที่แยกระหว่าง runtime errors และ control flow exceptions
pub const InterpreterWithErrors = struct {
    base: Interpreter,
    error_handler: ErrorHandler,
    
    pub const ErrorHandler = struct {
        errors: std.ArrayList(RuntimeError),
        allocator: std.mem.Allocator,
        
        pub fn init(allocator: std.mem.Allocator) ErrorHandler {
            return .{ .errors = std.ArrayList(RuntimeError).init(allocator), .allocator = allocator };
        }
        
        pub fn deinit(self: *ErrorHandler) void {
            for (self.errors.items) |err| {
                self.allocator.free(err.message);
                self.allocator.free(err.stack_trace);
            }
            self.errors.deinit();
        }
        
        pub fn report(self: *ErrorHandler, message: []const u8, call_stack: []const Interpreter.CallInfo) !void {
            const frames = try self.allocator.alloc(RuntimeError.StackFrame, call_stack.len);
            for (call_stack, 0..) |info, i| {
                frames[i] = .{
                    .function = info.function_name,
                    .file = "unknown",
                    .line = 0,
                };
            }
            
            const err_msg = try self.allocator.dupe(u8, message);
            try self.errors.append(RuntimeError{
                .message = err_msg,
                .stack_trace = frames,
            });
        }
    };
    
    pub fn init(allocator: std.mem.Allocator) !InterpreterWithErrors {
        return InterpreterWithErrors{
            .base = try Interpreter.init(allocator),
            .error_handler = ErrorHandler.init(allocator),
        };
    }
    
    pub fn deinit(self: *InterpreterWithErrors) void {
        self.base.deinit();
        self.error_handler.deinit();
    }
    
    pub fn safeExecute(self: *InterpreterWithErrors, stmt: *const StmtNode) !void {
        self.base.executeStatement(stmt) catch |err| {
            const msg = switch (err) {
                InterpreterError.UndefinedVariable => "Undefined variable",
                InterpreterError.NotCallable => "Value is not callable",
                InterpreterError.WrongArity => "Wrong number of arguments",
                InterpreterError.TypeError => "Type error",
                InterpreterError.DivisionByZero => "Division by zero",
                InterpreterError.StackOverflow => "Maximum recursion depth exceeded",
                else => @errorName(err),
            };
            
            try self.error_handler.report(msg, self.base.call_stack.items);
        };
    }
};
```

---

## Step 888: Standard Library for Interpreter

```zig
// stdlib.zig - Standard Library สำหรับ Interpreter

const std = @import("std");

pub fn registerStdlib(interp: *Interpreter) !void {
    const allocator = interp.allocator;
    
    // Math module
    try registerMath(interp, allocator);
    
    // String module
    try registerString(interp, allocator);
    
    // List module
    try registerList(interp, allocator);
    
    // IO module
    try registerIO(interp, allocator);
}

fn registerBuiltin(
    interp: *Interpreter,
    name: []const u8,
    func: *const fn ([]Value, std.mem.Allocator) InterpreterError!Value,
    arity: i8,
) !void {
    const b = try interp.allocator.create(BuiltinFnObj);
    b.* = .{ .name = name, .func = func, .arity = arity };
    try interp.globals.define(name, Value{ .builtin_fn = b });
}

fn registerMath(interp: *Interpreter, allocator: std.mem.Allocator) !void {
    _ = allocator;
    
    try registerBuiltin(interp, "abs", mathAbs, 1);
    try registerBuiltin(interp, "sqrt", mathSqrt, 1);
    try registerBuiltin(interp, "floor", mathFloor, 1);
    try registerBuiltin(interp, "ceil", mathCeil, 1);
    try registerBuiltin(interp, "max", mathMax, 2);
    try registerBuiltin(interp, "min", mathMin, 2);
}

fn mathAbs(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    return switch (args[0]) {
        .integer => |v| Value{ .integer = if (v < 0) -v else v },
        .float => |v| Value{ .float = @abs(v) },
        else => error.TypeError,
    };
}

fn mathSqrt(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    return switch (args[0]) {
        .integer => |v| Value{ .float = std.math.sqrt(@as(f64, @floatFromInt(v))) },
        .float => |v| Value{ .float = std.math.sqrt(v) },
        else => error.TypeError,
    };
}

fn mathFloor(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    return switch (args[0]) {
        .float => |v| Value{ .integer = @intFromFloat(@floor(v)) },
        .integer => args[0],
        else => error.TypeError,
    };
}

fn mathCeil(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    return switch (args[0]) {
        .float => |v| Value{ .integer = @intFromFloat(@ceil(v)) },
        .integer => args[0],
        else => error.TypeError,
    };
}

fn mathMax(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return error.WrongArity;
    return switch (args[0]) {
        .integer => |a| switch (args[1]) {
            .integer => |b| Value{ .integer = if (a > b) a else b },
            else => error.TypeError,
        },
        .float => |a| switch (args[1]) {
            .float => |b| Value{ .float = if (a > b) a else b },
            else => error.TypeError,
        },
        else => error.TypeError,
    };
}

fn mathMin(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return error.WrongArity;
    return switch (args[0]) {
        .integer => |a| switch (args[1]) {
            .integer => |b| Value{ .integer = if (a < b) a else b },
            else => error.TypeError,
        },
        else => error.TypeError,
    };
}

fn registerString(interp: *Interpreter, _: std.mem.Allocator) !void {
    try registerBuiltin(interp, "strlen", strLen, 1);
    try registerBuiltin(interp, "substr", strSub, 3);
    try registerBuiltin(interp, "str", strConvert, 1);
    try registerBuiltin(interp, "int", intConvert, 1);
}

fn strLen(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    return builtinLen(args, allocator);
}

fn strSub(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 3) return error.WrongArity;
    const s = switch (args[0]) {
        .string => |sv| sv.data,
        else => return error.TypeError,
    };
    const start: usize = switch (args[1]) {
        .integer => |v| @intCast(if (v < 0) 0 else v),
        else => return error.TypeError,
    };
    const len: usize = switch (args[2]) {
        .integer => |v| @intCast(if (v < 0) 0 else v),
        else => return error.TypeError,
    };
    
    const end = @min(start + len, s.len);
    const sub = if (start < s.len) s[start..end] else "";
    
    const result = try allocator.create(StringObj);
    result.* = .{ .data = sub, .hash = hashString(sub), .is_interned = false };
    return Value{ .string = result };
}

fn strConvert(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    const s = switch (args[0]) {
        .integer => |v| try std.fmt.allocPrint(allocator, "{d}", .{v}),
        .float => |v| try std.fmt.allocPrint(allocator, "{d}", .{v}),
        .boolean => |v| if (v) "true" else "false",
        .nil => "nil",
        .string => |sv| sv.data,
        else => return error.TypeError,
    };
    const result = try allocator.create(StringObj);
    result.* = .{ .data = s, .hash = hashString(s), .is_interned = false };
    return Value{ .string = result };
}

fn intConvert(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    return switch (args[0]) {
        .integer => args[0],
        .float => |v| Value{ .integer = @intFromFloat(v) },
        .string => |s| blk: {
            const v = std.fmt.parseInt(i64, s.data, 10) catch return error.TypeError;
            break :blk Value{ .integer = v };
        },
        .boolean => |b| Value{ .integer = if (b) 1 else 0 },
        else => error.TypeError,
    };
}

fn registerList(interp: *Interpreter, _: std.mem.Allocator) !void {
    try registerBuiltin(interp, "push", listPush, 2);
    try registerBuiltin(interp, "pop", listPop, 1);
    try registerBuiltin(interp, "concat", listConcat, 2);
}

fn listPush(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return error.WrongArity;
    const list = switch (args[0]) {
        .list => |l| l,
        else => return error.TypeError,
    };
    list.items.append(args[1]) catch return error.OutOfMemory;
    return Value{ .nil = {} };
}

fn listPop(args: []Value, _: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 1) return error.WrongArity;
    const list = switch (args[0]) {
        .list => |l| l,
        else => return error.TypeError,
    };
    if (list.items.items.len == 0) return error.IndexOutOfBounds;
    return list.items.pop();
}

fn listConcat(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    if (args.len != 2) return error.WrongArity;
    const l1 = switch (args[0]) {
        .list => |l| l,
        else => return error.TypeError,
    };
    const l2 = switch (args[1]) {
        .list => |l| l,
        else => return error.TypeError,
    };
    
    const result = try allocator.create(ListObj);
    result.* = ListObj{ .items = std.ArrayList(Value).init(allocator) };
    
    for (l1.items.items) |v| result.items.append(v) catch return error.OutOfMemory;
    for (l2.items.items) |v| result.items.append(v) catch return error.OutOfMemory;
    
    return Value{ .list = result };
}

fn registerIO(interp: *Interpreter, _: std.mem.Allocator) !void {
    try registerBuiltin(interp, "print", builtinPrint, -1);
    try registerBuiltin(interp, "println", builtinPrintln, -1);
    try registerBuiltin(interp, "input", builtinInput, 0);
}

fn builtinPrintln(args: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    _ = try builtinPrint(args, allocator);
    return Value{ .nil = {} };
}

fn builtinInput(_: []Value, allocator: std.mem.Allocator) InterpreterError!Value {
    const stdin = std.io.getStdIn().reader();
    const line = stdin.readUntilDelimiterAlloc(allocator, '\n', 4096) catch 
        return error.RuntimeError;
    const trimmed = std.mem.trimRight(u8, line, "\r");
    
    const result = try allocator.create(StringObj);
    result.* = .{ .data = trimmed, .hash = hashString(trimmed), .is_interned = false };
    return Value{ .string = result };
}
```

---

## Step 889: REPL Implementation

```zig
// repl.zig - Read-Eval-Print Loop

const std = @import("std");

pub const Repl = struct {
    allocator: std.mem.Allocator,
    interpreter: Interpreter,
    history: std.ArrayList([]const u8),
    
    pub fn init(allocator: std.mem.Allocator) !Repl {
        var repl = Repl{
            .allocator = allocator,
            .interpreter = try Interpreter.init(allocator),
            .history = std.ArrayList([]const u8).init(allocator),
        };
        
        // Register standard library
        try setupBuiltins(&repl.interpreter);
        try registerStdlib(&repl.interpreter);
        
        return repl;
    }
    
    pub fn deinit(self: *Repl) void {
        for (self.history.items) |line| {
            self.allocator.free(line);
        }
        self.history.deinit();
        self.interpreter.deinit();
    }
    
    pub fn run(self: *Repl) !void {
        const stdin = std.io.getStdIn().reader();
        const stdout = std.io.getStdOut().writer();
        
        try stdout.writeAll(
            \\Welcome to SimpleScript REPL v1.0
            \\Type 'exit' to quit, 'help' for help
            \\
        );
        
        var line_buf: [4096]u8 = undefined;
        var multiline_buf = std.ArrayList(u8).init(self.allocator);
        defer multiline_buf.deinit();
        var in_multiline = false;
        
        while (true) {
            // Prompt
            if (in_multiline) {
                try stdout.writeAll("... ");
            } else {
                try stdout.writeAll(">>> ");
            }
            
            // Read line
            const line = stdin.readUntilDelimiter(&line_buf, '\n') catch break;
            const trimmed = std.mem.trimRight(u8, line, "\r");
            
            // Special commands
            if (std.mem.eql(u8, trimmed, "exit") or std.mem.eql(u8, trimmed, "quit")) {
                try stdout.writeAll("Goodbye!\n");
                break;
            }
            
            if (std.mem.eql(u8, trimmed, "help")) {
                try self.printHelp(stdout);
                continue;
            }
            
            if (std.mem.eql(u8, trimmed, "history")) {
                for (self.history.items, 0..) |hist, i| {
                    try stdout.print("{d}: {s}\n", .{ i + 1, hist });
                }
                continue;
            }
            
            if (std.mem.eql(u8, trimmed, "clear")) {
                // Clear screen
                try stdout.writeAll("\x1b[2J\x1b[H");
                continue;
            }
            
            // Check for multiline input
            if (std.mem.endsWith(u8, trimmed, "{")) {
                in_multiline = true;
                try multiline_buf.appendSlice(trimmed);
                try multiline_buf.append('\n');
                continue;
            }
            
            if (in_multiline) {
                try multiline_buf.appendSlice(trimmed);
                try multiline_buf.append('\n');
                
                if (std.mem.eql(u8, trimmed, "}")) {
                    in_multiline = false;
                    const source = multiline_buf.items;
                    try self.evaluate(source, stdout);
                    multiline_buf.clearRetainingCapacity();
                }
                continue;
            }
            
            // Add to history
            const hist_line = try self.allocator.dupe(u8, trimmed);
            try self.history.append(hist_line);
            
            // Evaluate
            try self.evaluate(trimmed, stdout);
        }
    }
    
    fn evaluate(self: *Repl, source: []const u8, writer: anytype) !void {
        if (source.len == 0) return;
        
        // Try as expression first (ถ้าเป็น expression แสดงผล)
        // ถ้าไม่ใช่ expression ลอง parse เป็น statement
        
        var lexer = Lexer.init(source);
        const tokens = self.interpreter.allocator.alloc(Token, 0) catch return;
        _ = tokens;
        
        // Simplified - just try to lex and parse
        var lex = Lexer.init(source);
        const toks = lex.tokenize(self.allocator) catch |err| {
            try writer.print("Lex error: {}\n", .{err});
            return;
        };
        defer self.allocator.free(toks);
        
        var parser = CompleteParser.init(toks, self.allocator);
        defer parser.deinit();
        
        const stmts = parser.parseProgram() catch |err| {
            try writer.print("Parse error: {}\n", .{err});
            if (parser.errors.items.len > 0) {
                for (parser.errors.items) |e| {
                    try writer.print("  {s}\n", .{e.message});
                }
            }
            return;
        };
        defer self.allocator.free(stmts);
        
        for (stmts) |stmt| {
            self.interpreter.executeStatement(stmt) catch |err| {
                try writer.print("Runtime error: {}\n", .{err});
                return;
            };
        }
        
        // If last statement was expression, print result
        if (stmts.len > 0) {
            const last = stmts[stmts.len - 1];
            if (last.* == .expression_stmt) {
                const result = self.interpreter.evaluateExpr(last.expression_stmt) catch return;
                switch (result) {
                    .nil => {}, // don't print nil
                    else => try writer.print("=> {any}\n", .{result}),
                }
            }
        }
    }
    
    fn printHelp(_: *Repl, writer: anytype) !void {
        try writer.writeAll(
            \\Commands:
            \\  exit / quit  - Exit REPL
            \\  history      - Show command history
            \\  clear        - Clear screen
            \\
            \\Language:
            \\  var x = expr;          - Variable declaration
            \\  const x = expr;        - Constant declaration
            \\  fn name(params) { }    - Function declaration
            \\  if (cond) { } else { } - If statement
            \\  while (cond) { }       - While loop
            \\  for (init; cond; upd)  - For loop
            \\  return expr;           - Return from function
            \\
            \\Built-in functions:
            \\  print(...)   - Print values
            \\  len(x)       - Get length
            \\  abs(n)       - Absolute value
            \\  sqrt(n)      - Square root
            \\  str(x)       - Convert to string
            \\  int(x)       - Convert to integer
            \\
        );
    }
};
```

---

## Step 890: Complete Language Interpreter

```zig
// complete_interpreter.zig - Complete interpreter สำหรับ SimpleScript

const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);
    
    if (args.len > 1) {
        // Run file mode
        const filename = args[1];
        try runFile(filename, allocator);
    } else {
        // REPL mode
        var repl = try Repl.init(allocator);
        defer repl.deinit();
        try repl.run();
    }
}

fn runFile(filename: []const u8, allocator: std.mem.Allocator) !void {
    const source = try std.fs.cwd().readFileAlloc(allocator, filename, 10 * 1024 * 1024);
    defer allocator.free(source);
    
    var interp = try Interpreter.init(allocator);
    defer interp.deinit();
    
    try setupBuiltins(&interp);
    try registerStdlib(&interp);
    
    // Lex
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    // Parse
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    if (parser.errors.items.len > 0) {
        for (parser.errors.items) |err| {
            std.debug.print("Error: {s}\n", .{err.message});
        }
        return;
    }
    
    // Execute
    for (stmts) |stmt| {
        interp.executeStatement(stmt) catch |err| {
            std.debug.print("Runtime error: {}\n", .{err});
            
            // Print stack trace
            if (interp.call_stack.items.len > 0) {
                std.debug.print("Stack trace:\n", .{});
                var i = interp.call_stack.items.len;
                while (i > 0) {
                    i -= 1;
                    std.debug.print("  at {s}\n", .{interp.call_stack.items[i].function_name});
                }
            }
            return;
        };
    }
}

// ทดสอบ complete interpreter
test "complete interpreter: fibonacci" {
    const allocator = std.testing.allocator;
    
    const source =
        \\fn fib(n) {
        \\    if (n <= 1) {
        \\        return n;
        \\    }
        \\    return fib(n - 1) + fib(n - 2);
        \\}
        \\var result = fib(10);
    ;
    
    var interp = try Interpreter.init(allocator);
    defer interp.deinit();
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    try std.testing.expectEqual(@as(usize, 0), parser.errors.items.len);
    
    for (stmts) |stmt| {
        try interp.executeStatement(stmt);
    }
    
    const result = interp.globals.get("result");
    try std.testing.expect(result != null);
    try std.testing.expectEqual(@as(i64, 55), result.?.integer); // fib(10) = 55
}

test "closures work correctly" {
    const allocator = std.testing.allocator;
    
    const source =
        \\fn makeCounter() {
        \\    var count = 0;
        \\    fn increment() {
        \\        count = count + 1;
        \\        return count;
        \\    }
        \\    return increment;
        \\}
        \\var counter = makeCounter();
        \\var a = counter();
        \\var b = counter();
        \\var c = counter();
    ;
    
    var interp = try Interpreter.init(allocator);
    defer interp.deinit();
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    for (stmts) |stmt| {
        try interp.executeStatement(stmt);
    }
    
    // a=1, b=2, c=3 ถ้า closures ทำงานถูกต้อง
    const a = interp.globals.get("a");
    const b = interp.globals.get("b");
    const c = interp.globals.get("c");
    
    try std.testing.expect(a != null);
    try std.testing.expect(b != null);
    try std.testing.expect(c != null);
    
    try std.testing.expectEqual(@as(i64, 1), a.?.integer);
    try std.testing.expectEqual(@as(i64, 2), b.?.integer);
    try std.testing.expectEqual(@as(i64, 3), c.?.integer);
}
```

---

## สรุป Part 89

ในส่วนนี้เราได้เรียนรู้:

1. **Step 881**: Interpreter architecture - Value types และ tagged unions
2. **Step 882**: Environment/scope - Scope chain สำหรับ variable lookup
3. **Step 883**: Function calls - Call stack, arity checking, execution
4. **Step 884**: Closures - การ capture environment ใน function objects
5. **Step 885**: First-class functions - Higher-order functions, map/filter/reduce
6. **Step 886**: Garbage collection - Mark-and-sweep algorithm
7. **Step 887**: Error handling - Runtime errors with stack traces
8. **Step 888**: Standard library - Math, string, list, IO builtins
9. **Step 889**: REPL - Interactive interpreter
10. **Step 890**: Complete interpreter - รวมทุกอย่างเป็น SimpleScript interpreter

การสร้าง interpreter ที่ดีต้องจัดการ memory อย่างระมัดระวัง โดยเฉพาะกับ closures และ first-class functions ที่ทำให้ object lifetime ไม่ deterministic
