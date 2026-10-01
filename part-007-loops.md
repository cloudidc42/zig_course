# ภาค 7: การวนลูปใน Zig (Steps 61-70)

> **ซีรีส์**: เรียน Zig ตั้งแต่ศูนย์  
> **ระดับ**: ผู้เริ่มต้น → กลาง  
> **ข้อกำหนดเบื้องต้น**: ตัวแปร, ประเภทข้อมูล, เงื่อนไข (ภาค 1-6)

---

## บทนำ: ทำไม Loop ใน Zig ถึงแตกต่าง?

ก่อนที่เราจะเริ่มเรียน loop ใน Zig มาดูก่อนว่า Zig มีแนวคิดอะไรที่แตกต่างจากภาษาอื่น

ใน **C**:
```c
// C - for loop ธรรมดา
for (int i = 0; i < 10; i++) {
    printf("%d\n", i);
}
```

ใน **Python**:
```python
# Python - for loop ที่อ่านง่าย
for i in range(10):
    print(i)
```

ใน **Zig**:
```zig
// Zig - มีทั้ง while และ for แต่มีความสามารถพิเศษ
var i: usize = 0;
while (i < 10) : (i += 1) {
    std.debug.print("{d}\n", .{i});
}

// หรือใช้ for กับ range
for (0..10) |i| {
    std.debug.print("{d}\n", .{i});
}
```

Zig มีคุณสมบัติพิเศษของ loop ที่น่าสนใจ:

1. **`while` กับ continue expression** - แยกส่วน "update" ออกจาก body
2. **`while` กับ optional** - unwrap optional อัตโนมัติ
3. **`for` กับ multiple arrays** - zip หลาย array พร้อมกัน
4. **`inline for/while`** - loop ที่ compile-time
5. **Loop เป็น expression** - ใช้ `break` เพื่อส่งค่า return

---

## Step 61: while loop พื้นฐาน

### รูปแบบพื้นฐาน

`while` loop ใน Zig มีรูปแบบง่ายที่สุดคือ:

```zig
while (condition) {
    // body
}
```

**ตัวอย่าง 61.1: while loop นับเลข**

```zig
const std = @import("std");

pub fn main() void {
    var i: u32 = 0;
    while (i < 5) {
        std.debug.print("i = {d}\n", .{i});
        i += 1;
    }
}
```

**ผลลัพธ์:**
```
i = 0
i = 1
i = 2
i = 3
i = 4
```

**ตัวอย่าง 61.2: while loop ถอยหลัง**

```zig
const std = @import("std");

pub fn main() void {
    var count: i32 = 10;
    while (count > 0) {
        std.debug.print("นับถอยหลัง: {d}\n", .{count});
        count -= 1;
    }
    std.debug.print("ปล่อย!\n", .{});
}
```

**ผลลัพธ์:**
```
นับถอยหลัง: 10
นับถอยหลัง: 9
...
นับถอยหลัง: 1
ปล่อย!
```

**ตัวอย่าง 61.3: while loop กับเงื่อนไขซับซ้อน**

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 1;
    var sum: i32 = 0;

    // หาผลรวมของเลขที่หาร 3 ลงตัว ตั้งแต่ 1 ถึง 100
    while (x <= 100) {
        if (x % 3 == 0) {
            sum += x;
        }
        x += 1;
    }

    std.debug.print("ผลรวมของเลขที่หาร 3 ลงตัว (1-100) = {d}\n", .{sum});
}
```

**ผลลัพธ์:**
```
ผลรวมของเลขที่หาร 3 ลงตัว (1-100) = 1683
```

### ข้อแตกต่างจาก C

ใน C เราสามารถเขียน:
```c
// C - อันตราย! ไม่มี warning ถ้าลืม braces
while (i < 10)
    printf("%d\n", i++);
```

ใน Zig **ต้องมี braces เสมอ**:
```zig
// Zig - บังคับใช้ braces
while (i < 10) {
    std.debug.print("{d}\n", .{i});
    i += 1;
}
// ถ้าไม่มี braces จะ compile error
```

### infinite loop

```zig
const std = @import("std");

pub fn main() void {
    var count: u32 = 0;

    // infinite loop - ต้องใช้ break เพื่อออก
    while (true) {
        count += 1;
        if (count >= 5) break;
    }

    std.debug.print("นับได้ {d} ครั้ง\n", .{count});
}
```

---

## Step 62: while กับ continue expression

นี่คือหนึ่งในคุณสมบัติที่ **Zig แตกต่างจาก C** อย่างชัดเจน

### รูปแบบ

```zig
while (condition) : (continue_expression) {
    // body
}
```

`continue_expression` คือโค้ดที่จะรันทุกครั้งที่ iteration จบลง (ทั้งแบบปกติ และแบบที่ใช้ `continue`)

### ทำไมถึงมีฟีเจอร์นี้?

ใน C, `for` loop มี 3 ส่วน:
```c
for (init; condition; update) {
    // ถ้าใช้ continue ส่วน update ยังคงทำงาน
}
```

แต่ใน C เมื่อใช้ `continue` ใน `while`:
```c
int i = 0;
while (i < 10) {
    if (i == 5) {
        continue; // อันตราย! i ไม่ถูก increment -> infinite loop!
    }
    i++;
}
```

Zig แก้ปัญหานี้ด้วย continue expression:

**ตัวอย่าง 62.1: while กับ continue expression พื้นฐาน**

```zig
const std = @import("std");

pub fn main() void {
    var i: u32 = 0;

    // continue expression (i += 1) จะรันทุก iteration
    while (i < 10) : (i += 1) {
        if (i == 5) continue; // ข้าม 5 แต่ i ยังถูก +1
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
0 1 2 3 4 6 7 8 9
```

**ตัวอย่าง 62.2: เปรียบเทียบกับ C**

```zig
// Zig - ปลอดภัย
var i: u32 = 0;
while (i < 10) : (i += 1) {
    if (i % 2 == 0) continue; // ข้ามเลขคู่
    std.debug.print("{d} ", .{i}); // แสดงเฉพาะเลขคี่
}
```

```c
// C - ต้องระวัง!
int i = 0;
while (i < 10) {
    if (i % 2 == 0) {
        i++; // ต้องไม่ลืม increment ก่อน continue
        continue;
    }
    printf("%d ", i);
    i++;
}
```

**ตัวอย่าง 62.3: continue expression ที่ซับซ้อน**

```zig
const std = @import("std");

pub fn main() void {
    var x: i32 = 1;
    var y: i32 = 0;

    // continue expression สามารถมีหลาย statement ได้
    while (x <= 20) : ({
        x += 1;
        y = x * x; // คำนวณ y ทุก iteration
    }) {
        if (y > 100) continue;
        std.debug.print("x={d}, x²={d}\n", .{x, y});
    }
}
```

**ตัวอย่าง 62.4: นับจำนวนตัวอักษร**

```zig
const std = @import("std");

pub fn countChar(text: []const u8, target: u8) usize {
    var i: usize = 0;
    var count: usize = 0;

    while (i < text.len) : (i += 1) {
        if (text[i] == target) count += 1;
    }

    return count;
}

pub fn main() void {
    const sentence = "สวัสดีครับ ยินดีต้อนรับสู่ Zig";
    const vowelA = 'i';
    const result = countChar(sentence, vowelA);
    std.debug.print("จำนวน '{c}' ใน string: {d}\n", .{vowelA, result});
}
```

### continue expression กับ Fibonacci

**ตัวอย่าง 62.5: Fibonacci ด้วย while และ continue expression**

```zig
const std = @import("std");

pub fn main() void {
    var a: u64 = 0;
    var b: u64 = 1;
    var iterations: u32 = 0;

    std.debug.print("Fibonacci: ", .{});

    while (b < 1000) : ({
        const temp = b;
        b = a + b;
        a = temp;
        iterations += 1;
    }) {
        std.debug.print("{d} ", .{b});
    }

    std.debug.print("\nวนรอบทั้งหมด: {d}\n", .{iterations});
}
```

**ผลลัพธ์:**
```
Fibonacci: 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
วนรอบทั้งหมด: 16
```

---

## Step 63: while กับ Optional

Zig มีฟีเจอร์พิเศษที่ไม่มีในภาษาอื่น: **`while` กับ optional type**

### รูปแบบ

```zig
while (optional_value) |unwrapped| {
    // ใช้ unwrapped ที่ไม่มี null
}
```

Loop จะหยุดทันทีที่ค่า optional กลายเป็น `null`

### ทำไมถึงมีประโยชน์?

เป็น pattern ทั่วไปสำหรับ:
- อ่านข้อมูลจาก iterator
- ประมวลผล stream ของข้อมูล
- ดึงค่าจาก buffer ทีละค่า

**ตัวอย่าง 63.1: Iterator แบบง่าย**

```zig
const std = @import("std");

// Simple iterator struct
const NumberIterator = struct {
    current: u32,
    max: u32,

    pub fn next(self: *NumberIterator) ?u32 {
        if (self.current >= self.max) return null;
        const val = self.current;
        self.current += 1;
        return val;
    }
};

pub fn main() void {
    var iter = NumberIterator{ .current = 0, .max = 5 };

    // while กับ optional - หยุดเมื่อ next() return null
    while (iter.next()) |value| {
        std.debug.print("ค่า: {d}\n", .{value});
    }
}
```

**ผลลัพธ์:**
```
ค่า: 0
ค่า: 1
ค่า: 2
ค่า: 3
ค่า: 4
```

**ตัวอย่าง 63.2: อ่านจาก ArrayList**

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    try list.appendSlice(&[_]i32{ 10, 20, 30, 40, 50 });

    // ใช้ iterator ของ ArrayList
    var iter = list.items;
    var idx: usize = 0;

    // วิธีดั้งเดิม
    while (idx < iter.len) : (idx += 1) {
        std.debug.print("items[{d}] = {d}\n", .{idx, iter[idx]});
    }
}
```

**ตัวอย่าง 63.3: Optional กับ string parsing**

```zig
const std = @import("std");

// หาตำแหน่งของตัวอักษรใน string
fn findNext(text: []const u8, start: usize, target: u8) ?usize {
    var i = start;
    while (i < text.len) : (i += 1) {
        if (text[i] == target) return i;
    }
    return null;
}

pub fn main() void {
    const csv = "apple,banana,cherry,date";
    var pos: usize = 0;
    var word_count: usize = 0;

    std.debug.print("คำใน CSV:\n", .{});

    // วนหาทุก comma
    while (findNext(csv, pos, ',')) |comma_pos| {
        std.debug.print("  [{d}] '{s}'\n", .{word_count, csv[pos..comma_pos]});
        word_count += 1;
        pos = comma_pos + 1;
    }

    // คำสุดท้าย
    std.debug.print("  [{d}] '{s}'\n", .{word_count, csv[pos..]});
    std.debug.print("จำนวนคำทั้งหมด: {d}\n", .{word_count + 1});
}
```

**ผลลัพธ์:**
```
คำใน CSV:
  [0] 'apple'
  [1] 'banana'
  [2] 'cherry'
จำนวนคำทั้งหมด: 4
```

**ตัวอย่าง 63.4: while กับ optional และ else**

```zig
const std = @import("std");

fn getNextPrime(n: u32) ?u32 {
    var candidate = n + 1;
    while (candidate < 1000) : (candidate += 1) {
        var is_prime = true;
        var div: u32 = 2;
        while (div * div <= candidate) : (div += 1) {
            if (candidate % div == 0) {
                is_prime = false;
                break;
            }
        }
        if (is_prime) return candidate;
    }
    return null; // ไม่พบจำนวนเฉพาะ
}

pub fn main() void {
    var current: u32 = 1;
    var count: usize = 0;

    std.debug.print("จำนวนเฉพาะ 10 ตัวแรก: ", .{});

    // while กับ optional + else
    while (getNextPrime(current)) |prime| {
        std.debug.print("{d} ", .{prime});
        current = prime;
        count += 1;
        if (count >= 10) break;
    } else {
        std.debug.print("(ไม่พบจำนวนเฉพาะเพิ่มเติม)", .{});
    }

    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
จำนวนเฉพาะ 10 ตัวแรก: 2 3 5 7 11 13 17 19 23 29
```

### while กับ optional และ continue expression

สามารถรวมทั้งสองฟีเจอร์ได้:

```zig
const std = @import("std");

const TokenStream = struct {
    data: []const u8,
    pos: usize,

    pub fn nextToken(self: *TokenStream) ?[]const u8 {
        // ข้าม whitespace
        while (self.pos < self.data.len and self.data[self.pos] == ' ') {
            self.pos += 1;
        }
        if (self.pos >= self.data.len) return null;

        const start = self.pos;
        while (self.pos < self.data.len and self.data[self.pos] != ' ') {
            self.pos += 1;
        }
        return self.data[start..self.pos];
    }
};

pub fn main() void {
    var stream = TokenStream{
        .data = "hello world zig language",
        .pos = 0,
    };

    var token_num: usize = 0;
    // while กับ optional + continue expression
    while (stream.nextToken()) |token| : (token_num += 1) {
        std.debug.print("Token {d}: '{s}'\n", .{token_num, token});
    }
}
```

---

## Step 64: for loop - วนซ้ำใน array/slice

`for` loop ใน Zig ถูกออกแบบมาเพื่อ **iterate ผ่าน collections** โดยเฉพาะ

### รูปแบบพื้นฐาน

```zig
for (collection) |item| {
    // ใช้ item
}
```

**ตัวอย่าง 64.1: for กับ array**

```zig
const std = @import("std");

pub fn main() void {
    const fruits = [_][]const u8{
        "แอปเปิ้ล",
        "กล้วย",
        "แตงโม",
        "ส้ม",
        "มะม่วง",
    };

    std.debug.print("รายการผลไม้:\n", .{});
    for (fruits) |fruit| {
        std.debug.print("  - {s}\n", .{fruit});
    }
}
```

**ผลลัพธ์:**
```
รายการผลไม้:
  - แอปเปิ้ล
  - กล้วย
  - แตงโม
  - ส้ม
  - มะม่วง
```

**ตัวอย่าง 64.2: for กับตัวเลข**

```zig
const std = @import("std");

pub fn main() void {
    const numbers = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6, 5, 3 };
    var sum: i32 = 0;
    var max: i32 = numbers[0];
    var min: i32 = numbers[0];

    for (numbers) |n| {
        sum += n;
        if (n > max) max = n;
        if (n < min) min = n;
    }

    const avg = @as(f64, @floatFromInt(sum)) / @as(f64, @floatFromInt(numbers.len));

    std.debug.print("ข้อมูล: ", .{});
    for (numbers) |n| std.debug.print("{d} ", .{n});
    std.debug.print("\n", .{});
    std.debug.print("ผลรวม: {d}\n", .{sum});
    std.debug.print("เฉลี่ย: {d:.2}\n", .{avg});
    std.debug.print("สูงสุด: {d}\n", .{max});
    std.debug.print("ต่ำสุด: {d}\n", .{min});
}
```

**ผลลัพธ์:**
```
ข้อมูล: 3 1 4 1 5 9 2 6 5 3
ผลรวม: 39
เฉลี่ย: 3.90
สูงสุด: 9
ต่ำสุด: 1
```

**ตัวอย่าง 64.3: for กับ range (0..N)**

Zig รองรับการใช้ range syntax ใน for loop:

```zig
const std = @import("std");

pub fn main() void {
    // วนจาก 0 ถึง 9 (ไม่รวม 10)
    std.debug.print("0 ถึง 9: ", .{});
    for (0..10) |i| {
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});

    // วนจาก 1 ถึง 5
    std.debug.print("1 ถึง 5: ", .{});
    for (1..6) |i| {
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});

    // คำนวณ factorial
    var factorial: u64 = 1;
    for (1..11) |i| {
        factorial *= i;
    }
    std.debug.print("10! = {d}\n", .{factorial});
}
```

**ผลลัพธ์:**
```
0 ถึง 9: 0 1 2 3 4 5 6 7 8 9
1 ถึง 5: 1 2 3 4 5
10! = 3628800
```

**ตัวอย่าง 64.4: for กับ slice**

```zig
const std = @import("std");

pub fn sumSlice(data: []const i32) i32 {
    var total: i32 = 0;
    for (data) |val| {
        total += val;
    }
    return total;
}

pub fn main() void {
    const data = [_]i32{ 10, 20, 30, 40, 50, 60, 70, 80, 90, 100 };

    // ผลรวมทั้งหมด
    std.debug.print("ผลรวมทั้งหมด: {d}\n", .{sumSlice(&data)});

    // ผลรวม 5 ตัวแรก (slice)
    std.debug.print("ผลรวม 5 ตัวแรก: {d}\n", .{sumSlice(data[0..5])});

    // ผลรวม 5 ตัวหลัง (slice)
    std.debug.print("ผลรวม 5 ตัวหลัง: {d}\n", .{sumSlice(data[5..])});
}
```

**ผลลัพธ์:**
```
ผลรวมทั้งหมด: 550
ผลรวม 5 ตัวแรก: 150
ผลรวม 5 ตัวหลัง: 400
```

### for กับ string (bytes)

```zig
const std = @import("std");

pub fn main() void {
    const text = "Hello, Zig!";
    var uppercase_count: usize = 0;
    var lowercase_count: usize = 0;

    for (text) |char| {
        if (char >= 'A' and char <= 'Z') {
            uppercase_count += 1;
        } else if (char >= 'a' and char <= 'z') {
            lowercase_count += 1;
        }
    }

    std.debug.print("ข้อความ: {s}\n", .{text});
    std.debug.print("ตัวพิมพ์ใหญ่: {d} ตัว\n", .{uppercase_count});
    std.debug.print("ตัวพิมพ์เล็ก: {d} ตัว\n", .{lowercase_count});
}
```

---

## Step 65: for กับ index

บ่อยครั้งเราต้องการทั้ง **ค่า** และ **ตำแหน่ง (index)** ของแต่ละ element

### รูปแบบ

```zig
for (collection, 0..) |item, index| {
    // ใช้ทั้ง item และ index
}
```

**ตัวอย่าง 65.1: แสดง index และค่า**

```zig
const std = @import("std");

pub fn main() void {
    const languages = [_][]const u8{
        "Zig",
        "Rust",
        "Go",
        "C",
        "Python",
    };

    std.debug.print("ภาษาโปรแกรมมิ่ง:\n", .{});
    for (languages, 0..) |lang, i| {
        std.debug.print("  {d}. {s}\n", .{i + 1, lang});
    }
}
```

**ผลลัพธ์:**
```
ภาษาโปรแกรมมิ่ง:
  1. Zig
  2. Rust
  3. Go
  4. C
  5. Python
```

**ตัวอย่าง 65.2: หาตำแหน่งของค่าใน array**

```zig
const std = @import("std");

pub fn indexOf(arr: []const i32, target: i32) ?usize {
    for (arr, 0..) |val, i| {
        if (val == target) return i;
    }
    return null;
}

pub fn main() void {
    const data = [_]i32{ 15, 3, 8, 42, 7, 23, 11, 42, 5 };

    if (indexOf(&data, 42)) |pos| {
        std.debug.print("พบ 42 ที่ตำแหน่ง {d}\n", .{pos});
    } else {
        std.debug.print("ไม่พบ 42\n", .{});
    }

    if (indexOf(&data, 99)) |pos| {
        std.debug.print("พบ 99 ที่ตำแหน่ง {d}\n", .{pos});
    } else {
        std.debug.print("ไม่พบ 99 ในข้อมูล\n", .{});
    }
}
```

**ผลลัพธ์:**
```
พบ 42 ที่ตำแหน่ง 3
ไม่พบ 99 ในข้อมูล
```

**ตัวอย่าง 65.3: แก้ไขค่าใน array ด้วย index**

เมื่อต้องการแก้ไขค่า ต้องใช้ pointer:

```zig
const std = @import("std");

pub fn main() void {
    var scores = [_]i32{ 45, 78, 23, 91, 56, 67, 34, 88 };

    std.debug.print("คะแนนเดิม: ", .{});
    for (scores) |s| std.debug.print("{d} ", .{s});
    std.debug.print("\n", .{});

    // เพิ่มโบนัส 10 คะแนนให้ทุกคน แต่ไม่เกิน 100
    for (&scores) |*score| {
        score.* = @min(score.* + 10, 100);
    }

    std.debug.print("หลังเพิ่มโบนัส: ", .{});
    for (scores) |s| std.debug.print("{d} ", .{s});
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
คะแนนเดิม: 45 78 23 91 56 67 34 88
หลังเพิ่มโบนัส: 55 88 33 100 66 77 44 98
```

**ตัวอย่าง 65.4: สร้าง multiplication table**

```zig
const std = @import("std");

pub fn main() void {
    const size: usize = 5;

    std.debug.print("ตารางสูตรคูณ {d}x{d}:\n", .{size, size});
    std.debug.print("     ", .{});
    for (1..size+1) |col| {
        std.debug.print("{d:4}", .{col});
    }
    std.debug.print("\n", .{});

    for (1..size+1) |row| {
        std.debug.print("{d:4} ", .{row});
        for (1..size+1) |col| {
            std.debug.print("{d:4}", .{row * col});
        }
        std.debug.print("\n", .{});
    }
}
```

**ผลลัพธ์:**
```
ตารางสูตรคูณ 5x5:
        1   2   3   4   5
   1    1   2   3   4   5
   2    2   4   6   8  10
   3    3   6   9  12  15
   4    4   8  12  16  20
   5    5  10  15  20  25
```

**ตัวอย่าง 65.5: for กับ index แบบ custom start**

```zig
const std = @import("std");

pub fn main() void {
    const items = [_][]const u8{ "แรก", "สอง", "สาม", "สี่", "ห้า" };

    // เริ่ม index จาก 1 แทน 0
    for (items, 1..) |item, num| {
        std.debug.print("รายการที่ {d}: {s}\n", .{num, item});
    }
}
```

---

## Step 66: for กับหลาย arrays พร้อมกัน

Zig อนุญาตให้ **iterate ผ่านหลาย collections พร้อมกัน** ใน for loop เดียว - คล้าย `zip()` ใน Python

### รูปแบบ

```zig
for (array1, array2) |val1, val2| {
    // ใช้ val1 และ val2 พร้อมกัน
}
```

**ข้อสำคัญ**: ทั้ง array ต้องมีความยาวเท่ากัน ไม่เช่นนั้นจะเกิด compile error (สำหรับ array ที่รู้ขนาด compile-time) หรือ runtime panic

**ตัวอย่าง 66.1: zip สองรายการ**

```zig
const std = @import("std");

pub fn main() void {
    const names = [_][]const u8{
        "สมชาย",
        "สมหญิง",
        "อนุชา",
        "วิภา",
    };

    const scores = [_]u32{ 85, 92, 78, 95 };

    std.debug.print("ผลสอบ:\n", .{});
    for (names, scores) |name, score| {
        const grade = if (score >= 90) "A"
                     else if (score >= 80) "B"
                     else if (score >= 70) "C"
                     else "D";
        std.debug.print("  {s}: {d} คะแนน (เกรด {s})\n", .{name, score, grade});
    }
}
```

**ผลลัพธ์:**
```
ผลสอบ:
  สมชาย: 85 คะแนน (เกรด B)
  สมหญิง: 92 คะแนน (เกรด A)
  อนุชา: 78 คะแนน (เกรด C)
  วิภา: 95 คะแนน (เกรด A)
```

**ตัวอย่าง 66.2: คำนวณ dot product**

```zig
const std = @import("std");

pub fn dotProduct(a: []const f64, b: []const f64) f64 {
    var result: f64 = 0.0;
    for (a, b) |ai, bi| {
        result += ai * bi;
    }
    return result;
}

pub fn main() void {
    const vec1 = [_]f64{ 1.0, 2.0, 3.0, 4.0 };
    const vec2 = [_]f64{ 5.0, 6.0, 7.0, 8.0 };

    const dp = dotProduct(&vec1, &vec2);
    // 1*5 + 2*6 + 3*7 + 4*8 = 5 + 12 + 21 + 32 = 70
    std.debug.print("Dot product: {d:.1}\n", .{dp});
}
```

**ผลลัพธ์:**
```
Dot product: 70.0
```

**ตัวอย่าง 66.3: สาม arrays พร้อมกัน**

```zig
const std = @import("std");

pub fn main() void {
    const x_coords = [_]f32{ 0.0, 1.0, 2.0, 3.0 };
    const y_coords = [_]f32{ 0.0, 1.0, 4.0, 9.0 };
    const labels = [_][]const u8{ "A", "B", "C", "D" };

    std.debug.print("จุดบนกราฟ:\n", .{});
    for (x_coords, y_coords, labels) |x, y, label| {
        std.debug.print("  {s}: ({d:.1}, {d:.1})\n", .{label, x, y});
    }
}
```

**ผลลัพธ์:**
```
จุดบนกราฟ:
  A: (0.0, 0.0)
  B: (1.0, 1.0)
  C: (2.0, 4.0)
  D: (3.0, 9.0)
```

**ตัวอย่าง 66.4: หลาย arrays กับ index**

```zig
const std = @import("std");

pub fn main() void {
    const keys = [_][]const u8{ "name", "age", "city" };
    const values = [_][]const u8{ "สมชาย", "25", "กรุงเทพฯ" };

    std.debug.print("ข้อมูลบุคคล:\n", .{});
    for (keys, values, 0..) |k, v, i| {
        std.debug.print("  [{d}] {s} = {s}\n", .{i, k, v});
    }
}
```

**ผลลัพธ์:**
```
ข้อมูลบุคคล:
  [0] name = สมชาย
  [1] age = 25
  [2] city = กรุงเทพฯ
```

**ตัวอย่าง 66.5: เปรียบเทียบกับ Python**

```python
# Python
names = ["Alice", "Bob", "Charlie"]
scores = [85, 92, 78]

for name, score in zip(names, scores):
    print(f"{name}: {score}")
```

```zig
// Zig - ตรงไปตรงมากว่า ไม่ต้องเรียก zip()
const names = [_][]const u8{ "Alice", "Bob", "Charlie" };
const scores = [_]u32{ 85, 92, 78 };

for (names, scores) |name, score| {
    std.debug.print("{s}: {d}\n", .{name, score});
}
```

---

## Step 67: inline for และ inline while

`inline` loops เป็นหนึ่งในฟีเจอร์ที่ทรงพลังของ Zig: loop ที่ทำงานที่ **compile-time**

### ทำความเข้าใจ inline

**runtime loop** - code ทำงาน loop จริงๆ ตอน run program
**inline loop (compile-time)** - compiler "펼쳐" loop ออกเป็น code หลายๆ บรรทัด

### เมื่อไหรควรใช้ inline?

1. เมื่อ loop ทำงานกับ **compile-time known values**
2. เมื่อต้องการ **type-level operations** (แต่ละ iteration ใช้ type ต่างกัน)
3. เมื่อต้องการ **performance optimization** สำหรับ loop สั้นๆ ที่รู้จำนวนรอบ

**ตัวอย่าง 67.1: inline for พื้นฐาน**

```zig
const std = @import("std");

pub fn main() void {
    // inline for - compiler จะ unroll เป็น 5 print statements
    inline for (0..5) |i| {
        std.debug.print("compile-time i = {d}\n", .{i});
    }
}
```

โค้ดนี้เหมือนกับที่ compiler สร้าง:
```zig
std.debug.print("compile-time i = {d}\n", .{0});
std.debug.print("compile-time i = {d}\n", .{1});
std.debug.print("compile-time i = {d}\n", .{2});
std.debug.print("compile-time i = {d}\n", .{3});
std.debug.print("compile-time i = {d}\n", .{4});
```

**ตัวอย่าง 67.2: inline for กับ comptime types**

นี่คือกรณีที่ **ต้องใช้ inline** เพราะแต่ละ iteration ใช้ type ต่างกัน:

```zig
const std = @import("std");

pub fn main() void {
    // types เป็น compile-time value
    const types = .{ i32, f64, bool, u8 };

    inline for (types) |T| {
        // @sizeOf(T) ต้องรู้ T ตอน compile-time
        std.debug.print("ขนาดของ {s}: {d} bytes\n", .{
            @typeName(T),
            @sizeOf(T),
        });
    }
}
```

**ผลลัพธ์:**
```
ขนาดของ i32: 4 bytes
ขนาดของ f64: 8 bytes
ขนาดของ bool: 1 bytes
ขนาดของ u8: 1 bytes
```

**ตัวอย่าง 67.3: inline while**

```zig
const std = @import("std");

pub fn main() void {
    comptime var i: u32 = 1;

    // inline while - ทุกอย่างเกิดตอน compile-time
    inline while (i <= 8) : (i *= 2) {
        std.debug.print("power of 2: {d}\n", .{i});
    }
}
```

**ผลลัพธ์:**
```
power of 2: 1
power of 2: 2
power of 2: 4
power of 2: 8
```

**ตัวอย่าง 67.4: inline for กับ struct fields**

```zig
const std = @import("std");

const Point3D = struct {
    x: f32,
    y: f32,
    z: f32,
};

pub fn printStruct(value: anytype) void {
    const T = @TypeOf(value);
    const fields = @typeInfo(T).@"struct".fields;

    inline for (fields) |field| {
        std.debug.print("  {s} = {any}\n", .{
            field.name,
            @field(value, field.name),
        });
    }
}

pub fn main() void {
    const p = Point3D{ .x = 1.5, .y = 2.7, .z = -0.3 };
    std.debug.print("Point3D:\n", .{});
    printStruct(p);
}
```

**ผลลัพธ์:**
```
Point3D:
  x = 1.5e0
  y = 2.7e0
  z = -3.0e-1
```

### Performance: inline vs regular loop

```zig
// Regular loop - มี loop overhead
fn sumRegular(arr: []const i32) i32 {
    var sum: i32 = 0;
    for (arr) |x| sum += x;
    return sum;
}

// Inline loop - compiler เปลี่ยนเป็น operations ตรงๆ
// เหมาะสำหรับ array ขนาดเล็กที่รู้ขนาดตอน compile
fn sumInline(arr: [4]i32) i32 {
    var sum: i32 = 0;
    inline for (arr) |x| sum += x;
    return sum;
    // compiler อาจ optimize เป็น: return arr[0] + arr[1] + arr[2] + arr[3];
}
```

**ข้อควรระวัง**: อย่าใช้ `inline` กับ loop ที่มีจำนวนรอบมาก จะทำให้ binary ใหญ่ขึ้นมาก

---

## Step 68: break และ continue

### break - ออกจาก loop

**ตัวอย่าง 68.1: break พื้นฐาน**

```zig
const std = @import("std");

pub fn main() void {
    var i: u32 = 0;
    while (true) {
        if (i >= 5) break;
        std.debug.print("{d} ", .{i});
        i += 1;
    }
    std.debug.print("\n", .{});
    std.debug.print("ออกจาก loop เมื่อ i = {d}\n", .{i});
}
```

**ตัวอย่าง 68.2: break ใน for loop**

```zig
const std = @import("std");

pub fn findFirst(arr: []const i32, target: i32) ?usize {
    for (arr, 0..) |val, i| {
        if (val == target) {
            return i; // return ออกจาก function ทันที
        }
    }
    return null;
}

pub fn main() void {
    const data = [_]i32{ 5, 2, 8, 1, 9, 3, 7, 4, 6 };

    // หาค่าแรกที่มากกว่า 6
    for (data, 0..) |val, i| {
        if (val > 6) {
            std.debug.print("พบค่าแรกที่ > 6: data[{d}] = {d}\n", .{i, val});
            break;
        }
    }
}
```

**ผลลัพธ์:**
```
พบค่าแรกที่ > 6: data[2] = 8
```

### continue - ข้าม iteration

**ตัวอย่าง 68.3: continue พื้นฐาน**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("เลขคี่จาก 1 ถึง 10: ", .{});

    for (1..11) |i| {
        if (i % 2 == 0) continue; // ข้ามเลขคู่
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});
}
```

**ผลลัพธ์:**
```
เลขคี่จาก 1 ถึง 10: 1 3 5 7 9
```

**ตัวอย่าง 68.4: ความแตกต่างระหว่าง continue ใน for และ while**

```zig
const std = @import("std");

pub fn main() void {
    // for loop - continue ปลอดภัยเสมอ
    std.debug.print("for + continue: ", .{});
    for (0..10) |i| {
        if (i % 3 == 0) continue;
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});

    // while + continue expression - ปลอดภัย
    std.debug.print("while + continue expr: ", .{});
    var i: u32 = 0;
    while (i < 10) : (i += 1) {
        if (i % 3 == 0) continue;
        std.debug.print("{d} ", .{i});
    }
    std.debug.print("\n", .{});

    // while ไม่มี continue expression - อันตราย!
    // ถ้าใช้ continue จะ infinite loop เพราะ i ไม่ถูก increment
}
```

**ตัวอย่าง 68.5: break กับ else clause**

Zig มี `else` สำหรับ loop - จะรันถ้า loop จบโดยไม่ผ่าน `break`:

```zig
const std = @import("std");

pub fn isPrime(n: u32) bool {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;

    var i: u32 = 3;
    while (i * i <= n) : (i += 2) {
        if (n % i == 0) break; // พบตัวหาร - ไม่ใช่จำนวนเฉพาะ
    } else {
        return true; // ไม่เจอ break = ไม่มีตัวหาร = จำนวนเฉพาะ
    }

    return false;
}

pub fn main() void {
    std.debug.print("ทดสอบจำนวนเฉพาะ:\n", .{});
    const test_numbers = [_]u32{ 2, 3, 4, 7, 12, 17, 25, 29, 30, 31 };

    for (test_numbers) |n| {
        std.debug.print("  {d}: {s}\n", .{
            n,
            if (isPrime(n)) "จำนวนเฉพาะ" else "ไม่ใช่จำนวนเฉพาะ",
        });
    }
}
```

**ผลลัพธ์:**
```
ทดสอบจำนวนเฉพาะ:
  2: จำนวนเฉพาะ
  3: จำนวนเฉพาะ
  4: ไม่ใช่จำนวนเฉพาะ
  7: จำนวนเฉพาะ
  12: ไม่ใช่จำนวนเฉพาะ
  17: จำนวนเฉพาะ
  25: ไม่ใช่จำนวนเฉพาะ
  29: จำนวนเฉพาะ
  30: ไม่ใช่จำนวนเฉพาะ
  31: จำนวนเฉพาะ
```

**ตัวอย่าง 68.6: break กับ else ใน for loop**

```zig
const std = @import("std");

pub fn main() void {
    const targets = [_]i32{ 5, 15, 8 };
    const haystack = [_]i32{ 1, 2, 3, 4, 6, 7, 9, 10 }; // ไม่มี 5, 15, 8

    for (targets) |target| {
        for (haystack) |val| {
            if (val == target) {
                std.debug.print("พบ {d} ในข้อมูล\n", .{target});
                break;
            }
        } else {
            std.debug.print("ไม่พบ {d} ในข้อมูล\n", .{target});
        }
    }
}
```

**ผลลัพธ์:**
```
ไม่พบ 5 ในข้อมูล
ไม่พบ 15 ในข้อมูล
พบ 8 ในข้อมูล
```

---

## Step 69: Labeled loops และ break with value

### Labeled loops

เมื่อมี nested loops และต้องการ `break` ออกจาก outer loop:

```zig
outer: for (0..5) |i| {
    for (0..5) |j| {
        if (i + j >= 6) break :outer; // break ออกจาก outer loop
        std.debug.print("({d},{d}) ", .{i, j});
    }
}
```

**ตัวอย่าง 69.1: nested loops ปกติ vs labeled**

```zig
const std = @import("std");

pub fn main() void {
    // ไม่ใช้ label - break ออกแค่ inner loop
    std.debug.print("ไม่ใช้ label:\n", .{});
    for (0..3) |i| {
        for (0..3) |j| {
            if (j == 1) break; // ออกแค่ inner loop
            std.debug.print("  ({d},{d})\n", .{i, j});
        }
    }

    // ใช้ label - break ออกจาก outer loop
    std.debug.print("ใช้ label:\n", .{});
    outer: for (0..3) |i| {
        for (0..3) |j| {
            if (j == 1) break :outer; // ออกจาก outer loop ทันที
            std.debug.print("  ({d},{d})\n", .{i, j});
        }
    }
}
```

**ผลลัพธ์:**
```
ไม่ใช้ label:
  (0,0)
  (1,0)
  (2,0)
ใช้ label:
  (0,0)
```

**ตัวอย่าง 69.2: ค้นหาใน matrix ด้วย labeled break**

```zig
const std = @import("std");

pub fn main() void {
    const matrix = [4][4]i32{
        .{ 1,  2,  3,  4 },
        .{ 5,  6,  7,  8 },
        .{ 9, 10, 11, 12 },
        .{13, 14, 15, 16 },
    };

    const target: i32 = 11;
    var found_row: usize = 0;
    var found_col: usize = 0;
    var found = false;

    search: for (matrix, 0..) |row, i| {
        for (row, 0..) |val, j| {
            if (val == target) {
                found_row = i;
                found_col = j;
                found = true;
                break :search; // ออกจาก nested loop ทันที
            }
        }
    }

    if (found) {
        std.debug.print("พบ {d} ที่ matrix[{d}][{d}]\n", .{
            target, found_row, found_col
        });
    }
}
```

**ผลลัพธ์:**
```
พบ 11 ที่ matrix[2][2]
```

### break with value

Zig อนุญาตให้ `break` ส่งค่า return จาก loop (คล้ายกับ `break` ใน Rust):

```zig
const result = loop_label: while (condition) {
    if (found) break :loop_label value_to_return;
};
```

**ตัวอย่าง 69.3: break with value**

```zig
const std = @import("std");

pub fn main() void {
    const data = [_]i32{ 4, 7, 2, 9, 1, 5, 8, 3, 6 };

    // หาค่าแรกที่มากกว่า 6 และ return ค่านั้น
    const first_large = find: for (data) |val| {
        if (val > 6) break :find val;
    } else -1; // ถ้าไม่เจอ return -1

    std.debug.print("ค่าแรกที่ > 6: {d}\n", .{first_large});
}
```

**ผลลัพธ์:**
```
ค่าแรกที่ > 6: 7
```

**ตัวอย่าง 69.4: while กับ break with value**

```zig
const std = @import("std");

pub fn main() void {
    // Binary search ที่ return ผ่าน break with value
    const sorted = [_]i32{ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 };
    const target: i32 = 11;

    var lo: usize = 0;
    var hi: usize = sorted.len;

    const result = binary_search: while (lo < hi) {
        const mid = lo + (hi - lo) / 2;
        if (sorted[mid] == target) {
            break :binary_search mid; // return index
        } else if (sorted[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    } else sorted.len; // ไม่พบ - return len เป็น sentinel

    if (result < sorted.len) {
        std.debug.print("พบ {d} ที่ index {d}\n", .{target, result});
    } else {
        std.debug.print("ไม่พบ {d}\n", .{target});
    }
}
```

**ผลลัพธ์:**
```
พบ 11 ที่ index 5
```

**ตัวอย่าง 69.5: labeled continue**

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("คู่ (i,j) ที่ i+j เป็นจำนวนเฉพาะ:\n", .{});

    outer: for (1..6) |i| {
        for (1..6) |j| {
            const sum = i + j;
            // ตรวจสอบว่าเป็นจำนวนเฉพาะ
            var is_prime = sum >= 2;
            var d: usize = 2;
            while (d * d <= sum) : (d += 1) {
                if (sum % d == 0) {
                    is_prime = false;
                    break;
                }
            }

            if (!is_prime) continue :outer; // ข้ามไป i ถัดไป

            std.debug.print("  ({d},{d}) -> {d}+{d}={d}\n", .{i, j, i, j, sum});
            break; // ออกจาก inner loop
        }
    }
}
```

---

## Step 70: Loop เป็น expression

ใน Zig, `while` และ `for` เป็น **expressions** - สามารถ return ค่าได้ผ่าน `break`

### Loop Expression พื้นฐาน

```zig
const value = while (condition) {
    // ...
    if (found) break :label result;
} else default_value;
```

**ตัวอย่าง 70.1: for loop เป็น expression**

```zig
const std = @import("std");

pub fn main() void {
    const numbers = [_]i32{ 3, 7, 2, 9, 4, 6, 1, 8, 5 };

    // หาค่าสูงสุดด้วย for expression
    const max = blk: {
        var m = numbers[0];
        for (numbers[1..]) |n| {
            if (n > m) m = n;
        }
        break :blk m;
    };

    std.debug.print("ค่าสูงสุด: {d}\n", .{max});
}
```

**ตัวอย่าง 70.2: while loop เป็น expression (Collatz)**

```zig
const std = @import("std");

pub fn collatzLength(n: u64) u64 {
    var current = n;
    const steps = while (current != 1) {
        var count: u64 = 0;
        while (current != 1) : (count += 1) {
            if (current % 2 == 0) {
                current /= 2;
            } else {
                current = 3 * current + 1;
            }
        }
        break count;
    } else 0;

    return steps;
}

pub fn main() void {
    std.debug.print("Collatz sequence lengths:\n", .{});
    for (1..11) |n| {
        var current: u64 = @intCast(n);
        var steps: u64 = 0;

        while (current != 1) : (steps += 1) {
            if (current % 2 == 0) {
                current /= 2;
            } else {
                current = 3 * current + 1;
            }
        }

        std.debug.print("  n={d}: {d} steps\n", .{n, steps});
    }
}
```

**ผลลัพธ์:**
```
Collatz sequence lengths:
  n=1: 0 steps
  n=2: 1 steps
  n=3: 7 steps
  n=4: 2 steps
  n=5: 5 steps
  n=6: 8 steps
  n=7: 16 steps
  n=8: 3 steps
  n=9: 19 steps
  n=10: 6 steps
```

**ตัวอย่าง 70.3: nested loop expression**

```zig
const std = @import("std");

pub fn main() void {
    const matrix = [3][3]i32{
        .{ 1, 2, 3 },
        .{ 4, 5, 6 },
        .{ 7, 8, 9 },
    };

    // หาผลรวมของแนวทแยงหลัก ด้วย loop expression
    const diagonal_sum = blk: {
        var sum: i32 = 0;
        for (0..3) |i| {
            sum += matrix[i][i];
        }
        break :blk sum;
    };

    std.debug.print("ผลรวมแนวทแยงหลัก: {d}\n", .{diagonal_sum}); // 1+5+9=15
}
```

**ผลลัพธ์:**
```
ผลรวมแนวทแยงหลัก: 15
```

**ตัวอย่าง 70.4: loop expression ใน function return**

```zig
const std = @import("std");

pub fn findMax(arr: []const i32) ?i32 {
    if (arr.len == 0) return null;

    return blk: {
        var max = arr[0];
        for (arr[1..]) |val| {
            if (val > max) max = val;
        }
        break :blk max;
    };
}

pub fn findMin(arr: []const i32) ?i32 {
    if (arr.len == 0) return null;

    return for (arr) |val| {
        _ = val; // ไม่ได้ใช้วิธีนี้ - แสดงเป็น syntax เฉยๆ
        break val; // break ตัวแรกเสมอ - ไม่ใช่ min จริงๆ
    } else null;
}

pub fn main() void {
    const data = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };

    if (findMax(&data)) |max| {
        std.debug.print("ค่าสูงสุด: {d}\n", .{max});
    }
}
```

**ตัวอย่าง 70.5: รวมทุกอย่าง - loop expression ที่ซับซ้อน**

```zig
const std = @import("std");

// หาจำนวนเฉพาะ n ตัวแรก
pub fn sumFirstNPrimes(n: usize) u64 {
    var sum: u64 = 0;
    var count: usize = 0;
    var candidate: u64 = 2;

    while (count < n) : (candidate += 1) {
        const is_prime = check: {
            if (candidate < 2) break :check false;
            if (candidate == 2) break :check true;
            if (candidate % 2 == 0) break :check false;

            var d: u64 = 3;
            while (d * d <= candidate) : (d += 2) {
                if (candidate % d == 0) break :check false;
            }
            break :check true;
        };

        if (is_prime) {
            sum += candidate;
            count += 1;
        }
    }

    return sum;
}

pub fn main() void {
    const n = 10;
    const result = sumFirstNPrimes(n);
    std.debug.print("ผลรวมของจำนวนเฉพาะ {d} ตัวแรก: {d}\n", .{n, result});
    // 2+3+5+7+11+13+17+19+23+29 = 129
}
```

**ผลลัพธ์:**
```
ผลรวมของจำนวนเฉพาะ 10 ตัวแรก: 129
```

---

## สรุปเปรียบเทียบ Zig vs C vs Python

| ฟีเจอร์ | Zig | C | Python |
|---------|-----|---|--------|
| while loop | ✅ | ✅ | ✅ |
| for loop | ✅ (collections/range) | ✅ (3-part) | ✅ |
| continue expression | ✅ (unique!) | ❌ | ❌ |
| Optional while | ✅ (unique!) | ❌ | ❌ |
| Multi-array for | ✅ (built-in) | ❌ | zip() |
| inline loops | ✅ (compile-time) | ❌ | ❌ |
| Loop else clause | ✅ | ❌ | ✅ |
| Labeled loops | ✅ | ❌* | ❌ |
| break with value | ✅ | ❌ | ❌ |
| Loop as expression | ✅ | ❌ | ❌ |

*C มี goto แต่ไม่ใช่ labeled loop

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน

**แบบฝึกหัด 1**: เขียน function `sumEven(n: u32) u32` ที่หาผลรวมของเลขคู่ตั้งแต่ 2 ถึง n

```zig
// เติมโค้ด:
pub fn sumEven(n: u32) u32 {
    // TODO
}
```

**แนวทาง**: ใช้ while loop กับ continue expression

**เฉลย**:
```zig
pub fn sumEven(n: u32) u32 {
    var sum: u32 = 0;
    var i: u32 = 2;
    while (i <= n) : (i += 2) {
        sum += i;
    }
    return sum;
}
```

---

**แบบฝึกหัด 2**: เขียน function ที่รับ slice ของ string และ return slice ที่มีเฉพาะ string ที่ยาวกว่า 5 ตัวอักษร

```zig
pub fn filterLong(words: []const []const u8, out: [][]const u8) usize {
    // TODO: return จำนวน string ที่ผ่านการ filter
}
```

**เฉลย**:
```zig
pub fn filterLong(words: []const []const u8, out: [][]const u8) usize {
    var count: usize = 0;
    for (words) |word| {
        if (word.len > 5 and count < out.len) {
            out[count] = word;
            count += 1;
        }
    }
    return count;
}
```

---

### ระดับ 2: กลาง

**แบบฝึกหัด 3**: ใช้ labeled loop เพื่อหาคู่ของเลข (i, j) ในช่วง 1-20 ที่ i*j เท่ากับ target

```zig
pub fn findFactorPair(target: u32) ?struct { a: u32, b: u32 } {
    // TODO: ใช้ labeled break with value
}
```

**เฉลย**:
```zig
pub fn findFactorPair(target: u32) ?struct { a: u32, b: u32 } {
    return search: for (1..21) |i| {
        for (1..21) |j| {
            if (i * j == target) {
                break :search .{
                    .a = @intCast(i),
                    .b = @intCast(j),
                };
            }
        }
    } else null;
}
```

---

**แบบฝึกหัด 4**: เขียน function ที่ใช้ while กับ optional เพื่อนับจำนวน word ใน string (แยกด้วย space)

```zig
pub fn wordCount(text: []const u8) usize {
    // TODO: ใช้ while กับ optional-returning function
}
```

**เฉลย**:
```zig
fn nextWord(text: []const u8, start: *usize) ?[]const u8 {
    // ข้าม spaces
    while (start.* < text.len and text[start.*] == ' ') : (start.* += 1) {}
    if (start.* >= text.len) return null;

    const word_start = start.*;
    while (start.* < text.len and text[start.*] != ' ') : (start.* += 1) {}
    return text[word_start..start.*];
}

pub fn wordCount(text: []const u8) usize {
    var pos: usize = 0;
    var count: usize = 0;
    while (nextWord(text, &pos)) |_| {
        count += 1;
    }
    return count;
}
```

---

### ระดับ 3: ขั้นสูง

**แบบฝึกหัด 5**: เขียน function ที่ใช้ inline for เพื่อ serialize struct เป็น string format "key1=val1;key2=val2"

```zig
pub fn serializeStruct(value: anytype, buf: []u8) []u8 {
    // TODO: ใช้ inline for กับ @typeInfo
}
```

**แนวทาง**: ใช้ `@typeInfo(T).@"struct".fields` และ `inline for` เพื่อ iterate fields

---

**แบบฝึกหัด 6**: เขียน Sieve of Eratosthenes ด้วย Zig loops

```zig
pub fn sieve(limit: usize, result: []bool) void {
    // result[i] = true หมายความว่า i เป็นจำนวนเฉพาะ
    // TODO
}
```

**เฉลย**:
```zig
pub fn sieve(limit: usize, result: []bool) void {
    // เริ่มต้นทุกตัวเป็น true
    for (result) |*v| v.* = true;
    if (limit > 0) result[0] = false;
    if (limit > 1) result[1] = false;

    var i: usize = 2;
    while (i * i <= limit) : (i += 1) {
        if (!result[i]) continue;
        var j = i * i;
        while (j <= limit) : (j += i) {
            result[j] = false;
        }
    }
}
```

---

## บทสรุป

ใน Part 7 นี้เราได้เรียนรู้:

### สิ่งที่เรียนไป

| Step | หัวข้อ | จุดสำคัญ |
|------|--------|----------|
| 61 | while loop พื้นฐาน | ต้องมี braces เสมอ |
| 62 | while + continue expression | แก้ปัญหา infinite loop เมื่อใช้ continue |
| 63 | while + optional | หยุดอัตโนมัติเมื่อ null |
| 64 | for loop | iterate collections และ range |
| 65 | for + index | ดึงทั้งค่าและตำแหน่งพร้อมกัน |
| 66 | for + multiple arrays | zip built-in โดยไม่ต้องเรียก function |
| 67 | inline for/while | compile-time loop unrolling |
| 68 | break และ continue | ออกจาก/ข้าม iteration |
| 69 | labeled loops | ควบคุม nested loops |
| 70 | loop เป็น expression | return ค่าจาก loop |

### จุดเด่นของ Zig

1. **Continue expression** แก้ปัญหา infinite loop จาก `continue` ใน C
2. **Optional while** สร้าง iterator pattern ที่สะอาด
3. **Multi-array for** ไม่ต้องใช้ zip() function
4. **Inline loops** compile-time optimization และ type-level operations
5. **Loop as expression** โค้ดกระชับและชัดเจน
6. **Labeled break with value** ควบคุม nested loops ได้ละเอียด

### ขั้นตอนต่อไป

- **Part 8**: Functions - การสร้างและใช้งาน function ใน Zig
- **Part 9**: Pointers และ Memory Management
- **Part 10**: Structs และ Enums

---

*เขียนโดย: หลักสูตร Zig ภาษาไทย*  
*อ้างอิง: https://ziglang.org/documentation/master/*
