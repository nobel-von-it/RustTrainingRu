## Шаблоны тестирования для C++ программистов

> **Что вы узнаете:** Встроенную систему тестирования Rust — `#[test]`, `#[should_panic]`, тесты, возвращающие `Result`, шаблоны проектирования (строители/builders) для тестовых данных, имитацию (mocking) на основе типажей, тестирование свойств с `proptest`, моментальные снимки (snapshot testing) с `insta` и структуру интеграционных тестов. Это тестирование без конфигурации, заменяющее связку Google Test + CMake.

Тестирование на C++ обычно полагается на внешние фреймворки (Google Test, Catch2, Boost.Test) со сложной интеграцией в сборку. Система тестирования Rust **встроена непосредственно в язык и инструментарий** — никаких зависимостей, интеграции с CMake или настройки средства запуска тестов.

### Атрибуты тестирования помимо `#[test]`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic_pass() {
        assert_eq!(2 + 2, 4);
    }

    // Ожидание паники — эквивалент EXPECT_DEATH в GTest
    #[test]
    #[should_panic]
    fn out_of_bounds_panics() {
        let v = vec![1, 2, 3];
        let _ = v[10]; // Паникует — тест пройден
    }

    // Ожидание паники с определенной подстрокой в сообщении об ошибке
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn specific_panic_message() {
        let v = vec![1, 2, 3];
        let _ = v[10];
    }

    // Тесты, которые возвращают Result<(), E> — использование ? вместо unwrap()
    #[test]
    fn test_with_result() -> Result<(), String> {
        let value: u32 = "42".parse().map_err(|e| format!("{e}"))?;
        assert_eq!(value, 42);
        Ok(())
    }

    // Игнорировать медленные тесты по умолчанию — запускать с `cargo test -- --ignored`
    #[test]
    #[ignore]
    fn slow_integration_test() {
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}
```

```bash
cargo test                          # Запуск всех неигнорируемых тестов
cargo test -- --ignored             # Запуск только игнорируемых тестов
cargo test -- --include-ignored     # Запуск ВСЕХ тестов, включая игнорируемые
cargo test test_name                # Запуск тестов, соответствующих шаблону имени
cargo test -- --nocapture           # Показывать вывод макроса println! во время тестов
cargo test -- --test-threads=1      # Запускать тесты последовательно (для общего состояния)
```

### Вспомогательные элементы: шаблон строитель (builder) для тестовых данных

В C++ вы бы использовали тестовое окружение Google Test (`class MyTest : public ::testing::Test`). В Rust используйте функции-строители или типаж `Default`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Функция-строитель — создает тестовые данные с разумными значениями по умолчанию
    fn make_gpu_event(severity: Severity, fault_code: u32) -> DiagEvent {
        DiagEvent {
            source: "accel_diag".to_string(),
            severity,
            message: format!("Test event FC:{fault_code}"),
            fault_code,
        }
    }

    // Переиспользуемое тестовое окружение — набор предопределенных событий
    fn sample_events() -> Vec<DiagEvent> {
        vec![
            make_gpu_event(Severity::Critical, 67956),
            make_gpu_event(Severity::Warning, 32709),
            make_gpu_event(Severity::Info, 10001),
        ]
    }

    #[test]
    fn filter_critical_events() {
        let events = sample_events();
        let critical: Vec<_> = events.iter()
            .filter(|e| e.severity == Severity::Critical)
            .collect();
        assert_eq!(critical.len(), 1);
        assert_eq!(critical[0].fault_code, 67956);
    }
}
```

### Имитация (mocking) с помощью типажей

В C++ имитация требует использования фреймворков вроде Google Mock или ручного переопределения виртуальных методов. В Rust вы определяете типаж для зависимости и заменяете реализации в тестах:

```rust
// Промышленный типаж
trait SensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String>;
}

// Промышленная реализация
struct HwSensorReader;
impl SensorReader for HwSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        // Реальный вызов оборудования...
        Ok(72.5)
    }
}

// Тестовая имитация — возвращает предсказуемые значения
#[cfg(test)]
struct MockSensorReader {
    temperatures: std::collections::HashMap<u32, f64>,
}

#[cfg(test)]
impl SensorReader for MockSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        self.temperatures.get(&sensor_id)
            .copied()
            .ok_or_else(|| format!("Unknown sensor {sensor_id}"))
    }
}

// Тестируемая функция — обобщенная по отношению к SensorReader
fn check_overtemp(reader: &impl SensorReader, ids: &[u32], threshold: f64) -> Vec<u32> {
    ids.iter()
        .filter(|&&id| reader.read_temperature(id).unwrap_or(0.0) > threshold)
        .copied()
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn detect_overtemp_sensors() {
        let mut mock = MockSensorReader { temperatures: Default::default() };
        mock.temperatures.insert(0, 72.5);
        mock.temperatures.insert(1, 91.0);  // Выше порогового значения
        mock.temperatures.insert(2, 65.0);

        let hot = check_overtemp(&mock, &[0, 1, 2], 80.0);
        assert_eq!(hot, vec![1]);
    }
}
```

### Временные файлы и каталоги в тестах

Тесты в C++ часто используют платформозависимые временные папки. В Rust есть пакет `tempfile`:

```rust
// Cargo.toml: [dev-dependencies]
// tempfile = "3"

#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::NamedTempFile;
    use std::io::Write;

    #[test]
    fn parse_config_from_file() -> Result<(), Box<dyn std::error::Error>> {
        // Создаем временный файл, который автоматически удаляется при сбросе (drop)
        let mut file = NamedTempFile::new()?;
        writeln!(file, r#"{{"sku": "ServerNode", "level": "Quick"}}"#)?;

        let config = load_config(file.path().to_str().unwrap())?;
        assert_eq!(config.sku, "ServerNode");
        Ok(())
        // Файл удаляется здесь — код очистки не требуется
    }
}
```

### Тестирование на основе свойств с `proptest`

Вместо написания конкретных тестовых случаев опишите **свойства**, которые должны сохраняться для любых входных данных. Пакет `proptest` генерирует случайные входные данные и находит минимально воспроизводимый случай сбоя:

```rust
// Cargo.toml: [dev-dependencies]
// proptest = "1"

#[cfg(test)]
mod tests {
    use proptest::prelude::*;

    fn parse_and_format(n: u32) -> String {
        format!("{n}")
    }

    proptest! {
        #[test]
        fn roundtrip_u32(n: u32) {
            let formatted = parse_and_format(n);
            let parsed: u32 = formatted.parse().unwrap();
            prop_assert_eq!(n, parsed);
        }

        #[test]
        fn string_contains_no_null(s in "[a-zA-Z0-9 ]{0,100}") {
            prop_assert!(!s.contains('\0'));
        }
    }
}
```

### Снимок-тестирование (Snapshot testing) с помощью `insta`

Для тестов, которые производят сложный вывод (JSON, форматированные строки), `insta` автоматически генерирует снимки (snapshots) и управляет ими:

```rust
// Cargo.toml: [dev-dependencies]
// insta = { version = "1", features = ["json"] }

#[cfg(test)]
mod tests {
    use insta::assert_json_snapshot;

    #[test]
    fn der_entry_format() {
        let entry = DerEntry {
            fault_code: 67956,
            component: "GPU".to_string(),
            message: "ECC error detected".to_string(),
        };
        // Первый запуск: создает файл снимка в каталоге tests/snapshots/
        // Последующие запуски: сравнивает вывод с сохраненным снимком
        assert_json_snapshot!(entry);
    }
}
```

```bash
cargo insta test              # Запуск тестов и проверка новых/изменившихся снимков
cargo insta review            # Интерактивный просмотр изменений снимков
```

### Сравнение тестирования в C++ и Rust

| **C++ (Google Test)** | **Rust** | **Примечания** |
|----------------------|---------|----------|
| `TEST(Suite, Name) { }` | `#[test] fn name() { }` | Иерархия наборов/классов не требуется |
| `ASSERT_EQ(a, b)` | `assert_eq!(a, b)` | Встроенный макрос, фреймворк не требуется |
| `ASSERT_NEAR(a, b, eps)` | `assert!((a - b).abs() < eps)` | Или используйте пакет `approx` |
| `EXPECT_THROW(expr, type)` | `#[should_panic(expected = "...")]` | Или `catch_unwind` для более тонкого контроля |
| `EXPECT_DEATH(expr, "msg")` | `#[should_panic(expected = "msg")]` | |
| `class Fixture : public ::testing::Test` | builder-функции + `Default` | Наследование не требуется |
| Google Mock `MOCK_METHOD` | Типаж + реализация для тестов | Более явно, без магии макросов |
| `INSTANTIATE_TEST_SUITE_P` (параметризованные) | `proptest!` или сгенерированные макросом тесты | |
| `SetUp()` / `TearDown()` | RAII через `Drop` — очистка автоматическая | Переменные удаляются в конце теста |
| Отдельный тестовый бинарник + CMake | `cargo test` — нулевая конфигурация | |
| `ctest --output-on-failure` | `cargo test -- --nocapture` | |

----

### Интеграционные тесты: каталог `tests/`

Модульные тесты живут внутри модулей с атрибутом `#[cfg(test)]` рядом с вашим кодом. **Интеграционные тесты** живут в отдельном каталоге `tests/` в корне пакета и проверяют только публичный API вашей библиотеки с точки зрения внешнего потребителя:

```
my_crate/
├── src/
│   └── lib.rs          # Код вашей библиотеки
├── tests/
│   ├── smoke.rs        # Каждый файл .rs представляет собой отдельный тестовый бинарный файл
│   ├── regression.rs
│   └── common/
│       └── mod.rs      # Общие тестовые помощники (сами по себе не являются тестом)
└── Cargo.toml
```

```rust
// tests/smoke.rs — тестирует ваш пакет с точки зрения внешнего пользователя
use my_crate::DiagEngine;  // Доступен только публичный API

#[test]
fn engine_starts_successfully() {
    let engine = DiagEngine::new("test_config.json");
    assert!(engine.is_ok());
}

#[test]
fn engine_rejects_invalid_config() {
    let engine = DiagEngine::new("nonexistent.json");
    assert!(engine.is_err());
}
```

```rust
// tests/common/mod.rs — общие помощники, НЕ компилируемые как тестовый бинарный файл
pub fn setup_test_environment() -> tempfile::TempDir {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("config.json"), r#"{"log_level": "debug"}"#).unwrap();
    dir
}
```

```rust
// tests/regression.rs — может использовать общие помощники
mod common;

#[test]
fn regression_issue_42() {
    let env = common::setup_test_environment();
    let engine = my_crate::DiagEngine::new(
        env.path().join("config.json").to_str().unwrap()
    );
    assert!(engine.is_ok());
}
```

**Запуск интеграционных тестов:**
```bash
cargo test                          # Запускает модульные И интеграционные тесты
cargo test --test smoke             # Запуск только тестов из tests/smoke.rs
cargo test --test regression        # Запуск только тестов из tests/regression.rs
cargo test --lib                    # Запуск ТОЛЬКО модульных тестов (пропуская интеграционные)
```

> **Ключевое отличие от модульных тестов**: интеграционные тесты не могут получить доступ к приватным функциям или элементам с видимостью `pub(crate)`. Это заставляет вас убедиться, что вашего публичного API действительно достаточно для использования библиотеки. В терминах C++ это похоже на тестирование исключительно с подключением публичного заголовка без доступа через ключевое слово `friend`.

----
