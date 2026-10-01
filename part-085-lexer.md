# Part 85: Parsing: Lexer Implementation
## Steps 841-850

---

## Step 841: Lexer/Tokenizer Concepts

### Lexer คืออะไร?

Lexer (Lexical Analyzer) หรือ Tokenizer เป็นขั้นตอนแรกของกระบวนการ parsing ทำหน้าที่แปลง source code (text) ออกมาเป็น sequence ของ tokens

### Pipeline ของ Compiler/Interpreter

```
Source Code (text)
        ↓
    [Lexer/Tokenizer]
        ↓
    Token Stream
        ↓
    [Parser]
        ↓
   Abstract Syntax Tree (AST)
        ↓
    [Semantic Analyzer]
        ↓
    [Code Generator]
        ↓
    Output (bytecode/machine code)
```

### หลักการทำงานของ Lexer

1. **Input**: stream of characters
2. **Process**: จับกลุ่ม characters เป็น lexemes
3. **Output**: sequence of tokens (type + value)

```zig
// lexer_concepts.zig
const std = @import("std");

/// Token คือคู่ของ (type, value)
pub const Token = struct {
    kind: TokenKind,
    lexeme: []const u8, // text ดั้งเดิมจาก source
    line: u32,          // บรรทัดที่พบ
    column: u32,        // column ที่พบ
    
    pub fn format(
        self: Token,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("Token{{ .kind={s}, .lexeme=\"{s}\", .line={d}, .col={d} }}", .{
            @tagName(self.kind), self.lexeme, self.line, self.column
        });
    }
};

/// ประเภทของ Token
pub const TokenKind = enum {
    // Literals
    integer,
    float,
    string,
    boolean,
    null_literal,
    
    // Identifiers
    identifier,
    
    // Keywords
    keyword_if,
    keyword_else,
    keyword_while,
    keyword_for,
    keyword_return,
    keyword_let,
    keyword_fn,
    keyword_true,
    keyword_false,
    keyword_null,
    
    // Operators
    plus,           // +
    minus,          // -
    star,           // *
    slash,          // /
    percent,        // %
    equal,          // =
    equal_equal,    // ==
    bang,           // !
    bang_equal,     // !=
    less,           // <
    less_equal,     // <=
    greater,        // >
    greater_equal,  // >=
    ampersand,      // &
    pipe,           // |
    
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
    arrow,          // ->
    
    // Special
    eof,
    error_token,
};

/// Lexer Error
pub const LexerError = error{
    UnexpectedCharacter,
    UnterminatedString,
    InvalidNumber,
    OutOfMemory,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Lexer Concepts ===\n\n", .{});
    
    // แสดง token types
    try stdout.print("Token Types:\n", .{});
    
    const tokenExamples = [_]struct { kind: []const u8, example: []const u8 }{
        .{ .kind = "integer", .example = "42, -7, 0" },
        .{ .kind = "float", .example = "3.14, -2.7" },
        .{ .kind = "string", .example = "\"hello\", \"world\"" },
        .{ .kind = "identifier", .example = "x, myVar, count" },
        .{ .kind = "keyword", .example = "if, else, while, fn" },
        .{ .kind = "operator", .example = "+, -, *, /, ==, !=" },
        .{ .kind = "delimiter", .example = "(, ), {, }, [, ]" },
    };
    
    for (tokenExamples) |te| {
        try stdout.print("  {s:<15}: {s}\n", .{ te.kind, te.example });
    }
    
    // ตัวอย่าง source code
    const source = "let x = 42 + y * 3;";
    try stdout.print("\nSource: {s}\n", .{source});
    try stdout.print("Tokens: let, x, =, 42, +, y, *, 3, ;\n", .{});
}
```

---

## Step 842: Token Types Definition

### การออกแบบ Token Types

```zig
// token_types.zig
const std = @import("std");

/// ทุก token types ใน simple language
pub const TK = enum {
    // === Literals ===
    tk_int,        // 123
    tk_float,      // 3.14
    tk_string,     // "hello"
    tk_char,       // 'a'
    tk_bool,       // true/false
    
    // === Identifiers ===
    tk_ident,      // variable/function names
    
    // === Keywords (ต้อง list ก่อน identifiers ใน scan) ===
    tk_let,        // let
    tk_var,        // var
    tk_const,      // const
    tk_fn,         // fn
    tk_return,     // return
    tk_if,         // if
    tk_else,       // else
    tk_elif,       // elif
    tk_while,      // while
    tk_for,        // for
    tk_in,         // in
    tk_break,      // break
    tk_continue,   // continue
    tk_true,       // true
    tk_false,      // false
    tk_null,       // null
    tk_and,        // and
    tk_or,         // or
    tk_not,        // not
    tk_import,     // import
    tk_struct,     // struct
    tk_enum,       // enum
    tk_type,       // type
    
    // === Arithmetic Operators ===
    tk_plus,       // +
    tk_minus,      // -
    tk_star,       // *
    tk_slash,      // /
    tk_percent,    // %
    tk_power,      // **
    
    // === Comparison Operators ===
    tk_eq,         // ==
    tk_ne,         // !=
    tk_lt,         // <
    tk_le,         // <=
    tk_gt,         // >
    tk_ge,         // >=
    
    // === Logical Operators ===
    tk_and_and,    // &&
    tk_pipe_pipe,  // ||
    tk_bang,       // !
    
    // === Bitwise Operators ===
    tk_amp,        // &
    tk_pipe,       // |
    tk_caret,      // ^
    tk_tilde,      // ~
    tk_lshift,     // <<
    tk_rshift,     // >>
    
    // === Assignment Operators ===
    tk_assign,     // =
    tk_plus_eq,    // +=
    tk_minus_eq,   // -=
    tk_star_eq,    // *=
    tk_slash_eq,   // /=
    tk_percent_eq, // %=
    
    // === Delimiters ===
    tk_lparen,     // (
    tk_rparen,     // )
    tk_lbrace,     // {
    tk_rbrace,     // }
    tk_lbracket,   // [
    tk_rbracket,   // ]
    
    // === Punctuation ===
    tk_semi,       // ;
    tk_colon,      // :
    tk_comma,      // ,
    tk_dot,        // .
    tk_dotdot,     // ..
    tk_dotdotdot,  // ...
    tk_arrow,      // ->
    tk_fat_arrow,  // =>
    tk_hash,       // #
    tk_at,         // @
    tk_question,   // ?
    
    // === Special ===
    tk_newline,    // \n (ถ้า significant)
    tk_eof,        // end of file
    tk_error,      // lexer error
};

/// Keyword map
pub const keywords = std.StaticStringMap(TK).initComptime(.{
    .{ "let",      .tk_let },
    .{ "var",      .tk_var },
    .{ "const",    .tk_const },
    .{ "fn",       .tk_fn },
    .{ "return",   .tk_return },
    .{ "if",       .tk_if },
    .{ "else",     .tk_else },
    .{ "elif",     .tk_elif },
    .{ "while",    .tk_while },
    .{ "for",      .tk_for },
    .{ "in",       .tk_in },
    .{ "break",    .tk_break },
    .{ "continue", .tk_continue },
    .{ "true",     .tk_true },
    .{ "false",    .tk_false },
    .{ "null",     .tk_null },
    .{ "and",      .tk_and },
    .{ "or",       .tk_or },
    .{ "not",      .tk_not },
    .{ "import",   .tk_import },
    .{ "struct",   .tk_struct },
    .{ "enum",     .tk_enum },
    .{ "type",     .tk_type },
});

/// Token ที่มี metadata
pub const Token = struct {
    kind: TK,
    lexeme: []const u8,
    line: u32,
    col: u32,
    
    // สำหรับ numeric literals
    int_value: ?i64 = null,
    float_value: ?f64 = null,
};

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Token Types ===\n\n", .{});
    
    // แสดง keywords
    try stdout.print("Keywords ({d} total):\n", .{keywords.keys().len});
    for (keywords.keys()) |kw| {
        try stdout.print("  {s}\n", .{kw});
    }
    
    // ทดสอบ keyword lookup
    try stdout.print("\nKeyword Lookup:\n", .{});
    const tests = [_][]const u8{ "if", "hello", "while", "x", "fn" };
    for (tests) |word| {
        if (keywords.get(word)) |kind| {
            try stdout.print("  '{s}' -> keyword: {s}\n", .{ word, @tagName(kind) });
        } else {
            try stdout.print("  '{s}' -> identifier\n", .{word});
        }
    }
}
```

---

## Step 843: Lexer Struct Implementation

### Core Lexer Structure

```zig
// lexer_struct.zig
const std = @import("std");

/// Source position
pub const Position = struct {
    line: u32,
    column: u32,
    offset: usize, // byte offset ใน source
};

/// Lexer Error พร้อม context
pub const LexError = struct {
    message: []const u8,
    position: Position,
};

/// Lexer หลัก
pub const Lexer = struct {
    source: []const u8,
    pos: usize,     // byte offset ปัจจุบัน
    line: u32,      // บรรทัดปัจจุบัน
    column: u32,    // column ปัจจุบัน
    start: usize,   // start ของ token ปัจจุบัน
    errors: std.ArrayList(LexError),
    allocator: std.mem.Allocator,
    
    pub fn init(source: []const u8, allocator: std.mem.Allocator) Lexer {
        return .{
            .source = source,
            .pos = 0,
            .line = 1,
            .column = 1,
            .start = 0,
            .errors = std.ArrayList(LexError).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Lexer) void {
        self.errors.deinit();
    }
    
    /// ดู character ปัจจุบันโดยไม่เลื่อน
    pub fn peek(self: *const Lexer) u8 {
        if (self.pos >= self.source.len) return 0;
        return self.source[self.pos];
    }
    
    /// ดู character ถัดไป
    pub fn peekNext(self: *const Lexer) u8 {
        if (self.pos + 1 >= self.source.len) return 0;
        return self.source[self.pos + 1];
    }
    
    /// ดู character ที่ offset จาก pos
    pub fn peekAt(self: *const Lexer, offset: usize) u8 {
        if (self.pos + offset >= self.source.len) return 0;
        return self.source[self.pos + offset];
    }
    
    /// เลื่อนไปหนึ่ง character แล้วคืนค่าที่ผ่านมา
    pub fn advance(self: *Lexer) u8 {
        const c = self.source[self.pos];
        self.pos += 1;
        
        if (c == '\n') {
            self.line += 1;
            self.column = 1;
        } else {
            self.column += 1;
        }
        
        return c;
    }
    
    /// Match character ถัดไป ถ้าตรง advance แล้วคืน true
    pub fn match(self: *Lexer, expected: u8) bool {
        if (self.isAtEnd()) return false;
        if (self.source[self.pos] != expected) return false;
        _ = self.advance();
        return true;
    }
    
    /// ตรวจสอบว่าถึงจุดสิ้นสุดแล้วหรือยัง
    pub fn isAtEnd(self: *const Lexer) bool {
        return self.pos >= self.source.len;
    }
    
    /// ข้าม whitespace
    pub fn skipWhitespace(self: *Lexer) void {
        while (!self.isAtEnd()) {
            switch (self.peek()) {
                ' ', '\t', '\r', '\n' => _ = self.advance(),
                else => break,
            }
        }
    }
    
    /// ข้าม line comment (//)
    pub fn skipLineComment(self: *Lexer) void {
        while (!self.isAtEnd() and self.peek() != '\n') {
            _ = self.advance();
        }
    }
    
    /// ข้าม block comment (/* ... */)
    pub fn skipBlockComment(self: *Lexer) !void {
        while (!self.isAtEnd()) {
            if (self.peek() == '*' and self.peekNext() == '/') {
                _ = self.advance(); // *
                _ = self.advance(); // /
                return;
            }
            _ = self.advance();
        }
        // Unterminated block comment
        try self.errors.append(.{
            .message = "unterminated block comment",
            .position = .{ .line = self.line, .column = self.column, .offset = self.pos },
        });
    }
    
    /// คืน lexeme ของ token ปัจจุบัน
    pub fn currentLexeme(self: *const Lexer) []const u8 {
        return self.source[self.start..self.pos];
    }
    
    /// Position ปัจจุบัน
    pub fn currentPosition(self: *const Lexer) Position {
        return .{ .line = self.line, .column = self.column, .offset = self.pos };
    }
    
    /// รายงาน error
    pub fn reportError(self: *Lexer, msg: []const u8) !void {
        try self.errors.append(.{
            .message = msg,
            .position = .{ .line = self.line, .column = self.column, .offset = self.pos },
        });
    }
    
    /// แสดง errors
    pub fn printErrors(self: *const Lexer) !void {
        const stdout = std.io.getStdOut().writer();
        for (self.errors.items) |err| {
            try stdout.print("Lexer Error at line {d}, col {d}: {s}\n", .{
                err.position.line, err.position.column, err.message
            });
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    const source = "let x = 42;\n// comment\nlet y = x + 1;\n";
    
    var lexer = Lexer.init(source, allocator);
    defer lexer.deinit();
    
    try stdout.print("=== Lexer Struct ===\n\n", .{});
    try stdout.print("Source:\n{s}\n", .{source});
    
    // แสดงการทำงานของ peek/advance
    try stdout.print("Characters:\n", .{});
    var i: usize = 0;
    while (!lexer.isAtEnd() and i < 10) : (i += 1) {
        const c = lexer.advance();
        try stdout.print("  [{d}] '{c}' (line={d}, col={d})\n", .{
            i, c, lexer.line, lexer.column
        });
    }
}
```

---

## Step 844: Scanning Numbers

### การ Scan ตัวเลข

```zig
// scan_numbers.zig
const std = @import("std");

pub const NumberToken = struct {
    is_float: bool,
    int_val: i64,
    float_val: f64,
    lexeme: []const u8,
    line: u32,
    col: u32,
};

pub fn scanNumber(source: []const u8, pos: *usize, line: u32, col: u32) !NumberToken {
    const start = pos.*;
    var is_float = false;
    
    // เช็ค sign
    if (pos.* < source.len and (source[pos.*] == '-' or source[pos.*] == '+')) {
        pos.* += 1;
    }
    
    // Hexadecimal: 0x...
    if (pos.* + 1 < source.len and source[pos.*] == '0' and
        (source[pos.* + 1] == 'x' or source[pos.* + 1] == 'X'))
    {
        pos.* += 2;
        while (pos.* < source.len and std.ascii.isHex(source[pos.*])) pos.* += 1;
        
        const lexeme = source[start..pos.*];
        const val = try std.fmt.parseInt(i64, lexeme[2..], 16);
        return .{
            .is_float = false,
            .int_val = val,
            .float_val = @floatFromInt(val),
            .lexeme = lexeme,
            .line = line,
            .col = col,
        };
    }
    
    // Octal: 0o...
    if (pos.* + 1 < source.len and source[pos.*] == '0' and
        (source[pos.* + 1] == 'o' or source[pos.* + 1] == 'O'))
    {
        pos.* += 2;
        while (pos.* < source.len and source[pos.*] >= '0' and source[pos.*] <= '7') pos.* += 1;
        
        const lexeme = source[start..pos.*];
        const val = try std.fmt.parseInt(i64, lexeme[2..], 8);
        return .{
            .is_float = false,
            .int_val = val,
            .float_val = @floatFromInt(val),
            .lexeme = lexeme,
            .line = line,
            .col = col,
        };
    }
    
    // Binary: 0b...
    if (pos.* + 1 < source.len and source[pos.*] == '0' and
        (source[pos.* + 1] == 'b' or source[pos.* + 1] == 'B'))
    {
        pos.* += 2;
        while (pos.* < source.len and (source[pos.*] == '0' or source[pos.*] == '1')) pos.* += 1;
        
        const lexeme = source[start..pos.*];
        const val = try std.fmt.parseInt(i64, lexeme[2..], 2);
        return .{
            .is_float = false,
            .int_val = val,
            .float_val = @floatFromInt(val),
            .lexeme = lexeme,
            .line = line,
            .col = col,
        };
    }
    
    // Decimal integer (อาจเป็น float)
    while (pos.* < source.len and std.ascii.isDigit(source[pos.*])) {
        pos.* += 1;
    }
    
    // Decimal point
    if (pos.* < source.len and source[pos.*] == '.') {
        if (pos.* + 1 < source.len and std.ascii.isDigit(source[pos.* + 1])) {
            is_float = true;
            pos.* += 1; // .
            while (pos.* < source.len and std.ascii.isDigit(source[pos.*])) {
                pos.* += 1;
            }
        }
    }
    
    // Exponent: e/E
    if (pos.* < source.len and (source[pos.*] == 'e' or source[pos.*] == 'E')) {
        is_float = true;
        pos.* += 1;
        if (pos.* < source.len and (source[pos.*] == '+' or source[pos.*] == '-')) {
            pos.* += 1;
        }
        while (pos.* < source.len and std.ascii.isDigit(source[pos.*])) {
            pos.* += 1;
        }
    }
    
    // Numeric suffix: u, i, f, etc.
    if (pos.* < source.len) {
        switch (source[pos.*]) {
            'u', 'i', 'f', 'l', 'U', 'I', 'F', 'L' => pos.* += 1,
            else => {},
        }
    }
    
    const lexeme = source[start..pos.*];
    
    if (is_float) {
        const val = try std.fmt.parseFloat(f64, lexeme);
        return .{
            .is_float = true,
            .int_val = @intFromFloat(val),
            .float_val = val,
            .lexeme = lexeme,
            .line = line,
            .col = col,
        };
    } else {
        const val = std.fmt.parseInt(i64, lexeme, 10) catch |err| {
            _ = err;
            return error.InvalidNumber;
        };
        return .{
            .is_float = false,
            .int_val = val,
            .float_val = @floatFromInt(val),
            .lexeme = lexeme,
            .line = line,
            .col = col,
        };
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Scanning Numbers ===\n\n", .{});
    
    const numbers = [_][]const u8{
        "42",
        "3.14",
        "0xFF",
        "0b1010",
        "0o17",
        "1.5e10",
        "2.5e-3",
        "100u",
    };
    
    for (numbers) |num| {
        var pos: usize = 0;
        const tok = scanNumber(num, &pos, 1, 1) catch |err| {
            try stdout.print("Error scanning '{s}': {}\n", .{ num, err });
            continue;
        };
        
        if (tok.is_float) {
            try stdout.print("'{s}' -> float: {d}\n", .{ tok.lexeme, tok.float_val });
        } else {
            try stdout.print("'{s}' -> int: {d} (hex: 0x{X})\n", .{ tok.lexeme, tok.int_val, tok.int_val });
        }
    }
}
```

---

## Step 845: Scanning Strings

### การ Scan Strings

```zig
// scan_strings.zig
const std = @import("std");

pub const StringToken = struct {
    value: []u8, // decoded value (escape sequences processed)
    lexeme: []const u8, // raw source
    is_multiline: bool,
    allocator: std.mem.Allocator,
    
    pub fn deinit(self: *StringToken) void {
        self.allocator.free(self.value);
    }
};

/// Scan string literal ที่เริ่มต้น (ทันทีหลัง opening quote)
pub fn scanString(
    source: []const u8,
    pos: *usize,
    quote: u8,
    line: *u32,
    allocator: std.mem.Allocator,
) !StringToken {
    const start = pos.*;
    var value = std.ArrayList(u8).init(allocator);
    errdefer value.deinit();
    
    var is_multiline = false;
    
    // ตรวจสอบ triple quote
    if (pos.* + 1 < source.len and source[pos.*] == quote and source[pos.* + 1] == quote) {
        pos.* += 2;
        is_multiline = true;
        
        // Scan จนพบ triple quote ปิด
        while (pos.* < source.len) {
            if (source[pos.*] == quote and
                pos.* + 1 < source.len and source[pos.* + 1] == quote and
                pos.* + 2 < source.len and source[pos.* + 2] == quote)
            {
                pos.* += 3;
                break;
            }
            
            if (source[pos.*] == '\n') line.* += 1;
            try value.append(source[pos.*]);
            pos.* += 1;
        }
    } else {
        // Single-line string
        while (pos.* < source.len) {
            const c = source[pos.*];
            
            if (c == '\n' or c == '\r') {
                return error.UnterminatedString;
            }
            
            if (c == quote) {
                pos.* += 1;
                break;
            }
            
            if (c == '\\') {
                // Escape sequence
                pos.* += 1;
                if (pos.* >= source.len) return error.UnterminatedString;
                
                const escaped = source[pos.*];
                pos.* += 1;
                
                switch (escaped) {
                    'n'  => try value.append('\n'),
                    't'  => try value.append('\t'),
                    'r'  => try value.append('\r'),
                    '\\' => try value.append('\\'),
                    '\'' => try value.append('\''),
                    '"'  => try value.append('"'),
                    '0'  => try value.append(0),
                    'a'  => try value.append(7),   // bell
                    'b'  => try value.append(8),   // backspace
                    'f'  => try value.append(12),  // form feed
                    'v'  => try value.append(11),  // vertical tab
                    
                    // Hex escape: \xNN
                    'x' => {
                        if (pos.* + 1 >= source.len) return error.InvalidEscape;
                        const hi = std.fmt.charToDigit(source[pos.*], 16) catch return error.InvalidEscape;
                        const lo = std.fmt.charToDigit(source[pos.* + 1], 16) catch return error.InvalidEscape;
                        try value.append(@intCast(hi * 16 + lo));
                        pos.* += 2;
                    },
                    
                    // Unicode escape: \uNNNN
                    'u' => {
                        if (pos.* + 3 >= source.len) return error.InvalidEscape;
                        var codepoint: u32 = 0;
                        for (0..4) |_| {
                            const digit = std.fmt.charToDigit(source[pos.*], 16) catch return error.InvalidEscape;
                            codepoint = codepoint * 16 + digit;
                            pos.* += 1;
                        }
                        var buf: [4]u8 = undefined;
                        const len = std.unicode.utf8Encode(@intCast(codepoint), &buf) catch return error.InvalidEscape;
                        try value.appendSlice(buf[0..len]);
                    },
                    
                    else => {
                        // Unknown escape - include literally
                        try value.append('\\');
                        try value.append(escaped);
                    },
                }
            } else {
                try value.append(c);
                pos.* += 1;
            }
        }
    }
    
    return .{
        .value = try value.toOwnedSlice(),
        .lexeme = source[start..pos.*],
        .is_multiline = is_multiline,
        .allocator = allocator,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Scanning Strings ===\n\n", .{});
    
    const sources = [_][]const u8{
        "hello world\"",           // regular string (หลัง opening ")
        "hello\\nworld\"",         // with newline escape
        "tab\\there\"",            // with tab
        "unicode\\u0041\"",        // unicode A
        "hex\\x41\"",              // hex A
    };
    
    for (sources) |src| {
        var pos: usize = 0;
        var line: u32 = 1;
        var tok = scanString(src, &pos, '"', &line, allocator) catch |err| {
            try stdout.print("Error: {} for '{s}'\n", .{ err, src });
            continue;
        };
        defer tok.deinit();
        
        try stdout.print("Input: \"{s}\n", .{src});
        try stdout.print("Value: [{s}] (len={d})\n\n", .{ tok.value, tok.value.len });
    }
}
```

---

## Step 846: Scanning Identifiers และ Keywords

```zig
// scan_identifiers.zig
const std = @import("std");

const keywords = std.StaticStringMap([]const u8).initComptime(.{
    .{ "if",       "keyword_if" },
    .{ "else",     "keyword_else" },
    .{ "while",    "keyword_while" },
    .{ "for",      "keyword_for" },
    .{ "fn",       "keyword_fn" },
    .{ "return",   "keyword_return" },
    .{ "let",      "keyword_let" },
    .{ "const",    "keyword_const" },
    .{ "true",     "keyword_true" },
    .{ "false",    "keyword_false" },
    .{ "null",     "keyword_null" },
    .{ "and",      "keyword_and" },
    .{ "or",       "keyword_or" },
    .{ "not",      "keyword_not" },
    .{ "struct",   "keyword_struct" },
    .{ "enum",     "keyword_enum" },
    .{ "import",   "keyword_import" },
    .{ "break",    "keyword_break" },
    .{ "continue", "keyword_continue" },
});

/// ตรวจสอบว่าเป็น identifier start character
fn isIdentStart(c: u8) bool {
    return std.ascii.isAlphabetic(c) or c == '_';
}

/// ตรวจสอบว่าเป็น identifier continue character
fn isIdentContinue(c: u8) bool {
    return std.ascii.isAlphanumeric(c) or c == '_';
}

pub const IdentifierResult = struct {
    token_type: []const u8, // "identifier" หรือ keyword name
    lexeme: []const u8,
};

/// Scan identifier หรือ keyword
pub fn scanIdentifier(source: []const u8, pos: *usize) IdentifierResult {
    const start = pos.*;
    
    // Scan identifier characters
    while (pos.* < source.len and isIdentContinue(source[pos.*])) {
        pos.* += 1;
    }
    
    const lexeme = source[start..pos.*];
    
    // ตรวจสอบว่าเป็น keyword
    const token_type = if (keywords.get(lexeme)) |kw| kw else "identifier";
    
    return .{ .token_type = token_type, .lexeme = lexeme };
}

/// Scan identifier พร้อมรองรับ Unicode (UTF-8)
pub fn scanIdentifierUnicode(source: []const u8, pos: *usize) ![]const u8 {
    const start = pos.*;
    
    while (pos.* < source.len) {
        const c = source[pos.*];
        
        // ASCII identifiers
        if (isIdentContinue(c)) {
            pos.* += 1;
            continue;
        }
        
        // UTF-8 multibyte (ตัวอักษรไทย, etc.)
        if (c >= 0x80) {
            // หาความยาว UTF-8 sequence
            const seqLen: usize = if (c & 0xF8 == 0xF0) 4
                                  else if (c & 0xF0 == 0xE0) 3
                                  else if (c & 0xE0 == 0xC0) 2
                                  else 1;
            
            if (pos.* + seqLen > source.len) break;
            pos.* += seqLen;
            continue;
        }
        
        break;
    }
    
    return source[start..pos.*];
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Scanning Identifiers ===\n\n", .{});
    
    const sources = [_][]const u8{
        "if", "else", "myVariable", "_private", "count123",
        "let", "x", "maxValue", "true", "false",
        "my_func_1", "CamelCase",
    };
    
    for (sources) |src| {
        var pos: usize = 0;
        const result = scanIdentifier(src, &pos);
        try stdout.print("'{s}' -> {s}\n", .{ src, result.token_type });
    }
    
    // ทดสอบ Unicode identifiers
    try stdout.print("\nUnicode Identifiers:\n", .{});
    const unicode_sources = [_][]const u8{
        "ตัวแปร",
        "ฟังก์ชัน",
        "นับจำนวน",
    };
    
    for (unicode_sources) |src| {
        var pos: usize = 0;
        const ident = try scanIdentifierUnicode(src, &pos);
        try stdout.print("'{s}' -> identifier (len={d} bytes)\n", .{ ident, ident.len });
    }
}
```

---

## Step 847: Scanning Operators

```zig
// scan_operators.zig
const std = @import("std");

pub const OpToken = struct {
    kind: []const u8,
    lexeme: []const u8,
};

/// Scan operator (รองรับ multi-character operators)
pub fn scanOperator(source: []const u8, pos: *usize) ?OpToken {
    if (pos.* >= source.len) return null;
    
    const start = pos.*;
    const c = source[pos.*];
    pos.* += 1;
    
    const next: u8 = if (pos.* < source.len) source[pos.*] else 0;
    
    switch (c) {
        '+' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "plus_equal",   .lexeme = source[start..pos.*] }; }
            if (next == '+') { pos.* += 1; return .{ .kind = "plus_plus",    .lexeme = source[start..pos.*] }; }
            return .{ .kind = "plus", .lexeme = source[start..pos.*] };
        },
        '-' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "minus_equal",  .lexeme = source[start..pos.*] }; }
            if (next == '-') { pos.* += 1; return .{ .kind = "minus_minus",  .lexeme = source[start..pos.*] }; }
            if (next == '>') { pos.* += 1; return .{ .kind = "arrow",        .lexeme = source[start..pos.*] }; }
            return .{ .kind = "minus", .lexeme = source[start..pos.*] };
        },
        '*' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "star_equal",   .lexeme = source[start..pos.*] }; }
            if (next == '*') { pos.* += 1; return .{ .kind = "star_star",    .lexeme = source[start..pos.*] }; }
            return .{ .kind = "star", .lexeme = source[start..pos.*] };
        },
        '/' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "slash_equal",  .lexeme = source[start..pos.*] }; }
            if (next == '/') { return .{ .kind = "line_comment",  .lexeme = source[start..pos.*] }; }
            if (next == '*') { return .{ .kind = "block_comment", .lexeme = source[start..pos.*] }; }
            return .{ .kind = "slash", .lexeme = source[start..pos.*] };
        },
        '%' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "percent_equal", .lexeme = source[start..pos.*] }; }
            return .{ .kind = "percent", .lexeme = source[start..pos.*] };
        },
        '=' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "equal_equal",  .lexeme = source[start..pos.*] }; }
            if (next == '>') { pos.* += 1; return .{ .kind = "fat_arrow",    .lexeme = source[start..pos.*] }; }
            return .{ .kind = "equal", .lexeme = source[start..pos.*] };
        },
        '!' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "bang_equal",   .lexeme = source[start..pos.*] }; }
            return .{ .kind = "bang", .lexeme = source[start..pos.*] };
        },
        '<' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "less_equal",   .lexeme = source[start..pos.*] }; }
            if (next == '<') { pos.* += 1; return .{ .kind = "left_shift",   .lexeme = source[start..pos.*] }; }
            return .{ .kind = "less", .lexeme = source[start..pos.*] };
        },
        '>' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "greater_equal", .lexeme = source[start..pos.*] }; }
            if (next == '>') { pos.* += 1;  return .{ .kind = "right_shift",  .lexeme = source[start..pos.*] }; }
            return .{ .kind = "greater", .lexeme = source[start..pos.*] };
        },
        '&' => {
            if (next == '&') { pos.* += 1; return .{ .kind = "and_and",      .lexeme = source[start..pos.*] }; }
            if (next == '=') { pos.* += 1; return .{ .kind = "amp_equal",    .lexeme = source[start..pos.*] }; }
            return .{ .kind = "amp", .lexeme = source[start..pos.*] };
        },
        '|' => {
            if (next == '|') { pos.* += 1; return .{ .kind = "or_or",        .lexeme = source[start..pos.*] }; }
            if (next == '=') { pos.* += 1; return .{ .kind = "pipe_equal",   .lexeme = source[start..pos.*] }; }
            return .{ .kind = "pipe", .lexeme = source[start..pos.*] };
        },
        '(' => return .{ .kind = "lparen",    .lexeme = source[start..pos.*] },
        ')' => return .{ .kind = "rparen",    .lexeme = source[start..pos.*] },
        '{' => return .{ .kind = "lbrace",    .lexeme = source[start..pos.*] },
        '}' => return .{ .kind = "rbrace",    .lexeme = source[start..pos.*] },
        '[' => return .{ .kind = "lbracket",  .lexeme = source[start..pos.*] },
        ']' => return .{ .kind = "rbracket",  .lexeme = source[start..pos.*] },
        ';' => return .{ .kind = "semicolon", .lexeme = source[start..pos.*] },
        ':' => {
            if (next == ':') { pos.* += 1; return .{ .kind = "colon_colon",  .lexeme = source[start..pos.*] }; }
            return .{ .kind = "colon", .lexeme = source[start..pos.*] };
        },
        ',' => return .{ .kind = "comma",     .lexeme = source[start..pos.*] },
        '.' => {
            if (next == '.') {
                pos.* += 1;
                if (pos.* < source.len and source[pos.*] == '.') {
                    pos.* += 1;
                    return .{ .kind = "ellipsis", .lexeme = source[start..pos.*] };
                }
                return .{ .kind = "dotdot", .lexeme = source[start..pos.*] };
            }
            return .{ .kind = "dot", .lexeme = source[start..pos.*] };
        },
        '?' => return .{ .kind = "question",  .lexeme = source[start..pos.*] },
        '@' => return .{ .kind = "at",        .lexeme = source[start..pos.*] },
        '#' => return .{ .kind = "hash",      .lexeme = source[start..pos.*] },
        '^' => {
            if (next == '=') { pos.* += 1; return .{ .kind = "caret_equal",  .lexeme = source[start..pos.*] }; }
            return .{ .kind = "caret", .lexeme = source[start..pos.*] };
        },
        '~' => return .{ .kind = "tilde",     .lexeme = source[start..pos.*] },
        else => {
            pos.* -= 1; // ย้อนกลับ
            return null;
        },
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Scanning Operators ===\n\n", .{});
    
    const sources = [_][]const u8{
        "+", "-", "*", "/", "%",
        "==", "!=", "<=", ">=", "->",
        "&&", "||", "<<", ">>",
        "+=", "-=", "*=",
        "(", ")", "{", "}", "[", "]",
        ";", ":", ",", ".", "...", "?",
        "++", "--", "**",
    };
    
    for (sources) |src| {
        var pos: usize = 0;
        if (scanOperator(src, &pos)) |tok| {
            try stdout.print("'{s}' -> {s}\n", .{ src, tok.kind });
        } else {
            try stdout.print("'{s}' -> unknown\n", .{src});
        }
    }
}
```

---

## Step 848: Error Handling ใน Lexer

```zig
// lexer_errors.zig
const std = @import("std");

pub const ErrorKind = enum {
    unexpected_character,
    unterminated_string,
    unterminated_comment,
    invalid_number,
    invalid_escape,
    invalid_unicode,
    number_overflow,
};

pub const LexerDiagnostic = struct {
    kind: ErrorKind,
    message: []const u8,
    line: u32,
    column: u32,
    length: u32,  // ความยาวของ token ที่มีปัญหา
    source_line: []const u8, // บรรทัดที่มีปัญหา
    
    pub fn format(
        self: LexerDiagnostic,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("[Line {d}, Col {d}] {s}: {s}", .{
            self.line, self.column, @tagName(self.kind), self.message
        });
    }
};

/// Error recovery strategy
pub const RecoveryStrategy = enum {
    skip_character,      // ข้าม character ที่มีปัญหา
    skip_to_newline,     // ข้ามไปถึงบรรทัดใหม่
    skip_to_delimiter,   // ข้ามไปถึง delimiter ถัดไป
    insert_token,        // แทรก token ที่หายไป
};

/// Lexer ที่มี error recovery
pub const ResilientLexer = struct {
    source: []const u8,
    pos: usize,
    line: u32,
    col: u32,
    diagnostics: std.ArrayList(LexerDiagnostic),
    allocator: std.mem.Allocator,
    
    pub fn init(source: []const u8, allocator: std.mem.Allocator) ResilientLexer {
        return .{
            .source = source,
            .pos = 0,
            .line = 1,
            .col = 1,
            .diagnostics = std.ArrayList(LexerDiagnostic).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *ResilientLexer) void {
        self.diagnostics.deinit();
    }
    
    pub fn addDiagnostic(self: *ResilientLexer, kind: ErrorKind, msg: []const u8, length: u32) !void {
        // หา source line
        var lineStart: usize = 0;
        var lineEnd: usize = self.source.len;
        var l: u32 = 1;
        
        for (self.source, 0..) |c, i| {
            if (l == self.line) {
                lineStart = if (i == 0 or self.source[i - 1] == '\n') i else lineStart;
            }
            if (c == '\n') {
                if (l == self.line) {
                    lineEnd = i;
                    break;
                }
                l += 1;
            }
        }
        
        try self.diagnostics.append(.{
            .kind = kind,
            .message = msg,
            .line = self.line,
            .column = self.col,
            .length = length,
            .source_line = self.source[lineStart..lineEnd],
        });
    }
    
    /// แสดง diagnostics พร้อม source context
    pub fn printDiagnostics(self: *const ResilientLexer) !void {
        const stdout = std.io.getStdOut().writer();
        
        for (self.diagnostics.items) |diag| {
            try stdout.print("\nError: {}\n", .{diag});
            try stdout.print("  | {s}\n", .{diag.source_line});
            
            // แสดงลูกศรชี้
            try stdout.print("  | ", .{});
            for (0..diag.column - 1) |_| try stdout.print(" ", .{});
            for (0..@max(diag.length, 1)) |_| try stdout.print("^", .{});
            try stdout.print("\n", .{});
        }
    }
    
    /// Recovery: ข้ามไปถึง delimiter
    pub fn recoverToDelimiter(self: *ResilientLexer, delimiters: []const u8) void {
        while (self.pos < self.source.len) {
            const c = self.source[self.pos];
            for (delimiters) |d| {
                if (c == d) return;
            }
            if (c == '\n') {
                self.line += 1;
                self.col = 1;
            } else {
                self.col += 1;
            }
            self.pos += 1;
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Lexer Error Handling ===\n\n", .{});
    
    // ตัวอย่าง source ที่มี errors
    const source = "let x = 42;\nlet y = @#$;\nlet z = \"unterminated\n";
    
    try stdout.print("Source:\n{s}\n", .{source});
    
    var lexer = ResilientLexer.init(source, allocator);
    defer lexer.deinit();
    
    // จำลอง errors
    lexer.pos = 12;
    lexer.line = 2;
    lexer.col = 9;
    try lexer.addDiagnostic(.unexpected_character, "unexpected '@'", 1);
    
    lexer.pos = 13;
    lexer.col = 10;
    try lexer.addDiagnostic(.unexpected_character, "unexpected '#'", 1);
    
    lexer.pos = source.len - 1;
    lexer.line = 3;
    lexer.col = 10;
    try lexer.addDiagnostic(.unterminated_string, "string literal not closed before end of line", 12);
    
    try stdout.print("Diagnostics:\n", .{});
    try lexer.printDiagnostics();
    
    try stdout.print("\nError Recovery Strategies:\n", .{});
    try stdout.print("  1. skip_character - ข้าม char ที่ไม่รู้จัก แล้วทำต่อ\n", .{});
    try stdout.print("  2. skip_to_newline - ข้ามทั้งบรรทัด\n", .{});
    try stdout.print("  3. skip_to_delimiter - ข้ามไปถึง ; หรือ }}\n", .{});
    try stdout.print("  4. insert_token - แทรก token ที่คาดว่าหายไป\n", .{});
}
```

---

## Step 849: Lexer Testing

```zig
// lexer_test.zig
const std = @import("std");
const testing = std.testing;

// Simple test lexer
const TestToken = struct {
    kind: []const u8,
    lexeme: []const u8,
};

fn tokenize(source: []const u8, allocator: std.mem.Allocator) ![]TestToken {
    var tokens = std.ArrayList(TestToken).init(allocator);
    
    var i: usize = 0;
    while (i < source.len) {
        // Skip whitespace
        while (i < source.len and (source[i] == ' ' or source[i] == '\t' or source[i] == '\n')) {
            i += 1;
        }
        if (i >= source.len) break;
        
        const c = source[i];
        
        // Number
        if (std.ascii.isDigit(c)) {
            const start = i;
            var is_float = false;
            while (i < source.len and std.ascii.isDigit(source[i])) i += 1;
            if (i < source.len and source[i] == '.') {
                is_float = true;
                i += 1;
                while (i < source.len and std.ascii.isDigit(source[i])) i += 1;
            }
            _ = is_float;
            try tokens.append(.{ .kind = "number", .lexeme = source[start..i] });
            continue;
        }
        
        // String
        if (c == '"') {
            i += 1;
            const start = i;
            while (i < source.len and source[i] != '"') i += 1;
            try tokens.append(.{ .kind = "string", .lexeme = source[start..i] });
            if (i < source.len) i += 1;
            continue;
        }
        
        // Identifier/keyword
        if (std.ascii.isAlphabetic(c) or c == '_') {
            const start = i;
            while (i < source.len and (std.ascii.isAlphanumeric(source[i]) or source[i] == '_')) i += 1;
            const lexeme = source[start..i];
            
            const keywords_list = [_][]const u8{ "let", "fn", "if", "else", "while", "return", "true", "false" };
            var is_kw = false;
            for (keywords_list) |kw| {
                if (std.mem.eql(u8, lexeme, kw)) { is_kw = true; break; }
            }
            
            try tokens.append(.{ .kind = if (is_kw) "keyword" else "ident", .lexeme = lexeme });
            continue;
        }
        
        // Single char operators
        const kind = switch (c) {
            '+' => "plus",  '-' => "minus", '*' => "star",  '/' => "slash",
            '(' => "lparen",')'  => "rparen",'{' => "lbrace",'}' => "rbrace",
            ';' => "semi",  ',' => "comma", '=' => "equal", '!' => "bang",
            '<' => "lt",    '>' => "gt",
            else => "unknown",
        };
        try tokens.append(.{ .kind = kind, .lexeme = source[i..i+1] });
        i += 1;
    }
    
    try tokens.append(.{ .kind = "eof", .lexeme = "" });
    return tokens.toOwnedSlice();
}

test "tokenize simple expression" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const source = "let x = 42;";
    const tokens = try tokenize(source, allocator);
    
    try testing.expectEqual(@as(usize, 6), tokens.len); // let, x, =, 42, ;, eof
    try testing.expectEqualStrings("keyword", tokens[0].kind);
    try testing.expectEqualStrings("let", tokens[0].lexeme);
    try testing.expectEqualStrings("ident", tokens[1].kind);
    try testing.expectEqualStrings("x", tokens[1].lexeme);
    try testing.expectEqualStrings("number", tokens[3].kind);
    try testing.expectEqualStrings("42", tokens[3].lexeme);
}

test "tokenize function" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const source = "fn add(a, b) { return a + b; }";
    const tokens = try tokenize(source, allocator);
    
    try testing.expect(tokens.len > 0);
    try testing.expectEqualStrings("keyword", tokens[0].kind);
    try testing.expectEqualStrings("fn", tokens[0].lexeme);
}

test "tokenize string literal" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const source = "let msg = \"hello world\";";
    const tokens = try tokenize(source, allocator);
    
    // ค้นหา string token
    var found = false;
    for (tokens) |tok| {
        if (std.mem.eql(u8, tok.kind, "string")) {
            try testing.expectEqualStrings("hello world", tok.lexeme);
            found = true;
            break;
        }
    }
    try testing.expect(found);
}

test "tokenize numbers" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const cases = [_][]const u8{ "42", "3.14", "0", "1000" };
    for (cases) |num| {
        const tokens = try tokenize(num, allocator);
        try testing.expectEqualStrings("number", tokens[0].kind);
        try testing.expectEqualStrings(num, tokens[0].lexeme);
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Lexer Testing ===\n\n", .{});
    try stdout.print("รัน tests ด้วย: zig test lexer_test.zig\n\n", .{});
    
    // แสดงตัวอย่าง test cases
    const testCases = [_]struct {
        source: []const u8,
        desc: []const u8,
    }{
        .{ .source = "let x = 42;", .desc = "Variable declaration" },
        .{ .source = "fn add(a, b) { return a + b; }", .desc = "Function definition" },
        .{ .source = "if (x > 0) { true } else { false }", .desc = "Conditional" },
        .{ .source = "let msg = \"hello\";", .desc = "String literal" },
        .{ .source = "while (i < 10) { i = i + 1; }", .desc = "While loop" },
    };
    
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    for (testCases) |tc| {
        try stdout.print("Test: {s}\n", .{tc.desc});
        try stdout.print("  Source: {s}\n", .{tc.source});
        
        const tokens = try tokenize(tc.source, allocator);
        defer allocator.free(tokens);
        
        try stdout.print("  Tokens: ", .{});
        for (tokens) |tok| {
            if (!std.mem.eql(u8, tok.kind, "eof")) {
                try stdout.print("[{s}:{s}] ", .{ tok.kind, tok.lexeme });
            }
        }
        try stdout.print("\n\n", .{});
    }
}
```

---

## Step 850: Complete Lexer สำหรับ Simple Language

### Complete Lexer Implementation

```zig
// complete_lexer.zig
const std = @import("std");

pub const TokenKind = enum {
    // Literals
    integer, float, string, char,
    // Keywords
    kw_let, kw_fn, kw_if, kw_else, kw_while, kw_for, kw_in,
    kw_return, kw_true, kw_false, kw_null, kw_and, kw_or,
    kw_break, kw_continue, kw_struct, kw_import,
    // Identifiers
    identifier,
    // Operators
    plus, minus, star, slash, percent,
    equal, eq_eq, bang, bang_eq,
    lt, lt_eq, gt, gt_eq,
    amp_amp, pipe_pipe,
    // Assignments
    plus_eq, minus_eq, star_eq, slash_eq,
    // Delimiters
    lparen, rparen, lbrace, rbrace, lbracket, rbracket,
    semicolon, colon, colon_colon, comma, dot, dotdot,
    arrow, fat_arrow, question,
    // Special
    eof,
    error_tok,
};

pub const Token = struct {
    kind: TokenKind,
    lexeme: []const u8,
    line: u32,
    col: u32,
    
    // Literal values
    int_val: ?i64 = null,
    float_val: ?f64 = null,
    str_val: ?[]u8 = null, // decoded string
};

const keywords_map = std.StaticStringMap(TokenKind).initComptime(.{
    .{ "let",      .kw_let },
    .{ "fn",       .kw_fn },
    .{ "if",       .kw_if },
    .{ "else",     .kw_else },
    .{ "while",    .kw_while },
    .{ "for",      .kw_for },
    .{ "in",       .kw_in },
    .{ "return",   .kw_return },
    .{ "true",     .kw_true },
    .{ "false",    .kw_false },
    .{ "null",     .kw_null },
    .{ "and",      .kw_and },
    .{ "or",       .kw_or },
    .{ "break",    .kw_break },
    .{ "continue", .kw_continue },
    .{ "struct",   .kw_struct },
    .{ "import",   .kw_import },
});

pub const CompleteLexer = struct {
    source: []const u8,
    pos: usize,
    line: u32,
    col: u32,
    tokens: std.ArrayList(Token),
    errors: std.ArrayList(struct { msg: []const u8, line: u32, col: u32 }),
    allocator: std.mem.Allocator,
    
    pub fn init(source: []const u8, allocator: std.mem.Allocator) CompleteLexer {
        return .{
            .source = source,
            .pos = 0,
            .line = 1,
            .col = 1,
            .tokens = std.ArrayList(Token).init(allocator),
            .errors = std.ArrayList(struct { msg: []const u8, line: u32, col: u32 }).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *CompleteLexer) void {
        // Free string values
        for (self.tokens.items) |tok| {
            if (tok.str_val) |s| self.allocator.free(s);
        }
        self.tokens.deinit();
        self.errors.deinit();
    }
    
    fn peek(self: *const CompleteLexer) u8 {
        if (self.pos >= self.source.len) return 0;
        return self.source[self.pos];
    }
    
    fn peekNext(self: *const CompleteLexer) u8 {
        if (self.pos + 1 >= self.source.len) return 0;
        return self.source[self.pos + 1];
    }
    
    fn advance(self: *CompleteLexer) u8 {
        const c = self.source[self.pos];
        self.pos += 1;
        if (c == '\n') { self.line += 1; self.col = 1; }
        else { self.col += 1; }
        return c;
    }
    
    fn match(self: *CompleteLexer, expected: u8) bool {
        if (self.pos >= self.source.len or self.source[self.pos] != expected) return false;
        _ = self.advance();
        return true;
    }
    
    fn makeToken(self: *CompleteLexer, kind: TokenKind, start: usize, startLine: u32, startCol: u32) Token {
        return .{
            .kind = kind,
            .lexeme = self.source[start..self.pos],
            .line = startLine,
            .col = startCol,
        };
    }
    
    fn addError(self: *CompleteLexer, msg: []const u8) !void {
        try self.errors.append(.{ .msg = msg, .line = self.line, .col = self.col });
    }
    
    fn scanNumber(self: *CompleteLexer, start: usize, startLine: u32, startCol: u32) !Token {
        var is_float = false;
        
        while (self.pos < self.source.len and std.ascii.isDigit(self.peek())) _ = self.advance();
        
        if (self.peek() == '.' and std.ascii.isDigit(self.peekNext())) {
            is_float = true;
            _ = self.advance();
            while (self.pos < self.source.len and std.ascii.isDigit(self.peek())) _ = self.advance();
        }
        
        if (self.peek() == 'e' or self.peek() == 'E') {
            is_float = true;
            _ = self.advance();
            if (self.peek() == '+' or self.peek() == '-') _ = self.advance();
            while (self.pos < self.source.len and std.ascii.isDigit(self.peek())) _ = self.advance();
        }
        
        const lexeme = self.source[start..self.pos];
        
        if (is_float) {
            const val = std.fmt.parseFloat(f64, lexeme) catch {
                try self.addError("invalid float literal");
                return self.makeToken(.error_tok, start, startLine, startCol);
            };
            var tok = self.makeToken(.float, start, startLine, startCol);
            tok.float_val = val;
            return tok;
        } else {
            const val = std.fmt.parseInt(i64, lexeme, 10) catch {
                try self.addError("integer overflow or invalid integer");
                return self.makeToken(.error_tok, start, startLine, startCol);
            };
            var tok = self.makeToken(.integer, start, startLine, startCol);
            tok.int_val = val;
            return tok;
        }
    }
    
    fn scanString(self: *CompleteLexer, start: usize, startLine: u32, startCol: u32) !Token {
        var value = std.ArrayList(u8).init(self.allocator);
        errdefer value.deinit();
        
        while (self.pos < self.source.len and self.peek() != '"') {
            if (self.peek() == '\n') {
                try self.addError("unterminated string literal");
                break;
            }
            
            if (self.peek() == '\\') {
                _ = self.advance();
                switch (self.advance()) {
                    'n' => try value.append('\n'),
                    't' => try value.append('\t'),
                    'r' => try value.append('\r'),
                    '\\' => try value.append('\\'),
                    '"' => try value.append('"'),
                    '0' => try value.append(0),
                    else => |c| {
                        try self.addError("unknown escape sequence");
                        try value.append('\\');
                        try value.append(c);
                    },
                }
            } else {
                try value.append(self.advance());
            }
        }
        
        if (self.pos < self.source.len) _ = self.advance(); // closing "
        
        var tok = self.makeToken(.string, start, startLine, startCol);
        tok.str_val = try value.toOwnedSlice();
        return tok;
    }
    
    fn scanIdentOrKeyword(self: *CompleteLexer, start: usize, startLine: u32, startCol: u32) Token {
        while (self.pos < self.source.len and
               (std.ascii.isAlphanumeric(self.peek()) or self.peek() == '_'))
        {
            _ = self.advance();
        }
        
        const lexeme = self.source[start..self.pos];
        const kind = if (keywords_map.get(lexeme)) |kw| kw else TokenKind.identifier;
        return self.makeToken(kind, start, startLine, startCol);
    }
    
    fn skipLineComment(self: *CompleteLexer) void {
        while (self.pos < self.source.len and self.peek() != '\n') _ = self.advance();
    }
    
    fn skipBlockComment(self: *CompleteLexer) !void {
        while (self.pos + 1 < self.source.len) {
            if (self.peek() == '*' and self.peekNext() == '/') {
                _ = self.advance();
                _ = self.advance();
                return;
            }
            _ = self.advance();
        }
        try self.addError("unterminated block comment");
    }
    
    pub fn tokenize(self: *CompleteLexer) ![]Token {
        while (self.pos < self.source.len) {
            // Skip whitespace
            while (self.pos < self.source.len) {
                const c = self.peek();
                if (c == ' ' or c == '\t' or c == '\r' or c == '\n') {
                    _ = self.advance();
                } else break;
            }
            
            if (self.pos >= self.source.len) break;
            
            const startPos = self.pos;
            const startLine = self.line;
            const startCol = self.col;
            const c = self.advance();
            
            const tok: Token = switch (c) {
                // Numbers
                '0'...'9' => blk: {
                    self.pos -= 1;
                    self.col -= 1;
                    break :blk try self.scanNumber(startPos, startLine, startCol);
                },
                
                // Strings
                '"' => try self.scanString(startPos, startLine, startCol),
                
                // Identifiers/Keywords
                'a'...'z', 'A'...'Z', '_' => blk: {
                    self.pos -= 1;
                    self.col -= 1;
                    break :blk self.scanIdentOrKeyword(startPos, startLine, startCol);
                },
                
                // Comments
                '/' => blk: {
                    if (self.match('/')) {
                        self.skipLineComment();
                        continue; // ไม่สร้าง token
                    } else if (self.match('*')) {
                        try self.skipBlockComment();
                        continue;
                    } else if (self.match('=')) {
                        break :blk self.makeToken(.slash_eq, startPos, startLine, startCol);
                    }
                    break :blk self.makeToken(.slash, startPos, startLine, startCol);
                },
                
                // Operators
                '+' => if (self.match('=')) self.makeToken(.plus_eq, startPos, startLine, startCol)
                       else self.makeToken(.plus, startPos, startLine, startCol),
                '-' => blk: {
                    if (self.match('=')) break :blk self.makeToken(.minus_eq, startPos, startLine, startCol);
                    if (self.match('>')) break :blk self.makeToken(.arrow, startPos, startLine, startCol);
                    break :blk self.makeToken(.minus, startPos, startLine, startCol);
                },
                '*' => if (self.match('=')) self.makeToken(.star_eq, startPos, startLine, startCol)
                       else self.makeToken(.star, startPos, startLine, startCol),
                '%' => self.makeToken(.percent, startPos, startLine, startCol),
                '=' => blk: {
                    if (self.match('=')) break :blk self.makeToken(.eq_eq, startPos, startLine, startCol);
                    if (self.match('>')) break :blk self.makeToken(.fat_arrow, startPos, startLine, startCol);
                    break :blk self.makeToken(.equal, startPos, startLine, startCol);
                },
                '!' => if (self.match('=')) self.makeToken(.bang_eq, startPos, startLine, startCol)
                       else self.makeToken(.bang, startPos, startLine, startCol),
                '<' => if (self.match('=')) self.makeToken(.lt_eq, startPos, startLine, startCol)
                       else self.makeToken(.lt, startPos, startLine, startCol),
                '>' => if (self.match('=')) self.makeToken(.gt_eq, startPos, startLine, startCol)
                       else self.makeToken(.gt, startPos, startLine, startCol),
                '&' => if (self.match('&')) self.makeToken(.amp_amp, startPos, startLine, startCol)
                       else blk: {
                           try self.addError("expected '&&'");
                           break :blk self.makeToken(.error_tok, startPos, startLine, startCol);
                       },
                '|' => if (self.match('|')) self.makeToken(.pipe_pipe, startPos, startLine, startCol)
                       else blk: {
                           try self.addError("expected '||'");
                           break :blk self.makeToken(.error_tok, startPos, startLine, startCol);
                       },
                
                // Delimiters
                '(' => self.makeToken(.lparen,    startPos, startLine, startCol),
                ')' => self.makeToken(.rparen,    startPos, startLine, startCol),
                '{' => self.makeToken(.lbrace,    startPos, startLine, startCol),
                '}' => self.makeToken(.rbrace,    startPos, startLine, startCol),
                '[' => self.makeToken(.lbracket,  startPos, startLine, startCol),
                ']' => self.makeToken(.rbracket,  startPos, startLine, startCol),
                ';' => self.makeToken(.semicolon, startPos, startLine, startCol),
                ':' => if (self.match(':')) self.makeToken(.colon_colon, startPos, startLine, startCol)
                       else self.makeToken(.colon, startPos, startLine, startCol),
                ',' => self.makeToken(.comma,     startPos, startLine, startCol),
                '.' => if (self.match('.')) self.makeToken(.dotdot, startPos, startLine, startCol)
                       else self.makeToken(.dot, startPos, startLine, startCol),
                '?' => self.makeToken(.question,  startPos, startLine, startCol),
                
                else => blk: {
                    try self.addError("unexpected character");
                    break :blk self.makeToken(.error_tok, startPos, startLine, startCol);
                },
            };
            
            try self.tokens.append(tok);
        }
        
        // เพิ่ม EOF token
        try self.tokens.append(.{
            .kind = .eof,
            .lexeme = "",
            .line = self.line,
            .col = self.col,
        });
        
        return self.tokens.items;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    // โปรแกรมตัวอย่าง
    const source =
        \\// Simple calculator
        \\fn add(a, b) {
        \\    return a + b;
        \\}
        \\
        \\fn main() {
        \\    let x = 10;
        \\    let y = 3.14;
        \\    let msg = "result";
        \\    let sum = add(x, 5);
        \\    if (sum > 10) {
        \\        return true;
        \\    } else {
        \\        return false;
        \\    }
        \\}
    ;
    
    try stdout.print("=== Complete Lexer ===\n\n", .{});
    try stdout.print("Source:\n{s}\n\n", .{source});
    
    var lexer = CompleteLexer.init(source, allocator);
    defer lexer.deinit();
    
    const tokens = try lexer.tokenize();
    
    try stdout.print("Tokens ({d} total):\n", .{tokens.len});
    try stdout.print("{s:<4} {s:<20} {s:<15} {s:<6} {s:<6}\n", .{
        "#", "kind", "lexeme", "line", "col"
    });
    try stdout.print("{s}\n", .{"-" ** 55});
    
    for (tokens, 0..) |tok, i| {
        const lexeme = if (tok.lexeme.len > 15) tok.lexeme[0..15] else tok.lexeme;
        try stdout.print("{d:<4} {s:<20} {s:<15} {d:<6} {d:<6}", .{
            i, @tagName(tok.kind), lexeme, tok.line, tok.col
        });
        
        if (tok.int_val) |v| try stdout.print(" = {d}", .{v});
        if (tok.float_val) |v| try stdout.print(" = {d}", .{v});
        if (tok.str_val) |v| try stdout.print(" = \"{s}\"", .{v});
        
        try stdout.print("\n", .{});
    }
    
    if (lexer.errors.items.len > 0) {
        try stdout.print("\nErrors:\n", .{});
        for (lexer.errors.items) |err| {
            try stdout.print("  Line {d}, Col {d}: {s}\n", .{ err.line, err.col, err.msg });
        }
    } else {
        try stdout.print("\nNo errors!\n", .{});
    }
    
    // สรุป token statistics
    var counts = std.AutoHashMap(TokenKind, usize).init(allocator);
    defer counts.deinit();
    
    for (tokens) |tok| {
        const entry = try counts.getOrPutValue(tok.kind, 0);
        entry.value_ptr.* += 1;
    }
    
    try stdout.print("\nToken Statistics:\n", .{});
    var it = counts.iterator();
    while (it.next()) |entry| {
        try stdout.print("  {s}: {d}\n", .{ @tagName(entry.key_ptr.*), entry.value_ptr.* });
    }
}
```

### สรุป Part 85

ใน Part นี้เราได้สร้าง Lexer สมบูรณ์:
1. **Lexer Concepts** - ทำความเข้าใจ pipeline และหลักการ
2. **Token Types** - ออกแบบ token type system
3. **Lexer Struct** - โครงสร้างหลักของ Lexer
4. **Scanning Numbers** - int, float, hex, octal, binary
5. **Scanning Strings** - escape sequences, unicode
6. **Scanning Identifiers** - keywords lookup, unicode support
7. **Scanning Operators** - single และ multi-character operators
8. **Error Handling** - diagnostics, error recovery
9. **Testing** - unit tests และ integration tests
10. **Complete Lexer** - implementation สมบูรณ์พร้อมใช้งาน

### ขั้นตอนต่อไป

หลังจาก Lexer แล้ว จะต้องสร้าง:
1. **Parser** - แปลง tokens เป็น AST
2. **AST Nodes** - โครงสร้างข้อมูล AST
3. **Recursive Descent Parser**
4. **Pratt Parser** (operator precedence)
5. **Semantic Analyzer**
6. **Code Generator**

---

*ต่อไป: Part 86 - Parsing: Parser Implementation*
