---
title: "第三章 - 构建系统"
weight: 4
date: 2021-02-12 12:49:00
description: "第三章 - Zig语言的构建系统细节。"
---

# 构建模式

Zig提供了四种构建模式，其中调试是默认模式，因为它产生的编译时间最短。

|               | Runtime Safety | Optimizations |
|---------------|----------------|---------------|
| Debug         | Yes            | No            |
| ReleaseSafe  | Yes            | Yes, Speed    |
| ReleaseSmall | No             | Yes, Size     |
| ReleaseFast  | No             | Yes, Speed    |

这些可以通过参数`-O ReleaseSafe`、`-O ReleaseSmall`和`-O ReleaseFast`在`zig run`和`zig test`中启用。

建议用户在开发软件时启用运行时安全性，尽管它在速度上的缺点很小。

# 输出可执行文件

命令`zig build-exe`、`zig build-lib`和`zig build-obj`可分别用于输出可执行文件、库和对象。这些命令接受源文件和参数。

一些常见的参数：
- `-fsingle-threaded`，它断言二进制文件是单线程的。这将使线程安全措施（如互斥锁）变为无操作。
- `-fstrip`，从二进制文件中删除调试信息。
- `--dynamic`，它与`zig build-lib`一起用于输出动态/共享库。

让我们创建一个小小的hello world。将此保存为`tiny-hello.zig`。然后运行`zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded`。目前对于`x86_64-windows`，这将生成一个2.5KiB的可执行文件。

<!--no_test-->
```zig
const std = @import("std");

pub fn main() void {
    std.io.getStdOut().writeAll(
        "Hello World!",
    ) catch unreachable;
}
```

# 交叉编译

默认情况下，Zig将针对你的CPU和操作系统组合进行编译。这可以被`-target`覆盖。让我们将这个小小的hello world编译为64位arm linux平台。

`zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded -target aarch64-linux`

[QEMU](https://www.qemu.org/)或类似的工具可以用来方便地测试为其他平台制作的可执行文件。

一些可以交叉编译的CPU架构：
- `x86_64`
- `arm`
- `aarch64`
- `i386`
- `riscv64`
- `wasm32`

有些操作系统可以交叉编译：
- `linux`
- `macos`
- `windows`
- `freebsd`
- `netbsd`
- `dragonfly`
- `UEFI`

许多其他目标也可用于编译，但目前还没有经过很好的测试。有关更多信息，请参阅[Zig's support table](https://ziglang.org/learn/overview/#wide-range-of-targets-supported)；经过良好测试的目标名单正在慢慢扩大。

由于Zig默认为特定的CPU编译，因此这些二进制文件可能无法在CPU体系结构略有不同的其他计算机上运行。为获得更好的兼容性，指定一个特定的基准CPU模型可能会更有用。注意：选择较旧的CPU架构将带来更好的兼容性，但也意味着你也错过了较新的CPU指令；这里存在效率/速度与兼容性之间的权衡。

让我们为sandybridge CPU（Intel x86_64，大约2011年）编译一个二进制文件，这样我们就可以合理地确定使用x86_64 CPU的人可以运行我们的二进制文件。这里我们可以使用`native`来代替我们的CPU或OS，来使用我们的系统。

`zig build-exe .\tiny-hello.zig -target x86_64-native -mcpu sandybridge`

关于哪些架构、操作系统、CPU和ABI可用的详细信息（关于ABI的详细信息将在下一章中介绍）可以通过运行`zig targets`找到。注意：输出很长，你可能想把它管道到一个文件，例如：`zig targets > targets.json`。

# Zig构建

`zig build`命令允许用户基于构建进行编译`build.zig`文件。`zig init-exe`和`zig init-lib`可以用来给你一个基线项目。

让我们在一个新文件夹中使用`zig init-exe`。你就会发现这个。
```
.
├── build.zig
└── src
    └── main.zig
```
`build.zig`包含我们的构建脚本。*build runner*将使用这个`pub fn build`函数作为它的入口点——这是在运行`zig build`时执行的。

<!--no_test-->
```zig
const std = @import("std");

// Although this function looks imperative, note that its job is to
// declaratively construct a build graph that will be executed by an external
// runner.
pub fn build(b: *std.Build) void {
    // Standard target options allows the person running `zig build` to choose
    // what target to build for. Here we do not override the defaults, which
    // means any target is allowed, and the default is native. Other options
    // for restricting supported target set are available.
    const target = b.standardTargetOptions(.{});

    // Standard optimization options allow the person running `zig build` to select
    // between Debug, ReleaseSafe, ReleaseFast, and ReleaseSmall. Here we do not
    // set a preferred release mode, allowing the user to decide how to optimize.
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "tmp",
        // In this case the main source file is merely a path, however, in more
        // complicated build scripts, this could be a generated file.
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // This declares intent for the executable to be installed into the
    // standard location when the user invokes the "install" step (the default
    // step when running `zig build`).
    b.installArtifact(exe);

    // This *creates* a Run step in the build graph, to be executed when another
    // step is evaluated that depends on it. The next line below will establish
    // such a dependency.
    const run_cmd = b.addRunArtifact(exe);

    // By making the run step depend on the install step, it will be run from the
    // installation directory rather than directly from within the cache directory.
    // This is not necessary, however, if the application depends on other installed
    // files, this ensures they will be present and in the expected location.
    run_cmd.step.dependOn(b.getInstallStep());

    // This allows the user to pass arguments to the application in the build
    // command itself, like this: `zig build run -- arg1 arg2 etc`
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // This creates a build step. It will be visible in the `zig build --help` menu,
    // and can be selected like this: `zig build run`
    // This will evaluate the `run` step rather than the default, which is "install".
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    // Creates a step for unit testing. This only builds the test executable
    // but does not run it.
    const unit_tests = b.addTest(.{
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);

    // Similar to creating the run step earlier, this exposes a `test` step to
    // the `zig build --help` menu, providing a way for the user to request
    // running the unit tests.
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

`main.zig`包含我们的可执行文件的入口点。

<!--no_test-->
```zig
const std = @import("std");

pub fn main() !void {
    // Prints to stderr (it's a shortcut based on `std.io.getStdErr()`)
    std.debug.print("All your {s} are belong to us.\n", .{"codebase"});

    // stdout is for the actual output of your application, for example if you
    // are implementing gzip, then only the compressed bytes should be sent to
    // stdout, not any debugging messages.
    const stdout_file = std.io.getStdOut().writer();
    var bw = std.io.bufferedWriter(stdout_file);
    const stdout = bw.writer();

    try stdout.print("Run `zig build test` to run the tests.\n", .{});

    try bw.flush(); // don't forget to flush!
}

test "simple test" {
    var list = std.ArrayList(i32).init(std.testing.allocator);
    defer list.deinit(); // try commenting this out and see if zig detects the memory leak!
    try list.append(42);
    try std.testing.expectEqual(@as(i32, 42), list.pop());
}
```

使用`zig build`命令后，可执行文件将出现在安装路径中。这里我们没有指定安装路径，因此可执行文件将保存在`./zig-out/bin`中。可以使用`Step.Compile`结构中的`override_dest_dir`字段指定自定义安装路径。

# 构建器

Zig的[`std.Build`](https://ziglang.org/documentation/master/std/#A;std:Build)类型包含构建运行程序使用的信息。这包括以下信息：

- 构建目标
- 发布模式
- 库位置
- 安装路径
- 构建步骤

# CompileStep

`std.build.CompileStep` 类型包含构建库、可执行文件、对象或测试所需的信息。

让我们利用`Builder`并使用`Builder.addExecutable`创建一个`CompileStep`。它接受一个名称和到源的根目录的路径。

<!--no_test-->
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const exe = b.addExecutable(.{
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
    });
    b.installArtifact(exe);
}
```

# 模块

Zig构建系统有模块的概念，它是用Zig编写的其他源文件。让我们利用一个模块。

在一个新文件夹中，运行以下命令。
```
zig init-exe
mkdir libs
cd libs
git clone https://github.com/Sobeston/table-helper.git
```

你的目录结构应该如下所示。

```
.
├── build.zig
├── libs
│   └── table-helper
│       ├── example-test.zig
│       ├── README.md
│       ├── table-helper.zig
│       └── zig.mod
└── src
    └── main.zig
```

为你的新创建的`build.zig`，添加以下行。

<!--no_test-->
```zig
    const table_helper = b.addModule("table-helper", .{
        .source_file = .{ .path = "libs/table-helper/table-helper.zig" }
    });
    exe.addModule("table-helper", table_helper);
```

现在，当通过`zig build`运行时，[`@import`](https://ziglang.org/documentation/master/#import)在`main.zig`中将使用字符串“table-helper”。这意味着main具有table-helpe包。包（类型[`std.build.Pkg`](https://ziglang.org/documentation/master/std/#std;build.Pkg)）也有一个字段用于类型`?[]const Pkg`的依赖项，默认为null。这允许你拥有依赖于其他包的包。

将以下内容放入`main.zig`文件中，并运行`zig build run`。

<!--no_test-->
```zig
const std = @import("std");
const Table = @import("table-helper").Table;

pub fn main() !void {
    try std.io.getStdOut().writer().print("{}\n", .{
        Table(&[_][]const u8{ "Version", "Date" }){
            .data = &[_][2][]const u8{
                .{ "0.7.1", "2020-12-13" },
                .{ "0.7.0", "2020-11-08" },
                .{ "0.6.0", "2020-04-13" },
                .{ "0.5.0", "2019-09-30" },
            },
        },
    });
}
```

这会将该表打印到控制台中。

```
Version Date       
------- ---------- 
0.7.1   2020-12-13 
0.7.0   2020-11-08 
0.6.0   2020-04-13 
0.5.0   2019-09-30 
```


Zig还没有正式的包管理器。然而，一些非官方的实验包管理器确实存在，即[gyro](https://github.com/mattnite/gyro)和[zigmod](https://github.com/nektro/zigmod)。`table-helper` 包被设计为支持这两种方法。

一些找到包的好地方包括：[astrolabe.pm](https://astrolabe.pm)、[zpm](https://zpm.random-projects.net/)、[awesome-zig](https://github.com/nrdmn/awesome-zig/)，以及[GitHub上的zig标签](https://github.com/topics/zig)。

# 构建步骤

构建步骤是为构建运行程序提供要执行的任务的一种方式。让我们创建一个构建步骤，并将其设为默认值。当你运行`zig build`时，它将输出`Hello!`。

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const step = b.step("task", "do something");
    step.makeFn = myTask;
    b.default_step = step;
}

fn myTask(self: *std.build.Step, progress: *std.Progress.Node) !void {
    std.debug.print("Hello!\n", .{});
    _ = progress;
    _ = self;
}
```

我们之前调用了`b.installArtifact(exe)`——这添加了一个构建步骤，告诉构建器构建可执行文件。

# 生成文档

Zig编译器自带自动文档生成功能。这可以通过在`zig build-{exe, lib, obj}`或`zig run`命令中添加`-femit-docs`来调用。这个文档保存在`./docs`中，作为一个小的静态网站。

Zig的文档生成使用类似于注释的*文档注释*，使用`///`而不是`//`和前面的全局变量。

这里我们将把它保存为`x.zig`，并使用`zig build-lib -femit-docs x.zig -target native-windows`为其构建文档。这里有一些值得借鉴的地方：
-  只有带有文档注释的公开内容才会出现
-  可以使用空白文档注释
-  文档注释可以利用markdown的子集
-  只有当编译器分析它们时，才会出现在生成的文档中；你可能需要强迫分析发生，以使事情出现。

<!--no_test-->
```zig
const std = @import("std");
const w = std.os.windows;

///**Opens a process**, giving you a handle to it. 
///[MSDN](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)
pub extern "kernel32" fn OpenProcess(
    ///[The desired process access rights](https://docs.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights)
    dwDesiredAccess: w.DWORD,
    ///
    bInheritHandle: w.BOOL,
    dwProcessId: w.DWORD,
) callconv(w.WINAPI) ?w.HANDLE;

///spreadsheet position
pub const Pos = struct{
    ///row
    x: u32,
    ///column
    y: u32,
};

pub const message = "hello!";

//used to force analysis, as these things aren't otherwise referenced.
comptime {
    _ = OpenProcess;
    _ = Pos;
    _ = message;
}

//Alternate method to force analysis of everything automatically, but only in a test build:
test "Force analysis" {
    comptime {
        std.testing.refAllDecls(@This());
    }
}
```

使用`build.zig`时。这可以通过在`CompileStep`中将`emit_docs`字段设置为`.emit`来调用。我们可以创建一个生成文档的构建步骤，如下所示，并使用`$ zig build docs`调用它。

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const mode = b.standardReleaseOptions();

    const lib = b.addStaticLibrary("x", "src/x.zig");
    lib.setBuildMode(mode);
    lib.install();

    const tests = b.addTest("src/x.zig");
    tests.setBuildMode(mode);

    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&tests.step);

    //Build step to generate docs:
    const docs = b.addTest("src/x.zig");
    docs.setBuildMode(mode);
    docs.emit_docs = .emit;
    
    const docs_step = b.step("docs", "Generate docs");
    docs_step.dependOn(&docs.step);
}
```

这个生成器是实验性的，在复杂的例子上经常失败。目前为[标准库文档](https://ziglang.org/documentation/master/std/)所用。

合并错误集时，最左边的错误集的文档字符串优先于右边的。在这种情况下，`C.PathNotFound`的文档注释是`A`中提供的文档注释。

<!--no_test-->
```zig
const A = error{
    NotDir,

    /// A doc comment
    PathNotFound,
};
const B = error{
    OutOfMemory,

    /// B doc comment
    PathNotFound,
};

const C = A || B;
```

# 第三章结束

本章未完。在未来，它将包含`zig build`”的高级用法。

欢迎反馈和PR。
