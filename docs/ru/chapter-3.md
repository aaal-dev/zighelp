---
title: "Глава 3 - Система сборки"
weight: 4
date: 2021-02-12 12:49:00
description: "Глава 3 - Система сборки языка Zig в деталях."
---

## Режимы сборки  

| Режим сборки  | Безопасность в рантайме | Оптимизации         |
|---------------|-------------------------|---------------------|
| Debug         | Да                      | Нет                 |
| ReleaseSafe   | Да                      | Да, для скорости    |
| ReleaseSmall  | Нет                     | Да, для размера     |
| ReleaseFast   | Нет                     | Да, для скорости    |

Zig предоставляет четыре режима для сборки. Debug является режимом по умолчанию. Он тратит меньше всего времени на компиляцию.

Режимы можно применять для команд `zig run` и `zig test` через аргументы `-O ReleaseSafe`, `-O ReleaseSmall` и `-O ReleaseFast`.

Режимы с включенной безопасностью в рантайме имеют небольшие потери в скорости выполнения программы. Не смотря на это пользователям рекомендуется разрабатывать программы именно в этих режимах.

## Собираем код

Для сборки исполняемого приложения, библиотеки или объектного файла используются команды `zig build-exe`, `zig build-lib`, или `zig build-obj`, соответственно. Команды принимают аргументы в виде опций сборки и файлов с исходным кодом.

Пример часто используемых опций:
- `-fsingle-threaded`, указывает что результат компиляции будет выполняться в одном потоке. Поэтому будет отключена защита потоков за ненадобностью.
- `-fstrip`, удаляет debug-информацию из итогового файла.
- `--dynamic`, используется в связке с `zig build-lib` для сборки динамической библиотеки.

Создадим небольшой «Hello World». Сохраним файл как `tiny-hello.zig`, и запустим `zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded`. Результатом для целевой платформы `x86_64-windows` будет испольняемый файл размером 2.5 килобайта. Для другой целеовй платформы результат может отличаться.

<!--no_test-->
```zig
const std = @import("std");

pub fn main() void {
    std.io.getStdOut().writeAll(
        "Hello World!",
    ) catch unreachable;
}
```

## Кросс-компиляция

Компиляция в Zig по умолчанию выполняется под целевую платформу архитектуры вашего процессора и вашей операционной системы. Указать другую целевую платформу компиляции можно через опцию `-target`. Соберём предыдущий пример небольшого «Hello World» для 64-х битной платформы ARM и операционной системы семейства Linux.

```bash
zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded -target aarch64-linux
```

Для теста испольняемого файла собранного для других целевых платформ можно использовать приложения типа [QEMU](https://www.qemu.org/) и схожих.

Список поддерживаемых для кросс-компиляции архитектур процессора:
- `x86_64`
- `arm`
- `aarch64`
- `i386`
- `riscv64`
- `wasm32`

Список поддерживаемых для кросс-компиляции операционных систем:
- `linux`
- `macos`
- `windows`
- `freebsd`
- `netbsd`
- `dragonfly`
- `UEFI`

Поддерживаются и другие целевые платформы для компиляции, но они полноценно не протестированы. Список хорошо протестированных целевых платформ расширяется медленно. За дополниетльной информацией стоит заглянуть [на официальный сайт](https://ziglang.org/download/) в примечания к версии (release notes) компилятора, которой вы используете. [Ссылка](https://ziglang.org/download/0.11.0/release-notes.html#Support-Table) для текущей версии: 0.11.0. 

Также Zig по умолчанию компилируется под набор инструкций вашего процессора. То есть готовые файлы могут не заработать на других компьютерах с процессорами, у которых другой набор инструкций. Поэтому рекомендуется указывать конкретную минимальную микроархитектуру процессоров для большей совместимости. Примечание: выбор более старой микроархитектуры даст большую совместимость, но вместе с этим пропадёт возможность использовать новые процессорные инструкции, которые могут ускорять выполнение программ. Это компромисс между совместимостью и эффективностью (скоростью выполнения).

Соберём пример небольшого «Hello World» для микроархитектуры процессора SandyBridge (Intel x86_64, приблизительно 2011 год). Так мы будем уверены в том, что любой, у кого процессор имеет 64-х битную архитектуру x86 (или просто x64) сможет запустить скомпилированный исполняемый файл. Для простоты мы можем использовать слово `native` в замен архитекутуры процессора или операционной системы в опции целевой платформы, чтобы указать компилятору использовать информацию о нашей целевой платформе.

`zig build-exe .\tiny-hello.zig -target x86_64-native -mcpu sandybridge`

Детальная информация о том, какие целевые платформы, архитектуры и микроархитектуры процессора, операционные системы и типы ABI доступны можно получить через команду `zig targets`. Примечание: объем текста выводимого этой команды очень большой, рекомендуется выводить его в отдельный файл, например так `zig targets > targets.json`.

## Команда `zig build`

The `zig build` command allows users to compile based on a `build.zig` file. `zig init-exe` and `zig init-lib` can be used to give you a baseline project.

Let's use `zig init-exe` inside a new folder. This is what you will find.
```
.
├── build.zig
└── src
    └── main.zig
```
`build.zig` contains our build script. The *build runner* will use this `pub fn build` function as its entry point - this is what is executed when you run `zig build`.

<!--no_test-->
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
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
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // This declares intent for the executable to be installed into the
    // standard location when the user invokes the "install" step (the default
    // step when running `zig build`).
    b.installArtifact(exe);

    const run_cmd = exe.run();
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);
}
```

`main.zig` contains our executable's entry point.

<!--no_test-->
```zig
const std = @import("std");

pub fn main() anyerror!void {
    std.log.info("All your codebase are belong to us.", .{});
}
```

Upon using the `zig build` command, the executable will appear in the install path. Here we have not specified an install path, so the executable will be saved in `./zig-out/bin`.

## Сборщик

Zig's [`std.Build`](https://ziglang.org/documentation/master/std/#A;std:Build) type contains the information used by the build runner. This includes information such as:

- the build target
- the release mode
- locations of libraries
- the install path
- build steps

## CompileStep

The `std.build.CompileStep` type contains information required to build a library, executable, object, or test.

Let's make use of our `Builder` and create a `CompileStep` using `Builder.addExecutable`, which takes in a name and a path to the root of the source.

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

## Модули

The Zig build system has the concept of modules, which are other source files written in Zig. Let's make use of a module.

From a new folder, run the following commands.
```
zig init-exe
mkdir libs
cd libs
git clone https://github.com/Sobeston/table-helper.git
```

Your directory structure should be as follows.

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

To your newly made `build.zig`, add the following lines.

<!--no_test-->
```zig
    const table_helper = b.addModule("table-helper", .{
        .source_file = .{ .path = "libs/table-helper/table-helper.zig" }
    });
    exe.addModule("table-helper", table_helper);
```

Now when run via `zig build`, [`@import`](https://ziglang.org/documentation/master/#import) inside your `main.zig` will work with the string "table-helper". This means that main has the table-helper package. Packages (type [`std.build.Pkg`](https://ziglang.org/documentation/master/std/#std;build.Pkg)) also have a field for dependencies of type `?[]const Pkg`, which is defaulted to null. This allows you to have packages which rely on other packages. 

Place the following inside your `main.zig` and run `zig build run`. 

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

This should print this table to your console.

```
Version Date       
------- ---------- 
0.7.1   2020-12-13 
0.7.0   2020-11-08 
0.6.0   2020-04-13 
0.5.0   2019-09-30 
```


Zig does not yet have an official package manager. Some unofficial experimental package managers however do exist, namely [gyro](https://github.com/mattnite/gyro) and [zigmod](https://github.com/nektro/zigmod). The `table-helper` package is designed to support both of them.

Some good places to find packages include: [astrolabe.pm](https://astrolabe.pm), [zpm](https://zpm.random-projects.net/), [awesome-zig](https://github.com/nrdmn/awesome-zig/), and the [zig tag on GitHub](https://github.com/topics/zig).

## Шаги сборки

Build steps are a way of providing tasks for the build runner to  execute. Let's create a build step, and make it the default. When you run `zig build` this will output `Hello!`. 

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

We called `b.installArtifact(exe)` earlier - this adds a build step which tells the builder to build the executable.

## Генерируем документацию

The Zig compiler comes with automatic documentation generation. This can be invoked by adding `-femit-docs` to your `zig build-{exe, lib, obj}` or `zig run` command. This documentation is saved into `./docs`, as a small static website.

Zig's documentation generation makes use of *doc comments* which are similar to comments, using `///` instead of `//`, and preceding globals.

Here we will save this as `x.zig` and build documentation for it with `zig build-lib -femit-docs x.zig -target native-windows`. There are some things to take away here:
-  Only things that are public with a doc comment will appear
-  Blank doc comments may be used
-  Doc comments can make use of subset of markdown
-  Things will only appear inside generated documentation if the compiler analyses them; you may need to force analysis to happen to get things to appear.

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

When using a `build.zig` this may be invoked by setting the `emit_docs` field to `.emit` on a `CompileStep`. We can create a build step to generate docs as follows and invoke it with `$ zig build docs`.

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

This generation is experimental, and often fails with complex examples. This is used by the [standard library documentation](https://ziglang.org/documentation/master/std/).

When merging error sets, the left-most error set's documentation strings take priority over the right. In this case, the doc comment for `C.PathNotFound` is the doc comment provided in `A`.

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

## Конец главы 3

Эта глава ещё . In the future it will contain advanced usage of `zig build`.

Обратная связь и пулл реквесты приветствуются.
