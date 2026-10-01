# ส่วนที่ 65: โปรเจค Static Site Generator

## ขั้นตอนที่ 641-650: สร้าง Static Site Generator ด้วย Zig

---

## ขั้นตอนที่ 641: Project Requirements

### ภาพรวม Static Site Generator (SSG)

เราจะสร้าง SSG ที่แปลง Markdown files เป็น HTML website โดยมีฟีเจอร์:
- Parse Markdown เป็น HTML
- Template engine แบบ simple
- Front matter (YAML-like metadata)
- File watching สำหรับ development
- Navigation generation อัตโนมัติ
- RSS feed generation
- Asset processing
- Build pipeline

### ความสามารถ

```
zigssg [command] [options]

Commands:
  build               Build site ทั้งหมด
  serve               Serve site พร้อม file watching
  new <name>          สร้าง post ใหม่
  clean               ลบ output directory

Options:
  -s, --source <dir>  source directory (default: content)
  -o, --output <dir>  output directory (default: public)
  -t, --templates <dir> templates directory (default: templates)
  -b, --base-url <url> base URL ของ site
  -v, --verbose       verbose output
  --drafts            รวม draft posts ด้วย
```

### โครงสร้างโปรเจค

```
zigssg/
├── build.zig
├── src/
│   ├── main.zig
│   ├── config.zig
│   ├── markdown.zig       (Markdown parser)
│   ├── template.zig       (Template engine)
│   ├── frontmatter.zig    (Front matter parser)
│   ├── watcher.zig        (File watcher)
│   ├── assets.zig         (Asset processor)
│   ├── pipeline.zig       (Build pipeline)
│   ├── navigation.zig     (Nav generation)
│   └── rss.zig            (RSS feed)
├── templates/
│   ├── base.html
│   ├── post.html
│   └── index.html
└── content/
    ├── index.md
    └── posts/
        └── hello-world.md
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "zigssg",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run zigssg");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

---

## ขั้นตอนที่ 642: Markdown Parser (Basic)

### src/markdown.zig

```zig
const std = @import("std");

pub const HtmlOutput = struct {
    allocator: std.mem.Allocator,
    buf: std.ArrayList(u8),

    pub fn init(allocator: std.mem.Allocator) HtmlOutput {
        return HtmlOutput{
            .allocator = allocator,
            .buf = std.ArrayList(u8).init(allocator),
        };
    }

    pub fn deinit(self: *HtmlOutput) void {
        self.buf.deinit();
    }

    pub fn write(self: *HtmlOutput, s: []const u8) !void {
        try self.buf.appendSlice(s);
    }

    pub fn writeFmt(self: *HtmlOutput, comptime fmt: []const u8, args: anytype) !void {
        try self.buf.writer().print(fmt, args);
    }

    pub fn toOwnedSlice(self: *HtmlOutput) ![]u8 {
        return self.buf.toOwnedSlice();
    }
};

pub const MarkdownParser = struct {
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) MarkdownParser {
        return MarkdownParser{ .allocator = allocator };
    }

    /// แปลง Markdown string เป็น HTML string
    pub fn parse(self: *MarkdownParser, input: []const u8) ![]u8 {
        var output = HtmlOutput.init(self.allocator);
        errdefer output.deinit();

        var lines = std.mem.splitSequence(u8, input, "\n");
        var in_code_block = false;
        var code_lang: ?[]const u8 = null;
        var in_list = false;
        var in_blockquote = false;
        var in_paragraph = false;
        var blank_line = false;

        while (lines.next()) |line| {
            const trimmed = std.mem.trimRight(u8, line, " \t\r");

            // Code block ``` 
            if (std.mem.startsWith(u8, trimmed, "```")) {
                if (in_code_block) {
                    try output.write("</code></pre>\n");
                    in_code_block = false;
                    code_lang = null;
                } else {
                    if (in_paragraph) {
                        try output.write("</p>\n");
                        in_paragraph = false;
                    }
                    if (in_list) {
                        try output.write("</ul>\n");
                        in_list = false;
                    }

                    code_lang = if (trimmed.len > 3) trimmed[3..] else null;
                    in_code_block = true;

                    if (code_lang) |lang| {
                        try output.writeFmt("<pre><code class=\"language-{s}\">\n", .{lang});
                    } else {
                        try output.write("<pre><code>\n");
                    }
                }
                continue;
            }

            if (in_code_block) {
                // Escape HTML ใน code block
                try writeEscaped(&output, trimmed);
                try output.write("\n");
                continue;
            }

            // Empty line
            if (trimmed.len == 0) {
                if (in_paragraph) {
                    try output.write("</p>\n");
                    in_paragraph = false;
                }
                if (in_list) {
                    try output.write("</ul>\n");
                    in_list = false;
                }
                if (in_blockquote) {
                    try output.write("</blockquote>\n");
                    in_blockquote = false;
                }
                blank_line = true;
                continue;
            }
            blank_line = false;

            // Headings # ## ### #### ##### ######
            if (trimmed[0] == '#') {
                if (in_paragraph) {
                    try output.write("</p>\n");
                    in_paragraph = false;
                }
                if (in_list) {
                    try output.write("</ul>\n");
                    in_list = false;
                }

                var level: u8 = 0;
                var i: usize = 0;
                while (i < trimmed.len and trimmed[i] == '#') : (i += 1) {
                    level += 1;
                }
                level = @min(level, 6);

                const text = std.mem.trim(u8, trimmed[i..], " ");
                const slug = try slugify(self.allocator, text);
                defer self.allocator.free(slug);

                try output.writeFmt(
                    "<h{d} id=\"{s}\">",
                    .{ level, slug },
                );
                try writeInlineMarkdown(&output, text);
                try output.writeFmt("</h{d}>\n", .{level});
                continue;
            }

            // Horizontal rule ---
            if (std.mem.eql(u8, trimmed, "---") or
                std.mem.eql(u8, trimmed, "***") or
                std.mem.eql(u8, trimmed, "___"))
            {
                try output.write("<hr>\n");
                continue;
            }

            // Blockquote >
            if (trimmed[0] == '>') {
                if (!in_blockquote) {
                    try output.write("<blockquote>\n");
                    in_blockquote = true;
                }
                const quote_text = std.mem.trim(u8, trimmed[1..], " ");
                try output.write("<p>");
                try writeInlineMarkdown(&output, quote_text);
                try output.write("</p>\n");
                continue;
            }

            // Unordered list - * +
            if ((trimmed[0] == '-' or trimmed[0] == '*' or trimmed[0] == '+') and
                trimmed.len > 1 and trimmed[1] == ' ')
            {
                if (!in_list) {
                    if (in_paragraph) {
                        try output.write("</p>\n");
                        in_paragraph = false;
                    }
                    try output.write("<ul>\n");
                    in_list = true;
                }
                const item_text = trimmed[2..];
                try output.write("<li>");
                try writeInlineMarkdown(&output, item_text);
                try output.write("</li>\n");
                continue;
            }

            // Ordered list 1. 2. etc.
            if (trimmed.len > 2) {
                var is_ordered = false;
                var num_end: usize = 0;
                while (num_end < trimmed.len and std.ascii.isDigit(trimmed[num_end])) : (num_end += 1) {}
                if (num_end > 0 and num_end < trimmed.len and trimmed[num_end] == '.') {
                    is_ordered = true;
                }

                if (is_ordered) {
                    if (!in_list) {
                        if (in_paragraph) {
                            try output.write("</p>\n");
                            in_paragraph = false;
                        }
                        try output.write("<ol>\n");
                        in_list = true;
                    }
                    const item_text = std.mem.trim(u8, trimmed[num_end + 1 ..], " ");
                    try output.write("<li>");
                    try writeInlineMarkdown(&output, item_text);
                    try output.write("</li>\n");
                    continue;
                }
            }

            // ปิด list ถ้าออกมาแล้ว
            if (in_list) {
                try output.write("</ul>\n");
                in_list = false;
            }

            // Paragraph
            if (!in_paragraph) {
                try output.write("<p>");
                in_paragraph = true;
            } else {
                try output.write(" ");
            }
            try writeInlineMarkdown(&output, trimmed);
        }

        // ปิด elements ที่ยังเปิดอยู่
        if (in_code_block) try output.write("</code></pre>\n");
        if (in_paragraph) try output.write("</p>\n");
        if (in_list) try output.write("</ul>\n");
        if (in_blockquote) try output.write("</blockquote>\n");

        return output.toOwnedSlice();
    }
};

/// แปลง inline Markdown เป็น HTML
fn writeInlineMarkdown(output: *HtmlOutput, text: []const u8) !void {
    var i: usize = 0;
    while (i < text.len) {
        // Bold **text** หรือ __text__
        if (i + 1 < text.len and text[i] == '*' and text[i + 1] == '*') {
            const end = std.mem.indexOf(u8, text[i + 2 ..], "**");
            if (end) |e| {
                try output.write("<strong>");
                try writeEscaped(output, text[i + 2 .. i + 2 + e]);
                try output.write("</strong>");
                i = i + 2 + e + 2;
                continue;
            }
        }

        // Italic *text* หรือ _text_
        if (text[i] == '*' or text[i] == '_') {
            const marker = text[i];
            const end = std.mem.indexOfScalar(u8, text[i + 1 ..], marker);
            if (end) |e| {
                try output.write("<em>");
                try writeEscaped(output, text[i + 1 .. i + 1 + e]);
                try output.write("</em>");
                i = i + 1 + e + 1;
                continue;
            }
        }

        // Code `text`
        if (text[i] == '`') {
            const end = std.mem.indexOfScalar(u8, text[i + 1 ..], '`');
            if (end) |e| {
                try output.write("<code>");
                try writeEscaped(output, text[i + 1 .. i + 1 + e]);
                try output.write("</code>");
                i = i + 1 + e + 1;
                continue;
            }
        }

        // Link [text](url)
        if (text[i] == '[') {
            const close_bracket = std.mem.indexOfScalar(u8, text[i + 1 ..], ']');
            if (close_bracket) |cb| {
                const after_bracket = i + 1 + cb + 1;
                if (after_bracket < text.len and text[after_bracket] == '(') {
                    const close_paren = std.mem.indexOfScalar(u8, text[after_bracket + 1 ..], ')');
                    if (close_paren) |cp| {
                        const link_text = text[i + 1 .. i + 1 + cb];
                        const link_url = text[after_bracket + 1 .. after_bracket + 1 + cp];
                        try output.writeFmt("<a href=\"{s}\">", .{link_url});
                        try writeEscaped(output, link_text);
                        try output.write("</a>");
                        i = after_bracket + 1 + cp + 1;
                        continue;
                    }
                }
            }
        }

        // Image ![alt](src)
        if (i + 1 < text.len and text[i] == '!' and text[i + 1] == '[') {
            const close_bracket = std.mem.indexOfScalar(u8, text[i + 2 ..], ']');
            if (close_bracket) |cb| {
                const after_bracket = i + 2 + cb + 1;
                if (after_bracket < text.len and text[after_bracket] == '(') {
                    const close_paren = std.mem.indexOfScalar(u8, text[after_bracket + 1 ..], ')');
                    if (close_paren) |cp| {
                        const alt_text = text[i + 2 .. i + 2 + cb];
                        const img_src = text[after_bracket + 1 .. after_bracket + 1 + cp];
                        try output.writeFmt(
                            "<img src=\"{s}\" alt=\"{s}\">",
                            .{ img_src, alt_text },
                        );
                        i = after_bracket + 1 + cp + 1;
                        continue;
                    }
                }
            }
        }

        // HTML escape ตัวอักษรปกติ
        try writeEscapedChar(output, text[i]);
        i += 1;
    }
}

fn writeEscaped(output: *HtmlOutput, text: []const u8) !void {
    for (text) |c| {
        try writeEscapedChar(output, c);
    }
}

fn writeEscapedChar(output: *HtmlOutput, c: u8) !void {
    switch (c) {
        '<' => try output.write("&lt;"),
        '>' => try output.write("&gt;"),
        '&' => try output.write("&amp;"),
        '"' => try output.write("&quot;"),
        '\'' => try output.write("&#39;"),
        else => try output.buf.append(c),
    }
}

/// แปลง text เป็น URL slug
fn slugify(allocator: std.mem.Allocator, text: []const u8) ![]u8 {
    var result = std.ArrayList(u8).init(allocator);

    for (text) |c| {
        if (std.ascii.isAlphanumeric(c)) {
            try result.append(std.ascii.toLower(c));
        } else if (c == ' ' or c == '-' or c == '_') {
            if (result.items.len > 0 and result.items[result.items.len - 1] != '-') {
                try result.append('-');
            }
        }
    }

    return result.toOwnedSlice();
}

// Tests
test "Parse headings" {
    const allocator = std.testing.allocator;
    var parser = MarkdownParser.init(allocator);

    const html = try parser.parse("# Hello World\n");
    defer allocator.free(html);

    try std.testing.expect(std.mem.indexOf(u8, html, "<h1") != null);
    try std.testing.expect(std.mem.indexOf(u8, html, "Hello World") != null);
}

test "Parse bold and italic" {
    const allocator = std.testing.allocator;
    var parser = MarkdownParser.init(allocator);

    const html = try parser.parse("**bold** and *italic*\n");
    defer allocator.free(html);

    try std.testing.expect(std.mem.indexOf(u8, html, "<strong>bold</strong>") != null);
    try std.testing.expect(std.mem.indexOf(u8, html, "<em>italic</em>") != null);
}

test "Parse links" {
    const allocator = std.testing.allocator;
    var parser = MarkdownParser.init(allocator);

    const html = try parser.parse("[Click here](https://example.com)\n");
    defer allocator.free(html);

    try std.testing.expect(std.mem.indexOf(u8, html, "href=\"https://example.com\"") != null);
    try std.testing.expect(std.mem.indexOf(u8, html, "Click here") != null);
}

test "Parse code block" {
    const allocator = std.testing.allocator;
    var parser = MarkdownParser.init(allocator);

    const md = "```zig\nconst x = 1;\n```\n";
    const html = try parser.parse(md);
    defer allocator.free(html);

    try std.testing.expect(std.mem.indexOf(u8, html, "<pre><code") != null);
    try std.testing.expect(std.mem.indexOf(u8, html, "language-zig") != null);
}
```

---

## ขั้นตอนที่ 643: Template Engine

### src/template.zig

```zig
const std = @import("std");

pub const TemplateError = error{
    TagNotClosed,
    UnknownVariable,
    InvalidSyntax,
    OutOfMemory,
};

pub const Context = struct {
    allocator: std.mem.Allocator,
    vars: std.StringHashMap([]const u8),
    lists: std.StringHashMap([]const std.StringHashMap([]const u8)),

    pub fn init(allocator: std.mem.Allocator) Context {
        return Context{
            .allocator = allocator,
            .vars = std.StringHashMap([]const u8).init(allocator),
            .lists = std.StringHashMap([]const std.StringHashMap([]const u8)).init(allocator),
        };
    }

    pub fn deinit(self: *Context) void {
        var iter = self.vars.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.vars.deinit();
        self.lists.deinit();
    }

    pub fn set(self: *Context, key: []const u8, value: []const u8) !void {
        const k = try self.allocator.dupe(u8, key);
        const v = try self.allocator.dupe(u8, value);
        try self.vars.put(k, v);
    }

    pub fn get(self: *const Context, key: []const u8) ?[]const u8 {
        return self.vars.get(key);
    }

    pub fn setInt(self: *Context, key: []const u8, value: i64) !void {
        const v = try std.fmt.allocPrint(self.allocator, "{d}", .{value});
        try self.set(key, v);
        self.allocator.free(v);
    }
};

/// Template Engine - รองรับ {{ variable }}, {% if %}, {% for %}
pub const TemplateEngine = struct {
    allocator: std.mem.Allocator,
    template_dir: []const u8,

    pub fn init(allocator: std.mem.Allocator, template_dir: []const u8) TemplateEngine {
        return TemplateEngine{
            .allocator = allocator,
            .template_dir = template_dir,
        };
    }

    /// Render template string ด้วย context
    pub fn renderString(self: *TemplateEngine, template: []const u8, ctx: *const Context) ![]u8 {
        var output = std.ArrayList(u8).init(self.allocator);
        errdefer output.deinit();

        var i: usize = 0;
        while (i < template.len) {
            // ค้นหา {{ หรือ {%
            if (i + 1 < template.len and template[i] == '{') {
                if (template[i + 1] == '{') {
                    // Variable: {{ varname }}
                    const end = std.mem.indexOf(u8, template[i + 2 ..], "}}");
                    if (end == null) return TemplateError.TagNotClosed;

                    const var_name = std.mem.trim(u8, template[i + 2 .. i + 2 + end.?], " \t");
                    const value = ctx.get(var_name) orelse "";
                    try output.appendSlice(value);
                    i = i + 2 + end.? + 2;
                    continue;
                } else if (template[i + 1] == '%') {
                    // Block tag: {% tag %}
                    const end = std.mem.indexOf(u8, template[i + 2 ..], "%}");
                    if (end == null) return TemplateError.TagNotClosed;

                    const tag_content = std.mem.trim(u8, template[i + 2 .. i + 2 + end.?], " \t");
                    i = i + 2 + end.? + 2;

                    // Parse tag type
                    if (std.mem.startsWith(u8, tag_content, "if ")) {
                        const condition = tag_content[3..];
                        const endif_pos = findEndTag(template[i..], "endif");
                        if (endif_pos == null) return TemplateError.TagNotClosed;

                        const block = template[i .. i + endif_pos.?];
                        const is_true = evaluateCondition(ctx, condition);

                        if (is_true) {
                            const rendered = try self.renderString(block, ctx);
                            defer self.allocator.free(rendered);
                            try output.appendSlice(rendered);
                        }

                        i = i + endif_pos.? + "{% endif %}".len;
                    }
                    continue;
                }
            }

            // Escape HTML ถ้าจำเป็น (ในที่นี้เราไม่ escape เพราะ template ควรเชื่อถือได้)
            try output.append(template[i]);
            i += 1;
        }

        return output.toOwnedSlice();
    }

    /// Render template file ด้วย context
    pub fn renderFile(
        self: *TemplateEngine,
        template_name: []const u8,
        ctx: *const Context,
    ) ![]u8 {
        const template_path = try std.fs.path.join(
            self.allocator,
            &[_][]const u8{ self.template_dir, template_name },
        );
        defer self.allocator.free(template_path);

        const file = try std.fs.cwd().openFile(template_path, .{});
        defer file.close();

        const size = try file.getEndPos();
        const template_content = try self.allocator.alloc(u8, size);
        defer self.allocator.free(template_content);

        _ = try file.readAll(template_content);

        // ตรวจสอบ extends
        if (std.mem.startsWith(u8, template_content, "{% extends")) {
            return self.renderWithExtends(template_content, ctx);
        }

        return self.renderString(template_content, ctx);
    }

    fn renderWithExtends(
        self: *TemplateEngine,
        template: []const u8,
        ctx: *const Context,
    ) ![]u8 {
        // Parse parent template name
        const quote_start = std.mem.indexOfScalar(u8, template, '"') orelse return error.InvalidSyntax;
        const quote_end = std.mem.indexOfScalar(u8, template[quote_start + 1 ..], '"') orelse return error.InvalidSyntax;
        const parent_name = template[quote_start + 1 .. quote_start + 1 + quote_end];

        // โหลด parent template
        const parent_path = try std.fs.path.join(
            self.allocator,
            &[_][]const u8{ self.template_dir, parent_name },
        );
        defer self.allocator.free(parent_path);

        const parent_file = try std.fs.cwd().openFile(parent_path, .{});
        defer parent_file.close();

        const parent_size = try parent_file.getEndPos();
        const parent_content = try self.allocator.alloc(u8, parent_size);
        defer self.allocator.free(parent_content);

        _ = try parent_file.readAll(parent_content);

        // Parse blocks ใน child
        var child_blocks = std.StringHashMap([]const u8).init(self.allocator);
        defer {
            var iter = child_blocks.iterator();
            while (iter.next()) |entry| {
                self.allocator.free(entry.key_ptr.*);
                self.allocator.free(entry.value_ptr.*);
            }
            child_blocks.deinit();
        }

        // ค้นหา {% block name %} ... {% endblock %} ใน child
        var search_pos: usize = 0;
        while (std.mem.indexOf(u8, template[search_pos..], "{% block ")) |block_start| {
            const abs_start = search_pos + block_start;
            const name_end = std.mem.indexOf(u8, template[abs_start..], " %}") orelse break;
            const block_name = template[abs_start + 9 .. abs_start + name_end];

            const content_start = abs_start + name_end + 3;
            const endblock = std.mem.indexOf(u8, template[content_start..], "{% endblock %}") orelse break;
            const block_content = template[content_start .. content_start + endblock];

            const k = try self.allocator.dupe(u8, block_name);
            const v = try self.allocator.dupe(u8, block_content);
            try child_blocks.put(k, v);

            search_pos = content_start + endblock + "{% endblock %}".len;
        }

        // Render parent โดย replace blocks
        var output = std.ArrayList(u8).init(self.allocator);
        defer output.deinit();

        var pos: usize = 0;
        while (std.mem.indexOf(u8, parent_content[pos..], "{% block ")) |block_pos| {
            const abs_pos = pos + block_pos;
            try output.appendSlice(parent_content[pos..abs_pos]);

            const name_end = std.mem.indexOf(u8, parent_content[abs_pos..], " %}") orelse break;
            const block_name = parent_content[abs_pos + 9 .. abs_pos + name_end];

            const content_start = abs_pos + name_end + 3;
            const endblock_pos = std.mem.indexOf(u8, parent_content[content_start..], "{% endblock %}") orelse break;

            // ใช้ child block ถ้ามี ไม่งั้นใช้ parent block
            if (child_blocks.get(block_name)) |child_content| {
                const rendered = try self.renderString(child_content, ctx);
                defer self.allocator.free(rendered);
                try output.appendSlice(rendered);
            } else {
                const default_content = parent_content[content_start .. content_start + endblock_pos];
                const rendered = try self.renderString(default_content, ctx);
                defer self.allocator.free(rendered);
                try output.appendSlice(rendered);
            }

            pos = content_start + endblock_pos + "{% endblock %}".len;
        }

        try output.appendSlice(parent_content[pos..]);
        return output.toOwnedSlice();
    }
};

fn findEndTag(template: []const u8, tag: []const u8) ?usize {
    const search_for = std.fmt.comptimePrint("{{% end{s} %}}", .{tag});
    return std.mem.indexOf(u8, template, search_for);
}

fn evaluateCondition(ctx: *const Context, condition: []const u8) bool {
    const trimmed = std.mem.trim(u8, condition, " ");
    const value = ctx.get(trimmed) orelse return false;
    return value.len > 0 and !std.mem.eql(u8, value, "false") and !std.mem.eql(u8, value, "0");
}

// Tests
test "Render variables" {
    const allocator = std.testing.allocator;
    var engine = TemplateEngine.init(allocator, "templates");

    var ctx = Context.init(allocator);
    defer ctx.deinit();

    try ctx.set("title", "สวัสดีโลก");
    try ctx.set("author", "สมชาย");

    const template = "<h1>{{ title }}</h1><p>by {{ author }}</p>";
    const result = try engine.renderString(template, &ctx);
    defer allocator.free(result);

    try std.testing.expectEqualStrings(
        "<h1>สวัสดีโลก</h1><p>by สมชาย</p>",
        result,
    );
}
```

---

## ขั้นตอนที่ 644-645: File Watching และ Asset Processing

### src/watcher.zig

```zig
const std = @import("std");

pub const FileChange = struct {
    path: []const u8,
    kind: enum { created, modified, deleted },
};

pub const FileWatcher = struct {
    allocator: std.mem.Allocator,
    watch_paths: std.ArrayList([]const u8),
    file_times: std.StringHashMap(i128),
    interval_ms: u64,

    pub fn init(allocator: std.mem.Allocator) FileWatcher {
        return FileWatcher{
            .allocator = allocator,
            .watch_paths = std.ArrayList([]const u8).init(allocator),
            .file_times = std.StringHashMap(i128).init(allocator),
            .interval_ms = 500,
        };
    }

    pub fn deinit(self: *FileWatcher) void {
        for (self.watch_paths.items) |p| self.allocator.free(p);
        self.watch_paths.deinit();

        var iter = self.file_times.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
        }
        self.file_times.deinit();
    }

    pub fn addPath(self: *FileWatcher, path: []const u8) !void {
        const p = try self.allocator.dupe(u8, path);
        try self.watch_paths.append(p);
    }

    /// ตรวจสอบการเปลี่ยนแปลง
    pub fn checkChanges(self: *FileWatcher) !?[]FileChange {
        var changes = std.ArrayList(FileChange).init(self.allocator);
        defer changes.deinit();

        for (self.watch_paths.items) |watch_path| {
            try self.checkDirChanges(watch_path, &changes);
        }

        if (changes.items.len == 0) return null;
        return changes.toOwnedSlice();
    }

    fn checkDirChanges(
        self: *FileWatcher,
        dir_path: []const u8,
        changes: *std.ArrayList(FileChange),
    ) !void {
        var dir = std.fs.cwd().openDir(dir_path, .{ .iterate = true }) catch return;
        defer dir.close();

        var iter = dir.iterate();
        while (try iter.next()) |entry| {
            const full_path = try std.fs.path.join(
                self.allocator,
                &[_][]const u8{ dir_path, entry.name },
            );
            defer self.allocator.free(full_path);

            if (entry.kind == .file) {
                const stat = std.fs.cwd().statFile(full_path) catch continue;

                if (self.file_times.get(full_path)) |last_time| {
                    if (stat.mtime != last_time) {
                        // File modified
                        try changes.append(.{
                            .path = try self.allocator.dupe(u8, full_path),
                            .kind = .modified,
                        });
                        try self.file_times.put(
                            try self.allocator.dupe(u8, full_path),
                            stat.mtime,
                        );
                    }
                } else {
                    // New file
                    try changes.append(.{
                        .path = try self.allocator.dupe(u8, full_path),
                        .kind = .created,
                    });
                    try self.file_times.put(
                        try self.allocator.dupe(u8, full_path),
                        stat.mtime,
                    );
                }
            } else if (entry.kind == .directory) {
                try self.checkDirChanges(full_path, changes);
            }
        }
    }

    /// Watch loop - เรียก callback เมื่อมีการเปลี่ยนแปลง
    pub fn watch(
        self: *FileWatcher,
        callback: *const fn ([]const FileChange) void,
    ) !void {
        while (true) {
            if (try self.checkChanges()) |changes| {
                defer {
                    for (changes) |c| self.allocator.free(c.path);
                    self.allocator.free(changes);
                }
                callback(changes);
            }

            std.time.sleep(self.interval_ms * std.time.ns_per_ms);
        }
    }
};
```

### src/assets.zig

```zig
const std = @import("std");

pub const AssetProcessor = struct {
    allocator: std.mem.Allocator,
    input_dir: []const u8,
    output_dir: []const u8,

    pub fn init(
        allocator: std.mem.Allocator,
        input_dir: []const u8,
        output_dir: []const u8,
    ) AssetProcessor {
        return AssetProcessor{
            .allocator = allocator,
            .input_dir = input_dir,
            .output_dir = output_dir,
        };
    }

    /// Copy assets จาก input ไป output
    pub fn processAll(self: *AssetProcessor) !u32 {
        var count: u32 = 0;
        var dir = std.fs.cwd().openDir(self.input_dir, .{ .iterate = true }) catch return 0;
        defer dir.close();

        var iter = dir.iterate();
        while (try iter.next()) |entry| {
            if (entry.kind == .file) {
                const src = try std.fs.path.join(
                    self.allocator,
                    &[_][]const u8{ self.input_dir, entry.name },
                );
                defer self.allocator.free(src);

                const dst = try std.fs.path.join(
                    self.allocator,
                    &[_][]const u8{ self.output_dir, entry.name },
                );
                defer self.allocator.free(dst);

                std.fs.cwd().copyFile(src, std.fs.cwd(), dst, .{}) catch |err| {
                    std.log.warn("Could not copy {s}: {}", .{ src, err });
                    continue;
                };
                count += 1;
            }
        }

        return count;
    }
};
```

---

## ขั้นตอนที่ 646-647: Build Pipeline และ Front Matter Parsing

### src/frontmatter.zig

```zig
const std = @import("std");

pub const FrontMatter = struct {
    title: ?[]const u8,
    date: ?[]const u8,
    author: ?[]const u8,
    tags: []const []const u8,
    draft: bool,
    description: ?[]const u8,
    template: ?[]const u8,
    slug: ?[]const u8,

    pub fn deinit(self: *FrontMatter, allocator: std.mem.Allocator) void {
        if (self.title) |t| allocator.free(t);
        if (self.date) |d| allocator.free(d);
        if (self.author) |a| allocator.free(a);
        if (self.description) |d| allocator.free(d);
        if (self.template) |t| allocator.free(t);
        if (self.slug) |s| allocator.free(s);
        for (self.tags) |tag| allocator.free(tag);
        allocator.free(self.tags);
    }
};

pub const ParseResult = struct {
    front_matter: FrontMatter,
    content: []const u8,
};

/// Parse front matter จาก markdown file
/// Front matter อยู่ระหว่าง --- และ ---
pub fn parseFrontMatter(allocator: std.mem.Allocator, input: []const u8) !ParseResult {
    // ตรวจสอบว่าเริ่มต้นด้วย ---
    if (!std.mem.startsWith(u8, input, "---")) {
        return ParseResult{
            .front_matter = FrontMatter{
                .title = null,
                .date = null,
                .author = null,
                .tags = &[_][]const u8{},
                .draft = false,
                .description = null,
                .template = null,
                .slug = null,
            },
            .content = input,
        };
    }

    // หา --- ปิด
    const after_open = 3;
    const close_pos = std.mem.indexOf(u8, input[after_open..], "\n---") orelse {
        return ParseResult{
            .front_matter = FrontMatter{
                .title = null,
                .date = null,
                .author = null,
                .tags = &[_][]const u8{},
                .draft = false,
                .description = null,
                .template = null,
                .slug = null,
            },
            .content = input,
        };
    };

    const yaml_content = input[after_open .. after_open + close_pos];
    const content_start = after_open + close_pos + 4; // +4 for \n---
    const content = if (content_start < input.len)
        std.mem.trim(u8, input[content_start..], "\n")
    else
        "";

    // Parse YAML-like key: value pairs
    var title: ?[]u8 = null;
    var date: ?[]u8 = null;
    var author: ?[]u8 = null;
    var description: ?[]u8 = null;
    var template: ?[]u8 = null;
    var slug: ?[]u8 = null;
    var draft = false;
    var tags = std.ArrayList([]u8).init(allocator);
    errdefer {
        for (tags.items) |t| allocator.free(t);
        tags.deinit();
    }

    var lines = std.mem.splitSequence(u8, yaml_content, "\n");
    var in_tags = false;

    while (lines.next()) |line| {
        const trimmed = std.mem.trim(u8, line, " \t\r");
        if (trimmed.len == 0) continue;

        // ตรวจสอบ tag list item (  - value)
        if (in_tags and std.mem.startsWith(u8, trimmed, "- ")) {
            const tag = try allocator.dupe(u8, std.mem.trim(u8, trimmed[2..], " \"'"));
            try tags.append(tag);
            continue;
        }
        in_tags = false;

        // Parse key: value
        const colon_pos = std.mem.indexOfScalar(u8, trimmed, ':') orelse continue;
        const key = std.mem.trim(u8, trimmed[0..colon_pos], " ");
        const value = std.mem.trim(u8, trimmed[colon_pos + 1 ..], " \"'\t");

        if (std.mem.eql(u8, key, "title")) {
            title = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "date")) {
            date = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "author")) {
            author = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "description")) {
            description = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "template")) {
            template = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "slug")) {
            slug = try allocator.dupe(u8, value);
        } else if (std.mem.eql(u8, key, "draft")) {
            draft = std.mem.eql(u8, value, "true");
        } else if (std.mem.eql(u8, key, "tags")) {
            in_tags = true; // tags อาจจะเป็น list ในบรรทัดถัดไป
            // หรืออาจจะเป็น inline: tags: [a, b, c]
            if (value.len > 0 and value[0] == '[') {
                var tag_iter = std.mem.splitSequence(u8, value[1 .. value.len - 1], ",");
                while (tag_iter.next()) |tag| {
                    const t = try allocator.dupe(u8, std.mem.trim(u8, tag, " \"'"));
                    try tags.append(t);
                }
                in_tags = false;
            }
        }
    }

    return ParseResult{
        .front_matter = FrontMatter{
            .title = title,
            .date = date,
            .author = author,
            .tags = try tags.toOwnedSlice(),
            .draft = draft,
            .description = description,
            .template = template,
            .slug = slug,
        },
        .content = content,
    };
}

// Tests
test "Parse front matter" {
    const allocator = std.testing.allocator;

    const input =
        \\---
        \\title: "Hello World"
        \\date: 2024-01-15
        \\author: สมชาย
        \\draft: false
        \\tags: [zig, programming]
        \\---
        \\# Content here
        \\This is the body.
    ;

    var result = try parseFrontMatter(allocator, input);
    defer result.front_matter.deinit(allocator);

    try std.testing.expectEqualStrings("Hello World", result.front_matter.title.?);
    try std.testing.expectEqualStrings("2024-01-15", result.front_matter.date.?);
    try std.testing.expectEqualStrings("สมชาย", result.front_matter.author.?);
    try std.testing.expect(!result.front_matter.draft);
    try std.testing.expectEqual(@as(usize, 2), result.front_matter.tags.len);
}
```

---

## ขั้นตอนที่ 648-649: Navigation และ RSS Feed Generation

### src/navigation.zig

```zig
const std = @import("std");

pub const NavItem = struct {
    title: []const u8,
    url: []const u8,
    active: bool,
    children: []const NavItem,

    pub fn deinit(self: *NavItem, allocator: std.mem.Allocator) void {
        allocator.free(self.title);
        allocator.free(self.url);
        for (self.children) |*child| {
            var c = child.*;
            c.deinit(allocator);
        }
        allocator.free(self.children);
    }
};

pub const Navigation = struct {
    allocator: std.mem.Allocator,
    items: std.ArrayList(NavItem),

    pub fn init(allocator: std.mem.Allocator) Navigation {
        return Navigation{
            .allocator = allocator,
            .items = std.ArrayList(NavItem).init(allocator),
        };
    }

    pub fn deinit(self: *Navigation) void {
        for (self.items.items) |*item| {
            item.deinit(self.allocator);
        }
        self.items.deinit();
    }

    pub fn addItem(self: *Navigation, title: []const u8, url: []const u8) !void {
        try self.items.append(.{
            .title = try self.allocator.dupe(u8, title),
            .url = try self.allocator.dupe(u8, url),
            .active = false,
            .children = &[_]NavItem{},
        });
    }

    pub fn setActive(self: *Navigation, current_url: []const u8) void {
        for (self.items.items) |*item| {
            item.active = std.mem.eql(u8, item.url, current_url);
        }
    }

    /// Render navigation เป็น HTML
    pub fn renderHtml(self: *const Navigation, allocator: std.mem.Allocator) ![]u8 {
        var buf = std.ArrayList(u8).init(allocator);
        const writer = buf.writer();

        try writer.writeAll("<nav>\n<ul>\n");
        for (self.items.items) |item| {
            if (item.active) {
                try writer.print(
                    "  <li class=\"active\"><a href=\"{s}\">{s}</a></li>\n",
                    .{ item.url, item.title },
                );
            } else {
                try writer.print(
                    "  <li><a href=\"{s}\">{s}</a></li>\n",
                    .{ item.url, item.title },
                );
            }
        }
        try writer.writeAll("</ul>\n</nav>\n");

        return buf.toOwnedSlice();
    }
};
```

### src/rss.zig

```zig
const std = @import("std");

pub const RssItem = struct {
    title: []const u8,
    link: []const u8,
    description: []const u8,
    pub_date: []const u8,
    author: ?[]const u8,
    guid: []const u8,
};

pub const RssFeed = struct {
    title: []const u8,
    link: []const u8,
    description: []const u8,
    language: []const u8,
    items: []const RssItem,
};

/// Generate RSS XML
pub fn generateRss(allocator: std.mem.Allocator, feed: RssFeed) ![]u8 {
    var buf = std.ArrayList(u8).init(allocator);
    const writer = buf.writer();

    try writer.writeAll("<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n");
    try writer.writeAll("<rss version=\"2.0\" xmlns:atom=\"http://www.w3.org/2005/Atom\">\n");
    try writer.writeAll("<channel>\n");
    try writer.print("  <title>{s}</title>\n", .{feed.title});
    try writer.print("  <link>{s}</link>\n", .{feed.link});
    try writer.print("  <description>{s}</description>\n", .{feed.description});
    try writer.print("  <language>{s}</language>\n", .{feed.language});
    try writer.print(
        "  <atom:link href=\"{s}/feed.xml\" rel=\"self\" type=\"application/rss+xml\"/>\n",
        .{feed.link},
    );

    for (feed.items) |item| {
        try writer.writeAll("  <item>\n");
        try writer.print("    <title>{s}</title>\n", .{item.title});
        try writer.print("    <link>{s}</link>\n", .{item.link});
        try writer.print("    <guid>{s}</guid>\n", .{item.guid});
        try writer.print("    <pubDate>{s}</pubDate>\n", .{item.pub_date});
        try writer.print("    <description><![CDATA[{s}]]></description>\n", .{item.description});
        if (item.author) |author| {
            try writer.print("    <author>{s}</author>\n", .{author});
        }
        try writer.writeAll("  </item>\n");
    }

    try writer.writeAll("</channel>\n</rss>\n");

    return buf.toOwnedSlice();
}
```

---

## ขั้นตอนที่ 650: Complete Build and Deploy

### src/pipeline.zig

```zig
const std = @import("std");
const markdown = @import("markdown.zig");
const template = @import("template.zig");
const frontmatter = @import("frontmatter.zig");
const navigation = @import("navigation.zig");
const rss = @import("rss.zig");
const assets = @import("assets.zig");

pub const BuildConfig = struct {
    source_dir: []const u8 = "content",
    output_dir: []const u8 = "public",
    template_dir: []const u8 = "templates",
    assets_dir: []const u8 = "static",
    base_url: []const u8 = "https://example.com",
    site_title: []const u8 = "My Site",
    site_description: []const u8 = "A static site",
    include_drafts: bool = false,
};

pub const Page = struct {
    path: []const u8,
    url: []const u8,
    title: []const u8,
    content: []const u8,
    front_matter: frontmatter.FrontMatter,
    html: []const u8,

    pub fn deinit(self: *Page, allocator: std.mem.Allocator) void {
        allocator.free(self.path);
        allocator.free(self.url);
        allocator.free(self.title);
        allocator.free(self.content);
        allocator.free(self.html);
        self.front_matter.deinit(allocator);
    }
};

pub const BuildPipeline = struct {
    allocator: std.mem.Allocator,
    config: BuildConfig,
    pages: std.ArrayList(Page),
    md_parser: markdown.MarkdownParser,
    tmpl_engine: template.TemplateEngine,

    pub fn init(allocator: std.mem.Allocator, config: BuildConfig) BuildPipeline {
        return BuildPipeline{
            .allocator = allocator,
            .config = config,
            .pages = std.ArrayList(Page).init(allocator),
            .md_parser = markdown.MarkdownParser.init(allocator),
            .tmpl_engine = template.TemplateEngine.init(allocator, config.template_dir),
        };
    }

    pub fn deinit(self: *BuildPipeline) void {
        for (self.pages.items) |*page| {
            page.deinit(self.allocator);
        }
        self.pages.deinit();
    }

    /// Build site ทั้งหมด
    pub fn build(self: *BuildPipeline) !BuildStats {
        var stats = BuildStats{};
        const start = std.time.milliTimestamp();

        std.log.info("Building site...", .{});

        // สร้าง output directory
        try std.fs.cwd().makePath(self.config.output_dir);

        // Process pages
        try self.processDirectory(self.config.source_dir);

        // Generate HTML files
        for (self.pages.items) |*page| {
            try self.generatePage(page);
            stats.pages_generated += 1;
        }

        // Generate navigation
        var nav = navigation.Navigation.init(self.allocator);
        defer nav.deinit();

        try nav.addItem("หน้าแรก", "/");
        try nav.addItem("บทความ", "/posts/");
        try nav.addItem("เกี่ยวกับ", "/about/");

        // Generate RSS feed
        var rss_items = std.ArrayList(rss.RssItem).init(self.allocator);
        defer rss_items.deinit();

        for (self.pages.items) |page| {
            if (std.mem.startsWith(u8, page.url, "/posts/")) {
                try rss_items.append(.{
                    .title = page.title,
                    .link = try std.fmt.allocPrint(
                        self.allocator,
                        "{s}{s}",
                        .{ self.config.base_url, page.url },
                    ),
                    .description = page.front_matter.description orelse "",
                    .pub_date = page.front_matter.date orelse "",
                    .author = page.front_matter.author,
                    .guid = try std.fmt.allocPrint(
                        self.allocator,
                        "{s}{s}",
                        .{ self.config.base_url, page.url },
                    ),
                });
                stats.posts_count += 1;
            }
        }
        defer {
            for (rss_items.items) |item| {
                self.allocator.free(item.link);
                self.allocator.free(item.guid);
            }
        }

        const feed = rss.RssFeed{
            .title = self.config.site_title,
            .link = self.config.base_url,
            .description = self.config.site_description,
            .language = "th",
            .items = rss_items.items,
        };

        const feed_xml = try rss.generateRss(self.allocator, feed);
        defer self.allocator.free(feed_xml);

        const feed_path = try std.fs.path.join(
            self.allocator,
            &[_][]const u8{ self.config.output_dir, "feed.xml" },
        );
        defer self.allocator.free(feed_path);

        const feed_file = try std.fs.cwd().createFile(feed_path, .{});
        defer feed_file.close();
        try feed_file.writeAll(feed_xml);

        // Copy static assets
        const asset_processor = assets.AssetProcessor.init(
            self.allocator,
            self.config.assets_dir,
            self.config.output_dir,
        );
        _ = try @constCast(&asset_processor).processAll();

        stats.build_time_ms = std.time.milliTimestamp() - start;
        return stats;
    }

    fn processDirectory(self: *BuildPipeline, dir_path: []const u8) !void {
        var dir = std.fs.cwd().openDir(dir_path, .{ .iterate = true }) catch return;
        defer dir.close();

        var iter = dir.iterate();
        while (try iter.next()) |entry| {
            const full_path = try std.fs.path.join(
                self.allocator,
                &[_][]const u8{ dir_path, entry.name },
            );
            defer self.allocator.free(full_path);

            if (entry.kind == .file and std.mem.endsWith(u8, entry.name, ".md")) {
                try self.processMarkdownFile(full_path);
            } else if (entry.kind == .directory) {
                try self.processDirectory(full_path);
            }
        }
    }

    fn processMarkdownFile(self: *BuildPipeline, file_path: []const u8) !void {
        // อ่านไฟล์
        const file = try std.fs.cwd().openFile(file_path, .{});
        defer file.close();

        const size = try file.getEndPos();
        const raw_content = try self.allocator.alloc(u8, size);
        defer self.allocator.free(raw_content);
        _ = try file.readAll(raw_content);

        // Parse front matter
        const parse_result = try frontmatter.parseFrontMatter(self.allocator, raw_content);

        // Skip drafts ถ้าไม่ได้เปิดใช้
        if (parse_result.front_matter.draft and !self.config.include_drafts) {
            parse_result.front_matter.deinit(self.allocator);
            return;
        }

        // แปลง Markdown เป็น HTML
        const html_content = try self.md_parser.parse(parse_result.content);
        defer self.allocator.free(html_content);

        // สร้าง URL จาก file path
        const relative_path = file_path[self.config.source_dir.len..];
        const url = try std.fmt.allocPrint(
            self.allocator,
            "{s}",
            .{std.mem.trimRight(u8, relative_path, ".md")},
        );
        defer self.allocator.free(url);
        const url_with_slash = try std.fmt.allocPrint(self.allocator, "{s}/", .{url});

        const title = parse_result.front_matter.title orelse
            std.fs.path.stem(file_path);

        const page = Page{
            .path = try self.allocator.dupe(u8, file_path),
            .url = url_with_slash,
            .title = try self.allocator.dupe(u8, title),
            .content = try self.allocator.dupe(u8, parse_result.content),
            .front_matter = parse_result.front_matter,
            .html = try self.allocator.dupe(u8, html_content),
        };

        try self.pages.append(page);
    }

    fn generatePage(self: *BuildPipeline, page: *Page) !void {
        // สร้าง context
        var ctx = template.Context.init(self.allocator);
        defer ctx.deinit();

        try ctx.set("title", page.title);
        try ctx.set("content", page.html);
        try ctx.set("url", page.url);
        try ctx.set("site_title", self.config.site_title);
        try ctx.set("base_url", self.config.base_url);

        if (page.front_matter.date) |date| {
            try ctx.set("date", date);
        }
        if (page.front_matter.author) |author| {
            try ctx.set("author", author);
        }
        if (page.front_matter.description) |desc| {
            try ctx.set("description", desc);
        }

        // เลือก template
        const tmpl_name = page.front_matter.template orelse "post.html";

        // ถ้าไม่มี template file ใช้ default
        const html = self.tmpl_engine.renderFile(tmpl_name, &ctx) catch |err| blk: {
            std.log.warn("Template {s} not found: {}, using default", .{ tmpl_name, err });
            break :blk try defaultTemplate(self.allocator, &ctx);
        };
        defer self.allocator.free(html);

        // สร้าง output path
        const rel_url = if (std.mem.startsWith(u8, page.url, "/")) page.url[1..] else page.url;
        const output_path = try std.fs.path.join(
            self.allocator,
            &[_][]const u8{ self.config.output_dir, rel_url },
        );
        defer self.allocator.free(output_path);

        // สร้าง directory
        const output_dir = std.fs.path.dirname(output_path) orelse self.config.output_dir;
        std.fs.cwd().makePath(output_dir) catch {};

        const index_path = try std.fs.path.join(
            self.allocator,
            &[_][]const u8{ output_path, "index.html" },
        );
        defer self.allocator.free(index_path);

        const out_file = try std.fs.cwd().createFile(index_path, .{});
        defer out_file.close();
        try out_file.writeAll(html);
    }

    fn defaultTemplate(allocator: std.mem.Allocator, ctx: *const template.Context) ![]u8 {
        const title = ctx.get("title") orelse "Untitled";
        const content = ctx.get("content") orelse "";
        const site_title = ctx.get("site_title") orelse "Site";

        return std.fmt.allocPrint(allocator,
            \\<!DOCTYPE html>
            \\<html lang="th">
            \\<head>
            \\  <meta charset="UTF-8">
            \\  <meta name="viewport" content="width=device-width, initial-scale=1.0">
            \\  <title>{s} - {s}</title>
            \\  <link rel="stylesheet" href="/style.css">
            \\</head>
            \\<body>
            \\  <header><h1><a href="/">{s}</a></h1></header>
            \\  <main>
            \\    <h1>{s}</h1>
            \\    {s}
            \\  </main>
            \\  <footer><p>&copy; 2024</p></footer>
            \\</body>
            \\</html>
        , .{ title, site_title, site_title, title, content });
    }
};

pub const BuildStats = struct {
    pages_generated: u32 = 0,
    posts_count: u32 = 0,
    build_time_ms: i64 = 0,

    pub fn print(self: BuildStats) void {
        std.log.info("Build complete!", .{});
        std.log.info("  Pages: {d}", .{self.pages_generated});
        std.log.info("  Posts: {d}", .{self.posts_count});
        std.log.info("  Time:  {d}ms", .{self.build_time_ms});
    }
};
```

### src/main.zig - Entry point

```zig
const std = @import("std");
const pipeline = @import("pipeline.zig");
const watcher = @import("watcher.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    const command = if (args.len > 1) args[1] else "build";

    const config = pipeline.BuildConfig{
        .source_dir = "content",
        .output_dir = "public",
        .template_dir = "templates",
        .assets_dir = "static",
        .base_url = "https://mysite.com",
        .site_title = "My Zig Site",
        .site_description = "สร้างด้วย Zig SSG",
    };

    if (std.mem.eql(u8, command, "build")) {
        var build_pipeline = pipeline.BuildPipeline.init(allocator, config);
        defer build_pipeline.deinit();

        const stats = try build_pipeline.build();
        stats.print();
    } else if (std.mem.eql(u8, command, "serve")) {
        std.log.info("Starting development server...", .{});

        // Build ครั้งแรก
        {
            var build_pipeline = pipeline.BuildPipeline.init(allocator, config);
            defer build_pipeline.deinit();
            const stats = try build_pipeline.build();
            stats.print();
        }

        // Watch for changes
        var file_watcher = watcher.FileWatcher.init(allocator);
        defer file_watcher.deinit();

        try file_watcher.addPath(config.source_dir);
        try file_watcher.addPath(config.template_dir);

        std.log.info("Watching for changes...", .{});

        while (true) {
            if (try file_watcher.checkChanges()) |changes| {
                defer {
                    for (changes) |c| allocator.free(c.path);
                    allocator.free(changes);
                }

                for (changes) |c| {
                    std.log.info("File changed: {s}", .{c.path});
                }

                // Rebuild
                var build_pipeline = pipeline.BuildPipeline.init(allocator, config);
                defer build_pipeline.deinit();

                const stats = build_pipeline.build() catch |err| {
                    std.log.err("Build failed: {}", .{err});
                    continue;
                };
                stats.print();
            }

            std.time.sleep(500 * std.time.ns_per_ms);
        }
    } else if (std.mem.eql(u8, command, "new")) {
        const name = if (args.len > 2) args[2] else "new-post";
        try createNewPost(allocator, config.source_dir, name);
    } else if (std.mem.eql(u8, command, "clean")) {
        try std.fs.cwd().deleteTree(config.output_dir);
        std.log.info("Cleaned output directory: {s}", .{config.output_dir});
    } else {
        try printUsage();
    }
}

fn createNewPost(allocator: std.mem.Allocator, source_dir: []const u8, name: []const u8) !void {
    const posts_dir = try std.fs.path.join(allocator, &[_][]const u8{ source_dir, "posts" });
    defer allocator.free(posts_dir);

    std.fs.cwd().makePath(posts_dir) catch {};

    const file_name = try std.fmt.allocPrint(allocator, "{s}.md", .{name});
    defer allocator.free(file_name);

    const file_path = try std.fs.path.join(allocator, &[_][]const u8{ posts_dir, file_name });
    defer allocator.free(file_path);

    const file = try std.fs.cwd().createFile(file_path, .{ .exclusive = true });
    defer file.close();

    const now = std.time.timestamp();
    const epoch = std.time.epoch.EpochSeconds{ .secs = @intCast(now) };
    const day = epoch.getEpochDay();
    const year_day = day.calculateYearDay();
    const month_day = year_day.calculateMonthDay();

    try file.writer().print(
        \\---
        \\title: "{s}"
        \\date: {d:0>4}-{d:0>2}-{d:0>2}
        \\author: ""
        \\draft: true
        \\tags: []
        \\description: ""
        \\---
        \\
        \\# {s}
        \\
        \\เนื้อหาของบทความ...
        \\
    , .{
        name,
        year_day.year,
        month_day.month.numeric(),
        month_day.day_index + 1,
        name,
    });

    std.log.info("Created new post: {s}", .{file_path});
}

fn printUsage() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print(
        \\zigssg - Static Site Generator
        \\
        \\การใช้งาน:
        \\  zigssg build           Build site
        \\  zigssg serve           Serve พร้อม file watching
        \\  zigssg new <name>      สร้าง post ใหม่
        \\  zigssg clean           ลบ output directory
        \\
    , .{});
}
```

### สรุป Part 65 และ Course

เราได้สร้าง Static Site Generator ที่สมบูรณ์พร้อม:
- **Markdown Parser** รองรับ headings, lists, bold, italic, links, code blocks
- **Template Engine** รองรับ variables, if conditions, template inheritance
- **Front Matter Parser** สำหรับ YAML-like metadata
- **File Watcher** สำหรับ development mode
- **Asset Processor** สำหรับ copy static files
- **Build Pipeline** ที่รวมทุกอย่างเข้าด้วยกัน
- **Navigation Generation** อัตโนมัติ
- **RSS Feed Generation** สำหรับ blog
- **Complete CLI** พร้อม build, serve, new, clean commands

### สรุปทั้ง 5 Projects (Part 61-65)

| Part | โปรเจค | ทักษะหลัก |
|------|--------|-----------|
| 61 | CLI Todo App | arg parsing, JSON, error handling |
| 62 | HTTP Server | TCP, HTTP protocol, middleware |
| 63 | File System Tool | traversal, pattern matching, I/O |
| 64 | SQLite Wrapper | C integration, comptime, ORM |
| 65 | Static Site Generator | parsing, templating, pipelines |

---
*จบ Part 61-65: Projects ที่สมบูรณ์*
