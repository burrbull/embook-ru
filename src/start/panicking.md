# Паника

Паника - часть ядра языка Rust. Для безопасности памяти встроенные операции, такие как индексация,
проверяются в рантайме. Когда происходит выход за границы массива, происходит паника.

В стандартной библиотеке для паники определено поведение по умолчанию:
происходит раскрутка стека паникующего потока, если только пользователь не установил,
что при панике нужно аварийно завершить программу.

В программах без стандартной библиотеки, однако, поведение при панике не определено.
Поведение можно выбрать определив функцию `#[panic_handler]`.
Эта функция должна встречаться в графе зависимостей программы ровно *один раз*,
и должна иметь следующую сигнатуру: `fn(&PanicInfo) -> !`, где [`PanicInfo`] -
структура, содержащая информацию о местонахождении паники.

[`PanicInfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

Принимая во внимание, что встраимваемые системы варьируются от user facing to safety critical
(аварии недопустимы) there's no one size fits all panicking behavior, но есть много
общепринятых поведений. Эти общие поведения были упакованы в крейты, которые определяют
фунцию `#[panic_handler]`. Вот некоторые из примеров:

- [`panic-abort`]. Паника вызывает прерывание выполняемой инструкции.
- [`panic-halt`]. Паника вызывает остановку программы, или текущего потока, и входит в бесконечный цикл.
- [`panic-itm`]. Сообщение о панике передается по ITM, специальной периферии ARM Cortex-M.
- [`panic-semihosting`]. Сообщение о панике передается на хост с помощью техники semihosting.

[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

Вы можете найти и другие подобные крейты, произведя поиск по ключевому слову [`panic-handler`]
на crates.io.

[`panic-handler`]: https://crates.io/keywords/panic-handler

Программа может задействовать одно из таких поведений просто линкуясь с соответствующим
крейтом. То, что поведение при панике описано в исходном коде программы одной строкой
полезно не только как документация, но также может быть использовано, чтобы менять
поведение при панике в соответствии с профилем компиляции. Например:

``` rust,ignore
#![no_main]
#![no_std]

// профиль dev: легче отлаживать панику; можно установить точку останова на `rust_begin_unwind`
#[cfg(debug_assertions)]
extern crate panic_halt;

// профиль release: минимизирует бинарный размер программы
#[cfg(not(debug_assertions))]
extern crate panic_abort;

// ..
```

В этом примере крейт линкуется с крейтом `panic-halt` если собирается с профилем dev
(`cargo build`), и линкуется с крейтом `panic-abort` когда собирается с профилем release
(`cargo build --release`).

## Пример

Вот пример, в котором мы пытаемся получить элемент за границами массива. Операция приводит
к панике.

```rust,ignore
#![no_main]
#![no_std]

extern crate panic_semihosting;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len() + 1;
    let _y = xs[i]; // за границами массива

    loop {}
}
```

В этом примере выбрано поведение `panic-semihosting`, которое печатает сообщение о
панике в консоль хоста с помощью semihosting.

``` console
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

Вы можете изменить поведение на `panic-halt` и убедиться, что в этом случае сообщение не печатается.
