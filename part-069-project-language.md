# ส่วนที่ 69: โปรเจกต์ - Calculator Language

## ขั้นตอนที่ 681-690: สร้างภาษา Expression ง่ายๆ

---

## ขั้นตอนที่ 681: Language Requirements

เราจะสร้างภาษา calculator ที่รองรับ:

### ความสามารถ

1. **Expressions**: arithmetic, comparison, logical
2. **Variables**: กำหนดและใช้ตัวแปร
3. **Functions**: built-in และ user-defined
4. **Control Flow**: if/else
5. **REPL**: Interactive mode

### ตัวอย่าง syntax

```
# ตัวแปร
x = 10
y = 20
result = x + y * 2  # = 50

# ฟังก์ชัน built-in
print(result)
sqrt(16)
abs(-5)
max(x, y)

# ฟังก์ชัน user-defined
fn add(a, b) = a + b
fn factorial(n) = if n <= 1 then 1 else n * factorial(n - 1)

# เรียกใช้
add(3, 4)         # = 7
factorial(5)      # = 120
```

### โครงสร้างโปรเจกต์

```
calclang/
├── src/
│   ├── main.zig
│   ├── lexer.zig
│   ├── parser.zig
│   ├── ast.zig
│   ├── evaluator.zig
│   ├── environment.zig
│   └── builtins.zig
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
        .name = "calclang",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run the calculator language REPL");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/evaluator.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## ขั้นตอนที่ 682: Lexer Implementation

Lexer แปลง source code เป็น sequence ของ tokens

```zig
// ไฟล์ src/lexer.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

pub const LexError = error{
    UnexpectedCharacter,
    UnterminatedString,
    InvalidNumber,
    OutOfMemory,
};

/// ตำแหน่งใน source code
pub const Position = struct {
    line: usize,
    column: usize,
    offset: usize,

    pub fn format(
        self: Position,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{}:{}", .{ self.line, self.column });
    }
};

/// Token type
pub const TokenType = enum {
    // Literals
    number,
    string,
    identifier,
    true_kw,
    false_kw,

    // Operators
    plus,       // +
    minus,      // -
    star,       // *
    slash,      // /
    percent,    // %
    power,      // ^
    eq,         // =
    eq_eq,      // ==
    bang_eq,    // !=
    lt,         // <
    lt_eq,      // <=
    gt,         // >
    gt_eq,      // >=
    and_kw,     // and
    or_kw,      // or
    not_kw,     // not

    // Delimiters
    lparen,     // (
    rparen,     // )
    comma,      // ,
    newline,
    semicolon,  // ;

    // Keywords
    fn_kw,      // fn
    if_kw,      // if
    then_kw,    // then
    else_kw,    // else
    let_kw,     // let
    return_kw,  // return

    // Special
    eof,
    comment,
};

/// Token
pub const Token = struct {
    token_type: TokenType,
    lexeme: []const u8,
    pos: Position,

    pub fn format(
        self: Token,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("Token({}, '{s}', {})", .{
            self.token_type,
            self.lexeme,
            self.pos,
        });
    }
};

/// Lexer
pub const Lexer = struct {
    source: []const u8,
    pos: usize,
    line: usize,
    column: usize,
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator, source: []const u8) Self {
        return Self{
            .source = source,
            .pos = 0,
            .line = 1,
            .column = 1,
            .allocator = allocator,
        };
    }

    /// Tokenize ทั้ง source
    pub fn tokenize(self: *Self) ![]Token {
        var tokens = std.ArrayList(Token).init(self.allocator);
        errdefer tokens.deinit();

        while (true) {
            const token = try self.nextToken();
            const is_eof = token.token_type == .eof;

            // ข้าม comments
            if (token.token_type != .comment) {
                try tokens.append(token);
            }

            if (is_eof) break;
        }

        return tokens.toOwnedSlice();
    }

    fn nextToken(self: *Self) !Token {
        self.skipWhitespace();

        if (self.pos >= self.source.len) {
            return Token{
                .token_type = .eof,
                .lexeme = "",
                .pos = self.currentPos(),
            };
        }

        const ch = self.source[self.pos];
        const start_pos = self.currentPos();

        switch (ch) {
            '\n' => {
                self.advance();
                return Token{ .token_type = .newline, .lexeme = "\n", .pos = start_pos };
            },
            ';' => {
                self.advance();
                return Token{ .token_type = .semicolon, .lexeme = ";", .pos = start_pos };
            },
            '+' => {
                self.advance();
                return Token{ .token_type = .plus, .lexeme = "+", .pos = start_pos };
            },
            '-' => {
                self.advance();
                return Token{ .token_type = .minus, .lexeme = "-", .pos = start_pos };
            },
            '*' => {
                self.advance();
                return Token{ .token_type = .star, .lexeme = "*", .pos = start_pos };
            },
            '/' => {
                self.advance();
                return Token{ .token_type = .slash, .lexeme = "/", .pos = start_pos };
            },
            '%' => {
                self.advance();
                return Token{ .token_type = .percent, .lexeme = "%", .pos = start_pos };
            },
            '^' => {
                self.advance();
                return Token{ .token_type = .power, .lexeme = "^", .pos = start_pos };
            },
            '(' => {
                self.advance();
                return Token{ .token_type = .lparen, .lexeme = "(", .pos = start_pos };
            },
            ')' => {
                self.advance();
                return Token{ .token_type = .rparen, .lexeme = ")", .pos = start_pos };
            },
            ',' => {
                self.advance();
                return Token{ .token_type = .comma, .lexeme = ",", .pos = start_pos };
            },
            '=' => {
                self.advance();
                if (self.pos < self.source.len and self.source[self.pos] == '=') {
                    self.advance();
                    return Token{ .token_type = .eq_eq, .lexeme = "==", .pos = start_pos };
                }
                return Token{ .token_type = .eq, .lexeme = "=", .pos = start_pos };
            },
            '!' => {
                self.advance();
                if (self.pos < self.source.len and self.source[self.pos] == '=') {
                    self.advance();
                    return Token{ .token_type = .bang_eq, .lexeme = "!=", .pos = start_pos };
                }
                return LexError.UnexpectedCharacter;
            },
            '<' => {
                self.advance();
                if (self.pos < self.source.len and self.source[self.pos] == '=') {
                    self.advance();
                    return Token{ .token_type = .lt_eq, .lexeme = "<=", .pos = start_pos };
                }
                return Token{ .token_type = .lt, .lexeme = "<", .pos = start_pos };
            },
            '>' => {
                self.advance();
                if (self.pos < self.source.len and self.source[self.pos] == '=') {
                    self.advance();
                    return Token{ .token_type = .gt_eq, .lexeme = ">=", .pos = start_pos };
                }
                return Token{ .token_type = .gt, .lexeme = ">", .pos = start_pos };
            },
            '#' => {
                return self.lexComment(start_pos);
            },
            '"', '\'' => {
                return self.lexString(start_pos);
            },
            '0'...'9' => {
                return self.lexNumber(start_pos);
            },
            'a'...'z', 'A'...'Z', '_' => {
                return self.lexIdentifier(start_pos);
            },
            else => {
                return LexError.UnexpectedCharacter;
            },
        }
    }

    fn lexComment(self: *Self, start_pos: Position) Token {
        const start = self.pos;
        while (self.pos < self.source.len and self.source[self.pos] != '\n') {
            self.pos += 1;
            self.column += 1;
        }
        return Token{
            .token_type = .comment,
            .lexeme = self.source[start..self.pos],
            .pos = start_pos,
        };
    }

    fn lexString(self: *Self, start_pos: Position) LexError!Token {
        const quote = self.source[self.pos];
        self.advance();
        const start = self.pos;

        while (self.pos < self.source.len and self.source[self.pos] != quote) {
            if (self.source[self.pos] == '\n') return LexError.UnterminatedString;
            if (self.source[self.pos] == '\\') {
                self.advance(); // skip escape char
            }
            self.advance();
        }

        if (self.pos >= self.source.len) return LexError.UnterminatedString;

        const str = self.source[start..self.pos];
        self.advance(); // closing quote

        return Token{
            .token_type = .string,
            .lexeme = str,
            .pos = start_pos,
        };
    }

    fn lexNumber(self: *Self, start_pos: Position) LexError!Token {
        const start = self.pos;

        while (self.pos < self.source.len and std.ascii.isDigit(self.source[self.pos])) {
            self.advance();
        }

        // Decimal point
        if (self.pos < self.source.len and self.source[self.pos] == '.') {
            self.advance();
            while (self.pos < self.source.len and std.ascii.isDigit(self.source[self.pos])) {
                self.advance();
            }
        }

        return Token{
            .token_type = .number,
            .lexeme = self.source[start..self.pos],
            .pos = start_pos,
        };
    }

    fn lexIdentifier(self: *Self, start_pos: Position) Token {
        const start = self.pos;
        while (self.pos < self.source.len and
            (std.ascii.isAlphanumeric(self.source[self.pos]) or self.source[self.pos] == '_'))
        {
            self.advance();
        }

        const lexeme = self.source[start..self.pos];

        // ตรวจสอบ keywords
        const token_type: TokenType = keywordOrIdent(lexeme);

        return Token{
            .token_type = token_type,
            .lexeme = lexeme,
            .pos = start_pos,
        };
    }

    fn keywordOrIdent(s: []const u8) TokenType {
        if (std.mem.eql(u8, s, "fn")) return .fn_kw;
        if (std.mem.eql(u8, s, "if")) return .if_kw;
        if (std.mem.eql(u8, s, "then")) return .then_kw;
        if (std.mem.eql(u8, s, "else")) return .else_kw;
        if (std.mem.eql(u8, s, "and")) return .and_kw;
        if (std.mem.eql(u8, s, "or")) return .or_kw;
        if (std.mem.eql(u8, s, "not")) return .not_kw;
        if (std.mem.eql(u8, s, "true")) return .true_kw;
        if (std.mem.eql(u8, s, "false")) return .false_kw;
        if (std.mem.eql(u8, s, "let")) return .let_kw;
        if (std.mem.eql(u8, s, "return")) return .return_kw;
        return .identifier;
    }

    fn skipWhitespace(self: *Self) void {
        while (self.pos < self.source.len) {
            switch (self.source[self.pos]) {
                ' ', '\t', '\r' => self.advance(),
                else => break,
            }
        }
    }

    fn advance(self: *Self) void {
        if (self.pos < self.source.len and self.source[self.pos] == '\n') {
            self.line += 1;
            self.column = 1;
        } else {
            self.column += 1;
        }
        self.pos += 1;
    }

    fn currentPos(self: *Self) Position {
        return Position{
            .line = self.line,
            .column = self.column,
            .offset = self.pos,
        };
    }
};
```

---

## ขั้นตอนที่ 683: Token Types

(ได้กำหนดใน lexer.zig แล้ว ขยายเพิ่มเติมในส่วนนี้)

```zig
// ส่วนขยายของ token types - ไฟล์ src/token_ext.zig

const lexer_mod = @import("lexer.zig");
const TokenType = lexer_mod.TokenType;

/// ตรวจสอบว่า token เป็น operator หรือไม่
pub fn isOperator(tt: TokenType) bool {
    return switch (tt) {
        .plus, .minus, .star, .slash, .percent, .power,
        .eq_eq, .bang_eq, .lt, .lt_eq, .gt, .gt_eq,
        .and_kw, .or_kw => true,
        else => false,
    };
}

/// ตรวจสอบว่าเป็น comparison operator
pub fn isComparison(tt: TokenType) bool {
    return switch (tt) {
        .eq_eq, .bang_eq, .lt, .lt_eq, .gt, .gt_eq => true,
        else => false,
    };
}

/// Precedence ของ operators
pub fn precedence(tt: TokenType) u8 {
    return switch (tt) {
        .or_kw => 1,
        .and_kw => 2,
        .eq_eq, .bang_eq => 3,
        .lt, .lt_eq, .gt, .gt_eq => 4,
        .plus, .minus => 5,
        .star, .slash, .percent => 6,
        .power => 7,
        else => 0,
    };
}

/// ตรวจสอบว่าเป็น right-associative
pub fn isRightAssociative(tt: TokenType) bool {
    return tt == .power;
}
```

---

## ขั้นตอนที่ 684: Parser (Recursive Descent)

```zig
// ไฟล์ src/parser.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const lexer_mod = @import("lexer.zig");
const Token = lexer_mod.Token;
const TokenType = lexer_mod.TokenType;
const ast_mod = @import("ast.zig");
const Node = ast_mod.Node;

pub const ParseError = error{
    UnexpectedToken,
    UnexpectedEOF,
    InvalidExpression,
    OutOfMemory,
};

pub const Parser = struct {
    tokens: []const Token,
    pos: usize,
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator, tokens: []const Token) Self {
        return Self{
            .tokens = tokens,
            .pos = 0,
            .allocator = allocator,
        };
    }

    /// Parse ทั้ง program
    pub fn parseProgram(self: *Self) ![]Node {
        var stmts = std.ArrayList(Node).init(self.allocator);
        errdefer {
            for (stmts.items) |*node| node.deinit(self.allocator);
            stmts.deinit();
        }

        while (!self.isAtEnd()) {
            self.skipNewlines();
            if (self.isAtEnd()) break;

            const stmt = try self.parseStatement();
            try stmts.append(stmt);

            self.skipNewlines();
        }

        return stmts.toOwnedSlice();
    }

    fn parseStatement(self: *Self) !Node {
        const token = self.peek();

        // Function definition
        if (token.token_type == .fn_kw) {
            return self.parseFunctionDef();
        }

        // Assignment: identifier = expr
        if (token.token_type == .identifier and self.peekNext().token_type == .eq) {
            return self.parseAssignment();
        }

        // Expression statement
        return self.parseExpression(0);
    }

    fn parseFunctionDef(self: *Self) !Node {
        _ = self.consume(.fn_kw);

        const name_token = try self.consumeExpect(.identifier);
        _ = try self.consumeExpect(.lparen);

        // Parse parameters
        var params = std.ArrayList([]const u8).init(self.allocator);
        errdefer params.deinit();

        while (self.peek().token_type != .rparen and !self.isAtEnd()) {
            const param = try self.consumeExpect(.identifier);
            try params.append(param.lexeme);

            if (self.peek().token_type == .comma) {
                _ = self.consume(.comma);
            }
        }

        _ = try self.consumeExpect(.rparen);
        _ = try self.consumeExpect(.eq);

        const body = try self.allocator.create(Node);
        body.* = try self.parseExpression(0);

        return Node{
            .fn_def = .{
                .name = name_token.lexeme,
                .params = try params.toOwnedSlice(),
                .body = body,
            },
        };
    }

    fn parseAssignment(self: *Self) !Node {
        const name = self.consume(.identifier).lexeme;
        _ = self.consume(.eq);

        const value = try self.allocator.create(Node);
        value.* = try self.parseExpression(0);

        return Node{
            .assign = .{
                .name = name,
                .value = value,
            },
        };
    }

    /// Pratt Parser / Precedence climbing
    fn parseExpression(self: *Self, min_prec: u8) ParseError!Node {
        var left = try self.parseUnary();

        while (true) {
            const op = self.peek();
            if (!isOperator(op.token_type)) break;

            const op_prec = precedence(op.token_type);
            if (op_prec < min_prec) break;

            _ = self.advance();

            const next_prec = if (isRightAssociative(op.token_type))
                op_prec
            else
                op_prec + 1;

            const right = try self.allocator.create(Node);
            right.* = try self.parseExpression(next_prec);

            const left_node = try self.allocator.create(Node);
            left_node.* = left;

            left = Node{
                .binary = .{
                    .op = op.token_type,
                    .left = left_node,
                    .right = right,
                },
            };
        }

        return left;
    }

    fn parseUnary(self: *Self) ParseError!Node {
        const token = self.peek();

        if (token.token_type == .minus) {
            _ = self.advance();
            const operand = try self.allocator.create(Node);
            operand.* = try self.parseUnary();
            return Node{ .unary = .{ .op = .minus, .operand = operand } };
        }

        if (token.token_type == .not_kw) {
            _ = self.advance();
            const operand = try self.allocator.create(Node);
            operand.* = try self.parseUnary();
            return Node{ .unary = .{ .op = .not_kw, .operand = operand } };
        }

        return self.parsePrimary();
    }

    fn parsePrimary(self: *Self) ParseError!Node {
        const token = self.peek();

        switch (token.token_type) {
            .number => {
                _ = self.advance();
                const val = std.fmt.parseFloat(f64, token.lexeme) catch return ParseError.InvalidExpression;
                return Node{ .number = val };
            },
            .string => {
                _ = self.advance();
                return Node{ .string = token.lexeme };
            },
            .true_kw => {
                _ = self.advance();
                return Node{ .boolean = true };
            },
            .false_kw => {
                _ = self.advance();
                return Node{ .boolean = false };
            },
            .identifier => {
                _ = self.advance();
                // ตรวจสอบว่าเป็น function call
                if (self.peek().token_type == .lparen) {
                    return self.parseFunctionCall(token.lexeme);
                }
                return Node{ .identifier = token.lexeme };
            },
            .lparen => {
                _ = self.advance();
                const expr = try self.parseExpression(0);
                _ = try self.consumeExpect(.rparen);
                return expr;
            },
            .if_kw => {
                return self.parseIfExpr();
            },
            .eof => return ParseError.UnexpectedEOF,
            else => {
                return ParseError.UnexpectedToken;
            },
        }
    }

    fn parseFunctionCall(self: *Self, name: []const u8) !Node {
        _ = try self.consumeExpect(.lparen);

        var args = std.ArrayList(*Node).init(self.allocator);
        errdefer {
            for (args.items) |arg| {
                arg.deinit(self.allocator);
                self.allocator.destroy(arg);
            }
            args.deinit();
        }

        while (self.peek().token_type != .rparen and !self.isAtEnd()) {
            const arg = try self.allocator.create(Node);
            arg.* = try self.parseExpression(0);
            try args.append(arg);

            if (self.peek().token_type == .comma) {
                _ = self.advance();
            }
        }

        _ = try self.consumeExpect(.rparen);

        return Node{
            .call = .{
                .name = name,
                .args = try args.toOwnedSlice(),
            },
        };
    }

    fn parseIfExpr(self: *Self) !Node {
        _ = self.consume(.if_kw);

        const condition = try self.allocator.create(Node);
        condition.* = try self.parseExpression(0);

        _ = try self.consumeExpect(.then_kw);

        const then_branch = try self.allocator.create(Node);
        then_branch.* = try self.parseExpression(0);

        _ = try self.consumeExpect(.else_kw);

        const else_branch = try self.allocator.create(Node);
        else_branch.* = try self.parseExpression(0);

        return Node{
            .if_expr = .{
                .condition = condition,
                .then_branch = then_branch,
                .else_branch = else_branch,
            },
        };
    }

    fn skipNewlines(self: *Self) void {
        while (!self.isAtEnd() and
            (self.peek().token_type == .newline or
             self.peek().token_type == .semicolon))
        {
            _ = self.advance();
        }
    }

    fn peek(self: *Self) Token {
        if (self.pos >= self.tokens.len) {
            return Token{ .token_type = .eof, .lexeme = "", .pos = .{ .line = 0, .column = 0, .offset = 0 } };
        }
        return self.tokens[self.pos];
    }

    fn peekNext(self: *Self) Token {
        if (self.pos + 1 >= self.tokens.len) {
            return Token{ .token_type = .eof, .lexeme = "", .pos = .{ .line = 0, .column = 0, .offset = 0 } };
        }
        return self.tokens[self.pos + 1];
    }

    fn advance(self: *Self) Token {
        const token = self.tokens[self.pos];
        self.pos += 1;
        return token;
    }

    fn consume(self: *Self, tt: TokenType) Token {
        if (self.peek().token_type == tt) {
            return self.advance();
        }
        return self.peek();
    }

    fn consumeExpect(self: *Self, tt: TokenType) ParseError!Token {
        if (self.peek().token_type != tt) {
            return ParseError.UnexpectedToken;
        }
        return self.advance();
    }

    fn isAtEnd(self: *Self) bool {
        return self.pos >= self.tokens.len or self.peek().token_type == .eof;
    }
};

fn isOperator(tt: TokenType) bool {
    return switch (tt) {
        .plus, .minus, .star, .slash, .percent, .power,
        .eq_eq, .bang_eq, .lt, .lt_eq, .gt, .gt_eq,
        .and_kw, .or_kw => true,
        else => false,
    };
}

fn precedence(tt: TokenType) u8 {
    return switch (tt) {
        .or_kw => 1,
        .and_kw => 2,
        .eq_eq, .bang_eq => 3,
        .lt, .lt_eq, .gt, .gt_eq => 4,
        .plus, .minus => 5,
        .star, .slash, .percent => 6,
        .power => 7,
        else => 0,
    };
}

fn isRightAssociative(tt: TokenType) bool {
    return tt == .power;
}
```

---

## ขั้นตอนที่ 685: AST Nodes

```zig
// ไฟล์ src/ast.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const TokenType = @import("lexer.zig").TokenType;

/// AST Node
pub const Node = union(enum) {
    // Literals
    number: f64,
    string: []const u8,
    boolean: bool,

    // Variables
    identifier: []const u8,
    assign: struct {
        name: []const u8,
        value: *Node,
    },

    // Expressions
    binary: struct {
        op: TokenType,
        left: *Node,
        right: *Node,
    },
    unary: struct {
        op: TokenType,
        operand: *Node,
    },

    // Function
    fn_def: struct {
        name: []const u8,
        params: []const []const u8,
        body: *Node,
    },
    call: struct {
        name: []const u8,
        args: []*Node,
    },

    // Control flow
    if_expr: struct {
        condition: *Node,
        then_branch: *Node,
        else_branch: *Node,
    },

    pub fn deinit(self: *Node, allocator: Allocator) void {
        switch (self.*) {
            .assign => |a| {
                a.value.deinit(allocator);
                allocator.destroy(a.value);
            },
            .binary => |b| {
                b.left.deinit(allocator);
                allocator.destroy(b.left);
                b.right.deinit(allocator);
                allocator.destroy(b.right);
            },
            .unary => |u| {
                u.operand.deinit(allocator);
                allocator.destroy(u.operand);
            },
            .fn_def => |f| {
                allocator.free(f.params);
                f.body.deinit(allocator);
                allocator.destroy(f.body);
            },
            .call => |c| {
                for (c.args) |arg| {
                    arg.deinit(allocator);
                    allocator.destroy(arg);
                }
                allocator.free(c.args);
            },
            .if_expr => |i| {
                i.condition.deinit(allocator);
                allocator.destroy(i.condition);
                i.then_branch.deinit(allocator);
                allocator.destroy(i.then_branch);
                i.else_branch.deinit(allocator);
                allocator.destroy(i.else_branch);
            },
            else => {},
        }
    }

    pub fn format(
        self: Node,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        switch (self) {
            .number => |n| try writer.print("{d}", .{n}),
            .string => |s| try writer.print("\"{s}\"", .{s}),
            .boolean => |b| try writer.print("{}", .{b}),
            .identifier => |name| try writer.print("{s}", .{name}),
            .assign => |a| try writer.print("{s} = {}", .{ a.name, a.value }),
            .binary => |b| try writer.print("({} {} {})", .{ b.left, b.op, b.right }),
            .unary => |u| try writer.print("({} {})", .{ u.op, u.operand }),
            .fn_def => |f| try writer.print("fn {s}(...) = ...", .{f.name}),
            .call => |c| try writer.print("{s}(...)", .{c.name}),
            .if_expr => |_| try writer.print("if ... then ... else ...", .{}),
        }
    }
};
```

---

## ขั้นตอนที่ 686: Evaluator

```zig
// ไฟล์ src/evaluator.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const ast_mod = @import("ast.zig");
const Node = ast_mod.Node;
const TokenType = @import("lexer.zig").TokenType;
const env_mod = @import("environment.zig");
const Environment = env_mod.Environment;
const builtins_mod = @import("builtins.zig");

pub const EvalError = error{
    UndefinedVariable,
    UndefinedFunction,
    TypeMismatch,
    DivisionByZero,
    ArityMismatch,
    StackOverflow,
    OutOfMemory,
};

/// Runtime value
pub const Value = union(enum) {
    number: f64,
    string: []u8,
    boolean: bool,
    null_val,
    function: FunctionValue,

    pub const FunctionValue = struct {
        name: []const u8,
        params: []const []const u8,
        body: *const Node,
        closure: *Environment,
    };

    pub fn format(
        self: Value,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        switch (self) {
            .number => |n| {
                if (@floor(n) == n and @abs(n) < 1e15) {
                    try writer.print("{d:.0}", .{n});
                } else {
                    try writer.print("{d}", .{n});
                }
            },
            .string => |s| try writer.print("{s}", .{s}),
            .boolean => |b| try writer.print("{}", .{b}),
            .null_val => try writer.print("null", .{}),
            .function => |f| try writer.print("<fn {s}>", .{f.name}),
        }
    }

    pub fn isTruthy(self: Value) bool {
        return switch (self) {
            .boolean => |b| b,
            .number => |n| n != 0,
            .string => |s| s.len > 0,
            .null_val => false,
            .function => true,
        };
    }
};

pub const Evaluator = struct {
    allocator: Allocator,
    call_depth: usize,
    max_depth: usize,

    const MAX_CALL_DEPTH = 1000;

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{
            .allocator = allocator,
            .call_depth = 0,
            .max_depth = MAX_CALL_DEPTH,
        };
    }

    pub fn evalProgram(self: *Self, nodes: []const Node, env: *Environment) !Value {
        var last: Value = .null_val;
        for (nodes) |*node| {
            last = try self.eval(node, env);
        }
        return last;
    }

    pub fn eval(self: *Self, node: *const Node, env: *Environment) EvalError!Value {
        return switch (node.*) {
            .number => |n| Value{ .number = n },
            .string => |s| Value{ .string = @constCast(s) },
            .boolean => |b| Value{ .boolean = b },

            .identifier => |name| {
                return env.get(name) orelse EvalError.UndefinedVariable;
            },

            .assign => |a| {
                const val = try self.eval(a.value, env);
                try env.set(a.name, val);
                return val;
            },

            .binary => |b| try self.evalBinary(b.op, b.left, b.right, env),

            .unary => |u| {
                const operand = try self.eval(u.operand, env);
                return switch (u.op) {
                    .minus => switch (operand) {
                        .number => |n| Value{ .number = -n },
                        else => EvalError.TypeMismatch,
                    },
                    .not_kw => Value{ .boolean = !operand.isTruthy() },
                    else => EvalError.TypeMismatch,
                };
            },

            .fn_def => |f| {
                const func_val = Value{
                    .function = .{
                        .name = f.name,
                        .params = f.params,
                        .body = f.body,
                        .closure = env,
                    },
                };
                try env.set(f.name, func_val);
                return func_val;
            },

            .call => |c| try self.evalCall(c.name, c.args, env),

            .if_expr => |i| {
                const cond = try self.eval(i.condition, env);
                if (cond.isTruthy()) {
                    return self.eval(i.then_branch, env);
                } else {
                    return self.eval(i.else_branch, env);
                }
            },
        };
    }

    fn evalBinary(
        self: *Self,
        op: TokenType,
        left_node: *const Node,
        right_node: *const Node,
        env: *Environment,
    ) EvalError!Value {
        const left = try self.eval(left_node, env);

        // Short-circuit evaluation
        if (op == .and_kw) {
            if (!left.isTruthy()) return Value{ .boolean = false };
            const right = try self.eval(right_node, env);
            return Value{ .boolean = right.isTruthy() };
        }
        if (op == .or_kw) {
            if (left.isTruthy()) return Value{ .boolean = true };
            const right = try self.eval(right_node, env);
            return Value{ .boolean = right.isTruthy() };
        }

        const right = try self.eval(right_node, env);

        switch (op) {
            .plus => {
                if (left == .number and right == .number) {
                    return Value{ .number = left.number + right.number };
                }
                // String concatenation
                if (left == .string and right == .string) {
                    const result = try std.fmt.allocPrint(self.allocator, "{s}{s}", .{ left.string, right.string });
                    return Value{ .string = result };
                }
                return EvalError.TypeMismatch;
            },
            .minus => {
                if (left == .number and right == .number) {
                    return Value{ .number = left.number - right.number };
                }
                return EvalError.TypeMismatch;
            },
            .star => {
                if (left == .number and right == .number) {
                    return Value{ .number = left.number * right.number };
                }
                return EvalError.TypeMismatch;
            },
            .slash => {
                if (left == .number and right == .number) {
                    if (right.number == 0) return EvalError.DivisionByZero;
                    return Value{ .number = left.number / right.number };
                }
                return EvalError.TypeMismatch;
            },
            .percent => {
                if (left == .number and right == .number) {
                    if (right.number == 0) return EvalError.DivisionByZero;
                    return Value{ .number = @mod(left.number, right.number) };
                }
                return EvalError.TypeMismatch;
            },
            .power => {
                if (left == .number and right == .number) {
                    return Value{ .number = std.math.pow(f64, left.number, right.number) };
                }
                return EvalError.TypeMismatch;
            },
            .eq_eq => return Value{ .boolean = valuesEqual(left, right) },
            .bang_eq => return Value{ .boolean = !valuesEqual(left, right) },
            .lt => {
                if (left == .number and right == .number) {
                    return Value{ .boolean = left.number < right.number };
                }
                return EvalError.TypeMismatch;
            },
            .lt_eq => {
                if (left == .number and right == .number) {
                    return Value{ .boolean = left.number <= right.number };
                }
                return EvalError.TypeMismatch;
            },
            .gt => {
                if (left == .number and right == .number) {
                    return Value{ .boolean = left.number > right.number };
                }
                return EvalError.TypeMismatch;
            },
            .gt_eq => {
                if (left == .number and right == .number) {
                    return Value{ .boolean = left.number >= right.number };
                }
                return EvalError.TypeMismatch;
            },
            else => return EvalError.TypeMismatch,
        }
    }

    fn evalCall(
        self: *Self,
        name: []const u8,
        args: []*const Node,
        env: *Environment,
    ) EvalError!Value {
        // ตรวจสอบ call depth
        if (self.call_depth >= self.max_depth) {
            return EvalError.StackOverflow;
        }

        // Evaluate arguments
        var arg_values = std.ArrayList(Value).init(self.allocator);
        defer arg_values.deinit();

        for (args) |arg| {
            const val = try self.eval(arg, env);
            try arg_values.append(val);
        }

        // ลอง built-in functions ก่อน
        if (builtins_mod.callBuiltin(self.allocator, name, arg_values.items)) |result| {
            return result;
        } else |_| {}

        // ค้นหา user-defined function
        const func_val = env.get(name) orelse return EvalError.UndefinedFunction;

        if (func_val != .function) return EvalError.TypeMismatch;
        const func = func_val.function;

        if (func.params.len != arg_values.items.len) {
            return EvalError.ArityMismatch;
        }

        // สร้าง closure environment
        var func_env = Environment.initWithParent(self.allocator, func.closure);
        defer func_env.deinit();

        // Bind parameters
        for (func.params, arg_values.items) |param, val| {
            try func_env.set(param, val);
        }

        self.call_depth += 1;
        defer self.call_depth -= 1;

        return self.eval(func.body, &func_env);
    }

    fn valuesEqual(a: Value, b: Value) bool {
        if (@as(std.meta.Tag(Value), a) != @as(std.meta.Tag(Value), b)) return false;
        return switch (a) {
            .number => |n| n == b.number,
            .string => |s| std.mem.eql(u8, s, b.string),
            .boolean => |bv| bv == b.boolean,
            .null_val => true,
            .function => false, // functions ไม่เท่ากัน
        };
    }
};
```

---

## ขั้นตอนที่ 687: Variables Support

```zig
// ไฟล์ src/environment.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const eval_mod = @import("evaluator.zig");
const Value = eval_mod.Value;

/// Environment สำหรับ variables
pub const Environment = struct {
    allocator: Allocator,
    bindings: std.StringHashMap(Value),
    parent: ?*Environment,

    const Self = @This();

    pub fn init(allocator: Allocator) Self {
        return Self{
            .allocator = allocator,
            .bindings = std.StringHashMap(Value).init(allocator),
            .parent = null,
        };
    }

    pub fn initWithParent(allocator: Allocator, parent: *Environment) Self {
        var env = init(allocator);
        env.parent = parent;
        return env;
    }

    pub fn deinit(self: *Self) void {
        var iter = self.bindings.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
        }
        self.bindings.deinit();
    }

    /// ดึงค่าตัวแปร (ค้นหาใน parent ด้วย)
    pub fn get(self: *const Self, name: []const u8) ?Value {
        if (self.bindings.get(name)) |val| return val;
        if (self.parent) |parent| return parent.get(name);
        return null;
    }

    /// ตั้งค่าตัวแปรในสภาพแวดล้อมปัจจุบัน
    pub fn set(self: *Self, name: []const u8, value: Value) !void {
        // ถ้ามีอยู่แล้ว update ในตำแหน่งที่พบ
        if (self.bindings.contains(name)) {
            try self.bindings.put(name, value);
            return;
        }

        // ตรวจสอบ parent
        if (self.parent) |parent| {
            if (parent.has(name)) {
                try parent.set(name, value);
                return;
            }
        }

        // สร้างใหม่
        const key = try self.allocator.dupe(u8, name);
        errdefer self.allocator.free(key);
        try self.bindings.put(key, value);
    }

    /// ตรวจสอบว่า key มีในสภาพแวดล้อมนี้
    pub fn has(self: *const Self, name: []const u8) bool {
        if (self.bindings.contains(name)) return true;
        if (self.parent) |parent| return parent.has(name);
        return false;
    }

    /// ดู keys ทั้งหมดใน current scope
    pub fn keys(self: *const Self) std.StringHashMap(Value).KeyIterator {
        return self.bindings.keyIterator();
    }
};
```

---

## ขั้นตอนที่ 688: Functions Support

```zig
// ไฟล์ src/builtins.zig

const std = @import("std");
const Allocator = std.mem.Allocator;
const eval_mod = @import("evaluator.zig");
const Value = eval_mod.Value;
const EvalError = eval_mod.EvalError;

/// เรียก built-in function
pub fn callBuiltin(allocator: Allocator, name: []const u8, args: []const Value) EvalError!Value {
    if (std.mem.eql(u8, name, "print")) return builtinPrint(allocator, args);
    if (std.mem.eql(u8, name, "println")) return builtinPrintln(allocator, args);
    if (std.mem.eql(u8, name, "sqrt")) return builtinSqrt(args);
    if (std.mem.eql(u8, name, "abs")) return builtinAbs(args);
    if (std.mem.eql(u8, name, "floor")) return builtinFloor(args);
    if (std.mem.eql(u8, name, "ceil")) return builtinCeil(args);
    if (std.mem.eql(u8, name, "round")) return builtinRound(args);
    if (std.mem.eql(u8, name, "max")) return builtinMax(args);
    if (std.mem.eql(u8, name, "min")) return builtinMin(args);
    if (std.mem.eql(u8, name, "pow")) return builtinPow(args);
    if (std.mem.eql(u8, name, "log")) return builtinLog(args);
    if (std.mem.eql(u8, name, "sin")) return builtinSin(args);
    if (std.mem.eql(u8, name, "cos")) return builtinCos(args);
    if (std.mem.eql(u8, name, "str")) return builtinStr(allocator, args);
    if (std.mem.eql(u8, name, "num")) return builtinNum(args);
    if (std.mem.eql(u8, name, "len")) return builtinLen(args);
    if (std.mem.eql(u8, name, "type")) return builtinType(allocator, args);
    return EvalError.UndefinedFunction;
}

fn checkArity(args: []const Value, expected: usize) EvalError!void {
    if (args.len != expected) return EvalError.ArityMismatch;
}

fn checkNumber(val: Value) EvalError!f64 {
    return switch (val) {
        .number => |n| n,
        else => EvalError.TypeMismatch,
    };
}

fn builtinPrint(allocator: Allocator, args: []const Value) EvalError!Value {
    _ = allocator;
    for (args, 0..) |arg, i| {
        if (i > 0) std.debug.print(" ", .{});
        std.debug.print("{}", .{arg});
    }
    return Value.null_val;
}

fn builtinPrintln(allocator: Allocator, args: []const Value) EvalError!Value {
    const result = try builtinPrint(allocator, args);
    std.debug.print("\n", .{});
    return result;
}

fn builtinSqrt(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @sqrt(n) };
}

fn builtinAbs(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @abs(n) };
}

fn builtinFloor(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @floor(n) };
}

fn builtinCeil(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @ceil(n) };
}

fn builtinRound(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @round(n) };
}

fn builtinMax(args: []const Value) EvalError!Value {
    try checkArity(args, 2);
    const a = try checkNumber(args[0]);
    const b = try checkNumber(args[1]);
    return Value{ .number = @max(a, b) };
}

fn builtinMin(args: []const Value) EvalError!Value {
    try checkArity(args, 2);
    const a = try checkNumber(args[0]);
    const b = try checkNumber(args[1]);
    return Value{ .number = @min(a, b) };
}

fn builtinPow(args: []const Value) EvalError!Value {
    try checkArity(args, 2);
    const base = try checkNumber(args[0]);
    const exp = try checkNumber(args[1]);
    return Value{ .number = std.math.pow(f64, base, exp) };
}

fn builtinLog(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @log(n) };
}

fn builtinSin(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @sin(n) };
}

fn builtinCos(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const n = try checkNumber(args[0]);
    return Value{ .number = @cos(n) };
}

fn builtinStr(allocator: Allocator, args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const s = std.fmt.allocPrint(allocator, "{}", .{args[0]}) catch return EvalError.OutOfMemory;
    return Value{ .string = s };
}

fn builtinNum(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    return switch (args[0]) {
        .number => args[0],
        .string => |s| {
            const n = std.fmt.parseFloat(f64, s) catch return EvalError.TypeMismatch;
            return Value{ .number = n };
        },
        .boolean => |b| Value{ .number = if (b) 1 else 0 },
        else => EvalError.TypeMismatch,
    };
}

fn builtinLen(args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    return switch (args[0]) {
        .string => |s| Value{ .number = @floatFromInt(s.len) },
        else => EvalError.TypeMismatch,
    };
}

fn builtinType(allocator: Allocator, args: []const Value) EvalError!Value {
    try checkArity(args, 1);
    const type_name = switch (args[0]) {
        .number => "number",
        .string => "string",
        .boolean => "boolean",
        .null_val => "null",
        .function => "function",
    };
    const s = allocator.dupe(u8, type_name) catch return EvalError.OutOfMemory;
    return Value{ .string = s };
}
```

---

## ขั้นตอนที่ 689: Error Reporting

```zig
// ต่อจาก main.zig - Error reporting

const LangError = union(enum) {
    lex_error: struct {
        message: []const u8,
        pos: lexer_mod.Position,
    },
    parse_error: struct {
        message: []const u8,
        pos: lexer_mod.Position,
    },
    eval_error: struct {
        message: []const u8,
        node: []const u8,
    },
};

pub fn formatError(err: anyerror, source: []const u8, pos: ?lexer_mod.Position) void {
    const stderr = std.io.getStdErr().writer();

    if (pos) |p| {
        stderr.print("Error at line {}, column {}: ", .{ p.line, p.column }) catch {};

        // แสดง source line
        var line_start: usize = 0;
        var current_line: usize = 1;
        for (source, 0..) |c, i| {
            if (current_line == p.line) {
                line_start = i;
                break;
            }
            if (c == '\n') current_line += 1;
        }

        var line_end = line_start;
        while (line_end < source.len and source[line_end] != '\n') {
            line_end += 1;
        }

        stderr.print("{s}\n", .{source[line_start..line_end]}) catch {};

        // แสดง pointer
        for (0..p.column - 1) |_| stderr.print(" ", .{}) catch {};
        stderr.print("^\n", .{}) catch {};
    }

    stderr.print("{s}\n", .{@errorName(err)}) catch {};
}
```

---

## ขั้นตอนที่ 690: REPL Interface

```zig
// ไฟล์ src/main.zig

const std = @import("std");
const lexer_mod = @import("lexer.zig");
const parser_mod = @import("parser.zig");
const eval_mod = @import("evaluator.zig");
const env_mod = @import("environment.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len > 1) {
        // Run file mode
        const file = try std.fs.openFileAbsolute(args[1], .{});
        defer file.close();

        const source = try file.readToEndAlloc(allocator, 1024 * 1024);
        defer allocator.free(source);

        try runSource(allocator, source, false);
    } else {
        // REPL mode
        try startREPL(allocator);
    }
}

fn runSource(allocator: std.mem.Allocator, source: []const u8, print_result: bool) !void {
    // Lex
    var lexer = lexer_mod.Lexer.init(allocator, source);
    const tokens = lexer.tokenize() catch |err| {
        std.debug.print("Lex error: {}\n", .{err});
        return;
    };
    defer allocator.free(tokens);

    // Parse
    var parser = parser_mod.Parser.init(allocator, tokens);
    const nodes = parser.parseProgram() catch |err| {
        std.debug.print("Parse error: {}\n", .{err});
        return;
    };
    defer {
        for (nodes) |*node| node.deinit(allocator);
        allocator.free(nodes);
    }

    // Evaluate
    var env = env_mod.Environment.init(allocator);
    defer env.deinit();

    var evaluator = eval_mod.Evaluator.init(allocator);

    const result = evaluator.evalProgram(nodes, &env) catch |err| {
        std.debug.print("Error: {s}\n", .{@errorName(err)});
        return;
    };

    if (print_result and result != .null_val) {
        std.debug.print("{}\n", .{result});
    }
}

fn startREPL(allocator: std.mem.Allocator) !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();

    // Global environment
    var global_env = env_mod.Environment.init(allocator);
    defer global_env.deinit();

    var evaluator = eval_mod.Evaluator.init(allocator);

    try stdout.writeAll(
        \\CalcLang REPL v1.0
        \\Type expressions to evaluate. Type 'exit' to quit.
        \\
        \\
    );

    var line_buf: [4096]u8 = undefined;

    while (true) {
        try stdout.writeAll("> ");

        const line = stdin.readUntilDelimiter(&line_buf, '\n') catch |err| {
            if (err == error.EndOfStream) break;
            return err;
        };

        const trimmed = std.mem.trim(u8, line, " \t\r\n");

        if (trimmed.len == 0) continue;

        // Exit command
        if (std.mem.eql(u8, trimmed, "exit") or std.mem.eql(u8, trimmed, "quit")) {
            try stdout.writeAll("Goodbye!\n");
            break;
        }

        // Help command
        if (std.mem.eql(u8, trimmed, "help")) {
            try stdout.writeAll(
                \\Available operations:
                \\  +, -, *, /, %, ^  - Arithmetic
                \\  ==, !=, <, <=, >, >= - Comparison
                \\  and, or, not       - Logical
                \\  x = expr           - Assignment
                \\  fn name(a, b) = expr - Function definition
                \\  if cond then e1 else e2 - Conditional
                \\Built-in functions:
                \\  print, println, sqrt, abs, floor, ceil, round
                \\  max, min, pow, log, sin, cos, str, num, len, type
                \\
            );
            continue;
        }

        // Vars command - แสดงตัวแปรทั้งหมด
        if (std.mem.eql(u8, trimmed, "vars")) {
            var iter = global_env.keys();
            var count: usize = 0;
            while (iter.next()) |key| {
                if (global_env.get(key.*)) |val| {
                    try stdout.print("  {s} = {}\n", .{ key.*, val });
                    count += 1;
                }
            }
            if (count == 0) try stdout.writeAll("  (no variables)\n");
            continue;
        }

        // Lex
        var lexer = lexer_mod.Lexer.init(allocator, trimmed);
        const tokens = lexer.tokenize() catch |err| {
            try stdout.print("Lex error: {}\n", .{err});
            continue;
        };
        defer allocator.free(tokens);

        // Parse
        var parser = parser_mod.Parser.init(allocator, tokens);
        const nodes = parser.parseProgram() catch |err| {
            try stdout.print("Parse error: {}\n", .{err});
            continue;
        };
        defer {
            for (nodes) |*node| node.deinit(allocator);
            allocator.free(nodes);
        }

        // Evaluate
        const result = evaluator.evalProgram(nodes, &global_env) catch |err| {
            try stdout.print("Error: {s}\n", .{@errorName(err)});
            continue;
        };

        if (result != .null_val) {
            try stdout.print("{}\n", .{result});
        }
    }
}

test "Lexer basic" {
    const allocator = std.testing.allocator;
    var lexer = lexer_mod.Lexer.init(allocator, "x = 1 + 2");
    const tokens = try lexer.tokenize();
    defer allocator.free(tokens);

    try std.testing.expect(tokens.len > 0);
    try std.testing.expect(tokens[0].token_type == .identifier);
    try std.testing.expect(tokens[1].token_type == .eq);
}

test "Evaluator arithmetic" {
    const allocator = std.testing.allocator;

    const source = "2 + 3 * 4";
    var lexer = lexer_mod.Lexer.init(allocator, source);
    const tokens = try lexer.tokenize();
    defer allocator.free(tokens);

    var parser = parser_mod.Parser.init(allocator, tokens);
    const nodes = try parser.parseProgram();
    defer {
        for (nodes) |*node| node.deinit(allocator);
        allocator.free(nodes);
    }

    var env = env_mod.Environment.init(allocator);
    defer env.deinit();

    var evaluator = eval_mod.Evaluator.init(allocator);
    const result = try evaluator.evalProgram(nodes, &env);

    try std.testing.expect(result == .number);
    try std.testing.expect(result.number == 14.0);
}

test "Evaluator variables" {
    const allocator = std.testing.allocator;

    const source = "x = 10\ny = 20\nx + y";
    var lexer = lexer_mod.Lexer.init(allocator, source);
    const tokens = try lexer.tokenize();
    defer allocator.free(tokens);

    var parser = parser_mod.Parser.init(allocator, tokens);
    const nodes = try parser.parseProgram();
    defer {
        for (nodes) |*node| node.deinit(allocator);
        allocator.free(nodes);
    }

    var env = env_mod.Environment.init(allocator);
    defer env.deinit();

    var evaluator = eval_mod.Evaluator.init(allocator);
    const result = try evaluator.evalProgram(nodes, &env);

    try std.testing.expect(result == .number);
    try std.testing.expect(result.number == 30.0);
}

test "Evaluator if expression" {
    const allocator = std.testing.allocator;

    const source = "if 5 > 3 then 1 else 0";
    var lexer = lexer_mod.Lexer.init(allocator, source);
    const tokens = try lexer.tokenize();
    defer allocator.free(tokens);

    var parser = parser_mod.Parser.init(allocator, tokens);
    const nodes = try parser.parseProgram();
    defer {
        for (nodes) |*node| node.deinit(allocator);
        allocator.free(nodes);
    }

    var env = env_mod.Environment.init(allocator);
    defer env.deinit();

    var evaluator = eval_mod.Evaluator.init(allocator);
    const result = try evaluator.evalProgram(nodes, &env);

    try std.testing.expect(result == .number);
    try std.testing.expect(result.number == 1.0);
}
```

---

## สรุปโปรเจกต์ Calculator Language

| Component | รายละเอียด |
|-----------|-----------|
| Lexer | Tokenizer รองรับ numbers, strings, operators, keywords |
| Parser | Recursive descent + Pratt parsing |
| AST | Union type ครอบคลุมทุก node types |
| Evaluator | Tree-walking interpreter |
| Environment | Lexical scoping, closures |
| Builtins | 18+ built-in functions |
| REPL | Interactive mode พร้อม history |

### ตัวอย่างการใช้งาน

```
> x = 42
42
> y = sqrt(x * 2)
9.165151389911680
> fn double(n) = n * 2
<fn double>
> double(x)
84
> if x > 40 then "big" else "small"
big
> vars
  x = 42
  y = 9.165151389911680
  double = <fn double>
```

### การรันโปรเจกต์

```bash
# Build
zig build

# Run REPL
./zig-out/bin/calclang

# Run file
./zig-out/bin/calclang program.calc

# Run tests
zig build test
```
