# Part 47: Networking พื้นฐาน ใน Zig

## ภาพรวม

Networking เป็นส่วนสำคัญของโปรแกรมสมัยใหม่ Zig มีไลบรารี `std.net` ที่ให้ความสามารถในการสื่อสารผ่านเครือข่ายทั้ง TCP และ UDP ในบทนี้เราจะเรียนรู้การเขียนโปรแกรม network ตั้งแต่พื้นฐาน

---

## Step 461: Networking ใน Zig - std.net

`std.net` เป็น module หลักสำหรับงาน networking ใน Zig

### โครงสร้างของ std.net

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    // ตรวจสอบ IP address
    const addr = try net.Address.parseIp4("127.0.0.1", 8080);
    std.debug.print("Address: {}\n", .{addr});
    
    // IPv6
    const addr6 = try net.Address.parseIp6("::1", 8080);
    std.debug.print("IPv6: {}\n", .{addr6});
    
    // localhost
    const local = try net.Address.parseIp("0.0.0.0", 3000);
    std.debug.print("Any: {}\n", .{local});
}
```

### Network Address Types

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    // IPv4
    const ipv4 = try net.Address.parseIp4("192.168.1.1", 80);
    std.debug.print("IPv4: {}\n", .{ipv4});
    std.debug.print("Port: {d}\n", .{ipv4.getPort()});
    
    // Unix domain socket
    // const unix = try net.Address.initUnix("/tmp/my.sock");
    
    // ตรวจสอบประเภท
    switch (ipv4.any.family) {
        std.posix.AF.INET => std.debug.print("เป็น IPv4\n", .{}),
        std.posix.AF.INET6 => std.debug.print("เป็น IPv6\n", .{}),
        else => std.debug.print("ประเภทอื่น\n", .{}),
    }
}
```

---

## Step 462: TCP Client

TCP (Transmission Control Protocol) ให้การเชื่อมต่อที่เชื่อถือได้และมีลำดับ

### TCP Client พื้นฐาน

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    // เชื่อมต่อไปยัง server
    const address = try net.Address.parseIp4("127.0.0.1", 8080);
    
    const stream = try net.tcpConnectToAddress(address);
    defer stream.close();
    
    std.debug.print("เชื่อมต่อสำเร็จ!\n", .{});
    
    // ส่งข้อมูล
    const message = "Hello, Server!";
    try stream.writeAll(message);
    std.debug.print("ส่ง: {s}\n", .{message});
    
    // รับข้อมูล
    var buffer: [1024]u8 = undefined;
    const bytes_read = try stream.read(&buffer);
    std.debug.print("รับ: {s}\n", .{buffer[0..bytes_read]});
}
```

### TCP Client ที่สมบูรณ์

```zig
const std = @import("std");
const net = std.net;

const TcpClient = struct {
    stream: net.Stream,
    
    pub fn connect(host: []const u8, port: u16) !TcpClient {
        const address = try net.Address.parseIp4(host, port);
        const stream = try net.tcpConnectToAddress(address);
        return TcpClient{ .stream = stream };
    }
    
    pub fn connectByHost(host: []const u8, port: u16) !TcpClient {
        const stream = try net.tcpConnectToHost(std.heap.page_allocator, host, port);
        return TcpClient{ .stream = stream };
    }
    
    pub fn close(self: TcpClient) void {
        self.stream.close();
    }
    
    pub fn send(self: TcpClient, data: []const u8) !void {
        try self.stream.writeAll(data);
    }
    
    pub fn receive(self: TcpClient, buffer: []u8) !usize {
        return self.stream.read(buffer);
    }
    
    pub fn receiveAll(self: TcpClient, buffer: []u8) !usize {
        var total: usize = 0;
        while (total < buffer.len) {
            const n = try self.stream.read(buffer[total..]);
            if (n == 0) break;
            total += n;
        }
        return total;
    }
};

pub fn main() !void {
    const client = TcpClient.connect("127.0.0.1", 8080) catch |err| {
        std.debug.print("ไม่สามารถเชื่อมต่อได้: {}\n", .{err});
        return;
    };
    defer client.close();
    
    // ส่ง HTTP request
    const request = "GET / HTTP/1.1\r\nHost: localhost:8080\r\nConnection: close\r\n\r\n";
    try client.send(request);
    
    // รับ response
    var buffer: [4096]u8 = undefined;
    const n = try client.receive(&buffer);
    std.debug.print("Response ({d} bytes):\n{s}\n", .{ n, buffer[0..n] });
}
```

### HTTP Client แบบง่าย

```zig
const std = @import("std");
const net = std.net;

pub fn httpGet(allocator: std.mem.Allocator, host: []const u8, path: []const u8) ![]u8 {
    const stream = try net.tcpConnectToHost(allocator, host, 80);
    defer stream.close();
    
    // สร้าง HTTP request
    const request = try std.fmt.allocPrint(allocator,
        "GET {s} HTTP/1.1\r\nHost: {s}\r\nConnection: close\r\n\r\n",
        .{ path, host }
    );
    defer allocator.free(request);
    
    try stream.writeAll(request);
    
    // อ่าน response
    var response = std.ArrayList(u8).init(allocator);
    var buf: [4096]u8 = undefined;
    
    while (true) {
        const n = try stream.read(&buf);
        if (n == 0) break;
        try response.appendSlice(buf[0..n]);
    }
    
    return response.toOwnedSlice();
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const response = httpGet(allocator, "example.com", "/") catch |err| {
        std.debug.print("Error: {}\n", .{err});
        return;
    };
    defer allocator.free(response);
    
    std.debug.print("Response:\n{s}\n", .{response[0..@min(500, response.len)]});
}
```

---

## Step 463: TCP Server

TCP Server รับ connections จาก clients และจัดการแต่ละ connection

### TCP Server พื้นฐาน

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    const address = try net.Address.parseIp4("0.0.0.0", 8080);
    
    var server = try address.listen(.{
        .reuse_address = true,
    });
    defer server.deinit();
    
    std.debug.print("Server กำลัง listen ที่ port 8080\n", .{});
    
    while (true) {
        const connection = try server.accept();
        std.debug.print("Client เชื่อมต่อจาก: {}\n", .{connection.address});
        
        handleClient(connection.stream);
        connection.stream.close();
    }
}

fn handleClient(stream: net.Stream) void {
    var buffer: [1024]u8 = undefined;
    
    const n = stream.read(&buffer) catch |err| {
        std.debug.print("Error reading: {}\n", .{err});
        return;
    };
    
    std.debug.print("รับ: {s}\n", .{buffer[0..n]});
    
    const response = "Hello, Client!\n";
    stream.writeAll(response) catch |err| {
        std.debug.print("Error writing: {}\n", .{err});
    };
}
```

### Multi-client TCP Server ด้วย Threads

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;

const ClientContext = struct {
    stream: net.Stream,
    address: net.Address,
    client_id: u32,
};

fn handleClientThread(ctx: ClientContext) void {
    defer ctx.stream.close();
    
    std.debug.print("Client {d} ({}) เชื่อมต่อแล้ว\n", .{ ctx.client_id, ctx.address });
    
    var buffer: [4096]u8 = undefined;
    
    while (true) {
        const n = ctx.stream.read(&buffer) catch break;
        if (n == 0) break;
        
        const received = buffer[0..n];
        std.debug.print("Client {d}: {s}\n", .{ ctx.client_id, received });
        
        // Echo กลับ
        ctx.stream.writeAll(received) catch break;
    }
    
    std.debug.print("Client {d} ตัดการเชื่อมต่อ\n", .{ctx.client_id});
}

var client_counter = std.atomic.Value(u32).init(0);

pub fn main() !void {
    const address = try net.Address.parseIp4("0.0.0.0", 8080);
    
    var server = try address.listen(.{
        .reuse_address = true,
    });
    defer server.deinit();
    
    std.debug.print("Echo server รันที่ port 8080\n", .{});
    
    while (true) {
        const connection = try server.accept();
        const id = client_counter.fetchAdd(1, .monotonic);
        
        const ctx = ClientContext{
            .stream = connection.stream,
            .address = connection.address,
            .client_id = id,
        };
        
        const thread = Thread.spawn(.{}, handleClientThread, .{ctx}) catch |err| {
            std.debug.print("ไม่สามารถสร้าง thread: {}\n", .{err});
            connection.stream.close();
            continue;
        };
        thread.detach();
    }
}
```

---

## Step 464: UDP Client/Server

UDP (User Datagram Protocol) ไม่มีการเชื่อมต่อ ส่งข้อมูลเร็วแต่ไม่รับประกันการส่ง

### UDP Server

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

pub fn udpServer() !void {
    // สร้าง UDP socket
    const sock = try posix.socket(posix.AF.INET, posix.SOCK.DGRAM, 0);
    defer posix.close(sock);
    
    // Bind ที่ port 9090
    const addr = try net.Address.parseIp4("0.0.0.0", 9090);
    try posix.bind(sock, &addr.any, addr.getOsSockLen());
    
    std.debug.print("UDP Server listening ที่ port 9090\n", .{});
    
    var buffer: [1024]u8 = undefined;
    var client_addr: posix.sockaddr = undefined;
    var client_addr_len: posix.socklen_t = @sizeOf(posix.sockaddr);
    
    while (true) {
        const n = try posix.recvfrom(
            sock,
            &buffer,
            0,
            &client_addr,
            &client_addr_len,
        );
        
        const client = net.Address.initPosix(@alignCast(&client_addr));
        std.debug.print("จาก {}: {s}\n", .{ client, buffer[0..n] });
        
        // ส่ง response กลับ
        const response = "UDP ack";
        _ = try posix.sendto(
            sock,
            response,
            0,
            &client_addr,
            client_addr_len,
        );
    }
}
```

### UDP Client

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

pub fn udpClient() !void {
    const sock = try posix.socket(posix.AF.INET, posix.SOCK.DGRAM, 0);
    defer posix.close(sock);
    
    const server_addr = try net.Address.parseIp4("127.0.0.1", 9090);
    
    // ส่งข้อความ
    const messages = [_][]const u8{
        "Hello UDP",
        "Ping",
        "Test message",
    };
    
    for (messages) |msg| {
        _ = try posix.sendto(
            sock,
            msg,
            0,
            &server_addr.any,
            server_addr.getOsSockLen(),
        );
        std.debug.print("ส่ง: {s}\n", .{msg});
        
        // รับ response
        var buffer: [1024]u8 = undefined;
        var from: posix.sockaddr = undefined;
        var from_len: posix.socklen_t = @sizeOf(posix.sockaddr);
        
        const n = try posix.recvfrom(sock, &buffer, 0, &from, &from_len);
        std.debug.print("รับ: {s}\n", .{buffer[0..n]});
        
        std.time.sleep(100 * std.time.ns_per_ms);
    }
}

pub fn main() !void {
    try udpClient();
}
```

---

## Step 465: std.net.Address

`std.net.Address` จัดการ network addresses ทั้ง IPv4 และ IPv6

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    // สร้าง address ต่างๆ
    const ipv4 = try net.Address.parseIp4("192.168.1.100", 8080);
    const ipv6 = try net.Address.parseIp6("2001:db8::1", 443);
    const any4 = try net.Address.parseIp4("0.0.0.0", 0);
    
    // แสดงข้อมูล
    std.debug.print("IPv4: {}\n", .{ipv4});
    std.debug.print("IPv6: {}\n", .{ipv6});
    std.debug.print("Any: {}\n", .{any4});
    
    // เปรียบเทียบ
    const addr1 = try net.Address.parseIp4("10.0.0.1", 80);
    const addr2 = try net.Address.parseIp4("10.0.0.1", 80);
    const addr3 = try net.Address.parseIp4("10.0.0.2", 80);
    
    std.debug.print("addr1 == addr2: {}\n", .{addr1.eql(addr2)});
    std.debug.print("addr1 == addr3: {}\n", .{addr1.eql(addr3)});
    
    // ดู port
    std.debug.print("Port: {d}\n", .{ipv4.getPort()});
    
    // ดู family
    switch (ipv4.any.family) {
        std.posix.AF.INET => {
            const in = ipv4.in;
            std.debug.print("IPv4 bytes: {d}.{d}.{d}.{d}\n", .{
                in.sa.addr >> 24 & 0xff,
                in.sa.addr >> 16 & 0xff,
                in.sa.addr >> 8 & 0xff,
                in.sa.addr & 0xff,
            });
        },
        else => {},
    }
}
```

### Address ใน Network Byte Order

```zig
const std = @import("std");
const net = std.net;

pub fn printAddressInfo(addr: net.Address) void {
    switch (addr.any.family) {
        std.posix.AF.INET => {
            std.debug.print("IPv4: {} (port {})\n", .{ addr, addr.getPort() });
        },
        std.posix.AF.INET6 => {
            std.debug.print("IPv6: {} (port {})\n", .{ addr, addr.getPort() });
        },
        else => {
            std.debug.print("Unknown address family\n", .{});
        },
    }
}

pub fn main() !void {
    const addresses = [_][]const u8{
        "127.0.0.1",
        "10.0.0.1",
        "172.16.0.1",
        "192.168.0.1",
    };
    
    for (addresses) |addr_str| {
        const addr = try net.Address.parseIp4(addr_str, 80);
        printAddressInfo(addr);
    }
}
```

---

## Step 466: DNS Resolution

DNS (Domain Name System) แปลง hostname เป็น IP address

```zig
const std = @import("std");
const net = std.net;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // resolve hostname
    const hostname = "localhost";
    const port = 8080;
    
    // getAddressList คืนรายการ addresses ทั้งหมด
    const address_list = try net.getAddressList(allocator, hostname, port);
    defer address_list.deinit();
    
    std.debug.print("Resolved '{s}':\n", .{hostname});
    for (address_list.addrs) |addr| {
        std.debug.print("  {}\n", .{addr});
    }
}
```

### DNS Lookup ด้วย getaddrinfo

```zig
const std = @import("std");
const net = std.net;

fn resolveHost(allocator: std.mem.Allocator, hostname: []const u8, port: u16) !?net.Address {
    const list = net.getAddressList(allocator, hostname, port) catch |err| {
        std.debug.print("DNS lookup ล้มเหลวสำหรับ '{s}': {}\n", .{ hostname, err });
        return null;
    };
    defer list.deinit();
    
    if (list.addrs.len == 0) {
        std.debug.print("ไม่พบ address สำหรับ '{s}'\n", .{hostname});
        return null;
    }
    
    // คืน address แรกที่พบ
    return list.addrs[0];
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const hosts = [_][]const u8{
        "localhost",
        "127.0.0.1",
    };
    
    for (hosts) |host| {
        if (try resolveHost(allocator, host, 80)) |addr| {
            std.debug.print("{s} -> {}\n", .{ host, addr });
        }
    }
}
```

---

## Step 467: Socket options

Socket options ช่วยปรับแต่งพฤติกรรมของ socket

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

pub fn createOptimizedServer(port: u16) !posix.socket_t {
    const sock = try posix.socket(
        posix.AF.INET,
        posix.SOCK.STREAM,
        posix.IPPROTO.TCP,
    );
    errdefer posix.close(sock);
    
    // SO_REUSEADDR - ใช้ port ซ้ำได้หลัง restart
    try posix.setsockopt(
        sock,
        posix.SOL.SOCKET,
        posix.SO.REUSEADDR,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    // SO_REUSEPORT - หลาย processes ใช้ port เดียวกัน
    // try posix.setsockopt(
    //     sock,
    //     posix.SOL.SOCKET,
    //     posix.SO.REUSEPORT,
    //     &std.mem.toBytes(@as(c_int, 1)),
    // );
    
    // TCP_NODELAY - ปิด Nagle algorithm สำหรับ latency ต่ำ
    try posix.setsockopt(
        sock,
        posix.IPPROTO.TCP,
        std.c.TCP.NODELAY,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    // SO_KEEPALIVE - ตรวจสอบว่า connection ยังมีอยู่
    try posix.setsockopt(
        sock,
        posix.SOL.SOCKET,
        posix.SO.KEEPALIVE,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    // ตั้ง buffer sizes
    const RECV_BUFFER = 65536;
    const SEND_BUFFER = 65536;
    
    try posix.setsockopt(
        sock,
        posix.SOL.SOCKET,
        posix.SO.RCVBUF,
        &std.mem.toBytes(@as(c_int, RECV_BUFFER)),
    );
    
    try posix.setsockopt(
        sock,
        posix.SOL.SOCKET,
        posix.SO.SNDBUF,
        &std.mem.toBytes(@as(c_int, SEND_BUFFER)),
    );
    
    const addr = try net.Address.parseIp4("0.0.0.0", port);
    try posix.bind(sock, &addr.any, addr.getOsSockLen());
    try posix.listen(sock, 128);
    
    return sock;
}

pub fn main() !void {
    const sock = try createOptimizedServer(8080);
    defer posix.close(sock);
    
    std.debug.print("Optimized server ready on port 8080\n", .{});
    
    // ดู socket options ที่ตั้งไว้
    var val: c_int = 0;
    var val_len: posix.socklen_t = @sizeOf(c_int);
    
    try posix.getsockopt(sock, posix.SOL.SOCKET, posix.SO.RCVBUF, std.mem.asBytes(&val), &val_len);
    std.debug.print("Receive buffer: {d} bytes\n", .{val});
    
    try posix.getsockopt(sock, posix.SOL.SOCKET, posix.SO.SNDBUF, std.mem.asBytes(&val), &val_len);
    std.debug.print("Send buffer: {d} bytes\n", .{val});
}
```

---

## Step 468: Non-blocking sockets

Non-blocking sockets ช่วยให้ handle หลาย connections ได้โดยไม่ต้องใช้หลาย threads

```zig
const std = @import("std");
const net = std.net;
const posix = std.posix;

fn setNonBlocking(sock: posix.socket_t) !void {
    const flags = try posix.fcntl(sock, posix.F.GETFL, 0);
    _ = try posix.fcntl(sock, posix.F.SETFL, flags | posix.O.NONBLOCK);
}

pub fn nonBlockingExample() !void {
    const sock = try posix.socket(posix.AF.INET, posix.SOCK.STREAM, 0);
    defer posix.close(sock);
    
    try setNonBlocking(sock);
    
    const addr = try net.Address.parseIp4("127.0.0.1", 8080);
    
    // พยายาม connect - จะ return ทันทีถ้ายังไม่เสร็จ
    posix.connect(sock, &addr.any, addr.getOsSockLen()) catch |err| switch (err) {
        error.WouldBlock => {
            std.debug.print("Connection กำลัง proceed อยู่...\n", .{});
            // ใช้ select/poll/epoll รอ
        },
        else => return err,
    };
}

// Simple non-blocking server ด้วย poll
pub fn nonBlockingServer() !void {
    const server_sock = try posix.socket(posix.AF.INET, posix.SOCK.STREAM, 0);
    defer posix.close(server_sock);
    
    try posix.setsockopt(
        server_sock,
        posix.SOL.SOCKET,
        posix.SO.REUSEADDR,
        &std.mem.toBytes(@as(c_int, 1)),
    );
    
    try setNonBlocking(server_sock);
    
    const addr = try net.Address.parseIp4("0.0.0.0", 8081);
    try posix.bind(server_sock, &addr.any, addr.getOsSockLen());
    try posix.listen(server_sock, 128);
    
    std.debug.print("Non-blocking server on port 8081\n", .{});
    
    var fds = [_]posix.pollfd{
        .{
            .fd = server_sock,
            .events = posix.POLL.IN,
            .revents = 0,
        },
    };
    
    // Poll loop
    var iteration: u32 = 0;
    while (iteration < 5) : (iteration += 1) {
        const ready = try posix.poll(&fds, 1000); // timeout 1 second
        
        if (ready == 0) {
            std.debug.print("Timeout - ไม่มี connection\n", .{});
            continue;
        }
        
        if (fds[0].revents & posix.POLL.IN != 0) {
            var client_addr: posix.sockaddr = undefined;
            var client_len: posix.socklen_t = @sizeOf(posix.sockaddr);
            
            const client_sock = posix.accept(server_sock, &client_addr, &client_len, 0) catch |err| {
                if (err == error.WouldBlock) continue;
                return err;
            };
            defer posix.close(client_sock);
            
            const client_address = net.Address.initPosix(@alignCast(&client_addr));
            std.debug.print("Client: {}\n", .{client_address});
            
            _ = try posix.write(client_sock, "Hello non-blocking!\n");
        }
    }
}
```

---

## Step 469: Error handling ใน networking

Network operations มีโอกาสผิดพลาดสูง ต้องจัดการ errors อย่างระมัดระวัง

```zig
const std = @import("std");
const net = std.net;

const NetworkError = error{
    ConnectionRefused,
    Timeout,
    HostUnreachable,
    AddressInUse,
    TooManyConnections,
};

fn connectWithRetry(
    host: []const u8,
    port: u16,
    max_retries: u32,
    allocator: std.mem.Allocator,
) !net.Stream {
    var attempt: u32 = 0;
    var delay: u64 = 100; // เริ่มต้น 100ms
    
    while (attempt < max_retries) {
        const stream = net.tcpConnectToHost(allocator, host, port) catch |err| {
            attempt += 1;
            std.debug.print("Attempt {d}/{d} ล้มเหลว: {} รอ {d}ms\n", .{
                attempt, max_retries, err, delay,
            });
            
            if (attempt >= max_retries) return err;
            
            std.time.sleep(delay * std.time.ns_per_ms);
            delay = @min(delay * 2, 5000); // Exponential backoff สูงสุด 5s
            continue;
        };
        
        return stream;
    }
    
    return error.ConnectionRefused;
}

fn handleNetworkError(err: anyerror) void {
    switch (err) {
        error.ConnectionRefused => {
            std.debug.print("Server ปฏิเสธการเชื่อมต่อ\n", .{});
        },
        error.NetworkUnreachable => {
            std.debug.print("ไม่สามารถเข้าถึงเครือข่ายได้\n", .{});
        },
        error.ConnectionTimedOut => {
            std.debug.print("หมดเวลาการเชื่อมต่อ\n", .{});
        },
        error.AddressInUse => {
            std.debug.print("Port ถูกใช้งานอยู่แล้ว\n", .{});
        },
        error.ConnectionResetByPeer => {
            std.debug.print("Connection ถูกตัดจาก peer\n", .{});
        },
        else => {
            std.debug.print("Network error: {}\n", .{err});
        },
    }
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // ลองเชื่อมต่อด้วย retry
    const stream = connectWithRetry("127.0.0.1", 8080, 3, allocator) catch |err| {
        handleNetworkError(err);
        return;
    };
    defer stream.close();
    
    std.debug.print("เชื่อมต่อสำเร็จ!\n", .{});
}
```

### Graceful Error Recovery

```zig
const std = @import("std");
const net = std.net;

const ServerResult = union(enum) {
    success: []const u8,
    connection_error: anyerror,
    timeout,
    empty_response,
};

fn sendRequest(stream: net.Stream, request: []const u8, buffer: []u8) ServerResult {
    stream.writeAll(request) catch |err| {
        return .{ .connection_error = err };
    };
    
    const n = stream.read(buffer) catch |err| {
        return .{ .connection_error = err };
    };
    
    if (n == 0) return .empty_response;
    
    return .{ .success = buffer[0..n] };
}

pub fn main() !void {
    const address = net.Address.parseIp4("127.0.0.1", 8080) catch {
        std.debug.print("Invalid address\n", .{});
        return;
    };
    
    const stream = net.tcpConnectToAddress(address) catch |err| {
        std.debug.print("Cannot connect: {}\n", .{err});
        return;
    };
    defer stream.close();
    
    var buffer: [4096]u8 = undefined;
    const result = sendRequest(stream, "ping\n", &buffer);
    
    switch (result) {
        .success => |data| std.debug.print("Success: {s}\n", .{data}),
        .connection_error => |err| std.debug.print("Error: {}\n", .{err}),
        .timeout => std.debug.print("Timeout!\n", .{}),
        .empty_response => std.debug.print("Empty response\n", .{}),
    }
}
```

---

## Step 470: Network protocols พื้นฐาน

### Echo Protocol

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;

fn echoServer(port: u16) !void {
    const addr = try net.Address.parseIp4("0.0.0.0", port);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    std.debug.print("Echo server on :{d}\n", .{port});
    
    while (true) {
        const conn = try server.accept();
        const t = try Thread.spawn(.{}, echoHandler, .{conn.stream});
        t.detach();
    }
}

fn echoHandler(stream: net.Stream) void {
    defer stream.close();
    var buf: [1024]u8 = undefined;
    
    while (true) {
        const n = stream.read(&buf) catch break;
        if (n == 0) break;
        stream.writeAll(buf[0..n]) catch break;
    }
}
```

### Simple Line Protocol

```zig
const std = @import("std");
const net = std.net;

// โปรโตคอลแบบ line-based (เหมือน Redis/Memcached)
const LineReader = struct {
    stream: net.Stream,
    buffer: [8192]u8 = undefined,
    start: usize = 0,
    end: usize = 0,
    
    pub fn readLine(self: *LineReader, allocator: std.mem.Allocator) !?[]u8 {
        while (true) {
            // หา newline ในข้อมูลที่มีอยู่
            const data = self.buffer[self.start..self.end];
            if (std.mem.indexOfScalar(u8, data, '\n')) |pos| {
                const line_end = if (pos > 0 and data[pos - 1] == '\r') pos - 1 else pos;
                const line = try allocator.dupe(u8, data[0..line_end]);
                self.start += pos + 1;
                return line;
            }
            
            // ไม่มี newline - อ่านข้อมูลเพิ่ม
            if (self.start > 0) {
                const remaining = self.end - self.start;
                std.mem.copyForwards(u8, self.buffer[0..remaining], self.buffer[self.start..self.end]);
                self.start = 0;
                self.end = remaining;
            }
            
            const n = try self.stream.read(self.buffer[self.end..]);
            if (n == 0) return null;
            self.end += n;
        }
    }
};

const SimpleServer = struct {
    pub fn handleConnection(stream: net.Stream) !void {
        defer stream.close();
        
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        defer _ = gpa.deinit();
        const allocator = gpa.allocator();
        
        var reader = LineReader{ .stream = stream };
        
        // ส่ง welcome message
        try stream.writeAll("Welcome! Type 'quit' to exit\r\n");
        
        while (true) {
            const line = try reader.readLine(allocator) orelse break;
            defer allocator.free(line);
            
            std.debug.print("Received: {s}\n", .{line});
            
            if (std.mem.eql(u8, line, "quit")) {
                try stream.writeAll("Goodbye!\r\n");
                break;
            } else if (std.mem.eql(u8, line, "ping")) {
                try stream.writeAll("pong\r\n");
            } else if (std.mem.eql(u8, line, "time")) {
                var buf: [64]u8 = undefined;
                const time_str = try std.fmt.bufPrint(&buf, "time: {d}\r\n", .{std.time.timestamp()});
                try stream.writeAll(time_str);
            } else {
                const response = try std.fmt.allocPrint(allocator, "echo: {s}\r\n", .{line});
                defer allocator.free(response);
                try stream.writeAll(response);
            }
        }
    }
};

pub fn main() !void {
    const addr = try net.Address.parseIp4("0.0.0.0", 8080);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    std.debug.print("Line protocol server on :8080\n", .{});
    std.debug.print("Commands: ping, time, quit\n", .{});
    
    while (true) {
        const conn = try server.accept();
        std.debug.print("New connection: {}\n", .{conn.address});
        
        SimpleServer.handleConnection(conn.stream) catch |err| {
            std.debug.print("Error: {}\n", .{err});
        };
    }
}
```

---

## สรุป Part 47

ในบทนี้เราได้เรียนรู้:

| Topic | เนื้อหา |
|-------|---------|
| `std.net` | Network library พื้นฐาน |
| TCP Client | เชื่อมต่อและส่งข้อมูล |
| TCP Server | รับ connections |
| UDP | Connectionless communication |
| `net.Address` | จัดการ network addresses |
| DNS | Hostname resolution |
| Socket Options | ปรับแต่ง socket |
| Non-blocking | Async socket operations |
| Error handling | จัดการ network errors |
| Protocols | Line protocol, Echo |

### ข้อควรระวัง

1. **Resource Management**: ปิด socket เสมอด้วย `defer close()`
2. **Error Handling**: Network errors เกิดบ่อย ต้องจัดการทุก case
3. **Timeouts**: ตั้ง timeout เพื่อป้องกันการ block ตลอดไป
4. **Buffer Sizes**: เลือก buffer size ให้เหมาะกับ use case

ใน Part ถัดไปเราจะสร้าง HTTP Server ใน Zig!
