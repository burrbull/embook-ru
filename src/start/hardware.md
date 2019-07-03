# Оборудование

К настоящему моменту Вы должны уже ознакомиться с инструментами и процессом
разработки. В этом разделе мы перейдем к настоящему оборудованию; процесс
по большей части останется тем же самым. Давайте углубляться.

## Познай свое устройство

Перед тем, как начнем, Вам нужно определить некоторые характеристики целевого
устройства, потому что они будут использованы при настройке проекта:

- Ядро ARM. Например Cortex-M3.

- Включает ли ядро ARM FPU? Ядра Cortex-M4**F** и Cortex-M7**F** включают.

- Кокой объем памяти Flash и RAM имеется в целевом устройстве? Например 256 KiB
  Flash и 32 KiB RAM.

- Где память Flash и RAM отражаются в адресном пространстве? Например RAM
  обычно расположена по адресу `0x2000_0000`.

Вы можете найти информацию в data sheet или reference manual Вашего устройства.

В этом разделе мы будем пользоваться нашим эталонным оборудованием, STM32F3DISCOVERY.
Эта плата состоит из микроконтроллера STM32F303VCT6. У этого микроконтроллера в наличии:

- Ядро Cortex-M4F, включающее FPU одинарной точности

- 256 KiB Flash, расположенного по адресу 0x0800_0000.

- 40 KiB RAM, расположенного по адресу 0x2000_0000. (Есть еще один регион RAM, но для
  простоты мы будем его игнорировать).

## Настройка

Мы начнем все сначала со свежего шаблонного примера. Перейдите на
[предыдущий раздел по QEMU][previous section on QEMU],
чтобы вспомнить как это делается без `cargo-generate`.

[previous section on QEMU]: qemu.md

``` console
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

 $ cd app
```

Шаг первый - установить цель компиляции по умолчанию `.cargo/config`.

``` console
$ tail -n5 .cargo/config
```

``` toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

Мы будем использовать `thumbv7em-none-eabihf`, так как она охватывает
ядро Cortex-M4F.

Шаг второй - ввести информацию о регионе памяти в файл `memory.x`.

``` console
$ cat memory.x
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```

Убедитесь, что вызов `debug::exit()` закомментирован или удален, он
используется только для запуска на QEMU.

```rust,ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();
    // exit QEMU
    // ПРИМЕЧАНИЕ не выполняйте это на оборудовании; это может повредить состояние OpenOCD
    // debug::exit(debug::EXIT_SUCCESS);
    loop {}
}
```

Теперь Вы можете скомпилировать программу с помощью
`cargo build` и изучить бинарные файлы с помощью `cargo-binutils`,
как делали ранее. Крейт `cortex-m-rt` делает всю магию, нужную для
запуска Вашего чипа благодаря тому, что почти все ЦПУ Cortex-M загружаются одинаково.

``` console
$ cargo build --example hello
```

## Отладка

Отладка будет выглядеть несколько иначе. На самом деле первые шаги
могут быть разными в зависимости от целевого устройства. В этот разделе
мы увидим шаги, требуемые для отладки программы, запускаемой на
STM32F3DISCOVERY. Это, разумеется, рекомендация; с устройство-специфичной
информацией по отладке можно ознакомиться в
[Дебагономиконе](https://github.com/rust-embedded/debugonomicon).

Как и ранее, мы будем производить отладку удаленно, и клиентом будет
процесс. Но в этот раз сервером будет OpenOCD.

Подключите плату discovery к Вашему ноутбуку / ПК, как делали в разделе
[Проверка установки][verify] и проверьте, что ST-LINK header is populated.

[verify]: ../intro/install/verify.md

В терминале запустите `openocd`, чтобы подключиться к ST-LINK на плате
discovery. Выполняйте эту команду из корня шаблона;
`openocd` подхватит файл конфигурации `openocd.cfg`, который показывает,
какие интерфейсный и целевой файлы использовать.

``` console
$ cat openocd.cfg
```

``` text
# Пример конфигурации OpenOCD для отладочной платы STM32F3DISCOVERY

# В зависимости от ревизии Вашего устройства, Вам нужно выбрать ОДИН из
# интерфейсов. В любое время может быть раскомментирован только один интерфейс.

# Ревизия C (более новая)
source [find interface/stlink-v2-1.cfg]

# Ревизии A и B (старые)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **ПРИМЕЧАНИЕ** Если Вы обнаружили, что у Вас старая ревизия платы discovery,
> когда проходили раздел [проверки][verify], Вам нужно изменить файл `openocd.cfg`
> сейчас и использовать `interface/stlink-v2.cfg`.

``` console
$ openocd
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

В другом терминале запустите GDB также из корня шаблона.

``` console
$ <gdb> -q target/thumbv7em-none-eabihf/debug/examples/hello
```

Далее подключите GDB к OpenOCD, который ждет TCP-соединения на порту 3333.

``` console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

Далее *зашьем* (загрузим) программу в микроконтроллер командой `load`.

``` console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
```

Теперь программа загружена. Эта программа использует semihosting, поэтому
перед тем, как мы что-нибудь сделаем какой-либо запрос по интерфейсу
semihosting, мы должны сказать OpenOCD его включить. Вы можете отправлять
команды OpenOCD с помощью команды `monitor`.

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

> Чтобы увидеть все команды OpenOCD, введите команду `monitor help`.

Как и ранее мы можем пропустить весь путь к `main` перейдя к точке останова
командой `continue`.

``` console
(gdb) break main
Breakpoint 1 at 0x8000d18: file examples/hello.rs, line 15.

(gdb) continue
Continuing.
Note: automatically using hardware breakpoints for read-only addresses.

Breakpoint 1, main () at examples/hello.rs:15
15          let mut stdout = hio::hstdout().unwrap();
```

Перемещение по программе с помощью `next` должно произвести тот же
результат, что и ранее.

``` console
(gdb) next
16          writeln!(stdout, "Hello, world!").unwrap();

(gdb) next
19          debug::exit(debug::EXIT_SUCCESS);
```

Сейчас Вы должны видеть строку "Hello, world!", напечатанную в консоли OpenOCD,
не считая остальных.

``` console
$ openocd
(..)
Info : halted: PC: 0x08000e6c
Hello, world!
Info : halted: PC: 0x08000d62
Info : halted: PC: 0x08000d64
Info : halted: PC: 0x08000d66
Info : halted: PC: 0x08000d6a
Info : halted: PC: 0x08000a0c
Info : halted: PC: 0x08000d70
Info : halted: PC: 0x08000d72
```

Выполнение еще одного `next` заставит процессор выполнить `debug::exit`.
Это работает как точка останова и остановит процесс:

``` console
(gdb) next

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0800141a in __syscall ()
```

Это также вызывает вызовет вывод в консоли OpenOCD таких строк:

``` console
$ openocd
(..)
Info : halted: PC: 0x08001188
semihosting: *** application exited ***
Warn : target not halted
Warn : target not halted
target halted due to breakpoint, current mode: Thread
xPSR: 0x21000000 pc: 0x08000d76 msp: 0x20009fc0, semihosting
```

Однако работа микроконтроллера не прервана, и Вы можете его
возобновить с помощью `continue` или другой похожей команды.

Теперь можно выйти из GDB с помощью команды `quit`.

``` console
(gdb) quit
```

Теперь отладка требует немного больше шагов, поэтому мы упаковали их все
в один GDB-скрипт, назвав его `openocd.gdb`.

``` console
$ cat openocd.gdb
```

``` text
target remote :3333

# print demangled symbols
set print asm-demangle on

# определяет исключения без обработчиков, hard faults и паники
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# запускает процесс и сражу же останавливает процессор
stepi
```

Теперь запуск `<gdb> -x openocd.gdb $program` сразу же подключит GDB к
OpenOCD, включит semihosting, загрузит программу и запустит процесс.

В качестве альтернативы, Вы можете сделать `<gdb> -x openocd.gdb`
пользовательским пускателем и заставить
`cargo run` собирать программу *и* запускать сессию GDB.
Этот пускатель включен в `.cargo/config`, но закомментирован.

``` console
$ head -n10 .cargo/config
```

``` toml
[target.thumbv7m-none-eabi]
# раскомментируйте эту строку, чтобы заставить `cargo run` выполнять программы на QEMU
# runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
# раскомментируйте ОДНУ из этих трех строк чтобы заставить `cargo run` запускать  сессию GDB
# какую из строк включать зависит от вашей системы
runner = "arm-none-eabi-gdb -x openocd.gdb"
# runner = "gdb-multiarch -x openocd.gdb"
# runner = "gdb -x openocd.gdb"
```

``` console
$ cargo run --example hello
(..)
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
(gdb)
```
