# Конкурентность

Конкурентность возникает когда разные части вашей программы запускаются в разное
время или в разном порядке. В эмбеддед контексте она включает:

* обработчики прерываний, которые запускаются в любой момент, когда происходит прерывание,
* различные формы многопоточности, когда ваш микропроцессор регулярно переключается
  между частями программы,
* и в некоторых системах с многоядерными микропроцессорами, когда каждое ядро может
  независимо запускаю разные части ваше программы одновременно.

Так как многие программы для встраиваемых систем должны работать с прерываниями, конкурентность
рано или позно возникнет, и здесь могут появиться множество трудноуловимых ошибок.
К счастью, Rust предоставляет ряд абстракций и гарантий безопасности, чтобы помочь
нам писать корректный код.

## Когда нет конкуренции

Простейшая форма конкурентности в эмбеддед программах - это когда нет конкуренции:
ваш программный код состоит из одного главного цикла, который просто продолжает выполняться,
и прерывания отсутствуют вовсе. Иногда это идеально подходит для решения проблемы!
Обычно  таких случаях ваш цикл будет читать некоторые входные данные,
выполнять некоторую обработку и записывать некоторые выходные данные.

```rust,ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```

Так как конкуренции нет, то нет нужды беспокоиться о распределении данных
между частями вашей программы или синхронизировать доступ к периферии.
Если вы можете обойтись таким простым подходом, это может стать отличным решением.

## Глобальные изменяемые данные

В отличие от неэмбеддед Rust'а, у нас обычно не будет такой роскоши, как аллокации данных в куче
и передача ссылок на эти данные во вновь созданный поток.
Вместо этого наши обработчики прерываний могут быть вызваны в любое время и
должны значть, как получить доступ к любой разделяемой памяти, которую мы используем.
На самом нижнем уровне это значит, что мы должны иметь
_статически аллоцированную_ изменяемую память, к которой могут обращаться
как обработчик прерывания, так и главный код программы.

В Rust, такие [`static mut`] переменные всегда небезопасны для чтения или записи,
потому что без специальных мер предосторожности, Вы можете вызвать состояние гонки данных,
когда Ваше обращение к переменной прерывается на полпути
прерыванием, которое также обращается к этой переменной.

[`static mut`]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

Для примера как такое поведение может вызвать трудноуловимые ошибки в коде,
рассмотрим эмбеддед программу, которая считает передные фронты входных сигналов
в каждом односекундном периоде (счетчик частоты):

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // ОПАСНОСТЬ - На самом деле не безопасно! Может вызвать гонку данных.
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

Каждую секунду прерывание таймера сбрасывает счетчик. В тоже время
главный цикл постоянно измеряет сигнал, и увеличивает счетчик, когда видит
сигнал перешел из низкого состояния в высокое. Нам нужно использовать `unsafe`,
чтобы получить доступ к `COUNTER`, так как он `static mut`, а это значит, что
мы обещаем компилятору, что не вызовем неопределенное поведение. Можете
ли Вы обнаружить гонку данных? Инкремент `COUNTER`'a _не_ гарантирует атомарность —
на самом деле в большинстве встраиваемых платформ, он будет разделен
на загрузку, собственно инкремент и сохранение данных.
Если прерывание произойдет после загрузки, но до сохранения, сброс в 0
будет проигнорирован по завершению прерывания — и мы насчитаем вдвое
больше модуляций за период.

## Критические секции

Так что мы можем сделать с гонками данных? Простой метод - это использовать
_критические секции_, контекст, в котором прерывания отключены. Оборачивая доступ к
`COUNTER` в `main` в критическую секцию, мы можем убедить прерывание таймера
не запускаться, пока мы не закончим операцию преращения `COUNTER`:

```rust,ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Новая критическая секция проверят синхронизацию доступа к COUNTER
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

В этом примере мы используем `cortex_m::interrupt::free`, но на другие платформы
будут иметь похожие механизмы для выполнения кода в критической секции.
Они также отключают прерывания, запускают какой-то код, а затем перезапускают прерывания.

Заметьте, что нам не нужно вставлять критическую секцию внутрь прерывания таймера
по двум причинам:

  * На запись 0 в `COUNTER` не влияет гонка, так как мы его не читаем
  * Прерывание никогда не будет прервано потоком `main`

Если `COUNTER` было разделено множеством обработчиков, которые могут
_вытеснять_ друг друга, тогда каждый из них также может требовать наличие критической секции.

Это решило нашу первоочередную проблему, но мы всё ещё пишет много
`unsafe` кода, о котором нужно заботиться, и мы можем использовать критические секции
без необходимости - что обходится слишком дорого, задерживая прерывания и вызывая дребезг.

Стоит отметить, что хотя критическая секция гарантирует, что прерывания не запустится,
она не дает гарантий эксклюзивности для многоядерных систем! Другое ядро может без проблем
получить доступ к той же памяти, что и ваше ядро, даже без прерываний.
Вам понадобятся более мощные примитивы синхронизации, когда вы используете несколько ядер.

## Атомарный доступ

На некоторых платформах доступны атомарные инструкции, которые предоставляют
гарантии касательно операций чтения-изменения-записи.
Что касается Cortex-M, `thumbv6` (Cortex-M0) не предоставляет атомарных инструкций,
в то время как `thumbv7` (Cortex-M3 и выше) предоставляет. Эти инструкции дают
альтернативу тяжеловесному отключению всех прерываний: когда мы будем делать инкремент,
чаще всего он будет успешным, но если он будет прерван, то автоматически повторит всю
операцию приращения. Атомарные операции безопасны даже для многоядерных систем.

```rust,ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Используем `fetch_add`, чтобы атомарно добавить 1 к COUNTER
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // Используем `store`, чтобы записать 0 напрямую в COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

Теперь `COUNTER` - безопасная `static` переменная. Благодаря типу `AtomicUsize`
`COUNTER` можно безопасно изменять как из обработчика прерывания,
так и главного потока программы без отключения прерываний.
Когда это возможно — это лучшее решение, но оно может не поддерживаться Вашей платформой.

Примечание к [`Ordering`]: данный тип влият на то как, компилятор и оборудование
могут переупорядочивает инструкции, а также влияет на видимость кэша.
Предполагая, что целью является одноядерная платформа, `Relaxed` - достаточный
и наиболее эффективный выбор в этом конкретном случае.
Более строгое упорядочивание приведет к тому, что компилятор создаст барьеры памяти
вокруг атомарных операций; в зависимости от того, для чего вы используете атомные
инструкции, для вас может это может понадобиться, а может и нет!
Подробности атомарной модели сложны и лучше описаны в других местах.

Для подробностей по атомарным операциям и упорядочиванию, смотрите [nomicon].

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]: https://doc.rust-lang.org/nomicon/atomics.html


## Абстракции, Send, и Sync

Ни одно из вышеуказанных решений не является особенно удовлетворительным.
Они требуют `unsafe` блоки, которые нужно очень внимательно проверять и неприятно выглядят.
Конечно же мы можем делать это в Rust лучше!

Мы можем абстрагировать наш счетчик в безопасный интерфейс, который можно безопасно
использовать в любом месте кода. В этом примере мы будем использовать счетчик с
критической секцией, но Вы можете сделать что-то похожее с атомарными операциями.

```rust,ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// Наш счетчик - просто обертка вокруг UnsafeCell<u32>, который сердце
// interior mutability в Rust. Используя interior mutability, мы можем сделать
// COUNTER `static` вместо `static mut`, но при этом всё ещё иметь возможность
// изменять значение счетчика.
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // Требуя передачи CriticalSection в качестве аргумента, мы убеждаемся,
        // что будем работать внутри критической секции, и поэтому можем спокойно
        // использовать unsafe блок (требуемый для вызова UnsafeCell::get).
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// Нужно для static CSCounter. Смотреть объяснение ниже.
unsafe impl Sync for CSCounter {}

// COUNTER больше не `mut`, так как использует interior mutability;
// тем не менее он таже больше не требует unsafe блоков для доступа.
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Здесь нет unsafe!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // Нам нужно войти здесь в критическую секцию, только чтобы получить валидный
    // символ cs, даже если знаем, что никакие другие прерывания 
    //  не могут вытеснить текущее.
    interrupt::free(|cs| COUNTER.reset(cs));

    // Мы можем использовать unsafe код, чтобы сгенерировать искусственную CriticalSection,
    // если нам это действительно требуется, чтобы избежать накладных расходов:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```

Мы переместилт наш `unsafe` код внутрь нашей бережно спланированной абстракции,
и теперь код нашего приложения не содержит `unsafe` блоков.

Такой дизайн требует, чтобы приложение передавало символ `CriticalSection`:
эти символы безопасно создаются только с помощью `interrupt::free`, поэтому требуя
передачи такого символа, мы убеждаемся, что работаем в критической секции, without
having to actually do the lock ourselves. This guarantee is provided statically
by the compiler: there won't be any runtime overhead associated with `cs`.
If we had multiple counters, they could all be given the same `cs`, without
requiring multiple nested critical sections.

This also brings up an important topic for concurrency in Rust: the
[`Send` and `Sync`] traits. To summarise the Rust book, a type is Send
when it can safely be moved to another thread, while it is Sync when
it can be safely shared between multiple threads. In an embedded context,
we consider interrupts to be executing in a separate thread to the application
code, so variables accessed by both an interrupt and the main code must be
Sync.

[`Send` and `Sync`]: https://doc.rust-lang.org/nomicon/send-and-sync.html

For most types in Rust, both of these traits are automatically derived for you
by the compiler. However, because `CSCounter` contains an [`UnsafeCell`], it is
not Sync, and therefore we could not make a `static CSCounter`: `static`
variables _must_ be Sync, since they can be accessed by multiple threads.

[`UnsafeCell`]: https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

To tell the compiler we have taken care that the `CSCounter` is in fact safe
to share between threads, we implement the Sync trait explicitly. As with the
previous use of critical sections, this is only safe on single-core platforms:
with multiple cores you would need to go to greater lengths to ensure safety.

## Mutexes

We've created a useful abstraction specific to our counter problem, but
there are many common abstractions used for concurrency.

One such _synchronisation primitive_ is a mutex, short for mutual exclusion.
These constructs ensure exclusive access to a variable, such as our counter. A
thread can attempt to _lock_ (or _acquire_) the mutex, and either succeeds
immediately, or blocks waiting for the lock to be acquired, or returns an error
that the mutex could not be locked. While that thread holds the lock, it is
granted access to the protected data. When the thread is done, it _unlocks_ (or
_releases_) the mutex, allowing another thread to lock it. In Rust, we would
usually implement the unlock using the [`Drop`] trait to ensure it is always
released when the mutex goes out of scope.

[`Drop`]: https://doc.rust-lang.org/core/ops/trait.Drop.html

Using a mutex with interrupt handlers can be tricky: it is not normally
acceptable for the interrupt handler to block, and it would be especially
disastrous for it to block waiting for the main thread to release a lock,
since we would then _deadlock_ (the main thread will never release the lock
because execution stays in the interrupt handler). Deadlocking is not
considered unsafe: it is possible even in safe Rust.

To avoid this behaviour entirely, we could implement a mutex which requires
a critical section to lock, just like our counter example. So long as the
critical section must last as long as the lock, we can be sure we have
exclusive access to the wrapped variable without even needing to track
the lock/unlock state of the mutex.

This is in fact done for us in the `cortex_m` crate! We could have written
our counter using it:

```rust,ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We still need to enter a critical section here to satisfy the Mutex.
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

We're now using [`Cell`], which along with its sibling `RefCell` is used to
provide safe interior mutability. We've already seen `UnsafeCell` which is
the bottom layer of interior mutability in Rust: it allows you to obtain
multiple mutable references to its value, but only with unsafe code. A `Cell`
is like an `UnsafeCell` but it provides a safe interface: it only permits
taking a copy of the current value or replacing it, not taking a reference,
and since it is not Sync, it cannot be shared between threads. These
constraints mean it's safe to use, but we couldn't use it directly in a
`static` variable as a `static` must be Sync.

[`Cell`]: https://doc.rust-lang.org/core/cell/struct.Cell.html

So why does the example above work? The `Mutex<T>` implements Sync for any
`T` which is Send — such as a `Cell`. It can do this safely because it only
gives access to its contents during a critical section. We're therefore able
to get a safe counter with no unsafe code at all!

This is great for simple types like the `u32` of our counter, but what about
more complex types which are not Copy? An extremely common example in an
embedded context is a peripheral struct, which generally are not Copy.
For that we can turn to `RefCell`.

## Sharing Peripherals

Device crates generated using `svd2rust` and similar abstractions provide
safe access to peripherals by enforcing that only one instance of the
peripheral struct can exist at a time. This ensures safety, but makes it
difficult to access a peripheral from both the main thread and an interrupt
handler.

To safely share peripheral access, we can use the `Mutex` we saw before. We'll
also need to use [`RefCell`], which uses a runtime check to ensure only one
reference to a peripheral is given out at a time. This has more overhead than
the plain `Cell`, but since we are giving out references rather than copies,
we must be sure only one exists at a time.

[`RefCell`]: https://doc.rust-lang.org/core/cell/struct.RefCell.html

Finally, we'll also have to account for somehow moving the peripheral into
the shared variable after it has been initialised in the main code. To do
this we can use the `Option` type, initialised to `None` and later set to
the instance of the peripheral.

```rust,ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // Obtain the peripheral singletons and configure it.
    // This example is from an svd2rust-generated crate, but
    // most embedded device crates will be similar.
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // Some sort of configuration function.
    // Assume it sets PA0 to an input and PA1 to an output.
    configure_gpio(gpioa);

    // Store the GPIOA in the mutex, moving it.
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // We can no longer use `gpioa` or `dp.GPIOA`, and instead have to
    // access it via the mutex.

    // Be careful to enable the interrupt only after setting MY_GPIO:
    // otherwise the interrupt might fire while it still contains None,
    // and as-written (with `unwrap()`), it would panic.
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // We'll now read state as a digital input, via the mutex
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // Set PA1 high if we've seen a rising edge on PA0.
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // This time in the interrupt we'll just clear PA0.
    interrupt::free(|cs| {
        // We can use `unwrap()` because we know the interrupt wasn't enabled
        // until after MY_GPIO was set; otherwise we should handle the potential
        // for a None value.
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

That's quite a lot to take in, so let's break down the important lines.

```rust,ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

Our shared variable is now a `Mutex` around a `RefCell` which contains an
`Option`. The `Mutex` ensures we only have access during a critical section,
and therefore makes the variable Sync, even though a plain `RefCell` would not
be Sync. The `RefCell` gives us interior mutability with references, which
we'll need to use our `GPIOA`. The `Option` lets us initialise this variable
to something empty, and only later actually move the variable in. We cannot
access the peripheral singleton statically, only at runtime, so this is
required.

```rust,ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

Inside a critical section we can call `borrow()` on the mutex, which gives us
a reference to the `RefCell`. We then call `replace()` to move our new value
into the `RefCell`.

```rust,ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

Finally we use `MY_GPIO` in a safe and concurrent fashion. The critical section
prevents the interrupt firing as usual, and lets us borrow the mutex.  The
`RefCell` then gives us an `&Option<GPIOA>`, and tracks how long it remains
borrowed - once that reference goes out of scope, the `RefCell` will be updated
to indicate it is no longer borrowed.

Since we can't move the `GPIOA` out of the `&Option`, we need to convert it to
an `&Option<&GPIOA>` with `as_ref()`, which we can finally `unwrap()` to obtain
the `&GPIOA` which lets us modify the peripheral.

If we need a mutable references to shared resources, then `borrow_mut` and `deref_mut`
should be used instead. The following code shows an example using the TIM2 timer.

```rust,ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // Some sort of timer configuration function.
    // Assume it configures the TIM2 timer, its NVIC interrupt,
    // and finally starts the timer.
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim)) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

> **NOTE**
>
> At the moment, the `cortex-m` crate hides const versions of some functions
> (including `Mutex::new()`) behind the `const-fn` feature. So you need to add
> the `const-fn` feature as a dependency for cortex-m in the Cargo.toml to make
> the above examples work:
>
> ``` toml
> [dependencies.cortex-m]
> version="0.6.0"
> features=["const-fn"]
> ```
> Meanwhile, `const-fn` has been working on stable Rust for some time now.
> So this additional switch in Cargo.toml will not be needed as soon as 
> it is enabled in `cortex-m` by default.
>

Фух! This is safe, but it is also a little unwieldy. Is there anything else
we can do?

## RTFM

One alternative is the [RTFM framework], short for Real Time For the Masses. It
enforces static priorities and tracks accesses to `static mut` variables
("resources") to statically ensure that shared resources are always accessed
safely, without requiring the overhead of always entering critical sections and
using reference counting (as in `RefCell`). This has a number of advantages such
as guaranteeing no deadlocks and giving extremely low time and memory overhead.

[RTFM framework]: https://github.com/japaric/cortex-m-rtfm

The framework also includes other features like message passing, which reduces
the need for explicit http://baibako.tv/index.php?page=1shared state, and the ability to schedule tasks to run at
a given time, which can be used to implement periodic tasks. Check out [the
documentation] for more information!

[the documentation]: https://japaric.github.io/cortex-m-rtfm/book/ru/

## Real Time Operating Systems

Another common model for embedded concurrency is the real-time operating system
(RTOS). While currently less well explored in Rust, they are widely used in
traditional embedded development. Open source examples include [FreeRTOS] and
[ChibiOS]. These RTOSs provide support for running multiple application threads
which the CPU swaps between, either when the threads yield control (called
cooperative multitasking) or based on a regular timer or interrupts (preemptive
multitasking). The RTOS typically provide mutexes and other synchronisation
primitives, and often interoperate with hardware features such as DMA engines.

[FreeRTOS]: https://freertos.org/
[ChibiOS]: http://chibios.org/

At the time of writing there are not many Rust RTOS examples to point to,
but it's an interesting area so watch this space!

## Multiple Cores

It is becoming more common to have two or more cores in embedded processors,
which adds an extra layer of complexity to concurrency. All the examples using
a critical section (including the `cortex_m::interrupt::Mutex`) assume the only
other execution thread is the interrupt thread, but on a multi-core system
that's no longer true. Instead, we'll need synchronisation primitives designed
for multiple cores (also called SMP, for symmetric multi-processing).

These typically use the atomic instructions we saw earlier, since the
processing system will ensure that atomicity is maintained over all cores.

Covering these topics in detail is currently beyond the scope of this book,
but the general patterns are the same as for the single-core case.
