---
title: "Chapter 0 - Getting Started"
weight: 1
date: 2023-09-11 18:00:00
description: "Ziglearn - A Guide / Tutorial for the Zig programming language. Install and get started with ziglang here."
---

# 欢迎

[Zig](https://ziglang.org)是一种通用编程语言和工具链，用于维护 __健壮的__ 、 __最优的__ 和 __可重用__ 的软件。

警告：Zig目前未达1.0；不建议在生产环境中使用。

要遵循本指南，我们假设你具备：
   * 有编程经验
   * 对低级编程概念有一定的了解

了解一门语言，比如C、C++、Rust、Go、Pascal或类似的语言，将有助于你掌握本指南。你必须有一个编辑器，终端和互联网连接。本指南是非官方的，与Zig软件基金会无关，旨在从一开始就按顺序阅读。
# 安装

本指南假设使用Zig 0.11，这是撰写本文时最新的主要版本。

1.  下载并提取Zig主分支的预构建二进制版本：
```
https://ziglang.org/download/
```

2. 将Zig添加到你的路径中
   - linux, macos, bsd

      将Zig二进制文件的位置添加到`PATH`环境变量中。关于安装，添加`export PATH=$PATH:~/zig`或类似于`/etc/profile`（系统范围）或`$HOME/.profile`。如果这些更改不能立即应用，请从shell运行这行代码。
   - windows

      a) 系统范围（admin powershell）

      ```powershell
      [Environment]::SetEnvironmentVariable(
         "Path",
         [Environment]::GetEnvironmentVariable("Path", "Machine") + ";C:\your-path\zig-windows-x86_64-your-version",
         "Machine"
      )
      ```

      b) 用户级别（powershell）

      ```powershell
      [Environment]::SetEnvironmentVariable(
         "Path",
         [Environment]::GetEnvironmentVariable("Path", "User") + ";C:\your-path\zig-windows-x86_64-your-version",
         "User"
      )
      ```

      关闭终端并创建一个新终端。

3. 使用`zig version`验证安装。输出应该是这样的：
```
$ zig version
0.11
```

4. （可选，第三方）对于编辑器中的自动完成和跳转到定义，请从以下路径安装Zig语言服务器：
```
https://github.com/zigtools/zls/
```
5. （可选）加入一个[Zig community](https://github.com/ziglang/zig/wiki/Community)。

# Hello World

创建一个名为`main.zig`的文件，内容如下：

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, {s}!\n", .{"World"});
}
```
###### （注意：确保你的文件使用空格缩进、LF换行符及以UTF-8编码！）

使用`zig run main.zig`来构建和运行它。在这个例子中，`Hello, World!` 将被写入stderr，并假定永远不会失败。
