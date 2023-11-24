---
title: "第四章 - 和C一起工作"
weight: 5
date: 2023-01-11 18:00:00
description: "第四章 - 了解Zig编程语言如何使用C代码。本教程涵盖C数据类型、FFI、用C构建、translate-c等！"
---

Zig从一开始就以C互操作作为一流的特性来设计。在本节中，我们将讨论它是如何工作的。

# ABI

ABI *（应用程序二进制接口）* 是一种标准，涉及：

- 类型在内存中的布局（即类型的大小、对齐方式、偏移量及其字段的布局）
- 链接器中符号的命名（例如名称混淆）
- 函数的调用约定（即函数调用如何在二进制级别工作）

通过定义这些规则而不破坏它们，ABI就被认为是稳定的，这可以用于可靠地将单独编译的多个库、可执行文件或对象链接在一起(可能在不同的机器上或使用不同的编译器)。这允许FFI *（外部函数接口）* 产生，我们可以在编程语言之间共享代码。

Zig原生支持用于`extern`事物的C ABI；使用哪种C ABI取决于你要编译的目标（例如CPU架构，操作系统）。这允许与非Zig编写的代码进行近乎无缝的互操作；C ABI的使用是编程语言中的标准。

Zig内部不使用ABI，这意味着代码应该显式地遵守C ABI，其中需要可复制和定义的二进制级行为。

# C基本类型

Zig为符合C ABI提供了特殊的以`c_`为前缀的类型。它们没有固定的大小，而是根据所使用的ABI而改变大小。

| 类型         | C等效              | 最小尺寸（bits） |
|--------------|-------------------|---------------------|
| c_short      | short             | 16                  |
| c_ushort     | unsigned short    | 16                  |
| c_int        | int               | 16                  |
| c_uint       | unsigned int      | 16                  |
| c_long       | long              | 32                  |
| c_ulong      | unsigned long     | 32                  |
| c_longlong   | long long         | 64                  |
| c_ulonglong  | unsigned longlong | 64                  |
| c_longdouble | long double       | N/A                 |
| anyopaque    | void              | N/A                 |

注意：C的void（和Zig的`anyopaque`）有一个未知的非零大小。Zig的`void`是一个真正的零大小类型。

# 调用约定

调用约定描述了如何调用函数。这包括如何向函数提供参数（即它们在寄存器中或堆栈上的位置，以及如何），以及如何接收返回值。

在Zig中，可以将属性`callconv`赋给函数。可用的调用约定可以在[std.builtin.CallingConvention](https://ziglang.org/documentation/master/std/#A;std:builtin.CallingConvention)中找到。这里我们使用cdecl调用约定。
```zig
fn add(a: u32, b: u32) callconv(.C) u32 {
    return a + b;
}
```

当你从C调用Zig时，用C调用约定标记函数是至关重要的。

# 外部结构体

普通结构在Zig中没有定义的布局；当你想让结构体的布局与C ABI的布局相匹配时，需要`extern`结构体。

让我们创建一个外部结构体。这个测试应该使用带有`gnu` ABI的`x86_64`运行，可以使用`-target x86_64-native-gnu`来完成。

```zig
const expect = @import("std").testing.expect;

const Data = extern struct { a: i32, b: u8, c: f32, d: bool, e: bool };

test "hmm" {
    const x = Data{
        .a = 10005,
        .b = 42,
        .c = -10.5,
        .d = false,
        .e = true,
    };
    const z = @as([*]const u8, @ptrCast(&x));

    try expect(@as(*const i32, @ptrCast(@alignCast(z))).* == 10005);
    try expect(@as(*const u8, @ptrCast(@alignCast(z + 4))).* == 42);
    try expect(@as(*const f32, @ptrCast(@alignCast(z + 8))).* == -10.5);
    try expect(@as(*const bool, @ptrCast(@alignCast(z + 12))).* == false);
    try expect(@as(*const bool, @ptrCast(@alignCast(z + 13))).* == true);
}
```

这就是`x`值里面的内存。

| 字段 | a  | a  | a  | a  | b  |    |    |    | c  | c  | c  | c  | d  | e  |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| 字节 | 15 | 27 | 00 | 00 | 2A | 00 | 00 | 00 | 00 | 00 | 28 | C1 | 00 | 01 | 00 | 00 |

注意中间和末尾是如何有空隙的——这被称为“填充”。这个填充中的数据是未定义的内存，并不总是为零。

由于我们的`x`值是外部结构体的值，我们可以安全地将它传递给期望得到`Data`的C函数，前提是该C函数也使用相同的`gnu` ABI和CPU arch进行编译。

# 对齐

由于电路的原因，CPU在内存中以特定的倍数访问原始值。这可能意味着，例如，`f32`值的地址必须是4的倍数，这意味着`f32`的对齐顺序为4。这种所谓的原始数据类型的“自然对齐”取决于CPU架构。所有对齐都是2的幂次。

较大对齐的数据也具有每个较小对齐的对齐；例如，对齐方式为16的值也具有8、4、2和1的对齐方式。

我们可以使用`align(x)`属性制作特殊对齐的数据。在这里，我们以更大的一致性制作数据。
```zig
const a1: u8 align(8) = 100;
const a2 align(8) = @as(u8, 100);
```
并以较小的对齐方式生成数据。注意：创建较小对齐的数据并不是特别有用。
```zig
const b1: u64 align(1) = 100;
const b2 align(1) = @as(u64, 100);
```

和`const`一样，`align`也是指针的属性。
```zig
test "aligned pointers" {
    const a: u32 align(8) = 5;
    try expect(@TypeOf(&a) == *align(8) const u32);
}
```

让我们利用一个需要对齐指针的函数。

```zig
fn total(a: *align(64) const [64]u8) u32 {
    var sum: u32 = 0;
    for (a) |elem| sum += elem;
    return sum;
}

test "passing aligned data" {
    const x align(64) = [_]u8{10} ** 64;
    try expect(total(&x) == 640);
}
```

# 打包结构体

默认情况下，Zig中的所有结构字段都与[`@alignOf(FieldType)`](https://ziglang.org/documentation/master/#alignOf) （ABI大小）自然对齐，但没有定义布局。有时，你可能希望具有不符合你的C ABI的已定义布局的结构字段。`packed`结构体允许你对结构体字段进行极其精确的控制，允许你逐位放置字段。

在打包结构体中，Zig的整数在空间中取其位宽（例如，`u12`的[`@bitSizeOf`](https://ziglang.org/documentation/master/#bitSizeOf)为12，这意味着它将在打包结构体中占用12位）。bool也占用1位，这意味着你可以轻松实现位标志。

```zig
const MovementState = packed struct {
    running: bool,
    crouching: bool,
    jumping: bool,
    in_air: bool,
};

test "packed struct size" {
    try expect(@sizeOf(MovementState) == 1);
    try expect(@bitSizeOf(MovementState) == 4);
    const state = MovementState{
        .running = true,
        .crouching = true,
        .jumping = true,
        .in_air = true,
    };
    _ = state;
}
```

# 位对齐指针

与对齐指针类似，位对齐指针在其类型中有额外的信息，这些信息告知如何访问数据。当数据不是按字节对齐时，这些是必需的。位对齐信息通常需要在打包结构体中寻址字段。

```zig
test "bit aligned pointers" {
    var x = MovementState{
        .running = false,
        .crouching = false,
        .jumping = false,
        .in_air = false,
    };

    const running = &x.running;
    running.* = true;

    const crouching = &x.crouching;
    crouching.* = true;

    try expect(@TypeOf(running) == *align(1:0:1) bool);
    try expect(@TypeOf(crouching) == *align(1:1:1) bool);

    try expect(@import("std").meta.eql(x, .{
        .running = true,
        .crouching = true,
        .jumping = false,
        .in_air = false,
    }));
}
```

# C指针

到目前为止，我们已经使用了以下类型的指针：

- 单项指针 - `*T`
- 多项指针 - `[*]T`
- 切片 - `[]T`

与前面提到的指针不同，C指针不能处理特殊对齐的数据，可能指向地址`0`。C指针在整数之间来回强制转换，也强制转换为单项和多项指针。当值为`0`的C指针被强制为非可选指针时，这是可检测到的非法行为。

在自动翻译的C代码之外，使用`[*c]`几乎总是一个坏主意，几乎不应该使用。

# Translate-C

Zig提供了命令`zig translate-c`，用于从C源代码进行自动翻译。

用以下内容创建文件`main.c`。
```c
#include <stddef.h>

void int_sort(int* array, size_t count) {
    for (int i = 0; i < count - 1; i++) {
        for (int j = 0; j < count - i - 1; j++) {
            if (array[j] > array[j+1]) {
                int temp = array[j];
                array[j] = array[j+1];
                array[j+1] = temp;
            }
        }
    }
}
```
运行命令`zig translate-c main.c`，将等效的Zig代码输出到控制台（stdout）。你可能希望使用`zig translate-c main.c > int_sort.zig`将其管道到一个文件中（警告Windows用户：PowerShell中的管道会生成一个编码不正确的文件——使用你的编辑器来纠正这个问题）。

在另一个文件中，你可以使用`@import("int_sort.zig")`来使用此函数。

目前生成的代码可能不必要地冗长，尽管translate-c成功地将大多数C代码转换为Zig。在将其编辑成更习惯的代码之前，你可能希望使用translate-c来生成Zig代码；在代码库中从C语言逐渐转换到Zig语言是一个受支持的用例。

# cImport

Zig的[`@cImport`](https://ziglang.org/documentation/master/#cImport)内置是唯一的，因为它接受一个表达式，这个表达式只能接受[`@cInclude`](https://ziglang.org/documentation/master/#cInclude)、[`@cDefine`](https://ziglang.org/documentation/master/#cDefine)和[`@cUndef`](https://ziglang.org/documentation/master/#cUndef)。这类似于translate-c，将C代码在底层翻译成Zig语言。

[`@cInclude`](https://ziglang.org/documentation/master/#cInclude)接受一个路径字符串，并将该路径添加到包含列表中。

[`@cDefine`](https://ziglang.org/documentation/master/#cDefine)和[`@cUndef`](https://ziglang.org/documentation/master/#cUndef)为导入定义和取消定义。

这三个函数的工作方式与你在C代码中期望的完全一样。

与[`@import`](https://ziglang.org/documentation/master/#import)类似，它返回一个带有声明的结构类型。通常建议在一个应用程序中只使用一个[`@cImport`](https://ziglang.org/documentation/master/#cImport)实例，以避免符号冲突；在一个cImport中生成的类型将不等同于在另一个cImport中生成的类型。

cImport仅在链接libc时可用。

# 连接libc

链接libc可以通过命令行通过`-lc`或通过`build.zig`使用`exe.linkLibC();`来完成。使用的libc是编译目标的libc；Zig为许多目标提供libc。

# Zig cc，Zig c++

Zig可执行文件中嵌入了Clang，以及为其他操作系统和体系结构交叉编译所需的库和头文件。

这意味着`zig cc`和`zig c++`不仅可以编译C和C++代码（使用与clang兼容的参数），而且还可以在尊重Zig的目标三重参数的情况下这样做；你安装的单个Zig二进制文件可以针对多个不同的目标进行编译，而无需安装多个版本的编译器或任何插件。使用`zig cc`和`zig c++`还可以利用Zig的缓存系统来加快你的工作流程。

使用Zig，可以很容易地为使用C和/或C++编译器的语言构建一个交叉编译工具链。

野外的一些例子：

- [Using zig cc to cross compile LuaJIT to aarch64-linux from x86_64-linux](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html)

- [Using zig cc and zig c++ in combination with cgo to cross compile hugo from aarch64-macos to x86_64-linux, with full static linking](https://twitter.com/croloris/status/1349861344330330114)

# 第四章结束

本章未完。在未来，它将包含如下内容：
- 从Zig调用C代码，反之亦然
- 使用`zig build`与C和Zig代码的混合

欢迎反馈和PR。
