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

Команда `zig build` использует файл `build.zig` для компиляции кода. Команды `zig init-exe` и `zig init-lib` генерируют готовые простые шаблоны проектов исполняемого файла и библиотеки, соответственно.

Воспользуемся `zig init-exe` внутри свеже созданной папки. Вот что будет внутри по итогу.
```
.
├── build.zig
└── src
    └── main.zig
```
Файл `build.zig` содержит скрипт сборки. *Сборщтк* использует функцию `pub fn build`, как точку входа - это то, что будет вызвано, когда запуститься команда `zig build`.

<!--no_test-->
```zig
const std = @import("std");

// Несмотря на то, что эта функция выглядит императивной, имейте в виду, что её задача
// декларативно сконструировать граф сборки, который будет использован внешним сборщиком
pub fn build(b: *std.Build) void {
    // Функция предосталяют для команды `zig build` возможность выбрать целевую платформу
    // для сборки. Сейчас мы не перезаписывает значения по умолчанию, это означает,
    // что разрешается использование любой целевой платформы, и по умолчанию это нативная
    // (та, на которой выполняется команда). При этом доступны другие опции для ограничения
    // поддерживаемых целевых платформ.
    const target = b.standardTargetOptions(.{});

    // Функция предоставляет для команды `zig build` возможность выбрать между
    // режимами Debug, ReleaseSafe, ReleaseFast, или ReleaseSmall. Сейчас мы
    // не устанавливаем режим сборки, предоставляя пользователю возможность самому
    // выбрать необходимые оптимизации.
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "test-exe",
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

Файл `main.zig` содержит функцию `pub fn main`, как точку входа исполняемого приложения.

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

После выполнения команды `zig build`, исполняемый файл появится в папке пути установки. А так как мы не указывали путь установки, то исполняемый файл будет сохранён в папке `./zig-out/bin`, которая будет создана автоматически внутри папки проекта.

## Сборщик

Тип данных [`std.Build`](https://ziglang.org/documentation/master/std/#A;std:Build) содержит информацию, которая используется для сборки проекта. Внутри содержится информация о:

- целевой платформе,
- режиме сборки,
- путях до необходимых библиотек, 
- пути установки,
- шагах сборки.

## Шаги компиляции

Тип данных `std.Build.Step.Compile` содержит информацию о шаге компиляции, которая требуется для создания исполняемого файла, библиотеки, объектного файла или теста.

Для создания шага компиляции исполняемого файла используется функция `addExecutable`, которая принимает структуру с типом `ExecutableOptions`. Единственным обязательным к заполнению полем структуры явяяется поле `name`. Остальные поля опциональны. В примере ниже в структуре передаются поля имени и пути до основного для проекта файла с кодом языка Zig.  

<!--no_test-->
```zig
pub fn build(b: *std.Build) void {
    const exe = b.addExecutable(.{
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
    });
    b.installArtifact(exe);
}
```

## Модули

Система сборки языка Zig использует концепуию модулей, которые представляют собой другие файлы с кодом языка Zig. Let's make use of a module.

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
