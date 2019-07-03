# Semihosting

Semihosting - механизм, позволяющий встраиваем устройствам производить операции
ввода/вывода с хостом и чаще всего используется для передачи состояния устройства
в консоль хоста. Semihosting требует нахождения в состоянии отладочной сессии,
и практически больше ничего (никаких дополнительных проводов!), поэтому очень
удобен в работе. Недостаток его в том, что он чрезвычайно медленный:
каждая операция записи может занять несколько милисекунд в зависимости от
используемого программатора (например ST-Link).

Крейт [`cortex-m-semihosting`] предоставляет API для работы с операциями semihosting
на устройствах Cortex-M. В программе ниже показано как вывести "Hello, world!"
через semihosting:

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

```rust,ignore
#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

Если Вы запустите эту программу на оборудованиии, то увидите сообщение "Hello, world!"
в логах OpenOCD.

``` console
$ openocd
(..)
Hello, world!
(..)
```

Но сначала Вам нужно включить semihosting в OpenOCD из GDB:
``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU понимает операции semihosting, поэтому программа выше будет также работать
и в `qemu-system-arm` без необходимости запускать отладочную сессию. Заметьте, что
Вам нужно будет передать флаг `-semihosting-config`, чтобы QEMU включил поддержку semihosting;
эти флаги также уже включены в файле `.cargo/config` шаблона.

``` console
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

В semihosting также есть операция `exit`, которую можно использовать для завершения
работы QEMU. Важно: **не** используйте `debug::exit` на реальном оборудовании;
эта функция может сломать сессию OpenOCD, и Вы не сможете продолжить отладку программ до перезапуска.

```rust,ignore
#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

И последний совет: Вы можете установить поведение типа паника для `exit(EXIT_FAILURE)`.
Это позволит Вам писать `no_std` тесты на успешность выполнения и запускать их на QEMU.

Для удобства, в крейте `panic-semihosting` есть опция "exit", которая если включена будет
вызывать `exit(EXIT_FAILURE)` после распечатывания сообщения о панике в stderr хоста.

```rust,ignore
#![no_main]
#![no_std]

extern crate panic_semihosting; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```
