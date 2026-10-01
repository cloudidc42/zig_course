# Part 01: การติดตั้งและเริ่มต้นใช้งาน Zig
## Steps 1-10: Installation and Setup

---

> **หมายเหตุ**: เอกสารนี้เป็นส่วนที่ 1 ของหลักสูตร Zig ภาษาไทย ครอบคลุมการติดตั้งและการตั้งค่าเบื้องต้น
> เหมาะสำหรับผู้เริ่มต้นที่มีพื้นฐานการเขียนโปรแกรมบ้างแล้ว

---

## สารบัญ

- [Step 1: Zig คืออะไร? ประวัติและปรัชญา](#step-1)
- [Step 2: ทำไมต้องเรียน Zig?](#step-2)
- [Step 3: การติดตั้ง Zig บน Linux](#step-3)
- [Step 4: การติดตั้ง Zig บน macOS](#step-4)
- [Step 5: การติดตั้ง Zig บน Windows](#step-5)
- [Step 6: การตรวจสอบการติดตั้ง](#step-6)
- [Step 7: การตั้งค่า Editor](#step-7)
- [Step 8: โครงสร้างโปรเจค Zig เบื้องต้น](#step-8)
- [Step 9: การใช้ zig init และ zig build](#step-9)
- [Step 10: การเรียกใช้ zig run และ zig build-exe](#step-10)
- [แบบฝึกหัด](#exercises)
- [สรุป](#summary)

---

<a name="step-1"></a>
## Step 1: Zig คืออะไร? ประวัติและปรัชญา

### Zig คืออะไร?

**Zig** เป็นภาษาโปรแกรมระดับต่ำ (systems programming language) ที่ออกแบบมาเพื่อเป็นทางเลือกที่ทันสมัยและเรียบง่ายกว่าภาษา C โดยเน้นความโปร่งใส ความน่าเชื่อถือ และประสิทธิภาพสูงสุด

Zig ไม่ใช่แค่ภาษาโปรแกรม แต่ยังมาพร้อมกับ **Toolchain** ที่สมบูรณ์แบบซึ่งสามารถ:
- คอมไพล์โค้ด Zig
- คอมไพล์โค้ด C และ C++
- Cross-compile ไปยัง platform ต่าง ๆ ได้อย่างง่ายดาย

### ประวัติของ Zig

| ปี | เหตุการณ์ |
|----|-----------|
| 2015 | Andrew Kelley เริ่มพัฒนา Zig เป็นโปรเจคส่วนตัว |
| 2016 | เปิดตัว Zig ต่อสาธารณะครั้งแรก |
| 2017 | ชุมชนเริ่มเติบโต มีผู้ร่วมพัฒนามากขึ้น |
| 2020 | ก่อตั้ง Zig Software Foundation (ZSF) องค์กรไม่แสวงหากำไร |
| 2022 | ออก Zig 0.10 พร้อมระบบ build ใหม่ |
| 2023 | ออก Zig 0.11 และ 0.12 ปรับปรุงภาษาครั้งใหญ่ |
| 2024 | ออก Zig 0.13 เตรียมสู่ 1.0 |

### ผู้สร้าง: Andrew Kelley

Andrew Kelley เริ่มสร้าง Zig เพราะเขาไม่พอใจกับข้อจำกัดของภาษา C เขาต้องการภาษาที่:
- **ง่ายกว่า C++** แต่ยังคงประสิทธิภาพ
- **ปลอดภัยกว่า C** แต่ไม่ซับซ้อนเหมือน Rust
- **โปร่งใส** — ไม่มีสิ่งที่ซ่อนอยู่เบื้องหลัง
- **ควบคุมได้** — นักพัฒนามีการควบคุมเต็มที่

### ปรัชญาของ Zig

Zig มีปรัชญาหลักที่สำคัญ 4 ประการ:

#### 1. "No Hidden Control Flow" (ไม่มีการไหลของโปรแกรมที่ซ่อนอยู่)

ใน C++ มี exceptions, constructors, destructors ที่ทำงานโดยอัตโนมัติ
ใน Zig ทุกอย่างต้องเขียนอย่างชัดเจน ไม่มีเวทมนตร์ซ่อนอยู่

```zig
// ใน Zig คุณต้องจัดการ error อย่างชัดเจน
const result = someFunction() catch |err| {
    // จัดการ error ที่นี่
    return err;
};
```

#### 2. "No Hidden Memory Allocations" (ไม่มีการจัดสรรหน่วยความจำที่ซ่อนอยู่)

ใน Zig ฟังก์ชันที่ต้องการจัดสรรหน่วยความจำต้องรับ `Allocator` เป็น parameter เสมอ

```zig
// ฟังก์ชันที่ต้องการ allocator ต้องรับมันอย่างชัดเจน
fn createList(allocator: std.mem.Allocator) !std.ArrayList(i32) {
    return std.ArrayList(i32).init(allocator);
}
```

#### 3. "No Preprocessor" (ไม่มี Preprocessor)

ต่างจาก C ที่มี `#define`, `#ifdef` Zig ใช้ comptime (compile-time computation) แทน

```zig
// Comptime แทน #define
const MAX_SIZE = 1024; // ค่าคงที่

// Comptime condition แทน #ifdef
const is_debug = @import("builtin").mode == .Debug;
```

#### 4. "First-class Cross Compilation" (Cross-compilation เป็นคุณสมบัติหลัก)

Zig สามารถ cross-compile ได้โดยง่าย โดยไม่ต้องติดตั้ง toolchain พิเศษ

```bash
# คอมไพล์โปรแกรม Linux บน Windows
zig build-exe main.zig -target x86_64-linux

# คอมไพล์โปรแกรม macOS บน Linux
zig build-exe main.zig -target aarch64-macos
```

---

> **หมายเหตุ**: Zig ยังอยู่ในช่วงพัฒนา (pre-1.0) ดังนั้น API อาจมีการเปลี่ยนแปลงระหว่าง version
> เอกสารนี้ใช้ Zig 0.13.x เป็นฐาน

---

<a name="step-2"></a>
## Step 2: ทำไมต้องเรียน Zig? เปรียบเทียบกับ C, C++, Rust

### ตลาดและการใช้งาน

Zig ถูกใช้ในหลายบริบทสำคัญ:

- **Bun** — JavaScript runtime ที่เร็วที่สุด เขียนด้วย Zig
- **TigerBeetle** — ฐานข้อมูลทางการเงินที่มีความน่าเชื่อถือสูง
- **Mach** — Game engine และ multimedia framework
- **Systems programming** — OS kernels, embedded systems, compilers

### การเปรียบเทียบ Zig กับภาษาอื่น

#### Zig vs C

| คุณสมบัติ | C | Zig |
|-----------|---|-----|
| Error handling | Return codes (เสี่ยงถูกละเลย) | Compile-time forced error handling |
| Undefined behavior | มีมาก ตรวจจับยาก | ลดลงมาก มี safety checks |
| Memory management | Manual (malloc/free) | Manual แต่ชัดเจนกว่า |
| Standard library | แตกต่างกันตาม platform | Cross-platform และ consistent |
| Build system | Makefile (ซับซ้อน) | zig build (เรียบง่าย) |
| Cross compilation | ยาก ต้องติดตั้งเพิ่ม | ง่ายมาก built-in |
| Interoperability | - | เรียก C ได้โดยตรง |

```zig
// ตัวอย่าง: Error handling ใน Zig ชัดเจนกว่า C มาก
const std = @import("std");

// ฟังก์ชันที่อาจเกิด error ต้องคืน error union
fn divide(a: f64, b: f64) !f64 {
    if (b == 0.0) return error.DivisionByZero;
    return a / b;
}

pub fn main() !void {
    // ต้องจัดการ error อย่างชัดเจน
    const result = try divide(10.0, 2.0);
    std.debug.print("ผลลัพธ์: {d}\n", .{result});
}
```

#### Zig vs C++

| คุณสมบัติ | C++ | Zig |
|-----------|-----|-----|
| ความซับซ้อน | สูงมาก | ต่ำ |
| เวลาคอมไพล์ | ช้า | เร็ว |
| Hidden control flow | มี (exceptions, RAII) | ไม่มี |
| Template metaprogramming | ซับซ้อน | comptime (เรียบง่าย) |
| Object-oriented | เต็มรูปแบบ | ไม่มี (ใช้ struct + methods) |
| Backward compatibility | C compatible | C compatible |

```zig
// comptime ใน Zig แทน template ของ C++
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

pub fn main() void {
    const a = max(i32, 10, 20);   // max<int>(10, 20)
    const b = max(f64, 3.14, 2.72); // max<double>(3.14, 2.72)
    _ = a;
    _ = b;
}
```

#### Zig vs Rust

| คุณสมบัติ | Rust | Zig |
|-----------|------|-----|
| Memory safety | Compile-time (borrow checker) | Runtime safety checks + careful programming |
| Learning curve | ชัน | ปานกลาง |
| Compile time | ช้า | เร็ว |
| Error messages | ดีมาก | ดี |
| Async/await | สมบูรณ์ | กำลังพัฒนา |
| C interop | ได้ (ผ่าน FFI) | ง่ายกว่า (native) |
| Package manager | Cargo (สมบูรณ์) | zig build (กำลังพัฒนา) |

```zig
// Zig ไม่มี borrow checker แต่มี optional safety
const value: ?i32 = null; // optional type (คล้าย Option<i32> ใน Rust)

if (value) |v| {
    // ใช้ v ที่นี่ได้อย่างปลอดภัย
    std.debug.print("มีค่า: {d}\n", .{v});
} else {
    std.debug.print("ไม่มีค่า\n", .{});
}
```

### เมื่อไรควรเลือก Zig?

**เลือก Zig เมื่อ:**
- ต้องการความเร็วสูงสุดและการควบคุมหน่วยความจำ
- กำลังทำงานกับโค้ด C เดิมและต้องการปรับปรุง
- ต้องการ cross-compile ง่าย ๆ
- ต้องการภาษา systems programming ที่เรียนรู้ง่ายกว่า Rust

**อาจเลือกภาษาอื่นเมื่อ:**
- ต้องการ memory safety ในระดับ Rust สำหรับโปรเจคขนาดใหญ่
- ต้องการ ecosystem ที่สมบูรณ์ (ใช้ Rust หรือ Go)
- ทำ web development (ใช้ JavaScript/TypeScript, Go, หรือ Python)

---

<a name="step-3"></a>
## Step 3: การติดตั้ง Zig บน Linux

### วิธีที่ 1: ดาวน์โหลดจาก Official Website (แนะนำ)

#### ขั้นตอนการติดตั้ง

```bash
# 1. ดาวน์โหลด Zig สำหรับ Linux x86_64
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz

# 2. แตกไฟล์
tar -xf zig-linux-x86_64-0.13.0.tar.xz

# 3. ย้ายไปยัง directory ที่เหมาะสม
sudo mv zig-linux-x86_64-0.13.0 /usr/local/zig

# 4. เพิ่ม PATH (สำหรับ bash)
echo 'export PATH="$PATH:/usr/local/zig"' >> ~/.bashrc
source ~/.bashrc

# 5. เพิ่ม PATH (สำหรับ zsh)
echo 'export PATH="$PATH:/usr/local/zig"' >> ~/.zshrc
source ~/.zshrc
```

#### สำหรับ ARM (เช่น Raspberry Pi, Apple M1 Linux)

```bash
# สำหรับ aarch64 (ARM 64-bit)
wget https://ziglang.org/download/0.13.0/zig-linux-aarch64-0.13.0.tar.xz
tar -xf zig-linux-aarch64-0.13.0.tar.xz
sudo mv zig-linux-aarch64-0.13.0 /usr/local/zig
```

### วิธีที่ 2: ใช้ Package Manager

#### Ubuntu/Debian (ผ่าน snap)

```bash
sudo snap install zig --classic --beta
```

> **คำเตือน**: snap package อาจไม่ใช่ version ล่าสุดเสมอ แนะนำให้ใช้ official download แทน

#### Arch Linux (ผ่าน pacman)

```bash
# Zig อยู่ใน community repository
sudo pacman -S zig

# หรือใช้ AUR สำหรับ nightly build
yay -S zig-dev-bin
```

#### Fedora/RHEL

```bash
sudo dnf install zig
```

#### NixOS

```nix
# เพิ่มใน configuration.nix
environment.systemPackages = with pkgs; [
    zig
];
```

### วิธีที่ 3: ใช้ zigup (Zig Version Manager)

```bash
# ติดตั้ง zigup
curl -L https://github.com/marler8997/zigup/releases/latest/download/zigup-linux-x86_64 -o zigup
chmod +x zigup
sudo mv zigup /usr/local/bin/

# ติดตั้ง Zig version ล่าสุด
zigup fetch latest

# ติดตั้ง version เฉพาะ
zigup fetch 0.13.0

# ใช้ version ที่ต้องการ
zigup 0.13.0
```

---

> **เคล็ดลับ**: สำหรับการพัฒนา ควรใช้ `master` (nightly) build เพื่อให้ได้ฟีเจอร์ล่าสุด
> แต่สำหรับโปรเจคที่ต้องการความเสถียร ใช้ release version

---

<a name="step-4"></a>
## Step 4: การติดตั้ง Zig บน macOS

### วิธีที่ 1: Homebrew (แนะนำสำหรับ macOS)

```bash
# ติดตั้ง Homebrew ถ้ายังไม่มี
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Zig
brew install zig

# อัปเดตเป็น version ล่าสุด
brew upgrade zig
```

### วิธีที่ 2: ดาวน์โหลดจาก Official Website

```bash
# สำหรับ Apple Silicon (M1/M2/M3)
curl -LO https://ziglang.org/download/0.13.0/zig-macos-aarch64-0.13.0.tar.xz
tar -xf zig-macos-aarch64-0.13.0.tar.xz
sudo mv zig-macos-aarch64-0.13.0 /usr/local/zig

# สำหรับ Intel Mac
curl -LO https://ziglang.org/download/0.13.0/zig-macos-x86_64-0.13.0.tar.xz
tar -xf zig-macos-x86_64-0.13.0.tar.xz
sudo mv zig-macos-x86_64-0.13.0 /usr/local/zig
```

#### เพิ่ม PATH สำหรับ macOS

```bash
# สำหรับ bash (~/.bash_profile หรือ ~/.bashrc)
echo 'export PATH="$PATH:/usr/local/zig"' >> ~/.bash_profile

# สำหรับ zsh (~/.zshrc) — default shell บน macOS Catalina ขึ้นไป
echo 'export PATH="$PATH:/usr/local/zig"' >> ~/.zshrc
source ~/.zshrc
```

### วิธีที่ 3: MacPorts

```bash
sudo port install zig
```

### การจัดการ Gatekeeper บน macOS

macOS อาจบล็อก Zig binary ที่ดาวน์โหลดมา:

```bash
# อนุญาต Zig ผ่าน Gatekeeper
xattr -d com.apple.quarantine /usr/local/zig/zig

# หรือผ่าน System Preferences
# System Preferences -> Security & Privacy -> Allow anyway
```

---

> **หมายเหตุสำหรับ Apple Silicon**: Zig รองรับ Apple Silicon (ARM) อย่างสมบูรณ์
> ประสิทธิภาพบน M1/M2/M3 ดีเยี่ยม

---

<a name="step-5"></a>
## Step 5: การติดตั้ง Zig บน Windows

### วิธีที่ 1: ดาวน์โหลดจาก Official Website (แนะนำ)

1. เปิด browser ไปที่ [https://ziglang.org/download/](https://ziglang.org/download/)
2. ดาวน์โหลด `zig-windows-x86_64-0.13.0.zip`
3. แตกไฟล์ไปยัง `C:\zig\` (หรือ path ที่ต้องการ)
4. เพิ่ม `C:\zig` ใน PATH

#### การเพิ่ม PATH บน Windows (ผ่าน GUI)

1. กด `Win + X` แล้วเลือก "System"
2. คลิก "Advanced system settings"
3. คลิก "Environment Variables"
4. ใน "System variables" ค้นหา "Path" แล้วคลิก "Edit"
5. คลิก "New" แล้วพิมพ์ `C:\zig`
6. คลิก "OK" ทุกหน้าต่าง

#### การเพิ่ม PATH บน Windows (ผ่าน PowerShell)

```powershell
# เปิด PowerShell ในฐานะ Administrator
$env:Path += ";C:\zig"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\zig", "Machine")
```

### วิธีที่ 2: Winget (Windows Package Manager)

```powershell
# Windows 10 1709+ และ Windows 11
winget install zig.zig
```

### วิธีที่ 3: Scoop

```powershell
# ติดตั้ง Scoop ถ้ายังไม่มี
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex

# ติดตั้ง Zig
scoop install zig
```

### วิธีที่ 4: Chocolatey

```powershell
# ต้องติดตั้ง Chocolatey ก่อน
choco install zig
```

### WSL (Windows Subsystem for Linux)

ถ้าคุณใช้ WSL แนะนำให้ติดตั้ง Zig ภายใน WSL ตามวิธี Linux:

```bash
# ใน WSL terminal
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar -xf zig-linux-x86_64-0.13.0.tar.xz
sudo mv zig-linux-x86_64-0.13.0 /usr/local/zig
echo 'export PATH="$PATH:/usr/local/zig"' >> ~/.bashrc
source ~/.bashrc
```

---

> **เคล็ดลับ Windows**: แนะนำให้ใช้ Windows Terminal พร้อม PowerShell หรือ Git Bash
> เพื่อประสบการณ์การพัฒนา Zig ที่ดีขึ้น

---

<a name="step-6"></a>
## Step 6: การตรวจสอบการติดตั้ง และ zig version

### ตรวจสอบ Version

หลังจากติดตั้งเสร็จ ให้เปิด terminal ใหม่แล้วรัน:

```bash
zig version
```

ผลลัพธ์ที่คาดหวัง:
```
0.13.0
```

### ตรวจสอบว่า Zig ทำงานได้

สร้างไฟล์ `hello.zig` และทดสอบ:

```zig
// hello.zig — โปรแกรมทดสอบแรก
const std = @import("std");

pub fn main() void {
    std.debug.print("สวัสดี Zig!\n", .{});
}
```

```bash
# รันโดยตรง
zig run hello.zig
```

ผลลัพธ์:
```
สวัสดี Zig!
```

### ตรวจสอบ Target ที่รองรับ

```bash
# ดู target ทั้งหมดที่ Zig รองรับ
zig targets

# ดู CPU ที่รองรับ
zig targets | grep -A 5 "cpu:"
```

### ตรวจสอบ Standard Library

```bash
# ดู path ของ Zig standard library
zig env
```

ผลลัพธ์ตัวอย่าง:
```json
{
 "zig_exe": "/usr/local/zig/zig",
 "lib_dir": "/usr/local/zig/lib",
 "std_dir": "/usr/local/zig/lib/std",
 "global_cache_dir": "/home/user/.cache/zig",
 "local_cache_dir": "/home/user/project/.zig-cache",
 "version": "0.13.0"
}
```

### ปัญหาที่พบบ่อยและวิธีแก้ไข

#### ปัญหา: "zig: command not found"

```bash
# ตรวจสอบว่า PATH ถูกต้อง
echo $PATH

# ตรวจสอบว่า Zig อยู่ที่ไหน
which zig          # Linux/macOS
where zig          # Windows

# ถ้าไม่พบ ให้เพิ่ม PATH อีกครั้ง
export PATH="$PATH:/usr/local/zig"  # แก้ path ให้ถูกต้อง
```

#### ปัญหา: Permission denied บน Linux/macOS

```bash
# ตรวจสอบ permission
ls -la /usr/local/zig/zig

# แก้ไข permission
chmod +x /usr/local/zig/zig
```

#### ปัญหา: Library not found บน Linux

```bash
# ติดตั้ง dependencies ที่จำเป็น
# Ubuntu/Debian
sudo apt-get install libc6

# Fedora
sudo dnf install glibc
```

---

<a name="step-7"></a>
## Step 7: การตั้งค่า Editor

### Visual Studio Code (แนะนำสำหรับผู้เริ่มต้น)

VS Code เป็น editor ที่นิยมที่สุดสำหรับ Zig ด้วย extension ที่ดี

#### การติดตั้ง Extension

1. เปิด VS Code
2. กด `Ctrl+Shift+X` (หรือ `Cmd+Shift+X` บน macOS)
3. ค้นหา "Zig Language" โดย Ziglang
4. คลิก Install

หรือผ่าน command line:

```bash
code --install-extension ziglang.vscode-zig
```

#### การตั้งค่า VS Code สำหรับ Zig

เพิ่มใน `settings.json` (`Ctrl+Shift+P` -> "Open User Settings JSON"):

```json
{
    "zig.zls.path": "/usr/local/zls/zls",
    "zig.path": "/usr/local/zig/zig",
    "[zig]": {
        "editor.formatOnSave": true,
        "editor.tabSize": 4,
        "editor.insertSpaces": true
    }
}
```

#### ติดตั้ง ZLS (Zig Language Server)

ZLS ให้ฟีเจอร์ต่าง ๆ เช่น auto-complete, go-to-definition, error checking

```bash
# วิธีที่ 1: ดาวน์โหลดจาก GitHub releases
# https://github.com/zigtools/zls/releases

# วิธีที่ 2: Build จาก source
git clone https://github.com/zigtools/zls
cd zls
zig build -Doptimize=ReleaseSafe

# ย้าย binary ไป PATH
sudo mv zig-out/bin/zls /usr/local/bin/
```

### Neovim

Neovim เหมาะสำหรับผู้ใช้ที่ชอบ keyboard-driven workflow

#### ใช้ lazy.nvim

```lua
-- ~/.config/nvim/lua/plugins/zig.lua
return {
    -- Zig syntax highlighting
    {
        "ziglang/zig.vim",
        ft = "zig",
    },
    -- LSP configuration
    {
        "neovim/nvim-lspconfig",
        config = function()
            require("lspconfig").zls.setup({
                cmd = { "zls" },
                filetypes = { "zig", "zir" },
                root_dir = require("lspconfig").util.root_pattern("build.zig"),
            })
        end,
    },
}
```

#### ตั้งค่า LSP สำหรับ Neovim

```lua
-- ใน init.lua หรือ lua/lsp.lua
local lspconfig = require('lspconfig')

lspconfig.zls.setup({
    on_attach = function(client, bufnr)
        -- keymaps สำหรับ LSP
        local opts = { buffer = bufnr }
        vim.keymap.set('n', 'gd', vim.lsp.buf.definition, opts)
        vim.keymap.set('n', 'K', vim.lsp.buf.hover, opts)
        vim.keymap.set('n', '<leader>rn', vim.lsp.buf.rename, opts)
    end,
})
```

### Vim

```vim
" ติดตั้ง vim-plug ถ้ายังไม่มี
" เพิ่มใน ~/.vimrc

call plug#begin()
Plug 'ziglang/zig.vim'
call plug#end()

" ตั้งค่า
let g:zig_fmt_autosave = 1  " auto format เมื่อ save
```

### JetBrains IDEs (CLion, IntelliJ)

ติดตั้ง plugin "Zig" จาก JetBrains Marketplace:

1. `File` -> `Settings` -> `Plugins`
2. ค้นหา "Zig"
3. ติดตั้ง plugin โดย `@nishtahir`

### Emacs

```elisp
;; ใช้ use-package
(use-package zig-mode
  :ensure t
  :mode "\\.zig\\'"
  :config
  (add-hook 'zig-mode-hook 'lsp))

;; ตั้งค่า LSP
(use-package lsp-mode
  :ensure t
  :hook (zig-mode . lsp))
```

### Sublime Text

ติดตั้ง Package Control แล้ว:
1. `Ctrl+Shift+P` -> "Package Control: Install Package"
2. ค้นหา "Zig"
3. ติดตั้ง

---

> **แนะนำ**: ถ้าคุณไม่แน่ใจจะเลือก editor ไหน ให้เริ่มด้วย VS Code
> เพราะมี extension ที่ดีที่สุดและใช้งานง่ายที่สุดสำหรับ Zig

---

<a name="step-8"></a>
## Step 8: โครงสร้างโปรเจค Zig เบื้องต้น

### โปรเจคแบบง่าย (Single File)

สำหรับโปรแกรมเล็ก ๆ สามารถมีไฟล์เดียว:

```
my_program/
└── main.zig
```

```zig
// main.zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello from single file!\n", .{});
}
```

```bash
zig run main.zig
```

### โปรเจคมาตรฐาน (ด้วย build.zig)

```
my_project/
├── build.zig          # Build configuration
├── build.zig.zon      # Package manifest (Zig 0.12+)
├── src/
│   ├── main.zig       # Entry point
│   ├── utils.zig      # Utility functions
│   └── math.zig       # Math functions
└── tests/
    └── test_math.zig  # Test files
```

### โครงสร้างของ build.zig

```zig
// build.zig — ไฟล์กำหนดการ build โปรเจค
const std = @import("std");

pub fn build(b: *std.Build) void {
    // กำหนด optimization level และ target
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // สร้าง executable
    const exe = b.addExecutable(.{
        .name = "my_project",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // ติดตั้ง executable
    b.installArtifact(exe);

    // สร้าง run command
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    // เพิ่ม "zig build run" command
    const run_step = b.step("run", "Run the application");
    run_step.dependOn(&run_cmd.step);

    // สร้าง test executable
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // เพิ่ม "zig build test" command
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

### โครงสร้างของ build.zig.zon

```zig
// build.zig.zon — Package manifest
.{
    .name = "my_project",
    .version = "0.1.0",
    .minimum_zig_version = "0.13.0",
    .dependencies = .{
        // เพิ่ม dependencies ที่นี่
        // .some_lib = .{
        //     .url = "https://github.com/...",
        //     .hash = "...",
        // },
    },
    .paths = .{
        "build.zig",
        "build.zig.zon",
        "src",
    },
}
```

### โครงสร้างไฟล์ต่าง ๆ ใน src/

```zig
// src/main.zig — Entry point
const std = @import("std");
const utils = @import("utils.zig");
const math = @import("math.zig");

pub fn main() !void {
    const result = math.add(10, 20);
    utils.printResult(result);
}
```

```zig
// src/utils.zig — Utility functions
const std = @import("std");

pub fn printResult(value: i32) void {
    std.debug.print("ผลลัพธ์: {d}\n", .{value});
}

pub fn printBanner(title: []const u8) void {
    std.debug.print("=== {s} ===\n", .{title});
}
```

```zig
// src/math.zig — Math functions
pub fn add(a: i32, b: i32) i32 {
    return a + b;
}

pub fn multiply(a: i32, b: i32) i32 {
    return a * b;
}

// Test สำหรับฟังก์ชัน math
test "add function" {
    const result = add(2, 3);
    try @import("std").testing.expectEqual(@as(i32, 5), result);
}
```

### โครงสร้างโปรเจคขนาดใหญ่

```
large_project/
├── build.zig
├── build.zig.zon
├── src/
│   ├── main.zig
│   ├── core/
│   │   ├── engine.zig
│   │   └── config.zig
│   ├── network/
│   │   ├── http.zig
│   │   └── tcp.zig
│   └── utils/
│       ├── string.zig
│       └── memory.zig
├── tests/
│   ├── core_test.zig
│   └── network_test.zig
├── examples/
│   ├── simple_server.zig
│   └── cli_app.zig
└── docs/
    └── api.md
```

### ไฟล์ Output หลัง Build

```
my_project/
├── .zig-cache/         # Cache ของ Zig compiler (อย่า commit)
│   ├── h/
│   └── o/
└── zig-out/            # Output files (อาจ commit หรือไม่ก็ได้)
    ├── bin/
    │   └── my_project  # Executable
    └── lib/
        └── libmy.a     # Library (ถ้า build เป็น lib)
```

### .gitignore สำหรับ Zig

```gitignore
# Zig build artifacts
zig-out/
.zig-cache/

# Editor files
.vscode/
*.swp
*.swo
.DS_Store
```

---

<a name="step-9"></a>
## Step 9: การใช้ zig init และ zig build

### zig init — สร้างโปรเจคใหม่

```bash
# สร้างโฟลเดอร์และเข้าไป
mkdir my_first_project
cd my_first_project

# Initialize โปรเจค Zig
zig init
```

คำสั่งนี้จะสร้างไฟล์:
```
my_first_project/
├── build.zig
├── build.zig.zon
└── src/
    ├── main.zig
    └── root.zig
```

ตรวจสอบ `src/main.zig` ที่ถูกสร้างมา:

```zig
// src/main.zig ที่ถูกสร้างโดย zig init
const std = @import("std");

pub fn main() !void {
    std.debug.print("All your {s} are belong to us.\n", .{"codebase"});
}
```

### zig build — Build โปรเจค

```bash
# Build โปรเจค (default mode)
zig build

# Build แบบ Debug (default)
zig build -Doptimize=Debug

# Build แบบ ReleaseSafe (แนะนำสำหรับ production)
zig build -Doptimize=ReleaseSafe

# Build แบบ ReleaseFast (เร็วสุด ไม่มี safety checks)
zig build -Doptimize=ReleaseFast

# Build แบบ ReleaseSmall (ขนาดเล็กสุด)
zig build -Doptimize=ReleaseSmall
```

### Optimization Modes

| Mode | ความเร็ว | ขนาด | Safety Checks | ใช้เมื่อ |
|------|----------|------|---------------|---------|
| Debug | ช้า | ใหญ่ | เต็ม | Development |
| ReleaseSafe | เร็ว | กลาง | บางส่วน | Production (แนะนำ) |
| ReleaseFast | เร็วสุด | กลาง | ไม่มี | Performance-critical |
| ReleaseSmall | ปานกลาง | เล็กสุด | บางส่วน | Embedded, WASM |

### zig build run — Build และรันพร้อมกัน

```bash
# Build และรัน
zig build run

# ส่ง arguments ไปยังโปรแกรม
zig build run -- arg1 arg2 arg3
```

### zig build test — รัน Tests

```bash
# รัน tests ทั้งหมด
zig build test

# รัน tests พร้อม verbose output
zig build test --summary all
```

### Custom Build Steps

สามารถเพิ่ม custom step ใน build.zig:

```zig
// build.zig — เพิ่ม custom steps
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "my_project",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    // สร้าง "zig build run" step
    const run_cmd = b.addRunArtifact(exe);
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    // สร้าง custom "clean" step
    const clean_step = b.step("clean", "Remove build artifacts");
    const clean_cmd = b.addSystemCommand(&[_][]const u8{
        "rm", "-rf", "zig-out", ".zig-cache",
    });
    clean_step.dependOn(&clean_cmd.step);
}
```

```bash
# ใช้ custom step
zig build clean
```

### ดู Build Steps ที่มีอยู่

```bash
# แสดง help ของ build
zig build --help
```

ผลลัพธ์ตัวอย่าง:
```
Usage: zig build [steps] [options]

Steps:
  install (default)    Copy build artifacts to prefix path
  uninstall            Remove build artifacts from prefix path
  run                  Run the application
  test                 Run unit tests

General Options:
  -p, --prefix [path]  Override default install prefix
  ...
```

---

<a name="step-10"></a>
## Step 10: การเรียกใช้ zig run และ zig build-exe

### zig run — รันไฟล์โดยตรง

`zig run` เหมาะสำหรับการทดสอบโค้ดอย่างรวดเร็ว โดยไม่ต้องสร้างโปรเจค

```bash
# รันไฟล์ Zig โดยตรง
zig run main.zig

# ส่ง arguments ไปยังโปรแกรม
zig run main.zig -- hello world

# รันพร้อม optimization
zig run -O ReleaseFast main.zig
```

ตัวอย่างโค้ดที่รับ arguments:

```zig
// main.zig — รับ command line arguments
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // รับ args
    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    std.debug.print("จำนวน arguments: {d}\n", .{args.len});
    for (args, 0..) |arg, i| {
        std.debug.print("  args[{d}] = {s}\n", .{ i, arg });
    }
}
```

```bash
zig run main.zig -- foo bar baz
```

ผลลัพธ์:
```
จำนวน arguments: 4
  args[0] = /tmp/zig-run-xxx/main
  args[1] = foo
  args[2] = bar
  args[3] = baz
```

### zig build-exe — Compile เป็น Executable

`zig build-exe` คอมไพล์ไฟล์เป็น executable โดยตรง (ไม่ใช้ build.zig)

```bash
# สร้าง executable ชื่อ "main"
zig build-exe main.zig

# กำหนดชื่อ output
zig build-exe main.zig -femit-bin=my_program

# Build สำหรับ platform อื่น (cross-compilation)
zig build-exe main.zig -target x86_64-linux

# Build แบบ static linking
zig build-exe main.zig -target x86_64-linux-musl

# Build พร้อม optimization
zig build-exe main.zig -O ReleaseFast
```

### zig build-lib — Compile เป็น Library

```bash
# สร้าง static library
zig build-lib mylib.zig

# สร้าง shared library (dynamic)
zig build-lib mylib.zig -dynamic

# กำหนดชื่อ output
zig build-lib mylib.zig -femit-bin=libmylib.a
```

### zig build-obj — Compile เป็น Object File

```bash
# สร้าง object file
zig build-obj main.zig

# ใช้ object file กับ linker อื่น
zig build-obj main.zig -femit-bin=main.o
gcc main.o -o my_program -lc
```

### เปรียบเทียบ zig run vs zig build

| คุณสมบัติ | zig run | zig build |
|-----------|---------|-----------|
| ความเร็วในการใช้ | เร็ว | ต้องมี build.zig |
| Caching | ไม่มี | มี (เร็วขึ้นมาก) |
| เหมาะกับ | Testing, scripts | Projects |
| Output file | ไม่สร้าง | สร้างใน zig-out/ |
| Dependencies | จำกัด | เต็มรูปแบบ |

### ตัวอย่างโปรแกรม Hello World ฉบับสมบูรณ์

```zig
// complete_hello.zig
const std = @import("std");

pub fn main() !void {
    // ใช้ stdout สำหรับ output
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("สวัสดีชาวโลก!\n", .{});
    try stdout.print("ยินดีต้อนรับสู่ Zig Programming\n", .{});
    
    // แสดงข้อมูล environment
    try stdout.print("\nข้อมูล:\n", .{});
    try stdout.print("  OS: {s}\n", .{@tagName(@import("builtin").os.tag)});
    try stdout.print("  CPU: {s}\n", .{@tagName(@import("builtin").cpu.arch)});
}
```

```bash
zig run complete_hello.zig
```

ผลลัพธ์:
```
สวัสดีชาวโลก!
ยินดีต้อนรับสู่ Zig Programming

ข้อมูล:
  OS: linux
  CPU: x86_64
```

### การ Debug โปรแกรม

```bash
# Build แบบ Debug (default)
zig build-exe -O Debug main.zig

# รันด้วย GDB
gdb ./main

# รันด้วย LLDB
lldb ./main

# ใช้ Address Sanitizer
zig run main.zig -fsanitize-c
```

### Cross Compilation ตัวอย่าง

```bash
# คอมไพล์ไปทุก platform หลัก
zig build-exe main.zig -target x86_64-linux-musl    # Linux static
zig build-exe main.zig -target x86_64-windows       # Windows
zig build-exe main.zig -target x86_64-macos         # macOS Intel
zig build-exe main.zig -target aarch64-macos        # macOS Apple Silicon
zig build-exe main.zig -target aarch64-linux        # Linux ARM64
zig build-exe main.zig -target wasm32-freestanding  # WebAssembly
```

---

> **เคล็ดลับ Cross Compilation**: Zig มี C standard library (musl) built-in
> ทำให้ cross-compile ได้โดยไม่ต้องติดตั้ง toolchain พิเศษ
> ใช้ `-target x86_64-linux-musl` เพื่อ binary ที่ทำงานได้บน Linux ทุก distro

---

<a name="exercises"></a>
## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: การติดตั้งและทดสอบ

**เป้าหมาย**: ติดตั้ง Zig และยืนยันว่าทำงานได้

1. ติดตั้ง Zig บน OS ของคุณ
2. รันคำสั่ง `zig version` และจดบันทึก version
3. สร้างไฟล์ `test_install.zig` ที่แสดงข้อความ "Zig ทำงานได้แล้ว!"
4. รันด้วย `zig run test_install.zig`

```zig
// แนวทางคำตอบ
const std = @import("std");

pub fn main() void {
    std.debug.print("Zig ทำงานได้แล้ว!\n", .{});
    std.debug.print("Version ที่ติดตั้ง: ตรวจสอบด้วย 'zig version'\n", .{});
}
```

### แบบฝึกหัดที่ 2: สร้างโปรเจคแรก

**เป้าหมาย**: ใช้ `zig init` สร้างและรันโปรเจค

1. สร้างโฟลเดอร์ `my_zig_project`
2. รัน `zig init` ภายในโฟลเดอร์
3. ตรวจสอบไฟล์ที่ถูกสร้าง
4. รัน `zig build run`
5. แก้ไข `src/main.zig` ให้แสดงชื่อของคุณ

```zig
// src/main.zig ที่แก้ไขแล้ว
const std = @import("std");

pub fn main() !void {
    std.debug.print("สวัสดี! ฉันชื่อ [ชื่อของคุณ]\n", .{});
    std.debug.print("นี่คือโปรเจค Zig แรกของฉัน!\n", .{});
}
```

### แบบฝึกหัดที่ 3: Cross Compilation

**เป้าหมาย**: ทดสอบ cross-compilation

1. สร้างไฟล์ `greet.zig` ง่าย ๆ
2. คอมไพล์สำหรับ Linux: `zig build-exe greet.zig -target x86_64-linux-musl`
3. คอมไพล์สำหรับ Windows: `zig build-exe greet.zig -target x86_64-windows`
4. สังเกตขนาดไฟล์ที่ได้

```zig
// greet.zig
const std = @import("std");

pub fn main() void {
    std.debug.print("สวัสดีจาก {s}!\n", .{@tagName(@import("builtin").os.tag)});
}
```

### แบบฝึกหัดที่ 4: ตั้งค่า Editor

**เป้าหมาย**: ตั้งค่า VS Code สำหรับ Zig

1. ติดตั้ง VS Code extension "Zig Language"
2. ติดตั้ง ZLS (Zig Language Server)
3. ตั้งค่า auto-format on save
4. เขียนโค้ดและทดสอบ auto-complete

### แบบฝึกหัดที่ 5: โปรแกรม Calculator เล็ก ๆ

**เป้าหมาย**: สร้างโปรเจคที่ใช้หลาย files

สร้างโปรเจค calculator ที่มีโครงสร้าง:
```
calculator/
├── build.zig
├── build.zig.zon
└── src/
    ├── main.zig
    └── math.zig
```

```zig
// src/math.zig
pub fn add(a: f64, b: f64) f64 {
    return a + b;
}

pub fn subtract(a: f64, b: f64) f64 {
    return a - b;
}

pub fn multiply(a: f64, b: f64) f64 {
    return a * b;
}

pub fn divide(a: f64, b: f64) !f64 {
    if (b == 0.0) return error.DivisionByZero;
    return a / b;
}

test "basic arithmetic" {
    const testing = @import("std").testing;
    try testing.expectEqual(@as(f64, 5.0), add(2.0, 3.0));
    try testing.expectEqual(@as(f64, 6.0), multiply(2.0, 3.0));
}
```

```zig
// src/main.zig
const std = @import("std");
const math = @import("math.zig");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("Calculator Zig\n", .{});
    try stdout.print("10 + 5 = {d}\n", .{math.add(10, 5)});
    try stdout.print("10 - 5 = {d}\n", .{math.subtract(10, 5)});
    try stdout.print("10 * 5 = {d}\n", .{math.multiply(10, 5)});
    
    const div_result = try math.divide(10, 5);
    try stdout.print("10 / 5 = {d}\n", .{div_result});
    
    // ทดสอบ division by zero
    const err_result = math.divide(10, 0);
    if (err_result) |_| {
        try stdout.print("ไม่ควรถึงตรงนี้\n", .{});
    } else |err| {
        try stdout.print("Error: {}\n", .{err});
    }
}
```

---

<a name="summary"></a>
## สรุป (Summary)

ในส่วนที่ 1 นี้ เราได้เรียนรู้:

### สิ่งที่เรียนรู้ไปแล้ว

| Step | หัวข้อ | สิ่งสำคัญที่ต้องจำ |
|------|--------|-------------------|
| 1 | Zig คืออะไร | ภาษา systems programming ที่เน้น simplicity และ explicitness |
| 2 | ทำไมเรียน Zig | ง่ายกว่า C++, safe กว่า C, cross-compile ง่าย |
| 3 | ติดตั้งบน Linux | ดาวน์โหลด tarball, แตกไฟล์, เพิ่ม PATH |
| 4 | ติดตั้งบน macOS | Homebrew หรือดาวน์โหลดโดยตรง |
| 5 | ติดตั้งบน Windows | winget, scoop, หรือดาวน์โหลดโดยตรง |
| 6 | ตรวจสอบการติดตั้ง | `zig version`, `zig env` |
| 7 | ตั้งค่า Editor | VS Code + ZLS extension |
| 8 | โครงสร้างโปรเจค | build.zig, build.zig.zon, src/ |
| 9 | zig init & build | สร้างโปรเจค, build modes |
| 10 | zig run & build-exe | รันไฟล์โดยตรง, cross-compilation |

### คำสั่งสำคัญที่ต้องจำ

```bash
# ตรวจสอบ version
zig version

# รันไฟล์โดยตรง
zig run main.zig

# สร้างโปรเจคใหม่
zig init

# Build โปรเจค
zig build

# Build และรัน
zig build run

# รัน tests
zig build test

# คอมไพล์เป็น executable
zig build-exe main.zig

# Cross compile
zig build-exe main.zig -target x86_64-linux-musl
```

### ปรัชญาหลักของ Zig ที่ต้องจำ

1. **ไม่มี hidden control flow** — ทุกอย่างต้องชัดเจน
2. **ไม่มี hidden memory allocations** — ต้อง pass allocator เสมอ
3. **ไม่มี preprocessor** — ใช้ comptime แทน
4. **Cross-compilation เป็น first-class feature**

### ขั้นตอนต่อไป

ใน Part 02 เราจะเรียนรู้:
- ประเภทข้อมูลพื้นฐาน (Basic Types)
- Variables และ Constants
- Operators
- การควบคุมการไหลของโปรแกรม (Control Flow)

---

### แหล่งข้อมูลเพิ่มเติม

| ทรัพยากร | URL |
|---------|-----|
| Official Website | https://ziglang.org |
| Official Docs | https://ziglang.org/documentation/master/ |
| Standard Library Docs | https://ziglang.org/documentation/master/std/ |
| Ziglearn.org | https://ziglearn.org |
| Zig News | https://zig.news |
| GitHub | https://github.com/ziglang/zig |
| Discord | https://discord.gg/gxsFFjE |

---

> **หมายเหตุสำคัญ**: Zig ยังอยู่ในช่วงพัฒนา API อาจเปลี่ยนแปลงได้
> ตรวจสอบ [release notes](https://ziglang.org/download/) เสมอเมื่ออัปเดต version

---

*Part 01 | การติดตั้งและเริ่มต้นใช้งาน Zig | จบสมบูรณ์*

*ถัดไป → [Part 02: ประเภทข้อมูลและตัวแปร]*
