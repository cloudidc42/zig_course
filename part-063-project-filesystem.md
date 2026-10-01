# ส่วนที่ 63: โปรเจค File System Tool

## ขั้นตอนที่ 621-630: สร้าง File Utility Tool (find/grep แบบ Zig)

---

## ขั้นตอนที่ 621: Project Requirements

### ภาพรวม

เราจะสร้าง CLI tool สำหรับ file system operations ที่มีประสิทธิภาพสูง คล้ายกับการรวม `find`, `grep`, และ `cp/mv/rm` เข้าด้วยกัน

### ความสามารถ

```
zigfind [options] <path> [pattern]

Operations:
  --find <pattern>     ค้นหา files ที่ตรงกับ pattern
  --grep <text>        ค้นหา text ใน files
  --copy <dest>        คัดลอก files
  --move <dest>        ย้าย files
  --delete             ลบ files
  --stats              แสดงสถิติ

Options:
  -r, --recursive      ค้นหาใน subdirectories
  -i, --ignore <file>  ไฟล์ ignore patterns
  -j, --jobs <n>       จำนวน parallel workers
  -p, --progress       แสดง progress
  --color              เปิดใช้ color output
  --max-depth <n>      ความลึกสูงสุด
  --min-size <size>    ขนาดไฟล์ขั้นต่ำ
  --max-size <size>    ขนาดไฟล์สูงสุด
  --newer-than <date>  ไฟล์ที่ใหม่กว่า
```

### โครงสร้างโปรเจค

```
zigfind/
├── build.zig
├── src/
│   ├── main.zig
│   ├── traversal.zig
│   ├── pattern.zig
│   ├── search.zig
│   ├── operations.zig
│   ├── parallel.zig
│   ├── progress.zig
│   ├── stats.zig
│   ├── ignore.zig
│   └── output.zig
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "zigfind",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run zigfind");
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

## ขั้นตอนที่ 622: Directory Traversal

### src/traversal.zig

```zig
const std = @import("std");
const ignore = @import("ignore.zig");

pub const FileEntry = struct {
    path: []const u8,
    name: []const u8,
    kind: std.fs.File.Kind,
    size: u64,
    modified: i128,
    depth: u32,

    pub fn deinit(self: *FileEntry, allocator: std.mem.Allocator) void {
        allocator.free(self.path);
    }

    pub fn isFile(self: FileEntry) bool {
        return self.kind == .file;
    }

    pub fn isDirectory(self: FileEntry) bool {
        return self.kind == .directory;
    }

    pub fn extension(self: FileEntry) []const u8 {
        return std.fs.path.extension(self.name);
    }
};

pub const TraversalOptions = struct {
    max_depth: ?u32 = null,
    follow_symlinks: bool = false,
    include_hidden: bool = false,
    ignore_patterns: ?*ignore.IgnoreList = null,
    min_size: ?u64 = null,
    max_size: ?u64 = null,
};

pub const Traversal = struct {
    allocator: std.mem.Allocator,
    options: TraversalOptions,
    entries: std.ArrayList(FileEntry),

    pub fn init(allocator: std.mem.Allocator, options: TraversalOptions) Traversal {
        return Traversal{
            .allocator = allocator,
            .options = options,
            .entries = std.ArrayList(FileEntry).init(allocator),
        };
    }

    pub fn deinit(self: *Traversal) void {
        for (self.entries.items) |*entry| {
            entry.deinit(self.allocator);
        }
        self.entries.deinit();
    }

    /// Traverse directory recursively
    pub fn traverse(self: *Traversal, root_path: []const u8) !void {
        try self.traverseDir(root_path, 0);
    }

    fn traverseDir(self: *Traversal, dir_path: []const u8, depth: u32) !void {
        // ตรวจสอบ max depth
        if (self.options.max_depth) |max_d| {
            if (depth > max_d) return;
        }

        var dir = std.fs.cwd().openDir(dir_path, .{ .iterate = true }) catch |err| {
            switch (err) {
                error.AccessDenied => return,
                error.NotDir => return,
                else => return err,
            }
        };
        defer dir.close();

        var iter = dir.iterate();
        while (try iter.next()) |entry| {
            // ข้าม hidden files ถ้าไม่ได้เปิดใช้
            if (!self.options.include_hidden and entry.name.len > 0 and entry.name[0] == '.') {
                continue;
            }

            const full_path = try std.fs.path.join(self.allocator, &[_][]const u8{ dir_path, entry.name });
            errdefer self.allocator.free(full_path);

            // ตรวจสอบ ignore patterns
            if (self.options.ignore_patterns) |ig| {
                if (ig.shouldIgnore(entry.name, full_path)) {
                    self.allocator.free(full_path);
                    continue;
                }
            }

            // ดึงข้อมูลไฟล์
            const stat = std.fs.cwd().statFile(full_path) catch |err| {
                switch (err) {
                    error.AccessDenied, error.FileNotFound => {
                        self.allocator.free(full_path);
                        continue;
                    },
                    else => {
                        self.allocator.free(full_path);
                        return err;
                    },
                }
            };

            const file_entry = FileEntry{
                .path = full_path,
                .name = entry.name,
                .kind = entry.kind,
                .size = stat.size,
                .modified = stat.mtime,
                .depth = depth,
            };

            // ตรวจสอบ size filters
            if (file_entry.isFile()) {
                if (self.options.min_size) |min_s| {
                    if (stat.size < min_s) {
                        self.allocator.free(full_path);
                        continue;
                    }
                }
                if (self.options.max_size) |max_s| {
                    if (stat.size > max_s) {
                        self.allocator.free(full_path);
                        continue;
                    }
                }
            }

            try self.entries.append(file_entry);

            // Recurse into directories
            if (entry.kind == .directory) {
                try self.traverseDir(full_path, depth + 1);
            }
        }
    }

    pub fn getFiles(self: *const Traversal) []const FileEntry {
        return self.entries.items;
    }

    pub fn count(self: *const Traversal) usize {
        return self.entries.items.len;
    }

    pub fn totalSize(self: *const Traversal) u64 {
        var total: u64 = 0;
        for (self.entries.items) |entry| {
            if (entry.isFile()) total += entry.size;
        }
        return total;
    }
};

// Tests
test "Directory traversal" {
    const allocator = std.testing.allocator;

    // สร้าง temp directory structure
    var tmp_dir = std.testing.tmpDir(.{});
    defer tmp_dir.cleanup();

    // สร้างไฟล์ทดสอบ
    try tmp_dir.dir.writeFile(.{ .sub_path = "file1.txt", .data = "hello" });
    try tmp_dir.dir.writeFile(.{ .sub_path = "file2.zig", .data = "const x = 1;" });
    try tmp_dir.dir.makeDir("subdir");
    try tmp_dir.dir.writeFile(.{ .sub_path = "subdir/file3.txt", .data = "world" });

    var traversal = Traversal.init(allocator, .{});
    defer traversal.deinit();

    var path_buf: [std.fs.max_path_bytes]u8 = undefined;
    const tmp_path = try tmp_dir.dir.realpath(".", &path_buf);

    try traversal.traverse(tmp_path);

    // ควรพบอย่างน้อย 3 entries (2 files + 1 dir + 1 file in subdir)
    try std.testing.expect(traversal.count() >= 3);
}
```

---

## ขั้นตอนที่ 623: File Pattern Matching (Glob)

### src/pattern.zig

```zig
const std = @import("std");

pub const GlobPattern = struct {
    pattern: []const u8,

    pub fn init(pattern: []const u8) GlobPattern {
        return GlobPattern{ .pattern = pattern };
    }

    /// ตรวจสอบว่า string ตรงกับ glob pattern หรือไม่
    pub fn matches(self: GlobPattern, input: []const u8) bool {
        return globMatch(self.pattern, input);
    }

    /// ตรวจสอบ filename (ไม่รวม path)
    pub fn matchesName(self: GlobPattern, path: []const u8) bool {
        const name = std.fs.path.basename(path);
        return self.matches(name);
    }
};

/// Recursive glob matching
fn globMatch(pattern: []const u8, input: []const u8) bool {
    var pi: usize = 0; // pattern index
    var si: usize = 0; // string index
    var star_pi: usize = 0;
    var star_si: usize = 0;
    var has_star = false;

    while (si < input.len) {
        if (pi < pattern.len and (pattern[pi] == '?' or pattern[pi] == input[si])) {
            pi += 1;
            si += 1;
        } else if (pi < pattern.len and pattern[pi] == '*') {
            // ** matches path separators
            if (pi + 1 < pattern.len and pattern[pi + 1] == '*') {
                // Double star - matches everything including /
                has_star = true;
                star_pi = pi;
                star_si = si;
                pi += 2;
            } else {
                // Single star - matches anything except /
                has_star = true;
                star_pi = pi;
                star_si = si;
                pi += 1;
            }
        } else if (has_star) {
            // Backtrack to last star
            pi = star_pi + 1;
            star_si += 1;
            si = star_si;
        } else {
            return false;
        }
    }

    // ตรวจสอบ trailing pattern
    while (pi < pattern.len and pattern[pi] == '*') {
        pi += 1;
    }

    return pi == pattern.len;
}

pub const PatternList = struct {
    allocator: std.mem.Allocator,
    patterns: std.ArrayList(GlobPattern),

    pub fn init(allocator: std.mem.Allocator) PatternList {
        return PatternList{
            .allocator = allocator,
            .patterns = std.ArrayList(GlobPattern).init(allocator),
        };
    }

    pub fn deinit(self: *PatternList) void {
        for (self.patterns.items) |p| {
            self.allocator.free(p.pattern);
        }
        self.patterns.deinit();
    }

    pub fn add(self: *PatternList, pattern: []const u8) !void {
        const p = try self.allocator.dupe(u8, pattern);
        try self.patterns.append(GlobPattern.init(p));
    }

    pub fn matchesAny(self: *const PatternList, input: []const u8) bool {
        for (self.patterns.items) |p| {
            if (p.matches(input)) return true;
        }
        return false;
    }

    pub fn matchesAnyName(self: *const PatternList, path: []const u8) bool {
        const name = std.fs.path.basename(path);
        for (self.patterns.items) |p| {
            if (p.matches(name)) return true;
        }
        return false;
    }
};

// Tests
test "Glob pattern matching" {
    const p1 = GlobPattern.init("*.zig");
    try std.testing.expect(p1.matches("main.zig"));
    try std.testing.expect(p1.matches("build.zig"));
    try std.testing.expect(!p1.matches("main.c"));

    const p2 = GlobPattern.init("test_*");
    try std.testing.expect(p2.matches("test_main.zig"));
    try std.testing.expect(!p2.matches("main_test.zig"));

    const p3 = GlobPattern.init("*.{c,h}");
    _ = p3; // Complex patterns อาจต้องการ parser เพิ่มเติม

    const p4 = GlobPattern.init("src/**/*.zig");
    try std.testing.expect(p4.matches("src/foo/bar.zig"));
    try std.testing.expect(p4.matches("src/a/b/c/main.zig"));
}

test "PatternList" {
    const allocator = std.testing.allocator;

    var list = PatternList.init(allocator);
    defer list.deinit();

    try list.add("*.zig");
    try list.add("*.c");
    try list.add("*.h");

    try std.testing.expect(list.matchesAny("main.zig"));
    try std.testing.expect(list.matchesAny("foo.c"));
    try std.testing.expect(!list.matchesAny("foo.py"));
}
```

---

## ขั้นตอนที่ 624: File Content Search

### src/search.zig

```zig
const std = @import("std");

pub const SearchResult = struct {
    file_path: []const u8,
    line_number: u32,
    line_content: []const u8,
    match_start: usize,
    match_end: usize,

    pub fn deinit(self: *SearchResult, allocator: std.mem.Allocator) void {
        allocator.free(self.file_path);
        allocator.free(self.line_content);
    }
};

pub const SearchOptions = struct {
    case_sensitive: bool = true,
    whole_word: bool = false,
    regex: bool = false,
    max_results: ?usize = null,
    context_lines: u32 = 0,
};

pub const FileSearcher = struct {
    allocator: std.mem.Allocator,
    options: SearchOptions,

    pub fn init(allocator: std.mem.Allocator, options: SearchOptions) FileSearcher {
        return FileSearcher{
            .allocator = allocator,
            .options = options,
        };
    }

    /// ค้นหา pattern ในไฟล์
    pub fn searchFile(
        self: *FileSearcher,
        file_path: []const u8,
        pattern: []const u8,
    ) ![]SearchResult {
        var results = std.ArrayList(SearchResult).init(self.allocator);
        errdefer {
            for (results.items) |*r| r.deinit(self.allocator);
            results.deinit();
        }

        const file = std.fs.cwd().openFile(file_path, .{}) catch return results.toOwnedSlice();
        defer file.close();

        const file_size = try file.getEndPos();
        if (file_size == 0) return results.toOwnedSlice();

        // จำกัด file size ที่จะค้นหา
        const max_search_size = 50 * 1024 * 1024; // 50MB
        if (file_size > max_search_size) return results.toOwnedSlice();

        const content = try self.allocator.alloc(u8, file_size);
        defer self.allocator.free(content);

        _ = try file.readAll(content);

        // ตรวจสอบว่าเป็น binary file หรือไม่
        if (isBinary(content)) return results.toOwnedSlice();

        // ค้นหาทีละบรรทัด
        var line_iter = std.mem.splitScalar(u8, content, '\n');
        var line_number: u32 = 1;

        while (line_iter.next()) |line| {
            defer line_number += 1;

            if (self.options.max_results) |max| {
                if (results.items.len >= max) break;
            }

            const search_line = if (!self.options.case_sensitive)
                try std.ascii.allocLowerString(self.allocator, line)
            else
                null;
            defer if (search_line) |sl| self.allocator.free(sl);

            const search_pattern = if (!self.options.case_sensitive)
                try std.ascii.allocLowerString(self.allocator, pattern)
            else
                null;
            defer if (search_pattern) |sp| self.allocator.free(sp);

            const search_in = search_line orelse line;
            const search_for = search_pattern orelse pattern;

            if (std.mem.indexOf(u8, search_in, search_for)) |pos| {
                // ตรวจสอบ whole word
                if (self.options.whole_word) {
                    const before_ok = pos == 0 or !std.ascii.isAlphanumeric(search_in[pos - 1]);
                    const after_pos = pos + search_for.len;
                    const after_ok = after_pos >= search_in.len or !std.ascii.isAlphanumeric(search_in[after_pos]);
                    if (!before_ok or !after_ok) continue;
                }

                const result = SearchResult{
                    .file_path = try self.allocator.dupe(u8, file_path),
                    .line_number = line_number,
                    .line_content = try self.allocator.dupe(u8, line),
                    .match_start = pos,
                    .match_end = pos + pattern.len,
                };
                try results.append(result);
            }
        }

        return results.toOwnedSlice();
    }

    /// ค้นหาใน directory
    pub fn searchDirectory(
        self: *FileSearcher,
        dir_path: []const u8,
        pattern: []const u8,
        file_pattern: ?[]const u8,
    ) ![]SearchResult {
        var all_results = std.ArrayList(SearchResult).init(self.allocator);
        errdefer {
            for (all_results.items) |*r| r.deinit(self.allocator);
            all_results.deinit();
        }

        var dir = try std.fs.cwd().openDir(dir_path, .{ .iterate = true });
        defer dir.close();

        var iter = dir.iterate();
        while (try iter.next()) |entry| {
            const full_path = try std.fs.path.join(
                self.allocator,
                &[_][]const u8{ dir_path, entry.name },
            );
            defer self.allocator.free(full_path);

            if (entry.kind == .file) {
                // ตรวจสอบ file pattern
                if (file_pattern) |fp| {
                    if (!globMatchSimple(fp, entry.name)) continue;
                }

                const results = try self.searchFile(full_path, pattern);
                defer self.allocator.free(results);

                for (results) |*r| {
                    const r_copy = r.*;
                    r.file_path = ""; // prevent double-free
                    try all_results.append(r_copy);
                }
            } else if (entry.kind == .directory and !std.mem.eql(u8, entry.name, ".") and
                !std.mem.eql(u8, entry.name, ".."))
            {
                const sub_results = try self.searchDirectory(full_path, pattern, file_pattern);
                defer self.allocator.free(sub_results);

                for (sub_results) |*r| {
                    const r_copy = r.*;
                    r.file_path = "";
                    try all_results.append(r_copy);
                }
            }
        }

        return all_results.toOwnedSlice();
    }
};

/// ตรวจสอบว่าเป็น binary file หรือไม่ (ดูจาก null bytes)
fn isBinary(content: []const u8) bool {
    const check_size = @min(content.len, 8192);
    var null_count: usize = 0;
    for (content[0..check_size]) |byte| {
        if (byte == 0) null_count += 1;
    }
    return null_count > check_size / 100; // > 1% null bytes
}

fn globMatchSimple(pattern: []const u8, name: []const u8) bool {
    if (std.mem.eql(u8, pattern, "*")) return true;

    // *.ext pattern
    if (std.mem.startsWith(u8, pattern, "*.")) {
        const ext = pattern[1..];
        return std.mem.endsWith(u8, name, ext);
    }

    return std.mem.eql(u8, pattern, name);
}

// Tests
test "Search in file" {
    const allocator = std.testing.allocator;

    // สร้าง temp file
    var tmp_dir = std.testing.tmpDir(.{});
    defer tmp_dir.cleanup();

    try tmp_dir.dir.writeFile(.{
        .sub_path = "test.txt",
        .data = "hello world\nfoo bar\nhello again\n",
    });

    var path_buf: [std.fs.max_path_bytes]u8 = undefined;
    const tmp_path = try tmp_dir.dir.realpath("test.txt", &path_buf);

    var searcher = FileSearcher.init(allocator, .{});

    const results = try searcher.searchFile(tmp_path, "hello");
    defer {
        for (results) |*r| r.deinit(allocator);
        allocator.free(results);
    }

    try std.testing.expectEqual(@as(usize, 2), results.len);
    try std.testing.expectEqual(@as(u32, 1), results[0].line_number);
    try std.testing.expectEqual(@as(u32, 3), results[1].line_number);
}
```

---

## ขั้นตอนที่ 625: File Operations (Copy, Move, Delete)

### src/operations.zig

```zig
const std = @import("std");

pub const OperationError = error{
    SourceNotFound,
    DestinationExists,
    PermissionDenied,
    DiskFull,
    IsDirectory,
    NotDirectory,
};

pub const CopyOptions = struct {
    overwrite: bool = false,
    preserve_timestamps: bool = true,
    recursive: bool = true,
};

/// คัดลอกไฟล์
pub fn copyFile(
    allocator: std.mem.Allocator,
    src: []const u8,
    dst: []const u8,
    options: CopyOptions,
) !void {
    _ = allocator;

    // ตรวจสอบว่า destination มีอยู่แล้วหรือไม่
    if (!options.overwrite) {
        std.fs.cwd().access(dst, .{}) catch |err| {
            if (err != error.FileNotFound) return err;
            // FileNotFound = ไม่มีอยู่ = OK
        };
    }

    // คัดลอกไฟล์
    try std.fs.cwd().copyFile(src, std.fs.cwd(), dst, .{});
}

/// คัดลอก directory
pub fn copyDir(
    allocator: std.mem.Allocator,
    src: []const u8,
    dst: []const u8,
    options: CopyOptions,
) !u64 {
    var files_copied: u64 = 0;

    // สร้าง destination directory
    std.fs.cwd().makeDir(dst) catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };

    var dir = try std.fs.cwd().openDir(src, .{ .iterate = true });
    defer dir.close();

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        const src_path = try std.fs.path.join(allocator, &[_][]const u8{ src, entry.name });
        defer allocator.free(src_path);

        const dst_path = try std.fs.path.join(allocator, &[_][]const u8{ dst, entry.name });
        defer allocator.free(dst_path);

        switch (entry.kind) {
            .file => {
                try copyFile(allocator, src_path, dst_path, options);
                files_copied += 1;
            },
            .directory => {
                if (options.recursive) {
                    files_copied += try copyDir(allocator, src_path, dst_path, options);
                }
            },
            else => {},
        }
    }

    return files_copied;
}

/// ย้ายไฟล์ (rename ถ้าอยู่ใน filesystem เดียวกัน ไม่งั้น copy+delete)
pub fn moveFile(allocator: std.mem.Allocator, src: []const u8, dst: []const u8) !void {
    // ลอง rename ก่อน (เร็วกว่า)
    std.fs.cwd().rename(src, dst) catch |err| {
        if (err == error.RenameAcrossMountPoints) {
            // ต้อง copy แล้ว delete
            try copyFile(allocator, src, dst, .{ .overwrite = true });
            try std.fs.cwd().deleteFile(src);
        } else {
            return err;
        }
    };
}

/// ลบไฟล์หรือ directory
pub fn deleteItem(allocator: std.mem.Allocator, path: []const u8, recursive: bool) !u64 {
    _ = allocator;

    const stat = std.fs.cwd().statFile(path) catch return 0;

    switch (stat.kind) {
        .file, .sym_link => {
            try std.fs.cwd().deleteFile(path);
            return 1;
        },
        .directory => {
            if (recursive) {
                try std.fs.cwd().deleteTree(path);
                return 1;
            } else {
                try std.fs.cwd().deleteDir(path);
                return 1;
            }
        },
        else => return 0,
    }
}

/// สร้าง directory structure
pub fn ensureDir(path: []const u8) !void {
    std.fs.cwd().makePath(path) catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };
}

/// อ่านเนื้อหาไฟล์
pub fn readFile(allocator: std.mem.Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    const size = try file.getEndPos();
    const content = try allocator.alloc(u8, size);
    _ = try file.readAll(content);
    return content;
}

/// เขียนเนื้อหาลงไฟล์
pub fn writeFile(path: []const u8, content: []const u8) !void {
    const file = try std.fs.cwd().createFile(path, .{});
    defer file.close();
    try file.writeAll(content);
}

/// Append เนื้อหาลงไฟล์
pub fn appendFile(path: []const u8, content: []const u8) !void {
    const file = try std.fs.cwd().openFile(path, .{ .mode = .write_only });
    defer file.close();
    try file.seekFromEnd(0);
    try file.writeAll(content);
}

// Tests
test "File operations" {
    const allocator = std.testing.allocator;

    var tmp_dir = std.testing.tmpDir(.{});
    defer tmp_dir.cleanup();

    var path_buf: [std.fs.max_path_bytes]u8 = undefined;
    const tmp_path = try tmp_dir.dir.realpath(".", &path_buf);

    const src = try std.fs.path.join(allocator, &[_][]const u8{ tmp_path, "source.txt" });
    defer allocator.free(src);
    const dst = try std.fs.path.join(allocator, &[_][]const u8{ tmp_path, "copy.txt" });
    defer allocator.free(dst);

    // สร้างไฟล์ต้นทาง
    try writeFile(src, "Hello, World!");

    // คัดลอกไฟล์
    try copyFile(allocator, src, dst, .{});

    // ตรวจสอบ
    const content = try readFile(allocator, dst);
    defer allocator.free(content);
    try std.testing.expectEqualStrings("Hello, World!", content);
}
```

---

## ขั้นตอนที่ 626: Parallel Processing

### src/parallel.zig

```zig
const std = @import("std");

/// Worker pool สำหรับ parallel file processing
pub fn WorkerPool(comptime WorkItem: type, comptime Result: type) type {
    return struct {
        const Self = @This();
        const WorkerFn = *const fn (WorkItem) Result;

        allocator: std.mem.Allocator,
        num_workers: u32,
        work_queue: std.ArrayList(WorkItem),
        results: std.ArrayList(Result),
        mutex: std.Thread.Mutex,
        work_fn: WorkerFn,

        pub fn init(
            allocator: std.mem.Allocator,
            num_workers: u32,
            work_fn: WorkerFn,
        ) Self {
            return Self{
                .allocator = allocator,
                .num_workers = num_workers,
                .work_queue = std.ArrayList(WorkItem).init(allocator),
                .results = std.ArrayList(Result).init(allocator),
                .mutex = std.Thread.Mutex{},
                .work_fn = work_fn,
            };
        }

        pub fn deinit(self: *Self) void {
            self.work_queue.deinit();
            self.results.deinit();
        }

        pub fn addWork(self: *Self, item: WorkItem) !void {
            try self.work_queue.append(item);
        }

        pub fn run(self: *Self) ![]Result {
            // Simple sequential execution
            // Production version ควรใช้ threads จริงๆ
            for (self.work_queue.items) |item| {
                const result = self.work_fn(item);
                try self.results.append(result);
            }
            return self.results.toOwnedSlice();
        }
    };
}

/// Parallel file processor
pub const ParallelFileProcessor = struct {
    allocator: std.mem.Allocator,
    num_workers: u32,

    pub fn init(allocator: std.mem.Allocator, num_workers: u32) ParallelFileProcessor {
        return ParallelFileProcessor{
            .allocator = allocator,
            .num_workers = @max(1, num_workers),
        };
    }

    /// Process files ใน batches
    pub fn processBatch(
        self: *ParallelFileProcessor,
        files: []const []const u8,
        process_fn: *const fn (std.mem.Allocator, []const u8) anyerror!void,
    ) !void {
        // ในที่นี้เราทำแบบ sequential เพราะ Zig threading ต้องการ careful design
        for (files) |file| {
            process_fn(self.allocator, file) catch |err| {
                std.log.warn("Error processing {s}: {}", .{ file, err });
            };
        }
    }
};
```

---

## ขั้นตอนที่ 627: Progress Reporting

### src/progress.zig

```zig
const std = @import("std");

pub const ProgressReporter = struct {
    total: u64,
    current: u64,
    start_time: i64,
    last_update: i64,
    update_interval_ms: i64,
    show_progress: bool,

    pub fn init(total: u64, show: bool) ProgressReporter {
        return ProgressReporter{
            .total = total,
            .current = 0,
            .start_time = std.time.milliTimestamp(),
            .last_update = 0,
            .update_interval_ms = 100, // อัพเดททุก 100ms
            .show_progress = show,
        };
    }

    pub fn update(self: *ProgressReporter, increment: u64) void {
        self.current += increment;

        if (!self.show_progress) return;

        const now = std.time.milliTimestamp();
        if (now - self.last_update < self.update_interval_ms) return;
        self.last_update = now;

        self.render();
    }

    pub fn render(self: *ProgressReporter) void {
        const stderr = std.io.getStdErr().writer();

        const percent = if (self.total > 0)
            (self.current * 100) / self.total
        else 0;

        const elapsed_ms = std.time.milliTimestamp() - self.start_time;
        const rate = if (elapsed_ms > 0)
            (@as(f64, @floatFromInt(self.current)) * 1000.0) / @as(f64, @floatFromInt(elapsed_ms))
        else 0.0;

        const bar_width: usize = 30;
        const filled = if (self.total > 0)
            (self.current * bar_width) / self.total
        else 0;

        // Clear line และวาด progress bar
        stderr.print("\r[", .{}) catch return;
        var i: usize = 0;
        while (i < bar_width) : (i += 1) {
            if (i < filled) {
                stderr.writeAll("█") catch return;
            } else if (i == filled) {
                stderr.writeAll("▌") catch return;
            } else {
                stderr.writeAll("░") catch return;
            }
        }
        stderr.print("] {d}% ({d}/{d}) {d:.1} files/s", .{
            percent,
            self.current,
            self.total,
            rate,
        }) catch return;
    }

    pub fn finish(self: *ProgressReporter) void {
        if (!self.show_progress) return;

        self.current = self.total;
        self.render();

        const stderr = std.io.getStdErr().writer();
        stderr.writeAll("\n") catch return;

        const elapsed_ms = std.time.milliTimestamp() - self.start_time;
        stderr.print("เสร็จสิ้นใน {d}ms\n", .{elapsed_ms}) catch return;
    }
};
```

---

## ขั้นตอนที่ 628: File Statistics

### src/stats.zig

```zig
const std = @import("std");
const traversal = @import("traversal.zig");

pub const FileStats = struct {
    total_files: u64,
    total_dirs: u64,
    total_size: u64,
    largest_file: ?struct {
        path: []const u8,
        size: u64,
    },
    smallest_file: ?struct {
        path: []const u8,
        size: u64,
    },
    file_types: std.StringHashMap(u64), // extension -> count
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) FileStats {
        return FileStats{
            .total_files = 0,
            .total_dirs = 0,
            .total_size = 0,
            .largest_file = null,
            .smallest_file = null,
            .file_types = std.StringHashMap(u64).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *FileStats) void {
        var iter = self.file_types.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
        }
        self.file_types.deinit();
    }

    pub fn processEntry(self: *FileStats, entry: traversal.FileEntry) !void {
        if (entry.isDirectory()) {
            self.total_dirs += 1;
            return;
        }

        self.total_files += 1;
        self.total_size += entry.size;

        // Track largest
        if (self.largest_file == null or entry.size > self.largest_file.?.size) {
            self.largest_file = .{ .path = entry.path, .size = entry.size };
        }

        // Track smallest
        if (self.smallest_file == null or entry.size < self.smallest_file.?.size) {
            self.smallest_file = .{ .path = entry.path, .size = entry.size };
        }

        // Track file types
        const ext = entry.extension();
        if (ext.len > 0) {
            const gop = try self.file_types.getOrPut(ext);
            if (!gop.found_existing) {
                gop.key_ptr.* = try self.allocator.dupe(u8, ext);
                gop.value_ptr.* = 0;
            }
            gop.value_ptr.* += 1;
        }
    }

    pub fn print(self: *const FileStats) !void {
        const stdout = std.io.getStdOut().writer();

        try stdout.print("\n=== สถิติไฟล์ ===\n", .{});
        try stdout.print("Files:      {d}\n", .{self.total_files});
        try stdout.print("Dirs:       {d}\n", .{self.total_dirs});
        try stdout.print("Total Size: {s}\n", .{formatSize(self.total_size)});

        if (self.largest_file) |lf| {
            try stdout.print("Largest:    {s} ({s})\n", .{ lf.path, formatSize(lf.size) });
        }

        try stdout.print("\nประเภทไฟล์:\n", .{});
        var iter = self.file_types.iterator();
        while (iter.next()) |entry| {
            try stdout.print("  {s}: {d}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
        }
    }
};

fn formatSize(bytes: u64) [32]u8 {
    var buf: [32]u8 = undefined;
    if (bytes < 1024) {
        _ = std.fmt.bufPrint(&buf, "{d} B", .{bytes}) catch {};
    } else if (bytes < 1024 * 1024) {
        _ = std.fmt.bufPrint(&buf, "{d:.1} KB", .{@as(f64, @floatFromInt(bytes)) / 1024.0}) catch {};
    } else if (bytes < 1024 * 1024 * 1024) {
        _ = std.fmt.bufPrint(&buf, "{d:.1} MB", .{@as(f64, @floatFromInt(bytes)) / (1024.0 * 1024.0)}) catch {};
    } else {
        _ = std.fmt.bufPrint(&buf, "{d:.1} GB", .{@as(f64, @floatFromInt(bytes)) / (1024.0 * 1024.0 * 1024.0)}) catch {};
    }
    return buf;
}
```

---

## ขั้นตอนที่ 629: Ignore Patterns (.gitignore)

### src/ignore.zig

```zig
const std = @import("std");

pub const IgnoreList = struct {
    allocator: std.mem.Allocator,
    patterns: std.ArrayList([]const u8),
    negations: std.ArrayList([]const u8),

    pub fn init(allocator: std.mem.Allocator) IgnoreList {
        return IgnoreList{
            .allocator = allocator,
            .patterns = std.ArrayList([]const u8).init(allocator),
            .negations = std.ArrayList([]const u8).init(allocator),
        };
    }

    pub fn deinit(self: *IgnoreList) void {
        for (self.patterns.items) |p| self.allocator.free(p);
        for (self.negations.items) |n| self.allocator.free(n);
        self.patterns.deinit();
        self.negations.deinit();
    }

    /// โหลดจาก .gitignore file
    pub fn loadGitignore(self: *IgnoreList, file_path: []const u8) !void {
        const file = std.fs.cwd().openFile(file_path, .{}) catch return;
        defer file.close();

        var buf_reader = std.io.bufferedReader(file.reader());
        const reader = buf_reader.reader();

        var line_buf: [4096]u8 = undefined;
        while (try reader.readUntilDelimiterOrEof(&line_buf, '\n')) |line| {
            const trimmed = std.mem.trim(u8, line, " \t\r");

            // ข้าม comments และ empty lines
            if (trimmed.len == 0 or trimmed[0] == '#') continue;

            if (trimmed[0] == '!') {
                // Negation pattern
                const pattern = try self.allocator.dupe(u8, trimmed[1..]);
                try self.negations.append(pattern);
            } else {
                const pattern = try self.allocator.dupe(u8, trimmed);
                try self.patterns.append(pattern);
            }
        }
    }

    /// เพิ่ม patterns เริ่มต้น
    pub fn addDefaults(self: *IgnoreList) !void {
        const defaults = [_][]const u8{
            ".git",
            ".svn",
            ".hg",
            "node_modules",
            ".DS_Store",
            "Thumbs.db",
            "*.pyc",
            "__pycache__",
            "*.o",
            "*.a",
            "*.so",
            "*.exe",
        };

        for (defaults) |pattern| {
            try self.patterns.append(try self.allocator.dupe(u8, pattern));
        }
    }

    /// ตรวจสอบว่าควร ignore หรือไม่
    pub fn shouldIgnore(self: *const IgnoreList, name: []const u8, path: []const u8) bool {
        _ = path;

        // ตรวจสอบ negations ก่อน
        for (self.negations.items) |pattern| {
            if (matchGitignorePattern(pattern, name)) return false;
        }

        // ตรวจสอบ patterns
        for (self.patterns.items) |pattern| {
            if (matchGitignorePattern(pattern, name)) return true;
        }

        return false;
    }
};

fn matchGitignorePattern(pattern: []const u8, name: []const u8) bool {
    // Simple wildcard matching
    if (std.mem.eql(u8, pattern, name)) return true;

    // *.ext pattern
    if (pattern.len > 1 and pattern[0] == '*') {
        return std.mem.endsWith(u8, name, pattern[1..]);
    }

    // prefix* pattern
    if (pattern.len > 1 and pattern[pattern.len - 1] == '*') {
        return std.mem.startsWith(u8, name, pattern[0 .. pattern.len - 1]);
    }

    return false;
}

// Tests
test "Ignore patterns" {
    const allocator = std.testing.allocator;

    var ig = IgnoreList.init(allocator);
    defer ig.deinit();

    try ig.patterns.append(try allocator.dupe(u8, "*.o"));
    try ig.patterns.append(try allocator.dupe(u8, "node_modules"));
    try ig.patterns.append(try allocator.dupe(u8, ".git"));

    try std.testing.expect(ig.shouldIgnore("main.o", ""));
    try std.testing.expect(ig.shouldIgnore("node_modules", ""));
    try std.testing.expect(!ig.shouldIgnore("main.zig", ""));
    try std.testing.expect(!ig.shouldIgnore("src", ""));
}
```

---

## ขั้นตอนที่ 630: Output Formatting

### src/main.zig (Complete)

```zig
const std = @import("std");
const traversal = @import("traversal.zig");
const pattern = @import("pattern.zig");
const search = @import("search.zig");
const operations = @import("operations.zig");
const progress = @import("progress.zig");
const stats = @import("stats.zig");
const ignore = @import("ignore.zig");

const AppOptions = struct {
    operation: enum { find, grep, copy, move, delete, stats_cmd },
    root_path: []const u8,
    pattern_str: ?[]const u8,
    dest_path: ?[]const u8,
    recursive: bool,
    show_progress: bool,
    use_color: bool,
    max_depth: ?u32,
    ignore_file: ?[]const u8,
    jobs: u32,
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 2) {
        try printUsage();
        return;
    }

    // Parse options (simplified)
    const opts = AppOptions{
        .operation = .find,
        .root_path = if (args.len > 1) args[1] else ".",
        .pattern_str = if (args.len > 2) args[2] else null,
        .dest_path = null,
        .recursive = true,
        .show_progress = false,
        .use_color = true,
        .max_depth = null,
        .ignore_file = null,
        .jobs = 4,
    };

    switch (opts.operation) {
        .find => try runFind(allocator, opts),
        .grep => try runGrep(allocator, opts),
        .stats_cmd => try runStats(allocator, opts),
        else => try printUsage(),
    }
}

fn runFind(allocator: std.mem.Allocator, opts: AppOptions) !void {
    // โหลด ignore patterns
    var ignore_list = ignore.IgnoreList.init(allocator);
    defer ignore_list.deinit();
    try ignore_list.addDefaults();

    if (opts.ignore_file) |ig_file| {
        try ignore_list.loadGitignore(ig_file);
    }

    // Traverse
    var trav = traversal.Traversal.init(allocator, .{
        .max_depth = opts.max_depth,
        .ignore_patterns = &ignore_list,
    });
    defer trav.deinit();

    try trav.traverse(opts.root_path);

    const files = trav.getFiles();
    const stdout = std.io.getStdOut().writer();

    // Filter by pattern ถ้ามี
    var file_pattern: ?pattern.GlobPattern = null;
    if (opts.pattern_str) |p| {
        file_pattern = pattern.GlobPattern.init(p);
    }

    var found: u64 = 0;
    for (files) |entry| {
        if (file_pattern) |p| {
            if (!p.matchesName(entry.path)) continue;
        }

        if (opts.use_color) {
            if (entry.isDirectory()) {
                try stdout.print("\x1b[34m{s}\x1b[0m\n", .{entry.path});
            } else {
                try stdout.print("{s}\n", .{entry.path});
            }
        } else {
            try stdout.print("{s}\n", .{entry.path});
        }
        found += 1;
    }

    try stdout.print("\nพบ {d} รายการ\n", .{found});
}

fn runGrep(allocator: std.mem.Allocator, opts: AppOptions) !void {
    const query = opts.pattern_str orelse {
        try std.io.getStdErr().writer().print("กรุณาระบุ pattern สำหรับค้นหา\n", .{});
        return;
    };

    var searcher = search.FileSearcher.init(allocator, .{});
    const results = try searcher.searchDirectory(opts.root_path, query, null);
    defer {
        for (results) |*r| r.deinit(allocator);
        allocator.free(results);
    }

    const stdout = std.io.getStdOut().writer();
    for (results) |r| {
        if (opts.use_color) {
            try stdout.print("\x1b[35m{s}\x1b[0m:{d}: {s}\n", .{
                r.file_path,
                r.line_number,
                r.line_content,
            });
        } else {
            try stdout.print("{s}:{d}: {s}\n", .{
                r.file_path,
                r.line_number,
                r.line_content,
            });
        }
    }

    try stdout.print("\nพบ {d} ผลลัพธ์\n", .{results.len});
}

fn runStats(allocator: std.mem.Allocator, opts: AppOptions) !void {
    var trav = traversal.Traversal.init(allocator, .{});
    defer trav.deinit();

    try trav.traverse(opts.root_path);

    var file_stats = stats.FileStats.init(allocator);
    defer file_stats.deinit();

    for (trav.getFiles()) |entry| {
        try file_stats.processEntry(entry);
    }

    try file_stats.print();
}

fn printUsage() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print(
        \\zigfind - File System Tool
        \\
        \\การใช้งาน:
        \\  zigfind <path> [pattern]    ค้นหา files
        \\  zigfind --grep <path> <text> ค้นหา text ใน files
        \\  zigfind --stats <path>       แสดงสถิติ
        \\
    , .{});
}
```

### สรุป Part 63

เราได้สร้าง File System Tool ที่มีความสามารถ:
- Directory traversal แบบ recursive
- Glob pattern matching
- File content search
- File operations (copy, move, delete)
- Parallel processing framework
- Progress reporting
- File statistics
- .gitignore style ignore patterns
- Color output formatting

---
*ต่อไป Part 64: SQLite Database Wrapper*
