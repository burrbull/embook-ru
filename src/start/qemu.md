# QEMU

Мы начнем писать программу для [LM3S6965], микроконтроллера Cortex-M3.
Мы выбрали его в качестве начальной цели, потому что его [можно промоделировать](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm)
на QEMU, поэтому Вам не нужно баловаться с оборудованием в этом разделе
и мы можем сосредоточиться на инструментах и процессе разработки.

[LM3S6965]: http://www.ti.com/product/LM3S6965

**ВАЖНО**
Мы будем использовать имя "app" в качестве названия проекта в этом руководстве.
Где бы Вы не увидели слово "app", Вы должны заменить его на название,
выбранное Вами для проекта.  Или можете назвать свой проект "app"
и избежать замен.

## Создание нестандартной программы на Rust

Мы будем использовать проект шаблона [`cortex-m-quickstart`] для
генерации проекта на его основе.

[`cortex-m-quickstart`]: https://github.com/rust-embedded/cortex-m-quickstart

### Используя `cargo-generate`
Для начала установите cargo-generate
```console
cargo install cargo-generate
```
Затем сгенерируйте новый проект
```console
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```

```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```

```console
cd app
```

### Используя `git`

Клонируйте репозитарий

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

А затем заполните поля в файле `Cargo.toml`

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
test = false
bench = false
```

### Не используя ни тот ни другой

Скачайте последнюю версию шаблона `cortex-m-quickstart` и распакуйте его.

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

Или можете открыть в браузере [`cortex-m-quickstart`], нажать зеленую кнопку "Clone or
download", а затем нажать "Download ZIP".

Затем заполните пустующие поля в файле `Cargo.toml`, как показано во второй части
версии "Используя `git`".

## Program Overview

Для удобства мы приведем здесь главные куски исходного кода из `src/main.rs`:

```rust,ignore
#![no_std]
#![no_main]

extern crate panic_halt;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // your code goes here
    }
}
```

Эта программа немного отличается от обычной программы Rust, давайте присмотримся.

`#![no_std]` показывает, что эта программа *не* будет линковаться со стандартной
библиотекой `std`. Вместо этого она будет линковаться с ее подмножеством:
крейтом `core`.

`#![no_main]` показывает, что эта программа не будет использовать стандартный
интерфейс `main`, который используют большинство программ на Rust.
Основная причина существования `no_main` - использование интерфейса `main` в
контексте `no_std` требует ночную сборку.

`extern crate panic_halt;`. Эта библиотека предоставляет `panic_handler`, который
определяет поведение программы при появлении паники. Более подробно об
этом написано в главе [Паника](panicking.md) этой кини.

[`#[entry]`][entry] - атрибут предоставляемый крейтом [`cortex-m-rt`], который используется
для обозначения точки входа в программу. Так как мы не используем стандартный
интерфейс `main`, нам нужен другой способ обозначить точку входа в программу, и это `#[entry]`.

[entry]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt

`fn main() -> !`. Наша программа будет *единственным* процессом, выполняемым
на целевом оборудовании, поэтому мы не хотим, чтобы она закончилась!
Мы используем [бесконечную функцию](https://doc.rust-lang.org/rust-by-example/fn/diverging.html)
(знак `-> !` в сигнатуре функции), чтобы
убедиться на этапе компиляции, что этого не произойдет.

## Кросскомпиляция

Следующий гаг *кросс*компилировать программу для архитектуры Cortex-M3.
Это также просто, как запустить `cargo build --target $TRIPLE`, если Вы знаете,
каким должен быть (`$TRIPLE`). К счатью, в `.cargo/config` шаблона есть ответ:

```console
tail -n6 .cargo/config
```

```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

Для того, чтобы кросскомпилировать для архитектуры Cortex-M3, мы должны
использовать `thumbv7m-none-eabi`. Эта цель компиляции установлена по умолчанию, поэтому эти две команды будут делать одно и тоже:

```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## Проверка

Теперь у нас есть ненативный бинарный ELF-файл в `target/thumbv7m-none-eabi/debug/app`. Мы можем изучить его с помощью `cargo-binutils`.

С помощью `cargo-readobj` мы можем распечатать заголовки ELF, чтобы
убедиться, что это бинарник ARM.

``` console
cargo readobj --bin app -- -file-headers
```

Обратите внимание:
* `--bin app` - сокращение, чтобы изучить бинарник по пути `target/$TRIPLE/debug/app`
* `--bin app` также (пере)компилирует бинарник, если необходимо


``` text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index: 18
```

`cargo-size` может отобразить размер линковочных секций бинарного файла.

> **NOTE** this output assumes that rust-embedded/cortex-m-rt#111 has been
> merged

``` console
cargo size --bin app --release -- -A
```
мы используем `--release`, чтобы получить информацию об оптимизированной версии

``` text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

> Памятка по линковочным секциям ELF
>
> - `.text` содержит инструкции программы
> - `.rodata` содержит константные значения, такие как строки
> - `.data` содержит статически аллоцированные переменные, чьи начальные
>   значения *не* нулевые
> - `.bss` также содержит статически аллоцированные переменные, чьи начальные
>   значения *нулевые*
> - `.vector_table` - *не*стандартная секция, которую мы используем, чтобы
>   разместить таблицу вектора прерываний
> - секции `.ARM.attributes` и `.debug_*` содержат метаданные и
>   *не* будут загружены в устройство во время прошивки бинарника.

**ВАЖНО**: ELF-файлы содержат и другие метаданные, такие как
отладочная информация, поэтому их *размер на диске*, *не* точно
отражает размер, который программа займет на устрйстве. *Всегда*
используйте `cargo-size`, чтобы проверить, насколько велика прошивка
на самом деле.

`cargo-objdump` можно использовать для дисассемблирования бинарного файла.

``` console
cargo objdump --bin app --release -- -disassemble -no-show-raw-insn -print-imm-hex
```

> **ПРИМЕЧАНИЕ** в вашей системе результат может быть другим. Новые версии rustc, LLVM
> и библиотек могут генерировать другой ассемблерный код. Мы обрезали некоторые инструкции
> чтобы сделать пример короче.

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. остальные инструкции обрезаны .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## Запуск

Идем дальше,давайте посмотрим как запустить embedded программу на QEMU!
В этот раз мы будем использовать пример `hello`, который действительно
что-то делает.

Для удобства, приведем исходный код `examples/hello.rs`:

```rust,ignore
//! Печатает "Hello, world!" на хосте, с помощью semihosting

#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    let mut stdout = hio::hstdout().unwrap();
    hprintln!("Hello, world!").unwrap();

    // выйти из QEMU
    // ПРИМЕЧАНИЕ не запускайте на реальном оборудовании; это может повредить состояние OpenOCD
    debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

Эта программа использует что-то, называемое semihosting, чтобы распечатать
текст в консоли *хоста* (компьютера). Когда будем использовать настоящее
оборудование, это потребует отладочную сессию, но в QEMU оно Просто Работает.

Давайте начнем с компиляции примера:

```console
cargo build --example hello
```

Результирующий бинарник будет расположен по пути
`target/thumbv7m-none-eabi/debug/examples/hello`.

Чтобы запустить бинарник на QEMU, выполним следующую команду:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

```text
Hello, world!
```

Команда должна успешно завершиться (код выхода = 0) после того, как
напечает текст. На системе *nix Вы можете проверить это следующей командой:

```console
echo $?
```
```text
0
```

Позвольте расшифровать для Вас, для чего эта длинная команда QEMU:

- `qemu-system-arm`. Это собственно эмулятор QEMU. Есть несколько вариантов
  бинарников QEMU; этот делает полную *системную* эмуляцию машины *ARM* как
  видно из названия.

- `-cpu cortex-m3`. Говорит QEMU эмулировать ЦПУ Cortex-M3. Определяет
  модель ЦПУ, позволяя нам перехватывать некоторые ошибки неправильной
  компиляции: например, запуск программы, скомпилированной для Cortex-M4F,
  которая имеет аппаратное FPU, вызовет ошибку QEMU во время выполнения.

- `-machine lm3s6965evb`. Говорит QEMU эмулировать устройство LM3S6965EVB,
  вычислительную плату, содержащую микроконтроллер LM3S6965.

- `-nographic`. Говорит QEMU не запускать графический интерфейс.

- `-semihosting-config (..)`. Говорит QEMU включить semihosting. Semihosting
  позволяет эмулируемому устройству, среди прочего, использовать стандарный
  ввод/вывод и создавать файлы на хосте.

- `-kernel $file`. Говорит QEMU, какой бинарный файл загружать и запускать
  на эмулируемом устройстве.

Набирать эту длинную команду QEMU - слишком долго! Но мы можем определить
свою команду запуска, чтобы упростить процесс. В `.cargo/config` есть
закоментированная строка запуска, которая вызывает QEMU; давайте ее раскомментируем:

```console
head -n3 .cargo/config
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```

Эта команда будет применяться только для целевой платформы `thumbv7m-none-eabi`,
которая у нас является целью по умолчанию. Теперь `cargo run` будет
компилировать программу и запускать ее на QEMU:

```console
cargo run --example hello --release
```

```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## Отладка

Процесс отладки критичен для embedded разработки. Давайте посмотрим как
это происходит.

Отладка embedded устройства подразумевает *удаленную* отладку, так как программа,
которую мы хотим отлаживать, будет запускаться не на той же машине, что
и программа отладки (GDB или LLDB).

Удаленная отладка включает клиент и сервер. В случае с QEMU клиентом
будет процесс GDB (или LLDB), а сервером будет процесс QEMU, который
также выполняет embedded программу.

В этом разделе мы используем пример `hello`, который уже скомпилировали.

Первый шаг отладки - перевод QEMU в отладочный режим:

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

Эта команда ничего не напечатает, но заблокирует терминал. Мы также
передали в этот раз два дополнительных флага:

- `-gdb tcp::3333`. Говорит QEMU ждать соединения с GDB по TCP-порту 3333.

- `-S`. Говорит QEMU остановить машину на старте. Без этого программа
  дойдет до конца функции main до того, как у нас появится шанс запустить
  отладчик!

Далее мы запускаем GDB в другом терминале и говорим ему загрузить
отладочные символы из примера:

```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**ПРИМЕЧАНИЕ**: возможно Вам нужна не `gdb-multiarch`, а другая версия gdb,
та, которую Вы установили в главе установки. Это может быть `arm-none-eabi-gdb` или просто `gdb`.

Затем из оболочки GDB мы соединяемся с QEMU, который ждет соединения по
TCP-порту 3333.

```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```

Вы увидите, что процесс прервался и программный счетчик указывает
на функцию по имени `Reset`. Это обработчик сброса: который ядра Cortex-M
выполняют во время загрузки.

Этот обработчик сброса в конечном счете запустит нашу функцию main.
Давайте пропустим весь этот процесс используя точку останова и
команду `continue`:

```console
break main
```

```text
Breakpoint 1 at 0x400: file examples/panic.rs, line 29.
```

```console
continue
```

```text
Continuing.

Breakpoint 1, main () at examples/hello.rs:17
17          let mut stdout = hio::hstdout().unwrap();
```

Теперь мы достигли кода, который печатает "Hello, world!". Давайте перейдем
дальше командой `next`.

``` console
next
```

```text
18          writeln!(stdout, "Hello, world!").unwrap();
```

```console
next
```

```text
20          debug::exit(debug::EXIT_SUCCESS);
```

В этой точке Вы должны увидеть "Hello, world!", напечатанный на
терминале, в котором запущен `qemu-system-arm`.

```text
$ qemu-system-arm (..)
Hello, world!
```

Еще один вызов `next` прервет процесс QEMU.

```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

Теперь можете завершить сессию GDB.

``` console
quit
```
