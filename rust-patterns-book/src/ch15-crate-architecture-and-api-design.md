# 15. Архитектура крейтов и проектирование API 🟡

> **Что вы узнаете:**
> - Соглашения о структуре модулей и стратегии реэкспорта
> - Контрольный список проектирования публичного API для качественных крейтов
> - Эргономичные паттерны параметров: `impl Into`, `AsRef`, `Cow`
> - Подход «парсите, а не валидируйте» (Parse, don't validate) с использованием `TryFrom` и валидированных типов
> - Feature-флаги, условная компиляция и организация рабочих пространств (workspaces)

## Соглашения о структуре модулей

```text
my_crate/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Корневой файл крейта — реэкспорты и публичный API
│   ├── config.rs       # Модуль конфигурации (фичи)
│   ├── parser/         # Сложный модуль с подмодулями
│   │   ├── mod.rs      # или parser.rs на родительском уровне (Rust 2018+)
│   │   ├── lexer.rs
│   │   └── ast.rs
│   ├── error.rs        # Типы ошибок
│   └── utils.rs        # Внутренние вспомогательные элементы (pub(crate))
├── tests/
│   └── integration.rs  # Интеграционные тесты
├── benches/
│   └── perf.rs         # Бенчмарки
└── examples/
    └── basic.rs        # cargo run --example basic
```

```rust
// lib.rs — формируйте ваш публичный API с помощью реэкспорта:
mod config;
mod error;
mod parser;
mod utils;

// Реэкспортируйте то, что нужно пользователям:
pub use config::Config;
pub use error::Error;
pub use parser::Parser;

// Публичные типы находятся в корне крейта — пользователи пишут:
// use my_crate::Config;
// НО НЕ: use my_crate::config::Config;
```

**Модификаторы видимости**:

| Модификатор | Кому виден |
|-------------|------------|
| `pub` | Всем |
| `pub(crate)` | Только внутри этого крейта |
| `pub(super)` | Родителю текущего модуля |
| `pub(in path)` | Конкретному модулю-предку (по пути) |
| (нет) | Текущему модулю и его дочерним модулям |

### Контрольный список для проектирования публичного API

1. **Принимайте ссылки, возвращайте владеемые типы** — `fn process(input: &str) -> String`
2. **Используйте `impl Trait` для параметров** — `fn read(r: impl Read)` вместо `fn read<R: Read>(r: R)` для более лаконичных сигнатур
3. **Возвращайте `Result`, а не `panic!`** — пусть вызывающий код решает, как обрабатывать ошибки
4. **Реализуйте стандартные типажи (traits)** — `Debug`, `Display`, `Clone`, `Default`, `From`/`Into`
5. **Делайте невалидные состояния невыразимыми** — используйте состояния типов (type states) и новые типы (newtypes)
6. **Следуйте паттерну «строитель» (builder) для сложных конфигураций** — со свойствами состояний типов (type-state), если поля обязательны
7. **Запечатывайте типажи (seal traits), которые пользователи не должны реализовывать самостоятельно** — `pub trait Sealed: private::Sealed {}`
8. **Помечайте типы и функции атрибутом `#[must_use]`** — это предотвращает случайное игнорирование важных `Result`, гардов (guards) или значений. Применяйте к любым типам, где игнорирование возвращаемого значения почти наверняка является ошибкой:
   ```rust
   #[must_use = "dropping the guard immediately releases the lock"]
   pub struct LockGuard<'a, T> { /* ... */ }

   #[must_use]
   pub fn validate(input: &str) -> Result<ValidInput, ValidationError> { /* ... */ }
   ```

```rust
// Паттерн «запечатанный типаж» (Sealed trait pattern) — пользователи могут использовать его, но не реализовывать:
mod private {
    pub trait Sealed {}
}

pub trait DatabaseDriver: private::Sealed {
    fn connect(&self, url: &str) -> Connection;
}

// Только типы внутри ЭТОГО крейта могут реализовать Sealed → только мы можем реализовать DatabaseDriver
pub struct PostgresDriver;
impl private::Sealed for PostgresDriver {}
impl DatabaseDriver for PostgresDriver {
    fn connect(&self, url: &str) -> Connection { /* ... */ }
}
```

> **`#[non_exhaustive]`** — помечайте этим атрибутом публичные перечисления (enums) и структуры, чтобы добавление новых вариантов или полей не приводило к критическим изменениям (breaking changes). Зависимые крейты будут вынуждены использовать шаблон соответствия по умолчанию (`_ =>`) в выражениях `match`, а также не смогут конструировать тип с помощью синтаксиса структурных литералов:
> ```rust
> #[non_exhaustive]
> pub enum DiagError {
>     Timeout,
>     HardwareFault,
>     // Добавление нового варианта в будущем релизе НЕ нарушает семантическое версионирование (semver).
> }
> ```

### Эргономичные паттерны параметров — `impl Into`, `AsRef`, `Cow`

Один из наиболее эффективных паттернов API в Rust — это принятие **наиболее общего типа** в параметрах функций, чтобы вызывающей стороне не приходилось многократно писать `.to_string()`, `&*s` или `.as_ref()` при каждом вызове. Это специфичная для Rust версия принципа «будьте либеральны к тому, что принимаете».

#### `impl Into<T>` — Принятие любых преобразуемых типов

```rust
// ❌ Неудобно: вызывающий код должен выполнять преобразование вручную
fn connect(host: String, port: u16) -> Connection {
    // ...
}
connect("localhost".to_string(), 5432);  // Раздражающий .to_string()
connect(hostname.clone(), 5432);          // Ненужный клон, если у нас уже есть String

// ✅ Эргономично: принимаем все, что может быть преобразовано в String
fn connect(host: impl Into<String>, port: u16) -> Connection {
    let host = host.into();  // Преобразуем один раз внутри функции
    // ...
}
connect("localhost", 5432);     // &str — без лишних усилий
connect(hostname, 5432);        // String — перемещено, без клонирования
```

Это работает, потому что пара типажей `From`/`Into` в Rust предоставляет обобщенные реализации преобразований (blanket conversions). Когда вы принимаете `impl Into<T>`, вы говорите: «дайте мне всё, что умеет превращаться в `T`».

#### `AsRef<T>` — Заимствование в виде ссылки

`AsRef<T>` — это аналог `Into<T>` для заимствования. Используйте его, когда вам нужно только *прочитать* данные, а не забрать владение:

```rust
use std::path::Path;

// ❌ Заставляет вызывающий код выполнять преобразование в &Path
fn file_exists(path: &Path) -> bool {
    path.exists()
}
file_exists(Path::new("/tmp/test.txt"));  // Неудобно

// ✅ Принимает все, что может вести себя как &Path
fn file_exists(path: impl AsRef<Path>) -> bool {
    path.as_ref().exists()
}
file_exists("/tmp/test.txt");                    // &str ✅
file_exists(String::from("/tmp/test.txt"));      // String ✅
file_exists(Path::new("/tmp/test.txt"));         // &Path ✅
file_exists(PathBuf::from("/tmp/test.txt"));     // PathBuf ✅

// Тот же паттерн для строкоподобных параметров:
fn log_message(msg: impl AsRef<str>) {
    println!("[LOG] {}", msg.as_ref());
}
log_message("hello");                    // &str ✅
log_message(String::from("hello"));      // String ✅
```

#### `Cow<T>` — Клонирование при записи (Clone on Write)

`Cow<'a, T>` (Clone on Write, клонирование при записи) откладывает выделение памяти до тех пор, пока не потребуется изменение данных. Оно хранит либо заимствованное значение `&T`, либо владеемое `T::Owned`. Это идеальный выбор, когда в большинстве вызовов модифицировать данные не нужно:

```rust
use std::borrow::Cow;

/// Нормализует диагностическое сообщение — выделяет память только при необходимости изменений.
fn normalize_message(msg: &str) -> Cow<'_, str> {
    if msg.contains('\t') || msg.contains('\r') {
        // Требуется выделение памяти — нужно изменить содержимое
        Cow::Owned(msg.replace('\t', "    ").replace('\r', ""))
    } else {
        // Без выделения памяти — просто заимствуем оригинал
        Cow::Borrowed(msg)
    }
}

// Большинство сообщений проходят без выделения памяти:
let clean = normalize_message("All tests passed");          // Заимствовано (Borrowed) — бесплатно
let fixed = normalize_message("Error:\tfailed\r\n");        // Владеется (Owned) — память выделена

// Cow<str> реализует Deref<Target=str>, поэтому работает так же, как &str:
println!("{}", clean);
println!("{}", fixed.to_uppercase());
```

#### Краткий справочник: что использовать

```text
Нужно ли вам владение данными внутри функции?
├── ДА  → impl Into<T>
│         «Дайте мне всё, что может быть преобразовано в T»
└── НЕТ → Вам нужно только прочитать данные?
     ├── ДА  → impl AsRef<T> или &T
     │         «Дайте мне всё, что я могу заимствовать как &T»
     └── ВОЗМОЖНО (может потребоваться иногда изменять?)
          └── Cow<'_, T>
              «Заимствовать, если возможно; клонировать только при необходимости»
```

| Паттерн | Владение | Выделение памяти | Когда использовать |
|---------|-----------|------------|-------------|
| `&str` | Заимствованное | Никогда | Простые строковые параметры |
| `impl AsRef<str>` | Заимствованное | Никогда | Принятие String, &str и т. д. — только для чтения |
| `impl Into<String>` | Владеемое | При преобразовании | Принятие &str, String — для хранения/владения |
| `Cow<'_, str>` | И то, и другое | Только при модификации | Обработка, которая обычно не изменяет данные |
| `&[u8]` / `impl AsRef<[u8]>` | Заимствованное | Никогда | API, работающие с байтами |

> **`Borrow<T>` против `AsRef<T>`**: оба возвращают ссылку `&T`, но `Borrow<T>` дополнительно
> гарантирует, что поведение `Eq`, `Ord` и `Hash` является **согласованным** между исходной
> и заимствованной формами. Вот почему метод `HashMap<String, V>::get()` принимает тип `&Q, где String: Borrow<Q>`, а не `AsRef`. Используйте `Borrow`, когда заимствованная форма выступает в качестве ключа поиска; используйте `AsRef` для обычных параметров типа «предоставить ссылку».

#### Сочетание преобразований в API

```rust
/// Хорошо спроектированный API диагностики с использованием эргономичных параметров:
pub struct DiagRunner {
    name: String,
    config_path: PathBuf,
    results: HashMap<String, TestResult>,
}

impl DiagRunner {
    /// Принимает любой строкоподобный тип для name и любой путиподобный тип для config.
    pub fn new(
        name: impl Into<String>,
        config_path: impl Into<PathBuf>,
    ) -> Self {
        DiagRunner {
            name: name.into(),
            config_path: config_path.into(),
        }
    }

    /// Принимает любой AsRef<str> для поиска только для чтения.
    pub fn get_result(&self, test_name: impl AsRef<str>) -> Option<&TestResult> {
        self.results.get(test_name.as_ref())
    }
}

// Все эти варианты работают без лишних сложностей для вызывающей стороны:
let runner = DiagRunner::new("GPU Diag", "/etc/diag_tool/config.json");
let runner = DiagRunner::new(format!("Diag-{}", node_id), config_path);
let runner = DiagRunner::new(name_string, path_buf);
```

***

## Практический пример: проектирование публичного API крейта — До и после

Реальный пример эволюции внутреннего API со строковой типизацией (stringly-typed) в эргономичный, типизированный публичный API. Рассмотрим крейт для парсинга конфигурации:

**До** (со строковой типизацией, легко ошибиться при использовании):

```rust
// ❌ Все параметры являются строками — нет валидации на этапе компиляции
pub fn parse_config(path: &str, format: &str, strict: bool) -> Result<Config, String> {
    // Какие форматы валидны? "json"? "JSON"? "Json"?
    // Является ли path путем к файлу или URL?
    // Что вообще означает "strict"?
    todo!()
}
```

**После** (типобезопасный, самодокументируемый):

```rust
use std::path::Path;

/// Поддерживаемые форматы конфигурации.
#[derive(Debug, Clone, Copy)]
#[non_exhaustive]  // Добавление новых форматов в будущем не сломает код пользователей
pub enum Format {
    Json,
    Toml,
    Yaml,
}

/// Управляет строгостью парсинга.
#[derive(Debug, Clone, Copy, Default)]
pub enum Strictness {
    /// Отклонять неизвестные поля (по умолчанию для библиотек)
    #[default]
    Strict,
    /// Игнорировать неизвестные поля (полезно для конфигураций с обратной совместимостью)
    Lenient,
}

pub fn parse_config(
    path: &Path,          // Гарантировано типом: должен быть путь в файловой системе
    format: Format,       // Перечисление: невозможно передать невалидный формат
    strictness: Strictness,  // Именованные альтернативы вместо простого bool
) -> Result<Config, ConfigError> {
    todo!()
}
```

**Что улучшилось**:

| Аспект | До | После |
|--------|--------|-------|
| Валидация формата | Сравнение строк во время выполнения | Перечисление (enum) на этапе компиляции |
| Тип пути | Простой `&str` (может быть чем угодно) | `&Path` (специфичный для файловой системы) |
| Строгость | Непонятный `bool` | Самодокументируемое перечисление |
| Тип ошибки | `String` (непрозрачный) | `ConfigError` (структурированный) |
| Расширяемость | Критические изменения (breaking changes) | Использование `#[non_exhaustive]` |

> **Практическое правило**: Если вы ловите себя на написании `match` по строковым значениям,
> подумайте о замене параметра на перечисление (enum). Если параметр представляет собой логическое значение (boolean),
> смысл которого не очевиден из контекста, используйте вместо него enum с двумя вариантами.

***

### Парсите, а не валидируйте — `TryFrom` и валидированные типы

Принцип «парсите, а не валидируйте» (Parse, don't validate) гласит: **не проверяйте данные с тем, чтобы затем передавать их сырую непроверенную форму — вместо этого преобразуйте (спарсите) их в тип, который может существовать только в том случае, если данные валидны.** Стандартным инструментом для этого в Rust является типаж `TryFrom`.

#### Проблема: валидация без принуждения к соблюдению правил

```rust
// ❌ Сначала проверяем, потом используем: ничто не мешает использовать невалидное значение после проверки
fn process_port(port: u16) {
    if port == 0 || port > 65535 {
        panic!("Invalid port");           // Мы проверили, но...
    }
    start_server(port);                    // Что если кто-то вызовет start_server(0) напрямую?
}

// ❌ Строковая типизация (stringly-typed): электронная почта — это просто String, любой мусор пройдет
fn send_email(to: String, body: String) {
    // Является ли `to` действительно валидным email? Мы не знаем.
    // Кто-то может передать "not-an-email", и мы узнаем об этом только при отправке на SMTP-сервер.
}
```

#### Решение: парсинг в валидированные новые типы (newtypes) с помощью `TryFrom`

```rust
use std::convert::TryFrom;
use std::fmt;

/// Валидированный номер TCP-порта (1–65535).
/// Если у вас есть `Port`, он гарантированно валиден.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = PortError;

    fn try_from(value: u16) -> Result<Self, Self::Error> {
        if value == 0 {
            Err(PortError::Zero)
        } else {
            Ok(Port(value))
        }
    }
}

impl Port {
    pub fn get(&self) -> u16 { self.0 }
}

#[derive(Debug)]
pub enum PortError {
    Zero,
    InvalidFormat,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::Zero => write!(f, "порт не должен быть равен нулю"),
            PortError::InvalidFormat => write!(f, "неверный формат порта"),
        }
    }
}

impl std::error::Error for PortError {}

// Теперь валидность гарантируется системой типов:
fn start_server(port: Port) {
    // Валидация не требуется — Port может быть сконструирован только через TryFrom,
    // который уже проверил его корректность.
    println!("Listening on port {}", port.get());
}

// Использование:
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let port = Port::try_from(8080)?;   // ✅ Валидировано один раз на границе системы
    start_server(port);                  // Никакой повторной валидации далее по коду

    let bad = Port::try_from(0);         // ❌ Err(PortError::Zero)
    Ok(())
}
```

#### Реальный пример: валидированный IPMI-адрес

```rust
/// Валидированный подчиненный (slave) IPMI-адрес (0x20–0xFE, только четные).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct IpmiAddr(u8);

#[derive(Debug)]
pub enum IpmiAddrError {
    Odd(u8),
    OutOfRange(u8),
}

impl fmt::Display for IpmiAddrError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            IpmiAddrError::Odd(v) => write!(f, "IPMI-адрес 0x{v:02X} должен быть четным"),
            IpmiAddrError::OutOfRange(v) => {
                write!(f, "IPMI-адрес 0x{v:02X} вне диапазона (0x20..=0xFE)")
            }
        }
    }
}

impl TryFrom<u8> for IpmiAddr {
    type Error = IpmiAddrError;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        if value % 2 != 0 {
            Err(IpmiAddrError::Odd(value))
        } else if value < 0x20 || value > 0xFE {
            Err(IpmiAddrError::OutOfRange(value))
        } else {
            Ok(IpmiAddr(value))
        }
    }
}

impl IpmiAddr {
    pub fn get(&self) -> u8 { self.0 }
}

// Нижележащему коду больше никогда не нужно делать повторную проверку:
fn send_ipmi_command(addr: IpmiAddr, cmd: u8, data: &[u8]) -> Result<Vec<u8>, IpmiError> {
    // Гарантируется, что addr.get() является валидным четным IPMI-адресом
    raw_ipmi_send(addr.get(), cmd, data)
}
```

#### Парсинг строк с помощью `FromStr`

Для типов, которые часто парсятся из текста (аргументы командной строки, конфигурационные файлы), реализуйте `FromStr`:

```rust
use std::str::FromStr;

impl FromStr for Port {
    type Err = PortError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let n: u16 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        Port::try_from(n)
    }
}

// Теперь работает с .parse():
let port: Port = "8080".parse()?;   // Валидация выполняется в один шаг

// И с парсингом CLI-аргументов в clap:
// #[derive(Parser)]
// struct Args {
//     #[arg(short, long)]
//     port: Port,   // clap автоматически вызывает FromStr
// }
```

#### Цепочка `TryFrom` для сложной валидации

```rust
// Заглушки типов для этого примера — в реальном проекте они находились бы
// в отдельных модулях со своими собственными реализациями TryFrom.
```

```rust
# struct Hostname(String);
# impl TryFrom<String> for Hostname {
#     type Error = String;
#     fn try_from(s: String) -> Result<Self, String> { Ok(Hostname(s)) }
# }
# struct Timeout(u64);
# impl TryFrom<u64> for Timeout {
#     type Error = String;
#     fn try_from(ms: u64) -> Result<Self, String> {
#         if ms == 0 { Err("таймаут должен быть > 0".into()) } else { Ok(Timeout(ms)) }
#     }
# }
# struct RawConfig { host: String, port: u16, timeout_ms: u64 }
# #[derive(Debug)]
# enum ConfigError {
#     InvalidHost(String),
#     InvalidPort(PortError),
#     InvalidTimeout(String),
# }
# impl From<std::io::Error> for ConfigError {
#     fn from(e: std::io::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
# impl From<serde_json::Error> for ConfigError {
#     fn from(e: serde_json::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
/// Валидированная конфигурация, которая может существовать только при условии валидности всех полей.
pub struct ValidConfig {
    pub host: Hostname,
    pub port: Port,
    pub timeout_ms: Timeout,
}

impl TryFrom<RawConfig> for ValidConfig {
    type Error = ConfigError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        Ok(ValidConfig {
            host: Hostname::try_from(raw.host)
                .map_err(ConfigError::InvalidHost)?,
            port: Port::try_from(raw.port)
                .map_err(ConfigError::InvalidPort)?,
            timeout_ms: Timeout::try_from(raw.timeout_ms)
                .map_err(ConfigError::InvalidTimeout)?,
        })
    }
}

// Парсим один раз на границе, используем валидированный тип везде внутри:
fn load_config(path: &str) -> Result<ValidConfig, ConfigError> {
    let raw: RawConfig = serde_json::from_str(&std::fs::read_to_string(path)?)?;
    ValidConfig::try_from(raw)  // Вся валидация происходит здесь
}
```

#### Резюме: Валидация против Парсинга

| Подход | Данные проверяются? | Компилятор гарантирует валидность? | Нужна повторная валидация? |
|----------|:---:|:---:|:---:|
| Проверки рантайма (if/assert) | ✅ | ❌ | На границе каждой функции |
| Валидированный newtype + `TryFrom` | ✅ | ✅ | Никогда — тип служит доказательством |

Правило: **парсите на границе системы, используйте валидированные типы повсюду внутри.**
Сырые строки, целые числа и срезы байтов поступают в вашу систему, преобразуются в
валидированные типы через `TryFrom`/`FromStr`, и с этого момента система типов
гарантирует их корректность.

### Feature-флаги и условная компиляция

```toml
# Cargo.toml
[features]
default = ["json"]          # Включено по умолчанию
json = ["dep:serde_json"]   # Включает поддержку JSON
xml = ["dep:quick-xml"]     # Включает поддержку XML
full = ["json", "xml"]      # Мета-фича: включает всё

[dependencies]
serde = "1"
serde_json = { version = "1", optional = true }
quick-xml = { version = "0.31", optional = true }
```

```rust
// Условная компиляция на основе фич:
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}

#[cfg(feature = "xml")]
pub fn to_xml<T: serde::Serialize>(value: &T) -> String {
    quick_xml::se::to_string(value).unwrap()
}

// Ошибка компиляции, если ни одна из требуемых фич не включена:
#[cfg(not(any(feature = "json", feature = "xml")))]
compile_error!("Должна быть включена хотя бы одна фича формата (json, xml)");
```

**Лучшие практики**:
- Минимизируйте количество фич по умолчанию (`default`) — пользователи должны иметь возможность выбирать их явно
- Используйте синтаксис `dep:` (Rust 1.60+) для опциональных зависимостей, чтобы избежать создания неявных фич
- Документируйте фичи в файле README и в документации на уровне крейта

### Организация рабочего пространства (Workspace)

Для больших проектов используйте рабочее пространство (workspace) Cargo для совместного использования зависимостей и артефактов сборки:

```toml
# Корневой Cargo.toml
[workspace]
members = [
    "core",         # Общие типы и типажи
    "parser",       # Библиотека парсинга
    "server",       # Бинарный файл — основное приложение
    "client",       # Клиентская библиотека
    "cli",          # CLI-инструмент (бинарный файл)
]

# Общие версии зависимостей:
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"

# В Cargo.toml каждого участника (member):
# [dependencies]
# serde = { workspace = true }
```

**Преимущества**:

- Единый файл `Cargo.lock` — все крейты используют одинаковые версии зависимостей
- Команда `cargo test --workspace` запускает все тесты
- Общий кэш сборки — компиляция одного крейта ускоряет сборку других
- Четкие границы зависимостей между компонентами

### `.cargo/config.toml`: Конфигурация на уровне проекта

Файл `.cargo/config.toml` (в корне рабочего пространства или в `$HOME/.cargo/`) настраивает поведение Cargo без изменения самого `Cargo.toml`:

```toml
# .cargo/config.toml

# Целевая платформа (target) по умолчанию для этого рабочего пространства
[build]
target = "x86_64-unknown-linux-gnu"

# Кастомный запуск — например, запуск через QEMU для кросс-компилированных бинарников
[target.aarch64-unknown-linux-gnu]
runner = "qemu-aarch64-static"
linker = "aarch64-linux-gnu-gcc"

# Алиасы (псевдонимы) Cargo — кастомные сокращения команд
[alias]
xt = "test --workspace --release"        # cargo xt = запустить все тесты в release-режиме
ci = "clippy --workspace -- -D warnings" # cargo ci = запустить clippy и считать предупреждения ошибками
cov = "llvm-cov --workspace"             # cargo cov = расчет покрытия кода (требуется cargo-llvm-cov)

# Переменные окружения для скриптов сборки (build.rs)
[env]
IPMI_LIB_PATH = "/usr/lib/bmc"

# Использование кастомного реестра (для внутренних пакетов)
# [registries.internal]
# index = "https://gitlab.internal/crates/index"
```

Распространенные паттерны конфигурации:

| Настройка | Назначение | Пример |
|---------|---------|---------|
| `[build] target` | Целевая платформа по умолчанию | `x86_64-unknown-linux-musl` для статических сборок |
| `[target.X] runner` | Как запускать бинарный файл | `"qemu-aarch64-static"` для кросс-компиляции |
| `[target.X] linker` | Какой компоновщик использовать | `"aarch64-linux-gnu-gcc"` |
| `[alias]` | Пользовательские подкоманды `cargo` | `xt = "test --workspace"` |
| `[env]` | Переменные окружения во время сборки | Пути к библиотекам, переключатели фич |
| `[net] offline` | Запретить доступ к сети | `true` для изолированных (air-gapped) сборок |

### Переменные окружения на этапе компиляции: `env!()` и `option_env!()`

Rust позволяет внедрять переменные окружения в бинарный файл на этапе компиляции. Это полезно для хранения строк версий, метаданных сборки и конфигурации:

```rust
// env!() — вызывает панику на этапе компиляции, если переменная отсутствует
const VERSION: &str = env!("CARGO_PKG_VERSION"); // "0.1.0" из Cargo.toml
const PKG_NAME: &str = env!("CARGO_PKG_NAME");   // имя крейта из Cargo.toml

// option_env!() — возвращает Option<&str>, не паникует при отсутствии переменной
const BUILD_SHA: Option<&str> = option_env!("GIT_SHA");
const BUILD_TIME: Option<&str> = option_env!("BUILD_TIMESTAMP");

fn print_version() {
    println!("{PKG_NAME} v{VERSION}");
    if let Some(sha) = BUILD_SHA {
        println!("  commit: {sha}");
    }
    if let Some(time) = BUILD_TIME {
        println!("  built:  {time}");
    }
}
```

Cargo автоматически устанавливает множество полезных переменных окружения:

| Переменная | Значение | Сценарий использования |
|----------|-------|----------|
| `CARGO_PKG_VERSION` | `"1.2.3"` | Вывод версии |
| `CARGO_PKG_NAME` | `"diag_tool"` | Идентификация бинарного файла |
| `CARGO_PKG_AUTHORS` | Из `Cargo.toml` | Текст раздела "О программе" или справки |
| `CARGO_MANIFEST_DIR` | Абсолютный путь к `Cargo.toml` | Поиск файлов с тестовыми данными |
| `OUT_DIR` | Выходной каталог сборки | Целевой каталог генерации кода в `build.rs` |
| `TARGET` | Тройка целевой платформы (target triple) | Платформозависимая логика в `build.rs` |

Вы можете устанавливать собственные переменные окружения из `build.rs`:
```rust
// build.rs
fn main() {
    println!("cargo::rustc-env=GIT_SHA={}", git_sha());
    println!("cargo::rustc-env=BUILD_TIMESTAMP={}", timestamp());
}
```

### `cfg_attr`: Условные атрибуты

`cfg_attr` применяет атрибут **только тогда**, когда условие истинно. Это более точечный инструмент по сравнению с `#[cfg()]`, который включает или исключает элементы целиком:

```rust
// Выполняем автодеревацию (derive) Serialize только при включенной фиче "serde":
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[derive(Debug, Clone)]
pub struct DiagResult {
    pub fc: u32,
    pub passed: bool,
    pub message: String,
}
// Без фичи "serde": зависимость от serde вообще не требуется
// С фичей "serde": DiagResult поддерживает сериализацию

// Условный атрибут для тестирования:
#[cfg_attr(test, derive(PartialEq))]  // Выводить PartialEq только для тестовых сборок
pub struct LargeStruct { /* ... */ }

// Платформозависимые атрибуты функций:
#[cfg_attr(target_os = "linux", link_name = "ioctl")]
#[cfg_attr(target_os = "freebsd", link_name = "__ioctl")]
extern "C" fn platform_ioctl(fd: i32, request: u64) -> i32;
```

| Шаблон | Что делает |
|---------|-------------|
| `#[cfg(feature = "x")]` | Включает/исключает элемент целиком |
| `#[cfg_attr(feature = "x", derive(Foo))]` | Добавляет `derive(Foo)` только при включенной фиче "x" |
| `#[cfg_attr(test, allow(unused))]` | Подавляет предупреждения только в тестовых сборках |
| `#[cfg_attr(doc, doc = "...")]` | Документация видна только при сборке через `cargo doc` |

### `cargo deny` и `cargo audit`: безопасность цепочки поставок

```bash
# Установка инструментов аудита безопасности
cargo install cargo-deny
cargo install cargo-audit

# Проверка зависимостей на наличие известных уязвимостей
cargo audit

# Комплексные проверки: лицензии, запреты, рекомендации, источники
cargo deny check
```

Настройте `cargo deny` с помощью файла `deny.toml` в корне рабочего пространства:

```toml
# deny.toml
[advisories]
vulnerability = "deny"      # Завершать сборку ошибкой при известных уязвимостях
unmaintained = "warn"        # Предупреждать о заброшенных (unmaintained) крейтах

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-3.0"]          # Отклонять копилефт-лицензии

[bans]
multiple-versions = "warn"  # Предупреждать, если используется несколько версий одного крейта
deny = [
    { name = "openssl" },   # Принудительно использовать rustls вместо openssl
]

[sources]
allow-git = []              # Запретить git-зависимости в продакшене
```

| Инструмент | Назначение | Когда запускать |
|------|---------|-------------|
| `cargo audit` | Проверка известных CVE в зависимостях | В CI-пайплайне, перед релизом |
| `cargo deny check` | Лицензии, запреты, рекомендации, источники | В CI-пайплайне |
| `cargo deny check licenses` | Только проверка соответствия лицензий | Перед открытием исходного кода |
| `cargo deny check bans` | Запрет определенных крейтов | Для контроля архитектурных решений |

### Тесты в документации (Doc-тесты)

Комментарии документации в Rust (`///`) могут содержать блоки кода, которые **компилируются и запускаются в качестве тестов**:

```rust
/// Парсит код диагностической ошибки (fault code) из строки.
///
/// # Примеры
///
/// ```
/// use my_crate::parse_fc;
///
/// let fc = parse_fc("FC:12345").unwrap();
/// assert_eq!(fc, 12345);
/// ```
///
/// Некорректный ввод возвращает ошибку:
///
/// ```
/// use my_crate::parse_fc;
///
/// assert!(parse_fc("not-a-fc").is_err());
/// ```
pub fn parse_fc(input: &str) -> Result<u32, ParseError> {
    input.strip_prefix("FC:")
        .ok_or(ParseError::MissingPrefix)?
        .parse()
        .map_err(ParseError::InvalidNumber)
}
```

```bash
cargo test --doc  # Запуск только тестов документации
cargo test        # Запуск юнит-тестов + интеграционных тестов + doc-тестов
```

Документация на уровне модуля использует синтаксис `//!` в самом начале файла:

```rust
//! # Диагностический фреймворк (Diagnostic Framework)
//!
//! Этот крейт предоставляет основное ядро выполнения диагностики.
//! Он поддерживает запуск диагностических тестов, сбор результатов
//! и отправку отчетов в BMC через IPMI.
//!
//! ## Быстрый старт
//!
//! ```no_run
//! use diag_framework::Framework;
//!
//! let mut fw = Framework::new("config.json")?;
//! fw.run_all_tests()?;
//! ```
```

### Бенчмаркинг с использованием Criterion

> **Полное руководство**: Подробную информацию о настройке `criterion`, примерах API и сравнительную таблицу по отношению к `cargo bench` см. в разделе [Бенчмаркинг с использованием criterion](ch14-testing-and-benchmarking-patterns.md#benchmarking-with-criterion) главы 13 («Шаблоны тестирования и бенчмаркинга»). Ниже приведена краткая справка по специфике использования в архитектуре.

При профилировании публичного API вашего крейта размещайте бенчмарки в каталоге `benches/` и концентрируйте их на критических путях («горячих точках») — обычно это парсеры, сериализаторы или границы валидации:

```bash
cargo bench                  # Запустить все бенчмарки
cargo bench -- parse_config  # Запустить конкретный бенчмарк
# Результаты сохраняются в target/criterion/ с отчетами в формате HTML
```

> **Основные выводы — архитектура и проектирование API**
> - Принимайте наиболее общий тип (`impl Into`, `impl AsRef`, `Cow`); возвращайте наиболее специфичный
> - Принцип «парсите, а не валидируйте» (Parse Don't Validate): используйте `TryFrom` для создания типов, корректных по самой своей конструкции
> - `#[non_exhaustive]` для публичных перечислений предотвращает критические изменения при добавлении новых вариантов
> - `#[must_use]` уберегает от случайного игнорирования важных возвращаемых значений

> **См. также:** раздел [Глава 9 — Обработка ошибок](ch10-error-handling-patterns.md) для проектирования типов ошибок в публичных API; [Глава 13 — Тестирование](ch14-testing-and-benchmarking-patterns.md) для тестирования публичного API вашего крейта.

---

### Упражнение: рефакторинг API крейта ★★ (~30 мин)

Выполните рефакторинг следующего API со «строковой типизацией» в версию, использующую `TryFrom`, новые типы (newtypes) и паттерн «строитель»:

```rust,ignore
// ДО: легко ошибиться при использовании
fn create_server(host: &str, port: &str, max_conn: &str) -> Server { ... }
```

Спроектируйте `ServerConfig` с валидированными типами `Host`, `Port` (1–65535) и `MaxConnections` (1–10000), которые отклоняют некорректные значения на этапе парсинга.

<details>
<summary>🔑 Решение</summary>

```rust
#[derive(Debug, Clone)]
struct Host(String);

impl TryFrom<&str> for Host {
    type Error = String;
    fn try_from(s: &str) -> Result<Self, String> {
        if s.is_empty() { return Err("хост не может быть пустым".into()); }
        if s.contains(' ') { return Err("хост не может содержать пробелы".into()); }
        Ok(Host(s.to_string()))
    }
}

#[derive(Debug, Clone, Copy)]
struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = String;
    fn try_from(p: u16) -> Result<Self, String> {
        if p == 0 { return Err("порт должен быть >= 1".into()); }
        Ok(Port(p))
    }
}

#[derive(Debug, Clone, Copy)]
struct MaxConnections(u32);

impl TryFrom<u32> for MaxConnections {
    type Error = String;
    fn try_from(n: u32) -> Result<Self, String> {
        if n == 0 || n > 10_000 {
            return Err(format!("max_connections должно быть в диапазоне 1–10000, получено {n}"));
        }
        Ok(MaxConnections(n))
    }
}

#[derive(Debug)]
struct ServerConfig {
    host: Host,
    port: Port,
    max_connections: MaxConnections,
}

impl ServerConfig {
    fn new(host: Host, port: Port, max_connections: MaxConnections) -> Self {
        ServerConfig { host, port, max_connections }
    }
}

fn main() {
    let config = ServerConfig::new(
        Host::try_from("localhost").unwrap(),
        Port::try_from(8080).unwrap(),
        MaxConnections::try_from(100).unwrap(),
    );
    println!("{config:?}");

    // Невалидные значения отлавливаются на этапе парсинга:
    assert!(Host::try_from("").is_err());
    assert!(Port::try_from(0).is_err());
    assert!(MaxConnections::try_from(99999).is_err());
}
```

</details>

***
