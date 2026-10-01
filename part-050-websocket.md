# Part 50: WebSocket ใน Zig

## ภาพรวม

WebSocket เป็น protocol ที่ให้การสื่อสารแบบ full-duplex ระหว่าง client (browser) กับ server ผ่าน TCP connection เดียว ต่างจาก HTTP ที่เป็น request-response WebSocket อนุญาตให้ server push ข้อมูลหา client ได้โดยตรง ในบทนี้เราจะ implement WebSocket server ตั้งแต่ต้นใน Zig

---

## Step 491: WebSocket protocol overview

### ทำไมต้อง WebSocket

```
HTTP (Request-Response):
Client -> [Request] -> Server
Client <- [Response] <- Server
(ต้องส่ง request ทุกครั้งที่ต้องการข้อมูล)

WebSocket (Full-duplex):
Client <-> [Persistent Connection] <-> Server
(ทั้งสองฝั่งส่งข้อมูลได้ทันทีเมื่อต้องการ)
```

### WebSocket vs HTTP vs SSE

```zig
const std = @import("std");

pub fn main() void {
    // WebSocket use cases:
    // - Real-time chat
    // - Live gaming
    // - Financial tickers (ราคาหุ้น)
    // - Collaborative editing
    // - IoT sensor data
    // - Live notifications
    
    std.debug.print(
        \\WebSocket Protocol Overview:
        \\- เริ่มต้นด้วย HTTP Upgrade
        \\- ใช้ TCP connection เดียวกัน
        \\- Full-duplex communication
        \\- Low latency
        \\- Framing protocol สำหรับ messages
        \\
    , .{});
}
```

### WebSocket URL Scheme

```
ws://localhost:8080/chat        (unencrypted)
wss://example.com/chat          (TLS encrypted)
```

### RFC 6455 - WebSocket Protocol

WebSocket ถูกกำหนดใน RFC 6455 ประกอบด้วย:
1. **Handshake**: HTTP Upgrade
2. **Framing**: บรรจุข้อมูลในรูปแบบ frames
3. **Messages**: อาจประกอบด้วยหลาย frames
4. **Control frames**: Ping, Pong, Close

---

## Step 492: WebSocket handshake

### HTTP Upgrade Request จาก Client

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat
```

### HTTP Upgrade Response จาก Server

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### การคำนวณ Sec-WebSocket-Accept

```zig
const std = @import("std");

const WEBSOCKET_MAGIC = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

pub fn computeAcceptKey(client_key: []const u8, buf: []u8) ![]const u8 {
    // 1. รวม key กับ magic string
    var combined: [128]u8 = undefined;
    const combined_str = try std.fmt.bufPrint(&combined, "{s}{s}", .{
        client_key, WEBSOCKET_MAGIC,
    });
    
    // 2. SHA-1 hash
    var hash: [20]u8 = undefined;
    std.crypto.hash.Sha1.hash(combined_str, &hash, .{});
    
    // 3. Base64 encode
    const encoded = std.base64.standard.Encoder.encode(buf, &hash);
    return encoded;
}

pub fn performHandshake(stream: anytype, request: []const u8, allocator: std.mem.Allocator) !bool {
    // ค้นหา Sec-WebSocket-Key
    const key_header = "Sec-WebSocket-Key: ";
    const key_start = std.mem.indexOf(u8, request, key_header) orelse return false;
    const key_value_start = key_start + key_header.len;
    const key_end = std.mem.indexOf(u8, request[key_value_start..], "\r\n") orelse return false;
    const client_key = std.mem.trim(u8, request[key_value_start..key_value_start + key_end], " ");
    
    // คำนวณ accept key
    var accept_buf: [64]u8 = undefined;
    const accept_key = try computeAcceptKey(client_key, &accept_buf);
    
    // ส่ง response
    const response = try std.fmt.allocPrint(
        allocator,
        "HTTP/1.1 101 Switching Protocols\r\n" ++
        "Upgrade: websocket\r\n" ++
        "Connection: Upgrade\r\n" ++
        "Sec-WebSocket-Accept: {s}\r\n\r\n",
        .{accept_key},
    );
    defer allocator.free(response);
    
    try stream.writeAll(response);
    std.debug.print("WebSocket handshake สำเร็จ\n", .{});
    return true;
}

pub fn isWebSocketRequest(request: []const u8) bool {
    return std.mem.indexOf(u8, request, "Upgrade: websocket") != null;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    _ = allocator;
    
    // ทดสอบการคำนวณ accept key
    const client_key = "dGhlIHNhbXBsZSBub25jZQ==";
    var buf: [64]u8 = undefined;
    const accept = try computeAcceptKey(client_key, &buf);
    
    // ผลลัพธ์ที่ถูกต้องตาม RFC 6455: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
    std.debug.print("Accept key: {s}\n", .{accept});
    std.debug.print("Expected:   s3pPLMBiTxaQ9kYGzzhZRbK+xOo=\n", .{});
}
```

---

## Step 493: WebSocket frame format

### Frame Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - -+-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

```zig
const std = @import("std");

pub const Opcode = enum(u4) {
    continuation = 0x0,
    text = 0x1,
    binary = 0x2,
    // 0x3-0x7: reserved
    close = 0x8,
    ping = 0x9,
    pong = 0xA,
    // 0xB-0xF: reserved
};

pub const Frame = struct {
    fin: bool,          // Final fragment
    rsv1: bool = false, // Reserved (ใช้สำหรับ extensions)
    rsv2: bool = false,
    rsv3: bool = false,
    opcode: Opcode,
    masked: bool,       // Client -> Server ต้อง masked
    payload: []const u8,
    mask_key: [4]u8 = .{0, 0, 0, 0},
    
    pub fn encode(self: Frame, allocator: std.mem.Allocator) ![]u8 {
        const payload_len = self.payload.len;
        
        // คำนวณขนาด header
        var header_size: usize = 2;
        if (payload_len > 65535) {
            header_size += 8;
        } else if (payload_len > 125) {
            header_size += 2;
        }
        if (self.masked) header_size += 4;
        
        const buf = try allocator.alloc(u8, header_size + payload_len);
        errdefer allocator.free(buf);
        
        // Byte 0: FIN + RSV + Opcode
        buf[0] = 0;
        if (self.fin) buf[0] |= 0x80;
        if (self.rsv1) buf[0] |= 0x40;
        if (self.rsv2) buf[0] |= 0x20;
        if (self.rsv3) buf[0] |= 0x10;
        buf[0] |= @intFromEnum(self.opcode);
        
        // Byte 1: MASK + Payload length
        var pos: usize = 1;
        buf[pos] = if (self.masked) 0x80 else 0;
        
        if (payload_len > 65535) {
            buf[pos] |= 127;
            pos += 1;
            std.mem.writeInt(u64, buf[pos..pos+8][0..8], @as(u64, payload_len), .big);
            pos += 8;
        } else if (payload_len > 125) {
            buf[pos] |= 126;
            pos += 1;
            std.mem.writeInt(u16, buf[pos..pos+2][0..2], @as(u16, payload_len), .big);
            pos += 2;
        } else {
            buf[pos] |= @as(u8, @intCast(payload_len));
            pos += 1;
        }
        
        // Masking key
        if (self.masked) {
            @memcpy(buf[pos..pos+4], &self.mask_key);
            pos += 4;
        }
        
        // Payload
        @memcpy(buf[pos..pos+payload_len], self.payload);
        
        // Apply mask
        if (self.masked) {
            for (buf[pos..pos+payload_len], 0..) |*b, i| {
                b.* ^= self.mask_key[i % 4];
            }
        }
        
        return buf;
    }
    
    pub fn decode(data: []const u8, allocator: std.mem.Allocator) !struct { frame: Frame, consumed: usize } {
        if (data.len < 2) return error.InsufficientData;
        
        const byte0 = data[0];
        const byte1 = data[1];
        
        const fin = (byte0 & 0x80) != 0;
        const rsv1 = (byte0 & 0x40) != 0;
        const rsv2 = (byte0 & 0x20) != 0;
        const rsv3 = (byte0 & 0x10) != 0;
        const opcode = std.meta.intToEnum(Opcode, byte0 & 0x0F) catch return error.InvalidOpcode;
        const masked = (byte1 & 0x80) != 0;
        
        var payload_len: u64 = byte1 & 0x7F;
        var pos: usize = 2;
        
        if (payload_len == 126) {
            if (data.len < pos + 2) return error.InsufficientData;
            payload_len = std.mem.readInt(u16, data[pos..pos+2][0..2], .big);
            pos += 2;
        } else if (payload_len == 127) {
            if (data.len < pos + 8) return error.InsufficientData;
            payload_len = std.mem.readInt(u64, data[pos..pos+8][0..8], .big);
            pos += 8;
        }
        
        var mask_key = [4]u8{0, 0, 0, 0};
        if (masked) {
            if (data.len < pos + 4) return error.InsufficientData;
            @memcpy(&mask_key, data[pos..pos+4]);
            pos += 4;
        }
        
        if (data.len < pos + payload_len) return error.InsufficientData;
        
        const payload = try allocator.alloc(u8, payload_len);
        errdefer allocator.free(payload);
        
        @memcpy(payload, data[pos..pos+payload_len]);
        
        // Unmask
        if (masked) {
            for (payload, 0..) |*b, i| {
                b.* ^= mask_key[i % 4];
            }
        }
        
        return .{
            .frame = .{
                .fin = fin,
                .rsv1 = rsv1,
                .rsv2 = rsv2,
                .rsv3 = rsv3,
                .opcode = opcode,
                .masked = masked,
                .payload = payload,
                .mask_key = mask_key,
            },
            .consumed = pos + payload_len,
        };
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    // สร้าง text frame
    const payload = "Hello, WebSocket!";
    const frame = Frame{
        .fin = true,
        .opcode = .text,
        .masked = false,
        .payload = payload,
    };
    
    const encoded = try frame.encode(allocator);
    defer allocator.free(encoded);
    
    std.debug.print("Encoded frame ({d} bytes):\n", .{encoded.len});
    for (encoded) |b| {
        std.debug.print("{x:0>2} ", .{b});
    }
    std.debug.print("\n", .{});
    
    // Decode กลับ
    const result = try Frame.decode(encoded, allocator);
    defer allocator.free(result.frame.payload);
    
    std.debug.print("Decoded: opcode={}, payload='{s}'\n", .{
        result.frame.opcode, result.frame.payload,
    });
}
```

---

## Step 494: Sending/receiving messages

```zig
const std = @import("std");
const net = std.net;

pub const WebSocketConn = struct {
    stream: net.Stream,
    allocator: std.mem.Allocator,
    recv_buf: std.ArrayList(u8),
    is_server: bool,
    closed: bool = false,
    
    pub fn init(stream: net.Stream, allocator: std.mem.Allocator, is_server: bool) WebSocketConn {
        return .{
            .stream = stream,
            .allocator = allocator,
            .recv_buf = std.ArrayList(u8).init(allocator),
            .is_server = is_server,
        };
    }
    
    pub fn deinit(self: *WebSocketConn) void {
        self.recv_buf.deinit();
    }
    
    pub fn sendText(self: *WebSocketConn, text: []const u8) !void {
        try self.sendFrame(.text, text, false);
    }
    
    pub fn sendBinary(self: *WebSocketConn, data: []const u8) !void {
        try self.sendFrame(.binary, data, false);
    }
    
    pub fn sendFrame(self: *WebSocketConn, opcode: Opcode, payload: []const u8, is_fragment: bool) !void {
        const frame = Frame{
            .fin = !is_fragment,
            .opcode = opcode,
            .masked = !self.is_server, // Client ต้อง mask, Server ไม่ต้อง
            .payload = payload,
            .mask_key = if (!self.is_server) generateMaskKey() else .{0,0,0,0},
        };
        
        const encoded = try frame.encode(self.allocator);
        defer self.allocator.free(encoded);
        
        try self.stream.writeAll(encoded);
    }
    
    pub fn receiveMessage(self: *WebSocketConn) !?WebSocketMessage {
        var message_data = std.ArrayList(u8).init(self.allocator);
        errdefer message_data.deinit();
        
        var message_opcode: Opcode = .continuation;
        
        while (true) {
            // อ่าน frame
            var header_buf: [14]u8 = undefined;
            const n = self.stream.read(header_buf[0..2]) catch return null;
            if (n < 2) return null;
            
            const byte0 = header_buf[0];
            const byte1 = header_buf[1];
            
            const fin = (byte0 & 0x80) != 0;
            const opcode_val = byte0 & 0x0F;
            const masked = (byte1 & 0x80) != 0;
            var payload_len: u64 = byte1 & 0x7F;
            
            if (payload_len == 126) {
                _ = try self.stream.read(header_buf[0..2]);
                payload_len = std.mem.readInt(u16, header_buf[0..2], .big);
            } else if (payload_len == 127) {
                _ = try self.stream.read(header_buf[0..8]);
                payload_len = std.mem.readInt(u64, header_buf[0..8], .big);
            }
            
            var mask_key = [4]u8{0, 0, 0, 0};
            if (masked) {
                _ = try self.stream.read(&mask_key);
            }
            
            // อ่าน payload
            const payload = try self.allocator.alloc(u8, payload_len);
            defer self.allocator.free(payload);
            
            _ = try self.stream.readAll(payload);
            
            // Unmask
            if (masked) {
                for (payload, 0..) |*b, i| {
                    b.* ^= mask_key[i % 4];
                }
            }
            
            const opcode = std.meta.intToEnum(Opcode, opcode_val) catch return error.InvalidOpcode;
            
            // จัดการ control frames
            switch (opcode) {
                .ping => {
                    try self.sendFrame(.pong, payload, false);
                    continue;
                },
                .pong => continue,
                .close => {
                    self.closed = true;
                    return .{ .kind = .close, .data = &.{} };
                },
                .continuation => {
                    try message_data.appendSlice(payload);
                },
                .text, .binary => {
                    message_opcode = opcode;
                    try message_data.appendSlice(payload);
                },
            }
            
            if (fin) {
                return .{
                    .kind = if (message_opcode == .text) .text else .binary,
                    .data = try message_data.toOwnedSlice(),
                };
            }
        }
    }
};

pub const MessageKind = enum { text, binary, close };

pub const WebSocketMessage = struct {
    kind: MessageKind,
    data: []const u8,
};

fn generateMaskKey() [4]u8 {
    var key: [4]u8 = undefined;
    std.crypto.random.bytes(&key);
    return key;
}

// Opcode ต้องนำมาจาก step ก่อน
const Opcode = enum(u4) {
    continuation = 0x0,
    text = 0x1,
    binary = 0x2,
    close = 0x8,
    ping = 0x9,
    pong = 0xA,
};

const Frame = struct {
    fin: bool,
    opcode: Opcode,
    masked: bool,
    payload: []const u8,
    mask_key: [4]u8,
    rsv1: bool = false,
    rsv2: bool = false,
    rsv3: bool = false,
    
    pub fn encode(self: Frame, allocator: std.mem.Allocator) ![]u8 {
        const payload_len = self.payload.len;
        var header_size: usize = 2;
        if (payload_len > 65535) header_size += 8
        else if (payload_len > 125) header_size += 2;
        if (self.masked) header_size += 4;
        
        const buf = try allocator.alloc(u8, header_size + payload_len);
        buf[0] = if (self.fin) @as(u8, 0x80) else 0;
        buf[0] |= @intFromEnum(self.opcode);
        
        var pos: usize = 1;
        buf[pos] = if (self.masked) @as(u8, 0x80) else 0;
        
        if (payload_len > 65535) {
            buf[pos] |= 127; pos += 1;
            std.mem.writeInt(u64, buf[pos..pos+8][0..8], payload_len, .big); pos += 8;
        } else if (payload_len > 125) {
            buf[pos] |= 126; pos += 1;
            std.mem.writeInt(u16, buf[pos..pos+2][0..2], @as(u16, @intCast(payload_len)), .big); pos += 2;
        } else {
            buf[pos] |= @as(u8, @intCast(payload_len)); pos += 1;
        }
        
        if (self.masked) {
            @memcpy(buf[pos..pos+4], &self.mask_key); pos += 4;
        }
        
        @memcpy(buf[pos..pos+payload_len], self.payload);
        if (self.masked) {
            for (buf[pos..pos+payload_len], 0..) |*b, i| b.* ^= self.mask_key[i % 4];
        }
        
        return buf;
    }
};
```

---

## Step 495: WebSocket fragmentation

ข้อความขนาดใหญ่สามารถแบ่งเป็นหลาย fragments ได้

```zig
const std = @import("std");

pub const FragmentedSender = struct {
    conn: *WebSocketConn,
    fragment_size: usize,
    
    pub const WebSocketConn = struct {
        stream: anytype,
        allocator: std.mem.Allocator,
        
        pub fn sendRawFrame(self: *@This(), data: []const u8) !void {
            try self.stream.writeAll(data);
        }
    };
    
    pub fn sendLargeText(self: *FragmentedSender, text: []const u8) !void {
        if (text.len <= self.fragment_size) {
            // ส่งเป็น single frame
            try self.sendSingleFrame(.text, text, true);
            return;
        }
        
        // ส่ง fragment แรก
        try self.sendSingleFrame(.text, text[0..self.fragment_size], false);
        
        var offset = self.fragment_size;
        while (offset < text.len) {
            const end = @min(offset + self.fragment_size, text.len);
            const is_final = end == text.len;
            
            try self.sendSingleFrame(.continuation, text[offset..end], is_final);
            offset = end;
        }
    }
    
    fn sendSingleFrame(
        self: *FragmentedSender,
        opcode: Opcode,
        payload: []const u8,
        fin: bool,
    ) !void {
        _ = self;
        _ = opcode;
        _ = payload;
        _ = fin;
        // Implementation ตาม Frame.encode()
    }
};

const Opcode = enum(u4) {
    continuation = 0x0,
    text = 0x1,
    binary = 0x2,
    close = 0x8,
    ping = 0x9,
    pong = 0xA,
};

// ตัวอย่างการส่งข้อมูลขนาดใหญ่
pub fn demonstrateFragmentation() void {
    // Frame sequence สำหรับข้อความ "Hello World" แบ่งเป็น 3 fragments:
    
    // Fragment 1: FIN=0, opcode=TEXT, payload="Hel"
    // |0|0|0|0|0001|0|0000011| = 0x01 0x03 "Hel"
    
    // Fragment 2: FIN=0, opcode=CONTINUATION, payload="lo "
    // |0|0|0|0|0000|0|0000011| = 0x00 0x03 "lo "
    
    // Fragment 3: FIN=1, opcode=CONTINUATION, payload="World"
    // |1|0|0|0|0000|0|0000101| = 0x80 0x05 "World"
    
    std.debug.print("Fragmentation example:\n", .{});
    std.debug.print("Frame 1: FIN=0, opcode=TEXT\n", .{});
    std.debug.print("Frame 2: FIN=0, opcode=CONTINUATION\n", .{});
    std.debug.print("Frame 3: FIN=1, opcode=CONTINUATION\n", .{});
}
```

---

## Step 496: WebSocket ping/pong

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;

pub const HeartbeatManager = struct {
    conn: net.Stream,
    interval_s: u64,
    timeout_s: u64,
    last_pong: std.atomic.Value(i64),
    running: std.atomic.Value(bool),
    allocator: std.mem.Allocator,
    
    pub fn init(
        conn: net.Stream,
        allocator: std.mem.Allocator,
        interval_s: u64,
        timeout_s: u64,
    ) HeartbeatManager {
        return .{
            .conn = conn,
            .interval_s = interval_s,
            .timeout_s = timeout_s,
            .last_pong = std.atomic.Value(i64).init(std.time.timestamp()),
            .running = std.atomic.Value(bool).init(true),
            .allocator = allocator,
        };
    }
    
    pub fn start(self: *HeartbeatManager) !Thread {
        return Thread.spawn(.{}, heartbeatLoop, .{self});
    }
    
    pub fn stop(self: *HeartbeatManager) void {
        self.running.store(false, .release);
    }
    
    pub fn onPongReceived(self: *HeartbeatManager) void {
        self.last_pong.store(std.time.timestamp(), .release);
    }
    
    pub fn isAlive(self: *const HeartbeatManager) bool {
        const last = self.last_pong.load(.acquire);
        const now = std.time.timestamp();
        return now - last < @as(i64, @intCast(self.timeout_s));
    }
    
    fn heartbeatLoop(self: *HeartbeatManager) void {
        while (self.running.load(.acquire)) {
            std.time.sleep(self.interval_s * std.time.ns_per_s);
            
            if (!self.isAlive()) {
                std.debug.print("Connection ไม่มีการตอบสนอง - ปิด connection\n", .{});
                self.conn.close();
                self.running.store(false, .release);
                return;
            }
            
            // ส่ง Ping frame
            const ping_frame = buildPingFrame(self.allocator) catch {
                self.running.store(false, .release);
                return;
            };
            defer self.allocator.free(ping_frame);
            
            self.conn.writeAll(ping_frame) catch {
                self.running.store(false, .release);
                return;
            };
            
            std.debug.print("Ping ส่งแล้ว\n", .{});
        }
    }
    
    fn buildPingFrame(allocator: std.mem.Allocator) ![]u8 {
        const buf = try allocator.alloc(u8, 2);
        buf[0] = 0x89; // FIN=1, opcode=ping (0x9)
        buf[1] = 0x00; // no mask, payload_len=0
        return buf;
    }
};

pub fn buildPongFrame(payload: []const u8, allocator: std.mem.Allocator) ![]u8 {
    const buf = try allocator.alloc(u8, 2 + payload.len);
    buf[0] = 0x8A; // FIN=1, opcode=pong (0xA)
    buf[1] = @intCast(payload.len);
    if (payload.len > 0) {
        @memcpy(buf[2..], payload);
    }
    return buf;
}
```

---

## Step 497: WebSocket close handshake

```zig
const std = @import("std");
const net = std.net;

pub const CloseCode = enum(u16) {
    normal = 1000,
    going_away = 1001,
    protocol_error = 1002,
    unsupported_data = 1003,
    no_status = 1005,
    abnormal_closure = 1006,
    invalid_payload = 1007,
    policy_violation = 1008,
    message_too_big = 1009,
    mandatory_extension = 1010,
    internal_error = 1011,
    service_restart = 1012,
    try_again_later = 1013,
    tls_handshake_failure = 1015,
};

pub fn buildCloseFrame(
    code: CloseCode,
    reason: []const u8,
    allocator: std.mem.Allocator,
) ![]u8 {
    const payload_len = 2 + reason.len;
    const buf = try allocator.alloc(u8, 2 + payload_len);
    
    buf[0] = 0x88; // FIN=1, opcode=close (0x8)
    buf[1] = @intCast(payload_len);
    
    std.mem.writeInt(u16, buf[2..4], @intFromEnum(code), .big);
    if (reason.len > 0) {
        @memcpy(buf[4..], reason);
    }
    
    return buf;
}

pub fn parseCloseFrame(payload: []const u8) struct { code: u16, reason: []const u8 } {
    if (payload.len < 2) {
        return .{ .code = 1005, .reason = "" };
    }
    
    const code = std.mem.readInt(u16, payload[0..2], .big);
    const reason = if (payload.len > 2) payload[2..] else "";
    
    return .{ .code = code, .reason = reason };
}

pub const CloseHandshake = struct {
    stream: net.Stream,
    allocator: std.mem.Allocator,
    state: enum { open, closing, closed } = .open,
    
    pub fn initiateClose(self: *CloseHandshake, code: CloseCode, reason: []const u8) !void {
        if (self.state != .open) return;
        
        self.state = .closing;
        
        const close_frame = try buildCloseFrame(code, reason, self.allocator);
        defer self.allocator.free(close_frame);
        
        try self.stream.writeAll(close_frame);
        std.debug.print("ส่ง Close frame: code={d}, reason='{s}'\n", .{
            @intFromEnum(code), reason,
        });
    }
    
    pub fn respondToClose(self: *CloseHandshake, payload: []const u8) !void {
        const info = parseCloseFrame(payload);
        std.debug.print("รับ Close: code={d}, reason='{s}'\n", .{
            info.code, info.reason,
        });
        
        if (self.state == .open) {
            // ส่ง Close กลับ (echo)
            const close_frame = try buildCloseFrame(
                @enumFromInt(info.code),
                info.reason,
                self.allocator,
            );
            defer self.allocator.free(close_frame);
            
            try self.stream.writeAll(close_frame);
        }
        
        self.state = .closed;
        self.stream.close();
    }
};
```

---

## Step 498: WebSocket security

```zig
const std = @import("std");

// WebSocket Security Considerations

pub const SecurityValidator = struct {
    allowed_origins: []const []const u8,
    max_message_size: usize,
    
    pub fn validateOrigin(self: *SecurityValidator, origin: []const u8) bool {
        for (self.allowed_origins) |allowed| {
            if (std.mem.eql(u8, origin, allowed)) return true;
        }
        return false;
    }
    
    pub fn validateMessageSize(self: *SecurityValidator, size: usize) bool {
        return size <= self.max_message_size;
    }
    
    pub fn sanitizeInput(input: []const u8, allocator: std.mem.Allocator) ![]u8 {
        var output = std.ArrayList(u8).init(allocator);
        errdefer output.deinit();
        
        for (input) |c| {
            switch (c) {
                '<' => try output.appendSlice("&lt;"),
                '>' => try output.appendSlice("&gt;"),
                '&' => try output.appendSlice("&amp;"),
                '"' => try output.appendSlice("&quot;"),
                '\'' => try output.appendSlice("&#x27;"),
                else => try output.append(c),
            }
        }
        
        return output.toOwnedSlice();
    }
};

// Rate limiting สำหรับ WebSocket
pub const WsRateLimiter = struct {
    const ClientState = struct {
        message_count: u32,
        window_start: i64,
        bytes_this_second: u64,
    };
    
    mutex: std.Thread.Mutex = .{},
    clients: std.AutoHashMap(u32, ClientState),
    max_messages_per_second: u32,
    max_bytes_per_second: u64,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, max_msg: u32, max_bytes: u64) WsRateLimiter {
        return .{
            .clients = std.AutoHashMap(u32, ClientState).init(allocator),
            .max_messages_per_second = max_msg,
            .max_bytes_per_second = max_bytes,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *WsRateLimiter) void {
        self.clients.deinit();
    }
    
    pub fn checkLimit(self: *WsRateLimiter, client_id: u32, message_size: usize) !bool {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        const now = std.time.timestamp();
        
        const state = try self.clients.getOrPut(client_id);
        if (!state.found_existing) {
            state.value_ptr.* = .{
                .message_count = 0,
                .window_start = now,
                .bytes_this_second = 0,
            };
        }
        
        var s = state.value_ptr;
        
        // Reset window ถ้าผ่าน 1 วินาทีไปแล้ว
        if (now - s.window_start >= 1) {
            s.message_count = 0;
            s.bytes_this_second = 0;
            s.window_start = now;
        }
        
        // ตรวจสอบ limits
        if (s.message_count >= self.max_messages_per_second) return false;
        if (s.bytes_this_second + message_size > self.max_bytes_per_second) return false;
        
        s.message_count += 1;
        s.bytes_this_second += message_size;
        
        return true;
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const validator = SecurityValidator{
        .allowed_origins = &.{ "http://localhost:3000", "https://myapp.com" },
        .max_message_size = 64 * 1024,
    };
    _ = validator;
    
    var rate_limiter = WsRateLimiter.init(allocator, 100, 1024 * 1024);
    defer rate_limiter.deinit();
    
    // ทดสอบ
    const can_send = try rate_limiter.checkLimit(1, 100);
    std.debug.print("Can send: {}\n", .{can_send});
    
    // Sanitize user input
    const dangerous = "<script>alert('xss')</script>";
    const safe = try SecurityValidator.sanitizeInput(dangerous, allocator);
    defer allocator.free(safe);
    std.debug.print("Sanitized: {s}\n", .{safe});
}
```

---

## Step 499: Real-time application patterns

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;
const Mutex = Thread.Mutex;

// Pub/Sub pattern สำหรับ WebSocket
pub const Channel = struct {
    name: []const u8,
    subscribers: std.ArrayList(*WsClient),
    mutex: Mutex = .{},
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, name: []const u8) Channel {
        return .{
            .name = name,
            .subscribers = std.ArrayList(*WsClient).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Channel) void {
        self.subscribers.deinit();
    }
    
    pub fn subscribe(self: *Channel, client: *WsClient) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        try self.subscribers.append(client);
        std.debug.print("Client {d} subscribed to #{s}\n", .{ client.id, self.name });
    }
    
    pub fn unsubscribe(self: *Channel, client_id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        for (self.subscribers.items, 0..) |s, i| {
            if (s.id == client_id) {
                _ = self.subscribers.swapRemove(i);
                return;
            }
        }
    }
    
    pub fn publish(self: *Channel, message: []const u8) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (self.subscribers.items) |client| {
            client.sendText(message) catch |err| {
                std.debug.print("Send error to {d}: {}\n", .{ client.id, err });
            };
        }
    }
    
    pub fn publishExcept(self: *Channel, message: []const u8, exclude_id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (self.subscribers.items) |client| {
            if (client.id == exclude_id) continue;
            client.sendText(message) catch {};
        }
    }
};

pub const WsClient = struct {
    id: u32,
    stream: net.Stream,
    username: []const u8,
    allocator: std.mem.Allocator,
    
    pub fn sendText(self: *WsClient, text: []const u8) !void {
        // Build WebSocket text frame
        const frame = try buildTextFrame(text, self.allocator);
        defer self.allocator.free(frame);
        try self.stream.writeAll(frame);
    }
    
    fn buildTextFrame(text: []const u8, allocator: std.mem.Allocator) ![]u8 {
        var buf = std.ArrayList(u8).init(allocator);
        errdefer buf.deinit();
        
        // FIN=1, opcode=text
        try buf.append(0x81);
        
        if (text.len < 126) {
            try buf.append(@intCast(text.len));
        } else if (text.len < 65536) {
            try buf.append(126);
            try buf.append(@intCast(text.len >> 8));
            try buf.append(@intCast(text.len & 0xFF));
        } else {
            try buf.append(127);
            var i: u3 = 7;
            while (true) {
                try buf.append(@intCast((text.len >> (@as(u6, i) * 8)) & 0xFF));
                if (i == 0) break;
                i -= 1;
            }
        }
        
        try buf.appendSlice(text);
        return buf.toOwnedSlice();
    }
};

// Event-driven message router
pub const MessageRouter = struct {
    channels: std.StringHashMap(Channel),
    mutex: Mutex = .{},
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) MessageRouter {
        return .{
            .channels = std.StringHashMap(Channel).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *MessageRouter) void {
        var it = self.channels.valueIterator();
        while (it.next()) |ch| ch.deinit();
        self.channels.deinit();
    }
    
    pub fn getOrCreateChannel(self: *MessageRouter, name: []const u8) !*Channel {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        const entry = try self.channels.getOrPut(name);
        if (!entry.found_existing) {
            entry.value_ptr.* = Channel.init(self.allocator, name);
        }
        return entry.value_ptr;
    }
    
    pub fn routeMessage(self: *MessageRouter, channel_name: []const u8, message: []const u8, sender_id: u32) void {
        self.mutex.lock();
        const channel = self.channels.getPtr(channel_name);
        self.mutex.unlock();
        
        if (channel) |ch| {
            ch.publishExcept(message, sender_id);
        }
    }
};
```

---

## Step 500: Project: Real-time chat with WebSocket

```zig
const std = @import("std");
const net = std.net;
const Thread = std.Thread;
const Mutex = Thread.Mutex;

const WEBSOCKET_MAGIC = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

// Complete WebSocket Chat Server

const ChatMessage = struct {
    event: []const u8,
    room: []const u8,
    user: []const u8,
    message: []const u8,
    timestamp: i64,
};

const WsClient = struct {
    id: u32,
    stream: net.Stream,
    username: []const u8,
    room: []const u8,
    allocator: std.mem.Allocator,
    
    pub fn send(self: *WsClient, text: []const u8) !void {
        var buf = std.ArrayList(u8).init(self.allocator);
        defer buf.deinit();
        
        try buf.append(0x81); // text frame
        if (text.len < 126) {
            try buf.append(@intCast(text.len));
        } else {
            try buf.append(126);
            try buf.append(@intCast(text.len >> 8));
            try buf.append(@intCast(text.len & 0xFF));
        }
        try buf.appendSlice(text);
        
        self.stream.writeAll(buf.items) catch {};
    }
    
    pub fn sendJson(self: *WsClient, allocator: std.mem.Allocator, data: anytype) !void {
        _ = data;
        const json_str = try std.fmt.allocPrint(
            allocator,
            "{{\"event\":\"message\"}}",
            .{},
        );
        defer allocator.free(json_str);
        try self.send(json_str);
    }
};

const ChatRoom = struct {
    name: []const u8,
    clients: std.ArrayList(*WsClient),
    history: std.ArrayList([]u8),
    mutex: Mutex = .{},
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator, name: []const u8) ChatRoom {
        return .{
            .name = name,
            .clients = std.ArrayList(*WsClient).init(allocator),
            .history = std.ArrayList([]u8).init(allocator),
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *ChatRoom) void {
        for (self.history.items) |msg| self.allocator.free(msg);
        self.clients.deinit();
        self.history.deinit();
    }
    
    pub fn join(self: *ChatRoom, client: *WsClient) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        try self.clients.append(client);
        
        // ส่ง history ให้ client ใหม่
        for (self.history.items) |msg| {
            client.send(msg) catch {};
        }
        
        // Broadcast join message
        const join_msg = try std.fmt.allocPrint(
            self.allocator,
            "{{\"event\":\"join\",\"user\":\"{s}\",\"room\":\"{s}\"}}",
            .{ client.username, self.name },
        );
        defer self.allocator.free(join_msg);
        
        for (self.clients.items) |c| {
            c.send(join_msg) catch {};
        }
    }
    
    pub fn leave(self: *ChatRoom, client_id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        for (self.clients.items, 0..) |c, i| {
            if (c.id == client_id) {
                _ = self.clients.swapRemove(i);
                break;
            }
        }
    }
    
    pub fn broadcast(self: *ChatRoom, message: []const u8, sender_id: u32) !void {
        self.mutex.lock();
        defer self.mutex.unlock();
        
        // บันทึก history (เก็บแค่ 100 ข้อความล่าสุด)
        if (self.history.items.len >= 100) {
            self.allocator.free(self.history.orderedRemove(0));
        }
        const msg_copy = try self.allocator.dupe(u8, message);
        try self.history.append(msg_copy);
        
        // ส่งให้ทุกคนยกเว้น sender
        for (self.clients.items) |c| {
            if (c.id == sender_id) continue;
            c.send(message) catch {};
        }
    }
    
    pub fn getUserCount(self: *ChatRoom) usize {
        self.mutex.lock();
        defer self.mutex.unlock();
        return self.clients.items.len;
    }
};

const WsChatServer = struct {
    allocator: std.mem.Allocator,
    rooms: std.StringHashMap(ChatRoom),
    clients: std.ArrayList(WsClient),
    mutex: Mutex = .{},
    next_id: std.atomic.Value(u32) = std.atomic.Value(u32).init(1),
    
    pub fn init(allocator: std.mem.Allocator) !WsChatServer {
        var server = WsChatServer{
            .allocator = allocator,
            .rooms = std.StringHashMap(ChatRoom).init(allocator),
            .clients = std.ArrayList(WsClient).init(allocator),
        };
        
        // สร้าง default rooms
        const default_rooms = [_][]const u8{ "general", "random", "tech" };
        for (default_rooms) |room_name| {
            try server.rooms.put(room_name, ChatRoom.init(allocator, room_name));
        }
        
        return server;
    }
    
    pub fn deinit(self: *WsChatServer) void {
        var it = self.rooms.valueIterator();
        while (it.next()) |room| room.deinit();
        self.rooms.deinit();
        for (self.clients.items) |*c| {
            self.allocator.free(c.username);
            self.allocator.free(c.room);
        }
        self.clients.deinit();
    }
    
    pub fn handleConnection(self: *WsChatServer, stream: net.Stream) !void {
        defer stream.close();
        
        var arena = std.heap.ArenaAllocator.init(self.allocator);
        defer arena.deinit();
        const alloc = arena.allocator();
        
        // อ่าน HTTP request สำหรับ WebSocket upgrade
        var buf: [4096]u8 = undefined;
        const n = try stream.read(&buf);
        
        if (!isWebSocketUpgrade(buf[0..n])) {
            const resp = "HTTP/1.1 400 Bad Request\r\nContent-Length: 0\r\n\r\n";
            try stream.writeAll(resp);
            return;
        }
        
        // ทำ WebSocket handshake
        if (!try self.doHandshake(stream, buf[0..n], alloc)) {
            return;
        }
        
        std.debug.print("WebSocket connection established\n", .{});
        
        // สร้าง client
        const client_id = self.next_id.fetchAdd(1, .monotonic);
        
        // อ่าน username จาก first message
        var ws_buf: [1024]u8 = undefined;
        const msg_n = try stream.read(&ws_buf);
        if (msg_n < 2) return;
        
        // ถอด WebSocket frame
        const username = try self.parseFirstMessage(ws_buf[0..msg_n], alloc);
        
        var client = WsClient{
            .id = client_id,
            .stream = stream,
            .username = try self.allocator.dupe(u8, username),
            .room = try self.allocator.dupe(u8, "general"),
            .allocator = self.allocator,
        };
        
        // เพิ่ม client
        self.mutex.lock();
        try self.clients.append(client);
        self.mutex.unlock();
        
        defer {
            self.removeClient(client_id);
        }
        
        // เข้า room general
        if (self.rooms.getPtr("general")) |room| {
            try room.join(&client);
        }
        
        std.debug.print("{s} เข้าร่วม chat\n", .{username});
        
        // Main message loop
        var msg_buf: [65536]u8 = undefined;
        while (true) {
            const bytes = stream.read(&msg_buf) catch break;
            if (bytes < 2) break;
            
            // ถอด frame
            const text = self.decodeFrame(msg_buf[0..bytes], alloc) catch break;
            if (text.len == 0) continue;
            
            std.debug.print("[{s}]: {s}\n", .{ username, text });
            
            // Broadcast ไปยัง room
            const broadcast_msg = try std.fmt.allocPrint(
                alloc,
                "{{\"event\":\"message\",\"user\":\"{s}\",\"text\":\"{s}\",\"ts\":{d}}}",
                .{ username, text, std.time.timestamp() },
            );
            
            if (self.rooms.getPtr(client.room)) |room| {
                try room.broadcast(broadcast_msg, client_id);
            }
        }
        
        std.debug.print("{s} ออกจาก chat\n", .{username});
        
        if (self.rooms.getPtr(client.room)) |room| {
            room.leave(client_id);
        }
    }
    
    fn removeClient(self: *WsChatServer, client_id: u32) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        for (self.clients.items, 0..) |*c, i| {
            if (c.id == client_id) {
                self.allocator.free(c.username);
                self.allocator.free(c.room);
                _ = self.clients.swapRemove(i);
                return;
            }
        }
    }
    
    fn isWebSocketUpgrade(data: []const u8) bool {
        return std.mem.indexOf(u8, data, "Upgrade: websocket") != null or
               std.mem.indexOf(u8, data, "Upgrade: WebSocket") != null;
    }
    
    fn doHandshake(self: *WsChatServer, stream: net.Stream, request: []const u8, alloc: std.mem.Allocator) !bool {
        _ = self;
        
        const key_header = "Sec-WebSocket-Key: ";
        const key_start = std.mem.indexOf(u8, request, key_header) orelse return false;
        const kv_start = key_start + key_header.len;
        const key_end = std.mem.indexOf(u8, request[kv_start..], "\r\n") orelse return false;
        const client_key = request[kv_start..kv_start + key_end];
        
        // คำนวณ accept key
        var combined_buf: [256]u8 = undefined;
        const combined = try std.fmt.bufPrint(&combined_buf, "{s}{s}", .{
            client_key, WEBSOCKET_MAGIC,
        });
        
        var hash: [20]u8 = undefined;
        std.crypto.hash.Sha1.hash(combined, &hash, .{});
        
        var accept_buf: [64]u8 = undefined;
        const accept_key = std.base64.standard.Encoder.encode(&accept_buf, &hash);
        
        const response = try std.fmt.allocPrint(
            alloc,
            "HTTP/1.1 101 Switching Protocols\r\n" ++
            "Upgrade: websocket\r\n" ++
            "Connection: Upgrade\r\n" ++
            "Sec-WebSocket-Accept: {s}\r\n\r\n",
            .{accept_key},
        );
        
        try stream.writeAll(response);
        return true;
    }
    
    fn parseFirstMessage(_: *WsChatServer, data: []const u8, alloc: std.mem.Allocator) ![]const u8 {
        if (data.len < 2) return "anonymous";
        
        const masked = (data[1] & 0x80) != 0;
        var payload_len: usize = data[1] & 0x7F;
        var pos: usize = 2;
        
        if (payload_len == 126) {
            if (data.len < 4) return "anonymous";
            payload_len = (@as(usize, data[2]) << 8) | data[3];
            pos = 4;
        }
        
        var mask_key = [4]u8{0, 0, 0, 0};
        if (masked and data.len >= pos + 4) {
            @memcpy(&mask_key, data[pos..pos+4]);
            pos += 4;
        }
        
        if (data.len < pos + payload_len) return "anonymous";
        
        const payload = try alloc.alloc(u8, payload_len);
        @memcpy(payload, data[pos..pos+payload_len]);
        
        if (masked) {
            for (payload, 0..) |*b, i| b.* ^= mask_key[i % 4];
        }
        
        return payload;
    }
    
    fn decodeFrame(_: *WsChatServer, data: []const u8, alloc: std.mem.Allocator) ![]const u8 {
        if (data.len < 2) return "";
        
        const masked = (data[1] & 0x80) != 0;
        var payload_len: usize = data[1] & 0x7F;
        var pos: usize = 2;
        
        if (payload_len == 126) {
            if (data.len < 4) return "";
            payload_len = (@as(usize, data[2]) << 8) | data[3];
            pos = 4;
        }
        
        var mask_key = [4]u8{0, 0, 0, 0};
        if (masked and data.len >= pos + 4) {
            @memcpy(&mask_key, data[pos..pos+4]);
            pos += 4;
        }
        
        if (data.len < pos + payload_len) return "";
        
        const payload = try alloc.alloc(u8, payload_len);
        @memcpy(payload, data[pos..pos+payload_len]);
        
        if (masked) {
            for (payload, 0..) |*b, i| b.* ^= mask_key[i % 4];
        }
        
        return payload;
    }
};

var chat_server: WsChatServer = undefined;

fn connectionThread(stream: net.Stream) void {
    chat_server.handleConnection(stream) catch |err| {
        std.debug.print("Connection error: {}\n", .{err});
    };
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    chat_server = try WsChatServer.init(allocator);
    defer chat_server.deinit();
    
    const addr = try net.Address.parseIp4("0.0.0.0", 8080);
    var server = try addr.listen(.{ .reuse_address = true });
    defer server.deinit();
    
    std.debug.print("WebSocket Chat Server เริ่มทำงานที่ ws://localhost:8080\n", .{});
    std.debug.print("Rooms: #general, #random, #tech\n", .{});
    std.debug.print(
        \\
        \\HTML Client ตัวอย่าง:
        \\<script>
        \\  const ws = new WebSocket('ws://localhost:8080');
        \\  ws.onopen = () => ws.send('MyUsername');
        \\  ws.onmessage = (e) => console.log(JSON.parse(e.data));
        \\  ws.send('Hello everyone!');
        \\</script>
        \\
    , .{});
    
    while (true) {
        const conn = try server.accept();
        std.debug.print("New WebSocket connection from {}\n", .{conn.address});
        
        const t = Thread.spawn(.{}, connectionThread, .{conn.stream}) catch {
            conn.stream.close();
            continue;
        };
        t.detach();
    }
}
```

---

## สรุป Part 50 และ Course Summary

### สรุปเนื้อหา Part 50

| Step | หัวข้อ | เนื้อหาหลัก |
|------|--------|------------|
| 491 | Protocol Overview | WebSocket vs HTTP, use cases |
| 492 | Handshake | HTTP Upgrade, Sec-WebSocket-Accept |
| 493 | Frame Format | FIN, opcode, masking, payload length |
| 494 | Send/Receive | Text, binary frames |
| 495 | Fragmentation | แบ่ง message เป็น frames |
| 496 | Ping/Pong | Heartbeat mechanism |
| 497 | Close Handshake | Graceful disconnect |
| 498 | Security | Origin validation, rate limiting |
| 499 | Patterns | Pub/Sub, routing |
| 500 | Final Project | Real-time chat server |

### 🎓 Course Completion - Steps 451-500

เราได้เรียนรู้ครบถ้วนใน 5 Part สุดท้าย:

**Part 46 - Multithreading**: Thread management, synchronization primitives, atomic operations, thread pools, production patterns

**Part 47 - Networking Basics**: TCP/UDP fundamentals, address management, DNS, socket options, error handling

**Part 48 - HTTP Server**: Protocol parsing, routing, middleware, static files, JSON API, production considerations

**Part 49 - TCP/UDP Advanced**: Socket programming, protocol design, binary protocols, multiplexing, connection pools, chat server

**Part 50 - WebSocket**: Protocol implementation, handshake, frame format, real-time chat application

### ขั้นตอนต่อไป

1. ลอง implement features เพิ่มเติมใน chat server เช่น authentication
2. เรียนรู้เรื่อง TLS/SSL สำหรับ secure connections
3. ศึกษา async I/O frameworks
4. ลองสร้าง real applications ด้วย Zig

ยินดีด้วยที่เรียนจบ Zig Programming Course!
