# Part 54: Embedded Systems ด้วย Zig
## Steps 531-540

Zig เป็นภาษาที่ยอดเยี่ยมสำหรับ embedded systems development เพราะมีขนาดเล็ก ไม่มี runtime overhead และสามารถ target hardware โดยตรงได้ ในบทนี้เราจะเรียนรู้การเขียน firmware สำหรับ microcontrollers

---

## Step 531: Zig สำหรับ Embedded Systems

### ทำไมต้องใช้ Zig สำหรับ Embedded?

```
เปรียบเทียบ:
┌────────────────┬──────────┬──────────┬──────────┬──────────┐
│ Feature        │ C        │ C++      │ Rust     │ Zig      │
├────────────────┼──────────┼──────────┼──────────┼──────────┤
│ No GC          │ ✓        │ ✓        │ ✓        │ ✓        │
│ No runtime     │ ✓        │ ✗*       │ ✗*       │ ✓        │
│ No stdlib dep  │ ✗*       │ ✗        │ ✗*       │ ✓        │
│ C interop      │ native   │ native   │ complex  │ native   │
│ comptime       │ limited  │ template │ macro    │ powerful │
│ Safety         │ minimal  │ minimal  │ strong   │ configurable│
└────────────────┴──────────┴──────────┴──────────┴──────────┘
* สามารถทำได้แต่ต้องตั้งค่าเพิ่มเติม
```

```zig
// Zig สำหรับ embedded ไม่ต้องการ OS หรือ standard library
// เพียงตั้งค่า target และ freestanding

// build.zig สำหรับ STM32F103 (Cortex-M3)
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Target: ARM Cortex-M3
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .thumb,
        .cpu_model = .{ .explicit = &std.Target.arm.cpu.cortex_m3 },
        .os_tag = .freestanding,
        .abi = .eabi,
    });
    
    const optimize = b.standardOptimizeOption(.{ .preferred_optimize_mode = .ReleaseSmall });
    
    const firmware = b.addExecutable(.{
        .name = "firmware",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    
    // Link ด้วย linker script
    firmware.setLinkerScriptPath(b.path("linker.ld"));
    
    b.installArtifact(firmware);
    
    // สร้าง .bin file สำหรับ flash
    const objcopy = b.addObjCopy(firmware.getEmittedBin(), .{
        .format = .bin,
    });
    
    const install_bin = b.addInstallBinFile(
        objcopy.getOutput(),
        "firmware.bin"
    );
    
    b.default_step.dependOn(&install_bin.step);
}
```

### Zig Freestanding Environment

```zig
// ใน freestanding mode ไม่มี std library features บางอย่าง
// ต้องประกาศเองหรือใช้ built-ins

// ไม่ใช้ std.debug.print ใน freestanding
// ใช้ UART หรือ semihosting แทน

// Panic handler ที่จำเป็น
pub fn panic(msg: []const u8, _: ?*std.builtin.StackTrace, _: ?usize) noreturn {
    // ใน embedded: อาจ blink LED หรือ log ผ่าน UART
    _ = msg;
    while (true) {
        // Halt
        asm volatile ("bkpt #0");
    }
}

const std = @import("std");
```

---

## Step 532: Bare-Metal Programming

### Startup Code

```zig
// src/startup.zig - Cortex-M startup code
const std = @import("std");

// Vector table สำหรับ Cortex-M
export const vector_table linksection(".vector_table") = [_]?*const fn() callconv(.C) void{
    // Stack pointer (initial value)
    @ptrCast(&__stack_top),
    // Reset handler
    resetHandler,
    // NMI
    nmiHandler,
    // HardFault
    hardFaultHandler,
    // MemManage
    memManageHandler,
    // BusFault
    busFaultHandler,
    // UsageFault
    usageFaultHandler,
    // Reserved (4 entries)
    null, null, null, null,
    // SVC
    svcHandler,
    // DebugMon
    debugMonHandler,
    // Reserved
    null,
    // PendSV
    pendSvHandler,
    // SysTick
    sysTickHandler,
    // External interrupts (IRQs 0-67 for STM32F103)
    // ... (more entries)
};

// External symbols จาก linker script
extern var __stack_top: u32;
extern var __data_start: u32;
extern var __data_end: u32;
extern var __data_load: u32;
extern var __bss_start: u32;
extern var __bss_end: u32;

// Reset handler - entry point หลัง power on/reset
export fn resetHandler() callconv(.C) void {
    // 1. Copy initialized data จาก flash ไปยัง RAM
    const data_start: [*]u8 = @ptrCast(&__data_start);
    const data_end: [*]const u8 = @ptrCast(&__data_end);
    const data_load: [*]const u8 = @ptrCast(&__data_load);
    const data_len = @intFromPtr(data_end) - @intFromPtr(data_start);
    
    @memcpy(data_start[0..data_len], data_load[0..data_len]);
    
    // 2. Zero-initialize BSS section
    const bss_start: [*]u8 = @ptrCast(&__bss_start);
    const bss_end: [*]const u8 = @ptrCast(&__bss_end);
    const bss_len = @intFromPtr(bss_end) - @intFromPtr(bss_start);
    
    @memset(bss_start[0..bss_len], 0);
    
    // 3. เรียก main
    main();
    
    // 4. ถ้า main return, loop forever
    while (true) {}
}

// Default fault handlers
export fn nmiHandler() callconv(.C) void {
    while (true) {}
}

export fn hardFaultHandler() callconv(.C) void {
    // อ่านข้อมูล fault จาก registers
    asm volatile (
        \\tst lr, #4
        \\ite eq
        \\mrseq r0, msp
        \\mrsne r0, psp
        \\b hardFaultHandlerC
        ::: "memory"
    );
}

export fn memManageHandler() callconv(.C) void { while (true) {} }
export fn busFaultHandler() callconv(.C) void { while (true) {} }
export fn usageFaultHandler() callconv(.C) void { while (true) {} }
export fn svcHandler() callconv(.C) void {}
export fn debugMonHandler() callconv(.C) void {}
export fn pendSvHandler() callconv(.C) void {}

// Weak SysTick (สามารถ override ได้)
export fn sysTickHandler() callconv(.C) void {}

// Main application
fn main() void {
    // Initialize hardware
    // ...
    
    // Application loop
    while (true) {
        // Main loop
    }
}
```

---

## Step 533: Memory-Mapped I/O

### Register Access Patterns

```zig
// src/registers.zig - Memory-mapped register definitions
const std = @import("std");

// Register access ด้วย volatile pointer
pub fn Reg(comptime T: type) type {
    return struct {
        const Self = @This();
        addr: usize,
        
        pub fn read(self: Self) T {
            const ptr: *volatile T = @ptrFromInt(self.addr);
            return ptr.*;
        }
        
        pub fn write(self: Self, value: T) void {
            const ptr: *volatile T = @ptrFromInt(self.addr);
            ptr.* = value;
        }
        
        pub fn modify(self: Self, comptime field: []const u8, value: anytype) void {
            var current = self.read();
            @field(current, field) = value;
            self.write(current);
        }
        
        pub fn setBits(self: Self, mask: T) void {
            self.write(self.read() | mask);
        }
        
        pub fn clearBits(self: Self, mask: T) void {
            self.write(self.read() & ~mask);
        }
        
        pub fn toggleBits(self: Self, mask: T) void {
            self.write(self.read() ^ mask);
        }
    };
}

// STM32F103 Peripheral Base Addresses
pub const PERIPH_BASE: usize = 0x40000000;
pub const APB2PERIPH_BASE: usize = PERIPH_BASE + 0x10000;
pub const AHB1PERIPH_BASE: usize = PERIPH_BASE + 0x20000;

// RCC (Reset and Clock Control)
pub const RCC_BASE: usize = AHB1PERIPH_BASE + 0x1000;

// RCC Registers
pub const RCC = struct {
    pub const CR = Reg(u32){ .addr = RCC_BASE + 0x00 };
    pub const CFGR = Reg(u32){ .addr = RCC_BASE + 0x04 };
    pub const CIR = Reg(u32){ .addr = RCC_BASE + 0x08 };
    pub const APB2ENR = Reg(u32){ .addr = RCC_BASE + 0x18 };
    pub const APB1ENR = Reg(u32){ .addr = RCC_BASE + 0x1C };
};

// GPIO (General Purpose I/O)
pub const GPIOC_BASE: usize = APB2PERIPH_BASE + 0x1000;

pub const GPIOC = struct {
    pub const CRL = Reg(u32){ .addr = GPIOC_BASE + 0x00 }; // Config low
    pub const CRH = Reg(u32){ .addr = GPIOC_BASE + 0x04 }; // Config high
    pub const IDR = Reg(u32){ .addr = GPIOC_BASE + 0x08 }; // Input data
    pub const ODR = Reg(u32){ .addr = GPIOC_BASE + 0x0C }; // Output data
    pub const BSRR = Reg(u32){ .addr = GPIOC_BASE + 0x10 }; // Bit set/reset
    pub const BRR = Reg(u32){ .addr = GPIOC_BASE + 0x14 };  // Bit reset
    pub const LCKR = Reg(u32){ .addr = GPIOC_BASE + 0x18 }; // Config lock
};

// Bit definitions
pub const RCC_APB2ENR_IOPCEN: u32 = 1 << 4; // GPIOC clock enable
pub const GPIO_PIN_13: u32 = 1 << 13;       // Pin 13 (onboard LED)
```

### Typed Register Structs

```zig
// ใช้ packed struct สำหรับ type-safe register access
pub const RCC_CR_Bits = packed struct(u32) {
    HSION: u1,    // Internal high-speed clock enable
    HSIRDY: u1,   // Internal high-speed clock ready flag
    _reserved1: u1,
    HSITRIM: u5,  // Internal high-speed clock trimming
    HSICAL: u8,   // Internal high-speed clock calibration
    HSEON: u1,    // External clock enable
    HSERDY: u1,   // External high-speed clock ready flag
    HSEBYP: u1,   // External high-speed clock bypass
    CSSON: u1,    // Clock security system enable
    _reserved2: u4,
    PLLON: u1,    // PLL enable
    PLLRDY: u1,   // PLL clock ready flag
    _reserved3: u6,
};

pub const TypedRCC = struct {
    pub fn readCR() RCC_CR_Bits {
        const ptr: *volatile RCC_CR_Bits = @ptrFromInt(RCC_BASE);
        return ptr.*;
    }
    
    pub fn writeCR(val: RCC_CR_Bits) void {
        const ptr: *volatile RCC_CR_Bits = @ptrFromInt(RCC_BASE);
        ptr.* = val;
    }
    
    pub fn enableHSE() void {
        var cr = readCR();
        cr.HSEON = 1;
        writeCR(cr);
        
        // รอจนกว่า HSE จะพร้อม
        while (readCR().HSERDY == 0) {}
    }
};
```

---

## Step 534: Interrupt Handlers

### การตั้งค่า Interrupts

```zig
// src/interrupts.zig
const std = @import("std");

// NVIC (Nested Vectored Interrupt Controller) registers
pub const NVIC_BASE: usize = 0xE000E100;

pub const NVIC = struct {
    // ISER = Interrupt Set Enable Register
    pub fn enableIRQ(irq: u8) void {
        const reg_index = irq / 32;
        const bit_index: u5 = @truncate(irq % 32);
        
        const iser: [*]volatile u32 = @ptrFromInt(NVIC_BASE);
        iser[reg_index] |= @as(u32, 1) << bit_index;
    }
    
    // ICER = Interrupt Clear Enable Register
    pub fn disableIRQ(irq: u8) void {
        const reg_index = irq / 32;
        const bit_index: u5 = @truncate(irq % 32);
        
        const icer: [*]volatile u32 = @ptrFromInt(NVIC_BASE + 0x80);
        icer[reg_index] = @as(u32, 1) << bit_index;
    }
    
    // IPR = Interrupt Priority Register
    pub fn setPriority(irq: u8, priority: u4) void {
        const ipr: [*]volatile u8 = @ptrFromInt(NVIC_BASE + 0x300);
        ipr[irq] = @as(u8, priority) << 4; // Priority in upper nibble
    }
};

// SysTick Configuration
pub const SYSTICK_BASE: usize = 0xE000E010;

pub const SysTick = struct {
    const CTRL = SYSTICK_BASE + 0x00;
    const LOAD = SYSTICK_BASE + 0x04;
    const VAL  = SYSTICK_BASE + 0x08;
    
    pub fn init(reload_value: u24) void {
        // Set reload value
        const load_ptr: *volatile u32 = @ptrFromInt(LOAD);
        load_ptr.* = reload_value;
        
        // Clear current value
        const val_ptr: *volatile u32 = @ptrFromInt(VAL);
        val_ptr.* = 0;
        
        // Enable SysTick with CPU clock, enable interrupt
        const ctrl_ptr: *volatile u32 = @ptrFromInt(CTRL);
        ctrl_ptr.* = 0b111; // ENABLE | TICKINT | CLKSOURCE
    }
};

// Global tick counter
var system_ticks: u32 = 0;

// SysTick interrupt handler
export fn sysTickHandler() callconv(.C) void {
    system_ticks +%= 1; // Wrapping add
}

pub fn getTicks() u32 {
    return system_ticks;
}

pub fn delayMs(ms: u32) void {
    const start = getTicks();
    while (getTicks() -% start < ms) {
        // Wait
    }
}
```

### EXTI (External Interrupt) Handler

```zig
// External interrupt สำหรับ button press
pub const EXTI_BASE: usize = 0x40010400;

pub const EXTI = struct {
    pub const IMR  = Reg(u32){ .addr = EXTI_BASE + 0x00 }; // Interrupt mask
    pub const RTSR = Reg(u32){ .addr = EXTI_BASE + 0x08 }; // Rising trigger
    pub const FTSR = Reg(u32){ .addr = EXTI_BASE + 0x0C }; // Falling trigger
    pub const PR   = Reg(u32){ .addr = EXTI_BASE + 0x14 }; // Pending register
};

// AFIO สำหรับ GPIO-EXTI mapping
pub const AFIO_BASE: usize = 0x40010000;
pub const AFIO_EXTICR1 = Reg(u32){ .addr = AFIO_BASE + 0x08 };

pub fn configureButton() void {
    // Enable GPIOA and AFIO clocks
    RCC.APB2ENR.setBits(1 | (1 << 3)); // AFIO + GPIOA
    
    // Configure PA0 as input with pull-up
    GPIOA.CRL.write(0x8); // Input mode, pull-up/pull-down
    GPIOA.ODR.setBits(1); // Pull-up
    
    // Map PA0 to EXTI0
    AFIO_EXTICR1.clearBits(0xF); // Select Port A for EXTI0
    
    // Configure EXTI0 for falling edge
    EXTI.IMR.setBits(1);  // Unmask EXTI0
    EXTI.FTSR.setBits(1); // Falling edge trigger
    
    // Enable EXTI0 interrupt in NVIC
    NVIC.enableIRQ(6); // EXTI0 IRQ number
    NVIC.setPriority(6, 8); // Medium priority
}

var button_pressed: bool = false;

// EXTI0 interrupt handler
export fn EXTI0_IRQHandler() callconv(.C) void {
    // Check if EXTI0 triggered
    if (EXTI.PR.read() & 1 != 0) {
        button_pressed = true;
        EXTI.PR.write(1); // Clear by writing 1
    }
}
```

---

## Step 535: Freestanding Target (No OS)

### Minimal Freestanding Program

```zig
// main.zig สำหรับ freestanding target
// ไม่ import std ในระดับ top-level สำหรับ freestanding
const builtin = @import("builtin");

// ต้องประกาศ panic handler เอง
pub fn panic(message: []const u8, stack_trace: ?*@import("std").builtin.StackTrace, ret_addr: ?usize) noreturn {
    _ = message;
    _ = stack_trace;
    _ = ret_addr;
    
    // Disable interrupts
    asm volatile ("cpsid i");
    
    // Halt
    while (true) {
        asm volatile ("wfi"); // Wait for interrupt (power saving)
    }
}

// Entry point
pub export fn main() callconv(.C) noreturn {
    initHardware();
    
    while (true) {
        mainLoop();
    }
}

fn initHardware() void {
    // Initialize clocks, GPIO, etc.
}

fn mainLoop() void {
    // Main application logic
}
```

### Custom Allocator สำหรับ Embedded

```zig
// Fixed-size memory pool สำหรับ embedded (ไม่ใช้ heap)
pub fn FixedPool(comptime block_size: usize, comptime num_blocks: usize) type {
    return struct {
        const Self = @This();
        const POOL_SIZE = block_size * num_blocks;
        
        pool: [POOL_SIZE]u8 align(8) = undefined,
        free_blocks: [num_blocks]bool = [_]bool{true} ** num_blocks,
        
        pub fn alloc(self: *Self, size: usize) ?[]u8 {
            if (size > block_size) return null;
            
            for (&self.free_blocks, 0..) |*free, i| {
                if (free.*) {
                    free.* = false;
                    const start = i * block_size;
                    return self.pool[start .. start + size];
                }
            }
            
            return null;
        }
        
        pub fn free(self: *Self, block: []u8) void {
            const offset = @intFromPtr(block.ptr) - @intFromPtr(&self.pool[0]);
            const index = offset / block_size;
            
            if (index < num_blocks) {
                self.free_blocks[index] = true;
            }
        }
        
        pub fn usedBlocks(self: *const Self) usize {
            var count: usize = 0;
            for (self.free_blocks) |free| {
                if (!free) count += 1;
            }
            return count;
        }
    };
}

// ใช้งาน
var memory_pool = FixedPool(64, 16){};

pub fn allocateBuffer() ?[]u8 {
    return memory_pool.alloc(64);
}
```

---

## Step 536: Linker Script Integration

### Linker Script พื้นฐาน (STM32F103)

```ld
/* linker.ld สำหรับ STM32F103C8T6 */
/* Flash: 64KB at 0x08000000 */
/* RAM: 20KB at 0x20000000 */

MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}

ENTRY(resetHandler)

SECTIONS {
    /* Vector table and code in flash */
    .vector_table : {
        KEEP(*(.vector_table))
    } > FLASH
    
    .text : {
        *(.text)
        *(.text.*)
        *(.rodata)
        *(.rodata.*)
    } > FLASH
    
    /* Data section - loaded from flash, runs in RAM */
    .data : {
        __data_start = .;
        *(.data)
        *(.data.*)
        __data_end = .;
    } > RAM AT > FLASH
    
    __data_load = LOADADDR(.data);
    
    /* BSS section - zero-initialized */
    .bss : {
        __bss_start = .;
        *(.bss)
        *(.bss.*)
        *(COMMON)
        __bss_end = .;
    } > RAM
    
    /* Stack */
    . = ALIGN(8);
    . = . + 4096;  /* 4KB stack */
    __stack_top = .;
    
    /* Discard debugging info */
    /DISCARD/ : {
        *(.ARM.exidx*)
        *(.ARM.extab*)
    }
}
```

### Accessing Linker Symbols ใน Zig

```zig
// เข้าถึง linker symbols จาก Zig
extern var __data_start: u8;
extern var __data_end: u8;
extern var __data_load: u8;
extern var __bss_start: u8;
extern var __bss_end: u8;
extern var __stack_top: u8;

pub fn getFlashUsage() usize {
    return @intFromPtr(&__data_load) + 
           (@intFromPtr(&__data_end) - @intFromPtr(&__data_start)) -
           0x08000000; // Flash start address
}

pub fn getRamUsage() usize {
    return @intFromPtr(&__bss_end) - 0x20000000; // RAM start address
}

// Custom sections
export var device_id linksection(".device_id") = [_]u8{0} ** 12;

pub fn readDeviceId() []const u8 {
    return &device_id;
}
```

---

## Step 537: GPIO Programming (STM32 Example)

### GPIO Driver

```zig
// src/gpio.zig
pub const Pin = struct {
    port: *Port,
    number: u4, // 0-15
    
    pub fn setHigh(self: Pin) void {
        self.port.bsrr.write(@as(u32, 1) << self.number);
    }
    
    pub fn setLow(self: Pin) void {
        self.port.bsrr.write(@as(u32, 1) << (@as(u5, self.number) + 16));
    }
    
    pub fn toggle(self: Pin) void {
        self.port.odr.toggleBits(@as(u32, 1) << self.number);
    }
    
    pub fn read(self: Pin) bool {
        return (self.port.idr.read() >> self.number) & 1 == 1;
    }
    
    pub fn configure(self: Pin, mode: Mode, config: Config) void {
        const shift: u5 = @as(u5, self.number % 8) * 4;
        const value = (@as(u32, @intFromEnum(config)) << 2) | @as(u32, @intFromEnum(mode));
        
        if (self.number < 8) {
            var crl = self.port.crl.read();
            crl &= ~(@as(u32, 0xF) << shift);
            crl |= value << shift;
            self.port.crl.write(crl);
        } else {
            var crh = self.port.crh.read();
            crh &= ~(@as(u32, 0xF) << shift);
            crh |= value << shift;
            self.port.crh.write(crh);
        }
    }
};

pub const Mode = enum(u2) {
    input = 0b00,
    output_10mhz = 0b01,
    output_2mhz = 0b10,
    output_50mhz = 0b11,
};

pub const Config = enum(u2) {
    // Input modes
    analog_input = 0b00,
    floating_input = 0b01,
    input_pull = 0b10,
    // Output modes
    push_pull = 0b00,
    open_drain = 0b01,
    alt_push_pull = 0b10,
    alt_open_drain = 0b11,
};

pub const Port = struct {
    crl: Reg(u32),
    crh: Reg(u32),
    idr: Reg(u32),
    odr: Reg(u32),
    bsrr: Reg(u32),
    brr: Reg(u32),
    lckr: Reg(u32),
    
    pub fn pin(self: *Port, number: u4) Pin {
        return .{ .port = self, .number = number };
    }
};

// Port C (contains onboard LED on PC13)
pub var PORTC = Port{
    .crl  = .{ .addr = GPIOC_BASE + 0x00 },
    .crh  = .{ .addr = GPIOC_BASE + 0x04 },
    .idr  = .{ .addr = GPIOC_BASE + 0x08 },
    .odr  = .{ .addr = GPIOC_BASE + 0x0C },
    .bsrr = .{ .addr = GPIOC_BASE + 0x10 },
    .brr  = .{ .addr = GPIOC_BASE + 0x14 },
    .lckr = .{ .addr = GPIOC_BASE + 0x18 },
};

// LED Blink Example
pub fn blinkLed() void {
    // Enable GPIOC clock
    RCC.APB2ENR.setBits(RCC_APB2ENR_IOPCEN);
    
    // Configure PC13 as output
    const led = PORTC.pin(13);
    led.configure(.output_2mhz, .push_pull);
    
    var i: u32 = 0;
    while (true) : (i += 1) {
        led.toggle();
        delayMs(500);
    }
}
```

---

## Step 538: UART Communication

### UART Driver

```zig
// src/uart.zig
const USART1_BASE: usize = 0x40013800;

pub const USART = struct {
    base: usize,
    
    const SR_OFFSET  = 0x00;
    const DR_OFFSET  = 0x04;
    const BRR_OFFSET = 0x08;
    const CR1_OFFSET = 0x0C;
    const CR2_OFFSET = 0x10;
    
    // Status register bits
    const SR_TXE  = 1 << 7; // Transmit empty
    const SR_TC   = 1 << 6; // Transmission complete
    const SR_RXNE = 1 << 5; // Read data register not empty
    
    // CR1 bits
    const CR1_UE  = 1 << 13; // USART enable
    const CR1_M   = 1 << 12; // Word length (0=8bit, 1=9bit)
    const CR1_TE  = 1 << 3;  // Transmitter enable
    const CR1_RE  = 1 << 2;  // Receiver enable
    
    pub fn init(self: USART, baud: u32, apb_clock: u32) void {
        // Enable USART1 clock
        RCC.APB2ENR.setBits(1 << 14);
        
        // Configure GPIO (TX=PA9, RX=PA10 for USART1)
        // PA9: Alternate function push-pull output
        // PA10: Input floating
        
        // Set baud rate
        // BRR = apb_clock / baud
        const brr: u32 = apb_clock / baud;
        const brr_ptr: *volatile u32 = @ptrFromInt(self.base + BRR_OFFSET);
        brr_ptr.* = brr;
        
        // Enable USART, TX, RX
        const cr1_ptr: *volatile u32 = @ptrFromInt(self.base + CR1_OFFSET);
        cr1_ptr.* = CR1_UE | CR1_TE | CR1_RE;
    }
    
    pub fn sendByte(self: USART, byte: u8) void {
        // Wait until TX empty
        const sr_ptr: *volatile u32 = @ptrFromInt(self.base + SR_OFFSET);
        while (sr_ptr.* & SR_TXE == 0) {}
        
        // Write byte
        const dr_ptr: *volatile u32 = @ptrFromInt(self.base + DR_OFFSET);
        dr_ptr.* = byte;
    }
    
    pub fn sendString(self: USART, str: []const u8) void {
        for (str) |byte| {
            self.sendByte(byte);
        }
    }
    
    pub fn receiveByte(self: USART) u8 {
        // Wait until data available
        const sr_ptr: *volatile u32 = @ptrFromInt(self.base + SR_OFFSET);
        while (sr_ptr.* & SR_RXNE == 0) {}
        
        // Read byte
        const dr_ptr: *volatile u32 = @ptrFromInt(self.base + DR_OFFSET);
        return @truncate(dr_ptr.*);
    }
    
    pub fn tryReceiveByte(self: USART) ?u8 {
        const sr_ptr: *volatile u32 = @ptrFromInt(self.base + SR_OFFSET);
        if (sr_ptr.* & SR_RXNE == 0) return null;
        
        const dr_ptr: *volatile u32 = @ptrFromInt(self.base + DR_OFFSET);
        return @truncate(dr_ptr.*);
    }
    
    // Printf-like function
    pub fn print(self: USART, comptime fmt: []const u8, args: anytype) void {
        var buf: [256]u8 = undefined;
        const len = formatBuffer(&buf, fmt, args);
        self.sendString(buf[0..len]);
    }
};

pub const uart1 = USART{ .base = USART1_BASE };

// Simple format function
fn formatBuffer(buf: []u8, comptime fmt: []const u8, args: anytype) usize {
    // ใช้ std.fmt.bufPrint หรือ implement เอง
    const result = @import("std").fmt.bufPrint(buf, fmt, args) catch buf[0..0];
    return result.len;
}
```

---

## Step 539: SPI/I2C Protocols

### SPI Driver

```zig
// src/spi.zig
const SPI1_BASE: usize = 0x40013000;

pub const SPI = struct {
    base: usize,
    
    const CR1_OFFSET = 0x00;
    const CR2_OFFSET = 0x04;
    const SR_OFFSET  = 0x08;
    const DR_OFFSET  = 0x0C;
    
    // CR1 bits
    const CR1_SPE     = 1 << 6;  // SPI enable
    const CR1_MSTR    = 1 << 2;  // Master mode
    const CR1_CPOL    = 1 << 1;  // Clock polarity
    const CR1_CPHA    = 1 << 0;  // Clock phase
    const CR1_SSI     = 1 << 8;  // Internal slave select
    const CR1_SSM     = 1 << 9;  // Software slave management
    const CR1_BR_DIV4 = 1 << 3;  // Baud rate: fPCLK/4
    
    // SR bits
    const SR_TXE  = 1 << 1; // Transmit buffer empty
    const SR_RXNE = 1 << 0; // Receive buffer not empty
    const SR_BSY  = 1 << 7; // Busy flag
    
    pub fn init(self: SPI) void {
        // Enable SPI1 clock
        RCC.APB2ENR.setBits(1 << 12);
        
        const cr1_ptr: *volatile u32 = @ptrFromInt(self.base + CR1_OFFSET);
        
        // Configure: Master mode, software NSS, baud rate fPCLK/4
        cr1_ptr.* = CR1_MSTR | CR1_SSI | CR1_SSM | CR1_BR_DIV4;
        
        // Enable SPI
        cr1_ptr.* |= CR1_SPE;
    }
    
    pub fn transfer(self: SPI, data: u8) u8 {
        const dr_ptr: *volatile u8 = @ptrFromInt(self.base + DR_OFFSET);
        const sr_ptr: *volatile u32 = @ptrFromInt(self.base + SR_OFFSET);
        
        // Wait for TX buffer empty
        while (sr_ptr.* & SR_TXE == 0) {}
        
        // Send byte
        dr_ptr.* = data;
        
        // Wait for RX data
        while (sr_ptr.* & SR_RXNE == 0) {}
        
        return dr_ptr.*;
    }
    
    pub fn transferBuffer(self: SPI, tx: []const u8, rx: []u8) void {
        var i: usize = 0;
        while (i < tx.len and i < rx.len) : (i += 1) {
            rx[i] = self.transfer(tx[i]);
        }
    }
    
    pub fn sendBuffer(self: SPI, data: []const u8) void {
        for (data) |byte| {
            _ = self.transfer(byte);
        }
        
        // Wait for transfer complete
        const sr_ptr: *volatile u32 = @ptrFromInt(self.base + SR_OFFSET);
        while (sr_ptr.* & SR_BSY != 0) {}
    }
};

pub const spi1 = SPI{ .base = SPI1_BASE };
```

### I2C Driver

```zig
// src/i2c.zig
const I2C1_BASE: usize = 0x40005400;

pub const I2C = struct {
    base: usize,
    
    const CR1_OFFSET  = 0x00;
    const CR2_OFFSET  = 0x04;
    const OAR1_OFFSET = 0x08;
    const DR_OFFSET   = 0x10;
    const SR1_OFFSET  = 0x14;
    const SR2_OFFSET  = 0x18;
    const CCR_OFFSET  = 0x1C;
    const TRISE_OFFSET= 0x20;
    
    const CR1_PE    = 1 << 0;  // Peripheral enable
    const CR1_START = 1 << 8;  // Generate START condition
    const CR1_STOP  = 1 << 9;  // Generate STOP condition
    const CR1_ACK   = 1 << 10; // Acknowledge enable
    
    const SR1_SB    = 1 << 0;  // Start bit generated
    const SR1_ADDR  = 1 << 1;  // Address sent
    const SR1_TXE   = 1 << 7;  // Data register empty
    const SR1_RXNE  = 1 << 6;  // Data register not empty
    const SR1_BTF   = 1 << 2;  // Byte transfer finished
    
    pub fn init(self: I2C, pclk1_mhz: u32) void {
        // Enable I2C1 clock
        // RCC.APB1ENR |= (1 << 21);
        
        const cr2_ptr: *volatile u32 = @ptrFromInt(self.base + CR2_OFFSET);
        cr2_ptr.* = pclk1_mhz & 0x3F; // Set peripheral clock frequency
        
        // Configure for standard mode (100kHz)
        const ccr_ptr: *volatile u32 = @ptrFromInt(self.base + CCR_OFFSET);
        ccr_ptr.* = (pclk1_mhz * 1000) / (2 * 100); // T_high = T_low = 5us
        
        // Set maximum rise time
        const trise_ptr: *volatile u32 = @ptrFromInt(self.base + TRISE_OFFSET);
        trise_ptr.* = pclk1_mhz + 1;
        
        // Enable I2C
        const cr1_ptr: *volatile u32 = @ptrFromInt(self.base + CR1_OFFSET);
        cr1_ptr.* = CR1_PE;
    }
    
    fn waitFlag(self: I2C, sr: usize, flag: u32) void {
        const sr_ptr: *volatile u32 = @ptrFromInt(sr);
        while (sr_ptr.* & flag == 0) {}
    }
    
    pub fn write(self: I2C, addr: u7, data: []const u8) void {
        const cr1_ptr: *volatile u32 = @ptrFromInt(self.base + CR1_OFFSET);
        const dr_ptr: *volatile u8 = @ptrFromInt(self.base + DR_OFFSET);
        
        // Generate START
        cr1_ptr.* |= CR1_START;
        self.waitFlag(self.base + SR1_OFFSET, SR1_SB);
        _ = (@as(*volatile u32, @ptrFromInt(self.base + SR1_OFFSET)).*); // Clear SB
        
        // Send address + write bit
        dr_ptr.* = @as(u8, addr) << 1;
        self.waitFlag(self.base + SR1_OFFSET, SR1_ADDR);
        _ = (@as(*volatile u32, @ptrFromInt(self.base + SR2_OFFSET)).*); // Clear ADDR
        
        // Send data bytes
        for (data) |byte| {
            self.waitFlag(self.base + SR1_OFFSET, SR1_TXE);
            dr_ptr.* = byte;
        }
        
        // Wait for transfer complete
        self.waitFlag(self.base + SR1_OFFSET, SR1_BTF);
        
        // Generate STOP
        cr1_ptr.* |= CR1_STOP;
    }
    
    pub fn read(self: I2C, addr: u7, buffer: []u8) void {
        const cr1_ptr: *volatile u32 = @ptrFromInt(self.base + CR1_OFFSET);
        const dr_ptr: *volatile u8 = @ptrFromInt(self.base + DR_OFFSET);
        
        // Enable ACK
        cr1_ptr.* |= CR1_ACK;
        
        // Generate START
        cr1_ptr.* |= CR1_START;
        self.waitFlag(self.base + SR1_OFFSET, SR1_SB);
        _ = (@as(*volatile u32, @ptrFromInt(self.base + SR1_OFFSET)).*);
        
        // Send address + read bit
        dr_ptr.* = (@as(u8, addr) << 1) | 1;
        self.waitFlag(self.base + SR1_OFFSET, SR1_ADDR);
        _ = (@as(*volatile u32, @ptrFromInt(self.base + SR2_OFFSET)).*);
        
        // Read data
        for (buffer, 0..) |*byte, i| {
            if (i == buffer.len - 1) {
                cr1_ptr.* &= ~@as(u32, CR1_ACK); // NACK last byte
            }
            self.waitFlag(self.base + SR1_OFFSET, SR1_RXNE);
            byte.* = dr_ptr.*;
        }
        
        // Generate STOP
        cr1_ptr.* |= CR1_STOP;
    }
};

pub const i2c1 = I2C{ .base = I2C1_BASE };
```

---

## Step 540: Real-World Embedded Project

### Temperature Monitor with OLED Display

```zig
// src/main.zig - Temperature Monitor Project
// Hardware: STM32F103, DS18B20 temperature sensor, SSD1306 OLED

const GPIO = @import("gpio.zig");
const UART = @import("uart.zig");
const I2C = @import("i2c.zig");
const SPI = @import("spi.zig");
const Interrupts = @import("interrupts.zig");

// System configuration
const CPU_FREQ = 72_000_000; // 72 MHz
const APB1_FREQ = 36_000_000; // 36 MHz
const APB2_FREQ = 72_000_000; // 72 MHz
const UART_BAUD = 115200;

// DS18B20 Temperature sensor (1-Wire protocol)
const DS18B20 = struct {
    pin: GPIO.Pin,
    
    const CMD_CONVERT  = 0x44; // Start temperature conversion
    const CMD_READ_ROM = 0x33; // Read ROM (single device)
    const CMD_SKIP_ROM = 0xCC; // Skip ROM (broadcast)
    const CMD_READ_SCR = 0xBE; // Read scratchpad
    
    pub fn init(self: DS18B20) void {
        self.pin.configure(.output_2mhz, .open_drain);
        self.pin.setHigh();
    }
    
    fn reset(self: DS18B20) bool {
        // Pull low for 480us
        self.pin.setLow();
        delayUs(480);
        self.pin.setHigh();
        
        // Wait for presence pulse
        delayUs(70);
        const presence = !self.pin.read();
        delayUs(410);
        
        return presence;
    }
    
    fn writeBit(self: DS18B20, bit: u1) void {
        self.pin.setLow();
        if (bit == 1) {
            delayUs(6);
            self.pin.setHigh();
            delayUs(64);
        } else {
            delayUs(60);
            self.pin.setHigh();
            delayUs(10);
        }
    }
    
    fn readBit(self: DS18B20) u1 {
        self.pin.setLow();
        delayUs(6);
        self.pin.setHigh();
        delayUs(9);
        const bit: u1 = if (self.pin.read()) 1 else 0;
        delayUs(55);
        return bit;
    }
    
    fn writeByte(self: DS18B20, byte: u8) void {
        var b = byte;
        for (0..8) |_| {
            self.writeBit(@truncate(b & 1));
            b >>= 1;
        }
    }
    
    fn readByte(self: DS18B20) u8 {
        var byte: u8 = 0;
        for (0..8) |i| {
            byte |= @as(u8, self.readBit()) << @intCast(i);
        }
        return byte;
    }
    
    pub fn readTemperature(self: DS18B20) ?i16 {
        if (!self.reset()) return null;
        
        self.writeByte(CMD_SKIP_ROM);
        self.writeByte(CMD_CONVERT);
        
        // Wait for conversion (750ms for 12-bit)
        delayMs(750);
        
        if (!self.reset()) return null;
        
        self.writeByte(CMD_SKIP_ROM);
        self.writeByte(CMD_READ_SCR);
        
        const lsb = self.readByte();
        const msb = self.readByte();
        
        // Temperature in 1/16 degree units
        const raw: i16 = @as(i16, @bitCast(@as(u16, msb) << 8 | lsb));
        
        return raw; // Divide by 16 for Celsius
    }
};

// SSD1306 OLED Display (I2C)
const SSD1306 = struct {
    i2c: I2C.I2C,
    addr: u7,
    
    const WIDTH = 128;
    const HEIGHT = 64;
    
    var framebuffer: [WIDTH * HEIGHT / 8]u8 = [_]u8{0} ** (WIDTH * HEIGHT / 8);
    
    pub fn init(self: SSD1306) void {
        const init_cmds = [_]u8{
            0xAE, // Display off
            0xD5, 0x80, // Set display clock
            0xA8, 63,   // Set multiplex ratio
            0xD3, 0x00, // Set display offset
            0x40,       // Set start line
            0x8D, 0x14, // Charge pump on
            0x20, 0x00, // Memory addressing mode: horizontal
            0xA1,       // Segment remap
            0xC8,       // COM output scan direction
            0xDA, 0x12, // COM pins hardware config
            0x81, 0xCF, // Set contrast
            0xD9, 0xF1, // Set pre-charge period
            0xDB, 0x40, // Set VCOMH deselect level
            0xA4,       // Display from RAM
            0xA6,       // Normal display (not inverted)
            0xAF,       // Display on
        };
        
        self.sendCommands(&init_cmds);
    }
    
    fn sendCommands(self: SSD1306, cmds: []const u8) void {
        var buf: [2]u8 = undefined;
        for (cmds) |cmd| {
            buf[0] = 0x00; // Command mode
            buf[1] = cmd;
            self.i2c.write(self.addr, &buf);
        }
    }
    
    pub fn clearScreen(self: SSD1306) void {
        _ = self;
        @memset(&framebuffer, 0);
    }
    
    pub fn drawPixel(x: u7, y: u6, on: bool) void {
        if (x >= WIDTH or y >= HEIGHT) return;
        
        const page = y / 8;
        const bit: u3 = @truncate(y % 8);
        const index = @as(usize, page) * WIDTH + x;
        
        if (on) {
            framebuffer[index] |= @as(u8, 1) << bit;
        } else {
            framebuffer[index] &= ~(@as(u8, 1) << bit);
        }
    }
    
    pub fn flush(self: SSD1306) void {
        // Set page address 0-7
        const set_addr = [_]u8{ 0x21, 0, 127, 0x22, 0, 7 };
        self.sendCommands(&set_addr);
        
        // Send framebuffer
        var buf: [129]u8 = undefined;
        buf[0] = 0x40; // Data mode
        
        var page: u8 = 0;
        while (page < 8) : (page += 1) {
            const start = page * WIDTH;
            @memcpy(buf[1..], framebuffer[start .. start + WIDTH]);
            self.i2c.write(self.addr, &buf);
        }
    }
};

// Main program
pub export fn main() callconv(.C) noreturn {
    // Initialize SysTick (1ms tick)
    Interrupts.SysTick.init(CPU_FREQ / 1000 - 1);
    
    // Initialize UART for debug
    uart1.init(UART_BAUD, APB2_FREQ);
    uart1.sendString("Temperature Monitor Started\r\n");
    
    // Initialize I2C for OLED
    i2c1.init(APB1_FREQ / 1_000_000);
    
    // Initialize display
    var display = SSD1306{ .i2c = i2c1, .addr = 0x3C };
    display.init();
    display.clearScreen();
    
    // Initialize temperature sensor
    const temp_pin = GPIO.PORTC.pin(4);
    var sensor = DS18B20{ .pin = temp_pin };
    sensor.init();
    
    uart1.sendString("Initialized! Reading temperature...\r\n");
    
    while (true) {
        // Read temperature
        if (sensor.readTemperature()) |raw_temp| {
            const celsius_x16 = raw_temp;
            const celsius = @divTrunc(celsius_x16, 16);
            const frac = @abs(@rem(celsius_x16, 16)) * 625; // 0.0625 per unit
            
            // Log to UART
            uart1.print("Temperature: {}.{:0>4}C\r\n", .{ celsius, frac });
            
            // Display on OLED
            display.clearScreen();
            // (drawText function would go here)
            display.flush();
        } else {
            uart1.sendString("Sensor error!\r\n");
        }
        
        delayMs(2000); // Update every 2 seconds
    }
}

fn delayUs(us: u32) void {
    // Simple busy-wait delay
    var i: u32 = 0;
    while (i < us * (CPU_FREQ / 1_000_000 / 10)) : (i += 1) {
        asm volatile ("nop");
    }
}
```

---

## สรุป Part 54

ในบทนี้เราได้เรียนรู้:

1. **Embedded Basics** - ทำไม Zig เหมาะกับ embedded systems
2. **Startup Code** - vector table, reset handler, memory init
3. **Memory-Mapped I/O** - register access patterns, typed registers
4. **Interrupt Handlers** - NVIC, SysTick, EXTI
5. **Freestanding Target** - ไม่มี OS, custom allocator
6. **Linker Script** - memory layout, custom sections
7. **GPIO** - digital I/O, configuration, pin driver
8. **UART** - serial communication
9. **SPI/I2C** - hardware protocols
10. **Real Project** - temperature monitor ครบวงจร

> **หมายเหตุ**: code ในบทนี้เป็น example สำหรับ STM32F103 หากใช้ MCU อื่น ต้องปรับ base addresses และ register definitions ให้ตรงกับ datasheet
