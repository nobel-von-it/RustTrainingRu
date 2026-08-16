## Глубокое погружение в семантику C++ → Rust

> **Чему вы научитесь:** Детальному сопоставлению концепций C++, которые не имеют очевидных эквивалентов в Rust — четырем именованным приведениям типов, SFINAE против ограничений типажей (trait bounds), CRTP против ассоциированных типов и другим частым точкам трения при переводе кода.

В разделах ниже рассматривается сопоставление концепций C++, не имеющих очевидного эквивалента 1:1 в Rust. Эти различия часто вызывают затруднения у программистов на C++ в процессе портирования кода.

### Иерархия приведения типов: четыре приведения C++ → эквиваленты в Rust

В C++ есть четыре именованных приведения типов. Rust заменяет их другими, более явными механизмами:

```cpp
// Иерархия приведения типов C++
int i = static_cast<int>(3.14);            // 1. Численное приведение / приведение вверх (up-cast)
Derived* d = dynamic_cast<Derived*>(base); // 2. Динамическое приведение вниз (downcasting) во время выполнения
int* p = const_cast<int*>(cp);              // 3. Снятие константности (const_cast)
auto* raw = reinterpret_cast<char*>(&obj); // 4. Побитовая реинтерпретация
```

| Приведение в C++ | Эквивалент в Rust | Безопасность | Примечания |
|----------|----------------|--------|-------|
| `static_cast` (численное) | ключевое слово `as` | Безопасно, но может приводить к усечению/зацикливанию | `let i = 3.14_f64 as i32;` — усекается до 3 |
| `static_cast` (численное, проверенное) | `From`/`Into` | Безопасно, проверяется на этапе компиляции | `let i: i32 = 42_u8.into();` — только расширение типа |
| `static_cast` (численное, с возможностью ошибки) | `TryFrom`/`TryInto` | Безопасно, возвращает `Result` | `let i: u8 = 300_u16.try_into()?;` — возвращает Err |
| `dynamic_cast` (приведение вниз) | `match` по enum / `Any::downcast_ref` | Безопасно | Сопоставление с образцом для enum; `Any` для объектов типажей (trait objects) |
| `const_cast` | Нет эквивалента | | В Rust нет возможности преобразовать `&` в `&mut` в безопасном коде. Используйте `Cell`/`RefCell` для внутренней мутабельности (interior mutability) |
| `reinterpret_cast` | `std::mem::transmute` | **`unsafe`** | Реинтерпретирует битовое представление. Почти всегда является ошибкой — предпочитайте `from_le_bytes()` и т.д. |

```rust
// Эквиваленты в Rust:

// 1. Численные приведения — предпочитайте From/Into вместо `as`
let widened: u32 = 42_u8.into();             // Безошибочное расширение типа — всегда предпочитайте его
let truncated = 300_u16 as u8;                // ⚠ Значение превращается в 44! Незаметная потеря данных
let checked: Result<u8, _> = 300_u16.try_into(); // Err — безопасное преобразование с возможностью ошибки

// 2. Приведение вниз (Downcast): enum (предпочтительно) или Any (при необходимости сокрытия типов)
use std::any::Any;

fn handle_any(val: &dyn Any) {
    if let Some(s) = val.downcast_ref::<String>() {
        println!("Got string: {s}");
    } else if let Some(n) = val.downcast_ref::<i32>() {
        println!("Got int: {n}");
    }
}

// 3. "const_cast" → внутренняя мутабельность (interior mutability) (unsafe не требуется)
use std::cell::Cell;
struct Sensor {
    read_count: Cell<u32>,  // Изменение состояния через &self
}
impl Sensor {
    fn read(&self) -> f64 {
        self.read_count.set(self.read_count.get() + 1); // Используется &self, а не &mut self
        42.0
    }
}

// 4. reinterpret_cast → transmute (практически никогда не требуется)
// Предпочитайте безопасные альтернативы:
let bytes: [u8; 4] = 0x12345678_u32.to_ne_bytes();  // ✅ Безопасно
let val = u32::from_ne_bytes(bytes);                   // ✅ Безопасно
// unsafe { std::mem::transmute::<u32, [u8; 4]>(val) } // ❌ Избегайте этого
```

> **Рекомендация**: В идиоматичном Rust использование `as` должно быть редким
> (используйте `From`/`Into` для расширения типов, `TryFrom`/`TryInto` для сужения),
> `transmute` должен быть исключительным случаем, а у `const_cast` нет эквивалента,
> так как типы внутренней мутабельности делают его ненужным.

---

### Препроцессор → `cfg`, флаги возможностей (Feature Flags) и `macro_rules!`

C++ в значительной степени полагается на препроцессор для условной компиляции, объявления констант и генерации кода. Rust заменяет все эти механизмы встроенными языковыми возможностями первого класса.

#### Константы `#define` → `const` или `const fn`

```cpp
// C++
#define MAX_RETRIES 5
#define BUFFER_SIZE (1024 * 64)
#define SQUARE(x) ((x) * (x))  // Макрос — текстовая подстановка, нет типобезопасности
```

```rust
// Rust — типобезопасно, имеет область видимости, без текстовой подстановки
const MAX_RETRIES: u32 = 5;
const BUFFER_SIZE: usize = 1024 * 64;
const fn square(x: u32) -> u32 { x * x }  // Вычисляется во время компиляции

// Может использоваться в контексте констант (const contexts):
const AREA: u32 = square(12);  // Вычисляется во время компиляции
static BUFFER: [u8; BUFFER_SIZE] = [0; BUFFER_SIZE];
```

#### `#ifdef` / `#if` → `#[cfg()]` и `cfg!()`

```cpp
// C++
#ifdef DEBUG
    log_verbose("Step 1 complete");
#endif

#if defined(LINUX) && !defined(ARM)
    use_x86_path();
#else
    use_generic_path();
#endif
```

```rust
// Rust — условная компиляция на основе атрибутов
#[cfg(debug_assertions)]
fn log_verbose(msg: &str) { eprintln!("[VERBOSE] {msg}"); }

#[cfg(not(debug_assertions))]
fn log_verbose(_msg: &str) { /* вырезается компилятором в релизной сборке */ }

// Объединение условий:
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn use_x86_path() { /* ... */ }

#[cfg(not(all(target_os = "linux", target_arch = "x86_64")))]
fn use_generic_path() { /* ... */ }

// Проверка во время выполнения (условие по-прежнему вычисляется во время компиляции, но его можно использовать в выражениях):
if cfg!(target_os = "windows") {
    println!("Running on Windows");
}
```

#### Флаги возможностей (Feature flags) в `Cargo.toml`

```toml
# Cargo.toml — замена #ifdef FEATURE_FOO
[features]
default = ["json"]
json = ["dep:serde_json"]       # Необязательная зависимость
verbose-logging = []            # Флаг без дополнительных зависимостей
gpu-support = ["dep:cuda-sys"]  # Необязательная поддержка GPU
```

```rust
// Условный код на основе флагов возможностей:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose-logging")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose-logging"))]
macro_rules! verbose {
    ($($arg:tt)*) => { }; // Компилируется в пустоту
}
```

#### `#define MACRO(x)` → `macro_rules!`

```cpp
// C++ — текстовая подстановка, крайне подверженная ошибкам
#define DIAG_CHECK(cond, msg) \
    do { if (!(cond)) { log_error(msg); return false; } } while(0)
```

```rust
// Rust — гигиенические макросы, проверяемые по типам, работающие на уровне синтаксического дерева
macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            log_error($msg);
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_test() -> Result<(), DiagError> {
    diag_check!(temperature < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    Ok(())
}
```

| Препроцессор C++ | Эквивалент в Rust | Преимущество |
|-----------------|----------------|-----------|
| `#define PI 3.14` | `const PI: f64 = 3.14;` | Типизировано, имеет область видимости, видно в отладчике |
| `#define MAX(a,b) ((a)>(b)?(a):(b))` | `macro_rules!` или обобщенная функция `fn max<T: Ord>` | Нет багов с повторным вычислением аргументов |
| `#ifdef DEBUG` | `#[cfg(debug_assertions)]` | Проверяется компилятором, нет риска опечатки |
| `#ifdef FEATURE_X` | `#[cfg(feature = "x")]` | Cargoрует возможности; учитываются зависимости |
| `#include "header.h"` | `mod module;` + `use module::Item;` | Нет необходимости в include guards, нет циклического включения |
| `#pragma once` | Не требуется | Каждый файл `.rs` является модулем — подключается ровно один раз |

---

### Заголовочные файлы и `#include` → Модули и `use`

В C++ модель компиляции строится вокруг текстового включения файлов:

```cpp
// widget.h — каждая единица трансляции, использующая Widget, включает этот файл
#pragma once
#include <string>
#include <vector>

class Widget {
public:
    Widget(std::string name);
    void activate();
private:
    std::string name_;
    std::vector<int> data_;
};
```

```cpp
// widget.cpp — отдельное определение
#include "widget.h"
Widget::Widget(std::string name) : name_(std::move(name)) {}
void Widget::activate() { /* ... */ }
```

В Rust **нет заглавных файлов, опережающих объявлений (forward declarations) и макросов защиты от повторного включения (include guards)**:

```rust
// src/widget.rs — объявление И определение в одном файле
pub struct Widget {
    name: String,         // Приватное по умолчанию
    data: Vec<i32>,
}

impl Widget {
    pub fn new(name: String) -> Self {
        Widget { name, data: Vec::new() }
    }
    pub fn activate(&self) { /* ... */ }
}
```

```rust
// src/main.rs — импорт по пути модуля
mod widget;  // Указывает компилятору включить src/widget.rs
use widget::Widget;

fn main() {
    let w = Widget::new("sensor".to_string());
    w.activate();
}
```

| C++ | Rust | Почему это лучше |
|-----|------|-----------------|
| `#include "foo.h"` | `mod foo;` в родительском файле + `use foo::Item;` | Нет текстового включения, нет нарушений правила одного определения (ODR) |
| `#pragma once` / include guards | Не требуется | Каждый файл `.rs` является модулем и компилируется один раз |
| Опережающие объявления | Не требуется | Компилятор видит весь крейт; порядок объявления не важен |
| `class Foo;` (неполный тип) | Не требуется | Нет разделения на объявление и определение |
| `.h` + `.cpp` для каждого класса | Один файл `.rs` | Исключены ошибки рассогласования объявлений и определений |
| `using namespace std;` | `use std::collections::HashMap;` | Всегда явно — нет засорения глобального пространства имен |
| Вложенные `namespace a::b` | Вложенные `mod a { mod b { } }` или структура папок `a/b.rs` | Файловая система отражает дерево модулей |

---

### `friend` и управление доступом → Видимость на уровне модулей

C++ использует `friend` для предоставления определенным классам или функциям доступа к приватным членам. В Rust нет ключевого слова `friend` — вместо этого **область видимости (приватность) привязана к модулю**:

```cpp
// C++
class Engine {
    friend class Car;   // Car может обращаться к приватным членам
    int rpm_;
    void set_rpm(int r) { rpm_ = r; }
public:
    int rpm() const { return rpm_; }
};
```

```rust
// Rust — элементы одного модуля имеют доступ ко всем полям, friend не требуется
mod vehicle {
    pub struct Engine {
        rpm: u32,  // Приватно для модуля (не для структуры!)
    }

    impl Engine {
        pub fn new() -> Self { Engine { rpm: 0 } }
        pub fn rpm(&self) -> u32 { self.rpm }
    }

    pub struct Car {
        engine: Engine,
    }

    impl Car {
        pub fn new() -> Self { Car { engine: Engine::new() } }
        pub fn accelerate(&mut self) {
            self.engine.rpm = 3000; // ✅ Тот же модуль — прямой доступ к полю
        }
        pub fn rpm(&self) -> u32 {
            self.engine.rpm  // ✅ Тот же модуль — доступ к приватному полю на чтение
        }
    }
}

fn main() {
    let mut car = vehicle::Car::new();
    car.accelerate();
    // car.engine.rpm = 9000;  // ❌ Ошибка компиляции: `engine` приватно
    println!("RPM: {}", car.rpm()); // ✅ Публичный метод у Car
}
```

| Доступ в C++ | Эквивалент в Rust | Область видимости |
|-----------|----------------|-------|
| `private` | (по умолчанию, без ключевого слова) | Доступно только в пределах текущего модуля |
| `protected` | Нет прямого эквивалента | Используйте `pub(super)` для доступа из родительского модуля |
| `public` | `pub` | Доступно отовсюду |
| `friend class Foo` | Поместите `Foo` в тот же модуль | Приватность на уровне модулей заменяет friend |
| — | `pub(crate)` | Видимо внутри текущего крейта, но не для внешних зависимостей |
| — | `pub(super)` | Видимо только для родительского модуля |
| — | `pub(in crate::path)` | Видимо внутри определенного поддерева модулей |

> **Ключевая идея**: В C++ приватность привязана к классу. В Rust приватность привязана к модулю. Это означает, что вы управляете доступом, группируя связанные типы в одном модуле — расположенные рядом типы имеют полный доступ к приватным полям друг друга.

---

### `volatile` → Атомики и `read_volatile`/`write_volatile`

В C++ ключевое слово `volatile` сообщает компилятору, что операции чтения/записи не должны оптимизироваться (вырезаться). Обычно оно используется для регистров аппаратуры, проецируемых на память (MMIO). **В Rust нет ключевого слова `volatile`.**

```cpp
// C++: volatile для регистров аппаратуры
volatile uint32_t* const GPIO_REG = reinterpret_cast<volatile uint32_t*>(0x4002'0000);
*GPIO_REG = 0x01;              // Запись не оптимизируется
uint32_t val = *GPIO_REG;     // Чтение не оптимизируется
```

```rust
// Rust: явные операции volatile — только в unsafe-коде
use std::ptr;

const GPIO_REG: *mut u32 = 0x4002_0000 as *mut u32;

// SAFETY: GPIO_REG — это корректный адрес ввода-вывода, проецируемый на память.
unsafe {
    ptr::write_volatile(GPIO_REG, 0x01);   // Запись не оптимизируется
    let val = ptr::read_volatile(GPIO_REG); // Чтение не оптимизируется
}
```

Для **совместно используемого состояния при многопоточности** (другое частое применение `volatile` в C++) в Rust используются атомики:

```cpp
// C++: volatile НЕдостаточно для потокобезопасности (частая ошибка!)
volatile bool stop_flag = false;  // ❌ Состояние гонки (Data race) — неопределенное поведение в C++11+

// Корректный C++:
std::atomic<bool> stop_flag{false};
```

```rust
// Rust: атомики — единственный способ разделять изменяемое состояние между потоками
use std::sync::atomic::{AtomicBool, Ordering};

static STOP_FLAG: AtomicBool = AtomicBool::new(false);

// Из другого потока:
STOP_FLAG.store(true, Ordering::Release);

// Проверка:
if STOP_FLAG.load(Ordering::Acquire) {
    println!("Stopping");
}
```

| Применение в C++ | Эквивалент в Rust | Примечания |
|-----------|----------------|-------|
| `volatile` для аппаратных регистров | `ptr::read_volatile` / `ptr::write_volatile` | Требует `unsafe` — корректно для MMIO |
| `volatile` для сигнализации между потоками | `AtomicBool` / `AtomicU32` и т.д. | Использовать `volatile` в C++ для этого тоже некорректно! |
| `std::atomic<T>` | `std::sync::atomic::AtomicT` | Та же семантика, те же типы упорядочивания (orderings) |
| `std::atomic<T>::load(memory_order_acquire)` | `AtomicT::load(Ordering::Acquire)` | Сопоставление 1:1 |

---

### Переменные `static` → `static`, `const`, `LazyLock`, `OnceLock`

#### Базовые `static` и `const`

```cpp
// C++
const int MAX_RETRIES = 5;                    // Константа времени компиляции
static std::string CONFIG_PATH = "/etc/app";  // Статическая инициализация — порядок не определен!
```

```rust
// Rust
const MAX_RETRIES: u32 = 5;                   // Константа времени компиляции, встраивается по месту (inlined)
static CONFIG_PATH: &str = "/etc/app";         // Время жизни 'static, фиксированный адрес
```

#### Проблема порядка инициализации статических переменных (Static initialization order fiasco)

В C++ есть известная проблема: глобальные конструкторы в разных единицах трансляции выполняются в **неопределенном порядке**. Rust полностью избегает этого — статические переменные (`static`) должны быть константами времени компиляции (без вызова конструкторов).

Для глобальных переменных, инициализируемых во время выполнения, используйте `LazyLock` (начиная с Rust 1.80) или `OnceLock`:

```rust
use std::sync::LazyLock;

// Эквивалент C++ `static std::regex` — инициализируется при первом доступе, потокобезопасен
static CONFIG_REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^[a-z]+_diag$").expect("invalid regex")
});

fn is_valid_diag(name: &str) -> bool {
    CONFIG_REGEX.is_match(name)  // Первый вызов инициализирует; последующие выполняются быстро
}
```

```rust
use std::sync::OnceLock;

// OnceLock: инициализируется один раз, значение может быть задано на этапе выполнения
static DB_CONN: OnceLock<String> = OnceLock::new();

fn init_db(connection_string: &str) {
    DB_CONN.set(connection_string.to_string())
        .expect("DB_CONN already initialized");
}

fn get_db() -> &'static str {
    DB_CONN.get().expect("DB not initialized")
}
```

| C++ | Rust | Примечания |
|-----|------|-------|
| `const int X = 5;` | `const X: i32 = 5;` | Обе константы времени компиляции. Rust требует явного указания типа |
| `constexpr int X = 5;` | `const X: i32 = 5;` | Константы `const` в Rust всегда вычисляются во время компиляции (constexpr) |
| `static int count = 0;` (область видимости файла) | `static COUNT: AtomicI32 = AtomicI32::new(0);` | Изменяемые статические переменные требуют `unsafe` или использования атомиков |
| `static std::string s = "hi";` | `static S: &str = "hi";` или `LazyLock<String>` | Отсутствует конструктор времени выполнения для простых случаев |
| `static MyObj obj;` (сложная инициализация) | `static OBJ: LazyLock<MyObj> = LazyLock::new(\|\| { ... });` | Потокобезопасно, лениво, нет проблем с порядком инициализации |
| `thread_local` | `thread_local! { static X: Cell<u32> = Cell::new(0); }` | Та же семантика |

---

### `constexpr` → `const fn`

В C++ ключевое слово `constexpr` помечает функции и переменные для их вычисления во время компиляции. В Rust для этой цели используются `const fn` и `const`:

```cpp
// C++
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // Вычисляется во время компиляции → 120
```

```rust
// Rust
const fn factorial(n: u32) -> u32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}
const VAL: u32 = factorial(5);  // Вычисляется во время компиляции → 120

// Также работает в размерах массивов и шаблонах match:
const LOOKUP: [u32; 5] = [factorial(1), factorial(2), factorial(3),
                           factorial(4), factorial(5)];
```

| C++ | Rust | Примечания |
|-----|------|-------|
| `constexpr int f()` | `const fn f() -> i32` | Тот же смысл — вычисляется во время компиляции |
| Переменная `constexpr` | Переменная `const` | В Rust константы `const` всегда вычисляются на этапе компиляции |
| `consteval` (C++20) | Нет эквивалента | `const fn` может также выполняться во время работы программы (runtime) |
| `if constexpr` (C++17) | Нет эквивалента (используйте `cfg!` или обобщения) | Специализация типажей (trait specialization) покрывает часть этих сценариев |
| `constinit` (C++20) | `static` с константным инициализатором | Статические переменные в Rust по умолчанию должны инициализироваться константой |

> **Текущие ограничения `const fn`** (по состоянию на Rust 1.82):
> - Нет методов типажей (нельзя вызвать `.len()` у `Vec` в контексте const)
> - Нет аллокации в куче (`Box::new`, `Vec::new` не являются const)
> - ~~Нет арифметики с плавающей точкой~~ — **стабилизировано в Rust 1.82**
> - Нельзя использовать циклы `for` (используйте рекурсию или `while` с ручным инкрементом индекса)

---

### SFINAE и `enable_if` → Ограничения типажей (Trait Bounds) и секции `where`

В C++ механизм SFINAE (Substitution Failure Is Not An Error — ошибка подстановки не является ошибкой) лежит в основе условного обобщенного программирования. Он мощный, но печально известен своей нечитаемостью. Rust полностью заменяет его **ограничениями типажей (trait bounds)**:

```cpp
// C++: Условная функция на основе SFINAE (до C++20)
template<typename T,
         std::enable_if_t<std::is_integral_v<T>, int> = 0>
T double_it(T val) { return val * 2; }

template<typename T,
         std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
T double_it(T val) { return val * 2.0; }

// Концепты C++20 — чище, но все еще многословно:
template<std::integral T>
T double_it(T val) { return val * 2; }
```

```rust
// Rust: ограничения типажей — читаемые, композируемые, с отличными сообщениями об ошибках
use std::ops::Mul;

fn double_it<T: Mul<Output = T> + From<u8>>(val: T) -> T {
    val * T::from(2)
}

// Или с использованием секции where для сложных ограничений:
fn process<T>(val: T) -> String
where
    T: std::fmt::Display + Clone + Send,
{
    format!("Processing: {}", val)
}

// Условное поведение через раздельные реализации impl (заменяет перегрузки SFINAE):
trait Describable {
    fn describe(&self) -> String;
}

impl Describable for u32 {
    fn describe(&self) -> String { format!("integer: {self}") }
}

impl Describable for f64 {
    fn describe(&self) -> String { format!("float: {self:.2}") }
}
```

| Метапрограммирование шаблонов C++ | Эквивалент в Rust | Читаемость |
|-----------------------------|----------------|-------------|
| `std::enable_if_t<cond>` | `where T: Trait` | 🟢 Предельно понятный синтаксис |
| `std::is_integral_v<T>` | Ограничение числовым типажом или конкретными типами | 🟢 Нет суффиксов `_v` / `_t` |
| Наборы перегрузок SFINAE | Раздельные блоки `impl Trait for ConcreteType` | 🟢 Каждая реализация независима |
| `if constexpr (std::is_same_v<T, int>)` | Специализация через реализации типажей | 🟢 Разрешается на этапе компиляции |
| Концепт (`concept`) в C++20 | Типаж (`trait`) | 🟢 Практически идентичная цель |
| Секция `requires` | Секция `where` | 🟢 То же положение в сигнатуре, схожий синтаксис |
| Ошибка компиляции глубоко внутри шаблона | Ошибка компиляции в месте вызова при несовпадении типажа | 🟢 Нет каскадов ошибок на 200 строк |

> **Ключевая идея**: Концепты C++20 ближе всего подобрались к типажам Rust. Если вы знакомы с концептами C++20, думайте о типажах Rust как о концептах, которые были полноценной частью языка с версии 1.0, обладая при этом понятной моделью реализации (`trait impl`) вместо утиной типизации.

---

### `std::function` → Указатели на функции, `impl Fn` и `Box<dyn Fn>`

Класс `std::function<R(Args...)>` в C++ представляет собой объект с сокрытием типа (type-erased callable). Rust предлагает три альтернативы со своими преимуществами и недостатками:

```cpp
// C++: универсальное решение (выделение памяти в куче, сокрытие типа)
#include <functional>
std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };
}
```

```rust
// Вариант 1 в Rust: указатель fn — просто, без захвата контекста, без аллокаций
fn add_one(x: i32) -> i32 { x + 1 }
let f: fn(i32) -> i32 = add_one;
println!("{}", f(5)); // 6

// Вариант 2 в Rust: impl Fn — мономорфизировано, нулевые накладные расходы, может захватывать контекст
fn apply(val: i32, f: impl Fn(i32) -> i32) -> i32 { f(val) }
let n = 10;
let result = apply(5, |x| x + n);  // Замыкание захватывает `n`

// Вариант 3 в Rust: Box<dyn Fn> — сокрытие типа, аллокация в куче (как и std::function)
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}
let adder = make_adder(10);
println!("{}", adder(5));  // 15

// Хранение разнородных замыканий (аналог vector<function<int(int)>>):
let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(|x| x + 1),
    Box::new(|x| x * 2),
    Box::new(make_adder(100)),
];
for cb in &callbacks {
    println!("{}", cb(5));  // 6, 10, 105
}
```

| Когда использовать | Эквивалент в C++ | Выбор в Rust |
|------------|---------------|-------------|
| Функция верхнего уровня без захвата контекста | Указатель на функцию | `fn(Args) -> Ret` |
| Обобщенная функция, принимающая вызываемые объекты | Параметр шаблона | `impl Fn(Args) -> Ret` (статическая диспетчеризация) |
| Ограничение типажа в обобщениях | `template<typename F>` | `F: Fn(Args) -> Ret` |
| Сохраняемый вызываемый объект с сокрытием типа | `std::function<R(Args)>` | `Box<dyn Fn(Args) -> Ret>` |
| Обратный вызов (callback), изменяющий свое состояние | `std::function` с изменяемой лямбдой | `Box<dyn FnMut(Args) -> Ret>` |
| Одноразовый обратный вызов (потребляющий ресурсы) | `std::function` (переданная перемещением) | `Box<dyn FnOnce(Args) -> Ret>` |

> **Примечание о производительности**: Использование `impl Fn` имеет нулевые накладные расходы (мономорфизируется подобно шаблонам C++). Вариант с `Box<dyn Fn>` несет те же издержки, что и `std::function` (таблица виртуальных методов + аллокация в куче). Предпочитайте `impl Fn`, если вам не требуется хранить коллекции разнородных вызываемых объектов.

---

### Сопоставление контейнеров: C++ STL → Rust `std::collections`

| Контейнер C++ STL | Эквивалент в Rust | Примечания |
|------------------|----------------|-------|
| `std::vector<T>` | `Vec<T>` | Почти идентичный API. В Rust по умолчанию проверяются границы индексов |
| `std::array<T, N>` | `[T; N]` | Выделенный на стеке массив фиксированного размера |
| `std::deque<T>` | `std::collections::VecDeque<T>` | Кольцевой буфер. Эффективные операции push/pop на обоих концах |
| `std::list<T>` | `std::collections::LinkedList<T>` | Редко используется в Rust — `Vec` почти всегда быстрее |
| `std::forward_list<T>` | Нет эквивалента | Используйте `Vec` или `VecDeque` |
| `std::unordered_map<K, V>` | `std::collections::HashMap<K, V>` | По умолчанию использует `SipHash` (устойчив к DoS-атакам) |
| `std::map<K, V>` | `std::collections::BTreeMap<K, V>` | B-дерево; ключи отсортированы; требуется `K: Ord` |
| `std::unordered_set<T>` | `std::collections::HashSet<T>` | Требуется `T: Hash + Eq` |
| `std::set<T>` | `std::collections::BTreeSet<T>` | Отсортированное множество; требуется `T: Ord` |
| `std::priority_queue<T>` | `std::collections::BinaryHeap<T>` | Двоичная куча, по умолчанию max-heap (как и в C++) |
| `std::stack<T>` | `Vec<T>` с методами `.push()` / `.pop()` | Отдельный тип для стека не требуется |
| `std::queue<T>` | `VecDeque<T>` с методами `.push_back()` / `.pop_front()` | Отдельный тип для очереди не требуется |
| `std::string` | `String` | Гарантированная кодировка UTF-8, без нуль-терминатора в конце |
| `std::string_view` | `&str` | Заимствованный срез (slice) UTF-8 |
| `std::span<T>` (C++20) | `&[T]` / `&mut [T]` | Срезы (slices) в Rust являются полноценными типами первого класса с версии 1.0 |
| `std::tuple<A, B, C>` | `(A, B, C)` | Встроенный синтаксис кортежей, поддерживающий деструктуризацию |
| `std::pair<A, B>` | `(A, B)` | Обычный двухэлементный кортеж |
| `std::bitset<N>` | Нет эквивалента в std | Используйте крейт `bitvec` или массив `[u8; N/8]` |

**Ключевые отличия**:
- `HashMap`/`HashSet` in Rust требуют `K: Hash + Eq` — компилятор проверяет это на уровне типов, в отличие от C++, где использование нехешируемого ключа приводит к сложным ошибкам шаблонов глубоко внутри STL.
- Индексация в `Vec` (`v[i]`) по умолчанию вызывает панику (panic) при выходе за границы. Используйте `.get(i)` для получения `Option<&T>` или итераторы, чтобы полностью исключить проверки границ.
- Отсутствуют аналоги `std::multimap` или `std::multiset` — используйте `HashMap<K, Vec<V>>` or `BTreeMap<K, Vec<V>>`.

---

### Безопасность при исключениях (Exception Safety) → Безопасность при панике (Panic Safety)

В C++ определены три уровня безопасности исключений (гарантии Абрахамса):

| Уровень в C++ | Значение | Эквивалент в Rust |
|----------|---------|----------------|
| **No-throw** (без генерации исключений) | Функция никогда не выбрасывает исключение | Функция никогда не вызывает панику (возвращает `Result`) |
| **Strong** (строгая гарантия: коммит или откат) | Если выбрасывается исключение, состояние программы не меняется | Модель владения делает это естественным — если оператор `?` вызывает ранний возврат, частично созданные значения уничтожаются (drop) |
| **Basic** (базовая гарантия) | Если выбрасывается исключение, инварианты сохраняются | Поведение Rust по умолчанию — вызываются деструкторы (`Drop`), утечек памяти нет |

#### Как помогает модель владения в Rust

```rust
// Строгая гарантия бесплатно — если file.write() завершится ошибкой, config останется без изменений
fn update_config(config: &mut Config, path: &str) -> Result<(), Error> {
    let new_data = fetch_from_network()?; // Err → ранний возврат, config не затронут
    let validated = validate(new_data)?;   // Err → ранний возврат, config не затронут
    *config = validated;                   // Выполняется только при успехе (коммит)
    Ok(())
}
```

В C++ для реализации строгой гарантии требуется ручной откат или идиома copy-and-swap. В Rust распространение ошибок с помощью `?` дает вам строгую гарантию по умолчанию для большинства вариантов кода.

#### `catch_unwind` — аналог `catch(...)` из C++ в Rust

```rust
use std::panic;

// Перехват паники (аналог catch(...) в C++) — требуется крайне редко
let result = panic::catch_unwind(|| {
    // Код, который может вызвать панику
    let v = vec![1, 2, 3];
    v[10]  // Паника! (выход за границы вектора)
});

match result {
    Ok(val) => println!("Got: {val}"),
    Err(_) => eprintln!("Caught a panic — cleaned up"), // Паника перехвачена — ресурсы очищены
}
```

#### `UnwindSafe` — пометка типов как безопасных при раскрутке стека

```rust
use std::panic::UnwindSafe;

// Типы под изменяемой ссылкой &mut по умолчанию НЕ являются UnwindSafe —
// паника могла оставить их в частично измененном состоянии
fn safe_execute<F: FnOnce() + UnwindSafe>(f: F) {
    let _ = std::panic::catch_unwind(f);
}

// Используйте AssertUnwindSafe для обхода этого ограничения после проверки кода:
use std::panic::AssertUnwindSafe;
let mut data = vec![1, 2, 3];
let _ = std::panic::catch_unwind(AssertUnwindSafe(|| {
    data.push(4);
}));
```

| Шаблон исключений в C++ | Эквивалент в Rust |
|-----------------------|-----------------|
| `throw MyException()` | `return Err(MyError::...)` (предпочтительно) или `panic!("...")` |
| `try { } catch (const E& e)` | `match result { Ok(v) => ..., Err(e) => ... }` или оператор `?` |
| `catch (...)` | `std::panic::catch_unwind(...)` |
| `noexcept` | `-> Result<T, E>` (ошибки представляют собой значения, а не исключения) |
| Очистка RAII при раскрутке стека | `Drop::drop()` вызывается при раскрутке стека во время паники |
| `std::uncaught_exceptions()` | `std::thread::panicking()` |
| Флаг компилятора `-fno-exceptions` | `panic = "abort"` в секции [profile] файла `Cargo.toml` |

> **Итог**: В Rust большая часть кода использует `Result<T, E>` вместо исключений, что делает пути обработки ошибок явными и композируемыми. Макрос `panic!` зарезервирован для программных ошибок (таких как срабатывание `assert!`), а не для штатных ситуаций. Это означает, что «безопасность исключений» в основном перестает быть проблемой — система владения автоматически управляет освобождением ресурсов.

---

## Шаблоны миграции с C++ на Rust

### Краткий справочник: Карта идиом C++ → Rust

| **Шаблон C++** | **Идиома Rust** | **Примечания** |
|----------------|---------------|----------|
| `class Derived : public Base` | `enum Variant { A {...}, B {...} }` | Предпочитайте enums для закрытых множеств типов |
| `virtual void method() = 0` | `trait MyTrait { fn method(&self); }` | Используйте для открытых/расширяемых интерфейсов |
| `dynamic_cast<Derived*>(ptr)` | `match value { Variant::A(data) => ..., }` | Исчерпывающе (exhaustive), исключает ошибки во время выполнения |
| `vector<unique_ptr<Base>>` | `Vec<Box<dyn Trait>>` | Только когда действительно необходим полиморфизм |
| `shared_ptr<T>` | `Rc<T>` или `Arc<T>` | В первую очередь рассматривайте `Box<T>` или обычное владение |
| `enable_shared_from_this<T>` | Шаблон «Арена» (`Vec<T>` + индексы) | Полностью устраняет циклы ссылок |
| `Base* m_pFramework` в каждом классе | `fn execute(&mut self, ctx: &mut Context)` | Передавайте контекст в методы, а не сохраняйте указатели |
| `try { } catch (...) { }` | `match result { Ok(v) => ..., Err(e) => ... }` | Или используйте `?` для распространения ошибок |
| `std::optional<T>` | `Option<T>` | Обязывает использовать `match`, невозможно «забыть» обработать `None` |
| Параметр `const std::string&` | Параметр `&str` | Принимает как `String`, так и `&str` |
| `enum class Foo { A, B, C }` | `enum Foo { A, B, C }` | Перечисления в Rust могут также нести ассоциированные данные |
| `auto x = std::move(obj)` | `let x = obj;` | Перемещение является поведением по умолчанию, `std::move` не требуется |
| CMake + make + линтер | `cargo build / test / clippy / fmt` | Один инструмент для всех задач разработки |

### Стратегия миграции
1. **Начинайте с типов данных**: Сначала переведите структуры и перечисления — это заставит вас подумать о владении ресурсами.
2. **Преобразуйте фабрики в перечисления (enums)**: Если фабрика создает различные производные типы, скорее всего, вам нужен `enum` и сопоставление `match`.
3. **Разделяйте «божественные объекты» (god objects)**: Группируйте связанные поля в небольшие специализированные структуры.
4. **Заменяйте указатели заимствованиями**: Преобразуйте сохраненные указатели `Base*` в заимствования `&'a T` с ограниченным временем жизни.
5. **Используйте `Box<dyn Trait>` умеренно**: Только для систем плагинов и создания заглушек (mocking) при тестировании.
6. **Позвольте компилятору вести вас**: Сообщения об ошибках в Rust превосходны — внимательно читайте их.
