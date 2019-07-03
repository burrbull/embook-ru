# Linux

Здесь находятся команды установки для некоторых дистрибутивов Linux.

## Пакеты

- Ubuntu 18.04 или новее / Debian stretch или новее

> **ПРИМЕЧАНИЕ** `gdb-multiarch` - команда GDB, которую Вы будете использовать для
> отладки программ на ARM Cortex-M

<!-- Debian stretch -->
<!-- GDB 7.12 -->
<!-- OpenOCD 0.9.0 -->
<!-- QEMU 2.8.1 -->

<!-- Ubuntu 18.04 -->
<!-- GDB 8.1 -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.11.1 -->

``` console
sudo apt install gdb-multiarch openocd qemu-system-arm
```

- Ubuntu 14.04 и 16.04

> **ПРИМЕЧАНИЕ** `arm-none-eabi-gdb` - команда GDB с помощью которой
Вы будете отлаживать программы на ARM Cortex-M

<!-- Ubuntu 14.04 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.7.0 (?) -->
<!-- QEMU 2.0.0 (?) -->

``` console
sudo apt install gdb-arm-none-eabi openocd qemu-system-arm
```

- Fedora 27 или новее

> **ПРИМЕЧАНИЕ** `arm-none-eabi-gdb` - команда GDB, которую Вы будете использовать для
> отладки программ на ARM Cortex-M

<!-- Fedora 27 -->
<!-- GDB 7.6 (!) -->
<!-- OpenOCD 0.10.0 -->
<!-- QEMU 2.10.2 -->

``` console
sudo dnf install arm-none-eabi-gdb openocd qemu-system-arm
```

- Arch Linux

> **ПРИМЕЧАНИЕ** `arm-none-eabi-gdb` - команда GDB, которую Вы будете использовать для
> отладки программ на ARM Cortex-M

``` console
sudo pacman -S arm-none-eabi-gdb qemu-arch-extra openocd
```

## Правила udev

Эти правила позволяют использовать OpenOCD с платой Discovery без root прав.

Создайте файл `/etc/udev/rules.d/70-st-link.rules` с содержимым как показано ниже.

``` text
# STM32F3DISCOVERY rev A/B - ST-LINK/V2
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", TAG+="uaccess"

# STM32F3DISCOVERY rev C+ - ST-LINK/V2-1
ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", TAG+="uaccess"
```

Затем перезагрузите все правила udev с помощью:

``` console
sudo udevadm control --reload-rules
```

Если у Вас плата подключена к ноутбуку, отключите ее и затем включите снова.

Вы можете провери права доступа запустив эту команду:

``` console
lsusb
```

Которая должна вывести что-то подобное:

```text
(..)
Bus 001 Device 018: ID 0483:374b STMicroelectronics ST-LINK/V2.1
(..)
```

Запишите номера шины и устройства. Используйте эти номера, чтобы создать пути вида
`/dev/bus/usb/<bus>/<device>`. Затем используйте из так:

``` console
$ # the format of the path is /dev/bus/usb/<bus>/<device>
ls -l /dev/bus/usb/001/018
```

```text
crw-------+ 1 root root 189, 17 Sep 13 12:34 /dev/bus/usb/001/018
```

```console
getfacl /dev/bus/usb/001/018 | grep user
```

```text
user::rw-
user:you:rw-
```

Знак `+` после прав означает присутствие расширенного доступа.
Команда `getfacl` говорит, что пользователь `you` может пользоваться устройством.

Теперь переходите к [следующему разделу].

[следующему разделу]: verify.md
