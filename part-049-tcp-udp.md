# Part 49: TCP/UDP Programming ขั้นสูง

## ภาพรวม

ในบทนี้เราจะลงลึกกับ TCP และ UDP programming ขั้นสูง รวมถึงการออกแบบ protocol, การจัดการ multiple clients, multiplexing ด้วย epoll/kqueue และการสร้าง chat server จริงๆ

---

## Step 481: TCP socket programming ขั้นสูง

### TCP State Machine

```
CLOSED -> LISTEN -> SYN_RCVD -> ESTABLISHED -> CLOSE_WAIT -> LAST_ACK -> CLOSED
                                            -> FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT -> CLOSED
```

### Advanced TCP Socket Setup

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;

pub const TcpSocket = struct {
    fd: posix.socket_t,
    
    pub fn create() !TcpSocket {
        const fd = try posix.socket(
            posix.AF.INET,
            posix.SOCK.STREAM | posix.SOCK.CLOEXEC,
            posix.IPPROTO.TCP,
        );
        return .{ .fd = fd };
    }
    
    pub fn close(self: TcpSocket) void {
        posix.close(self.fd);
    }
    
    pub fn setReuseAddr(self: TcpSocket) !void {
        try posix.setsockopt(
            self.fd,
            posix.SOL.SOCKET,
            posix.SO.REUSEADDR,
            &std.mem.toBytes(@as(c_int, 1)),
        );
    }
    
    pub fn setNoDelay(self: TcpSocket) !void {
        try posix.setsockopt(
            self.fd,
            posix.IPPROTO.TCP,
            std.c.TCP.NODELAY,
            &std.mem.toBytes(@as(c_int, 1)),
        );
    }
    
    pub fn setKeepAlive(self: TcpSocket, idle_s: u32, interval_s: u32, count: u32) !void {
        try posix.setsockopt(
            self.fd,
            posix.SOL.SOCKET,
            posix.SO.KEEPALIVE,
            &std.mem.toBytes(@as(c_int, 1)),
        );
        
        // TCP_KEEPIDLE - เวลาก่อนส่ง keepalive probe แรก
        try posix.setsockopt(
            self.fd,
            posix.IPPROTO.TCP,
            std.c.TCP.KEEPIDLE,
            &std.mem.toBytes(@as(c_int, @intCast(idle_s))),
        );
        
        // TCP_KEEPINTVL - interval ระหว่าง probes
        try posix.setsockopt(
            self.fd,
            posix.IPPROTO.TCP,
            std.c.TCP.KEEPINTVL,
            &std.mem.toBytes(@as(c_int, @intCast(interval_s))),
        );
        
        // TCP_KEEPCNT - จำนวน probes
        try posix.setsockopt(
            self.fd,
            posix.IPPROTO.TCP,
            std.c.TCP.KEEPCNT,
            &std.mem.toBytes(@as(c_int, @intCast(count))),
        );
    }
    
    pub fn setRecvBuffer(self: TcpSocket, size: u32) !void {
        try posix.setsockopt(
            self.fd,
            posix.SOL.SOCKET,
            posix.SO.RCVBUF,
            &std.mem.toBytes(@as(c_int, @intCast(size))),
        );
    }
    
    pub fn setSendBuffer(self: TcpSocket, size: u32) !void {
        try posix.setsockopt(
            self.fd,
            posix.SOL.SOCKET,
            posix.SO.SNDBUF,
            &std.mem.toBytes(@as(c_int, @intCast(size))),
        );
    }
    
    pub fn setNonBlocking(self: TcpSocket) !void {
        const flags = try posix.fcntl(self.fd, posix.F.GETFL, 0);
        _ = try posix.fcntl(self.fd, posix.F.SETFL, flags | posix.O.NONBLOCK);
    }
    
    pub fn bind(self: TcpSocket, addr: net.Address) !void {
        try posix.bind(self.fd, &addr.any, addr.getOsSockLen());
    }
    
    pub fn listen(self: TcpSocket, backlog: u31) !void {
        try posix.listen(self.fd, backlog);
    }
    
    pub fn connect(self: TcpSocket, addr: net.Address) !void {
        try posix.connect(self.fd, &addr.any, addr.getOsSockLen());
    }
    
    pub fn accept(self: TcpSocket) !struct { fd: posix.socket_t, addr: net.Address } {
        var addr: posix.sockaddr = undefined;
        var addr_len: posix.socklen_t = @sizeOf(posix.sockaddr);
        
        const client_fd = try posix.accept(self.fd, &addr, &addr_len, posix.SOCK.CLOEXEC);
        const client_addr = net.Address.initPosix(@alignCast(&addr));
        
        return .{ .fd = client_fd, .addr = client_addr };
    }
};

pub fn main() !void {
    const sock = try TcpSocket.create();
    defer sock.close();
    
    try sock.setReuseAddr();
    try sock.setNoDelay();
    try sock.setKeepAlive(60, 10, 3);
    try sock.setRecvBuffer(256 * 1024);
    try sock.setSendBuffer(256 * 1024);
    
    const addr = try net.Address.parseIp4("0.0.0.0", 8080);
    try sock.bind(addr);
    try sock.listen(128);
    
    std.debug.print("Advanced TCP server on :8080\n", .{});
}
```

---

## Step 482: Multiple client handling

### Thread-per-connection Model

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Atomic = std.atomic.Value;

const ClientManager = struct {
    mutex: Mutex = .{},
    clients: std.ArrayList(ClientInfo),
    next_id: Atomic(u32) = Atomic(u32).init(1),
    max_clients: u32,
    
    const ClientInfo = struct {
        id: u32,
        address: net.Address,
        connected_at: i64,
        bytes_sent: u64,
        bytes_received: u64,
    };
    
    pub fn init(allocator: std.mem.Allocator, max_clients: u32) ClientManager {
        return .{
            .clients = std.ArrayList(ClientInfo).init(allocator),
            .max_clients = max_clients,
        };
    }
    
    pub fn deinit(self: *ClientManager) void {
        self.clients.deinit();
    }
    
    pub fn addClient(self: *ClientManager, addr: net.Address) !u32 {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        if (self.clients.items.len >= self.max_clients) {
            return error.TooManyClients;
        }
        
        const id = self.next_id.fetchAdd(1, .monotonic);
        try self.clients.append(.{
            .id = id,
            .address = addr,
            .connected_at = std.time.timestamp(),
            .bytes_sent = 0,
            .bytes_received = 0,
        });
        
        return id;
    }
    
    pub fn removeClient(self: *ClientManager, id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (self.clients.items, 0..) |client, i| {
            if (client.id == id) {
                _ = self.clients.swapRemove(i);
                return;
            }
        }
    }
    
    pub fn getCount(self: *ClientManager) usize {
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.clients.items.len;
    }
    
    pub fn printStats(self: *ClientManager) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        std.debug.print("Connected clients: {d}\n", .{self.clients.items.len});
        for (self.clients.items) |c| {
            std.debug.print("  [{d}] {} (since {d})\n", .{ c.id, c.address, c.connected_at });
        }
    }
};

const ConnContext = struct {
    stream: net.Stream,
    address: net.Address,
    client_id: u32,
    manager: *ClientManager,
    allocator: std.mem.Allocator,
};

fn handleClient(ctx: ConnContext) void {
    defer {
        ctx.stream.close();
        ctx.manager.removeClient(ctx.client_id);
        std.debug.print("Client {d} disconnected\n", .{ctx.client_id});
    }
    
    var buf: [4096]u8 = undefined;
    
    while (true) {
        const n = ctx.stream.read(&buf) catch break;
        if (n == 0) break;
        
        // Echo server
        ctx.stream.writeAll(buf[0..n]) catch break;
    }
}

var client_mgr: ClientManager = undefined;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    client_mgr = ClientManager.init(allocator, 100);
    defer client_mgr.deinit();
    
    const addr = try net.Address.parseIp4("0.0.0.0", 8080);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    std.debug.print("Multi-client server on :8080\n", .{});
    
    // Stats thread
    const stats_t = try Thread.spawn(.{}, statsLoop, .{});
    stats_t.detach();
    
    while (true) {
        const conn = server.accept() catch continue;
        
        const client_id = client_mgr.addClient(conn.address) catch |err| {
            std.debug.print("Cannot add client: {}\n", .{err});
            conn.stream.close();
            continue;
        };
        
        std.debug.print("Client {d} connected from {}\n", .{ client_id, conn.address });
        
        const ctx = ConnContext{
            .stream = conn.stream,
            .address = conn.address,
            .client_id = client_id,
            .manager = &client_mgr,
            .allocator = allocator,
        };
        
        const t = Thread.spawn(.{}, handleClient, .{ctx}) catch |err| {
            std.debug.print("Thread error: {}\n", .{err});
            conn.stream.close();
            client_mgr.removeClient(client_id);
            continue;
        };
        t.detach();
    }
}

fn statsLoop() void {
    while (true) {
        std.time.sleep(10 * std.time.ns_per_s);
        client_mgr.printStats();
    }
}
```

---

## Step 483: Protocol design

การออกแบบ protocol ที่ดีควรมี:
1. **Length-prefixed messages**: รู้ขนาด message ก่อนอ่าน
2. **Magic bytes**: ตรวจสอบว่าเป็น protocol ที่ถูกต้อง
3. **Version**: รองรับการ upgrade ในอนาคต
4. **Checksum**: ตรวจสอบความสมบูรณ์ของข้อมูล

```zig
const std = @import("std");

// Protocol header: [MAGIC:4][VERSION:1][TYPE:1][LENGTH:4][CHECKSUM:4]
// Total header: 14 bytes

pub const MAGIC: u32 = 0x5A494750; // "ZIGP"
pub const VERSION: u8 = 1;

pub const MessageType = enum(u8) {
    handshake = 0x01,
    data = 0x02,
    ping = 0x03,
    pong = 0x04,
    error_msg = 0x05,
    close = 0xFF,
};

pub const Header = extern struct {
    magic: u32,
    version: u8,
    msg_type: u8,
    length: u32,
    checksum: u32,
    
    pub fn encode(self: Header) [14]u8 {
        var buf: [14]u8 = undefined;
        std.mem.writeInt(u32, buf[0..4], self.magic, .big);
        buf[4] = self.version;
        buf[5] = self.msg_type;
        std.mem.writeInt(u32, buf[6..10], self.length, .big);
        std.mem.writeInt(u32, buf[10..14], self.checksum, .big);
        return buf;
    }
    
    pub fn decode(buf: [14]u8) Header {
        return .{
            .magic = std.mem.readInt(u32, buf[0..4], .big),
            .version = buf[4],
            .msg_type = buf[5],
            .length = std.mem.readInt(u32, buf[6..10], .big),
            .checksum = std.mem.readInt(u32, buf[10..14], .big),
        };
    }
    
    pub fn isValid(self: Header) bool {
        return self.magic == MAGIC and self.version == VERSION;
    }
};

pub fn computeChecksum(data: []const u8) u32 {
    var sum: u32 = 0;
    for (data) |b| {
        sum = sum *% 31 +% b;
    }
    return sum;
}

pub const Message = struct {
    header: Header,
    payload: []const u8,
    
    pub fn encode(
        allocator: std.mem.Allocator,
        msg_type: MessageType,
        payload: []const u8,
    ) ![]u8 {
        const checksum = computeChecksum(payload);
        const header = Header{
            .magic = MAGIC,
            .version = VERSION,
            .msg_type = @intFromEnum(msg_type),
            .length = @intCast(payload.len),
            .checksum = checksum,
        };
        
        const encoded = try allocator.alloc(u8, 14 + payload.len);
        const header_bytes = header.encode();
        @memcpy(encoded[0..14], &header_bytes);
        @memcpy(encoded[14..], payload);
        
        return encoded;
    }
    
    pub fn readFrom(reader: anytype, allocator: std.mem.Allocator) !Message {
        var header_buf: [14]u8 = undefined;
        try reader.readNoEof(&header_buf);
        
        const header = Header.decode(header_buf);
        
        if (!header.isValid()) {
            return error.InvalidMagic;
        }
        
        if (header.length > 10 * 1024 * 1024) { // 10MB max
            return error.MessageTooLarge;
        }
        
        const payload = try allocator.alloc(u8, header.length);
        errdefer allocator.free(payload);
        
        try reader.readNoEof(payload);
        
        // Verify checksum
        const actual_checksum = computeChecksum(payload);
        if (actual_checksum != header.checksum) {
            return error.ChecksumMismatch;
        }
        
        return .{ .header = header, .payload = payload };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง message
    const payload = "Hello, Protocol!";
    const encoded = try Message.encode(allocator, .data, payload);
    defer allocator.free(encoded);
    
    std.debug.print("Encoded ({d} bytes): ", .{encoded.len});
    for (encoded[0..14]) |b| {
        std.debug.print("{x:0>2} ", .{b});
    }
    std.debug.print("\n", .{});
    
    // Decode message
    var stream = std.io.fixedBufferStream(encoded);
    const msg = try Message.readFrom(stream.reader(), allocator);
    defer allocator.free(msg.payload);
    
    std.debug.print("Decoded: type={d}, payload='{s}'\n", .{
        msg.header.msg_type, msg.payload,
    });
}
```

---

## Step 484: Binary protocol implementation

```zig
const std = @import("std");
const net = std.net;

// Redis-like binary protocol (RESP)
pub const RespType = enum {
    simple_string,
    error_msg,
    integer,
    bulk_string,
    array,
    null_bulk,
};

pub const RespValue = union(RespType) {
    simple_string: []const u8,
    error_msg: []const u8,
    integer: i64,
    bulk_string: []const u8,
    array: []RespValue,
    null_bulk,
};

pub fn encodeResp(allocator: std.mem.Allocator, value: RespValue) ![]u8 {
    var buf = std.ArrayList(u8).init(allocator);
    errdefer buf.deinit();
    
    switch (value) {
        .simple_string => |s| {
            try buf.writer().print("+{s}\r\n", .{s});
        },
        .error_msg => |e| {
            try buf.writer().print("-{s}\r\n", .{e});
        },
        .integer => |i| {
            try buf.writer().print(":{d}\r\n", .{i});
        },
        .bulk_string => |s| {
            try buf.writer().print("${d}\r\n{s}\r\n", .{ s.len, s });
        },
        .null_bulk => {
            try buf.appendSlice("$-1\r\n");
        },
        .array => |items| {
            try buf.writer().print("*{d}\r\n", .{items.len});
            for (items) |item| {
                const encoded = try encodeResp(allocator, item);
                defer allocator.free(encoded);
                try buf.appendSlice(encoded);
            }
        },
    }
    
    return buf.toOwnedSlice();
}

pub fn parseCommand(allocator: std.mem.Allocator, data: []const u8) ![][]const u8 {
    var args = std.ArrayList([]const u8).init(allocator);
    errdefer args.deinit();
    
    if (data.len == 0 or data[0] != '*') return error.InvalidFormat;
    
    const newline = std.mem.indexOf(u8, data, "\r\n") orelse return error.InvalidFormat;
    const count = try std.fmt.parseInt(usize, data[1..newline], 10);
    
    var pos = newline + 2;
    var i: usize = 0;
    while (i < count) : (i += 1) {
        if (pos >= data.len or data[pos] != '$') return error.InvalidFormat;
        
        const len_end = std.mem.indexOf(u8, data[pos..], "\r\n") orelse return error.InvalidFormat;
        const arg_len = try std.fmt.parseInt(usize, data[pos + 1..pos + len_end], 10);
        
        pos += len_end + 2;
        
        if (pos + arg_len > data.len) return error.InvalidFormat;
        
        try args.append(data[pos..pos + arg_len]);
        pos += arg_len + 2; // skip \r\n
    }
    
    return args.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // Encode responses
    const ok = try encodeResp(allocator, .{ .simple_string = "OK" });
    defer allocator.free(ok);
    std.debug.print("OK: {s}", .{ok});
    
    const num = try encodeResp(allocator, .{ .integer = 42 });
    defer allocator.free(num);
    std.debug.print("42: {s}", .{num});
    
    const str = try encodeResp(allocator, .{ .bulk_string = "Hello" });
    defer allocator.free(str);
    std.debug.print("Hello: {s}", .{str});
    
    // Parse command
    const cmd = "*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n";
    const args = try parseCommand(allocator, cmd);
    defer allocator.free(args);
    
    std.debug.print("Command: ", .{});
    for (args) |arg| {
        std.debug.print("{s} ", .{arg});
    }
    std.debug.print("\n", .{});
}
```

---

## Step 485: UDP reliable transmission

UDP ไม่มีการรับประกัน แต่เราสามารถสร้างความน่าเชื่อถือเองได้

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;

// Simple ARQ (Automatic Repeat reQuest) protocol
pub const PacketType = enum(u8) {
    data = 0,
    ack = 1,
    nack = 2,
};

pub const Packet = packed struct {
    seq_num: u32,
    ptype: u8,
    payload_len: u16,
    // payload follows
};

const HEADER_SIZE = @sizeOf(Packet);
const MAX_PAYLOAD = 1400; // สูงสุด 1400 bytes (MTU safe)
const TIMEOUT_MS = 500;
const MAX_RETRIES = 5;

pub const ReliableUdp = struct {
    sock: posix.socket_t,
    remote_addr: net.Address,
    next_seq: u32 = 0,
    expected_seq: u32 = 0,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, local_port: u16, remote: net.Address) !ReliableUdp {
        const sock = try posix.socket(posix.AF.INET, posix.SOCK.DGRAM, 0);
        errdefer posix.close(sock);
        
        const local_addr = try net.Address.parseIp4("0.0.0.0", local_port);
        try posix.bind(sock, &local_addr.any, local_addr.getOsSockLen());
        
        // ตั้ง receive timeout
        const timeout = posix.timeval{
            .tv_sec = 0,
            .tv_usec = TIMEOUT_MS * 1000,
        };
        try posix.setsockopt(
            sock,
            posix.SOL.SOCKET,
            posix.SO.RCVTIMEO,
            std.mem.asBytes(&timeout),
        );
        
        return .{
            .sock = sock,
            .remote_addr = remote,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: ReliableUdp) void {
        posix.close(self.sock);
    }
    
    pub fn send(self: *ReliableUdp, data: []const u8) !void {
        var offset: usize = 0;
        
        while (offset < data.len) {
            const chunk_size = @min(MAX_PAYLOAD, data.len - offset);
            const chunk = data[offset..offset + chunk_size];
            
            const seq = self.next_seq;
            self.next_seq += 1;
            
            var retries: u32 = 0;
            while (retries < MAX_RETRIES) : (retries += 1) {
                try self.sendPacket(seq, .data, chunk);
                
                // รอ ACK
                var ack_buf: [HEADER_SIZE]u8 = undefined;
                _ = posix.recv(self.sock, &ack_buf, 0) catch |err| {
                    if (err == error.WouldBlock) {
                        std.debug.print("Timeout, retry {d}\n", .{retries + 1});
                        continue;
                    }
                    return err;
                };
                
                const ack_header: *const Packet = @ptrCast(&ack_buf);
                if (ack_header.ptype == @intFromEnum(PacketType.ack) and
                    ack_header.seq_num == seq)
                {
                    break; // ACK ได้รับ
                }
            } else {
                return error.MaxRetriesExceeded;
            }
            
            offset += chunk_size;
        }
    }
    
    pub fn receive(self: *ReliableUdp, buffer: []u8) !usize {
        var buf: [HEADER_SIZE + MAX_PAYLOAD]u8 = undefined;
        var from: posix.sockaddr = undefined;
        var from_len: posix.socklen_t = @sizeOf(posix.sockaddr);
        
        const n = try posix.recvfrom(self.sock, &buf, 0, &from, &from_len);
        if (n < HEADER_SIZE) return error.PacketTooSmall;
        
        const header: *const Packet = @ptrCast(&buf);
        const payload = buf[HEADER_SIZE..n];
        
        if (header.ptype != @intFromEnum(PacketType.data)) return 0;
        
        if (header.seq_num != self.expected_seq) {
            // ส่ง NACK
            try self.sendAckTo(&from, from_len, header.seq_num, .nack);
            return 0;
        }
        
        // ส่ง ACK
        try self.sendAckTo(&from, from_len, header.seq_num, .ack);
        self.expected_seq += 1;
        
        const copy_len = @min(buffer.len, payload.len);
        @memcpy(buffer[0..copy_len], payload[0..copy_len]);
        return copy_len;
    }
    
    fn sendPacket(self: *ReliableUdp, seq: u32, ptype: PacketType, payload: []const u8) !void {
        var buf: [HEADER_SIZE + MAX_PAYLOAD]u8 = undefined;
        
        const header: *Packet = @ptrCast(&buf);
        header.seq_num = seq;
        header.ptype = @intFromEnum(ptype);
        header.payload_len = @intCast(payload.len);
        
        @memcpy(buf[HEADER_SIZE..HEADER_SIZE + payload.len], payload);
        
        _ = try posix.sendto(
            self.sock,
            buf[0..HEADER_SIZE + payload.len],
            0,
            &self.remote_addr.any,
            self.remote_addr.getOsSockLen(),
        );
    }
    
    fn sendAckTo(
        self: *ReliableUdp,
        addr: *const posix.sockaddr,
        addr_len: posix.socklen_t,
        seq: u32,
        ptype: PacketType,
    ) !void {
        var buf: [HEADER_SIZE]u8 = undefined;
        const header: *Packet = @ptrCast(&buf);
        header.seq_num = seq;
        header.ptype = @intFromEnum(ptype);
        header.payload_len = 0;
        
        _ = try posix.sendto(self.sock, &buf, 0, addr, addr_len);
    }
};
```

---

## Step 486: Multiplexing (epoll/kqueue)

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;
const linux = std.os.linux;

// epoll-based event loop (Linux)
pub const EventLoop = struct {
    epoll_fd: posix.fd_t,
    events: [64]linux.epoll_event,
    allocator: std.mem.Allocator,
    handlers: std.AutoHashMap(posix.fd_t, *EventHandler),
    
    pub const EventHandler = struct {
        fd: posix.fd_t,
        on_read: ?*const fn (fd: posix.fd_t) void = null,
        on_write: ?*const fn (fd: posix.fd_t) void = null,
        on_error: ?*const fn (fd: posix.fd_t) void = null,
    };
    
    pub fn init(allocator: std.mem.Allocator) !EventLoop {
        const epoll_fd = try posix.epoll_create1(posix.EPOLL.CLOEXEC);
        return .{
            .epoll_fd = epoll_fd,
            .events = undefined,
            .allocator = allocator,
            .handlers = std.AutoHashMap(posix.fd_t, *EventHandler).init(allocator),
        };
    }
    
    pub fn deinit(self: *EventLoop) void {
        posix.close(self.epoll_fd);
        self.handlers.deinit();
    }
    
    pub fn addFd(self: *EventLoop, handler: *EventHandler) !void {
        var ev = linux.epoll_event{
            .events = linux.EPOLL.IN | linux.EPOLL.ET, // Edge-triggered
            .data = .{ .fd = handler.fd },
        };
        
        try posix.epoll_ctl(self.epoll_fd, linux.EPOLL.CTL_ADD, handler.fd, &ev);
        try self.handlers.put(handler.fd, handler);
    }
    
    pub fn removeFd(self: *EventLoop, fd: posix.fd_t) !void {
        try posix.epoll_ctl(self.epoll_fd, linux.EPOLL.CTL_DEL, fd, null);
        _ = self.handlers.remove(fd);
    }
    
    pub fn run(self: *EventLoop, timeout_ms: i32) !void {
        const num_events = try posix.epoll_wait(
            self.epoll_fd,
            &self.events,
            timeout_ms,
        );
        
        for (self.events[0..num_events]) |ev| {
            const fd = ev.data.fd;
            const handler = self.handlers.get(fd) orelse continue;
            
            if (ev.events & linux.EPOLL.IN != 0) {
                if (handler.on_read) |on_read| {
                    on_read(fd);
                }
            }
            
            if (ev.events & linux.EPOLL.OUT != 0) {
                if (handler.on_write) |on_write| {
                    on_write(fd);
                }
            }
            
            if (ev.events & (linux.EPOLL.ERR | linux.EPOLL.HUP) != 0) {
                if (handler.on_error) |on_error| {
                    on_error(fd);
                }
            }
        }
    }
    
    pub fn runForever(self: *EventLoop) !void {
        while (true) {
            try self.run(-1); // block จนมี event
        }
    }
};
```

---

## Step 487: Connection pooling

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;
const Mutex = Thread.Mutex;
const Condition = Thread.Condition;

pub const ConnectionPool = struct {
    const Conn = struct {
        stream: net.Stream,
        in_use: bool = false,
        created_at: i64,
        last_used: i64,
    };
    
    mutex: Mutex = .{},
    available: Condition = .{},
    connections: std.ArrayList(Conn),
    host: []const u8,
    port: u16,
    max_size: usize,
    min_size: usize,
    max_idle_s: u64,
    allocator: std.mem.Allocator,
    
    pub fn init(
        allocator: std.mem.Allocator,
        host: []const u8,
        port: u16,
        min_size: usize,
        max_size: usize,
    ) !ConnectionPool {
        var pool = ConnectionPool{
            .connections = std.ArrayList(Conn).init(allocator),
            .host = host,
            .port = port,
            .max_size = max_size,
            .min_size = min_size,
            .max_idle_s = 300,
            .allocator = allocator,
        };
        
        // สร้าง minimum connections
        var i: usize = 0;
        while (i < min_size) : (i += 1) {
            const stream = try net.tcpConnectToHost(allocator, host, port);
            const now = std.time.timestamp();
            try pool.connections.append(.{
                .stream = stream,
                .created_at = now,
                .last_used = now,
            });
        }
        
        return pool;
    }
    
    pub fn deinit(self: *ConnectionPool) void {
        for (self.connections.items) |conn| {
            conn.stream.close();
        }
        self.connections.deinit();
    }
    
    pub fn acquire(self: *ConnectionPool) !*Conn {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        // หา connection ที่ว่าง
        for (self.connections.items) |*conn| {
            if (!conn.in_use) {
                conn.in_use = true;
                conn.last_used = std.time.timestamp();
                return conn;
            }
        }
        
        // สร้าง connection ใหม่ถ้ายังไม่เต็ม
        if (self.connections.items.len < self.max_size) {
            const stream = try net.tcpConnectToHost(self.allocator, self.host, self.port);
            const now = std.time.timestamp();
            try self.connections.append(.{
                .stream = stream,
                .in_use = true,
                .created_at = now,
                .last_used = now,
            });
            return &self.connections.items[self.connections.items.len - 1];
        }
        
        // รอ connection ว่าง
        while (true) {
            self.available.wait(&self.mutex);
            for (self.connections.items) |*conn| {
                if (!conn.in_use) {
                    conn.in_use = true;
                    return conn;
                }
            }
        }
    }
    
    pub fn release(self: *ConnectionPool, conn: *Conn) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        conn.in_use = false;
        conn.last_used = std.time.timestamp();
        self.available.signal();
    }
    
    pub fn stats(self: *ConnectionPool) struct { total: usize, in_use: usize } {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        var in_use: usize = 0;
        for (self.connections.items) |conn| {
            if (conn.in_use) in_use += 1;
        }
        
        return .{ .total = self.connections.items.len, .in_use = in_use };
    }
};
```

---

## Step 488: Backpressure handling

```zig
const std = @import("std");
const net = std.net;

// Backpressure: ควบคุมอัตราการส่งข้อมูลเมื่อ receiver ไม่สามารถรับได้ทัน

pub const BufferedWriter = struct {
    stream: net.Stream,
    buffer: std.ArrayList(u8),
    max_buffer_size: usize,
    
    pub fn init(allocator: std.mem.Allocator, stream: net.Stream, max_size: usize) BufferedWriter {
        return .{
            .stream = stream,
            .buffer = std.ArrayList(u8).init(allocator),
            .max_buffer_size = max_size,
        };
    }
    
    pub fn deinit(self: *BufferedWriter) void {
        self.buffer.deinit();
    }
    
    pub fn write(self: *BufferedWriter, data: []const u8) !void {
        if (self.buffer.items.len + data.len > self.max_buffer_size) {
            // Buffer เต็ม - flush ก่อน
            try self.flush();
            
            if (data.len > self.max_buffer_size) {
                // ข้อมูลใหญ่กว่า buffer - ส่งตรง
                try self.stream.writeAll(data);
                return;
            }
        }
        
        try self.buffer.appendSlice(data);
    }
    
    pub fn flush(self: *BufferedWriter) !void {
        if (self.buffer.items.len == 0) return;
        
        try self.stream.writeAll(self.buffer.items);
        self.buffer.clearRetainingCapacity();
    }
    
    pub fn isPressured(self: *const BufferedWriter) bool {
        return self.buffer.items.len > self.max_buffer_size / 2;
    }
};

// Token bucket rate limiter
pub const TokenBucket = struct {
    capacity: f64,
    tokens: f64,
    refill_rate: f64, // tokens per second
    last_refill: i64,
    mutex: std.Thread.Mutex = .{},
    
    pub fn init(capacity: f64, refill_rate: f64) TokenBucket {
        return .{
            .capacity = capacity,
            .tokens = capacity,
            .refill_rate = refill_rate,
            .last_refill = std.time.timestamp(),
        };
    }
    
    pub fn consume(self: *TokenBucket, count: f64) bool {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        self.refill();
        
        if (self.tokens >= count) {
            self.tokens -= count;
            return true;
        }
        return false;
    }
    
    pub fn waitForTokens(self: *TokenBucket, count: f64) void {
        while (!self.consume(count)) {
            std.time.sleep(10 * std.time.ns_per_ms);
        }
    }
    
    fn refill(self: *TokenBucket) void {
        const now = std.time.timestamp();
        const elapsed = @as(f64, @floatFromInt(now - self.last_refill));
        
        self.tokens = @min(
            self.capacity,
            self.tokens + elapsed * self.refill_rate,
        );
        self.last_refill = now;
    }
};
```

---

## Step 489: Network performance optimization

```zig
const std = @import("std");
const posix = std.posix;
const net = std.net;

// Zero-copy sendfile
pub fn sendFile(sock: posix.socket_t, file: std.fs.File) !void {
    const stat = try file.stat();
    
    var offset: posix.off_t = 0;
    var remaining: usize = stat.size;
    
    while (remaining > 0) {
        const sent = try posix.sendfile(
            sock,
            file.handle,
            &offset,
            @min(remaining, 1024 * 1024), // 1MB chunks
        );
        remaining -= sent;
    }
}

// Vectored I/O (scatter/gather)
pub fn sendVectored(sock: posix.socket_t, parts: []const []const u8) !void {
    const iovecs = try std.heap.page_allocator.alloc(posix.iovec_const, parts.len);
    defer std.heap.page_allocator.free(iovecs);
    
    for (parts, iovecs) |part, *iov| {
        iov.* = .{
            .base = part.ptr,
            .len = part.len,
        };
    }
    
    _ = try posix.writev(sock, iovecs);
}

// Buffer pool เพื่อลด allocations
pub const BufferPool = struct {
    const BUFFER_SIZE = 4096;
    
    mutex: std.Thread.Mutex = .{},
    available: std.ArrayList([]u8),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, pool_size: usize) !BufferPool {
        var pool = BufferPool{
            .available = std.ArrayList([]u8).init(allocator),
            .allocator = allocator,
        };
        
        var i: usize = 0;
        while (i < pool_size) : (i += 1) {
            const buf = try allocator.alloc(u8, BUFFER_SIZE);
            try pool.available.append(buf);
        }
        
        return pool;
    }
    
    pub fn deinit(self: *BufferPool) void {
        for (self.available.items) |buf| {
            self.allocator.free(buf);
        }
        self.available.deinit();
    }
    
    pub fn get(self: *BufferPool) ![]u8 {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        if (self.available.items.len > 0) {
            return self.available.pop();
        }
        
        return self.allocator.alloc(u8, BUFFER_SIZE);
    }
    
    pub fn put(self: *BufferPool, buf: []u8) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        self.available.append(buf) catch {
            self.allocator.free(buf);
        };
    }
};
```

---

## Step 490: Real project: Chat server

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;
const Mutex = Thread.Mutex;

// Chat server ที่รองรับหลาย rooms และ users

const Message = struct {
    room: []const u8,
    sender: []const u8,
    content: []const u8,
    timestamp: i64,
};

const Client = struct {
    id: u32,
    username: []const u8,
    room: []const u8,
    stream: net.Stream,
};

const ChatServer = struct {
    mutex: Mutex = .{},
    clients: std.ArrayList(Client),
    rooms: std.StringHashMap(std.ArrayList(u32)),
    allocator: std.mem.Allocator,
    next_id: std.atomic.Value(u32) = std.atomic.Value(u32).init(1),
    
    pub fn init(allocator: std.mem.Allocator) ChatServer {
        return .{
            .clients = std.ArrayList(Client).init(allocator),
            .rooms = std.StringHashMap(std.ArrayList(u32)).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *ChatServer) void {
        for (self.clients.items) |c| {
            self.allocator.free(c.username);
            self.allocator.free(c.room);
        }
        self.clients.deinit();
        var it = self.rooms.valueIterator();
        while (it.next()) |list| list.deinit();
        self.rooms.deinit();
    }
    
    pub fn addClient(self: *ChatServer, stream: net.Stream, username: []const u8) !u32 {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        const id = self.next_id.fetchAdd(1, .monotonic);
        const uname = try self.allocator.dupe(u8, username);
        const room = try self.allocator.dupe(u8, "general");
        
        try self.clients.append(.{
            .id = id,
            .username = uname,
            .room = room,
            .stream = stream,
        });
        
        // เพิ่มเข้า general room
        const entry = try self.rooms.getOrPut("general");
        if (!entry.found_existing) {
            entry.value_ptr.* = std.ArrayList(u32).init(self.allocator);
        }
        try entry.value_ptr.append(id);
        
        return id;
    }
    
    pub fn removeClient(self: *ChatServer, id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (self.clients.items, 0..) |*c, i| {
            if (c.id == id) {
                // ลบออกจาก room
                if (self.rooms.getPtr(c.room)) |room_members| {
                    for (room_members.items, 0..) |mid, j| {
                        if (mid == id) {
                            _ = room_members.swapRemove(j);
                            break;
                        }
                    }
                }
                
                self.allocator.free(c.username);
                self.allocator.free(c.room);
                _ = self.clients.swapRemove(i);
                return;
            }
        }
    }
    
    pub fn broadcast(self: *ChatServer, room: []const u8, sender_id: u32, msg: []const u8) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        const members = self.rooms.get(room) orelse return;
        
        for (members.items) |mid| {
            if (mid == sender_id) continue; // ไม่ส่งกลับหาตัวเอง
            
            for (self.clients.items) |c| {
                if (c.id == mid) {
                    c.stream.writeAll(msg) catch {};
                    break;
                }
            }
        }
    }
    
    fn handleClientConn(self: *ChatServer, stream: net.Stream) !void {
        defer stream.close();
        
        // ขอ username
        try stream.writeAll("Enter username: ");
        
        var buf: [256]u8 = undefined;
        const n = try stream.read(&buf);
        if (n == 0) return;
        
        const username = std.mem.trimRight(u8, buf[0..n], "\r\n");
        const client_id = try self.addClient(stream, username);
        defer self.removeClient(client_id);
        
        const welcome = try std.fmt.allocPrint(
            self.allocator,
            "Welcome {s}! You are in #general\r\n",
            .{username},
        );
        defer self.allocator.free(welcome);
        try stream.writeAll(welcome);
        
        // Announce join
        const join_msg = try std.fmt.allocPrint(
            self.allocator,
            "{s} joined #general\r\n",
            .{username},
        );
        defer self.allocator.free(join_msg);
        self.broadcast("general", client_id, join_msg);
        
        // Chat loop
        var chat_buf: [1024]u8 = undefined;
        while (true) {
            const bytes = stream.read(&chat_buf) catch break;
            if (bytes == 0) break;
            
            const text = std.mem.trimRight(u8, chat_buf[0..bytes], "\r\n");
            if (text.len == 0) continue;
            
            const full_msg = try std.fmt.allocPrint(
                self.allocator,
                "[{s}]: {s}\r\n",
                .{ username, text },
            );
            defer self.allocator.free(full_msg);
            
            self.broadcast("general", client_id, full_msg);
        }
        
        // Announce leave
        const leave_msg = try std.fmt.allocPrint(
            self.allocator,
            "{s} left #general\r\n",
            .{username},
        );
        defer self.allocator.free(leave_msg);
        self.broadcast("general", client_id, leave_msg);
    }
};

var chat_server: ChatServer = undefined;

fn clientThread(stream: net.Stream) void {
    chat_server.handleClientConn(stream) catch |err| {
        std.debug.print("Client error: {}\n", .{err});
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    chat_server = ChatServer.init(allocator);
    defer chat_server.deinit();
    
    const addr = try net.Address.parseIp4("0.0.0.0", 9000);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    std.debug.print("Chat server on :9000\n", .{});
    std.debug.print("Connect with: nc localhost 9000\n", .{});
    
    while (true) {
        const conn = try server.accept();
        const t = try Thread.spawn(.{}, clientThread, .{conn.stream});
        t.detach();
    }
}
```

---

## สรุป Part 49

ในบทนี้เราได้เรียนรู้:

| Topic | เนื้อหา |
|-------|---------|
| Advanced TCP | Socket options ขั้นสูง |
| Multiple Clients | Thread-per-connection |
| Protocol Design | Header, checksum, versioning |
| Binary Protocol | RESP-like protocol |
| UDP Reliable | ARQ protocol |
| epoll | Event-driven I/O |
| Connection Pool | จัดการ connections |
| Backpressure | Rate limiting, buffering |
| Performance | sendfile, vectored I/O |
| Chat Server | Real-world project |

ใน Part ถัดไปเราจะเรียนรู้ WebSocket ใน Zig!
