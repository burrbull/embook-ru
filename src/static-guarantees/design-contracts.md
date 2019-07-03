# Соглашения по проектированию

В предыдущей главе мы описали интерфейс, который *не* соответствует критериям проектирования.
Давайте взглянем с другой стороны на наш воображаемый регистр настройки GPIO:

| Название     | Номер бита | Значение | Состояние | Примечания |
| ---:         | ------------: | ----: | ------:   | ----: |
| enable       | 0             | 0     | disabled  | Отключает GPIO |
|              |               | 1     | enabled   | Включает GPIO |
| direction    | 1             | 0     | input     | Настраивает на Вход |
|              |               | 1     | output    | Настраивает на Выход |
| input_mode   | 2..3          | 00    | hi-z      | Вход с высоким сопротивлением |
|              |               | 01    | pull-low  | Вход подтянут к земле |
|              |               | 10    | pull-high | Вход подтянут к питанию |
|              |               | 11    | n/a       | Неверное сотояние. Не установливать |
| output_mode  | 4             | 0     | set-low   | Низкое напряжение на Выходе |
|              |               | 1     | set-high  | Высокое напряжение на Выходе |
| input_status | 5             | x     | in-val    | 0 если Вход < 1.5В, 1 если >= 1.5В |

Если бы мы вместо проверки состояния перед использованием оборудования, проверяли
критерии в рантайме, то написали бы код подобный этому:

```rust,ignore
/// Интерфейс GPIO
struct GpioConfig {
    /// Структура конфигурации GPIO, сгенерированная svd2rust
    periph: GPIO_CONFIG,
}

impl Gpio {
    pub fn set_enable(&mut self, is_enabled: bool) {
        self.periph.modify(|_r, w| {
            w.enable().set_bit(is_enabled)
        });
    }

    pub fn set_direction(&mut self, is_output: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Должен быть включен перед установкой направления
            return Err(());
        }

        self.periph.modify(|r, w| {
            w.direction().set_bit(is_output)
        });

        Ok(())
    }

    pub fn set_input_mode(&mut self, variant: InputMode) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Должен быть включен перед установкой режима Входа
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Должен быть настроен на Вход
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.input_mode().variant(variant)
        });

        Ok(())
    }

    pub fn set_output_status(&mut self, is_high: bool) -> Result<(), ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Должен быть включен перед установкой состояния Выхода
            return Err(());
        }

        if self.periph.read().direction().bit_is_clear() {
            // Должен быть настроен на Выход
            return Err(());
        }

        self.periph.modify(|_r, w| {
            w.output_mode.set_bit(is_high)
        });

        Ok(())
    }

    pub fn get_input_status(&self) -> Result<bool, ()> {
        if self.periph.read().enable().bit_is_clear() {
            // Должен быть включен, чтобы получить состояние
            return Err(());
        }

        if self.periph.read().direction().bit_is_set() {
            // Должен быть настроен на Вход
            return Err(());
        }

        Ok(self.periph.read().input_status().bit_is_set())
    }
}
```

Так как нам нужно проверять ограничения на оборудовании, в итоге вы делаем кучу проверок во время выполнения,
которые расходуют время и ресурсы, и этот код будет гораздо менее приятным для разработчика.

## Типы-состояния

Но что если мы вместо этого будем использовать систему типов Rust, чтобы убедиться в правильности выполнения правил перехода? Взглянем на пример:

```rust,ignore
/// Интерфейс GPIO
struct GpioConfig<ENABLED, DIRECTION, MODE> {
    /// Структура конфигурации GPIO, сгенерированная svd2rust
    periph: GPIO_CONFIG,
    enabled: ENABLED,
    direction: DIRECTION,
    mode: MODE,
}

// Маркерные типы для поля MODE вn GpioConfig
struct Disabled;
struct Enabled;
struct Output;
struct Input;
struct PulledLow;
struct PulledHigh;
struct HighZ;
struct DontCare;

/// Эти функции можно использовать для любого пина GPIO
impl<EN, DIR, IN_MODE> GpioConfig<EN, DIR, IN_MODE> {
    pub fn into_disabled(self) -> GpioConfig<Disabled, DontCare, DontCare> {
        self.periph.modify(|_r, w| w.enable.disabled());
        GpioConfig {
            periph: self.periph,
            enabled: Disabled,
            direction: DontCare,
            mode: DontCare,
        }
    }

    pub fn into_enabled_input(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.input()
             .input_mode.high_z()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_enabled_output(self) -> GpioConfig<Enabled, Output, DontCare> {
        self.periph.modify(|_r, w| {
            w.enable.enabled()
             .direction.output()
             .input_mode.set_high()
        });
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Output,
            mode: DontCare,
        }
    }
}

/// Эти фунции можно использовать для пина-Выхода
impl GpioConfig<Enabled, Output, DontCare> {
    pub fn set_bit(&mut self, set_high: bool) {
        self.periph.modify(|_r, w| w.output_mode.set_bit(set_high));
    }
}

/// Эти методы можно использовать на любом включенном входе GPIO
impl<IN_MODE> GpioConfig<Enabled, Input, IN_MODE> {
    pub fn bit_is_set(&self) -> bool {
        self.periph.read().input_status.bit_is_set()
    }

    pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
        self.periph.modify(|_r, w| w.input_mode().high_z());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: HighZ,
        }
    }

    pub fn into_input_pull_down(self) -> GpioConfig<Enabled, Input, PulledLow> {
        self.periph.modify(|_r, w| w.input_mode().pull_low());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledLow,
        }
    }

    pub fn into_input_pull_up(self) -> GpioConfig<Enabled, Input, PulledHigh> {
        self.periph.modify(|_r, w| w.input_mode().pull_high());
        GpioConfig {
            periph: self.periph,
            enabled: Enabled,
            direction: Input,
            mode: PulledHigh,
        }
    }
}
```

Теперь посмотрим, как будет выглядеть код, использующий эти функции:

```rust,ignore
/*
 * Пример 1: Из несконфигурированного в High-Z вход
 */
let pin: GpioConfig<Disabled, _, _> = get_gpio();

// Не могу это сделать, пин не включен!
// pin.into_input_pull_down();

// Переключаем пин из несконфигурированного в High-Z вход
let input_pin = pin.into_enabled_input();

// Читаем состояние пина
let pin_state = input_pin.bit_is_set();

// Не могу это сделать, у входных пинов нет такого интерфейса!
// input_pin.set_bit(true);

/*
 * Пример 2: Из High-Z входа в притянутый к земле вход
 */
let pulled_low = input_pin.into_input_pull_down();
let pin_state = pulled_low.bit_is_set();

/*
 * Пример 3: Из притянутого к земле входа в выход с высоким состоянием
 */
let output_pin = pulled_low.into_enabled_output();
output_pin.set_bit(false);

// Не могу это сделать, у выходных пинов нет такого интерфейса!
// output_pin.into_input_pull_down();
```

Это, безусловно, удобный способ хранить состояние пина, но зачем это делать? Почему это лучше, чем хранить состояние в виде `enum` внутри нашей структуры` GpioConfig`?

## Функциональная безопасность времени компиляции

Так как мы проверяем наши проектные ограничения полностью на этапе компиляции, то не несем расходов
во время выполнения. Невозможно установить режим выхода для входного пина.
Вместо этого Вы должны пройти через ряд состояний, чтобы преобраховать его в выходной пин, а затем
установить его выходной режим. Из-за этого мы не имеем потерь в рантайме, т.к. проверяем текущее состояние
перед выполнением функции.

К тому же, т.к эти состояния проверяются системой типов, пользователи этого интерфейса не будут допускать
ошибок. Если они попытаются выполнить недопустимое переключение состояния, код cкомпилируется!
