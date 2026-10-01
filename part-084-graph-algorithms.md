# Part 84: Algorithms: Graph Algorithms
## Steps 831-840

---

## Step 831: Graph Traversal Review (BFS/DFS)

### Graph คืออะไร?

Graph เป็นโครงสร้างข้อมูลที่ประกอบด้วย Vertices (nodes) และ Edges (connections)

**ประเภทของ Graph:**
- Directed/Undirected
- Weighted/Unweighted
- Cyclic/Acyclic (DAG)

```zig
// graph_traversal.zig
const std = @import("std");

/// Graph แบบ Adjacency List
pub const Graph = struct {
    adj: std.ArrayList(std.ArrayList(usize)),
    n: usize, // จำนวน vertices
    allocator: std.mem.Allocator,
    
    pub fn init(n: usize, allocator: std.mem.Allocator) !Graph {
        var adj = std.ArrayList(std.ArrayList(usize)).init(allocator);
        for (0..n) |_| {
            try adj.append(std.ArrayList(usize).init(allocator));
        }
        return .{ .adj = adj, .n = n, .allocator = allocator };
    }
    
    pub fn deinit(self: *Graph) void {
        for (self.adj.items) |*list| list.deinit();
        self.adj.deinit();
    }
    
    pub fn addEdge(self: *Graph, u: usize, v: usize) !void {
        try self.adj.items[u].append(v);
        try self.adj.items[v].append(u); // undirected
    }
    
    pub fn addDirectedEdge(self: *Graph, u: usize, v: usize) !void {
        try self.adj.items[u].append(v);
    }
    
    /// BFS - Breadth-First Search
    pub fn bfs(self: *Graph, start: usize) ![]usize {
        var visited = try self.allocator.alloc(bool, self.n);
        defer self.allocator.free(visited);
        @memset(visited, false);
        
        var order = std.ArrayList(usize).init(self.allocator);
        errdefer order.deinit();
        
        var queue = std.ArrayList(usize).init(self.allocator);
        defer queue.deinit();
        
        visited[start] = true;
        try queue.append(start);
        
        var head: usize = 0;
        while (head < queue.items.len) : (head += 1) {
            const v = queue.items[head];
            try order.append(v);
            
            for (self.adj.items[v].items) |neighbor| {
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    try queue.append(neighbor);
                }
            }
        }
        
        return order.toOwnedSlice();
    }
    
    /// DFS - Depth-First Search (iterative)
    pub fn dfs(self: *Graph, start: usize) ![]usize {
        var visited = try self.allocator.alloc(bool, self.n);
        defer self.allocator.free(visited);
        @memset(visited, false);
        
        var order = std.ArrayList(usize).init(self.allocator);
        errdefer order.deinit();
        
        var stack = std.ArrayList(usize).init(self.allocator);
        defer stack.deinit();
        
        try stack.append(start);
        
        while (stack.items.len > 0) {
            const v = stack.pop();
            
            if (visited[v]) continue;
            visited[v] = true;
            try order.append(v);
            
            // เพิ่ม neighbors ใน stack (reverse order เพื่อให้ผลเหมือน recursive DFS)
            var i = self.adj.items[v].items.len;
            while (i > 0) : (i -= 1) {
                const neighbor = self.adj.items[v].items[i - 1];
                if (!visited[neighbor]) {
                    try stack.append(neighbor);
                }
            }
        }
        
        return order.toOwnedSlice();
    }
    
    /// DFS recursive
    fn dfsRecHelper(self: *Graph, v: usize, visited: []bool, order: *std.ArrayList(usize)) !void {
        visited[v] = true;
        try order.append(v);
        
        for (self.adj.items[v].items) |neighbor| {
            if (!visited[neighbor]) {
                try self.dfsRecHelper(neighbor, visited, order);
            }
        }
    }
    
    pub fn dfsRecursive(self: *Graph, start: usize) ![]usize {
        var visited = try self.allocator.alloc(bool, self.n);
        defer self.allocator.free(visited);
        @memset(visited, false);
        
        var order = std.ArrayList(usize).init(self.allocator);
        errdefer order.deinit();
        
        try self.dfsRecHelper(start, visited, &order);
        
        return order.toOwnedSlice();
    }
    
    /// ตรวจสอบ connectivity
    pub fn isConnected(self: *Graph) !bool {
        const order = try self.bfs(0);
        defer self.allocator.free(order);
        return order.len == self.n;
    }
    
    /// หา shortest path (unweighted) ด้วย BFS
    pub fn shortestPath(self: *Graph, src: usize, dst: usize) ![]usize {
        var dist = try self.allocator.alloc(i64, self.n);
        defer self.allocator.free(dist);
        var parent = try self.allocator.alloc(i64, self.n);
        defer self.allocator.free(parent);
        
        @memset(dist, -1);
        for (parent) |*p| p.* = -1;
        
        dist[src] = 0;
        
        var queue = std.ArrayList(usize).init(self.allocator);
        defer queue.deinit();
        try queue.append(src);
        
        var head: usize = 0;
        while (head < queue.items.len) : (head += 1) {
            const v = queue.items[head];
            if (v == dst) break;
            
            for (self.adj.items[v].items) |neighbor| {
                if (dist[neighbor] == -1) {
                    dist[neighbor] = dist[v] + 1;
                    parent[neighbor] = @intCast(v);
                    try queue.append(neighbor);
                }
            }
        }
        
        if (dist[dst] == -1) return &[_]usize{};
        
        // Backtrack path
        var path = std.ArrayList(usize).init(self.allocator);
        errdefer path.deinit();
        
        var curr: i64 = @intCast(dst);
        while (curr != -1) {
            try path.append(@intCast(curr));
            curr = parent[@intCast(curr)];
        }
        
        std.mem.reverse(usize, path.items);
        return path.toOwnedSlice();
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Graph Traversal ===\n\n", .{});
    
    var g = try Graph.init(6, allocator);
    defer g.deinit();
    
    try g.addEdge(0, 1);
    try g.addEdge(0, 2);
    try g.addEdge(1, 3);
    try g.addEdge(2, 4);
    try g.addEdge(3, 5);
    try g.addEdge(4, 5);
    
    //   0
    //  / \
    // 1   2
    // |   |
    // 3   4
    //  \ /
    //   5
    
    const bfsOrder = try g.bfs(0);
    defer allocator.free(bfsOrder);
    
    try stdout.print("BFS จาก 0: ", .{});
    for (bfsOrder) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    const dfsOrder = try g.dfs(0);
    defer allocator.free(dfsOrder);
    
    try stdout.print("DFS จาก 0: ", .{});
    for (dfsOrder) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    const path = try g.shortestPath(0, 5);
    defer allocator.free(path);
    
    try stdout.print("Shortest path 0->5: ", .{});
    for (path) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n", .{});
    
    const connected = try g.isConnected();
    try stdout.print("Connected: {}\n", .{connected});
}
```

---

## Step 832: Dijkstra's Algorithm

### Dijkstra คืออะไร?

Dijkstra หาเส้นทางสั้นที่สุดจาก source vertex ไปยัง vertices อื่นทั้งหมดใน weighted graph (น้ำหนักบวก)

**Complexity:**
- Time: O((V + E) log V) ด้วย priority queue
- Space: O(V)

```zig
// dijkstra.zig
const std = @import("std");

const INF = std.math.maxInt(u64);

pub const WeightedGraph = struct {
    const Edge = struct { to: usize, weight: u64 };
    
    adj: std.ArrayList(std.ArrayList(Edge)),
    n: usize,
    allocator: std.mem.Allocator,
    
    pub fn init(n: usize, allocator: std.mem.Allocator) !WeightedGraph {
        var adj = std.ArrayList(std.ArrayList(Edge)).init(allocator);
        for (0..n) |_| {
            try adj.append(std.ArrayList(Edge).init(allocator));
        }
        return .{ .adj = adj, .n = n, .allocator = allocator };
    }
    
    pub fn deinit(self: *WeightedGraph) void {
        for (self.adj.items) |*list| list.deinit();
        self.adj.deinit();
    }
    
    pub fn addEdge(self: *WeightedGraph, u: usize, v: usize, w: u64) !void {
        try self.adj.items[u].append(.{ .to = v, .weight = w });
        try self.adj.items[v].append(.{ .to = u, .weight = w });
    }
    
    pub fn addDirectedEdge(self: *WeightedGraph, u: usize, v: usize, w: u64) !void {
        try self.adj.items[u].append(.{ .to = v, .weight = w });
    }
    
    /// Dijkstra's Algorithm
    pub fn dijkstra(self: *WeightedGraph, src: usize) ![]u64 {
        var dist = try self.allocator.alloc(u64, self.n);
        @memset(dist, INF);
        dist[src] = 0;
        
        // Priority Queue (min-heap simulation)
        const PQItem = struct { dist: u64, vertex: usize };
        var pq = std.ArrayList(PQItem).init(self.allocator);
        defer pq.deinit();
        
        try pq.append(.{ .dist = 0, .vertex = src });
        
        while (pq.items.len > 0) {
            // หา minimum distance item
            var minIdx: usize = 0;
            for (pq.items, 0..) |item, i| {
                if (item.dist < pq.items[minIdx].dist) minIdx = i;
            }
            const curr = pq.items[minIdx];
            _ = pq.swapRemove(minIdx);
            
            if (curr.dist > dist[curr.vertex]) continue; // outdated entry
            
            for (self.adj.items[curr.vertex].items) |edge| {
                const newDist = dist[curr.vertex] + edge.weight;
                if (newDist < dist[edge.to]) {
                    dist[edge.to] = newDist;
                    try pq.append(.{ .dist = newDist, .vertex = edge.to });
                }
            }
        }
        
        return dist;
    }
    
    /// Dijkstra พร้อม path reconstruction
    pub fn dijkstraPath(self: *WeightedGraph, src: usize, dst: usize) !struct {
        dist: u64,
        path: []usize,
    } {
        var dist = try self.allocator.alloc(u64, self.n);
        defer self.allocator.free(dist);
        @memset(dist, INF);
        dist[src] = 0;
        
        var parent = try self.allocator.alloc(i64, self.n);
        defer self.allocator.free(parent);
        for (parent) |*p| p.* = -1;
        
        const PQItem = struct { dist: u64, vertex: usize };
        var pq = std.ArrayList(PQItem).init(self.allocator);
        defer pq.deinit();
        
        try pq.append(.{ .dist = 0, .vertex = src });
        
        while (pq.items.len > 0) {
            var minIdx: usize = 0;
            for (pq.items, 0..) |item, i| {
                if (item.dist < pq.items[minIdx].dist) minIdx = i;
            }
            const curr = pq.items[minIdx];
            _ = pq.swapRemove(minIdx);
            
            if (curr.vertex == dst) break;
            if (curr.dist > dist[curr.vertex]) continue;
            
            for (self.adj.items[curr.vertex].items) |edge| {
                const newDist = dist[curr.vertex] + edge.weight;
                if (newDist < dist[edge.to]) {
                    dist[edge.to] = newDist;
                    parent[edge.to] = @intCast(curr.vertex);
                    try pq.append(.{ .dist = newDist, .vertex = edge.to });
                }
            }
        }
        
        // Reconstruct path
        var path = std.ArrayList(usize).init(self.allocator);
        errdefer path.deinit();
        
        if (dist[dst] != INF) {
            var curr: i64 = @intCast(dst);
            while (curr != -1) {
                try path.append(@intCast(curr));
                curr = parent[@intCast(curr)];
            }
            std.mem.reverse(usize, path.items);
        }
        
        return .{ .dist = dist[dst], .path = try path.toOwnedSlice() };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Dijkstra's Algorithm ===\n\n", .{});
    
    var g = try WeightedGraph.init(5, allocator);
    defer g.deinit();
    
    // กราฟตัวอย่าง:
    // 0 --4-- 1 --8-- 2
    // |       |       |
    // 8       11      2
    // |       |       |
    // 7 --9-- 6 --4-- 5
    
    try g.addEdge(0, 1, 4);
    try g.addEdge(0, 2, 8);
    try g.addEdge(1, 2, 11);
    try g.addEdge(1, 3, 7);
    try g.addEdge(2, 4, 2);
    try g.addEdge(3, 4, 9);
    
    const distances = try g.dijkstra(0);
    defer allocator.free(distances);
    
    try stdout.print("ระยะทางจาก vertex 0:\n", .{});
    for (distances, 0..) |d, v| {
        if (d == INF) {
            try stdout.print("  0 -> {d}: ไม่สามารถถึงได้\n", .{v});
        } else {
            try stdout.print("  0 -> {d}: {d}\n", .{ v, d });
        }
    }
    
    // หา path
    const result = try g.dijkstraPath(0, 4);
    defer allocator.free(result.path);
    
    try stdout.print("\nเส้นทาง 0 -> 4:\n", .{});
    try stdout.print("  Path: ", .{});
    for (result.path) |v| try stdout.print("{d} ", .{v});
    try stdout.print("\n  Distance: {d}\n", .{result.dist});
}
```

---

## Step 833: A* Pathfinding

### A* Algorithm คืออะไร?

A* เป็นการผสมผสาน Dijkstra กับ heuristic function เพื่อหาเส้นทางสั้นที่สุดอย่างมีประสิทธิภาพมากขึ้น

```zig
// astar.zig
const std = @import("std");

const INF = std.math.maxInt(u64);

/// Grid สำหรับ A* pathfinding
pub const Grid = struct {
    width: usize,
    height: usize,
    walls: [][]bool,
    allocator: std.mem.Allocator,
    
    pub const Point = struct { x: usize, y: usize };
    
    pub fn init(width: usize, height: usize, allocator: std.mem.Allocator) !Grid {
        const walls = try allocator.alloc([]bool, height);
        for (walls) |*row| {
            row.* = try allocator.alloc(bool, width);
            @memset(row.*, false);
        }
        return .{ .width = width, .height = height, .walls = walls, .allocator = allocator };
    }
    
    pub fn deinit(self: *Grid) void {
        for (self.walls) |row| self.allocator.free(row);
        self.allocator.free(self.walls);
    }
    
    pub fn setWall(self: *Grid, x: usize, y: usize) void {
        self.walls[y][x] = true;
    }
    
    /// Manhattan distance heuristic
    fn heuristic(a: Point, b: Point) u64 {
        const dx = if (a.x > b.x) a.x - b.x else b.x - a.x;
        const dy = if (a.y > b.y) a.y - b.y else b.y - a.y;
        return dx + dy;
    }
    
    /// A* algorithm
    pub fn aStar(self: *Grid, start: Point, goal: Point) !?[]Point {
        const n = self.width * self.height;
        
        const gScore = try self.allocator.alloc(u64, n);
        defer self.allocator.free(gScore);
        @memset(gScore, INF);
        
        const fScore = try self.allocator.alloc(u64, n);
        defer self.allocator.free(fScore);
        @memset(fScore, INF);
        
        const parent = try self.allocator.alloc(i64, n);
        defer self.allocator.free(parent);
        for (parent) |*p| p.* = -1;
        
        const idx = start.y * self.width + start.x;
        gScore[idx] = 0;
        fScore[idx] = heuristic(start, goal);
        
        // Open set เป็น priority queue แบบ simple
        const PQItem = struct { f: u64, point: Point };
        var openSet = std.ArrayList(PQItem).init(self.allocator);
        defer openSet.deinit();
        
        var inOpen = try self.allocator.alloc(bool, n);
        defer self.allocator.free(inOpen);
        @memset(inOpen, false);
        
        try openSet.append(.{ .f = fScore[idx], .point = start });
        inOpen[idx] = true;
        
        const dirs = [_][2]i32{ .{ 0, -1 }, .{ 0, 1 }, .{ -1, 0 }, .{ 1, 0 } };
        
        while (openSet.items.len > 0) {
            // หา node ที่ fScore น้อยที่สุด
            var minIdx: usize = 0;
            for (openSet.items, 0..) |item, i| {
                if (item.f < openSet.items[minIdx].f) minIdx = i;
            }
            const curr = openSet.items[minIdx].point;
            _ = openSet.swapRemove(minIdx);
            
            if (curr.x == goal.x and curr.y == goal.y) {
                // Reconstruct path
                var path = std.ArrayList(Point).init(self.allocator);
                errdefer path.deinit();
                
                var currIdx: i64 = @intCast(curr.y * self.width + curr.x);
                while (currIdx != -1) {
                    const ci: usize = @intCast(currIdx);
                    try path.append(.{
                        .x = ci % self.width,
                        .y = ci / self.width,
                    });
                    currIdx = parent[ci];
                }
                
                std.mem.reverse(Point, path.items);
                return try path.toOwnedSlice();
            }
            
            const currI = curr.y * self.width + curr.x;
            inOpen[currI] = false;
            
            for (dirs) |dir| {
                const nx = @as(i64, @intCast(curr.x)) + dir[0];
                const ny = @as(i64, @intCast(curr.y)) + dir[1];
                
                if (nx < 0 or nx >= @as(i64, @intCast(self.width))) continue;
                if (ny < 0 or ny >= @as(i64, @intCast(self.height))) continue;
                
                const neighbor = Point{ .x = @intCast(nx), .y = @intCast(ny) };
                if (self.walls[neighbor.y][neighbor.x]) continue;
                
                const nIdx = neighbor.y * self.width + neighbor.x;
                const tentativeG = gScore[currI] + 1;
                
                if (tentativeG < gScore[nIdx]) {
                    parent[nIdx] = @intCast(currI);
                    gScore[nIdx] = tentativeG;
                    fScore[nIdx] = tentativeG + heuristic(neighbor, goal);
                    
                    if (!inOpen[nIdx]) {
                        try openSet.append(.{ .f = fScore[nIdx], .point = neighbor });
                        inOpen[nIdx] = true;
                    }
                }
            }
        }
        
        return null; // ไม่พบเส้นทาง
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== A* Pathfinding ===\n\n", .{});
    
    var grid = try Grid.init(10, 10, allocator);
    defer grid.deinit();
    
    // วาง walls
    grid.setWall(3, 1);
    grid.setWall(3, 2);
    grid.setWall(3, 3);
    grid.setWall(3, 4);
    grid.setWall(3, 5);
    grid.setWall(5, 4);
    grid.setWall(5, 5);
    grid.setWall(5, 6);
    grid.setWall(5, 7);
    
    const start = Grid.Point{ .x = 0, .y = 0 };
    const goal = Grid.Point{ .x = 9, .y = 9 };
    
    if (try grid.aStar(start, goal)) |path| {
        defer allocator.free(path);
        
        try stdout.print("พบเส้นทาง! ความยาว: {d}\n", .{path.len});
        try stdout.print("Path: ", .{});
        for (path) |p| try stdout.print("({d},{d}) ", .{ p.x, p.y });
        try stdout.print("\n\n", .{});
        
        // แสดง grid
        try stdout.print("Grid (S=start, G=goal, #=wall, .=path, ' '=empty):\n", .{});
        for (0..10) |y| {
            for (0..10) |x| {
                var onPath = false;
                for (path) |p| {
                    if (p.x == x and p.y == y) { onPath = true; break; }
                }
                
                if (x == start.x and y == start.y) {
                    try stdout.print("S", .{});
                } else if (x == goal.x and y == goal.y) {
                    try stdout.print("G", .{});
                } else if (grid.walls[y][x]) {
                    try stdout.print("#", .{});
                } else if (onPath) {
                    try stdout.print(".", .{});
                } else {
                    try stdout.print(" ", .{});
                }
            }
            try stdout.print("\n", .{});
        }
    } else {
        try stdout.print("ไม่พบเส้นทาง\n", .{});
    }
}
```

---

## Step 834: Bellman-Ford Algorithm

### Bellman-Ford คืออะไร?

Bellman-Ford หาเส้นทางสั้นที่สุดในกราฟที่อาจมี negative weights และตรวจสอบ negative cycles

**Complexity:**
- Time: O(VE)
- Space: O(V)

```zig
// bellman_ford.zig
const std = @import("std");

const INF: i64 = std.math.maxInt(i64) / 2;

pub const WGraph = struct {
    const Edge = struct { from: usize, to: usize, weight: i64 };
    
    edges: std.ArrayList(Edge),
    n: usize,
    allocator: std.mem.Allocator,
    
    pub fn init(n: usize, allocator: std.mem.Allocator) WGraph {
        return .{
            .edges = std.ArrayList(Edge).init(allocator),
            .n = n,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *WGraph) void {
        self.edges.deinit();
    }
    
    pub fn addEdge(self: *WGraph, from: usize, to: usize, weight: i64) !void {
        try self.edges.append(.{ .from = from, .to = to, .weight = weight });
    }
    
    /// Bellman-Ford Algorithm
    pub fn bellmanFord(self: *WGraph, src: usize) !struct {
        dist: []i64,
        hasNegCycle: bool,
    } {
        var dist = try self.allocator.alloc(i64, self.n);
        @memset(dist, INF);
        dist[src] = 0;
        
        // Relax edges V-1 ครั้ง
        for (0..self.n - 1) |_| {
            for (self.edges.items) |edge| {
                if (dist[edge.from] != INF) {
                    const newDist = dist[edge.from] + edge.weight;
                    if (newDist < dist[edge.to]) {
                        dist[edge.to] = newDist;
                    }
                }
            }
        }
        
        // ตรวจสอบ negative cycle
        var hasNegCycle = false;
        for (self.edges.items) |edge| {
            if (dist[edge.from] != INF and dist[edge.from] + edge.weight < dist[edge.to]) {
                hasNegCycle = true;
                break;
            }
        }
        
        return .{ .dist = dist, .hasNegCycle = hasNegCycle };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Bellman-Ford Algorithm ===\n\n", .{});
    
    var g = WGraph.init(5, allocator);
    defer g.deinit();
    
    // กราฟที่มี negative weights
    try g.addEdge(0, 1, -1);
    try g.addEdge(0, 2, 4);
    try g.addEdge(1, 2, 3);
    try g.addEdge(1, 3, 2);
    try g.addEdge(1, 4, 2);
    try g.addEdge(3, 2, 5);
    try g.addEdge(3, 1, 1);
    try g.addEdge(4, 3, -3);
    
    const result = try g.bellmanFord(0);
    defer allocator.free(result.dist);
    
    if (result.hasNegCycle) {
        try stdout.print("พบ Negative Cycle!\n", .{});
    } else {
        try stdout.print("ระยะทางสั้นที่สุดจาก vertex 0:\n", .{});
        for (result.dist, 0..) |d, v| {
            if (d == INF) {
                try stdout.print("  0 -> {d}: ไม่สามารถถึงได้\n", .{v});
            } else {
                try stdout.print("  0 -> {d}: {d}\n", .{ v, d });
            }
        }
    }
    
    // ทดสอบกราฟที่มี negative cycle
    try stdout.print("\nทดสอบ Negative Cycle:\n", .{});
    var g2 = WGraph.init(3, allocator);
    defer g2.deinit();
    
    try g2.addEdge(0, 1, 1);
    try g2.addEdge(1, 2, -3);
    try g2.addEdge(2, 0, 1);
    
    const result2 = try g2.bellmanFord(0);
    defer allocator.free(result2.dist);
    
    try stdout.print("Has negative cycle: {}\n", .{result2.hasNegCycle});
}
```

---

## Step 835: Floyd-Warshall Algorithm

### Floyd-Warshall คืออะไร?

Floyd-Warshall หาระยะทางสั้นที่สุดระหว่างทุกคู่ของ vertices

**Complexity:**
- Time: O(V³)
- Space: O(V²)

```zig
// floyd_warshall.zig
const std = @import("std");

const INF: i64 = std.math.maxInt(i64) / 2;

/// Floyd-Warshall Algorithm
pub fn floydWarshall(
    n: usize,
    dist: [][]i64,
) bool {
    // Initialize diagonal
    for (0..n) |i| dist[i][i] = 0;
    
    for (0..n) |k| {
        for (0..n) |i| {
            for (0..n) |j| {
                if (dist[i][k] != INF and dist[k][j] != INF) {
                    if (dist[i][k] + dist[k][j] < dist[i][j]) {
                        dist[i][j] = dist[i][k] + dist[k][j];
                    }
                }
            }
        }
    }
    
    // ตรวจสอบ negative cycle
    for (0..n) |i| {
        if (dist[i][i] < 0) return true;
    }
    return false;
}

/// Transitive Closure ด้วย Floyd-Warshall
pub fn transitiveClosure(
    n: usize,
    reach: [][]bool,
) void {
    for (0..n) |k| {
        for (0..n) |i| {
            for (0..n) |j| {
                reach[i][j] = reach[i][j] or (reach[i][k] and reach[k][j]);
            }
        }
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Floyd-Warshall Algorithm ===\n\n", .{});
    
    const n: usize = 4;
    
    // สร้าง distance matrix
    const dist = try allocator.alloc([]i64, n);
    defer {
        for (dist) |row| allocator.free(row);
        allocator.free(dist);
    }
    
    for (dist) |*row| {
        row.* = try allocator.alloc(i64, n);
        @memset(row.*, INF);
    }
    
    // กำหนด edges
    dist[0][1] = 5;
    dist[0][3] = 10;
    dist[1][2] = 3;
    dist[2][3] = 1;
    
    try stdout.print("Distance matrix ก่อน Floyd-Warshall:\n", .{});
    for (dist) |row| {
        for (row) |d| {
            if (d == INF) {
                try stdout.print("INF\t", .{});
            } else {
                try stdout.print("{d}\t", .{d});
            }
        }
        try stdout.print("\n", .{});
    }
    
    const hasNegCycle = floydWarshall(n, dist);
    
    if (hasNegCycle) {
        try stdout.print("\nพบ Negative Cycle!\n", .{});
    } else {
        try stdout.print("\nDistance matrix หลัง Floyd-Warshall:\n", .{});
        for (dist) |row| {
            for (row) |d| {
                if (d == INF) {
                    try stdout.print("INF\t", .{});
                } else {
                    try stdout.print("{d}\t", .{d});
                }
            }
            try stdout.print("\n", .{});
        }
    }
}
```

---

## Step 836: Kruskal's MST

### Kruskal's Algorithm คืออะไร?

Kruskal หา Minimum Spanning Tree โดยเรียงลำดับ edges ตาม weight แล้วเพิ่ม edges ที่ไม่ทำให้เกิด cycle

```zig
// kruskal.zig
const std = @import("std");

/// Union-Find (Disjoint Set Union)
pub const DSU = struct {
    parent: []usize,
    rank: []usize,
    allocator: std.mem.Allocator,
    
    pub fn init(n: usize, allocator: std.mem.Allocator) !DSU {
        const parent = try allocator.alloc(usize, n);
        const rank = try allocator.alloc(usize, n);
        for (parent, 0..) |*p, i| p.* = i;
        @memset(rank, 0);
        return .{ .parent = parent, .rank = rank, .allocator = allocator };
    }
    
    pub fn deinit(self: *DSU) void {
        self.allocator.free(self.parent);
        self.allocator.free(self.rank);
    }
    
    pub fn find(self: *DSU, x: usize) usize {
        if (self.parent[x] != x) {
            self.parent[x] = self.find(self.parent[x]); // path compression
        }
        return self.parent[x];
    }
    
    pub fn union(self: *DSU, x: usize, y: usize) bool {
        const px = self.find(x);
        const py = self.find(y);
        if (px == py) return false;
        
        // union by rank
        if (self.rank[px] < self.rank[py]) {
            self.parent[px] = py;
        } else if (self.rank[px] > self.rank[py]) {
            self.parent[py] = px;
        } else {
            self.parent[py] = px;
            self.rank[px] += 1;
        }
        return true;
    }
};

const Edge = struct { u: usize, v: usize, weight: u64 };

/// Kruskal's MST
pub fn kruskal(n: usize, edges: []Edge, allocator: std.mem.Allocator) !struct {
    mstEdges: []Edge,
    totalWeight: u64,
} {
    // เรียงลำดับ edges ตาม weight
    var sortedEdges = try allocator.dupe(Edge, edges);
    errdefer allocator.free(sortedEdges);
    
    for (0..sortedEdges.len) |i| {
        for (i + 1..sortedEdges.len) |j| {
            if (sortedEdges[j].weight < sortedEdges[i].weight) {
                const temp = sortedEdges[i];
                sortedEdges[i] = sortedEdges[j];
                sortedEdges[j] = temp;
            }
        }
    }
    
    var dsu = try DSU.init(n, allocator);
    defer dsu.deinit();
    
    var mstEdges = std.ArrayList(Edge).init(allocator);
    errdefer mstEdges.deinit();
    
    var totalWeight: u64 = 0;
    
    for (sortedEdges) |edge| {
        if (dsu.union(edge.u, edge.v)) {
            try mstEdges.append(edge);
            totalWeight += edge.weight;
            if (mstEdges.items.len == n - 1) break; // MST complete
        }
    }
    
    allocator.free(sortedEdges);
    return .{ .mstEdges = try mstEdges.toOwnedSlice(), .totalWeight = totalWeight };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Kruskal's MST ===\n\n", .{});
    
    var edges = [_]Edge{
        .{ .u = 0, .v = 1, .weight = 4 },
        .{ .u = 0, .v = 7, .weight = 8 },
        .{ .u = 1, .v = 2, .weight = 8 },
        .{ .u = 1, .v = 7, .weight = 11 },
        .{ .u = 2, .v = 3, .weight = 7 },
        .{ .u = 2, .v = 5, .weight = 4 },
        .{ .u = 2, .v = 8, .weight = 2 },
        .{ .u = 3, .v = 4, .weight = 9 },
        .{ .u = 3, .v = 5, .weight = 14 },
        .{ .u = 4, .v = 5, .weight = 10 },
        .{ .u = 5, .v = 6, .weight = 2 },
        .{ .u = 6, .v = 7, .weight = 1 },
        .{ .u = 6, .v = 8, .weight = 6 },
        .{ .u = 7, .v = 8, .weight = 7 },
    };
    
    const result = try kruskal(9, &edges, allocator);
    defer allocator.free(result.mstEdges);
    
    try stdout.print("MST Edges:\n", .{});
    for (result.mstEdges) |edge| {
        try stdout.print("  {d} -- {d}: weight {d}\n", .{ edge.u, edge.v, edge.weight });
    }
    try stdout.print("Total MST weight: {d}\n", .{result.totalWeight});
}
```

---

## Step 837: Prim's MST

```zig
// prims.zig
const std = @import("std");

const INF = std.math.maxInt(u64);

/// Prim's MST Algorithm
pub fn prims(
    n: usize,
    adj: []const []const struct { to: usize, weight: u64 },
    allocator: std.mem.Allocator,
) !struct { mstWeight: u64, parent: []i64 } {
    var key = try allocator.alloc(u64, n);
    defer allocator.free(key);
    @memset(key, INF);
    
    var parent = try allocator.alloc(i64, n);
    for (parent) |*p| p.* = -1;
    
    var inMST = try allocator.alloc(bool, n);
    defer allocator.free(inMST);
    @memset(inMST, false);
    
    key[0] = 0;
    
    var totalWeight: u64 = 0;
    
    for (0..n) |_| {
        // หา vertex ที่มี key น้อยที่สุดที่ยังไม่อยู่ใน MST
        var u: usize = 0;
        var minKey: u64 = INF;
        for (0..n) |v| {
            if (!inMST[v] and key[v] < minKey) {
                minKey = key[v];
                u = v;
            }
        }
        
        inMST[u] = true;
        if (key[u] != INF) totalWeight += key[u];
        
        // Update key ของ neighbors
        for (adj[u]) |edge| {
            if (!inMST[edge.to] and edge.weight < key[edge.to]) {
                key[edge.to] = edge.weight;
                parent[edge.to] = @intCast(u);
            }
        }
    }
    
    return .{ .mstWeight = totalWeight, .parent = parent };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Prim's MST ===\n\n", .{});
    
    const n = 5;
    
    // สร้าง adjacency list
    var adj: [n][]const struct { to: usize, weight: u64 } = undefined;
    
    const adj0 = [_]struct { to: usize, weight: u64 }{
        .{ .to = 1, .weight = 2 },
        .{ .to = 3, .weight = 6 },
    };
    const adj1 = [_]struct { to: usize, weight: u64 }{
        .{ .to = 0, .weight = 2 },
        .{ .to = 2, .weight = 3 },
        .{ .to = 3, .weight = 8 },
        .{ .to = 4, .weight = 5 },
    };
    const adj2 = [_]struct { to: usize, weight: u64 }{
        .{ .to = 1, .weight = 3 },
        .{ .to = 4, .weight = 7 },
    };
    const adj3 = [_]struct { to: usize, weight: u64 }{
        .{ .to = 0, .weight = 6 },
        .{ .to = 1, .weight = 8 },
        .{ .to = 4, .weight = 9 },
    };
    const adj4 = [_]struct { to: usize, weight: u64 }{
        .{ .to = 1, .weight = 5 },
        .{ .to = 2, .weight = 7 },
        .{ .to = 3, .weight = 9 },
    };
    
    adj[0] = &adj0;
    adj[1] = &adj1;
    adj[2] = &adj2;
    adj[3] = &adj3;
    adj[4] = &adj4;
    
    const result = try prims(n, &adj, allocator);
    defer allocator.free(result.parent);
    
    try stdout.print("MST (Prim's Algorithm):\n", .{});
    for (1..n) |v| {
        if (result.parent[v] >= 0) {
            try stdout.print("  {d} -- {d}\n", .{ result.parent[v], v });
        }
    }
    try stdout.print("Total MST weight: {d}\n", .{result.mstWeight});
}
```

---

## Step 838: Ford-Fulkerson Max Flow

```zig
// max_flow.zig
const std = @import("std");

/// Ford-Fulkerson Max Flow
pub const FlowGraph = struct {
    capacity: [][]u64,
    n: usize,
    allocator: std.mem.Allocator,
    
    pub fn init(n: usize, allocator: std.mem.Allocator) !FlowGraph {
        const cap = try allocator.alloc([]u64, n);
        for (cap) |*row| {
            row.* = try allocator.alloc(u64, n);
            @memset(row.*, 0);
        }
        return .{ .capacity = cap, .n = n, .allocator = allocator };
    }
    
    pub fn deinit(self: *FlowGraph) void {
        for (self.capacity) |row| self.allocator.free(row);
        self.allocator.free(self.capacity);
    }
    
    pub fn addEdge(self: *FlowGraph, u: usize, v: usize, cap: u64) void {
        self.capacity[u][v] += cap;
    }
    
    /// BFS หา augmenting path
    fn bfs(self: *FlowGraph, src: usize, sink: usize, parent: []i64) !bool {
        var visited = try self.allocator.alloc(bool, self.n);
        defer self.allocator.free(visited);
        @memset(visited, false);
        
        var queue = std.ArrayList(usize).init(self.allocator);
        defer queue.deinit();
        
        visited[src] = true;
        try queue.append(src);
        
        var head: usize = 0;
        while (head < queue.items.len) : (head += 1) {
            const u = queue.items[head];
            
            for (0..self.n) |v| {
                if (!visited[v] and self.capacity[u][v] > 0) {
                    visited[v] = true;
                    parent[v] = @intCast(u);
                    if (v == sink) return true;
                    try queue.append(v);
                }
            }
        }
        
        return false;
    }
    
    /// Edmonds-Karp (BFS-based Ford-Fulkerson)
    pub fn maxFlow(self: *FlowGraph, src: usize, sink: usize) !u64 {
        var parent = try self.allocator.alloc(i64, self.n);
        defer self.allocator.free(parent);
        
        var flow: u64 = 0;
        
        while (try self.bfs(src, sink, parent)) {
            // หา bottleneck
            var pathFlow: u64 = std.math.maxInt(u64);
            var v = sink;
            while (v != src) {
                const u: usize = @intCast(parent[v]);
                pathFlow = @min(pathFlow, self.capacity[u][v]);
                v = u;
            }
            
            // Update capacities
            v = sink;
            while (v != src) {
                const u: usize = @intCast(parent[v]);
                self.capacity[u][v] -= pathFlow;
                self.capacity[v][u] += pathFlow;
                v = u;
            }
            
            flow += pathFlow;
        }
        
        return flow;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Ford-Fulkerson Max Flow ===\n\n", .{});
    
    var g = try FlowGraph.init(6, allocator);
    defer g.deinit();
    
    // กราฟตัวอย่าง
    g.addEdge(0, 1, 16);
    g.addEdge(0, 2, 13);
    g.addEdge(1, 2, 10);
    g.addEdge(1, 3, 12);
    g.addEdge(2, 1, 4);
    g.addEdge(2, 4, 14);
    g.addEdge(3, 2, 9);
    g.addEdge(3, 5, 20);
    g.addEdge(4, 3, 7);
    g.addEdge(4, 5, 4);
    
    const mf = try g.maxFlow(0, 5);
    try stdout.print("Max Flow จาก 0 ถึง 5: {d}\n", .{mf});
    try stdout.print("(Expected: 23)\n", .{});
}
```

---

## Step 839: Tarjan's SCC

```zig
// tarjan_scc.zig
const std = @import("std");

/// Tarjan's Strongly Connected Components
pub const TarjanSCC = struct {
    adj: []const []const usize,
    n: usize,
    allocator: std.mem.Allocator,
    
    disc: []i64,
    low: []i64,
    onStack: []bool,
    stack: std.ArrayList(usize),
    timer: i64,
    components: std.ArrayList(std.ArrayList(usize)),
    
    pub fn init(n: usize, adj: []const []const usize, allocator: std.mem.Allocator) !TarjanSCC {
        var t = TarjanSCC{
            .adj = adj,
            .n = n,
            .allocator = allocator,
            .disc = try allocator.alloc(i64, n),
            .low = try allocator.alloc(i64, n),
            .onStack = try allocator.alloc(bool, n),
            .stack = std.ArrayList(usize).init(allocator),
            .timer = 0,
            .components = std.ArrayList(std.ArrayList(usize)).init(allocator),
        };
        @memset(t.disc, -1);
        @memset(t.low, -1);
        @memset(t.onStack, false);
        return t;
    }
    
    pub fn deinit(self: *TarjanSCC) void {
        self.allocator.free(self.disc);
        self.allocator.free(self.low);
        self.allocator.free(self.onStack);
        self.stack.deinit();
        for (self.components.items) |*comp| comp.deinit();
        self.components.deinit();
    }
    
    fn dfsHelper(self: *TarjanSCC, v: usize) !void {
        self.disc[v] = self.timer;
        self.low[v] = self.timer;
        self.timer += 1;
        
        try self.stack.append(v);
        self.onStack[v] = true;
        
        for (self.adj[v]) |neighbor| {
            if (self.disc[neighbor] == -1) {
                try self.dfsHelper(neighbor);
                self.low[v] = @min(self.low[v], self.low[neighbor]);
            } else if (self.onStack[neighbor]) {
                self.low[v] = @min(self.low[v], self.disc[neighbor]);
            }
        }
        
        // Root of SCC
        if (self.low[v] == self.disc[v]) {
            var component = std.ArrayList(usize).init(self.allocator);
            
            while (true) {
                const w = self.stack.pop();
                self.onStack[w] = false;
                try component.append(w);
                if (w == v) break;
            }
            
            try self.components.append(component);
        }
    }
    
    pub fn findSCC(self: *TarjanSCC) ![]std.ArrayList(usize) {
        for (0..self.n) |v| {
            if (self.disc[v] == -1) {
                try self.dfsHelper(v);
            }
        }
        return self.components.items;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Tarjan's SCC ===\n\n", .{});
    
    // กราฟตัวอย่าง
    const adj0 = [_]usize{ 2, 3 };
    const adj1 = [_]usize{ 0 };
    const adj2 = [_]usize{ 1 };
    const adj3 = [_]usize{ 4 };
    const adj4 = [_]usize{};
    
    const adj = [_][]const usize{ &adj0, &adj1, &adj2, &adj3, &adj4 };
    
    var tarjan = try TarjanSCC.init(5, &adj, allocator);
    defer tarjan.deinit();
    
    const sccs = try tarjan.findSCC();
    
    try stdout.print("Strongly Connected Components:\n", .{});
    for (sccs, 0..) |scc, i| {
        try stdout.print("  SCC {d}: ", .{i + 1});
        for (scc.items) |v| try stdout.print("{d} ", .{v});
        try stdout.print("\n", .{});
    }
}
```

---

## Step 840: Practical Graph Problems

```zig
// practical_graphs.zig
const std = @import("std");

/// Topological Sort (Kahn's Algorithm)
pub fn topologicalSort(n: usize, adj: []const []const usize, allocator: std.mem.Allocator) !?[]usize {
    var inDegree = try allocator.alloc(usize, n);
    defer allocator.free(inDegree);
    @memset(inDegree, 0);
    
    for (adj) |neighbors| {
        for (neighbors) |v| inDegree[v] += 1;
    }
    
    var queue = std.ArrayList(usize).init(allocator);
    defer queue.deinit();
    
    for (0..n) |v| {
        if (inDegree[v] == 0) try queue.append(v);
    }
    
    var order = std.ArrayList(usize).init(allocator);
    errdefer order.deinit();
    
    var head: usize = 0;
    while (head < queue.items.len) : (head += 1) {
        const v = queue.items[head];
        try order.append(v);
        
        for (adj[v]) |neighbor| {
            inDegree[neighbor] -= 1;
            if (inDegree[neighbor] == 0) try queue.append(neighbor);
        }
    }
    
    if (order.items.len != n) return null; // มี cycle
    return order.toOwnedSlice();
}

/// ตรวจสอบ Bipartite Graph
pub fn isBipartite(n: usize, adj: []const []const usize, allocator: std.mem.Allocator) !bool {
    var color = try allocator.alloc(i32, n);
    defer allocator.free(color);
    @memset(color, -1);
    
    for (0..n) |start| {
        if (color[start] != -1) continue;
        
        var queue = std.ArrayList(usize).init(allocator);
        defer queue.deinit();
        
        color[start] = 0;
        try queue.append(start);
        
        var head: usize = 0;
        while (head < queue.items.len) : (head += 1) {
            const v = queue.items[head];
            
            for (adj[v]) |neighbor| {
                if (color[neighbor] == -1) {
                    color[neighbor] = 1 - color[v];
                    try queue.append(neighbor);
                } else if (color[neighbor] == color[v]) {
                    return false;
                }
            }
        }
    }
    
    return true;
}

/// จำนวน Connected Components
pub fn countComponents(n: usize, adj: []const []const usize, allocator: std.mem.Allocator) !usize {
    var visited = try allocator.alloc(bool, n);
    defer allocator.free(visited);
    @memset(visited, false);
    
    var count: usize = 0;
    
    for (0..n) |start| {
        if (visited[start]) continue;
        
        count += 1;
        var stack = std.ArrayList(usize).init(allocator);
        defer stack.deinit();
        
        try stack.append(start);
        while (stack.items.len > 0) {
            const v = stack.pop();
            if (visited[v]) continue;
            visited[v] = true;
            
            for (adj[v]) |neighbor| {
                if (!visited[neighbor]) try stack.append(neighbor);
            }
        }
    }
    
    return count;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const stdout = std.io.getStdOut().writer();
    
    try stdout.print("=== Practical Graph Problems ===\n\n", .{});
    
    // 1. Topological Sort
    try stdout.print("1. Topological Sort (Course Prerequisites):\n", .{});
    const courses = [_][]const usize{
        &[_]usize{ 1, 2 }, // Course 0 -> 1, 2
        &[_]usize{3},       // Course 1 -> 3
        &[_]usize{3},       // Course 2 -> 3
        &[_]usize{4},       // Course 3 -> 4
        &[_]usize{},        // Course 4
    };
    
    if (try topologicalSort(5, &courses, allocator)) |order| {
        defer allocator.free(order);
        try stdout.print("   Course order: ", .{});
        for (order) |v| try stdout.print("{d} ", .{v});
        try stdout.print("\n\n", .{});
    }
    
    // 2. Bipartite Check
    try stdout.print("2. Bipartite Graph Check:\n", .{});
    const bipartiteAdj = [_][]const usize{
        &[_]usize{ 1, 3 },
        &[_]usize{ 0, 2 },
        &[_]usize{ 1, 3 },
        &[_]usize{ 0, 2 },
    };
    
    const isBi = try isBipartite(4, &bipartiteAdj, allocator);
    try stdout.print("   Is Bipartite: {}\n\n", .{isBi});
    
    // 3. Connected Components
    try stdout.print("3. Connected Components:\n", .{});
    const componentAdj = [_][]const usize{
        &[_]usize{1},
        &[_]usize{0},
        &[_]usize{3},
        &[_]usize{2},
        &[_]usize{}, // isolated
    };
    
    const components = try countComponents(5, &componentAdj, allocator);
    try stdout.print("   จำนวน components: {d}\n\n", .{components});
    
    // สรุป
    try stdout.print("=== สรุป Graph Algorithms ===\n", .{});
    try stdout.print("Traversal: BFS (shortest unweighted), DFS\n", .{});
    try stdout.print("Shortest Path: Dijkstra (pos weights), Bellman-Ford (neg weights)\n", .{});
    try stdout.print("All-pairs: Floyd-Warshall\n", .{});
    try stdout.print("MST: Kruskal (sort edges), Prim (grow tree)\n", .{});
    try stdout.print("Flow: Ford-Fulkerson, Edmonds-Karp\n", .{});
    try stdout.print("SCC: Tarjan, Kosaraju\n", .{});
    try stdout.print("Others: Topological Sort, Bipartite Check\n", .{});
}
```

### สรุป Part 84

ใน Part นี้เราได้เรียนรู้:
1. BFS/DFS traversal - พื้นฐานของ graph algorithms
2. Dijkstra - shortest path สำหรับ positive weights
3. A* - shortest path พร้อม heuristic
4. Bellman-Ford - รับ negative weights
5. Floyd-Warshall - all-pairs shortest path
6. Kruskal's MST - เรียง edges แล้วเพิ่ม
7. Prim's MST - grow tree จาก vertex
8. Ford-Fulkerson - max flow
9. Tarjan's SCC - strongly connected components
10. Practical problems - topological sort, bipartite

---

*ต่อไป: Part 85 - Parsing: Lexer Implementation*
