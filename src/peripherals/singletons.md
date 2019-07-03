# Одиночки

> В программной инжинерии, одиночка - шаблон проектирования программ, ограничивающий использование класса одним экземпляром.
>
> *Wikipedia: [Singleton Pattern]*

[Singleton Pattern]: https://en.wikipedia.org/wiki/Singleton_pattern


## Но почему мы не можем просто использовать глобальную(ые) переменную(ые)?

Мы могли бы делать все на публичных static'ах, как-то так:

```rust,ignore
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```

Но здесь есть несколько проблем. Это глобальная изменяемая переменная, а в Rust, взаимодействие с ними - всегда unsafe. Такие переменные к тому же в области сидимости всей программы, а это значит, что borrow checker не
может отследить ссылки и владение этих переменных.

## Как мы это делаем в Rust?

Вместо того, чтобы делать нашу периферию глобальной переменной, мы могли бы решить создать
глобальную переменную, назовем её `PERIPHERALS`, которая содержит `Option<T>` для каждого
из наших периферийных устройств

```rust,ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

Эта структура позволяет нам получить единственный экземпляр нашей периферии. Ели мы попытаемся вызвать
`take_serial()` более чем единожды, в нашем коде будет паника!

```rust,ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // This panics!
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

Несмотря на то, что взаимодествие с этой структурой `unsafe`, как только мы получим `SerialPort`, содержащийся в ней, нам
больше не нужно будет использовать `unsafe`, или структуру `PERIPHERALS`.

У такого подхода есть небольшие накладные расходы времени исполнения потому, что нам нужно
оборачивать структуру `SerialPort` в option, и нам нужно один раз вызвать `take_serial()`,
однако эта маленькая стоимость, уплаченная заранее, позволяет нам использовать мощь контроля
владения по всей остальной части программы.

## Поддержка существующих библиотек

Несмотря на то, что выше мы создавали нашу собственную структуру `Peripherals`, Вам не нужно
делать это в вашей программе. Крейт `cortex_m` содержит макрос `singleton!()`, который возьмет эту работу на себя.

```rust,ignore
#[macro_use(singleton)]
extern crate cortex_m;

fn main() {
    // OK если `main` выполяняется только раз
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

К тому же, если Вы используете `cortex-m-rtfm`, весь процесс определения и получения периферии во владение
абстрагировано от Вас, вместо этого вы орудуете структурой `Peripherals`, содержащей не-`Option<T>` версии
всех полей, которые Вы определите.

```rust,ignore
// cortex-m-rtfm v0.3.x
app! {
    resources: {
        static RX: Rx<USART1>;
        static TX: Tx<USART1>;
    }
}
fn init(p: init::Peripherals) -> init::LateResources {
    // Заметьте, что теперь мы владеем значением, а не ссылкой
    let usart1: USART1 = p.device.USART1;
}
```

[japaric.io rtfm v3](https://blog.japaric.io/rtfm-v3/)

## Но зачем?

Но есть ли ощутимая разница в том, как код Rust работает с одиночками и без?

```rust,ignore
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ Это очень, очень важно
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```

Здесь есть два важных фактора:

* Т.к. мы используем одиночки, есть только один способ или место получить владение структурой `SerialPort`
* Вызывая метод `read_speed()`, мы должны владеть структурой `SerialPort` или ссылкой на неё

Эти два фактора, взятые вместе, значат что есть только один возможный способ доступа к оборудованию - удовлетворить контроль владения, т.е. нельзя получить несколько мутабельных ссылок на одно и то же оборудование!

```rust,ignore
fn main() {
    // отсутствует ссылка на `self`! Не заработает.
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // Вы можете читать только то, к чему получили доступ
    let _ = serial_1.read_speed();
}
```

## Относитесь к оборудованию как к данным

Ктоме того, т.к. некоторые ссылки изменяемые, а другие неизменяемые, становится очевидным, какая
функция/метод может потенциально изменять состояние оборудования. Например,

Эта функция позволяет изменять настройки оборудования:

```rust,ignore
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

А эта нет:

```rust,ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

Это позволяет нам контролировать, может ли код влиять на оборудование на **этапе компиляции**, а не
во время исполнения. Отметим, что в общем случае это работает для одного приложения, но во встраиваемых системах,
код компилируется в одну программу, поэтому обычно это не является ограничением.
