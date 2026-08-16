# Пакеты (crates) и модули в Rust

> **Что вы узнаете:** Как Rust организует код в модули и пакеты (crates) — видимость по умолчанию («приватный по умолчанию»), модификаторы `pub`, рабочие пространства (workspaces) и экосистему `crates.io`. Это заменяет заголовочные файлы C/C++, директиву `#include` и управление зависимостями через CMake.

- Модули являются фундаментальной организационной единицей кода внутри пакетов.
    - Каждый исходный файл (.rs) представляет собой отдельный модуль и может создавать вложенные модули с помощью ключевого слова ```mod```.
    - Все типы в (под)модуле по умолчанию являются **приватными** и не видны извне в том же пакете, если они явно не помечены как ```pub``` (публичные). Область видимости ```pub``` может быть дополнительно ограничена, например, как ```pub(crate)``` и т. д.
    - Даже если тип является публичным, он не становится автоматически видимым в области видимости другого модуля, пока не будет импортирован с помощью ключевого слова ```use```. Дочерние подмодули могут ссылаться на типы в родительской области видимости с помощью ```use super::```.
    - Исходные файлы (.rs) не включаются автоматически в пакет, **если** они явно не указаны в ```main.rs``` (исполняемый файл) или ```lib.rs``` (библиотека).

# Упражнение: Модули и функции
- Мы рассмотрим изменение нашего примера [hello world](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=522d86dbb8c4af71ff2ec081fb76aee7) для вызова другой функции.
    - Как упоминалось ранее, функции объявляются с помощью ключевого слова ```fn```. Символ ```->``` указывает на то, что функция возвращает значение (по умолчанию тип возвращаемого значения — void, то есть пустой кортеж `()`) с типом ```u32``` (беззнаковое 32-битное целое число).
    - Функции разделены областями видимости модулей, то есть две функции с одинаковым именем в двух разных модулях не вызовут конфликта имен.
        - Разделение областями видимости модулей распространяется на все типы (например, структура ```struct foo``` в ```mod a { struct foo; }``` является отдельным типом (```a::foo```) и отличается от структуры в ```mod b { struct foo; }``` (```b::foo```)).

**Стартовый код** — напишите реализации функций:
```rust
mod math {
    // TODO: реализуйте pub fn add(a: u32, b: u32) -> u32
}

fn greet(name: &str) -> String {
    // TODO: вернуть "Hello, <name>! The secret number is <math::add(21,21)>"
    todo!()
}

fn main() {
    println!("{}", greet("Rustacean"));
}
```

<details><summary>Решение (нажмите, чтобы развернуть)</summary>

```rust
mod math {
    pub fn add(a: u32, b: u32) -> u32 {
        a + b
    }
}

fn greet(name: &str) -> String {
    format!("Hello, {}! The secret number is {}", name, math::add(21, 21))
}

fn main() {
    println!("{}", greet("Rustacean"));
}
// Вывод: Hello, Rustacean! The secret number is 42
```

</details>


## Рабочие пространства и пакеты

- Любой крупный проект на Rust должен использовать рабочие пространства (workspaces) для организации составных пакетов (crates).
    - Рабочее пространство — это просто набор локальных пакетов, которые используются для сборки целевых исполняемых файлов. Файл `Cargo.toml` в корне рабочего пространства должен содержать указатели на входящие в него пакеты.

```toml
[workspace]
resolver = "2"
members = ["package1", "package2"]
```

```text
workspace_root/
|-- Cargo.toml      # Конфигурация рабочего пространства
|-- package1/
|   |-- Cargo.toml  # Конфигурация пакета 1
|   `-- src/
|       `-- lib.rs  # Исходный код пакета 1
|-- package2/
|   |-- Cargo.toml  # Конфигурация пакета 2
|   `-- src/
|       `-- main.rs # Исходный код пакета 2
```

---
## Упражнение: Использование рабочих пространств и зависимостей пакетов
- Мы создадим простой пакет и будем использовать его из нашей программы ```hello world```.
- Создайте каталог рабочего пространства:
```bash
mkdir workspace
cd workspace
```
- Создайте файл Cargo.toml и добавьте в него следующее содержимое. Это создаст пустое рабочее пространство:
```toml
[workspace]
resolver = "2"
members = []
```
- Добавьте пакеты (параметр ```cargo new --lib``` указывает на создание библиотеки вместо исполняемого файла):
```bash
cargo new hello
cargo new --lib hellolib
```

## Упражнение: Использование рабочих пространств и зависимостей пакетов
- Взгляните на сгенерированные файлы Cargo.toml в каталогах ```hello``` и ```hellolib```. Обратите внимание, что оба они были добавлены в корневой ```Cargo.toml```.
- Наличие файла ```lib.rs``` в каталоге ```hellolib``` указывает на то, что это пакет библиотеки (настройки конфигурации сборки см. на https://doc.rust-lang.org/cargo/reference/cargo-targets.html).
- Добавление зависимости от ```hellolib``` в ```Cargo.toml``` пакета ```hello```:
```toml
[dependencies]
hellolib = {path = "../hellolib"}
```
- Использование функции ```add()``` из пакета ```hellolib```:
```rust
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
```

<details><summary>Решение (нажмите, чтобы развернуть)</summary>

Полная настройка рабочего пространства:

```bash
# Команды в терминале
mkdir workspace && cd workspace

# Создаем Cargo.toml рабочего пространства
cat > Cargo.toml << 'EOF'
[workspace]
resolver = "2"
members = ["hello", "hellolib"]
EOF

cargo new hello
cargo new --lib hellolib
```

```toml
# hello/Cargo.toml — добавляем зависимость
[dependencies]
hellolib = {path = "../hellolib"}
```

```rust
// hellolib/src/lib.rs — уже содержит функцию add() после cargo new --lib
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
```

```rust,ignore
// hello/src/main.rs
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
// Вывод: Hello, world! 42
```

</details>

# Использование сторонних пакетов из crates.io
- Rust имеет богатую экосистему пакетов от сообщества (см. https://crates.io/).
    - Философия Rust заключается в том, чтобы держать стандартную библиотеку компактной и выносить функциональность в сторонние пакеты.
    - Нет жестких правил использования сторонних пакетов, но практическое правило состоит в том, чтобы убедиться, что пакет имеет достаточный уровень зрелости (на что указывает номер версии) и активно поддерживается. В случае сомнений по поводу пакета проконсультируйтесь с внутренними ресурсами команды.
- Каждый пакет, опубликованный на ```crates.io```, имеет мажорную и минорную версии.
    - Ожидается, что пакеты соблюдают правила семантического версионирования ```SemVer```, описанные здесь: https://doc.rust-lang.org/cargo/reference/semver.html.
    - В двух словах: не должно быть никаких ломающих изменений в пределах одной минорной версии. Например, v0.11 должна быть совместима с v0.15 (но в v0.20 могут появиться ломающие изменения).

# Зависимости пакетов и SemVer
- Пакеты могут определять зависимости от конкретных версий другого пакета, от определенной минорной или мажорной версии, либо указывать, что это не имеет значения. Ниже приведены примеры из файла ```Cargo.toml``` для объявления зависимости от пакета ```rand```.
- Как минимум ```0.10.0```, но подойдет любая версия ```< 0.11.0```:
```toml
[dependencies]
rand = { version = "0.10.0"}
```
- Только ```0.10.0``` и ничего больше:
```toml
[dependencies]
rand = { version = "=0.10.0"}
```
- Не имеет значения; ```cargo``` выберет последнюю версию:
```toml
[dependencies]
rand = { version = "*"}
```
- Справочник: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
----
# Упражнение: Использование пакета rand
- Измените пример ```helloworld```, чтобы он выводил случайное число.
- Используйте команду ```cargo add rand``` для добавления зависимости.
- Ссылайтесь на API по адресу ```https://docs.rs/rand/latest/rand/```.

**Стартовый код** — добавьте это в `main.rs` после выполнения команды `cargo add rand`:
```rust,ignore
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    // TODO: Сгенерировать и вывести случайное число u32 в диапазоне 1..=100
    // TODO: Сгенерировать и вывести случайное значение bool
    // TODO: Сгенерировать и вывести случайное число f64
}
```

<details><summary>Решение (нажмите, чтобы развернуть)</summary>

```rust
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    let n: u32 = rng.random_range(1..=100);
    println!("Random number (1-100): {n}");

    // Генерируем случайное логическое значение
    let b: bool = rng.random();
    println!("Random bool: {b}");

    // Генерируем случайное число с плавающей запятой от 0.0 до 1.0
    let f: f64 = rng.random();
    println!("Random float: {f:.4}");
}
```

</details>

# Файлы Cargo.toml и Cargo.lock
- Как упоминалось ранее, файл Cargo.lock генерируется автоматически на основе Cargo.toml.
    - Основная идея Cargo.lock — гарантировать воспроизводимость сборок. Например, если в ```Cargo.toml``` указана версия ```0.10.0```, cargo имеет право выбрать любую версию ```< 0.11.0```.
    - Файл Cargo.lock содержит *конкретную* версию пакета rand, которая использовалась во время сборки.
    - Рекомендуется включать ```Cargo.lock``` в репозиторий git, чтобы гарантировать воспроизводимость сборок.

## Возможности Cargo для тестирования
- По соглашению модульные тесты в Rust находятся в том же файле с кодом и обычно сгруппированы в отдельный модуль.
    - Тестовый код никогда не включается в сборку финального бинарного файла. Это становится возможным благодаря атрибуту конфигурации ```cfg```. Конфигурации полезны, например, для создания платформозависимого кода (```Linux``` против ```Windows```).
    - Тесты можно запустить с помощью команды ```cargo test```. Справочник: https://doc.rust-lang.org/reference/conditional-compilation.html

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
// Будет включено только во время тестирования
#[cfg(test)]
mod tests {
    use super::*; // Делает видимыми все типы из родительской области видимости
    #[test]
    fn it_works() {
        let result = add(2, 2); // Альтернативный вариант: super::add(2, 2);
        assert_eq!(result, 4);
    }
}
```

# Другие возможности Cargo
- У ```cargo``` есть несколько других полезных функций, включая:
    - ```cargo clippy``` — отличный инструмент статического анализа (линтер) для кода на Rust. В целом, предупреждения следует исправлять (или в редких случаях подавлять, если это действительно оправдано).
    - ```cargo format``` — запускает инструмент ```rustfmt``` для форматирования исходного кода. Использование этого инструмента гарантирует стандартное форматирование отправляемого в репозиторий кода и кладет конец спорам о стиле.
    - ```cargo doc``` — может использоваться для генерации документации из комментариев в стиле ```///```. Документация для всех пакетов на ```crates.io``` генерируется именно таким способом.

### Профили сборки: Управление оптимизацией

В C вы передаете флаги `-O0`, `-O2`, `-Os`, `-flto` компиляторам `gcc`/`clang`. В Rust вы настраиваете профили сборки в файле `Cargo.toml`:

```toml
# Cargo.toml — настройка профилей сборки

[profile.dev]
opt-level = 0          # Без оптимизации (быстрая компиляция, аналогично -O0)
debug = true           # Полные отладочные символы (аналогично -g)

[profile.release]
opt-level = 3          # Максимальная оптимизация (аналогично -O3)
lto = "fat"            # Оптимизация на этапе линковки (LTO, аналогично -flto)
strip = true           # Очистка отладочных символов (аналогично команде strip)
codegen-units = 1      # Один блок кодогенерации — медленнее компиляция, лучше оптимизация
panic = "abort"        # Без таблиц раскрутки стека (меньше размер бинарного файла)
```

| Флаг C/GCC | Ключ в Cargo.toml | Значения |
|------------|---------------|--------|
| `-O0` / `-O2` / `-O3` | `opt-level` | `0`, `1`, `2`, `3`, `"s"`, `"z"` |
| `-flto` | `lto` | `false`, `"thin"`, `"fat"` |
| `-g` / без `-g` | `debug` | `true`, `false`, `"line-tables-only"` |
| команда `strip` | `strip` | `"none"`, `"debuginfo"`, `"symbols"`, `true`/`false` |
| — | `codegen-units` | `1` = лучшая оптимизация, медленнее компиляция |

```bash
cargo build              # Использует профиль [profile.dev]
cargo build --release    # Использует профиль [profile.release]
```

### Скрипты сборки (`build.rs`): Линовка библиотек C

В C вы используете файлы Makefile или CMake для сборки библиотек и генерации кода. Rust использует файл `build.rs` в корне пакета:

```rust
// build.rs — запускается перед компиляцией пакета

fn main() {
    // Линковка системной библиотеки C (как -lbmc_ipmi в gcc)
    println!("cargo::rustc-link-lib=bmc_ipmi");

    // Путь поиска библиотеки (как -L/usr/lib/bmc)
    println!("cargo::rustc-link-search=/usr/lib/bmc");

    // Пересобирать, если изменился заголовочный файл C
    println!("cargo::rerun-if-changed=wrapper.h");
}
```

Вы даже можете компилировать файлы исходного кода C прямо из пакета Rust:

```toml
# Cargo.toml
[build-dependencies]
cc = "1"  # Интеграция с компилятором C
```

```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("src/c_helpers/ipmi_raw.c")
        .include("/usr/include/bmc")
        .compile("ipmi_raw");   // Создает libipmi_raw.a, линкуется автоматически
    println!("cargo::rerun-if-changed=src/c_helpers/ipmi_raw.c");
}
```

| C / Make / CMake | Скрипт Rust `build.rs` |
|-----------------|-----------------|
| `-lfoo` | `println!("cargo::rustc-link-lib=foo")` |
| `-L/path` | `println!("cargo::rustc-link-search=/path")` |
| Компиляция исходников C | `cc::Build::new().file("foo.c").compile("foo")` |
| Генерация кода | Запись файлов в `$OUT_DIR`, затем их подключение через `include!()` |

### Кросс-компиляция

В C кросс-компиляция требует установки отдельного тулчейна (`arm-linux-gnueabihf-gcc`) и настройки Make/CMake. В Rust:

```bash
# Установка целевой платформы для кросс-компиляции
rustup target add aarch64-unknown-linux-gnu

# Выполнение кросс-компиляции
cargo build --target aarch64-unknown-linux-gnu --release
```

Укажите линковщик в файле `.cargo/config.toml`:

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

| Кросс-компиляция C | Аналог в Rust |
|-----------------|-----------------|
| `apt install gcc-aarch64-linux-gnu` | `rustup target add aarch64-unknown-linux-gnu` + установка линковщика |
| `CC=aarch64-linux-gnu-gcc make` | Настройка `.cargo/config.toml`: `[target.X] linker = "..."` |
| `#ifdef __aarch64__` | `#[cfg(target_arch = "aarch64")]` |
| Отдельные цели в Makefile | Команда `cargo build --target ...` |

### Флаги возможностей: Условная компиляция

В C используются директивы `#ifdef` и флаг `-DFOO` для условной компиляции. В Rust используются флаги возможностей (features), определенные в файле `Cargo.toml`:

```toml
# Cargo.toml
[features]
default = ["json"]         # Включено по умолчанию
json = ["dep:serde_json"]  # Необязательная зависимость
verbose = []               # Флаг без зависимостей
gpu = ["dep:cuda-sys"]     # Необязательная поддержка GPU
```

```rust
// Код, защищенный флагами возможностей:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose"))]
macro_rules! verbose {
    ($($arg:tt)*) => {}; // Компилируется в пустоту
}
```

| Препроцессор C | Флаги возможностей Rust |
|---------------|-------------------|
| `gcc -DDEBUG` | `cargo build --features verbose` |
| `#ifdef DEBUG` | `#[cfg(feature = "verbose")]` |
| `#define MAX 100` | `const MAX: u32 = 100;` |
| `#ifdef __linux__` | `#[cfg(target_os = "linux")]` |

### Интеграционные тесты против модульных тестов

Модульные тесты находятся непосредственно рядом с кодом и помечаются `#[cfg(test)]`. **Интеграционные тесты** располагаются в каталоге `tests/` и проверяют **только публичный API** вашего пакета:

```rust
// tests/smoke_test.rs — аннотация #[cfg(test)] не требуется
use my_crate::parse_config;

#[test]
fn parse_valid_config() {
    let config = parse_config("test_data/valid.json").unwrap();
    assert_eq!(config.max_retries, 5);
}
```

| Аспект | Модульные тесты (`#[cfg(test)]`) | Интеграционные тесты (`tests/`) |
|--------|----------------------------|------------------------------|
| Расположение | В том же файле, что и код | Отдельный каталог `tests/` |
| Доступ | Приватные и публичные элементы | **Только публичный API** |
| Запуск | Команда `cargo test` | Команда `cargo test --test smoke_test` |


### Шаблоны и стратегии тестирования

Разработчики встроенного ПО на C обычно пишут тесты с помощью CUnit, CMocka или собственных фреймворков с большим количеством шаблонного кода. Встроенная система тестирования Rust гораздо мощнее. В этом разделе описываются шаблоны, которые понадобятся вам для написания промышленного кода.

#### `#[should_panic]` — Тестирование ожидаемых сбоев

```rust
// Тестирование того, что определенные условия вызывают панику (аналогично сбоям assert в C)
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_bounds_check() {
    let v = vec![1, 2, 3];
    let _ = v[10];  // Должно вызвать панику
}

#[test]
#[should_panic(expected = "temperature exceeds safe limit")]
fn test_thermal_shutdown() {
    fn check_temperature(celsius: f64) {
        if celsius > 105.0 {
            panic!("temperature exceeds safe limit: {celsius}°C");
        }
    }
    check_temperature(110.0);
}
```

#### `#[ignore]` — Медленные или аппаратно-зависимые тесты

```rust
// Пометка тестов, требующих особых условий (аналог #ifdef HARDWARE_TEST в C)
#[test]
#[ignore = "requires GPU hardware"]
fn test_gpu_ecc_scrub() {
    // Этот тест запускается только на машинах с графическими процессорами (GPU)
    // Запуск: cargo test -- --ignored
    // Запуск: cargo test -- --include-ignored  (запуск ВСЕХ тестов)
}
```

#### Тесты, возвращающие Result (замена цепочек `unwrap`)

```rust
// Вместо множества вызовов unwrap(), скрывающих причину реального сбоя:
#[test]
fn test_config_parsing() -> Result<(), Box<dyn std::error::Error>> {
    let json = r#"{"hostname": "node-01", "port": 8080}"#;
    let config: ServerConfig = serde_json::from_str(json)?;  // Использование ? вместо unwrap()
    assert_eq!(config.hostname, "node-01");
    assert_eq!(config.port, 8080);
    Ok(())  // Тест пройден, если выполнение дошло до этой точки без ошибок
}
```

#### Тестовое окружение с функциями-конструкторами

В C используются функции `setUp()`/`tearDown()`. В Rust используются вспомогательные функции и типаж `Drop`:

```rust
struct TestFixture {
    temp_dir: std::path::PathBuf,
    config: Config,
}

impl TestFixture {
    fn new() -> Self {
        let temp_dir = std::env::temp_dir().join(format!("test_{}", std::process::id()));
        std::fs::create_dir_all(&temp_dir).unwrap();
        let config = Config {
            log_dir: temp_dir.clone(),
            max_retries: 3,
            ..Default::default()
        };
        Self { temp_dir, config }
    }
}

impl Drop for TestFixture {
    fn drop(&mut self) {
        // Автоматическая очистка — аналог tearDown() в C, который невозможно забыть вызвать
        let _ = std::fs::remove_dir_all(&self.temp_dir);
    }
}

#[test]
fn test_with_fixture() {
    let fixture = TestFixture::new();
    // Использование fixture.config, fixture.temp_dir...
    assert!(fixture.temp_dir.exists());
    // fixture автоматически удаляется здесь → запускается очистка памяти и ресурсов
}
```

#### Имитация типажей для аппаратных интерфейсов

В C имитация (mocking) оборудования требует трюков с препроцессором или замены указателей на функции. В Rust типажи делают этот процесс естественным:

```rust
// Промышленный типаж для IPMI-коммуникации
trait IpmiTransport {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String>;
}

// Реальная реализация (используется в продакшене)
struct RealIpmi { /* Данные для подключения к BMC */ }
impl IpmiTransport for RealIpmi {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String> {
        // Непосредственное взаимодействие с оборудованием BMC
        todo!("Real IPMI call")
    }
}

// Имитированная реализация (используется в tests)
struct MockIpmi {
    responses: std::collections::HashMap<u8, Vec<u8>>,
}
impl IpmiTransport for MockIpmi {
    fn send_command(&self, cmd: u8, _data: &[u8]) -> Result<Vec<u8>, String> {
        self.responses.get(&cmd)
            .cloned()
            .ok_or_else(|| format!("No mock response for cmd 0x{cmd:02x}"))
    }
}

// Обобщенная функция, которая работает как с реальной, так и с имитированной реализацией
fn read_sensor_temperature(transport: &dyn IpmiTransport) -> Result<f64, String> {
    let response = transport.send_command(0x2D, &[])?;
    if response.len() < 2 {
        return Err("Response too short".into());
    }
    Ok(response[0] as f64 + (response[1] as f64 / 256.0))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_temperature_reading() {
        let mut mock = MockIpmi { responses: std::collections::HashMap::new() };
        mock.responses.insert(0x2D, vec![72, 128]); // 72.5°C

        let temp = read_sensor_temperature(&mock).unwrap();
        assert!((temp - 72.5).abs() < 0.01);
    }

    #[test]
    fn test_short_response() {
        let mock = MockIpmi { responses: std::collections::HashMap::new() };
        // Ответ не настроен → ожидается ошибка
        assert!(read_sensor_temperature(&mock).is_err());
    }
}
```

#### Тестирование на основе свойств с `proptest`

Вместо проверки конкретных значений проверяйте **свойства**, которые должны соблюдаться всегда:

```rust
// Cargo.toml: [dev-dependencies] proptest = "1"
use proptest::prelude::*;

fn parse_sensor_id(s: &str) -> Option<u32> {
    s.strip_prefix("sensor_")?.parse().ok()
}

fn format_sensor_id(id: u32) -> String {
    format!("sensor_{id}")
}

proptest! {
    #[test]
    fn roundtrip_sensor_id(id in 0u32..10000) {
        // Свойство: форматирование с последующим парсингом должно вернуть исходное значение
        let formatted = format_sensor_id(id);
        let parsed = parse_sensor_id(&formatted);
        prop_assert_eq!(parsed, Some(id));
    }

    #[test]
    fn parse_rejects_garbage(s in "[^s].*") {
        // Свойство: строки, не начинающиеся с 's', никогда не должны распарситься
        let result = parse_sensor_id(&s);
        prop_assert!(result.is_none());
    }
}
```

#### Сравнение тестирования в C и Rust

| Тестирование в C | Аналог в Rust |
|-----------|----------------|
| `CUnit`, `CMocka`, собственный фреймворк | Встроенный атрибут `#[test]` + `cargo test` |
| `setUp()` / `tearDown()` | Функция-конструктор + типаж `Drop` |
| Имитация функций через `#ifdef TEST` | Внедрение зависимостей на основе типажей |
| `assert(x == y)` | `assert_eq!(x, y)` с выводом автоматического diff |
| Отдельный исполняемый файл для тестов | Тот же бинарный файл, условная компиляция с `#[cfg(test)]` |
| `valgrind --leak-check=full ./test` | `cargo test` (безопасно по памяти по умолчанию) + `cargo miri test` |
| Покрытие кода: `gcov` / `lcov` | `cargo tarpaulin` or `cargo llvm-cov` |
| Поиск тестов: ручная регистрация | Автоматически — обнаруживается любая функция с `#[test]` |
