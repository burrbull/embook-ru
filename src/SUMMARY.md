# Summary

<!--

Definition of the organization of this book is still a work in process.

Refer to https://github.com/rust-embedded/book/issues for
more information and coordination

-->

- [Введение](./intro/index.md)
    - [Оборудование](./intro/hardware.md)
    - [`no_std`](./intro/no-std.md)
    - [Инструменты](./intro/tooling.md)
    - [Установка](./intro/install.md)
        - [Linux](./intro/install/linux.md)
        - [MacOS](./intro/install/macos.md)
        - [Windows](./intro/install/windows.md)
        - [Проверка установки](./intro/install/verify.md)
- [Начало работы](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [Оборудование](./start/hardware.md)
  - [Memory-mapped Registers](./start/registers.md)
  - [Semihosting](./start/semihosting.md)
  - [Паника](./start/panicking.md)
  - [Исключения](./start/exceptions.md)
  - [Прерывания](./start/interrupts.md)
  - [Ввод/вывод](./start/io.md)
- [Периферия](./peripherals/index.md)
    - [Первая попытка на Rust](./peripherals/a-first-attempt.md)
    - [Контроль владения](./peripherals/borrowck.md)
    - [Одиночки](./peripherals/singletons.md)
- [Статические гарантии](./static-guarantees/index.md)
    - [Typestate программирование](./static-guarantees/typestate-programming.md)
    - [Периферия как конечный автомат](./static-guarantees/state-machines.md)
    - [Соглашения по проектированию](./static-guarantees/design-contracts.md)
    - [Абстракции нулевой стоимости](./static-guarantees/zero-cost-abstractions.md)
- [Переносимость](./portability/index.md)
- [Параллелизм](./concurrency/index.md)
- [Коллекции](./collections/index.md)
- [Советы для embedded C разработчиков](./c-tips/index.md)
    <!-- TODO: Define Sections -->
- [Интероперабельность](./interoperability/index.md)
    - [Капелька C в Вашем Rust](./interoperability/c-with-rust.md)
    - [Капелька Rust в Вашем C](./interoperability/rust-with-c.md)
- [Unsorted topics](./unsorted/index.md)
  - [Оптимизации: компромисс скорости/размера](./unsorted/speed-vs-size.md)
