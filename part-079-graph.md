# ส่วนที่ 79: Data Structures: Graph (ขั้นตอนที่ 781-790)

Graph เป็น data structure ที่ประกอบด้วย vertices (nodes) และ edges (connections) ใช้แทนความสัมพันธ์ระหว่าง entities ต่างๆ เช่น social networks, road maps, dependencies

---

## ขั้นตอนที่ 781: Graph Representation (Adjacency List)

Adjacency List ใช้ list หรือ array เก็บ neighbors ของแต่ละ vertex - ประหยัด memory สำหรับ sparse graphs

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const Edge = struct {
    to: usize,
    weight: f64 = 1.0,
};

const GraphList = struct {
    allocator: Allocator,
    adj: []std.ArrayList(Edge),
    vertices: usize,
    directed: bool,

    pub fn init(allocator: Allocator, vertices: usize, directed: bool) !GraphList {
        const adj = try allocator.alloc(std.ArrayList(Edge), vertices);
        for (adj) |*list| {
            list.* = std.ArrayList(Edge).init(allocator);
        }
        return GraphList{
            .allocator = allocator,
            .adj = adj,
            .vertices = vertices,
            .directed = directed,
        };
    }

    pub fn deinit(self: *GraphList) void {
        for (self.adj) |*list| list.deinit();
        self.allocator.free(self.adj);
    }

    pub fn addEdge(self: *GraphList, from: usize, to: usize, weight: f64) !void {
        try self.adj[from].append(Edge{ .to = to, .weight = weight });
        if (!self.directed) {
            try self.adj[to].append(Edge{ .to = from, .weight = weight });
        }
    }

    pub fn removeEdge(self: *GraphList, from: usize, to: usize) void {
        var list = &self.adj[from];
        var i: usize = 0;
        while (i < list.items.len) {
            if (list.items[i].to == to) {
                _ = list.orderedRemove(i);
            } else {
                i += 1;
            }
        }
        if (!self.directed) {
            var list2 = &self.adj[to];
            i = 0;
            while (i < list2.items.len) {
                if (list2.items[i].to == from) {
                    _ = list2.orderedRemove(i);
                } else {
                    i += 1;
                }
            }
        }
    }

    pub fn hasEdge(self: *GraphList, from: usize, to: usize) bool {
        for (self.adj[from].items) |edge| {
            if (edge.to == to) return true;
        }
        return false;
    }

    pub fn getNeighbors(self: *GraphList, vertex: usize) []Edge {
        return self.adj[vertex].items;
    }

    pub fn degree(self: *GraphList, vertex: usize) usize {
        return self.adj[vertex].items.len;
    }

    pub fn edgeCount(self: *GraphList) usize {
        var count: usize = 0;
        for (self.adj) |list| count += list.items.len;
        return if (self.directed) count else count / 2;
    }

    pub fn print(self: *GraphList) void {
        std.debug.print("Graph (V={d}, E={d}, directed={}):\n", .{
            self.vertices,
            self.edgeCount(),
            self.directed,
        });
        for (self.adj, 0..) |list, v| {
            std.debug.print("  {d} -> ", .{v});
            for (list.items) |edge| {
                std.debug.print("{d}({d:.1}) ", .{ edge.to, edge.weight });
            }
            std.debug.print("\n", .{});
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Adjacency List Graph ===\n", .{});

    // Undirected weighted graph
    var g = try GraphList.init(allocator, 6, false);
    defer g.deinit();

    try g.addEdge(0, 1, 4.0);
    try g.addEdge(0, 2, 2.0);
    try g.addEdge(1, 2, 1.0);
    try g.addEdge(1, 3, 5.0);
    try g.addEdge(2, 3, 8.0);
    try g.addEdge(2, 4, 10.0);
    try g.addEdge(3, 4, 2.0);
    try g.addEdge(3, 5, 6.0);
    try g.addEdge(4, 5, 3.0);

    g.print();

    std.debug.print("Degree of vertex 2: {d}\n", .{g.degree(2)});
    std.debug.print("Has edge 0->1: {}\n", .{g.hasEdge(0, 1)});
    std.debug.print("Has edge 0->5: {}\n", .{g.hasEdge(0, 5)});
}
```

---

## ขั้นตอนที่ 782: Graph Representation (Adjacency Matrix)

Adjacency Matrix ใช้ 2D array เก็บ edges - เหมาะกับ dense graphs และ O(1) edge lookup

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const INF = std.math.inf(f64);

const GraphMatrix = struct {
    allocator: Allocator,
    matrix: [][]f64,
    vertices: usize,
    directed: bool,

    pub fn init(allocator: Allocator, vertices: usize, directed: bool) !GraphMatrix {
        const matrix = try allocator.alloc([]f64, vertices);
        for (matrix) |*row| {
            row.* = try allocator.alloc(f64, vertices);
            for (row.*) |*cell| cell.* = INF;
        }
        // Diagonal = 0
        for (0..vertices) |i| matrix[i][i] = 0;

        return GraphMatrix{
            .allocator = allocator,
            .matrix = matrix,
            .vertices = vertices,
            .directed = directed,
        };
    }

    pub fn deinit(self: *GraphMatrix) void {
        for (self.matrix) |row| self.allocator.free(row);
        self.allocator.free(self.matrix);
    }

    pub fn addEdge(self: *GraphMatrix, from: usize, to: usize, weight: f64) void {
        self.matrix[from][to] = weight;
        if (!self.directed) self.matrix[to][from] = weight;
    }

    pub fn removeEdge(self: *GraphMatrix, from: usize, to: usize) void {
        self.matrix[from][to] = INF;
        if (!self.directed) self.matrix[to][from] = INF;
    }

    pub fn hasEdge(self: *GraphMatrix, from: usize, to: usize) bool {
        return self.matrix[from][to] != INF;
    }

    pub fn getWeight(self: *GraphMatrix, from: usize, to: usize) f64 {
        return self.matrix[from][to];
    }

    // Floyd-Warshall All-Pairs Shortest Path O(V^3)
    pub fn floydWarshall(self: *GraphMatrix) ![][]f64 {
        const n = self.vertices;
        const dist = try self.allocator.alloc([]f64, n);
        for (dist, 0..) |*row, i| {
            row.* = try self.allocator.alloc(f64, n);
            @memcpy(row.*, self.matrix[i]);
        }

        for (0..n) |k| {
            for (0..n) |i| {
                for (0..n) |j| {
                    if (dist[i][k] != INF and dist[k][j] != INF) {
                        const new_dist = dist[i][k] + dist[k][j];
                        if (new_dist < dist[i][j]) {
                            dist[i][j] = new_dist;
                        }
                    }
                }
            }
        }
        return dist;
    }

    pub fn print(self: *GraphMatrix) void {
        std.debug.print("Adjacency Matrix ({d}x{d}):\n", .{ self.vertices, self.vertices });
        std.debug.print("    ", .{});
        for (0..self.vertices) |j| std.debug.print("{d:4}", .{j});
        std.debug.print("\n", .{});
        for (self.matrix, 0..) |row, i| {
            std.debug.print("{d:3}: ", .{i});
            for (row) |cell| {
                if (cell == INF) {
                    std.debug.print(" INF", .{});
                } else if (cell == 0) {
                    std.debug.print("   0", .{});
                } else {
                    std.debug.print("{d:4.0}", .{cell});
                }
            }
            std.debug.print("\n", .{});
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Adjacency Matrix Graph ===\n", .{});

    var g = try GraphMatrix.init(allocator, 4, true);
    defer g.deinit();

    g.addEdge(0, 1, 3.0);
    g.addEdge(0, 3, 7.0);
    g.addEdge(1, 0, 8.0);
    g.addEdge(1, 2, 2.0);
    g.addEdge(2, 0, 5.0);
    g.addEdge(2, 3, 1.0);
    g.addEdge(3, 0, 2.0);

    g.print();

    // Floyd-Warshall
    std.debug.print("\nAll-pairs shortest paths:\n", .{});
    const dist = try g.floydWarshall();
    defer {
        for (dist) |row| allocator.free(row);
        allocator.free(dist);
    }

    std.debug.print("    ", .{});
    for (0..4) |j| std.debug.print("{d:6}", .{j});
    std.debug.print("\n", .{});
    for (dist, 0..) |row, i| {
        std.debug.print("{d:3}: ", .{i});
        for (row) |d| {
            if (d == INF) std.debug.print("   INF", .{})
            else std.debug.print("{d:6.1}", .{d});
        }
        std.debug.print("\n", .{});
    }
}
```

---

## ขั้นตอนที่ 783: BFS (Breadth-First Search)

BFS สำรวจ graph ทีละ level ใช้ queue เหมาะกับหา shortest path ใน unweighted graph

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const BFSResult = struct {
    distances: []i32,
    parents: []i32,
    order: []usize,
    allocator: Allocator,

    pub fn deinit(self: *BFSResult) void {
        self.allocator.free(self.distances);
        self.allocator.free(self.parents);
        self.allocator.free(self.order);
    }
};

fn bfs(allocator: Allocator, adj: [][]usize, start: usize) !BFSResult {
    const n = adj.len;
    const distances = try allocator.alloc(i32, n);
    const parents = try allocator.alloc(i32, n);
    const order = try allocator.alloc(usize, n);
    var order_len: usize = 0;

    @memset(distances, -1);
    @memset(parents, -1);
    distances[start] = 0;

    var queue = std.ArrayList(usize).init(allocator);
    defer queue.deinit();

    try queue.append(start);

    while (queue.items.len > 0) {
        const current = queue.orderedRemove(0);
        order[order_len] = current;
        order_len += 1;

        for (adj[current]) |neighbor| {
            if (distances[neighbor] == -1) {
                distances[neighbor] = distances[current] + 1;
                parents[neighbor] = @intCast(current);
                try queue.append(neighbor);
            }
        }
    }

    return BFSResult{
        .distances = distances,
        .parents = parents,
        .order = order[0..order_len],
        .allocator = allocator,
    };
}

fn reconstructPath(result: *BFSResult, target: usize) !std.ArrayList(usize) {
    var path = std.ArrayList(usize).init(result.allocator);
    var current: i32 = @intCast(target);
    while (current != -1) {
        try path.insert(0, @intCast(current));
        current = result.parents[@intCast(current)];
    }
    return path;
}

// BFS สำหรับ Word Ladder
fn wordLadder(allocator: Allocator, begin: []const u8, end: []const u8, word_list: [][]const u8) !usize {
    if (begin.len != end.len) return 0;

    var visited = std.StringHashMap(bool).init(allocator);
    defer visited.deinit();

    var queue = std.ArrayList(struct { word: []const u8, steps: usize }).init(allocator);
    defer queue.deinit();

    try queue.append(.{ .word = begin, .steps = 1 });
    try visited.put(begin, true);

    while (queue.items.len > 0) {
        const item = queue.orderedRemove(0);
        const word = item.word;
        const steps = item.steps;

        if (std.mem.eql(u8, word, end)) return steps;

        for (word_list) |candidate| {
            if (visited.contains(candidate)) continue;

            // ตรวจสอบว่าต่างกัน 1 ตัวอักษร
            var diff: usize = 0;
            for (word, 0..) |c, i| {
                if (i < candidate.len and c != candidate[i]) diff += 1;
            }
            if (diff == 1) {
                try visited.put(candidate, true);
                try queue.append(.{ .word = candidate, .steps = steps + 1 });
            }
        }
    }
    return 0;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== BFS Demo ===\n", .{});

    // Build graph
    var adj_data = [_][4]usize{
        [_]usize{ 1, 2, 0, 0 }, // 0 -> 1, 2
        [_]usize{ 0, 2, 3, 0 }, // 1 -> 0, 2, 3 (ใช้ 0 เป็น sentinel)
        [_]usize{ 0, 1, 4, 0 }, // 2 -> 0, 1, 4
        [_]usize{ 1, 4, 5, 0 }, // 3 -> 1, 4, 5
        [_]usize{ 2, 3, 5, 0 }, // 4 -> 2, 3, 5
        [_]usize{ 3, 4, 0, 0 }, // 5 -> 3, 4
    };

    // Convert to slices for actual edges
    const edges_0 = [_]usize{ 1, 2 };
    const edges_1 = [_]usize{ 0, 2, 3 };
    const edges_2 = [_]usize{ 0, 1, 4 };
    const edges_3 = [_]usize{ 1, 4, 5 };
    const edges_4 = [_]usize{ 2, 3, 5 };
    const edges_5 = [_]usize{ 3, 4 };
    _ = adj_data;

    var adj = [_][]const usize{
        &edges_0, &edges_1, &edges_2, &edges_3, &edges_4, &edges_5,
    };

    var adj_mut: [6][]usize = undefined;
    for (&adj_mut, 0..) |*a, i| {
        a.* = try allocator.dupe(usize, adj[i]);
    }
    defer for (adj_mut) |a| allocator.free(a);

    var result = try bfs(allocator, &adj_mut, 0);
    defer result.deinit();

    std.debug.print("BFS from vertex 0:\n", .{});
    std.debug.print("Order: ", .{});
    for (result.order) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});

    std.debug.print("Distances: ", .{});
    for (result.distances) |d| std.debug.print("{d} ", .{d});
    std.debug.print("\n", .{});

    var path = try reconstructPath(&result, 5);
    defer path.deinit();
    std.debug.print("Path 0->5: ", .{});
    for (path.items) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});

    // Word Ladder
    std.debug.print("\n--- Word Ladder ---\n", .{});
    var words = [_][]const u8{ "hot", "dot", "dog", "lot", "log", "cog" };
    const steps = try wordLadder(allocator, "hit", "cog", &words);
    std.debug.print("hit -> cog: {d} steps\n", .{steps});
}
```

---

## ขั้นตอนที่ 784: DFS (Depth-First Search)

DFS สำรวจ graph ลึกที่สุดก่อน ใช้ stack หรือ recursion เหมาะกับ cycle detection, topological sort

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const DFSColor = enum { white, gray, black }; // unvisited, in-progress, done

const DFS = struct {
    allocator: Allocator,
    adj: []std.ArrayList(usize),
    color: []DFSColor,
    discovery: []u32,
    finish: []u32,
    parent: []i32,
    timer: u32 = 0,

    pub fn init(allocator: Allocator, vertices: usize) !DFS {
        const adj = try allocator.alloc(std.ArrayList(usize), vertices);
        for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);

        return DFS{
            .allocator = allocator,
            .adj = adj,
            .color = try allocator.alloc(DFSColor, vertices),
            .discovery = try allocator.alloc(u32, vertices),
            .finish = try allocator.alloc(u32, vertices),
            .parent = try allocator.alloc(i32, vertices),
        };
    }

    pub fn deinit(self: *DFS) void {
        for (self.adj) |*list| list.deinit();
        self.allocator.free(self.adj);
        self.allocator.free(self.color);
        self.allocator.free(self.discovery);
        self.allocator.free(self.finish);
        self.allocator.free(self.parent);
    }

    pub fn addEdge(self: *DFS, from: usize, to: usize) !void {
        try self.adj[from].append(to);
    }

    pub fn run(self: *DFS) void {
        const n = self.adj.len;
        @memset(self.color, .white);
        @memset(self.parent, -1);
        self.timer = 0;

        for (0..n) |u| {
            if (self.color[u] == .white) {
                self.dfsVisit(u);
            }
        }
    }

    fn dfsVisit(self: *DFS, u: usize) void {
        self.timer += 1;
        self.discovery[u] = self.timer;
        self.color[u] = .gray;

        for (self.adj[u].items) |v| {
            if (self.color[v] == .white) {
                self.parent[v] = @intCast(u);
                self.dfsVisit(v);
            }
        }

        self.color[u] = .black;
        self.timer += 1;
        self.finish[u] = self.timer;
    }

    // Iterative DFS ใช้ explicit stack
    pub fn iterativeDFS(self: *DFS, start: usize, visited: []bool) !std.ArrayList(usize) {
        var result = std.ArrayList(usize).init(self.allocator);
        var stack = std.ArrayList(usize).init(self.allocator);
        defer stack.deinit();

        try stack.append(start);

        while (stack.items.len > 0) {
            const u = stack.pop();
            if (visited[u]) continue;
            visited[u] = true;
            try result.append(u);

            // Push ย้อนกลับเพื่อให้ process ตามลำดับ
            var i: i32 = @intCast(self.adj[u].items.len);
            i -= 1;
            while (i >= 0) : (i -= 1) {
                const v = self.adj[u].items[@intCast(i)];
                if (!visited[v]) try stack.append(v);
            }
        }
        return result;
    }

    pub fn printTimestamps(self: *DFS) void {
        std.debug.print("DFS Timestamps:\n", .{});
        for (0..self.adj.len) |u| {
            std.debug.print("  vertex {d}: d={d}, f={d}\n", .{
                u, self.discovery[u], self.finish[u],
            });
        }
    }
};

// Connected Components
fn countComponents(allocator: Allocator, n: usize, edges: [][2]usize) !usize {
    var adj = try allocator.alloc(std.ArrayList(usize), n);
    defer {
        for (adj) |*list| list.deinit();
        allocator.free(adj);
    }
    for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);
    for (edges) |edge| {
        try adj[edge[0]].append(edge[1]);
        try adj[edge[1]].append(edge[0]);
    }

    var visited = try allocator.alloc(bool, n);
    defer allocator.free(visited);
    @memset(visited, false);

    var components: usize = 0;
    var stack = std.ArrayList(usize).init(allocator);
    defer stack.deinit();

    for (0..n) |i| {
        if (!visited[i]) {
            components += 1;
            try stack.append(i);
            while (stack.items.len > 0) {
                const u = stack.pop();
                if (visited[u]) continue;
                visited[u] = true;
                for (adj[u].items) |v| {
                    if (!visited[v]) try stack.append(v);
                }
            }
        }
    }
    return components;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== DFS Demo ===\n", .{});

    var dfs = try DFS.init(allocator, 6);
    defer dfs.deinit();

    try dfs.addEdge(0, 1);
    try dfs.addEdge(0, 2);
    try dfs.addEdge(1, 3);
    try dfs.addEdge(2, 4);
    try dfs.addEdge(3, 5);
    try dfs.addEdge(4, 5);

    dfs.run();
    dfs.printTimestamps();

    // Iterative DFS
    var visited = [_]bool{false} ** 6;
    var order = try dfs.iterativeDFS(0, &visited);
    defer order.deinit();

    std.debug.print("\nIterative DFS from 0: ", .{});
    for (order.items) |v| std.debug.print("{d} ", .{v});
    std.debug.print("\n", .{});

    // Connected Components
    const edges = [_][2]usize{ .{ 0, 1 }, .{ 1, 2 }, .{ 3, 4 } };
    const components = try countComponents(allocator, 5, &edges);
    std.debug.print("\nConnected Components: {d}\n", .{components});
}
```

---

## ขั้นตอนที่ 785: Topological Sort

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn topologicalSort(allocator: Allocator, n: usize, adj: []std.ArrayList(usize)) !?std.ArrayList(usize) {
    // Kahn's Algorithm (BFS-based)
    var in_degree = try allocator.alloc(usize, n);
    defer allocator.free(in_degree);
    @memset(in_degree, 0);

    for (adj) |list| {
        for (list.items) |v| in_degree[v] += 1;
    }

    var queue = std.ArrayList(usize).init(allocator);
    defer queue.deinit();

    for (0..n) |i| {
        if (in_degree[i] == 0) try queue.append(i);
    }

    var result = std.ArrayList(usize).init(allocator);
    var count: usize = 0;

    while (queue.items.len > 0) {
        const u = queue.orderedRemove(0);
        try result.append(u);
        count += 1;

        for (adj[u].items) |v| {
            in_degree[v] -= 1;
            if (in_degree[v] == 0) try queue.append(v);
        }
    }

    if (count != n) {
        result.deinit();
        return null; // มี cycle
    }
    return result;
}

// DFS-based topological sort
fn topoSortDFS(allocator: Allocator, n: usize, adj: []std.ArrayList(usize)) !?std.ArrayList(usize) {
    var visited = try allocator.alloc(bool, n);
    defer allocator.free(visited);
    var in_stack = try allocator.alloc(bool, n);
    defer allocator.free(in_stack);
    @memset(visited, false);
    @memset(in_stack, false);

    var result = std.ArrayList(usize).init(allocator);
    var has_cycle = false;

    // DFS visit function using explicit stack
    for (0..n) |start| {
        if (!visited[start]) {
            var dfs_stack = std.ArrayList(struct { node: usize, idx: usize }).init(allocator);
            defer dfs_stack.deinit();

            try dfs_stack.append(.{ .node = start, .idx = 0 });
            visited[start] = true;
            in_stack[start] = true;

            while (dfs_stack.items.len > 0) {
                const top = &dfs_stack.items[dfs_stack.items.len - 1];
                if (top.idx < adj[top.node].items.len) {
                    const v = adj[top.node].items[top.idx];
                    top.idx += 1;
                    if (!visited[v]) {
                        visited[v] = true;
                        in_stack[v] = true;
                        try dfs_stack.append(.{ .node = v, .idx = 0 });
                    } else if (in_stack[v]) {
                        has_cycle = true;
                        break;
                    }
                } else {
                    in_stack[top.node] = false;
                    try result.insert(0, top.node);
                    _ = dfs_stack.pop();
                }
            }
            if (has_cycle) break;
        }
    }

    if (has_cycle) {
        result.deinit();
        return null;
    }
    return result;
}

// Course Scheduling Problem
const Course = struct {
    id: usize,
    name: []const u8,
};

fn canFinishCourses(allocator: Allocator, num_courses: usize, prerequisites: [][2]usize) !bool {
    var adj = try allocator.alloc(std.ArrayList(usize), num_courses);
    defer {
        for (adj) |*list| list.deinit();
        allocator.free(adj);
    }
    for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);

    for (prerequisites) |prereq| {
        try adj[prereq[1]].append(prereq[0]); // prereq[1] ต้องเรียนก่อน prereq[0]
    }

    const result = try topologicalSort(allocator, num_courses, adj);
    if (result) |r| {
        r.deinit();
        return true;
    }
    return false;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Topological Sort Demo ===\n", .{});

    // Build DAG: task dependencies
    // 0: ออกแบบ, 1: พัฒนา, 2: ทดสอบ, 3: deploy, 4: เอกสาร, 5: รีวิว
    var adj = try allocator.alloc(std.ArrayList(usize), 6);
    defer {
        for (adj) |*list| list.deinit();
        allocator.free(adj);
    }
    for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);

    try adj[0].append(1); // ออกแบบ -> พัฒนา
    try adj[1].append(2); // พัฒนา -> ทดสอบ
    try adj[1].append(4); // พัฒนา -> เอกสาร
    try adj[2].append(5); // ทดสอบ -> รีวิว
    try adj[5].append(3); // รีวิว -> deploy
    try adj[4].append(3); // เอกสาร -> deploy

    const task_names = [_][]const u8{ "ออกแบบ", "พัฒนา", "ทดสอบ", "deploy", "เอกสาร", "รีวิว" };

    if (try topologicalSort(allocator, 6, adj)) |order| {
        defer order.deinit();
        std.debug.print("ลำดับการทำงาน:\n", .{});
        for (order.items, 1..) |task, i| {
            std.debug.print("  {d}. {s}\n", .{ i, task_names[task] });
        }
    }

    // Course scheduling
    std.debug.print("\n--- Course Scheduling ---\n", .{});
    const prereqs = [_][2]usize{ .{ 1, 0 }, .{ 2, 1 }, .{ 3, 2 } };
    const can_finish = try canFinishCourses(allocator, 4, &prereqs);
    std.debug.print("สามารถเรียนครบ 4 วิชาได้: {}\n", .{can_finish});

    // Test with cycle
    const prereqs_cycle = [_][2]usize{ .{ 1, 0 }, .{ 0, 1 } };
    const can_finish_cycle = try canFinishCourses(allocator, 2, &prereqs_cycle);
    std.debug.print("กรณีมี cycle สามารถเรียนได้: {}\n", .{can_finish_cycle});
}
```

---

## ขั้นตอนที่ 786: Shortest Path (Dijkstra)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

const INF: f64 = std.math.inf(f64);

const DijkstraResult = struct {
    distances: []f64,
    parents: []i32,
    allocator: Allocator,

    pub fn deinit(self: *DijkstraResult) void {
        self.allocator.free(self.distances);
        self.allocator.free(self.parents);
    }

    pub fn getPath(self: *DijkstraResult, target: usize) !std.ArrayList(usize) {
        var path = std.ArrayList(usize).init(self.allocator);
        var current: i32 = @intCast(target);
        while (current != -1) {
            try path.insert(0, @intCast(current));
            current = self.parents[@intCast(current)];
        }
        return path;
    }
};

fn dijkstra(allocator: Allocator, adj: []std.ArrayList(struct { to: usize, weight: f64 }), source: usize) !DijkstraResult {
    const n = adj.len;
    const dist = try allocator.alloc(f64, n);
    const parent = try allocator.alloc(i32, n);
    @memset(dist, INF);
    @memset(parent, -1);
    dist[source] = 0;

    // Priority Queue: (distance, vertex)
    const PQItem = struct {
        dist: f64,
        vertex: usize,

        fn lessThan(_: void, a: @This(), b: @This()) std.math.Order {
            return if (a.dist < b.dist) .lt else if (a.dist > b.dist) .gt else .eq;
        }
    };

    var pq = std.PriorityQueue(PQItem, void, PQItem.lessThan).init(allocator, {});
    defer pq.deinit();

    try pq.add(.{ .dist = 0, .vertex = source });

    while (pq.removeOrNull()) |item| {
        const u = item.vertex;
        const d = item.dist;

        if (d > dist[u]) continue; // stale entry

        for (adj[u].items) |edge| {
            const new_dist = dist[u] + edge.weight;
            if (new_dist < dist[edge.to]) {
                dist[edge.to] = new_dist;
                parent[edge.to] = @intCast(u);
                try pq.add(.{ .dist = new_dist, .vertex = edge.to });
            }
        }
    }

    return DijkstraResult{
        .distances = dist,
        .parents = parent,
        .allocator = allocator,
    };
}

// Bellman-Ford สำหรับ negative weights
const BellmanFordResult = struct {
    distances: []f64,
    has_negative_cycle: bool,
    allocator: Allocator,

    pub fn deinit(self: *BellmanFordResult) void {
        self.allocator.free(self.distances);
    }
};

const WeightedEdge = struct { from: usize, to: usize, weight: f64 };

fn bellmanFord(allocator: Allocator, n: usize, edges: []WeightedEdge, source: usize) !BellmanFordResult {
    const dist = try allocator.alloc(f64, n);
    @memset(dist, INF);
    dist[source] = 0;

    // Relax edges n-1 times
    var i: usize = 0;
    while (i < n - 1) : (i += 1) {
        for (edges) |edge| {
            if (dist[edge.from] != INF and dist[edge.from] + edge.weight < dist[edge.to]) {
                dist[edge.to] = dist[edge.from] + edge.weight;
            }
        }
    }

    // Check for negative cycles
    var has_cycle = false;
    for (edges) |edge| {
        if (dist[edge.from] != INF and dist[edge.from] + edge.weight < dist[edge.to]) {
            has_cycle = true;
            break;
        }
    }

    return BellmanFordResult{
        .distances = dist,
        .has_negative_cycle = has_cycle,
        .allocator = allocator,
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Dijkstra's Shortest Path ===\n", .{});

    const AdjEdge = struct { to: usize, weight: f64 };
    var adj = try allocator.alloc(std.ArrayList(AdjEdge), 6);
    defer {
        for (adj) |*list| list.deinit();
        allocator.free(adj);
    }
    for (adj) |*list| list.* = std.ArrayList(AdjEdge).init(allocator);

    // Bangkok road network (simplified)
    try adj[0].append(.{ .to = 1, .weight = 4 }); // BKK -> สีลม
    try adj[0].append(.{ .to = 2, .weight = 2 }); // BKK -> อโศก
    try adj[1].append(.{ .to = 2, .weight = 1 }); // สีลม -> อโศก
    try adj[1].append(.{ .to = 3, .weight = 5 }); // สีลม -> มีนบุรี
    try adj[2].append(.{ .to = 3, .weight = 8 }); // อโศก -> มีนบุรี
    try adj[2].append(.{ .to = 4, .weight = 10 }); // อโศก -> ดอนเมือง
    try adj[3].append(.{ .to = 4, .weight = 2 }); // มีนบุรี -> ดอนเมือง
    try adj[3].append(.{ .to = 5, .weight = 6 }); // มีนบุรี -> สุวรรณภูมิ
    try adj[4].append(.{ .to = 5, .weight = 3 }); // ดอนเมือง -> สุวรรณภูมิ

    const station_names = [_][]const u8{
        "กรุงเทพ", "สีลม", "อโศก", "มีนบุรี", "ดอนเมือง", "สุวรรณภูมิ"
    };

    var result = try dijkstra(allocator, adj, 0);
    defer result.deinit();

    std.debug.print("ระยะทางจากกรุงเทพ:\n", .{});
    for (result.distances, 0..) |d, i| {
        std.debug.print("  -> {s}: {d:.1} km\n", .{ station_names[i], d });
    }

    std.debug.print("\nเส้นทาง กรุงเทพ -> สุวรรณภูมิ:\n", .{});
    var path = try result.getPath(5);
    defer path.deinit();
    for (path.items, 0..) |v, i| {
        if (i > 0) std.debug.print(" -> ", .{});
        std.debug.print("{s}", .{station_names[v]});
    }
    std.debug.print("\n", .{});

    // Bellman-Ford
    std.debug.print("\n--- Bellman-Ford (supports negative weights) ---\n", .{});
    const bf_edges = [_]WeightedEdge{
        .{ .from = 0, .to = 1, .weight = 4 },
        .{ .from = 0, .to = 2, .weight = 5 },
        .{ .from = 1, .to = 2, .weight = -3 },
        .{ .from = 2, .to = 3, .weight = 2 },
    };

    var bf_result = try bellmanFord(allocator, 4, &bf_edges, 0);
    defer bf_result.deinit();

    std.debug.print("Has negative cycle: {}\n", .{bf_result.has_negative_cycle});
    std.debug.print("Distances: ", .{});
    for (bf_result.distances) |d| {
        if (d == INF) std.debug.print("INF ") else std.debug.print("{d:.1} ", .{d});
    }
    std.debug.print("\n", .{});
}
```

---

## ขั้นตอนที่ 787: Minimum Spanning Tree (Kruskal, Prim)

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Union-Find สำหรับ Kruskal
const UnionFind = struct {
    parent: []usize,
    rank: []usize,
    allocator: Allocator,

    pub fn init(allocator: Allocator, n: usize) !UnionFind {
        const parent = try allocator.alloc(usize, n);
        const rank = try allocator.alloc(usize, n);
        for (0..n) |i| parent[i] = i;
        @memset(rank, 0);
        return .{ .parent = parent, .rank = rank, .allocator = allocator };
    }

    pub fn deinit(self: *UnionFind) void {
        self.allocator.free(self.parent);
        self.allocator.free(self.rank);
    }

    pub fn find(self: *UnionFind, x: usize) usize {
        if (self.parent[x] != x) {
            self.parent[x] = self.find(self.parent[x]); // path compression
        }
        return self.parent[x];
    }

    pub fn union(self: *UnionFind, x: usize, y: usize) bool {
        const rx = self.find(x);
        const ry = self.find(y);
        if (rx == ry) return false;

        if (self.rank[rx] < self.rank[ry]) {
            self.parent[rx] = ry;
        } else if (self.rank[rx] > self.rank[ry]) {
            self.parent[ry] = rx;
        } else {
            self.parent[ry] = rx;
            self.rank[rx] += 1;
        }
        return true;
    }
};

const MSTEdge = struct { from: usize, to: usize, weight: f64 };

// Kruskal's Algorithm
fn kruskal(allocator: Allocator, n: usize, edges: []MSTEdge) !struct { edges: []MSTEdge, total_weight: f64 } {
    // Sort edges by weight
    var sorted_edges = try allocator.dupe(MSTEdge, edges);
    std.mem.sort(MSTEdge, sorted_edges, {}, struct {
        fn lessThan(_: void, a: MSTEdge, b: MSTEdge) bool {
            return a.weight < b.weight;
        }
    }.lessThan);

    var uf = try UnionFind.init(allocator, n);
    defer uf.deinit();

    var mst = std.ArrayList(MSTEdge).init(allocator);
    var total_weight: f64 = 0;

    for (sorted_edges) |edge| {
        if (uf.union(edge.from, edge.to)) {
            try mst.append(edge);
            total_weight += edge.weight;
            if (mst.items.len == n - 1) break;
        }
    }

    allocator.free(sorted_edges);
    return .{ .edges = try mst.toOwnedSlice(), .total_weight = total_weight };
}

// Prim's Algorithm
fn prim(allocator: Allocator, adj: []std.ArrayList(struct { to: usize, weight: f64 })) !struct { edges: []MSTEdge, total_weight: f64 } {
    const n = adj.len;
    const INF: f64 = std.math.inf(f64);

    var in_mst = try allocator.alloc(bool, n);
    defer allocator.free(in_mst);
    var key = try allocator.alloc(f64, n);
    defer allocator.free(key);
    var parent = try allocator.alloc(i32, n);
    defer allocator.free(parent);

    @memset(in_mst, false);
    @memset(key, INF);
    @memset(parent, -1);
    key[0] = 0;

    const PQItem = struct {
        key: f64,
        vertex: usize,
        fn lessThan(_: void, a: @This(), b: @This()) std.math.Order {
            return if (a.key < b.key) .lt else if (a.key > b.key) .gt else .eq;
        }
    };

    var pq = std.PriorityQueue(PQItem, void, PQItem.lessThan).init(allocator, {});
    defer pq.deinit();
    try pq.add(.{ .key = 0, .vertex = 0 });

    var total_weight: f64 = 0;
    var mst_edges = std.ArrayList(MSTEdge).init(allocator);

    while (pq.removeOrNull()) |item| {
        const u = item.vertex;
        if (in_mst[u]) continue;
        in_mst[u] = true;

        if (parent[u] != -1) {
            total_weight += key[u];
            try mst_edges.append(.{
                .from = @intCast(parent[u]),
                .to = u,
                .weight = key[u],
            });
        }

        for (adj[u].items) |edge| {
            if (!in_mst[edge.to] and edge.weight < key[edge.to]) {
                key[edge.to] = edge.weight;
                parent[edge.to] = @intCast(u);
                try pq.add(.{ .key = edge.weight, .vertex = edge.to });
            }
        }
    }

    return .{ .edges = try mst_edges.toOwnedSlice(), .total_weight = total_weight };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Minimum Spanning Tree ===\n", .{});

    const edges = [_]MSTEdge{
        .{ .from = 0, .to = 1, .weight = 4 },
        .{ .from = 0, .to = 2, .weight = 3 },
        .{ .from = 1, .to = 2, .weight = 1 },
        .{ .from = 1, .to = 3, .weight = 2 },
        .{ .from = 2, .to = 3, .weight = 4 },
        .{ .from = 3, .to = 4, .weight = 2 },
        .{ .from = 2, .to = 4, .weight = 5 },
    };

    // Kruskal
    var kruskal_result = try kruskal(allocator, 5, @constCast(&edges));
    defer allocator.free(kruskal_result.edges);

    std.debug.print("Kruskal MST:\n", .{});
    for (kruskal_result.edges) |e| {
        std.debug.print("  {d} -- {d} ({d:.0})\n", .{ e.from, e.to, e.weight });
    }
    std.debug.print("Total weight: {d:.0}\n", .{kruskal_result.total_weight});

    // Prim
    const AdjEdge = struct { to: usize, weight: f64 };
    var adj = try allocator.alloc(std.ArrayList(AdjEdge), 5);
    defer { for (adj) |*list| list.deinit(); allocator.free(adj); }
    for (adj) |*list| list.* = std.ArrayList(AdjEdge).init(allocator);

    for (edges) |e| {
        try adj[e.from].append(.{ .to = e.to, .weight = e.weight });
        try adj[e.to].append(.{ .to = e.from, .weight = e.weight });
    }

    var prim_result = try prim(allocator, adj);
    defer allocator.free(prim_result.edges);

    std.debug.print("\nPrim MST:\n", .{});
    for (prim_result.edges) |e| {
        std.debug.print("  {d} -- {d} ({d:.0})\n", .{ e.from, e.to, e.weight });
    }
    std.debug.print("Total weight: {d:.0}\n", .{prim_result.total_weight});
}
```

---

## ขั้นตอนที่ 788: Cycle Detection

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Cycle detection ใน undirected graph (Union-Find)
fn hasCycleUndirected(allocator: Allocator, n: usize, edges: [][2]usize) !bool {
    var parent = try allocator.alloc(usize, n);
    defer allocator.free(parent);
    for (0..n) |i| parent[i] = i;

    const find = struct {
        fn f(p: []usize, x: usize) usize {
            if (p[x] != x) p[x] = f(p, p[x]);
            return p[x];
        }
    }.f;

    for (edges) |edge| {
        const rx = find(parent, edge[0]);
        const ry = find(parent, edge[1]);
        if (rx == ry) return true;
        parent[rx] = ry;
    }
    return false;
}

// Cycle detection ใน directed graph (DFS)
fn hasCycleDirected(allocator: Allocator, n: usize, adj: []std.ArrayList(usize)) !bool {
    const Color = enum { white, gray, black };
    var color = try allocator.alloc(Color, n);
    defer allocator.free(color);
    @memset(color, .white);

    const dfsHasCycle = struct {
        fn f(clr: []Color, a: []std.ArrayList(usize), u: usize) bool {
            clr[u] = .gray;
            for (a[u].items) |v| {
                if (clr[v] == .gray) return true;
                if (clr[v] == .white and f(clr, a, v)) return true;
            }
            clr[u] = .black;
            return false;
        }
    }.f;

    for (0..n) |i| {
        if (color[i] == .white) {
            if (dfsHasCycle(color, adj, i)) return true;
        }
    }
    return false;
}

// Floyd's Cycle Detection สำหรับ Linked List
fn floydCycleDetect(comptime T: type, head: ?*struct { val: T, next: ?*@This() }) bool {
    var slow = head;
    var fast = head;
    while (fast != null and fast.?.next != null) {
        slow = slow.?.next;
        fast = fast.?.next.?.next;
        if (slow == fast) return true;
    }
    return false;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Cycle Detection Demo ===\n", .{});

    // Undirected graph without cycle
    const edges_no_cycle = [_][2]usize{ .{ 0, 1 }, .{ 1, 2 }, .{ 2, 3 } };
    std.debug.print("Undirected (0-1-2-3) has cycle: {}\n", .{
        try hasCycleUndirected(allocator, 4, &edges_no_cycle),
    });

    // Undirected graph with cycle
    const edges_cycle = [_][2]usize{ .{ 0, 1 }, .{ 1, 2 }, .{ 2, 0 } };
    std.debug.print("Undirected (0-1-2-0) has cycle: {}\n", .{
        try hasCycleUndirected(allocator, 3, &edges_cycle),
    });

    // Directed graph
    var adj = try allocator.alloc(std.ArrayList(usize), 4);
    defer { for (adj) |*list| list.deinit(); allocator.free(adj); }
    for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);

    try adj[0].append(1);
    try adj[1].append(2);
    try adj[2].append(3);

    std.debug.print("Directed (0->1->2->3) has cycle: {}\n", .{
        try hasCycleDirected(allocator, 4, adj),
    });

    try adj[3].append(1); // สร้าง cycle 1->2->3->1
    std.debug.print("Directed (0->1->2->3->1) has cycle: {}\n", .{
        try hasCycleDirected(allocator, 4, adj),
    });
}
```

---

## ขั้นตอนที่ 789: Strongly Connected Components

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Kosaraju's Algorithm สำหรับ SCC
fn kosarajuSCC(allocator: Allocator, n: usize, adj: []std.ArrayList(usize)) ![][]usize {
    // Step 1: DFS บน original graph, เก็บ finish order
    var visited = try allocator.alloc(bool, n);
    defer allocator.free(visited);
    @memset(visited, false);

    var finish_order = std.ArrayList(usize).init(allocator);
    defer finish_order.deinit();

    const dfs1 = struct {
        fn f(a: []std.ArrayList(usize), vis: []bool, u: usize, order: *std.ArrayList(usize)) void {
            vis[u] = true;
            for (a[u].items) |v| {
                if (!vis[v]) f(a, vis, v, order);
            }
            order.append(u) catch {};
        }
    }.f;

    for (0..n) |i| {
        if (!visited[i]) dfs1(adj, visited, i, &finish_order);
    }

    // Step 2: สร้าง reverse graph
    var rev_adj = try allocator.alloc(std.ArrayList(usize), n);
    defer { for (rev_adj) |*list| list.deinit(); allocator.free(rev_adj); }
    for (rev_adj) |*list| list.* = std.ArrayList(usize).init(allocator);

    for (0..n) |u| {
        for (adj[u].items) |v| {
            try rev_adj[v].append(u);
        }
    }

    // Step 3: DFS บน reverse graph ตาม finish order
    @memset(visited, false);
    var sccs = std.ArrayList([]usize).init(allocator);
    var current_scc = std.ArrayList(usize).init(allocator);

    const dfs2 = struct {
        fn f(a: []std.ArrayList(usize), vis: []bool, u: usize, scc: *std.ArrayList(usize)) void {
            vis[u] = true;
            scc.append(u) catch {};
            for (a[u].items) |v| {
                if (!vis[v]) f(a, vis, v, scc);
            }
        }
    }.f;

    var i: i32 = @intCast(finish_order.items.len);
    i -= 1;
    while (i >= 0) : (i -= 1) {
        const u = finish_order.items[@intCast(i)];
        if (!visited[u]) {
            current_scc.clearRetainingCapacity();
            dfs2(rev_adj, visited, u, &current_scc);
            try sccs.append(try allocator.dupe(usize, current_scc.items));
        }
    }
    current_scc.deinit();

    return sccs.toOwnedSlice();
}

// Tarjan's Algorithm
fn tarjanSCC(allocator: Allocator, n: usize, adj: []std.ArrayList(usize)) ![][]usize {
    var disc = try allocator.alloc(i32, n);
    defer allocator.free(disc);
    var low = try allocator.alloc(i32, n);
    defer allocator.free(low);
    var on_stack = try allocator.alloc(bool, n);
    defer allocator.free(on_stack);
    @memset(disc, -1);
    @memset(low, 0);
    @memset(on_stack, false);

    var stack = std.ArrayList(usize).init(allocator);
    defer stack.deinit();
    var sccs = std.ArrayList([]usize).init(allocator);
    var timer: i32 = 0;

    const dfs = struct {
        fn f(a: []std.ArrayList(usize), d: []i32, lo: []i32, os: []bool, s: *std.ArrayList(usize), result: *std.ArrayList([]usize), alloc: Allocator, u: usize, t: *i32) void {
            d[u] = t.*;
            lo[u] = t.*;
            t.* += 1;
            s.append(u) catch return;
            os[u] = true;

            for (a[u].items) |v| {
                if (d[v] == -1) {
                    f(a, d, lo, os, s, result, alloc, v, t);
                    lo[u] = @min(lo[u], lo[v]);
                } else if (os[v]) {
                    lo[u] = @min(lo[u], d[v]);
                }
            }

            if (lo[u] == d[u]) {
                var scc = std.ArrayList(usize).init(alloc);
                while (true) {
                    const v = s.pop();
                    os[v] = false;
                    scc.append(v) catch break;
                    if (v == u) break;
                }
                result.append(scc.toOwnedSlice() catch return) catch {};
            }
        }
    }.f;

    for (0..n) |i| {
        if (disc[i] == -1) {
            dfs(adj, disc, low, on_stack, &stack, &sccs, allocator, i, &timer);
        }
    }

    return sccs.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Strongly Connected Components ===\n", .{});

    // สร้าง directed graph
    var adj = try allocator.alloc(std.ArrayList(usize), 8);
    defer { for (adj) |*list| list.deinit(); allocator.free(adj); }
    for (adj) |*list| list.* = std.ArrayList(usize).init(allocator);

    // SCC1: {0, 1, 2}, SCC2: {3}, SCC3: {4, 5, 6}, SCC4: {7}
    try adj[0].append(1);
    try adj[1].append(2);
    try adj[2].append(0);
    try adj[1].append(3);
    try adj[3].append(4);
    try adj[4].append(5);
    try adj[5].append(6);
    try adj[6].append(4);
    try adj[3].append(7);

    // Kosaraju
    const sccs = try kosarajuSCC(allocator, 8, adj);
    defer {
        for (sccs) |scc| allocator.free(scc);
        allocator.free(sccs);
    }

    std.debug.print("Strongly Connected Components (Kosaraju):\n", .{});
    for (sccs, 1..) |scc, i| {
        std.debug.print("  SCC {d}: ", .{i});
        for (scc) |v| std.debug.print("{d} ", .{v});
        std.debug.print("\n", .{});
    }
}
```

---

## ขั้นตอนที่ 790: Real-world Graph Usage

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

// Social Network Graph
const Person = struct {
    id: u32,
    name: []const u8,
};

const SocialNetwork = struct {
    allocator: Allocator,
    people: std.ArrayList(Person),
    adjacency: std.AutoHashMap(u32, std.ArrayList(u32)),

    pub fn init(allocator: Allocator) SocialNetwork {
        return .{
            .allocator = allocator,
            .people = std.ArrayList(Person).init(allocator),
            .adjacency = std.AutoHashMap(u32, std.ArrayList(u32)).init(allocator),
        };
    }

    pub fn deinit(self: *SocialNetwork) void {
        self.people.deinit();
        var iter = self.adjacency.valueIterator();
        while (iter.next()) |list| list.deinit();
        self.adjacency.deinit();
    }

    pub fn addPerson(self: *SocialNetwork, id: u32, name: []const u8) !void {
        try self.people.append(.{ .id = id, .name = name });
        try self.adjacency.put(id, std.ArrayList(u32).init(self.allocator));
    }

    pub fn addFriendship(self: *SocialNetwork, a: u32, b: u32) !void {
        if (self.adjacency.getPtr(a)) |list| try list.append(b);
        if (self.adjacency.getPtr(b)) |list| try list.append(a);
    }

    // หาเพื่อนที่ 2 ระดับ (friend of friends)
    pub fn getFriendsOfFriends(self: *SocialNetwork, allocator: Allocator, person_id: u32) !std.ArrayList(u32) {
        var result = std.AutoHashMap(u32, void).init(allocator);
        defer result.deinit();

        const direct_friends = self.adjacency.get(person_id) orelse return std.ArrayList(u32).init(allocator);

        for (direct_friends.items) |friend_id| {
            try result.put(friend_id, {});
            const fof = self.adjacency.get(friend_id) orelse continue;
            for (fof.items) |fof_id| {
                if (fof_id != person_id) {
                    try result.put(fof_id, {});
                }
            }
        }

        // ลบ direct friends
        for (direct_friends.items) |friend_id| {
            _ = result.remove(friend_id);
        }
        _ = result.remove(person_id);

        var list = std.ArrayList(u32).init(allocator);
        var iter = result.keyIterator();
        while (iter.next()) |id| try list.append(id.*);
        return list;
    }

    // หาระยะห่างระหว่างคน (degrees of separation)
    pub fn degreesOfSeparation(self: *SocialNetwork, allocator: Allocator, from: u32, to: u32) !i32 {
        var visited = std.AutoHashMap(u32, i32).init(allocator);
        defer visited.deinit();

        var queue = std.ArrayList(u32).init(allocator);
        defer queue.deinit();

        try visited.put(from, 0);
        try queue.append(from);

        while (queue.items.len > 0) {
            const current = queue.orderedRemove(0);
            const dist = visited.get(current).?;

            if (current == to) return dist;

            const friends = self.adjacency.get(current) orelse continue;
            for (friends.items) |friend| {
                if (!visited.contains(friend)) {
                    try visited.put(friend, dist + 1);
                    try queue.append(friend);
                }
            }
        }
        return -1; // ไม่เชื่อมต่อกัน
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    std.debug.print("=== Real-world Graph: Social Network ===\n", .{});

    var network = SocialNetwork.init(allocator);
    defer network.deinit();

    try network.addPerson(1, "สมชาย");
    try network.addPerson(2, "สมหญิง");
    try network.addPerson(3, "สมศักดิ์");
    try network.addPerson(4, "สมฤทัย");
    try network.addPerson(5, "สมประสงค์");

    try network.addFriendship(1, 2);
    try network.addFriendship(1, 3);
    try network.addFriendship(2, 4);
    try network.addFriendship(3, 4);
    try network.addFriendship(4, 5);

    // Friends of friends
    var fof = try network.getFriendsOfFriends(allocator, 1);
    defer fof.deinit();
    std.debug.print("เพื่อนระดับ 2 ของสมชาย: ", .{});
    for (fof.items) |id| {
        for (network.people.items) |p| {
            if (p.id == id) std.debug.print("{s} ", .{p.name});
        }
    }
    std.debug.print("\n", .{});

    // Degrees of separation
    const deg = try network.degreesOfSeparation(allocator, 1, 5);
    std.debug.print("ระยะห่างระหว่างสมชายและสมประสงค์: {d}\n", .{deg});
    const deg2 = try network.degreesOfSeparation(allocator, 1, 4);
    std.debug.print("ระยะห่างระหว่างสมชายและสมฤทัย: {d}\n", .{deg2});
}
```

---

## สรุปส่วนที่ 79: Graph Algorithms

| Algorithm | Complexity | ใช้ทำอะไร |
|-----------|------------|----------|
| BFS | O(V+E) | Shortest path (unweighted), level traversal |
| DFS | O(V+E) | Cycle detection, connectivity |
| Topological Sort | O(V+E) | Task scheduling, dependency resolution |
| Dijkstra | O((V+E) log V) | Shortest path (non-negative weights) |
| Bellman-Ford | O(VE) | Shortest path (negative weights) |
| Floyd-Warshall | O(V³) | All-pairs shortest path |
| Kruskal | O(E log E) | Minimum spanning tree |
| Prim | O((V+E) log V) | Minimum spanning tree |
| Kosaraju/Tarjan | O(V+E) | Strongly connected components |

---

*ส่วนถัดไป: Part 80 - Data Structures: Hash Table*
