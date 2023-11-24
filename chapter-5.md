---
title: "第五章 - 异步"
weight: 6
date: 2023-04-28 18:00:00
description: "第五章 - 了解zig语言的异步是如何工作的"
---

警告：Zig编译器的异步支持在自托管时回归，并且在0.11中不可用。这是由于0.12的变化。

# 异步

要有效地理解Zig的异步，需要熟悉调用堆栈的概念。如果你以前没有听说过这个，请[查看维基百科页面](https://en.wikipedia.org/wiki/Call_stack)。

<!-- TODO: actually explain the call stack? -->

传统的函数调用由三部分组成：
1. 用它的参数初始化被调用的函数，推入函数的堆栈帧
2. 将控制传递给函数
3. 在函数完成后，将控制权交还给调用者，检索函数的返回值并弹出函数的堆栈帧

使用Zig的异步函数，我们可以做得更多，因为控制权的转移是一个正在进行的双向对话（即，我们可以多次将控制权交给函数并将其收回）。因此，在异步上下文中调用函数时必须特别考虑；我们不能再像往常一样推送和弹出堆栈帧（因为堆栈是易失的，当前堆栈帧“上面”的东西可能会被覆盖），而是显式地存储异步函数的帧。虽然大多数人不会使用它的完整功能集，但这种风格的异步对于创建更强大的结构（如事件循环）很有用。

Zig的异步风格可以被描述为可挂起的无堆栈协程。Zig的异步与有堆栈的操作系统线程非常不同，并且只能由内核挂起。此外，Zig的async为你提供控制流结构和代码生成；异步并不意味着并行性或线程的使用。

# 暂停（Suspend）/恢复（Resume）

在上一节中，我们讨论了async函数如何将控制权交还给调用者，以及async函数如何在稍后将控制权收回。该功能由关键字[`suspend`和`resume`](https://ziglang.org/documentation/master/#Suspend-and-Resume)提供。当函数挂起时，控制流返回到上次恢复的位置；当通过`async`调用调用函数时，这是隐式恢复。

这些示例中的注释指示了执行的顺序。这里有几点需要注意：
*  `async`关键字用于在异步上下文中调用函数。
*  `async func()`返回函数的帧。
*  我们必须存储这个帧。
*  `resume`关键字用于帧，而`suspend`则用于被调用的函数。

这个例子有一个挂起，但没有匹配的恢复。
```zig
const expect = @import("std").testing.expect;

var foo: i32 = 1;

test "suspend with no resume" {
    var frame = async func(); //1
    _ = frame;
    try expect(foo == 2);     //4
}

fn func() void {
    foo += 1;                 //2
    suspend {}                //3
    foo += 1;                 //永不可达！
}
```

在格式良好的代码中，每个挂起都与一个恢复相匹配。

```zig
var bar: i32 = 1;

test "suspend with resume" {
    var frame = async func2();  //1
    resume frame;               //4
    try expect(bar == 3);       //6
}

fn func2() void {
    bar += 1;                   //2
    suspend {}                  //3
    bar += 1;                   //5
}
```

# Async / Await

与格式良好的代码对每个resume都有一个suspend类似，每个具有返回值的`async`函数调用都必须与`await`匹配。`await`在异步帧上产生的值对应于函数的返回值。

你可能会注意到这里的`func3`是一个普通函数（也就是说，它没有挂起点——它不是一个async函数）。尽管如此，在异步调用中调用`func3`可以作为异步函数；`func3`的调用约定不必更改为async - `func3`可以是任何调用约定。

```zig
fn func3() u32 {
    return 5;
}

test "async / await" {
    var frame = async func3();
    try expect(await frame == 5);
}
```

对可能挂起的函数的异步帧使用`await`只能在异步函数中使用。因此，在异步函数的框架上使用`await`的函数也被认为是异步函数。如果你可以确定潜在的挂起不会发生，`nosuspend await`将阻止它发生。

# Nosuspend

当调用一个被确定为异步（即它可能挂起）的函数而没有`async`调用时，调用它的函数也被视为异步。当确定具有具体（非异步）调用约定的函数具有挂起点时，这是一个编译错误，因为异步需要自己的调用约定。这意味着，例如，main不能是异步的。

<!--no_test-->
```zig
pub fn main() !void {
    suspend {}
}
```
（从windows编译）
```
C:\zig\lib\zig\std\start.zig:165:1: error: function with calling convention 'Stdcall' cannot be async
fn WinStartup() callconv(.Stdcall) noreturn {
^
C:\zig\lib\zig\std\start.zig:173:65: note: async function call here
    std.os.windows.kernel32.ExitProcess(initEventLoopAndCallMain());
                                                                ^
C:\zig\lib\zig\std\start.zig:276:12: note: async function call here
    return @call(.{ .modifier = .always_inline }, callMain, .{});
           ^
C:\zig\lib\zig\std\start.zig:334:37: note: async function call here
            const result = root.main() catch |err| {
                                    ^
.\main.zig:12:5: note: suspends here
    suspend {}
    ^
```

如果你希望在不使用`async`调用的情况下调用异步函数，并且该函数的调用方也不是异步的，那么`nosuspend`关键字就派上了用场。通过断言潜在的挂起不会发生，这允许async函数的调用者也不是异步的。

<!--no_test-->
```zig
const std = @import("std");

fn doTicksDuration(ticker: *u32) i64 {
    const start = std.time.milliTimestamp();

    while (ticker.* > 0) {
        suspend {}
        ticker.* -= 1;
    }

    return std.time.milliTimestamp() - start;
}

pub fn main() !void {
    var ticker: u32 = 0;
    const duration = nosuspend doTicksDuration(&ticker);
}
```

在上面的代码中，如果我们将`ticker`的值更改为0以上，这是可检测到的非法行为。如果我们运行该代码，在安全构建模式下就会出现这样的错误。与Zig中的其他非法行为类似，在不安全模式下发生这些行为将导致未定义行为。

```
async function called in nosuspend scope suspended
.\main.zig:16:47: 0x7ff661dd3414 in main (main.obj)
    const duration = nosuspend doTicksDuration(&ticker);
                                              ^
C:\zig\lib\zig\std\start.zig:173:65: 0x7ff661dd18ce in std.start.WinStartup (main.obj)
    std.os.windows.kernel32.ExitProcess(initEventLoopAndCallMain());
                                                                ^
```

# 异步帧、挂起块

`@Frame(function)`返回函数的帧类型。这适用于异步函数和没有特定调用约定的函数。

```zig
fn add(a: i32, b: i32) i64 {
    return a + b;
}

test "@Frame" {
    var frame: @Frame(add) = async add(1, 2);
    try expect(await frame == 3);
}
```

[`@frame()`](https://ziglang.org/documentation/master/#frame)返回一个指向当前函数帧的指针。与`suspend`点类似，如果在函数中找到此调用，则推断它是异步的。所有指向帧的指针都强制转换为特殊类型`anyframe`，你可以在此基础上使用`resume`。

例如，这允许我们编写一个恢复自身的函数。
```zig
fn double(value: u8) u9 {
    suspend {
        resume @frame();
    }
    return value * 2;
}

test "@frame 1" {
    var f = async double(1);
    try expect(nosuspend await f == 2);
}
```

或者，更有趣的是，我们可以用它来告诉其他函数恢复我们。这里我们引入**挂起块**。当进入一个挂起块时，async函数已经被认为挂起了（即它可以被恢复）。这意味着我们可以使用除上次恢复程序之外的其他程序来恢复函数。

```zig
const std = @import("std");

fn callLater(comptime laterFn: fn () void, ms: u64) void {
    suspend {
        wakeupLater(@frame(), ms);
    }
    laterFn();
}

fn wakeupLater(frame: anyframe, ms: u64) void {
    std.time.sleep(ms * std.time.ns_per_ms);
    resume frame;
}

fn alarm() void {
    std.debug.print("Time's Up!\n", .{});
}

test "@frame 2" {
    nosuspend callLater(alarm, 1000);
}
```

使用`anyframe`数据类型可以被认为是一种类型擦除，因为我们不再确定函数或函数帧的具体类型。这是有用的，因为它仍然允许我们恢复帧——在许多代码中，我们不会关心细节，只会想要恢复它。这给了我们一个可以用于异步逻辑的具体类型。

`anyframe`的自然缺点是我们丢失了类型信息，并且我们不再知道函数的返回类型是什么。这意味着我们不能await一个`anyframe`。Zig对此的解决方案是`anyframe->T`类型，其中`T`是帧的返回类型。

```zig
fn zero(comptime x: anytype) x {
    return 0;
}

fn awaiter(x: anyframe->f32) f32 {
    return nosuspend await x;
}

test "anyframe->T" {
    var frame = async zero(f32);
    try expect(awaiter(&frame) == 0);
}
```

# 基本事件循环实现

事件循环是一种设计模式，在这种模式中，事件被分派和/或等待。这将意味着在满足条件时恢复挂起的异步帧的某种服务或运行时。这是Zig异步功能最强大、最有用的用例。

这里我们将实现一个基本的事件循环。这将允许我们在给定的时间内提交要执行的任务。我们将使用它来提交成对的任务，这些任务将打印程序开始以来的时间。下面是输出的一个示例。

```
[task-pair b] it is now 499 ms since start!
[task-pair a] it is now 1000 ms since start!
[task-pair b] it is now 1819 ms since start!
[task-pair a] it is now 2201 ms since start!
```

这里是实现。

<!--no_test-->
```zig
const std = @import("std");

// 用来获取单调时间，而不是时钟时间
var timer: ?std.time.Timer = null;
fn nanotime() u64 {
    if (timer == null) {
        timer = std.time.Timer.start() catch unreachable;
    }
    return timer.?.read();
}

// 保存帧，以及何时恢复帧的纳时
const Delay = struct {
    frame: anyframe,
    expires: u64,
};

// 挂起调用者，稍后由操作循环恢复
fn waitForTime(time_ms: u64) void {
    suspend timer_queue.add(Delay{
        .frame = @frame(),
        .expires = nanotime() + (time_ms * std.time.ns_per_ms),
    }) catch unreachable;
}

fn waitUntilAndPrint(
    time1: u64,
    time2: u64,
    name: []const u8,
) void {
    const start = nanotime();

    // 挂起自己，当time1过去时被唤醒
    waitForTime(time1);
    std.debug.print(
        "[{s}] it is now {} ms since start!\n",
        .{ name, (nanotime() - start) / std.time.ns_per_ms },
    );

    // 挂起自己，当time2过去时被唤醒
    waitForTime(time2);
    std.debug.print(
        "[{s}] it is now {} ms since start!\n",
        .{ name, (nanotime() - start) / std.time.ns_per_ms },
    );
}

fn asyncMain() void {
    // 存储任务的异步帧
    var tasks = [_]@Frame(waitUntilAndPrint){
        async waitUntilAndPrint(1000, 1200, "task-pair a"),
        async waitUntilAndPrint(500, 1300, "task-pair b"),
    };
    // 使用|*t|，因为|t|将是一个不能等待的*const @Frame(...)
    for (tasks) |*t| await t;
}

// 任务优先级队列
// 较低的.expires => 高优先级 => 待执行的任务队列
var timer_queue: std.PriorityQueue(Delay, void, cmp) = undefined;
fn cmp(context: void, a: Delay, b: Delay) std.math.Order {
    _ = context;
    return std.math.order(a.expires, b.expires);
}

pub fn main() !void {
    timer_queue = std.PriorityQueue(Delay, void, cmp).init(
        std.heap.page_allocator, undefined
    );
    defer timer_queue.deinit();

    var main_task = async asyncMain();

    // 事件循环体弹出下一个要执行的任务
    while (timer_queue.removeOrNull()) |delay| {
        // 等到执行下一个任务的时间到了
        const now = nanotime();
        if (now < delay.expires) {
            std.time.sleep(delay.expires - now);
        }
        // 执行下一个任务
        resume delay.frame;
    }

    nosuspend await main_task;
}
```

# 第五章结束

本章未完，将来应该包含[`std.event.Loop`](https://ziglang.org/documentation/master/std/#std;event.Loop)和事件IO的用法。

欢迎反馈和PR。
