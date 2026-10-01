# Part 53: Inline Assembly ใน Zig
## Steps 521-530

Inline Assembly ช่วยให้นักพัฒนาสามารถเขียน assembly code โดยตรงภายใน Zig code ได้ ทำให้สามารถใช้ hardware features พิเศษ ทำ low-level optimization หรือเขียน system code ได้อย่างแม่นยำ

---

## Step 521: asm Statement ใน Zig

### Syntax พื้นฐาน

```zig
// รูปแบบของ asm statement:
// asm [volatile] (assembly_template
//     : output_constraints
//     : input_constraints
//     : clobber_list
// );

const std = @import("std");

pub fn basicAsm() void {
    // Assembly ง่ายๆ ที่ไม่มี inputs/outputs
    asm volatile ("nop"); // No Operation
    
    // หลาย instructions
    asm volatile (
        \\nop
        \\nop
        \\nop
    );
}

// asm volatile vs asm:
// - volatile: บอก compiler อย่า optimize out (สำหรับ side effects)
// - ไม่มี volatile: compiler อาจ optimize out ถ้าไม่ใช้ output
pub fn main() void {
    basicAsm();
    std.debug.print("Basic ASM done!\n", .{});
}
```

### asm ที่มี Output

```zig
const std = @import("std");

// อ่านค่าจาก register
pub fn getStackPointer() usize {
    return asm ("mov %[sp], %%rsp"
        : [sp] "=r" (-> usize)
    );
}

// ตัวอย่าง: simple add
pub fn addAsm(a: u64, b: u64) u64 {
    return asm ("add %[b], %[a]"
        : [a] "=r" (-> u64),
        : [a] "0" (a),
          [b] "r" (b),
    );
}

pub fn main() void {
    const sp = getStackPointer();
    std.debug.print("Stack pointer: 0x{x}\n", .{sp});
    
    const result = addAsm(10, 20);
    std.debug.print("10 + 20 = {}\n", .{result});
}
```

### Multi-output asm

```zig
// asm ที่มีหลาย outputs (ต้องใช้กับ variable outputs)
pub fn divMod(dividend: u64, divisor: u64) struct { quotient: u64, remainder: u64 } {
    var quotient: u64 = undefined;
    var remainder: u64 = undefined;
    
    asm volatile ("div %[divisor]"
        : [quotient] "={rax}" (quotient),
          [remainder] "={rdx}" (remainder),
        : [dividend_lo] "{rax}" (dividend),
          [dividend_hi] "{rdx}" (@as(u64, 0)),
          [divisor] "r" (divisor),
    );
    
    return .{ .quotient = quotient, .remainder = remainder };
}
```

---

## Step 522: Input/Output Constraints

### Output Constraints

```zig
// Output constraint format: [name] "constraint" (variable)
// Common output constraints:
// "=r"  - general purpose register (write)
// "=m"  - memory location (write)
// "+r"  - general purpose register (read & write)
// "=&r" - early clobber register (written before inputs are read)

pub fn outputConstraints() u64 {
    // "=r" - compiler เลือก register ที่เหมาะสม
    const result = asm volatile ("mov $42, %[out]"
        : [out] "=r" (-> u64),
    );
    return result;
}

// "+r" - register ที่เป็นทั้ง input และ output
pub fn increment(value: u64) u64 {
    var result = value;
    asm volatile ("inc %[val]"
        : [val] "+r" (result),
    );
    return result;
}

// "=m" - เขียนไปยัง memory
pub fn writeToMemory(ptr: *u64, value: u64) void {
    asm volatile ("mov %[val], %[dst]"
        : [dst] "=m" (ptr.*),
        : [val] "r" (value),
    );
}
```

### Input Constraints

```zig
// Input constraint format: [name] "constraint" (expression)
// Common input constraints:
// "r"  - general purpose register
// "m"  - memory location
// "i"  - immediate integer constant
// "n"  - immediate integer (with known value at compile time)
// "0"  - same register as output 0

// "r" constraint
pub fn addRegisters(a: u64, b: u64) u64 {
    return asm ("lea (%[a], %[b]), %[result]"
        : [result] "=r" (-> u64),
        : [a] "r" (a),
          [b] "r" (b),
    );
}

// "i" constraint สำหรับ immediate values
pub fn shiftLeft(value: u64, comptime shift: u6) u64 {
    return asm ("shl %[shift], %[val]"
        : [val] "=r" (-> u64),
        : [val] "0" (value),
          [shift] "n" (shift),
    );
}

// "m" constraint สำหรับ memory
pub fn atomicIncrement(ptr: *u64) void {
    asm volatile ("lock incq %[val]"
        : [val] "+m" (ptr.*),
    );
}

// "0" constraint - tie กับ output ตัวแรก
pub fn xorSwap(a: *u64, b: *u64) void {
    asm volatile (
        \\xor %[b], %[a]
        \\xor %[a], %[b]
        \\xor %[b], %[a]
        : [a] "+r" (a.*),
          [b] "+r" (b.*),
    );
}
```

### Named Constraints และ Specific Registers

```zig
// ระบุ specific register
pub fn callSyscall(number: u64, arg1: u64) u64 {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> u64),
        : [number] "{rax}" (number),
          [arg1] "{rdi}" (arg1),
        : "rcx", "r11", "memory"
    );
}

// x86-64 specific constraints
pub fn readFlags() u64 {
    var flags: u64 = undefined;
    asm volatile (
        \\pushfq
        \\pop %[flags]
        : [flags] "=r" (flags),
        :
        : "memory"
    );
    return flags;
}

pub fn setFlags(flags: u64) void {
    asm volatile (
        \\push %[flags]
        \\popfq
        :
        : [flags] "r" (flags),
        : "cc", "memory"
    );
}
```

---

## Step 523: Clobber List

### Clobbers คืออะไร?

Clobber list บอก compiler ว่า assembly code จะทำลาย (clobber) อะไรบ้าง นอกจาก output ที่ระบุไว้แล้ว

```zig
// Clobber types:
// "memory" - asm อาจ read/write memory arbitrarily
// "cc"     - condition codes (flags register) ถูก modify
// "rax", "rbx", ... - specific registers ถูก clobber
// "xmm0", "ymm0", ... - SIMD registers ถูก clobber

pub fn clobberExamples(a: u64) u64 {
    var result: u64 = undefined;
    
    // "memory" clobber - บอก compiler ว่า memory อาจ change
    asm volatile (
        \\mov %[input], %[output]
        \\nop
        : [output] "=r" (result),
        : [input] "r" (a),
        : "memory"  // memory barrier
    );
    
    return result;
}

// "cc" clobber - arithmetic ที่ set flags
pub fn checkCarry(a: u64, b: u64) bool {
    var carry: u8 = undefined;
    asm volatile (
        \\add %[b], %[a]
        \\setc %[carry]
        : [carry] "=r" (carry),
        : [a] "r" (a),
          [b] "r" (b),
        : "cc"
    );
    return carry != 0;
}

// ตัวอย่าง: function ที่ใช้ temporary registers
pub fn complexOp(x: u64, y: u64) u64 {
    return asm volatile (
        \\mov %[x], %%rax
        \\imul %[y], %%rax
        \\add %[x], %%rax
        : "={rax}" (-> u64),
        : [x] "r" (x),
          [y] "r" (y),
        : "cc"
    );
}
```

### Memory Barriers

```zig
// Full memory barrier
pub fn memoryBarrier() void {
    asm volatile ("mfence" ::: "memory");
}

// Load fence
pub fn loadFence() void {
    asm volatile ("lfence" ::: "memory");
}

// Store fence
pub fn storeFence() void {
    asm volatile ("sfence" ::: "memory");
}

// Compiler-only barrier (ไม่ใช่ CPU instruction)
pub fn compilerBarrier() void {
    asm volatile ("" ::: "memory");
}

// ตัวอย่างการใช้ barrier ใน lock-free code
pub fn atomicLoad(ptr: *const u64) u64 {
    const val = ptr.*;
    asm volatile ("" ::: "memory"); // Prevent reordering
    return val;
}

pub fn atomicStore(ptr: *u64, val: u64) void {
    asm volatile ("" ::: "memory"); // Prevent reordering
    ptr.* = val;
}
```

---

## Step 524: x86_64 Assembly Patterns

### Arithmetic Operations

```zig
const std = @import("std");

// 128-bit integer multiplication
pub fn mul128(a: u64, b: u64) struct { lo: u64, hi: u64 } {
    var lo: u64 = undefined;
    var hi: u64 = undefined;
    
    asm ("mulq %[b]"
        : [lo] "={rax}" (lo),
          [hi] "={rdx}" (hi),
        : [a] "{rax}" (a),
          [b] "r" (b),
    );
    
    return .{ .lo = lo, .hi = hi };
}

// Bit scan forward (find lowest set bit)
pub fn bsf(x: u64) u6 {
    return asm ("bsfq %[x], %[result]"
        : [result] "=r" (-> u6),
        : [x] "r" (x),
        : "cc"
    );
}

// Bit scan reverse (find highest set bit)
pub fn bsr(x: u64) u6 {
    return asm ("bsrq %[x], %[result]"
        : [result] "=r" (-> u6),
        : [x] "r" (x),
        : "cc"
    );
}

// Population count (count set bits)
pub fn popcnt(x: u64) u7 {
    return asm ("popcntq %[x], %[result]"
        : [result] "=r" (-> u7),
        : [x] "r" (x),
        : "cc"
    );
}

// Count leading zeros
pub fn clz(x: u64) u7 {
    return asm ("lzcntq %[x], %[result]"
        : [result] "=r" (-> u7),
        : [x] "r" (x),
        : "cc"
    );
}

pub fn main() void {
    const result = mul128(0xFFFFFFFFFFFFFFFF, 0xFFFFFFFFFFFFFFFF);
    std.debug.print("mul128: hi=0x{x}, lo=0x{x}\n", .{ result.hi, result.lo });
    
    std.debug.print("bsf(0b1010): {}\n", .{bsf(0b1010)});
    std.debug.print("bsr(0b1010): {}\n", .{bsr(0b1010)});
    std.debug.print("popcnt(0xFF): {}\n", .{popcnt(0xFF)});
    std.debug.print("clz(1): {}\n", .{clz(1)});
}
```

### String Operations

```zig
// Fast memory copy ด้วย rep movsb
pub fn fastMemCpy(dest: [*]u8, src: [*]const u8, len: usize) void {
    asm volatile (
        \\rep movsb
        :
        : [dest] "{rdi}" (dest),
          [src] "{rsi}" (src),
          [len] "{rcx}" (len),
        : "rdi", "rsi", "rcx", "memory"
    );
}

// Fast memset ด้วย rep stosb
pub fn fastMemSet(dest: [*]u8, value: u8, len: usize) void {
    asm volatile (
        \\rep stosb
        :
        : [dest] "{rdi}" (dest),
          [value] "{al}" (value),
          [len] "{rcx}" (len),
        : "rdi", "rcx", "memory"
    );
}

// Find byte ด้วย repne scasb
pub fn findByte(haystack: [*]const u8, needle: u8, max_len: usize) ?usize {
    var found: u8 = undefined;
    var remaining = max_len;
    var ptr = haystack;
    
    asm volatile (
        \\repne scasb
        : [remaining] "={rcx}" (remaining),
          [ptr] "={rdi}" (ptr),
          [found] "={al}" (found),
        : [haystack] "{rdi}" (ptr),
          [needle] "{al}" (needle),
          [len] "{rcx}" (remaining),
        : "cc", "memory"
    );
    
    if (found == needle) {
        return max_len - remaining - 1;
    }
    return null;
}
```

### Control Flow

```zig
// Conditional moves (cmov) - branchless
pub fn cmovMax(a: i64, b: i64) i64 {
    return asm volatile (
        \\cmp %[b], %[a]
        \\cmovl %[b], %[a]
        : [a] "+r" (-> i64),
        : [a] "0" (a),
          [b] "r" (b),
        : "cc"
    );
}

// Loop ใน assembly
pub fn sumLoop(data: [*]const u32, len: usize) u64 {
    return asm volatile (
        \\xor %%rax, %%rax
        \\xor %%rcx, %%rcx
        \\.loop:
        \\  movl (%[data], %%rcx, 4), %%edx
        \\  add %%rdx, %%rax
        \\  inc %%rcx
        \\  cmp %[len], %%rcx
        \\  jl .loop
        : "={rax}" (-> u64),
        : [data] "r" (data),
          [len] "r" (len),
        : "rcx", "rdx", "cc", "memory"
    );
}
```

---

## Step 525: ARM Assembly Patterns

### ARM64 (AArch64) Basics

```zig
const builtin = @import("builtin");
const std = @import("std");

// ARM64 instruction examples
// ใช้เฉพาะบน AArch64
pub fn arm64Examples() void {
    if (comptime builtin.cpu.arch != .aarch64) {
        std.debug.print("Not AArch64\n", .{});
        return;
    }
    
    // Simple move
    const val = asm ("mov %[out], #42"
        : [out] "=r" (-> u64),
    );
    std.debug.print("val: {}\n", .{val});
}

// ARM64 multiply-accumulate
pub fn mulAdd(a: u64, b: u64, c: u64) u64 {
    if (comptime builtin.cpu.arch != .aarch64) return a * b + c;
    
    return asm ("madd %[result], %[a], %[b], %[c]"
        : [result] "=r" (-> u64),
        : [a] "r" (a),
          [b] "r" (b),
          [c] "r" (c),
    );
}

// ARM64 bit count
pub fn countBitsArm(x: u64) u7 {
    if (comptime builtin.cpu.arch != .aarch64) {
        return @popCount(x);
    }
    
    // NEON approach for ARM64
    return asm (
        \\fmov d0, %[x]
        \\cnt v0.8b, v0.8b
        \\addv b0, v0.8b
        \\fmov %[result], s0
        : [result] "=r" (-> u7),
        : [x] "r" (x),
        : "v0"
    );
}
```

### ARM NEON Instructions

```zig
// ARM NEON vector operations
pub fn neonAdd(a: @Vector(4, f32), b: @Vector(4, f32)) @Vector(4, f32) {
    if (comptime @import("builtin").cpu.arch != .aarch64) {
        return a + b;
    }
    
    return asm (
        \\fadd v2.4s, v0.4s, v1.4s
        : "={v2}" (-> @Vector(4, f32)),
        : "{v0}" (a),
          "{v1}" (b),
    );
}

// NEON dot product
pub fn neonDot4(a: @Vector(4, f32), b: @Vector(4, f32)) f32 {
    if (comptime @import("builtin").cpu.arch != .aarch64) {
        return @reduce(.Add, a * b);
    }
    
    return asm (
        \\fmul v2.4s, v0.4s, v1.4s
        \\faddp v2.4s, v2.4s, v2.4s
        \\faddp s2, v2.2s
        : "={s2}" (-> f32),
        : "{v0}" (a),
          "{v1}" (b),
        : "v2"
    );
}
```

### Cross-Platform Assembly

```zig
const builtin = @import("builtin");

// Cross-platform pause/yield
pub fn cpuPause() void {
    switch (builtin.cpu.arch) {
        .x86_64, .x86 => asm volatile ("pause"),
        .aarch64 => asm volatile ("yield"),
        .riscv64 => asm volatile ("pause"),
        else => {}, // No-op for other architectures
    }
}

// Cross-platform fence
pub fn fullFence() void {
    switch (builtin.cpu.arch) {
        .x86_64, .x86 => asm volatile ("mfence" ::: "memory"),
        .aarch64 => asm volatile ("dmb sy" ::: "memory"),
        .riscv64 => asm volatile ("fence rw, rw" ::: "memory"),
        else => @fence(.seq_cst),
    }
}

// Cross-platform CLZ
pub fn countLeadingZeros(x: u64) u7 {
    return switch (builtin.cpu.arch) {
        .x86_64 => asm ("lzcntq %[x], %[result]"
            : [result] "=r" (-> u7),
            : [x] "r" (x),
            : "cc"
        ),
        .aarch64 => asm ("clz %[result], %[x]"
            : [result] "=r" (-> u7),
            : [x] "r" (x),
        ),
        else => @clz(x),
    };
}
```

---

## Step 526: CPU Feature Detection

### Runtime CPU Feature Detection

```zig
const std = @import("std");
const builtin = @import("builtin");

// Detect CPU features at runtime (x86_64)
pub const CpuFeatures = struct {
    sse2: bool = false,
    sse4_2: bool = false,
    avx: bool = false,
    avx2: bool = false,
    avx512f: bool = false,
    bmi2: bool = false,
    popcnt: bool = false,
    aes: bool = false,
    
    pub fn detect() CpuFeatures {
        if (comptime builtin.cpu.arch != .x86_64) {
            return .{};
        }
        
        var features = CpuFeatures{};
        
        // CPUID leaf 1
        var eax: u32 = undefined;
        var ebx: u32 = undefined;
        var ecx: u32 = undefined;
        var edx: u32 = undefined;
        
        cpuid(1, 0, &eax, &ebx, &ecx, &edx);
        
        features.sse2 = (edx >> 26) & 1 == 1;
        features.sse4_2 = (ecx >> 20) & 1 == 1;
        features.avx = (ecx >> 28) & 1 == 1;
        features.popcnt = (ecx >> 23) & 1 == 1;
        features.aes = (ecx >> 25) & 1 == 1;
        
        // CPUID leaf 7
        cpuid(7, 0, &eax, &ebx, &ecx, &edx);
        
        features.avx2 = (ebx >> 5) & 1 == 1;
        features.avx512f = (ebx >> 16) & 1 == 1;
        features.bmi2 = (ebx >> 8) & 1 == 1;
        
        return features;
    }
};

pub fn cpuid(leaf: u32, subleaf: u32, eax: *u32, ebx: *u32, ecx: *u32, edx: *u32) void {
    asm volatile (
        \\cpuid
        : [eax] "={eax}" (eax.*),
          [ebx] "={ebx}" (ebx.*),
          [ecx] "={ecx}" (ecx.*),
          [edx] "={edx}" (edx.*),
        : [leaf] "{eax}" (leaf),
          [subleaf] "{ecx}" (subleaf),
    );
}

pub fn main() void {
    const features = CpuFeatures.detect();
    std.debug.print("CPU Features:\n", .{});
    std.debug.print("  SSE2: {}\n", .{features.sse2});
    std.debug.print("  SSE4.2: {}\n", .{features.sse4_2});
    std.debug.print("  AVX: {}\n", .{features.avx});
    std.debug.print("  AVX2: {}\n", .{features.avx2});
    std.debug.print("  AVX-512: {}\n", .{features.avx512f});
    std.debug.print("  POPCNT: {}\n", .{features.popcnt});
    std.debug.print("  AES-NI: {}\n", .{features.aes});
}
```

### Dynamic Dispatch ตาม CPU Features

```zig
const std = @import("std");

// Function dispatch ตาม CPU features
pub const Processor = struct {
    process: *const fn([]u8) void,
    
    pub fn init() Processor {
        const features = CpuFeatures.detect();
        
        if (features.avx2) {
            return .{ .process = processAVX2 };
        } else if (features.sse4_2) {
            return .{ .process = processSSE4 };
        } else {
            return .{ .process = processScalar };
        }
    }
};

fn processAVX2(data: []u8) void {
    std.debug.print("Processing with AVX2 ({} bytes)\n", .{data.len});
    // AVX2 implementation
}

fn processSSE4(data: []u8) void {
    std.debug.print("Processing with SSE4 ({} bytes)\n", .{data.len});
    // SSE4 implementation
}

fn processScalar(data: []u8) void {
    std.debug.print("Processing scalar ({} bytes)\n", .{data.len});
    // Scalar implementation
}
```

---

## Step 527: Atomic Operations ด้วย Assembly

### Compare-and-Swap (CAS)

```zig
const std = @import("std");

// CAS ด้วย assembly (x86_64)
pub fn compareAndSwap(ptr: *u64, expected: u64, new_val: u64) bool {
    var success: bool = undefined;
    
    asm volatile (
        \\lock cmpxchgq %[new], %[dst]
        \\setz %[success]
        : [success] "=r" (success),
          [dst] "+m" (ptr.*),
        : [expected] "{rax}" (expected),
          [new] "r" (new_val),
        : "cc"
    );
    
    return success;
}

// Atomic fetch-and-add
pub fn fetchAdd(ptr: *u64, delta: u64) u64 {
    return asm volatile (
        \\lock xaddq %[delta], %[dst]
        : [delta] "+r" (-> u64),
          [dst] "+m" (ptr.*),
        : [delta] "0" (delta),
        : "cc"
    );
}

// Atomic fetch-and-or
pub fn fetchOr(ptr: *u64, mask: u64) u64 {
    var old: u64 = ptr.*;
    while (true) {
        const new_val = old | mask;
        if (compareAndSwap(ptr, old, new_val)) {
            return old;
        }
        old = ptr.*;
    }
}

// Spin lock implementation
pub const SpinLock = struct {
    state: u64 = 0,
    
    pub fn lock(self: *SpinLock) void {
        while (!compareAndSwap(&self.state, 0, 1)) {
            // Spin with PAUSE hint
            asm volatile ("pause");
        }
    }
    
    pub fn unlock(self: *SpinLock) void {
        asm volatile ("" ::: "memory"); // Compiler fence
        self.state = 0;
    }
    
    pub fn tryLock(self: *SpinLock) bool {
        return compareAndSwap(&self.state, 0, 1);
    }
};

pub fn main() void {
    var counter: u64 = 0;
    
    _ = fetchAdd(&counter, 5);
    _ = fetchAdd(&counter, 3);
    std.debug.print("Counter: {}\n", .{counter}); // 8
    
    const success = compareAndSwap(&counter, 8, 100);
    std.debug.print("CAS success: {}, counter: {}\n", .{ success, counter });
    
    var lock = SpinLock{};
    lock.lock();
    // Critical section
    counter += 1;
    lock.unlock();
    
    std.debug.print("Final counter: {}\n", .{counter});
}
```

### Read-Modify-Write Operations

```zig
// Atomic exchange
pub fn exchange(ptr: *u64, new_val: u64) u64 {
    return asm volatile (
        \\xchgq %[new], %[dst]
        : [new] "+r" (-> u64),
          [dst] "+m" (ptr.*),
        : [new] "0" (new_val),
        : "cc", "memory"
    );
}

// Atomic bit test and set
pub fn bitTestAndSet(ptr: *u64, bit: u6) bool {
    var carry: bool = undefined;
    asm volatile (
        \\lock btsq %[bit], %[dst]
        \\setc %[carry]
        : [carry] "=r" (carry),
          [dst] "+m" (ptr.*),
        : [bit] "r" (@as(u64, bit)),
        : "cc"
    );
    return carry;
}

// Atomic bit test and clear
pub fn bitTestAndClear(ptr: *u64, bit: u6) bool {
    var carry: bool = undefined;
    asm volatile (
        \\lock btrq %[bit], %[dst]
        \\setc %[carry]
        : [carry] "=r" (carry),
          [dst] "+m" (ptr.*),
        : [bit] "r" (@as(u64, bit)),
        : "cc"
    );
    return carry;
}
```

---

## Step 528: System Calls ด้วย Assembly

### Linux System Calls (x86_64)

```zig
const std = @import("std");
const builtin = @import("builtin");

// Linux syscall numbers
const SYS_READ = 0;
const SYS_WRITE = 1;
const SYS_OPEN = 2;
const SYS_CLOSE = 3;
const SYS_EXIT = 60;
const SYS_GETPID = 39;
const SYS_GETTIME = 228;

// Generic syscall wrapper
pub fn syscall0(number: u64) u64 {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> u64),
        : [number] "{rax}" (number),
        : "rcx", "r11", "memory"
    );
}

pub fn syscall1(number: u64, a1: u64) u64 {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> u64),
        : [number] "{rax}" (number),
          [a1] "{rdi}" (a1),
        : "rcx", "r11", "memory"
    );
}

pub fn syscall2(number: u64, a1: u64, a2: u64) u64 {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> u64),
        : [number] "{rax}" (number),
          [a1] "{rdi}" (a1),
          [a2] "{rsi}" (a2),
        : "rcx", "r11", "memory"
    );
}

pub fn syscall3(number: u64, a1: u64, a2: u64, a3: u64) u64 {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> u64),
        : [number] "{rax}" (number),
          [a1] "{rdi}" (a1),
          [a2] "{rsi}" (a2),
          [a3] "{rdx}" (a3),
        : "rcx", "r11", "memory"
    );
}

// Get process ID
pub fn getpid() u32 {
    return @truncate(syscall0(SYS_GETPID));
}

// Write to file descriptor
pub fn write(fd: u32, buf: []const u8) usize {
    return @bitCast(syscall3(
        SYS_WRITE,
        fd,
        @intFromPtr(buf.ptr),
        buf.len,
    ));
}

// Exit process
pub fn exit(code: u8) noreturn {
    _ = syscall1(SYS_EXIT, code);
    unreachable;
}

pub fn main() void {
    const pid = getpid();
    std.debug.print("PID: {}\n", .{pid});
    
    // Direct write syscall
    const msg = "Hello from syscall!\n";
    _ = write(1, msg); // stdout = fd 1
}
```

### System Call Error Handling

```zig
const SyscallError = error{
    EPERM,
    ENOENT,
    EINTR,
    EIO,
    EBADF,
    EAGAIN,
    ENOMEM,
    EACCES,
    Unknown,
};

pub fn checkSyscallResult(result: u64) SyscallError!u64 {
    // Linux syscalls return negative errno on error
    const signed: i64 = @bitCast(result);
    if (signed >= 0) return result;
    
    const errno: u64 = @bitCast(-signed);
    return switch (errno) {
        1 => error.EPERM,
        2 => error.ENOENT,
        4 => error.EINTR,
        5 => error.EIO,
        9 => error.EBADF,
        11 => error.EAGAIN,
        12 => error.ENOMEM,
        13 => error.EACCES,
        else => error.Unknown,
    };
}
```

---

## Step 529: CPUID และ CPU Info

### CPUID Instruction

```zig
const std = @import("std");
const builtin = @import("builtin");

pub const CpuInfo = struct {
    vendor: [12]u8,
    brand: [48]u8,
    max_basic_leaf: u32,
    max_extended_leaf: u32,
    
    pub fn init() CpuInfo {
        var info = CpuInfo{
            .vendor = undefined,
            .brand = undefined,
            .max_basic_leaf = 0,
            .max_extended_leaf = 0,
        };
        
        if (comptime builtin.cpu.arch != .x86_64) {
            return info;
        }
        
        var eax: u32 = undefined;
        var ebx: u32 = undefined;
        var ecx: u32 = undefined;
        var edx: u32 = undefined;
        
        // Leaf 0: Vendor string + max leaf
        cpuid(0, 0, &eax, &ebx, &ecx, &edx);
        info.max_basic_leaf = eax;
        
        // Vendor ID: ebx, edx, ecx (in that order!)
        @memcpy(info.vendor[0..4], std.mem.asBytes(&ebx));
        @memcpy(info.vendor[4..8], std.mem.asBytes(&edx));
        @memcpy(info.vendor[8..12], std.mem.asBytes(&ecx));
        
        // Extended leaves for brand string
        cpuid(0x80000000, 0, &eax, &ebx, &ecx, &edx);
        info.max_extended_leaf = eax;
        
        if (info.max_extended_leaf >= 0x80000004) {
            // Brand string is in leaves 0x80000002, 0x80000003, 0x80000004
            var brand_parts: [3][4]u32 = undefined;
            
            for (0..3) |i| {
                cpuid(0x80000002 + @as(u32, @intCast(i)), 0,
                    &brand_parts[i][0], &brand_parts[i][1],
                    &brand_parts[i][2], &brand_parts[i][3]);
            }
            
            @memcpy(&info.brand, std.mem.asBytes(&brand_parts));
        }
        
        return info;
    }
    
    pub fn printVendor(self: CpuInfo) void {
        std.debug.print("Vendor: {s}\n", .{self.vendor});
    }
    
    pub fn printBrand(self: CpuInfo) void {
        // Brand string อาจมี null terminator หรือ spaces
        var end: usize = self.brand.len;
        while (end > 0 and (self.brand[end - 1] == 0 or self.brand[end - 1] == ' ')) {
            end -= 1;
        }
        std.debug.print("CPU: {s}\n", .{self.brand[0..end]});
    }
};

// Cache information
pub fn getCacheInfo() void {
    if (comptime builtin.cpu.arch != .x86_64) return;
    
    var eax: u32 = undefined;
    var ebx: u32 = undefined;
    var ecx: u32 = undefined;
    var edx: u32 = undefined;
    
    // Leaf 4: Cache info
    var sub_leaf: u32 = 0;
    while (true) : (sub_leaf += 1) {
        cpuid(4, sub_leaf, &eax, &ebx, &ecx, &edx);
        
        const cache_type = eax & 0x1F;
        if (cache_type == 0) break; // No more caches
        
        const level = (eax >> 5) & 0x7;
        const line_size = (ebx & 0xFFF) + 1;
        const partitions = ((ebx >> 12) & 0x3FF) + 1;
        const ways = ((ebx >> 22) & 0x3FF) + 1;
        const sets = ecx + 1;
        const size = ways * partitions * line_size * sets;
        
        const cache_type_name = switch (cache_type) {
            1 => "Data",
            2 => "Instruction",
            3 => "Unified",
            else => "Unknown",
        };
        
        std.debug.print("L{} {} Cache: {} bytes (line={}, ways={}, sets={})\n", .{
            level, cache_type_name, size, line_size, ways, sets
        });
    }
}

pub fn main() void {
    const info = CpuInfo.init();
    info.printVendor();
    info.printBrand();
    
    std.debug.print("\nCache Information:\n", .{});
    getCacheInfo();
}
```

---

## Step 530: Assembly Optimization Patterns

### Optimized Memory Copy

```zig
const std = @import("std");

// AVX2 memory copy (256-bit at a time)
pub fn avx2MemCpy(dest: [*]u8, src: [*]const u8, len: usize) void {
    var i: usize = 0;
    
    // Copy 32 bytes at a time with AVX2
    while (i + 32 <= len) : (i += 32) {
        asm volatile (
            \\vmovdqu (%[src]), %%ymm0
            \\vmovdqu %%ymm0, (%[dst])
            :
            : [src] "r" (src + i),
              [dst] "r" (dest + i),
            : "ymm0", "memory"
        );
    }
    
    // Handle remaining bytes
    while (i < len) : (i += 1) {
        dest[i] = src[i];
    }
}

// Non-temporal (streaming) copy สำหรับ large buffers
pub fn streamingCopy(dest: [*]u8, src: [*]const u8, len: usize) void {
    var i: usize = 0;
    
    while (i + 16 <= len) : (i += 16) {
        asm volatile (
            \\movdqu (%[src]), %%xmm0
            \\movntdq %%xmm0, (%[dst])
            :
            : [src] "r" (src + i),
              [dst] "r" (dest + i),
            : "xmm0", "memory"
        );
    }
    
    // Flush store buffer
    asm volatile ("sfence");
    
    while (i < len) : (i += 1) {
        dest[i] = src[i];
    }
}
```

### Hash Function Optimization ด้วย CRC32

```zig
// CRC32 hardware acceleration (requires SSE4.2)
pub fn crc32c(data: []const u8) u32 {
    var crc: u32 = 0xFFFFFFFF;
    var i: usize = 0;
    
    // Process 8 bytes at a time
    while (i + 8 <= data.len) : (i += 8) {
        const chunk: u64 = @as(*const u64, @ptrCast(@alignCast(data[i..].ptr))).*;
        asm volatile (
            \\crc32q %[chunk], %[crc]
            : [crc] "+r" (crc),
            : [chunk] "r" (chunk),
        );
    }
    
    // Process remaining bytes
    while (i < data.len) : (i += 1) {
        asm volatile (
            \\crc32b %[byte], %[crc]
            : [crc] "+r" (crc),
            : [byte] "r" (@as(u32, data[i])),
        );
    }
    
    return crc ^ 0xFFFFFFFF;
}
```

### Optimized Sorting with SIMD/Assembly

```zig
// Branchless insertion sort ด้วย conditional moves
pub fn branchlessInsertSort(arr: []i32) void {
    var i: usize = 1;
    while (i < arr.len) : (i += 1) {
        const key = arr[i];
        var j: usize = i;
        
        while (j > 0) : (j -= 1) {
            const prev = arr[j - 1];
            const should_swap = prev > key;
            
            // Branchless: conditional move
            arr[j] = if (should_swap) prev else arr[j];
            arr[j - 1] = if (should_swap) key else arr[j - 1];
            
            if (!should_swap) break;
        }
    }
}

pub fn main() void {
    std.debug.print("=== Inline Assembly Demo ===\n\n", .{});
    
    // Test atomic operations
    var counter: u64 = 0;
    _ = fetchAdd(&counter, 10);
    _ = fetchAdd(&counter, 5);
    std.debug.print("Atomic counter: {}\n", .{counter});
    
    // Test bit operations
    std.debug.print("popcnt(255): {}\n", .{popcnt(255)});
    std.debug.print("bsr(1024): {}\n", .{bsr(1024)});
    
    // Test hash
    const msg = "Hello, World!";
    const hash = crc32c(msg);
    std.debug.print("CRC32C('{s}'): 0x{x}\n", .{ msg, hash });
    
    // Test sort
    var arr = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };
    branchlessInsertSort(&arr);
    std.debug.print("Sorted: {any}\n", .{arr});
}
```

---

## สรุป Part 53

ในบทนี้เราได้เรียนรู้:

1. **asm Statement** - syntax พื้นฐานของ inline assembly ใน Zig
2. **Input/Output Constraints** - การส่งและรับค่าระหว่าง Zig และ assembly
3. **Clobber List** - บอก compiler ว่า assembly ทำลายอะไรบ้าง
4. **x86_64 Patterns** - multiplication, bit ops, string ops
5. **ARM Patterns** - AArch64 และ NEON instructions
6. **CPU Feature Detection** - ตรวจสอบ CPU capabilities ขณะรันไทม์
7. **Atomic Operations** - CAS, fetch-and-add, spin locks
8. **System Calls** - เรียก Linux system calls โดยตรง
9. **CPUID** - อ่านข้อมูล CPU อย่างละเอียด
10. **Optimization Patterns** - memcpy, CRC32, sorting

> **คำเตือน**: Inline assembly เป็น platform-specific เสมอ ใช้เฉพาะเมื่อจำเป็นจริงๆ และมักควรมี fallback สำหรับ platform อื่นๆ
