# Встречайте оборудование

Давайте познакомимся с оборудованием, с которым будем работать.

## STM32F3DISCOVERY (или "F3")

<p align="center">
<img title="F3" src="../assets/f3.jpg">
</p>

Мы будем называит эту плату "F3" в этой книге.

Что эта плата содержит?

- Микроконтроллер [STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html).
  В этом микроконтроллере есть
  - одноядерный процессор ARM Cortex-M4F с аппаратной поддержкой работы с
    вещественными числами одинарной точности и максимальной тактовой частотой 72 МГц;

  - 256 КиБ "Flash" памяти. (1 КиБ = 10**24** байт);

  - 48 КиБ ОЗУ;

  - множество встроенных периферийных устройств, такие как таймеры, I2C, SPI и USART;

  - входы/выходы общего назначения (GPIO) и другие типы "пинов", которые выведены на две "гребенки" по бокам.

  - интерфейс USB, доступный через порт USB, помеченный как "USB USER".

- [Акселерометр](https://en.wikipedia.org/wiki/Accelerometer) как часть чипа [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html).

- [Магнитометр](https://en.wikipedia.org/wiki/Magnetometer) как часть чипа  [LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html).

- [Гироскоп](https://en.wikipedia.org/wiki/Gyroscope) как часть чипа [L3GD20](https://www.pololu.com/file/0J563/L3GD20.pdf).

- 8 пользовательских светодиодов, расположенных в форме компаса.

- Второй микроконтроллер: [STM32F103](https://www.st.com/en/microcontrollers/stm32f103cb.html).
Этот микроконтроллер на самом деле часть размещенного на плате программатора-отладчика, называемого ST-LINK, к которому можно подключиться по порту USB, помеченному как "USB ST-LINK".

Предупреждение: будьте осторожны, если хотите подавать внешние сигналы на плату.
Пины микроконтроллера STM32F303VCT6 имеют номинальное напряжение 3.3 В.
Для дополнительной информации изучите [раздел 6.2 Absolute maximum ratings руководства](https://www.st.com/resource/en/datasheet/stm32f303vc.pdf)
