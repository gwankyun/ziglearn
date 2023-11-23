---
title: "第二章 - 标准模式"
weight: 3
date: 2023-09-11 18:00:00
description: "第二章 - 本章将涵盖Zig语言的标准库细节。"
---

可以在[这里](https://ziglang.org/documentation/master/std/)找到自动生成的标准库文档。安装[ZLS](https://github.com/zigtools/zls/) 还可以帮助你探索提供补全功能的标准库。

# 分配器

Zig标准库提供了一种分配内存的模式，它允许程序员精确地选择如何在标准库中进行内存分配——标准库中不会在你背后进行任何分配。

最基本的分配器是[`std.heap.page_allocator`](https://ziglang.org/documentation/master/std/#A;std:heap.page_allocator)。每当这个分配器进行分配时，它都会向你的操作系统请求整个内存页；单个字节的分配可能会保留多个千兆字节。由于向操作系统请求内存需要一个系统调用，这对于速度而言也是极其低效的。

在这里，我们将100个字节分配为`[]u8`。请注意，defer是如何与free结合使用的——这是Zig中内存管理的一种常见模式。

```zig
const std = @import("std");
const expect = std.testing.expect;

test "allocation" {
    const allocator = std.heap.page_allocator;

    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);

    try expect(memory.len == 100);
    try expect(@TypeOf(memory) == []u8);
}
```

[`std.heap.FixedBufferAllocator`](https://ziglang.org/documentation/master/std/#A;std:heap.FixedBufferAllocator)是一个分配器，它将内存分配到一个固定的缓冲区中，并且不进行任何堆分配。这在不需要使用堆时很有用，例如，在编写内核时。也可能是性能方面的原因。如果字节耗尽，它会给你错误`OutOfMemory`。

```zig
test "fixed buffer allocator" {
    var buffer: [1000]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);

    try expect(memory.len == 100);
    try expect(@TypeOf(memory) == []u8);
}
```

[`std.heap.ArenaAllocator`](https://ziglang.org/documentation/master/std/#A;std:heap.ArenaAllocator)接受一个子分配器，允许你多次分配，只释放一次。在这里，`.deinit()`在竞技场上被调用，这将释放所有内存。在这个例子中使用`allocator.free`将是no-op（即什么都不做）。

```zig
test "arena allocator" {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    _ = try allocator.alloc(u8, 1);
    _ = try allocator.alloc(u8, 10);
    _ = try allocator.alloc(u8, 100);
}
```

`alloc`和`free`用于切片。对于单个项，考虑使用`create`和`destroy`。

```zig
test "allocator create/destroy" {
    const byte = try std.heap.page_allocator.create(u8);
    defer std.heap.page_allocator.destroy(byte);
    byte.* = 128;
}
```

Zig标准库也有一个通用的分配器。这是一个安全的分配器，可以防止双重释放，释放后使用，并可以检测泄漏。安全检查和线程安全可以通过它的配置结构（下面留空）关闭。Zig的GPA是为了安全性而不是性能而设计的，但可能仍然比page_allocator快很多倍。

```zig
test "GPA" {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();
    defer {
        const deinit_status = gpa.deinit();
        //失败的测试；在返回后，不能尝试在defer中执行defer
        if (deinit_status == .leak) expect(false) catch @panic("TEST FAIL");
    }

    const bytes = try allocator.alloc(u8, 100);
    defer allocator.free(bytes);
}
```

对于高性能（但很少有安全特性！），可以考虑使用[`std.heap.c_allocator`](https://ziglang.org/documentation/master/std/#A;std:heap.c_allocator)。然而，这样做的缺点是需要链接Libc，这可以用`-lc`来完成。

Benjamin Feng的讲座[*什么是内存分配器？*](https://www.youtube.com/watch?v=vHWiDx_l4V0)详细介绍这个主题，并介绍分配器的实现。

# Arraylist

[`std.ArrayList`](https://ziglang.org/documentation/master/std/#A;std:ArrayList)在整个Zig中都很常用，它可以作为一个大小可以改变的缓冲区。`std.ArrayList(T)`类似于C++的`std::vector<T>`和Rust的`Vec<T>`。`deinit()`方法释放了ArrayList的所有内存。内存可以通过它的切片字段——`.items`进行读写。

这里我们将介绍测试分配器的用法。这是一个特殊的分配器，只在测试中工作，可以检测内存泄漏。在你的代码中，使用任何合适的分配器。

```zig
const eql = std.mem.eql;
const ArrayList = std.ArrayList;
const test_allocator = std.testing.allocator;

test "arraylist" {
    var list = ArrayList(u8).init(test_allocator);
    defer list.deinit();
    try list.append('H');
    try list.append('e');
    try list.append('l');
    try list.append('l');
    try list.append('o');
    try list.appendSlice(" World!");

    try expect(eql(u8, list.items, "Hello World!"));
}
```

# 文件系统

让我们在当前工作目录中创建并打开一个文件，向其写入，然后从中读取。在这里，我们必须使用`.seekTo`返回到文件的开头，然后再读取我们所写的内容。

```zig
test "createFile, write, seekTo, read" {
    const file = try std.fs.cwd().createFile(
        "junk_file.txt",
        .{ .read = true },
    );
    defer file.close();

    const bytes_written = try file.writeAll("Hello File!");
    _ = bytes_written;

    var buffer: [100]u8 = undefined;
    try file.seekTo(0);
    const bytes_read = try file.readAll(&buffer);

    try expect(eql(u8, buffer[0..bytes_read], "Hello File!"));
}
```

函数[`std.fs.openFileAbsolute`](https://ziglang.org/documentation/master/std/#A;std:fs.openFileAbsolute)和类似的绝对函数是存在的，但是我们在这里不测试它们。

通过对文件使用`.stat()`，我们可以获得关于文件的各种信息。`Stat`还包含.inode和.mode字段，但这里没有对它们进行测试，因为它们依赖于当前操作系统的类型。

```zig
test "file stat" {
    const file = try std.fs.cwd().createFile(
        "junk_file2.txt",
        .{ .read = true },
    );
    defer file.close();
    const stat = try file.stat();
    try expect(stat.size == 0);
    try expect(stat.kind == .file);
    try expect(stat.ctime <= std.time.nanoTimestamp());
    try expect(stat.mtime <= std.time.nanoTimestamp());
    try expect(stat.atime <= std.time.nanoTimestamp());
}
```

我们可以创建目录并遍历其内容。这里我们将使用迭代器（稍后讨论）。这个目录（及其内容）将在测试结束后被删除。

```zig
test "make dir" {
    try std.fs.cwd().makeDir("test-tmp");
    var iter_dir = try std.fs.cwd().openIterableDir(
        "test-tmp",
        .{},
    );
    defer {
        iter_dir.close();
        std.fs.cwd().deleteTree("test-tmp") catch unreachable;
    }

    _ = try iter_dir.dir.createFile("x", .{});
    _ = try iter_dir.dir.createFile("y", .{});
    _ = try iter_dir.dir.createFile("z", .{});

    var file_count: usize = 0;
    var iter = iter_dir.iterate();
    while (try iter.next()) |entry| {
        if (entry.kind == .file) file_count += 1;
    }

    try expect(file_count == 3);
}
```

# Reader和Writer

[`std.io.Writer`](https://ziglang.org/documentation/master/std/#A;std:io.Writer)和[`std.io.Reader`](https://ziglang.org/documentation/master/std/#A;std:io.Reader)提供了使用IO的标准方法。`std.ArrayList(u8)`有一个`writer`方法，它提供了一个写入器。让我们用它。

```zig
test "io writer usage" {
    var list = ArrayList(u8).init(test_allocator);
    defer list.deinit();
    const bytes_written = try list.writer().write(
        "Hello World!",
    );
    try expect(bytes_written == 12);
    try expect(eql(u8, list.items, "Hello World!"));
}
```

这里我们将使用读取器将文件的内容复制到已分配的缓冲区中。[`readAllAlloc`](https://ziglang.org/documentation/master/std/#A;std:io.Reader.readAllAlloc)的第二个参数是它可以分配的最大大小；如果文件比这个大，它将返回`error.StreamTooLong`。

```zig
test "io reader usage" {
    const message = "Hello File!";

    const file = try std.fs.cwd().createFile(
        "junk_file2.txt",
        .{ .read = true },
    );
    defer file.close();

    try file.writeAll(message);
    try file.seekTo(0);

    const contents = try file.reader().readAllAlloc(
        test_allocator,
        message.len,
    );
    defer test_allocator.free(contents);

    try expect(eql(u8, contents, message));
}
```

读取器的一个常见用例是读取到下一行（例如用户输入）。在这里，我们将使用[`std.io.getStdIn()`](https://ziglang.org/documentation/master/std/#A;std:io.getStdIn)文件完成此操作。

```zig
fn nextLine(reader: anytype, buffer: []u8) !?[]const u8 {
    var line = (try reader.readUntilDelimiterOrEof(
        buffer,
        '\n',
    )) orelse return null;
    // 删除恼人的windows特有回车符
    if (@import("builtin").os.tag == .windows) {
        return std.mem.trimRight(u8, line, "\r");
    } else {
        return line;
    }
}

test "read until next line" {
    const stdout = std.io.getStdOut();
    const stdin = std.io.getStdIn();

    try stdout.writeAll(
        \\ Enter your name:
    );

    var buffer: [100]u8 = undefined;
    const input = (try nextLine(stdin.reader(), &buffer)).?;
    try stdout.writer().print(
        "Your name is: \"{s}\"\n",
        .{input},
    );
}
```

[`std.io.Writer`](https://ziglang.org/documentation/master/std/#A;std:io.Writer)类型包括上下文类型、错误集和写函数。写函数必须接受上下文类型和字节切片。write函数还必须返回Writer类型的错误集和写入字节数的错误联合。让我们创建一个实现写入器的类型。

```zig
// 不要创建这样的类型！使用一个
// 带有固定缓冲区分配器的数组列表
const MyByteList = struct {
    data: [100]u8 = undefined,
    items: []u8 = &[_]u8{},

    const Writer = std.io.Writer(
        *MyByteList,
        error{EndOfBuffer},
        appendWrite,
    );

    fn appendWrite(
        self: *MyByteList,
        data: []const u8,
    ) error{EndOfBuffer}!usize {
        if (self.items.len + data.len > self.data.len) {
            return error.EndOfBuffer;
        }
        std.mem.copy(
            u8,
            self.data[self.items.len..],
            data,
        );
        self.items = self.data[0 .. self.items.len + data.len];
        return data.len;
    }

    fn writer(self: *MyByteList) Writer {
        return .{ .context = self };
    }
};

test "custom writer" {
    var bytes = MyByteList{};
    _ = try bytes.writer().write("Hello");
    _ = try bytes.writer().write(" Writer!");
    try expect(eql(u8, bytes.items, "Hello Writer!"));
}
```

# 格式化

[`std.fmt`](https://ziglang.org/documentation/master/std/#A;std:fmt) 提供了格式化字符串数据的方法。

创建格式化字符串的基本示例。格式字符串必须是编译时已知的。这里的`d`表示我们需要一个十进制数。

```zig
test "fmt" {
    const string = try std.fmt.allocPrint(
        test_allocator,
        "{d} + {d} = {d}",
        .{ 9, 10, 19 },
    );
    defer test_allocator.free(string);

    try expect(eql(u8, string, "9 + 10 = 19"));
}
```

Writer方便地使用`print`方法，其工作原理类似。

```zig
test "print" {
    var list = std.ArrayList(u8).init(test_allocator);
    defer list.deinit();
    try list.writer().print(
        "{} + {} = {}",
        .{ 9, 10, 19 },
    );
    try expect(eql(u8, list.items, "9 + 10 = 19"));
}
```

花点时间欣赏一下，你现在已经从上到下了解了打印Hello World的工作原理。[`std.debug.print`](https://ziglang.org/documentation/master/std/#A;std:debug.print)的工作原理相同，除了它写到stderr，并由互斥锁保护。

```zig
test "hello world" {
    const out_file = std.io.getStdOut();
    try out_file.writer().print(
        "Hello, {s}!\n",
        .{"World"},
    );
}
```

到目前为止，我们已经使用了`{s}`格式说明符来打印字符串。在这里，我们将使用`{any}`，它给了我们默认格式。

```zig
test "array printing" {
    const string = try std.fmt.allocPrint(
        test_allocator,
        "{any} + {any} = {any}",
        .{
            @as([]const u8, &[_]u8{ 1, 4 }),
            @as([]const u8, &[_]u8{ 2, 5 }),
            @as([]const u8, &[_]u8{ 3, 9 }),
        },
    );
    defer test_allocator.free(string);

    try expect(eql(
        u8,
        string,
        "{ 1, 4 } + { 2, 5 } = { 3, 9 }",
    ));
}
```

让我们通过给一个`format`函数来创建一个自定义格式的类型。此函数必须标记为`pub`，以便std.fmt可以访问它（稍后将详细介绍包）。你可能会注意到使用`{s}`而不是`{}`——这是字符串的格式说明符（更多格式说明符在稍后）。在这里，`{}`默认为数组打印而不是字符串打印。

```zig
const Person = struct {
    name: []const u8,
    birth_year: i32,
    death_year: ?i32,
    pub fn format(
        self: Person,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;

        try writer.print("{s} ({}-", .{
            self.name, self.birth_year,
        });

        if (self.death_year) |year| {
            try writer.print("{}", .{year});
        }

        try writer.writeAll(")");
    }
};

test "custom fmt" {
    const john = Person{
        .name = "John Carmack",
        .birth_year = 1970,
        .death_year = null,
    };

    const john_string = try std.fmt.allocPrint(
        test_allocator,
        "{s}",
        .{john},
    );
    defer test_allocator.free(john_string);

    try expect(eql(
        u8,
        john_string,
        "John Carmack (1970-)",
    ));

    const claude = Person{
        .name = "Claude Shannon",
        .birth_year = 1916,
        .death_year = 2001,
    };

    const claude_string = try std.fmt.allocPrint(
        test_allocator,
        "{s}",
        .{claude},
    );
    defer test_allocator.free(claude_string);

    try expect(eql(
        u8,
        claude_string,
        "Claude Shannon (1916-2001)",
    ));
}
```

# JSON

让我们使用流解析器将JSON字符串解析为结构类型。

```zig
const Place = struct { lat: f32, long: f32 };

test "json parse" {
    const parsed = try std.json.parseFromSlice(
        Place,
        test_allocator,
        \\{ "lat": 40.684540, "long": -74.401422 }
    ,
        .{},
    );
    defer parsed.deinit();

    const place = parsed.value;

    try expect(place.lat == 40.684540);
    try expect(place.long == -74.401422);
}
```

并使用stringify将任意数据转换为字符串。

```zig
test "json stringify" {
    const x = Place{
        .lat = 51.997664,
        .long = -0.740687,
    };

    var buf: [100]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buf);
    var string = std.ArrayList(u8).init(fba.allocator());
    try std.json.stringify(x, .{}, string.writer());

    try expect(eql(u8, string.items,
        \\{"lat":5.199766540527344e+01,"long":-7.406870126724243e-01}
    ));
}
```

JSON解析器需要为javascript的字符串、数组和映射类型分配一个分配器。

```zig
test "json parse with strings" {
    const User = struct { name: []u8, age: u16 };

    const parsed = try std.json.parseFromSlice(User, test_allocator,
        \\{ "name": "Joe", "age": 25 }
    , .{},);
    defer parsed.deinit();

    const user = parsed.value;

    try expect(eql(u8, user.name, "Joe"));
    try expect(user.age == 25);
}
```

# 随机数

这里，我们使用64位随机种子创建一个新的prng。a、b、c和d是通过这个prng给出的随机值。给出的表达式c和d值是相等的。`DefaultPrng`是`Xoroshiro128`；在std.rand中还有其他可用的prng。

```zig
test "random numbers" {
    var prng = std.rand.DefaultPrng.init(blk: {
        var seed: u64 = undefined;
        try std.os.getrandom(std.mem.asBytes(&seed));
        break :blk seed;
    });
    const rand = prng.random();

    const a = rand.float(f32);
    const b = rand.boolean();
    const c = rand.int(u8);
    const d = rand.intRangeAtMost(u8, 0, 255);

    //禁止未使用的常量编译错误
    _ = .{ a, b, c, d };
}
```

加密安全随机也是可用的。

```zig
test "crypto random numbers" {
    const rand = std.crypto.random;

    const a = rand.float(f32);
    const b = rand.boolean();
    const c = rand.int(u8);
    const d = rand.intRangeAtMost(u8, 0, 255);

    //禁止未使用的常量编译错误
    _ = .{ a, b, c, d };
}
```

# 加密

[`std.crypto`](https://ziglang.org/documentation/master/std/#A;std:crypto)包含许多加密实用程序，包括：
-  AES（Aes128、Aes256）
-  Diffie-Hellman密钥交换（x25519）
-  椭圆曲线算法（curve25519、edwards25519、ristretto255）
-  加密安全散列（blake2、Blake3、Gimli、Md5、sha1、sha2、sha3）
-  MAC函数（Ghash、Poly1305）
-  流密码（ChaCha20IETF、ChaCha20With64BitNonce、XChaCha20IETF、Salsa20、XSalsa20）

这个列表是不详尽的。要了解更深入的信息，请尝试看看[A tour of std.crypto in Zig 0.7.0 - Frank Denis](https://www.youtube.com/watch?v=9t6Y7KoCvyk)。

# 线程

虽然Zig提供了更高级的编写并发和并行代码的方法，但[`std.Thread`](https://ziglang.org/documentation/master/std/#A;std:Thread)也可用于利用操作系统线程。让我们利用一个操作系统线程。

```zig
fn ticker(step: u8) void {
    while (true) {
        std.time.sleep(1 * std.time.ns_per_s);
        tick += @as(isize, step);
    }
}

var tick: isize = 0;

test "threading" {
    var thread = try std.Thread.spawn(.{}, ticker, .{@as(u8, 1)});
    _ = thread;
    try expect(tick == 0);
    std.time.sleep(3 * std.time.ns_per_s / 2);
    try expect(tick == 1);
}
```

但是，如果没有线程安全策略，线程就不是特别有用。

# 散列映射

标准库提供了[`std.AutoHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoHashMap)，它允许你轻松地从键类型和值类型创建散列映射类型。这必须用到分配器。

让我们把一些值放到哈希映射中。

```zig
test "hashing" {
    const Point = struct { x: i32, y: i32 };

    var map = std.AutoHashMap(u32, Point).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put(1525, .{ .x = 1, .y = -4 });
    try map.put(1550, .{ .x = 2, .y = -3 });
    try map.put(1575, .{ .x = 3, .y = -2 });
    try map.put(1600, .{ .x = 4, .y = -1 });

    try expect(map.count() == 4);

    var sum = Point{ .x = 0, .y = 0 };
    var iterator = map.iterator();

    while (iterator.next()) |entry| {
        sum.x += entry.value_ptr.x;
        sum.y += entry.value_ptr.y;
    }

    try expect(sum.x == 10);
    try expect(sum.y == -10);
}
```

`.fetchPut`将一个值放入哈希映射中，如果该键先前有值，则返回一个值。

```zig
test "fetchPut" {
    var map = std.AutoHashMap(u8, f32).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put(255, 10);
    const old = try map.fetchPut(255, 100);

    try expect(old.?.value == 10);
    try expect(map.get(255).? == 100);
}
```

当你需要字符串作为键时，还提供了[`std.StringHashMap`](https://ziglang.org/documentation/master/std/#A;std:StringHashMap)。

```zig
test "string hashmap" {
    var map = std.StringHashMap(enum { cool, uncool }).init(
        test_allocator,
    );
    defer map.deinit();

    try map.put("loris", .uncool);
    try map.put("me", .cool);

    try expect(map.get("me").? == .cool);
    try expect(map.get("loris").? == .uncool);
}
```

[`std.StringHashMap`](https://ziglang.org/documentation/master/std/#A;std:StringHashMap)和[`std.AutoHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoHashMap)只是[`std.HashMap`](https://ziglang.org/documentation/master/std/#A;std:HashMap)的包装。如果这两种方法不能满足你的需求，那么直接使用[`std.HashMap`](https://ziglang.org/documentation/master/std/#A;std:HashMap)可以为你提供更多的控制。

如果你的元素由数组支持是需要的行为，请尝试[`std.ArrayHashMap`](https://ziglang.org/documentation/master/std/#A;std:ArrayHashMap)及其包装器[`std.AutoArrayHashMap`](https://ziglang.org/documentation/master/std/#A;std:AutoArrayHashMap)。

# 栈

[`std.ArrayList`](https://ziglang.org/documentation/master/std/#A;std:ArrayList)提供了将其用作堆栈所需的方法。下面是创建匹配括号列表的示例。

```zig
test "stack" {
    const string = "(()())";
    var stack = std.ArrayList(usize).init(
        test_allocator,
    );
    defer stack.deinit();

    const Pair = struct { open: usize, close: usize };
    var pairs = std.ArrayList(Pair).init(
        test_allocator,
    );
    defer pairs.deinit();

    for (string, 0..) |char, i| {
        if (char == '(') try stack.append(i);
        if (char == ')')
            try pairs.append(.{
                .open = stack.pop(),
                .close = i,
            });
    }

    for (pairs.items, 0..) |pair, i| {
        try expect(std.meta.eql(pair, switch (i) {
            0 => Pair{ .open = 1, .close = 2 },
            1 => Pair{ .open = 3, .close = 4 },
            2 => Pair{ .open = 0, .close = 5 },
            else => unreachable,
        }));
    }
}
```

# 排序

标准库提供了用于就地排序片的实用程序。其基本用法如下。

```zig
test "sorting" {
    var data = [_]u8{ 10, 240, 0, 0, 10, 5 };
    std.mem.sort(u8, &data, {}, comptime std.sort.asc(u8));
    try expect(eql(u8, &data, &[_]u8{ 0, 0, 5, 10, 10, 240 }));
    std.mem.sort(u8, &data, {}, comptime std.sort.desc(u8));
    try expect(eql(u8, &data, &[_]u8{ 240, 10, 10, 5, 0, 0 }));
}
```

[`std.sort.asc`](https://ziglang.org/documentation/master/std/#A;std:sort.asc)和[`.desc`](https://ziglang.org/documentation/master/std/#A;std:sort.desc)在运行时为给定类型创建一个比较函数；如果非数值类型需要排序，用户必须提供自己的比较函数。

[`std.sort.sort`](https://ziglang.org/documentation/master/std/#A;std:sort.sort)的最佳情况为O(n)，平均和最差情况为O(n*log(n))。

# 迭代器

一个常见的习惯用法是，在一个结构类型中有一个`next`函数，它的返回类型是可选的，这样函数就可以返回一个null来表示迭代已经结束。

[`std.mem.SplitIterator`](https://ziglang.org/documentation/master/std/#A;std:mem.SplitIterator)（和略有不同的[`std.mem.TokenIterator`](https://ziglang.org/documentation/master/std/#A;std:mem.TokenIterator)）就是这种模式的一个例子。
```zig
test "split iterator" {
    const text = "robust, optimal, reusable, maintainable, ";
    var iter = std.mem.split(u8, text, ", ");
    try expect(eql(u8, iter.next().?, "robust"));
    try expect(eql(u8, iter.next().?, "optimal"));
    try expect(eql(u8, iter.next().?, "reusable"));
    try expect(eql(u8, iter.next().?, "maintainable"));
    try expect(eql(u8, iter.next().?, ""));
    try expect(iter.next() == null);
}
```

有些迭代器有`!?T`返回类型，而不是?T。`!?T`要求我们在可选的之前解包错误联合，这意味着为到达下一个迭代所做的工作可能会出错。下面是一个使用循环执行此操作的示例。为了使目录迭代器工作，必须使用迭代权限打开[`cwd`](https://ziglang.org/documentation/master/std/#std;fs.cwd)。

```zig
test "iterator looping" {
    var iter = (try std.fs.cwd().openIterableDir(
        ".",
        .{},
    )).iterate();

    var file_count: usize = 0;
    while (try iter.next()) |entry| {
        if (entry.kind == .file) file_count += 1;
    }

    try expect(file_count > 0);
}
```

这里我们将实现一个自定义迭代器。这将迭代字符串切片，生成包含给定字符串的字符串。

```zig
const ContainsIterator = struct {
    strings: []const []const u8,
    needle: []const u8,
    index: usize = 0,
    fn next(self: *ContainsIterator) ?[]const u8 {
        const index = self.index;
        for (self.strings[index..]) |string| {
            self.index += 1;
            if (std.mem.indexOf(u8, string, self.needle)) |_| {
                return string;
            }
        }
        return null;
    }
};

test "custom iterator" {
    var iter = ContainsIterator{
        .strings = &[_][]const u8{ "one", "two", "three" },
        .needle = "e",
    };

    try expect(eql(u8, iter.next().?, "one"));
    try expect(eql(u8, iter.next().?, "three"));
    try expect(iter.next() == null);
}
```

# 格式说明符
[`std.fmt`](https://ziglang.org/documentation/master/std/#std;fmt)提供了格式化各种数据类型的选项。

`std.fmt.fmtSliceHexLower`和`std.fmt.fmtSliceHexUpper`为字符串提供十六进制格式，为整数提供`{x}`和`{X}`。
```zig
const bufPrint = std.fmt.bufPrint;

test "hex" {
    var b: [8]u8 = undefined;

    _ = try bufPrint(&b, "{X}", .{4294967294});
    try expect(eql(u8, &b, "FFFFFFFE"));

    _ = try bufPrint(&b, "{x}", .{4294967294});
    try expect(eql(u8, &b, "fffffffe"));

    _ = try bufPrint(&b, "{}", .{std.fmt.fmtSliceHexLower("Zig!")});
    try expect(eql(u8, &b, "5a696721"));
}
```

`{d}`对数字类型执行十进制格式化。

```zig
test "decimal float" {
    var b: [4]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{d}", .{16.5}),
        "16.5",
    ));
}
```

`{c}`将字节格式化为ascii字符。
```zig
test "ascii fmt" {
    var b: [1]u8 = undefined;
    _ = try bufPrint(&b, "{c}", .{66});
    try expect(eql(u8, &b, "B"));
}
```

`std.fmt.fmtIntSizeDec`和`std.fmt.fmtIntSizeBin`以公制（1000）和2次幂（1024）为基础的表示法输出内存大小。

```zig
test "B Bi" {
    var b: [32]u8 = undefined;

    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1)}), "1B"));
    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1)}), "1B"));

    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1024)}), "1.024kB"));
    try expect(eql(u8, try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1024)}), "1KiB"));

    try expect(eql(
        u8,
        try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeDec(1024 * 1024 * 1024)}),
        "1.073741824GB",
    ));
    try expect(eql(
        u8,
        try bufPrint(&b, "{}", .{std.fmt.fmtIntSizeBin(1024 * 1024 * 1024)}),
        "1GiB",
    ));
}
```

`{b}`和`{o}`以二进制和八进制格式输出整数。

```zig
test "binary, octal fmt" {
    var b: [8]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{b}", .{254}),
        "11111110",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{o}", .{254}),
        "376",
    ));
}
```

`{*}`执行指针格式化，打印地址而不是值。
```zig
test "pointer fmt" {
    var b: [16]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{*}", .{@as(*u8, @ptrFromInt(0xDEADBEEF))}),
        "u8@deadbeef",
    ));
}
```

`{e}`以科学记数法输出浮点数。
```zig
test "scientific" {
    var b: [16]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{e}", .{3.14159}),
        "3.14159e+00",
    ));
}
```

`{s}`输出字符串。
```zig
test "string fmt" {
    var b: [6]u8 = undefined;
    const hello: [*:0]const u8 = "hello!";

    try expect(eql(
        u8,
        try bufPrint(&b, "{s}", .{hello}),
        "hello!",
    ));
}
```

这个列表并未详尽。

# 高级格式化

到目前为止，我们只讨论了格式化说明符。格式字符串实际上遵循这种格式，在每对方括号之间是一个参数，你必须用某些东西替换。

`{[position][specifier]:[fill][alignment][width].[precision]}`

| 名字      | 意义                                                                                 |
|-----------|-----------------------------------------------------------------------------------------|
| Position  | 要插入参数的索引                                       |
| Specifier | 与类型相关的格式化选项                                                      |
| Fill      | 用于填充的单个字符                                                     |
| Alignment | '<'、'^'或'>'之一，代表左、中及右对齐 |
| Width     | 字段的总宽度（字符）                                               |
| Precision | 一个格式化的数字应该有多少位小数                                        |


位置使用。
```zig
test "position" {
    var b: [3]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{0s}{0s}{1s}", .{ "a", "b" }),
        "aab",
    ));
}
```

填充、对齐和宽度正在使用。
```zig
test "fill, alignment, width" {
    var b: [6]u8 = undefined;

    try expect(eql(
        u8,
        try bufPrint(&b, "{s: <5}", .{"hi!"}),
        "hi!  ",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{s:_^6}", .{"hi!"}),
        "_hi!__",
    ));

    try expect(eql(
        u8,
        try bufPrint(&b, "{s:!>4}", .{"hi!"}),
        "!hi!",
    ));
}
```

使用具有精度的说明符。
```zig
test "precision" {
    var b: [4]u8 = undefined;
    try expect(eql(
        u8,
        try bufPrint(&b, "{d:.2}", .{3.14159}),
        "3.14",
    ));
}
```

# 第二章结束

本章未完。在未来，它将包含如下内容：

- 任意精度数学
- 链表
- 队列
- 互斥锁
- 原子
- 搜索
- 日志

欢迎反馈和PR。
