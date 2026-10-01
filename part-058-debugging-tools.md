# Part 58: Debugging Tools สำหรับ Zig (Steps 571-580)

การ debug โปรแกรม Zig สามารถทำได้ด้วยเครื่องมือหลายอย่าง ตั้งแต่ command-line debuggers อย่าง GDB และ LLDB ไปจนถึง IDE integrations และ memory analysis tools บทนี้จะครอบคลุมทุกเครื่องมือที่จำเป็นสำหรับการ debug อย่างมีประสิทธิภาพ

---

## Step 571: GDB Setup สำหรับ Zig

### ติดตั้งและ Configure GDB

```bash
# ติดตั้ง GDB (Linux)
sudo apt-get install gdb            # Ubuntu/Debian
sudo dnf install gdb                # Fedora/RHEL
sudo pacman -S gdb                  # Arch Linux

# ตรวจสอบ version
gdb --version
# GNU gdb 13.x หรือใหม่กว่า
```

### Build พร้อม Debug Information

```bash
# Build สำหรับ debugging (เก็บ debug info)
zig build-exe main.zig -O Debug
# หรือใน build.zig: .optimize = .Debug (ค่าเริ่มต้น)

# ตรวจสอบ debug information
file main
# main: ELF 64-bit LSB executable, ... with debug_info

objdump --dwarf=info main | head -50
# แสดง DWARF debug information
```

### GDB Python Extension สำหรับ Zig

```python
# ~/.gdbinit.d/zig.py - GDB extension สำหรับ Zig types

import gdb

class ZigSlicePrinter:
    """Pretty printer สำหรับ Zig slice"""
    
    def __init__(self, val):
        self.val = val
    
    def to_string(self):
        ptr = self.val['ptr']
        length = self.val['len']
        return f"Zig slice of length {length} at {ptr}"
    
    def children(self):
        ptr = self.val['ptr']
        length = int(self.val['len'])
        for i in range(min(length, 20)):  # แสดงสูงสุด 20 elements
            yield f'[{i}]', ptr[i]

class ZigOptionalPrinter:
    """Pretty printer สำหรับ Zig optional"""
    
    def __init__(self, val):
        self.val = val
    
    def to_string(self):
        # ตรวจสอบ null
        if self.val['is_null']:
            return "null"
        return f"?{self.val['payload']}"

def register_zig_printers(objfile):
    if objfile is None:
        objfile = gdb
    # Register printers here
    pass

gdb.events.new_objfile.connect(register_zig_printers)
```

### .gdbinit Configuration

```
# ~/.gdbinit - GDB configuration สำหรับ Zig

# เปิดใช้ pretty printing
set print pretty on
set print array on
set print array-indexes on

# ตั้ง pagination
set pagination off

# ใช้ Intel syntax สำหรับ x86
set disassembly-flavor intel

# แสดง source ขณะ debug
set listsize 20

# Auto load scripts
set auto-load safe-path /home/user/zig_course

# Custom commands สำหรับ Zig
define zig-bt
    backtrace full
end
document zig-bt
    Print a full Zig backtrace with local variables
end
```

### ตัวอย่าง Program สำหรับ Debug

```zig
// debug_example.zig - program สำหรับทดสอบ GDB
const std = @import("std");

const Node = struct {
    value: i32,
    next: ?*Node,
};

fn createLinkedList(allocator: std.mem.Allocator, count: u32) !*Node {
    var head: ?*Node = null;
    var i: u32 = 0;
    
    while (i < count) : (i += 1) {
        const node = try allocator.create(Node);
        node.* = .{
            .value = @intCast(i),
            .next = head,
        };
        head = node;
    }
    
    return head orelse error.EmptyList;
}

fn sumList(head: *const Node) i64 {
    var sum: i64 = 0;
    var current: ?*const Node = head;
    
    while (current) |node| {
        sum += node.value;
        current = node.next;
    }
    
    return sum;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const list = try createLinkedList(allocator, 10);
    const sum = sumList(list);
    
    std.debug.print("Sum of list: {}\n", .{sum});
}
```

---

## Step 572: GDB Commands สำหรับ Zig Programs

### เริ่มต้น Debug Session

```bash
# เริ่ม GDB
gdb ./debug_example

# หรือ attach กับ process ที่กำลัง run
gdb -p <pid>

# หรือ open core dump
gdb ./debug_example core
```

### Basic GDB Commands

```
# เริ่ม program
(gdb) run
(gdb) r           # shortcut

# ตั้ง breakpoint
(gdb) break main
(gdb) break debug_example.zig:25
(gdb) b 25        # shortcut
(gdb) break sumList if sum > 100  # conditional breakpoint

# ดู breakpoints
(gdb) info breakpoints
(gdb) info b

# ลบ breakpoint
(gdb) delete 1    # ลบ breakpoint หมายเลข 1
(gdb) clear main  # ลบ breakpoints ที่ function main

# Step through code
(gdb) next        # execute บรรทัดถัดไป (ไม่เข้า function)
(gdb) n           # shortcut
(gdb) step        # step into function call
(gdb) s           # shortcut
(gdb) finish      # run จนออกจาก function ปัจจุบัน
(gdb) continue    # run ต่อจนถึง breakpoint ถัดไป
(gdb) c           # shortcut
```

### ดู Variables และ Memory

```
# แสดงค่า variable
(gdb) print sum
(gdb) p sum

# แสดง expression
(gdb) print node.value
(gdb) print *node

# แสดง local variables ทั้งหมด
(gdb) info locals

# แสดง function arguments
(gdb) info args

# ดู memory
(gdb) x/10xb address    # แสดง 10 bytes ใน hex
(gdb) x/4xw &variable   # แสดง 4 words
(gdb) x/s string_ptr    # แสดง null-terminated string

# Watchpoint (หยุดเมื่อค่าเปลี่ยน)
(gdb) watch sum
(gdb) rwatch sum   # หยุดเมื่ออ่าน
(gdb) awatch sum   # หยุดเมื่ออ่านหรือเขียน
```

### Backtrace และ Stack Frame

```
# แสดง call stack
(gdb) backtrace
(gdb) bt
(gdb) bt full      # พร้อม local variables

# เลื่อน frame
(gdb) frame 0      # ไปที่ frame 0 (current)
(gdb) frame 1      # ไปที่ frame ที่เรียก current
(gdb) up           # ไป frame ที่สูงกว่า
(gdb) down         # ไป frame ที่ต่ำกว่า

# ดู source code ใน frame ปัจจุบัน
(gdb) list
(gdb) l
```

### ตัวอย่าง GDB Session

```
$ gdb ./debug_example
(gdb) break createLinkedList
Breakpoint 1 at 0x401234: file debug_example.zig, line 10

(gdb) run
Starting program: ./debug_example
Breakpoint 1, createLinkedList (...) at debug_example.zig:10

(gdb) info args
allocator = {ptr = 0x7fffffffe..., vtable = 0x...}
count = 10

(gdb) next
(gdb) print i
$1 = 0

(gdb) print head
$2 = (debug_example.Node *) 0x0  # null

(gdb) continue
(gdb) break sumList
(gdb) continue

Breakpoint 2, sumList (...) at debug_example.zig:24

(gdb) print *head
$3 = {value = 9, next = 0x...}

(gdb) finish
Run till exit from sumList
$4 = 45   # 0+1+2+...+9 = 45

(gdb) quit
```

---

## Step 573: LLDB Setup และ Usage

### ติดตั้ง LLDB

```bash
# Linux
sudo apt-get install lldb
sudo dnf install lldb

# macOS (มาพร้อมกับ Xcode Command Line Tools)
xcode-select --install

# ตรวจสอบ version
lldb --version
```

### LLDB Commands เปรียบเทียบกับ GDB

```
GDB Command          | LLDB Command
---------------------|---------------------------
run                  | run (r)
break main           | breakpoint set --name main
break file.c:line    | breakpoint set --file file.c --line line
delete 1             | breakpoint delete 1
next                 | next (n)
step                 | step (s)
finish               | finish
continue             | continue (c)
print var            | frame variable var
info locals          | frame variable
backtrace            | thread backtrace (bt)
info threads         | thread list
```

### ตัวอย่าง LLDB Session

```bash
$ lldb ./debug_example

(lldb) breakpoint set --name main
Breakpoint 1: where = debug_example`main, address = 0x...

(lldb) run
Process started, stopped at main...

(lldb) breakpoint set --file debug_example.zig --line 25
Breakpoint 2 set at debug_example.zig:25

(lldb) continue
Process stopped at line 25...

(lldb) frame variable
(i64) sum = 0
(debug_example.Node *) current = 0x...

(lldb) expr sum
(i64) $0 = 0

(lldb) memory read -s4 -fx node
0x...: 0x00000009 0x00000000 0x...

(lldb) thread backtrace
* thread #1, name = 'debug_example'
  * frame #0: debug_example.zig:25 sumList(head=...)
    frame #1: debug_example.zig:38 main(...)
```

### LLDB Commands ที่มีประโยชน์

```
# แสดง disassembly
(lldb) disassemble --name sumList
(lldb) di -n sumList

# ดู registers
(lldb) register read
(lldb) register read rsp rbp rax

# ดู memory
(lldb) memory read 0x... --count 16 --size 1
(lldb) me r -c 16 -s 1 0x...

# ดู types
(lldb) type lookup Node
(lldb) image lookup --type Node

# ค้นหา symbols
(lldb) image lookup --name sumList
(lldb) image list  # แสดง loaded modules
```

---

## Step 574: VS Code Debugger Integration

### ติดตั้ง Extensions

```json
// .vscode/extensions.json - recommended extensions
{
    "recommendations": [
        "ziglang.vscode-zig",
        "vadimcn.vscode-lldb",
        "ms-vscode.cpptools"
    ]
}
```

### Launch Configuration

```json
// .vscode/launch.json - debug configuration
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Zig (LLDB)",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/zig-out/bin/main",
            "args": [],
            "cwd": "${workspaceFolder}",
            "stopOnEntry": false,
            "preLaunchTask": "zig build debug"
        },
        {
            "name": "Debug Zig (GDB)",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/zig-out/bin/main",
            "args": [],
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "zig build debug"
        },
        {
            "name": "Debug Test",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/zig-out/bin/main-test",
            "args": [],
            "preLaunchTask": "zig test debug"
        }
    ]
}
```

### Tasks Configuration

```json
// .vscode/tasks.json - build tasks
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "zig build debug",
            "type": "shell",
            "command": "zig",
            "args": ["build", "-Doptimize=Debug"],
            "group": "build",
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false
            },
            "problemMatcher": {
                "pattern": {
                    "regexp": "^(.+):(\\d+):(\\d+): (error|warning|note): (.+)$",
                    "file": 1,
                    "line": 2,
                    "column": 3,
                    "severity": 4,
                    "message": 5
                }
            }
        },
        {
            "label": "zig test debug",
            "type": "shell",
            "command": "zig",
            "args": ["test", "src/main.zig", "--test-runner", ""],
            "group": "test"
        }
    ]
}
```

### VS Code Settings สำหรับ Zig

```json
// .vscode/settings.json
{
    "zig.buildOnSave": true,
    "zig.buildOption": "build",
    "zig.buildArgs": ["-Doptimize=Debug"],
    "zig.zls.path": "/usr/local/bin/zls",
    "editor.formatOnSave": true,
    "[zig]": {
        "editor.defaultFormatter": "ziglang.vscode-zig",
        "editor.tabSize": 4,
        "editor.insertSpaces": true
    }
}
```

---

## Step 575: Memory Debugging ด้วย Valgrind

### Valgrind Basics

```bash
# ติดตั้ง Valgrind
sudo apt-get install valgrind

# Run program ด้วย Valgrind
valgrind ./main

# Memcheck (ตรวจหา memory errors)
valgrind --tool=memcheck --leak-check=full ./main

# Verbose output
valgrind --tool=memcheck \
         --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --verbose \
         ./main 2>&1 | tee valgrind_report.txt
```

### ตัวอย่าง Memory Bugs

```zig
// memory_bugs.zig - ตัวอย่าง bugs สำหรับ Valgrind
const std = @import("std");

// Bug 1: Memory leak
fn leakyFunction(allocator: std.mem.Allocator) !void {
    const buf = try allocator.alloc(u8, 100);
    // ลืม free! - Valgrind จะตรวจพบ
    _ = buf;
}

// Bug 2: Use after free (ใน unsafe mode)
fn useAfterFreeDemo(allocator: std.mem.Allocator) !void {
    const ptr = try allocator.create(i32);
    ptr.* = 42;
    allocator.destroy(ptr);
    // Accessing freed memory - dangerous!
    // std.debug.print("{}\n", .{ptr.*}); // UB!
}

// Bug 3: ถูกต้อง - ไม่มี leak
fn correctAllocation(allocator: std.mem.Allocator) !void {
    const buf = try allocator.alloc(u8, 100);
    defer allocator.free(buf);
    
    @memset(buf, 0);
    std.debug.print("Buffer allocated and will be freed\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    try leakyFunction(allocator);
    try correctAllocation(allocator);
}
```

### ตีความ Valgrind Output

```
==12345== HEAP SUMMARY:
==12345==     in use at exit: 100 bytes in 1 blocks
==12345==   total heap usage: 2 allocs, 1 frees, 200 bytes allocated
==12345==
==12345== LEAK SUMMARY:
==12345==    definitely lost: 100 bytes in 1 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 0 bytes in 0 blocks
==12345==         suppressed: 0 bytes in 0 blocks
==12345==
==12345== For counts of detected and suppressed errors, ...

ความหมาย:
- definitely lost: memory ที่ leak แน่นอน
- indirectly lost: memory ที่ leak ผ่าน pointer chain
- possibly lost: memory ที่อาจ leak (pointer arithmetic)
- still reachable: memory ที่ยังมี pointer ชี้อยู่ตอน exit
```

### Zig GeneralPurposeAllocator vs Valgrind

```zig
// gpa_demo.zig - ใช้ GPA ตรวจหา leaks
const std = @import("std");

pub fn main() !void {
    // GPA ตรวจหา memory bugs ใน Debug mode
    var gpa = std.heap.GeneralPurposeAllocator(.{
        .safety = true,          // เปิด safety checks
        .never_unmap = false,    // unmap เมื่อ free
        .retain_metadata = true, // เก็บ metadata สำหรับ debugging
    }){};
    defer {
        const result = gpa.deinit();
        if (result == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }
    
    const allocator = gpa.allocator();
    
    // Intentional leak สำหรับ demo
    const buf = try allocator.alloc(u8, 42);
    _ = buf;
    // ไม่ free - GPA จะตรวจพบตอน deinit
}
```

---

## Step 576: Address Sanitizer (ASan)

### เปิดใช้ ASan ใน Zig

```bash
# Build พร้อม ASan
zig build-exe main.zig -fsanitize=address

# ใน build.zig
# exe.sanitize_address = true;
```

### ASan ตรวจหา Errors เหล่านี้:

```
ASan ตรวจพบ:
├─ Stack buffer overflow/underflow
├─ Heap buffer overflow/underflow
├─ Use after free
├─ Use after return
├─ Use after scope
├─ Double free
└─ Memory leaks (ด้วย LeakSanitizer)
```

### ตัวอย่าง ASan Detection

```zig
// asan_demo.zig - ตัวอย่างที่ ASan ตรวจพบ
const std = @import("std");

pub fn main() void {
    // Stack buffer overflow
    var arr = [_]u8{ 1, 2, 3, 4, 5 };
    
    // Zig จะ compile-time error ส่วนใหญ่
    // แต่ runtime checks ผ่าน ASan
    
    // ใน unsafe code:
    const ptr: [*]u8 = &arr;
    
    // ASan จะตรวจพบการเข้าถึงนอก bounds
    // ptr[10] = 0xFF;  // ASan ERROR!
    
    std.debug.print("ASan demo complete\n", .{});
}
```

### Build Script พร้อม Sanitizers

```zig
// build.zig - เปิดใช้ sanitizers
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    
    // Debug build พร้อม sanitizers
    const exe = b.addExecutable(.{
        .name = "safe_app",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .Debug,
    });
    
    // เปิดใช้ Address Sanitizer
    exe.sanitize_address = true;
    
    // เปิดใช้ Thread Sanitizer
    // exe.sanitize_thread = true;  // ไม่ compatible กับ ASan
    
    b.installArtifact(exe);
}
```

### ตีความ ASan Output

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow
READ of size 1 at 0x... thread T0
    #0 0x... in main debug_example.zig:15
    #1 0x... in start

0x... is located 0 bytes to the right of 5-byte region
allocated by thread T0 here:
    #0 0x... in malloc (libasan.so.x)
    #1 0x... in main debug_example.zig:12

การอ่าน output:
- "heap-buffer-overflow": ประเภทของ error
- Stack trace: ตำแหน่งที่เกิด error
- Allocation trace: ที่ allocate memory ครั้งแรก
```

### ASan Options

```bash
# ตั้ง ASan options ผ่าน environment variable
ASAN_OPTIONS=detect_leaks=1:print_stats=1:abort_on_error=1 ./main

# Options ที่มีประโยชน์:
# detect_leaks=1        - ตรวจหา memory leaks
# abort_on_error=1      - abort เมื่อพบ error
# halt_on_error=0       - ทำต่อหลังพบ error (ดู errors ทั้งหมด)
# log_path=/tmp/asan    - บันทึก log ไปยังไฟล์
# print_stats=1         - แสดง statistics
```

---

## Step 577: Memory Sanitizer (MSan)

### Memory Sanitizer คืออะไร?

MSan ตรวจหาการใช้ uninitialized memory (ซึ่ง ASan ไม่ตรวจ):

```zig
// msan_demo.zig - uninitialized memory bugs
const std = @import("std");

pub fn buggyFunction() i32 {
    var x: i32 = undefined;  // Zig บอกว่า undefined แต่ค่าจริงไม่แน่นอน
    
    // ใน unsafe context, อาจใช้ค่า garbage
    // MSan จะตรวจพบการอ่าน uninitialized memory
    return x;  // MSan ERROR: uninitialized value
}

pub fn safeFunction() i32 {
    var x: i32 = 0;  // Initialize properly
    return x;
}
```

### ใช้ MSan ใน Zig

```bash
# Build พร้อม MSan
zig build-exe main.zig -fsanitize=memory

# MSan ต้องการ instrumented stdlib
# ใน Zig มักใช้ร่วมกับ musl libc
zig build-exe main.zig \
    -fsanitize=memory \
    -target x86_64-linux-musl
```

### MSan กับ Zig's undefined Behavior

```zig
// undefined_demo.zig - Zig's undefined ≠ uninitialized
const std = @import("std");

pub fn main() void {
    // ใน Zig, undefined ใน Debug mode จะถูกเติมด้วย 0xAA
    // เพื่อช่วยตรวจหา bugs
    var buf: [10]u8 = undefined;
    
    // Debug mode: buf[0] == 0xAA
    // Release mode: buf[0] == garbage
    
    // ควร initialize ก่อนใช้
    @memset(&buf, 0);
    std.debug.print("Buffer: {any}\n", .{buf});
    
    // หรือใช้ค่าที่กำหนด
    var value: i32 = undefined;
    value = 42;  // ต้อง assign ก่อนอ่าน
    std.debug.print("Value: {}\n", .{value});
}
```

---

## Step 578: Thread Sanitizer (TSan)

### TSan ตรวจหา Data Races

Data race เกิดเมื่อสอง threads เข้าถึง shared memory พร้อมกัน โดยไม่มี synchronization:

```zig
// tsan_demo.zig - data race example
const std = @import("std");

var shared_counter: i32 = 0;

fn incrementCounter(count: usize) !void {
    var i: usize = 0;
    while (i < count) : (i += 1) {
        // DATA RACE! ไม่มี synchronization
        shared_counter += 1;
    }
}

pub fn main() !void {
    const thread1 = try std.Thread.spawn(.{}, incrementCounter, .{1000});
    const thread2 = try std.Thread.spawn(.{}, incrementCounter, .{1000});
    
    thread1.join();
    thread2.join();
    
    // ผลลัพธ์ไม่แน่นอนเพราะ data race
    std.debug.print("Counter: {} (expected 2000)\n", .{shared_counter});
}
```

### แก้ไข Data Race ด้วย Mutex

```zig
// fixed_tsan.zig - แก้ data race
const std = @import("std");

var shared_counter: i32 = 0;
var mutex = std.Thread.Mutex{};

fn safeIncrement(count: usize) !void {
    var i: usize = 0;
    while (i < count) : (i += 1) {
        mutex.lock();
        defer mutex.unlock();
        shared_counter += 1;
    }
}

pub fn main() !void {
    const thread1 = try std.Thread.spawn(.{}, safeIncrement, .{1000});
    const thread2 = try std.Thread.spawn(.{}, safeIncrement, .{1000});
    
    thread1.join();
    thread2.join();
    
    std.debug.print("Counter: {} (expected 2000)\n", .{shared_counter});
}
```

### เปิดใช้ TSan

```bash
# Build พร้อม TSan
zig build-exe main.zig -fsanitize=thread

# ตั้ง TSan options
TSAN_OPTIONS=halt_on_error=1:log_path=/tmp/tsan ./main
```

### TSan Report ตัวอย่าง

```
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x... by thread T2:
    #0 incrementCounter tsan_demo.zig:10
    #1 Thread.spawn.closure ...

  Previous write of size 4 at 0x... by thread T1:
    #0 incrementCounter tsan_demo.zig:10
    #1 Thread.spawn.closure ...

  Location is global 'shared_counter' of size 4 at 0x...
    thread T1 started at:
    #0 main tsan_demo.zig:15
==================
```

---

## Step 579: Core Dumps การวิเคราะห์

### เปิดใช้ Core Dumps

```bash
# เปิด core dumps (Linux)
ulimit -c unlimited

# ตั้ง core dump location
echo '/tmp/core-%e-%p-%t' | sudo tee /proc/sys/kernel/core_pattern

# ตรวจสอบ current setting
ulimit -c
cat /proc/sys/kernel/core_pattern
```

### สร้าง Program ที่ Crash

```zig
// crash_demo.zig - program ที่จะ crash
const std = @import("std");

fn causeSegfault() void {
    // Null pointer dereference
    const ptr: *const i32 = @ptrFromInt(0);
    _ = ptr.*;  // SEGFAULT!
}

fn causeStackOverflow() i64 {
    return causeStackOverflow() + 1;  // Infinite recursion
}

pub fn main() void {
    std.debug.print("About to crash...\n", .{});
    
    // เลือก crash type:
    causeSegfault();
    // causeStackOverflow();
}
```

```bash
# Build พร้อม debug info
zig build-exe crash_demo.zig -O Debug

# Run เพื่อสร้าง core dump
./crash_demo
# Segmentation fault (core dumped)
# core file จะอยู่ใน /tmp/ หรือ current directory
```

### วิเคราะห์ Core Dump ด้วย GDB

```bash
# เปิด core dump
gdb ./crash_demo /tmp/core-crash_demo-12345-...

# หรือ
gdb -c /tmp/core-crash_demo-12345 ./crash_demo

# ดู backtrace
(gdb) bt
#0  0x0000000000401234 in causeSegfault () at crash_demo.zig:6
#1  0x0000000000401289 in main () at crash_demo.zig:18

# ดู registers ตอน crash
(gdb) info registers

# ดู stack frame
(gdb) frame 0
(gdb) info locals

# ดู memory รอบๆ crash
(gdb) x/20xb 0  # ดู null address area
```

### วิเคราะห์ Core Dump ด้วย LLDB

```bash
# เปิด core dump
lldb ./crash_demo -c /tmp/core-crash_demo-12345

# ดู backtrace
(lldb) bt
(lldb) thread backtrace all

# ดู registers
(lldb) register read
(lldb) register read rip rsp rbp

# ดู crash location
(lldb) frame info
(lldb) disassemble --frame
```

---

## Step 580: Debugging Crash Dumps

### ระบบ Crash Reporting

```zig
// crash_handler.zig - custom crash handler
const std = @import("std");
const builtin = @import("builtin");

pub fn installCrashHandler() void {
    const posix = std.posix;
    
    // Handler สำหรับ SIGSEGV
    posix.signal(posix.SIG.SEGV, handleCrash) catch {};
    // Handler สำหรับ SIGABRT
    posix.signal(posix.SIG.ABRT, handleCrash) catch {};
    // Handler สำหรับ SIGFPE
    posix.signal(posix.SIG.FPE, handleCrash) catch {};
}

fn handleCrash(sig: c_int) callconv(.C) void {
    const stderr = std.io.getStdErr().writer();
    
    stderr.print("\n=== CRASH DETECTED ===\n", .{}) catch {};
    stderr.print("Signal: {}\n", .{sig}) catch {};
    
    // Print stack trace
    var addresses: [32]usize = undefined;
    const count = std.debug.captureStackTrace(null, &addresses) catch 0;
    
    stderr.print("Stack trace ({} frames):\n", .{count}) catch {};
    
    // ใน production, บันทึก crash report
    writeCrashReport(sig, addresses[0..count]) catch {};
    
    // Exit
    std.process.exit(1);
}

fn writeCrashReport(sig: c_int, stack: []const usize) !void {
    const file = try std.fs.cwd().createFile("crash_report.txt", .{});
    defer file.close();
    
    const writer = file.writer();
    try writer.print("=== Crash Report ===\n", .{});
    try writer.print("Signal: {}\n", .{sig});
    try writer.print("Timestamp: {}\n", .{std.time.timestamp()});
    try writer.print("Stack trace:\n", .{});
    
    for (stack, 0..) |addr, i| {
        try writer.print("  [{d:0>2}] 0x{x:0>16}\n", .{ i, addr });
    }
}
```

### Stack Trace Symbolization

```zig
// symbolize.zig - แปลง addresses เป็น function names
const std = @import("std");
const debug = std.debug;

pub fn printStackTrace() !void {
    var stderr = std.io.getStdErr();
    
    // ดู current stack trace
    const debug_info = try debug.openSelfDebugInfo(std.heap.page_allocator);
    defer debug_info.deinit();
    
    var addresses: [20]usize = undefined;
    var stack_trace = std.builtin.StackTrace{
        .instruction_addresses = &addresses,
        .index = 0,
    };
    
    std.debug.captureStackTrace(null, &stack_trace) catch {};
    
    try debug.writeStackTrace(
        stack_trace,
        stderr.writer(),
        std.heap.page_allocator,
        debug_info,
        std.debug.detectTTY(stderr),
    );
}
```

### MinidumpWriter สำหรับ Windows Crashes

```zig
// windows_crash.zig - Windows crash handling
const std = @import("std");

pub fn setupWindowsCrashHandler() void {
    if (comptime @import("builtin").os.tag != .windows) return;
    
    _ = std.os.windows.SetUnhandledExceptionFilter(crashHandler);
}

fn crashHandler(info: *std.os.windows.EXCEPTION_POINTERS) callconv(.winapi) std.os.windows.LONG {
    const stderr = std.io.getStdErr().writer();
    
    stderr.print("\n=== Windows Exception ===\n", .{}) catch {};
    
    if (info.ExceptionRecord) |record| {
        stderr.print("Exception Code: 0x{x:0>8}\n", .{record.ExceptionCode}) catch {};
        stderr.print("Exception Address: 0x{x}\n", .{
            @intFromPtr(record.ExceptionAddress)
        }) catch {};
    }
    
    // Write minidump
    writeMinidump() catch {};
    
    return 0; // EXCEPTION_CONTINUE_SEARCH
}

fn writeMinidump() !void {
    // ใช้ MiniDumpWriteDump API
    std.debug.print("Writing crash dump...\n", .{});
}
```

### Automated Crash Analysis Script

```python
#!/usr/bin/env python3
# analyze_crash.py - วิเคราะห์ crash dump อัตโนมัติ

import subprocess
import sys
import re
from pathlib import Path

def analyze_core_dump(executable: str, core_file: str):
    """วิเคราะห์ core dump ด้วย GDB"""
    
    gdb_commands = """
set pagination off
backtrace full
info registers
x/20i $rip-10
quit
"""
    
    result = subprocess.run(
        ['gdb', '-batch', '-ex', gdb_commands, executable, core_file],
        capture_output=True,
        text=True
    )
    
    output = result.stdout + result.stderr
    
    print("=== Crash Analysis ===")
    print(f"Executable: {executable}")
    print(f"Core dump: {core_file}")
    print()
    
    # ดึง backtrace
    bt_match = re.search(r'(#\d+.*?)(?=\n\n|\Z)', output, re.DOTALL)
    if bt_match:
        print("Backtrace:")
        print(bt_match.group(1))
    
    # ดึง crash address
    rip_match = re.search(r'rip\s+0x([0-9a-f]+)', output)
    if rip_match:
        print(f"\nCrash at address: 0x{rip_match.group(1)}")
    
    return output

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(f"Usage: {sys.argv[0]} <executable> <core_dump>")
        sys.exit(1)
    
    analyze_core_dump(sys.argv[1], sys.argv[2])
```

### Debugging Best Practices สรุป

```
Debugging Workflow ที่แนะนำ:
┌────────────────────────────────────────────────────────┐
│ 1. ก่อน Bug เกิด (Prevention)                         │
│    ├─ Build ด้วย Debug mode ระหว่าง development       │
│    ├─ ใช้ GPA (GeneralPurposeAllocator)               │
│    ├─ เปิด Runtime safety checks ของ Zig             │
│    └─ เขียน tests                                     │
│                                                       │
│ 2. เมื่อ Bug เกิด (Detection)                         │
│    ├─ อ่าน error message และ stack trace              │
│    ├─ ใช้ std.debug.print สำหรับ quick debugging     │
│    ├─ ใช้ GDB/LLDB สำหรับ interactive debugging      │
│    └─ ใช้ Valgrind/ASan สำหรับ memory issues         │
│                                                       │
│ 3. Memory Issues                                      │
│    ├─ Leak: Valgrind หรือ GPA.deinit()               │
│    ├─ Overflow: ASan                                  │
│    ├─ Uninitialized: MSan                             │
│    └─ Thread issues: TSan                             │
│                                                       │
│ 4. Crash Analysis                                     │
│    ├─ เปิด core dumps                                │
│    ├─ ใช้ GDB/LLDB กับ core file                     │
│    └─ ดู backtrace และ registers                     │
└────────────────────────────────────────────────────────┘
```

---

**บทสรุป Part 58:**

ในบทนี้เราได้เรียนรู้เครื่องมือ debugging สำหรับ Zig:

1. **Step 571**: GDB setup และ configuration สำหรับ Zig programs
2. **Step 572**: GDB commands ที่จำเป็น: breakpoints, stepping, inspection
3. **Step 573**: LLDB setup และ commands เปรียบเทียบกับ GDB
4. **Step 574**: VS Code debugger integration ด้วย launch.json
5. **Step 575**: Valgrind สำหรับ memory leak detection
6. **Step 576**: ASan ตรวจหา buffer overflows และ use-after-free
7. **Step 577**: MSan ตรวจหา uninitialized memory usage
8. **Step 578**: TSan ตรวจหา data races ใน multi-threaded programs
9. **Step 579**: Core dumps และการวิเคราะห์ crash
10. **Step 580**: Crash reporting และ automated analysis

การมีทักษะ debugging ที่ดีจะช่วยลดเวลาในการแก้ไข bugs ได้อย่างมาก และทำให้โปรแกรม Zig มีความน่าเชื่อถือมากขึ้น
