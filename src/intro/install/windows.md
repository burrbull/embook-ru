# Windows

## `arm-none-eabi-gdb`

ARM предоставляет `.exe` инсталляторы для Windows. Скачайте один [отсюда][gcc], и следуйте инструкциям.
До того как установка завершится отметьте/выберите опцию "Add path to environment variable".
Затем проверьте, что инструменты в Вашей переменной `%PATH%`:

``` console
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads

## OpenOCD

Официального релиза OpenOCD для Windows нет, но есть неофициальные, они доступны [здесь][openocd].
Скачайте zip-файл 0.10.x и распакуйте его где-нибудь на диске (я рекомендую `C:\OpenOCD` but with the drive letter that makes sense to you), а затем обновите Вашу переменную среды `%PATH%`, чтобы
включить следующий петь: `C:\OpenOCD\bin` (или путь используемый Вами).

[openocd]: https://github.com/gnu-mcu-eclipse/openocd/releases

Проверьте, что OpenOCD в Вашей переменной пути `%PATH%` с помощью:

``` console
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```

## QEMU

Скачайте QEMU с [официального веб-сайта][qemu].

[qemu]: https://www.qemu.org/download/#windows

## Драйвер ST-LINK USB

Вам также понадобится [этот USB драйвер][usb-driver] иначе OpenOCD не заработает. Следуйте
инструкциям по установке, убедитесь, что установили правильную версию (32-битную или 64-битную) драйвера.

[usb-driver]: http://www.st.com/en/embedded-software/stsw-link009.html

Всё! Переходите к [следующему разделу].

[следующему разделу]: verify.md
