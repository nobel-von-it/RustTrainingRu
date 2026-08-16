## Логирование и трассировка: переход с syslog/printf на `log` + `tracing`

> **Чему вы научитесь:** Двухуровневой архитектуре логирования в Rust (фасад + бэкенд), крейтам `log` и `tracing`, структурированному логированию со спанами (spans), а также тому, как это заменяет отладку с помощью `printf`/`syslog`.

Диагностический код на C++ обычно использует `printf`, `syslog` или кастомные фреймворки логирования. В Rust принята стандартизированная двухуровневая архитектура логирования: крейт-**фасад** (`log` или `tracing`) и **бэкенд** (непосредственная реализация логера).

### Фасад `log` — универсальный API логирования в Rust

Крейт `log` предоставляет макросы, соответствующие уровням важности syslog. Библиотеки используют макросы `log`, а исполняемые файлы (binaries) выбирают конкретный бэкенд:

```rust
// Cargo.toml
// [dependencies]
// log = "0.4"
// env_logger = "0.11"    # Один из множества бэкендов

use log::{info, warn, error, debug, trace};

fn check_sensor(id: u32, temp: f64) {
    trace!("Reading sensor {id}");           // Максимальная детализация
    debug!("Sensor {id} raw value: {temp}"); // Подробности для этапа разработки

    if temp > 85.0 {
        warn!("Sensor {id} high temperature: {temp}°C");
    }
    if temp > 95.0 {
        error!("Sensor {id} CRITICAL: {temp}°C — initiating shutdown");
    }
    info!("Sensor {id} check complete");     // Обычный режим работы
}

fn main() {
    // Инициализируем бэкенд — обычно это делается один раз в main()
    env_logger::init();  // Управляется переменной окружения RUST_LOG

    check_sensor(0, 72.5);
    check_sensor(1, 91.0);
}
```

```bash
# Управление уровнем логирования через переменную окружения
RUST_LOG=debug cargo run          # Показать логи уровня debug и выше
RUST_LOG=warn cargo run           # Показать только warn и error
RUST_LOG=my_crate=trace cargo run # Фильтрация по модулям
RUST_LOG=my_crate::gpu=debug,warn cargo run  # Смешивание уровней
```

### Сравнение с C++

| C++ | Rust (`log`) | Примечания |
|-----|-------------|-------|
| `printf("DEBUG: %s\n", msg)` | `debug!("{msg}")` | Строка формата проверяется при компиляции |
| `syslog(LOG_ERR, "...")` | `error!("...")` | Бэкенд решает, куда направлять вывод |
| `#ifdef DEBUG` вокруг вызовов логирования | Вызовы `trace!` / `debug!` исключаются при компиляции на max_level | Нулевая стоимость при отключении |
| Кастомный `Logger::log(level, msg)` | `log::info!("...")` — все крейты используют общий API | Универсальный фасад, заменяемый бэкенд |
| Детализация логов для отдельных файлов | `RUST_LOG=crate::module=level` | Настраивается через окружение, без перекомпиляции |

### Крейт `tracing` — структурированное логирование со спанами (spans)

Крейт `tracing` расширяет возможности `log`, добавляя **структурированные поля** и **спаны** (spans, временные контексты). Это особенно полезно в диагностическом коде, где важно отслеживать контекст:

```rust
// Cargo.toml
// [dependencies]
// tracing = "0.1"
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }

use tracing::{info, warn, error, instrument, info_span};

#[instrument(skip(data), fields(gpu_id = gpu_id, data_len = data.len()))]
fn run_gpu_test(gpu_id: u32, data: &[u8]) -> Result<(), String> {
    info!("Starting GPU test");

    let span = info_span!("ecc_check", gpu_id);
    let _guard = span.enter();  // Все логи внутри этой области будут содержать gpu_id

    if data.is_empty() {
        error!(gpu_id, "No test data provided");
        return Err("empty data".to_string());
    }

    // Структурированные поля — машиночитаемые, а не просто интерполяция строк
    info!(
        gpu_id,
        temp_celsius = 72.5,
        ecc_errors = 0,
        "ECC check passed"
    );

    Ok(())
}

fn main() {
    // Инициализируем подписчик (subscriber) tracing
    tracing_subscriber::fmt()
        .with_env_filter("debug")  // Или используйте переменную окружения RUST_LOG
        .with_target(true)          // Показывать путь к модулю
        .with_thread_ids(true)      // Показывать ID потоков
        .init();

    let _ = run_gpu_test(0, &[1, 2, 3]);
}
```

Вывод с использованием `tracing-subscriber`:
```rust
2026-02-15T10:30:00.123Z DEBUG ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}: my_crate: Starting GPU test
2026-02-15T10:30:00.124Z  INFO ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}:ecc_check{gpu_id=0}: my_crate: ECC check passed gpu_id=0 temp_celsius=72.5 ecc_errors=0
```

### `#[instrument]` — автоматическое создание спанов

Атрибут `#[instrument]` автоматически создает спан с именем функции и ее аргументами:

```rust
use tracing::instrument;

#[instrument]
fn parse_sel_record(record_id: u16, sensor_type: u8, data: &[u8]) -> Result<(), String> {
    // Каждый лог внутри этой функции автоматически включает в себя:
    // record_id, sensor_type и data (если реализован типаж Debug)
    tracing::debug!("Parsing SEL record");
    Ok(())
}

// skip: исключает большие/чувствительные аргументы из спана
// fields: добавляет вычисляемые поля
#[instrument(skip(raw_buffer), fields(buf_len = raw_buffer.len()))]
fn decode_ipmi_response(raw_buffer: &[u8]) -> Result<Vec<u8>, String> {
    tracing::trace!("Decoding {} bytes", raw_buffer.len());
    Ok(raw_buffer.to_vec())
}
```

### Сравнение `log` и `tracing`

| Аспект | `log` | `tracing` |
|--------|-------|-----------|
| **Сложность** | Простой — 5 макросов | Более богатый — спаны, поля, инспектирование |
| **Структурированные данные** | Только интерполяция строк | Поля ключ-значение: `info!(gpu_id = 0, "msg")` |
| **Замеры времени / спаны** | Нет | Да — `#[instrument]`, `span.enter()` |
| **Асинхронность** | Базовая поддержка | Полноценная поддержка — спаны переносятся через точки `.await` |
| **Совместимость** | Универсальный фасад | Совместим с `log` (имеет мост для `log`) |
| **Когда использовать** | Простые приложения, библиотеки | Инструменты диагностики, асинхронный код, наблюдаемость (observability) |

> **Рекомендация**: Используйте `tracing` для продакшн-проектов диагностической направленности (утилиты диагностики со структурированным выводом). Используйте `log` для простых библиотек, где требуется минимум зависимостей. `tracing` содержит уровень совместимости, поэтому библиотеки, использующие макросы `log`, будут корректно работать с подписчиком `tracing`.

### Варианты бэкендов

| Крейт бэкенда | Вывод | Сценарий использования |
|--------------|--------|----------|
| `env_logger` | stderr, цветной | Разработка, простые CLI-инструменты |
| `tracing-subscriber` | stderr, форматированный | Продакшн с использованием `tracing` |
| `syslog` | Системный syslog | Системные службы Linux |
| `tracing-journald` | Журнал systemd | Службы под управлением systemd |
| `tracing-appender` | Ротируемые файлы логов | Демоны (фоновые процессы) с долгим временем работы |
| `tracing-opentelemetry` | Коллектор OpenTelemetry | Распределенная трассировка |

----
