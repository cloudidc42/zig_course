# Part 37: String Handling ขั้นสูงใน Zig

## บทนำ

การจัดการ String ใน Zig แตกต่างจากภาษาอื่นๆ เพราะ Zig ไม่มี built-in String type แต่ใช้ `[]u8` หรือ `[]const u8` แทน บทนี้จะครอบคลุมเทคนิคและ utilities สำหรับการทำงานกับ string ขั้นสูง

---

## Step 361: std.ascii Utilities

### พื้นฐาน std.ascii

```zig
const std = @import("std");
const ascii = std.ascii;

pub fn main() void {
    // ตรวจสอบประเภทของ character
    const c1: u8 = 'A';
    const c2: u8 = '5';
    const c3: u8 = ' ';

    std.debug.print("=== Character Classification ===\n", .{});
    std.debug.print("isAlpha('A'): {}\n", .{ascii.isAlpha(c1)});
    std.debug.print("isDigit('5'): {}\n", .{ascii.isDigit(c2)});
    std.debug.print("isWhitespace(' '): {}\n", .{ascii.isWhitespace(c3)});
    std.debug.print("isAlphanumeric('A'): {}\n", .{ascii.isAlphanumeric(c1)});
    std.debug.print("isUpper('A'): {}\n", .{ascii.isUpper(c1)});
    std.debug.print("isLower('a'): {}\n", .{ascii.isLower('a')});
    std.debug.print("isPrint('A'): {}\n", .{ascii.isPrint(c1)});
    std.debug.print("isControl('\\n'): {}\n", .{ascii.isControl('\n')});
}
```

### การแปลง Case

```zig
const std = @import("std");
const ascii = std.ascii;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const text = "Hello, World! 123";

    // แปลงเป็น uppercase
    const upper = try ascii.allocUpperString(allocator, text);
    defer allocator.free(upper);
    std.debug.print("Upper: {s}\n", .{upper});

    // แปลงเป็น lowercase
    const lower = try ascii.allocLowerString(allocator, text);
    defer allocator.free(lower);
    std.debug.print("Lower: {s}\n", .{lower});

    // แปลง in-place
    var buffer: [100]u8 = undefined;
    const copy = buffer[0..text.len];
    @memcpy(copy, text);
    ascii.lowerString(copy, copy);
    std.debug.print("Lower in-place: {s}\n", .{copy});

    // แปลง character เดี่ยว
    std.debug.print("toLower('A'): {c}\n", .{ascii.toLower('A')});
    std.debug.print("toUpper('a'): {c}\n", .{ascii.toUpper('a')});
}
```

### String Comparison (case-insensitive)

```zig
const std = @import("std");
const ascii = std.ascii;

pub fn main() void {
    // เปรียบเทียบแบบ case-insensitive
    const s1 = "Hello";
    const s2 = "hello";
    const s3 = "HELLO";
    const s4 = "World";

    std.debug.print("eqlIgnoreCase('Hello', 'hello'): {}\n", .{
        ascii.eqlIgnoreCase(s1, s2),
    });
    std.debug.print("eqlIgnoreCase('Hello', 'HELLO'): {}\n", .{
        ascii.eqlIgnoreCase(s1, s3),
    });
    std.debug.print("eqlIgnoreCase('Hello', 'World'): {}\n", .{
        ascii.eqlIgnoreCase(s1, s4),
    });

    // startsWithIgnoreCase, endsWithIgnoreCase
    const haystack = "Hello, World!";
    std.debug.print("starts with 'hello': {}\n", .{
        ascii.startsWithIgnoreCase(haystack, "hello"),
    });
    std.debug.print("ends with 'WORLD!': {}\n", .{
        ascii.endsWithIgnoreCase(haystack, "WORLD!"),
    });
}
```

### การตรวจสอบ String

```zig
const std = @import("std");
const ascii = std.ascii;

pub fn isAllDigits(s: []const u8) bool {
    for (s) |c| {
        if (!ascii.isDigit(c)) return false;
    }
    return s.len > 0;
}

pub fn isValidIdentifier(s: []const u8) bool {
    if (s.len == 0) return false;
    if (!ascii.isAlpha(s[0]) and s[0] != '_') return false;
    for (s[1..]) |c| {
        if (!ascii.isAlphanumeric(c) and c != '_') return false;
    }
    return true;
}

pub fn countWords(s: []const u8) usize {
    var count: usize = 0;
    var in_word = false;
    for (s) |c| {
        if (ascii.isWhitespace(c)) {
            in_word = false;
        } else if (!in_word) {
            count += 1;
            in_word = true;
        }
    }
    return count;
}

pub fn main() void {
    std.debug.print("isAllDigits('12345'): {}\n", .{isAllDigits("12345")});
    std.debug.print("isAllDigits('123a5'): {}\n", .{isAllDigits("123a5")});
    std.debug.print("isValidIdentifier('myVar'): {}\n", .{isValidIdentifier("myVar")});
    std.debug.print("isValidIdentifier('123var'): {}\n", .{isValidIdentifier("123var")});
    std.debug.print("countWords('hello world foo'): {}\n", .{countWords("hello world foo")});
}
```

---

## Step 362: std.unicode Functions

### Unicode Basics

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    // UTF-8 encoding
    const thai_text = "สวัสดี"; // Thai: Hello
    const english_text = "Hello";

    std.debug.print("Thai: '{s}'\n", .{thai_text});
    std.debug.print("Thai bytes: {}\n", .{thai_text.len});
    std.debug.print("English: '{s}'\n", .{english_text});
    std.debug.print("English bytes: {}\n", .{english_text.len});

    // นับ Unicode codepoints
    const thai_len = try unicode.utf8CountCodepoints(thai_text);
    const english_len = try unicode.utf8CountCodepoints(english_text);

    std.debug.print("Thai codepoints: {}\n", .{thai_len});
    std.debug.print("English codepoints: {}\n", .{english_len});
}
```

### UTF-8 Iterator

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    const text = "Hello, สวัสดี! 🌍";

    std.debug.print("Iterating codepoints:\n", .{});

    var iter = unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    var count: usize = 0;
    while (iter.nextCodepoint()) |codepoint| {
        count += 1;
        if (codepoint < 128) {
            std.debug.print("  [{}] U+{X:0>4} = '{c}'\n", .{ count, codepoint, @as(u8, @intCast(codepoint)) });
        } else {
            std.debug.print("  [{}] U+{X:0>4}\n", .{ count, codepoint });
        }
    }

    std.debug.print("Total codepoints: {}\n", .{count});
}
```

### UTF-8 Encoding/Decoding

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn main() !void {
    // Encode codepoint เป็น UTF-8 bytes
    var buf: [4]u8 = undefined;

    // ASCII character
    const ascii_len = try unicode.utf8Encode('A', &buf);
    std.debug.print("'A' encodes to {} byte(s): {any}\n", .{ ascii_len, buf[0..ascii_len] });

    // Thai character (สระ อ)
    const thai_cp: u21 = 0x0E2D; // 'อ'
    const thai_len = try unicode.utf8Encode(thai_cp, &buf);
    std.debug.print("Thai 'อ' (U+0E2D) encodes to {} bytes: {any}\n", .{ thai_len, buf[0..thai_len] });

    // Emoji (Globe emoji 🌍)
    const emoji_cp: u21 = 0x1F30D;
    const emoji_len = try unicode.utf8Encode(emoji_cp, &buf);
    std.debug.print("🌍 emoji encodes to {} bytes: {any}\n", .{ emoji_len, buf[0..emoji_len] });

    // Decode UTF-8 bytes เป็น codepoint
    const thai_str = "ก";
    const result = try unicode.utf8Decode(thai_str[0..]);
    std.debug.print("Decoded Thai 'ก': U+{X:0>4}\n", .{result});
}
```

### การทำงานกับ Thai Text

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn reverseCodepoints(allocator: std.mem.Allocator, text: []const u8) ![]u8 {
    // เก็บ codepoints ทั้งหมด
    var codepoints = std.ArrayList(u21).init(allocator);
    defer codepoints.deinit();

    var iter = unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    while (iter.nextCodepoint()) |cp| {
        try codepoints.append(cp);
    }

    // Reverse
    std.mem.reverse(u21, codepoints.items);

    // Encode กลับเป็น UTF-8
    var result = std.ArrayList(u8).init(allocator);
    var buf: [4]u8 = undefined;

    for (codepoints.items) |cp| {
        const len = try unicode.utf8Encode(cp, &buf);
        try result.appendSlice(buf[0..len]);
    }

    return result.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const text = "Hello World";
    const reversed = try reverseCodepoints(allocator, text);
    defer allocator.free(reversed);

    std.debug.print("Original: {s}\n", .{text});
    std.debug.print("Reversed: {s}\n", .{reversed});
}
```

---

## Step 363: StringBuilder Pattern

### StringBuilder Implementation

```zig
const std = @import("std");

pub const StringBuilder = struct {
    buffer: std.ArrayList(u8),

    pub fn init(allocator: std.mem.Allocator) StringBuilder {
        return .{ .buffer = std.ArrayList(u8).init(allocator) };
    }

    pub fn initCapacity(allocator: std.mem.Allocator, capacity: usize) !StringBuilder {
        return .{ .buffer = try std.ArrayList(u8).initCapacity(allocator, capacity) };
    }

    pub fn deinit(self: *StringBuilder) void {
        self.buffer.deinit();
    }

    pub fn append(self: *StringBuilder, s: []const u8) !*StringBuilder {
        try self.buffer.appendSlice(s);
        return self;
    }

    pub fn appendChar(self: *StringBuilder, c: u8) !*StringBuilder {
        try self.buffer.append(c);
        return self;
    }

    pub fn appendFmt(self: *StringBuilder, comptime fmt: []const u8, args: anytype) !*StringBuilder {
        try self.buffer.writer().print(fmt, args);
        return self;
    }

    pub fn appendLine(self: *StringBuilder, s: []const u8) !*StringBuilder {
        try self.buffer.appendSlice(s);
        try self.buffer.append('\n');
        return self;
    }

    pub fn prepend(self: *StringBuilder, s: []const u8) !*StringBuilder {
        try self.buffer.insertSlice(0, s);
        return self;
    }

    pub fn repeat(self: *StringBuilder, s: []const u8, n: usize) !*StringBuilder {
        var i: usize = 0;
        while (i < n) : (i += 1) {
            try self.buffer.appendSlice(s);
        }
        return self;
    }

    pub fn clear(self: *StringBuilder) void {
        self.buffer.clearRetainingCapacity();
    }

    pub fn toString(self: *const StringBuilder) []const u8 {
        return self.buffer.items;
    }

    pub fn toOwnedString(self: *StringBuilder) ![]u8 {
        return self.buffer.toOwnedSlice();
    }

    pub fn len(self: *const StringBuilder) usize {
        return self.buffer.items.len;
    }

    pub fn isEmpty(self: *const StringBuilder) bool {
        return self.buffer.items.len == 0;
    }

    pub fn trimRight(self: *StringBuilder) void {
        var end = self.buffer.items.len;
        while (end > 0 and std.ascii.isWhitespace(self.buffer.items[end - 1])) {
            end -= 1;
        }
        self.buffer.shrinkRetainingCapacity(end);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var sb = StringBuilder.init(allocator);
    defer sb.deinit();

    _ = try sb.append("Hello");
    _ = try sb.append(", ");
    _ = try sb.append("World");
    _ = try sb.appendChar('!');
    _ = try sb.appendLine("");

    _ = try sb.appendFmt("Value: {d:.2}\n", .{3.14159});
    _ = try sb.repeat("=-", 10);

    std.debug.print("{s}\n", .{sb.toString()});
    std.debug.print("Length: {}\n", .{sb.len()});
}
```

### HTML Builder ตัวอย่าง

```zig
const std = @import("std");

const HtmlBuilder = struct {
    sb: @import("part-037-string-handling.md").StringBuilder,

    // ใช้ StringBuilder ที่กำหนดไว้ข้างบน
    // (ในการใช้งานจริง import จาก file อื่น)
};

// Simplified version ที่ไม่ต้อง import
pub fn buildHtml(allocator: std.mem.Allocator) ![]u8 {
    var buffer = std.ArrayList(u8).init(allocator);
    const w = buffer.writer();

    try w.writeAll("<!DOCTYPE html>\n<html>\n<head>\n");
    try w.print("  <title>{s}</title>\n", .{"My Page"});
    try w.writeAll("</head>\n<body>\n");
    try w.print("  <h1>{s}</h1>\n", .{"Hello, World!"});
    try w.writeAll("  <ul>\n");

    const items = [_][]const u8{ "Item 1", "Item 2", "Item 3" };
    for (items) |item| {
        try w.print("    <li>{s}</li>\n", .{item});
    }

    try w.writeAll("  </ul>\n</body>\n</html>");

    return buffer.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const html = try buildHtml(allocator);
    defer allocator.free(html);

    std.debug.print("{s}\n", .{html});
}
```

---

## Step 364: String Interning

### String Interning คืออะไร?

String interning คือเทคนิคที่เก็บ string เดียวกันไว้ที่เดียว แทนที่จะสร้าง copy หลายครั้ง ช่วยประหยัดหน่วยความจำและทำให้เปรียบเทียบเร็วขึ้น (pointer comparison แทน byte comparison)

```zig
const std = @import("std");

pub const StringInterner = struct {
    map: std.StringHashMap([]const u8),
    arena: std.heap.ArenaAllocator,

    pub fn init(allocator: std.mem.Allocator) StringInterner {
        return .{
            .map = std.StringHashMap([]const u8).init(allocator),
            .arena = std.heap.ArenaAllocator.init(allocator),
        };
    }

    pub fn deinit(self: *StringInterner) void {
        self.map.deinit();
        self.arena.deinit();
    }

    pub fn intern(self: *StringInterner, s: []const u8) ![]const u8 {
        // ถ้ามีอยู่แล้ว return pointer เดิม
        if (self.map.get(s)) |existing| {
            return existing;
        }

        // สร้าง copy ใน arena
        const owned = try self.arena.allocator().dupe(u8, s);

        // เก็บใน map
        try self.map.put(owned, owned);

        return owned;
    }

    pub fn count(self: *const StringInterner) usize {
        return self.map.count();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var interner = StringInterner.init(allocator);
    defer interner.deinit();

    const s1 = try interner.intern("hello");
    const s2 = try interner.intern("world");
    const s3 = try interner.intern("hello"); // ได้ pointer เดิม

    std.debug.print("s1: {s} (ptr: {})\n", .{ s1, @intFromPtr(s1.ptr) });
    std.debug.print("s2: {s} (ptr: {})\n", .{ s2, @intFromPtr(s2.ptr) });
    std.debug.print("s3: {s} (ptr: {})\n", .{ s3, @intFromPtr(s3.ptr) });
    std.debug.print("s1 and s3 same ptr: {}\n", .{s1.ptr == s3.ptr});
    std.debug.print("Unique strings: {}\n", .{interner.count()});

    // เปรียบเทียบ pointer แทน bytes (เร็วกว่า)
    if (s1.ptr == s3.ptr) {
        std.debug.print("Fast comparison: same string!\n", .{});
    }
}
```

---

## Step 365: Regular Expression Alternatives

### Pattern Matching ด้วย std.mem

```zig
const std = @import("std");

pub fn main() void {
    const text = "Hello, World! This is a test string.";

    // indexOf - หาตำแหน่งของ substring
    if (std.mem.indexOf(u8, text, "World")) |pos| {
        std.debug.print("'World' found at position: {}\n", .{pos});
    }

    // lastIndexOf - หาจากท้าย
    if (std.mem.lastIndexOf(u8, text, "is")) |pos| {
        std.debug.print("Last 'is' at position: {}\n", .{pos});
    }

    // startsWith / endsWith
    std.debug.print("Starts with 'Hello': {}\n", .{
        std.mem.startsWith(u8, text, "Hello"),
    });
    std.debug.print("Ends with 'string.': {}\n", .{
        std.mem.endsWith(u8, text, "string."),
    });

    // containsAtLeast - มีอย่างน้อย n ครั้ง
    std.debug.print("Contains 'is' at least 2 times: {}\n", .{
        std.mem.containsAtLeast(u8, text, 2, "is"),
    });

    // count - นับจำนวนครั้ง
    const count = std.mem.count(u8, text, "s");
    std.debug.print("Count of 's': {}\n", .{count});
}
```

### Simple Pattern Matching Implementation

```zig
const std = @import("std");

pub const Pattern = struct {
    pub fn match(pattern: []const u8, text: []const u8) bool {
        return matchHelper(pattern, text, 0, 0);
    }

    fn matchHelper(pattern: []const u8, text: []const u8, pi: usize, ti: usize) bool {
        if (pi == pattern.len) return ti == text.len;

        if (pattern[pi] == '*') {
            // '*' matches zero or more characters
            var i: usize = ti;
            while (i <= text.len) : (i += 1) {
                if (matchHelper(pattern, text, pi + 1, i)) return true;
            }
            return false;
        }

        if (ti == text.len) return false;

        if (pattern[pi] == '?' or pattern[pi] == text[ti]) {
            return matchHelper(pattern, text, pi + 1, ti + 1);
        }

        return false;
    }
};

pub fn main() void {
    std.debug.print("=== Wildcard Pattern Matching ===\n", .{});

    const tests = [_]struct { pattern: []const u8, text: []const u8, expected: bool }{
        .{ .pattern = "hello", .text = "hello", .expected = true },
        .{ .pattern = "h*o", .text = "hello", .expected = true },
        .{ .pattern = "h?llo", .text = "hello", .expected = true },
        .{ .pattern = "*.zig", .text = "main.zig", .expected = true },
        .{ .pattern = "*.zig", .text = "main.go", .expected = false },
        .{ .pattern = "test_*", .text = "test_case1", .expected = true },
    };

    for (tests) |t| {
        const result = Pattern.match(t.pattern, t.text);
        std.debug.print("match('{s}', '{s}') = {} (expected: {})\n", .{
            t.pattern, t.text, result, t.expected,
        });
    }
}
```

### Glob Pattern Matching

```zig
const std = @import("std");

pub fn globMatch(pattern: []const u8, path: []const u8) bool {
    var p: usize = 0;
    var t: usize = 0;
    var star_p: usize = std.math.maxInt(usize);
    var star_t: usize = 0;

    while (t < path.len) {
        if (p < pattern.len and (pattern[p] == '?' or pattern[p] == path[t])) {
            p += 1;
            t += 1;
        } else if (p < pattern.len and pattern[p] == '*') {
            star_p = p;
            star_t = t;
            p += 1;
        } else if (star_p != std.math.maxInt(usize)) {
            star_t += 1;
            t = star_t;
            p = star_p + 1;
        } else {
            return false;
        }
    }

    while (p < pattern.len and pattern[p] == '*') {
        p += 1;
    }

    return p == pattern.len;
}

pub fn main() void {
    const patterns = [_][]const u8{
        "src/*.zig",
        "**/*.md",
        "test_?.zig",
    };

    const paths = [_][]const u8{
        "src/main.zig",
        "src/utils.zig",
        "docs/readme.md",
        "test_1.zig",
        "test_12.zig",
    };

    for (patterns) |pattern| {
        std.debug.print("Pattern: {s}\n", .{pattern});
        for (paths) |path| {
            std.debug.print("  {s}: {}\n", .{ path, globMatch(pattern, path) });
        }
    }
}
```

---

## Step 366: String Tokenization

### Tokenizer พื้นฐาน

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const text = "hello world  foo   bar";

    // tokenizeAny - แยกด้วย whitespace (skip empty tokens)
    std.debug.print("Tokens:\n", .{});
    var iter = std.mem.tokenizeAny(u8, text, " \t\n");
    while (iter.next()) |token| {
        std.debug.print("  '{s}'\n", .{token});
    }

    // splitAny - แยกรวม empty tokens
    std.debug.print("\nSplit results (with empty):\n", .{});
    var split_iter = std.mem.splitAny(u8, text, " ");
    while (split_iter.next()) |part| {
        std.debug.print("  '{s}'\n", .{part});
    }

    // splitScalar - แยกด้วย character เดียว
    const csv = "a,b,,c,d";
    std.debug.print("\nCSV split:\n", .{});
    var csv_iter = std.mem.splitScalar(u8, csv, ',');
    while (csv_iter.next()) |field| {
        std.debug.print("  '{s}'\n", .{field});
    }

    // เก็บ tokens ใน ArrayList
    var tokens = std.ArrayList([]const u8).init(allocator);
    defer tokens.deinit();

    var token_iter = std.mem.tokenizeAny(u8, text, " ");
    while (token_iter.next()) |token| {
        try tokens.append(token);
    }
    std.debug.print("\nCollected {} tokens\n", .{tokens.items.len});
}
```

### CSV Parser

```zig
const std = @import("std");

const CsvParser = struct {
    pub fn parseRow(
        allocator: std.mem.Allocator,
        row: []const u8,
    ) ![][]const u8 {
        var fields = std.ArrayList([]const u8).init(allocator);
        var i: usize = 0;
        var field_start: usize = 0;
        var in_quotes = false;

        while (i < row.len) : (i += 1) {
            switch (row[i]) {
                '"' => in_quotes = !in_quotes,
                ',' => {
                    if (!in_quotes) {
                        var field = row[field_start..i];
                        // Strip quotes
                        if (field.len >= 2 and field[0] == '"' and field[field.len - 1] == '"') {
                            field = field[1 .. field.len - 1];
                        }
                        try fields.append(field);
                        field_start = i + 1;
                    }
                },
                else => {},
            }
        }

        // Last field
        var last_field = row[field_start..];
        if (last_field.len >= 2 and last_field[0] == '"' and last_field[last_field.len - 1] == '"') {
            last_field = last_field[1 .. last_field.len - 1];
        }
        try fields.append(last_field);

        return fields.toOwnedSlice();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const rows = [_][]const u8{
        "John,25,\"New York, NY\",Engineer",
        "Jane,30,Boston,Manager",
        "Bob,22,\"Los Angeles\",Developer",
    };

    std.debug.print("Parsed CSV:\n", .{});
    for (rows) |row| {
        const fields = try CsvParser.parseRow(allocator, row);
        defer allocator.free(fields);

        std.debug.print("  Row:\n", .{});
        for (fields, 0..) |field, i| {
            std.debug.print("    [{}]: {s}\n", .{ i, field });
        }
    }
}
```

---

## Step 367: String Formatting ขั้นสูง

### Format Specifiers

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Numbers
    std.debug.print("Decimal: {d}\n", .{42});
    std.debug.print("Hex: {x}\n", .{255});
    std.debug.print("Hex upper: {X}\n", .{255});
    std.debug.print("Octal: {o}\n", .{8});
    std.debug.print("Binary: {b}\n", .{10});
    std.debug.print("Float: {f}\n", .{3.14});
    std.debug.print("Scientific: {e}\n", .{12345.0});
    std.debug.print("Decimal float: {d:.3}\n", .{3.14159});

    // Width and alignment
    std.debug.print("Right-aligned: '{d:>10}'\n", .{42});
    std.debug.print("Left-aligned: '{d:<10}'\n", .{42});
    std.debug.print("Center: '{d:^10}'\n", .{42});
    std.debug.print("Zero-padded: '{d:0>5}'\n", .{42});

    // Strings
    std.debug.print("String: '{s:>20}'\n", .{"hello"});
    std.debug.print("String: '{s:<20}'\n", .{"hello"});

    // Custom formatting ด้วย allocPrint
    const formatted = try std.fmt.allocPrint(allocator, "Name: {s}, Age: {d}, Score: {d:.1}%", .{
        "Alice", 25, 98.6,
    });
    defer allocator.free(formatted);
    std.debug.print("{s}\n", .{formatted});
}
```

### Custom Formatter

```zig
const std = @import("std");

const Color = struct {
    r: u8,
    g: u8,
    b: u8,

    pub fn format(
        self: Color,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = options;
        if (std.mem.eql(u8, fmt, "hex")) {
            try writer.print("#{X:0>2}{X:0>2}{X:0>2}", .{ self.r, self.g, self.b });
        } else if (std.mem.eql(u8, fmt, "rgb")) {
            try writer.print("rgb({}, {}, {})", .{ self.r, self.g, self.b });
        } else {
            try writer.print("Color(r={}, g={}, b={})", .{ self.r, self.g, self.b });
        }
    }
};

const Temperature = struct {
    celsius: f64,

    pub fn format(
        self: Temperature,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = options;
        if (std.mem.eql(u8, fmt, "f")) {
            // Fahrenheit
            const fahrenheit = self.celsius * 9.0 / 5.0 + 32.0;
            try writer.print("{d:.1}°F", .{fahrenheit});
        } else {
            try writer.print("{d:.1}°C", .{self.celsius});
        }
    }
};

pub fn main() !void {
    const red = Color{ .r = 255, .g = 0, .b = 0 };
    const skyblue = Color{ .r = 135, .g = 206, .b = 235 };

    std.debug.print("Red: {hex}\n", .{red});
    std.debug.print("Red: {rgb}\n", .{red});
    std.debug.print("Sky Blue: {hex}\n", .{skyblue});

    const boiling = Temperature{ .celsius = 100.0 };
    const freezing = Temperature{ .celsius = 0.0 };

    std.debug.print("Boiling: {} = {f}\n", .{ boiling, boiling });
    std.debug.print("Freezing: {} = {f}\n", .{ freezing, freezing });
}
```

---

## Step 368: Locale-aware String Operations

### Thai String Support

```zig
const std = @import("std");
const unicode = std.unicode;

pub fn countThaiChars(text: []const u8) !usize {
    var count: usize = 0;
    var iter = unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    while (iter.nextCodepoint()) |cp| {
        // Thai Unicode range: U+0E00–U+0E7F
        if (cp >= 0x0E00 and cp <= 0x0E7F) {
            count += 1;
        }
    }
    return count;
}

pub fn isThaiChar(cp: u21) bool {
    return cp >= 0x0E00 and cp <= 0x0E7F;
}

pub fn containsThai(text: []const u8) !bool {
    var iter = unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    while (iter.nextCodepoint()) |cp| {
        if (isThaiChar(cp)) return true;
    }
    return false;
}

pub fn splitByScript(
    allocator: std.mem.Allocator,
    text: []const u8,
) !struct { thai: []u8, other: []u8 } {
    var thai_buf = std.ArrayList(u8).init(allocator);
    var other_buf = std.ArrayList(u8).init(allocator);

    var iter = unicode.Utf8Iterator{ .bytes = text, .i = 0 };
    var encode_buf: [4]u8 = undefined;

    while (iter.nextCodepointSlice()) |slice| {
        const cp = try unicode.utf8Decode(slice);
        if (isThaiChar(cp)) {
            const len = try unicode.utf8Encode(cp, &encode_buf);
            try thai_buf.appendSlice(encode_buf[0..len]);
        } else if (cp != ' ') {
            const len = try unicode.utf8Encode(cp, &encode_buf);
            try other_buf.appendSlice(encode_buf[0..len]);
        }
    }

    return .{
        .thai = try thai_buf.toOwnedSlice(),
        .other = try other_buf.toOwnedSlice(),
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const mixed = "Hello สวัสดี World ครับ";

    const thai_count = try countThaiChars(mixed);
    std.debug.print("Thai characters: {}\n", .{thai_count});
    std.debug.print("Contains Thai: {}\n", .{try containsThai(mixed)});

    const split = try splitByScript(allocator, mixed);
    defer allocator.free(split.thai);
    defer allocator.free(split.other);

    std.debug.print("Thai part: {s}\n", .{split.thai});
    std.debug.print("Other part: {s}\n", .{split.other});
}
```

---

## Step 369: String Hashing

### ใช้ std.hash สำหรับ String

```zig
const std = @import("std");
const hash = std.hash;

pub fn main() void {
    const strings = [_][]const u8{
        "hello",
        "world",
        "hello",
        "Hello",
        "hello world",
    };

    std.debug.print("=== String Hashing ===\n", .{});

    for (strings) |s| {
        // Wyhash - fast, non-cryptographic
        const h1 = hash.Wyhash.hash(0, s);
        // Fnv hash
        const h2 = hash.Fnv1a_64.hash(s);
        std.debug.print("'{s}': wyhash={X:0>16}, fnv={X:0>16}\n", .{ s, h1, h2 });
    }
}
```

### Consistent Hashing สำหรับ HashMap

```zig
const std = @import("std");

pub const StringContext = struct {
    pub fn hash(self: @This(), s: []const u8) u64 {
        _ = self;
        return std.hash.Wyhash.hash(0, s);
    }

    pub fn eql(self: @This(), a: []const u8, b: []const u8) bool {
        _ = self;
        return std.mem.eql(u8, a, b);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // HashMap ที่ใช้ string เป็น key
    var map = std.HashMap([]const u8, i32, StringContext, std.hash_map.default_max_load_percentage).init(allocator);
    defer map.deinit();

    try map.put("apple", 1);
    try map.put("banana", 2);
    try map.put("cherry", 3);

    if (map.get("apple")) |v| {
        std.debug.print("apple = {}\n", .{v});
    }

    // StringHashMap เป็น convenience type
    var str_map = std.StringHashMap(f64).init(allocator);
    defer str_map.deinit();

    try str_map.put("pi", 3.14159);
    try str_map.put("e", 2.71828);
    try str_map.put("phi", 1.61803);

    var iter = str_map.iterator();
    while (iter.next()) |entry| {
        std.debug.print("{s} = {d:.5}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
}
```

### Rolling Hash สำหรับ Substring Search

```zig
const std = @import("std");

pub fn rabinKarp(text: []const u8, pattern: []const u8) ?usize {
    if (pattern.len > text.len) return null;
    if (pattern.len == 0) return 0;

    const BASE: u64 = 256;
    const MOD: u64 = 101;

    var p_hash: u64 = 0;
    var t_hash: u64 = 0;
    var h: u64 = 1;

    // h = BASE^(m-1) % MOD
    var i: usize = 0;
    while (i < pattern.len - 1) : (i += 1) {
        h = (h * BASE) % MOD;
    }

    // Initial hash values
    for (0..pattern.len) |j| {
        p_hash = (BASE * p_hash + pattern[j]) % MOD;
        t_hash = (BASE * t_hash + text[j]) % MOD;
    }

    // Slide pattern over text
    var start: usize = 0;
    while (start <= text.len - pattern.len) : (start += 1) {
        if (p_hash == t_hash) {
            // Verify actual match
            if (std.mem.eql(u8, pattern, text[start..start + pattern.len])) {
                return start;
            }
        }

        if (start < text.len - pattern.len) {
            t_hash = (BASE * (t_hash + MOD - (h * text[start]) % MOD) + text[start + pattern.len]) % MOD;
        }
    }

    return null;
}

pub fn main() void {
    const text = "GEEKS FOR GEEKS";
    const pattern = "GEEK";

    if (rabinKarp(text, pattern)) |pos| {
        std.debug.print("Pattern '{s}' found at position {}\n", .{ pattern, pos });
    } else {
        std.debug.print("Pattern '{s}' not found\n", .{pattern});
    }
}
```

---

## Step 370: Real-world String Processing

### Log Parser

```zig
const std = @import("std");

const LogLevel = enum {
    debug,
    info,
    warn,
    err,

    pub fn fromString(s: []const u8) ?LogLevel {
        const lower_map = [_]struct { name: []const u8, level: LogLevel }{
            .{ .name = "debug", .level = .debug },
            .{ .name = "info", .level = .info },
            .{ .name = "warn", .level = .warn },
            .{ .name = "error", .level = .err },
        };

        for (lower_map) |entry| {
            if (std.ascii.eqlIgnoreCase(s, entry.name)) {
                return entry.level;
            }
        }
        return null;
    }
};

const LogEntry = struct {
    timestamp: []const u8,
    level: LogLevel,
    message: []const u8,
};

pub fn parseLogLine(line: []const u8) ?LogEntry {
    // Format: [TIMESTAMP] LEVEL: MESSAGE
    // Example: [2024-01-15 10:30:00] INFO: Server started

    if (line.len == 0) return null;

    // Parse timestamp
    if (line[0] != '[') return null;
    const ts_end = std.mem.indexOf(u8, line, "]") orelse return null;
    const timestamp = line[1..ts_end];

    // Skip "] "
    const rest = std.mem.trimLeft(u8, line[ts_end + 1 ..], " ");

    // Parse level
    const level_end = std.mem.indexOf(u8, rest, ":") orelse return null;
    const level_str = rest[0..level_end];
    const level = LogLevel.fromString(level_str) orelse return null;

    // Parse message
    const message = std.mem.trimLeft(u8, rest[level_end + 1 ..], " ");

    return LogEntry{
        .timestamp = timestamp,
        .level = level,
        .message = message,
    };
}

pub fn main() !void {
    const logs = [_][]const u8{
        "[2024-01-15 10:30:00] INFO: Server started",
        "[2024-01-15 10:30:01] DEBUG: Loading config",
        "[2024-01-15 10:30:05] WARN: Low memory",
        "[2024-01-15 10:31:00] ERROR: Connection failed",
        "invalid log line",
    };

    for (logs) |line| {
        if (parseLogLine(line)) |entry| {
            std.debug.print("[{s}] [{s}] {s}\n", .{
                entry.timestamp,
                @tagName(entry.level),
                entry.message,
            });
        } else {
            std.debug.print("Could not parse: '{s}'\n", .{line});
        }
    }
}
```

### Template Engine

```zig
const std = @import("std");

pub fn renderTemplate(
    allocator: std.mem.Allocator,
    template: []const u8,
    vars: std.StringHashMap([]const u8),
) ![]u8 {
    var result = std.ArrayList(u8).init(allocator);

    var i: usize = 0;
    while (i < template.len) {
        if (i + 1 < template.len and template[i] == '{' and template[i + 1] == '{') {
            // Find closing "}}"
            const end = std.mem.indexOf(u8, template[i + 2 ..], "}}") orelse {
                try result.append(template[i]);
                i += 1;
                continue;
            };

            const var_name = std.mem.trim(u8, template[i + 2 .. i + 2 + end], " ");

            if (vars.get(var_name)) |value| {
                try result.appendSlice(value);
            } else {
                // Keep original if var not found
                try result.appendSlice(template[i .. i + 2 + end + 2]);
            }

            i += 2 + end + 2;
        } else {
            try result.append(template[i]);
            i += 1;
        }
    }

    return result.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const template =
        \\Dear {{ name }},
        \\
        \\Thank you for ordering {{ product }}.
        \\Your order total is {{ amount }} THB.
        \\
        \\Best regards,
        \\{{ company }}
    ;

    var vars = std.StringHashMap([]const u8).init(allocator);
    defer vars.deinit();

    try vars.put("name", "คุณสมชาย");
    try vars.put("product", "Zig Programming Book");
    try vars.put("amount", "850");
    try vars.put("company", "ZigCourse Ltd.");

    const rendered = try renderTemplate(allocator, template, vars);
    defer allocator.free(rendered);

    std.debug.print("{s}\n", .{rendered});
}
```

### URL Parser

```zig
const std = @import("std");

const Url = struct {
    scheme: []const u8,
    host: []const u8,
    port: ?u16,
    path: []const u8,
    query: ?[]const u8,
    fragment: ?[]const u8,
};

pub fn parseUrl(url: []const u8) ?Url {
    var rest = url;

    // Parse scheme
    const scheme_end = std.mem.indexOf(u8, rest, "://") orelse return null;
    const scheme = rest[0..scheme_end];
    rest = rest[scheme_end + 3 ..];

    // Parse fragment
    var fragment: ?[]const u8 = null;
    if (std.mem.indexOf(u8, rest, "#")) |frag_pos| {
        fragment = rest[frag_pos + 1 ..];
        rest = rest[0..frag_pos];
    }

    // Parse query
    var query: ?[]const u8 = null;
    if (std.mem.indexOf(u8, rest, "?")) |query_pos| {
        query = rest[query_pos + 1 ..];
        rest = rest[0..query_pos];
    }

    // Parse path
    var path: []const u8 = "/";
    if (std.mem.indexOf(u8, rest, "/")) |path_pos| {
        path = rest[path_pos..];
        rest = rest[0..path_pos];
    }

    // Parse host and port
    var host = rest;
    var port: ?u16 = null;
    if (std.mem.indexOf(u8, rest, ":")) |port_pos| {
        host = rest[0..port_pos];
        const port_str = rest[port_pos + 1 ..];
        port = std.fmt.parseInt(u16, port_str, 10) catch null;
    }

    return Url{
        .scheme = scheme,
        .host = host,
        .port = port,
        .path = path,
        .query = query,
        .fragment = fragment,
    };
}

pub fn main() void {
    const urls = [_][]const u8{
        "https://example.com/path/to/page",
        "http://api.server.com:8080/v1/users?limit=10&page=2",
        "https://docs.zig.lang.org/std/mem.html#indexOf",
    };

    for (urls) |url| {
        std.debug.print("URL: {s}\n", .{url});
        if (parseUrl(url)) |parsed| {
            std.debug.print("  scheme: {s}\n", .{parsed.scheme});
            std.debug.print("  host: {s}\n", .{parsed.host});
            if (parsed.port) |p| std.debug.print("  port: {}\n", .{p});
            std.debug.print("  path: {s}\n", .{parsed.path});
            if (parsed.query) |q| std.debug.print("  query: {s}\n", .{q});
            if (parsed.fragment) |f| std.debug.print("  fragment: {s}\n", .{f});
        }
        std.debug.print("\n", .{});
    }
}
```

---

## สรุปบทที่ 37

ในบทนี้เราได้เรียนรู้เรื่อง String Handling ขั้นสูงใน Zig:

1. **std.ascii** - utilities สำหรับ ASCII characters รวมถึง classification, conversion, และ comparison
2. **std.unicode** - การทำงานกับ UTF-8 encoding, iteration, และ codepoint operations
3. **StringBuilder** - Pattern สำหรับ build strings อย่างมีประสิทธิภาพ
4. **String Interning** - เทคนิคประหยัดหน่วยความจำสำหรับ string ซ้ำๆ
5. **Pattern Matching** - Wildcard และ Glob patterns แทน regex
6. **Tokenization** - การแยก string เป็น tokens รวมถึง CSV parser
7. **String Formatting** - Format specifiers และ custom formatters
8. **Locale-aware** - การทำงานกับ Thai text และ Unicode
9. **String Hashing** - Wyhash, FNV, Rabin-Karp algorithm
10. **Real-world** - Log parser, Template engine, URL parser
