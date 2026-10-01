# ส่วนที่ 78: Data Structures: Binary Tree (ขั้นตอนที่ 771-780)

Binary Tree เป็นโครงสร้างข้อมูลที่แต่ละ node มีลูกได้ไม่เกิน 2 node ใช้งานกว้างขวางตั้งแต่ database indexes จนถึง expression parsing

---

## ขั้นตอนที่ 771: Binary Search Tree (BST)

BST คือ Binary Tree ที่ node ซ้ายมีค่าน้อยกว่า node แม่ และ node ขวามีค่ามากกว่า

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn BST(comptime T: type) type {
    return struct {
        const Self = @This();

        pub const Node = struct {
            key: T,
            left: ?*Node = null,
            right: ?*Node = null,
            parent: ?*Node = null,

            pub fn isLeaf(self: *Node) bool {
                return self.left == null and self.right == null;
            }

            pub fn childCount(self: *Node) u2 {
                var count: u2 = 0;
                if (self.left != null) count += 1;
                if (self.right != null) count += 1;
                return count;
            }
        };

        root: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return .{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            self.destroyNode(self.root);
        }

        fn destroyNode(self: *Self, node: ?*Node) void {
            if (node) |n| {
                self.destroyNode(n.left);
                self.destroyNode(n.right);
                self.allocator.destroy(n);
            }
        }

        pub fn insert(self: *Self, key: T) !void {
            const node = try self.allocator.create(Node);
            node.* = .{ .key = key };

            if (self.root == null) {
                self.root = node;
                self.size += 1;
                return;
            }

            var current = self.root.?;
            while (true) {
                if (key < current.key) {
                    if (current.left) |left| {
                        current = left;
                    } else {
                        current.left = node;
                        node.parent = current;
                        break;
                    }
                } else if (key > current.key) {
                    if (current.right) |right| {
                        current = right;
                    } else {
                        current.right = node;
                        node.parent = current;
                        break;
                    }
                } else {
                    // Duplicate - ไม่เพิ่ม
                    self.allocator.destroy(node);
                    return;
                }
            }
            self.size += 1;
        }

        pub fn search(self: *Self, key: T) ?*Node {
            var current = self.root;
            while (current) |node| {
                if (key == node.key) return node;
                current = if (key < node.key) node.left else node.right;
            }
            return null;
        }

        pub fn minimum(self: *Self) ?*Node {
            return self.minimumNode(self.root);
        }

        fn minimumNode(_: *Self, node: ?*Node) ?*Node {
            var current = node orelse return null;
            while (current.left) |left| {
                current = left;
            }
            return current;
        }

        pub fn maximum(self: *Self) ?*Node {
            var current = self.root orelse return null;
            while (current.right) |right| {
                current = right;
            }
            return current;
        }

        // Successor - node ที่มีค่าถัดไปที่ใหญ่กว่า
        pub fn successor(self: *Self, node: *Node) ?*Node {
            if (node.right) |right| {
                return self.minimumNode(right);
            }
            var current: ?*Node = node;
            var parent = node.parent;
            while (parent != null and current == parent.?.right) {
                current = parent;
                parent = parent.?.parent;
            }
            return parent;
        }

        // Height ของ tree
        pub fn height(self: *Self) usize {
            return self.nodeHeight(self.root);
        }

        fn nodeHeight(_: *Self, node: ?*Node) usize {
            const n = node orelse return 0;
            const left_h = BST(T).nodeHeight(undefined, n.left);
            const right_h = BST(T).nodeHeight(undefined, n.right);
            return 1 + @max(left_h, right_h);
        }

        // ตรวจสอบว่า BST valid หรือไม่
        pub fn isValid(self: *Self) bool {
            return self.isValidNode(self.root, null, null);
        }

        fn isValidNode(_: *Self, node: ?*Node, min_val: ?T, max_val: ?T) bool {
            const n = node orelse return true;
            if (min_val) |min| {
                if (n.key <= min) return false;
            }
            if (max_val) |max| {
                if (n.key >= max) return false;
            }
            return BST(T).isValidNode(undefined, n.left, min_val, n.key) and
                   BST(T).isValidNode(undefined, n.right, n.key, max_val);
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Binary Search Tree Demo ===\n", .{});

    var tree = BST(i32).init(allocator);
    defer tree.deinit();

    const values = [_]i32{ 50, 30, 70, 20, 40, 60, 80, 10, 25, 35, 45 };
    for (values) |v| try tree.insert(v);

    std.debug.print("Size: {d}\n", .{tree.size});
    std.debug.print("Height: {d}\n", .{tree.height()});
    std.debug.print("Valid BST: {}\n", .{tree.isValid()});

    if (tree.minimum()) |min| std.debug.print("Min: {d}\n", .{min.key});
    if (tree.maximum()) |max| std.debug.print("Max: {d}\n", .{max.key});

    if (tree.search(40)) |node| {
        std.debug.print("Found 40\n", .{});
        if (tree.successor(node)) |succ| {
            std.debug.print("Successor of 40: {d}\n", .{succ.key});
        }
    }
}
```

---

## ขั้นตอนที่ 772: BST Insert, Search, Delete

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn BSTFull(comptime T: type) type {
    return struct {
        const Self = @This();
        const Node = struct {
            key: T,
            left: ?*Node = null,
            right: ?*Node = null,
        };

        root: ?*Node = null,
        size: usize = 0,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return .{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            self.freeNode(self.root);
        }

        fn freeNode(self: *Self, node: ?*Node) void {
            if (node) |n| {
                self.freeNode(n.left);
                self.freeNode(n.right);
                self.allocator.destroy(n);
            }
        }

        // Insert iterative
        pub fn insert(self: *Self, key: T) !bool {
            const node = try self.allocator.create(Node);
            node.* = .{ .key = key };

            if (self.root == null) {
                self.root = node;
                self.size += 1;
                return true;
            }

            var current = self.root.?;
            while (true) {
                if (key < current.key) {
                    if (current.left) |left| {
                        current = left;
                    } else {
                        current.left = node;
                        self.size += 1;
                        return true;
                    }
                } else if (key > current.key) {
                    if (current.right) |right| {
                        current = right;
                    } else {
                        current.right = node;
                        self.size += 1;
                        return true;
                    }
                } else {
                    self.allocator.destroy(node);
                    return false; // duplicate
                }
            }
        }

        // Search iterative
        pub fn search(self: *Self, key: T) bool {
            var current = self.root;
            while (current) |node| {
                if (key == node.key) return true;
                current = if (key < node.key) node.left else node.right;
            }
            return false;
        }

        // Delete - มี 3 กรณี
        pub fn delete(self: *Self, key: T) bool {
            self.root = self.deleteNode(self.root, key);
            return true; // simplified
        }

        fn deleteNode(self: *Self, node: ?*Node, key: T) ?*Node {
            const n = node orelse return null;

            if (key < n.key) {
                n.left = self.deleteNode(n.left, key);
            } else if (key > n.key) {
                n.right = self.deleteNode(n.right, key);
            } else {
                // พบ node ที่จะลบ
                self.size -= 1;

                // Case 1: leaf node
                if (n.left == null and n.right == null) {
                    self.allocator.destroy(n);
                    return null;
                }

                // Case 2: มีลูกข้างเดียว
                if (n.left == null) {
                    const right = n.right;
                    self.allocator.destroy(n);
                    return right;
                }
                if (n.right == null) {
                    const left = n.left;
                    self.allocator.destroy(n);
                    return left;
                }

                // Case 3: มีลูกสองข้าง - หา in-order successor
                var successor = n.right.?;
                while (successor.left) |left| {
                    successor = left;
                }
                n.key = successor.key;
                n.right = self.deleteNode(n.right, successor.key);
                self.size += 1; // compensate
            }
            return n;
        }

        // In-order traversal (sorted)
        pub fn inorder(self: *Self, result: *std.ArrayList(T)) !void {
            try self.inorderNode(self.root, result);
        }

        fn inorderNode(_: *Self, node: ?*Node, result: *std.ArrayList(T)) !void {
            const n = node orelse return;
            try BSTFull(T).inorderNode(undefined, n.left, result);
            try result.append(n.key);
            try BSTFull(T).inorderNode(undefined, n.right, result);
        }

        pub fn printTree(self: *Self) void {
            std.debug.print("BST ({d} nodes):\n", .{self.size});
            self.printNode(self.root, "", true);
        }

        fn printNode(_: *Self, node: ?*Node, prefix: []const u8, is_right: bool) void {
            const n = node orelse return;
            std.debug.print("{s}{s}{d}\n", .{
                prefix,
                if (is_right) "└── " else "├── ",
                n.key,
            });
            const new_prefix = if (is_right) "    " else "│   ";
            var buf: [256]u8 = undefined;
            const full_prefix = std.fmt.bufPrint(&buf, "{s}{s}", .{ prefix, new_prefix }) catch return;
            BSTFull(T).printNode(undefined, n.left, full_prefix, false);
            BSTFull(T).printNode(undefined, n.right, full_prefix, true);
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== BST Insert, Search, Delete ===\n", .{});

    var bst = BSTFull(i32).init(allocator);
    defer bst.deinit();

    const values = [_]i32{ 5, 3, 7, 1, 4, 6, 8 };
    for (values) |v| _ = try bst.insert(v);

    bst.printTree();

    std.debug.print("\nSearch 4: {}\n", .{bst.search(4)});
    std.debug.print("Search 9: {}\n", .{bst.search(9)});

    // Delete cases
    std.debug.print("\nDelete 1 (leaf):\n", .{});
    _ = bst.delete(1);
    bst.printTree();

    std.debug.print("\nDelete 3 (one child):\n", .{});
    _ = bst.delete(3);
    bst.printTree();

    std.debug.print("\nDelete 7 (two children):\n", .{});
    _ = bst.delete(7);
    bst.printTree();

    // In-order
    var sorted = std.ArrayList(i32).init(allocator);
    defer sorted.deinit();
    try bst.inorder(&sorted);
    std.debug.print("\nIn-order: ", .{});
    for (sorted.items) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
}
```

---

## ขั้นตอนที่ 773: BST Traversal (In-order, Pre-order, Post-order)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const TreeNode = struct {
    key: i32,
    left: ?*TreeNode = null,
    right: ?*TreeNode = null,
};

fn buildTree(allocator: Allocator) !*TreeNode {
    var nodes = try allocator.alloc(TreeNode, 7);
    nodes[0] = .{ .key = 4 };
    nodes[1] = .{ .key = 2 };
    nodes[2] = .{ .key = 6 };
    nodes[3] = .{ .key = 1 };
    nodes[4] = .{ .key = 3 };
    nodes[5] = .{ .key = 5 };
    nodes[6] = .{ .key = 7 };
    nodes[0].left = &nodes[1];
    nodes[0].right = &nodes[2];
    nodes[1].left = &nodes[3];
    nodes[1].right = &nodes[4];
    nodes[2].left = &nodes[5];
    nodes[2].right = &nodes[6];
    return &nodes[0];
}

// Recursive traversals
fn inorder(node: ?*TreeNode) void {
    const n = node orelse return;
    inorder(n.left);
    std.debug.print("{d} ", .{n.key});
    inorder(n.right);
}

fn preorder(node: ?*TreeNode) void {
    const n = node orelse return;
    std.debug.print("{d} ", .{n.key});
    preorder(n.left);
    preorder(n.right);
}

fn postorder(node: ?*TreeNode) void {
    const n = node orelse return;
    postorder(n.left);
    postorder(n.right);
    std.debug.print("{d} ", .{n.key});
}

// Level-order (BFS) traversal
fn levelOrder(allocator: Allocator, root: ?*TreeNode) !void {
    if (root == null) return;
    var queue = std.ArrayList(*TreeNode).init(allocator);
    defer queue.deinit();

    try queue.append(root.?);
    var level: usize = 0;

    while (queue.items.len > 0) {
        const level_size = queue.items.len;
        std.debug.print("Level {d}: ", .{level});

        var i: usize = 0;
        while (i < level_size) : (i += 1) {
            const node = queue.orderedRemove(0);
            std.debug.print("{d} ", .{node.key});
            if (node.left) |left| try queue.append(left);
            if (node.right) |right| try queue.append(right);
        }
        std.debug.print("\n", .{});
        level += 1;
    }
}

// Iterative in-order ใช้ stack
fn iterativeInorder(allocator: Allocator, root: ?*TreeNode) !void {
    var stack = std.ArrayList(*TreeNode).init(allocator);
    defer stack.deinit();

    var current = root;
    std.debug.print("Iterative In-order: ", .{});
    while (current != null or stack.items.len > 0) {
        while (current) |node| {
            try stack.append(node);
            current = node.left;
        }
        current = stack.pop();
        std.debug.print("{d} ", .{current.?.key});
        current = current.?.right;
    }
    std.debug.print("\n", .{});
}

// Morris traversal - O(1) space
fn morrisInorder(root: ?*TreeNode) void {
    var current = root;
    std.debug.print("Morris In-order: ", .{});
    while (current) |cur| {
        if (cur.left == null) {
            std.debug.print("{d} ", .{cur.key});
            current = cur.right;
        } else {
            // หา in-order predecessor
            var pred = cur.left.?;
            while (pred.right != null and pred.right != cur) {
                pred = pred.right.?;
            }
            if (pred.right == null) {
                pred.right = cur;
                current = cur.left;
            } else {
                pred.right = null;
                std.debug.print("{d} ", .{cur.key});
                current = cur.right;
            }
        }
    }
    std.debug.print("\n", .{});
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== BST Traversal Demo ===\n", .{});

    var nodes = try allocator.alloc(TreeNode, 7);
    defer allocator.free(nodes);
    const root = try buildTree(allocator);
    defer allocator.free(@as([*]TreeNode, @ptrCast(root))[0..7]);
    _ = nodes;

    // ใช้ root จาก buildTree
    var root2_nodes = try allocator.alloc(TreeNode, 7);
    defer allocator.free(root2_nodes);
    root2_nodes[0] = .{ .key = 4 };
    root2_nodes[1] = .{ .key = 2 };
    root2_nodes[2] = .{ .key = 6 };
    root2_nodes[3] = .{ .key = 1 };
    root2_nodes[4] = .{ .key = 3 };
    root2_nodes[5] = .{ .key = 5 };
    root2_nodes[6] = .{ .key = 7 };
    root2_nodes[0].left = &root2_nodes[1];
    root2_nodes[0].right = &root2_nodes[2];
    root2_nodes[1].left = &root2_nodes[3];
    root2_nodes[1].right = &root2_nodes[4];
    root2_nodes[2].left = &root2_nodes[5];
    root2_nodes[2].right = &root2_nodes[6];
    const r = &root2_nodes[0];

    std.debug.print("In-order (LNR): ");
    inorder(r);
    std.debug.print("\n", .{});

    std.debug.print("Pre-order (NLR): ");
    preorder(r);
    std.debug.print("\n", .{});

    std.debug.print("Post-order (LRN): ");
    postorder(r);
    std.debug.print("\n", .{});

    std.debug.print("\nLevel-order (BFS):\n", .{});
    try levelOrder(allocator, r);

    try iterativeInorder(allocator, r);
    morrisInorder(r);
    _ = root;
}
```

---

## ขั้นตอนที่ 774: AVL Tree (Self-balancing)

AVL Tree เป็น BST ที่ balance factor ของแต่ละ node อยู่ระหว่าง -1 ถึง 1

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const AVLNode = struct {
    key: i32,
    height: i32 = 1,
    left: ?*AVLNode = null,
    right: ?*AVLNode = null,
};

const AVLTree = struct {
    root: ?*AVLNode = null,
    size: usize = 0,
    allocator: Allocator,
    rotations: u32 = 0,

    pub fn init(allocator: Allocator) AVLTree {
        return .{ .allocator = allocator };
    }

    pub fn deinit(self: *AVLTree) void {
        self.freeNode(self.root);
    }

    fn freeNode(self: *AVLTree, node: ?*AVLNode) void {
        if (node) |n| {
            self.freeNode(n.left);
            self.freeNode(n.right);
            self.allocator.destroy(n);
        }
    }

    fn height(node: ?*AVLNode) i32 {
        return if (node) |n| n.height else 0;
    }

    fn balanceFactor(node: *AVLNode) i32 {
        return height(node.left) - height(node.right);
    }

    fn updateHeight(node: *AVLNode) void {
        node.height = 1 + @max(height(node.left), height(node.right));
    }

    // Right rotation
    fn rotateRight(self: *AVLTree, y: *AVLNode) *AVLNode {
        const x = y.left.?;
        const t2 = x.right;

        x.right = y;
        y.left = t2;

        updateHeight(y);
        updateHeight(x);
        self.rotations += 1;
        return x;
    }

    // Left rotation
    fn rotateLeft(self: *AVLTree, x: *AVLNode) *AVLNode {
        const y = x.right.?;
        const t2 = y.left;

        y.left = x;
        x.right = t2;

        updateHeight(x);
        updateHeight(y);
        self.rotations += 1;
        return y;
    }

    fn balance(self: *AVLTree, node: *AVLNode) *AVLNode {
        updateHeight(node);
        const bf = balanceFactor(node);

        // Left heavy
        if (bf > 1) {
            if (balanceFactor(node.left.?) < 0) {
                node.left = self.rotateLeft(node.left.?);
            }
            return self.rotateRight(node);
        }

        // Right heavy
        if (bf < -1) {
            if (balanceFactor(node.right.?) > 0) {
                node.right = self.rotateRight(node.right.?);
            }
            return self.rotateLeft(node);
        }

        return node;
    }

    pub fn insert(self: *AVLTree, key: i32) !void {
        self.root = try self.insertNode(self.root, key);
    }

    fn insertNode(self: *AVLTree, node: ?*AVLNode, key: i32) !*AVLNode {
        if (node == null) {
            const n = try self.allocator.create(AVLNode);
            n.* = .{ .key = key };
            self.size += 1;
            return n;
        }

        const n = node.?;
        if (key < n.key) {
            n.left = try self.insertNode(n.left, key);
        } else if (key > n.key) {
            n.right = try self.insertNode(n.right, key);
        } else {
            return n; // duplicate
        }

        return self.balance(n);
    }

    pub fn delete(self: *AVLTree, key: i32) void {
        self.root = self.deleteNode(self.root, key);
    }

    fn minNode(node: *AVLNode) *AVLNode {
        var cur = node;
        while (cur.left) |left| cur = left;
        return cur;
    }

    fn deleteNode(self: *AVLTree, node: ?*AVLNode, key: i32) ?*AVLNode {
        const n = node orelse return null;

        if (key < n.key) {
            n.left = self.deleteNode(n.left, key);
        } else if (key > n.key) {
            n.right = self.deleteNode(n.right, key);
        } else {
            self.size -= 1;
            if (n.left == null or n.right == null) {
                const child = n.left orelse n.right;
                self.allocator.destroy(n);
                return child;
            }
            const successor = minNode(n.right.?);
            n.key = successor.key;
            self.size += 1; // compensate
            n.right = self.deleteNode(n.right, successor.key);
        }

        return self.balance(n);
    }

    pub fn printStats(self: *AVLTree) void {
        std.debug.print("AVL Tree: size={d}, height={d}, rotations={d}\n", .{
            self.size,
            height(self.root),
            self.rotations,
        });
    }

    pub fn inorder(self: *AVLTree) void {
        std.debug.print("In-order: ", .{});
        self.inorderNode(self.root);
        std.debug.print("\n", .{});
    }

    fn inorderNode(_: *AVLTree, node: ?*AVLNode) void {
        const n = node orelse return;
        AVLTree.inorderNode(undefined, n.left);
        std.debug.print("{d}({d}) ", .{ n.key, balanceFactor(n) });
        AVLTree.inorderNode(undefined, n.right);
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== AVL Tree Demo ===\n", .{});

    var avl = AVLTree.init(allocator);
    defer avl.deinit();

    // Insert in sorted order (worst case for regular BST)
    var i: i32 = 1;
    while (i <= 15) : (i += 1) {
        try avl.insert(i);
    }

    avl.printStats();
    avl.inorder();

    // ลบบาง nodes
    avl.delete(5);
    avl.delete(10);
    avl.printStats();
}
```

---

## ขั้นตอนที่ 775: Red-Black Tree Concept

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Red-Black Tree Properties:
// 1. ทุก node มีสีแดงหรือดำ
// 2. Root เป็นสีดำ
// 3. ทุก leaf (null) เป็นสีดำ
// 4. ถ้า node สีแดง ลูกต้องเป็นสีดำทั้งคู่
// 5. ทุก path จาก node ไป leaf มีจำนวน black nodes เท่ากัน

const Color = enum { red, black };

const RBNode = struct {
    key: i32,
    color: Color = .red,
    left: ?*RBNode = null,
    right: ?*RBNode = null,
    parent: ?*RBNode = null,
};

const RBTree = struct {
    root: ?*RBNode = null,
    nil: RBNode, // sentinel node
    size: usize = 0,
    allocator: Allocator,

    pub fn init(allocator: Allocator) RBTree {
        return RBTree{
            .nil = RBNode{ .key = 0, .color = .black },
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *RBTree) void {
        self.freeNode(self.root);
    }

    fn freeNode(self: *RBTree, node: ?*RBNode) void {
        if (node) |n| {
            if (n == &self.nil) return;
            self.freeNode(n.left);
            self.freeNode(n.right);
            self.allocator.destroy(n);
        }
    }

    fn rotateLeft(self: *RBTree, x: *RBNode) void {
        const y = x.right.?;
        x.right = y.left;
        if (y.left) |yl| yl.parent = x;
        y.parent = x.parent;

        if (x.parent == null) {
            self.root = y;
        } else if (x == x.parent.?.left) {
            x.parent.?.left = y;
        } else {
            x.parent.?.right = y;
        }
        y.left = x;
        x.parent = y;
    }

    fn rotateRight(self: *RBTree, y: *RBNode) void {
        const x = y.left.?;
        y.left = x.right;
        if (x.right) |xr| xr.parent = y;
        x.parent = y.parent;

        if (y.parent == null) {
            self.root = x;
        } else if (y == y.parent.?.left) {
            y.parent.?.left = x;
        } else {
            y.parent.?.right = x;
        }
        x.right = y;
        y.parent = x;
    }

    pub fn insert(self: *RBTree, key: i32) !void {
        const node = try self.allocator.create(RBNode);
        node.* = .{ .key = key, .color = .red };

        var parent: ?*RBNode = null;
        var current = self.root;

        while (current) |cur| {
            parent = cur;
            if (key < cur.key) {
                current = cur.left;
            } else if (key > cur.key) {
                current = cur.right;
            } else {
                self.allocator.destroy(node);
                return; // duplicate
            }
        }

        node.parent = parent;
        if (parent == null) {
            self.root = node;
        } else if (key < parent.?.key) {
            parent.?.left = node;
        } else {
            parent.?.right = node;
        }

        self.size += 1;
        self.fixInsert(node);
    }

    fn fixInsert(self: *RBTree, z: *RBNode) void {
        var node = z;
        while (node.parent != null and node.parent.?.color == .red) {
            const parent = node.parent.?;
            const grandparent = parent.parent orelse break;

            if (parent == grandparent.left) {
                const uncle = grandparent.right;
                if (uncle != null and uncle.?.color == .red) {
                    // Case 1: Uncle สีแดง
                    parent.color = .black;
                    uncle.?.color = .black;
                    grandparent.color = .red;
                    node = grandparent;
                } else {
                    if (node == parent.right) {
                        // Case 2: Uncle สีดำ, node เป็นลูกขวา
                        node = parent;
                        self.rotateLeft(node);
                    }
                    // Case 3: Uncle สีดำ, node เป็นลูกซ้าย
                    node.parent.?.color = .black;
                    node.parent.?.parent.?.color = .red;
                    self.rotateRight(node.parent.?.parent.?);
                }
            } else {
                // Mirror cases
                const uncle = grandparent.left;
                if (uncle != null and uncle.?.color == .red) {
                    parent.color = .black;
                    uncle.?.color = .black;
                    grandparent.color = .red;
                    node = grandparent;
                } else {
                    if (node == parent.left) {
                        node = parent;
                        self.rotateRight(node);
                    }
                    node.parent.?.color = .black;
                    node.parent.?.parent.?.color = .red;
                    self.rotateLeft(node.parent.?.parent.?);
                }
            }
        }
        if (self.root) |root| root.color = .black;
    }

    pub fn verifyProperties(self: *RBTree) bool {
        if (self.root) |root| {
            if (root.color != .black) return false;
        }
        return self.verifyNode(self.root) >= 0;
    }

    fn verifyNode(_: *RBTree, node: ?*RBNode) i32 {
        const n = node orelse return 1; // null = black leaf
        if (n.color == .red) {
            if (n.left) |l| { if (l.color == .red) return -1; }
            if (n.right) |r| { if (r.color == .red) return -1; }
        }
        const left_bh = RBTree.verifyNode(undefined, n.left);
        const right_bh = RBTree.verifyNode(undefined, n.right);
        if (left_bh < 0 or right_bh < 0 or left_bh != right_bh) return -1;
        return left_bh + (if (n.color == .black) @as(i32, 1) else 0);
    }

    pub fn printStats(self: *RBTree) void {
        std.debug.print("RB Tree: size={d}, valid={}\n", .{
            self.size,
            self.verifyProperties(),
        });
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Red-Black Tree Demo ===\n", .{});

    var rbt = RBTree.init(allocator);
    defer rbt.deinit();

    var i: i32 = 1;
    while (i <= 20) : (i += 1) {
        try rbt.insert(i);
    }

    rbt.printStats();
}
```

---

## ขั้นตอนที่ 776: B-tree Concept

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// B-tree of order t (minimum degree)
fn BTree(comptime t: comptime_int) type {
    const MAX_KEYS = 2 * t - 1;
    const MAX_CHILDREN = 2 * t;

    return struct {
        const Self = @This();

        const Node = struct {
            keys: [MAX_KEYS]i32 = undefined,
            children: [MAX_CHILDREN]?*Node = [_]?*Node{null} ** MAX_CHILDREN,
            n: usize = 0, // จำนวน keys ปัจจุบัน
            leaf: bool = true,

            pub fn isFull(self: *Node) bool {
                return self.n == MAX_KEYS;
            }
        };

        root: ?*Node = null,
        allocator: Allocator,

        pub fn init(allocator: Allocator) Self {
            return .{ .allocator = allocator };
        }

        pub fn deinit(self: *Self) void {
            self.freeNode(self.root);
        }

        fn freeNode(self: *Self, node: ?*Node) void {
            if (node) |n| {
                if (!n.leaf) {
                    for (0..n.n + 1) |i| {
                        self.freeNode(n.children[i]);
                    }
                }
                self.allocator.destroy(n);
            }
        }

        pub fn search(self: *Self, key: i32) bool {
            return self.searchNode(self.root, key);
        }

        fn searchNode(_: *Self, node: ?*Node, key: i32) bool {
            const n = node orelse return false;
            var i: usize = 0;
            while (i < n.n and key > n.keys[i]) i += 1;
            if (i < n.n and key == n.keys[i]) return true;
            if (n.leaf) return false;
            return BTree(t).searchNode(undefined, n.children[i], key);
        }

        pub fn insert(self: *Self, key: i32) !void {
            if (self.root == null) {
                const node = try self.allocator.create(Node);
                node.* = .{};
                self.root = node;
            }

            const root = self.root.?;
            if (root.isFull()) {
                const new_root = try self.allocator.create(Node);
                new_root.* = .{ .leaf = false };
                new_root.children[0] = root;
                try self.splitChild(new_root, 0);
                self.root = new_root;
            }
            try self.insertNonFull(self.root.?, key);
        }

        fn splitChild(self: *Self, parent: *Node, i: usize) !void {
            const full_child = parent.children[i].?;
            const new_child = try self.allocator.create(Node);
            new_child.* = .{ .leaf = full_child.leaf };
            new_child.n = t - 1;

            // Copy second half of keys
            for (0..t - 1) |j| {
                new_child.keys[j] = full_child.keys[j + t];
            }

            // Copy children if not leaf
            if (!full_child.leaf) {
                for (0..t) |j| {
                    new_child.children[j] = full_child.children[j + t];
                }
            }

            full_child.n = t - 1;

            // Shift parent's children right
            var j: usize = parent.n;
            while (j > i) : (j -= 1) {
                parent.children[j + 1] = parent.children[j];
            }
            parent.children[i + 1] = new_child;

            // Shift parent's keys right
            j = parent.n;
            while (j > i) : (j -= 1) {
                parent.keys[j] = parent.keys[j - 1];
            }
            parent.keys[i] = full_child.keys[t - 1];
            parent.n += 1;
        }

        fn insertNonFull(self: *Self, node: *Node, key: i32) !void {
            var i: i32 = @intCast(node.n);
            i -= 1;

            if (node.leaf) {
                while (i >= 0 and key < node.keys[@intCast(i)]) {
                    node.keys[@intCast(i + 1)] = node.keys[@intCast(i)];
                    i -= 1;
                }
                node.keys[@intCast(i + 1)] = key;
                node.n += 1;
            } else {
                while (i >= 0 and key < node.keys[@intCast(i)]) {
                    i -= 1;
                }
                i += 1;
                const child_idx: usize = @intCast(i);
                if (node.children[child_idx].?.isFull()) {
                    try self.splitChild(node, child_idx);
                    if (key > node.keys[child_idx]) i += 1;
                }
                try self.insertNonFull(node.children[@intCast(i)].?, key);
            }
        }

        pub fn printStats(self: *Self) void {
            std.debug.print("B-Tree (t={d}): ", .{t});
            self.printNode(self.root, 0);
        }

        fn printNode(_: *Self, node: ?*Node, depth: usize) void {
            const n = node orelse return;
            for (0..depth) |_| std.debug.print("  ", .{});
            std.debug.print("[", .{});
            for (0..n.n) |i| {
                std.debug.print("{d}", .{n.keys[i]});
                if (i < n.n - 1) std.debug.print(",", .{});
            }
            std.debug.print("]\n", .{});
            if (!n.leaf) {
                for (0..n.n + 1) |i| {
                    BTree(t).printNode(undefined, n.children[i], depth + 1);
                }
            }
        }
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== B-Tree Demo (t=3) ===\n", .{});

    var btree = BTree(3).init(allocator);
    defer btree.deinit();

    const values = [_]i32{ 10, 20, 5, 6, 12, 30, 7, 17 };
    for (values) |v| try btree.insert(v);

    btree.printStats();
    std.debug.print("\nSearch 6: {}\n", .{btree.search(6)});
    std.debug.print("Search 15: {}\n", .{btree.search(15)});
}
```

---

## ขั้นตอนที่ 777: Heap (Min-Heap, Max-Heap)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn Heap(comptime T: type, comptime less_than: fn (T, T) bool) type {
    return struct {
        const Self = @This();

        data: std.ArrayList(T),

        pub fn init(allocator: Allocator) Self {
            return .{ .data = std.ArrayList(T).init(allocator) };
        }

        pub fn deinit(self: *Self) void {
            self.data.deinit();
        }

        fn parent(i: usize) usize { return (i - 1) / 2; }
        fn leftChild(i: usize) usize { return 2 * i + 1; }
        fn rightChild(i: usize) usize { return 2 * i + 2; }

        pub fn push(self: *Self, item: T) !void {
            try self.data.append(item);
            self.siftUp(self.data.items.len - 1);
        }

        pub fn pop(self: *Self) ?T {
            if (self.data.items.len == 0) return null;
            const top = self.data.items[0];
            const last = self.data.pop();
            if (self.data.items.len > 0) {
                self.data.items[0] = last;
                self.siftDown(0);
            }
            return top;
        }

        pub fn peek(self: *Self) ?T {
            if (self.data.items.len == 0) return null;
            return self.data.items[0];
        }

        fn siftUp(self: *Self, idx: usize) void {
            var i = idx;
            while (i > 0) {
                const p = parent(i);
                if (less_than(self.data.items[i], self.data.items[p])) {
                    std.mem.swap(T, &self.data.items[i], &self.data.items[p]);
                    i = p;
                } else break;
            }
        }

        fn siftDown(self: *Self, idx: usize) void {
            var i = idx;
            const n = self.data.items.len;
            while (true) {
                var smallest = i;
                const left = leftChild(i);
                const right = rightChild(i);

                if (left < n and less_than(self.data.items[left], self.data.items[smallest])) {
                    smallest = left;
                }
                if (right < n and less_than(self.data.items[right], self.data.items[smallest])) {
                    smallest = right;
                }

                if (smallest == i) break;
                std.mem.swap(T, &self.data.items[i], &self.data.items[smallest]);
                i = smallest;
            }
        }

        pub fn len(self: *Self) usize {
            return self.data.items.len;
        }

        // Heap sort
        pub fn sort(self: *Self) ![]T {
            const result = try self.data.allocator.alloc(T, self.data.items.len);
            var i: usize = 0;
            while (self.pop()) |item| {
                result[i] = item;
                i += 1;
            }
            return result;
        }
    };
}

fn lessThanI32(a: i32, b: i32) bool { return a < b; }
fn greaterThanI32(a: i32, b: i32) bool { return a > b; }

const MinHeap = Heap(i32, lessThanI32);
const MaxHeap = Heap(i32, greaterThanI32);

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Heap Demo ===\n", .{});

    var min_heap = MinHeap.init(allocator);
    defer min_heap.deinit();

    var max_heap = MaxHeap.init(allocator);
    defer max_heap.deinit();

    const values = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };
    for (values) |v| {
        try min_heap.push(v);
        try max_heap.push(v);
    }

    std.debug.print("Min Heap sort: ", .{});
    const sorted = try min_heap.sort();
    defer allocator.free(sorted);
    for (sorted) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});

    std.debug.print("Max Heap pop: ", .{});
    while (max_heap.pop()) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});
}
```

---

## ขั้นตอนที่ 778: Priority Queue

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const Priority = enum(u8) { low = 1, normal = 2, high = 3, critical = 4 };

const Task = struct {
    id: u32,
    name: []const u8,
    priority: Priority,
    deadline: i64,

    fn lessThan(_: void, a: Task, b: Task) std.math.Order {
        // สูงกว่า = critical ก่อน, ถ้าเท่ากันใช้ deadline
        const pa = @intFromEnum(a.priority);
        const pb = @intFromEnum(b.priority);
        if (pa != pb) return if (pa > pb) .lt else .gt;
        return if (a.deadline < b.deadline) .lt else .gt;
    }
};

const PriorityQueue = std.PriorityQueue(Task, void, Task.lessThan);

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Priority Queue Demo ===\n", .{});

    var pq = PriorityQueue.init(allocator, {});
    defer pq.deinit();

    const now = std.time.milliTimestamp();
    try pq.add(.{ .id = 1, .name = "Backup DB", .priority = .low, .deadline = now + 3600000 });
    try pq.add(.{ .id = 2, .name = "Process Payment", .priority = .critical, .deadline = now + 1000 });
    try pq.add(.{ .id = 3, .name = "Send Report", .priority = .normal, .deadline = now + 7200000 });
    try pq.add(.{ .id = 4, .name = "Security Alert", .priority = .critical, .deadline = now + 500 });
    try pq.add(.{ .id = 5, .name = "Analytics", .priority = .high, .deadline = now + 1800000 });

    std.debug.print("ประมวลผลตาม priority:\n", .{});
    while (pq.removeOrNull()) |task| {
        std.debug.print("  [{s}] #{d}: {s}\n", .{
            @tagName(task.priority),
            task.id,
            task.name,
        });
    }
}
```

---

## ขั้นตอนที่ 779: Trie (Prefix Tree)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const ALPHABET_SIZE = 26;

const TrieNode = struct {
    children: [ALPHABET_SIZE]?*TrieNode = [_]?*TrieNode{null} ** ALPHABET_SIZE,
    is_end: bool = false,
    count: u32 = 0, // จำนวนคำที่ผ่านมา
};

const Trie = struct {
    root: *TrieNode,
    allocator: Allocator,
    word_count: usize = 0,

    pub fn init(allocator: Allocator) !Trie {
        const root = try allocator.create(TrieNode);
        root.* = .{};
        return Trie{ .root = root, .allocator = allocator };
    }

    pub fn deinit(self: *Trie) void {
        self.freeNode(self.root);
    }

    fn freeNode(self: *Trie, node: *TrieNode) void {
        for (node.children) |child| {
            if (child) |c| self.freeNode(c);
        }
        self.allocator.destroy(node);
    }

    pub fn insert(self: *Trie, word: []const u8) !void {
        var current = self.root;
        for (word) |c| {
            const idx = std.ascii.toLower(c) - 'a';
            if (current.children[idx] == null) {
                const node = try self.allocator.create(TrieNode);
                node.* = .{};
                current.children[idx] = node;
            }
            current = current.children[idx].?;
            current.count += 1;
        }
        if (!current.is_end) {
            current.is_end = true;
            self.word_count += 1;
        }
    }

    pub fn search(self: *Trie, word: []const u8) bool {
        var current = self.root;
        for (word) |c| {
            const idx = std.ascii.toLower(c) - 'a';
            current = current.children[idx] orelse return false;
        }
        return current.is_end;
    }

    pub fn startsWith(self: *Trie, prefix: []const u8) bool {
        var current = self.root;
        for (prefix) |c| {
            const idx = std.ascii.toLower(c) - 'a';
            current = current.children[idx] orelse return false;
        }
        return true;
    }

    pub fn countWordsWithPrefix(self: *Trie, prefix: []const u8) u32 {
        var current = self.root;
        for (prefix) |c| {
            const idx = std.ascii.toLower(c) - 'a';
            current = current.children[idx] orelse return 0;
        }
        return current.count;
    }

    pub fn autocomplete(self: *Trie, prefix: []const u8, results: *std.ArrayList([]u8)) !void {
        var current = self.root;
        for (prefix) |c| {
            const idx = std.ascii.toLower(c) - 'a';
            current = current.children[idx] orelse return;
        }
        var buf = std.ArrayList(u8).init(self.allocator);
        defer buf.deinit();
        try buf.appendSlice(prefix);
        try self.collectWords(current, &buf, results);
    }

    fn collectWords(self: *Trie, node: *TrieNode, prefix: *std.ArrayList(u8), results: *std.ArrayList([]u8)) !void {
        if (node.is_end) {
            try results.append(try self.allocator.dupe(u8, prefix.items));
        }
        for (node.children, 0..) |child, i| {
            if (child) |c| {
                try prefix.append(@intCast('a' + i));
                try self.collectWords(c, prefix, results);
                _ = prefix.pop();
            }
        }
    }

    pub fn delete(self: *Trie, word: []const u8) bool {
        return self.deleteHelper(self.root, word, 0);
    }

    fn deleteHelper(self: *Trie, node: *TrieNode, word: []const u8, depth: usize) bool {
        if (depth == word.len) {
            if (!node.is_end) return false;
            node.is_end = false;
            self.word_count -= 1;
            return true; // ลบได้ถ้าไม่มีลูก
        }
        const idx = std.ascii.toLower(word[depth]) - 'a';
        const child = node.children[idx] orelse return false;
        const should_delete = self.deleteHelper(child, word, depth + 1);
        if (should_delete) {
            node.count -= 1;
        }
        return should_delete;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Trie Demo ===\n", .{});

    var trie = try Trie.init(allocator);
    defer trie.deinit();

    const words = [_][]const u8{
        "apple", "app", "application", "apply",
        "banana", "band", "bandana", "ban",
        "cat", "car", "card", "care",
    };

    for (words) |word| try trie.insert(word);

    std.debug.print("คำทั้งหมด: {d}\n", .{trie.word_count});
    std.debug.print("Search 'app': {}\n", .{trie.search("app")});
    std.debug.print("Search 'apps': {}\n", .{trie.search("apps")});
    std.debug.print("StartsWith 'ban': {}\n", .{trie.startsWith("ban")});
    std.debug.print("คำที่ขึ้นต้นด้วย 'app': {d}\n", .{trie.countWordsWithPrefix("app")});

    std.debug.print("\nAutocomplete 'app':\n", .{});
    var results = std.ArrayList([]u8).init(allocator);
    defer {
        for (results.items) |r| allocator.free(r);
        results.deinit();
    }
    try trie.autocomplete("app", &results);
    for (results.items) |word| std.debug.print("  {s}\n", .{word});
}
```

---

## ขั้นตอนที่ 780: Real-world Tree Usage

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Expression Tree สำหรับคำนวณ expressions
const TokenType = enum { number, add, sub, mul, div };

const ExprNode = struct {
    token_type: TokenType,
    value: f64 = 0,
    left: ?*ExprNode = null,
    right: ?*ExprNode = null,

    pub fn evaluate(self: *ExprNode) f64 {
        return switch (self.token_type) {
            .number => self.value,
            .add => self.left.?.evaluate() + self.right.?.evaluate(),
            .sub => self.left.?.evaluate() - self.right.?.evaluate(),
            .mul => self.left.?.evaluate() * self.right.?.evaluate(),
            .div => self.left.?.evaluate() / self.right.?.evaluate(),
        };
    }

    pub fn print(self: *ExprNode) void {
        switch (self.token_type) {
            .number => std.debug.print("{d}", .{self.value}),
            else => {
                std.debug.print("(", .{});
                self.left.?.print();
                std.debug.print(" {s} ", .{switch (self.token_type) {
                    .add => "+",
                    .sub => "-",
                    .mul => "*",
                    .div => "/",
                    else => "?",
                }});
                self.right.?.print();
                std.debug.print(")", .{});
            },
        }
    }
};

// File System Tree
const FileType = enum { file, directory };

const FileNode = struct {
    name: []const u8,
    file_type: FileType,
    size: u64,
    children: std.ArrayList(*FileNode),
    allocator: Allocator,

    pub fn init(allocator: Allocator, name: []const u8, file_type: FileType, size: u64) FileNode {
        return .{
            .name = name,
            .file_type = file_type,
            .size = size,
            .children = std.ArrayList(*FileNode).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *FileNode) void {
        for (self.children.items) |child| {
            child.deinit();
            self.allocator.destroy(child);
        }
        self.children.deinit();
    }

    pub fn addChild(self: *FileNode, child: *FileNode) !void {
        try self.children.append(child);
    }

    pub fn totalSize(self: *FileNode) u64 {
        if (self.file_type == .file) return self.size;
        var total: u64 = 0;
        for (self.children.items) |child| {
            total += child.totalSize();
        }
        return total;
    }

    pub fn print(self: *FileNode, depth: usize) void {
        for (0..depth) |_| std.debug.print("  ", .{});
        const icon = if (self.file_type == .directory) "📁" else "📄";
        _ = icon;
        std.debug.print("[{s}] {s} ({d} bytes)\n", .{
            if (self.file_type == .directory) "DIR" else "FILE",
            self.name,
            self.totalSize(),
        });
        for (self.children.items) |child| {
            child.print(depth + 1);
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Real-world Tree Usage ===\n", .{});

    // Expression Tree: (3 + 4) * (5 - 2)
    std.debug.print("\n--- Expression Tree ---\n", .{});
    var nodes: [7]ExprNode = undefined;
    nodes[0] = .{ .token_type = .number, .value = 3 };
    nodes[1] = .{ .token_type = .number, .value = 4 };
    nodes[2] = .{ .token_type = .add, .left = &nodes[0], .right = &nodes[1] };
    nodes[3] = .{ .token_type = .number, .value = 5 };
    nodes[4] = .{ .token_type = .number, .value = 2 };
    nodes[5] = .{ .token_type = .sub, .left = &nodes[3], .right = &nodes[4] };
    nodes[6] = .{ .token_type = .mul, .left = &nodes[2], .right = &nodes[5] };

    std.debug.print("Expression: ", .{});
    nodes[6].print();
    std.debug.print("\nResult: {d}\n", .{nodes[6].evaluate()});

    // File System Tree
    std.debug.print("\n--- File System Tree ---\n", .{});
    var root = FileNode.init(allocator, "/home/user", .directory, 0);
    defer root.deinit();

    var docs = try allocator.create(FileNode);
    docs.* = FileNode.init(allocator, "Documents", .directory, 0);
    var report = try allocator.create(FileNode);
    report.* = FileNode.init(allocator, "report.pdf", .file, 1024000);
    var notes = try allocator.create(FileNode);
    notes.* = FileNode.init(allocator, "notes.txt", .file, 2048);
    try docs.addChild(report);
    try docs.addChild(notes);

    var code = try allocator.create(FileNode);
    code.* = FileNode.init(allocator, "code", .directory, 0);
    var main_zig = try allocator.create(FileNode);
    main_zig.* = FileNode.init(allocator, "main.zig", .file, 8192);
    try code.addChild(main_zig);

    try root.addChild(docs);
    try root.addChild(code);

    root.print(0);
    std.debug.print("Total size: {d} bytes\n", .{root.totalSize()});
}
```

---

## สรุปส่วนที่ 78: Binary Trees

| Tree | Insert | Search | Delete | Space | ลักษณะพิเศษ |
|------|--------|--------|--------|-------|-------------|
| BST | O(h) | O(h) | O(h) | O(n) | Simple, unbalanced |
| AVL | O(log n) | O(log n) | O(log n) | O(n) | Strictly balanced |
| Red-Black | O(log n) | O(log n) | O(log n) | O(n) | Loosely balanced |
| B-Tree | O(log n) | O(log n) | O(log n) | O(n) | Disk-friendly |
| Heap | O(log n) | O(n) | O(log n) | O(n) | Min/Max ใน O(1) |
| Trie | O(m) | O(m) | O(m) | O(ALPHA*n*m) | String operations |

h = ความสูง, m = ความยาว string

---

*ส่วนถัดไป: Part 79 - Data Structures: Graph*
