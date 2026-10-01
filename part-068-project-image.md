# ส่วนที่ 68: โปรเจกต์ - Image Processing

## ขั้นตอนที่ 671-680: สร้างไลบรารีประมวลผลภาพ

---

## ขั้นตอนที่ 671: BMP File Format Reading

BMP (Bitmap) เป็นรูปแบบไฟล์ภาพที่ไม่บีบอัด ง่ายต่อการอ่านและเขียน

### โครงสร้างโปรเจกต์

```
imgproc/
├── src/
│   ├── main.zig
│   ├── image.zig
│   ├── bmp.zig
│   ├── filters.zig
│   ├── transform.zig
│   └── color.zig
├── build.zig
└── README.md
```

### ไฟล์ build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const imgproc_module = b.addModule("imgproc", .{
        .root_source_file = b.path("src/image.zig"),
    });
    _ = imgproc_module;

    const exe = b.addExecutable(.{
        .name = "imgproc",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run image processor");
    run_step.dependOn(&run_cmd.step);

    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/image.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_unit_tests = b.addRunArtifact(unit_tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

### ไฟล์ src/bmp.zig - BMP Reader

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;
const image_mod = @import("image.zig");
const Image = image_mod.Image;
const Pixel = image_mod.Pixel;

/// BMP File Header (14 bytes)
pub const BmpFileHeader = packed struct {
    signature: u16,     // 'BM' = 0x4D42
    file_size: u32,
    reserved1: u16,
    reserved2: u16,
    data_offset: u32,
};

/// BMP Info Header (40 bytes - BITMAPINFOHEADER)
pub const BmpInfoHeader = packed struct {
    header_size: u32,   // = 40
    width: i32,
    height: i32,
    color_planes: u16,  // = 1
    bits_per_pixel: u16,
    compression: u32,   // 0 = BI_RGB (no compression)
    image_size: u32,
    x_pixels_per_meter: i32,
    y_pixels_per_meter: i32,
    colors_used: u32,
    important_colors: u32,
};

pub const BmpError = error{
    InvalidSignature,
    UnsupportedBitsPerPixel,
    UnsupportedCompression,
    InvalidDimensions,
    ReadError,
    OutOfMemory,
};

/// อ่าน BMP file
pub fn readBmp(allocator: Allocator, path: []const u8) !Image {
    const file = try std.fs.openFileAbsolute(path, .{});
    defer file.close();

    const reader = file.reader();

    // อ่าน File Header
    var file_header: BmpFileHeader = undefined;
    try reader.readNoEof(std.mem.asBytes(&file_header));

    // ตรวจสอบ signature 'BM'
    if (file_header.signature != 0x4D42) {
        return BmpError.InvalidSignature;
    }

    // อ่าน Info Header
    var info_header: BmpInfoHeader = undefined;
    try reader.readNoEof(std.mem.asBytes(&info_header));

    // รองรับแค่ 24-bit และ 32-bit BMP
    if (info_header.bits_per_pixel != 24 and info_header.bits_per_pixel != 32) {
        return BmpError.UnsupportedBitsPerPixel;
    }

    // รองรับแค่ uncompressed
    if (info_header.compression != 0) {
        return BmpError.UnsupportedCompression;
    }

    const width = @as(u32, @intCast(@abs(info_header.width)));
    const height = @as(u32, @intCast(@abs(info_header.height)));

    if (width == 0 or height == 0) {
        return BmpError.InvalidDimensions;
    }

    // สร้าง image
    var img = try Image.init(allocator, width, height);
    errdefer img.deinit();

    // ไปที่ตำแหน่งข้อมูล pixel
    try file.seekTo(file_header.data_offset);

    const bytes_per_pixel = info_header.bits_per_pixel / 8;
    // Row padding ใน BMP ต้องเป็น multiple ของ 4
    const row_size = ((width * bytes_per_pixel + 3) / 4) * 4;

    var row_buf = try allocator.alloc(u8, row_size);
    defer allocator.free(row_buf);

    // BMP เก็บ pixel จากล่างขึ้นบน (bottom-up)
    const is_top_down = info_header.height < 0;

    for (0..height) |row_idx| {
        try reader.readNoEof(row_buf);

        const actual_row = if (is_top_down)
            row_idx
        else
            height - 1 - row_idx;

        for (0..width) |col| {
            const offset = col * bytes_per_pixel;
            const b = row_buf[offset];
            const g = row_buf[offset + 1];
            const r = row_buf[offset + 2];
            const a: u8 = if (bytes_per_pixel == 4) row_buf[offset + 3] else 255;

            img.setPixel(col, actual_row, Pixel{ .r = r, .g = g, .b = b, .a = a });
        }
    }

    return img;
}
```

---

## ขั้นตอนที่ 672: BMP File Format Writing

```zig
// ต่อจาก bmp.zig

/// เขียน BMP file
pub fn writeBmp(img: *const Image, path: []const u8) !void {
    const file = try std.fs.createFileAbsolute(path, .{});
    defer file.close();

    const writer = file.writer();
    var buf_writer = std.io.bufferedWriter(writer);
    const bw = buf_writer.writer();

    const bytes_per_pixel: u32 = 3; // 24-bit
    const row_size = ((img.width * bytes_per_pixel + 3) / 4) * 4;
    const image_data_size = row_size * img.height;
    const file_size = @sizeOf(BmpFileHeader) + @sizeOf(BmpInfoHeader) + image_data_size;

    // เขียน File Header
    const file_header = BmpFileHeader{
        .signature = 0x4D42,
        .file_size = @intCast(file_size),
        .reserved1 = 0,
        .reserved2 = 0,
        .data_offset = @sizeOf(BmpFileHeader) + @sizeOf(BmpInfoHeader),
    };
    try bw.writeAll(std.mem.asBytes(&file_header));

    // เขียน Info Header
    const info_header = BmpInfoHeader{
        .header_size = @sizeOf(BmpInfoHeader),
        .width = @intCast(img.width),
        .height = @intCast(img.height), // positive = bottom-up
        .color_planes = 1,
        .bits_per_pixel = 24,
        .compression = 0,
        .image_size = @intCast(image_data_size),
        .x_pixels_per_meter = 2835, // 72 DPI
        .y_pixels_per_meter = 2835,
        .colors_used = 0,
        .important_colors = 0,
    };
    try bw.writeAll(std.mem.asBytes(&info_header));

    // เขียน pixel data (bottom-up)
    const padding_size = row_size - img.width * bytes_per_pixel;
    const padding = [_]u8{0} ** 4;

    var row: usize = img.height;
    while (row > 0) {
        row -= 1;
        for (0..img.width) |col| {
            const pixel = img.getPixel(col, row);
            try bw.writeByte(pixel.b);
            try bw.writeByte(pixel.g);
            try bw.writeByte(pixel.r);
        }
        if (padding_size > 0) {
            try bw.writeAll(padding[0..padding_size]);
        }
    }

    try buf_writer.flush();
}
```

---

## ขั้นตอนที่ 673: Pixel Manipulation

```zig
// ไฟล์ src/image.zig

const std = @import("std");
const Allocator = std.mem.Allocator;

/// RGBA Pixel
pub const Pixel = struct {
    r: u8,
    g: u8,
    b: u8,
    a: u8 = 255,

    pub const BLACK = Pixel{ .r = 0, .g = 0, .b = 0 };
    pub const WHITE = Pixel{ .r = 255, .g = 255, .b = 255 };
    pub const RED = Pixel{ .r = 255, .g = 0, .b = 0 };
    pub const GREEN = Pixel{ .r = 0, .g = 255, .b = 0 };
    pub const BLUE = Pixel{ .r = 0, .g = 0, .b = 255 };
    pub const TRANSPARENT = Pixel{ .r = 0, .g = 0, .b = 0, .a = 0 };

    /// ผสม 2 pixels ด้วย alpha blending
    pub fn blend(self: Pixel, other: Pixel) Pixel {
        const alpha = @as(f32, @floatFromInt(other.a)) / 255.0;
        const inv_alpha = 1.0 - alpha;

        return Pixel{
            .r = @intFromFloat(@as(f32, @floatFromInt(self.r)) * inv_alpha +
                @as(f32, @floatFromInt(other.r)) * alpha),
            .g = @intFromFloat(@as(f32, @floatFromInt(self.g)) * inv_alpha +
                @as(f32, @floatFromInt(other.g)) * alpha),
            .b = @intFromFloat(@as(f32, @floatFromInt(self.b)) * inv_alpha +
                @as(f32, @floatFromInt(other.b)) * alpha),
            .a = 255,
        };
    }

    /// แปลงเป็น grayscale value
    pub fn luminance(self: Pixel) u8 {
        const r = @as(f32, @floatFromInt(self.r));
        const g = @as(f32, @floatFromInt(self.g));
        const b = @as(f32, @floatFromInt(self.b));
        return @intFromFloat(0.2126 * r + 0.7152 * g + 0.0722 * b);
    }

    /// เปรียบเทียบ pixels
    pub fn eql(self: Pixel, other: Pixel) bool {
        return self.r == other.r and self.g == other.g and
            self.b == other.b and self.a == other.a;
    }

    /// clamp ค่าสี
    pub fn clamp(r: i32, g: i32, b: i32) Pixel {
        return Pixel{
            .r = @intCast(std.math.clamp(r, 0, 255)),
            .g = @intCast(std.math.clamp(g, 0, 255)),
            .b = @intCast(std.math.clamp(b, 0, 255)),
        };
    }
};

/// Image container
pub const Image = struct {
    width: u32,
    height: u32,
    pixels: []Pixel,
    allocator: Allocator,

    const Self = @This();

    pub fn init(allocator: Allocator, width: u32, height: u32) !Self {
        const pixels = try allocator.alloc(Pixel, width * height);
        @memset(pixels, Pixel.BLACK);
        return Self{
            .width = width,
            .height = height,
            .pixels = pixels,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Self) void {
        self.allocator.free(self.pixels);
    }

    /// ดึง pixel ที่ตำแหน่ง (x, y)
    pub fn getPixel(self: *const Self, x: usize, y: usize) Pixel {
        return self.pixels[y * self.width + x];
    }

    /// ตั้งค่า pixel ที่ตำแหน่ง (x, y)
    pub fn setPixel(self: *Self, x: usize, y: usize, pixel: Pixel) void {
        self.pixels[y * self.width + x] = pixel;
    }

    /// ดึง pixel ด้วย boundary check (clamp edges)
    pub fn getPixelSafe(self: *const Self, x: i64, y: i64) Pixel {
        const cx = @as(usize, @intCast(std.math.clamp(x, 0, @as(i64, @intCast(self.width - 1)))));
        const cy = @as(usize, @intCast(std.math.clamp(y, 0, @as(i64, @intCast(self.height - 1)))));
        return self.getPixel(cx, cy);
    }

    /// Copy image
    pub fn clone(self: *const Self) !Self {
        var new_img = try Self.init(self.allocator, self.width, self.height);
        @memcpy(new_img.pixels, self.pixels);
        return new_img;
    }

    /// Fill ทั้งภาพด้วยสีเดียว
    pub fn fill(self: *Self, pixel: Pixel) void {
        @memset(self.pixels, pixel);
    }

    /// Copy region จาก source image
    pub fn copyFrom(
        self: *Self,
        src: *const Self,
        src_x: usize, src_y: usize,
        dst_x: usize, dst_y: usize,
        copy_width: usize, copy_height: usize,
    ) void {
        for (0..copy_height) |row| {
            for (0..copy_width) |col| {
                if (src_x + col >= src.width or src_y + row >= src.height) continue;
                if (dst_x + col >= self.width or dst_y + row >= self.height) continue;
                const p = src.getPixel(src_x + col, src_y + row);
                self.setPixel(dst_x + col, dst_y + row, p);
            }
        }
    }

    /// วาด rectangle
    pub fn drawRect(self: *Self, x: usize, y: usize, w: usize, h: usize, color: Pixel) void {
        for (0..h) |row| {
            for (0..w) |col| {
                if (x + col < self.width and y + row < self.height) {
                    self.setPixel(x + col, y + row, color);
                }
            }
        }
    }

    /// สถิติของภาพ
    pub fn statistics(self: *const Self) ImageStats {
        var min_r: u8 = 255;
        var max_r: u8 = 0;
        var min_g: u8 = 255;
        var max_g: u8 = 0;
        var min_b: u8 = 255;
        var max_b: u8 = 0;
        var sum_r: u64 = 0;
        var sum_g: u64 = 0;
        var sum_b: u64 = 0;

        for (self.pixels) |p| {
            min_r = @min(min_r, p.r);
            max_r = @max(max_r, p.r);
            min_g = @min(min_g, p.g);
            max_g = @max(max_g, p.g);
            min_b = @min(min_b, p.b);
            max_b = @max(max_b, p.b);
            sum_r += p.r;
            sum_g += p.g;
            sum_b += p.b;
        }

        const n = self.pixels.len;
        return ImageStats{
            .min_r = min_r, .max_r = max_r,
            .min_g = min_g, .max_g = max_g,
            .min_b = min_b, .max_b = max_b,
            .avg_r = @as(f32, @floatFromInt(sum_r)) / @as(f32, @floatFromInt(n)),
            .avg_g = @as(f32, @floatFromInt(sum_g)) / @as(f32, @floatFromInt(n)),
            .avg_b = @as(f32, @floatFromInt(sum_b)) / @as(f32, @floatFromInt(n)),
        };
    }
};

pub const ImageStats = struct {
    min_r: u8, max_r: u8,
    min_g: u8, max_g: u8,
    min_b: u8, max_b: u8,
    avg_r: f32,
    avg_g: f32,
    avg_b: f32,
};
```

---

## ขั้นตอนที่ 674: Grayscale Conversion

```zig
// ไฟล์ src/filters.zig (ส่วนที่ 1)

const std = @import("std");
const image_mod = @import("image.zig");
const Image = image_mod.Image;
const Pixel = image_mod.Pixel;
const Allocator = std.mem.Allocator;

/// แปลงภาพเป็น Grayscale
pub fn grayscale(img: *Image) void {
    for (img.pixels) |*pixel| {
        const gray = pixel.luminance();
        pixel.r = gray;
        pixel.g = gray;
        pixel.b = gray;
    }
}

/// แปลงเป็น grayscale โดยใช้ average method
pub fn grayscaleAverage(img: *Image) void {
    for (img.pixels) |*pixel| {
        const avg = (@as(u16, pixel.r) + pixel.g + pixel.b) / 3;
        pixel.r = @intCast(avg);
        pixel.g = @intCast(avg);
        pixel.b = @intCast(avg);
    }
}

/// แปลงเป็น black and white ด้วย threshold
pub fn threshold(img: *Image, thresh: u8) void {
    for (img.pixels) |*pixel| {
        const gray = pixel.luminance();
        const bw: u8 = if (gray >= thresh) 255 else 0;
        pixel.r = bw;
        pixel.g = bw;
        pixel.b = bw;
    }
}

/// Invert colors
pub fn invert(img: *Image) void {
    for (img.pixels) |*pixel| {
        pixel.r = 255 - pixel.r;
        pixel.g = 255 - pixel.g;
        pixel.b = 255 - pixel.b;
    }
}

/// สร้าง histogram ของ grayscale values
pub fn histogram(img: *const Image) [256]u32 {
    var hist = [_]u32{0} ** 256;
    for (img.pixels) |pixel| {
        hist[pixel.luminance()] += 1;
    }
    return hist;
}

/// Histogram equalization
pub fn equalizeHistogram(img: *Image) void {
    const hist = histogram(img);
    const total = img.pixels.len;

    // สร้าง cumulative distribution
    var cdf = [_]u64{0} ** 256;
    cdf[0] = hist[0];
    for (1..256) |i| {
        cdf[i] = cdf[i - 1] + hist[i];
    }

    // หา cdf minimum ที่ไม่ใช่ 0
    var cdf_min: u64 = 0;
    for (cdf) |c| {
        if (c > 0) {
            cdf_min = c;
            break;
        }
    }

    // สร้าง lookup table
    var lut: [256]u8 = undefined;
    for (0..256) |i| {
        if (cdf[i] == cdf_min) {
            lut[i] = 0;
        } else {
            const val = @as(f64, @floatFromInt(cdf[i] - cdf_min)) /
                @as(f64, @floatFromInt(total - cdf_min)) * 255.0;
            lut[i] = @intFromFloat(@round(val));
        }
    }

    // Apply lookup table
    for (img.pixels) |*pixel| {
        const gray = pixel.luminance();
        const eq_val = lut[gray];
        pixel.r = eq_val;
        pixel.g = eq_val;
        pixel.b = eq_val;
    }
}
```

---

## ขั้นตอนที่ 675: Brightness/Contrast

```zig
// ต่อจาก filters.zig

/// ปรับ brightness (-255 ถึง +255)
pub fn brightness(img: *Image, delta: i16) void {
    for (img.pixels) |*pixel| {
        pixel.r = @intCast(std.math.clamp(@as(i16, pixel.r) + delta, 0, 255));
        pixel.g = @intCast(std.math.clamp(@as(i16, pixel.g) + delta, 0, 255));
        pixel.b = @intCast(std.math.clamp(@as(i16, pixel.b) + delta, 0, 255));
    }
}

/// ปรับ contrast (factor: 0.0 ถึง 2.0, 1.0 = ไม่เปลี่ยน)
pub fn contrast(img: *Image, factor: f32) void {
    const correction = 128.0 * (1.0 - factor);

    for (img.pixels) |*pixel| {
        const r = @as(f32, @floatFromInt(pixel.r)) * factor + correction;
        const g = @as(f32, @floatFromInt(pixel.g)) * factor + correction;
        const b = @as(f32, @floatFromInt(pixel.b)) * factor + correction;

        pixel.r = @intCast(std.math.clamp(@as(i32, @intFromFloat(r)), 0, 255));
        pixel.g = @intCast(std.math.clamp(@as(i32, @intFromFloat(g)), 0, 255));
        pixel.b = @intCast(std.math.clamp(@as(i32, @intFromFloat(b)), 0, 255));
    }
}

/// Gamma correction
pub fn gammaCorrect(img: *Image, gamma: f32) void {
    var lut: [256]u8 = undefined;
    const inv_gamma = 1.0 / gamma;

    for (0..256) |i| {
        const normalized = @as(f32, @floatFromInt(i)) / 255.0;
        lut[i] = @intFromFloat(std.math.pow(f32, normalized, inv_gamma) * 255.0);
    }

    for (img.pixels) |*pixel| {
        pixel.r = lut[pixel.r];
        pixel.g = lut[pixel.g];
        pixel.b = lut[pixel.b];
    }
}

/// ปรับ saturation (0.0 = grayscale, 1.0 = ไม่เปลี่ยน, 2.0 = เพิ่มสีสัน)
pub fn saturation(img: *Image, factor: f32) void {
    for (img.pixels) |*pixel| {
        const r = @as(f32, @floatFromInt(pixel.r));
        const g = @as(f32, @floatFromInt(pixel.g));
        const b = @as(f32, @floatFromInt(pixel.b));

        // Convert to grayscale
        const gray = 0.2126 * r + 0.7152 * g + 0.0722 * b;

        // Interpolate between gray and original
        const new_r = gray + factor * (r - gray);
        const new_g = gray + factor * (g - gray);
        const new_b = gray + factor * (b - gray);

        pixel.r = @intCast(std.math.clamp(@as(i32, @intFromFloat(new_r)), 0, 255));
        pixel.g = @intCast(std.math.clamp(@as(i32, @intFromFloat(new_g)), 0, 255));
        pixel.b = @intCast(std.math.clamp(@as(i32, @intFromFloat(new_b)), 0, 255));
    }
}

/// Levels adjustment (เหมือน Photoshop levels)
pub fn levels(img: *Image, in_min: u8, in_max: u8, out_min: u8, out_max: u8) void {
    var lut: [256]u8 = undefined;
    const in_range = @as(f32, @floatFromInt(in_max - in_min));
    const out_range = @as(f32, @floatFromInt(out_max - out_min));

    for (0..256) |i| {
        const clamped = std.math.clamp(@as(f32, @floatFromInt(i)), @as(f32, @floatFromInt(in_min)), @as(f32, @floatFromInt(in_max)));
        const normalized = if (in_range > 0) (clamped - @as(f32, @floatFromInt(in_min))) / in_range else 0.0;
        lut[i] = @intFromFloat(normalized * out_range + @as(f32, @floatFromInt(out_min)));
    }

    for (img.pixels) |*pixel| {
        pixel.r = lut[pixel.r];
        pixel.g = lut[pixel.g];
        pixel.b = lut[pixel.b];
    }
}
```

---

## ขั้นตอนที่ 676: Blur Filter (Box Blur)

```zig
// ต่อจาก filters.zig

/// Box Blur
pub fn boxBlur(allocator: Allocator, img: *Image, radius: u32) !void {
    var temp = try img.clone();
    defer temp.deinit();

    const r = @as(i64, @intCast(radius));
    const size = 2 * r + 1;
    const divisor = @as(i64, size * size);

    for (0..img.height) |y| {
        for (0..img.width) |x| {
            var sum_r: i64 = 0;
            var sum_g: i64 = 0;
            var sum_b: i64 = 0;

            var dy: i64 = -r;
            while (dy <= r) : (dy += 1) {
                var dx: i64 = -r;
                while (dx <= r) : (dx += 1) {
                    const px = @as(i64, @intCast(x)) + dx;
                    const py = @as(i64, @intCast(y)) + dy;
                    const p = temp.getPixelSafe(px, py);
                    sum_r += p.r;
                    sum_g += p.g;
                    sum_b += p.b;
                }
            }

            img.setPixel(x, y, Pixel{
                .r = @intCast(@divTrunc(sum_r, divisor)),
                .g = @intCast(@divTrunc(sum_g, divisor)),
                .b = @intCast(@divTrunc(sum_b, divisor)),
            });
        }
    }
}

/// Gaussian Blur
pub fn gaussianBlur(allocator: Allocator, img: *Image, sigma: f32) !void {
    const radius = @as(u32, @intFromFloat(@ceil(3.0 * sigma)));
    const size = 2 * radius + 1;

    // สร้าง Gaussian kernel
    var kernel = try allocator.alloc(f32, size * size);
    defer allocator.free(kernel);

    var kernel_sum: f32 = 0;
    const inv_2sigma2 = 1.0 / (2.0 * sigma * sigma);

    for (0..size) |ky| {
        for (0..size) |kx| {
            const dy = @as(f32, @floatFromInt(ky)) - @as(f32, @floatFromInt(radius));
            const dx = @as(f32, @floatFromInt(kx)) - @as(f32, @floatFromInt(radius));
            const val = @exp(-(dx * dx + dy * dy) * inv_2sigma2);
            kernel[ky * size + kx] = val;
            kernel_sum += val;
        }
    }

    // Normalize kernel
    for (kernel) |*k| k.* /= kernel_sum;

    // Apply kernel
    var temp = try img.clone();
    defer temp.deinit();

    for (0..img.height) |y| {
        for (0..img.width) |x| {
            var sum_r: f32 = 0;
            var sum_g: f32 = 0;
            var sum_b: f32 = 0;

            for (0..size) |ky| {
                for (0..size) |kx| {
                    const px = @as(i64, @intCast(x)) + @as(i64, @intCast(kx)) - @as(i64, @intCast(radius));
                    const py = @as(i64, @intCast(y)) + @as(i64, @intCast(ky)) - @as(i64, @intCast(radius));
                    const p = temp.getPixelSafe(px, py);
                    const k = kernel[ky * size + kx];
                    sum_r += @as(f32, @floatFromInt(p.r)) * k;
                    sum_g += @as(f32, @floatFromInt(p.g)) * k;
                    sum_b += @as(f32, @floatFromInt(p.b)) * k;
                }
            }

            img.setPixel(x, y, Pixel{
                .r = @intCast(std.math.clamp(@as(i32, @intFromFloat(sum_r)), 0, 255)),
                .g = @intCast(std.math.clamp(@as(i32, @intFromFloat(sum_g)), 0, 255)),
                .b = @intCast(std.math.clamp(@as(i32, @intFromFloat(sum_b)), 0, 255)),
            });
        }
    }
}

/// Sharpen filter
pub fn sharpen(allocator: Allocator, img: *Image) !void {
    // Sharpen kernel: 0 -1 0 / -1 5 -1 / 0 -1 0
    const kernel = [9]i32{ 0, -1, 0, -1, 5, -1, 0, -1, 0 };

    var temp = try img.clone();
    defer temp.deinit();

    for (1..img.height - 1) |y| {
        for (1..img.width - 1) |x| {
            var sum_r: i32 = 0;
            var sum_g: i32 = 0;
            var sum_b: i32 = 0;

            for (0..3) |ky| {
                for (0..3) |kx| {
                    const px = x + kx - 1;
                    const py = y + ky - 1;
                    const p = temp.getPixel(px, py);
                    const k = kernel[ky * 3 + kx];
                    sum_r += @as(i32, p.r) * k;
                    sum_g += @as(i32, p.g) * k;
                    sum_b += @as(i32, p.b) * k;
                }
            }

            img.setPixel(x, y, Pixel.clamp(sum_r, sum_g, sum_b));
        }
    }
    _ = allocator;
}
```

---

## ขั้นตอนที่ 677: Edge Detection

```zig
// ต่อจาก filters.zig

/// Sobel Edge Detection
pub fn sobelEdge(allocator: Allocator, img: *Image) !void {
    // Sobel kernels
    const sobel_x = [9]i32{ -1, 0, 1, -2, 0, 2, -1, 0, 1 };
    const sobel_y = [9]i32{ -1, -2, -1, 0, 0, 0, 1, 2, 1 };

    var temp = try img.clone();
    defer temp.deinit();

    // แปลงเป็น grayscale ก่อน
    for (temp.pixels) |*p| {
        const gray = p.luminance();
        p.r = gray;
        p.g = gray;
        p.b = gray;
    }

    for (1..img.height - 1) |y| {
        for (1..img.width - 1) |x| {
            var gx: i32 = 0;
            var gy: i32 = 0;

            for (0..3) |ky| {
                for (0..3) |kx| {
                    const px = x + kx - 1;
                    const py = y + ky - 1;
                    const p = temp.getPixel(px, py);
                    const gray: i32 = p.r; // already grayscale

                    gx += gray * sobel_x[ky * 3 + kx];
                    gy += gray * sobel_y[ky * 3 + kx];
                }
            }

            const magnitude = @min(255, @as(u32, @intFromFloat(@sqrt(
                @as(f32, @floatFromInt(gx * gx)) + @as(f32, @floatFromInt(gy * gy)),
            ))));

            const edge: u8 = @intCast(magnitude);
            img.setPixel(x, y, Pixel{ .r = edge, .g = edge, .b = edge });
        }
    }
}

/// Canny Edge Detection (simplified)
pub fn cannyEdge(allocator: Allocator, img: *Image, low_thresh: u8, high_thresh: u8) !void {
    // Step 1: Gaussian blur
    try gaussianBlur(allocator, img, 1.4);

    // Step 2: Sobel
    try sobelEdge(allocator, img);

    // Step 3: Double threshold
    for (img.pixels) |*p| {
        if (p.r >= high_thresh) {
            p.r = 255;
            p.g = 255;
            p.b = 255;
        } else if (p.r >= low_thresh) {
            p.r = 128;
            p.g = 128;
            p.b = 128;
        } else {
            p.r = 0;
            p.g = 0;
            p.b = 0;
        }
    }

    // Step 4: Hysteresis (simplified)
    var changed = true;
    while (changed) {
        changed = false;
        for (1..img.height - 1) |y| {
            for (1..img.width - 1) |x| {
                const p = img.getPixel(x, y);
                if (p.r != 128) continue;

                // ตรวจสอบ neighbors
                var has_strong_neighbor = false;
                for (0..3) |ky| {
                    for (0..3) |kx| {
                        if (ky == 1 and kx == 1) continue;
                        const neighbor = img.getPixel(x + kx - 1, y + ky - 1);
                        if (neighbor.r == 255) {
                            has_strong_neighbor = true;
                            break;
                        }
                    }
                    if (has_strong_neighbor) break;
                }

                if (has_strong_neighbor) {
                    img.setPixel(x, y, Pixel{ .r = 255, .g = 255, .b = 255 });
                    changed = true;
                }
            }
        }
    }

    // ลบ weak edges ที่ไม่ติดกับ strong edges
    for (img.pixels) |*p| {
        if (p.r == 128) {
            p.r = 0;
            p.g = 0;
            p.b = 0;
        }
    }
}
```

---

## ขั้นตอนที่ 678: Resize (Nearest Neighbor)

```zig
// ไฟล์ src/transform.zig

const std = @import("std");
const image_mod = @import("image.zig");
const Image = image_mod.Image;
const Pixel = image_mod.Pixel;
const Allocator = std.mem.Allocator;

/// Resize ด้วย Nearest Neighbor interpolation
pub fn resizeNearest(
    allocator: Allocator,
    src: *const Image,
    new_width: u32,
    new_height: u32,
) !Image {
    var dst = try Image.init(allocator, new_width, new_height);

    const x_ratio = @as(f32, @floatFromInt(src.width)) / @as(f32, @floatFromInt(new_width));
    const y_ratio = @as(f32, @floatFromInt(src.height)) / @as(f32, @floatFromInt(new_height));

    for (0..new_height) |y| {
        for (0..new_width) |x| {
            const src_x = @as(usize, @intFromFloat(@as(f32, @floatFromInt(x)) * x_ratio));
            const src_y = @as(usize, @intFromFloat(@as(f32, @floatFromInt(y)) * y_ratio));

            const clamped_x = @min(src_x, src.width - 1);
            const clamped_y = @min(src_y, src.height - 1);

            dst.setPixel(x, y, src.getPixel(clamped_x, clamped_y));
        }
    }

    return dst;
}

/// Resize ด้วย Bilinear interpolation
pub fn resizeBilinear(
    allocator: Allocator,
    src: *const Image,
    new_width: u32,
    new_height: u32,
) !Image {
    var dst = try Image.init(allocator, new_width, new_height);

    const x_ratio = @as(f32, @floatFromInt(src.width - 1)) / @as(f32, @floatFromInt(new_width - 1));
    const y_ratio = @as(f32, @floatFromInt(src.height - 1)) / @as(f32, @floatFromInt(new_height - 1));

    for (0..new_height) |y| {
        for (0..new_width) |x| {
            const gx = @as(f32, @floatFromInt(x)) * x_ratio;
            const gy = @as(f32, @floatFromInt(y)) * y_ratio;

            const x0 = @as(usize, @intFromFloat(gx));
            const y0 = @as(usize, @intFromFloat(gy));
            const x1 = @min(x0 + 1, src.width - 1);
            const y1 = @min(y0 + 1, src.height - 1);

            const tx = gx - @as(f32, @floatFromInt(x0));
            const ty = gy - @as(f32, @floatFromInt(y0));

            const p00 = src.getPixel(x0, y0);
            const p10 = src.getPixel(x1, y0);
            const p01 = src.getPixel(x0, y1);
            const p11 = src.getPixel(x1, y1);

            // Bilinear interpolation
            dst.setPixel(x, y, Pixel{
                .r = @intFromFloat(
                    @as(f32, @floatFromInt(p00.r)) * (1 - tx) * (1 - ty) +
                    @as(f32, @floatFromInt(p10.r)) * tx * (1 - ty) +
                    @as(f32, @floatFromInt(p01.r)) * (1 - tx) * ty +
                    @as(f32, @floatFromInt(p11.r)) * tx * ty,
                ),
                .g = @intFromFloat(
                    @as(f32, @floatFromInt(p00.g)) * (1 - tx) * (1 - ty) +
                    @as(f32, @floatFromInt(p10.g)) * tx * (1 - ty) +
                    @as(f32, @floatFromInt(p01.g)) * (1 - tx) * ty +
                    @as(f32, @floatFromInt(p11.g)) * tx * ty,
                ),
                .b = @intFromFloat(
                    @as(f32, @floatFromInt(p00.b)) * (1 - tx) * (1 - ty) +
                    @as(f32, @floatFromInt(p10.b)) * tx * (1 - ty) +
                    @as(f32, @floatFromInt(p01.b)) * (1 - tx) * ty +
                    @as(f32, @floatFromInt(p11.b)) * tx * ty,
                ),
            });
        }
    }

    return dst;
}

/// Crop ส่วนของภาพ
pub fn crop(
    allocator: Allocator,
    src: *const Image,
    x: usize, y: usize,
    crop_width: usize, crop_height: usize,
) !Image {
    var dst = try Image.init(allocator, @intCast(crop_width), @intCast(crop_height));
    dst.copyFrom(src, x, y, 0, 0, crop_width, crop_height);
    return dst;
}

/// Rotate 90 degrees clockwise
pub fn rotate90(allocator: Allocator, src: *const Image) !Image {
    var dst = try Image.init(allocator, src.height, src.width);

    for (0..src.height) |y| {
        for (0..src.width) |x| {
            const p = src.getPixel(x, y);
            dst.setPixel(src.height - 1 - y, x, p);
        }
    }

    return dst;
}

/// Flip horizontal
pub fn flipHorizontal(img: *Image) void {
    for (0..img.height) |y| {
        var left: usize = 0;
        var right = img.width - 1;
        while (left < right) {
            const pl = img.getPixel(left, y);
            const pr = img.getPixel(right, y);
            img.setPixel(left, y, pr);
            img.setPixel(right, y, pl);
            left += 1;
            right -= 1;
        }
    }
}

/// Flip vertical
pub fn flipVertical(img: *Image) void {
    var top: usize = 0;
    var bottom = img.height - 1;
    while (top < bottom) {
        for (0..img.width) |x| {
            const pt = img.getPixel(x, top);
            const pb = img.getPixel(x, bottom);
            img.setPixel(x, top, pb);
            img.setPixel(x, bottom, pt);
        }
        top += 1;
        bottom -= 1;
    }
}
```

---

## ขั้นตอนที่ 679: Color Manipulation

```zig
// ไฟล์ src/color.zig

const std = @import("std");
const image_mod = @import("image.zig");
const Image = image_mod.Image;
const Pixel = image_mod.Pixel;

/// HSL Color space
pub const HSL = struct {
    h: f32, // Hue: 0-360
    s: f32, // Saturation: 0-1
    l: f32, // Lightness: 0-1

    pub fn fromRGB(r: u8, g: u8, b: u8) HSL {
        const rf = @as(f32, @floatFromInt(r)) / 255.0;
        const gf = @as(f32, @floatFromInt(g)) / 255.0;
        const bf = @as(f32, @floatFromInt(b)) / 255.0;

        const max_c = @max(rf, @max(gf, bf));
        const min_c = @min(rf, @min(gf, bf));
        const delta = max_c - min_c;

        const l = (max_c + min_c) / 2.0;
        var s: f32 = 0;
        var h: f32 = 0;

        if (delta > 0) {
            s = delta / (1.0 - @abs(2.0 * l - 1.0));

            if (max_c == rf) {
                h = 60.0 * @mod((gf - bf) / delta, 6.0);
            } else if (max_c == gf) {
                h = 60.0 * ((bf - rf) / delta + 2.0);
            } else {
                h = 60.0 * ((rf - gf) / delta + 4.0);
            }

            if (h < 0) h += 360.0;
        }

        return HSL{ .h = h, .s = s, .l = l };
    }

    pub fn toRGB(self: HSL) Pixel {
        const c = (1.0 - @abs(2.0 * self.l - 1.0)) * self.s;
        const x = c * (1.0 - @abs(@mod(self.h / 60.0, 2.0) - 1.0));
        const m = self.l - c / 2.0;

        var r: f32 = 0;
        var g: f32 = 0;
        var b: f32 = 0;

        if (self.h < 60) {
            r = c; g = x; b = 0;
        } else if (self.h < 120) {
            r = x; g = c; b = 0;
        } else if (self.h < 180) {
            r = 0; g = c; b = x;
        } else if (self.h < 240) {
            r = 0; g = x; b = c;
        } else if (self.h < 300) {
            r = x; g = 0; b = c;
        } else {
            r = c; g = 0; b = x;
        }

        return Pixel{
            .r = @intFromFloat((r + m) * 255.0),
            .g = @intFromFloat((g + m) * 255.0),
            .b = @intFromFloat((b + m) * 255.0),
        };
    }
};

/// Hue rotation
pub fn rotateHue(img: *Image, degrees: f32) void {
    for (img.pixels) |*pixel| {
        var hsl = HSL.fromRGB(pixel.r, pixel.g, pixel.b);
        hsl.h = @mod(hsl.h + degrees, 360.0);
        const rgb = hsl.toRGB();
        pixel.r = rgb.r;
        pixel.g = rgb.g;
        pixel.b = rgb.b;
    }
}

/// Color tinting
pub fn tint(img: *Image, color: Pixel, amount: f32) void {
    for (img.pixels) |*pixel| {
        pixel.r = @intFromFloat(
            @as(f32, @floatFromInt(pixel.r)) * (1.0 - amount) +
            @as(f32, @floatFromInt(color.r)) * amount
        );
        pixel.g = @intFromFloat(
            @as(f32, @floatFromInt(pixel.g)) * (1.0 - amount) +
            @as(f32, @floatFromInt(color.g)) * amount
        );
        pixel.b = @intFromFloat(
            @as(f32, @floatFromInt(pixel.b)) * (1.0 - amount) +
            @as(f32, @floatFromInt(color.b)) * amount
        );
    }
}

/// Sepia tone effect
pub fn sepia(img: *Image) void {
    for (img.pixels) |*pixel| {
        const r = @as(f32, @floatFromInt(pixel.r));
        const g = @as(f32, @floatFromInt(pixel.g));
        const b = @as(f32, @floatFromInt(pixel.b));

        const new_r = @min(255.0, r * 0.393 + g * 0.769 + b * 0.189);
        const new_g = @min(255.0, r * 0.349 + g * 0.686 + b * 0.168);
        const new_b = @min(255.0, r * 0.272 + g * 0.534 + b * 0.131);

        pixel.r = @intFromFloat(new_r);
        pixel.g = @intFromFloat(new_g);
        pixel.b = @intFromFloat(new_b);
    }
}

/// Posterize (ลดจำนวนสี)
pub fn posterize(img: *Image, levels: u8) void {
    if (levels < 2) return;
    const step = 255.0 / @as(f32, @floatFromInt(levels - 1));

    for (img.pixels) |*pixel| {
        pixel.r = @intFromFloat(@round(@as(f32, @floatFromInt(pixel.r)) / step) * step);
        pixel.g = @intFromFloat(@round(@as(f32, @floatFromInt(pixel.g)) / step) * step);
        pixel.b = @intFromFloat(@round(@as(f32, @floatFromInt(pixel.b)) / step) * step);
    }
}
```

---

## ขั้นตอนที่ 680: Complete Image Processing Pipeline

```zig
// ไฟล์ src/main.zig

const std = @import("std");
const image_mod = @import("image.zig");
const bmp_mod = @import("bmp.zig");
const filters_mod = @import("filters.zig");
const transform_mod = @import("transform.zig");
const color_mod = @import("color.zig");

/// Pipeline operation
pub const Operation = union(enum) {
    grayscale,
    invert,
    blur: u32,        // radius
    sharpen,
    brightness: i16,  // delta
    contrast: f32,    // factor
    sepia,
    edge_detect,
    resize: struct { width: u32, height: u32 },
    rotate90,
    flip_h,
    flip_v,
    threshold: u8,
    saturation: f32,
    hue_rotate: f32,
    posterize: u8,
    crop: struct { x: usize, y: usize, w: usize, h: usize },
};

/// Image Processing Pipeline
pub const Pipeline = struct {
    allocator: std.mem.Allocator,
    operations: std.ArrayList(Operation),

    const Self = @This();

    pub fn init(allocator: std.mem.Allocator) Self {
        return Self{
            .allocator = allocator,
            .operations = std.ArrayList(Operation).init(allocator),
        };
    }

    pub fn deinit(self: *Self) void {
        self.operations.deinit();
    }

    pub fn add(self: *Self, op: Operation) !void {
        try self.operations.append(op);
    }

    pub fn execute(self: *Self, img: *image_mod.Image) !void {
        for (self.operations.items) |op| {
            try self.applyOperation(img, op);
        }
    }

    fn applyOperation(self: *Self, img: *image_mod.Image, op: Operation) !void {
        switch (op) {
            .grayscale => filters_mod.grayscale(img),
            .invert => filters_mod.invert(img),
            .blur => |radius| try filters_mod.boxBlur(self.allocator, img, radius),
            .sharpen => try filters_mod.sharpen(self.allocator, img),
            .brightness => |delta| filters_mod.brightness(img, delta),
            .contrast => |factor| filters_mod.contrast(img, factor),
            .sepia => color_mod.sepia(img),
            .edge_detect => try filters_mod.sobelEdge(self.allocator, img),
            .rotate90 => {
                var rotated = try transform_mod.rotate90(self.allocator, img);
                img.deinit();
                img.* = rotated;
            },
            .flip_h => transform_mod.flipHorizontal(img),
            .flip_v => transform_mod.flipVertical(img),
            .threshold => |t| filters_mod.threshold(img, t),
            .saturation => |factor| filters_mod.saturation(img, factor),
            .hue_rotate => |degrees| color_mod.rotateHue(img, degrees),
            .posterize => |levels| color_mod.posterize(img, levels),
            .resize => |r| {
                var resized = try transform_mod.resizeBilinear(self.allocator, img, r.width, r.height);
                img.deinit();
                img.* = resized;
            },
            .crop => |c| {
                var cropped = try transform_mod.crop(self.allocator, img, c.x, c.y, c.w, c.h);
                img.deinit();
                img.* = cropped;
            },
        }
    }
};

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 3) {
        std.debug.print("Usage: imgproc <input.bmp> <output.bmp> [operations...]\n", .{});
        std.debug.print("Operations: grayscale, invert, blur:3, sharpen, brightness:50,\n", .{});
        std.debug.print("            contrast:1.5, sepia, edges, resize:800x600,\n", .{});
        std.debug.print("            rotate90, fliph, flipv, threshold:128\n", .{});
        return;
    }

    const input_path = args[1];
    const output_path = args[2];

    std.log.info("Loading image: {s}", .{input_path});
    var img = bmp_mod.readBmp(allocator, input_path) catch |err| {
        std.log.err("Failed to load image: {}", .{err});
        return err;
    };
    defer img.deinit();

    std.log.info("Image size: {}x{}", .{ img.width, img.height });

    // สร้าง pipeline จาก arguments
    var pipeline = Pipeline.init(allocator);
    defer pipeline.deinit();

    for (args[3..]) |arg| {
        if (std.mem.eql(u8, arg, "grayscale")) {
            try pipeline.add(.grayscale);
        } else if (std.mem.eql(u8, arg, "invert")) {
            try pipeline.add(.invert);
        } else if (std.mem.eql(u8, arg, "sepia")) {
            try pipeline.add(.sepia);
        } else if (std.mem.eql(u8, arg, "edges")) {
            try pipeline.add(.edge_detect);
        } else if (std.mem.eql(u8, arg, "sharpen")) {
            try pipeline.add(.sharpen);
        } else if (std.mem.eql(u8, arg, "rotate90")) {
            try pipeline.add(.rotate90);
        } else if (std.mem.eql(u8, arg, "fliph")) {
            try pipeline.add(.flip_h);
        } else if (std.mem.eql(u8, arg, "flipv")) {
            try pipeline.add(.flip_v);
        } else if (std.mem.startsWith(u8, arg, "blur:")) {
            const radius = std.fmt.parseInt(u32, arg[5..], 10) catch 3;
            try pipeline.add(.{ .blur = radius });
        } else if (std.mem.startsWith(u8, arg, "brightness:")) {
            const delta = std.fmt.parseInt(i16, arg[11..], 10) catch 0;
            try pipeline.add(.{ .brightness = delta });
        } else if (std.mem.startsWith(u8, arg, "contrast:")) {
            const factor = std.fmt.parseFloat(f32, arg[9..]) catch 1.0;
            try pipeline.add(.{ .contrast = factor });
        } else if (std.mem.startsWith(u8, arg, "threshold:")) {
            const t = std.fmt.parseInt(u8, arg[10..], 10) catch 128;
            try pipeline.add(.{ .threshold = t });
        } else if (std.mem.startsWith(u8, arg, "resize:")) {
            const dims = arg[7..];
            if (std.mem.indexOf(u8, dims, "x")) |x_pos| {
                const w = std.fmt.parseInt(u32, dims[0..x_pos], 10) catch 800;
                const h = std.fmt.parseInt(u32, dims[x_pos + 1 ..], 10) catch 600;
                try pipeline.add(.{ .resize = .{ .width = w, .height = h } });
            }
        } else {
            std.log.warn("Unknown operation: {s}", .{arg});
        }
    }

    // Execute pipeline
    std.log.info("Applying {} operations...", .{pipeline.operations.items.len});
    try pipeline.execute(&img);

    // Save output
    std.log.info("Saving to: {s}", .{output_path});
    try bmp_mod.writeBmp(&img, output_path);

    std.log.info("Done!", .{});
}

test "Image basic operations" {
    const allocator = std.testing.allocator;

    var img = try image_mod.Image.init(allocator, 100, 100);
    defer img.deinit();

    img.fill(image_mod.Pixel.WHITE);

    const p = img.getPixel(50, 50);
    try std.testing.expectEqual(image_mod.Pixel.WHITE, p);

    img.setPixel(0, 0, image_mod.Pixel.RED);
    const red = img.getPixel(0, 0);
    try std.testing.expectEqual(image_mod.Pixel.RED, red);
}

test "Grayscale conversion" {
    const allocator = std.testing.allocator;
    var img = try image_mod.Image.init(allocator, 10, 10);
    defer img.deinit();

    img.fill(.{ .r = 100, .g = 150, .b = 200 });
    filters_mod.grayscale(&img);

    const p = img.getPixel(0, 0);
    try std.testing.expect(p.r == p.g and p.g == p.b);
}

test "Resize" {
    const allocator = std.testing.allocator;
    var src = try image_mod.Image.init(allocator, 100, 100);
    defer src.deinit();

    src.fill(image_mod.Pixel.RED);

    var resized = try transform_mod.resizeNearest(allocator, &src, 50, 50);
    defer resized.deinit();

    try std.testing.expect(resized.width == 50);
    try std.testing.expect(resized.height == 50);
    try std.testing.expectEqual(image_mod.Pixel.RED, resized.getPixel(25, 25));
}
```

---

## สรุปโปรเจกต์ Image Processing

| ความสามารถ | รายละเอียด |
|-------------|-----------|
| File I/O | BMP read/write (24-bit, 32-bit) |
| Pixel Ops | Direct manipulation, blending |
| Color | Grayscale, threshold, invert |
| Adjustment | Brightness, contrast, gamma, levels |
| Filters | Box blur, Gaussian, sharpen |
| Edge | Sobel, Canny |
| Transform | Resize (NN/bilinear), crop, rotate, flip |
| Color Space | RGB ↔ HSL conversion |
| Effects | Sepia, posterize, hue rotation |
| Pipeline | Chainable operations |

### การใช้งาน

```bash
# Build
zig build

# Grayscale + blur
./zig-out/bin/imgproc input.bmp output.bmp grayscale blur:5

# Edge detection
./zig-out/bin/imgproc input.bmp edges.bmp edges

# Sepia + resize
./zig-out/bin/imgproc input.bmp output.bmp sepia resize:800x600

# Run tests
zig build test
```
