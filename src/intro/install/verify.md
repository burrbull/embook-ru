# Проверка правильности установки

В этом разделе мы проверим, что некоторые из требуемых инструментов/драйверов
правильно установлены и сконфигурированы.

Подсоедините Ваш ноутбук / ПК к плате discovery с помощью микро-USB кабеля.
На плате discovery есть два гнезда USB; используйте то, которое называется "USB ST-LINK" и
расположено поцентру края платы.

Также проверьте, что гребенка ST-LINK is populated. Смотрите на картинке
ниже; гребенка ST-LINK обведена красным.

<p align="center">
<img title="Connected discovery board" src="../../assets/verify.jpeg">
</p>

Теперь запустите следующую команду:

``` console
$ openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```

Вы должны получить следующий вывод и пограмма должна заблокировать консоль:

``` text
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
Info : Target voltage: 2.919881
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

Содержимое может не совпадать в точности, но Вы должны получить последнюю строку
о точках останова и наблюдения. Если Вы получили её, остановите процесс OpenOCD 
и переходите к [следующему разделу][next section].

[next section]: ../hardware.md

Если Вы не получили строку "breakpoints", тогда попробуйте следующую команду.

``` console
$ openocd -f interface/stlink-v2.cfg -f target/stm32f3x.cfg
```

Если команда работает, это значит у Вас старая версия платы
discovery. Это не должно стать проблемой, но учитывайте этот факт, потому что
Вам нужно будет настраивать вещи в дальнейшем несколько иначе. Можете переходить к
[следующему разделу][next section].

Если ни одна из команд не работает для обычного пользователя, попробуйте запустить их с
правами root (например `sudo openocd ..`). Если команды не работают и с правами root,
проверьте, что [правила udev][udev rules] настроены правильно.

[udev rules]: linux.md#udev-rules

Если Вы дошли сюда, а OpenOCD не работает, пожалуйста сообщите о [проблеме][an issue]
и мы Вам поможем!

[an issue]: https://github.com/rust-embedded/book/issues
