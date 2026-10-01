# ส่วนที่ 86: Parsing - Parser Implementation

## ขั้นตอนที่ 851-860: การสร้าง Parser ใน Zig

Parser คือส่วนสำคัญของ compiler หรือ interpreter ที่แปลง token stream ให้กลายเป็นโครงสร้างข้อมูลที่มีความหมาย (เช่น AST) ในส่วนนี้เราจะสร้าง parser ที่สมบูรณ์โดยใช้ Zig

---

## Step 851: Parser Concepts - Recursive Descent

### แนวคิดพื้นฐานของ Recursive Descent Parser

Recursive Descent Parser เป็นเทคนิคการเขียน parser ที่ง่ายและอ่านได้ง่าย โดยแต่ละ grammar rule จะมี function ของตัวเอง

```
Grammar ตัวอย่าง:
expression  = term (('+' | '-') term)*
term        = factor (('*' | '/') factor)*
factor      = NUMBER | '(' expression ')'
```

```zig
// ตัวอย่าง: แนวคิด Recursive Descent
// แต่ละ rule ใน grammar จะมี function ที่สอดคล้องกัน

const std = @import("std");

// Token types ที่เราจะใช้
pub const TokenType = enum {
    // Literals
    number,
    string,
    identifier,
    
    // Operators
    plus,       // +
    minus,      // -
    star,       // *
    slash,      // /
    percent,    // %
    
    // Comparison
    equal_equal,    // ==
    bang_equal,     // !=
    less,           // <
    less_equal,     // <=
    greater,        // >
    greater_equal,  // >=
    
    // Logical
    amp_amp,        // &&
    pipe_pipe,      // ||
    bang,           // !
    
    // Assignment
    equal,          // =
    
    // Delimiters
    left_paren,     // (
    right_paren,    // )
    left_brace,     // {
    right_brace,    // }
    left_bracket,   // [
    right_bracket,  // ]
    semicolon,      // ;
    colon,          // :
    comma,          // ,
    dot,            // .
    
    // Keywords
    keyword_if,
    keyword_else,
    keyword_while,
    keyword_for,
    keyword_fn,
    keyword_return,
    keyword_var,
    keyword_const,
    keyword_true,
    keyword_false,
    keyword_null,
    
    // Special
    eof,
    invalid,
};

pub const Token = struct {
    type: TokenType,
    lexeme: []const u8,  // ข้อความดั้งเดิม
    line: usize,
    column: usize,
    
    pub fn format(
        self: Token,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("Token({s}, '{s}', L{d}:C{d})", .{
            @tagName(self.type),
            self.lexeme,
            self.line,
            self.column,
        });
    }
};

// Lexer อย่างง่ายสำหรับทดสอบ
pub const Lexer = struct {
    source: []const u8,
    pos: usize,
    line: usize,
    column: usize,
    
    pub fn init(source: []const u8) Lexer {
        return Lexer{
            .source = source,
            .pos = 0,
            .line = 1,
            .column = 1,
        };
    }
    
    fn peek(self: *Lexer) ?u8 {
        if (self.pos >= self.source.len) return null;
        return self.source[self.pos];
    }
    
    fn advance(self: *Lexer) ?u8 {
        if (self.pos >= self.source.len) return null;
        const ch = self.source[self.pos];
        self.pos += 1;
        if (ch == '\n') {
            self.line += 1;
            self.column = 1;
        } else {
            self.column += 1;
        }
        return ch;
    }
    
    fn skipWhitespace(self: *Lexer) void {
        while (self.peek()) |ch| {
            if (ch == ' ' or ch == '\t' or ch == '\n' or ch == '\r') {
                _ = self.advance();
            } else break;
        }
    }
    
    pub fn nextToken(self: *Lexer) Token {
        self.skipWhitespace();
        
        const start = self.pos;
        const start_line = self.line;
        const start_col = self.column;
        
        const ch = self.advance() orelse {
            return Token{
                .type = .eof,
                .lexeme = "",
                .line = start_line,
                .column = start_col,
            };
        };
        
        const token_type: TokenType = switch (ch) {
            '+' => .plus,
            '-' => .minus,
            '*' => .star,
            '/' => .slash,
            '%' => .percent,
            '(' => .left_paren,
            ')' => .right_paren,
            '{' => .left_brace,
            '}' => .right_brace,
            '[' => .left_bracket,
            ']' => .right_bracket,
            ';' => .semicolon,
            ':' => .colon,
            ',' => .comma,
            '.' => .dot,
            '=' => if (self.peek() == '=') blk: {
                _ = self.advance();
                break :blk .equal_equal;
            } else .equal,
            '!' => if (self.peek() == '=') blk: {
                _ = self.advance();
                break :blk .bang_equal;
            } else .bang,
            '<' => if (self.peek() == '=') blk: {
                _ = self.advance();
                break :blk .less_equal;
            } else .less,
            '>' => if (self.peek() == '=') blk: {
                _ = self.advance();
                break :blk .greater_equal;
            } else .greater,
            '&' => if (self.peek() == '&') blk: {
                _ = self.advance();
                break :blk .amp_amp;
            } else .invalid,
            '|' => if (self.peek() == '|') blk: {
                _ = self.advance();
                break :blk .pipe_pipe;
            } else .invalid,
            '0'...'9' => blk: {
                while (self.peek()) |c| {
                    if (c >= '0' and c <= '9') {
                        _ = self.advance();
                    } else break;
                }
                break :blk .number;
            },
            'a'...'z', 'A'...'Z', '_' => blk: {
                while (self.peek()) |c| {
                    if ((c >= 'a' and c <= 'z') or
                        (c >= 'A' and c <= 'Z') or
                        (c >= '0' and c <= '9') or
                        c == '_') {
                        _ = self.advance();
                    } else break;
                }
                const lexeme = self.source[start..self.pos];
                break :blk if (std.mem.eql(u8, lexeme, "if")) .keyword_if
                    else if (std.mem.eql(u8, lexeme, "else")) .keyword_else
                    else if (std.mem.eql(u8, lexeme, "while")) .keyword_while
                    else if (std.mem.eql(u8, lexeme, "for")) .keyword_for
                    else if (std.mem.eql(u8, lexeme, "fn")) .keyword_fn
                    else if (std.mem.eql(u8, lexeme, "return")) .keyword_return
                    else if (std.mem.eql(u8, lexeme, "var")) .keyword_var
                    else if (std.mem.eql(u8, lexeme, "const")) .keyword_const
                    else if (std.mem.eql(u8, lexeme, "true")) .keyword_true
                    else if (std.mem.eql(u8, lexeme, "false")) .keyword_false
                    else if (std.mem.eql(u8, lexeme, "null")) .keyword_null
                    else .identifier;
            },
            else => .invalid,
        };
        
        return Token{
            .type = token_type,
            .lexeme = self.source[start..self.pos],
            .line = start_line,
            .column = start_col,
        };
    }
    
    pub fn tokenize(self: *Lexer, allocator: std.mem.Allocator) ![]Token {
        var tokens = std.ArrayList(Token).init(allocator);
        defer tokens.deinit();
        
        while (true) {
            const token = self.nextToken();
            try tokens.append(token);
            if (token.type == .eof) break;
        }
        
        return tokens.toOwnedSlice();
    }
};

test "lexer tokenizes basic expression" {
    const source = "1 + 2 * 3";
    var lexer = Lexer.init(source);
    
    const allocator = std.testing.allocator;
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    try std.testing.expectEqual(@as(usize, 6), tokens.len); // 5 tokens + EOF
    try std.testing.expectEqual(TokenType.number, tokens[0].type);
    try std.testing.expectEqual(TokenType.plus, tokens[1].type);
    try std.testing.expectEqual(TokenType.number, tokens[2].type);
    try std.testing.expectEqual(TokenType.star, tokens[3].type);
    try std.testing.expectEqual(TokenType.number, tokens[4].type);
    try std.testing.expectEqual(TokenType.eof, tokens[5].type);
}
```

---

## Step 852: Parser Struct

### โครงสร้างของ Parser

```zig
// parser.zig - โครงสร้างหลักของ Parser

const std = @import("std");
const Token = @import("lexer.zig").Token;
const TokenType = @import("lexer.zig").TokenType;

pub const ParseError = error{
    UnexpectedToken,
    ExpectedExpression,
    ExpectedSemicolon,
    ExpectedIdentifier,
    ExpectedLeftParen,
    ExpectedRightParen,
    ExpectedLeftBrace,
    ExpectedRightBrace,
    OutOfMemory,
};

pub const Parser = struct {
    tokens: []const Token,   // Token stream ทั้งหมด
    current: usize,           // ตำแหน่งปัจจุบัน
    allocator: std.mem.Allocator,
    
    // สำหรับเก็บ error messages
    errors: std.ArrayList([]const u8),
    
    pub fn init(tokens: []const Token, allocator: std.mem.Allocator) Parser {
        return Parser{
            .tokens = tokens,
            .current = 0,
            .allocator = allocator,
            .errors = std.ArrayList([]const u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *Parser) void {
        for (self.errors.items) |err| {
            self.allocator.free(err);
        }
        self.errors.deinit();
    }
    
    // ดู token ปัจจุบันโดยไม่เลื่อน position
    pub fn peek(self: *Parser) Token {
        return self.tokens[self.current];
    }
    
    // ดู token ถัดไปโดยไม่เลื่อน position
    pub fn peekNext(self: *Parser) ?Token {
        if (self.current + 1 >= self.tokens.len) return null;
        return self.tokens[self.current + 1];
    }
    
    // เช็คว่า token ปัจจุบันเป็น type ที่ต้องการหรือไม่
    pub fn check(self: *Parser, token_type: TokenType) bool {
        if (self.isAtEnd()) return false;
        return self.peek().type == token_type;
    }
    
    // เลื่อน position ไปข้างหน้า
    pub fn advance(self: *Parser) Token {
        if (!self.isAtEnd()) self.current += 1;
        return self.previous();
    }
    
    // ได้ token ก่อนหน้า
    pub fn previous(self: *Parser) Token {
        return self.tokens[self.current - 1];
    }
    
    // เช็คว่าถึง EOF แล้วหรือยัง
    pub fn isAtEnd(self: *Parser) bool {
        return self.peek().type == .eof;
    }
    
    // ถ้า match แล้วเลื่อนไป, ถ้าไม่ match คงที่
    pub fn match(self: *Parser, types: []const TokenType) bool {
        for (types) |t| {
            if (self.check(t)) {
                _ = self.advance();
                return true;
            }
        }
        return false;
    }
    
    // ต้องการ token type นี้, ถ้าไม่ใช่ให้ error
    pub fn consume(self: *Parser, token_type: TokenType, message: []const u8) ParseError!Token {
        if (self.check(token_type)) return self.advance();
        
        const token = self.peek();
        const err_msg = try std.fmt.allocPrint(
            self.allocator,
            "Line {d}: {s}, got '{s}'",
            .{ token.line, message, token.lexeme },
        );
        try self.errors.append(err_msg);
        return ParseError.UnexpectedToken;
    }
    
    // รายงาน error โดยไม่หยุด parsing
    pub fn reportError(self: *Parser, token: Token, message: []const u8) void {
        const err_msg = std.fmt.allocPrint(
            self.allocator,
            "Line {d}: {s}",
            .{ token.line, message },
        ) catch return;
        self.errors.append(err_msg) catch {};
    }
    
    // Synchronize หลัง error (สำหรับ error recovery)
    pub fn synchronize(self: *Parser) void {
        _ = self.advance();
        
        while (!self.isAtEnd()) {
            if (self.previous().type == .semicolon) return;
            
            switch (self.peek().type) {
                .keyword_fn,
                .keyword_var,
                .keyword_const,
                .keyword_if,
                .keyword_while,
                .keyword_for,
                .keyword_return,
                => return,
                else => {},
            }
            
            _ = self.advance();
        }
    }
    
    pub fn hasErrors(self: *Parser) bool {
        return self.errors.items.len > 0;
    }
    
    pub fn printErrors(self: *Parser) void {
        for (self.errors.items) |err| {
            std.debug.print("Parse Error: {s}\n", .{err});
        }
    }
};
```

---

## Step 853: Expression Parsing - Pratt Parsing

### Pratt Parser คือ Top-Down Operator Precedence Parser

```zig
// pratt_parser.zig - การ implement Pratt Parser

const std = @import("std");

// Operator Precedence (ลำดับความสำคัญ)
pub const Precedence = enum(u8) {
    none = 0,
    assignment = 1,   // =
    or_prec = 2,      // ||
    and_prec = 3,     // &&
    equality = 4,     // == !=
    comparison = 5,   // < > <= >=
    term = 6,         // + -
    factor = 7,       // * /
    unary = 8,        // ! -
    call = 9,         // . ()
    primary = 10,
    
    pub fn higher(self: Precedence) Precedence {
        const val = @intFromEnum(self);
        if (val + 1 > @intFromEnum(Precedence.primary)) return .primary;
        return @enumFromInt(val + 1);
    }
};

// Node types สำหรับ AST Expression
pub const ExprNode = union(enum) {
    number: f64,
    string: []const u8,
    boolean: bool,
    null_val: void,
    identifier: []const u8,
    
    binary: struct {
        left: *ExprNode,
        op: []const u8,
        right: *ExprNode,
    },
    
    unary: struct {
        op: []const u8,
        operand: *ExprNode,
    },
    
    call: struct {
        callee: *ExprNode,
        args: []*ExprNode,
    },
    
    index: struct {
        object: *ExprNode,
        idx: *ExprNode,
    },
    
    field_access: struct {
        object: *ExprNode,
        field: []const u8,
    },
    
    assign: struct {
        target: *ExprNode,
        value: *ExprNode,
    },
    
    group: *ExprNode,
};

// Pratt Parser สำหรับ Expressions
pub const ExprParser = struct {
    const Self = @This();
    
    // Function pointer types
    const PrefixFn = *const fn (self: *Self) ParseError!*ExprNode;
    const InfixFn = *const fn (self: *Self, left: *ExprNode) ParseError!*ExprNode;
    
    const ParseRule = struct {
        prefix: ?PrefixFn,
        infix: ?InfixFn,
        precedence: Precedence,
    };
    
    tokens: []const Token,
    current: usize,
    allocator: std.mem.Allocator,
    
    pub fn init(tokens: []const Token, allocator: std.mem.Allocator) Self {
        return Self{
            .tokens = tokens,
            .current = 0,
            .allocator = allocator,
        };
    }
    
    fn peek(self: *Self) Token {
        return self.tokens[self.current];
    }
    
    fn advance(self: *Self) Token {
        const tok = self.tokens[self.current];
        if (self.current < self.tokens.len - 1) self.current += 1;
        return tok;
    }
    
    fn check(self: *Self, t: TokenType) bool {
        return self.peek().type == t;
    }
    
    // หา parse rule สำหรับ token type นี้
    fn getRule(token_type: TokenType) ParseRule {
        return switch (token_type) {
            .number => .{ .prefix = parseNumber, .infix = null, .precedence = .none },
            .string => .{ .prefix = parseString, .infix = null, .precedence = .none },
            .keyword_true, .keyword_false => .{ .prefix = parseBool, .infix = null, .precedence = .none },
            .keyword_null => .{ .prefix = parseNull, .infix = null, .precedence = .none },
            .identifier => .{ .prefix = parseIdentifier, .infix = null, .precedence = .none },
            
            .left_paren => .{ .prefix = parseGrouping, .infix = parseCall, .precedence = .call },
            .left_bracket => .{ .prefix = parseArray, .infix = parseIndex, .precedence = .call },
            .dot => .{ .prefix = null, .infix = parseFieldAccess, .precedence = .call },
            
            .minus => .{ .prefix = parseUnary, .infix = parseBinary, .precedence = .term },
            .bang => .{ .prefix = parseUnary, .infix = null, .precedence = .none },
            
            .plus => .{ .prefix = null, .infix = parseBinary, .precedence = .term },
            .star => .{ .prefix = null, .infix = parseBinary, .precedence = .factor },
            .slash => .{ .prefix = null, .infix = parseBinary, .precedence = .factor },
            .percent => .{ .prefix = null, .infix = parseBinary, .precedence = .factor },
            
            .equal_equal, .bang_equal => .{ .prefix = null, .infix = parseBinary, .precedence = .equality },
            .less, .less_equal, .greater, .greater_equal => .{ .prefix = null, .infix = parseBinary, .precedence = .comparison },
            
            .amp_amp => .{ .prefix = null, .infix = parseBinary, .precedence = .and_prec },
            .pipe_pipe => .{ .prefix = null, .infix = parseBinary, .precedence = .or_prec },
            
            .equal => .{ .prefix = null, .infix = parseAssign, .precedence = .assignment },
            
            else => .{ .prefix = null, .infix = null, .precedence = .none },
        };
    }
    
    // Parse expression ด้วย Pratt algorithm
    pub fn parseExpression(self: *Self, min_prec: Precedence) !*ExprNode {
        const token = self.advance();
        const rule = getRule(token.type);
        
        const prefix_fn = rule.prefix orelse {
            return error.ExpectedExpression;
        };
        
        var left = try prefix_fn(self);
        
        while (true) {
            const next_rule = getRule(self.peek().type);
            if (@intFromEnum(next_rule.precedence) <= @intFromEnum(min_prec)) break;
            
            const op_token = self.advance();
            const infix_rule = getRule(op_token.type);
            const infix_fn = infix_rule.infix orelse break;
            
            // ต้อง backup ก่อนเรียก infix
            self.current -= 1; // ย้อนกลับไปที่ op token
            _ = self.advance(); // advance อีกครั้ง
            
            left = try infix_fn(self, left);
        }
        
        return left;
    }
    
    // Prefix parsers
    fn parseNumber(self: *Self) !*ExprNode {
        const token = self.tokens[self.current - 1];
        const value = try std.fmt.parseFloat(f64, token.lexeme);
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .number = value };
        return node;
    }
    
    fn parseString(self: *Self) !*ExprNode {
        const token = self.tokens[self.current - 1];
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .string = token.lexeme };
        return node;
    }
    
    fn parseBool(self: *Self) !*ExprNode {
        const token = self.tokens[self.current - 1];
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .boolean = token.type == .keyword_true };
        return node;
    }
    
    fn parseNull(self: *Self) !*ExprNode {
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .null_val = {} };
        return node;
    }
    
    fn parseIdentifier(self: *Self) !*ExprNode {
        const token = self.tokens[self.current - 1];
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .identifier = token.lexeme };
        return node;
    }
    
    fn parseGrouping(self: *Self) !*ExprNode {
        const expr = try self.parseExpression(.none);
        if (!self.check(.right_paren)) return error.UnexpectedToken;
        _ = self.advance();
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .group = expr };
        return node;
    }
    
    fn parseUnary(self: *Self) !*ExprNode {
        const op_token = self.tokens[self.current - 1];
        const operand = try self.parseExpression(.unary);
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .unary = .{ .op = op_token.lexeme, .operand = operand } };
        return node;
    }
    
    fn parseArray(self: *Self) !*ExprNode {
        _ = self;
        // TODO: implement array literal parsing
        return error.ExpectedExpression;
    }
    
    // Infix parsers
    fn parseBinary(self: *Self, left: *ExprNode) !*ExprNode {
        const op_token = self.tokens[self.current - 1];
        const rule = getRule(op_token.type);
        const right = try self.parseExpression(rule.precedence);
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .binary = .{ .left = left, .op = op_token.lexeme, .right = right } };
        return node;
    }
    
    fn parseCall(self: *Self, callee: *ExprNode) !*ExprNode {
        var args = std.ArrayList(*ExprNode).init(self.allocator);
        
        if (!self.check(.right_paren)) {
            while (true) {
                const arg = try self.parseExpression(.none);
                try args.append(arg);
                if (!self.check(.comma)) break;
                _ = self.advance(); // consume ','
            }
        }
        
        if (!self.check(.right_paren)) return error.UnexpectedToken;
        _ = self.advance();
        
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .call = .{
            .callee = callee,
            .args = try args.toOwnedSlice(),
        }};
        return node;
    }
    
    fn parseIndex(self: *Self, object: *ExprNode) !*ExprNode {
        const idx = try self.parseExpression(.none);
        if (!self.check(.right_bracket)) return error.UnexpectedToken;
        _ = self.advance();
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .index = .{ .object = object, .idx = idx } };
        return node;
    }
    
    fn parseFieldAccess(self: *Self, object: *ExprNode) !*ExprNode {
        if (!self.check(.identifier)) return error.ExpectedIdentifier;
        const field_token = self.advance();
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .field_access = .{ .object = object, .field = field_token.lexeme } };
        return node;
    }
    
    fn parseAssign(self: *Self, target: *ExprNode) !*ExprNode {
        const value = try self.parseExpression(.none);
        const node = try self.allocator.create(ExprNode);
        node.* = .{ .assign = .{ .target = target, .value = value } };
        return node;
    }
};
```

---

## Step 854: Statement Parsing

```zig
// statement_parser.zig - การ parse statements

pub const StmtNode = union(enum) {
    expression_stmt: *ExprNode,
    
    var_decl: struct {
        name: []const u8,
        is_const: bool,
        type_annotation: ?[]const u8,
        initializer: ?*ExprNode,
    },
    
    block: []*StmtNode,
    
    if_stmt: struct {
        condition: *ExprNode,
        then_branch: *StmtNode,
        else_branch: ?*StmtNode,
    },
    
    while_stmt: struct {
        condition: *ExprNode,
        body: *StmtNode,
    },
    
    for_stmt: struct {
        init: ?*StmtNode,
        condition: ?*ExprNode,
        update: ?*ExprNode,
        body: *StmtNode,
    },
    
    return_stmt: ?*ExprNode,
    
    fn_decl: struct {
        name: []const u8,
        params: []Param,
        return_type: ?[]const u8,
        body: *StmtNode,
    },
    
    pub const Param = struct {
        name: []const u8,
        type_name: []const u8,
    };
};

pub fn parseStatement(parser: *Parser) ParseError!*StmtNode {
    const token = parser.peek();
    
    return switch (token.type) {
        .keyword_var, .keyword_const => parseVarDecl(parser),
        .keyword_if => parseIfStmt(parser),
        .keyword_while => parseWhileStmt(parser),
        .keyword_for => parseForStmt(parser),
        .keyword_fn => parseFnDecl(parser),
        .keyword_return => parseReturnStmt(parser),
        .left_brace => parseBlock(parser),
        else => parseExpressionStmt(parser),
    };
}

fn parseExpressionStmt(parser: *Parser) ParseError!*StmtNode {
    const expr = try parseExpr(parser);
    _ = try parser.consume(.semicolon, "Expected ';' after expression");
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .expression_stmt = expr };
    return node;
}

fn parseVarDecl(parser: *Parser) ParseError!*StmtNode {
    const is_const = parser.peek().type == .keyword_const;
    _ = parser.advance(); // consume 'var' or 'const'
    
    const name_token = try parser.consume(.identifier, "Expected variable name");
    
    var type_annotation: ?[]const u8 = null;
    if (parser.check(.colon)) {
        _ = parser.advance(); // consume ':'
        const type_token = try parser.consume(.identifier, "Expected type name");
        type_annotation = type_token.lexeme;
    }
    
    var initializer: ?*ExprNode = null;
    if (parser.check(.equal)) {
        _ = parser.advance(); // consume '='
        initializer = try parseExpr(parser);
    }
    
    _ = try parser.consume(.semicolon, "Expected ';' after variable declaration");
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .var_decl = .{
        .name = name_token.lexeme,
        .is_const = is_const,
        .type_annotation = type_annotation,
        .initializer = initializer,
    }};
    return node;
}

fn parseBlock(parser: *Parser) ParseError!*StmtNode {
    _ = try parser.consume(.left_brace, "Expected '{'");
    
    var stmts = std.ArrayList(*StmtNode).init(parser.allocator);
    
    while (!parser.check(.right_brace) and !parser.isAtEnd()) {
        const stmt = parseStatement(parser) catch {
            parser.synchronize();
            continue;
        };
        try stmts.append(stmt);
    }
    
    _ = try parser.consume(.right_brace, "Expected '}'");
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .block = try stmts.toOwnedSlice() };
    return node;
}

fn parseIfStmt(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'if'
    _ = try parser.consume(.left_paren, "Expected '(' after 'if'");
    const condition = try parseExpr(parser);
    _ = try parser.consume(.right_paren, "Expected ')' after condition");
    
    const then_branch = try parseStatement(parser);
    
    var else_branch: ?*StmtNode = null;
    if (parser.check(.keyword_else)) {
        _ = parser.advance(); // consume 'else'
        else_branch = try parseStatement(parser);
    }
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .if_stmt = .{
        .condition = condition,
        .then_branch = then_branch,
        .else_branch = else_branch,
    }};
    return node;
}

fn parseWhileStmt(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'while'
    _ = try parser.consume(.left_paren, "Expected '(' after 'while'");
    const condition = try parseExpr(parser);
    _ = try parser.consume(.right_paren, "Expected ')' after condition");
    
    const body = try parseStatement(parser);
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .while_stmt = .{
        .condition = condition,
        .body = body,
    }};
    return node;
}
```

---

## Step 855: Function Declaration Parsing

```zig
// function parsing

fn parseFnDecl(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'fn'
    
    const name_token = try parser.consume(.identifier, "Expected function name");
    _ = try parser.consume(.left_paren, "Expected '(' after function name");
    
    var params = std.ArrayList(StmtNode.Param).init(parser.allocator);
    
    if (!parser.check(.right_paren)) {
        while (true) {
            const param_name = try parser.consume(.identifier, "Expected parameter name");
            _ = try parser.consume(.colon, "Expected ':' after parameter name");
            const param_type = try parser.consume(.identifier, "Expected parameter type");
            
            try params.append(.{
                .name = param_name.lexeme,
                .type_name = param_type.lexeme,
            });
            
            if (!parser.check(.comma)) break;
            _ = parser.advance(); // consume ','
        }
    }
    
    _ = try parser.consume(.right_paren, "Expected ')' after parameters");
    
    var return_type: ?[]const u8 = null;
    if (parser.check(.colon)) {
        _ = parser.advance(); // consume ':'
        const type_token = try parser.consume(.identifier, "Expected return type");
        return_type = type_token.lexeme;
    }
    
    const body = try parseBlock(parser);
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .fn_decl = .{
        .name = name_token.lexeme,
        .params = try params.toOwnedSlice(),
        .return_type = return_type,
        .body = body,
    }};
    return node;
}

fn parseReturnStmt(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'return'
    
    var value: ?*ExprNode = null;
    if (!parser.check(.semicolon)) {
        value = try parseExpr(parser);
    }
    
    _ = try parser.consume(.semicolon, "Expected ';' after return value");
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .return_stmt = value };
    return node;
}

fn parseForStmt(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'for'
    _ = try parser.consume(.left_paren, "Expected '(' after 'for'");
    
    var init_stmt: ?*StmtNode = null;
    if (!parser.check(.semicolon)) {
        if (parser.check(.keyword_var) or parser.check(.keyword_const)) {
            init_stmt = try parseVarDecl(parser);
        } else {
            init_stmt = try parseExpressionStmt(parser);
        }
    } else {
        _ = parser.advance(); // consume ';'
    }
    
    var condition: ?*ExprNode = null;
    if (!parser.check(.semicolon)) {
        condition = try parseExpr(parser);
    }
    _ = try parser.consume(.semicolon, "Expected ';' after for condition");
    
    var update: ?*ExprNode = null;
    if (!parser.check(.right_paren)) {
        update = try parseExpr(parser);
    }
    _ = try parser.consume(.right_paren, "Expected ')' after for clauses");
    
    const body = try parseStatement(parser);
    
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .for_stmt = .{
        .init = init_stmt,
        .condition = condition,
        .update = update,
        .body = body,
    }};
    return node;
}
```

---

## Step 856: Control Flow Parsing

```zig
// control_flow.zig - การ parse control flow statements

pub const BreakStmt = struct {};
pub const ContinueStmt = struct {};

pub const ExtendedStmtNode = union(enum) {
    // ... previous types ...
    break_stmt: BreakStmt,
    continue_stmt: ContinueStmt,
    switch_stmt: SwitchStmt,
    
    pub const SwitchCase = struct {
        value: *ExprNode,
        body: []*StmtNode,
    };
    
    pub const SwitchStmt = struct {
        subject: *ExprNode,
        cases: []SwitchCase,
        default_case: ?[]*StmtNode,
    };
};

fn parseSwitchStmt(parser: *Parser) ParseError!*StmtNode {
    _ = parser.advance(); // consume 'switch'
    _ = try parser.consume(.left_paren, "Expected '(' after 'switch'");
    const subject = try parseExpr(parser);
    _ = try parser.consume(.right_paren, "Expected ')' after switch subject");
    _ = try parser.consume(.left_brace, "Expected '{' before switch body");
    
    var cases = std.ArrayList(ExtendedStmtNode.SwitchCase).init(parser.allocator);
    var default_case: ?[]*StmtNode = null;
    
    while (!parser.check(.right_brace) and !parser.isAtEnd()) {
        if (parser.check(.identifier) and 
            std.mem.eql(u8, parser.peek().lexeme, "case")) 
        {
            _ = parser.advance(); // consume 'case'
            const value = try parseExpr(parser);
            _ = try parser.consume(.colon, "Expected ':' after case value");
            
            var body = std.ArrayList(*StmtNode).init(parser.allocator);
            while (!parser.check(.identifier) and 
                   !parser.check(.right_brace) and 
                   !parser.isAtEnd()) 
            {
                const stmt = try parseStatement(parser);
                try body.append(stmt);
            }
            
            try cases.append(.{
                .value = value,
                .body = try body.toOwnedSlice(),
            });
        } else if (parser.check(.identifier) and 
                   std.mem.eql(u8, parser.peek().lexeme, "default")) 
        {
            _ = parser.advance(); // consume 'default'
            _ = try parser.consume(.colon, "Expected ':' after 'default'");
            
            var body = std.ArrayList(*StmtNode).init(parser.allocator);
            while (!parser.check(.right_brace) and !parser.isAtEnd()) {
                const stmt = try parseStatement(parser);
                try body.append(stmt);
            }
            
            default_case = try body.toOwnedSlice();
        } else {
            break;
        }
    }
    
    _ = try parser.consume(.right_brace, "Expected '}' after switch body");
    
    _ = subject; // TODO: use in actual implementation
    _ = cases;
    _ = default_case;
    
    // placeholder
    const node = try parser.allocator.create(StmtNode);
    node.* = .{ .block = &.{} };
    return node;
}
```

---

## Step 857: Error Recovery in Parsers

### การ Recover จาก Parse Error

```zig
// error_recovery.zig - เทคนิคการ recover จาก parse errors

pub const RecoveryStrategy = enum {
    panic_mode,       // เลื่อนไปจนกว่าจะเจอ synchronization point
    phrase_level,     // แทนที่/เพิ่ม/ลบ token เพื่อ fix error ในที่นั้น
    error_productions, // มี grammar rules สำหรับ error cases
};

pub const ErrorRecoveryParser = struct {
    base: Parser,
    recovery_points: []const TokenType,
    
    pub fn init(tokens: []const Token, allocator: std.mem.Allocator) ErrorRecoveryParser {
        return ErrorRecoveryParser{
            .base = Parser.init(tokens, allocator),
            .recovery_points = &.{
                .semicolon,
                .right_brace,
                .keyword_fn,
                .keyword_var,
                .keyword_if,
                .keyword_while,
                .eof,
            },
        };
    }
    
    // Panic mode recovery - เลื่อนไปจนกว่าจะเจอ sync point
    pub fn panicModeRecover(self: *ErrorRecoveryParser) void {
        while (!self.base.isAtEnd()) {
            for (self.recovery_points) |rp| {
                if (self.base.peek().type == rp) {
                    if (rp == .semicolon) {
                        _ = self.base.advance(); // consume the semicolon
                    }
                    return;
                }
            }
            _ = self.base.advance();
        }
    }
    
    // Parse ด้วย error collection (ไม่หยุดที่ error แรก)
    pub fn parseProgram(self: *ErrorRecoveryParser) ![]*StmtNode {
        var stmts = std.ArrayList(*StmtNode).init(self.base.allocator);
        
        while (!self.base.isAtEnd()) {
            const stmt = parseStatement(&self.base) catch |err| {
                // บันทึก error และ recover
                std.debug.print("Recovery from: {}\n", .{err});
                self.panicModeRecover();
                continue;
            };
            try stmts.append(stmt);
        }
        
        return stmts.toOwnedSlice();
    }
    
    // Error production - handle common mistakes
    pub fn tryFixMissingSemicolon(self: *ErrorRecoveryParser) bool {
        // ถ้า token ต่อไปหน้าตาเหมือน start ของ statement ใหม่
        const next = self.base.peek();
        return switch (next.type) {
            .keyword_var,
            .keyword_const,
            .keyword_if,
            .keyword_while,
            .keyword_fn,
            .right_brace,
            .eof,
            => true,
            else => false,
        };
    }
};

// ตัวอย่างการใช้งาน error recovery
pub fn parseWithRecovery(source: []const u8, allocator: std.mem.Allocator) !void {
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var recovery_parser = ErrorRecoveryParser.init(tokens, allocator);
    
    const stmts = try recovery_parser.parseProgram();
    defer allocator.free(stmts);
    
    if (recovery_parser.base.hasErrors()) {
        std.debug.print("\nParse completed with errors:\n", .{});
        recovery_parser.base.printErrors();
    } else {
        std.debug.print("Parse successful: {} statements\n", .{stmts.len});
    }
}
```

---

## Step 858: Parser Testing

```zig
// parser_test.zig - การทดสอบ parser

const testing = std.testing;

test "parse simple expression" {
    const source = "1 + 2 * 3;";
    const allocator = testing.allocator;
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = Parser.init(tokens, allocator);
    defer parser.deinit();
    
    // Parse expression statement
    const stmt = try parseStatement(&parser);
    try testing.expect(stmt.* == .expression_stmt);
    
    const expr = stmt.expression_stmt;
    try testing.expect(expr.* == .binary);
    try testing.expectEqualStrings("+", expr.binary.op);
    
    // Left side should be number 1
    try testing.expect(expr.binary.left.* == .number);
    try testing.expectEqual(@as(f64, 1.0), expr.binary.left.number);
    
    // Right side should be 2 * 3
    try testing.expect(expr.binary.right.* == .binary);
    try testing.expectEqualStrings("*", expr.binary.right.binary.op);
}

test "parse variable declaration" {
    const source = "var x: int = 42;";
    const allocator = testing.allocator;
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = Parser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmt = try parseStatement(&parser);
    try testing.expect(stmt.* == .var_decl);
    try testing.expectEqualStrings("x", stmt.var_decl.name);
    try testing.expect(!stmt.var_decl.is_const);
    try testing.expectEqualStrings("int", stmt.var_decl.type_annotation.?);
    try testing.expect(stmt.var_decl.initializer != null);
}

test "parse if statement" {
    const source = "if (x > 0) { return x; }";
    const allocator = testing.allocator;
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = Parser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmt = try parseStatement(&parser);
    try testing.expect(stmt.* == .if_stmt);
    try testing.expect(stmt.if_stmt.else_branch == null);
}

test "parse function declaration" {
    const source = 
        \\fn add(a: int, b: int): int {
        \\    return a + b;
        \\}
    ;
    const allocator = testing.allocator;
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = Parser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmt = try parseStatement(&parser);
    try testing.expect(stmt.* == .fn_decl);
    try testing.expectEqualStrings("add", stmt.fn_decl.name);
    try testing.expectEqual(@as(usize, 2), stmt.fn_decl.params.len);
    try testing.expectEqualStrings("int", stmt.fn_decl.return_type.?);
}

test "parser error recovery" {
    // Source มี error: ขาด semicolon
    const source = "var x = 1\nvar y = 2;";
    const allocator = testing.allocator;
    
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var recovery_parser = ErrorRecoveryParser.init(tokens, allocator);
    const stmts = try recovery_parser.parseProgram();
    defer allocator.free(stmts);
    
    // ควรจะ parse ได้บางส่วน แม้จะมี error
    try testing.expect(stmts.len > 0);
    try testing.expect(recovery_parser.base.hasErrors());
}
```

---

## Step 859: Precedence Climbing

### Precedence Climbing Algorithm

```zig
// precedence_climbing.zig - อีก algorithm สำหรับ expression parsing

pub const BinOp = struct {
    symbol: []const u8,
    precedence: u8,
    associativity: enum { left, right },
};

pub const BIN_OPS = [_]BinOp{
    .{ .symbol = "=",  .precedence = 1, .associativity = .right },
    .{ .symbol = "||", .precedence = 2, .associativity = .left },
    .{ .symbol = "&&", .precedence = 3, .associativity = .left },
    .{ .symbol = "==", .precedence = 4, .associativity = .left },
    .{ .symbol = "!=", .precedence = 4, .associativity = .left },
    .{ .symbol = "<",  .precedence = 5, .associativity = .left },
    .{ .symbol = "<=", .precedence = 5, .associativity = .left },
    .{ .symbol = ">",  .precedence = 5, .associativity = .left },
    .{ .symbol = ">=", .precedence = 5, .associativity = .left },
    .{ .symbol = "+",  .precedence = 6, .associativity = .left },
    .{ .symbol = "-",  .precedence = 6, .associativity = .left },
    .{ .symbol = "*",  .precedence = 7, .associativity = .left },
    .{ .symbol = "/",  .precedence = 7, .associativity = .left },
    .{ .symbol = "%",  .precedence = 7, .associativity = .left },
};

fn getBinOp(symbol: []const u8) ?BinOp {
    for (BIN_OPS) |op| {
        if (std.mem.eql(u8, op.symbol, symbol)) return op;
    }
    return null;
}

// Precedence Climbing parser
pub fn climbExpression(parser: *Parser, min_prec: u8) !*ExprNode {
    var left = try parsePrimary(parser);
    
    while (true) {
        const token = parser.peek();
        const op = getBinOp(token.lexeme) orelse break;
        
        if (op.precedence < min_prec) break;
        _ = parser.advance();
        
        const next_min = if (op.associativity == .left)
            op.precedence + 1
        else
            op.precedence;
        
        const right = try climbExpression(parser, next_min);
        
        const node = try parser.allocator.create(ExprNode);
        node.* = .{ .binary = .{
            .left = left,
            .op = token.lexeme,
            .right = right,
        }};
        left = node;
    }
    
    return left;
}

fn parsePrimary(parser: *Parser) !*ExprNode {
    const token = parser.advance();
    
    return switch (token.type) {
        .number => blk: {
            const val = try std.fmt.parseFloat(f64, token.lexeme);
            const node = try parser.allocator.create(ExprNode);
            node.* = .{ .number = val };
            break :blk node;
        },
        .identifier => blk: {
            const node = try parser.allocator.create(ExprNode);
            node.* = .{ .identifier = token.lexeme };
            break :blk node;
        },
        .left_paren => blk: {
            const expr = try climbExpression(parser, 0);
            _ = try parser.consume(.right_paren, "Expected ')'");
            break :blk expr;
        },
        .minus, .bang => blk: {
            const operand = try climbExpression(parser, 8); // unary precedence
            const node = try parser.allocator.create(ExprNode);
            node.* = .{ .unary = .{ .op = token.lexeme, .operand = operand } };
            break :blk node;
        },
        else => error.ExpectedExpression,
    };
}
```

---

## Step 860: Complete Parser Implementation

```zig
// complete_parser.zig - การ implement parser ที่สมบูรณ์

const std = @import("std");

pub const CompleteParser = struct {
    const Self = @This();
    
    tokens: []const Token,
    current: usize,
    allocator: std.mem.Allocator,
    errors: std.ArrayList(ParseDiagnostic),
    
    pub const ParseDiagnostic = struct {
        message: []const u8,
        line: usize,
        column: usize,
        severity: enum { err, warning, note },
    };
    
    pub fn init(tokens: []const Token, allocator: std.mem.Allocator) Self {
        return Self{
            .tokens = tokens,
            .current = 0,
            .allocator = allocator,
            .errors = std.ArrayList(ParseDiagnostic).init(allocator),
        };
    }
    
    pub fn deinit(self: *Self) void {
        for (self.errors.items) |diag| {
            self.allocator.free(diag.message);
        }
        self.errors.deinit();
    }
    
    // Parse program (list of statements/declarations)
    pub fn parseProgram(self: *Self) ![]*StmtNode {
        var statements = std.ArrayList(*StmtNode).init(self.allocator);
        
        while (!self.isAtEnd()) {
            const stmt = self.parseDeclaration() catch {
                self.synchronize();
                continue;
            };
            try statements.append(stmt);
        }
        
        return statements.toOwnedSlice();
    }
    
    fn parseDeclaration(self: *Self) !*StmtNode {
        if (self.check(.keyword_fn)) return self.parseFunctionDecl();
        if (self.check(.keyword_var) or self.check(.keyword_const)) return self.parseVarDecl();
        return self.parseStatement();
    }
    
    fn parseStatement(self: *Self) !*StmtNode {
        return switch (self.peek().type) {
            .keyword_if => self.parseIf(),
            .keyword_while => self.parseWhile(),
            .keyword_for => self.parseFor(),
            .keyword_return => self.parseReturn(),
            .left_brace => self.parseBlock(),
            else => self.parseExprStmt(),
        };
    }
    
    // ... (ใช้ functions จาก steps ก่อนหน้า)
    
    fn isAtEnd(self: *Self) bool {
        return self.peek().type == .eof;
    }
    
    fn peek(self: *Self) Token {
        return self.tokens[self.current];
    }
    
    fn advance(self: *Self) Token {
        if (!self.isAtEnd()) self.current += 1;
        return self.tokens[self.current - 1];
    }
    
    fn check(self: *Self, t: TokenType) bool {
        return self.peek().type == t;
    }
    
    fn consume(self: *Self, t: TokenType, msg: []const u8) !Token {
        if (self.check(t)) return self.advance();
        const tok = self.peek();
        const message = try std.fmt.allocPrint(self.allocator, 
            "Line {d}:{d}: {s}", .{tok.line, tok.column, msg});
        try self.errors.append(.{
            .message = message,
            .line = tok.line,
            .column = tok.column,
            .severity = .err,
        });
        return error.UnexpectedToken;
    }
    
    fn synchronize(self: *Self) void {
        _ = self.advance();
        while (!self.isAtEnd()) {
            if (self.tokens[self.current - 1].type == .semicolon) return;
            switch (self.peek().type) {
                .keyword_fn, .keyword_var, .keyword_const,
                .keyword_if, .keyword_while, .keyword_for,
                .keyword_return => return,
                else => _ = self.advance(),
            }
        }
    }
    
    fn parseFunctionDecl(self: *Self) !*StmtNode {
        _ = self.advance(); // 'fn'
        const name = try self.consume(.identifier, "Expected function name");
        _ = try self.consume(.left_paren, "Expected '('");
        
        var params = std.ArrayList(StmtNode.Param).init(self.allocator);
        
        while (!self.check(.right_paren) and !self.isAtEnd()) {
            if (params.items.len > 0) {
                _ = try self.consume(.comma, "Expected ',' between parameters");
            }
            const pname = try self.consume(.identifier, "Expected parameter name");
            _ = try self.consume(.colon, "Expected ':'");
            const ptype = try self.consume(.identifier, "Expected type");
            try params.append(.{ .name = pname.lexeme, .type_name = ptype.lexeme });
        }
        
        _ = try self.consume(.right_paren, "Expected ')'");
        
        var ret_type: ?[]const u8 = null;
        if (self.check(.colon)) {
            _ = self.advance();
            ret_type = (try self.consume(.identifier, "Expected return type")).lexeme;
        }
        
        const body = try self.parseBlock();
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .fn_decl = .{
            .name = name.lexeme,
            .params = try params.toOwnedSlice(),
            .return_type = ret_type,
            .body = body,
        }};
        return node;
    }
    
    fn parseVarDecl(self: *Self) !*StmtNode {
        const is_const = self.peek().type == .keyword_const;
        _ = self.advance();
        const name = try self.consume(.identifier, "Expected name");
        var type_ann: ?[]const u8 = null;
        if (self.check(.colon)) {
            _ = self.advance();
            type_ann = (try self.consume(.identifier, "Expected type")).lexeme;
        }
        var init_val: ?*ExprNode = null;
        if (self.check(.equal)) {
            _ = self.advance();
            init_val = try self.parseExpr();
        }
        _ = try self.consume(.semicolon, "Expected ';'");
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .var_decl = .{
            .name = name.lexeme,
            .is_const = is_const,
            .type_annotation = type_ann,
            .initializer = init_val,
        }};
        return node;
    }
    
    fn parseIf(self: *Self) !*StmtNode {
        _ = self.advance();
        _ = try self.consume(.left_paren, "Expected '('");
        const cond = try self.parseExpr();
        _ = try self.consume(.right_paren, "Expected ')'");
        const then_b = try self.parseStatement();
        var else_b: ?*StmtNode = null;
        if (self.check(.keyword_else)) {
            _ = self.advance();
            else_b = try self.parseStatement();
        }
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .if_stmt = .{ .condition = cond, .then_branch = then_b, .else_branch = else_b } };
        return node;
    }
    
    fn parseWhile(self: *Self) !*StmtNode {
        _ = self.advance();
        _ = try self.consume(.left_paren, "Expected '('");
        const cond = try self.parseExpr();
        _ = try self.consume(.right_paren, "Expected ')'");
        const body = try self.parseStatement();
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .while_stmt = .{ .condition = cond, .body = body } };
        return node;
    }
    
    fn parseFor(self: *Self) !*StmtNode {
        _ = self.advance();
        _ = try self.consume(.left_paren, "Expected '('");
        // simplified for loop
        var init_s: ?*StmtNode = null;
        if (!self.check(.semicolon)) {
            init_s = try self.parseVarDecl();
        } else _ = self.advance();
        var cond: ?*ExprNode = null;
        if (!self.check(.semicolon)) cond = try self.parseExpr();
        _ = try self.consume(.semicolon, "Expected ';'");
        var upd: ?*ExprNode = null;
        if (!self.check(.right_paren)) upd = try self.parseExpr();
        _ = try self.consume(.right_paren, "Expected ')'");
        const body = try self.parseStatement();
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .for_stmt = .{ .init = init_s, .condition = cond, .update = upd, .body = body } };
        return node;
    }
    
    fn parseReturn(self: *Self) !*StmtNode {
        _ = self.advance();
        var val: ?*ExprNode = null;
        if (!self.check(.semicolon)) val = try self.parseExpr();
        _ = try self.consume(.semicolon, "Expected ';'");
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .return_stmt = val };
        return node;
    }
    
    fn parseBlock(self: *Self) !*StmtNode {
        _ = try self.consume(.left_brace, "Expected '{'");
        var stmts = std.ArrayList(*StmtNode).init(self.allocator);
        while (!self.check(.right_brace) and !self.isAtEnd()) {
            const stmt = self.parseDeclaration() catch {
                self.synchronize();
                continue;
            };
            try stmts.append(stmt);
        }
        _ = try self.consume(.right_brace, "Expected '}'");
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .block = try stmts.toOwnedSlice() };
        return node;
    }
    
    fn parseExprStmt(self: *Self) !*StmtNode {
        const expr = try self.parseExpr();
        _ = try self.consume(.semicolon, "Expected ';'");
        const node = try self.allocator.create(StmtNode);
        node.* = .{ .expression_stmt = expr };
        return node;
    }
    
    fn parseExpr(self: *Self) !*ExprNode {
        return self.parseAssignment();
    }
    
    fn parseAssignment(self: *Self) !*ExprNode {
        var expr = try self.parseOr();
        if (self.check(.equal)) {
            _ = self.advance();
            const val = try self.parseAssignment();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .assign = .{ .target = expr, .value = val } };
            return node;
        }
        return expr;
    }
    
    fn parseOr(self: *Self) !*ExprNode {
        var left = try self.parseAnd();
        while (self.check(.pipe_pipe)) {
            const op = self.advance();
            const right = try self.parseAnd();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseAnd(self: *Self) !*ExprNode {
        var left = try self.parseEquality();
        while (self.check(.amp_amp)) {
            const op = self.advance();
            const right = try self.parseEquality();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseEquality(self: *Self) !*ExprNode {
        var left = try self.parseComparison();
        while (self.check(.equal_equal) or self.check(.bang_equal)) {
            const op = self.advance();
            const right = try self.parseComparison();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseComparison(self: *Self) !*ExprNode {
        var left = try self.parseTerm();
        while (self.check(.less) or self.check(.less_equal) or
               self.check(.greater) or self.check(.greater_equal)) 
        {
            const op = self.advance();
            const right = try self.parseTerm();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseTerm(self: *Self) !*ExprNode {
        var left = try self.parseFactor();
        while (self.check(.plus) or self.check(.minus)) {
            const op = self.advance();
            const right = try self.parseFactor();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseFactor(self: *Self) !*ExprNode {
        var left = try self.parseUnary();
        while (self.check(.star) or self.check(.slash) or self.check(.percent)) {
            const op = self.advance();
            const right = try self.parseUnary();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .binary = .{ .left = left, .op = op.lexeme, .right = right } };
            left = node;
        }
        return left;
    }
    
    fn parseUnary(self: *Self) !*ExprNode {
        if (self.check(.bang) or self.check(.minus)) {
            const op = self.advance();
            const operand = try self.parseUnary();
            const node = try self.allocator.create(ExprNode);
            node.* = .{ .unary = .{ .op = op.lexeme, .operand = operand } };
            return node;
        }
        return self.parseCall();
    }
    
    fn parseCall(self: *Self) !*ExprNode {
        var expr = try self.parsePrimary();
        
        while (true) {
            if (self.check(.left_paren)) {
                _ = self.advance();
                var args = std.ArrayList(*ExprNode).init(self.allocator);
                if (!self.check(.right_paren)) {
                    while (true) {
                        try args.append(try self.parseExpr());
                        if (!self.check(.comma)) break;
                        _ = self.advance();
                    }
                }
                _ = try self.consume(.right_paren, "Expected ')'");
                const node = try self.allocator.create(ExprNode);
                node.* = .{ .call = .{ .callee = expr, .args = try args.toOwnedSlice() } };
                expr = node;
            } else if (self.check(.dot)) {
                _ = self.advance();
                const field = try self.consume(.identifier, "Expected field name");
                const node = try self.allocator.create(ExprNode);
                node.* = .{ .field_access = .{ .object = expr, .field = field.lexeme } };
                expr = node;
            } else if (self.check(.left_bracket)) {
                _ = self.advance();
                const idx = try self.parseExpr();
                _ = try self.consume(.right_bracket, "Expected ']'");
                const node = try self.allocator.create(ExprNode);
                node.* = .{ .index = .{ .object = expr, .idx = idx } };
                expr = node;
            } else break;
        }
        
        return expr;
    }
    
    fn parsePrimary(self: *Self) !*ExprNode {
        const token = self.advance();
        const node = try self.allocator.create(ExprNode);
        
        switch (token.type) {
            .number => {
                const val = std.fmt.parseFloat(f64, token.lexeme) catch 0.0;
                node.* = .{ .number = val };
            },
            .string => node.* = .{ .string = token.lexeme },
            .keyword_true => node.* = .{ .boolean = true },
            .keyword_false => node.* = .{ .boolean = false },
            .keyword_null => node.* = .{ .null_val = {} },
            .identifier => node.* = .{ .identifier = token.lexeme },
            .left_paren => {
                const expr = try self.parseExpr();
                _ = try self.consume(.right_paren, "Expected ')'");
                node.* = .{ .group = expr };
            },
            else => {
                const msg = try std.fmt.allocPrint(self.allocator,
                    "Unexpected token '{s}'", .{token.lexeme});
                try self.errors.append(.{
                    .message = msg,
                    .line = token.line,
                    .column = token.column,
                    .severity = .err,
                });
                return error.ExpectedExpression;
            },
        }
        
        return node;
    }
};

// ทดสอบ complete parser
test "complete parser: parse program" {
    const source =
        \\fn factorial(n: int): int {
        \\    if (n <= 1) {
        \\        return 1;
        \\    }
        \\    return n * factorial(n - 1);
        \\}
        \\
        \\var result: int = factorial(5);
    ;
    
    const allocator = std.testing.allocator;
    var lexer = Lexer.init(source);
    const tokens = try lexer.tokenize(allocator);
    defer allocator.free(tokens);
    
    var parser = CompleteParser.init(tokens, allocator);
    defer parser.deinit();
    
    const stmts = try parser.parseProgram();
    defer allocator.free(stmts);
    
    try std.testing.expectEqual(@as(usize, 0), parser.errors.items.len);
    try std.testing.expectEqual(@as(usize, 2), stmts.len);
    
    // First statement should be function declaration
    try std.testing.expect(stmts[0].* == .fn_decl);
    try std.testing.expectEqualStrings("factorial", stmts[0].fn_decl.name);
    
    // Second statement should be variable declaration
    try std.testing.expect(stmts[1].* == .var_decl);
    try std.testing.expectEqualStrings("result", stmts[1].var_decl.name);
}
```

---

## สรุป Part 86

ในส่วนนี้เราได้เรียนรู้:

1. **Step 851**: Recursive Descent Parser concepts และ Lexer implementation
2. **Step 852**: โครงสร้าง Parser struct พร้อม error handling
3. **Step 853**: Pratt Parser สำหรับ expression parsing ที่มี operator precedence
4. **Step 854**: Statement parsing (variable declarations, blocks)
5. **Step 855**: Function declaration parsing
6. **Step 856**: Control flow parsing (if, while, for, switch)
7. **Step 857**: Error recovery strategies (panic mode, phrase level)
8. **Step 858**: Parser testing techniques
9. **Step 859**: Precedence climbing algorithm
10. **Step 860**: Complete parser implementation ที่รวมทุกอย่างไว้ด้วยกัน

Parser ที่ดีต้องมีทั้ง correctness, good error messages, และ error recovery ที่ดี เพื่อให้ผู้ใช้สามารถเข้าใจและแก้ไข syntax errors ได้อย่างมีประสิทธิภาพ
