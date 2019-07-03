# Первая попытка

## Регистры

Давайте взглянем на периерию 'SysTick' - простой таймер, пристствующий в каждом ядре процессора Cortex-M.
Обычно Вы можете найти его описание в даташите производителя чипа или *Техническом справочном руководстве*,
но этот пример общий для всех ядер ARM Cortex-M, давайте взглянем в [ARM reference manual].
Там мы видим четыре регистра:

[ARM reference manual]: http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

| Сдвиг  | Имя         | Описание                    | Ширина |
|--------|-------------|-----------------------------|--------|
| 0x00   | SYST_CSR    | Control and Status Register | 32 bits|
| 0x04   | SYST_RVR    | Reload Value Register       | 32 bits|
| 0x08   | SYST_CVR    | Current Value Register      | 32 bits|
| 0x0C   | SYST_CALIB  | Calibration Value Register  | 32 bits|

## Метод в стиле C

В Rust, мы можем представить коллекцию регистров точно токим же способом как мы делаем это в C - структурой (`struct`).

```rust,ignore
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```

Спецификатор `#[repr(C)]` говорит компилятору Rust располагать эту структуру в памяти также,
как это сделал бы компилятор C. Это очень важно, так как Rust позволяет полям структуры менять порядок,
тогда как C не позволяет. Можете себе представить как бы происходила отладка, если бы эти поля
были незаметно перемешаны компилятором! С наличием этого спецификатора, наши 4 32-битных поля
соответствуют таблице серху. Но, конечно, эта структура бессмыслена сама по себе - нам нужна переменная.

```rust,ignore
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## Volatile операции доступа

Пока что есть ряд проблем с методом, описанным выше.

1. Мы должны использовать unsafe каждый раз, когда хотим получить доступ к Периферии.
2. У нас нет способа определить, какие регистры имеют доступ только для чтения, а какие только для записи.
3. Любой кусок кода где-либо в Вашей программе может получить доступ к оборудованию по этой структуре.
4. Но что более важно, это на самом деле не работает...

Теперь проблема в том, что компиляторы умные. Если у Вас идет две записи одна за другой в один
и тот же кусок ОЗУ, компилятор может просто полностью пропустить первую запись.
В C мы можем пометить переменные как `volatile`, чтобы убедиться, что каждое чтение или запись
там, где должны быть. В Rust мы наоборот помечаем *операции доступа* как volatile, а не переменные.

```rust,ignore
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```

Чтож, мы исправили одну из четырех проблем, но теперь у нас еще больше `unsafe` кода!
К счастью, есть сторонний крейт, который может помочь - [`volatile_register`].

[`volatile_register`]: https://crates.io/crates/volatile_register

```rust,ignore
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

Теперь volatile операции доступа выполняются автоматически методами `read` и `write`.
Выполнение записи все еще `unsafe`, но справедливости ради, оборудование - это набор изменяемых
состояний и компилятор не может знать какие операции записи безопасны поэтому
это хорошее поведение по умолчанию.

## Обертка в стиле Rust

Нам нужно обернуть эту `struct` в высокоуровневое API, безопасное для вызова пользователем.
Как авторы драйверов, мы вручную проеряем, что небезопасный код корректен, а затем
представляем безопасное API для наших пользователей, поэтому они не должны беспокоиться об этом
(при условии, что они доверяют нам сделать это правильно!).

Один из возможных примеров:

```rust,ignore
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

Теперь проблема данного метода в том, что следующий код полностью приемлем компилятором:

```rust,ignore
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

Наш аргумент `&mut self` для функции `set_reload` проверяет, что нет других ссылок на *ту* конкретную
структуру `SystemTimer`, но не останавливает пользователя от создания второго `SystemTimer`,
который указывает на ту же периферию! Код, написанный в такой манере будет работать, если автор
достаточно пункцтуален, чтобы обнаружить все такие 'дублирующие' экземпляры драйвера, но как только
код разбивается на множество модулей, драйверов, разработчиков и дней, становится все легче и легче
допускать такого рода ошибки.
