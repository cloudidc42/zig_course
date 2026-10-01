# Part 81: Algorithms: Sorting ใน Zig
## Steps 801-810

---

## Step 801: Comparison Sorting Theory (O Notation)

### ทฤษฎีการเรียงลำดับแบบเปรียบเทียบ

การเรียงลำดับ (Sorting) เป็นหนึ่งในปัญหาพื้นฐานที่สำคัญที่สุดในวิทยาการคอมพิวเตอร์ อัลกอริทึมการเรียงลำดับแบบเปรียบเทียบ (Comparison-based Sorting) ใช้การเปรียบเทียบระหว่างคู่ของ element เพื่อกำหนดลำดับ

### Big O Notation

Big O Notation ใช้อธิบาย complexity ของอัลกอริทึม:

| สัญลักษณ์ | ความหมาย | ตัวอย่าง |
|-----------|---------|---------|
| O(1) | Constant time | การเข้าถึง array index |
| O(log n) | Logarithmic | Binary search |
| O(n) | Linear | Linear search |
| O(n log n) | Linearithmic | Merge sort, Heap sort |
| O(n²) | Quadratic | Bubble sort, Insertion sort |
| O(2ⁿ) | Exponential | Recursive Fibonacci |

### Lower Bound สำหรับ Comparison Sorting

ทฤษฎีพิสูจน์ว่าอัลกอริทึม comparison-based sorting ดีที่สุดที่ทำได้คือ O(n log n) ในกรณีเลวร้ายที่สุด

```zig
// part-081-sorting.zig - ตัวอย่างการวัด complexity
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // แสดงขนาดข้อมูลและจำนวนการดำเนินการโดยประมาณ
    const sizes = [_]u32{ 10, 100, 1000, 10000, 100000 };
    
    try stdout.print("n\t\tO(n)\t\tO(n log n)\tO(n²)\n", .{});
    try stdout.print("{s}\n", .{"=" ** 60});
    
    for (sizes) |n| {
        const linear = n;
        const nlogn = n * std.math.log2_int(u32, @max(n, 1));
        const quadratic = n * n;
        try stdout.print("{d}\t\t{d}\t\t{d}\t\t{d}\n", .{
            n, linear, nlogn, quadratic
        });
    }
}
```

### ประเภทของ Sorting Algorithms

1. **In-place sorts**: ไม่ต้องการหน่วยความจำเพิ่มเติม (Bubble, Insertion, Heap sort)
2. **Stable sorts**: รักษาลำดับของ element ที่เท่ากัน (Merge sort, Insertion sort)
3. **Adaptive sorts**: ทำงานได้ดีกับข้อมูลที่เรียงลำดับบางส่วนแล้ว (Insertion sort, Tim sort)

---

## Step 802: Bubble Sort Implementation

### Bubble Sort คืออะไร?

Bubble Sort เป็นอัลกอริทึมที่ง่ายที่สุด ทำงานโดยการ "ฟอง" (bubble) ค่าที่มากขึ้นไปยังด้านท้ายของ array ในแต่ละรอบ

**Complexity:**
- Best case: O(n) - เมื่อข้อมูลเรียงลำดับแล้ว
- Average case: O(n²)
- Worst case: O(n²)
- Space: O(1) - in-place

```zig
// bubble_sort.zig
const std = @import("std");

/// Bubble Sort แบบพื้นฐาน
pub fn bubbleSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    var i: usize = 0;
    while (i < n - 1) : (i += 1) {
        var j: usize = 0;
        while (j < n - 1 - i) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                // swap
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

/// Bubble Sort แบบปรับปรุง (Optimized) - หยุดเมื่อไม่มีการ swap
pub fn bubbleSortOptimized(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    var i: usize = 0;
    while (i < n - 1) : (i += 1) {
        var swapped = false;
        var j: usize = 0;
        while (j < n - 1 - i) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        // ถ้าไม่มีการ swap แสดงว่าเรียงลำดับแล้ว
        if (!swapped) break;
    }
}

/// Cocktail Shaker Sort - Bubble Sort ที่เดินสองทิศทาง
pub fn cocktailShakerSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    var left: usize = 0;
    var right: usize = n - 1;
    
    while (left < right) {
        var newRight: usize = left;
        // เดินจากซ้ายไปขวา
        var i = left;
        while (i < right) : (i += 1) {
            if (arr[i] > arr[i + 1]) {
                const temp = arr[i];
                arr[i] = arr[i + 1];
                arr[i + 1] = temp;
                newRight = i;
            }
        }
        right = newRight;
        
        var newLeft: usize = right;
        // เดินจากขวาไปซ้าย
        if (right == 0) break;
        var j = right;
        while (j > left) : (j -= 1) {
            if (arr[j - 1] > arr[j]) {
                const temp = arr[j - 1];
                arr[j - 1] = arr[j];
                arr[j] = temp;
                newLeft = j;
            }
        }
        left = newLeft;
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    var arr1 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    var arr2 = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    
    try stdout.print("ก่อนเรียงลำดับ: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    bubbleSort(i32, &arr1);
    
    try stdout.print("หลัง Bubble Sort: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    bubbleSortOptimized(i32, &arr2);
    
    try stdout.print("หลัง Optimized Bubble Sort: ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
}
```

---

## Step 803: Insertion Sort Implementation

### Insertion Sort คืออะไร?

Insertion Sort ทำงานคล้ายกับการเรียงไพ่ในมือ โดยนำ element แต่ละตัวไปแทรกในตำแหน่งที่ถูกต้องใน subarray ที่เรียงลำดับแล้ว

**Complexity:**
- Best case: O(n) - ข้อมูลเรียงลำดับแล้ว
- Average case: O(n²)
- Worst case: O(n²)
- Space: O(1) - in-place, Stable

```zig
// insertion_sort.zig
const std = @import("std");

/// Insertion Sort พื้นฐาน
pub fn insertionSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    var i: usize = 1;
    while (i < n) : (i += 1) {
        const key = arr[i];
        var j: usize = i;
        
        // เลื่อน elements ที่มากกว่า key ไปทางขวา
        while (j > 0 and arr[j - 1] > key) : (j -= 1) {
            arr[j] = arr[j - 1];
        }
        arr[j] = key;
    }
}

/// Binary Insertion Sort - ใช้ Binary Search หาตำแหน่งแทรก
pub fn binaryInsertionSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    var i: usize = 1;
    while (i < n) : (i += 1) {
        const key = arr[i];
        
        // Binary search หาตำแหน่งที่จะแทรก
        var left: usize = 0;
        var right: usize = i;
        
        while (left < right) {
            const mid = left + (right - left) / 2;
            if (arr[mid] <= key) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        
        // เลื่อน elements
        var j: usize = i;
        while (j > left) : (j -= 1) {
            arr[j] = arr[j - 1];
        }
        arr[left] = key;
    }
}

/// Shell Sort - ปรับปรุงจาก Insertion Sort
pub fn shellSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    // Ciura's gap sequence
    const gaps = [_]usize{ 701, 301, 132, 57, 23, 10, 4, 1 };
    
    for (gaps) |gap| {
        if (gap >= n) continue;
        
        var i: usize = gap;
        while (i < n) : (i += 1) {
            const temp = arr[i];
            var j: usize = i;
            
            while (j >= gap and arr[j - gap] > temp) {
                arr[j] = arr[j - gap];
                j -= gap;
            }
            arr[j] = temp;
        }
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    // ทดสอบกับข้อมูลต่างๆ
    var arr1 = [_]i32{ 12, 11, 13, 5, 6 };
    var arr2 = [_]i32{ 12, 11, 13, 5, 6 };
    var arr3 = [_]i32{ 12, 11, 13, 5, 6, 2, 8, 15, 3, 9 };
    
    try stdout.print("=== Insertion Sort ===\n", .{});
    
    try stdout.print("ก่อน: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    insertionSort(i32, &arr1);
    try stdout.print("Insertion Sort: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    binaryInsertionSort(i32, &arr2);
    try stdout.print("Binary Insertion Sort: ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    shellSort(i32, &arr3);
    try stdout.print("Shell Sort: ", .{});
    for (arr3) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    // ทดสอบกับข้อมูลที่เรียงลำดับแล้ว (best case)
    var sorted = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    insertionSort(i32, &sorted);
    try stdout.print("\nข้อมูลที่เรียงแล้ว: ", .{});
    for (sorted) |v| try stdout.print("{d} ", .{v});
    try stdout.print(" (ไม่มีการเปลี่ยนแปลง)\n", .{});
}
```

---

## Step 804: Merge Sort Implementation

### Merge Sort คืออะไร?

Merge Sort ใช้หลักการ Divide and Conquer แบ่งปัญหาออกเป็นส่วนย่อยๆ แก้แต่ละส่วน แล้วรวมผลลัพธ์

**Complexity:**
- Best/Average/Worst case: O(n log n)
- Space: O(n) - ต้องการหน่วยความจำเพิ่มเติม
- Stable sort

```zig
// merge_sort.zig
const std = @import("std");

/// รวม (merge) สอง sorted subarrays
fn merge(comptime T: type, arr: []T, left: usize, mid: usize, right: usize, allocator: std.mem.Allocator) !void {
    const leftLen = mid - left + 1;
    const rightLen = right - mid;
    
    // สร้าง temporary arrays
    const leftArr = try allocator.alloc(T, leftLen);
    defer allocator.free(leftArr);
    const rightArr = try allocator.alloc(T, rightLen);
    defer allocator.free(rightArr);
    
    // คัดลอกข้อมูล
    for (0..leftLen) |i| leftArr[i] = arr[left + i];
    for (0..rightLen) |j| rightArr[j] = arr[mid + 1 + j];
    
    // รวมกลับ
    var i: usize = 0;
    var j: usize = 0;
    var k: usize = left;
    
    while (i < leftLen and j < rightLen) {
        if (leftArr[i] <= rightArr[j]) {
            arr[k] = leftArr[i];
            i += 1;
        } else {
            arr[k] = rightArr[j];
            j += 1;
        }
        k += 1;
    }
    
    while (i < leftLen) {
        arr[k] = leftArr[i];
        i += 1;
        k += 1;
    }
    
    while (j < rightLen) {
        arr[k] = rightArr[j];
        j += 1;
        k += 1;
    }
}

/// Merge Sort แบบ recursive
pub fn mergeSort(comptime T: type, arr: []T, left: usize, right: usize, allocator: std.mem.Allocator) !void {
    if (left >= right) return;
    
    const mid = left + (right - left) / 2;
    
    try mergeSort(T, arr, left, mid, allocator);
    try mergeSort(T, arr, mid + 1, right, allocator);
    try merge(T, arr, left, mid, right, allocator);
}

/// Bottom-up Merge Sort (iterative)
pub fn mergeSortBottomUp(comptime T: type, arr: []T, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    if (n <= 1) return;
    
    var width: usize = 1;
    while (width < n) : (width *= 2) {
        var i: usize = 0;
        while (i < n) : (i += 2 * width) {
            const left = i;
            const mid = @min(i + width - 1, n - 1);
            const right = @min(i + 2 * width - 1, n - 1);
            
            if (mid < right) {
                try merge(T, arr, left, mid, right, allocator);
            }
        }
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    var arr1 = [_]i32{ 38, 27, 43, 3, 9, 82, 10 };
    var arr2 = [_]i32{ 38, 27, 43, 3, 9, 82, 10 };
    
    try stdout.print("ก่อน Merge Sort: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try mergeSort(i32, &arr1, 0, arr1.len - 1, allocator);
    
    try stdout.print("หลัง Merge Sort (recursive): ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try mergeSortBottomUp(i32, &arr2, allocator);
    
    try stdout.print("หลัง Merge Sort (bottom-up): ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    // ทดสอบ stability - elements ที่เท่ากันควรรักษาลำดับเดิม
    const Entry = struct { value: i32, order: usize };
    var entries = [_]Entry{
        .{ .value = 3, .order = 0 },
        .{ .value = 1, .order = 1 },
        .{ .value = 3, .order = 2 },
        .{ .value = 2, .order = 3 },
        .{ .value = 1, .order = 4 },
    };
    
    // Merge sort สำหรับ struct
    // (simplified - ในความเป็นจริงต้องใช้ comparator)
    try stdout.print("\nMerge Sort เป็น stable sort\n", .{});
    _ = entries;
}
```

---

## Step 805: Quick Sort Implementation

### Quick Sort คืออะไร?

Quick Sort เป็นอัลกอริทึมที่ใช้บ่อยที่สุดในทางปฏิบัติ ใช้หลักการ Divide and Conquer โดยเลือก pivot แล้วแบ่ง array เป็นสองส่วน

**Complexity:**
- Best/Average case: O(n log n)
- Worst case: O(n²) - เมื่อ pivot แย่มาก
- Space: O(log n) - stack space
- In-place (ส่วนใหญ่)

```zig
// quick_sort.zig
const std = @import("std");

/// Partition แบบ Lomuto
fn partitionLomuto(comptime T: type, arr: []T, low: usize, high: usize) usize {
    const pivot = arr[high];
    var i: usize = low;
    
    var j: usize = low;
    while (j < high) : (j += 1) {
        if (arr[j] <= pivot) {
            const temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
            i += 1;
        }
    }
    
    const temp = arr[i];
    arr[i] = arr[high];
    arr[high] = temp;
    
    return i;
}

/// Partition แบบ Hoare (มีประสิทธิภาพกว่า)
fn partitionHoare(comptime T: type, arr: []T, low: usize, high: usize) usize {
    const pivot = arr[low + (high - low) / 2];
    var i: usize = low;
    var j: usize = high;
    
    while (true) {
        while (arr[i] < pivot) i += 1;
        while (arr[j] > pivot) j -= 1;
        
        if (i >= j) return j;
        
        const temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
        i += 1;
        if (j > 0) j -= 1;
    }
}

/// Quick Sort แบบพื้นฐาน
pub fn quickSort(comptime T: type, arr: []T, low: usize, high: usize) void {
    if (low >= high) return;
    
    const pi = partitionLomuto(T, arr, low, high);
    
    if (pi > 0) quickSort(T, arr, low, pi - 1);
    quickSort(T, arr, pi + 1, high);
}

/// Quick Sort แบบ 3-way partition (สำหรับข้อมูลที่มีค่าซ้ำมาก)
pub fn quickSort3Way(comptime T: type, arr: []T, low: usize, high: usize) void {
    if (low >= high) return;
    
    const pivot = arr[low];
    var lt = low;   // arr[low..lt] < pivot
    var gt = high;  // arr[gt+1..high] > pivot
    var i = low + 1;
    
    while (i <= gt) {
        if (arr[i] < pivot) {
            const temp = arr[lt];
            arr[lt] = arr[i];
            arr[i] = temp;
            lt += 1;
            i += 1;
        } else if (arr[i] > pivot) {
            const temp = arr[i];
            arr[i] = arr[gt];
            arr[gt] = temp;
            if (gt == 0) break;
            gt -= 1;
        } else {
            i += 1;
        }
    }
    
    if (lt > low) quickSort3Way(T, arr, low, lt - 1);
    if (gt < high) quickSort3Way(T, arr, gt + 1, high);
}

/// Median-of-3 pivot selection
fn medianOfThree(comptime T: type, arr: []T, low: usize, high: usize) void {
    const mid = low + (high - low) / 2;
    
    if (arr[low] > arr[mid]) {
        const temp = arr[low];
        arr[low] = arr[mid];
        arr[mid] = temp;
    }
    if (arr[low] > arr[high]) {
        const temp = arr[low];
        arr[low] = arr[high];
        arr[high] = temp;
    }
    if (arr[mid] > arr[high]) {
        const temp = arr[mid];
        arr[mid] = arr[high];
        arr[high] = temp;
    }
    
    // ย้าย median ไปที่ high-1
    const temp = arr[mid];
    arr[mid] = arr[high - 1];
    arr[high - 1] = temp;
}

/// Introspective Sort (เริ่มด้วย Quick Sort แล้วเปลี่ยนเป็น Heap Sort เมื่อ depth เยอะ)
pub fn introSort(comptime T: type, arr: []T) void {
    const maxDepth = 2 * std.math.log2_int(usize, @max(arr.len, 1));
    introSortHelper(T, arr, 0, arr.len, maxDepth);
}

fn introSortHelper(comptime T: type, arr: []T, low: usize, high: usize, depth: usize) void {
    if (high - low <= 1) return;
    
    // ถ้า depth เกิน switch ไป heap sort
    if (depth == 0) {
        heapSortRange(T, arr, low, high);
        return;
    }
    
    // ถ้าขนาดเล็ก ใช้ insertion sort
    if (high - low <= 16) {
        insertionSortRange(T, arr, low, high);
        return;
    }
    
    const pi = partitionLomuto(T, arr, low, high - 1);
    if (pi > low) introSortHelper(T, arr, low, pi, depth - 1);
    introSortHelper(T, arr, pi + 1, high, depth - 1);
}

fn heapSortRange(comptime T: type, arr: []T, low: usize, high: usize) void {
    const sub = arr[low..high];
    heapifyAll(T, sub);
    var end = sub.len - 1;
    while (end > 0) : (end -= 1) {
        const temp = sub[0];
        sub[0] = sub[end];
        sub[end] = temp;
        siftDown(T, sub, 0, end);
    }
}

fn heapifyAll(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    var i: usize = n / 2;
    while (true) {
        siftDown(T, arr, i, n);
        if (i == 0) break;
        i -= 1;
    }
}

fn siftDown(comptime T: type, arr: []T, start: usize, end: usize) void {
    var root = start;
    while (true) {
        var child = 2 * root + 1;
        if (child >= end) break;
        
        if (child + 1 < end and arr[child] < arr[child + 1]) {
            child += 1;
        }
        
        if (arr[root] >= arr[child]) break;
        
        const temp = arr[root];
        arr[root] = arr[child];
        arr[child] = temp;
        root = child;
    }
}

fn insertionSortRange(comptime T: type, arr: []T, low: usize, high: usize) void {
    var i: usize = low + 1;
    while (i < high) : (i += 1) {
        const key = arr[i];
        var j = i;
        while (j > low and arr[j - 1] > key) : (j -= 1) {
            arr[j] = arr[j - 1];
        }
        arr[j] = key;
    }
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    
    var arr1 = [_]i32{ 10, 7, 8, 9, 1, 5 };
    var arr2 = [_]i32{ 4, 2, 4, 3, 1, 4, 2 }; // มีค่าซ้ำ
    var arr3 = [_]i32{ 10, 7, 8, 9, 1, 5, 3, 15, 22, 6 };
    
    try stdout.print("=== Quick Sort ===\n", .{});
    
    try stdout.print("ก่อน: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    quickSort(i32, &arr1, 0, arr1.len - 1);
    try stdout.print("Quick Sort: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    quickSort3Way(i32, &arr2, 0, arr2.len - 1);
    try stdout.print("3-Way Quick Sort (ข้อมูลซ้ำ): ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    introSort(i32, &arr3);
    try stdout.print("Intro Sort: ", .{});
    for (arr3) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
}
```

---

## Step 806: Heap Sort Implementation

### Heap Sort คืออะไร?

Heap Sort ใช้โครงสร้างข้อมูล Heap (Binary Heap) ในการเรียงลำดับ

**Complexity:**
- Best/Average/Worst case: O(n log n)
- Space: O(1) - in-place
- ไม่ stable

```zig
// heap_sort.zig
const std = @import("std");

/// สร้าง max-heap จาก subtree ที่ root คือ i
fn heapify(comptime T: type, arr: []T, n: usize, i: usize) void {
    var largest = i;
    const left = 2 * i + 1;
    const right = 2 * i + 2;
    
    if (left < n and arr[left] > arr[largest]) {
        largest = left;
    }
    
    if (right < n and arr[right] > arr[largest]) {
        largest = right;
    }
    
    if (largest != i) {
        const temp = arr[i];
        arr[i] = arr[largest];
        arr[largest] = temp;
        heapify(T, arr, n, largest);
    }
}

/// Heap Sort หลัก
pub fn heapSort(comptime T: type, arr: []T) void {
    const n = arr.len;
    if (n <= 1) return;
    
    // Build max-heap
    // เริ่มจาก non-leaf node สุดท้าย
    var i: usize = n / 2;
    while (true) {
        heapify(T, arr, n, i);
        if (i == 0) break;
        i -= 1;
    }
    
    // Extract elements จาก heap ทีละตัว
    var end = n - 1;
    while (end > 0) : (end -= 1) {
        // ย้าย root (max) ไปท้าย
        const temp = arr[0];
        arr[0] = arr[end];
        arr[end] = temp;
        
        // heapify ส่วนที่เหลือ
        heapify(T, arr, end, 0);
    }
}

/// Min-Heap สำหรับเรียงลำดับจากมากไปน้อย
fn heapifyMin(comptime T: type, arr: []T, n: usize, i: usize) void {
    var smallest = i;
    const left = 2 * i + 1;
    const right = 2 * i + 2;
    
    if (left < n and arr[left] < arr[smallest]) {
        smallest = left;
    }
    
    if (right < n and arr[right] < arr[smallest]) {
        smallest = right;
    }
    
    if (smallest != i) {
        const temp = arr[i];
        arr[i] = arr[smallest];
        arr[smallest] = temp;
        heapifyMin(T, arr, n, smallest);
    }
}

/// Priority Queue โดยใช้ Heap
pub fn PriorityQueue(comptime T: type) type {
    return struct {
        const Self = @This();
        
        data: []T,
        size: usize,
        allocator: std.mem.Allocator,
        
        pub fn init(allocator: std.mem.Allocator, capacity: usize) !Self {
            return Self{
                .data = try allocator.alloc(T, capacity),
                .size = 0,
                .allocator = allocator,
            };
        }
        
        pub fn deinit(self: *Self) void {
            self.allocator.free(self.data);
        }
        
        pub fn push(self: *Self, value: T) !void {
            if (self.size >= self.data.len) return error.Overflow;
            
            self.data[self.size] = value;
            self.size += 1;
            
            // Bubble up
            var i = self.size - 1;
            while (i > 0) {
                const parent = (i - 1) / 2;
                if (self.data[parent] >= self.data[i]) break;
                
                const temp = self.data[parent];
                self.data[parent] = self.data[i];
                self.data[i] = temp;
                i = parent;
            }
        }
        
        pub fn pop(self: *Self) ?T {
            if (self.size == 0) return null;
            
            const top = self.data[0];
            self.size -= 1;
            self.data[0] = self.data[self.size];
            
            // Sift down
            heapify(T, self.data[0..self.size], self.size, 0);
            
            return top;
        }
        
        pub fn peek(self: *const Self) ?T {
            if (self.size == 0) return null;
            return self.data[0];
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    var arr = [_]i32{ 12, 11, 13, 5, 6, 7 };
    
    try stdout.print("=== Heap Sort ===\n", .{});
    try stdout.print("ก่อน: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    heapSort(i32, &arr);
    
    try stdout.print("หลัง Heap Sort: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    // ทดสอบ Priority Queue
    try stdout.print("\n=== Priority Queue ===\n", .{});
    var pq = try PriorityQueue(i32).init(allocator, 10);
    defer pq.deinit();
    
    try pq.push(3);
    try pq.push(1);
    try pq.push(4);
    try pq.push(1);
    try pq.push(5);
    
    try stdout.print("Priority Queue (max-heap): ", .{});
    while (pq.pop()) |v| {
        try stdout.print("{d} ", .{v});
    }
    try stdout.print("\n", .{});
}
```

---

## Step 807: Counting Sort (Non-comparison)

### Counting Sort คืออะไร?

Counting Sort เป็น non-comparison sort ที่ทำงานโดยนับจำนวนของแต่ละค่า แล้วใช้ข้อมูลการนับนั้นสร้าง output ที่เรียงลำดับแล้ว

**Complexity:**
- Time: O(n + k) โดย k คือ range ของข้อมูล
- Space: O(n + k)
- Stable sort

```zig
// counting_sort.zig
const std = @import("std");

/// Counting Sort สำหรับ non-negative integers
pub fn countingSort(arr: []u32, max_val: u32, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    if (n <= 1) return;
    
    const k = max_val + 1;
    
    // สร้าง count array
    const count = try allocator.alloc(usize, k);
    defer allocator.free(count);
    @memset(count, 0);
    
    // นับจำนวนของแต่ละค่า
    for (arr) |v| {
        count[v] += 1;
    }
    
    // คำนวณ cumulative count (prefix sum)
    var i: usize = 1;
    while (i < k) : (i += 1) {
        count[i] += count[i - 1];
    }
    
    // สร้าง output array
    const output = try allocator.alloc(u32, n);
    defer allocator.free(output);
    
    // วาง element ในตำแหน่งที่ถูกต้อง (เดินจากขวาเพื่อ stability)
    var j = n;
    while (j > 0) {
        j -= 1;
        const v = arr[j];
        count[v] -= 1;
        output[count[v]] = v;
    }
    
    // คัดลอกกลับ
    @memcpy(arr, output);
}

/// Counting Sort สำหรับ characters (สำหรับ Radix Sort)
pub fn countingSortByDigit(arr: []u32, exp: u32, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    const output = try allocator.alloc(u32, n);
    defer allocator.free(output);
    
    var count = [_]usize{0} ** 10;
    
    // นับตัวเลขในตำแหน่ง exp
    for (arr) |v| {
        const digit = (v / exp) % 10;
        count[digit] += 1;
    }
    
    // Cumulative count
    var i: usize = 1;
    while (i < 10) : (i += 1) {
        count[i] += count[i - 1];
    }
    
    // สร้าง output (stable)
    var j = n;
    while (j > 0) {
        j -= 1;
        const digit = (arr[j] / exp) % 10;
        count[digit] -= 1;
        output[count[digit]] = arr[j];
    }
    
    @memcpy(arr, output);
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    var arr = [_]u32{ 4, 2, 2, 8, 3, 3, 1 };
    
    try stdout.print("=== Counting Sort ===\n", .{});
    try stdout.print("ก่อน: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try countingSort(&arr, 8, allocator);
    
    try stdout.print("หลัง Counting Sort: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    // ตัวอย่าง: เรียงลำดับ grades
    var grades = [_]u32{ 85, 92, 70, 88, 92, 75, 65, 88, 92, 100 };
    try stdout.print("\nGrades ก่อนเรียง: ", .{});
    for (grades) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try countingSort(&grades, 100, allocator);
    try stdout.print("Grades หลังเรียง: ", .{});
    for (grades) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
}
```

---

## Step 808: Radix Sort Implementation

### Radix Sort คืออะไร?

Radix Sort เรียงลำดับโดยประมวลผลทีละหลัก (digit) ของตัวเลข ใช้ Counting Sort เป็น subroutine

**Complexity:**
- Time: O(d × (n + k)) โดย d คือจำนวนหลัก
- Space: O(n + k)
- Stable sort

```zig
// radix_sort.zig
const std = @import("std");

/// หาค่าสูงสุดใน array
fn findMax(arr: []const u32) u32 {
    var max: u32 = 0;
    for (arr) |v| {
        if (v > max) max = v;
    }
    return max;
}

/// Radix Sort แบบ LSD (Least Significant Digit)
pub fn radixSortLSD(arr: []u32, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    if (n <= 1) return;
    
    const max = findMax(arr);
    
    // ประมวลผลทีละหลัก ตั้งแต่หลักหน่วยจนถึงหลักสูงสุด
    var exp: u32 = 1;
    while (max / exp > 0) : (exp *= 10) {
        try countingSortByDigit(arr, exp, allocator);
    }
}

fn countingSortByDigit(arr: []u32, exp: u32, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    const output = try allocator.alloc(u32, n);
    defer allocator.free(output);
    
    var count = [_]usize{0} ** 10;
    
    for (arr) |v| {
        const digit = (v / exp) % 10;
        count[digit] += 1;
    }
    
    var i: usize = 1;
    while (i < 10) : (i += 1) {
        count[i] += count[i - 1];
    }
    
    var j = n;
    while (j > 0) {
        j -= 1;
        const digit = (arr[j] / exp) % 10;
        count[digit] -= 1;
        output[count[digit]] = arr[j];
    }
    
    @memcpy(arr, output);
}

/// Radix Sort แบบ MSD (Most Significant Digit)
pub fn radixSortMSD(arr: []u32, allocator: std.mem.Allocator) !void {
    if (arr.len <= 1) return;
    
    const max = findMax(arr);
    
    // หาจำนวนหลัก
    var digits: u32 = 0;
    var temp = max;
    while (temp > 0) : (temp /= 10) digits += 1;
    if (digits == 0) digits = 1;
    
    var exp: u32 = 1;
    var d: u32 = 0;
    while (d < digits - 1) : (d += 1) exp *= 10;
    
    try radixSortMSDHelper(arr, exp, allocator);
}

fn radixSortMSDHelper(arr: []u32, exp: u32, allocator: std.mem.Allocator) !void {
    if (arr.len <= 1 or exp == 0) return;
    
    const n = arr.len;
    var buckets: [10]std.ArrayListUnmanaged(u32) = undefined;
    for (&buckets) |*b| b.* = .{};
    defer {
        for (&buckets) |*b| b.deinit(allocator);
    }
    
    // แจกลง buckets
    for (arr) |v| {
        const digit = (v / exp) % 10;
        try buckets[digit].append(allocator, v);
    }
    
    // เรียงลำดับแต่ละ bucket แบบ recursive
    if (exp >= 10) {
        for (&buckets) |*b| {
            if (b.items.len > 1) {
                try radixSortMSDHelper(b.items, exp / 10, allocator);
            }
        }
    }
    
    // รวมกลับ
    var i: usize = 0;
    for (buckets) |b| {
        for (b.items) |v| {
            arr[i] = v;
            i += 1;
        }
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    var arr1 = [_]u32{ 170, 45, 75, 90, 802, 24, 2, 66 };
    var arr2 = [_]u32{ 170, 45, 75, 90, 802, 24, 2, 66 };
    
    try stdout.print("=== Radix Sort ===\n", .{});
    try stdout.print("ก่อน: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try radixSortLSD(&arr1, allocator);
    try stdout.print("Radix Sort LSD: ", .{});
    for (arr1) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try radixSortMSD(&arr2, allocator);
    try stdout.print("Radix Sort MSD: ", .{});
    for (arr2) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
}
```

---

## Step 809: Tim Sort (Hybrid Sort)

### Tim Sort คืออะไร?

Tim Sort เป็น hybrid algorithm ที่ใช้ Insertion Sort สำหรับข้อมูลขนาดเล็ก และ Merge Sort สำหรับข้อมูลขนาดใหญ่ เป็น default sort ใน Python และ Java

**Complexity:**
- Best case: O(n) - ข้อมูลเรียงลำดับแล้ว
- Average/Worst case: O(n log n)
- Space: O(n)
- Stable sort

```zig
// tim_sort.zig
const std = @import("std");

const RUN = 32; // ขนาด run สำหรับ Tim Sort

/// Insertion Sort สำหรับ subarray
fn insertionSortTim(comptime T: type, arr: []T, left: usize, right: usize) void {
    var i: usize = left + 1;
    while (i <= right) : (i += 1) {
        const key = arr[i];
        var j = i;
        while (j > left and arr[j - 1] > key) : (j -= 1) {
            arr[j] = arr[j - 1];
        }
        arr[j] = key;
    }
}

/// Merge สำหรับ Tim Sort
fn timMerge(comptime T: type, arr: []T, left: usize, mid: usize, right: usize, allocator: std.mem.Allocator) !void {
    const leftLen = mid - left + 1;
    const rightLen = right - mid;
    
    const leftArr = try allocator.alloc(T, leftLen);
    defer allocator.free(leftArr);
    const rightArr = try allocator.alloc(T, rightLen);
    defer allocator.free(rightArr);
    
    @memcpy(leftArr, arr[left .. left + leftLen]);
    @memcpy(rightArr, arr[mid + 1 .. mid + 1 + rightLen]);
    
    var i: usize = 0;
    var j: usize = 0;
    var k = left;
    
    while (i < leftLen and j < rightLen) {
        if (leftArr[i] <= rightArr[j]) {
            arr[k] = leftArr[i];
            i += 1;
        } else {
            arr[k] = rightArr[j];
            j += 1;
        }
        k += 1;
    }
    
    while (i < leftLen) {
        arr[k] = leftArr[i];
        i += 1;
        k += 1;
    }
    
    while (j < rightLen) {
        arr[k] = rightArr[j];
        j += 1;
        k += 1;
    }
}

/// Tim Sort หลัก
pub fn timSort(comptime T: type, arr: []T, allocator: std.mem.Allocator) !void {
    const n = arr.len;
    if (n <= 1) return;
    
    // เรียง runs ด้วย Insertion Sort
    var i: usize = 0;
    while (i < n) : (i += RUN) {
        const right = @min(i + RUN - 1, n - 1);
        insertionSortTim(T, arr, i, right);
    }
    
    // Merge runs ที่เรียงแล้ว
    var size: usize = RUN;
    while (size < n) : (size *= 2) {
        var left: usize = 0;
        while (left < n) : (left += 2 * size) {
            const mid = @min(left + size - 1, n - 1);
            const right = @min(left + 2 * size - 1, n - 1);
            
            if (mid < right) {
                try timMerge(T, arr, left, mid, right, allocator);
            }
        }
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    var arr = [_]i32{ 5, 21, 7, 23, 19, 3, 11, 1, 17, 13, 15, 9 };
    
    try stdout.print("=== Tim Sort ===\n", .{});
    try stdout.print("ก่อน: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try timSort(i32, &arr, allocator);
    
    try stdout.print("หลัง Tim Sort: ", .{});
    for (arr) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    // ทดสอบกับข้อมูล nearly-sorted (Tim Sort ทำงานได้ดี)
    var nearly_sorted = [_]i32{ 1, 2, 3, 5, 4, 6, 7, 8, 10, 9 };
    try stdout.print("\nNearly sorted ก่อน: ", .{});
    for (nearly_sorted) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    try timSort(i32, &nearly_sorted, allocator);
    try stdout.print("Nearly sorted หลัง: ", .{});
    for (nearly_sorted) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
}
```

---

## Step 810: Sort Algorithm Comparison และ Benchmarks

### สรุปการเปรียบเทียบ Sorting Algorithms

```zig
// sort_benchmark.zig
const std = @import("std");

/// สร้าง random array
fn generateRandom(arr: []u32, seed: u64) void {
    var rng = std.rand.DefaultPrng.init(seed);
    const random = rng.random();
    for (arr) |*v| {
        v.* = random.int(u32) % 10000;
    }
}

/// สร้าง sorted array
fn generateSorted(arr: []u32) void {
    for (arr, 0..) |*v, i| {
        v.* = @intCast(i);
    }
}

/// สร้าง reverse sorted array
fn generateReverseSorted(arr: []u32) void {
    const n = arr.len;
    for (arr, 0..) |*v, i| {
        v.* = @intCast(n - 1 - i);
    }
}

/// วัดเวลาการเรียงลำดับ
fn benchmark(
    name: []const u8,
    arr: []u32,
    sortFn: *const fn ([]u32) void,
    stdout: anytype,
) !void {
    const start = std.time.nanoTimestamp();
    sortFn(arr);
    const end = std.time.nanoTimestamp();
    const elapsed = end - start;
    
    try stdout.print("  {s}: {d} ns\n", .{ name, elapsed });
}

fn bubbleSortU32(arr: []u32) void {
    const n = arr.len;
    var i: usize = 0;
    while (i < n - 1) : (i += 1) {
        var swapped = false;
        var j: usize = 0;
        while (j < n - 1 - i) : (j += 1) {
            if (arr[j] > arr[j + 1]) {
                const temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        if (!swapped) break;
    }
}

fn insertionSortU32(arr: []u32) void {
    const n = arr.len;
    var i: usize = 1;
    while (i < n) : (i += 1) {
        const key = arr[i];
        var j = i;
        while (j > 0 and arr[j - 1] > key) : (j -= 1) {
            arr[j] = arr[j - 1];
        }
        arr[j] = key;
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    const sizes = [_]usize{ 100, 1000, 5000 };
    
    try stdout.print("=== Sorting Algorithm Benchmark ===\n\n", .{});
    
    for (sizes) |size| {
        try stdout.print("ขนาดข้อมูล: {d}\n", .{size});
        
        // Random data
        try stdout.print("  [Random Data]\n", .{});
        
        const arr1 = try allocator.alloc(u32, size);
        defer allocator.free(arr1);
        const arr2 = try allocator.alloc(u32, size);
        defer allocator.free(arr2);
        
        generateRandom(arr1, 42);
        @memcpy(arr2, arr1);
        
        try benchmark("Bubble Sort", arr1, bubbleSortU32, stdout);
        try benchmark("Insertion Sort", arr2, insertionSortU32, stdout);
        
        // Sorted data
        try stdout.print("  [Sorted Data]\n", .{});
        generateSorted(arr1);
        @memcpy(arr2, arr1);
        
        try benchmark("Bubble Sort (sorted)", arr1, bubbleSortU32, stdout);
        try benchmark("Insertion Sort (sorted)", arr2, insertionSortU32, stdout);
        
        try stdout.print("\n", .{});
    }
    
    // สรุปตาราง
    try stdout.print("=== สรุปความซับซ้อน ===\n\n", .{});
    try stdout.print("{s:<20} {s:<12} {s:<12} {s:<12} {s:<8} {s:<8}\n", .{
        "Algorithm", "Best", "Average", "Worst", "Space", "Stable"
    });
    try stdout.print("{s}\n", .{"=" ** 80});
    
    const algorithms = [_][6][]const u8{
        .{ "Bubble Sort",     "O(n)",      "O(n²)",      "O(n²)",      "O(1)",    "Yes" },
        .{ "Insertion Sort",  "O(n)",      "O(n²)",      "O(n²)",      "O(1)",    "Yes" },
        .{ "Merge Sort",      "O(n log n)","O(n log n)", "O(n log n)", "O(n)",    "Yes" },
        .{ "Quick Sort",      "O(n log n)","O(n log n)", "O(n²)",      "O(log n)","No"  },
        .{ "Heap Sort",       "O(n log n)","O(n log n)", "O(n log n)", "O(1)",    "No"  },
        .{ "Counting Sort",   "O(n+k)",    "O(n+k)",     "O(n+k)",     "O(n+k)", "Yes" },
        .{ "Radix Sort",      "O(nk)",     "O(nk)",      "O(nk)",      "O(n+k)", "Yes" },
        .{ "Tim Sort",        "O(n)",      "O(n log n)", "O(n log n)", "O(n)",    "Yes" },
    };
    
    for (algorithms) |alg| {
        try stdout.print("{s:<20} {s:<12} {s:<12} {s:<12} {s:<8} {s:<8}\n", .{
            alg[0], alg[1], alg[2], alg[3], alg[4], alg[5]
        });
    }
    
    try stdout.print("\n=== คำแนะนำการเลือกใช้ ===\n", .{});
    try stdout.print("- ข้อมูลเล็ก (< 50): Insertion Sort\n", .{});
    try stdout.print("- ข้อมูลทั่วไป: Quick Sort หรือ Tim Sort\n", .{});
    try stdout.print("- ต้องการ stable sort: Merge Sort หรือ Tim Sort\n", .{});
    try stdout.print("- ต้องการ worst-case guarantee: Merge Sort หรือ Heap Sort\n", .{});
    try stdout.print("- integers ใน range จำกัด: Counting Sort หรือ Radix Sort\n", .{});
}
```

### สรุป Part 81

ในส่วนนี้เราได้เรียนรู้:
1. ทฤษฎีเบื้องหลัง sorting algorithms และ Big O notation
2. Bubble Sort - ง่ายแต่ช้า
3. Insertion Sort - ดีสำหรับข้อมูลเล็กหรือ nearly-sorted
4. Merge Sort - O(n log n) guaranteed, stable
5. Quick Sort - เร็วในทางปฏิบัติ, in-place
6. Heap Sort - O(n log n) guaranteed, in-place
7. Counting Sort - O(n+k), ดีสำหรับ integer range จำกัด
8. Radix Sort - เรียงตาม digit
9. Tim Sort - hybrid, ใช้จริงใน Python/Java
10. การ benchmark และเปรียบเทียบ

---

*ต่อไป: Part 82 - Algorithms: Searching ใน Zig*
