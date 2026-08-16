## Доступ к регистрам через MMIO и Volatile

> **Чему вы научитесь:** Типобезопасному доступу к аппаратным регистрам в embedded Rust — паттернам volatile MMIO, крейтам абстракции регистров, а также тому, как система типов Rust позволяет кодировать права доступа к регистрам, чего не может ключевое слово `volatile` в языке C.

В прошивках на C доступ к аппаратным регистрам осуществляется через указатели `volatile` на определенные адреса памяти. В Rust есть аналогичные механизмы, но с типобезопасностью.

### C volatile против Rust volatile

```c
// C — типичный доступ к регистрам через MMIO
#define GPIO_BASE     0x40020000
#define GPIO_MODER    (*(volatile uint32_t*)(GPIO_BASE + 0x00))
#define GPIO_ODR      (*(volatile uint32_t*)(GPIO_BASE + 0x14))

void toggle_led(void) {
    GPIO_ODR ^= (1 << 5);  // Переключение пина 5
}
```

```rust
// Rust — «сырой» volatile (низкоуровневый, редко используется напрямую)
use core::ptr;

const GPIO_BASE: usize = 0x4002_0000;
const GPIO_ODR: *mut u32 = (GPIO_BASE + 0x14) as *mut u32;

/// # Safety
/// Вызывающий код должен гарантировать, что GPIO_BASE является валидным отображенным адресом периферии.
unsafe fn toggle_led() {
    // SAFETY: GPIO_ODR является валидным адресом регистра, отображенного в память.
    let current = unsafe { ptr::read_volatile(GPIO_ODR) };
    unsafe { ptr::write_volatile(GPIO_ODR, current ^ (1 << 5)) };
}
```

### svd2rust — типобезопасный доступ к регистрам (подход Rust)

На практике вы **никогда** не пишете сырые указатели volatile вручную. Вместо этого `svd2rust` генерирует **пакет доступа к периферии (Peripheral Access Crate, PAC)** из SVD-файла чипа (того самого XML-файла, который используется вашей IDE в режиме отладки):

```rust
// Сгенерированный код PAC (вы не пишете его вручную — его создает svd2rust)
// PAC делает некорректный доступ к регистрам ошибкой компиляции

// Использование с PAC:
use stm32f4::stm32f401;  // PAC крейт для вашего чипа

fn configure_gpio(dp: stm32f401::Peripherals) {
    // Включение тактирования GPIOA — типобезопасно, без магических чисел
    dp.RCC.ahb1enr.modify(|_, w| w.gpioaen().enabled());

    // Настройка пина 5 на вывод — невозможно случайно записать в поле, доступное только для чтения
    dp.GPIOA.moder.modify(|_, w| w.moder5().output());

    // Переключение пина 5 — проверка типов при доступе к полю
    dp.GPIOA.odr.modify(|r, w| {
        // SAFETY: переключение одного бита в валидном поле регистра.
        unsafe { w.bits(r.bits() ^ (1 << 5)) }
    });
}
```

| Доступ к регистрам в C | Эквивалент в Rust PAC |
|-------------------|---------------------|
| `#define REG (*(volatile uint32_t*)ADDR)` | Крейт PAC, сгенерированный с помощью `svd2rust` |
| `REG |= BITMASK;` | `periph.reg.modify(\|_, w\| w.field().variant())` |
| `value = REG;` | `let val = periph.reg.read().field().bits()` |
| Неверное поле регистра → тихое неопределенное поведение (UB) | Ошибка компиляции — поле не существует |
| Неверная разрядность регистра → тихое неопределенное поведение (UB) | Проверка типов — u8 против u16 против u32 |

## Обработка прерываний и критические секции

В прошивках на C используются функции `__disable_irq()` / `__enable_irq()` и обработчики прерываний (ISR) с сигнатурой `void`. Rust предлагает типобезопасные аналоги.

### Шаблоны обработки прерываний: C против Rust

```c
// C — традиционный обработчик прерывания
volatile uint32_t tick_count = 0;

void SysTick_Handler(void) {   // Соглашение об именовании критически важно — ошибка приведет к HardFault
    tick_count++;
}

uint32_t get_ticks(void) {
    __disable_irq();
    uint32_t t = tick_count;   // Чтение внутри критической секции
    __enable_irq();
    return t;
}
```

```rust
// Rust — использование cortex-m и критических секций
use core::cell::Cell;
use cortex_m::interrupt::{self, Mutex};

// Разделяемое состояние, защищенное мьютексом критической секции
static TICK_COUNT: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[cortex_m_rt::exception]     // Атрибут гарантирует корректное размещение в таблице векторов
fn SysTick() {                // Ошибка компиляции, если имя не соответствует валидному исключению
    interrupt::free(|cs| {    // cs = токен критической секции (доказательство отключенных прерываний IRQ)
        let count = TICK_COUNT.borrow(cs).get();
        TICK_COUNT.borrow(cs).set(count + 1);
    });
}

fn get_ticks() -> u32 {
    interrupt::free(|cs| TICK_COUNT.borrow(cs).get())
}
```

### RTIC — Real-Time Interrupt-driven Concurrency (конкурентность на основе прерываний реального времени)

Для сложных прошивок с несколькими приоритетами прерываний RTIC (ранее RTFM) предоставляет **планирование задач на этапе компиляции с нулевыми накладными расходами**:

```rust
#[rtic::app(device = stm32f4xx_hal::pac, dispatchers = [USART1])]
mod app {
    use stm32f4xx_hal::prelude::*;

    #[shared]
    struct Shared {
        temperature: f32,   // Разделяется между задачами — RTIC создает блокировки
    }

    #[local]
    struct Local {
        led: stm32f4xx_hal::gpio::Pin<'A', 5, stm32f4xx_hal::gpio::Output>,
    }

    #[init]
    fn init(cx: init::Context) -> (Shared, Local) {
        let dp = cx.device;
        let gpioa = dp.GPIOA.split();
        let led = gpioa.pa5.into_push_pull_output();
        (Shared { temperature: 25.0 }, Local { led })
    }

    // Аппаратная задача: запускается по прерыванию SysTick
    #[task(binds = SysTick, shared = [temperature], local = [led])]
    fn tick(mut cx: tick::Context) {
        cx.local.led.toggle();
        cx.shared.temperature.lock(|temp| {
            // RTIC гарантирует монопольный доступ здесь — ручные блокировки не требуются
            *temp += 0.1;
        });
    }
}
```

**Почему RTIC важен для разработчиков встроенного ПО на C:**
- Аннотация `#[shared]` заменяет ручное управление мьютексами.
- Вытеснение на основе приоритетов настраивается на этапе компиляции — без накладных расходов в процессе выполнения.
- Безопасность от взаимных блокировок (deadlocks) гарантирована по построению (фреймворк доказывает это во время компиляции).
- Ошибки в именах обработчиков прерываний (ISR) выявляются при компиляции, а не приводят к HardFault во время выполнения.

## Стратегии обработчиков паники (Panic Handlers)

В C, когда во встроенном ПО что-то идет не так, вы обычно сбрасываете систему или мигаете светодиодом. Встроенный в Rust обработчик паники дает вам структурированный контроль:

```rust
// Стратегия 1: Остановка (для отладки — подключение отладчика, анализ состояния)
use panic_halt as _;  // Бесконечный цикл при панике

// Стратегия 2: Сброс микроконтроллера
use panic_reset as _;  // Вызывает системный сброс

// Стратегия 3: Логирование через отладочный зонд (разработка)
use panic_probe as _;  // Отправляет информацию о панике через отладочный зонд (с использованием defmt)

// Стратегия 4: Логирование через defmt, затем остановка
use defmt_panic as _;  // Информативные сообщения о панике через ITM/RTT

// Стратегия 5: Пользовательский обработчик (для продакшн-прошивки)
use core::panic::PanicInfo;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    // 1. Отключение прерываний для предотвращения дальнейших сбоев
    cortex_m::interrupt::disable();

    // 2. Запись информации о панике в зарезервированную область RAM (сохраняется после сброса)
    // SAFETY: PANIC_LOG — это зарезервированная область памяти, определенная в скрипте компоновщика.
    unsafe {
        let log = 0x2000_0000 as *mut [u8; 256];
        // Запись усеченного сообщения о панике
        use core::fmt::Write;
        let mut writer = FixedWriter::new(&mut *log);
        let _ = write!(writer, "{}", info);
    }

    // 3. Запуск сброса сторожевым таймером (или мигание светодиодом ошибки)
    loop {
        cortex_m::asm::wfi();  // Ожидание прерывания (низкое энергопотребление во время остановки)
    }
}
```

## Скрипты компоновщика и разметка памяти

Разработчики прошивок на C пишут скрипты компоновщика (linker scripts) для определения областей FLASH/RAM. В embedded-разработке на Rust используется та же концепция с помощью файла `memory.x`:

```ld
/* memory.x — размещается в корне крейта, используется cortex-m-rt */
MEMORY
{
  /* Настройте под ваш микроконтроллер — это значения для STM32F401 */
  FLASH : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 96K
}

/* Необязательно: резервирование места для лога паники (см. обработчик паники выше) */
_panic_log_start = ORIGIN(RAM);
_panic_log_size  = 256;
```

```toml
# .cargo/config.toml — настройка целевой платформы и флагов компоновщика
[target.thumbv7em-none-eabihf]
runner = "probe-rs run --chip STM32F401RE"  # прошивка и запуск через отладочный зонд
rustflags = [
    "-C", "link-arg=-Tlink.x",              # скрипт компоновщика cortex-m-rt
]

[build]
target = "thumbv7em-none-eabihf"            # Cortex-M4F с аппаратным FPU
```

| Скрипт компоновщика C | Эквивалент в Rust |
|-----------------|-----------------|
| `MEMORY { FLASH ..., RAM ... }` | `memory.x` в корне крейта |
| `__attribute__((section(".data")))` | `#[link_section = ".data"]` |
| `-T linker.ld` в Makefile | `-C link-arg=-Tlink.x` в `.cargo/config.toml` |
| `__bss_start__`, `__bss_end__` | Обрабатывается `cortex-m-rt` автоматически |
| Ассемблер инициализации (`startup.s`) | Макрос `#[entry]` из `cortex-m-rt` |

## Написание драйверов на основе `embedded-hal`

Крейт `embedded-hal` определяет типажи (traits) для SPI, I2C, GPIO, UART и т. д. Драйверы, написанные с использованием этих типажей, работают на **любом микроконтроллере** — это ключевое преимущество Rust для повторного использования кода во встраиваемых системах.

### C против Rust: драйвер датчика температуры

```c
// C — драйвер жестко привязан к STM32 HAL
#include "stm32f4xx_hal.h"

float read_temperature(I2C_HandleTypeDef* hi2c, uint8_t addr) {
    uint8_t buf[2];
    HAL_I2C_Mem_Read(hi2c, addr << 1, 0x00, I2C_MEMADD_SIZE_8BIT,
                     buf, 2, HAL_MAX_DELAY);
    int16_t raw = ((int16_t)buf[0] << 4) | (buf[1] >> 4);
    return raw * 0.0625;
}
// Проблема: этот драйвер работает ТОЛЬКО с STM32 HAL. Перенос на Nordic = переписывание.
```

```rust
// Rust — драйвер работает на ЛЮБОМ микроконтроллере, реализующем embedded-hal
use embedded_hal::i2c::I2c;

pub struct Tmp102<I2C> {
    i2c: I2C,
    address: u8,
}

impl<I2C: I2c> Tmp102<I2C> {
    pub fn new(i2c: I2C, address: u8) -> Self {
        Self { i2c, address }
    }

    pub fn read_temperature(&mut self) -> Result<f32, I2C::Error> {
        let mut buf = [0u8; 2];
        self.i2c.write_read(self.address, &[0x00], &mut buf)?;
        let raw = ((buf[0] as i16) << 4) | ((buf[1] as i16) >> 4);
        Ok(raw as f32 * 0.0625)
    }
}

// Работает на STM32, Nordic nRF, ESP32, RP2040 — на любом чипе с реализацией I2C из embedded-hal
```

```mermaid
graph TD
    subgraph "Архитектура драйверов C"
        CD["Драйвер температуры"]
        CD --> STM["STM32 HAL"]
        CD -.->|"Портирование = ПЕРЕПИСЫВАНИЕ"| NRF["Nordic HAL"]
        CD -.->|"Портирование = ПЕРЕПИСЫВАНИЕ"| ESP["ESP-IDF"]
    end
    
    subgraph "Архитектура Rust embedded-hal"
        RD["Драйвер температуры<br/>impl&lt;I2C: I2c&gt;"]
        RD --> EHAL["типаж embedded-hal::I2c"]
        EHAL --> STM2["stm32f4xx-hal"]
        EHAL --> NRF2["nrf52-hal"]
        EHAL --> ESP2["esp-hal"]
        EHAL --> RP2["rp2040-hal"]
        NOTE["Напишите драйвер ОДИН РАЗ,<br/>он будет работать на ВСЕХ чипах"]
    end
    
    style CD fill:#ffa07a,color:#000
    style RD fill:#91e5a3,color:#000
    style EHAL fill:#91e5a3,color:#000
    style NOTE fill:#91e5a3,color:#000
```

## Настройка глобального аллокатора

Крейт `alloc` предоставляет вам `Vec`, `String`, `Box`, но вам нужно указать Rust, откуда берется куча (heap). Это эквивалентно реализации `malloc()` для вашей платформы:

```rust
#![no_std]
extern crate alloc;

use alloc::vec::Vec;
use alloc::string::String;
use embedded_alloc::LlffHeap as Heap;

#[global_allocator]
static HEAP: Heap = Heap::empty();

#[cortex_m_rt::entry]
fn main() -> ! {
    // Инициализируем аллокатор областью памяти
    // (обычно часть RAM, не используемая стеком или статическими данными)
    {
        const HEAP_SIZE: usize = 4096;
        static mut HEAP_MEM: [u8; HEAP_SIZE] = [0; HEAP_SIZE];
        // SAFETY: HEAP_MEM считывается/записывается только здесь во время инициализации, до каких-либо выделений памяти.
        unsafe { HEAP.init(HEAP_MEM.as_ptr() as usize, HEAP_SIZE) }
    }

    // Теперь можно использовать типы, размещаемые в куче!
    let mut log_buffer: Vec<u8> = Vec::with_capacity(256);
    let name: String = String::from("sensor_01");
    // ...

    loop {}
}
```

| Настройка кучи в C | Эквивалент в Rust |
|-------------|-----------------|
| `_sbrk()` / кастомный `malloc()` | `#[global_allocator]` + `Heap::init()` |
| `configTOTAL_HEAP_SIZE` (FreeRTOS) | Константа `HEAP_SIZE` |
| `pvPortMalloc()` | `alloc::vec::Vec::new()` — автоматически |
| Исчерпание кучи → неопределенное поведение (UB) | `alloc_error_handler` → контролируемая паника |

## Смешанные рабочие пространства `no_std` + `std`

Реальные проекты (например, крупное рабочее пространство Rust) часто содержат:
- библиотеки без стандартной библиотеки (`no_std`) для аппаратно-независимой логики;
- исполняемые файлы со стандартной библиотекой (`std`) для прикладного уровня под Linux.

```text
workspace_root/
├── Cargo.toml              # [workspace] members = [...]
├── protocol/               # no_std — сетевой протокол, парсинг
│   ├── Cargo.toml          # no default-features, no std
│   └── src/lib.rs          # #![no_std]
├── driver/                 # no_std — абстракция оборудования
│   ├── Cargo.toml
│   └── src/lib.rs          # #![no_std], использует типажи embedded-hal
├── firmware/               # no_std — бинарный файл микроконтроллера
│   ├── Cargo.toml          # зависит от protocol, driver
│   └── src/main.rs         # #![no_std] #![no_main]
└── host_tool/              # std — CLI-утилита для Linux
    ├── Cargo.toml          # зависит от protocol (тот же самый крейт!)
    └── src/main.rs         # Использует std::fs, std::net и т. д.
```

Ключевой паттерн: крейт `protocol` использует `#![no_std]`, поэтому он компилируется **как** для прошивки микроконтроллера, **так и** для хост-утилиты под Linux. Общий код, нулевое дублирование.

```toml
# protocol/Cargo.toml
[package]
name = "protocol"

[features]
default = []
std = []  # Необязательно: включение специфичных для std функций при сборке для хоста

[dependencies]
serde = { version = "1", default-features = false, features = ["derive"] }
# Примечание: default-features = false убирает зависимость serde от std
```

```rust
// protocol/src/lib.rs
#![cfg_attr(not(feature = "std"), no_std)]

#[cfg(feature = "std")]
extern crate std;

extern crate alloc;
use alloc::vec::Vec;
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct DiagPacket {
    pub sensor_id: u16,
    pub value: i32,
    pub fault_code: u16,
}

// Эта функция работает как в контексте no_std, так и в std
pub fn parse_packet(data: &[u8]) -> Result<DiagPacket, &'static str> {
    if data.len() < 8 {
        return Err("packet too short");
    }
    Ok(DiagPacket {
        sensor_id: u16::from_le_bytes([data[0], data[1]]),
        value: i32::from_le_bytes([data[2], data[3], data[4], data[5]]),
        fault_code: u16::from_le_bytes([data[6], data[7]]),
    })
}
```

## Упражнение: Драйвер для уровня абстракции оборудования (HAL)

Напишите `no_std` драйвер для гипотетического контроллера светодиодов, взаимодействующего по SPI. Драйвер должен быть обобщенным (generic) для любой реализации SPI с использованием `embedded-hal`.

**Требования:**
1. Определить структуру `LedController<SPI>`.
2. Реализовать методы `new()`, `set_brightness(led: u8, brightness: u8)` и `all_off()`.
3. Протокол SPI: отправлять `[led_index, brightness_value]` в виде 2-байтовой транзакции.
4. Написать тесты с использованием mock-реализации SPI.

```rust
// Начальный код
#![no_std]
use embedded_hal::spi::SpiDevice;

pub struct LedController<SPI> {
    spi: SPI,
    num_leds: u8,
}

// TODO: Реализовать new(), set_brightness(), all_off()
// TODO: Создать MockSpi для тестирования
```

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
#![no_std]
use embedded_hal::spi::SpiDevice;

pub struct LedController<SPI> {
    spi: SPI,
    num_leds: u8,
}

impl<SPI: SpiDevice> LedController<SPI> {
    pub fn new(spi: SPI, num_leds: u8) -> Self {
        Self { spi, num_leds }
    }

    pub fn set_brightness(&mut self, led: u8, brightness: u8) -> Result<(), SPI::Error> {
        if led >= self.num_leds {
            return Ok(()); // Молча игнорируем светодиоды вне допустимого диапазона
        }
        self.spi.write(&[led, brightness])
    }

    pub fn all_off(&mut self) -> Result<(), SPI::Error> {
        for led in 0..self.num_leds {
            self.spi.write(&[led, 0])?;
        }
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Mock SPI для записи всех транзакций
    struct MockSpi {
        transactions: Vec<Vec<u8>>,
    }

    // Минимальный тип ошибки для mock
    #[derive(Debug)]
    struct MockError;
    impl embedded_hal::spi::Error for MockError {
        fn kind(&self) -> embedded_hal::spi::ErrorKind {
            embedded_hal::spi::ErrorKind::Other
        }
    }

    impl embedded_hal::spi::ErrorType for MockSpi {
        type Error = MockError;
    }

    impl SpiDevice for MockSpi {
        fn write(&mut self, buf: &[u8]) -> Result<(), Self::Error> {
            self.transactions.push(buf.to_vec());
            Ok(())
        }
        fn read(&mut self, _buf: &mut [u8]) -> Result<(), Self::Error> { Ok(()) }
        fn transfer(&mut self, _r: &mut [u8], _w: &[u8]) -> Result<(), Self::Error> { Ok(()) }
        fn transfer_in_place(&mut self, _buf: &mut [u8]) -> Result<(), Self::Error> { Ok(()) }
        fn transaction(&mut self, _ops: &mut [embedded_hal::spi::Operation<'_, u8>]) -> Result<(), Self::Error> { Ok(()) }
    }

    #[test]
    fn test_set_brightness() {
        let mock = MockSpi { transactions: vec![] };
        let mut ctrl = LedController::new(mock, 4);
        ctrl.set_brightness(2, 128).unwrap();
        assert_eq!(ctrl.spi.transactions, vec![vec![2, 128]]);
    }

    #[test]
    fn test_all_off() {
        let mock = MockSpi { transactions: vec![] };
        let mut ctrl = LedController::new(mock, 3);
        ctrl.all_off().unwrap();
        assert_eq!(ctrl.spi.transactions, vec![
            vec![0, 0], vec![1, 0], vec![2, 0],
        ]);
    }

    #[test]
    fn test_out_of_range_led() {
        let mock = MockSpi { transactions: vec![] };
        let mut ctrl = LedController::new(mock, 2);
        ctrl.set_brightness(5, 255).unwrap(); // Вне диапазона — игнорируется
        assert!(ctrl.spi.transactions.is_empty());
    }
}
```

</details>

## Отладка встроенного Rust — probe-rs, defmt и VS Code

Разработчики прошивок на C обычно используют для отладки OpenOCD + GDB или коммерческие IDE (Keil, IAR, Segger Ozone). Экосистема встроенного Rust объединилась вокруг **probe-rs** как унифицированного интерфейса для отладочных зондов, заменив связку OpenOCD + GDB одним нативным инструментом на Rust.

### probe-rs — инструмент отладки «все в одном»

`probe-rs` заменяет комбинацию OpenOCD + GDB. Он поддерживает CMSIS-DAP, ST-Link, J-Link и другие отладочные зонды «из коробки»:

```bash
# Установка probe-rs (включает cargo-flash и cargo-embed)
cargo install probe-rs-tools

# Прошивка и запуск вашей программы
cargo flash --chip STM32F401RE --release

# Прошивка, запуск и открытие консоли RTT (Real-Time Transfer)
cargo embed --chip STM32F401RE
```

**probe-rs против OpenOCD + GDB**:

| Аспект | OpenOCD + GDB | probe-rs |
|--------|--------------|----------|
| Установка | 2 отдельных пакета + скрипты | `cargo install probe-rs-tools` |
| Конфигурация | Файлы `.cfg` для плат/зондов | Флаг `--chip` или файл `Embed.toml` |
| Вывод в консоль | Семихостинг (очень медленно) | RTT (быстрее примерно в 10 раз) |
| Фреймворк логирования | `printf` | `defmt` (структурированный, с нулевой стоимостью) |
| Алгоритм прошивки | Файлы пакетов XML | Встроенный для более 1000 чипов |
| Поддержка GDB | Нативная | Адаптер `probe-rs gdb` |

### `Embed.toml` — Конфигурация проекта

Вместо манипуляций с файлами `.cfg` и `.gdbinit`, probe-rs использует единый файл конфигурации:

```toml
# Embed.toml — размещается в корне вашего проекта
[default.general]
chip = "STM32F401RETx"

[default.rtt]
enabled = true           # Включение консоли Real-Time Transfer
channels = [
    { up = 0, mode = "BlockIfFull", name = "Terminal" },
]

[default.flashing]
enabled = true           # Прошивка перед запуском
restore_unwritten_bytes = false

[default.reset]
halt_afterwards = false  # Запуск выполнения после прошивки и сброса

[default.gdb]
enabled = false          # Установите true для запуска GDB-сервера на порту 1337
gdb_connection_string = "127.0.0.1:1337"
```

```bash
# С файлом Embed.toml достаточно выполнить:
cargo embed              # Прошивка + RTT-консоль — никаких флагов не требуется
cargo embed --release    # Сборка для релиза
```

### defmt — отложенное форматирование для логирования во встроенных системах

`defmt` (deferred formatting) заменяет отладку с помощью `printf`. Строки форматирования сохраняются в ELF-файле, а не во FLASH-памяти чипа, поэтому вызовы логирования на устройстве передают только индекс строки и байты аргументов. Это делает логирование **в 10–100 раз быстрее**, чем `printf`, и экономит память FLASH:

```rust
#![no_std]
#![no_main]

use defmt::{info, warn, error, debug, trace};
use defmt_rtt as _; // Транспорт RTT — связывает вывод defmt с probe-rs

#[cortex_m_rt::entry]
fn main() -> ! {
    info!("Boot complete, firmware v{}", env!("CARGO_PKG_VERSION"));

    let sensor_id: u16 = 0x4A;
    let temperature: f32 = 23.5;

    // Строки форматирования остаются в ELF, а не во FLASH-памяти — накладные расходы близки к нулю
    debug!("Sensor {:#06X}: {:.1}°C", sensor_id, temperature);

    if temperature > 80.0 {
        warn!("Overtemp on sensor {:#06X}: {:.1}°C", sensor_id, temperature);
    }

    loop {
        cortex_m::asm::wfi(); // Ожидание прерывания
    }
}

// Пользовательские типы — выводим defmt::Format вместо Debug
#[derive(defmt::Format)]
struct SensorReading {
    id: u16,
    value: i32,
    status: SensorStatus,
}

#[derive(defmt::Format)]
enum SensorStatus {
    Ok,
    Warning,
    Fault(u8),
}

// Использование:
// info!("Reading: {:?}", reading);  // <-- использует defmt::Format, а НЕ std Debug
```

**defmt против `printf` против `log`**:

| Возможность | C `printf` (семихостинг) | Крейт Rust `log` | `defmt` |
|---------|-------------------------|-------------------|---------|
| Скорость | ~100 мс на вызов | Недоступно (требуется `std`) | ~1 мкс на вызов |
| Использование Flash | Полные строки формата | Полные строки формата | Только индекс (байты) |
| Транспорт | Семихостинг (останавливает CPU) | Serial/UART | RTT (неблокирующий) |
| Структурированный вывод | Нет | Только текст | Типизированный, бинарно-кодированный |
| `no_std` | Через семихостинг | Только фасад (бекендам нужен `std`) | ✅ Нативно |
| Уровни фильтрации | Ручной `#ifdef` | `RUST_LOG=debug` | `defmt::println` + фичи |

### Настройка отладки в VS Code

С расширением `probe-rs` для VS Code вы получаете полноценную графическую отладку: точки останова, инспектирование переменных, стек вызовов и просмотр регистров:

```jsonc
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "probe-rs-debug",
            "request": "launch",
            "name": "Flash & Debug (probe-rs)",
            "chip": "STM32F401RETx",
            "coreConfigs": [
                {
                    "programBinary": "target/thumbv7em-none-eabihf/debug/${workspaceFolderBasename}",
                    "rttEnabled": true,
                    "rttChannelFormats": [
                        {
                            "channelNumber": 0,
                            "dataFormat": "Defmt",
                            "showTimestamps": true
                        }
                    ]
                }
            ],
            "connectUnderReset": true,
            "speed": 4000
        }
    ]
}
```

Установите расширение:
```rust
ext install probe-rs.probe-rs-debugger
```

### Процесс отладки на C против отладки в Rust Embedded

```mermaid
graph LR
    subgraph "Процесс работы в C (традиционный)"
        C1["Написание кода"] --> C2["make flash"]
        C2 --> C3["openocd -f board.cfg"]
        C3 --> C4["arm-none-eabi-gdb<br/>target remote :3333"]
        C4 --> C5["printf через семихостинг<br/>(~100 мс на вызов, останавливает CPU)"]
    end
    
    subgraph "Процесс работы в Rust (probe-rs)"
        R1["Написание кода"] --> R2["cargo embed"]
        R2 --> R3["Прошивка + консоль RTT<br/>одной командой"]
        R3 --> R4["Поток логов defmt<br/>в реальном времени (~1 мкс)"]
        R2 -.->|"Или"| R5["F5 в VS Code<br/>Полноценный графический отладчик"]
    end
    
    style C5 fill:#ffa07a,color:#000
    style R3 fill:#91e5a3,color:#000
    style R4 fill:#91e5a3,color:#000
    style R5 fill:#91e5a3,color:#000
```

| Действие отладки в C | Эквивалент в Rust |
|---------------|-----------------|
| `openocd -f board/st_nucleo_f4.cfg` | `probe-rs info` (автоматически определяет отладчик и чип) |
| `arm-none-eabi-gdb -x .gdbinit` | `probe-rs gdb --chip STM32F401RE` |
| `target remote :3333` | GDB подключается к `localhost:1337` |
| `monitor reset halt` | `probe-rs reset --chip ...` |
| `load firmware.elf` | `cargo flash --chip ...` |
| `printf("debug: %d\n", val)` (семихостинг) | `defmt::info!("debug: {}", val)` (RTT) |
| Графический отладчик Keil/IAR | VS Code + расширение `probe-rs-debugger` |
| Segger SystemView | Просмотрщик RTT `defmt` + `probe-rs` |

> **Перекрестная ссылка**: Информацию о продвинутых небезопасных паттернах (unsafe patterns), используемых во встроенных драйверах (проекции пинов, кастомные аллокаторы arena/slab), см. в сопутствующем руководстве *Шаблоны проектирования на Rust*, разделы «Проекции Pin — структурное закрепление» (Pin Projections — Structural Pinning) и «Кастомные аллокаторы — паттерны Arena и Slab» (Custom Allocators — Arena and Slab Patterns).

---
