# Ключевые выводы по Option и Result в Rust

> **Что вы узнаете:** Идиоматические шаблоны обработки ошибок — безопасные альтернативы `unwrap()`, оператор `?` для распространения ошибок, пользовательские типы ошибок, а также когда использовать `anyhow` и `thiserror` в промышленном коде.

- `Option` и `Result` являются неотъемлемой частью идиоматического Rust.
- **Безопасные альтернативы методу `unwrap()`**:
```rust
// Безопасные альтернативы для Option<T>
let value = opt.unwrap_or(default);              // Предоставить резервное значение (fallback)
let value = opt.unwrap_or_else(|| compute());    // Отложенное вычисление резервного значения
let value = opt.unwrap_or_default();             // Использовать реализацию типажа Default
let value = opt.expect("descriptive message");   // Только если паника допустима

// Безопасные альтернативы для Result<T, E>  
let value = result.unwrap_or(fallback);          // Проигнорировать ошибку, использовать резервное значение
let value = result.unwrap_or_else(|e| handle(e)); // Обработать ошибку, вернуть резервное значение
let value = result.unwrap_or_default();          // Использовать типаж Default
```
- **Сопоставление с шаблоном для явного управления**:
```rust
match some_option {
    Some(value) => println!("Got: {}", value),
    None => println!("No value found"),
}

match some_result {
    Ok(value) => process(value),
    Err(error) => log_error(error),
}
```
- **Использование оператора `?` для распространения ошибок**: немедленный возврат и передача ошибок выше по стеку вызовов
```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?; // Автоматически возвращает ошибку
    Ok(content.to_uppercase())
}
```
- **Методы преобразования**:
    - `map()`: преобразует успешное значение `Ok(T)` -> `Ok(U)` or `Some(T)` -> `Some(U)`.
    - `map_err()`: преобразует тип ошибки `Err(E)` -> `Err(F)`.
    - `and_then()`: последовательное связывание в цепочку операций, которые могут завершиться ошибкой.
- **Использование в собственных API**: отдавайте предпочтение `Result<T, E>` перед исключениями или кодами ошибок.
- **Справочники**: [Документация по Option](https://doc.rust-lang.org/std/option/enum.Option.html) | [Документация по Result](https://doc.rust-lang.org/std/result/enum.Result.html)

# Распространенные ошибки в Rust и советы по отладке
- **Проблемы с заимствованием**: самая частая ошибка новичков
    - "cannot borrow as mutable" -> разрешена только одна изменяемая ссылка одновременно.
    - "borrowed value does not live long enough" -> ссылка живет дольше, чем данные, на которые она указывает.
    - **Решение**: используйте блоки видимости `{}` для ограничения времени жизни ссылок или клонируйте данные при необходимости.
- **Отсутствие реализаций типажей**: ошибки вида "method not found"
    - **Решение**: добавьте `#[derive(Debug, Clone, PartialEq)]` для реализации часто используемых типажей.
    - Используйте команду `cargo check` для получения более информативных сообщений об ошибках по сравнению с `cargo run`.
- **Переполнение целых чисел в режиме отладки**: Rust паникует при переполнении
    - **Решение**: используйте методы `wrapping_add()`, `saturating_add()` или `checked_add()` для явного указания желаемого поведения.
- **Путаница со String и &str**: разные типы для разных сценариев использования
    - Используйте `&str` для срезов строк (заимствованные строки), а `String` — для строк, которыми вы владеете.
    - **Решение**: используйте методы `.to_string()` или `String::from()` для преобразования `&str` в `String`.
- **Борьба с проверкой заимствований (borrow checker)**: не пытайтесь перехитрить ее
    - **Решение**: перестройте структуру кода так, чтобы он работал в соответствии с правилами владения, а не вопреки им.
    - Рассмотрите возможность использования `Rc<RefCell<T>>` для сложных сценариев совместного использования (но пользуйтесь этим умеренно).

## Примеры обработки ошибок: хорошо против плохо
```rust
// [ОШИБКА] ПЛОХО: может неожиданно вызвать панику
fn bad_config_reader() -> String {
    let config = std::env::var("CONFIG_FILE").unwrap(); // Паника, если переменная не установлена!
    std::fs::read_to_string(config).unwrap()           // Паника, если файл отсутствует!
}

// [OK] ХОРОШО: корректно обрабатывает ошибки
fn good_config_reader() -> Result<String, ConfigError> {
    let config_path = std::env::var("CONFIG_FILE")
        .unwrap_or_else(|_| "default.conf".to_string()); // Переход к резервному значению

    let content = std::fs::read_to_string(config_path)
        .map_err(ConfigError::FileRead)?;                // Преобразование и передача ошибки выше

    Ok(content)
}

// [OK] ЕЩЕ ЛУЧШЕ: с использованием подходящих типов ошибок
use thiserror::Error;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("Failed to read config file: {0}")]
    FileRead(#[from] std::io::Error),

    #[error("Invalid configuration: {message}")]
    Invalid { message: String },
}
```

Разберем, что здесь происходит. Перечисление `ConfigError` имеет всего **два варианта** — один для ошибок ввода-вывода (I/O) и один для ошибок валидации. Это правильная отправная точка для большинства модулей:

| Вариант `ConfigError` | Содержимое | Чем создается |
|----------------------|-------|-----------|
| `FileRead(io::Error)` | Исходная ошибка ввода-вывода | `#[from]` автоматически преобразует её с помощью оператора `?` |
| `Invalid { message }` | Человекочитаемое объяснение | Ваш код валидации |

Теперь вы можете писать функции, возвращающие `Result<T, ConfigError>`:

```rust
fn read_config(path: &str) -> Result<String, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError::FileRead
    if content.is_empty() {
        return Err(ConfigError::Invalid {
            message: "config file is empty".to_string(),
        });
    }
    Ok(content)
}
```

> **🟢 Контрольная точка для самопроверки:** Прежде чем продолжить, убедитесь, что можете ответить на вопросы:
> 1. Почему работает оператор `?` при вызове `read_to_string`? (Потому что атрибут `#[from]` генерирует `impl From<io::Error> for ConfigError`).
> 2. Что произойдет, если добавить третий вариант `MissingKey(String)` — какие изменения потребуются в коде? (Просто добавьте вариант; существующий код продолжит компилироваться).

## Типы ошибок на уровне пакета и псевдонимы Result

По мере роста вашего проекта за рамки одного файла вы начнете объединять несколько ошибок на уровне модулей в один **тип ошибок на уровне пакета (crate-level error type)**. Это стандартный шаблон проектирования в промышленном Rust. Давайте разовьем пример на основе `ConfigError`, описанного выше.

В реальных проектах на Rust каждый пакет (или крупный модуль) определяет свое собственное перечисление `Error` и псевдоним типа `Result`. Это идиоматический шаблон, аналогичный тому, как в C++ вы определяете иерархию исключений для библиотеки и объявляете псевдоним `using Result = std::expected<T, Error>`.

### Шаблон проектирования

```rust
// src/error.rs  (или в начале файла lib.rs)
use thiserror::Error;

/// Любая ошибка, которую может сгенерировать данный пакет.
#[derive(Error, Debug)]
pub enum Error {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),          // автоматическое преобразование через From

    #[error("JSON parse error: {0}")]
    Json(#[from] serde_json::Error),     // автоматическое преобразование через From

    #[error("Invalid sensor id: {0}")]
    InvalidSensor(u32),                  // специфическая для домена ошибка

    #[error("Timeout after {ms} ms")]
    Timeout { ms: u64 },
}

/// Псевдоним Result на уровне пакета — избавляет от лишнего ввода кода во всем пакете.
pub type Result<T> = core::result::Result<T, Error>;
```

### Как это упрощает написание функций

Без использования псевдонима вам пришлось бы писать:

```rust
// Избыточно — тип ошибки повторяется повсюду
fn read_sensor(id: u32) -> Result<f64, crate::Error> { ... }
fn parse_config(path: &str) -> Result<Config, crate::Error> { ... }
```

С использованием псевдонима:

```rust
// Чисто — просто `Result<T>`
use crate::{Error, Result};

fn read_sensor(id: u32) -> Result<f64> {
    if id > 128 {
        return Err(Error::InvalidSensor(id));
    }
    let raw = std::fs::read_to_string(format!("/dev/sensor/{id}"))?; // io::Error → Error::Io
    let value: f64 = raw.trim().parse()
        .map_err(|_| Error::InvalidSensor(id))?;
    Ok(value)
}
```

Атрибут `#[from]` для варианта `Io` автоматически бесплатно генерирует следующую реализацию типажа `From`:

```rust
// Автоматически генерируется макросом #[from] библиотеки thiserror
impl From<std::io::Error> for Error {
    fn from(source: std::io::Error) -> Self {
        Error::Io(source)
    }
}
```

Именно благодаря этому работает оператор `?`: когда функция возвращает `std::io::Error`, а ваша функция возвращает `Result<T>` (ваш псевдоним), компилятор вызывает `From::from()` для ее автоматического преобразования.

### Объединение ошибок уровня модулей

Крупные пакеты разделяют ошибки по модулям, а затем объединяют их в корне пакета:

```rust
// src/config/error.rs
#[derive(thiserror::Error, Debug)]
pub enum ConfigError {
    #[error("Missing key: {0}")]
    MissingKey(String),
    #[error("Invalid value for '{key}': {reason}")]
    InvalidValue { key: String, reason: String },
}

// src/error.rs  (уровень пакета)
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error(transparent)]               // делегирует Display внутреннему типу ошибки
    Config(#[from] crate::config::ConfigError),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
pub type Result<T> = core::result::Result<T, Error>;
```

Вызывающий код по-прежнему может сопоставлять конкретные ошибки конфигурации:

```rust
match result {
    Err(Error::Config(ConfigError::MissingKey(k))) => eprintln!("Add '{k}' to config"),
    Err(e) => eprintln!("Other error: {e}"),
    Ok(v) => use_value(v),
}
```

### Сравнение с C++

| Концепция | C++ | Rust |
|---------|-----|------|
| Иерархия ошибок | `class AppError : public std::runtime_error` | `#[derive(thiserror::Error)] enum Error { ... }` |
| Возврат ошибки | `std::expected<T, Error>` или `throw` | `fn foo() -> Result<T>` |
| Преобразование ошибки | Ручной блок `try/catch` + повторный выброс исключения | `#[from]` + `?` — полное отсутствие шаблонного кода |
| Псевдоним Result | `template<class T> using Result = std::expected<T, Error>;` | `pub type Result<T> = core::result::Result<T, Error>;` |
| Сообщение об ошибке | Переопределение `what()` | `#[error("...")]` — преобразуется в реализацию типажа `Display` |
