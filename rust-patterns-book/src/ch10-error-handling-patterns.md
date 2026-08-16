# 10. Шаблоны обработки ошибок 🟢

> **Чему вы научитесь:**
> - Когда использовать `thiserror` (библиотеки), а когда `anyhow` (приложения)
> - Цепочки преобразования ошибок с помощью `#[from]` и обёрток `.context()`
> - Как раскрывается оператор `?` и как он работает в `main()`
> - Когда вызывать панику (`panic`), а когда возвращать ошибки, и использование `catch_unwind` для границ FFI

## thiserror против anyhow — библиотеки против приложений

Обработка ошибок в Rust сосредоточена вокруг типа `Result<T, E>`. Здесь доминируют две библиотеки (crates):

```rust,ignore
// --- thiserror: Для БИБЛИОТЕК ---
// Генерирует реализации Display, Error и From с помощью макросов derive
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("ошибка соединения: {0}")]
    ConnectionFailed(String),

    #[error("ошибка запроса: {source}")]
    QueryError {
        #[source]
        source: sqlx::Error,
    },

    #[error("запись не найдена: table={table} id={id}")]
    NotFound { table: String, id: u64 },

    #[error(transparent)] // Делегирует Display внутреннему типу ошибки
    Io(#[from] std::io::Error), // Автоматически генерирует From<io::Error>
}

// --- anyhow: Для ПРИЛОЖЕНИЙ ---
// Динамический тип ошибки — отлично подходит для высокоуровневого кода, где нужно просто передавать ошибки выше
use anyhow::{Context, Result, bail, ensure};

fn read_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("не удалось прочитать конфигурацию из {path}"))?;

    let config: Config = serde_json::from_str(&content)
        .context("не удалось разобрать JSON конфигурации")?;

    ensure!(config.port > 0, "порт должен быть положительным, получено {}", config.port);

    Ok(config)
}

fn main() -> Result<()> {
    let config = read_config("server.toml")?;

    if config.name.is_empty() {
        bail!("имя сервера не может быть пустым"); // Немедленно возвращает Err
    }

    Ok(())
}
```

**Когда что использовать**:

| | `thiserror` | `anyhow` |
|---|---|---|
| **Использование в** | Библиотеках, общих пакетах (crates) | Приложениях, бинарных файлах |
| **Типы ошибок** | Конкретные перечисления (enums) — вызывающая сторона может делать по ним сопоставление | `anyhow::Error` — непрозрачный тип |
| **Сложность** | Необходимо определить собственное перечисление ошибок | Достаточно использовать `Result<T>` |
| **Приведение типов (Downcasting)** | Не требуется — используется сопоставление с образцом | `error.downcast_ref::<MyError>()` |

### Цепочки преобразования ошибок (#[from])

```rust,ignore
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("ошибка ввода-вывода (I/O): {0}")]
    Io(#[from] std::io::Error),

    #[error("ошибка JSON: {0}")]
    Json(#[from] serde_json::Error),

    #[error("ошибка HTTP: {0}")]
    Http(#[from] reqwest::Error),
}

// Теперь оператор ? автоматически преобразует типы:
fn fetch_and_parse(url: &str) -> Result<Config, AppError> {
    let body = reqwest::blocking::get(url)?.text()?;  // reqwest::Error → AppError::Http
    let config: Config = serde_json::from_str(&body)?; // serde_json::Error → AppError::Json
    Ok(config)
}
```

### Контекст и оборачивание ошибок

Добавляйте понятный человеку контекст к ошибкам без потери исходной ошибки:

```rust,ignore
use anyhow::{Context, Result};

fn process_file(path: &str) -> Result<Data> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("не удалось прочитать {path}"))?;

    let data = parse_content(&content)
        .with_context(|| format!("не удалось разобрать {path}"))?;

    validate(&data)
        .context("валидация завершилась ошибкой")?;

    Ok(data)
}

// Вывод ошибки:
// Error: валидация завершилась ошибкой
//
// Вызвано:
//    0: не удалось разобрать config.json
//    1: ожидалась ',' на строке 5 в колонке 12
```

### Подробно об операторе ?

Оператор `?` — это синтаксический сахар для конструкции `match` с преобразованием `From` и ранним возвратом:

```rust
// Это:
let value = operation()?;

// Раскрывается в:
let value = match operation() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
    //                  ^^^^^^^^^^^^^^
    //                  Автоматическое преобразование через типаж From
};
```

**Оператор `?` также работает с `Option`** (в функциях, возвращающих `Option`):

```rust
fn find_user_email(users: &[User], name: &str) -> Option<String> {
    let user = users.iter().find(|u| u.name == name)?; // Возвращает None, если не найдено
    let email = user.email.as_ref()?; // Возвращает None, если email равен None
    Some(email.to_uppercase())
}
```

### Паника, catch_unwind и когда нужно аварийно завершать работу (Abort)

```rust
// Паника: для ОШИБОК ПРОГРАММИРОВАНИЯ (багов), а не ожидаемых ошибок
fn get_element(data: &[i32], index: usize) -> &i32 {
    // Если здесь происходит паника, это ошибка программирования (баг).
    // Не надо «обрабатывать» её — исправьте вызывающий код.
    &data[index]
}

// catch_unwind: для границ сред выполнения (FFI, пулы потоков)
use std::panic;

let result = panic::catch_unwind(|| {
    // Безопасный запуск кода, который потенциально может вызвать панику
    risky_operation()
});

match result {
    Ok(value) => println!("Успех: {value:?}"),
    Err(_) => eprintln!("Операция вызвала панику — продолжаем работу в безопасном режиме"),
}

// Когда что использовать:
// - Result<T, E> → ожидаемые сбои (файл не найден, таймаут сети)
// - panic!()     → баги в коде (выход индекса за границы, нарушение инварианта)
// - process::abort() → невосстановимое состояние (нарушение безопасности, повреждение данных)
```

> **Сравнение с C++**: `Result<T, E>` заменяет исключения для ожидаемых ошибок.
> `panic!()` похож на `assert()` или `std::terminate()` — он предназначен для багов, а не
> для управления потоком выполнения. Оператор `?` в Rust делает распространение ошибок столь же эргономичным, как
> и исключения, но без непредсказуемого потока управления.

> **Основные выводы — Обработка ошибок**
> - Библиотеки: `thiserror` для структурированных перечислений ошибок; приложения: `anyhow` для эргономичного распространения ошибок.
> - `#[from]` автоматически генерирует реализации `From`; `.context()` добавляет понятные человеку обёртки.
> - `?` раскрывается в `From::from()` + ранний возврат; работает в функции `main()`, если та возвращает `Result`.

> **См. также:** [Глава 14 — Проектирование API](ch15-crate-architecture-and-api-design.md) для шаблонов «разбирайте, а не валидируйте» (parse, don't validate). [Глава 11 — Сериализация](ch11-serialization-zero-copy-and-binary-data.md) для обработки ошибок в serde.

```mermaid
flowchart LR
    A["std::io::Error"] -->|"#[from]"| B["AppError::Io"]
    C["serde_json::Error"] -->|"#[from]"| D["AppError::Json"]
    E["Пользовательская валидация"] -->|"manual"| F["AppError::Validation"]

    B --> G["Оператор ?"]
    D --> G
    F --> G
    G --> H["Result&lt;T, AppError&gt;"]

    style A fill:#e8f4f8,stroke:#2980b9,color:#000
    style C fill:#e8f4f8,stroke:#2980b9,color:#000
    style E fill:#e8f4f8,stroke:#2980b9,color:#000
    style B fill:#fdebd0,stroke:#e67e22,color:#000
    style D fill:#fdebd0,stroke:#e67e22,color:#000
    style F fill:#fdebd0,stroke:#e67e22,color:#000
    style G fill:#fef9e7,stroke:#f1c40f,color:#000
    style H fill:#d4efdf,stroke:#27ae60,color:#000
```

---

### Упражнение: Иерархия ошибок с помощью thiserror ★★ (~30 мин)

Разработайте иерархию типов ошибок для приложения обработки файлов, работа которого может завершиться ошибкой во время ввода-вывода (I/O), парсинга (JSON и CSV) и валидации. Используйте библиотеку `thiserror` и продемонстрируйте распространение ошибок с помощью оператора `?`.

<details>
<summary>🔑 Решение</summary>

```rust,ignore
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("ошибка ввода-вывода (I/O): {0}")]
    Io(#[from] std::io::Error),

    #[error("ошибка парсинга JSON: {0}")]
    Json(#[from] serde_json::Error),

    #[error("ошибка CSV на строке {line}: {message}")]
    Csv { line: usize, message: String },

    #[error("ошибка валидации: {field} — {reason}")]
    Validation { field: String, reason: String },
}

fn read_file(path: &str) -> Result<String, AppError> {
    Ok(std::fs::read_to_string(path)?) // io::Error → AppError::Io через #[from]
}

fn parse_json(content: &str) -> Result<serde_json::Value, AppError> {
    Ok(serde_json::from_str(content)?) // serde_json::Error → AppError::Json
}

fn validate_name(value: &serde_json::Value) -> Result<String, AppError> {
    let name = value.get("name")
        .and_then(|v| v.as_str())
        .ok_or_else(|| AppError::Validation {
            field: "name".into(),
            reason: "должно быть ненулевой строкой".into(),
        })?;

    if name.is_empty() {
        return Err(AppError::Validation {
            field: "name".into(),
            reason: "не должно быть пустым".into(),
        });
    }

    Ok(name.to_string())
}

fn process_file(path: &str) -> Result<String, AppError> {
    let content = read_file(path)?;
    let json = parse_json(&content)?;
    let name = validate_name(&json)?;
    Ok(name)
}

fn main() {
    match process_file("config.json") {
        Ok(name) => println!("Имя: {name}"),
        Err(e) => eprintln!("Ошибка: {e}"),
    }
}
```

</details>

***
