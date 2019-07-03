# Typestate программирование

Концепт [typestates] предполагает кодирование информации о текущем состоянии объекта в типе самого объекта.
Хотя это и может звучать несколько загадочно, если Вы изучили [Builder Pattern] в Rust, то уже начали
использовать Typestate программирование!

[typestates]: https://en.wikipedia.org/wiki/Typestate_analysis
[Builder Pattern]: https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
#[derive(Debug)]
struct Foo {
    inner: u32,
}

struct FooBuilder {
    a: u32,
    b: u32,
}

impl FooBuilder {
    pub fn new(starter: u32) -> Self {
        Self {
            a: starter,
            b: starter,
        }
    }

    pub fn double_a(self) -> Self {
        Self {
            a: self.a * 2,
            b: self.b,
        }
    }

    pub fn into_foo(self) -> Foo {
        Foo {
            inner: self.a + self.b,
        }
    }
}

fn main() {
    let x = FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

В этом примере не прямого способа создать объект `Foo`. Мы должны создать `FooBuilder` и правильно
инициализировать его перед тем, как сформировать нужный нам объект `Foo`.

Этот минимальный пример кодирует 2 состояния:

* `FooBuilder`, который представляет состояние "ненастроен", или "в процессе настройки"
* `Foo`, который представляет состояние "настроен", или "готов к работе".

## Сильные типы

Так как у Rust [Сильная система типов][Strong Type System], то нет простого способа магически создать
экземпляр `Foo` или превратить `FooBuilder` в `Foo` без вызова метода `into_foo()`. К тому же, вызов
метода `into_foo()` поглощает начальную структуру `FooBuilder`, а это значит, что её нельзя использовать
не создав новый экземпляр.

[Strong Type System]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

Это позволяет представлять состояния нашей системы как типы, а необходимые действия смены состояний как методы,
менящие один тип на другой. Создавая `FooBuilder` и меняя его на объект `Foo`, мы проходим пошагово
базовый конечный автомат.