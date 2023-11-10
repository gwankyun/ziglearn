---
title: "章节一 - 基础"
weight: 2
date: 2023-09-11 18:00:00
description: "章节一 - 这将使你快速掌握Zig编程语言几乎所有的内容。本教程的这一部分应该可以在一个小时内完成。"
---

# 赋值

赋值的语法如下：`(const|var) identifier[: type] = value`。

* `const`表示`identifier`是存储不可变值的**常量**。
* `var`表示标识符是存储**可变值**的变量。
* `: type`是`identifier`的类型注释，如果`value`的数据类型可以推断，则可以省略。

<!--no_test-->
```zig
const constant: i32 = 5;  // 有符号32位常量
var variable: u32 = 5000; // 无符号32位变量

// @as执行显式类型强制转换
const inferred_constant = @as(i32, 5);
var inferred_variable = @as(u32, 5000);
```

常量和变量*必须*有一个值。如果不能给出已知值，则只要提供了类型注释，就可以使用强制转换为任何类型的[`undefined`](https://ziglang.org/documentation/master/#undefined)值。

<!--no_test-->
```zig
const a: i32 = undefined;
var b: u32 = undefined;
```

在可能的情况下，`const`值优先于`var`值。

# 数组

数组用`[N]T`表示，其中`N`是数组中元素的数量，`T`是这些元素的类型（即数组的子类型）。

对于数组字面量，可以用`_`代替`N`来推断数组的大小。

<!--no_test-->
```zig
const a = [5]u8{ 'h', 'e', 'l', 'l', 'o' };
const b = [_]u8{ 'w', 'o', 'r', 'l', 'd' };
```

要获取数组的大小，只需访问数组的`len`字段。

<!--no_test-->
```zig
const array = [_]u8{ 'h', 'e', 'l', 'l', 'o' };
const length = array.len; // 5
```

# If

Zig的if语句只接受`bool`值（即`true`或`false`）。没有类真值或类假值的概念。

在这里，我们将介绍测试。保存下面的代码，并使用`zig test file-name.zig`编译并运行它。我们将使用标准库中的[`expect`](https://ziglang.org/documentation/master/std/#std;testing.expect)函数，如果给出的值为`false`，该函数将导致测试失败。当测试失败时，将显示错误和堆栈跟踪。

```zig
const expect = @import("std").testing.expect;

test "if statement" {
    const a = true;
    var x: u16 = 0;
    if (a) {
        x += 1;
    } else {
        x += 2;
    }
    try expect(x == 1);
}
```

If语句也可以作为表达式使用。

```zig
test "if statement expression" {
    const a = true;
    var x: u16 = 0;
    x += if (a) 1 else 2;
    try expect(x == 1);
}
```

# While

Zig的while循环有三个部分——一个条件、一个块和一个继续表达式。

没有continue表达式。
```zig
test "while" {
    var i: u8 = 2;
    while (i < 100) {
        i *= 2;
    }
    try expect(i == 128);
}
```

带一个continue表达式。
```zig
test "while with continue expression" {
    var sum: u8 = 0;
    var i: u8 = 1;
    while (i <= 10) : (i += 1) {
        sum += i;
    }
    try expect(sum == 55);
}
```

带一个`continue`.

```zig
test "while with continue" {
    var sum: u8 = 0;
    var i: u8 = 0;
    while (i <= 3) : (i += 1) {
        if (i == 2) continue;
        sum += i;
    }
    try expect(sum == 4);
}
```

带一个`break`。

```zig
test "while with break" {
    var sum: u8 = 0;
    var i: u8 = 0;
    while (i <= 3) : (i += 1) {
        if (i == 2) break;
        sum += i;
    }
    try expect(sum == 1);
}
```

# For
For循环用于遍历数组（以及后面将讨论的其他类型）。For循环遵循这种语法。和while一样，for循环也可以使用`break`和`continue`。这里，我们必须给`_`赋值，因为Zig不允许我们使用未使用的值。

```zig
test "for" {
    //字符字面值等同于整数字面值
    const string = [_]u8{ 'a', 'b', 'c' };

    for (string, 0..) |character, index| {
        _ = character;
        _ = index;
    }

    for (string) |character| {
        _ = character;
    }

    for (string, 0..) |_, index| {
        _ = index;
    }

    for (string) |_| {}
}
```

# 函数

__所有函数参数都是不可变的__——如果需要复制，用户必须显式地创建一个。不像变量是snake_case，函数是camelCase。下面是声明和调用一个简单函数的例子。

```zig
fn addFive(x: u32) u32 {
    return x + 5;
}

test "function" {
    const y = addFive(0);
    try expect(@TypeOf(y) == u32);
    try expect(y == 5);
}
```

允许递归：

```zig
fn fibonacci(n: u16) u16 {
    if (n == 0 or n == 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

test "function recursion" {
    const x = fibonacci(10);
    try expect(x == 55);
}
```
当发生递归时，编译器不再能够计算出最大堆栈大小，这可能导致不安全的行为-堆栈溢出。关于如何实现安全递归的细节将在以后讨论。

可以使用`_`而不是变量或const声明来忽略值。这在全局范围内不起作用（即它只在函数和块内部起作用），如果你不需要函数返回的值，可以用它可以来忽略。

<!--no_test-->
```zig
_ = 10;
```

# Defer

Defer用于在退出当前块时执行语句。

```zig
test "defer" {
    var x: i16 = 5;
    {
        defer x += 2;
        try expect(x == 5);
    }
    try expect(x == 7);
}
```

当一个块中有多个defer时，它们以相反的顺序执行。

```zig
test "multi defer" {
    var x: f32 = 5;
    {
        defer x += 2;
        defer x /= 2;
    }
    try expect(x == 4.5);
}
```

# Errors

错误集类似于枚举（稍后详细介绍Zig的枚举），其中集合中的每个错误都是一个值。Zig中没有异常；错误就是值。让我们创建一个错误集。

```zig
const FileOpenError = error{
    AccessDenied,
    OutOfMemory,
    FileNotFound,
};
```
错误集强制转换到它们的超集。

```zig
const AllocationError = error{OutOfMemory};

test "coerce error from a subset to a superset" {
    const err: FileOpenError = AllocationError.OutOfMemory;
    try expect(err == FileOpenError.OutOfMemory);
}
```

一个错误集类型和另一个类型可以用`!`操作符形成错误联合类型。这些类型的值可以是错误值或其他类型的值。

让我们创建一个错误联合类型的值。这里使用[`catch`](https://ziglang.org/documentation/master/#catch)，它后面跟着一个表达式，当它前面的值是错误时计算该表达式。这里的catch用于提供回退值，但也可以是[`noreturn`](https://ziglang.org/documentation/master/#noreturn)——`return`、`while (true)`和其他的类型。

```zig
test "error union" {
    const maybe_error: AllocationError!u16 = 10;
    const no_error = maybe_error catch 0;

    try expect(@TypeOf(no_error) == u16);
    try expect(no_error == 10);
}
```

函数经常返回错误联合。下面是一个使用catch的例子，其中`|err|`语法接收错误的值。这被称为 __有效载荷捕获__，在许多地方都有类似的使用。我们将在本章后面更详细地讨论它。旁注：一些语言对lambda使用类似的语法——但这对Zig来说并非如此。

```zig
fn failingFunction() error{Oops}!void {
    return error.Oops;
}

test "returning an error" {
    failingFunction() catch |err| {
        try expect(err == error.Oops);
        return;
    };
}
```

`try x`是`x catch |err| return err`的快捷方式，通常用于不适合处理错误的地方。Zig的[`try`](https://ziglang.org/documentation/master/#try)和[`catch`](https://ziglang.org/documentation/master/#catch)与其他语言中的try-catch无关。

```zig
fn failFn() error{Oops}!i32 {
    try failingFunction();
    return 12;
}

test "try" {
    var v = failFn() catch |err| {
        try expect(err == error.Oops);
        return;
    };
    try expect(v == 12); // 永远不会到达
}
```

[`errdefer`](https://ziglang.org/documentation/master/#errdefer)的工作方式类似于[`defer`](https://ziglang.org/documentation/master/#defer)，但只有当函数从[`errdefer`](https://ziglang.org/documentation/master/#errdefer)的块中返回错误时才执行。

```zig
var problems: u32 = 98;

fn failFnCounter() error{Oops}!void {
    errdefer problems += 1;
    try failingFunction();
}

test "errdefer" {
    failFnCounter() catch |err| {
        try expect(err == error.Oops);
        try expect(problems == 99);
        return;
    };
}
```

从函数返回的错误联合可以通过没有显式错误集来推断其错误集。这个推断的错误集包含函数可能返回的所有可能的错误。

```zig
fn createFile() !void {
    return error.AccessDenied;
}

test "inferred error set" {
    //类型强制转换成功执行
    const x: error{AccessDenied}!void = createFile();

    //Zig不允许我们通过_ = x忽略错误联合；
    //我们必须用“try”、“catch”或“if”来打开它
    _ = x catch {};
}
```

可以合并错误集。

```zig
const A = error{ NotDir, PathNotFound };
const B = error{ OutOfMemory, PathNotFound };
const C = A || B;
```

`anyerror`是全局错误集，由于它是所有错误集的超集，因此可以将任何集合的错误强制转换到它。一般应该避免使用它。

# Switch

Zig的`switch`既可以作为语句，也可以作为表达式。所有分支的类型必须强制转换为正在切换的类型。所有可能的值都必须有关联的分支——不能遗漏值。情况不能转到其他分支。

switch语句的示例。需要else来满足这个switch的穷尽性。

```zig
test "switch statement" {
    var x: i8 = 10;
    switch (x) {
        -1...1 => {
            x = -x;
        },
        10, 100 => {
            //必须作出特别考虑
            //当除有符号整数
            x = @divExact(x, 10);
        },
        else => {},
    }
    try expect(x == 1);
}
```

这里是前者，但作为一个switch表达式。
```zig
test "switch expression" {
    var x: i8 = 10;
    x = switch (x) {
        -1...1 => -x,
        10, 100 => @divExact(x, 10),
        else => x,
    };
    try expect(x == 1);
}
```

# 运行时安全

Zig提供了一定级别的安全，在执行过程中可能会发现问题。安全开关可以打开，也可以关闭。Zig有许多所谓的 __可检测的非法行为__，这意味着非法行为将被捕获（引起恐慌），但将导致未定义行为的安全关闭。强烈建议用户在开发和测试软件时打开安全开关，尽管这会对速度造成影响。

例如，运行时安全性保护你免受越界索引的影响。

<!--fail_test-->
```zig
test "out of bounds" {
    const a = [3]u8{ 1, 2, 3 };
    var index: u8 = 5;
    const b = a[index];
    _ = b;
}
```
```
test "out of bounds"...index out of bounds
.\tests.zig:43:14: 0x7ff698cc1b82 in test "out of bounds" (test.obj)
    const b = a[index];
             ^
```

用户可以使用内置函数[`@setRuntimeSafety`](https://ziglang.org/documentation/master/#setRuntimeSafety)禁用当前块的运行时安全。

```zig
test "out of bounds, no safety" {
    @setRuntimeSafety(false);
    const a = [3]u8{ 1, 2, 3 };
    var index: u8 = 5;
    const b = a[index];
    _ = b;
}
```

对于某些构建模式（稍后讨论），安全性是关闭的。

# Unreachable

[`unreachable`](https://ziglang.org/documentation/master/#unreachable)是对编译器的一个断言，即无法到达该语句。它可以告诉编译器一个分支是不可能的，然后优化器可以利用它。到达一个[`unreachable`](https://ziglang.org/documentation/master/#unreachable)的地方是明显的非法行为。

由于它是[`noreturn`](https://ziglang.org/documentation/master/#noreturn)类型，因此它与所有其他类型兼容。这里它强制转换到u32。
<!--fail_test-->
```zig
test "unreachable" {
    const x: i32 = 1;
    const y: u32 = if (x == 2) 5 else unreachable;
    _ = y;
}
```
```
test "unreachable"...reached unreachable code
.\tests.zig:211:39: 0x7ff7e29b2049 in test "unreachable" (test.obj)
    const y: u32 = if (x == 2) 5 else unreachable;
                                      ^
```

这里是switch中使用的unreachable。
```zig
fn asciiToUpper(x: u8) u8 {
    return switch (x) {
        'a'...'z' => x + 'A' - 'a',
        'A'...'Z' => x,
        else => unreachable,
    };
}

test "unreachable switch" {
    try expect(asciiToUpper('a') == 'A');
    try expect(asciiToUpper('A') == 'A');
}
```

# 指针

在Zig中，普通指针的值不能为0或null。它们遵循语法`*T`，其中`T`是子类型。

引用语法为`&variable`，解引用语法为`variable.*`。

```zig
fn increment(num: *u8) void {
    num.* += 1;
}

test "pointers" {
    var x: u8 = 1;
    increment(&x);
    try expect(x == 2);
}
```

试图将一个`*T`设置为0是可检测到的非法行为。

<!--fail_test-->
```zig
test "naughty pointer" {
    var x: u16 = 0;
    var y: *u8 = @ptrFromInt(x);
    _ = y;
}
```
```
Test [23/126] test.naughty pointer... thread 21598 panic: cast causes pointer to be null
./test-c1.zig:252:18: 0x260a91 in test.naughty pointer (test)
    var y: *u8 = @ptrFromInt(x);
                 ^
```

Zig也有const指针，不能用来修改引用的数据。引用const变量将产生const指针。

<!--fail_test-->
```zig
test "const pointers" {
    const x: u8 = 1;
    var y = &x;
    y.* += 1;
}
```
```
error: cannot assign to constant
    y.* += 1;
        ^
```

`*T`强制转换为`*const T`。


# 指针大小的整数

`usize`和`isize`分别以无符号整数和有符号整数的形式给出，它们的大小与指针相同。

```zig
test "usize" {
    try expect(@sizeOf(usize) == @sizeOf(*u8));
    try expect(@sizeOf(isize) == @sizeOf(*u8));
}
```

# 多项指针

有时，你可能有一个指向未知数量元素的指针。`[*]T`是这个问题的解决方案，它的工作原理类似于`*T`，但也支持索引语法、指针算术和切片。与`*T`不同，它不能指向未知大小的类型。`*T`强制转换到`[*]T`。

这些指针可以指向任意数量的元素，包括0和1。

# 切片

切片可以被认为是一对`[*]T`（指向数据的指针）和一个`usize`（元素计数）。它们的语法是`[]T`，其中`T`是子类型。在整个Zig中，当你需要对任意数量的数据进行操作时，会大量使用切片。切片具有与指针相同的属性，这意味着也存在const切片。For循环也对切片进行操作。Zig格式的字符串强制转换为`[]const u8`。

这里，语法`x[n..m]`用于从数组中创建切片。这被称为 __切片__，并创建一个从`x[n]`开始到`x[m - 1]`结束的元素切片。本例使用const切片，因为切片点指向的值不需要修改。

```zig
fn total(values: []const u8) usize {
    var sum: usize = 0;
    for (values) |v| sum += v;
    return sum;
}
test "slices" {
    const array = [_]u8{ 1, 2, 3, 4, 5 };
    const slice = array[0..3];
    try expect(total(slice) == 6);
}
```

当这些`n`和`m`值在编译时都是已知的，切片实际上会生成一个指向数组的指针。这不是一个指向数组的指针的问题，即`*[N]T`将强制转换为`[]T`。

```zig
test "slices 2" {
    const array = [_]u8{ 1, 2, 3, 4, 5 };
    const slice = array[0..3];
    try expect(@TypeOf(slice) == *const [3]u8);
}
```

语法`x[n..]`也可以用在你想切到最后的时候。

```zig
test "slices 3" {
    var array = [_]u8{ 1, 2, 3, 4, 5 };
    var slice = array[0..];
    _ = slice;
}
```

可以切片的类型有数组、指针和切片。

# 枚举

Zig的枚举允许你使用一组受限的命名值来定义类型。

让我们声明一个枚举。
```zig
const Direction = enum { north, south, east, west };
```

枚举类型可以有指定的（整数）标记类型。
```zig
const Value = enum(u2) { zero, one, two };
```

Enum的序数从0开始。它们可以通过内置函数[`@intFromEnum`](https://ziglang.org/documentation/master/#intFromEnum)访问。
```zig
test "enum ordinal value" {
    try expect(@intFromEnum(Value.zero) == 0);
    try expect(@intFromEnum(Value.one) == 1);
    try expect(@intFromEnum(Value.two) == 2);
}
```

值可以被覆盖，下一个值从那里继续。
```zig
const Value2 = enum(u32) {
    hundred = 100,
    thousand = 1000,
    million = 1000000,
    next,
};

test "set enum ordinal value" {
    try expect(@intFromEnum(Value2.hundred) == 100);
    try expect(@intFromEnum(Value2.thousand) == 1000);
    try expect(@intFromEnum(Value2.million) == 1000000);
    try expect(@intFromEnum(Value2.next) == 1000001);
}
```

可以将方法赋给枚举。它们充当可以用点语法调用的命名空间函数。

```zig
const Suit = enum {
    clubs,
    spades,
    diamonds,
    hearts,
    pub fn isClubs(self: Suit) bool {
        return self == Suit.clubs;
    }
};

test "enum method" {
    try expect(Suit.spades.isClubs() == Suit.isClubs(.spades));
}
```

枚举也可以被赋予`var`和`const`声明。它们充当具有名称空间的全局变量，它们的值与枚举类型的实例无关且不附加。

```zig
const Mode = enum {
    var count: u32 = 0;
    on,
    off,
};

test "hmm" {
    Mode.count += 1;
    try expect(Mode.count == 1);
}
```


# 结构体

结构体是Zig最常见的复合数据类型，允许你定义可以存储固定的命名对象集的类型字段。Zig不保证结构体中字段的内存顺序或其大小。和数组一样，结构体也很简洁用`T{}`语法构造。下面是一个声明和填充结构体的例子。
```zig
const Vec3 = struct { x: f32, y: f32, z: f32 };

test "struct usage" {
    const my_vector = Vec3{
        .x = 0,
        .y = 100,
        .z = 50,
    };
    _ = my_vector;
}
```

所有字段都必须给定一个值：

<!--fail_test-->
```zig
test "missing struct field" {
    const my_vector = Vec3{
        .x = 0,
        .z = 50,
    };
    _ = my_vector;
}
```
```
error: missing field: 'y'
    const my_vector = Vec3{
                        ^
```

字段可以给出默认值：
```zig
const Vec4 = struct { x: f32, y: f32, z: f32 = 0, w: f32 = undefined };

test "struct defaults" {
    const my_vector = Vec4{
        .x = 25,
        .y = -50,
    };
    _ = my_vector;
}
```

和枚举一样，结构体也可以包含函数和声明。

结构体有一个独特的属性，当给定一个指向结构体的指针时，将自动执行一层解引用访问字段。注意，在这个例子中，在swap函数中访问self.x和self.y不需要解引用self指针。

```zig
const Stuff = struct {
    x: i32,
    y: i32,
    fn swap(self: *Stuff) void {
        const tmp = self.x;
        self.x = self.y;
        self.y = tmp;
    }
};

test "automatic dereference" {
    var thing = Stuff{ .x = 10, .y = 20 };
    thing.swap();
    try expect(thing.x == 20);
    try expect(thing.y == 10);
}
```

# 联合

Zig的联合允许你定义存储许多可能类型字段的一个值的类型；一次只能有一个字段可用。

裸联合类型没有保证的内存布局。因此，裸联合不能用于重新解释内存。访问union中未激活的字段是可检测到的非法行为。

<!--fail_test-->
```zig
const Result = union {
    int: i64,
    float: f64,
    bool: bool,
};

test "simple union" {
    var result = Result{ .int = 1234 };
    result.float = 12.34;
}
```
```
test "simple union"...access of inactive union field
.\tests.zig:342:12: 0x7ff62c89244a in test "simple union" (test.obj)
    result.float = 12.34;
           ^
```

标记联合是使用枚举来检测哪个字段是活动的联合。这里我们再次利用有效载荷捕获来切换获取联合的标记类型，同时捕获其包含的值。这里使用*指针捕获*；捕获的值是不可变，但是使用`|*value|`语法，我们可以捕获指向值的指针，而不是值本身。这使我们能够使用解引用来改变原始值。

```zig
const Tag = enum { a, b, c };

const Tagged = union(Tag) { a: u8, b: f32, c: bool };

test "switch on tagged union" {
    var value = Tagged{ .b = 1.5 };
    switch (value) {
        .a => |*byte| byte.* += 1,
        .b => |*float| float.* *= 2,
        .c => |*b| b.* = !b.*,
    }
    try expect(value.b == 3);
}
```

还可以推断标记联合的标记类型。这相当于上面的Tagged类型。

<!--no_test-->
```zig
const Tagged = union(enum) { a: u8, b: f32, c: bool };
```

`void`成员类型的类型可以从语法中省略。这里，没有`void`类型的。

```zig
const Tagged2 = union(enum) { a: u8, b: f32, c: bool, none };
```

# 整数规则

Zig支持十六进制、八进制和二进制整数字面值。
```zig
const decimal_int: i32 = 98222;
const hex_int: u8 = 0xff;
const another_hex_int: u8 = 0xFF;
const octal_int: u16 = 0o755;
const binary_int: u8 = 0b11110000;
```
下划线也可以放在数字之间作为视觉分隔符。
```zig
const one_billion: u64 = 1_000_000_000;
const binary_mask: u64 = 0b1_1111_1111;
const permissions: u64 = 0o7_5_5;
const big_address: u64 = 0xFF80_0000_0000_0000;
```

允许“整数扩大”，这意味着一种类型的整数可以强制转换为另一种类型的整数，前提是新类型可以容纳旧类型所能容纳的所有值。

```zig
test "integer widening" {
    const a: u8 = 250;
    const b: u16 = a;
    const c: u32 = b;
    try expect(c == a);
}
```

如果存储在整型中的值不能强制转换为所需的类型，则可以使用[`@intCast`](https://ziglang.org/documentation/master/#intCast)显式地从一种类型转换为另一种类型。如果给定的值超出目标类型的范围，这是可检测到的非法行为。

```zig
test "@intCast" {
    const x: u64 = 200;
    const y = @as(u8, @intCast(x));
    try expect(@TypeOf(y) == u8);
}
```

默认情况下，整数不允许溢出。溢出是可察觉的非法行为。有时，能够以良好定义的方式溢出整数是需要的行为。对于这个用例，Zig提供了溢出操作符。

| 一般操作符       | 溢出操作符          |
|-----------------|-------------------|
| +               | +%                |
| -               | -%                |
| *               | *%                |
| +=              | +%=               |
| -=              | -%=               |
| *=              | *%=               |

```zig
test "well defined overflow" {
    var a: u8 = 255;
    a +%= 1;
    try expect(a == 0);
}
```

# 浮点数

Zig的浮点数是严格符合IEEE的，除非使用[`@setFloatMode(.Optimized)`](https://ziglang.org/documentation/master/#setFloatMode)，这相当于GCC的`-ffast-math`。float强制转换为更大的float类型

```zig
test "float widening" {
    const a: f16 = 0;
    const b: f32 = a;
    const c: f128 = b;
    try expect(c == @as(f128, a));
}
```

浮点支持多种字面量。
```zig
const floating_point: f64 = 123.0E+77;
const another_float: f64 = 123.0;
const yet_another: f64 = 123.0e+77;

const hex_floating_point: f64 = 0x103.70p-5;
const another_hex_float: f64 = 0x103.70;
const yet_another_hex_float: f64 = 0x103.70P-5;
```
下划线也可以放在数字之间。
```zig
const lightspeed: f64 = 299_792_458.000_000;
const nanosecond: f64 = 0.000_000_001;
const more_hex: f64 = 0x1234_5678.9ABC_CDEFp-10;
```

整数和浮点可以使用内置函数[`@floatFromInt`](https://ziglang.org/documentation/0.11.0/#floatFromInt)和[`@intFromFloat`](https://ziglang.org/documentation/0.11.0/#intFromFloat)进行转换。[`@floatFromInt`](https://ziglang.org/documentation/0.11.0/#floatFromInt)总是安全的，而[`@intFromFloat`](https://ziglang.org/documentation/0.11.0/#intFromFloat)是可检测的非法行为，如果浮点值不适合整型目标类型。

```zig
test "int-float conversion" {
    const a: i32 = 0;
    const b = @as(f32, @floatFromInt(a));
    const c = @as(i32, @intFromFloat(b));
    try expect(c == a);
}
```

# 带标签的块

Zig中的块是表达式，可以给出用于生成值的标签。这里，我们使用一个名为`blk`的标签。块生成值，这意味着它们可以用来代替值。空块`{}`的值是`void`类型的值。

```zig
test "labelled blocks" {
    const count = blk: {
        var sum: u32 = 0;
        var i: u32 = 0;
        while (i < 10) : (i += 1) sum += i;
        break :blk sum;
    };
    try expect(count == 45);
    try expect(@TypeOf(count) == u32);
}
```

这可以看作是相当于C的`i++`。
<!--no_test-->
```zig
blk: {
    const tmp = i;
    i += 1;
    break :blk tmp;
}
```

# 带标签的循环

循环可以被赋予标签，允许你`break`并`continue`进行外部循环。

```zig
test "nested continue" {
    var count: usize = 0;
    outer: for ([_]i32{ 1, 2, 3, 4, 5, 6, 7, 8 }) |_| {
        for ([_]i32{ 1, 2, 3, 4, 5 }) |_| {
            count += 1;
            continue :outer;
        }
    }
    try expect(count == 8);
}
```

# 循环作为表达式

和`return`一样，`break`也接受一个值。这可用于从循环中产生一个值。Zig中的循环也有一个`else`分支，该分支在没有以`break`退出循环时进行计算。

```zig
fn rangeHasNumber(begin: usize, end: usize, number: usize) bool {
    var i = begin;
    return while (i < end) : (i += 1) {
        if (i == number) {
            break true;
        }
    } else false;
}

test "while loop expression" {
    try expect(rangeHasNumber(0, 10, 3));
}
```

# 可选项

可选项使用语法`?T`，用于存储数据[`null`](https://ziglang.org/documentation/master/#null)或类型为`T`的值。

```zig
test "optional" {
    var found_index: ?usize = null;
    const data = [_]i32{ 1, 2, 3, 4, 5, 6, 7, 8, 12 };
    for (data, 0..) |v, i| {
        if (v == 10) found_index = i;
    }
    try expect(found_index == null);
}
```

可选项支持`orelse`表达式，该表达式在可选项为[`null`](https://ziglang.org/documentation/master/#null)时起作用。这将可选对象展开为它的子类型。

```zig
test "orelse" {
    var a: ?f32 = null;
    var b = a orelse 0;
    try expect(b == 0);
    try expect(@TypeOf(b) == f32);
}
```

`.?`是`orelse unreachable`的简写。当你知道可选值不可能为空时，使用此方法来打开[`null`](https://ziglang.org/documentation/master/#null)值是可检测到的非法行为。

```zig
test "orelse unreachable" {
    const a: ?f32 = 5;
    const b = a orelse unreachable;
    const c = a.?;
    try expect(b == c);
    try expect(@TypeOf(c) == f32);
}
```

有效载荷捕获在很多地方都适用于可选项，这意味着在它是非空的情况下，我们可以“捕获”它的非空值。

这里我们使用一个`if`负载捕获；a和b在这里是等价的。`if (b) |value|`捕获`b`的值（在`b`不为空的情况下），并使其作为`value`可用。与union示例一样，捕获的值是不可变的，但仍然可以使用指针捕获来修改存储在`b`中的值。

```zig
test "if optional payload capture" {
    const a: ?i32 = 5;
    if (a != null) {
        const value = a.?;
        _ = value;
    }

    var b: ?i32 = 5;
    if (b) |*value| {
        value.* += 1;
    }
    try expect(b.? == 6);
}
```

和`while`一起用时：
```zig
var numbers_left: u32 = 4;
fn eventuallyNullSequence() ?u32 {
    if (numbers_left == 0) return null;
    numbers_left -= 1;
    return numbers_left;
}

test "while null capture" {
    var sum: u32 = 0;
    while (eventuallyNullSequence()) |value| {
        sum += value;
    }
    try expect(sum == 6); // 3 + 2 + 1
}
```

与非可选类型相比，可选指针和可选切片类型不会占用任何额外的内存。这是因为它们在内部使用指针的0值来表示`null`。

这就是Zig中的空指针的工作方式——在解引用之前，它们必须被解包装为非可选的，这可以防止空指针解引用意外发生。

# Comptime

代码块可以在编译时使用[`comptime`](https://ziglang.org/documentation/master/#comptime)关键字强制执行。在这个例子中，变量x和y是等价的。

```zig
test "comptime blocks" {
    var x = comptime fibonacci(10);
    _ = x;

    var y = comptime blk: {
        break :blk fibonacci(10);
    };
    _ = y;
}
```

整型字面值为`comptime_int`类型。它们的特殊之处在于它们没有大小（它们不能在运行时使用！），并且它们具有任意精度。`comptime_int`值强制转换为可以容纳它们的任何整数类型。它们还强制浮动。字符字面量就是这种类型。

```zig
test "comptime_int" {
    const a = 12;
    const b = a + 10;

    const c: u4 = a;
    _ = c;
    const d: f32 = b;
    _ = d;
}
```

`comptime_float`也是可用的，它在内部是一个`f128`。不能将它们强制转换为整数，即使它们保存的是整数值。

Zig中的类型是`type`类型的值。这些在编译时可用。我们以前通过检查[`@TypeOf`](https://ziglang.org/documentation/master/#TypeOf)和与其他类型比较遇到过它们，但是我们可以做更多。

```zig
test "branching on types" {
    const a = 5;
    const b: if (a < 10) f32 else i32 = 5;
    _ = b;
}
```

可以将Zig中的函数参数标记为[`comptime`](https://ziglang.org/documentation/master/#comptime)。这意味着传递给该函数形参的值必须在编译时已知。让我们创建一个返回类型的函数。注意这个函数是如何使用PascalCase的，因为它返回一个类型。

```zig
fn Matrix(
    comptime T: type,
    comptime width: comptime_int,
    comptime height: comptime_int,
) type {
    return [height][width]T;
}

test "returning a type" {
    try expect(Matrix(f32, 4, 4) == [4][4]f32);
}
```

我们可以使用内置的[`@typeInfo`](https://ziglang.org/documentation/master/#typeInfo)来反射类型，它接受一个`type`并返回一个带标签的联合。这个带标签的联合类型可以在[`std.builtin.TypeInfo`](https://ziglang.org/documentation/master/std/#std;builtin.TypeInfo)中找到（稍后会提供如何使用导入和std的信息）。

```zig
fn addSmallInts(comptime T: type, a: T, b: T) T {
    return switch (@typeInfo(T)) {
        .ComptimeInt => a + b,
        .Int => |info| if (info.bits <= 16)
            a + b
        else
            @compileError("ints too large"),
        else => @compileError("only ints accepted"),
    };
}

test "typeinfo switch" {
    const x = addSmallInts(u16, 20, 30);
    try expect(@TypeOf(x) == u16);
    try expect(x == 50);
}
```

我们可以使用[`@Type`](https://ziglang.org/documentation/master/#Type)函数从[`@typeInfo`](https://ziglang.org/documentation/master/#typeInfo)创建类型。[`@Type`](https://ziglang.org/documentation/master/#Type)对大多数类型都实现了，但对枚举、联合、函数和结构体没有实现。

这里匿名结构语法与`.{}`一起使用，因为`T{}`中的`T`可以被推断出来。稍后将详细介绍匿名结构。在这个例子中，如果没有设置`Int`标记，我们将得到一个编译错误。

```zig
fn GetBiggerInt(comptime T: type) type {
    return @Type(.{
        .Int = .{
            .bits = @typeInfo(T).Int.bits + 1,
            .signedness = @typeInfo(T).Int.signedness,
        },
    });
}

test "@Type" {
    try expect(GetBiggerInt(u8) == u9);
    try expect(GetBiggerInt(i31) == i32);
}
```

返回结构类型是在Zig中创建泛型数据结构的方法。这里需要使用[`@This`](https://ziglang.org/documentation/master/#This)，它获取最内层结构体、联合或枚举的类型。这里还使用了[`std.mem.eql`](https://ziglang.org/documentation/master/std/#std;mem.eql)来比较两个片。

```zig
fn Vec(
    comptime count: comptime_int,
    comptime T: type,
) type {
    return struct {
        data: [count]T,
        const Self = @This();

        fn abs(self: Self) Self {
            var tmp = Self{ .data = undefined };
            for (self.data, 0..) |elem, i| {
                tmp.data[i] = if (elem < 0)
                    -elem
                else
                    elem;
            }
            return tmp;
        }

        fn init(data: [count]T) Self {
            return Self{ .data = data };
        }
    };
}

const eql = @import("std").mem.eql;

test "generic vector" {
    const x = Vec(3, f32).init([_]f32{ 10, -10, 5 });
    const y = x.abs();
    try expect(eql(f32, &y.data, &[_]f32{ 10, 10, 5 }));
}
```

也可以通过使用`anytype`代替类型来推断函数参数的类型。然后可以在参数上使用[`@TypeOf`](https://ziglang.org/documentation/master/#TypeOf)。

```zig
fn plusOne(x: anytype) @TypeOf(x) {
    return x + 1;
}

test "inferred function parameter" {
    try expect(plusOne(@as(u32, 1)) == 2);
}
```

Comptime还引入了操作符`++`和`**`，用于连接和重复数组和切片。这些操作符在运行时不起作用。

```zig
test "++" {
    const x: [4]u8 = undefined;
    const y = x[0..];

    const a: [6]u8 = undefined;
    const b = a[0..];

    const new = y ++ b;
    try expect(new.len == 10);
}

test "**" {
    const pattern = [_]u8{ 0xCC, 0xAA };
    const memory = pattern ** 3;
    try expect(eql(u8, &memory, &[_]u8{ 0xCC, 0xAA, 0xCC, 0xAA, 0xCC, 0xAA }));
}
```

# 有效载荷捕获

有效负载捕获使用语法`|value|`，出现在许多地方，其中一些我们已经看到了。无论它们出现在哪里，它们都是用来“获取”某物的价值。

if语句和可选项中使用。
```zig
test "optional-if" {
    var maybe_num: ?usize = 10;
    if (maybe_num) |n| {
        try expect(@TypeOf(n) == usize);
        try expect(n == 10);
    } else {
        unreachable;
    }
}
```

if语句和错误联合中使用。这里需要带有错误捕获的else。
```zig
test "error union if" {
    var ent_num: error{UnknownEntity}!u32 = 5;
    if (ent_num) |entity| {
        try expect(@TypeOf(entity) == u32);
        try expect(entity == 5);
    } else |err| {
        _ = err catch {};
        unreachable;
    }
}
```

使用while循环和可选项。这可能有一个else块。
```zig
test "while optional" {
    var i: ?u32 = 10;
    while (i) |num| : (i.? -= 1) {
        try expect(@TypeOf(num) == u32);
        if (num == 1) {
            i = null;
            break;
        }
    }
    try expect(i == null);
}
```

在while循环和错误联合中使用。这里需要带有错误捕获的else。

```zig
var numbers_left2: u32 = undefined;

fn eventuallyErrorSequence() !u32 {
    return if (numbers_left2 == 0) error.ReachedZero else blk: {
        numbers_left2 -= 1;
        break :blk numbers_left2;
    };
}

test "while error union capture" {
    var sum: u32 = 0;
    numbers_left2 = 3;
    while (eventuallyErrorSequence()) |value| {
        sum += value;
    } else |err| {
        try expect(err == error.ReachedZero);
    }
}
```

For循环。
```zig
test "for capture" {
    const x = [_]i8{ 1, 5, 120, -5 };
    for (x) |v| try expect(@TypeOf(v) == i8);
}
```

带标签的联合的switch case。
```zig
const Info = union(enum) {
    a: u32,
    b: []const u8,
    c,
    d: u32,
};

test "switch capture" {
    var b = Info{ .a = 10 };
    const x = switch (b) {
        .b => |str| blk: {
            try expect(@TypeOf(str) == []const u8);
            break :blk 1;
        },
        .c => 2,
        //如果它们是相同类型的，它们
        //可以在同一个捕获组中
        .a, .d => |num| blk: {
            try expect(@TypeOf(num) == u32);
            break :blk num * 2;
        },
    };
    try expect(x == 20);
}
```

正如我们在上面的联合和可选项小节中看到的，用`|val|`语法捕获的值是不可变的（类似于函数参数），但是我们可以使用指针捕获来修改原始值。这捕获的值作为指针本身仍然是不可变的，但因为值现在是一个指针，我们可以通过解引用来修改原始值:

```zig
test "for with pointer capture" {
    var data = [_]u8{ 1, 2, 3 };
    for (&data) |*byte| byte.* += 1;
    try expect(eql(u8, &data, &[_]u8{ 2, 3, 4 }));
}
```

# 内嵌循环

`inline`循环是展开的，允许一些只在编译时起作用的事情发生。这里我们用[`for`](https://ziglang.org/documentation/master/#inline-for)，但[`while`](https://ziglang.org/documentation/master/#inline-while)的用法类似。
```zig
test "inline for" {
    const types = [_]type{ i32, f32, u8, bool };
    var sum: usize = 0;
    inline for (types) |T| sum += @sizeOf(T);
    try expect(sum == 10);
}
```

出于性能原因使用这些是不明智的，除非你已经测试过显式展开更快；编译器往往比你做出更好的决定。

# 不透明的

在Zig中，[`opaque`](https://ziglang.org/documentation/master/#opaque)类型的大小和对齐方式是未知的（尽管不是零）。因此，这些数据类型不能直接存储。它们用于使用指向我们没有信息的类型的指针来维护类型安全。

<!--fail_test-->
```zig
const Window = opaque {};
const Button = opaque {};

extern fn show_window(*Window) callconv(.C) void;

test "opaque" {
    var main_window: *Window = undefined;
    show_window(main_window);

    var ok_button: *Button = undefined;
    show_window(ok_button);
}
```
```
./test-c1.zig:653:17: error: expected type '*Window', found '*Button'
    show_window(ok_button);
                ^
./test-c1.zig:653:17: note: pointer type child 'Button' cannot cast into pointer type child 'Window'
    show_window(ok_button);
                ^
```

不透明类型可以在其定义中进行声明（与结构体、枚举和联合类型相同）。

<!--no_test-->
```zig
const Window = opaque {
    fn show(self: *Window) void {
        show_window(self);
    }
};

extern fn show_window(*Window) callconv(.C) void;

test "opaque with declarations" {
    var main_window: *Window = undefined;
    main_window.show();
}
```

不透明的典型用例是在与不公开完整类型信息的C代码互操作时维护类型安全。

# 匿名结构体

结构字面值可以省略结构类型。这些字面值可以强制转换为其他结构类型。

```zig
test "anonymous struct literal" {
    const Point = struct { x: i32, y: i32 };

    var pt: Point = .{
        .x = 13,
        .y = 67,
    };
    try expect(pt.x == 13);
    try expect(pt.y == 67);
}
```

匿名结构可以是完全匿名的，即没有被强制转换到另一个结构类型。

```zig
test "fully anonymous struct" {
    try dump(.{
        .int = @as(u32, 1234),
        .float = @as(f64, 12.34),
        .b = true,
        .s = "hi",
    });
}

fn dump(args: anytype) !void {
    try expect(args.int == 1234);
    try expect(args.float == 12.34);
    try expect(args.b);
    try expect(args.s[0] == 'h');
    try expect(args.s[1] == 'i');
}
```
<!-- TODO: mention tuple slicing when it's implemented -->

可以创建没有字段名的匿名结构，并将其称为 __元组__。它们具有数组所具有的许多属性；元组可以被迭代，索引，可以与`++`和`**`操作符一起使用，并且有一个len字段。在内部，它们有以`"0"`开头的编号字段名，可以使用特殊语法`@"0"`访问，该语法充当语法的转义——`@""`内部的内容始终被视为标识符。

这里必须使用`inline`循环来遍历元组，因为每个元组字段的类型可能不同。

```zig
test "tuple" {
    const values = .{
        @as(u32, 1234),
        @as(f64, 12.34),
        true,
        "hi",
    } ++ .{false} ** 2;
    try expect(values[0] == 1234);
    try expect(values[4] == false);
    inline for (values, 0..) |v, i| {
        if (i != 2) continue;
        try expect(v);
    }
    try expect(values.len == 6);
    try expect(values.@"3"[0] == 'h');
}
```

# 哨兵终止

数组、切片和许多指针都可以用其子类型的值结束。这就是所谓的哨点终止。它们遵循语法`[N:t]T`、`[:t]T`和`[*:t]T`，其中`t`是子类型`T`的值。

一个哨兵终止数组的例子。内置的[`@ptrCast`](https://ziglang.org/documentation/master/#ptrCast)用于执行不安全的类型转换。这向我们展示了数组的最后一个元素后面跟着一个0字节。

```zig
test "sentinel termination" {
    const terminated = [3:0]u8{ 3, 2, 1 };
    try expect(terminated.len == 3);
    try expect(@as(*const [4]u8, @ptrCast(&terminated))[3] == 0);
}
```

字符串字面值的类型是`*const [N:0]u8`，其中N是字符串的长度。这允许字符串字面值强制使用前哨终止的切片，以及前哨终止的许多指针。注意：字符串文字是UTF-8编码的。

```zig
test "string literal" {
    try expect(@TypeOf("hello") == *const [5:0]u8);
}
```

`[*:0]u8`和`[*:0]const u8`完美地模拟了C的字符串。

```zig
test "C string" {
    const c_string: [*:0]const u8 = "hello";
    var array: [5]u8 = undefined;

    var i: usize = 0;
    while (c_string[i] != 0) : (i += 1) {
        array[i] = c_string[i];
    }
}
```

哨兵终止类型强制转换为非哨兵终止类型。

```zig
test "coercion" {
    var a: [*:0]u8 = undefined;
    const b: [*]u8 = a;
    _ = b;

    var c: [5:0]u8 = undefined;
    const d: [5]u8 = c;
    _ = d;

    var e: [:0]f32 = undefined;
    const f: []f32 = e;
    _ = f;
}
```

提供了哨兵终止切片，可用于使用语法`x[n..m:t]`创建哨兵终止切片，其中`t`为终止值。这样做是一个断言，从程序员，内存终止在它应该如此——得到这个错误是可检测的非法行为。

```zig
test "sentinel terminated slicing" {
    var x = [_:0]u8{255} ** 3;
    const y = x[0..3 :0];
    _ = y;
}
```

# 向量

Zig为SIMD提供了向量类型。从数学意义上讲，它们不能与vector或像C++的std::vector这样的vector（关于这一点，请参阅第2章的“Arraylist”）混淆。向量可以使用我们之前使用的内置[`@Type`](https://ziglang.org/documentation/master/#Type)创建，[`std.meta.Vector`](https://ziglang.org/documentation/master/std/#std;meta.Vector)也提供了一种简写。

向量只能有布尔值、整数、浮点数和指针的子类型。

Operations between vectors with the same child type and length can take place. These operations are performed on each of the values in the vector.[`std.meta.eql`](https://ziglang.org/documentation/master/std/#std;meta.eql) is used here to check for equality between two vectors (also useful for other types like structs).

具有相同子类型和长度的向量之间可以进行操作。这些操作是对向量中的每个值执行的。这里使用[`std.meta.eql`](https://ziglang.org/documentation/master/std/#std;meta.eql)来检查两个向量之间是否相等（对于结构体等其他类型也很有用）。

```zig
const meta = @import("std").meta;

test "vector add" {
    const x: @Vector(4, f32) = .{ 1, -10, 20, -1 };
    const y: @Vector(4, f32) = .{ 2, 10, 0, 1 };
    const z = x + y;
    try expect(meta.eql(z, @Vector(4, f32){ 3, 0, 20, 0 }));
}
```

向量是可索引的。
```zig
test "vector indexing" {
    const x: @Vector(4, u8) = .{ 255, 0, 255, 0 };
    try expect(x[0] == 255);
}
```

内置函数[`@splat`](https://ziglang.org/documentation/master/#splat)可以用来构造一个所有值都相同的向量。这里我们用它来用一个向量乘以一个标量。

```zig
test "vector * scalar" {
    const x: @Vector(3, f32) = .{ 12.5, 37.5, 2.5 };
    const y = x * @as(@Vector(3, f32), @splat(2));
    try expect(meta.eql(y, @Vector(3, f32){ 25, 75, 5 }));
}
```

向量不像数组那样有`len`字段，但仍然可以循环。

```zig
test "vector looping" {
    const x = @Vector(4, u8){ 255, 0, 255, 0 };
    var sum = blk: {
        var tmp: u10 = 0;
        var i: u8 = 0;
        while (i < 4) : (i += 1) tmp += x[i];
        break :blk tmp;
    };
    try expect(sum == 510);
}
```

向量强制转换到它们各自的数组。

```zig
const arr: [4]f32 = @Vector(4, f32){ 1, 2, 3, 4 };
```

值得注意的是，如果你没有做出正确的决定，使用显式向量可能会导致软件速度变慢——编译器的自动矢量化是相当聪明的。

# Imports

内置函数[`@import`](https://ziglang.org/documentation/master/#import)接受一个文件，并基于该文件为你提供一个结构类型。所有标记为`pub`（表示public）的声明都将在此结构类型中结束，以备使用。

`@import("std")`是编译器中的一种特殊情况，它允许你访问标准库。其他的[`@import`](https://ziglang.org/documentation/master/#import)会接受一个文件路径，或者一个包名（后面的章节会有更多关于包的内容）。

我们将在后面的章节中探索更多的标准库。

# 第一章结束
在下一章中，我们将介绍标准模式，包括标准库中许多有用的领域。

欢迎反馈和PR。
