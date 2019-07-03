# Установка инструментов

Эта страница содержит инструкции по установке нескольких инструментов, не зависящие от ОС:

### Rust Toolchain

Установите rustup следуя инструкциям из [https://rustup.rs](https://rustup.rs).

Затем переключитесь на beta канал.

``` console
$ rustup default beta
```

**ПРИМЕЧАНИЕ** Убедитесь, что используете компилятор версии `1.31` или новее. `rustc
-V` должен вернуть дату новее указанной ниже.

``` console
$ rustc -V
rustc 1.31.1 (b6c32da9b 2018-12-18)
```

Для ускорения загрузки и снижения занимаемого места на диске, по умолчанию
поддерживается только нативная компиляция. Чтобы добавить поддержку
кросс-компиляции для требуемой архитектуры, выберите одну из следующих целей
компиляции. Для платы STM32F3DISCOVERY, используемой в примерах данной книги,
необходимо наличие цели `thumbv7em-none-eabihf`.

Cortex-M0, M0+, and M1 (архитектура ARMv6-M):
``` console
$ rustup target add thumbv6m-none-eabi
```

Cortex-M3 (архитектура ARMv7-M):
``` console
$ rustup target add thumbv7m-none-eabi
```

Cortex-M4 и M7 без аппаратной поддержки плавающей запятой (FPU) (архитектура ARMv7E-M):
``` console
$ rustup target add thumbv7em-none-eabi
```

Cortex-M4F и M7F с модулем FPU (ARMv7E-M architecture):
``` console
$ rustup target add thumbv7em-none-eabihf
```

### ОС-специфичные инструкции

Далее следуйте инструкциям, специфичным для ОС, которую используете:

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)
