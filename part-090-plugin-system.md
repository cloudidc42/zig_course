# ส่วนที่ 90: Plugin System ใน Zig

## ขั้นตอนที่ 891-900: การสร้าง Plugin System ที่สมบูรณ์

Plugin system ทำให้ application สามารถ extend functionality ได้โดยไม่ต้อง recompile ในส่วนนี้เราจะสร้าง plugin system ที่สมบูรณ์ รองรับ dynamic loading, versioning, sandboxing, hot reloading, และ WASM plugins

---

## Step 891: Plugin Architecture Concepts

### แนวคิดพื้นฐานของ Plugin System

```
Application (Host)
    ├── Plugin Loader
    │   ├── DLL/SO Loader
    │   └── WASM Loader
    ├── Plugin Registry
    │   ├── Discovery
    │   └── Versioning
    ├── Plugin Manager
    │   ├── Lifecycle (load/unload/reload)
    │   └── Communication (API)
    └── Sandbox
        ├── Memory isolation
        └── Permission control
```

```zig
// plugin_arch.zig - โครงสร้างหลักของ Plugin System

const std = @import("std");
const builtin = @import("builtin");

// Plugin metadata
pub const PluginInfo = struct {
    name: []const u8,
    version: Version,
    description: []const u8,
    author: []const u8,
    api_version: Version,
    capabilities: []const Capability,
    
    pub const Capability = enum {
        read_files,
        write_files,
        network_access,
        system_calls,
        ui_access,
    };
};

pub const Version = struct {
    major: u16,
    minor: u16,
    patch: u16,
    
    pub fn parse(s: []const u8) !Version {
        var parts = std.mem.splitScalar(u8, s, '.');
        const major = try std.fmt.parseInt(u16, parts.next() orelse "0", 10);
        const minor = try std.fmt.parseInt(u16, parts.next() orelse "0", 10);
        const patch = try std.fmt.parseInt(u16, parts.next() orelse "0", 10);
        return Version{ .major = major, .minor = minor, .patch = patch };
    }
    
    pub fn isCompatible(self: Version, required: Version) bool {
        // Major version must match, minor/patch can be higher
        return self.major == required.major and
               (self.minor > required.minor or
               (self.minor == required.minor and self.patch >= required.patch));
    }
    
    pub fn format(
        self: Version,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{d}.{d}.{d}", .{ self.major, self.minor, self.patch });
    }
    
    pub fn order(a: Version, b: Version) std.math.Order {
        if (a.major != b.major) return std.math.order(a.major, b.major);
        if (a.minor != b.minor) return std.math.order(a.minor, b.minor);
        return std.math.order(a.patch, b.patch);
    }
};

// Plugin API version ที่ host รองรับ
pub const HOST_API_VERSION = Version{ .major = 1, .minor = 0, .patch = 0 };

// Plugin state
pub const PluginState = enum {
    unloaded,
    loading,
    loaded,
    active,
    error_state,
    unloading,
};

// Plugin handle
pub const Plugin = struct {
    info: PluginInfo,
    state: PluginState,
    handle: ?*anyopaque, // OS-specific handle (HMODULE/void*)
    path: []const u8,
    
    // Function pointers
    init_fn: ?*const fn (*HostApi) callconv(.C) bool,
    deinit_fn: ?*const fn () callconv(.C) void,
    on_event_fn: ?*const fn (event_type: u32, data: ?*anyopaque) callconv(.C) void,
};

// Host API ที่ plugins ใช้ได้
pub const HostApi = extern struct {
    version: Version,
    
    // Memory allocation
    alloc: *const fn (size: usize) callconv(.C) ?*anyopaque,
    free: *const fn (ptr: *anyopaque) callconv(.C) void,
    
    // Logging
    log: *const fn (level: u32, msg: [*:0]const u8) callconv(.C) void,
    
    // Event system
    emit_event: *const fn (event_type: u32, data: ?*anyopaque) callconv(.C) void,
    subscribe_event: *const fn (event_type: u32, handler: *anyopaque) callconv(.C) void,
    
    // File I/O (sandboxed)
    read_file: *const fn (path: [*:0]const u8, buf: [*]u8, len: *usize) callconv(.C) bool,
    write_file: *const fn (path: [*:0]const u8, data: [*]const u8, len: usize) callconv(.C) bool,
};
```

---

## Step 892: Dynamic Library Loading (dlopen)

```zig
// dynamic_loader.zig - การโหลด shared libraries

const std = @import("std");
const builtin = @import("builtin");

pub const DynLib = struct {
    handle: *anyopaque,
    
    // Platform-specific loading
    pub fn open(path: [:0]const u8) !DynLib {
        switch (builtin.os.tag) {
            .linux, .macos, .freebsd => {
                const handle = std.c.dlopen(path.ptr, 0x101) orelse {
                    const err_msg = std.c.dlerror();
                    if (err_msg) |msg| {
                        std.debug.print("dlopen error: {s}\n", .{msg});
                    }
                    return error.LibraryNotFound;
                };
                return DynLib{ .handle = handle };
            },
            .windows => {
                const win = std.os.windows;
                const handle = win.kernel32.LoadLibraryA(path.ptr) orelse
                    return error.LibraryNotFound;
                return DynLib{ .handle = @ptrCast(handle) };
            },
            else => return error.UnsupportedPlatform,
        }
    }
    
    pub fn close(self: *DynLib) void {
        switch (builtin.os.tag) {
            .linux, .macos, .freebsd => {
                _ = std.c.dlclose(self.handle);
            },
            .windows => {
                const win = std.os.windows;
                _ = win.kernel32.FreeLibrary(@ptrCast(self.handle));
            },
            else => {},
        }
    }
    
    pub fn lookup(self: *DynLib, comptime T: type, symbol: [:0]const u8) ?T {
        switch (builtin.os.tag) {
            .linux, .macos, .freebsd => {
                const ptr = std.c.dlsym(self.handle, symbol.ptr) orelse return null;
                return @ptrCast(@alignCast(ptr));
            },
            .windows => {
                const win = std.os.windows;
                const ptr = win.kernel32.GetProcAddress(@ptrCast(self.handle), symbol.ptr) orelse
                    return null;
                return @ptrCast(@alignCast(ptr));
            },
            else => return null,
        }
    }
};

// Plugin Loader
pub const PluginLoader = struct {
    allocator: std.mem.Allocator,
    loaded_libs: std.StringHashMap(DynLib),
    
    // Symbols ที่ plugins ต้อง export
    pub const REQUIRED_SYMBOLS = [_][]const u8{
        "plugin_get_info",
        "plugin_init",
        "plugin_deinit",
    };
    
    pub fn init(allocator: std.mem.Allocator) PluginLoader {
        return PluginLoader{
            .allocator = allocator,
            .loaded_libs = std.StringHashMap(DynLib).init(allocator),
        };
    }
    
    pub fn deinit(self: *PluginLoader) void {
        var it = self.loaded_libs.valueIterator();
        while (it.next()) |lib| {
            lib.close();
        }
        self.loaded_libs.deinit();
    }
    
    pub fn loadPlugin(self: *PluginLoader, path: []const u8) !*Plugin {
        // Null-terminate path
        const path_z = try self.allocator.dupeZ(u8, path);
        defer self.allocator.free(path_z);
        
        var lib = DynLib.open(path_z) catch |err| {
            std.debug.print("Failed to load library '{s}': {}\n", .{ path, err });
            return err;
        };
        
        // Verify required symbols exist
        const get_info = lib.lookup(
            *const fn () callconv(.C) *const PluginInfo,
            "plugin_get_info",
        ) orelse {
            lib.close();
            return error.MissingSymbol;
        };
        
        const init_fn = lib.lookup(
            *const fn (*HostApi) callconv(.C) bool,
            "plugin_init",
        ) orelse {
            lib.close();
            return error.MissingSymbol;
        };
        
        const deinit_fn = lib.lookup(
            *const fn () callconv(.C) void,
            "plugin_deinit",
        );
        
        // Get plugin info
        const info = get_info();
        
        // Check API version compatibility
        if (!info.api_version.isCompatible(HOST_API_VERSION)) {
            std.debug.print("Plugin API version {} incompatible with host {}\n", .{
                info.api_version,
                HOST_API_VERSION,
            });
            lib.close();
            return error.IncompatibleVersion;
        }
        
        // Store loaded library
        try self.loaded_libs.put(path, lib);
        
        const plugin = try self.allocator.create(Plugin);
        plugin.* = Plugin{
            .info = info.*,
            .state = .loaded,
            .handle = lib.handle,
            .path = try self.allocator.dupe(u8, path),
            .init_fn = init_fn,
            .deinit_fn = deinit_fn,
            .on_event_fn = lib.lookup(
                *const fn (u32, ?*anyopaque) callconv(.C) void,
                "plugin_on_event",
            ),
        };
        
        std.debug.print("Loaded plugin: {s} v{}\n", .{ info.name, info.version });
        return plugin;
    }
    
    pub fn unloadPlugin(self: *PluginLoader, plugin: *Plugin) void {
        if (plugin.state == .active) {
            if (plugin.deinit_fn) |deinit| {
                deinit();
            }
        }
        
        if (self.loaded_libs.getPtr(plugin.path)) |lib| {
            lib.close();
            _ = self.loaded_libs.remove(plugin.path);
        }
        
        self.allocator.free(plugin.path);
        plugin.state = .unloaded;
        self.allocator.destroy(plugin);
    }
};

// ตัวอย่าง plugin ที่สร้างเอง (สำหรับทดสอบ)
// จะต้อง compile เป็น shared library
pub const ExamplePlugin = struct {
    pub export fn plugin_get_info() *const PluginInfo {
        const info = comptime PluginInfo{
            .name = "example_plugin",
            .version = Version{ .major = 1, .minor = 0, .patch = 0 },
            .description = "Example plugin for demonstration",
            .author = "Your Name",
            .api_version = HOST_API_VERSION,
            .capabilities = &.{.read_files},
        };
        return &info;
    }
    
    pub export fn plugin_init(host_api: *HostApi) callconv(.C) bool {
        host_api.log(1, "Example plugin initialized!");
        return true;
    }
    
    pub export fn plugin_deinit() callconv(.C) void {
        std.debug.print("Example plugin deinitialized\n", .{});
    }
    
    pub export fn plugin_on_event(event_type: u32, data: ?*anyopaque) callconv(.C) void {
        _ = data;
        std.debug.print("Plugin received event: {d}\n", .{event_type});
    }
};
```

---

## Step 893: Plugin Interface Definition

```zig
// plugin_interface.zig - กำหนด interface ที่ชัดเจนสำหรับ plugins

const std = @import("std");

// C ABI-compatible interface (สำคัญมากสำหรับ dynamic linking)
// ใช้ extern struct เพื่อให้ layout predictable

pub const PluginVTable = extern struct {
    // Lifecycle
    init: *const fn (host: *HostContext) callconv(.C) PluginResult,
    deinit: *const fn () callconv(.C) void,
    
    // Event handling
    on_event: *const fn (event: *const Event) callconv(.C) void,
    
    // Plugin info
    get_info: *const fn () callconv(.C) *const PluginMetadata,
    
    // Custom commands
    execute_command: ?*const fn (cmd: [*:0]const u8, args: *const Args) callconv(.C) PluginResult,
    
    // UI hooks (optional)
    get_ui: ?*const fn () callconv(.C) ?*const UiDefinition,
    
    // Configuration
    configure: ?*const fn (config: *const Config) callconv(.C) PluginResult,
    get_config_schema: ?*const fn () callconv(.C) ?*const ConfigSchema,
};

pub const PluginResult = extern struct {
    success: bool,
    error_code: u32,
    message: ?[*:0]const u8,
    
    pub fn ok() PluginResult {
        return .{ .success = true, .error_code = 0, .message = null };
    }
    
    pub fn err(code: u32, msg: [*:0]const u8) PluginResult {
        return .{ .success = false, .error_code = code, .message = msg };
    }
};

pub const PluginMetadata = extern struct {
    name: [*:0]const u8,
    version_major: u16,
    version_minor: u16,
    version_patch: u16,
    description: [*:0]const u8,
    author: [*:0]const u8,
    api_version_major: u16,
    api_version_minor: u16,
    api_version_patch: u16,
    required_capabilities: u32, // Bitfield
};

pub const Event = extern struct {
    type_id: u32,
    source: u64,
    timestamp: u64,
    data_ptr: ?*anyopaque,
    data_len: usize,
};

pub const Args = extern struct {
    count: u32,
    keys: [*][*:0]const u8,
    values: [*][*:0]const u8,
};

pub const HostContext = extern struct {
    api_version: u64, // packed major/minor/patch
    
    // Memory management (plugin should use these, not malloc/free)
    alloc_fn: *const fn (size: usize, align: usize) callconv(.C) ?*anyopaque,
    free_fn: *const fn (ptr: *anyopaque, size: usize, align: usize) callconv(.C) void,
    
    // Logging
    log_fn: *const fn (level: u32, fmt: [*:0]const u8, ...) callconv(.C) void,
    
    // Event system
    emit_fn: *const fn (event: *const Event) callconv(.C) void,
    
    // File operations (sandboxed)
    file_read: *const fn (path: [*:0]const u8, out_len: *usize) callconv(.C) ?[*]u8,
    file_write: *const fn (path: [*:0]const u8, data: [*]const u8, len: usize) callconv(.C) bool,
    
    // State storage
    state_get: *const fn (key: [*:0]const u8, out_len: *usize) callconv(.C) ?[*]u8,
    state_set: *const fn (key: [*:0]const u8, data: [*]const u8, len: usize) callconv(.C) bool,
};

// Config schema สำหรับ plugin configuration
pub const ConfigSchema = extern struct {
    fields: [*]const Field,
    field_count: u32,
    
    pub const Field = extern struct {
        name: [*:0]const u8,
        type: FieldType,
        description: [*:0]const u8,
        required: bool,
        default_value: ?[*:0]const u8,
        
        pub const FieldType = enum(u32) {
            string,
            integer,
            float,
            boolean,
            enum_type,
        };
    };
};

pub const Config = extern struct {
    data: [*:0]const u8, // JSON string
    len: usize,
};

pub const UiDefinition = extern struct {
    type: UiType,
    data: ?*anyopaque,
    
    pub const UiType = enum(u32) {
        none,
        panel,
        toolbar_item,
        menu_item,
        status_bar,
    };
};

// Macro สำหรับ plugins ที่ต้องการ export VTable
// ใช้แบบนี้ใน plugin:
// pub export fn get_plugin_vtable() *const PluginVTable {
//     return &my_plugin_vtable;
// }
```

---

## Step 894: Plugin Discovery

```zig
// plugin_discovery.zig - การค้นหา plugins

const std = @import("std");

pub const DiscoveryConfig = struct {
    search_paths: []const []const u8,
    file_extensions: []const []const u8,
    recursive: bool,
    max_depth: u32,
};

pub const PluginManifest = struct {
    path: []const u8,
    name: []const u8,
    version: Version,
    checksum: [32]u8, // SHA-256
    file_size: u64,
    last_modified: i64,
};

pub const PluginDiscovery = struct {
    allocator: std.mem.Allocator,
    config: DiscoveryConfig,
    
    pub fn init(allocator: std.mem.Allocator, config: DiscoveryConfig) PluginDiscovery {
        return PluginDiscovery{
            .allocator = allocator,
            .config = config,
        };
    }
    
    // Scan directories สำหรับ plugin files
    pub fn scan(self: *PluginDiscovery) ![]PluginManifest {
        var manifests = std.ArrayList(PluginManifest).init(self.allocator);
        
        for (self.config.search_paths) |search_path| {
            try self.scanDirectory(search_path, 0, &manifests);
        }
        
        return manifests.toOwnedSlice();
    }
    
    fn scanDirectory(
        self: *PluginDiscovery,
        dir_path: []const u8,
        depth: u32,
        manifests: *std.ArrayList(PluginManifest),
    ) !void {
        if (depth > self.config.max_depth) return;
        
        var dir = std.fs.cwd().openDir(dir_path, .{ .iterate = true }) catch |err| {
            if (err == error.FileNotFound) return;
            return err;
        };
        defer dir.close();
        
        var it = dir.iterate();
        while (try it.next()) |entry| {
            const full_path = try std.fs.path.join(
                self.allocator,
                &.{ dir_path, entry.name },
            );
            defer self.allocator.free(full_path);
            
            switch (entry.kind) {
                .file => {
                    // Check extension
                    for (self.config.file_extensions) |ext| {
                        if (std.mem.endsWith(u8, entry.name, ext)) {
                            const manifest = self.loadManifest(full_path) catch continue;
                            try manifests.append(manifest);
                            break;
                        }
                    }
                },
                .directory => {
                    if (self.config.recursive) {
                        try self.scanDirectory(full_path, depth + 1, manifests);
                    }
                },
                else => {},
            }
        }
    }
    
    fn loadManifest(self: *PluginDiscovery, path: []const u8) !PluginManifest {
        // Get file metadata
        const stat = try std.fs.cwd().statFile(path);
        
        // Calculate checksum
        var checksum: [32]u8 = undefined;
        try self.calculateChecksum(path, &checksum);
        
        // Try to load and get plugin info
        // (simplified - in practice would load and call plugin_get_info)
        
        return PluginManifest{
            .path = try self.allocator.dupe(u8, path),
            .name = std.fs.path.stem(path),
            .version = Version{ .major = 1, .minor = 0, .patch = 0 },
            .checksum = checksum,
            .file_size = stat.size,
            .last_modified = @intCast(stat.mtime),
        };
    }
    
    fn calculateChecksum(self: *PluginDiscovery, path: []const u8, out: *[32]u8) !void {
        _ = self;
        const file = try std.fs.cwd().openFile(path, .{});
        defer file.close();
        
        var hasher = std.crypto.hash.sha2.Sha256.init(.{});
        var buf: [4096]u8 = undefined;
        
        while (true) {
            const n = try file.read(&buf);
            if (n == 0) break;
            hasher.update(buf[0..n]);
        }
        
        hasher.final(out);
    }
    
    // Watch สำหรับ changes (hot reload)
    pub fn watchForChanges(self: *PluginDiscovery, callback: *const fn (path: []const u8) void) !void {
        _ = self;
        _ = callback;
        // ใช้ inotify (Linux), FSEvents (macOS), ReadDirectoryChanges (Windows)
        // Simplified implementation
        std.debug.print("File watching not implemented in this demo\n", .{});
    }
};

// Plugin Registry - เก็บข้อมูลทุก plugin ที่รู้จัก
pub const PluginRegistry = struct {
    allocator: std.mem.Allocator,
    entries: std.StringHashMap(RegistryEntry),
    
    pub const RegistryEntry = struct {
        manifest: PluginManifest,
        state: PluginState,
        plugin: ?*Plugin,
        load_time: i64,
        dependencies: [][]const u8,
    };
    
    pub fn init(allocator: std.mem.Allocator) PluginRegistry {
        return PluginRegistry{
            .allocator = allocator,
            .entries = std.StringHashMap(RegistryEntry).init(allocator),
        };
    }
    
    pub fn deinit(self: *PluginRegistry) void {
        var it = self.entries.valueIterator();
        while (it.next()) |entry| {
            self.allocator.free(entry.manifest.path);
        }
        self.entries.deinit();
    }
    
    pub fn register(self: *PluginRegistry, manifest: PluginManifest) !void {
        try self.entries.put(manifest.name, .{
            .manifest = manifest,
            .state = .unloaded,
            .plugin = null,
            .load_time = 0,
            .dependencies = &.{},
        });
    }
    
    pub fn getEntry(self: *PluginRegistry, name: []const u8) ?*RegistryEntry {
        return self.entries.getPtr(name);
    }
};
```

---

## Step 895: Plugin Versioning

```zig
// plugin_versioning.zig - Version management สำหรับ plugins

const std = @import("std");

// Semantic Versioning ที่สมบูรณ์
pub const SemVer = struct {
    major: u32,
    minor: u32,
    patch: u32,
    pre_release: ?[]const u8, // e.g., "alpha.1"
    build_metadata: ?[]const u8, // e.g., "20241201"
    
    pub fn parse(s: []const u8) !SemVer {
        var result = SemVer{
            .major = 0,
            .minor = 0,
            .patch = 0,
            .pre_release = null,
            .build_metadata = null,
        };
        
        // Find + for build metadata
        var main = s;
        if (std.mem.indexOf(u8, s, "+")) |plus_idx| {
            result.build_metadata = s[plus_idx + 1..];
            main = s[0..plus_idx];
        }
        
        // Find - for pre-release
        if (std.mem.indexOf(u8, main, "-")) |dash_idx| {
            result.pre_release = main[dash_idx + 1..];
            main = main[0..dash_idx];
        }
        
        var parts = std.mem.splitScalar(u8, main, '.');
        result.major = try std.fmt.parseInt(u32, parts.next() orelse return error.InvalidVersion, 10);
        result.minor = try std.fmt.parseInt(u32, parts.next() orelse "0", 10);
        result.patch = try std.fmt.parseInt(u32, parts.next() orelse "0", 10);
        
        return result;
    }
    
    // ตรวจสอบ compatibility ตาม semver rules
    pub fn satisfies(self: SemVer, requirement: VersionRequirement) bool {
        return switch (requirement.operator) {
            .exact => self.compareCore(requirement.version) == .eq,
            .compatible => self.major == requirement.version.major and
                self.compareCore(requirement.version) != .lt,
            .at_least => self.compareCore(requirement.version) != .lt,
            .at_most => self.compareCore(requirement.version) != .gt,
            .any => true,
        };
    }
    
    fn compareCore(self: SemVer, other: SemVer) std.math.Order {
        if (self.major != other.major) return std.math.order(self.major, other.major);
        if (self.minor != other.minor) return std.math.order(self.minor, other.minor);
        return std.math.order(self.patch, other.patch);
    }
    
    pub fn format(
        self: SemVer,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("{d}.{d}.{d}", .{ self.major, self.minor, self.patch });
        if (self.pre_release) |pre| try writer.print("-{s}", .{pre});
        if (self.build_metadata) |build| try writer.print("+{s}", .{build});
    }
};

pub const VersionRequirement = struct {
    operator: Operator,
    version: SemVer,
    
    pub const Operator = enum {
        exact,     // =1.0.0
        compatible, // ^1.0.0 (compatible with)
        at_least,   // >=1.0.0
        at_most,    // <=1.0.0
        any,        // *
    };
    
    pub fn parse(s: []const u8) !VersionRequirement {
        if (std.mem.eql(u8, s, "*")) {
            return VersionRequirement{
                .operator = .any,
                .version = SemVer{ .major = 0, .minor = 0, .patch = 0, .pre_release = null, .build_metadata = null },
            };
        }
        
        const op: Operator = if (std.mem.startsWith(u8, s, ">=")) .at_least
            else if (std.mem.startsWith(u8, s, "<=")) .at_most
            else if (std.mem.startsWith(u8, s, "^")) .compatible
            else if (std.mem.startsWith(u8, s, "=")) .exact
            else .compatible;
        
        const version_str = switch (op) {
            .at_least, .at_most => s[2..],
            .compatible, .exact => s[1..],
            .any => s,
        };
        
        return VersionRequirement{
            .operator = op,
            .version = try SemVer.parse(version_str),
        };
    }
};

// Dependency resolver
pub const DependencyResolver = struct {
    allocator: std.mem.Allocator,
    registry: *PluginRegistry,
    
    pub fn init(allocator: std.mem.Allocator, registry: *PluginRegistry) DependencyResolver {
        return .{ .allocator = allocator, .registry = registry };
    }
    
    // Resolve load order ตาม dependencies
    pub fn resolveLoadOrder(self: *DependencyResolver, plugin_name: []const u8) ![][]const u8 {
        var order = std.ArrayList([]const u8).init(self.allocator);
        var visited = std.StringHashMap(bool).init(self.allocator);
        defer visited.deinit();
        
        try self.dfs(plugin_name, &order, &visited);
        return order.toOwnedSlice();
    }
    
    fn dfs(
        self: *DependencyResolver,
        name: []const u8,
        order: *std.ArrayList([]const u8),
        visited: *std.StringHashMap(bool),
    ) !void {
        if (visited.get(name) != null) return; // Already visited
        try visited.put(name, true);
        
        const entry = self.registry.getEntry(name) orelse return;
        
        // Recurse on dependencies
        for (entry.dependencies) |dep| {
            try self.dfs(dep, order, visited);
        }
        
        try order.append(name);
    }
};
```

---

## Step 896: Plugin Sandboxing

```zig
// plugin_sandbox.zig - Sandboxing สำหรับ plugin isolation

const std = @import("std");

pub const Sandbox = struct {
    allocator: std.mem.Allocator,
    permissions: PermissionSet,
    memory_limit: usize,
    allocated: usize,
    allowed_paths: std.ArrayList([]const u8),
    
    pub const PermissionSet = packed struct {
        read_files: bool = false,
        write_files: bool = false,
        network_access: bool = false,
        system_calls: bool = false,
        create_threads: bool = false,
        load_libs: bool = false,
        _padding: u26 = 0,
    };
    
    pub fn init(allocator: std.mem.Allocator, permissions: PermissionSet) Sandbox {
        return Sandbox{
            .allocator = allocator,
            .permissions = permissions,
            .memory_limit = 10 * 1024 * 1024, // 10 MB default
            .allocated = 0,
            .allowed_paths = std.ArrayList([]const u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *Sandbox) void {
        for (self.allowed_paths.items) |p| self.allocator.free(p);
        self.allowed_paths.deinit();
    }
    
    // Sandboxed allocator
    pub fn sandboxedAlloc(self: *Sandbox, size: usize, align_val: usize) ?*anyopaque {
        if (self.allocated + size > self.memory_limit) {
            std.debug.print("Plugin memory limit exceeded!\n", .{});
            return null;
        }
        
        const mem = self.allocator.rawAlloc(size, @intCast(std.math.log2(align_val)), 0) orelse return null;
        self.allocated += size;
        return mem;
    }
    
    pub fn sandboxedFree(self: *Sandbox, ptr: *anyopaque, size: usize, align_val: usize) void {
        self.allocator.rawFree(@as([*]u8, @ptrCast(ptr))[0..size], @intCast(std.math.log2(align_val)), 0);
        if (self.allocated >= size) self.allocated -= size;
    }
    
    // Check file access permission
    pub fn checkFileAccess(self: *Sandbox, path: []const u8, write: bool) bool {
        if (write and !self.permissions.write_files) return false;
        if (!write and !self.permissions.read_files) return false;
        
        // Check if path is in allowed list
        for (self.allowed_paths.items) |allowed| {
            if (std.mem.startsWith(u8, path, allowed)) return true;
        }
        
        return self.allowed_paths.items.len == 0; // ถ้าไม่มีกำหนด allow ทั้งหมด
    }
    
    pub fn allowPath(self: *Sandbox, path: []const u8) !void {
        try self.allowed_paths.append(try self.allocator.dupe(u8, path));
    }
    
    // Create sandboxed HostContext
    pub fn createHostContext(self: *Sandbox) HostContext {
        return HostContext{
            .api_version = 0x0001_0000, // 1.0.0 packed
            .alloc_fn = sandboxAllocFn,
            .free_fn = sandboxFreeFn,
            .log_fn = sandboxLogFn,
            .emit_fn = sandboxEmitFn,
            .file_read = sandboxFileRead,
            .file_write = sandboxFileWrite,
            .state_get = sandboxStateGet,
            .state_set = sandboxStateSet,
        };
    }
    
    // C-compatible callbacks (simplified)
    fn sandboxAllocFn(size: usize, align_val: usize) callconv(.C) ?*anyopaque {
        _ = align_val;
        // ต้องผ่าน sandbox reference จริงๆ - ใช้ global state หรือ thread-local
        const mem = std.heap.page_allocator.alloc(u8, size) catch return null;
        return mem.ptr;
    }
    
    fn sandboxFreeFn(ptr: *anyopaque, size: usize, align_val: usize) callconv(.C) void {
        _ = size;
        _ = align_val;
        _ = ptr;
        // Simplified
    }
    
    fn sandboxLogFn(level: u32, msg: [*:0]const u8) callconv(.C) void {
        const level_name = switch (level) {
            0 => "DEBUG",
            1 => "INFO",
            2 => "WARN",
            3 => "ERROR",
            else => "UNKNOWN",
        };
        std.debug.print("[Plugin/{s}] {s}\n", .{ level_name, msg });
    }
    
    fn sandboxEmitFn(event: *const Event) callconv(.C) void {
        std.debug.print("Plugin event: type={d}\n", .{event.type_id});
    }
    
    fn sandboxFileRead(path: [*:0]const u8, out_len: *usize) callconv(.C) ?[*]u8 {
        const path_slice = std.mem.span(path);
        const file = std.fs.cwd().openFile(path_slice, .{}) catch return null;
        defer file.close();
        
        const stat = file.stat() catch return null;
        const size = stat.size;
        
        const buf = std.heap.page_allocator.alloc(u8, size) catch return null;
        const n = file.readAll(buf) catch {
            std.heap.page_allocator.free(buf);
            return null;
        };
        
        out_len.* = n;
        return buf.ptr;
    }
    
    fn sandboxFileWrite(path: [*:0]const u8, data: [*]const u8, len: usize) callconv(.C) bool {
        const path_slice = std.mem.span(path);
        const file = std.fs.cwd().createFile(path_slice, .{}) catch return false;
        defer file.close();
        file.writeAll(data[0..len]) catch return false;
        return true;
    }
    
    fn sandboxStateGet(key: [*:0]const u8, out_len: *usize) callconv(.C) ?[*]u8 {
        _ = key;
        out_len.* = 0;
        return null;
    }
    
    fn sandboxStateSet(key: [*:0]const u8, data: [*]const u8, len: usize) callconv(.C) bool {
        _ = key;
        _ = data;
        _ = len;
        return true;
    }
};
```

---

## Step 897: Hot Reloading

```zig
// hot_reload.zig - Hot reloading ของ plugins

const std = @import("std");

pub const HotReloader = struct {
    allocator: std.mem.Allocator,
    manager: *PluginManager,
    watch_thread: ?std.Thread,
    running: std.atomic.Value(bool),
    reload_queue: std.ArrayList([]const u8),
    mutex: std.Thread.Mutex,
    
    pub fn init(allocator: std.mem.Allocator, manager: *PluginManager) HotReloader {
        return HotReloader{
            .allocator = allocator,
            .manager = manager,
            .watch_thread = null,
            .running = std.atomic.Value(bool).init(false),
            .reload_queue = std.ArrayList([]const u8).init(allocator),
            .mutex = .{},
        };
    }
    
    pub fn deinit(self: *HotReloader) void {
        self.stop();
        for (self.reload_queue.items) |path| self.allocator.free(path);
        self.reload_queue.deinit();
    }
    
    pub fn start(self: *HotReloader) !void {
        self.running.store(true, .release);
        self.watch_thread = try std.Thread.spawn(.{}, watchLoop, .{self});
    }
    
    pub fn stop(self: *HotReloader) void {
        self.running.store(false, .release);
        if (self.watch_thread) |thread| {
            thread.join();
            self.watch_thread = null;
        }
    }
    
    fn watchLoop(self: *HotReloader) void {
        // Watch สำหรับ file changes
        while (self.running.load(.acquire)) {
            self.checkForChanges() catch {};
            std.time.sleep(std.time.ns_per_s); // Check every second
        }
    }
    
    fn checkForChanges(self: *HotReloader) !void {
        // ในทางปฏิบัติใช้ inotify/FSEvents/ReadDirectoryChanges
        // Simplified: check file modification times
        
        var it = self.manager.registry.entries.iterator();
        while (it.next()) |kv| {
            const entry = kv.value_ptr;
            const stat = std.fs.cwd().statFile(entry.manifest.path) catch continue;
            
            const current_mtime: i64 = @intCast(stat.mtime);
            if (current_mtime > entry.manifest.last_modified) {
                // File changed!
                self.mutex.lock();
                defer self.mutex.unlock();
                
                const path = try self.allocator.dupe(u8, entry.manifest.path);
                try self.reload_queue.append(path);
            }
        }
    }
    
    // Process pending reloads (call from main thread)
    pub fn processReloads(self: *HotReloader) !void {
        self.mutex.lock();
        const queue = try self.reload_queue.toOwnedSlice();
        self.mutex.unlock();
        defer self.allocator.free(queue);
        
        for (queue) |path| {
            defer self.allocator.free(path);
            std.debug.print("Hot reloading: {s}\n", .{path});
            try self.manager.reloadPlugin(path);
        }
    }
};

pub const PluginManager = struct {
    allocator: std.mem.Allocator,
    loader: PluginLoader,
    registry: PluginRegistry,
    hot_reloader: ?HotReloader,
    loaded_plugins: std.StringHashMap(*Plugin),
    
    pub fn init(allocator: std.mem.Allocator) !PluginManager {
        return PluginManager{
            .allocator = allocator,
            .loader = PluginLoader.init(allocator),
            .registry = PluginRegistry.init(allocator),
            .hot_reloader = null,
            .loaded_plugins = std.StringHashMap(*Plugin).init(allocator),
        };
    }
    
    pub fn deinit(self: *PluginManager) void {
        if (self.hot_reloader) |*hr| hr.deinit();
        var it = self.loaded_plugins.valueIterator();
        while (it.next()) |plugin| {
            self.loader.unloadPlugin(plugin.*);
        }
        self.loaded_plugins.deinit();
        self.registry.deinit();
        self.loader.deinit();
    }
    
    pub fn loadPlugin(self: *PluginManager, path: []const u8) !void {
        const plugin = try self.loader.loadPlugin(path);
        
        // Initialize plugin
        if (plugin.init_fn) |init| {
            var sandbox = Sandbox.init(self.allocator, .{ .read_files = true });
            defer sandbox.deinit();
            
            var host_ctx = sandbox.createHostContext();
            if (!init(&host_ctx)) {
                self.loader.unloadPlugin(plugin);
                return error.PluginInitFailed;
            }
        }
        
        plugin.state = .active;
        try self.loaded_plugins.put(plugin.info.name, plugin);
    }
    
    pub fn unloadPlugin(self: *PluginManager, name: []const u8) void {
        if (self.loaded_plugins.fetchRemove(name)) |kv| {
            self.loader.unloadPlugin(kv.value);
        }
    }
    
    pub fn reloadPlugin(self: *PluginManager, path: []const u8) !void {
        // Find plugin by path
        var plugin_name: ?[]const u8 = null;
        var it = self.loaded_plugins.iterator();
        while (it.next()) |kv| {
            if (std.mem.eql(u8, kv.value_ptr.*.path, path)) {
                plugin_name = kv.key_ptr.*;
                break;
            }
        }
        
        if (plugin_name) |name| {
            std.debug.print("Reloading plugin: {s}\n", .{name});
            self.unloadPlugin(name);
        }
        
        try self.loadPlugin(path);
        std.debug.print("Plugin reloaded successfully\n", .{});
    }
    
    pub fn enableHotReload(self: *PluginManager) !void {
        self.hot_reloader = HotReloader.init(self.allocator, self);
        try self.hot_reloader.?.start();
    }
};
```

---

## Step 898: WASM-Based Plugins

```zig
// wasm_plugins.zig - WebAssembly-based plugin system

const std = @import("std");

// WASM binary format constants
pub const WASM_MAGIC: u32 = 0x6d736100; // "\0asm"
pub const WASM_VERSION: u32 = 0x1;

pub const WasmPlugin = struct {
    allocator: std.mem.Allocator,
    module_bytes: []const u8,
    memory: []u8,
    exports: std.StringHashMap(WasmExport),
    
    pub const WasmExport = union(enum) {
        function: WasmFunc,
        memory: WasmMemory,
        table: WasmTable,
        global: WasmGlobal,
    };
    
    pub const WasmFunc = struct {
        type_idx: u32,
        body_offset: u32,
    };
    
    pub const WasmMemory = struct {
        initial: u32,
        maximum: ?u32,
    };
    
    pub const WasmTable = struct {
        element_type: u8,
        initial: u32,
    };
    
    pub const WasmGlobal = struct {
        value_type: u8,
        is_mutable: bool,
        value: i64,
    };
    
    pub fn load(allocator: std.mem.Allocator, wasm_bytes: []const u8) !WasmPlugin {
        // Validate WASM magic number and version
        if (wasm_bytes.len < 8) return error.InvalidWasm;
        
        const magic = std.mem.readInt(u32, wasm_bytes[0..4], .little);
        const version = std.mem.readInt(u32, wasm_bytes[4..8], .big);
        
        if (magic != WASM_MAGIC) return error.InvalidWasm;
        if (version != WASM_VERSION) return error.UnsupportedWasmVersion;
        
        var plugin = WasmPlugin{
            .allocator = allocator,
            .module_bytes = wasm_bytes,
            .memory = &.{},
            .exports = std.StringHashMap(WasmExport).init(allocator),
        };
        
        // Parse sections
        var offset: usize = 8;
        while (offset < wasm_bytes.len) {
            const section_id = wasm_bytes[offset];
            offset += 1;
            
            const section_size = try readUleb128(wasm_bytes, &offset);
            const section_end = offset + section_size;
            
            switch (section_id) {
                1 => try plugin.parseTypeSection(wasm_bytes[offset..section_end]),
                2 => try plugin.parseImportSection(wasm_bytes[offset..section_end]),
                3 => {}, // Function section
                5 => try plugin.parseMemorySection(wasm_bytes[offset..section_end]),
                7 => try plugin.parseExportSection(wasm_bytes[offset..section_end]),
                10 => {}, // Code section
                else => {}, // Unknown section, skip
            }
            
            offset = section_end;
        }
        
        return plugin;
    }
    
    pub fn deinit(self: *WasmPlugin) void {
        self.exports.deinit();
        if (self.memory.len > 0) self.allocator.free(self.memory);
    }
    
    fn parseTypeSection(_: *WasmPlugin, _: []const u8) !void {
        // Parse function types
    }
    
    fn parseImportSection(_: *WasmPlugin, _: []const u8) !void {
        // Parse imports
    }
    
    fn parseMemorySection(self: *WasmPlugin, data: []const u8) !void {
        var offset: usize = 0;
        const count = try readUleb128(data, &offset);
        
        if (count > 0) {
            const flags = data[offset];
            offset += 1;
            const initial = try readUleb128(data, &offset);
            _ = flags;
            
            // Allocate WASM memory (64KB pages)
            const mem_size = initial * 65536;
            self.memory = try self.allocator.alloc(u8, mem_size);
            @memset(self.memory, 0);
        }
    }
    
    fn parseExportSection(self: *WasmPlugin, data: []const u8) !void {
        var offset: usize = 0;
        const count = try readUleb128(data, &offset);
        
        var i: u32 = 0;
        while (i < count) : (i += 1) {
            // Read export name
            const name_len = try readUleb128(data, &offset);
            const name = data[offset..offset + name_len];
            offset += name_len;
            
            // Export type and index
            const export_type = data[offset];
            offset += 1;
            const export_idx = try readUleb128(data, &offset);
            
            const export_val: WasmExport = switch (export_type) {
                0x00 => WasmExport{ .function = .{ .type_idx = export_idx, .body_offset = 0 } },
                0x02 => WasmExport{ .memory = .{ .initial = 0, .maximum = null } },
                else => continue,
            };
            
            try self.exports.put(name, export_val);
        }
    }
    
    // Call a WASM function (simplified interpreter)
    pub fn callFunction(self: *WasmPlugin, name: []const u8, args: []const i64) ![]i64 {
        _ = self;
        _ = name;
        _ = args;
        // Full WASM interpreter implementation would be here
        // For production, use a library like wasmer or wasmtime via C API
        return &.{};
    }
};

fn readUleb128(data: []const u8, offset: *usize) !u32 {
    var result: u32 = 0;
    var shift: u5 = 0;
    
    while (offset.* < data.len) {
        const byte = data[offset.*];
        offset.* += 1;
        result |= @as(u32, byte & 0x7F) << shift;
        if (byte & 0x80 == 0) break;
        shift += 7;
    }
    
    return result;
}
```

---

## Step 899: Plugin Configuration

```zig
// plugin_config.zig - Plugin configuration management

const std = @import("std");

pub const PluginConfig = struct {
    allocator: std.mem.Allocator,
    data: std.StringHashMap(ConfigValue),
    
    pub const ConfigValue = union(enum) {
        string: []const u8,
        integer: i64,
        float: f64,
        boolean: bool,
        array: []ConfigValue,
        object: std.StringHashMap(ConfigValue),
        null_val: void,
    };
    
    pub fn init(allocator: std.mem.Allocator) PluginConfig {
        return PluginConfig{
            .allocator = allocator,
            .data = std.StringHashMap(ConfigValue).init(allocator),
        };
    }
    
    pub fn deinit(self: *PluginConfig) void {
        self.data.deinit();
    }
    
    // Load config จาก JSON file
    pub fn loadFromFile(self: *PluginConfig, path: []const u8) !void {
        const file = try std.fs.cwd().openFile(path, .{});
        defer file.close();
        
        const content = try file.readToEndAlloc(self.allocator, 1024 * 1024);
        defer self.allocator.free(content);
        
        try self.parseJson(content);
    }
    
    fn parseJson(self: *PluginConfig, json: []const u8) !void {
        // Simplified JSON parser
        const parsed = try std.json.parseFromSlice(
            std.json.Value,
            self.allocator,
            json,
            .{},
        );
        defer parsed.deinit();
        
        if (parsed.value != .object) return error.InvalidConfig;
        
        var it = parsed.value.object.iterator();
        while (it.next()) |kv| {
            const val = try self.jsonToConfig(kv.value_ptr.*);
            try self.data.put(kv.key_ptr.*, val);
        }
    }
    
    fn jsonToConfig(self: *PluginConfig, json_val: std.json.Value) !ConfigValue {
        return switch (json_val) {
            .null => ConfigValue{ .null_val = {} },
            .bool => |b| ConfigValue{ .boolean = b },
            .integer => |i| ConfigValue{ .integer = i },
            .float => |f| ConfigValue{ .float = f },
            .string => |s| ConfigValue{ .string = try self.allocator.dupe(u8, s) },
            .array => |arr| blk: {
                const items = try self.allocator.alloc(ConfigValue, arr.items.len);
                for (arr.items, 0..) |item, i| {
                    items[i] = try self.jsonToConfig(item);
                }
                break :blk ConfigValue{ .array = items };
            },
            .object => |obj| blk: {
                var result = std.StringHashMap(ConfigValue).init(self.allocator);
                var it = obj.iterator();
                while (it.next()) |kv| {
                    const val = try self.jsonToConfig(kv.value_ptr.*);
                    try result.put(kv.key_ptr.*, val);
                }
                break :blk ConfigValue{ .object = result };
            },
            .number_string => |s| blk: {
                if (std.fmt.parseInt(i64, s, 10)) |i| {
                    break :blk ConfigValue{ .integer = i };
                } else |_| {
                    if (std.fmt.parseFloat(f64, s)) |f| {
                        break :blk ConfigValue{ .float = f };
                    } else |_| {
                        break :blk ConfigValue{ .string = try self.allocator.dupe(u8, s) };
                    }
                }
            },
        };
    }
    
    pub fn getString(self: *PluginConfig, key: []const u8) ?[]const u8 {
        const val = self.data.get(key) orelse return null;
        return switch (val) {
            .string => |s| s,
            else => null,
        };
    }
    
    pub fn getInt(self: *PluginConfig, key: []const u8) ?i64 {
        const val = self.data.get(key) orelse return null;
        return switch (val) {
            .integer => |i| i,
            .float => |f| @intFromFloat(f),
            else => null,
        };
    }
    
    pub fn getBool(self: *PluginConfig, key: []const u8) ?bool {
        const val = self.data.get(key) orelse return null;
        return switch (val) {
            .boolean => |b| b,
            else => null,
        };
    }
};

// Config validator
pub const ConfigValidator = struct {
    schema: *const ConfigSchema,
    
    pub fn validate(self: *ConfigValidator, config: *const PluginConfig) !void {
        const fields = self.schema.fields[0..self.schema.field_count];
        
        for (fields) |field| {
            const field_name = std.mem.span(field.name);
            
            if (field.required and !config.data.contains(field_name)) {
                std.debug.print("Missing required config field: {s}\n", .{field_name});
                return error.MissingRequiredField;
            }
        }
    }
};
```

---

## Step 900: Complete Plugin System

```zig
// complete_plugin_system.zig - Plugin system ที่สมบูรณ์

const std = @import("std");

pub const PluginSystem = struct {
    allocator: std.mem.Allocator,
    manager: PluginManager,
    discovery: PluginDiscovery,
    event_bus: EventBus,
    config_dir: []const u8,
    
    pub const EventBus = struct {
        handlers: std.AutoHashMap(u32, std.ArrayList(*HandlerFn)),
        allocator: std.mem.Allocator,
        
        pub const HandlerFn = fn (event: *const Event) void;
        
        pub fn init(allocator: std.mem.Allocator) EventBus {
            return .{
                .handlers = std.AutoHashMap(u32, std.ArrayList(*HandlerFn)).init(allocator),
                .allocator = allocator,
            };
        }
        
        pub fn deinit(self: *EventBus) void {
            var it = self.handlers.valueIterator();
            while (it.next()) |list| list.deinit();
            self.handlers.deinit();
        }
        
        pub fn subscribe(self: *EventBus, event_type: u32, handler: *HandlerFn) !void {
            const result = try self.handlers.getOrPut(event_type);
            if (!result.found_existing) {
                result.value_ptr.* = std.ArrayList(*HandlerFn).init(self.allocator);
            }
            try result.value_ptr.append(handler);
        }
        
        pub fn emit(self: *EventBus, event: *const Event) void {
            if (self.handlers.get(event.type_id)) |handlers| {
                for (handlers.items) |handler| {
                    handler(event);
                }
            }
        }
    };
    
    pub fn init(allocator: std.mem.Allocator, plugin_dir: []const u8) !PluginSystem {
        const manager = try PluginManager.init(allocator);
        
        const extensions = switch (@import("builtin").os.tag) {
            .linux => &[_][]const u8{".so"},
            .macos => &[_][]const u8{".dylib"},
            .windows => &[_][]const u8{".dll"},
            else => &[_][]const u8{".so"},
        };
        
        const discovery = PluginDiscovery.init(allocator, .{
            .search_paths = &.{plugin_dir},
            .file_extensions = extensions,
            .recursive = true,
            .max_depth = 3,
        });
        
        return PluginSystem{
            .allocator = allocator,
            .manager = manager,
            .discovery = discovery,
            .event_bus = EventBus.init(allocator),
            .config_dir = try allocator.dupe(u8, plugin_dir),
        };
    }
    
    pub fn deinit(self: *PluginSystem) void {
        self.event_bus.deinit();
        self.manager.deinit();
        self.allocator.free(self.config_dir);
    }
    
    // Discover และ load plugins ทั้งหมด
    pub fn loadAllPlugins(self: *PluginSystem) !void {
        const manifests = self.discovery.scan() catch |err| {
            std.debug.print("Discovery error: {}\n", .{err});
            return;
        };
        defer {
            for (manifests) |m| self.allocator.free(m.path);
            self.allocator.free(manifests);
        }
        
        std.debug.print("Found {} plugins\n", .{manifests.len});
        
        for (manifests) |manifest| {
            self.manager.loadPlugin(manifest.path) catch |err| {
                std.debug.print("Failed to load {s}: {}\n", .{ manifest.path, err });
            };
        }
    }
    
    // Send event to all plugins
    pub fn broadcastEvent(self: *PluginSystem, event_type: u32, data: ?*anyopaque) void {
        const event = Event{
            .type_id = event_type,
            .source = 0,
            .timestamp = @intCast(std.time.milliTimestamp()),
            .data_ptr = data,
            .data_len = 0,
        };
        
        // Broadcast to host handlers
        self.event_bus.emit(&event);
        
        // Broadcast to plugins
        var it = self.manager.loaded_plugins.valueIterator();
        while (it.next()) |plugin| {
            if (plugin.*.on_event_fn) |handler| {
                handler(event_type, data);
            }
        }
    }
    
    // Enable hot reloading
    pub fn enableHotReload(self: *PluginSystem) !void {
        try self.manager.enableHotReload();
        std.debug.print("Hot reload enabled\n", .{});
    }
    
    // Get plugin by name
    pub fn getPlugin(self: *PluginSystem, name: []const u8) ?*Plugin {
        return self.manager.loaded_plugins.get(name);
    }
    
    // List loaded plugins
    pub fn listPlugins(self: *PluginSystem, writer: anytype) !void {
        try writer.writeAll("Loaded plugins:\n");
        var it = self.manager.loaded_plugins.valueIterator();
        while (it.next()) |plugin| {
            try writer.print("  - {s} v{} ({s})\n", .{
                plugin.*.info.name,
                plugin.*.info.version,
                @tagName(plugin.*.state),
            });
        }
    }
};

// ตัวอย่างการใช้งาน Plugin System
pub fn pluginSystemDemo(allocator: std.mem.Allocator) !void {
    std.debug.print("=== Plugin System Demo ===\n\n", .{});
    
    // สร้าง plugin system
    var ps = try PluginSystem.init(allocator, "./plugins");
    defer ps.deinit();
    
    // Load all plugins (จะสร้าง directory ถ้าไม่มี)
    std.fs.cwd().makeDir("./plugins") catch {};
    
    std.debug.print("Loading plugins from ./plugins/\n", .{});
    try ps.loadAllPlugins();
    
    // List loaded plugins
    const stdout = std.io.getStdOut().writer();
    try ps.listPlugins(stdout);
    
    // Test WASM plugin loading
    const test_wasm = [_]u8{
        0x00, 0x61, 0x73, 0x6d, // magic: \0asm
        0x01, 0x00, 0x00, 0x00, // version: 1
        // ... minimal WASM module
    };
    
    var wasm_plugin = WasmPlugin.load(allocator, &test_wasm) catch |err| {
        std.debug.print("WASM load error (expected with minimal module): {}\n", .{err});
        return;
    };
    defer wasm_plugin.deinit();
    
    std.debug.print("WASM plugin loaded, exports: {d}\n", .{wasm_plugin.exports.count()});
    
    // Broadcast an event
    std.debug.print("\nBroadcasting test event...\n", .{});
    ps.broadcastEvent(42, null);
    
    std.debug.print("\nPlugin system demo complete!\n", .{});
}

test "plugin system basic functionality" {
    const allocator = std.testing.allocator;
    
    // สร้าง plugin system
    std.fs.cwd().makeDir("./test_plugins") catch {};
    defer std.fs.cwd().deleteDir("./test_plugins") catch {};
    
    var ps = try PluginSystem.init(allocator, "./test_plugins");
    defer ps.deinit();
    
    // Load (ไม่มี plugins ใน test directory)
    try ps.loadAllPlugins();
    
    // Verify state
    try std.testing.expectEqual(@as(usize, 0), ps.manager.loaded_plugins.count());
}

test "version parsing and compatibility" {
    // Test SemVer parsing
    const v1 = try SemVer.parse("1.2.3");
    try std.testing.expectEqual(@as(u32, 1), v1.major);
    try std.testing.expectEqual(@as(u32, 2), v1.minor);
    try std.testing.expectEqual(@as(u32, 3), v1.patch);
    
    // Test compatibility
    const req = try VersionRequirement.parse("^1.0.0");
    
    const v100 = try SemVer.parse("1.0.0");
    const v110 = try SemVer.parse("1.1.0");
    const v200 = try SemVer.parse("2.0.0");
    
    try std.testing.expect(v100.satisfies(req));
    try std.testing.expect(v110.satisfies(req));
    try std.testing.expect(!v200.satisfies(req));
}

test "plugin config loading" {
    const allocator = std.testing.allocator;
    
    var config = PluginConfig.init(allocator);
    defer config.deinit();
    
    // Write test config
    const test_json =
        \\{"name": "test", "count": 42, "enabled": true}
    ;
    
    const tmp_file = "/tmp/test_plugin_config.json";
    {
        const f = try std.fs.cwd().createFile(tmp_file, .{});
        defer f.close();
        try f.writeAll(test_json);
    }
    defer std.fs.cwd().deleteFile(tmp_file) catch {};
    
    try config.loadFromFile(tmp_file);
    
    try std.testing.expectEqualStrings("test", config.getString("name").?);
    try std.testing.expectEqual(@as(i64, 42), config.getInt("count").?);
    try std.testing.expect(config.getBool("enabled").?);
}
```

---

## สรุป Part 90

ในส่วนนี้เราได้เรียนรู้:

1. **Step 891**: Plugin architecture - โครงสร้างหลักและ HostApi design
2. **Step 892**: Dynamic library loading - dlopen/LoadLibrary สำหรับ Linux/Windows/macOS
3. **Step 893**: Plugin interface - C ABI-compatible interfaces ด้วย extern struct
4. **Step 894**: Plugin discovery - การค้นหาและ inventory plugins
5. **Step 895**: Plugin versioning - Semantic versioning และ dependency resolution
6. **Step 896**: Plugin sandboxing - Memory limits และ permission control
7. **Step 897**: Hot reloading - File watching และ reload ระหว่าง runtime
8. **Step 898**: WASM plugins - WebAssembly binary parsing concepts
9. **Step 899**: Plugin configuration - JSON config loading และ validation
10. **Step 900**: Complete plugin system - ระบบ plugin ที่สมบูรณ์

Plugin system ที่ดีต้องคำนึงถึง:
- **Safety**: Sandbox plugins เพื่อป้องกัน malicious code
- **Versioning**: ดูแล API compatibility ระหว่าง versions
- **Performance**: Minimize overhead จาก plugin calls
- **Developer experience**: API ที่ชัดเจนและ documentation ที่ดี
- **Hot reloading**: รองรับ development workflow

ด้วย Zig เราสามารถสร้าง plugin system ที่ทั้ง safe และ performant ได้ เนื่องจาก Zig มี built-in support สำหรับ C ABI, compile-time programming, และ explicit memory management
