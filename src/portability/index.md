# Переносимость

В эмбеддед окружении переносимость очень важное свойство: Каждый производитель и даже каждое семейство одного производства
предлагает различные периферийные устройства и возможности, способы взаимодействия с периферийными устройствами также будут отличаться.

Распространенный способ выравнивания таких различий - через уровень аппаратной абстракции (Hardware Abstraction layer) или **HAL**.

> Аппаратные абстракции - набор операций, которые имитируют некоторые специфичные для платформы детали, предоставляя программам прямой доступ к аппаратным ресурсам.
>
> Они часто позволяют программистам писать аппаратнонезависимые, высокоэффективные приложения предоставляя стандартные вызовы операционной системы (ОС) оборудованию.
>
> *Wikipedia: [Hardware Abstraction Layer]*

[Hardware Abstraction Layer]: https://en.wikipedia.org/wiki/Hardware_abstraction

Встраиваиваемые системы - немного особенные в этом смысле, т.к. обычно не имеют операционной системы и пользовательского ПО, а имеют фирменные прошивки,
компилируемые целиком, а также ряд других ограничений. Таким образом, хотя традиционный подход, определенный в Википедии, потенциально может работать,
это, вероятно, не самый продуктивный подход для обеспечения переносимости.

Как мы боремся с этим в Rust? Введите **embedded-hal**...

## Что такое embedded-hal?

В двух словах, это набор типажей, которые определяют соглашения как реализовывать взаимодействие между **реализациями HAL**, **драйверами** и **приложениями (или прошивками)**.
Эти соглашения включают как характеристики (например, если типаж реализован для конкретного типа, **реализация HAL** предоставляет определенную характеристику),
так и методы (например, если Вы можете сконструировать тип, реализующий типаж, это гарантирует, что вам доступны методы, определенные в типаже).

Типичные уровни выглядят так:

![](../assets/rust_layers.svg)

Некоторые из типажей, определенных в **embedded-hal**:
* GPIO (пины ввода/вывода)
* Взаимодействие по последовательному интерфейсу
* I2C
* SPI
* Таймеры/Счетчики
* АЦП

Главная причина сущестования типажей из **embedded-hal** и крейтов, их реализующих и использующих - контроль сложности.
Если вы считаете, что приложению может потребоваться реализовать использование периферии в оборудовании, также
как приложениям и потенциально драйверам дополнительных аппаратных компонентов, то обнаружите, что переиспользование
очень ограничено. Выражаясь математически, если **M** - это количество HAL реализаций периферии, а **N** - количество драйверов,
тогда если мы переизобретаем колесо для каждого приложения, то прийдем к **M*N** реализаций, в то время когда использование *API*,
предоставлямого типажами **embedded-hal** даст сложность реализации **M+N**. Конечно есть и дополнительные преимущества,
такие как меньшая число проб и ошибок благодаря хорошо структурированному и готовому API.

## Пользователи embedded-hal

Как сказано выше, есть три главных пользователя HAL:

### Реализация HAL

Реализации HAL предлагают способ взаимодействия между оборудованием и пользователями типажей HAL. Обычно реализации сосотоят из трех частей:
* Один или более аппаратно-специфичных типа
* Функции для создания и инициализации этих типов, часто предоставляющие различные опции настройки (скорость, режим работы, используемые пины и т.д.)
* один или более реализаций (`impl`) типажей (`trait`) из числа типажей **embedded-hal** для типа

Такие **реализации HAL** могут быть разного рода:
* Через низкоуровневый доступ к оборудованию, например через регистры
* Через операционную систему, например используя `sysfs` под Linux
* Через адаптер, например фиктивные типы для юнит-тестирования
* Через драйвер аппаратных адаптеров, например концентратор I2C или расширитель GPIO

### Драйвер

Драйвер реализует набор пользовательских функций для внутренних и внешних компонентов, подключаемых к периферии, реализующей типажи embedded-hal.
Примеры таких драйверов обычно включают различные датчики (температурный, магнитометр, акселерометр, световой), дисплеи (светодиодные матрицы, LCD-дисплеи)
и исполнительные механизмы (двигатели, передатчики).

Драйвер должен быть инициализирован экземпляром типа, реализующим определенный `типаж` из числа embedded-hal, который удовлетворяет ограничениям типажа
и предоставляет собственный экземпляр типа с пользовательским набором методов, позволяющим взаимодействовать с управляемым устройством.

### Приложение

В приложении объединяются различные куски вместе и проверяется наличие требуемой функциональности. Когда происходит портирование на другую систему,
это та часть, которая требует наибольших усилий по адаптации, т.к. программа должа корректно инициализировать реальное оборудование с помощью
реализаций HAL, а инициализация другого оборудования отличаестся, иногда коренным образом. А часто выбор пользователя играет большую роль,
т.к. компоненты могут быть физически подключены к другим выводам, шины оборудования иногда требуют дополнительных устройств, чтобы удовлетворить
настройкам, или существуют различные компромиссы при использовании внутренних периферийных устройств (например, доступно несколько таймеров
с различными возможностями, или периферийные устройства конфликтуют с другими).