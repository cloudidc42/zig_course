# ส่วนที่ 77: Data Structures: Linked List (ขั้นตอนที่ 761-770)

Linked List เป็นโครงสร้างข้อมูลพื้นฐานที่ nodes เชื่อมต่อกันผ่าน pointers ใน Zig เราสามารถสร้าง linked list ที่มีประสิทธิภาพสูงโดยควบคุม memory ได้อย่างละเอียด

---

## ขั้นตอนที่ 761: Singly Linked List

Singly Linked List คือ list ที่แต่ละ node ชี้ไปยัง node ถัดไปเพียงทิศทางเดียว

### โครงสร้างพื้นฐาน

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn SinglyLinkedList(comptime T: type) type {
    return struct {
        const Self = @This();

        const Node = struct {
            data: T,
            next: ?*Node = null,
        };

        head: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return Self{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            var current = self.head;
            while (current) |node| {
                const next = node.next;
                self.allocator.destroy(node);
                current = next;
            }
            self.head = null;
            self.size = 0;
        }

        // เพิ่ม node ที่ต้นลิสต์ O(1)
        pub fn prepend(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = Node{ .data = data, .next = self.head };
            self.head = node;
            self.size += 1;
        }

        // เพิ่ม node ที่ท้ายลิสต์ O(n)
        pub fn append(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = Node{ .data = data, .next = null };

            if (self.head == null) {
                self.head = node;
            } else {
                var current = self.head.?;
                while (current.next) |next| {
                    current = next;
                }
                current.next = node;
            }
            self.size += 1;
        }

        // ลบ node แรก O(1)
        pub fn removeFirst(self: *Self) ?T {
            const node = self.head orelse return null;
            self.head = node.next;
            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        // ค้นหา data ใน list O(n)
        pub fn contains(self: *Self, data: T) bool {
            var current = self.head;
            while (current) |node| {
                if (node.data == data) return true;
                current = node.next;
            }
            return false;
        }

        // ลบ node ที่มีค่า data O(n)
        pub fn remove(self: *Self, data: T) bool {
            if (self.head == null) return false;

            // กรณีลบ head
            if (self.head.?.data == data) {
                _ = self.removeFirst();
                return true;
            }

            var current = self.head.?;
            while (current.next) |next| {
                if (next.data == data) {
                    current.next = next.next;
                    self.allocator.destroy(next);
                    self.size -= 1;
                    return true;
                }
                current = next;
            }
            return false;
        }

        // แสดงรายการทั้งหมด
        pub fn print(self: *Self) void {
            std.debug.print("List({d}): ", .{self.size});
            var current = self.head;
            while (current) |node| {
                if (@typeInfo(T) == .Int) {
                    std.debug.print("{d}", .{node.data});
                } else {
                    std.debug.print("{any}", .{node.data});
                }
                if (node.next != null) std.debug.print(" -> ", .{});
                current = node.next;
            }
            std.debug.print(" -> null\n", .{});
        }

        // Reverse the list O(n)
        pub fn reverse(self: *Self) void {
            var prev: ?*Node = null;
            var current = self.head;
            while (current) |node| {
                const next = node.next;
                node.next = prev;
                prev = node;
                current = next;
            }
            self.head = prev;
        }

        // หา middle node O(n) - Floyd's algorithm
        pub fn findMiddle(self: *Self) ?*Node {
            var slow = self.head;
            var fast = self.head;
            while (fast != null and fast.?.next != null) {
                slow = slow.?.next;
                fast = fast.?.next.?.next;
            }
            return slow;
        }

        // ตรวจสอบว่ามี cycle หรือไม่
        pub fn hasCycle(self: *Self) bool {
            var slow = self.head;
            var fast = self.head;
            while (fast != null and fast.?.next != null) {
                slow = slow.?.next;
                fast = fast.?.next.?.next;
                if (slow == fast) return true;
            }
            return false;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Singly Linked List Demo ===\n", .{});

    var list = SinglyLinkedList(i32).init(allocator);
    defer list.deinit();

    // เพิ่มข้อมูล
    try list.append(1);
    try list.append(2);
    try list.append(3);
    try list.append(4);
    try list.append(5);
    list.print();

    // Prepend
    try list.prepend(0);
    list.print();

    // ค้นหา
    std.debug.print("มี 3? {}\n", .{list.contains(3)});
    std.debug.print("มี 10? {}\n", .{list.contains(10)});

    // ลบ
    _ = list.remove(3);
    list.print();

    // Reverse
    list.reverse();
    list.print();

    // Middle
    if (list.findMiddle()) |mid| {
        std.debug.print("Middle: {d}\n", .{mid.data});
    }
}
```

---

## ขั้นตอนที่ 762: Doubly Linked List

Doubly Linked List มี pointer ทั้งไปข้างหน้าและข้างหลัง ทำให้การ traverse ได้ทั้งสองทิศทาง

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn DoublyLinkedList(comptime T: type) type {
    return struct {
        const Self = @This();

        pub const Node = struct {
            data: T,
            prev: ?*Node = null,
            next: ?*Node = null,
        };

        head: ?*Node = null,
        tail: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return Self{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            var current = self.head;
            while (current) |node| {
                const next = node.next;
                self.allocator.destroy(node);
                current = next;
            }
        }

        pub fn pushFront(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = Node{ .data = data };

            if (self.head) |head| {
                node.next = head;
                head.prev = node;
            } else {
                self.tail = node;
            }
            self.head = node;
            self.size += 1;
        }

        pub fn pushBack(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = Node{ .data = data };

            if (self.tail) |tail| {
                tail.next = node;
                node.prev = tail;
            } else {
                self.head = node;
            }
            self.tail = node;
            self.size += 1;
        }

        pub fn popFront(self: *Self) ?T {
            const node = self.head orelse return null;
            self.head = node.next;
            if (self.head) |new_head| {
                new_head.prev = null;
            } else {
                self.tail = null;
            }
            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        pub fn popBack(self: *Self) ?T {
            const node = self.tail orelse return null;
            self.tail = node.prev;
            if (self.tail) |new_tail| {
                new_tail.next = null;
            } else {
                self.head = null;
            }
            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        // Insert หลัง node ที่กำหนด
        pub fn insertAfter(self: *Self, node: *Node, data: T) !void {
            const new_node = try self.allocator.create(Node);
            new_node.* = Node{
                .data = data,
                .prev = node,
                .next = node.next,
            };

            if (node.next) |next| {
                next.prev = new_node;
            } else {
                self.tail = new_node;
            }
            node.next = new_node;
            self.size += 1;
        }

        // ลบ node ที่กำหนด O(1)
        pub fn removeNode(self: *Self, node: *Node) T {
            if (node.prev) |prev| {
                prev.next = node.next;
            } else {
                self.head = node.next;
            }

            if (node.next) |next| {
                next.prev = node.prev;
            } else {
                self.tail = node.prev;
            }

            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        pub fn printForward(self: *Self) void {
            std.debug.print("Forward: null <-> ", .{});
            var current = self.head;
            while (current) |node| {
                std.debug.print("{d} <-> ", .{node.data});
                current = node.next;
            }
            std.debug.print("null\n", .{});
        }

        pub fn printBackward(self: *Self) void {
            std.debug.print("Backward: null <-> ", .{});
            var current = self.tail;
            while (current) |node| {
                std.debug.print("{d} <-> ", .{node.data});
                current = node.prev;
            }
            std.debug.print("null\n", .{});
        }
    };
}

// LRU Cache ใช้ Doubly Linked List + HashMap
fn LRUCache(comptime K: type, comptime V: type) type {
    return struct {
        const Self = @This();
        const List = DoublyLinkedList(struct { key: K, value: V });
        const Map = std.AutoHashMap(K, *List.Node);

        list: List,
        map: Map,
        capacity: usize,

        pub fn init(allocator: Allocator, capacity: usize) Self {
            return Self{
                .list = List.init(allocator),
                .map = Map.init(allocator),
                .capacity = capacity,
            };
        }

        pub fn deinit(self: *Self) void {
            self.list.deinit();
            self.map.deinit();
        }

        pub fn get(self: *Self, key: K) ?V {
            if (self.map.get(key)) |node| {
                // Move to front (most recently used)
                const data = self.list.removeNode(node);
                self.list.pushFront(data) catch return null;
                const new_node = self.list.head.?;
                self.map.put(key, new_node) catch return null;
                return new_node.data.value;
            }
            return null;
        }

        pub fn put(self: *Self, key: K, value: V) !void {
            if (self.map.get(key)) |node| {
                _ = self.list.removeNode(node);
                _ = self.map.remove(key);
            } else if (self.list.size >= self.capacity) {
                // Evict LRU (tail)
                if (self.list.tail) |tail| {
                    _ = self.map.remove(tail.data.key);
                    _ = self.list.popBack();
                }
            }

            try self.list.pushFront(.{ .key = key, .value = value });
            try self.map.put(key, self.list.head.?);
        }

        pub fn printCache(self: *Self) void {
            std.debug.print("LRU Cache: ", .{});
            var current = self.list.head;
            while (current) |node| {
                std.debug.print("[{d}:{d}] ", .{ node.data.key, node.data.value });
                current = node.next;
            }
            std.debug.print("\n", .{});
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Doubly Linked List Demo ===\n", .{});

    var list = DoublyLinkedList(i32).init(allocator);
    defer list.deinit();

    try list.pushBack(1);
    try list.pushBack(2);
    try list.pushBack(3);
    try list.pushFront(0);
    list.printForward();
    list.printBackward();

    _ = list.popFront();
    _ = list.popBack();
    list.printForward();

    // LRU Cache
    std.debug.print("\n=== LRU Cache Demo ===\n", .{});
    var cache = LRUCache(i32, i32).init(allocator, 3);
    defer cache.deinit();

    try cache.put(1, 100);
    try cache.put(2, 200);
    try cache.put(3, 300);
    cache.printCache();

    _ = cache.get(1); // Access 1
    cache.printCache();

    try cache.put(4, 400); // Evict LRU (2)
    cache.printCache();
}
```

---

## ขั้นตอนที่ 763: Circular Linked List

Circular Linked List คือ list ที่ node สุดท้ายชี้กลับไปยัง node แรก

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn CircularLinkedList(comptime T: type) type {
    return struct {
        const Self = @This();

        const Node = struct {
            data: T,
            next: *Node = undefined,
        };

        tail: ?*Node = null, // ใช้ tail เพื่อ O(1) append
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return Self{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            if (self.tail == null) return;
            var current = self.tail.?.next; // head
            const stop = self.tail.?.next;
            var i: usize = 0;
            while (i < self.size) : (i += 1) {
                const next = current.next;
                self.allocator.destroy(current);
                current = next;
            }
            _ = stop;
            self.tail = null;
        }

        pub fn append(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = Node{ .data = data };

            if (self.tail) |tail| {
                node.next = tail.next; // ชี้ไป head
                tail.next = node;
                self.tail = node;
            } else {
                node.next = node; // ชี้ตัวเอง
                self.tail = node;
            }
            self.size += 1;
        }

        pub fn prepend(self: *Self, data: T) !void {
            try self.append(data);
            // Move tail pointer back one (making new node the head)
            self.tail = self.tail.?.prev_conceptual() orelse self.tail;
            // Actually for circular singly linked, just append then rotate tail
        }

        pub fn removeHead(self: *Self) ?T {
            const tail = self.tail orelse return null;
            const head = tail.next;

            if (head == tail) {
                // Only one node
                const data = head.data;
                self.allocator.destroy(head);
                self.tail = null;
                self.size -= 1;
                return data;
            }

            tail.next = head.next;
            const data = head.data;
            self.allocator.destroy(head);
            self.size -= 1;
            return data;
        }

        pub fn print(self: *Self) void {
            const tail = self.tail orelse {
                std.debug.print("Circular List: empty\n", .{});
                return;
            };

            std.debug.print("Circular({d}): ", .{self.size});
            var current = tail.next; // start from head
            var i: usize = 0;
            while (i < self.size) : (i += 1) {
                std.debug.print("{d} -> ", .{current.data});
                current = current.next;
            }
            std.debug.print("(back to start)\n", .{});
        }
    };
}

// Josephus Problem using Circular Linked List
fn josephus(allocator: Allocator, n: u32, k: u32) !u32 {
    const Node = struct {
        id: u32,
        next: *@This() = undefined,
    };

    // สร้าง circular list
    var nodes = try allocator.alloc(Node, n);
    defer allocator.free(nodes);

    for (0..n) |i| {
        nodes[i] = .{ .id = @intCast(i + 1) };
        if (i > 0) nodes[i - 1].next = &nodes[i];
    }
    nodes[n - 1].next = &nodes[0]; // วงกลม

    var current = &nodes[0];
    var count = n;

    while (count > 1) {
        // นับไป k-1 คน
        var i: u32 = 0;
        while (i < k - 2) : (i += 1) {
            current = current.next;
        }
        // ลบ node ถัดไป
        std.debug.print("ออก: {d}\n", .{current.next.id});
        current.next = current.next.next;
        current = current.next;
        count -= 1;
    }

    return current.id;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Circular Linked List Demo ===\n", .{});

    var list = CircularLinkedList(i32).init(allocator);
    defer list.deinit();

    try list.append(1);
    try list.append(2);
    try list.append(3);
    try list.append(4);
    try list.append(5);
    list.print();

    _ = list.removeHead();
    list.print();

    // Josephus Problem
    std.debug.print("\n=== Josephus Problem (n=7, k=3) ===\n", .{});
    const winner = try josephus(allocator, 7, 3);
    std.debug.print("ผู้รอดสุดท้าย: {d}\n", .{winner});
}
```

---

## ขั้นตอนที่ 764: Skip List

Skip List คือ data structure ที่ให้ O(log n) average time สำหรับ search, insert, delete

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const MAX_LEVEL = 16;
const P = 0.5; // ความน่าจะเป็นที่จะ promote ขึ้น level

fn SkipList(comptime T: type) type {
    return struct {
        const Self = @This();

        const Node = struct {
            value: T,
            forward: [MAX_LEVEL]?*Node,
            level: usize,

            fn init(allocator: Allocator, value: T, level: usize) !*Node {
                const node = try allocator.create(Node);
                node.* = Node{
                    .value = value,
                    .forward = [_]?*Node{null} ** MAX_LEVEL,
                    .level = level,
                };
                return node;
            }
        };

        head: *Node,
        level: usize = 0,
        size: usize = 0,
        allocator: Allocator,
        rng: std.rand.DefaultPrng,

        pub fn init(allocator: Allocator) !Self {
            const head = try Node.init(allocator, undefined, MAX_LEVEL - 1);
            return Self{
                .head = head,
                .allocator = allocator,
                .rng = std.rand.DefaultPrng.init(@intCast(std.time.milliTimestamp())),
            };
        }

        pub fn deinit(self: *Self) void {
            var current: ?*Node = self.head;
            while (current) |node| {
                const next = node.forward[0];
                self.allocator.destroy(node);
                current = next;
            }
        }

        fn randomLevel(self: *Self) usize {
            var lvl: usize = 0;
            while (lvl < MAX_LEVEL - 1 and self.rng.random().float(f64) < P) {
                lvl += 1;
            }
            return lvl;
        }

        pub fn insert(self: *Self, value: T) !void {
            var update: [MAX_LEVEL]?*Node = [_]?*Node{null} ** MAX_LEVEL;
            var current = self.head;

            var i: i32 = @intCast(self.level);
            while (i >= 0) : (i -= 1) {
                const idx: usize = @intCast(i);
                while (current.forward[idx]) |next| {
                    if (next.value >= value) break;
                    current = next;
                }
                update[idx] = current;
            }

            const new_level = self.randomLevel();
            if (new_level > self.level) {
                var j = self.level + 1;
                while (j <= new_level) : (j += 1) {
                    update[j] = self.head;
                }
                self.level = new_level;
            }

            const new_node = try Node.init(self.allocator, value, new_level);
            var k: usize = 0;
            while (k <= new_level) : (k += 1) {
                new_node.forward[k] = update[k].?.forward[k];
                update[k].?.forward[k] = new_node;
            }
            self.size += 1;
        }

        pub fn search(self: *Self, value: T) bool {
            var current = self.head;
            var i: i32 = @intCast(self.level);
            while (i >= 0) : (i -= 1) {
                const idx: usize = @intCast(i);
                while (current.forward[idx]) |next| {
                    if (next.value == value) return true;
                    if (next.value > value) break;
                    current = next;
                }
            }
            return false;
        }

        pub fn delete(self: *Self, value: T) bool {
            var update: [MAX_LEVEL]?*Node = [_]?*Node{null} ** MAX_LEVEL;
            var current = self.head;

            var i: i32 = @intCast(self.level);
            while (i >= 0) : (i -= 1) {
                const idx: usize = @intCast(i);
                while (current.forward[idx]) |next| {
                    if (next.value >= value) break;
                    current = next;
                }
                update[idx] = current;
            }

            const target = current.forward[0] orelse return false;
            if (target.value != value) return false;

            var k: usize = 0;
            while (k <= self.level) : (k += 1) {
                if (update[k]) |upd| {
                    if (upd.forward[k] != target) break;
                    upd.forward[k] = target.forward[k];
                }
            }

            self.allocator.destroy(target);
            self.size -= 1;
            return true;
        }

        pub fn print(self: *Self) void {
            std.debug.print("Skip List (level={d}, size={d}):\n", .{ self.level, self.size });
            var lvl: i32 = @intCast(self.level);
            while (lvl >= 0) : (lvl -= 1) {
                const idx: usize = @intCast(lvl);
                std.debug.print("Level {d}: ", .{lvl});
                var current: ?*Node = self.head.forward[idx];
                while (current) |node| {
                    std.debug.print("{d} -> ", .{node.value});
                    current = node.forward[idx];
                }
                std.debug.print("null\n", .{});
            }
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Skip List Demo ===\n", .{});

    var sl = try SkipList(i32).init(allocator);
    defer sl.deinit();

    const values = [_]i32{ 3, 6, 7, 9, 12, 19, 17, 26, 21, 25 };
    for (values) |v| {
        try sl.insert(v);
    }

    sl.print();

    std.debug.print("\nSearch 19: {}\n", .{sl.search(19)});
    std.debug.print("Search 15: {}\n", .{sl.search(15)});

    _ = sl.delete(19);
    std.debug.print("\nหลังลบ 19:\n", .{});
    sl.print();
}
```

---

## ขั้นตอนที่ 765: Intrusive Linked List (std.SinglyLinkedList)

Zig มี std.SinglyLinkedList และ std.TailQueue ที่ใช้ intrusive design

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// std.SinglyLinkedList แบบ intrusive
const IntNode = std.SinglyLinkedList(i32).Node;
const IntList = std.SinglyLinkedList(i32);

// Custom intrusive list
fn IntrusiveList(comptime T: type, comptime field: []const u8) type {
    return struct {
        const Self = @This();
        const Node = T;

        head: ?*Node = null,
        size: usize = 0,

        fn getNext(node: *Node) *?*Node {
            return &@field(node, field);
        }

        pub fn prepend(self: *Self, node: *Node) void {
            getNext(node).* = self.head;
            self.head = node;
            self.size += 1;
        }

        pub fn removeFirst(self: *Self) ?*Node {
            const node = self.head orelse return null;
            self.head = getNext(node).*;
            self.size -= 1;
            return node;
        }

        pub fn print(self: *Self, comptime fmt: []const u8) void {
            var current = self.head;
            std.debug.print("IntrusiveList({d}): ", .{self.size});
            while (current) |node| {
                std.debug.print(fmt ++ " -> ", .{node});
                current = getNext(node).*;
            }
            std.debug.print("null\n", .{});
        }
    };
}

// TailQueue แบบ doubly intrusive
const Task = struct {
    id: u32,
    priority: u8,
    name: []const u8,
    node: std.TailQueue(void).Node = .{},
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Intrusive Linked List Demo ===\n", .{});

    // ใช้ std.SinglyLinkedList
    var std_list = IntList{};
    var node1 = IntNode{ .data = 10 };
    var node2 = IntNode{ .data = 20 };
    var node3 = IntNode{ .data = 30 };

    std_list.prepend(&node1);
    std_list.prepend(&node2);
    std_list.prepend(&node3);

    std.debug.print("std.SinglyLinkedList: ", .{});
    var it = std_list.first;
    while (it) |node| {
        std.debug.print("{d} -> ", .{node.data});
        it = node.next;
    }
    std.debug.print("null\n", .{});

    // TailQueue สำหรับ Task scheduling
    std.debug.print("\nTask Queue:\n", .{});
    var queue = std.TailQueue(void){};

    var tasks = [_]Task{
        .{ .id = 1, .priority = 3, .name = "Task A" },
        .{ .id = 2, .priority = 1, .name = "Task B" },
        .{ .id = 3, .priority = 2, .name = "Task C" },
    };

    for (&tasks) |*task| {
        queue.append(&task.node);
    }

    std.debug.print("Queue size: {d}\n", .{queue.len});
    _ = allocator;
}
```

---

## ขั้นตอนที่ 766: Generic Linked List

Generic Linked List ใน Zig ใช้ comptime generics

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Fully generic linked list with multiple operations
fn GenericList(comptime T: type) type {
    return struct {
        const Self = @This();

        const Node = struct {
            data: T,
            next: ?*Node = null,
        };

        head: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return .{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            var cur = self.head;
            while (cur) |n| {
                const nxt = n.next;
                self.allocator.destroy(n);
                cur = nxt;
            }
        }

        pub fn push(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .data = data, .next = self.head };
            self.head = node;
            self.size += 1;
        }

        pub fn pop(self: *Self) ?T {
            const node = self.head orelse return null;
            self.head = node.next;
            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        // Map: สร้าง list ใหม่โดย apply function
        pub fn map(self: *Self, comptime R: type, f: *const fn (T) R) !GenericList(R) {
            var result = GenericList(R).init(self.allocator);
            var nodes = try self.allocator.alloc(*Node, self.size);
            defer self.allocator.free(nodes);

            var cur = self.head;
            var i: usize = 0;
            while (cur) |n| {
                nodes[i] = n;
                cur = n.next;
                i += 1;
            }

            // Reverse order to maintain order
            var j: i32 = @intCast(i);
            j -= 1;
            while (j >= 0) : (j -= 1) {
                try result.push(f(nodes[@intCast(j)].data));
            }
            return result;
        }

        // Filter: กรองเฉพาะ elements ที่ตรงเงื่อนไข
        pub fn filter(self: *Self, predicate: *const fn (T) bool) !Self {
            var result = Self.init(self.allocator);
            var nodes = std.ArrayList(*Node).init(self.allocator);
            defer nodes.deinit();

            var cur = self.head;
            while (cur) |n| {
                if (predicate(n.data)) try nodes.append(n);
                cur = n.next;
            }

            var i: i32 = @intCast(nodes.items.len);
            i -= 1;
            while (i >= 0) : (i -= 1) {
                try result.push(nodes.items[@intCast(i)].data);
            }
            return result;
        }

        // Fold/Reduce
        pub fn fold(self: *Self, initial: T, f: *const fn (T, T) T) T {
            var acc = initial;
            var cur = self.head;
            while (cur) |n| {
                acc = f(acc, n.data);
                cur = n.next;
            }
            return acc;
        }

        pub fn toSlice(self: *Self) ![]T {
            var slice = try self.allocator.alloc(T, self.size);
            var cur = self.head;
            var i: usize = 0;
            while (cur) |n| {
                slice[i] = n.data;
                cur = n.next;
                i += 1;
            }
            return slice;
        }

        pub fn fromSlice(allocator: Allocator, items: []const T) !Self {
            var list = Self.init(allocator);
            var i: i32 = @intCast(items.len);
            i -= 1;
            while (i >= 0) : (i -= 1) {
                try list.push(items[@intCast(i)]);
            }
            return list;
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Generic Linked List Demo ===\n", .{});

    // สร้าง list จาก slice
    const numbers = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
    var list = try GenericList(i32).fromSlice(allocator, &numbers);
    defer list.deinit();

    // Filter เลขคู่
    var evens = try list.filter(struct {
        fn f(x: i32) bool { return @mod(x, 2) == 0; }
    }.f);
    defer evens.deinit();

    // Map กำลังสอง
    var squared = try evens.map(i32, struct {
        fn f(x: i32) i32 { return x * x; }
    }.f);
    defer squared.deinit();

    // Fold รวม
    const total = squared.fold(0, struct {
        fn f(a: i32, b: i32) i32 { return a + b; }
    }.f);

    const slice = try squared.toSlice();
    defer allocator.free(slice);

    std.debug.print("เลขคู่กำลังสอง: ", .{});
    for (slice) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\nผลรวม: {d}\n", .{total});
}
```

---

## ขั้นตอนที่ 767: Linked List Operations (Insert, Delete, Search)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn LinkedListOps(comptime T: type) type {
    return struct {
        const Self = @This();
        const Node = struct { data: T, next: ?*Node = null };

        head: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return .{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            var cur = self.head;
            while (cur) |n| { const nxt = n.next; self.allocator.destroy(n); cur = nxt; }
        }

        // Insert ที่ตำแหน่งที่กำหนด O(n)
        pub fn insertAt(self: *Self, index: usize, data: T) !void {
            if (index > self.size) return error.IndexOutOfBounds;

            const node = try self.allocator.create(Node);
            node.* = .{ .data = data };

            if (index == 0) {
                node.next = self.head;
                self.head = node;
            } else {
                var current = self.head.?;
                var i: usize = 0;
                while (i < index - 1) : (i += 1) {
                    current = current.next.?;
                }
                node.next = current.next;
                current.next = node;
            }
            self.size += 1;
        }

        // Insert แบบ sorted O(n)
        pub fn insertSorted(self: *Self, data: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .data = data };

            if (self.head == null or self.head.?.data >= data) {
                node.next = self.head;
                self.head = node;
                self.size += 1;
                return;
            }

            var current = self.head.?;
            while (current.next) |next| {
                if (next.data >= data) break;
                current = next;
            }
            node.next = current.next;
            current.next = node;
            self.size += 1;
        }

        // Delete ที่ตำแหน่งที่กำหนด O(n)
        pub fn deleteAt(self: *Self, index: usize) !T {
            if (index >= self.size) return error.IndexOutOfBounds;

            if (index == 0) {
                const node = self.head.?;
                self.head = node.next;
                const data = node.data;
                self.allocator.destroy(node);
                self.size -= 1;
                return data;
            }

            var current = self.head.?;
            var i: usize = 0;
            while (i < index - 1) : (i += 1) {
                current = current.next.?;
            }
            const node = current.next.?;
            current.next = node.next;
            const data = node.data;
            self.allocator.destroy(node);
            self.size -= 1;
            return data;
        }

        // Search แบบ linear O(n)
        pub fn searchLinear(self: *Self, data: T) ?usize {
            var current = self.head;
            var idx: usize = 0;
            while (current) |node| {
                if (node.data == data) return idx;
                current = node.next;
                idx += 1;
            }
            return null;
        }

        // Get element ที่ index O(n)
        pub fn get(self: *Self, index: usize) ?T {
            var current = self.head;
            var i: usize = 0;
            while (current) |node| {
                if (i == index) return node.data;
                current = node.next;
                i += 1;
            }
            return null;
        }

        // ลบ duplicates O(n^2) หรือ O(n) ถ้าใช้ hash set
        pub fn removeDuplicates(self: *Self) !void {
            var seen = std.AutoHashMap(T, void).init(self.allocator);
            defer seen.deinit();

            var prev: ?*Node = null;
            var current = self.head;
            while (current) |node| {
                if (seen.contains(node.data)) {
                    // ลบ node
                    const next = node.next;
                    if (prev) |p| {
                        p.next = next;
                    } else {
                        self.head = next;
                    }
                    self.allocator.destroy(node);
                    self.size -= 1;
                    current = next;
                } else {
                    try seen.put(node.data, {});
                    prev = node;
                    current = node.next;
                }
            }
        }

        pub fn print(self: *Self) void {
            var cur = self.head;
            while (cur) |n| { std.debug.print("{d} -> ", .{n.data}); cur = n.next; }
            std.debug.print("null\n", .{});
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Linked List Operations Demo ===\n", .{});

    var list = LinkedListOps(i32).init(allocator);
    defer list.deinit();

    // Insert sorted
    const values = [_]i32{ 5, 3, 7, 1, 9, 4, 6, 2, 8 };
    for (values) |v| try list.insertSorted(v);
    std.debug.print("Sorted insert: ");
    list.print();

    // Insert ที่ตำแหน่ง
    try list.insertAt(3, 100);
    std.debug.print("After insert 100 at index 3: ");
    list.print();

    // Delete
    const deleted = try list.deleteAt(3);
    std.debug.print("Deleted: {d}\n", .{deleted});
    list.print();

    // Search
    if (list.searchLinear(7)) |idx| {
        std.debug.print("พบ 7 ที่ index {d}\n", .{idx});
    }

    // Remove duplicates
    var dup_list = LinkedListOps(i32).init(allocator);
    defer dup_list.deinit();
    const dups = [_]i32{ 1, 2, 3, 2, 1, 4, 3, 5 };
    for (dups) |v| try dup_list.insertAt(dup_list.size, v);
    std.debug.print("\nBefore dedup: ");
    dup_list.print();
    try dup_list.removeDuplicates();
    std.debug.print("After dedup: ");
    dup_list.print();
}
```

---

## ขั้นตอนที่ 768: Linked List ขั้นสูง (Merge Sort)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const Node = struct {
    data: i32,
    next: ?*Node = null,
};

fn splitList(head: *Node) struct { left: *Node, right: ?*Node } {
    var slow: *Node = head;
    var fast: ?*Node = head.next;

    while (fast != null) {
        fast = fast.?.next;
        if (fast != null) {
            slow = slow.next.?;
            fast = fast.?.next;
        }
    }

    const right = slow.next;
    slow.next = null;
    return .{ .left = head, .right = right };
}

fn merge(left: ?*Node, right: ?*Node) ?*Node {
    if (left == null) return right;
    if (right == null) return left;

    if (left.?.data <= right.?.data) {
        left.?.next = merge(left.?.next, right);
        return left;
    } else {
        right.?.next = merge(left, right.?.next);
        return right;
    }
}

fn mergeSort(head: ?*Node) ?*Node {
    if (head == null or head.?.next == null) return head;

    const parts = splitList(head.?);
    const sorted_left = mergeSort(parts.left);
    const sorted_right = mergeSort(parts.right);
    return merge(sorted_left, sorted_right);
}

fn printList(head: ?*Node) void {
    var cur = head;
    while (cur) |n| { std.debug.print("{d} -> ", .{n.data}); cur = n.next; }
    std.debug.print("null\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Merge Sort on Linked List ===\n", .{});

    // สร้าง list ที่ไม่เรียงลำดับ
    const values = [_]i32{ 64, 34, 25, 12, 22, 11, 90 };
    var nodes = try allocator.alloc(Node, values.len);
    defer allocator.free(nodes);

    for (values, 0..) |v, i| {
        nodes[i] = .{ .data = v };
        if (i > 0) nodes[i - 1].next = &nodes[i];
    }

    std.debug.print("ก่อน sort: ");
    printList(&nodes[0]);

    const sorted = mergeSort(&nodes[0]);
    std.debug.print("หลัง sort: ");
    printList(sorted);

    // Merge two sorted lists
    std.debug.print("\nMerge two sorted lists:\n", .{});
    var list1_nodes = [_]Node{
        .{ .data = 1 }, .{ .data = 3 }, .{ .data = 5 },
    };
    var list2_nodes = [_]Node{
        .{ .data = 2 }, .{ .data = 4 }, .{ .data = 6 },
    };
    list1_nodes[0].next = &list1_nodes[1];
    list1_nodes[1].next = &list1_nodes[2];
    list2_nodes[0].next = &list2_nodes[1];
    list2_nodes[1].next = &list2_nodes[2];

    const merged = merge(&list1_nodes[0], &list2_nodes[0]);
    printList(merged);
}
```

---

## ขั้นตอนที่ 769: XOR Linked List

XOR Linked List ประหยัด memory โดยเก็บ XOR ของ addresses แทนที่จะเก็บสอง pointers

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// XOR Linked List - ใช้ XOR ของ addresses
// แต่ละ node เก็บ prev XOR next แทน
const XORNode = struct {
    data: i32,
    both: usize = 0, // prev XOR next

    fn xorPtr(a: ?*XORNode, b: ?*XORNode) usize {
        const a_int: usize = if (a) |p| @intFromPtr(p) else 0;
        const b_int: usize = if (b) |p| @intFromPtr(p) else 0;
        return a_int ^ b_int;
    }

    fn toPtr(val: usize) ?*XORNode {
        if (val == 0) return null;
        return @ptrFromInt(val);
    }
};

const XORLinkedList = struct {
    head: ?*XORNode = null,
    tail: ?*XORNode = null,
    size: usize = 0,
    allocator: Allocator,

    pub fn init(allocator: Allocator) XORLinkedList {
        return .{ .allocator = allocator };
    }

    pub fn deinit(self: *XORLinkedList) void {
        var prev: ?*XORNode = null;
        var current = self.head;
        while (current) |node| {
            const next = XORNode.toPtr(node.both ^ (if (prev) |p| @intFromPtr(p) else 0));
            self.allocator.destroy(node);
            prev = node;
            current = next;
        }
    }

    pub fn append(self: *XORLinkedList, data: i32) !void {
        const node = try self.allocator.create(XORNode);
        node.* = .{
            .data = data,
            .both = XORNode.xorPtr(self.tail, null),
        };

        if (self.tail) |tail| {
            tail.both ^= @intFromPtr(node);
        } else {
            self.head = node;
        }
        self.tail = node;
        self.size += 1;
    }

    pub fn printForward(self: *XORLinkedList) void {
        std.debug.print("XOR List (forward): ", .{});
        var prev: usize = 0;
        var current = self.head;
        while (current) |node| {
            std.debug.print("{d} -> ", .{node.data});
            const next_addr = node.both ^ prev;
            prev = @intFromPtr(node);
            current = XORNode.toPtr(next_addr);
        }
        std.debug.print("null\n", .{});
    }

    pub fn printBackward(self: *XORLinkedList) void {
        std.debug.print("XOR List (backward): ", .{});
        var next: usize = 0;
        var current = self.tail;
        while (current) |node| {
            std.debug.print("{d} -> ", .{node.data});
            const prev_addr = node.both ^ next;
            next = @intFromPtr(node);
            current = XORNode.toPtr(prev_addr);
        }
        std.debug.print("null\n", .{});
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== XOR Linked List Demo ===\n", .{});
    std.debug.print("(ประหยัด memory โดยใช้ XOR แทน prev+next pointers)\n\n", .{});

    var list = XORLinkedList.init(allocator);
    defer list.deinit();

    try list.append(10);
    try list.append(20);
    try list.append(30);
    try list.append(40);
    try list.append(50);

    list.printForward();
    list.printBackward();

    std.debug.print("\nขนาด node: {d} bytes (เท่ากับ singly linked list!)\n", .{
        @sizeOf(XORNode),
    });
    std.debug.print("ขนาด doubly linked node: {d} bytes\n", .{
        @sizeOf(struct { data: i32, prev: ?*anyopaque, next: ?*anyopaque }),
    });
}
```

---

## ขั้นตอนที่ 770: Real-world Linked List Usage

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Memory Pool ใช้ Linked List
const MemoryPool = struct {
    const Block = struct {
        next: ?*Block = null,
        // data follows
    };

    allocator: Allocator,
    free_list: ?*Block = null,
    block_size: usize,
    allocated: usize = 0,
    freed: usize = 0,

    pub fn init(allocator: Allocator, block_size: usize, initial_count: usize) !MemoryPool {
        var pool = MemoryPool{
            .allocator = allocator,
            .block_size = block_size,
        };
        // Pre-allocate blocks
        var i: usize = 0;
        while (i < initial_count) : (i += 1) {
            const block = try allocator.create(Block);
            block.next = pool.free_list;
            pool.free_list = block;
        }
        return pool;
    }

    pub fn deinit(self: *MemoryPool) void {
        var current = self.free_list;
        while (current) |block| {
            const next = block.next;
            self.allocator.destroy(block);
            current = next;
        }
    }

    pub fn alloc(self: *MemoryPool) ?*Block {
        const block = self.free_list orelse return null;
        self.free_list = block.next;
        block.next = null;
        self.allocated += 1;
        return block;
    }

    pub fn free(self: *MemoryPool, block: *Block) void {
        block.next = self.free_list;
        self.free_list = block;
        self.freed += 1;
    }

    pub fn printStats(self: *MemoryPool) void {
        std.debug.print("Pool: allocated={d}, freed={d}, available=", .{
            self.allocated,
            self.freed,
        });
        var count: usize = 0;
        var cur = self.free_list;
        while (cur) |b| { count += 1; cur = b.next; }
        std.debug.print("{d}\n", .{count});
    }
};

// Job Queue ใช้ Linked List
const Priority = enum(u8) { low = 0, normal = 1, high = 2, critical = 3 };

const Job = struct {
    id: u32,
    name: []const u8,
    priority: Priority,
    next: ?*Job = null,
};

const JobQueue = struct {
    allocator: Allocator,
    heads: [4]?*Job = [_]?*Job{null} ** 4, // head สำหรับแต่ละ priority
    tails: [4]?*Job = [_]?*Job{null} ** 4,
    total: usize = 0,

    pub fn init(allocator: Allocator) JobQueue {
        return .{ .allocator = allocator };
    }

    pub fn deinit(self: *JobQueue) void {
        for (0..4) |p| {
            var cur = self.heads[p];
            while (cur) |job| {
                const next = job.next;
                self.allocator.destroy(job);
                cur = next;
            }
        }
    }

    pub fn enqueue(self: *JobQueue, id: u32, name: []const u8, priority: Priority) !void {
        const job = try self.allocator.create(Job);
        job.* = .{ .id = id, .name = name, .priority = priority };

        const p = @intFromEnum(priority);
        if (self.tails[p]) |tail| {
            tail.next = job;
        } else {
            self.heads[p] = job;
        }
        self.tails[p] = job;
        self.total += 1;
    }

    // Dequeue ตาม priority สูงสุดก่อน
    pub fn dequeue(self: *JobQueue) ?*Job {
        var p: i32 = 3; // เริ่มจาก critical
        while (p >= 0) : (p -= 1) {
            const idx: usize = @intCast(p);
            if (self.heads[idx]) |job| {
                self.heads[idx] = job.next;
                if (self.heads[idx] == null) self.tails[idx] = null;
                job.next = null;
                self.total -= 1;
                return job;
            }
        }
        return null;
    }

    pub fn printQueue(self: *JobQueue) void {
        std.debug.print("Job Queue (total={d}):\n", .{self.total});
        const priority_names = [_][]const u8{ "LOW", "NORMAL", "HIGH", "CRITICAL" };
        var p: i32 = 3;
        while (p >= 0) : (p -= 1) {
            const idx: usize = @intCast(p);
            var cur = self.heads[idx];
            if (cur == null) continue;
            std.debug.print("  [{s}]: ", .{priority_names[idx]});
            while (cur) |job| {
                std.debug.print("#{d}({s}) ", .{ job.id, job.name });
                cur = job.next;
            }
            std.debug.print("\n", .{});
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Real-world Linked List Usage ===\n", .{});

    // Memory Pool
    std.debug.print("\n--- Memory Pool ---\n", .{});
    var pool = try MemoryPool.init(allocator, 64, 5);
    defer pool.deinit();

    const b1 = pool.alloc();
    const b2 = pool.alloc();
    const b3 = pool.alloc();
    pool.printStats();

    if (b1) |block| pool.free(block);
    if (b2) |block| pool.free(block);
    pool.printStats();
    _ = b3;

    // Priority Job Queue
    std.debug.print("\n--- Priority Job Queue ---\n", .{});
    var queue = JobQueue.init(allocator);
    defer queue.deinit();

    try queue.enqueue(1, "Cleanup", .low);
    try queue.enqueue(2, "Email", .normal);
    try queue.enqueue(3, "Payment", .high);
    try queue.enqueue(4, "Alert", .critical);
    try queue.enqueue(5, "Report", .normal);
    try queue.enqueue(6, "Backup", .low);

    queue.printQueue();

    std.debug.print("\nประมวลผลตาม priority:\n", .{});
    while (queue.dequeue()) |job| {
        std.debug.print("  ประมวลผล: #{d} {s} [{s}]\n", .{
            job.id,
            job.name,
            @tagName(job.priority),
        });
        allocator.destroy(job);
    }
}
```

---

## สรุปส่วนที่ 77: Linked Lists

| โครงสร้าง | Insert (head) | Insert (tail) | Search | Delete | Memory |
|----------|--------------|--------------|--------|--------|--------|
| Singly LL | O(1) | O(n) | O(n) | O(n) | น้อย |
| Doubly LL | O(1) | O(1) | O(n) | O(1)* | ปานกลาง |
| Circular LL | O(1) | O(1) | O(n) | O(n) | น้อย |
| Skip List | O(log n) avg | O(log n) avg | O(log n) avg | O(log n) avg | มาก |
| XOR LL | O(1) | O(1) | O(n) | O(1)* | น้อยมาก |

*กรณีที่รู้ pointer ของ node

### เคล็ดลับ Zig

1. ใช้ `?*Node` สำหรับ nullable pointers
2. `allocator.destroy(node)` สำคัญมาก - ป้องกัน memory leak
3. Intrusive list จาก `std.SinglyLinkedList` และ `std.TailQueue` ประหยัด allocation
4. Generic list ใช้ `comptime T: type` ทำให้ reuse ได้
5. XOR list ประหยัด memory แต่ debug ยากกว่า

---

*ส่วนถัดไป: Part 78 - Data Structures: Binary Tree*
