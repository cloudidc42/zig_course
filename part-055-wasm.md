# Part 55: WebAssembly ด้วย Zig
## Steps 541-550

WebAssembly (WASM) เป็น binary format ที่ให้โค้ดรันใน web browser หรือ server-side runtime ด้วยความเร็วใกล้เคียง native Zig สนับสนุน WASM ได้ดีเยี่ยม ทำให้สามารถนำ performance-critical code ไปรันใน browser หรือ WASI environment ได้

---

## Step 541: WebAssembly คืออะไร?

### WASM Overview

```
WebAssembly คือ:
- Binary instruction format สำหรับ stack-based virtual machine
- Designed เป็น portable compilation target
- รันใน web browsers ทุกตัว (Chrome, Firefox, Safari, Edge)
- ใช้ได้กับ server-side ผ่าน WASI (WebAssembly System Interface)
- Performance ใกล้เคียง native code

เปรียบเทียบกับ JavaScript:
┌─────────────────────┬──────────────┬──────────────┐
│                     │ JavaScript   │ WebAssembly  │
├─────────────────────┼──────────────┼──────────────┤
│ Type safety         │ Dynamic      │ Static       │
│ Performance         │ Interpreted  │ Near-native  │
│ Memory              │ GC managed   │ Manual       │
│ Binary size         │ Text source  │ Compact bin  │
│ Startup time        │ Fast         │ Fast         │
│ DOM access          │ Direct       │ Via JS       │
└─────────────────────┴──────────────┴──────────────┘
```

### WASM Use Cases

```zig
// WASM เหมาะสำหรับ:
// 1. Computation-heavy tasks (image processing, crypto, ML)
// 2. Port existing native libraries to web
// 3. Games (physics, rendering)
// 4. Server-side scripting (WASI)
// 5. Plugin systems (sandboxed)
// 6. Edge computing

// ตัวอย่าง: ทำไม Zig กับ WASM จึงเข้ากัน
// - Zig ไม่มี GC → ไม่มี garbage collector overhead ใน WASM
// - Zig ควบคุม memory โดยตรง → efficient WASM memory usage
// - Zig มี comptime → optimization เวลา compile
// - Zig รองรับ freestanding → ไม่ต้องการ OS syscalls

const std = @import("std");

pub fn main() void {
    std.debug.print("Learning WebAssembly with Zig!\n", .{});
}
```

### WASM Architecture

```
WASM Binary Format:
┌──────────────────┐
│ Magic + Version  │ 0x00 0x61 0x73 0x6D + version
├──────────────────┤
│ Type Section     │ Function signatures
├──────────────────┤
│ Import Section   │ Imported functions/memory
├──────────────────┤
│ Function Section │ Function type indices
├──────────────────┤
│ Memory Section   │ Memory specifications
├──────────────────┤
│ Export Section   │ Exported functions/memory
├──────────────────┤
│ Code Section     │ Function bodies
└──────────────────┘
```

---

## Step 542: Compile Zig to WASM

### Basic Compilation

```bash
# สร้าง WASM binary
zig build-exe main.zig \
    -target wasm32-freestanding \
    -O ReleaseSmall \
    -fno-entry

# หรือผ่าน build system
zig build -Dtarget=wasm32-freestanding

# ดู WASM binary (ต้องติดตั้ง wabt)
wasm-objdump -d main.wasm
wasm2wat main.wasm  # แปลงเป็น text format
```

### build.zig สำหรับ WASM

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // WASM สำหรับ browser (freestanding)
    const wasm_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });
    
    const wasm = b.addExecutable(.{
        .name = "app",
        .root_source_file = b.path("src/main.zig"),
        .target = wasm_target,
        .optimize = .ReleaseSmall,
    });
    
    // ไม่ต้องการ entry point (export functions แทน)
    wasm.entry = .disabled;
    
    // Export dynamic memory ให้ JS เข้าถึงได้
    wasm.export_memory = true;
    
    b.installArtifact(wasm);
    
    // WASI สำหรับ server-side
    const wasi_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .wasi,
    });
    
    const wasi_exe = b.addExecutable(.{
        .name = "server-app",
        .root_source_file = b.path("src/wasi_main.zig"),
        .target = wasi_target,
        .optimize = .ReleaseSafe,
    });
    
    b.installArtifact(wasi_exe);
}
```

### Minimal WASM Module

```zig
// src/main.zig - Minimal WASM module
// ไม่ต้อง import std สำหรับ freestanding WASM

// Export function ให้ JavaScript เรียกได้
export fn add(a: i32, b: i32) i32 {
    return a + b;
}

export fn multiply(a: f64, b: f64) f64 {
    return a * b;
}

export fn fibonacci(n: u32) u32 {
    if (n <= 1) return n;
    
    var a: u32 = 0;
    var b: u32 = 1;
    var i: u32 = 2;
    
    while (i <= n) : (i += 1) {
        const c = a + b;
        a = b;
        b = c;
    }
    
    return b;
}

// Panic handler สำหรับ freestanding
pub fn panic(message: []const u8, _: ?*@import("std").builtin.StackTrace, _: ?usize) noreturn {
    _ = message;
    @trap();
}
```

---

## Step 543: Calling Zig Functions from JavaScript

### JavaScript Integration

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Zig WASM Demo</title>
</head>
<body>
    <h1>Zig WebAssembly Demo</h1>
    <div id="output"></div>
    
    <script>
    async function loadWasm() {
        // Load และ instantiate WASM module
        const response = await fetch('app.wasm');
        const bytes = await response.arrayBuffer();
        
        const { instance } = await WebAssembly.instantiate(bytes, {
            env: {
                // Imported functions จาก JS
                console_log: (ptr, len) => {
                    const memory = instance.exports.memory;
                    const bytes = new Uint8Array(memory.buffer, ptr, len);
                    const text = new TextDecoder().decode(bytes);
                    console.log(text);
                    document.getElementById('output').textContent += text + '\n';
                }
            }
        });
        
        const exports = instance.exports;
        
        // เรียก Zig functions
        console.log('add(3, 4) =', exports.add(3, 4));
        console.log('multiply(2.5, 4.0) =', exports.multiply(2.5, 4.0));
        console.log('fibonacci(10) =', exports.fibonacci(10));
        
        // ทำงานกับ arrays ผ่าน shared memory
        const memory = new Uint8Array(exports.memory.buffer);
        
        // เรียก allocate function เพื่อขอ memory ใน WASM
        const bufPtr = exports.allocate(100);
        
        // เขียนข้อมูลลงใน WASM memory
        const encoder = new TextEncoder();
        const data = encoder.encode("Hello from JavaScript!");
        memory.set(data, bufPtr);
        
        // เรียก function ที่ process data
        exports.processString(bufPtr, data.length);
        
        // Free memory
        exports.deallocate(bufPtr, 100);
    }
    
    loadWasm().catch(console.error);
    </script>
</body>
</html>
```

### Zig ที่ Export สำหรับ JS

```zig
// src/main.zig - WASM exports สำหรับ JavaScript
const std = @import("std");

// Import function จาก JavaScript
extern fn console_log(ptr: [*]const u8, len: usize) void;

// Global allocator สำหรับ WASM
var wasm_allocator = std.heap.WasmAllocator;

// Log ไปยัง JavaScript console
fn jsLog(comptime fmt: []const u8, args: anytype) void {
    var buf: [1024]u8 = undefined;
    const msg = std.fmt.bufPrint(&buf, fmt, args) catch return;
    console_log(msg.ptr, msg.len);
}

// Allocate memory ที่ JS สามารถ access ได้
export fn allocate(size: usize) [*]u8 {
    const slice = wasm_allocator.alloc(u8, size) catch return @ptrFromInt(0);
    return slice.ptr;
}

export fn deallocate(ptr: [*]u8, size: usize) void {
    wasm_allocator.free(ptr[0..size]);
}

// Process string จาก JavaScript
export fn processString(ptr: [*]const u8, len: usize) void {
    const str = ptr[0..len];
    jsLog("Received from JS: {s}\n", .{str});
}

// Array operations
export fn sumArray(ptr: [*]const f64, len: usize) f64 {
    var sum: f64 = 0;
    for (ptr[0..len]) |x| {
        sum += x;
    }
    return sum;
}

// Return pointer to string result
var result_buffer: [256]u8 = undefined;
var result_len: usize = 0;

export fn getGreeting(name_ptr: [*]const u8, name_len: usize) void {
    const name = name_ptr[0..name_len];
    const msg = std.fmt.bufPrint(&result_buffer, "สวัสดี, {s}! จาก Zig WASM", .{name}) catch return;
    result_len = msg.len;
}

export fn getResultPtr() [*]const u8 {
    return &result_buffer;
}

export fn getResultLen() usize {
    return result_len;
}

// Math operations
export fn add(a: i32, b: i32) i32 { return a + b; }
export fn fibonacci(n: u32) u32 {
    if (n <= 1) return n;
    var a: u32 = 0;
    var b: u32 = 1;
    var i: u32 = 2;
    while (i <= n) : (i += 1) {
        const c = a + b;
        a = b;
        b = c;
    }
    return b;
}

pub fn panic(_: []const u8, _: ?*std.builtin.StackTrace, _: ?usize) noreturn {
    @trap();
}
```

---

## Step 544: Calling JavaScript from Zig

### การ Import JS Functions

```zig
// src/js_interop.zig
const std = @import("std");

// === JavaScript API Imports ===

// DOM manipulation
extern fn js_getElementById(id_ptr: [*]const u8, id_len: usize) u32;
extern fn js_setInnerHTML(element_id: u32, html_ptr: [*]const u8, html_len: usize) void;
extern fn js_appendChild(parent_id: u32, child_id: u32) void;
extern fn js_createElement(tag_ptr: [*]const u8, tag_len: usize) u32;
extern fn js_setAttribute(element_id: u32, 
                           attr_ptr: [*]const u8, attr_len: usize,
                           val_ptr: [*]const u8, val_len: usize) void;

// Event handling
extern fn js_addEventListener(element_id: u32, 
                               event_ptr: [*]const u8, event_len: usize,
                               callback_id: u32) void;

// Timing
extern fn js_setTimeout(callback_id: u32, delay_ms: u32) u32;
extern fn js_clearTimeout(timeout_id: u32) void;

// Math/Random
extern fn js_random() f64;
extern fn js_now() f64; // Date.now()

// Canvas API
extern fn js_canvas_getContext(canvas_id: u32) u32;
extern fn js_canvas_fillRect(ctx_id: u32, x: f64, y: f64, w: f64, h: f64) void;
extern fn js_canvas_setFillStyle(ctx_id: u32, color_ptr: [*]const u8, color_len: usize) void;
extern fn js_canvas_clearRect(ctx_id: u32, x: f64, y: f64, w: f64, h: f64) void;
extern fn js_canvas_fillText(ctx_id: u32, text_ptr: [*]const u8, text_len: usize, x: f64, y: f64) void;

// === Zig wrapper types ===

pub const Element = struct {
    id: u32,
    
    pub fn setInnerHTML(self: Element, html: []const u8) void {
        js_setInnerHTML(self.id, html.ptr, html.len);
    }
    
    pub fn setAttribute(self: Element, attr: []const u8, value: []const u8) void {
        js_setAttribute(self.id, attr.ptr, attr.len, value.ptr, value.len);
    }
    
    pub fn appendChild(self: Element, child: Element) void {
        js_appendChild(self.id, child.id);
    }
};

pub fn getElementById(id: []const u8) Element {
    return .{ .id = js_getElementById(id.ptr, id.len) };
}

pub fn createElement(tag: []const u8) Element {
    return .{ .id = js_createElement(tag.ptr, tag.len) };
}

// Canvas wrapper
pub const Canvas2D = struct {
    ctx_id: u32,
    
    pub fn fillRect(self: Canvas2D, x: f64, y: f64, w: f64, h: f64) void {
        js_canvas_fillRect(self.ctx_id, x, y, w, h);
    }
    
    pub fn setFillStyle(self: Canvas2D, color: []const u8) void {
        js_canvas_setFillStyle(self.ctx_id, color.ptr, color.len);
    }
    
    pub fn clearRect(self: Canvas2D, x: f64, y: f64, w: f64, h: f64) void {
        js_canvas_clearRect(self.ctx_id, x, y, w, h);
    }
    
    pub fn fillText(self: Canvas2D, text: []const u8, x: f64, y: f64) void {
        js_canvas_fillText(self.ctx_id, text.ptr, text.len, x, y);
    }
};

// Random number
pub fn random() f64 {
    return js_random();
}

pub fn now() f64 {
    return js_now();
}
```

### JavaScript Glue Code

```javascript
// js_imports.js - กำหนด imported functions สำหรับ WASM

const memory = new WebAssembly.Memory({ initial: 16, maximum: 256 });

// Helper: อ่าน string จาก WASM memory
function readString(ptr, len) {
    const bytes = new Uint8Array(memory.buffer, ptr, len);
    return new TextDecoder().decode(bytes);
}

// Element registry (JS-side)
const elements = new Map();
let nextId = 1;

function registerElement(el) {
    const id = nextId++;
    elements.set(id, el);
    return id;
}

// Callback registry
const callbacks = new Map();
let nextCallbackId = 1;

export const imports = {
    env: {
        memory,
        
        // DOM
        js_getElementById: (ptr, len) => {
            const id = readString(ptr, len);
            const el = document.getElementById(id);
            return el ? registerElement(el) : 0;
        },
        
        js_setInnerHTML: (elemId, ptr, len) => {
            const el = elements.get(elemId);
            if (el) el.innerHTML = readString(ptr, len);
        },
        
        js_createElement: (ptr, len) => {
            const tag = readString(ptr, len);
            const el = document.createElement(tag);
            return registerElement(el);
        },
        
        js_setAttribute: (elemId, attrPtr, attrLen, valPtr, valLen) => {
            const el = elements.get(elemId);
            if (el) {
                const attr = readString(attrPtr, attrLen);
                const val = readString(valPtr, valLen);
                el.setAttribute(attr, val);
            }
        },
        
        js_appendChild: (parentId, childId) => {
            const parent = elements.get(parentId);
            const child = elements.get(childId);
            if (parent && child) parent.appendChild(child);
        },
        
        // Events
        js_addEventListener: (elemId, eventPtr, eventLen, callbackId) => {
            const el = elements.get(elemId);
            const event = readString(eventPtr, eventLen);
            if (el) {
                el.addEventListener(event, () => {
                    // Call back into WASM
                    instance.exports.handleCallback(callbackId);
                });
            }
        },
        
        // Timing
        js_setTimeout: (callbackId, delay) => {
            return setTimeout(() => {
                instance.exports.handleCallback(callbackId);
            }, delay);
        },
        
        js_clearTimeout: (id) => clearTimeout(id),
        
        // Utility
        js_random: () => Math.random(),
        js_now: () => Date.now(),
        
        // Canvas
        js_canvas_getContext: (canvasId) => {
            const canvas = elements.get(canvasId);
            if (!canvas) return 0;
            const ctx = canvas.getContext('2d');
            return registerElement(ctx);
        },
        
        js_canvas_fillRect: (ctxId, x, y, w, h) => {
            const ctx = elements.get(ctxId);
            if (ctx) ctx.fillRect(x, y, w, h);
        },
        
        js_canvas_setFillStyle: (ctxId, ptr, len) => {
            const ctx = elements.get(ctxId);
            if (ctx) ctx.fillStyle = readString(ptr, len);
        },
        
        js_canvas_clearRect: (ctxId, x, y, w, h) => {
            const ctx = elements.get(ctxId);
            if (ctx) ctx.clearRect(x, y, w, h);
        },
        
        js_canvas_fillText: (ctxId, ptr, len, x, y) => {
            const ctx = elements.get(ctxId);
            if (ctx) ctx.fillText(readString(ptr, len), x, y);
        },
    }
};
```

---

## Step 545: Memory Management ใน WASM

### WASM Memory Model

```zig
// src/memory.zig
// WASM มี linear memory model - เป็น contiguous byte array

const std = @import("std");

// WASM memory เริ่มที่ address 0 และขยายขึ้นได้
// ใน Zig เข้าถึงผ่าน WasmAllocator หรือ page_allocator

pub const WasmPageAllocator = struct {
    // WASM page = 65536 bytes (64KB)
    const PAGE_SIZE = 65536;
    
    // ขอ memory เพิ่มด้วย memory.grow instruction
    pub fn growPages(pages: u32) i32 {
        return asm volatile ("memory.grow %[result], %[pages]"
            : [result] "=r" (-> i32),
            : [pages] "r" (pages),
        );
    }
    
    pub fn currentPages() u32 {
        return asm volatile ("memory.size %[result]"
            : [result] "=r" (-> u32),
        );
    }
};

// Simple bump allocator สำหรับ WASM
pub const BumpAllocator = struct {
    next: usize,
    end: usize,
    
    pub fn init(start: usize, size: usize) BumpAllocator {
        return .{ .next = start, .end = start + size };
    }
    
    pub fn alloc(self: *BumpAllocator, size: usize, alignment: usize) ?[]u8 {
        const aligned = std.mem.alignForward(usize, self.next, alignment);
        const new_next = aligned + size;
        
        if (new_next > self.end) return null;
        
        self.next = new_next;
        return @as([*]u8, @ptrFromInt(aligned))[0..size];
    }
    
    pub fn reset(self: *BumpAllocator, start: usize) void {
        self.next = start;
    }
    
    pub fn used(self: BumpAllocator, start: usize) usize {
        return self.next - start;
    }
};

// เปิดเผย memory ให้ JavaScript
export var __heap_base: u32 = 0;

// Memory stats
export fn getMemoryUsage() u32 {
    return WasmPageAllocator.currentPages() * WasmPageAllocator.PAGE_SIZE;
}

export fn getHeapBase() u32 {
    return __heap_base;
}
```

### Sharing Memory with JavaScript

```zig
// src/shared_memory.zig
// Sharing buffers ระหว่าง Zig WASM และ JavaScript

// Shared ring buffer สำหรับ communication
pub const RingBuffer = struct {
    const CAPACITY = 1024;
    
    data: [CAPACITY]u8 = undefined,
    read_pos: u32 = 0,
    write_pos: u32 = 0,
    
    pub fn push(self: *RingBuffer, byte: u8) bool {
        const next = (self.write_pos + 1) % CAPACITY;
        if (next == self.read_pos) return false; // Full
        
        self.data[self.write_pos] = byte;
        self.write_pos = next;
        return true;
    }
    
    pub fn pop(self: *RingBuffer) ?u8 {
        if (self.read_pos == self.write_pos) return null; // Empty
        
        const byte = self.data[self.read_pos];
        self.read_pos = (self.read_pos + 1) % CAPACITY;
        return byte;
    }
    
    pub fn available(self: *RingBuffer) u32 {
        if (self.write_pos >= self.read_pos) {
            return self.write_pos - self.read_pos;
        }
        return CAPACITY - self.read_pos + self.write_pos;
    }
};

// Shared state ที่ JavaScript สามารถเข้าถึงได้
export var input_buffer = RingBuffer{};
export var output_buffer = RingBuffer{};

export fn getInputBufferPtr() *RingBuffer {
    return &input_buffer;
}

export fn getOutputBufferPtr() *RingBuffer {
    return &output_buffer;
}

// SharedArrayBuffer สำหรับ multi-threaded WASM
// (ต้องมี Cross-Origin-Isolation headers)
export var shared_counter: i32 = 0;

export fn atomicIncrement() i32 {
    return @atomicRmw(i32, &shared_counter, .Add, 1, .seq_cst);
}
```

---

## Step 546: WASM Imports/Exports

### Comprehensive Export API

```zig
// src/api.zig - WASM public API
const std = @import("std");

// === Memory Management API ===
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

export fn malloc(size: usize) u32 {
    const slice = allocator.alloc(u8, size) catch return 0;
    return @intFromPtr(slice.ptr);
}

export fn free(ptr: u32, size: usize) void {
    const slice = @as([*]u8, @ptrFromInt(ptr))[0..size];
    allocator.free(slice);
}

// === String API ===
export fn stringLength(ptr: u32) u32 {
    const cstr: [*:0]const u8 = @ptrFromInt(ptr);
    return @intCast(std.mem.len(cstr));
}

export fn stringCompare(a_ptr: u32, b_ptr: u32) i32 {
    const a: [*:0]const u8 = @ptrFromInt(a_ptr);
    const b: [*:0]const u8 = @ptrFromInt(b_ptr);
    return std.mem.orderZ(u8, a, b).compare(std.math.Order.eq);
}

// === JSON Processing API ===
const JsonValue = union(enum) {
    null_val: void,
    bool_val: bool,
    int_val: i64,
    float_val: f64,
    string_val: []const u8,
};

// === Math API ===
export fn vectorDot(a_ptr: u32, b_ptr: u32, len: u32) f32 {
    const a: [*]const f32 = @ptrFromInt(a_ptr);
    const b: [*]const f32 = @ptrFromInt(b_ptr);
    
    var sum: f32 = 0;
    for (a[0..len], b[0..len]) |av, bv| {
        sum += av * bv;
    }
    return sum;
}

export fn matrixMultiply4x4(
    a_ptr: u32,
    b_ptr: u32,
    result_ptr: u32,
) void {
    const a: *const [16]f32 = @ptrFromInt(a_ptr);
    const b: *const [16]f32 = @ptrFromInt(b_ptr);
    var result: *[16]f32 = @ptrFromInt(result_ptr);
    
    for (0..4) |i| {
        for (0..4) |j| {
            var sum: f32 = 0;
            for (0..4) |k| {
                sum += a[i * 4 + k] * b[k * 4 + j];
            }
            result[i * 4 + j] = sum;
        }
    }
}

// === Image Processing API ===
export fn imageGrayscale(
    data_ptr: u32,
    width: u32,
    height: u32,
) void {
    const data: [*]u8 = @ptrFromInt(data_ptr);
    const len = width * height * 4; // RGBA
    
    var i: usize = 0;
    while (i < len) : (i += 4) {
        const r: f32 = @floatFromInt(data[i]);
        const g: f32 = @floatFromInt(data[i + 1]);
        const b: f32 = @floatFromInt(data[i + 2]);
        
        const gray: u8 = @intFromFloat(r * 0.299 + g * 0.587 + b * 0.114);
        data[i] = gray;
        data[i + 1] = gray;
        data[i + 2] = gray;
        // alpha ไม่เปลี่ยน
    }
}

export fn imageBlur(
    src_ptr: u32,
    dst_ptr: u32,
    width: u32,
    height: u32,
) void {
    const src: [*]const u8 = @ptrFromInt(src_ptr);
    const dst: [*]u8 = @ptrFromInt(dst_ptr);
    
    // Simple box blur 3x3
    var y: u32 = 1;
    while (y < height - 1) : (y += 1) {
        var x: u32 = 1;
        while (x < width - 1) : (x += 1) {
            for (0..3) |c| { // RGB channels
                var sum: u32 = 0;
                for (0..3) |dy| {
                    for (0..3) |dx| {
                        const px = (y + dy - 1) * width + (x + dx - 1);
                        sum += src[px * 4 + c];
                    }
                }
                dst[(y * width + x) * 4 + c] = @intCast(sum / 9);
            }
            dst[(y * width + x) * 4 + 3] = src[(y * width + x) * 4 + 3]; // Alpha
        }
    }
}

pub fn panic(_: []const u8, _: ?*std.builtin.StackTrace, _: ?usize) noreturn {
    @trap();
}
```

---

## Step 547: WASM สำหรับ Web Applications

### Canvas Game ด้วย Zig WASM

```zig
// src/game.zig - Simple game ด้วย Zig WASM
const js = @import("js_interop.zig");

const CANVAS_WIDTH = 800;
const CANVAS_HEIGHT = 600;

const Player = struct {
    x: f64,
    y: f64,
    vx: f64,
    vy: f64,
    width: f64,
    height: f64,
    
    pub fn update(self: *Player, dt: f64) void {
        self.x += self.vx * dt;
        self.y += self.vy * dt;
        
        // Bounce off walls
        if (self.x <= 0 or self.x + self.width >= CANVAS_WIDTH) {
            self.vx = -self.vx;
        }
        if (self.y <= 0 or self.y + self.height >= CANVAS_HEIGHT) {
            self.vy = -self.vy;
        }
        
        // Clamp to canvas
        self.x = @max(0, @min(CANVAS_WIDTH - self.width, self.x));
        self.y = @max(0, @min(CANVAS_HEIGHT - self.height, self.y));
    }
};

const Particle = struct {
    x: f64,
    y: f64,
    vx: f64,
    vy: f64,
    life: f64,
    color: [7]u8, // "#RRGGBB"
};

var player = Player{
    .x = 400,
    .y = 300,
    .vx = 150,
    .vy = 100,
    .width = 30,
    .height = 30,
};

var particles: [100]Particle = undefined;
var particle_count: usize = 0;
var canvas_ctx: u32 = 0;
var last_time: f64 = 0;

export fn init(ctx_id: u32) void {
    canvas_ctx = ctx_id;
    last_time = js.now();
}

export fn update() void {
    const current_time = js.now();
    const dt = (current_time - last_time) / 1000.0; // Convert to seconds
    last_time = current_time;
    
    // Update player
    player.update(dt);
    
    // Spawn particles
    if (particle_count < 100) {
        particles[particle_count] = .{
            .x = player.x + player.width / 2,
            .y = player.y + player.height / 2,
            .vx = (js.random() - 0.5) * 200,
            .vy = (js.random() - 0.5) * 200,
            .life = 1.0,
            .color = "#FF8800".*,
        };
        particle_count += 1;
    }
    
    // Update particles
    var i: usize = 0;
    while (i < particle_count) {
        particles[i].x += particles[i].vx * dt;
        particles[i].y += particles[i].vy * dt;
        particles[i].life -= dt;
        
        if (particles[i].life <= 0) {
            particles[i] = particles[particle_count - 1];
            particle_count -= 1;
        } else {
            i += 1;
        }
    }
}

const ctx2d = js.Canvas2D{ .ctx_id = 0 }; // Will be set from canvas_ctx

export fn render() void {
    const ctx = js.Canvas2D{ .ctx_id = canvas_ctx };
    
    // Clear
    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
    
    // Draw background
    ctx.setFillStyle("#1a1a2e");
    ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
    
    // Draw particles
    for (particles[0..particle_count]) |p| {
        ctx.setFillStyle(&p.color);
        ctx.fillRect(p.x - 2, p.y - 2, 4, 4);
    }
    
    // Draw player
    ctx.setFillStyle("#00ff88");
    ctx.fillRect(player.x, player.y, player.width, player.height);
    
    // Draw score
    ctx.setFillStyle("#ffffff");
    ctx.fillText("Zig WASM Game", 10, 30);
}
```

---

## Step 548: WASM สำหรับ Server-Side (WASI)

### WASI Program

```zig
// src/wasi_main.zig - WASI-based server application
const std = @import("std");

// WASI มี standard POSIX-like interface
pub fn main() !void {
    const stdin = std.io.getStdIn();
    const stdout = std.io.getStdOut();
    const stderr = std.io.getStdErr();
    
    _ = stderr;
    
    const reader = stdin.reader();
    const writer = stdout.writer();
    
    try writer.writeAll("WASI Zig Server\n");
    try writer.writeAll("Enter commands (quit to exit):\n");
    
    var buf: [1024]u8 = undefined;
    
    while (true) {
        try writer.writeAll("> ");
        
        const line = reader.readUntilDelimiter(&buf, '\n') catch break;
        const trimmed = std.mem.trim(u8, line, &std.ascii.whitespace);
        
        if (std.mem.eql(u8, trimmed, "quit")) break;
        
        // Process command
        try processCommand(trimmed, writer);
    }
    
    try writer.writeAll("Goodbye!\n");
}

fn processCommand(cmd: []const u8, writer: anytype) !void {
    if (std.mem.startsWith(u8, cmd, "echo ")) {
        try writer.print("{s}\n", .{cmd[5..]});
    } else if (std.mem.eql(u8, cmd, "time")) {
        // WASI clock
        var ts: std.os.timespec = undefined;
        std.os.clock_gettime(std.os.CLOCK.REALTIME, &ts) catch {};
        try writer.print("Unix time: {d}\n", .{ts.tv_sec});
    } else if (std.mem.eql(u8, cmd, "help")) {
        try writer.writeAll("Commands: echo <text>, time, help, quit\n");
    } else {
        try writer.print("Unknown command: {s}\n", .{cmd});
    }
}
```

### WASI File System Access

```zig
// src/wasi_fs.zig - File system access ใน WASI
const std = @import("std");

pub fn readFileWasi(path: []const u8) ![]u8 {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();
    
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    
    const size = try file.getEndPos();
    const buf = try alloc.alloc(u8, size);
    
    _ = try file.readAll(buf);
    return buf;
}

pub fn writeFileWasi(path: []const u8, data: []const u8) !void {
    const file = try std.fs.cwd().createFile(path, .{});
    defer file.close();
    
    try file.writeAll(data);
}

// Process CSV ใน WASI
pub fn processCSV(csv_data: []const u8) !void {
    const stdout = std.io.getStdOut().writer();
    
    var lines = std.mem.splitScalar(u8, csv_data, '\n');
    var row_count: u32 = 0;
    
    while (lines.next()) |line| {
        if (line.len == 0) continue;
        
        var fields = std.mem.splitScalar(u8, line, ',');
        try stdout.writeAll("Row: ");
        
        while (fields.next()) |field| {
            try stdout.print("[{s}] ", .{field});
        }
        
        try stdout.writeAll("\n");
        row_count += 1;
    }
    
    try stdout.print("Total rows: {}\n", .{row_count});
}

pub fn main() !void {
    const args = try std.process.argsAlloc(std.heap.page_allocator);
    defer std.process.argsFree(std.heap.page_allocator, args);
    
    if (args.len < 2) {
        std.debug.print("Usage: {s} <csv_file>\n", .{args[0]});
        return;
    }
    
    const data = try readFileWasi(args[1]);
    defer std.heap.page_allocator.free(data);
    
    try processCSV(data);
}
```

---

## Step 549: WASM Optimization

### Reducing Binary Size

```zig
// src/minimal.zig - Minimal WASM module

// ไม่ใช้ std ที่ไม่จำเป็น
// ไม่มี panic handler ที่ verbose

// Panic handler แบบ minimal (แค่ trap)
pub fn panic(_: []const u8, _: ?*@import("std").builtin.StackTrace, _: ?usize) noreturn {
    @trap();
}

// ใช้ integer arithmetic แทน float เมื่อทำได้
export fn intSqrt(n: u32) u32 {
    if (n == 0) return 0;
    
    var x = n;
    var y = (x + 1) / 2;
    
    while (y < x) {
        x = y;
        y = (x + n / x) / 2;
    }
    
    return x;
}

// Avoid dynamic dispatch
export fn processFixed(data: [*]const u8, len: usize) u32 {
    var checksum: u32 = 0;
    for (data[0..len]) |byte| {
        checksum = checksum *% 31 +% byte;
    }
    return checksum;
}
```

```bash
# Build optimizations สำหรับ binary size
zig build-exe main.zig \
    -target wasm32-freestanding \
    -O ReleaseSmall \
    -fno-entry \
    --strip

# ใช้ wasm-opt สำหรับ further optimization (ต้องติดตั้ง binaryen)
wasm-opt -O3 --strip-debug -o app.opt.wasm app.wasm

# ดูขนาด
ls -la *.wasm
```

### Performance Tips

```zig
// src/performance.zig - WASM performance best practices
const std = @import("std");

// 1. หลีกเลี่ยง JS boundary crossing บ่อยเกินไป
// Bad: เรียก JS function ทุก pixel
// Good: process ทั้ง buffer ใน WASM แล้วส่งผลลัพธ์ครั้งเดียว

export fn processBuffer(
    input_ptr: u32,
    output_ptr: u32,
    width: u32,
    height: u32,
) void {
    const input: [*]const u8 = @ptrFromInt(input_ptr);
    const output: [*]u8 = @ptrFromInt(output_ptr);
    const len = width * height * 4;
    
    // Process ทั้งหมดใน Zig ไม่ต้อง cross JS boundary
    var i: usize = 0;
    while (i < len) : (i += 4) {
        const r: u16 = input[i];
        const g: u16 = input[i + 1];
        const b: u16 = input[i + 2];
        
        // Sepia effect
        output[i]     = @min(255, @as(u32, r * 393 + g * 769 + b * 189) / 1000);
        output[i + 1] = @min(255, @as(u32, r * 349 + g * 686 + b * 168) / 1000);
        output[i + 2] = @min(255, @as(u32, r * 272 + g * 534 + b * 131) / 1000);
        output[i + 3] = input[i + 3];
    }
}

// 2. ใช้ SIMD เมื่อทำได้
export fn sumFloatsSIMD(ptr: u32, len: u32) f32 {
    const data: [*]const f32 = @ptrFromInt(ptr);
    const slice = data[0..len];
    
    // WASM SIMD (wasm-simd128)
    const Vec4 = @Vector(4, f32);
    var sum_vec: Vec4 = @splat(0.0);
    
    var i: usize = 0;
    while (i + 4 <= len) : (i += 4) {
        const v: Vec4 = slice[i..][0..4].*;
        sum_vec += v;
    }
    
    var sum = @reduce(.Add, sum_vec);
    while (i < len) : (i += 1) {
        sum += slice[i];
    }
    
    return sum;
}

// 3. Memory allocation strategy
var frame_arena: std.heap.ArenaAllocator = undefined;
var initialized = false;

export fn initFrameAllocator() void {
    if (!initialized) {
        frame_arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
        initialized = true;
    }
}

export fn beginFrame() void {
    _ = frame_arena.reset(.retain_capacity);
}

// 4. Avoid unnecessary copies
export fn transformInPlace(ptr: u32, len: u32) void {
    const data: [*]f32 = @ptrFromInt(ptr);
    
    for (data[0..len]) |*x| {
        x.* = x.* * 2.0 + 1.0; // Transform in-place, no copy needed
    }
}
```

---

## Step 550: Real-World WASM Project

### Image Processor ใน Browser

```zig
// src/image_processor.zig - Complete image processing WASM module
const std = @import("std");

// Extern JS functions
extern fn js_log(ptr: [*]const u8, len: usize) void;

fn log(msg: []const u8) void {
    js_log(msg.ptr, msg.len);
}

// Image data structure (RGBA 8-bit)
pub const Image = struct {
    data: [*]u8,
    width: u32,
    height: u32,
    
    pub fn pixel(self: Image, x: u32, y: u32) *[4]u8 {
        return @ptrCast(self.data[(y * self.width + x) * 4..].ptr);
    }
    
    pub fn len(self: Image) usize {
        return self.width * self.height * 4;
    }
};

// === Filter Functions ===

export fn applyGrayscale(data_ptr: u32, width: u32, height: u32) void {
    const img = Image{
        .data = @ptrFromInt(data_ptr),
        .width = width,
        .height = height,
    };
    
    var y: u32 = 0;
    while (y < height) : (y += 1) {
        var x: u32 = 0;
        while (x < width) : (x += 1) {
            const px = img.pixel(x, y);
            const gray: u8 = @intFromFloat(
                @as(f32, @floatFromInt(px[0])) * 0.299 +
                @as(f32, @floatFromInt(px[1])) * 0.587 +
                @as(f32, @floatFromInt(px[2])) * 0.114
            );
            px[0] = gray;
            px[1] = gray;
            px[2] = gray;
        }
    }
    
    log("Grayscale applied");
}

export fn applyBrightness(data_ptr: u32, width: u32, height: u32, amount: i32) void {
    const img = Image{
        .data = @ptrFromInt(data_ptr),
        .width = width,
        .height = height,
    };
    
    const len = img.len();
    var i: usize = 0;
    while (i < len) : (i += 4) {
        for (0..3) |c| { // RGB only, skip alpha
            const val: i32 = @as(i32, img.data[i + c]) + amount;
            img.data[i + c] = @intCast(std.math.clamp(val, 0, 255));
        }
    }
}

export fn applyContrast(data_ptr: u32, width: u32, height: u32, factor: f32) void {
    const img = Image{
        .data = @ptrFromInt(data_ptr),
        .width = width,
        .height = height,
    };
    
    const len = img.len();
    var i: usize = 0;
    while (i < len) : (i += 4) {
        for (0..3) |c| {
            const val: f32 = (@as(f32, @floatFromInt(img.data[i + c])) - 128.0) * factor + 128.0;
            img.data[i + c] = @intCast(std.math.clamp(@as(i32, @intFromFloat(val)), 0, 255));
        }
    }
}

export fn applyInvert(data_ptr: u32, width: u32, height: u32) void {
    const img = Image{
        .data = @ptrFromInt(data_ptr),
        .width = width,
        .height = height,
    };
    
    const len = img.len();
    var i: usize = 0;
    while (i < len) : (i += 4) {
        img.data[i]     = 255 - img.data[i];
        img.data[i + 1] = 255 - img.data[i + 1];
        img.data[i + 2] = 255 - img.data[i + 2];
        // alpha ไม่เปลี่ยน
    }
}

export fn applyBlur(
    src_ptr: u32,
    dst_ptr: u32,
    width: u32,
    height: u32,
    radius: u32,
) void {
    const src: [*]const u8 = @ptrFromInt(src_ptr);
    const dst: [*]u8 = @ptrFromInt(dst_ptr);
    
    var y: u32 = 0;
    while (y < height) : (y += 1) {
        var x: u32 = 0;
        while (x < width) : (x += 1) {
            var r: u32 = 0;
            var g: u32 = 0;
            var b: u32 = 0;
            var count: u32 = 0;
            
            // Sample kernel
            var dy: i32 = -@as(i32, radius);
            while (dy <= @as(i32, radius)) : (dy += 1) {
                var dx: i32 = -@as(i32, radius);
                while (dx <= @as(i32, radius)) : (dx += 1) {
                    const nx = @as(i32, @intCast(x)) + dx;
                    const ny = @as(i32, @intCast(y)) + dy;
                    
                    if (nx >= 0 and nx < @as(i32, @intCast(width)) and
                        ny >= 0 and ny < @as(i32, @intCast(height)))
                    {
                        const px = (@as(usize, @intCast(ny)) * width + @as(usize, @intCast(nx))) * 4;
                        r += src[px];
                        g += src[px + 1];
                        b += src[px + 2];
                        count += 1;
                    }
                }
            }
            
            const dst_px = (y * width + x) * 4;
            dst[dst_px]     = @intCast(r / count);
            dst[dst_px + 1] = @intCast(g / count);
            dst[dst_px + 2] = @intCast(b / count);
            dst[dst_px + 3] = src[dst_px + 3]; // Preserve alpha
        }
    }
}

// Histogram calculation
export fn calculateHistogram(
    data_ptr: u32,
    width: u32,
    height: u32,
    hist_ptr: u32, // 256 * 3 * u32
) void {
    const img = Image{
        .data = @ptrFromInt(data_ptr),
        .width = width,
        .height = height,
    };
    const histogram: [*]u32 = @ptrFromInt(hist_ptr);
    
    // Clear histogram
    @memset(histogram[0..256 * 3], 0);
    
    const len = img.len();
    var i: usize = 0;
    while (i < len) : (i += 4) {
        histogram[img.data[i]] += 1;             // R histogram
        histogram[256 + img.data[i + 1]] += 1;  // G histogram
        histogram[512 + img.data[i + 2]] += 1;  // B histogram
    }
}

pub fn panic(_: []const u8, _: ?*std.builtin.StackTrace, _: ?usize) noreturn {
    @trap();
}
```

### HTML Page สำหรับ Image Processor

```html
<!-- image_processor.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Zig WASM Image Processor</title>
    <style>
        body { font-family: sans-serif; max-width: 1000px; margin: 0 auto; padding: 20px; }
        canvas { border: 1px solid #ccc; max-width: 100%; }
        .controls { display: flex; gap: 10px; flex-wrap: wrap; margin: 10px 0; }
        button { padding: 8px 16px; cursor: pointer; }
        input[type="range"] { width: 200px; }
    </style>
</head>
<body>
    <h1>🎨 Zig WASM Image Processor</h1>
    
    <input type="file" id="fileInput" accept="image/*">
    
    <div class="controls">
        <button onclick="applyFilter('grayscale')">Grayscale</button>
        <button onclick="applyFilter('invert')">Invert</button>
        <button onclick="resetImage()">Reset</button>
        
        <label>
            Brightness:
            <input type="range" min="-100" max="100" value="0" 
                   oninput="applyBrightness(this.value)">
        </label>
        
        <label>
            Contrast:
            <input type="range" min="0.1" max="3" step="0.1" value="1" 
                   oninput="applyContrast(this.value)">
        </label>
        
        <label>
            Blur:
            <input type="range" min="0" max="10" value="0" 
                   oninput="applyBlur(this.value)">
        </label>
    </div>
    
    <canvas id="canvas"></canvas>
    
    <div id="info"></div>
    
    <script type="module">
    let wasmInstance;
    let originalData;
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    
    async function loadWasm() {
        const response = await fetch('image_processor.wasm');
        const bytes = await response.arrayBuffer();
        
        const { instance } = await WebAssembly.instantiate(bytes, {
            env: {
                js_log: (ptr, len) => {
                    const bytes = new Uint8Array(instance.exports.memory.buffer, ptr, len);
                    console.log(new TextDecoder().decode(bytes));
                }
            }
        });
        
        wasmInstance = instance;
        document.getElementById('info').textContent = 'WASM loaded! Select an image.';
    }
    
    document.getElementById('fileInput').addEventListener('change', async (e) => {
        const file = e.target.files[0];
        if (!file) return;
        
        const img = new Image();
        img.onload = () => {
            canvas.width = img.width;
            canvas.height = img.height;
            ctx.drawImage(img, 0, 0);
            
            const imageData = ctx.getImageData(0, 0, img.width, img.height);
            originalData = new Uint8Array(imageData.data);
            
            document.getElementById('info').textContent = 
                `Image: ${img.width}x${img.height} (${(originalData.length / 1024).toFixed(1)} KB)`;
        };
        
        img.src = URL.createObjectURL(file);
    });
    
    function getCurrentImageData() {
        return ctx.getImageData(0, 0, canvas.width, canvas.height);
    }
    
    function putImageToWasm(imageData) {
        const memory = wasmInstance.exports.memory;
        const ptr = wasmInstance.exports.malloc(imageData.data.length);
        new Uint8Array(memory.buffer, ptr, imageData.data.length).set(imageData.data);
        return ptr;
    }
    
    window.applyFilter = function(filterName) {
        if (!wasmInstance || !canvas.width) return;
        
        const imageData = getCurrentImageData();
        const ptr = putImageToWasm(imageData);
        
        switch(filterName) {
            case 'grayscale':
                wasmInstance.exports.applyGrayscale(ptr, canvas.width, canvas.height);
                break;
            case 'invert':
                wasmInstance.exports.applyInvert(ptr, canvas.width, canvas.height);
                break;
        }
        
        const result = new Uint8Array(wasmInstance.exports.memory.buffer, ptr, imageData.data.length);
        const newImageData = new ImageData(new Uint8ClampedArray(result), canvas.width, canvas.height);
        ctx.putImageData(newImageData, 0, 0);
        
        wasmInstance.exports.free(ptr, imageData.data.length);
    };
    
    window.applyBrightness = function(amount) {
        if (!wasmInstance || !canvas.width) return;
        
        // Reset to original first
        const orig = new ImageData(
            new Uint8ClampedArray(originalData), 
            canvas.width, canvas.height
        );
        ctx.putImageData(orig, 0, 0);
        
        if (amount == 0) return;
        
        const imageData = getCurrentImageData();
        const ptr = putImageToWasm(imageData);
        
        wasmInstance.exports.applyBrightness(ptr, canvas.width, canvas.height, parseInt(amount));
        
        const result = new Uint8Array(wasmInstance.exports.memory.buffer, ptr, imageData.data.length);
        ctx.putImageData(new ImageData(new Uint8ClampedArray(result), canvas.width, canvas.height), 0, 0);
        wasmInstance.exports.free(ptr, imageData.data.length);
    };
    
    window.resetImage = function() {
        if (originalData && canvas.width) {
            const imageData = new ImageData(
                new Uint8ClampedArray(originalData), 
                canvas.width, canvas.height
            );
            ctx.putImageData(imageData, 0, 0);
        }
    };
    
    loadWasm();
    </script>
</body>
</html>
```

---

## สรุป Part 55

ในบทนี้เราได้เรียนรู้:

1. **WASM Basics** - WebAssembly คืออะไรและทำไมใช้กับ Zig
2. **Compilation** - build.zig สำหรับ WASM target
3. **JS Integration** - export/import functions ระหว่าง Zig และ JavaScript
4. **Memory Management** - WASM linear memory, sharing ระหว่าง Zig-JS
5. **Import/Export API** - ออกแบบ clean API สำหรับ WASM modules
6. **Web Applications** - canvas game, interactive demos
7. **WASI** - server-side WASM, file system access
8. **Optimization** - ลด binary size, เพิ่ม performance
9. **Real Project** - Image processor ครบวงจรพร้อม HTML interface

### ขั้นตอนถัดไป

```bash
# ทดสอบ WASM ด้วย Node.js
node --experimental-wasm-simd script.js

# ทดสอบ WASI ด้วย wasmtime
wasmtime --dir=. server-app.wasm

# หรือ wasmer
wasmer run --dir=. server-app.wasm

# Debug WASM ด้วย Chrome DevTools
# 1. เปิด Chrome DevTools
# 2. Sources tab
# 3. เลือก .wasm file
# 4. Set breakpoints ใน WAT format
```

> **Tips**: WASM เป็น technology ที่กำลังพัฒนา features ใหม่อยู่เสมอ เช่น WASM GC, WASM Threads, Component Model ควรติดตามข่าวสารจาก https://webassembly.org/
